pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    ansiColor('xterm')
  }

  tools {
    maven 'maven3'
    jdk 'jdk17'
  }

  environment {
    APP_NAME = 'hello-service'
    IMAGE_TAG = "${BUILD_NUMBER}"

    // SonarQube server name configured in Jenkins
    SONARQUBE_SERVER = 'My SonarQube Server'

    // Nexus Docker (docker hosted repo)
    NEXUS_DOCKER_REGISTRY = 'nexus.company.com:8082'
    NEXUS_DOCKER_REPO     = 'docker-hosted'
    NEXUS_IMAGE           = "${NEXUS_DOCKER_REGISTRY}/${NEXUS_DOCKER_REPO}/${APP_NAME}:${IMAGE_TAG}"

    // JFrog Docker repo
    JFROG_DOCKER_REGISTRY = 'company.jfrog.io'
    JFROG_DOCKER_REPO     = 'docker-local'
    JFROG_IMAGE           = "${JFROG_DOCKER_REGISTRY}/${JFROG_DOCKER_REPO}/${APP_NAME}:${IMAGE_TAG}"

    // GitOps repo watched by Argo CD
    GITOPS_REPO_URL = 'https://github.com/myorg/hello-service-gitops.git'
    GITOPS_BRANCH   = 'main'
    GITOPS_APP_PATH = 'apps/hello-service'

    // Argo CD
    ARGOCD_SERVER    = 'argocd.company.com'
    ARGOCD_APP_NEXUS = 'hello-service-nexus'
    ARGOCD_APP_JFROG = 'hello-service-jfrog'
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Maven Build') {
      steps { sh 'mvn -U -B clean compile' }
    }

    stage('Unit Tests') {
      steps { sh 'mvn -B test' }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
      }
    }

    stage('Functional Tests') {
      steps { sh 'mvn -B -Pfunctional-tests verify' }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/failsafe-reports/*.xml'
        }
      }
    }

    stage('Sonar Scan') {
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          withSonarQubeEnv("${SONARQUBE_SERVER}") {
            sh """
              mvn -B verify sonar:sonar \\
                -Dsonar.token=$SONAR_TOKEN \\
                -Dsonar.projectKey=${APP_NAME} \\
                -Dsonar.projectName=${APP_NAME}
            """
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 30, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh """
          docker build -t ${NEXUS_IMAGE} .
          docker tag ${NEXUS_IMAGE} ${JFROG_IMAGE}
        """
      }
    }

    stage('Performance Test (k6)') {
      steps {
        sh """
          docker rm -f ${APP_NAME} || true
          docker run -d --name ${APP_NAME} -p 8080:8080 ${NEXUS_IMAGE}

          docker run --rm --network host -v $WORKSPACE/perf:/perf grafana/k6 run /perf/k6.js

          docker rm -f ${APP_NAME} || true
        """
      }
    }

    stage('Push Images (Nexus + JFrog)') {
      parallel {

        stage('Push to Nexus') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
              sh """
                echo "$NEXUS_PASS" | docker login ${NEXUS_DOCKER_REGISTRY} -u "$NEXUS_USER" --password-stdin
                docker push ${NEXUS_IMAGE}
                docker logout ${NEXUS_DOCKER_REGISTRY}
              """
            }
          }
        }

        stage('Push to JFrog') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'jfrog-docker-creds', usernameVariable: 'JFROG_USER', passwordVariable: 'JFROG_PASS')]) {
              sh """
                echo "$JFROG_PASS" | docker login ${JFROG_DOCKER_REGISTRY} -u "$JFROG_USER" --password-stdin
                docker push ${JFROG_IMAGE}
                docker logout ${JFROG_DOCKER_REGISTRY}
              """
            }
          }
        }
      }
    }

    stage('Update GitOps repo (image tags)') {
      steps {
        withCredentials([string(credentialsId: 'gitops-pat', variable: 'GITOPS_PAT')]) {
          sh """
            rm -rf gitops
            git clone -b ${GITOPS_BRANCH} https://${GITOPS_PAT}@${GITOPS_REPO_URL.replace('https://','')} gitops
            cd gitops/${GITOPS_APP_PATH}

            # Requires yq installed on agent
            yq -i '.images[] |= (select(.name == "${APP_NAME}").newName = "${NEXUS_DOCKER_REGISTRY}/${NEXUS_DOCKER_REPO}/${APP_NAME}")' overlays/nexus/kustomization.yaml
            yq -i '.images[] |= (select(.name == "${APP_NAME}").newTag = "${IMAGE_TAG}")' overlays/nexus/kustomization.yaml

            yq -i '.images[] |= (select(.name == "${APP_NAME}").newName = "${JFROG_DOCKER_REGISTRY}/${JFROG_DOCKER_REPO}/${APP_NAME}")' overlays/jfrog/kustomization.yaml
            yq -i '.images[] |= (select(.name == "${APP_NAME}").newTag = "${IMAGE_TAG}")' overlays/jfrog/kustomization.yaml

            git config user.email 'ci-bot@company.com'
            git config user.name 'ci-bot'
            git add .
            git commit -m "chore(${APP_NAME}): bump image to ${IMAGE_TAG}" || true
            git push origin ${GITOPS_BRANCH}
          """
        }
      }
    }

    stage('Argo CD Sync (both apps)') {
      steps {
        withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_TOKEN')]) {
          sh """
            argocd login ${ARGOCD_SERVER} --auth-token $ARGOCD_TOKEN --insecure

            argocd app sync ${ARGOCD_APP_NEXUS}
            argocd app wait ${ARGOCD_APP_NEXUS} --health --timeout 300

            argocd app sync ${ARGOCD_APP_JFROG}
            argocd app wait ${ARGOCD_APP_JFROG} --health --timeout 300
          """
        }
      }
    }

  }

  post {
    always {
      sh 'docker image prune -f || true'
    }
  }
}
