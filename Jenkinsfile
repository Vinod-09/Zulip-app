pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'v1', description: 'Tag for mekumar/zulip (e.g. v1 or build number)')
  }

  environment {
    DOCKER_IMAGE = "mekumar/zulip"
    MANIFESTS_REPO = "https://github.com/Vinod-09/Zulip-ArgoCD.git"
    MANIFESTS_DIR = "manifests-repo"
    MANIFESTS_ROOT = "manifests"      // the root folder inside the cloned repo to update
  }

  stages {
    stage('Pre-check') {
      steps {
        echo "Will build and push ${DOCKER_IMAGE}:${params.IMAGE_TAG}"
        script {
          if (!fileExists('Dockerfile')) {
            error "Dockerfile not found in workspace root. Place your Zulip Dockerfile in repo root (where Jenkinsfile runs)."
          }
        }
      }
    }

    stage('Docker build') {
      steps {
        sh "docker build -t ${DOCKER_IMAGE}:${params.IMAGE_TAG} ."
      }
    }

    stage('Docker login & push Zulip image') {
      steps {
        withCredentials([string(credentialsId: 'docker-pat', variable: 'DOCKER_TOKEN')]) {
          sh '''
            set -euo pipefail
            echo "$DOCKER_TOKEN" | docker login -u mekumar --password-stdin
            docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Clone manifests repo') {
      steps {
        withCredentials([string(credentialsId: 'Git-pat', variable: 'GIT_TOKEN')]) {
          sh """
            set -euo pipefail
            rm -rf ${MANIFESTS_DIR}
            git clone https://${GIT_TOKEN}@github.com/Vinod-09/Zulip-ArgoCD.git ${MANIFESTS_DIR}
            ls -la ${MANIFESTS_DIR}/${MANIFESTS_ROOT} || true
          """
        }
      }
    }

    stage('Update images inside manifests/* subfolders') {
      steps {
        withCredentials([string(credentialsId: 'Git-pat', variable: 'GIT_TOKEN')]) {
          sh """
            set -euo pipefail
            cd ${MANIFESTS_DIR}

            echo "Files under ${MANIFESTS_ROOT}:"
            find ${MANIFESTS_ROOT} -maxdepth 2 -type f -name '*.yaml' -print || true

            # Replace Zulip app image (any occurrence of zulip/docker-zulip:...) -> mekumar/zulip:${IMAGE_TAG}
            find ${MANIFESTS_ROOT} -type f -name '*.yaml' -print0 | xargs -0 sed -i "s|zulip/docker-zulip:[^[:space:]]*|${DOCKER_IMAGE}:${IMAGE_TAG}|g"

            # Replace Postgres upstream -> mekumar/zulip:postgres-zulip
            find ${MANIFESTS_ROOT} -type f -name '*.yaml' -print0 | xargs -0 sed -i "s|zulip/zulip-postgresql:[^[:space:]]*|${DOCKER_IMAGE}:postgres-zulip|g"

            # Replace memcached -> mekumar/zulip:memcached
            find ${MANIFESTS_ROOT} -type f -name '*.yaml' -print0 | xargs -0 sed -i "s|memcached:[^[:space:]]*|${DOCKER_IMAGE}:memcached|g"

            # Replace rabbitmq -> mekumar/zulip:rabitmq
            find ${MANIFESTS_ROOT} -type f -name '*.yaml' -print0 | xargs -0 sed -i "s|rabbitmq:[^[:space:]]*|${DOCKER_IMAGE}:rabitmq|g"

            # Replace redis -> mekumar/zulip:redis
            find ${MANIFESTS_ROOT} -type f -name '*.yaml' -print0 | xargs -0 sed -i "s|redis:[^[:space:]]*|${DOCKER_IMAGE}:redis|g"

            echo "Sample updated zulip deployment lines:"
            grep -n "image:" -n ${MANIFESTS_ROOT}/zulip/zulip-deployment.yaml || true

            # Commit & push changes
            git config user.email "jenkins@ci.local"
            git config user.name "jenkins-ci"
            git add ${MANIFESTS_ROOT} || true
            git commit -m "ci: update images to ${DOCKER_IMAGE}:${IMAGE_TAG}" || echo "No changes to commit"
            git push https://${GIT_TOKEN}@github.com/Vinod-09/Zulip-ArgoCD.git HEAD:main
          """
        }
      }
    }
  }

  post {
    always {
      sh 'docker logout || true'
    }
    success {
      echo "SUCCESS: built & pushed ${DOCKER_IMAGE}:${params.IMAGE_TAG} and updated manifests under ${MANIFESTS_ROOT}."
    }
    failure {
      echo "FAILURE - check console logs."
    }
  }
}
