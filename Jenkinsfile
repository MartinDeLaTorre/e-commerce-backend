pipeline {
    agent {
        kubernetes {
          label 'idk'
          yamlFile 'pipeline-pod.yml'
        }
    }
    environment {
        registry = 'elrintowser/p3-backend'
        dockerHubCredentials = 'dockerHubCredentials'
        dockerImage = ''
        liveBranch = ''
        newColor = ''
    }
    stages {
        stage('Obtain live branch') {
            steps {
                container('kubectl') {
                    script{
                        liveBranch = sh(script:"kubectl get svc back-end-service -n p3-space -o=jsonpath='{.spec.selector.color}'", returnStdout:true).trim()
                        if (liveBranch.equals("blue")) {
                                newColor = "green"
                        } else {
                                newColor = "blue"
                        }
                    }
                }
            }
        }
        stage('Build') {
            steps {
                container('maven'){
                    echo 'Building...'
                    sh "export DB_PLATFORM=org.hibernate.dialect.H2Dialect"
                    sh "export DB_URL=jdbc:h2:mem:test;MODE=PostgreSQL"
                    sh "export DB_DRIVER=org.h2.Driver"
                    sh "mvn package"
                }
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Building Docker Image') {
            steps {
                container('docker') {
                    echo 'Building Image..'
                    script {
                        echo 'NOW BUILDING DOCKER IMAGE'
                        dockerImage = docker.build "$registry"
                    }
                }
            }
        }
        stage('Pushing Docker Image') {
            steps {
                container('docker') {
                    echo 'Pushing..'
                        script {
                            echo "NOW PUSHING TO DOCKER HUB"
                            docker.withRegistry('', dockerHubCredentials){
                                dockerImage.push("$currentBuild.number")
                                dockerImage.push("latest")
                            }
                        }
                    }
            }
        }
        stage('Deploy to test environment') {
            steps {
                container('kubectl') {
                    sh "kubectl delete -f ./resources/back-end-deployment-$newColor" + ".yml -n p3-space"
                    sh "kubectl apply -f ./resources/back-end-deployment-$newColor" + ".yml -n p3-space"
                }
            }
        }

        stage('k6 load & smoke tests'){
            steps {
                container('k6'){
                    sh "k6 run stress-test-$newColor" + ".js"
                    sh "k6 run smoke-test-$newColor" + ".js"
                }
            }
        }

        stage('Production Approve Request'){
            steps {
                script {
                    try {
                        approved = input message: 'Deploy to production?', ok: 'Continue',
                            parameters: [choice(name: 'approved?', choices: 'Yes\nNo', description: 'Deploy this build to production?')]
                        if(approved != 'Yes'){
                            error('Build not approved')
                        }
                    } catch (error){
                        error('Uh Oh')
                    }
                }
            }
        }

        stage('Deploy to Production') { //Switching service
            steps {
                container('kubectl') {
                    script {
                        sh "kubectl patch -f ./resources/back-end-service.yml -p '{\"spec\":{\"selector\":{\"color\":\"$newColor" + "\"}}}'"
                    }
                }
            }
        }
    }
}