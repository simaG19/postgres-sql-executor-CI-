pipeline {
    agent any

    /************************************************************************
     * 1. PARAMETERS:                                                      *
     *    - CLIENT     : Short client code to identify which "hostname"     *
     *    - DB_NAME    : Name of the database on that host (with default)   *
     *    - SQL_COMMAND: Multi-line text area for SQL (SELECT/DELETE/UPDATE)*
     ************************************************************************/
    parameters {
        // CLIENT: e.g. "hhco", "hhil", "hhva", "hhmd"
        string(
            name: 'CLIENT', 
            defaultValue: '', 
            description: 'Enter the client code (e.g., hhco, hhil, hhva, hhmd).'
        )

        // DB_NAME: name of the database on that host; you can override if needed.
        string(
            name: 'DB_NAME', 
            defaultValue: 'default_db', 
            description: 'Enter the target PostgreSQL database name (e.g., hhco_orders).'
        )

        // SQL_COMMAND: multi-line text area, supports SELECT, DELETE, and UPDATE only.
        text(
            name: 'SQL_COMMAND', 
            defaultValue: '', 
            description: '''\
Paste your multi-line SQL (only SELECT, DELETE, or UPDATE statements). 
e.g.:
DELETE FROM users
WHERE active = false;
'''.stripIndent()
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

                    // ---- 5) Save for later stages (so we don’t have to recompute) ----
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

        // (Later: Step 3 will add stages for Security Checks, Safety Count, Execution, etc.)
    }

    // (We’ll fill in 'post' blocks in a later step to handle success/failure.)
}
