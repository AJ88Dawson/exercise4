// Exercise 4 - workflow design, targeting the toolchain diagram rather than
// the AWS S3/Elastic Beanstalk/DynamoDB architecture used in
// .github/workflows/build-deploy.yml.
//
// This is a Jenkins declarative pipeline, not a GitHub Actions workflow,
// because the diagram names Jenkins as the CI engine. Mapping the diagram's
// swim lanes onto pipeline stages:
//   Developers -> GitHub holds the source; this Jenkinsfile lives in it
//   DevOps     -> Jenkins builds (Maven), tests (JUnit), and publishes the
//                 resulting artifact to JFrog Artifactory
//   Testers    -> the artifact is deployed, as a Docker container, through
//                 Test -> Staging -> Live, each a separate stage
//
// Secrets (JFrog credentials, the Docker registry login, and each
// environment's deploy credentials) are never written into this file. They
// are registered once in Jenkins' own credentials store (Manage Jenkins ->
// Credentials) and pulled in here only by ID, via `credentials()` below -
// Jenkins injects the real value as an environment variable at run time and
// masks it in the console log.

pipeline {
    agent any

    tools {
        // Matches the diagram's "Build tool: Maven (for Java)". A
        // Python-stack build would swap this block for pip instead.
        maven 'Maven-3.9'
        jdk 'Temurin-17'
    }

    environment {
        // credentials() looks the ID up in Jenkins' credentials store and
        // exposes it as an env var for this run only. Nothing sensitive is
        // ever stored in this file or in source control.
        JFROG_CREDS  = credentials('jfrog-artifactory')
        DOCKER_CREDS = credentials('docker-registry')
        ARTIFACTORY_URL = 'https://your-org.jfrog.io/artifactory'
        DOCKER_REGISTRY = 'your-org-docker-registry'
        IMAGE_NAME   = 'sample-app'
    }

    stages {
        stage('Checkout') {
            steps {
                // Source: GitHub, per the diagram's Developers lane.
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Automated testing') {
            steps {
                // JUnit, per the diagram. The pom.xml's surefire plugin is
                // what actually runs the tests; this just invokes it and
                // records the result for Jenkins' test report UI.
                sh 'mvn -B test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Publish artifact to JFrog') {
            steps {
                sh '''
                    curl -u "$JFROG_CREDS_USR:$JFROG_CREDS_PSW" \
                        -T target/*.jar \
                        "$ARTIFACTORY_URL/libs-release-local/${IMAGE_NAME}/${BUILD_NUMBER}/${IMAGE_NAME}-${BUILD_NUMBER}.jar"
                '''
            }
        }

        stage('Build and push Docker image') {
            steps {
                sh '''
                    docker build -t $DOCKER_REGISTRY/$IMAGE_NAME:$BUILD_NUMBER .
                    echo "$DOCKER_CREDS_PSW" | docker login $DOCKER_REGISTRY -u "$DOCKER_CREDS_USR" --password-stdin
                    docker push $DOCKER_REGISTRY/$IMAGE_NAME:$BUILD_NUMBER
                '''
            }
        }

        stage('Deploy to Testing environment') {
            steps {
                sh '''
                    docker rm -f ${IMAGE_NAME}-test || true
                    docker run -d --name ${IMAGE_NAME}-test -p 8081:8080 \
                        $DOCKER_REGISTRY/$IMAGE_NAME:$BUILD_NUMBER
                '''
            }
        }

        stage('Promote to Staging') {
            steps {
                // A manual gate: someone confirms the Testing environment
                // looks right before Staging is touched. `input` pauses the
                // pipeline and waits for a person to click through in the
                // Jenkins UI - this is where the Testers swim lane's manual
                // work happens.
                input message: 'Promote build to Staging?', ok: 'Deploy'
                sh '''
                    docker rm -f ${IMAGE_NAME}-staging || true
                    docker run -d --name ${IMAGE_NAME}-staging -p 8082:8080 \
                        $DOCKER_REGISTRY/$IMAGE_NAME:$BUILD_NUMBER
                '''
            }
        }

        stage('Promote to Live') {
            steps {
                // A second, separate gate. Staging passing is not the same
                // decision as "release to Live" - keep them as two
                // deliberate approvals, not one.
                input message: 'Promote build to Live?', ok: 'Deploy'
                sh '''
                    docker rm -f ${IMAGE_NAME}-live || true
                    docker run -d --name ${IMAGE_NAME}-live -p 8080:8080 \
                        $DOCKER_REGISTRY/$IMAGE_NAME:$BUILD_NUMBER
                '''
            }
        }
    }

    post {
        // Matches the diagram's "Email Server: Outlook, alternative Teams"
        // node - notify the team once the pipeline finishes either way.
        success {
            echo "Build #${BUILD_NUMBER} succeeded - notify team (Outlook/Teams webhook)."
        }
        failure {
            echo "Build #${BUILD_NUMBER} failed - notify team (Outlook/Teams webhook)."
        }
    }
}
