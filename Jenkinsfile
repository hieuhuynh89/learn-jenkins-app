pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID =  '1387e1da-a59f-4ca3-93fa-6379b677d4f0'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = '1.2.3'
    }

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
                    echo "add small changes to force rebuild"
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
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
                            npm test
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
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test  --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    //image 'node:18-alpine'
                    image 'node:18-bullseye'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    echo "Deploying staging to Netlify site ID: $NETLIFY_SITE_ID using netlify-cli"
                    # install netlify-cli locally (cache may speed this up)
                    npm ci --no-audit --no-fund || true
                    npm install --no-audit --no-fund netlify-cli node-jq
                    npx netlify --version

                    # show current site/project info
                    npx netlify status --auth=$NETLIFY_AUTH_TOKEN || true

                    # Direct deploy of pre-built artifacts. This uploads build/ contents directly.
                    npx netlify deploy --dir=build --json > deploy-output.json
                    node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                '''
                script {
                    env.STAGING_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json", returnStdout: true).trim()
                }   
            }
        }

        stage('Staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }

            steps {
                sh '''
                    npx playwright test  --reporter=html
                    '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Approval'){
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                     input message: 'Are you ready to Deploy', ok: 'Yes, I\'m ok to Deploy'
                }
               
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    //image 'node:18-alpine'
                    image 'node:18-bullseye'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    echo "Deploying prod to Netlify site ID: $NETLIFY_SITE_ID using netlify-cli"
                    # install netlify-cli locally (cache may speed this up)
                    npm ci --no-audit --no-fund || true
                    npm install --no-audit --no-fund netlify-cli
                    npx netlify --version

                    # show current site/project info
                    npx netlify status --auth=$NETLIFY_AUTH_TOKEN || true

                    # Direct deploy of pre-built artifacts. This uploads build/ contents directly.
                    npx netlify deploy --dir=build --prod --site=$NETLIFY_SITE_ID --auth=$NETLIFY_AUTH_TOKEN --message="Deployed by Jenkins build $BUILD_NUMBER"
                '''
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://hieuhuynh-learning-jenkins.netlify.app/'
            }

            steps {
                sh '''
                    npx playwright test  --reporter=html
                    '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
