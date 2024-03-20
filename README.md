stage ('Checkout') {
    steps {
        withPipelineCredentials(){
            script{
                // Setup Java version
                env.java_container = "ubn22-mvn3-${params.JAVA_VERSION}-nodejs18"
                echo "Java container: ${env.java_container}"
                
                // Install Java 11 SDK
                sh 'sudo apt-get update'
                sh 'sudo apt-get install -y openjdk-11-jdk'
                
                env.GIT_USR = PIPELINE_USR
                env.GITHUB_USR = PIPELINE_USR
                env.GIT_PSW = PIPELINE_PSW

                // Clone repo
                gitScmClone(
                    repository: params.GIT_REPO_URL,
                    branch: params.GIT_REPO_BRANCH,
                    targetDir: "."                
                )
            }
        }
    }
}
