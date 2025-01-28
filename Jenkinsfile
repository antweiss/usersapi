pipeline {
    agent any

    environment {
        COMPONENT = 'users-api'
        POSTGRES_HOST = 'postgres'
        POSTGRES_PASSWORD = credentials('postgres-password')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Python and Test') {
            steps {
                script {
                    // Start PostgreSQL container
                    docker.image('postgres:latest').withRun('-e POSTGRES_PASSWORD=${POSTGRES_PASSWORD} -e POSTGRES_DB=userdb') { pg ->
                        // Get the container ID for network connection
                        def postgresContainer = pg.id
                        
                        // Run Python container with tests
                        docker.image('python:3.9').inside("--link ${postgresContainer}:postgres") {
                            sh '''
                                python -m pip install --upgrade pip
                                pip install flake8 pylint black bandit nose
                                pip install -r requirements.txt

                                # Black formatter check
                                black --check .

                                # Flake8
                                flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
                                flake8 . --count --exit-zero --max-complexity=10 --max-line-length=120 --statistics

                                # Pylint
                                pylint **/*.py || true

                                # Run tests
                                POSTGRES_HOST=postgres \
                                POSTGRES_PORT=5432 \
                                python -m nose tests.py

                                # Bandit security scanner
                                bandit -r . -f sarif -o bandit-report.sarif || \
                                echo "BANDIT_FAIL=true" > bandit.status
                            '''

                            // Check Bandit results
                            sh '''
                                if [ -f bandit.status ]; then
                                    exit 1
                                fi
                            '''
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${env.COMPONENT}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    parallel(
                        "Scan Docker Image": {
                            sh '''
                                # Install Grype
                                curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | \
                                sh -s -- -b /usr/local/bin

                                # Scan image
                                grype ${COMPONENT}:${BUILD_NUMBER} --fail-on medium -o sarif > grype-report.sarif || \
                                echo "GRYPE_FAIL=true" > grype.status
                            '''

                            // Check Grype results
                            sh '''
                                if [ -f grype.status ]; then
                                    exit 1
                                fi
                            '''
                        },
                        "Scan Helm Charts": {
                            sh '''
                                # Install Kubescape
                                curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash

                                # Scan Helm charts
                                kubescape scan chart/ --format sarif --output kubescape-report.sarif || true
                            '''
                        }
                    )
                }
            }
        }

        // Uncomment and configure as needed for deployment
        /*
        stage('Deploy') {
            when {
                branch 'main'
            }
            environment {
                DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
            }
            steps {
                sh '''
                    echo $DOCKER_CREDENTIALS_PSW | docker login -u $DOCKER_CREDENTIALS_USR --password-stdin
                    docker push ${COMPONENT}:${BUILD_NUMBER}
                '''
            }
        }
        */
    }

    post {
        always {
            // Clean up
            node(null) {
                cleanWs()
                sh 'docker system prune -f'
            }
        }
    }
}
