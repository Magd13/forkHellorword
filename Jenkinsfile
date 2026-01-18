pipeline {
    agent none
    stages {
        // Aqui he instalado dependecias que jenkins no contaba como coverage, bandit, etc
        stage('Install deps') {
            parallel {
                stage('Install Dependencias') {
                    agent {
                        docker {
                            image 'python:3.11'
                            args '-u 131:139'
                        }
                    }
                    steps {
                        sh '''
                            id
                            hostname
                            echo "$WORKSPACE"

                            mkdir -p "$WORKSPACE/.deps"
                            python3 -m pip install \
                            --no-cache-dir \
                            --target="$WORKSPACE/.deps" \
                            flask coverage pytest bandit flake8
                        '''
                        stash name: 'deps', includes: '.deps/**'
                    }
                }
                stage('Construir imagen Docker para pruebas REST') {
                    agent {
                        node {
                            label 'built-in'
                        }
                    }
                    steps {
                        sh '''
                          docker build \
                            -t python-java:3.11 \
                            -f Dockerfile.python-java .
                        '''
                    }
                }
            }
        }

        stage ('Tests') {
            // Aqui ejecuto las pruebas unitarias y rest tal cual se expuso en clase y ejecutandolas en paralelo
            parallel {
                stage ('Unit') {
                    agent {
                        docker {
                            image 'python:3.11'
                            args '-u 131:139'
                        }
                    }
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            unstash 'deps'
                            sh '''
                                id
                                hostname
                                echo "$WORKSPACE"

                                export PYTHONPATH="$WORKSPACE/.deps:$WORKSPACE"
                                export PATH="$WORKSPACE/.deps/bin:$PATH"

                                python3 -m coverage run --branch --source=app --omit=app//__init__.py,app//api.py \
                                    -m pytest test/unit --junitxml=result_unit.xml
                                python3 -m coverage xml -o coverage.xml
                            '''   
                            junit 'result_unit.xml'
                            stash name: 'coverage', includes: 'coverage.xml'
                        }
                    }
                }
                stage('Rest') {
                    agent {
                        docker {
                            image 'python-java:3.11'
                            args '-u 131:139'
                        }
                    }
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            unstash 'deps'
                            sh '''
                                id
                                hostname
                                echo "$WORKSPACE"

                                export PYTHONPATH="$WORKSPACE/.deps:$WORKSPACE"
                                export PATH="$WORKSPACE/.deps/bin:$PATH"

                                export FLASK_APP=app/api.py

                                python3 -m flask run --host=0.0.0.0 --port=5000 &

                                mkdir -p tools/wiremock
                                if [ ! -f tools/wiremock/wiremock-standalone-3.3.1.jar ]; then
                                  curl -L -o tools/wiremock/wiremock-standalone-3.3.1.jar \
                                    https://repo1.maven.org/maven2/org/wiremock/wiremock-standalone/3.3.1/wiremock-standalone-3.3.1.jar
                                fi

                                java -jar tools/wiremock/wiremock-standalone-3.3.1.jar \
                                  --port 9090 \
                                  --root-dir test/wiremock &

                                sleep 5


                                python3 -m pytest --junitxml=result_rest.xml test/rest
                            '''
                            junit 'result_rest.xml'
                        }
                    }
                }
                // Pruebas de seguridad de codigo estatico
                stage ('Static && Security') {
                    agent {
                        docker {
                            image 'python:3.11'
                            args '-u 131:139'
                        }
                    }
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            unstash 'deps'
                            sh '''
                                id
                                hostname
                                echo "$WORKSPACE"

                                export PYTHONPATH="$WORKSPACE/.deps"
                                export PATH="$WORKSPACE/.deps/bin:$PATH"

                                flake8 --exit-zero --format=pylint app >flake8.out
                                bandit --exit-zero -r app -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                            '''
                            recordIssues qualityGates: [[criticality: 'NOTE', integerThreshold: 8, threshold: 8.0, type: 'TOTAL'], [criticality: 'ERROR', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']], tools: [flake8(pattern: 'flake8.out')]
                            recordIssues qualityGates: [[criticality: 'NOTE', integerThreshold: 2, threshold: 2.0, type: 'TOTAL'], [criticality: 'FAILURE', integerThreshold: 4, threshold: 4.0, type: 'TOTAL']], tools: [pyLint(pattern: 'bandit.out')]
                        }
                    }
                }
            }
        }
        // Pruebas de rendimiento
        stage('Performance') {
            agent {
                docker {
                    image 'justb4/jmeter'
                    args '-u 131:139'
                }
            }
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh'''
                        id
                        hostname
                        echo "$WORKSPACE"

                        /jmeter/bin/jmeter \
                          -n \
                          -t test/jmeter/flask.jmx \
                          -l flask.jtl \
                          -f
                    '''
                    perfReport sourceDataFiles: 'flask.jtl'
                }
            }
        }
        // Pruebas de cobertura
        stage ('Cobertura') {
            agent {
                docker {
                    image 'python:3.11'
                    args '-u 131:139'
                }
            }
            steps {
                unstash 'coverage'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                      id
                      hostname
                      echo "$WORKSPACE"
                    '''
                    recordCoverage qualityGates: [[criticality: 'ERROR', integerThreshold: 85, metric: 'LINE', threshold: 85.0], [criticality: 'NOTE', integerThreshold: 95, metric: 'LINE', threshold: 95.0], [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0], [criticality: 'NOTE', integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]], tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']]
                }
            }
        }
    }
    post {
        always {
            node('built-in') {
                cleanWs()
            }
        }
    }
}