pipeline {
    agent any

    stages {
        stage("docker login"){
            steps {
                sh 'eval $DOCKER_LOGIN_CMD'
            }
        }

        stage("Build and create PR"){
            when {
                not { branch "master" }
                not { expression { "$JOB_BASE_NAME".startsWith("PR-") } }
            }
            steps {
                script {
                    def lastTagCommit = sh(script: "git tag -l 'jk-*' --sort=committerdate | awk 'END{print}' | awk '{ gsub(\"jk-\", \"\", \$1); print \$1}'", returnStdout: true).trim()
                    def modifiedFiles = sh(script:"git log --name-only --pretty=format:$BRANCH_NAME ${lastTagCommit}..HEAD", returnStdout: true).trim()
                    Set buildedMicroservices = []

                    if( modifiedFiles ){
                        for(String modifiedFile: modifiedFiles.split("\\r?\\n")){
                            def microservice = modifiedFile.split('/')[0].trim()

                            if ( microservice != "$BRANCH_NAME" && microservice != "" && fileExists("$microservice/Dockerfile") && !buildedMicroservices.contains( microservice ) ) {
                                image = docker.build("imleo/robotshop-$microservice", "$microservice")
                                image.push( "$BRANCH_NAME-$GIT_COMMIT" )

                                sh "hub pull-request --no-edit --base master --head $BRANCH_NAME"

                                buildedMicroservices.add( microservice )
                            }
                        }
                    }

                    print "Modified microservices $buildedMicroservices"
                }
            }
        }

        stage("Deploy from master") {
            when {
                branch "master"
            }
            steps{
                script {
                    def lastTagCommit = sh(script: "git tag -l 'jk-*' --sort=committerdate | awk 'END{print}' | awk '{ gsub(\"jk-\", \"\", \$1); print \$1}'", returnStdout: true).trim()
                    def modifiedFiles = sh(script:"git log --name-only --pretty=format:$BRANCH_NAME ${lastTagCommit}..HEAD", returnStdout: true).trim()
                    Set buildedMicroservices = []

                    if( modifiedFiles ){
                        for(String modifiedFile: modifiedFiles.split("\\r?\\n")){
                            def microservice = modifiedFile.split('/')[0]

                            if ( fileExists("$microservice/Dockerfile") && !buildedMicroservices.contains( microservice )) {
                                image = docker.build("imleo/robotshop-$microservice", "$microservice")
                                image.push( "$GIT_COMMIT" )

                                sh "kubectl -n $ROBOT_SHOP_NAMESPACE set image deployments/$microservice $microservice=imleo/robotshop-$microservice:$GIT_COMMIT"

                                buildedMicroservices.add( microservice )
                            }
                        }

                        if( !buildedMicroservices.isEmpty() ){
                            sh(script: "eval $GITHUB_REMOTE_SET_URL", returnStdout: false)
                            sh "hub tag jk-$GIT_COMMIT"
                            sh "hub push origin jk-$GIT_COMMIT"
                        }
                    }

                    print "Deployed microservices $buildedMicroservices"
                }
            }
        }
    }
}
