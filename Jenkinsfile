pipeline {
    agent any

    tools {
      // Jenkins 'Global Tool Configuration' 에 설정한 버전과 연동
      maven "Maven"
    }

    environment {
        ECR_PATH = '851557167064.dkr.ecr.ap-northeast-2.amazonaws.com'
        ECR_IMAGE = 'demo-maven-springboot'
        REGION = 'ap-northeast-2'
        ACCOUNT_ID='851557167064'
    }

    stages {
        stage('Git Clone from gitSCM') {
            steps {
                script {
                    try {
                        git branch: 'main', 
                            credentialsId: 'github',
                            url: 'https://github.com/imyujinsim/bastion-final'
                        sh "ls -lat"
                        env.cloneResult=true
                        
                    } catch (error) {
                        print(error)
                        env.cloneResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
        stage("Build JAR with Maven") {
            when {
                expression {
                    return env.cloneResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }                
            }
            steps {
                script{
                    try {
                        sh """
                        rm -rf deploy
                        mkdir deploy
                        java -version
                        """
                        // sh "sed -i 's/  version:.*/  version: \${VERSION:v${env.BUILD_NUMBER}}/g' /var/lib/jenkins/workspace/${env.JOB_NAME}/src/main/resources/application.yaml"
                        // sh "cat /var/lib/jenkins/workspace/${env.JOB_NAME}/src/main/resources/application.yaml"
                        sh './mvnw package'
                        sh """
			cd /var/lib/jenkins/workspace/${env.JOB_NAME}/
                        cp /var/lib/jenkins/workspace/${env.JOB_NAME}/target/*.jar ./${ECR_IMAGE}.jar
                        """
                        env.mavenBuildResult=true
                    } catch (error) {
                        print(error)
                        echo 'Failed to build jar file'
                        env.maveBuildResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
        stage('Docker Build and Push to ECR'){
            when {
                expression {
                    return env.mavenBuildResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }
            }
            steps {
                script{
                    try {
                        sh"""
                        #!/bin/bash
                        cat>Dockerfile<<-EOF
FROM openjdk:11-jre-slim
ADD ./${ECR_IMAGE}.jar /home/${ECR_IMAGE}.jar
CMD ["nohup", "java", "-jar", "-Dspring.profiles.active='mysql'", "/home/${ECR_IMAGE}.jar"]
EOF
"""
                        docker.withRegistry("https://${ECR_PATH}", "ecr:ap-northeast-2:aws_credentials") {
                            def image = docker.build("${ECR_PATH}/${ECR_IMAGE}:${env.BUILD_NUMBER}")
                            image.push()
                        }
                        
                        echo 'Remove Deploy Files'
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        env.dockerBuildResult=true
                    } catch (error) {
                        print(error)
                        echo 'Remove Deploy Files'
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        env.dockerBuildResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
    stage('Push Yaml'){
            when {
                expression {
                    return env.dockerBuildResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }
            }
            steps {
                script{
                    try {
                        git url: 'https://github.com/imyujinsim/bastion-final-cd', branch: "main", credentialsId: 'github'
                        // sh "rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        sh """
                        mkdir yaml
                        cd yaml
                        #!/bin/bash
                        cat>deploy.yaml<<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: was
    spec:
      containers:
      - image: 851557167064.dkr.ecr.ap-northeast-2.amazonaws.com/demo-maven-springboot:ver${env.BUILD_NUMBER}
        name: petclinic
EOF"""
                        //sh "cat /var/lib/jenkins/workspace/${env.JOB_NAME}/yaml/deploy.yaml"
                        withCredentials([gitUsernamePassword(credentialsId: 'git-cd')]) {
                            sh """
                            git add .
                            git commit -m "Deploy ${env.JOB_NAME} ${env.BUILD_NUMBER}"
                            git push https://github.com/imyujinsim/bastion-final-cd.git
			    """
                        }                      
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        env.pushYamlResult=true
                    } catch (error) {
                        print(error)
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        env.pushYamlResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
	}
    }
}
