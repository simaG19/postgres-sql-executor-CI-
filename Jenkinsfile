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
        string(
            name: 'SQL_FILE_PATH',
            defaultValue: '',
            description: 'Relative path to a SQL file in the repo (alternative to SQL_COMMAND)'
        )
    }

    stages {
        stage('Validate Branch & Inputs') {
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
                    if (!params.SQL_COMMAND?.trim() && !params.SQL_FILE_PATH?.trim()) {
                        error "Either SQL_COMMAND or SQL_FILE_PATH must be provided"
                    }
                    env.CLIENT = params.CLIENT.trim()
                    env.DB_NAME = params.DB_NAME.trim()
                    env.TARGET_HOST = "${env.CLIENT}.db.${branchName}.corvesta.net"
                    echo "Target host: ${env.TARGET_HOST}"
                }
            }
        }

        stage('Load SQL') {
            when {
                expression { return params.SQL_FILE_PATH?.trim()?.length() > 0 }
            }
            steps {
                script {
                    def path = params.SQL_FILE_PATH.trim()
                    if (!fileExists(path)) {
                        error "SQL file not found at: ${path}"
                    }
                    env.SQL_TEXT = readFile(path).trim()
                    echo "Loaded SQL from file: ${path}"
                }
            }
        }

        stage('Define SQL Text') {
            when {
                expression { return !params.SQL_FILE_PATH?.trim() }
            }
            steps {
                script {
                    env.SQL_TEXT = params.SQL_COMMAND.trim()
                    echo "Using inline SQL text"
                }
            }
        }

        stage('Security & Count Check') {
            when {
                expression { return env.SQL_TEXT?.trim()?.length() > 0 }
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
                        if ((env.SQL_TEXT =~ pat).find()) {
                            error "Forbidden keyword in SQL: ${pat}"
                        }
                    }

                    // Simple COUNT(*) check for single-table DELETE/UPDATE
                    // Use a one-time match to avoid storing Matcher
                    def pattern = ~/(?i)^\s*(DELETE|UPDATE)\s+FROM\s+([^\s]+)\s+WHERE\s+(.+?);?\s*$/
                    def parts = (env.SQL_TEXT =~ pattern) ? (env.SQL_TEXT =~ pattern)[0] : null
                    if (parts) {
                        def table = parts[1]
                        def whereClause = parts[2]
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
                    def wrapped = "BEGIN;\n${env.SQL_TEXT}\nCOMMIT;"
                    writeFile file: 'wrapped_query.sql', text: wrapped
                }
            }
        }

        stage('Approval for Production') {
            when {
                expression { return env.BRANCH_NAME == 'prod' && env.SQL_TEXT?.trim()?.length() > 0 }
            }
            steps {
                script {
                    input message: "Confirm execution on PROD for client ${env.CLIENT}, DB ${env.DB_NAME}?", ok: "Proceed"
                }
            }
        }

        stage('Execute SQL') {
            when {
                expression { return env.SQL_TEXT?.trim()?.length() > 0 }
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
        always {
            script {
                def status = currentBuild.currentResult
                def payload = """{
                  "branch": "${env.BRANCH_NAME}",
                  "client": "${env.CLIENT}",
                  "db": "${env.DB_NAME}",
                  "status": "${status}"
                }"""
                // Replace with actual audit endpoint
                sh "curl -X POST -H 'Content-Type: application/json' -d '${payload}' https://audit.example.com/log"
            }
        }
        success {
            echo "SQL executed successfully"
        }
        failure {
            echo "SQL execution failed"
        }
    }
}
