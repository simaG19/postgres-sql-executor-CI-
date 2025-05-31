pipeline {
    agent any

    parameters {
        // 1) CLIENT: short code to identify which hostname to use (e.g., "hhco", "hhil", etc.)
        string(
            name: 'CLIENT',
            defaultValue: '',
            description: 'Enter the client code (e.g., hhco, hhil, hhva, hhmd).'
        )

        // 2) DB_NAME: the name of the PostgreSQL database on that host (default is a placeholder)
        string(
            name: 'DB_NAME',
            defaultValue: 'default_db',
            description: 'Enter the target PostgreSQL database name (e.g., hhco_orders).'
        )

        // 3) SQL_COMMAND: a multi-line text area for user‐supplied SQL (SELECT, DELETE, UPDATE only)
        text(
            name: 'SQL_COMMAND',
            defaultValue: '',
            description: '''\
            Paste your multi-line SQL here (only SELECT, DELETE, or UPDATE statements). For example:
            DELETE FROM users
            WHERE active = false;
            '''
        )
    }

    stages {
        stage('Validate Branch & Map to Host') {
            steps {
                script {
                    // ---- 1) Read and validate the branch name ----
                    def branchName = env.BRANCH_NAME
                    def validEnvs = ['dev', 'stage', 'qa', 'prep', 'prod']
                    if (!validEnvs.contains(branchName)) {
                        error "❌ Branch '${branchName}' is not a supported environment. Aborting."
                    }
                    echo "✅ Detected branch: ${branchName}"

                    // ---- 2) Validate the CLIENT parameter ----
                    if (params.CLIENT == null || params.CLIENT.trim().length() == 0) {
                        error "❌ The CLIENT parameter is empty. Please provide a valid client code."
                    }
                    def client = params.CLIENT.trim()
                    echo "→ Using CLIENT = '${client}'"

                    // ---- 3) Validate the DB_NAME parameter ----
                    if (params.DB_NAME == null || params.DB_NAME.trim().length() == 0) {
                        error "❌ The DB_NAME parameter is empty. Please provide a valid database name."
                    }
                    def dbName = params.DB_NAME.trim()
                    echo "→ Using DB_NAME = '${dbName}'"

                    // ---- 4) Construct the target host URL ----
                    def targetHost = "${client}.db.${branchName}.corvesta.net"
                    echo "→ Mapped branch '${branchName}' + client '${client}' → target host: ${targetHost}"

                    // ---- 5) Save for later stages ----
                    env.TARGET_HOST = targetHost
                    env.CLIENT = client
                    env.DB_NAME = dbName

                    // ---- 6) Echo raw SQL_COMMAND (for now, just to record what was passed) ----
                    if (params.SQL_COMMAND == null || params.SQL_COMMAND.trim().length() == 0) {
                        echo "→ SQL_COMMAND is empty. No SQL will be executed until Step 3 is implemented."
                    } else {
                        echo "→ SQL_COMMAND provided (showing raw text):"
                        echo "----- BEGIN SQL_COMMAND -----"
                        echo "${params.SQL_COMMAND}"
                        echo "------ END SQL_COMMAND ------"
                    }
                }
            }
        }

        // (Future: Step 3 will add stages for Security Checks, Count Check, and SQL execution.)
    }

    // (Future: Add 'post { success { … } failure { … } }' blocks in Step 4.)
}
