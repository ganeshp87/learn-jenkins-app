pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '5a82be02-297d-4084-a386-6b469b933782'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
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
                    echo 'Small Changes'
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
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            test -f build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.49.1-noble'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'Playwright Local Report',
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                script {
                    sh '''
                        npm install netlify-cli node-jq
                        node_modules/.bin/netlify --version
                        echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                        node_modules/.bin/netlify status
                        node_modules/.bin/netlify deploy --dir=build
                        node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    '''
                    
                    // Capture deploy URL and assign it to STAGING_URL
                    env.STAGING_URL = sh(
                        script: "node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json",
                        returnStdout: true
                    ).trim()

                    // Print the captured URL for debugging
                    echo "Staging URL: ${env.STAGING_URL}"

                    // Wait until the staging site is live
                    sh """
                        echo "Waiting for staging site to be live at $STAGING_URL..."
                        START_TIME=\$(date +%s)
                        TIMEOUT=300 # 5 minutes
                        while :; do
                            if curl -s -o /dev/null -w "%{http_code}" $STAGING_URL | grep -q 200; then
                                echo "Staging site is live!"
                                break
                            fi
                            CURRENT_TIME=\$(date +%s)
                            ELAPSED_TIME=\$((CURRENT_TIME - START_TIME))
                            if [ \$ELAPSED_TIME -gt \$TIMEOUT ]; then
                                echo "Timed out waiting for $STAGING_URL to be live."
                                exit 1
                            fi
                            echo "Still waiting for $STAGING_URL..."
                            sleep 5
                        done
                    """
                }
            }
        }

        stage('Staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }
            steps {
                withEnv(["CI_ENVIRONMENT_URL=${env.STAGING_URL}"]) {
                    sh '''
                        npx playwright test --reporter=html
                    '''
                }
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Staging E2E Report',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Ready to deploy?', ok: 'Yes, I am sure I want to deploy.'
                }
            }
        }

        stage('Deploy Prod') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'https://ganeshp1.netlify.app'
            }
            steps {
                withEnv(["CI_ENVIRONMENT_URL=${env.CI_ENVIRONMENT_URL}"]) {
                    sh '''
                        npx playwright test --reporter=html
                    '''
                }
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Prod E2E Report',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }
}
