pipeline {
    agent any
    environment {
        NETLIFY_SITE_ID = '57034658-182e-48a0-8955-ea0b5aee83d9'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID" //This will work if apt change done also in src/App.js
    }
    stages {
        stage('Docker') {
            steps {
                sh 'docker build -t my-playwright .'
            }
        }
        stage('Build') {
            agent {
                docker {
                    //image 'node:18-alpine'
                    image 'my-playwright'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        stage('Test') {
            parallel {
                stage('Unit Test') {
                    agent {
                        docker {
                            //image 'node:18-alpine'
                            image 'my-playwright'
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
                            //image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            image 'my-playwright'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            #npm install serve
                            #node_modules/.bin/serve -s build &
                            serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }
        stage('Deploy to Staging') {
            agent {
                docker {
                    //image 'node:18-alpine'
                    image 'my-playwright'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    #npm install netlify-cli node-jq
                    #node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    netlify deploy --dir=build --json > deploy-output.json
                '''
                script {
                    //env.STAGING_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json", returnStdout: true)
                    env.STAGING_URL = sh(script: "node-jq -r '.deploy_url' deploy-output.json", returnStdout: true)
                }
            }
        }
        stage('Staging E2E') {
            agent {
                docker {
                    //image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    image 'my-playwright'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = "$env.STAGING_URL"
            }
            steps {
                sh 'npx playwright test --reporter=html'
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2e Staging HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
        stage('Deploy to Prod') {
            agent {
                docker {
                    //image 'node:18-alpine'
                    image 'my-playwright'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    #npm install netlify-cli
                    #node_modules/.bin/netlify deploy --dir=build --prod
                    netlify deploy --dir=build --prod
                '''
            }
        }
        stage('Deploy to AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    args "--entrypoint=''"
                    reuseNode true
                }
            }
            environment {
                AWS_S3_BUCKET = 'paarvbucket'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-creds', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        aws s3 ls
                        echo "Hello s3" > index.html
                        aws s3 cp index.html s3://$AWS_S3_BUCKET/index.html
                        aws s3 sync build s3://$AWS_S3_BUCKET
                    '''
                }
            }
        }
        stage('Prod E2E') {
            agent {
                docker {
                    //image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    image 'my-playwright'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'https://magenta-quokka-a2016a.netlify.app'
            }
            steps {
                sh 'npx playwright test --reporter=html'
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2e Prod HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
