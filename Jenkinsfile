pipeline {
  agent any
  // Configuration for the variables used for this specific repo
  environment {
    EXT_GIT_BRANCH = 'master'
    EXT_USER = 'bookstackapp'
    EXT_REPO = 'bookstack'
    BUILD_VERSION_ARG = 'BOOKSTACK_RELEASE'
    LS_USER = 'linuxserver'
    LS_REPO = 'docker-bookstack'
    DOCKERHUB_IMAGE = 'lspipelive/bookstack'
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
    MULTIARCH = 'true'
    CI = 'true'
    CI_PORT = '80'
    CI_SSL = 'false'
    CI_DELAY = '5'
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
    // If this is an alpine base image determine the base package tag to use
    stage("Set Package tag Alpine"){
      when {
        environment name: 'DIST_IMAGE', value: 'alpine'
        not {environment name: 'DIST_PACKAGES', value: 'none'}
      }
      steps{
        sh '''docker pull alpine:${DIST_TAG}'''
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
        environment name: 'DIST_IMAGE', value: 'ubuntu'
        not {environment name: 'DIST_PACKAGES', value: 'none'}
      }
      steps{
        sh '''docker pull ubuntu:${DIST_TAG}'''
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
        environment name: 'DIST_PACKAGES', value: 'none'
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
    /* ###############
       Build Container
       ############### */
     // Build Docker container for push to LS Repo
     stage('Build-Single') {
       when {
         environment name: 'MULTIARCH', value: 'false'
       }
       steps {
         sh "docker build --no-cache -t ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} \
         --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
         script{
           env.CI_IMAGE = env.DOCKERHUB_IMAGE
           env.CI_TAGS = env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
           env.CI_META_TAG = env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
         }
       }
     }
     // Build MultiArch Docker container for push to LS Repo
     stage('Build-Multi') {
       when {
         environment name: 'MULTIARCH', value: 'true'
       }
       steps {
         sh "docker build --no-cache -f Dockerfile.amd64 -t ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} \
         --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
         sh "docker build --no-cache -f Dockerfile.armhf -t ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} \
         --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
         sh "docker build --no-cache -f Dockerfile.aarch64 -t ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} \
         --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=\"${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}\" --build-arg BUILD_DATE=${GITHUB_DATE} ."
         script{
           env.CI_IMAGE = env.DOCKERHUB_IMAGE
           env.CI_TAGS = 'amd64-' + env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER + '|arm32v6-' + env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER + '|arm64v8-' + env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
           env.CI_META_TAG = env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
         }
       }
     }
     // Tag to the Dev user dockerhub endpoint when this is a non master branch
     stage('Docker-Tag-Dev-Single') {
       when {
         not {branch "master"}
         environment name: 'CHANGE_ID', value: ''
         environment name: 'MULTIARCH', value: 'false'
       }
       steps {
         sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:latest"
         sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
         script{
           env.CI_IMAGE = env.DEV_DOCKERHUB_IMAGE
           env.CI_TAGS = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
           env.CI_META_TAG = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
         }
       }
     }
     // Tag to the Dev user dockerhub endpoint when this is a non master branch
     stage('Docker-Tag-Dev-Multi') {
       when {
         not {branch "master"}
         environment name: 'CHANGE_ID', value: ''
         environment name: 'MULTIARCH', value: 'true'
       }
       steps {
         sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:amd64-latest"
         sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:arm32v6-latest"
         sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:arm64v8-latest"
         sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
         sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
         sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
         script{
           env.CI_IMAGE = env.DEV_DOCKERHUB_IMAGE
           env.CI_TAGS = 'amd64-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA + '|arm32v6-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA + '|arm64v8-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
           env.CI_META_TAG = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-dev-' + env.COMMIT_SHA
         }
       }
     }
     // Tag to PR user dockerhub endpoint when this is a pull request
     stage('Docker-Tag-PR-Single') {
      when {
        not {environment name: 'CHANGE_ID', value: ''}
        environment name: 'MULTIARCH', value: 'false'
      }
      steps {
        sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:latest"
        sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
        script{
          env.CI_IMAGE = env.PR_DOCKERHUB_IMAGE
          env.CI_TAGS = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
          env.CI_META_TAG = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
        }
      }
     }
     // Tag to PR user dockerhub endpoint when this is a pull request
     stage('Docker-Tag-PR-Multi') {
       when {
         not {environment name: 'CHANGE_ID', value: ''}
         environment name: 'MULTIARCH', value: 'true'
       }
       steps {
         sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:amd64-latest"
         sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:arm32v6-latest"
         sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:arm64v8-latest"
         sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
         sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
         sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
         script{
           env.CI_IMAGE = env.PR_DOCKERHUB_IMAGE
           env.CI_TAGS = 'amd64-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST + '|arm32v6-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST + '|arm64v8-' + env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
           env.CI_META_TAG = env.EXT_RELEASE + '-pkg-' + env.PACKAGE_TAG + '-pr-' + env.PULL_REQUEST
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
      }
      steps {
        withCredentials([
          string(credentialsId: 'spaces-key', variable: 'DO_KEY'),
          string(credentialsId: 'spaces-secret', variable: 'DO_SECRET')
        ]) {
          sh '''#! /bin/bash
                docker pull lsiodev/ci
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                -e IMAGE=\"${CI_IMAGE}\" \
                -e DELAY_START=\"${CI_DELAY}\" \
                -e TAGS=\"${CI_TAGS}\" \
                -e META_TAG=\"${CI_META_TAG}\" \
                -e PORT=\"${CI_PORT}\" \
                -e SSL=\"${CI_SSL}\" \
                -e BASE=\"${DIST_IMAGE}\" \
                -e SECRET_KEY=\"${DO_SECRET}\" \
                -e ACCESS_KEY=\"${DO_KEY}\" \
                -e DOCKER_ENV=\"DB_HOST=${TEST_MYSQL_HOST}|DB_DATABASE=bookstack|DB_USERNAME=root|DB_PASSWORD=${TEST_MYSQL_PASSWORD}\" \
                -e WEB_AUTH=\"${CI_AUTH}\" \
                -e WEB_PATH=\"${CI_WEBPATH}\" \
                -e DO_REGION="ams3" \
                -e DO_BUCKET="lsio-ci" \
                -t lsiodev/ci:latest \
                python /ci/ci.py'''
          script{
            env.CI_URL = 'https://lsio-ci.ams3.digitaloceanspaces.com/' + env.CI_IMAGE + '/' + env.CI_META_TAG + '/index.html'
          }
        }
      }
    }
    /* ##################
       Live Release Logic
       ################## */
    // If this is a public release push this to the live repo triggered by an external repo update or LS repo update on master
    stage('Docker-Push-Release-Single') {
      when {
        branch "master"
        environment name: 'MULTIARCH', value: 'false'
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
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:latest"
          sh "docker push ${DOCKERHUB_IMAGE}:latest"
          sh "docker push ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
        }
      }
    }
    // If this is a public release push this to the live repo triggered by an external repo update or LS repo update on master
    stage('Docker-Push-Release-Multi') {
      when {
        branch "master"
        environment name: 'MULTIARCH', value: 'true'
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
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          sh "docker tag ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker push ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
          sh "docker push ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
          sh "docker push ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
          sh "docker push ${DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker push ${DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker push ${DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker manifest push --purge ${DOCKERHUB_IMAGE}:latest || :"
          sh "docker manifest create ${DOCKERHUB_IMAGE}:latest ${DOCKERHUB_IMAGE}:amd64-latest ${DOCKERHUB_IMAGE}:arm32v6-latest ${DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker manifest annotate ${DOCKERHUB_IMAGE}:latest ${DOCKERHUB_IMAGE}:arm32v6-latest --os linux --arch arm"
          sh "docker manifest annotate ${DOCKERHUB_IMAGE}:latest ${DOCKERHUB_IMAGE}:arm64v8-latest --os linux --arch arm64 --variant armv8"
          sh "docker manifest push --purge ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} || :"
          sh "docker manifest create ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
          sh "docker manifest annotate ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-ls${LS_TAG_NUMBER} --os linux --arch arm"
          sh "docker manifest annotate ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-ls${LS_TAG_NUMBER} --os linux --arch arm64 --variant armv8"
          sh "docker manifest push --purge ${DOCKERHUB_IMAGE}:latest"
          sh "docker manifest push --purge ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
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
          sh '''#! /bin/bash
                docker pull lsiodev/readme-sync
                docker run --rm=true \
                  -e DOCKERHUB_USERNAME=$DOCKERUSER \
                  -e DOCKERHUB_PASSWORD=$DOCKERPASS \
                  -e GIT_REPOSITORY=${LS_USER}/${LS_REPO} \
                  -e DOCKER_REPOSITORY=${DOCKERHUB_IMAGE} \
                  -e GIT_BRANCH=master \
                  lsiodev/readme-sync bash -c 'node sync' '''
        }
      }
    }
    /* #################
       Dev Release Logic
       ################# */
    // Push to the Dev user dockerhub endpoint when this is a non master branch
    stage('Docker-Push-Dev-Single') {
      when {
        not {branch "master"}
        environment name: 'CHANGE_ID', value: ''
        environment name: 'MULTIARCH', value: 'false'
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
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:latest"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
        }
        script{
          env.DOCKERHUB_LINK = 'https://hub.docker.com/r/' + env.DEV_DOCKERHUB_IMAGE + '/tags/'
        }
      }
    }
    // Push to the Dev user dockerhub endpoint when this is a non master branch
    stage('Docker-Push-Dev-Multi') {
      when {
        not {branch "master"}
        environment name: 'CHANGE_ID', value: ''
        environment name: 'MULTIARCH', value: 'true'
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
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker manifest push --purge ${DEV_DOCKERHUB_IMAGE}:latest || :"
          sh "docker manifest create ${DEV_DOCKERHUB_IMAGE}:latest ${DEV_DOCKERHUB_IMAGE}:amd64-latest ${DEV_DOCKERHUB_IMAGE}:arm32v6-latest ${DEV_DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker manifest annotate ${DEV_DOCKERHUB_IMAGE}:latest ${DEV_DOCKERHUB_IMAGE}:arm32v6-latest --os linux --arch arm"
          sh "docker manifest annotate ${DEV_DOCKERHUB_IMAGE}:latest ${DEV_DOCKERHUB_IMAGE}:arm64v8-latest --os linux --arch arm64 --variant armv8"
          sh "docker manifest push --purge ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} || :"
          sh "docker manifest create ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          sh "docker manifest annotate ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} --os linux --arch arm"
          sh "docker manifest annotate ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} ${DEV_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA} --os linux --arch arm64 --variant armv8"
          sh "docker manifest push --purge ${DEV_DOCKERHUB_IMAGE}:latest"
          sh "docker manifest push --purge ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
        }
        script{
          env.DOCKERHUB_LINK = 'https://hub.docker.com/r/' + env.DEV_DOCKERHUB_IMAGE + '/tags/'
        }
      }
    }
    /* ################
       PR Release Logic
       ################ */
    // Push to PR user dockerhub endpoint when this is a pull request
    stage('Docker-Push-PR-Single') {
     when {
       not {environment name: 'CHANGE_ID', value: ''}
       environment name: 'MULTIARCH', value: 'false'
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
         sh '''#! /bin/bash
            echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
            '''
         sh "docker push ${PR_DOCKERHUB_IMAGE}:latest"
         sh "docker push ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
       }
       script{
         env.CODE_URL = 'https://github.com/' + env.LS_USER + '/' + env.LS_REPO + '/pull/' + env.PULL_REQUEST
         env.DOCKERHUB_LINK = 'https://hub.docker.com/r/' + env.PR_DOCKERHUB_IMAGE + '/tags/'
       }
     }
    }
    // Push to PR user dockerhub endpoint when this is a pull request
    stage('Docker-Push-PR-Multi') {
      when {
        not {environment name: 'CHANGE_ID', value: ''}
        environment name: 'MULTIARCH', value: 'true'
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
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          sh "docker push ${PR_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:amd64-latest"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:arm32v6-latest"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker manifest push --purge ${PR_DOCKERHUB_IMAGE}:latest || :"
          sh "docker manifest create ${PR_DOCKERHUB_IMAGE}:latest ${PR_DOCKERHUB_IMAGE}:amd64-latest ${PR_DOCKERHUB_IMAGE}:arm32v6-latest ${PR_DOCKERHUB_IMAGE}:arm64v8-latest"
          sh "docker manifest annotate ${PR_DOCKERHUB_IMAGE}:latest ${PR_DOCKERHUB_IMAGE}:arm32v6-latest --os linux --arch arm"
          sh "docker manifest annotate ${PR_DOCKERHUB_IMAGE}:latest ${PR_DOCKERHUB_IMAGE}:arm64v8-latest --os linux --arch arm64 --variant armv8"
          sh "docker manifest create ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:amd64-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          sh "docker manifest push --purge ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} || :"
          sh "docker manifest annotate ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:arm32v6-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} --os linux --arch arm"
          sh "docker manifest annotate ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} ${PR_DOCKERHUB_IMAGE}:arm64v8-${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST} --os linux --arch arm64 --variant armv8"
          sh "docker manifest push --purge ${PR_DOCKERHUB_IMAGE}:latest"
          sh "docker manifest push --purge ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
        }
        script{
          env.CODE_URL = 'https://github.com/' + env.LS_USER + '/' + env.LS_REPO + '/pull/' + env.PULL_REQUEST
          env.DOCKERHUB_LINK = 'https://hub.docker.com/r/' + env.PR_DOCKERHUB_IMAGE + '/tags/'
        }
      }
    }
  }
  /* ######################
     Send status to Discord
     ###################### */
  post {
    success {
      sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 1681177,\
             "description": "**Build:**  '${BUILD_NUMBER}'\\n**CI Results:**  '${CI_URL}'\\n**Status:**  Success\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: '${RELEASE_LINK}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],\
             "username": "Jenkins"}' ${BUILDS_DISCORD} '''
    }
    failure {
      sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 16711680,\
             "description": "**Build:**  '${BUILD_NUMBER}'\\n**CI Results:**  '${CI_URL}'\\n**Status:**  failure\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: '${RELEASE_LINK}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],\
             "username": "Jenkins"}' ${BUILDS_DISCORD} '''
    }
  }
}
