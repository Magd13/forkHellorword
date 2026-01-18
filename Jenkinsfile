pipeline {
    agent none
    stages {
        // Aqui he instalado dependecias que jenkins no contaba como coverage, bandit, etc
        stage('Install deps') {
            agent {
                docker {
                    image 'python:3.11'
                }
            }
            steps {
                sh '''
                    whoami
                    hostname
                    echo $WORKSPACE

                    export PATH=$HOME/.local/bin:$PATH
                    python3 -m pip install --upgrade pip
                    python3 -m pip install flask coverage pytest bandit
                '''
                stash name: 'deps', includes: '**/*'
            }
        }

        stage ('Tests') {
            // Aqui ejecuto las pruebas unitarias y rest tal cual se expuso en clase y ejecutandolas en paralelo
            parallel {
                stage ('Unit') {
                    agent {
                        docker {
                            image 'python:3.11'
                        }
                    }
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            unstash 'deps'
                            sh '''
                                whoami
                                hostname
                                echo $WORKSPACE

                                PYTHONPATH=.
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
                            image 'python:3.11'
                        }
                    }
                    steps {
                        unstash 'deps'
                        sh '''
                            whoami
                            hostname
                            echo $WORKSPACE
                            
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
        
                            PYTHONPATH=$WORKSPACE
                            python3 -m pytest --junitxml=result_rest.xml test/rest
                        '''
                        junit 'result_rest.xml'
                    }
                }
                // Pruebas de seguridad de codigo estatico
                stage ('Static && Security') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            unstash 'deps'
                            sh '''
                                whoami
                                hostname
                                echo $WORKSPACE

                                export PATH=$HOME/.local/bin:$PATH
                                flake8 --exit-zero --format=pylint app >flake8.out
                                bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
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
                }
            }
            steps {
                sh'''
                    whoami
                    hostname
                    echo $WORKSPACE

                    /opt/jmeter/bin/jmeter \
                      -n \
                      -t test/jmeter/flask.jmx \
                      -l flask.jtl \
                      -f
                '''
                perfReport sourceDataFiles: 'flask.jtl'
            }
        }
        // Pruebas de cobertura
        stage ('Cobertura') {
            agent {
                docker {
                    image 'python:3.11'
                }
            }
            steps {
                unstash 'coverage'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                      whoami
                      hostname
                      echo $WORKSPACE
                    '''
                    recordCoverage qualityGates: [[criticality: 'ERROR', integerThreshold: 85, metric: 'LINE', threshold: 85.0], [criticality: 'NOTE', integerThreshold: 95, metric: 'LINE', threshold: 95.0], [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0], [criticality: 'NOTE', integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]], tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']]
                }
            }
        }
    }
    post {
        always {
            agent {
                docker {
                    image 'alpine'
                }
            }
            steps {
                cleanWs()
            }
        }
    }
}