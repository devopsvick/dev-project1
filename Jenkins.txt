pipeline {
    agent any
    tools{
        maven 'Jenkins-project'
    }
    
    environment{
        SCANNER_HOME=tool 'Jenkins-sonar'
    }

    stages {
        stage('Checkout Stage') {
            steps {
                checkout changelog: false, poll: false, scm: scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/devopsvick/dev-project1.git']])
                echo 'Checkout Completed'
            }
        }
        stage('Build Stage') {
            steps {
                sh 'mvn clean compile -DskipTests=true'
                echo 'Build Completed'
            }
        }
        stage('Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan .', odcInstallation: 'DP-Jenkins'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                echo 'Dependency Check Completed'
            }
        }
        stage('Sonarqube Check') {
            steps {
                withSonarQubeEnv('Jenkins-sonar') {  // Ensure this matches the name of the SonarQube server configured in Jenkins
                    sh ''' 
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Devops-Jenkins \
                        -Dsonar.java.binaries=. \
                        -Dsonar.projectKey=Devops-Jenkins \
                        -Dsonar.sources=.
                    '''
                    echo 'Sonar Scan Completed'
                        }
            }
        }
        stage('Package Build Stage') {
            steps {
                sh 'mvn clean package -DskipTests=true'
                echo 'Build Completed'
            }
        }
        
        stage('Docker Build Stage') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'Docker', toolName: 'Docker') {
                    sh 'docker build -t vicky275/ekart-project -f docker/Dockerfile .'// some block
                    sh 'docker push vicky275/ekart-project:latest'
                    }
                echo 'Docker Build Completed'
                }
            }
        }
        stage('Trivy Scan') {
            steps {
                sh 'trivy image vicky275/ekart-project:latest'
                echo 'Trivy scan Completed'
            }
        }
        stage('Docker Deployment Stage') {
            //label can be used only if we use Node setup in jenkins
            //agent{label 'Testnode'}
            steps {
                script{
                    withDockerRegistry(credentialsId: 'Docker', toolName: 'Docker') {
                    //sh 'docker build -t vicky275/ekart-project -f docker/Dockerfile .'// some block
                    //sh 'sudo docker run -d --name=ekart-project -p 8070:8070 vicky275/ekart-project'
                    sshPublisher(publishers: [sshPublisherDesc(configName: 'Jenkins-slave', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: 'sudo docker run -d --name=ekart-project -p 8070:8070 vicky275/ekart-project', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
                    }
                //sh 'mvn clean package -DskipTests=true'
                echo 'Docker Deployment Completed'
                }
            }    
        }
    }
    post{
        always{
    //        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            slackSend channel: 'prod-jenkins', color: '#439FE0', message: 'Build Completed'
        }
    }
}
