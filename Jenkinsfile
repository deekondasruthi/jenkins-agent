pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo "Building branch: ${env.BRANCH_NAME}"
                sh 'ls -l'
                echo "DEBUG: Current branch is ${env.BRANCH_NAME}"
            }
        }

        stage('Print File Content') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'sub-1') {
                        echo "📂 Printing sample.yaml content from sub-1"
                        sh '''
                            echo "---------------- sample.yaml ----------------"
                            cat sample.yaml || echo "❌ sample.yaml not found in sub-1 branch"
                            echo "--------------------------------------------"
                        '''
                    } else if (env.BRANCH_NAME == 'sub-2') {
                        echo "📂 Printing dockerfile content from sub-2"
                        sh '''
                            echo "---------------- dockerfile ----------------"
                            cat dockerfile || echo "❌ dockerfile not found in sub-2 branch"
                            echo "-------------------------------------------"
                        '''
                    } else {
                        echo "⚠️ No specific file to print for this branch: ${env.BRANCH_NAME}"
                    }
                }
            }
        }

        stage('Deploy to ENV') {
            when {
                anyOf {
                    branch 'sub-1'
                    branch 'sub-2'
                }
            }
            steps {
                script {
                    if (env.BRANCH_NAME == 'sub-1') {
                        echo "🚀 Deploying to SUB-1 environment"
                        // deployment steps for sub-1
                    } else if (env.BRANCH_NAME == 'sub-2') {
                        echo "🚀 Deploying to SUB-2 environment"
                        // deployment steps for sub-2
                    }
                }
            }
        }
    }
}

