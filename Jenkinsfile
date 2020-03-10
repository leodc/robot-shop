Set modifiedServices = []
pipeline {
    agent any

    stages {
        stage("Test"){
            when {
                not { branch "master" }
            }
            steps {
                script {
                    slackSend message: "Job Started - ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"

                    def lastTagCommit = sh(script: "git tag -l 'jk-*' --sort=committerdate | awk 'END{print}' | awk '{ gsub(\"jk-\", \"\", \$1); print \$1}'", returnStdout: true).trim()
                    def modifiedFiles = sh(script:"git log --name-only --pretty=format:$BRANCH_NAME ${lastTagCommit}..HEAD", returnStdout: true).trim()

                    if( modifiedFiles ){
                        sh "wget -q https://repo1.maven.org/maven2/org/junit/platform/junit-platform-console-standalone/1.6.0/junit-platform-console-standalone-1.6.0.jar"

                        for(String modifiedFile: modifiedFiles.split("\\r?\\n")){
                            def microservice = modifiedFile.split('/')[0].trim()

                            if ( microservice != "$BRANCH_NAME" && microservice != "" && fileExists("$microservice/Dockerfile") && !modifiedServices.contains( microservice ) ) {
                                try {
                                    sh """
                                    mkdir $microservice/build
                                    javac -d $microservice/build -cp $microservice/build:junit-platform-console-standalone-1.6.0.jar $microservice/tests/Test.java
                                    java -jar junit-platform-console-standalone-1.6.0.jar --class-path $microservice/build --scan-class-path --reports-dir=$microservice/build/test
                                    """
                                } catch(Exception e) {
                                    slackSend color: "danger", message: "Error during $microservice tests: ${e.toString()} (<${env.BUILD_URL}/console|check logs>)"
                                    junit "$microservice/build/test/*.xml"
                                    throw e
                                }

                                junit "$microservice/build/test/*.xml"

                                modifiedServices.add( microservice )
                            }
                        }

                        if( !modifiedServices.isEmpty() ){
                            slackSend color: "good", message: "All tests passed"
                        }
                    }
                }

            }
        }

        stage("Build and create PR"){
            when {
                not { branch "master" }
            }
            steps {
                sh 'eval $DOCKER_LOGIN_CMD'

                script {
                    if( !modifiedServices.isEmpty() ){
                        for(String microservice: modifiedServices){
                            try {
                                image = docker.build("imleo/robotshop-$microservice", "$microservice")
                                image.push( "$GIT_COMMIT" )
                            }catch(Exception e){
                                slackSend color: "danger", message: "Error during $microservice build: ${e.toString()} (<${env.BUILD_URL}/console|check logs>)"
                                throw e
                            }
                        }

                        def prLink = sh(script: "hub pull-request -m 'Created from jenkins' --base master --head $BRANCH_NAME", returnStdout: true).trim()
                        slackSend color: "good", message: "Pull request created: $prLink"
                    }

                    slackSend color: "good", message: "Microservices builded: $modifiedServices"
                    print "Microservices builded: $modifiedServices"
                }
            }
        }

        stage("Deploy from master") {
            when {
                branch "master"
            }
            steps{
                sh 'eval $DOCKER_LOGIN_CMD'

                script {
                    slackSend message: "Job Started - ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"

                    def lastTagCommit = sh(script: "git tag -l 'jk-*' --sort=committerdate | awk 'END{print}' | awk '{ gsub(\"jk-\", \"\", \$1); print \$1}'", returnStdout: true).trim()
                    def modifiedFiles = sh(script:"git log --name-only --pretty=format:$BRANCH_NAME ${lastTagCommit}..HEAD", returnStdout: true).trim()
                    Set buildedMicroservices = []

                    if( modifiedFiles ){
                        for(String modifiedFile: modifiedFiles.split("\\r?\\n")){
                            def microservice = modifiedFile.split('/')[0]

                            if ( fileExists("$microservice/Dockerfile") && !buildedMicroservices.contains( microservice )) {
                                image = docker.build("imleo/robotshop-$microservice", "$microservice")
                                image.push( "$GIT_COMMIT" )

                                sh "sed -i 's|image: imleo/robotshop-.*|image: imleo/robotshop-$microservice:$GIT_COMMIT|g' K8s/descriptors/$microservice-deployment.yaml"
                                sh "kubectl -n $ROBOT_SHOP_NAMESPACE apply -f K8s/descriptors/$microservice-deployment.yaml"

                                slackSend color: "good", message: "Updated microservice: $microservice"

                                buildedMicroservices.add( microservice )
                            }
                        }

                        if( !buildedMicroservices.isEmpty() ){
                            sh(script: "eval $GITHUB_REMOTE_SET_URL", returnStdout: false)
                            sh "hub tag jk-$GIT_COMMIT"
                            sh "hub push origin jk-$GIT_COMMIT"

                            slackSend color: "good", message: "Code tagged at jk-$GIT_COMMIT"
                        }
                    }

                    print "Deployed microservices $buildedMicroservices"
                }
            }
        }
    }
}
