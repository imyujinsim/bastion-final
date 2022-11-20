pipeline {
    agent any

    tools {
      // Jenkins 'Global Tool Configuration' 에 설정한 버전과 연동
      maven "Maven"
      dockerTool "docker"
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
			cd deploy
                        cp /var/lib/jenkins/workspace/${env.JOB_NAME}/target/*.jar .
                        """
                        env.mavenBuildResult=true
                    } catch (error) {
                        print(error)
                        echo 'Failed to build jar file'
                        env.mavenBuildResult=false
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
ENV JAVA_OPTS="-XX:InitialRAMPercentage=40.0 -XX:MaxRAMPercentage=80.0"
ADD ./deploy/${ECR_IMAGE}.jar /home/${ECR_IMAGE}.jar
CMD nohup java -jar -Dspring.profiles.active="mysql" /home/${ECR_IMAGE}.jar 1> /dev/null 2>&1
EXPOSE 8080
EOF"""
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
    }
}
