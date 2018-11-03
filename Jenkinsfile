pipeline {
  agent {
    label 'X86-64-MULTI'
  }
  // Input to determine if this is a package check
  parameters {
     string(defaultValue: 'false', description: 'package check run', name: 'PACKAGE_CHECK')
  }
  // Configuraiton for the variables used for this specific repo
  environment {
    EXT_GIT_BRANCH = 'master'
    EXT_USER = 'bookstackapp'
    EXT_REPO = 'bookstack'
    BUILD_VERSION_ARG = 'BOOKSTACK_RELEASE'
    LS_USER = 'linuxserver'
    LS_REPO = 'docker-bookstack'
    CONTAINER_NAME = 'bookstack'
    DOCKERHUB_IMAGE = 'linuxserver/bookstack'
    DEV_DOCKERHUB_IMAGE = 'lsiodev/bookstack'
    PR_DOCKERHUB_IMAGE = 'lspipepr/bookstack'
    BUILDS_DISCORD = credentials('build_webhook_url')
    GITHUB_TOKEN = credentials('498b4638-2d02-4ce5-832d-8a57d01d97ab')
    DIST_IMAGE = 'alpine'
    MULTIARCH = 'true'
    CI = 'true'
    CI_WEB = 'true'
    CI_PORT = '80'
    CI_SSL = 'false'
    CI_DELAY = '120'
    TEST_MYSQL_HOST = credentials('mysql_test_host')
    TEST_MYSQL_PASSWORD = credentials('mysql_test_password')
    CI_AUTH = 'user:password'
    CI_WEBPATH = ''
  }
  stages {
    // Setup all the basic environment variables needed for the build
    stage("Set ENV Variables base"){
      steps{
        script{
          env.EXIT_STATUS = ''
          env.LS_RELEASE = sh(
            script: '''curl -s https://api.github.com/repos/${LS_USER}/${LS_REPO}/releases/latest | jq -r '. | .tag_name' ''',
            returnStdout: true).trim()
          env.LS_RELEASE_NOTES = sh(
            script: '''git log -1 --pretty=%B | sed -E ':a;N;$!ba;s/\\r{0,1}\\n/\\\\n/g' ''',
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
                       http_code=$(curl --write-out %{http_code} -s -o /dev/null \
                                   https://raw.githubusercontent.com/${LS_USER}/${LS_REPO}/master/package_versions.txt)
                       if [[ "${http_code}" -ne 200 ]] ; then
                         echo none
                       else
                         curl -s \
                           https://raw.githubusercontent.com/${LS_USER}/${LS_REPO}/master/package_versions.txt \
                         | md5sum | cut -c1-8
                       fi''',
            returnStdout: true).trim()
        }
      }
    }
    /* ########################
       External Release Tagging
       ######################## */
    // If this is a stable github release use the latest endpoint from github to determine the ext tag
    stage("Set ENV github_stable"){
     steps{
       script{
         env.EXT_RELEASE = sh(
           script: '''curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/releases/latest | jq -r '. | .tag_name' ''',
           returnStdout: true).trim()
       }
     }
    }
    // If this is a stable or devel github release generate the link for the build message
    stage("Set ENV github_link"){
     steps{
       script{
         env.RELEASE_LINK = 'https://github.com/' + env.EXT_USER + '/' + env.EXT_REPO + '/releases/tag/' + env.EXT_RELEASE
       }
     }
    }
    // If this is a master build use live docker endpoints
    stage("Set ENV live build"){
      when {
        branch "master"
        environment name: 'CHANGE_ID', value: ''
      }
      steps {
        script{
          env.IMAGE = env.DOCKERHUB_IMAGE
          if (env.MULTIARCH == 'true') {
            env.CI_TAGS = 'amd64-' + env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER + '|arm32v6-' + env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER + '|arm64v8-' + env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
          } else {
            env.CI_TAGS = env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
          }
          env.META_TAG = env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
        }
      }
    }
    // If this is a dev build use dev docker endpoints
    stage("Set ENV dev build"){
      when {
        not {branch "master"}
        environment name: 'CHANGE_ID', value: ''
      }
      steps {
        script{
          env.IMAGE = env.DEV_DOCKERHUB_IMAGE
          if (env.MULTIARCH == 'true') {
            env.CI_TAGS = 'amd64-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA + '|arm32v6-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA + '|arm64v8-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
          } else {
            env.CI_TAGS = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
          }
          env.META_TAG = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
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
          if (env.MULTIARCH == 'true') {
            env.CI_TAGS = 'amd64-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST + '|arm32v6-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST + '|arm64v8-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
          } else {
            env.CI_TAGS = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
          }
          env.META_TAG = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
          env.CODE_URL = 'https://github.com/' + env.LS_USER + '/' + env.LS_REPO + '/pull/' + env.PULL_REQUEST
          env.DOCKERHUB_LINK = 'https://hub.docker.com/r/' + env.PR_DOCKERHUB_IMAGE + '/tags/'
        }
      }
    }
    // Use helper container to render a readme from the template if needed
    stage('Update-README') {
      when {
        branch "master"
        environment name: 'CHANGE_ID', value: ''
        expression {
          env.CONTAINER_NAME != null
        }
      }
      steps {
        sh '''#! /bin/bash
              TEMPDIR=$(mktemp -d)
              docker pull linuxserver/doc-builder:latest
              docker run --rm -e CONTAINER_NAME=${CONTAINER_NAME} -v ${TEMPDIR}:/ansible/readme linuxserver/doc-builder:latest
              if [ "$(md5sum ${TEMPDIR}/${CONTAINER_NAME}/README.md | awk '{ print $1 }')" != "$(md5sum README.md | awk '{ print $1 }')" ]; then
                git clone https://github.com/${LS_USER}/${LS_REPO}.git ${TEMPDIR}/${LS_REPO}
                cp ${TEMPDIR}/${CONTAINER_NAME}/README.md ${TEMPDIR}/${LS_REPO}/
                cd ${TEMPDIR}/${LS_REPO}/
                git --git-dir ${TEMPDIR}/${LS_REPO}/.git add README.md
                git --git-dir ${TEMPDIR}/${LS_REPO}/.git commit -m 'Bot Updating README from template'
                git --git-dir ${TEMPDIR}/${LS_REPO}/.git push https://LinuxServer-CI:${GITHUB_TOKEN}@github.com/${LS_USER}/${LS_REPO}.git --all
                echo "true" > /tmp/${COMMIT_SHA}-${BUILD_NUMBER}
              else
                echo "false" > /tmp/${COMMIT_SHA}-${BUILD_NUMBER}
              fi
              rm -Rf ${TEMPDIR}'''
        script{
          env.README_UPDATED = sh(
            script: '''cat /tmp/${COMMIT_SHA}-${BUILD_NUMBER}''',
            returnStdout: true).trim()
        }
      }
    }
    // Exit the build if the Readme was just updated
    stage('README-exit') {
      when {
        branch "master"
        environment name: 'CHANGE_ID', value: ''
        environment name: 'README_UPDATED', value: 'true'
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
        sh "docker build --no-cache -t ${IMAGE}:${META_TAG} \
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
            sh "docker build --no-cache -t ${IMAGE}:amd64-${META_TAG} \
            --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${META_TAG}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
          }
        }
        stage('Build ARMHF') {
          agent {
            label 'ARMHF'
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
              echo 'Logging into DockerHub'
              sh '''#! /bin/bash
                 echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
                 '''
              sh "curl https://lsio-ci.ams3.digitaloceanspaces.com/qemu-arm-static -o qemu-arm-static"
              sh "chmod +x qemu-*"
              sh "docker build --no-cache -f Dockerfile.armhf -t ${IMAGE}:arm32v6-${META_TAG} \
                           --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${META_TAG}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
              sh "docker tag ${IMAGE}:arm32v6-${META_TAG} lsiodev/buildcache:arm32v6-${COMMIT_SHA}-${BUILD_NUMBER}"
              sh "docker push lsiodev/buildcache:arm32v6-${COMMIT_SHA}-${BUILD_NUMBER}"
            }
          }
        }
        stage('Build ARM64') {
          agent {
            label 'ARM64'
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
              echo 'Logging into DockerHub'
              sh '''#! /bin/bash
                 echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
                 '''
              sh "curl https://lsio-ci.ams3.digitaloceanspaces.com/qemu-aarch64-static -o qemu-aarch64-static"
              sh "chmod +x qemu-*"
              sh "docker build --no-cache -f Dockerfile.aarch64 -t ${IMAGE}:arm64v8-${META_TAG} \
                           --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${META_TAG}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
              sh "docker tag ${IMAGE}:arm64v8-${META_TAG} lsiodev/buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER}"
              sh "docker push lsiodev/buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER}"
            }
          }
        }
      }
    }
    // Take the image we just built and dump package versions for comparison
    stage('Update-packages') {
      when {
        branch "master"
        environment name: 'CHANGE_ID', value: ''
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        sh '''#! /bin/bash
              TEMPDIR=$(mktemp -d)
              if [ "${MULTIARCH}" == "true" ]; then
                LOCAL_CONTAINER=${IMAGE}:amd64-${META_TAG}
              else
                LOCAL_CONTAINER=${IMAGE}:${META_TAG}
              fi
              if [ "${DIST_IMAGE}" == "alpine" ]; then
                docker run --rm -v ${TEMPDIR}:/tmp ${LOCAL_CONTAINER} sh -c '\
                  apk info > packages && \
                  apk info -v > versions && \
                  paste -d " " packages versions > /tmp/package_versions.txt'
              elif [ "${DIST_IMAGE}" == "ubuntu" ]; then
                docker run --rm -v ${TEMPDIR}:/tmp ${LOCAL_CONTAINER} sh -c '\
                  apt -qq list --installed | awk "{print \$1,\$2}" > /tmp/package_versions.txt'
              fi
              if [ "$(md5sum ${TEMPDIR}/package_versions.txt | cut -c1-8 )" != "${PACKAGE_TAG}" ]; then
                git clone https://github.com/${LS_USER}/${LS_REPO}.git ${TEMPDIR}/${LS_REPO}
                cp ${TEMPDIR}/package_versions.txt ${TEMPDIR}/${LS_REPO}/
                cd ${TEMPDIR}/${LS_REPO}/
                git --git-dir ${TEMPDIR}/${LS_REPO}/.git add package_versions.txt
                git --git-dir ${TEMPDIR}/${LS_REPO}/.git commit -m 'Bot Updating Package Versions'
                git --git-dir ${TEMPDIR}/${LS_REPO}/.git push https://LinuxServer-CI:${GITHUB_TOKEN}@github.com/${LS_USER}/${LS_REPO}.git --all
                echo "true" > /tmp/packages-${COMMIT_SHA}-${BUILD_NUMBER}
              else
                echo "false" > /tmp/packages-${COMMIT_SHA}-${BUILD_NUMBER}
              fi
              rm -Rf ${TEMPDIR}'''
        script{
          env.PACKAGE_UPDATED = sh(
            script: '''cat /tmp/packages-${COMMIT_SHA}-${BUILD_NUMBER}''',
            returnStdout: true).trim()
        }
      }
    }
    // Exit the build if the package file was just updated
    stage('PACKAGE-exit') {
      when {
        branch "master"
        environment name: 'CHANGE_ID', value: ''
        environment name: 'PACKAGE_UPDATED', value: 'true'
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        script{
          env.EXIT_STATUS = 'ABORTED'
        }
      }
    }
    // Exit the build if this is just a package check and there are no changes to push
    stage('PACKAGECHECK-exit') {
      when {
        branch "master"
        environment name: 'CHANGE_ID', value: ''
        environment name: 'PACKAGE_UPDATED', value: 'false'
        environment name: 'EXIT_STATUS', value: ''
        expression {
          params.PACKAGE_CHECK == 'true'
        }
      }
      steps {
        script{
          env.EXIT_STATUS = 'ABORTED'
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
          string(credentialsId: 'spaces-key', variable: 'DO_KEY'),
          string(credentialsId: 'spaces-secret', variable: 'DO_SECRET')
        ]) {
          sh '''#! /bin/bash
                docker pull lsiodev/ci:latest
                if [ "${MULTIARCH}" == "true" ]; then
                  docker pull lsiodev/buildcache:arm32v6-${COMMIT_SHA}-${BUILD_NUMBER}
                  docker pull lsiodev/buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER}
                  docker tag lsiodev/buildcache:arm32v6-${COMMIT_SHA}-${BUILD_NUMBER} ${IMAGE}:arm32v6-${META_TAG}
                  docker tag lsiodev/buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER} ${IMAGE}:arm64v8-${META_TAG}
                fi
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                -e IMAGE=\"${IMAGE}\" \
                -e DELAY_START=\"${CI_DELAY}\" \
                -e TAGS=\"${CI_TAGS}\" \
                -e META_TAG=\"${META_TAG}\" \
                -e PORT=\"${CI_PORT}\" \
                -e SSL=\"${CI_SSL}\" \
                -e BASE=\"${DIST_IMAGE}\" \
                -e SECRET_KEY=\"${DO_SECRET}\" \
                -e ACCESS_KEY=\"${DO_KEY}\" \
                -e DOCKER_ENV=\"DB_HOST=${TEST_MYSQL_HOST}|DB_DATABASE=bookstack|DB_USERNAME=root|DB_PASSWORD=${TEST_MYSQL_PASSWORD}\" \
                -e WEB_SCREENSHOT=\"${CI_WEB}\" \
                -e WEB_AUTH=\"${CI_AUTH}\" \
                -e WEB_PATH=\"${CI_WEBPATH}\" \
                -e DO_REGION="ams3" \
                -e DO_BUCKET="lsio-ci" \
                -t lsiodev/ci:latest \
                python /ci/ci.py'''
          script{
            env.CI_URL = 'https://lsio-ci.ams3.digitaloceanspaces.com/' + env.IMAGE + '/' + env.META_TAG + '/index.html'
          }
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
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          sh "docker tag ${IMAGE}:${META_TAG} ${IMAGE}:latest"
          sh "docker push ${IMAGE}:latest"
          sh "docker push ${IMAGE}:${META_TAG}"
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
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          sh '''#! /bin/bash
                if [ "${CI}" == "false" ]; then
                  docker pull lsiodev/buildcache:arm32v6-${COMMIT_SHA}-${BUILD_NUMBER}
                  docker pull lsiodev/buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER}
                  docker tag lsiodev/buildcache:arm32v6-${COMMIT_SHA}-${BUILD_NUMBER} ${IMAGE}:arm32v6-${META_TAG}
                  docker tag lsiodev/buildcache:arm64v8-${COMMIT_SHA}-${BUILD_NUMBER} ${IMAGE}:arm64v8-${META_TAG}
                fi'''
          sh "docker tag ${IMAGE}:amd64-${META_TAG} ${IMAGE}:amd64-latest"
          sh "docker tag ${IMAGE}:arm32v6-${META_TAG} ${IMAGE}:arm32v6-latest"
          sh "docker tag ${IMAGE}:arm64v8-${META_TAG} ${IMAGE}:arm64v8-latest"
          sh "docker push ${IMAGE}:amd64-${META_TAG}"
          sh "docker push ${IMAGE}:arm32v6-${META_TAG}"
          sh "docker push ${IMAGE}:arm64v8-${META_TAG}"
          sh "docker push ${IMAGE}:amd64-latest"
          sh "docker push ${IMAGE}:arm32v6-latest"
          sh "docker push ${IMAGE}:arm64v8-latest"
          sh "docker manifest push --purge ${IMAGE}:latest || :"
          sh "docker manifest create ${IMAGE}:latest ${IMAGE}:amd64-latest ${IMAGE}:arm32v6-latest ${IMAGE}:arm64v8-latest"
          sh "docker manifest annotate ${IMAGE}:latest ${IMAGE}:arm32v6-latest --os linux --arch arm"
          sh "docker manifest annotate ${IMAGE}:latest ${IMAGE}:arm64v8-latest --os linux --arch arm64 --variant v8"
          sh "docker manifest push --purge ${IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} || :"
          sh "docker manifest create ${IMAGE}:${META_TAG} ${IMAGE}:amd64-${META_TAG} ${IMAGE}:arm32v6-${META_TAG} ${IMAGE}:arm64v8-${META_TAG}"
          sh "docker manifest annotate ${IMAGE}:${META_TAG} ${IMAGE}:arm32v6-${META_TAG} --os linux --arch arm"
          sh "docker manifest annotate ${IMAGE}:${META_TAG} ${IMAGE}:arm64v8-${META_TAG} --os linux --arch arm64 --variant v8"
          sh "docker manifest push --purge ${IMAGE}:latest"
          sh "docker manifest push --purge ${IMAGE}:${META_TAG}"
        }
      }
    }
    // If this is a public release tag it in the LS Github and push a changelog from external repo and our internal one
    stage('Github-Tag-Push-Release') {
      when {
        branch "master"
        expression {
          env.LS_RELEASE != env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-ls' + env.LS_TAG_NUMBER
        }
        environment name: 'CHANGE_ID', value: ''
        environment name: 'EXIT_STATUS', value: ''
      }
      steps {
        echo "Pushing New tag for current commit ${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}"
        sh '''curl -H "Authorization: token ${GITHUB_TOKEN}" -X POST https://api.github.com/repos/${LS_USER}/${LS_REPO}/git/tags \
        -d '{"tag":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
             "object": "'${COMMIT_SHA}'",\
             "message": "Tagging Release '${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}' to master",\
             "type": "commit",\
             "tagger": {"name": "LinuxServer Jenkins","email": "jenkins@linuxserver.io","date": "'${GITHUB_DATE}'"}}' '''
        echo "Pushing New release for Tag"
        sh '''#! /bin/bash
              curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/releases/latest | jq '. |.body' | sed 's:^.\\(.*\\).$:\\1:' > releasebody.json
              echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                     "target_commitish": "master",\
                     "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                     "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**'${EXT_REPO}' Changes:**\\n\\n' > start
              printf '","draft": false,"prerelease": false}' >> releasebody.json
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
                docker pull lsiodev/readme-sync
                docker run --rm=true \
                  -e DOCKERHUB_USERNAME=$DOCKERUSER \
                  -e DOCKERHUB_PASSWORD=$DOCKERPASS \
                  -e GIT_REPOSITORY=${LS_USER}/${LS_REPO} \
                  -e DOCKER_REPOSITORY=${IMAGE} \
                  -e GIT_BRANCH=master \
                  lsiodev/readme-sync bash -c 'node sync' '''
        }
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
          sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 1681177,\
                 "description": "**Build:**  '${BUILD_NUMBER}'\\n**CI Results:**  '${CI_URL}'\\n**Status:**  Success\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: '${RELEASE_LINK}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],\
                 "username": "Jenkins"}' ${BUILDS_DISCORD} '''
        }
        else {
          sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 16711680,\
                 "description": "**Build:**  '${BUILD_NUMBER}'\\n**CI Results:**  '${CI_URL}'\\n**Status:**  failure\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: '${RELEASE_LINK}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],\
                 "username": "Jenkins"}' ${BUILDS_DISCORD} '''
        }
      }
    }
  }
}
