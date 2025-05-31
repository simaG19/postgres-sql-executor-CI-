pipeline {
    agent any

    stages {
        stage('Validate Branch & Map to Host') {
            steps {
                script {
                    // 1. Read the current branch name
                    def branchName = env.BRANCH_NAME  // e.g. "dev", "stage", "qa", "prep", or "prod"
                    
                    // 2. Define which branch names are valid
                    def validEnvs = ['dev', 'stage', 'qa', 'prep', 'prod']
                    if (!validEnvs.contains(branchName)) {
                        error "❌ Branch '${branchName}' is not a supported environment. Aborting."
                    }
                    
                    // 3. (No CLIENT parameter yet – we’ll add that in Step 2)
                    //    For now, just demonstrate mapping branch → host suffix
                    echo "✅ Detected branch: ${branchName}"
                    
                    // 4. Show how you’d build the host once CLIENT is provided
                    //    (We’ll replace 'CLIENT_PLACEHOLDER' with the real CLIENT param later)
                    def exampleClient = 'CLIENT_PLACEHOLDER'
                    def targetHost = "${exampleClient}.db.${branchName}.corvesta.net"
                    
                    echo "Mapped branch '${branchName}' → target host '${targetHost}'"
                    
                    // 5. Store the computed host in an environment variable for later stages:
                    env.TARGET_HOST = targetHost
                }
            }
        }
    }
}
