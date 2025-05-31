pipeline {
    agent any

    parameters {
        string(name: 'CLIENT', defaultValue: '', description: 'Client code (e.g., hhco, hhil)')
        string(name: 'DB_NAME', defaultValue: 'default_db', description: 'Database name')
        text(
            name: 'SQL_COMMAND',
            defaultValue: '',
            description: 'Multi-line SQL (only SELECT, DELETE, UPDATE)'
        )
    }

    stages {
        stage('Validate Branch & Map Host') {
            steps {
                script {
                    def branchName = env.BRANCH_NAME
                    def validEnvs = ['dev', 'stage', 'qa', 'prep', 'prod']
                    if (!validEnvs.contains(branchName)) {
                        error "Branch '${branchName}' is not supported"
                    }
                    if (!params.CLIENT?.trim()) {
                        error "CLIENT parameter is empty"
                    }
                    if (!params.DB_NAME?.trim()) {
                        error "DB_NAME parameter is empty"
                    }
                    env.CLIENT = params.CLIENT.trim()
                    env.DB_NAME = params.DB_NAME.trim()
                    env.TARGET_HOST = "${env.CLIENT}.db.${branchName}.corvesta.net"
                    echo "Target host: ${env.TARGET_HOST}"
                }
            }
        }

        stage('Security & Count Check') {
            when {
                expression { return params.SQL_COMMAND?.trim()?.length() > 0 }
            }
            steps {
                script {
                    // Disallow dangerous keywords
                    def forbidden = [
                        '(?i)\\bDROP\\b',
                        '(?i)\\bALTER\\b',
                        '(?i)\\bTRUNCATE\\b',
                        '(?i)\\bCREATE\\b',
                        '(?i)\\bGRANT\\b',
                        '(?i)\\bREVOKE\\b'
                    ]
                    forbidden.each { pat ->
                        if ((params.SQL_COMMAND =~ pat).find()) {
                            error "Forbidden keyword in SQL: ${pat}"
                        }
                    }

                    // Simple COUNT(*) check for single-table DELETE/UPDATE
                    def sqlText = params.SQL_COMMAND.trim()
                    def matcher = sqlText =~ /(?i)^\s*(DELETE|UPDATE)\s+FROM\s+([^\s]+)\s+WHERE\s+(.+?);?\s*$/
                    if (matcher.find()) {
                        def table = matcher.group(2)
                        def whereClause = matcher.group(3)
                        def countQuery = "SELECT COUNT(*) FROM ${table} WHERE ${whereClause};"
                        echo "Running COUNT check: ${countQuery}"
                        withCredentials([usernamePassword(credentialsId: 'pg-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                            def rawCount = sh(
                                script: "PGPASSWORD='${DB_PASS}' psql -h ${env.TARGET_HOST} -U ${DB_USER} -d ${env.DB_NAME} -t -c \"${countQuery}\"",
                                returnStdout: true
                            ).trim()
                            def rowCount = rawCount.replaceAll(/\s+/, '')
                            echo "Rows affected: ${rowCount}"
                        }
                    }

                    // Wrap SQL in a transaction
                    def wrapped = "BEGIN;\n${sqlText}\nCOMMIT;"
                    writeFile file: 'wrapped_query.sql', text: wrapped
                }
            }
        }

        stage('Execute SQL') {
            when {
                expression { return params.SQL_COMMAND?.trim()?.length() > 0 }
            }
            steps {
                script {
                    echo "Executing wrapped_query.sql on ${env.TARGET_HOST}/${env.DB_NAME}"
                    withCredentials([usernamePassword(credentialsId: 'pg-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                        sh """
                          PGPASSWORD='${DB_PASS}' \
                          psql -h ${env.TARGET_HOST} \
                               -U ${DB_USER} \
                               -d ${env.DB_NAME} \
                               -v ON_ERROR_STOP=1 \
                               -f wrapped_query.sql
                        """.stripIndent()
                    }
                }
            }
        }
    }

    post {
        success { echo "SQL executed successfully" }
        failure { echo "SQL execution failed" }
    }
}
