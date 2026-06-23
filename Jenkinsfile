pipeline {
    agent {
        label 'linux-agent'
    }

    parameters {
        string(name: 'REPO_URL', defaultValue: 'https://github.com/12Funday/springboot-sample.git', description: 'URL repo project')
        string(name: 'IMAGE_NAME', defaultValue: '12funday/java-rest-api', description: 'Docker image name')
        string(name: 'IMAGE_TAG', defaultValue: 'v.0.0.1', description: 'Unique Tag')
        string(name: 'DEPLOY_SERVER', defaultValue: '', description: 'Deployment server')
        string(name: 'PROJECT_KEY', defaultValue: '', description: 'Sonarqube project key')
        string(name: 'PROJECT_TOKEN', defaultValue: '', description: 'Sonarqube TOKEN')
        string(name: 'SONARQUBE_URL', defaultValue: 'https://sonarqube.asdf.my.id', description: 'Sonarqube URL')
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code'
                git branch: 'main', url: "${REPO_URL}"
            }
        }
        
        stage('Compile Code') {
            steps {
                /// compile project java dengan maven
                sh 'mvn clean compile'
           }
        }

        stage('Scanning Code') {
            steps {
                // maven bisa langsung kirim ke sonar tanpa install sonnarscan-cli
                sh """
                mvn clean verify sonar:sonar \
                    -Dsonar.host.url="${SONARQUBE_URL}" \
                    -Dsonar.token="${PROJECT_TOKEN}" \
                    -Dsonar.projectKey="${PROJECT_KEY}"
                """
            }
        }

        stage('Build Image'){
            steps{
                // build image docker, tiap repository harus ada Dockerfile
                sh """
                    docker build \
                    -t ${params.IMAGE_NAME}:${params:IMAGE_TAG} \
                    -t ${params.IMAGE_NAME}:latest .
                    """
            }
        }

        stage('Scan Image'){
            steps{
                // scan image dengan trivy, kemudian hasil scan akan di simpan dengan nama trivy-report.html
                sh """
                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    --format template \
                    --template "@/home/jenkins/template/html.tpl" \
                    -o trivy-report.html \
                    ${params.IMAGE_NAME}:latest
                    """
                // exit-code 1, pipeline akan dianggap gagal jika ditemukan vulnerability HIGH dan CRITICAL
                // exit-code 0, abaikan hasil scans dan lanjut ke stage selanjutnya
            }
        }

        stage('Push Image') {
            steps {
                // push image, dalam case ini ke dockerhub
                sh """
                    docker push ${params.IMAGE_NAME}:${params:IMAGE_TAG}
                    docker push ${params.IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy') {
            steps {
                // deploy ke server tujuan, dalam case ini deploymentnya menggunakan docker
                sshagent(credentials: ['ssh-deploy-server']) {

                    sh """
                        echo 'Test ssh'
                        ssh -o StrictHostKeyChecking=no pandey@${DEPLOY_SERVER} '
                            set -e

                            docker pull ${params.IMAGE_NAME}:latest

                            docker stop demo-api || true
                            docker rm demo-api || true

                            docker run -d \
                            --name demo-api \
                            --restart unless-stopped \
                            -p 8080:8080 \
                            ${params.IMAGE_NAME}:latest

                            docker ps | grep demo-api
                        '
                    """
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh """
                    echo 'Waiting for app to start...'
                    sleep 30

                    echo 'Checking root endpoint...'

                    RESPONSE=\$(curl -s -o /dev/null -w "%{http_code}" http://${DEPLOY_SERVER}:8080/)

                    if [ "\$RESPONSE" -ne 200 ]; then
                        echo "Smoke test FAILED. HTTP code: \$RESPONSE"
                        exit 1
                    fi

                    echo "Smoke test PASSED"
                """
            }
        }

        stage('Archive Security Report') {
            steps {
                // archieve hasil scan trivy
                archiveArtifacts artifacts: 'trivy-report.html'
            }
        }
    }
    post {
        success {
            echo "✅ Pipeline sukses!"
        }

        failure {
            echo "❌ Pipeline gagal!"
            // mail to: 'someone@mail.com',
            //      subject: "Pipeline Gagal: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            //      body: "Pipeline ${env.JOB_NAME} build #${env.BUILD_NUMBER} gagal.\nCek console output: ${env.BUILD_URL}console"
        }

        always {
            echo "Pipeline selesai"
        }
    }
}
