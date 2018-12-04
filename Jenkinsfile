pipeline {
  agent {
    label 'X86-64-MULTI'
  }
  // Configuration for the variables used for this specific repo
  environment {
    LS_USER='linuxserver'
    LS_REPO='docker-bookstack'
    CONTAINER_NAME='bookstack'
    GITHUB_TOKEN=credentials('498b4638-2d02-4ce5-832d-8a57d01d97ab')
  }
  stages {
    // Use helper containers to render templated files
    stage('Update-Templates') {
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
              docker pull linuxserver/jenkins-builder:latest
              docker run --rm -e CONTAINER_NAME=${CONTAINER_NAME} -e GITHUB_BRANCH=master -v ${TEMPDIR}:/ansible/jenkins linuxserver/jenkins-builder:latest
              docker pull linuxserver/doc-builder:latest
              docker run --rm -e CONTAINER_NAME=${CONTAINER_NAME} -e GITHUB_BRANCH=master -v ${TEMPDIR}:/ansible/readme linuxserver/doc-builder:latest
              if [ "$(md5sum ${TEMPDIR}/${LS_REPO}/Jenkinsfile | awk '{ print $1 }')" != "$(md5sum Jenkinsfile | awk '{ print $1 }')" ] || [ "$(md5sum ${TEMPDIR}/${CONTAINER_NAME}/README.md | awk '{ print $1 }')" != "$(md5sum README.md | awk '{ print $1 }')" ]; then
                mkdir -p ${TEMPDIR}/repo
                git clone https://github.com/${LS_USER}/${LS_REPO}.git ${TEMPDIR}/repo/${LS_REPO}
                git --git-dir ${TEMPDIR}/repo/${LS_REPO}/.git checkout master
                cp ${TEMPDIR}/${CONTAINER_NAME}/README.md ${TEMPDIR}/repo/${LS_REPO}/
                cp ${TEMPDIR}/docker-${CONTAINER_NAME}/Jenkinsfile ${TEMPDIR}/repo/${LS_REPO}/
                cd ${TEMPDIR}/repo/${LS_REPO}/
                git --git-dir ${TEMPDIR}/repo/${LS_REPO}/.git add Jenkinsfile README.md
                git --git-dir ${TEMPDIR}/repo/${LS_REPO}/.git commit -m 'Bot Updating Templated Files'
                git --git-dir ${TEMPDIR}/repo/${LS_REPO}/.git push https://LinuxServer-CI:${GITHUB_TOKEN}@github.com/${LS_USER}/${LS_REPO}.git --all
              fi
              rm -Rf ${TEMPDIR}'''
      }
    }
  }
}
