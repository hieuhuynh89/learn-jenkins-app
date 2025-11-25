pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    ls -la
                    node --version
                    npm --version
                    npm ci --no-audit --no-fund
                    npm run build
                    ls -la build
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            set -e
                            test -f build/index.html
                            npm ci --no-audit --no-fund || true
                            CI=true npm test -- --watchAll=false
                        '''
                    }
                    post {
                        always {
                            junit 'jest-junit/junit.xml'
                        }
                    }
                }

                stage('E2E Test') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            set -e
                            test -f build/index.html

                            # ensure dependencies are present
                            npm ci --no-audit --no-fund || true
                            npm install --no-audit --no-fund serve || true

                            # start a stable static server on port 3000 and capture logs
                            npx serve -s build -l 3000 > serve.log 2>&1 &

                            # wait for the server to become available (max ~30s)
                            for i in $(seq 1 30); do
                              if curl -sSf http://localhost:3000 >/dev/null 2>&1; then
                                echo "server is up"
                                break
                              fi
                              sleep 1
                            done

                            # show last lines of serve log for debugging
                            tail -n 200 serve.log || true

                            # run playwright tests
                            npx playwright test
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }
    }
}