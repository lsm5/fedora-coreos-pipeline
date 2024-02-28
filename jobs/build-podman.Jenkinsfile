def gitref, commit, shortcommit
def containername = 'podman-fcos'
node {
    checkout scm
    // these are script global vars
    pipeutils = load("utils.groovy")
    pipecfg = pipeutils.load_pipecfg()
}

properties([
    pipelineTriggers([
        [$class: 'GenericTrigger',
         genericVariables: [
          [
           key: 'PODMAN_MACHINE_GIT_REF',
           value: '$.ref',
           expressionType: 'JSONPath',
           regexpFilter: 'refs/heads/', //Optional, defaults to empty string
           defaultValue: ''  //Optional, defaults to empty string
          ]
         ],
         causeString: 'Triggered on $ref',
         token: 'build-podman',
         tokenCredentialId: '',
         printContributedVariables: true,
         printPostContent: true,
         silentResponse: false,
         regexpFilterText: '$PODMAN_MACHINE_GIT_REF',
         regexpFilterExpression: 'main|rhcos-.*'
        ]
    ]),
    parameters([
      string(name: 'ARCHES',
             description: 'Space-separated list of target architectures',
             defaultValue: "x86_64" + " " + "aarch64", 
             trim: true),
      string(name: 'PODMAN_MACHINE_GIT_URL',
             description: 'Override the coreos-assembler git repo to use',
             defaultValue: "https://github.com/baude/podman-machine-images.git",
             trim: true),
      string(name: 'PODMAN_MACHINE_GIT_REF',
             description: 'Override the coreos-assembler git ref to use',
             defaultValue: "main",
             trim: true),
      string(name: 'CONTAINER_REGISTRY_REPO',
             description: 'Override the registry to push the container to',
             defaultValue: "quay.io/ravanelli/${containername}",
             trim: true),
      string(name: 'CONTAINER_REGISTRY_STAGING_REPO',
             description: 'Override the staging registry where intermediate images go',
             defaultValue: "quay.io/ravanelli/staging",
             trim: true),
      string(name: 'COREOS_ASSEMBLER_IMAGE',
             description: 'Override the coreos-assembler image to use',
             defaultValue: "quay.io/coreos-assembler/coreos-assembler:main",
             trim: true),
      booleanParam(name: 'FORCE',
                   defaultValue: false,
                   description: 'Whether to force a rebuild'),
    ]),
    buildDiscarder(logRotator(
        numToKeepStr: '100',
        artifactNumToKeepStr: '100'
    )),
    durabilityHint('PERFORMANCE_OPTIMIZED')
])

node {
    change = checkout(
        changelog: true,
        poll: false,
        scm: [
            $class: 'GitSCM',
            branches: [[name: "origin/${params.PODMAN_MACHINE_GIT_REF}"]],
            userRemoteConfigs: [[url: params.PODMAN_MACHINE_GIT_URL]],
            extensions: [[$class: 'CloneOption',
                          noTags: true,
                          reference: '',
                          shallow: true]]
        ]
    )

    gitref = params.PODMAN_MACHINE_GIT_REF
    def output = shwrapCapture("git rev-parse HEAD")
    commit = output.substring(0,40)
    shortcommit = commit.substring(0,7)
}

currentBuild.description = "[${gitref}@${shortcommit}] Waiting"

// Get the list of requested architectures to build for
def basearches = params.ARCHES.split() as Set

lock(resource: "build-${containername}") {
    cosaPod(image: params.COREOS_ASSEMBLER_IMAGE,
            memory: "512Mi", kvm: false,
            serviceAccount: "jenkins") {
    timeout(time: 60, unit: 'MINUTES') {
    try {

        currentBuild.description = "[${gitref}@${shortcommit}] Running"

        // By default we will allow re-using cache layers for one day.
        // This is mostly so we can prevent re-downloading the RPMS
        // and repo metadata and over again in a given day for successive
        // builds.
        def cacheTTL = "24h"
        def force = ""
        if (params.FORCE) {
            force = '--force'
            // Also set cacheTTL to 0.1s to allow users an escape hatch
            // to force no cache layer usage.
            cacheTTL = "0.1s"
        }
        stage('Build Podman FCOS') {
             parallel basearches.collectEntries{arch -> [arch, {
                 pipeutils.withPodmanRemoteArchBuilder(arch: arch) {
                     shwrap("""
                     cosa remote-build-container \
                         --arch $arch --cache-ttl ${cacheTTL} \
                         --git-ref $commit ${force} \
                         --git-url ${params.PODMAN_MACHINE_GIT_URL} \
                         --git-sub-dir podman-image-daily \
                         --write-to-file podman-$commit.ociarchive.$arch
                     """)
                 }
             }]}
         }
         stage("Build Artifacts") {
             //def artifacts = pipecfg.default_artifacts.podman ?: []
             def artifacts = ["raw", "vhdx", "qcow2"]
             basearches.each { arch ->
                 artifacts.each { artifact ->
                     def jsonContent = """
                     {
                         "osname": "fedora-coreos",
                         "deploy-via-container": "true",
                         "ostree-container": "podman-$commit.ociarchive.$arch",
                         "image-type": "$artifact",
                         "container-imgref": "ostree-remote-registry:fedora:quay.io/containers/podman-machine-os:5.0",
                         "metal-image-size": "3072",
                         "cloud-image-size": "10240"
                     }
                     """
                     echo jsonContent
                     writeFile file: '/tmp/diskvars-artifact.json', text: jsonContent
                     shwrap("""ls -la /tmp/diskvars-artifact.json""")
                 }
            }

              
             
             parallel basearches.collectEntries{arch -> [arch, {
                 pipeutils.withPodmanRemoteArchBuilder(arch: arch) {
                     parallel artifacts.collectEntries{artifact -> [artifact, {
                         shwrap("""
                          /usr/lib/coreos-assembler/runvm-osbuild \
                          --config /tmp/diskvars-artifact.json \
                          --filepath /tmp/test \
                          --mpp /usr/lib/coreos-assemble/coreos.osbuild.aarch64.mpp.yaml
                         """)
                     }]}
              }
            }
           ]}
         }
        currentBuild.result = 'SUCCESS'

    } catch (e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (currentBuild.result == 'SUCCESS') {
            currentBuild.description = "[${gitref}@${shortcommit}] ⚡"
        } else {
            currentBuild.description = "[${gitref}@${shortcommit}] ❌"
        }
        if (currentBuild.result != 'SUCCESS') {
            message = "build-cosa #${env.BUILD_NUMBER} <${env.BUILD_URL}|:jenkins:> <${env.RUN_DISPLAY_URL}|:ocean:> [${gitref}@${shortcommit}]"
            pipeutils.trySlackSend(message: message)
        }
    }
}}} // cosaPod, timeout, and lock finish here

