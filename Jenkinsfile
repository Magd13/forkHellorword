pipeline {
    agent any
    stages {
        stage('Machine') {
            steps {
                sh '''
                    pwd
                    uname -a
                  '''
            }
        }
        
        // Aqui he instalado dependecias que jenkins no contaba como coverage, bandit, etc
        stage('Install deps') {
            steps {
                sh '''
                    export PATH=$HOME/.local/bin:$PATH
                    python3 -m pip install --upgrade pip
                    python3 -m pip install flask coverage pytest bandit
                '''
            }
        }

        stage ('Tests') {
            // Aqui ejecuto las pruebas unitarias y rest tal cual se expuso en clase y ejecutandolas en paralelo
            parallel {
                stage ('Unit') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            sh '''
                                PYTHONPATH=.
                                python3 -m coverage run --branch --source=app --omit=app//__init__.py,app//api.py \
                                    -m pytest test/unit --junitxml=result_unit.xml
                                python3 -m coverage xml -o coverage.xml
                            '''   
                            junit 'result_unit.xml'
                        }
                    }
                }
                stage('Rest') {
                    steps {
                        sh '''
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
            }
        }
        // Pruebas de seguridad de codigo estatico
        stage ('Static') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        flake8 --exit-zero --format=pylint app >flake8.out
                    '''
                    recordIssues qualityGates: [[criticality: 'NOTE', integerThreshold: 8, threshold: 8.0, type: 'TOTAL'], [criticality: 'ERROR', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']], tools: [flake8(pattern: 'flake8.out')]
                }
            }
        }
        // Pruebas de seguridad ejecutadas con bandit
        stage('Securty') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh'''
                        export PATH=$HOME/.local/bin:$PATH
                        bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                        cat bandit.out
                    '''
                    recordIssues qualityGates: [[criticality: 'NOTE', integerThreshold: 2, threshold: 2.0, type: 'TOTAL'], [criticality: 'FAILURE', integerThreshold: 4, threshold: 4.0, type: 'TOTAL']], tools: [pyLint(pattern: 'bandit.out')]
                }
            }
        }
        // Pruebas de rendimiento
        stage('Performance') {
            steps {
                sh'''
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
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    recordCoverage qualityGates: [[criticality: 'ERROR', integerThreshold: 85, metric: 'LINE', threshold: 85.0], [criticality: 'NOTE', integerThreshold: 95, metric: 'LINE', threshold: 95.0], [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0], [criticality: 'NOTE', integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]], tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']]
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}