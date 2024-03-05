def gitref, commit, shortcommit
def containername = 'machine-images'
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
             defaultValue: "quay.io/podman/${containername}",
             trim: true),
      string(name: 'CONTAINER_REGISTRY_STAGING_REPO',
             description: 'Override the staging registry where intermediate images go',
             defaultValue: "quay.io/podman/staging",
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
                                --write-to-file "podman-${commit}.ociarchive.${arch}"
                            """)
                    }
          }]}
      }

      def artifacts = [
         [platform: "qemu", suffix: "qcow2"],
         [platform: "applehv", suffix: "raw"],
         [platform: "hyperv", suffix: "vhdx"]
      ]
      stage("Generate OCI Archive") {

           basearches.each { arch ->
               artifacts.each { artifact ->
                   def workdir = pwd()
                   def jsonContent = """
                   {
                       "osname": "fedora-coreos",
                       "deploy-via-container": "true",
                       "ostree-container": "${workdir}/podman-${commit}.ociarchive.${arch}",
                       "image-type": "${artifact["platform"]}",
                       "container-imgref": "ostree-remote-registry:fedora:quay.io/containers/podman-machine-os:5.0",
                       "metal-image-size": "3072",
                       "cloud-image-size": "10240"
                   }
                   """
                   def file_path="./diskvars-${arch}-${artifact["suffix"]}.json"
                   writeFile file: file_path, text: jsonContent
               }
           }
      }
      parallel basearches.collectEntries{arch -> [arch, {
          pipeutils.withPodmanRemoteArchBuilder(arch: arch) {
          def session = pipeutils.makeCosaRemoteSession(
              env: ["OSBUILD_EXPORT_FORCE_NO_PRESERVE_OWNER"],
              expiration: "300m",
              image: "quay.io/ravanelli/coreos-assembler:podman",
              workdir: WORKSPACE,
          )
          withEnv(["COREOS_ASSEMBLER_REMOTE_SESSION=${session}"]) {
              shwrap("""cosa init --force https://github.com/coreos/fedora-coreos-config.git""")
              shwrap("""cp -Rf /usr/lib/coreos-assembler/osbuild-manifests/* .""")
              stage("Build Artficats  - ${arch}") {
                  shwrap("""
                      cosa remote-session sync ./diskvars-${arch}-*.json  {:,} && \
                      cosa remote-session sync ./coreos.osbuild.${arch}.mpp.yaml {:,} && \
                      cosa remote-session sync ./podman-${commit}.ociarchive.${arch} {:,} && \
                      cosa remote-session sync ./ platform.*.ipp.yaml {:,}
                  """)

                  // First Build qemu and cache it
                  shwrap("""cosa supermin-run --cache \
                         /usr/lib/coreos-assembler/runvm-osbuild \
                         --config ./diskvars-${arch}-qcow2.json \
                         --filepath podman-${commit}.qcow2 \
                         --mpp ./coreos.osbuild.${arch}.mpp.yaml
                  """)
 
                  artifacts.drop(1)
                  parallel artifacts.collectEntries{artifact -> [artifact, {
                      shwrap("""cosa supermin-run --snapshot \
                         /usr/lib/coreos-assembler/runvm-osbuild \
                         --config ./diskvars-${arch}-${artifact["suffix"]}.json \
                         --filepath podman-${commit}.${artifact["suffix"]} \
                         --mpp ./coreos.osbuild.${arch}.mpp.yaml
                      """)

                  }]}
              }
              stage ("Kola") {
                 shwrap("""cosa kola run  --basic-qemu-scenarios --qemu-image=./podman-${commit}.qcow2""")
              }
             stage("Upload Artifact") {
                 parallel push_containers.collectEntries{configname, val -> [configname, {
                     if (!registry_repos?."${configname}"?.'repo') {
                         echo "No registry repo config for ${configname}. Skipping"
                         return
                     }
                     withCredentials([file(variable: 'REGISTRY_SECRET',
                                           credentialsId: 'podman-push-registry-secret')]) {
                         def repo = registry_repos[configname]['repo']
                         def (artifact, metajsonname) = val
                         def tag_args = registry_repos[configname].tags.collect{"--tag=$it"}
                         def v2s2_arg = registry_repos.v2s2 ? "--v2s2" : ""
                         shwrap("""
                         export STORAGE_DRIVER=vfs # https://github.com/coreos/fedora-coreos-pipeline/issues/723#issuecomment-1297668507
                         cosa push-container-manifest --auth=\${REGISTRY_SECRET} \
                             --repo=${repo} ${tag_args.join(' ')} \
                             --artifact=${artifact} --metajsonname=${metajsonname} \
                             --build=${params.VERSION} ${v2s2_arg}
                         """)

             }
          }}
      }]}
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

