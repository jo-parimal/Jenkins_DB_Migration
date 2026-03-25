pipeline {
    agent any

    environment {
        DATESTAMP = sh(script: "date +%Y%m%d_%H%M%S", returnStdout: true).trim()
    }

    parameters {
        choice(name: 'SOURCE_ENV', choices: ['Dev', 'QA'], description: 'Select source environment')
        choice(name: 'DEST_ENV', choices: ['Dev', 'QA'], description: 'Select destination environment')

        string(name: 'SRC_DB_NAME', defaultValue: 'db_schema_1', description: 'Source DB name')
        password(name: 'SRC_DB_PASS', defaultValue: 'Pass@123', description: 'Source DB password')

        string(name: 'DEST_DB_NAME', defaultValue: 'db_schema_3', description: 'Destination DB name')
        password(name: 'DEST_DB_PASS', defaultValue: 'Pass@123', description: 'Destination DB password')
    }

    stages {

        stage('Map Environments') {
            steps {
                script {
                    def envMap = [
                        Dev: [ip: '174.129.191.246', user: 'dbuserdev', pem: 'devlogin'],
                        QA : [ip: '52.205.252.75',  user: 'dbuserqa',  pem: 'qalogin']
                    ]

                    def src = envMap[params.SOURCE_ENV]
                    def dest = envMap[params.DEST_ENV]

                    env.SRC_IP = src.ip
                    env.SRC_USER = src.user
                    env.SRC_PEM = src.pem

                    env.DEST_IP = dest.ip
                    env.DEST_USER = dest.user
                    env.DEST_PEM = dest.pem

                    env.SCHEMA_FILE = "dump_${params.SRC_DB_NAME}_${env.DATESTAMP}.sql"
                    env.DROP_SQL_FILE = "drop_${env.DATESTAMP}.sql"
                }
            }
        }

        stage('Dump Source DB') {
            steps {
                withCredentials([file(credentialsId: env.SRC_PEM, variable: 'SRC_PEM_PATH')]) {
                    sh """
                        chmod 400 \$SRC_PEM_PATH

                        echo "Dumping DB from \$SRC_IP..."

                        ssh -o StrictHostKeyChecking=no -i \$SRC_PEM_PATH ec2-user@\$SRC_IP \\
                        "MYSQL_PWD='${params.SRC_DB_PASS}' mysqldump -u \$SRC_USER ${params.SRC_DB_NAME} \\
                        --routines --triggers --events --single-transaction > /tmp/\$SCHEMA_FILE"

                        echo "Copying dump..."

                        scp -o StrictHostKeyChecking=no -i \$SRC_PEM_PATH \\
                        ec2-user@\$SRC_IP:/tmp/\$SCHEMA_FILE .
                    """
                }
            }
        }

        stage('Create Destination DB') {
            steps {
                withCredentials([file(credentialsId: env.DEST_PEM, variable: 'DEST_PEM_PATH')]) {
                    sh """
                        chmod 400 \$DEST_PEM_PATH

                        ssh -o StrictHostKeyChecking=no -i \$DEST_PEM_PATH ec2-user@\$DEST_IP \\
                        "MYSQL_PWD='${params.DEST_DB_PASS}' mysql -u \$DEST_USER -e \\
                        \\"CREATE DATABASE IF NOT EXISTS \\\\\`${params.DEST_DB_NAME}\\\\\`;\\\""
                    """
                }
            }
        }

        stage('Drop Destination Objects') {
            steps {
                script {
                    def dropSql = """
SET FOREIGN_KEY_CHECKS = 0;

SET @tables = NULL;
SELECT GROUP_CONCAT(CONCAT('`', table_name, '`')) INTO @tables FROM information_schema.tables WHERE table_schema = '${params.DEST_DB_NAME}';
SET @stmt = IF(@tables IS NOT NULL, CONCAT('DROP TABLE IF EXISTS ', @tables), '');
PREPARE stmt FROM @stmt; EXECUTE stmt; DEALLOCATE PREPARE stmt;

SET @views = NULL;
SELECT GROUP_CONCAT(CONCAT('`', table_name, '`')) INTO @views FROM information_schema.views WHERE table_schema = '${params.DEST_DB_NAME}';
SET @stmt = IF(@views IS NOT NULL, CONCAT('DROP VIEW IF EXISTS ', @views), '');
PREPARE stmt FROM @stmt; EXECUTE stmt; DEALLOCATE PREPARE stmt;

SET @procs = NULL;
SELECT GROUP_CONCAT(ROUTINE_NAME) INTO @procs FROM information_schema.routines WHERE routine_type='PROCEDURE' AND routine_schema='${params.DEST_DB_NAME}';
SET @stmt = IF(@procs IS NOT NULL, CONCAT('DROP PROCEDURE IF EXISTS ', @procs), '');
PREPARE stmt FROM @stmt; EXECUTE stmt; DEALLOCATE PREPARE stmt;

SET @funcs = NULL;
SELECT GROUP_CONCAT(ROUTINE_NAME) INTO @funcs FROM information_schema.routines WHERE routine_type='FUNCTION' AND routine_schema='${params.DEST_DB_NAME}';
SET @stmt = IF(@funcs IS NOT NULL, CONCAT('DROP FUNCTION IF EXISTS ', @funcs), '');
PREPARE stmt FROM @stmt; EXECUTE stmt; DEALLOCATE PREPARE stmt;

SET @trgs = NULL;
SELECT GROUP_CONCAT(TRIGGER_NAME) INTO @trgs FROM information_schema.triggers WHERE trigger_schema='${params.DEST_DB_NAME}';
SET @stmt = IF(@trgs IS NOT NULL, CONCAT('DROP TRIGGER IF EXISTS ', @trgs), '');
PREPARE stmt FROM @stmt; EXECUTE stmt; DEALLOCATE PREPARE stmt;

SET @evts = NULL;
SELECT GROUP_CONCAT(EVENT_NAME) INTO @evts FROM information_schema.events WHERE event_schema='${params.DEST_DB_NAME}';
SET @stmt = IF(@evts IS NOT NULL, CONCAT('DROP EVENT IF EXISTS ', @evts), '');
PREPARE stmt FROM @stmt; EXECUTE stmt; DEALLOCATE PREPARE stmt;

SET FOREIGN_KEY_CHECKS = 1;
"""

                    writeFile file: env.DROP_SQL_FILE, text: dropSql
                }

                withCredentials([file(credentialsId: env.DEST_PEM, variable: 'DEST_PEM_PATH')]) {
                    sh """
                        chmod 400 \$DEST_PEM_PATH

                        scp -i \$DEST_PEM_PATH ${env.DROP_SQL_FILE} ec2-user@\$DEST_IP:/tmp/

                        ssh -i \$DEST_PEM_PATH ec2-user@\$DEST_IP \\
                        "MYSQL_PWD='${params.DEST_DB_PASS}' mysql -u \$DEST_USER ${params.DEST_DB_NAME} < /tmp/${env.DROP_SQL_FILE}"
                    """
                }
            }
        }

        stage('Restore DB') {
            steps {
                withCredentials([file(credentialsId: env.DEST_PEM, variable: 'DEST_PEM_PATH')]) {
                    sh """
                        chmod 400 \$DEST_PEM_PATH

                        scp -i \$DEST_PEM_PATH ${env.SCHEMA_FILE} ec2-user@\$DEST_IP:/tmp/

                        ssh -i \$DEST_PEM_PATH ec2-user@\$DEST_IP \\
                        "MYSQL_PWD='${params.DEST_DB_PASS}' mysql -u \$DEST_USER ${params.DEST_DB_NAME} < /tmp/${env.SCHEMA_FILE}"
                    """
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh """
                    rm -f ${env.SCHEMA_FILE} ${env.DROP_SQL_FILE}
                """
            }
        }
    }
}
