# portfolio-jenkins-pipeline
젠킨스 파이프라인 코드

------------------------ SPRING SERVER ------------------------ 

pipeline {
    agent any
    
    environment {
        SLACK_CHANNEL = '#jenkins'
        registry = "jaesuk95/portfolio"
        DOCKERHUB_CREDENTIALS=credentials('docker-hub')
        dockerImage = '' 
        GIT_URL = "https://github.com/jaesuk95/portfolio.git"
    }
    
    stages {
        stage('Clone Git') {
            steps {
                git credentialsId: 'github-credential',
                    url: 'https://github.com/jaesuk95/portfolio.git',
                    branch: 'main'
            }
        }
        
        stage('Pull') {
            steps {
                git url: "${GIT_URL}", branch: "main", poll: true, changelog: true
                
                script {
                    env.GIT_COMMIT_MSG = sh (script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
                    env.GIT_AUTHOR = sh (script: 'git log -1 --pretty=%cn ${GIT_COMMIT}', returnStdout: true).trim()
                }
                print "${env.GIT_AUTHOR}"
            }
        }
        
        stage('Docker Image Version') {
            steps {
                script {
                    env.version = sh (returnStdout: true, script: "grep -r '^version' build.gradle | cut -d'=' -f2").trim()
                    print "${version}"
                    env.finalVersion = "${registry}:${version}"
                    
                    env.cleanedVersion = version.replaceAll("'", "")
                    print "${cleanedVersion}"
                    println cleanedVersion // prints 0.0.1
                }
            }
        }
        
        stage('SLACK') {
            steps {
                slackSend (channel: SLACK_CHANNEL, color: '#36DDAB', message: """${env.JOB_NAME} - #${env.BUILD_NUMBER} ${env.finalVersion}
Started by changes from ${env.GIT_AUTHOR}
Ubuntu - SPRING""")
            }
        }
        
        stage('Bulid Gradle') {
            steps {
                // gralew이 있어야됨. git clone해서 project를 가져옴.
                sh 'chmod +x gradlew'
                // jar 파일이 build/libs/ 에서 두개가 생성된다
                sh  './gradlew clean build -x test --exclude-task jar'
                
                sh 'ls -al ./build'
            }
        }
        
        stage('Build Image') {
            steps {
                sh 'docker build -t ${registry}:${cleanedVersion} .'
            }
        }
        
        // stage('Run Ansible playbook') {
        //     environment {
        //         ANSIBLE_HOST_KEY_CHECKING = false
        //     }
        //     steps {
        //         sh 'ansible-playbook playbook-sample.yml'
        //     }
        // }
        
        stage('Docker Login') {
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
            }
        }
        
        stage('Dockerhub Push') {
            steps {
                sh 'echo ${registry}:${cleanedVersion}'
                sh 'docker push ${registry}:${cleanedVersion}'
            }
        }
        
        stage('remote') {
            steps {
                script {
                    withCredentials ([sshUserPrivateKey(credentialsId: 'ubuntu-server', keyFileVariable: 'key', passphraseVariable: '', usernameVariable: 'central')]) {
                    def remote = [:]
                    remote.name = 'central'
                    remote.host = '192.168.64.2'
                    remote.allowAnyHosts = true
                    remote.user = central
                    remote.identityFile = key
                    
                    command=sh (returnStdout: true, script:"echo bash /home/central/scripts/spring-docker-run.sh ${cleanedVersion}")
                    }
                }
            }
        }
    }
    post {
        failure {
            slackSend (channel: SLACK_CHANNEL, color: '#FF0000', message: """${env.JOB_NAME} - #${env.BUILD_NUMBER} ${env.finalVersion}
Failure Commit from ${env.GIT_AUTHOR}
Ubuntu - SPRING""")
        }
        success {
            slackSend (channel: SLACK_CHANNEL, color: '#36DDAB', message: """${env.JOB_NAME} - #${env.BUILD_NUMBER} ${env.finalVersion}
Success Commit From ${env.GIT_AUTHOR}
Ubuntu - SPRING""")
        }
    }
}



------------------------ SMS SERVER NODEJS ------------------------ 

pipeline {
    agent any
    
    environment {
        SLACK_CHANNEL = '#jenkins'
        registry = "jaesuk95/portfolio-sms-nodejs"
        DOCKERHUB_CREDENTIALS=credentials('docker-hub')
        dockerImage = '' 
        GIT_URL = "https://github.com/jaesuk95/portfolio-sms-nodejs"
    }
    
    stages {
        stage('Clone Git') {
            steps {
                git credentialsId: 'github-credential',
                    url: 'https://github.com/jaesuk95/portfolio-sms-nodejs',
                    branch: 'main'
            }
        }
        
        stage('Pull') {
            steps {
                git url: "${GIT_URL}", branch: "main", poll: true, changelog: true
                
                script {
                    env.GIT_COMMIT_MSG = sh (script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
                    env.GIT_AUTHOR = sh (script: 'git log -1 --pretty=%cn ${GIT_COMMIT}', returnStdout: true).trim()
                }
                print "${env.GIT_AUTHOR}"
            }
        }
        
        stage('Docker Image Version') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    def version = packageJson.version
                    println "Package version is ${version}"
                    env.cleanedVersion = version
                    println "${cleanedVersion}"
                    
                    env.finalVersion = "${registry}:${cleanedVersion}"
                }
            }
        }
        
        stage('SLACK') {
            steps {
                slackSend (channel: SLACK_CHANNEL, color: '#36DDAB', message: """${env.JOB_NAME} - #${env.BUILD_NUMBER} ${env.finalVersion}
Started by changes from ${env.GIT_AUTHOR}
Ubuntu - SMS""")
            }
        }
        
        stage('Build Image') {
            steps {
                sh 'docker build -t ${registry}:${cleanedVersion} .'
            }
        }
        
        // stage('Run Ansible playbook') {
        //     environment {
        //         ANSIBLE_HOST_KEY_CHECKING = false
        //     }
        //     steps {
        //         sh 'ansible-playbook playbook-sample.yml'
        //     }
        // }
        
        stage('Docker Login') {
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
            }
        }
        
        stage('Dockerhub Push') {
            steps {
                sh 'echo ${registry}:${cleanedVersion}'
                sh 'docker push ${registry}:${cleanedVersion}'
            }
        }
        
        stage('remote') {
            steps {
                script {
                    withCredentials ([sshUserPrivateKey(credentialsId: 'ubuntu-server', keyFileVariable: 'key', passphraseVariable: '', usernameVariable: 'central')]) {
                    def remote = [:]
                    remote.name = 'central'
                    remote.host = '192.168.64.2'
                    remote.allowAnyHosts = true
                    remote.user = central
                    remote.identityFile = key
                    
                    command=sh (returnStdout: true, script:"echo bash /home/central/scripts/sms-docker-run.sh ${cleanedVersion}")
                    sshCommand remote:remote, command: "$command"
                    }
                }
            }
        }
    }
    post {
        failure {
            slackSend (channel: SLACK_CHANNEL, color: '#FF0000', message: """${env.JOB_NAME} - #${env.BUILD_NUMBER} ${env.finalVersion}
Failure Commit from ${env.GIT_AUTHOR}
Ubuntu - SMS""")
        }
        success {
            slackSend (channel: SLACK_CHANNEL, color: '#36DDAB', message: """${env.JOB_NAME} - #${env.BUILD_NUMBER} ${env.finalVersion}
Success Commit From ${env.GIT_AUTHOR}
Ubuntu - SMS""")
        }
    }
}

