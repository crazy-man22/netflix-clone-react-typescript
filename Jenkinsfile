pipeline {
    agent any

    tools {
        nodejs 'NodeJS' 
    }

    environment {
        SONARQUBE_SCANNER = tool 'Sonar'
        DOCKER_CREDENTIALS = credentials('Docker') 
        TMDB_CREDENTIALS = credentials('tmdb')
    }
    
    stages {
        stage('Clean Workspace') {
            steps {
                echo "Cleaning Workspace..."
                cleanWs()
                echo 'Workspace cleaned.'
            }
        }

        stage('Clone GitHub Repository') {
            steps {
                script {
                    echo 'Cloning GitHub repository...'
                    git branch: 'main', url: 'https://github.com/Abdullah-0-3/NetflixCloneK8s.git'
                }
                echo 'Cloning done.'
            }
        }

        stage('SonarQube Quality Analysis') {
            steps {
                script {
                    withSonarQubeEnv('Sonar') {
                        echo 'SonarQube scanner environment configured.'
                        sh "${SONARQUBE_SCANNER}/bin/sonar-scanner -Dsonar.projectName=NetflixClone -Dsonar.projectKey=NetflixClone"
                    }
                }
                echo 'SonarQube analysis started.'
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    sh 'npm install'
                    echo 'Dependencies installed.'
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                script {
                    dependencyCheck additionalArguments: '--scan', odcInstallation: 'dc'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'  
                    echo 'OWASP Dependency Check completed.'
                }
            }
        }

        stage('SonarQube Gate Analysis') {
            steps {
                script {
                    echo 'Waiting for SonarQube gate analysis...'
                    timeout(time: 2, unit: 'MINUTES'){
                        waitForQualityGate abortPipeline: false
                    }
                    
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                script {
                    sh 'trivy fs --quiet --ignore-unfixed --format json . > trivy-fs-scan.txt'
                    echo 'Trivy file system scan completed.'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageName = "muhammadabdullahabrar/devops:netflix-clone"
                    sh "docker build --build-arg TMDB_V3_API_KEY=${TMDB_CREDENTIALS} -t ${imageName} ."
                    echo 'Docker image built.'
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    def imageName = "muhammadabdullahabrar/devops:netflix-clone"
                    sh "trivy image --quiet --ignore-unfixed --format json ${imageName} > trivy-image-scan.txt"
                    echo 'Trivy image scan completed.'
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        def imageName = "muhammadabdullahabrar/devops:netflix-clone"
                        sh """
                        set -x  
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${imageName}
                        docker logout
                        """
                    }
                }
            }
        }

        stage('Send Email with Attachments') {
            steps {
                emailext(
                    attachLog: true,
                    subject: "Netflix Clone Pipeline - Jenkins",
                    body: """Project: ${env.JOB_NAME} - ${env.BUILD_NUMBER} has been built successfully. Check console output at ${env.BUILD_URL}.""",
                    to: 'abdullahabrar4843@gmail.com', // Change mail here
                    attachmentsPattern: 'trivy-fs-scan.txt,trivy-image-scan.txt,dependency-check-report.xml'
                )
            }
        }
    }
}