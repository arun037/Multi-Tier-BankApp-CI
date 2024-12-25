pipeline {
    agent any
    
    parameters {
        string(name: 'IMAGE_VERSION', defaultValue: 'latest', description: 'docker tag')
    }
    
    tools {
        jdk 'JDK 17'
        maven 'Maven 3.8.1'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('git checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/arun037/Multi-Tier-BankApp-CI.git']])
            }
        }
        
        stage('compile') {
            steps {
                sh 'mvn compile'
            }
        }
        
        stage('test') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }
        
        stage('trivy fs') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }
        
        stage('sonar_analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=bankapp -Dsonar.projectKey=bankapp \
                           -Dsonar.java.binaries=target '''
                }
            }
        }
        
        
        stage('publish to nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-setting', maven: 'Maven 3.8.1', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }
        
        stage('docker build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-token') {
                        sh "docker build -t arunagri03/bankapp:${params.IMAGE_VERSION} ."
                    }
                }
            }
        }

        stage('docker image scan') {
            steps {
                sh "trivy image --format table -o image.html arunagri03/bankapp:${params.IMAGE_VERSION}"
            }
        }
        
        
        stage('docker push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-token') {
                        sh "docker push arunagri03/bankapp:${params.IMAGE_VERSION}"
                    }
                }
            }
        }
        
        stage('Update YAML Manifest in Other Repo') {
            steps {
                script {
                    withCredentials([gitUsernamePassword(credentialsId: 'github-token', gitToolName: 'Default')]) {
                        sh '''
                        set -e
                        
                        # Clone the target repository
                        git clone https://github.com/arun037/Multi-Tier-BankApp-CD.git
                        cd Multi-Tier-BankApp-CD/bankapp
                        
                        # Update the YAML manifest file with the new image version
                        sed -i "s|image: arunagri03/bankapp:.*|image: arunagri03/bankapp:${IMAGE_VERSION}|g" bankapp-ds.yml
                        
                        # Confirm changes
                        echo "Updated YAML file:"
                        cat bankapp-ds.yml
                        
                        # Configure Git user for commits
                        cd ..
                        git config user.email "jenkins@gmail.com"
                        git config user.name "arun"
                        
                        # Commit and push the changes
                        git add bankapp/bankapp-ds.yml
                        git commit -m "Update image version to ${IMAGE_VERSION}"
                        git push origin main
                        '''
                    }
                }
            }
         
        }
    }
}
