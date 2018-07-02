pipeline {
  agent any
  // Configuraiton for the variables used for this specific repo
  environment {
    EXT_RELEASE_TYPE = 'github_stable'
    EXT_GIT_BRANCH = 'master'
    EXT_USER = 'bookstackapp'
    EXT_REPO = 'bookstack'
    EXT_NPM = 'none'
    EXT_PIP = 'none'
    EXT_BLOB = 'none'
    JSON_URL = 'none'
    JSON_PATH = 'none'
    BUILD_VERSION_ARG = 'BOOKSTACK_RELEASE'
    LS_USER = 'linuxserver'
    LS_REPO = 'docker-bookstack'
    DOCKERHUB_IMAGE = 'linuxserver/bookstack'
    DEV_DOCKERHUB_IMAGE = 'lspipetest/bookstack'
    PR_DOCKERHUB_IMAGE = 'lspipepr/bookstack'
    BUILDS_DISCORD = credentials('build_webhook_url')
    GITHUB_TOKEN = credentials('github_token')
    DIST_IMAGE = 'alpine'
    DIST_TAG = '3.7'
    DIST_PACKAGES = 'curl \
                    php7-openssl \
                    php7-pdo_mysql \
                    php7-mbstring \
                    php7-tidy \
                    php7-phar \
                    php7-dom \
                    php7-tokenizer \
                    php7-gd \
                    php7-mysqlnd \
                    php7-tidy \
                    php7-simplexml \
                    tar'
    DIST_REPO = 'none'
    DIST_REPO_PACKAGES = 'none'
    MULTIARCH='true'
  }
  stages {
    // Setup all the basic environment variables needed for the build
    stage("Set ENV Variables base"){
      steps{
        script{
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
          env.CODE_URL = sh(
            script: '''echo https://github.com/${LS_USER}/${LS_REPO}/commit/${GIT_COMMIT}''',
            returnStdout: true).trim()
          env.DOCKERHUB_LINK = sh(
            script: '''echo https://hub.docker.com/r/${DOCKERHUB_IMAGE}/tags/''',
            returnStdout: true).trim()
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
                       # Get the commit for the current tag
                       tagsha=$(git rev-list -n 1 ${LS_RELEASE} 2>/dev/null)
                       # If this is a new commit then increment the LinuxServer release version
                       if [ "${tagsha}" == "${COMMIT_SHA}" ]; then
                         echo ${LS_RELEASE_NUMBER}
                       # If the commit is empty for this job do not increment
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
    // If this is an alpine base image determine the base package tag to use
    stage("Set Package tag Alpine"){
      when {
        expression {
          env.DIST_IMAGE == 'alpine' && env.DIST_PACKAGES != 'none'
        }
      }
      steps{
        echo 'Grabbing the latest alpine base image'
        sh '''docker pull alpine:${DIST_TAG}'''
        echo 'Generating the package hash from the current versions'
        script{
          env.PACKAGE_TAG = sh(
            script: '''docker run --rm alpine:${DIST_TAG} sh -c 'apk update --quiet\
                       && apk info '"${DIST_PACKAGES}"' | md5sum | cut -c1-8' ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is an ubuntu base image determine the base package tag to use
    stage("Set Package tag Ubuntu"){
      when {
        expression {
          env.DIST_IMAGE == 'ubuntu' && env.DIST_PACKAGES != 'none'
        }
      }
      steps{
        echo 'Grabbing the latest alpine base image'
        sh '''docker pull ubuntu:${DIST_TAG}'''
        echo 'Generating the package hash from the current versions'
        script{
          env.PACKAGE_TAG = sh(
            script: '''docker run --rm ubuntu:${DIST_TAG} sh -c\
                       'apt-get --allow-unauthenticated update -qq >/dev/null 2>&1 &&\
                        apt-cache --no-all-versions show '"${DIST_PACKAGES}"' | md5sum | cut -c1-8' ''',
            returnStdout: true).trim()
        }
      }
    }
    // If there are no base packages to tag in this build config set to none
    stage("Set Package tag none"){
      when {
        expression {
          env.DIST_PACKAGES == 'none'
        }
      }
      steps{
        script{
          env.PACKAGE_TAG = sh(
            script: '''echo none''',
            returnStdout: true).trim()
        }
      }
    }
    /* ########################
       External Release Tagging
       ######################## */
    // If this is a stable github release use the latest endpoint from github to determine the ext tag
    stage("Set ENV github_stable"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'github_stable'
        }
      }
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: '''curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/releases/latest | jq -r '. | .tag_name' ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is an os release set release type to none to indicate no external release
    stage("Set ENV os"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'os'
        }
      }
      steps{
        script{
          env.EXT_RELEASE = env.PACKAGE_TAG
          env.RELEASE_LINK = 'none'
        }
      }
    }
    // If this is a stable or devel github release generate the link for the build message
    stage("Set ENV github_link"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'github_stable' || env.EXT_RELEASE_TYPE == 'github_devel'
        }
      }
      steps{
        script{
          env.RELEASE_LINK = sh(
            script: '''echo https://github.com/${EXT_USER}/${EXT_REPO}/releases/tag/${EXT_RELEASE}''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is a deb repo release calculate a hash for the package version
    stage("Set EXT tag deb repo"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'deb_repo'
        }
      }
      steps{
        echo 'Grabbing the latest base image'
        sh '''docker pull ${DIST_IMAGE}:${DIST_TAG}'''
        echo 'Generating the package hash from the current versions'
        script{
          env.EXT_RELEASE = sh(
            script: '''docker run --rm ${DIST_IMAGE}:${DIST_TAG} bash -c\
                       'echo -e "'"${DIST_REPO}"'" > /etc/apt/sources.list.d/check.list \
                        && apt-get --allow-unauthenticated update -qq >/dev/null 2>&1\
                        && apt-cache --no-all-versions show '"${DIST_REPO_PACKAGES}"' | md5sum | cut -c1-8' ''',
            returnStdout: true).trim()
          env.RELEASE_LINK = 'deb_repo'
        }
      }
    }
    // If this is an alpine repo change for external version determine an md5 from the version string
    stage("Set tag Alpine Repo"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'alpine_repo'
        }
      }
      steps{
        echo 'Grabbing the latest alpine base image'
        sh '''docker pull alpine:${DIST_TAG}'''
        echo 'Generating the package hash from the current versions'
        script{
          env.EXT_RELEASE = sh(
            script: '''docker run --rm alpine:${DIST_TAG} sh -c 'apk update --quiet --repository '"${DIST_REPO}"'\
                       && apk info --repository '"${DIST_REPO}"' '"${DIST_REPO_PACKAGES}"' | md5sum | cut -c1-8' ''',
            returnStdout: true).trim()
            env.RELEASE_LINK = 'alpine_repo'
        }
      }
    }
    // If this is a github commit trigger determine the current commit at head
    stage("Set ENV github_commit"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'github_commit'
        }
      }
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: '''curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/commits/${EXT_GIT_BRANCH} | jq -r '. | .sha' | cut -c1-8 ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is a github commit trigger Set the external release link
    stage("Set ENV commit_link"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'github_commit'
        }
      }
      steps{
        script{
          env.RELEASE_LINK = sh(
            script: '''echo https://github.com/${EXT_USER}/${EXT_REPO}/commit/${EXT_RELEASE} ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is a github tag trigger determine the current tag
    stage("Set ENV github_tag"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'github_tag'
        }
      }
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: '''curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/tags | jq -r '.[0] | .name' ''',
            returnStdout: true).trim()
          env.EXT_COMMIT_URL = sh(
            script: '''curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/tags | jq -r '.[0] | .commit.url' ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is a github tag trigger Set the external release link
    stage("Set ENV tag_link"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'github_tag'
        }
      }
      steps{
        script{
          env.RELEASE_LINK = sh(
            script: '''echo https://github.com/${EXT_USER}/${EXT_REPO}/releases/tag/${EXT_RELEASE} ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is a npm version change set the external release verison and link
    stage("Set ENV npm_version"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'npm_version'
        }
      }
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: '''curl -s https://skimdb.npmjs.com/registry/${EXT_NPM} |jq -r '. | .["dist-tags"].latest' ''',
            returnStdout: true).trim()
          env.RELEASE_LINK = sh(
            script: '''echo https://www.npmjs.com/package/${EXT_NPM} ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is a pip version change set the external release verison and link
    stage("Set ENV pip_version"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'pip_version'
        }
      }
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: '''curl -s  https://pypi.python.org/pypi/${EXT_PIP}/json |jq -r '. | .info.version' ''',
            returnStdout: true).trim()
          env.RELEASE_LINK = sh(
            script: '''echo https://pypi.python.org/pypi/${EXT_PIP} ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is a File blob set the ext version based on the remote files md5
    stage("Set ENV external_blob"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'external_blob'
        }
      }
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: '''#! /bin/bash
                       # Make sure the remote file returns a 200 status or fail
                       if [ $(curl -I -sL -w "%{http_code}" ${EXT_BLOB} -o /dev/null) == 200 ]; then
                         curl -s -L ${EXT_BLOB} | md5sum | cut -c1-8
                       else
                         exit 1
                       fi''',
            returnStdout: true).trim()
          env.RELEASE_LINK = sh(
            script: '''echo "Remote_Blob_Change" ''',
            returnStdout: true).trim()
        }
      }
    }
    // If this is a custom json endpoint parse the return to get external tag
    stage("Set ENV custom_json"){
      when {
        expression {
          env.EXT_RELEASE_TYPE == 'custom_json'
        }
      }
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: '''curl -s ${JSON_URL} | jq -r ". | ${JSON_PATH}" ''',
            returnStdout: true).trim()
          env.RELEASE_LINK = sh(
            script: '''echo "${JSON_URL}" ''',
            returnStdout: true).trim()
        }
      }
    }
    /* ###############
       Build Container
       ############### */
     // Build Docker container for push to LS Repo
     stage('Build-Single') {
       when {
         expression {
           env.MULTIARCH == 'false'
         }
       }
       steps {
           echo "Building most current release of ${EXT_REPO}"
           sh "docker build --no-cache -t ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} \
           --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
         }
     }
     // Build MultiArch Docker container for push to LS Repo
     stage('Build-Multi') {
       when {
         expression {
           env.MULTIARCH == 'true'
         }
       }
       steps {
           echo "Building most current release of ${EXT_REPO} x86_64"
           sh "docker build --no-cache -f Dockerfile.amd64 -t ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} \
           --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
           echo "Building most current release of ${EXT_REPO} Arm 32 Bit (Pis)"
           sh "docker build --no-cache -f Dockerfile.armhf -t ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} \
           --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
           echo "Building most current release of ${EXT_REPO} Arm 64 Bit"
           sh "docker build --no-cache -f Dockerfile.aarch64 -t ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} \
           --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
         }
     }
    /* #######
       Testing
       ####### */
    // Run Container tests
    stage('Test') {
      steps {
       echo 'CI Tests for future use'
      }
    }
    /* ##################
       Live Release Logic
       ################## */
    // If this is a public release push this to the live repo triggered by an external repo update or LS repo update on master
    stage('Docker-Push-Release-Single') {
      when {
        branch "master"
        expression {
          env.LS_RELEASE != env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-ls' + env.LS_TAG_NUMBER
        }
        expression{
          env.MULTIARCH == 'false'
        }
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          echo 'First push the latest tag'
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:latest"
          sh "docker push ${DOCKERHUB_IMAGE}:latest"
          echo 'Pushing by release tag'
          sh "docker push ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
        }
      }
    }
    // If this is a public release push this to the live repo triggered by an external repo update or LS repo update on master
    stage('Docker-Push-Release-Multi') {
      when {
        branch "master"
        expression {
          env.LS_RELEASE != env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-ls' + env.LS_TAG_NUMBER
        }
        expression{
          env.MULTIARCH == 'true'
        }
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          echo 'First Tag the releases to latest also'
          sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm64v8-latest"
          echo 'Push all image variants in case someone does not want to use manifests'
          sh "docker push ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
          sh "docker push ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
          sh "docker push ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
          sh "docker push ${DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker push ${DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker push ${DOCKERHUB_IMAGE}:arm64v8-latest"
          echo 'Generate Manifests based on tagging'
          sh "docker manifest create ${DOCKERHUB_IMAGE}:latest ${DOCKERHUB_IMAGE}:amd64-latest ${DOCKERHUB_IMAGE}:arm32v6-latest ${DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker manifest annotate ${DOCKERHUB_IMAGE}:latest ${DOCKERHUB_IMAGE}:arm32v6-latest --os linux --arch arm"
          sh "docker manifest annotate ${DOCKERHUB_IMAGE}:latest ${DOCKERHUB_IMAGE}:arm64v8-latest --os linux --arch arm64 --variant armv8"
          sh "docker manifest create ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
          sh "docker manifest annotate ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} --os linux --arch arm"
          sh "docker manifest annotate ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} --os linux --arch arm64 --variant armv8"
          echo 'Pushing by manifest tags'
          sh "docker manifest push ${DOCKERHUB_IMAGE}:latest"
          sh "docker manifest push ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
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
              if [ ${EXT_RELEASE_TYPE} == 'github_stable' ] || [ ${EXT_RELEASE_TYPE} == 'github_devel' ]; then
                # Grabbing the current release body from external repo
                curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/releases/latest | jq '. |.body' | sed 's:^.\\(.*\\).$:\\1:' > releasebody.json
                # Creating the start of the json payload
                echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "target_commitish": "master",\
                       "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**'${EXT_REPO}' Changes:**\\n\\n' > start
                       # Grabbing the current release body from external repo
              elif [ ${EXT_RELEASE_TYPE} == 'github_commit' ]; then
                curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/commits/${EXT_GIT_BRANCH} | jq '. | .commit.message' | sed 's:^.\\(.*\\).$:\\1:' > releasebody.json
                # Creating the start of the json payload
                echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "target_commitish": "master",\
                       "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**'${EXT_REPO}' Changes:**\\n\\n' > start
              elif [ ${EXT_RELEASE_TYPE} == 'github_tag' ]; then
                curl -s ${EXT_COMMIT_URL} | jq '. | .commit.message' | sed 's:^.\\(.*\\).$:\\1:' > releasebody.json
                # Creating the start of the json payload
                echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "target_commitish": "master",\
                       "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**'${EXT_REPO}' Changes:**\\n\\n' > start
              elif [ ${EXT_RELEASE_TYPE} == 'os' ]; then
                # Using base package version for release notes
                echo "Updating base packages to ${PACKAGE_TAG}" > releasebody.json
                # Creating the start of the json payload
                echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "target_commitish": "master",\
                       "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**OS Changes:**\\n\\n' > start
              elif [ ${EXT_RELEASE_TYPE} == 'deb_repo' ] || [ ${EXT_RELEASE_TYPE} == 'alpine_repo' ]; then
                # Using base package version for release notes
                echo "Updating external repo packages to ${EXT_RELEASE}" > releasebody.json
                # Creating the start of the json payload
                echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "target_commitish": "master",\
                       "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**Repo Changes:**\\n\\n' > start
              elif [ ${EXT_RELEASE_TYPE} == 'npm_version' ]; then
                # Using base package version for release notes
                echo "Updating NPM version of ${EXT_NPM} to ${EXT_RELEASE}" > releasebody.json
                # Creating the start of the json payload
                echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "target_commitish": "master",\
                       "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**NPM Changes:**\\n\\n' > start
              elif [ ${EXT_RELEASE_TYPE} == 'pip_version' ]; then
                # Using base package version for release notes
                echo "Updating PIP version of ${EXT_PIP} to ${EXT_RELEASE}" > releasebody.json
                # Creating the start of the json payload
                echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "target_commitish": "master",\
                       "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**PIP Changes:**\\n\\n' > start
              elif [ ${EXT_RELEASE_TYPE} == 'external_blob' ]; then
                # Using base package version for release notes
                echo "External Release file changed at ${EXT_BLOB}" > releasebody.json
                # Creating the start of the json payload
                echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "target_commitish": "master",\
                       "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**Remote Changes:**\\n\\n' > start
              elif [ ${EXT_RELEASE_TYPE} == 'custom_json' ]; then
                # Referencing the JSON endpoint for release notes
                echo "Data change at JSON endpoint ${JSON_URL}" > releasebody.json
                # Creating the start of the json payload
                echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "target_commitish": "master",\
                       "name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'",\
                       "body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**Remote Changes:**\\n\\n' > start
              fi
              # Add the end of the payload to the file
              printf '","draft": false,"prerelease": false}' >> releasebody.json
              # Combine the start and ending string This is needed do to incompatibility with JSON and Bash escape strings
              paste -d'\\0' start releasebody.json > releasebody.json.done
              # Send payload to github
              curl -H "Authorization: token ${GITHUB_TOKEN}" -X POST https://api.github.com/repos/${LS_USER}/${LS_REPO}/releases -d @releasebody.json.done'''
      }
    }
    // Use helper container to push the current README in master to the DockerHub Repo
    stage('Sync-README') {
      when {
        branch "master"
        expression {
          env.LS_RELEASE != env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-ls' + env.LS_TAG_NUMBER
        }
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Run Docker README Sync'
          sh '''#! /bin/bash
                docker pull lsiodev/readme-sync
                docker run --rm=true \
                  -e DOCKERHUB_USERNAME=$DOCKERUSER \
                  -e DOCKERHUB_PASSWORD=$DOCKERPASS \
                  -e GIT_REPOSITORY=${LS_USER}/${LS_REPO} \
                  -e DOCKER_REPOSITORY=${DOCKERHUB_IMAGE} \
                  -e GIT_BRANCH=master \
                  lsiodev/readme-sync bash -c 'node sync'
             '''
        }
      }
    }
    /* #################
       Dev Release Logic
       ################# */
    // Push to the Dev user dockerhub endpoint when this is a non master branch
    stage('Docker-Push-Dev-Single') {
      when {
        not {
         branch "master"
        }
        environment name: 'CHANGE_ID', value: ''
        expression{
          env.MULTIARCH == 'false'
        }
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          echo 'Tag images to the built one'
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          echo 'Pushing both tags'
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:latest"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
        }
        script{
          env.DOCKERHUB_LINK = sh(
            script: '''echo https://hub.docker.com/r/${DEV_DOCKERHUB_IMAGE}/tags/''',
            returnStdout: true).trim()
        }
      }
    }
    // Push to the Dev user dockerhub endpoint when this is a non master branch
    stage('Docker-Push-Dev-Multi') {
      when {
        not {
         branch "master"
        }
        environment name: 'CHANGE_ID', value: ''
        expression{
          env.MULTIARCH == 'true'
        }
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          echo 'First Tag the releases to latest also'
          sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          echo 'Push all image variants'
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:arm64v8-latest"
          echo 'Generate Manifests based on tagging'
          sh "docker manifest create ${DEV_DOCKERHUB_IMAGE}:latest ${DEV_DOCKERHUB_IMAGE}:amd64-latest ${DEV_DOCKERHUB_IMAGE}:arm32v6-latest ${DEV_DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker manifest annotate ${DEV_DOCKERHUB_IMAGE}:latest ${DEV_DOCKERHUB_IMAGE}:arm32v6-latest --os linux --arch arm"
          sh "docker manifest annotate ${DEV_DOCKERHUB_IMAGE}:latest ${DEV_DOCKERHUB_IMAGE}:arm64v8-latest --os linux --arch arm64 --variant armv8"
          sh "docker manifest create ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker manifest annotate ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} --os linux --arch arm"
          sh "docker manifest annotate ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} --os linux --arch arm64 --variant armv8"
          echo 'Pushing by manifest tags'
          sh "docker manifest push ${DEV_DOCKERHUB_IMAGE}:latest"
          sh "docker manifest push ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
        }
        script{
          env.DOCKERHUB_LINK = sh(
            script: '''echo https://hub.docker.com/r/${DEV_DOCKERHUB_IMAGE}/tags/''',
            returnStdout: true).trim()
        }
      }
    }
    /* ################
       PR Release Logic
       ################ */
    // Push to PR user dockerhub endpoint when this is a pull request
    stage('Docker-Push-PR-Single') {
     when {
       not {
         environment name: 'CHANGE_ID', value: ''
       }
       expression{
         env.MULTIARCH == 'false'
       }
     }
     steps {
       withCredentials([
         [
           $class: 'UsernamePasswordMultiBinding',
           credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
           usernameVariable: 'DOCKERUSER',
           passwordVariable: 'DOCKERPASS'
         ]
       ]) {
         echo 'Logging into DockerHub'
         sh '''#! /bin/bash
            echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
            '''
         echo 'Tag images to the built one'
         sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:latest"
         sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
         echo 'Pushing both tags'
         sh "docker push ${PR_DOCKERHUB_IMAGE}:latest"
         sh "docker push ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
       }
       script{
         env.CODE_URL = sh(
           script: '''echo https://github.com/${LS_USER}/${LS_REPO}/pull/${PULL_REQUEST}''',
           returnStdout: true).trim()
         env.DOCKERHUB_LINK = sh(
           script: '''echo https://hub.docker.com/r/${PR_DOCKERHUB_IMAGE}/tags/''',
           returnStdout: true).trim()
       }
     }
    }
    // Push to PR user dockerhub endpoint when this is a pull request
    stage('Docker-Push-PR-Multi') {
      when {
        not {
          environment name: 'CHANGE_ID', value: ''
        }
        expression{
          env.MULTIARCH == 'true'
        }
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          echo 'First Tag the releases to latest also'
          sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          echo 'Push all image variants'
          sh "docker push ${PR_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:arm64v8-latest"
          echo 'Generate Manifests based on tagging'
          sh "docker manifest create ${PR_DOCKERHUB_IMAGE}:latest ${PR_DOCKERHUB_IMAGE}:amd64-latest ${PR_DOCKERHUB_IMAGE}:arm32v6-latest ${PR_DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker manifest annotate ${PR_DOCKERHUB_IMAGE}:latest ${PR_DOCKERHUB_IMAGE}:arm32v6-latest --os linux --arch arm"
          sh "docker manifest annotate ${PR_DOCKERHUB_IMAGE}:latest ${PR_DOCKERHUB_IMAGE}:arm64v8-latest --os linux --arch arm64 --variant armv8"
          sh "docker manifest create ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker manifest annotate ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} --os linux --arch arm"
          sh "docker manifest annotate ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} --os linux --arch arm64 --variant armv8"
          echo 'Pushing by manifest tags'
          sh "docker manifest push ${PR_DOCKERHUB_IMAGE}:latest"
          sh "docker manifest push ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
        }
        script{
          env.CODE_URL = sh(
            script: '''echo https://github.com/${LS_USER}/${LS_REPO}/pull/${PULL_REQUEST}''',
            returnStdout: true).trim()
          env.DOCKERHUB_LINK = sh(
            script: '''echo https://hub.docker.com/r/${PR_DOCKERHUB_IMAGE}/tags/''',
            returnStdout: true).trim()
        }
      }
    }
  }
  /* ######################
     Send status to Discord
     ###################### */
  post {
    success {
      echo "Build good send details to discord"
      sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 1681177,\
             "description": "**Build:**  '${BUILD_NUMBER}'\\n**Status:**  Success\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: '${RELEASE_LINK}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],\
             "username": "Jenkins"}' ${BUILDS_DISCORD} '''
    }
    failure {
      echo "Build Bad sending details to discord"
      sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 16711680,\
             "description": "**Build:**  '${BUILD_NUMBER}'\\n**Status:**  failure\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: '${RELEASE_LINK}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],\
             "username": "Jenkins"}' ${BUILDS_DISCORD} '''
    }
  }
}
