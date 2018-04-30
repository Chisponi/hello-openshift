pipeline {
    agent {
        label "maven"
    }
    options { 
        skipDefaultCheckout() 
    }
    stages {
        stage("Checkout SCM") {
            steps {
                git branch: env.BRANCH, url: env.REPO
            }
        }
        stage("Compile") {
            steps {
                sh "mvn package -DskipTests"
            }
        }
        stage("Test") {
            steps {
                sh "mvn test"
            }
        }
        stage("Build Image") {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${APP}-dev") {
                            if (!openshift.selector("bc", "${APP}").exists()) {
                                openshift.newBuild("--image-stream=redhat-openjdk18-openshift:1.2", "--name=${app}").narrow("bc");                   
                            }
                            // Starts a new build and waits for its completion
                            openshift.selector("bc", "${APP}").startBuild("--from-file=target/${APP}-swarm.jar", "--wait=true");
                        }
                    }
                }
            }
        }
        stage("Deploy DEV") {
            steps {
                script {
                    openshift.verbose()
                    openshift.withCluster() {
                        openshift.withProject("${APP}-dev") {
                            if (!openshift.selector("dc", "${APP}").exists()) {
                                createApp("${APP}");                   
                            } else {
                                // Rollouts to latest version
                                openshift.selector("dc", "${APP}").rollout().latest();   
                            }          
                        }
                    }
                }
            }
        }
        stage("Deploy TEST") {
            steps {
                script {
                    // Asks for approval
                    input "Deploy TEST?"

                    openshift.withCluster {
                        openshift.withProject("${APP}-test") {
                            // Gets application version
                            pom = readMavenPom file: 'pom.xml'

                            // Promotes images between environments
                            openshift.tag("${APP}-dev/${APP}:latest", "${APP}-test/${APP}:${pom.version}");

                            if (!openshift.selector("dc", "${APP}").exists()) {
                                createApp("${APP}", pom.version);                   
                            } else {
                                // Rollouts to latest version
                                openshift.selector("dc", "${APP}").rollout().latest();   
                            }                            
                        }
                    }
                }
            }
        }
        stage("Deploy PROD") {
            steps {
                script {
                    // Asks for approval
                    input "Deploy PROD?"

                    openshift.withCluster {
                        openshift.withProject("${APP}-prod") {
                            // Gets application version
                            pom = readMavenPom file: 'pom.xml'

                            // Promotes images between environments
                            openshift.tag("${APP}-test/he${APP}llo:${pom.version}", "${APP}-prod/${APP}:${pom.version}");

                            if (!openshift.selector("dc", "${APP}").exists()) {
                                createApp("${APP}", pom.version);                   
                            } else {
                                // Rollouts to latest version
                                openshift.selector("dc", "${APP}").rollout().latest();   
                            }                            
                        }
                    }
                }
            }
        }          
    }
}

def createApp(app, tag) {
    // Creates the application and get the brand new BuildConfig
    openshift.newApp("${app}:${tag}");
    // Creates the hello Route
    openshift.selector("svc", app).expose();

    // Waits for the deployment to finish
    def latestDeploymentVersion = openshift.selector("dc", app).object().status.latestVersion
    def rc = openshift.selector("rc", "${app}-${latestDeploymentVersion}")

    rc.untilEach(1){
        def rcMap = it.object()
        return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
    }
    
    // Removes the triggers  
    openshift.set("triggers", "dc/${app}", "--manual");
}