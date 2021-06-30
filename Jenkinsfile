pipeline {
  agent {
    label 'X86-64-MULTI'
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '60'))
    parallelsAlwaysFailFast()
  }
  // Input to determine if this is a package check
  parameters {
     string(defaultValue: 'false', description: 'package check run', name: 'PACKAGE_CHECK')
  }
  // Configuration for the variables used for this specific repo
  environment {
    BUILDS_DISCORD=credentials('build_webhook_url')
    GITHUB_TOKEN=credentials('498b4638-2d02-4ce5-832d-8a57d01d97ab')
    GITLAB_TOKEN=credentials('b6f0f1dd-6952-4cf6-95d1-9c06380283f0')
    GITLAB_NAMESPACE=credentials('gitlab-namespace-id')
    BUILD_VERSION_ARG = 'FFMPEG_VERSION'
    LS_USER = 'linuxserver'
    LS_REPO = 'docker-ffmpeg'
    CONTAINER_NAME = 'ffmpeg'
    DOCKERHUB_IMAGE = 'lsiobase/ffmpeg'
    DEV_DOCKERHUB_IMAGE = 'lsiodev/ffmpeg'
    PR_DOCKERHUB_IMAGE = 'lspipepr/ffmpeg'
    DIST_IMAGE = 'ubuntu'
    MULTIARCH='true'
    CI='false'
  }
  stages {
    // Setup all the basic environment variables needed for the build
    stage("Set ENV Variables base"){
      steps{
        script{
          env.EXIT_STATUS = ''
          env.LS_RELEASE = sh(
            script: '''docker run --rm ghcr.io/linuxserver/alexeiled-skopeo sh -c 'skopeo inspect docker://docker.io/'${DOCKERHUB_IMAGE}':bin 2>/dev/null' | jq -r '.Labels.build_version' | awk '{print $3}' | grep '\\-ls' || : ''',
            returnStdout: true).trim()
          env.LS_RELEASE_NOTES = sh(
            script: '''cat readme-vars.yml | awk -F \\" '/date: "[0-9][0-9].[0-9][0-9].[0-9][0-9]:/ {print $4;exit;}' | sed -E ':a;N;$!ba;s/\\r{0,1}\\n/\\\\n/g' ''',
            returnStdout: true).trim()
          env.GITHUB_DATE = sh(
            script: '''date '+%Y-%m-%dT%H:%M:%S%:z' ''',
            returnStdout: true).trim()
          env.COMMIT_SHA = sh(
            script: '''git rev-parse HEAD''',
            returnStdout: true).trim()
          env.CODE_URL = 'https://github.com/' + env.LS_USER + '/' + env.LS_REPO + '/commit/' + env.GIT_COMMIT
          env.DOCKERHUB_LINK = 'https://hub.docker.com/r/' + env.DOCKERHUB_IMAGE + '/tags/'
          env.PULL_REQUEST = env.CHANGE_ID
          env.TEMPLATED_FILES = 'Jenkinsfile README.md LICENSE ./.github/CONTRIBUTING.md ./.github/FUNDING.yml ./.github/ISSUE_TEMPLATE.md ./.github/PULL_REQUEST_TEMPLATE.md ./.github/workflows/greetings.yml ./.github/workflows/stale.yml'
        }
        script{
          env.LS_RELEASE_NUMBER = sh(
            script: '''echo ${LS_RELEASE} |sed 's/^.*-ls//g' ''',
            returnStdout: true).trim()
        }
        script{
          env.LS_TAG_NUMBER = sh(
            script: '''#! /bin/bash
                       tagsha=$(git rev-list -n 1 ${LS_RELEASE} 2>/dev/null)
                       if [ "${tagsha}" == "${COMMIT_SHA}" ]; then
                         echo ${LS_RELEASE_NUMBER}
                       elif [ -z "${GIT_COMMIT}" ]; then
                         echo ${LS_RELEASE_NUMBER}
                       else
                         echo $((${LS_RELEASE_NUMBER} + 1))
                       fi''',
            returnStdout: true).trim()
        }
      }
    }
    /* #######################
       Package Version Tagging
       ####################### */
    // Grab the current package versions in Git to determine package tag
    stage("Set Package tag"){
      steps{
        script{
          env.PACKAGE_TAG = sh(
            script: '''#!/bin/bash
                       if [ -e package_versions.txt ] ; then
                         cat package_versions.txt | md5sum | cut -c1-8
                       else
                         echo none
                       fi''',
            returnStdout: true).trim()
        }
      }
    }
    /* ########################
       External Release Tagging
       ######################## */
    // If this is a custom command to determine version use that command
    stage("Set tag custom bash"){
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: ''' echo 4.4 ''',
            returnStdout: true).trim()
            env.RELEASE_LINK = 'custom_command'
        }
      }
    }
    // Sanitize the release tag and strip illegal docker or github characters
    stage("Sanitize tag"){
      steps{
        script{
          env.EXT_RELEASE_CLEAN = sh(
            script: '''echo ${EXT_RELEASE} | sed 's/[~,%@+;:/]//g' ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is a bin build use live docker endpoints
    stage("Set ENV live build"){
      when {
        branch "bin"
        environment name: 'CHANGE_ID', value: ''
      }
      steps {
        script{
          env.IMAGE = env.DOCKERHUB_IMAGE
          env.GITHUBIMAGE = 'ghcr.io/' + env.LS_USER + '/' + env.CONTAINER_NAME
          env.GITLABIMAGE = 'registry.gitlab.com/linuxserver.io/' + env.LS_REPO + '/' + env.CONTAINER_NAME
          if (env.MULTIARCH == 'true') {
            env.CI_TAGS = 'amd64-' + env.EXT_RELEASE_CLEAN + '-ls' + env.LS_TAG_NUMBER + '|arm32v7-' + env.EXT_RELEASE_CLEAN + '-ls' + env.LS_TAG_NUMBER + '|arm64v8-' + env.EXT_RELEASE_CLEAN + '-ls' + env.LS_TAG_NUMBER
          } else {
            env.CI_TAGS = env.EXT_RELEASE_CLEAN + '-ls' + env.LS_TAG_NUMBER
          }
          env.META_TAG = env.EXT_RELEASE_CLEAN + '-ls' + env.LS_TAG_NUMBER
          env.EXT_RELEASE_TAG = 'version-' + env.EXT_RELEASE_CLEAN
        }
      }
    }
    // If this is a dev build use dev docker endpoints
    stage("Set ENV dev build"){
      when {
        not {branch "bin"}
        environment name: 'CHANGE_ID', value: ''
      }
      steps {
        script{
          env.IMAGE = env.DEV_DOCKERHUB_IMAGE
          env.GITHUBIMAGE = 'ghcr.io/' + env.LS_USER + '/lsiodev-' + env.CONTAINER_NAME
          env.GITLABIMAGE = 'registry.gitlab.com/linuxserver.io/' + env.LS_REPO + '/lsiodev-' + env.CONTAINER_NAME
          if (env.MULTIARCH == 'true') {
            env.CI_TAGS = 'amd64-' + env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA + '|arm32v7-' + env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA + '|arm64v8-' + env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
          } else {
            env.CI_TAGS = env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
          }
          env.META_TAG = env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
          env.EXT_RELEASE_TAG = 'version-' + env.EXT_RELEASE_CLEAN
          env.DOCKERHUB_LINK = 'https://hub.docker.com/r/' + env.DEV_DOCKERHUB_IMAGE + '/tags/'
        }
      }
    }
    // If this is a pull request build use dev docker endpoints
    stage("Set ENV PR build"){
      when {
        not {environment name: 'CHANGE_ID', value: ''}
      }
      steps {
        script{
          env.IMAGE = env.PR_DOCKERHUB_IMAGE
          env.GITHUBIMAGE = 'ghcr.io/' + env.LS_USER + '/lspipepr-' + env.CONTAINER_NAME
          env.GITLABIMAGE = 'registry.gitlab.com/linuxserver.io/' + env.LS_REPO + '/lspipepr-' + env.CONTAINER_NAME
          if (env.MULTIARCH == 'true') {
            env.CI_TAGS = 'amd64-' + env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST + '|arm32v7-' + env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST + '|arm64v8-' + env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
          } else {
            env.CI_TAGS = env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
          }
          env.META_TAG = env.EXT_RELEASE_CLEAN + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
          env.EXT_RELEASE_TAG = 'version-' + env.EXT_RELEASE_CLEAN
          env.CODE_URL = 'https://github.com/' + env.LS_USER + '/' + env.LS_REPO + '/pull/' + env.PULL_REQUEST
          env.DOCKERHUB_LINK = 'https://hub.docker.com/r/' + env.PR_DOCKERHUB_IMAGE + '/tags/'
        }
      }
    }
    // Run ShellCheck
    stage('ShellCheck') {
      when {
        environment name: 'CI', value: 'true'
      }
      steps {
        withCredentials([
          string(credentialsId: 'ci-tests-s3-key-id', variable: 'S3_KEY'),
          string(credentialsId: 'ci-tests-s3-secret-access-key', variable: 'S3_SECRET')
        ]) {
          script{
            env.SHELLCHECK_URL = 'https://ci-tests.linuxserver.io/' + env.IMAGE + '/' + env.META_TAG + '/shellcheck-result.xml'
          }
          sh '''curl -sL https://raw.githubusercontent.com/linuxserver/docker-shellcheck/master/checkrun.sh | /bin/bash'''
          sh '''#! /bin/bash
                set -e
                docker pull ghcr.io/linuxserver/lsiodev-spaces-file-upload:latest
                docker run --rm \
                -e DESTINATION=\"${IMAGE}/${META_TAG}/shellcheck-result.xml\" \
                -e FILE_NAME="shellcheck-result.xml" \
                -e MIMETYPE="text/xml" \
                -v ${WORKSPACE}:/mnt \
                -e SECRET_KEY=\"${S3_SECRET}\" \
                -e ACCESS_KEY=\"${S3_KEY}\" \
                -t ghcr.io/linuxserver/lsiodev-spaces-file-upload:latest \
                python /upload.py'''
        }
      }
    }
    // Exit the build if the Templated files were just updated
    stage('Template-exit') {
      when {
        branch "bin"
        environment name: 'CHANGE_ID', value: ''
        environment name: 'FILES_UPDATED', value: 'true'
        expression {
          env.CONTAINER_NAME != null
        }
      }
      steps {
        script{
          env.EXIT_STATUS = 'ABORTED'
        }
      }
    }
    /* #######################
           GitLab Mirroring
       ####################### */
    // Ping into Gitlab to mirror this repo and have a registry endpoint
    stage("GitLab Mirror"){
      when {
        environment name: 'EXIT_STATUS', value: ''
      }
      steps{
        sh '''curl -H "Content-Type: application/json" -H "Private-Token: ${GITLAB_TOKEN}" -X POST https://gitlab.com/api/v4/projects \
        -d '{"namespace_id":'${GITLAB_NAMESPACE}',\
             "name":"'${LS_REPO}'",
             "mirror":true,\
             "import_url":"https://github.com/linuxserver/'${LS_REPO}'.git",\
             "issues_access_level":"disabled",\
             "merge_requests_access_level":"disabled",\
             "repository_access_level":"enabled",\
             "visibility":"public"}' '''
      } 
    }
    /* ###############
       Build Container
       ############### */
    // Build Docker container for push to LS Repo
    stage('Build-Single') {
      when {
        environment name: 'MULTIARCH', value: 'false'
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        sh "docker build --no-cache --pull -t ${IMAGE}:${META_TAG} \
        --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${META_TAG}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
      }
    }
    // Build MultiArch Docker containers for push to LS Repo
    stage('Build-Multi') {
      when {
        environment name: 'MULTIARCH', value: 'true'
        environment name: 'EXIT_STATUS', value: ''
      }
      parallel {
        stage('Build X86') {
          steps {
            sh "docker build --no-cache --pull -t ${IMAGE}:amd64-${META_TAG} \
            --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${META_TAG}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
          }
        }
        stage('Build ARMHF') {
          agent {
            label 'X86-64-MULTI'
          }
          steps {
            echo 'Logging into Github'
            sh '''#! /bin/bash
                  echo $GITHUB_TOKEN | docker login ghcr.io -u LinuxServer-CI --password-stdin
               '''
            sh "docker build --no-cache --pull -f Dockerfile.armhf -t ${IMAGE}:arm32v7-${META_TAG} \
                         --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${META_TAG}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
            sh "docker tag ${IMAGE}:arm32v7-${META_TAG} ghcr.io/linuxserver/lsiodev-buildcache:arm32v7-${COMMIT_SHA}-${BUILD_NUMBER}"
            retry(5) {
              sh "docker push ghcr.io/linuxserver/lsiodev-buildcache:arm32v7-${COMMIT_SHA}-${BUILD_NUMBER}"
            }
            sh '''docker rmi \
                  ${IMAGE}:arm32v7-${META_TAG} \
                  ghcr.io/linuxserver/lsiodev-buildcache:arm32v7-${COMMIT_SHA}-${BUILD_NUMBER} || :'''
          }
        }
        stage('Build ARM64') {
          agent {
            label 'X86-64-MULTI'
          }
          steps {
            echo 'Logging into Github'
            sh '''#! /bin/bash
                  echo $GITHUB_TOKEN | docker login ghcr.io -u LinuxServer-CI --password-stdin
               '''
            sh "docker build --no-cache --pull -f Dockerfile.aarch64 -t ${IMAGE}:arm64v8-${META_TAG} \
                         --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${META_TAG}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
            sh "docker tag ${IMAGE}:arm64v8-${META_TAG} ghcr.io/linuxserver/lsiodev-buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER}"
            retry(5) {
              sh "docker push ghcr.io/linuxserver/lsiodev-buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER}"
            }
            sh '''docker rmi \
                  ${IMAGE}:arm64v8-${META_TAG} \
                  ghcr.io/linuxserver/lsiodev-buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER} || :'''
          }
        }
      }
    }
    /* #######
       Testing
       ####### */
    // Run Container tests
    stage('Test') {
      when {
        environment name: 'CI', value: 'true'
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        withCredentials([
          string(credentialsId: 'ci-tests-s3-key-id', variable: 'S3_KEY'),
          string(credentialsId: 'ci-tests-s3-secret-access-key	', variable: 'S3_SECRET')
        ]) {
          script{
            env.CI_URL = 'https://ci-tests.linuxserver.io/' + env.IMAGE + '/' + env.META_TAG + '/index.html'
          }
          sh '''#! /bin/bash
                set -e
                docker pull ghcr.io/linuxserver/lsiodev-ci:latest
                if [ "${MULTIARCH}" == "true" ]; then
                  docker pull ghcr.io/linuxserver/lsiodev-buildcache:arm32v7-${COMMIT_SHA}-${BUILD_NUMBER}
                  docker pull ghcr.io/linuxserver/lsiodev-buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER}
                  docker tag ghcr.io/linuxserver/lsiodev-buildcache:arm32v7-${COMMIT_SHA}-${BUILD_NUMBER} ${IMAGE}:arm32v7-${META_TAG}
                  docker tag ghcr.io/linuxserver/lsiodev-buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER} ${IMAGE}:arm64v8-${META_TAG}
                fi
                docker run --rm \
                --shm-size=1gb \
                -v /var/run/docker.sock:/var/run/docker.sock \
                -e IMAGE=\"${IMAGE}\" \
                -e DELAY_START=\"${CI_DELAY}\" \
                -e TAGS=\"${CI_TAGS}\" \
                -e META_TAG=\"${META_TAG}\" \
                -e PORT=\"${CI_PORT}\" \
                -e SSL=\"${CI_SSL}\" \
                -e BASE=\"${DIST_IMAGE}\" \
                -e SECRET_KEY=\"${S3_SECRET}\" \
                -e ACCESS_KEY=\"${S3_KEY}\" \
                -e DOCKER_ENV=\"${CI_DOCKERENV}\" \
                -e WEB_SCREENSHOT=\"${CI_WEB}\" \
                -e WEB_AUTH=\"${CI_AUTH}\" \
                -e WEB_PATH=\"${CI_WEBPATH}\" \
                -e DO_REGION="ams3" \
                -e DO_BUCKET="lsio-ci" \
                -t ghcr.io/linuxserver/lsiodev-ci:latest \
                python /ci/ci.py'''
        }
      }
    }
    /* ##################
         Release Logic
       ################## */
    // If this is an amd64 only image only push a single image
    stage('Docker-Push-Single') {
      when {
        environment name: 'MULTIARCH', value: 'false'
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: '3f9ba4d5-100d-45b0-a3c4-633fd6061207',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          retry(5) {
            sh '''#! /bin/bash
                  set -e
                  echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
                  echo $GITHUB_TOKEN | docker login ghcr.io -u LinuxServer-CI --password-stdin
                  echo $GITLAB_TOKEN | docker login registry.gitlab.com -u LinuxServer.io --password-stdin
                  for PUSHIMAGE in "${GITHUBIMAGE}" "${GITLABIMAGE}" "${IMAGE}"; do
                    docker tag ${IMAGE}:${META_TAG} ${PUSHIMAGE}:${META_TAG}
                    docker tag ${PUSHIMAGE}:${META_TAG} ${PUSHIMAGE}:bin
                    docker tag ${PUSHIMAGE}:${META_TAG} ${PUSHIMAGE}:${EXT_RELEASE_TAG}
                    docker push ${PUSHIMAGE}:bin
                    docker push ${PUSHIMAGE}:${META_TAG}
                    docker push ${PUSHIMAGE}:${EXT_RELEASE_TAG}
                  done
               '''
          }
          sh '''#! /bin/bash
                for DELETEIMAGE in "${GITHUBIMAGE}" "{GITLABIMAGE}" "${IMAGE}"; do
                  docker rmi \
                  ${DELETEIMAGE}:${META_TAG} \
                  ${DELETEIMAGE}:${EXT_RELEASE_TAG} \
                  ${DELETEIMAGE}:bin || :
                done
             '''
        }
      }
    }
    // If this is a multi arch release push all images and define the manifest
    stage('Docker-Push-Multi') {
      when {
        environment name: 'MULTIARCH', value: 'true'
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: '3f9ba4d5-100d-45b0-a3c4-633fd6061207',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          retry(5) {
            sh '''#! /bin/bash
                  set -e
                  echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
                  echo $GITHUB_TOKEN | docker login ghcr.io -u LinuxServer-CI --password-stdin
                  echo $GITLAB_TOKEN | docker login registry.gitlab.com -u LinuxServer.io --password-stdin
                  if [ "${CI}" == "false" ]; then
                    docker pull ghcr.io/linuxserver/lsiodev-buildcache:arm32v7-${COMMIT_SHA}-${BUILD_NUMBER}
                    docker pull ghcr.io/linuxserver/lsiodev-buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER}
                    docker tag ghcr.io/linuxserver/lsiodev-buildcache:arm32v7-${COMMIT_SHA}-${BUILD_NUMBER} ${IMAGE}:arm32v7-${META_TAG}
                    docker tag ghcr.io/linuxserver/lsiodev-buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER} ${IMAGE}:arm64v8-${META_TAG}
                  fi
                  for MANIFESTIMAGE in "${IMAGE}" "${GITLABIMAGE}" "${GITHUBIMAGE}"; do
                    docker tag ${IMAGE}:amd64-${META_TAG} ${MANIFESTIMAGE}:amd64-${META_TAG}
                    docker tag ${IMAGE}:arm32v7-${META_TAG} ${MANIFESTIMAGE}:arm32v7-${META_TAG}
                    docker tag ${IMAGE}:arm64v8-${META_TAG} ${MANIFESTIMAGE}:arm64v8-${META_TAG}
                    docker tag ${MANIFESTIMAGE}:amd64-${META_TAG} ${MANIFESTIMAGE}:amd64-bin
                    docker tag ${MANIFESTIMAGE}:arm32v7-${META_TAG} ${MANIFESTIMAGE}:arm32v7-bin
                    docker tag ${MANIFESTIMAGE}:arm64v8-${META_TAG} ${MANIFESTIMAGE}:arm64v8-bin
                    docker tag ${MANIFESTIMAGE}:amd64-${META_TAG} ${MANIFESTIMAGE}:amd64-${EXT_RELEASE_TAG}
                    docker tag ${MANIFESTIMAGE}:arm32v7-${META_TAG} ${MANIFESTIMAGE}:arm32v7-${EXT_RELEASE_TAG}
                    docker tag ${MANIFESTIMAGE}:arm64v8-${META_TAG} ${MANIFESTIMAGE}:arm64v8-${EXT_RELEASE_TAG}
                    docker push ${MANIFESTIMAGE}:amd64-${META_TAG}
                    docker push ${MANIFESTIMAGE}:arm32v7-${META_TAG}
                    docker push ${MANIFESTIMAGE}:arm64v8-${META_TAG}
                    docker push ${MANIFESTIMAGE}:amd64-bin
                    docker push ${MANIFESTIMAGE}:arm32v7-bin
                    docker push ${MANIFESTIMAGE}:arm64v8-bin
                    docker push ${MANIFESTIMAGE}:amd64-${EXT_RELEASE_TAG}
                    docker push ${MANIFESTIMAGE}:arm32v7-${EXT_RELEASE_TAG}
                    docker push ${MANIFESTIMAGE}:arm64v8-${EXT_RELEASE_TAG}
                    docker manifest push --purge ${MANIFESTIMAGE}:bin || :
                    docker manifest create ${MANIFESTIMAGE}:bin ${MANIFESTIMAGE}:amd64-bin ${MANIFESTIMAGE}:arm32v7-bin ${MANIFESTIMAGE}:arm64v8-bin
                    docker manifest annotate ${MANIFESTIMAGE}:bin ${MANIFESTIMAGE}:arm32v7-bin --os linux --arch arm
                    docker manifest annotate ${MANIFESTIMAGE}:bin ${MANIFESTIMAGE}:arm64v8-bin --os linux --arch arm64 --variant v8
                    docker manifest push --purge ${MANIFESTIMAGE}:${META_TAG} || :
                    docker manifest create ${MANIFESTIMAGE}:${META_TAG} ${MANIFESTIMAGE}:amd64-${META_TAG} ${MANIFESTIMAGE}:arm32v7-${META_TAG} ${MANIFESTIMAGE}:arm64v8-${META_TAG}
                    docker manifest annotate ${MANIFESTIMAGE}:${META_TAG} ${MANIFESTIMAGE}:arm32v7-${META_TAG} --os linux --arch arm
                    docker manifest annotate ${MANIFESTIMAGE}:${META_TAG} ${MANIFESTIMAGE}:arm64v8-${META_TAG} --os linux --arch arm64 --variant v8
                    docker manifest create ${MANIFESTIMAGE}:${EXT_RELEASE_TAG} ${MANIFESTIMAGE}:amd64-${EXT_RELEASE_TAG} ${MANIFESTIMAGE}:arm32v7-${EXT_RELEASE_TAG} ${MANIFESTIMAGE}:arm64v8-${EXT_RELEASE_TAG}
                    docker manifest annotate ${MANIFESTIMAGE}:${EXT_RELEASE_TAG} ${MANIFESTIMAGE}:arm32v7-${EXT_RELEASE_TAG} --os linux --arch arm
                    docker manifest annotate ${MANIFESTIMAGE}:${EXT_RELEASE_TAG} ${MANIFESTIMAGE}:arm64v8-${EXT_RELEASE_TAG} --os linux --arch arm64 --variant v8
                    docker manifest push --purge ${MANIFESTIMAGE}:bin
                    docker manifest push --purge ${MANIFESTIMAGE}:${META_TAG} 
                    docker manifest push --purge ${MANIFESTIMAGE}:${EXT_RELEASE_TAG} 
                  done
               '''
          }
          sh '''#! /bin/bash
                for DELETEIMAGE in "${GITHUBIMAGE}" "${GITLABIMAGE}" "${IMAGE}"; do
                  docker rmi \
                  ${DELETEIMAGE}:amd64-${META_TAG} \
                  ${DELETEIMAGE}:amd64-bin \
                  ${DELETEIMAGE}:arm32v7-${META_TAG} \
                  ${DELETEIMAGE}:arm32v7-bin \
                  ${DELETEIMAGE}:arm64v8-${META_TAG} \
                  ${DELETEIMAGE}:arm64v8-bin || :
                done
                docker rmi \
                ghcr.io/linuxserver/lsiodev-buildcache:arm32v7-${COMMIT_SHA}-${BUILD_NUMBER} \
                ghcr.io/linuxserver/lsiodev-buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER} || :
             '''
        }
      }
    }
    // If this is a public release tag it in the LS Github
    stage('Github-Tag-Push-Release') {
      when {
        branch "bin"
        expression {
          env.LS_RELEASE != env.EXT_RELEASE_CLEAN + '-ls' + env.LS_TAG_NUMBER
        }
        environment name: 'CHANGE_ID', value: ''
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        echo "Pushing New tag for current commit ${EXT_RELEASE_CLEAN}-ls${LS_TAG_NUMBER}"
        sh '''curl -H "Authorization: token ${GITHUB_TOKEN}" -X POST https://api.github.com/repos/${LS_USER}/${LS_REPO}/git/tags \
        -d '{"tag":"'${EXT_RELEASE_CLEAN}'-ls'${LS_TAG_NUMBER}'",\
             "object": "'${COMMIT_SHA}'",\
             "message": "Tagging Release '${EXT_RELEASE_CLEAN}'-ls'${LS_TAG_NUMBER}' to bin",\
             "type": "commit",\
             "tagger": {"name": "LinuxServer Jenkins","email": "jenkins@linuxserver.io","date": "'${GITHUB_DATE}'"}}' '''
        echo "Pushing New release for Tag"
        sh '''#! /bin/bash
              echo "Updating to ${EXT_RELEASE_CLEAN}" > releasebody.json
              echo '{"tag_name":"'${EXT_RELEASE_CLEAN}'-ls'${LS_TAG_NUMBER}'",\
                     "target_commitish": "bin",\
                     "name": "'${EXT_RELEASE_CLEAN}'-ls'${LS_TAG_NUMBER}'",\
                     "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**Remote Changes:**\\n\\n' > start
              printf '","draft": false,"prerelease": true}' >> releasebody.json
              paste -d'\\0' start releasebody.json > releasebody.json.done
              curl -H "Authorization: token ${GITHUB_TOKEN}" -X POST https://api.github.com/repos/${LS_USER}/${LS_REPO}/releases -d @releasebody.json.done'''
      }
    }
    // Use helper container to sync the current README on master to the dockerhub endpoint
    stage('Sync-README') {
      when {
        environment name: 'CHANGE_ID', value: ''
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: '3f9ba4d5-100d-45b0-a3c4-633fd6061207',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          sh '''#! /bin/bash
                set -e
                TEMPDIR=$(mktemp -d)
                docker pull ghcr.io/linuxserver/jenkins-builder:latest
                docker run --rm -e CONTAINER_NAME=${CONTAINER_NAME} -e GITHUB_BRANCH="${BRANCH_NAME}" -v ${TEMPDIR}:/ansible/jenkins ghcr.io/linuxserver/jenkins-builder:latest 
                docker pull ghcr.io/linuxserver/lsiodev-readme-sync
                docker run --rm=true \
                  -e DOCKERHUB_USERNAME=$DOCKERUSER \
                  -e DOCKERHUB_PASSWORD=$DOCKERPASS \
                  -e GIT_REPOSITORY=${LS_USER}/${LS_REPO} \
                  -e DOCKER_REPOSITORY=${IMAGE} \
                  -e GIT_BRANCH=master \
                  -v ${TEMPDIR}/docker-${CONTAINER_NAME}:/mnt \
                  ghcr.io/linuxserver/lsiodev-readme-sync bash -c 'node sync' 
                rm -Rf ${TEMPDIR} '''
        }
      }
    }
    // If this is a Pull request send the CI link as a comment on it
    stage('Pull Request Comment') {
      when {
        not {environment name: 'CHANGE_ID', value: ''}
        environment name: 'CI', value: 'true'
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        sh '''curl -H "Authorization: token ${GITHUB_TOKEN}" -X POST https://api.github.com/repos/${LS_USER}/${LS_REPO}/issues/${PULL_REQUEST}/comments \
        -d '{"body": "I am a bot, here are the test results for this PR: \\n'${CI_URL}' \\n'${SHELLCHECK_URL}'"}' '''
      }
    }
  }
  /* ######################
     Send status to Discord
     ###################### */
  post {
    always {
      script{
        if (env.EXIT_STATUS == "ABORTED"){
          sh 'echo "build aborted"'
        }
        else if (currentBuild.currentResult == "SUCCESS"){
          sh ''' curl -X POST -H "Content-Type: application/json" --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 1681177,\
                 "description": "**Build:**  '${BUILD_NUMBER}'\\n**CI Results:**  '${CI_URL}'\\n**ShellCheck Results:**  '${SHELLCHECK_URL}'\\n**Status:**  Success\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: '${RELEASE_LINK}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],\
                 "username": "Jenkins"}' ${BUILDS_DISCORD} '''
        }
        else {
          sh ''' curl -X POST -H "Content-Type: application/json" --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 16711680,\
                 "description": "**Build:**  '${BUILD_NUMBER}'\\n**CI Results:**  '${CI_URL}'\\n**ShellCheck Results:**  '${SHELLCHECK_URL}'\\n**Status:**  failure\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: '${RELEASE_LINK}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],\
                 "username": "Jenkins"}' ${BUILDS_DISCORD} '''
        }
      }
    }
    cleanup {
      cleanWs()
    }
  }
}
