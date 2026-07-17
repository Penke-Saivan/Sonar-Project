pipeline {
    agent any

    tools {
        nodejs 'nodejs23'
    }
    // environment {
    //     SCANNER_HOME = tool 'sonar-8.0'
    // }
    stages {
        stage('git checkout') {
            steps {
                git branch: 'dev', url: 'https://github.com/Penke-Saivan/Sonar-Project.git'
            }
        }
        stage('Frontend Compilation ') {
            steps {
                dir('client') {
                    sh 'find . -name "*.js" -exec node --check {} +'
                }
            }
        }
        stage('BAckend Compilantion') {
            steps {
                dir('api') {
                    sh 'find . -name "*.js" -exec node --check {} +'
                }
            }
        }
        stage('GitLeaks Scan') {
            steps {
                sh 'gitleaks detect --source ./client --exit-code 1'
                sh 'gitleaks detect --source ./api --exit-code 1'
            }
        }

        // stage('SonarQube-Scanner-Agent') {
        //     steps {
        //         withSonarQubeEnv('sonar-server') {
        //             // some block
        //             sh """ $SCANNER_HOME/bin/sonar-scanner  -Dsonar.projectName=NodeJS-Project \
        //              -Dsonar.projectKey=NodeJS-Project"""
        //         }
        //     }
        // }
        // stage('Quality_gates') {
        //     steps {
        //         timeout(60) {
        //             waitForQualityGate abortPipeline: false, credentialsId: 'sonar-secret'
        //         }
        //     }
        // }
        // stage('Hello') {
        //     steps {
        //         echo '------------------Hello World-----------------------'
        //     }
        // }
        stage('Build-Tag & Push Backend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('api') {
                            sh 'docker build -t tonyduck/backend-deploy:v1 .'

                            sh 'docker push tonyduck/backend-deploy:v1'
                        }
                    }
                }
            }
        }

        stage('Build-Tag & Push Frontend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('client') {
                            sh 'docker build -t tonyduck/frontend-deploy:v1 .'

                            sh 'docker push tonyduck/frontend-deploy:v1'
                        }
                    }
                }
            }
        }
        // stage('k8s-deploy') {
        //     steps {
        //         script {
        //             withKubeConfig(caCertificate: '', clusterName: 'roboshop-dev', contextName: '', credentialsId: 'k8-token', namespace: 'dev', restrictKubeConfigAccess: false, serverUrl: 'https://C5382A0D15A8423BA81DF25B26360E15.gr7.us-east-1.eks.amazonaws.com') {
        //                 // some block
        //                 sh 'kubectl get pods'
        //                 sh 'kubectl config current-context'
        //                 sh 'kubectl config view'
        //                 sh 'kubectl auth can-i get pods'

        //             }
        //         }
        //     }
        // }
        stage('k8s-deploy') {
            steps {
                script {
                    withKubeConfig(
                caCertificate: '',
                clusterName: 'roboshop-dev',
                contextName: '',
                credentialsId: 'k8-token',
                namespace: 'dev',
                restrictKubeConfigAccess: false,
                serverUrl: 'https://C5382A0D15A8423BA81DF25B26360E15.gr7.us-east-1.eks.amazonaws.com'
            ) {
                        sh '''
                set -e

                echo "===== Kubernetes Context ====="
                kubectl config current-context
                kubectl get nodes
                kubectl get pods -n dev

                echo "===== Installing MySQL ====="
                helm upgrade --install mysql ./mysql-helm -n dev

                echo "Waiting for MySQL to become Ready..."
                kubectl rollout status statefulset/mysql -n dev --timeout=180s

                echo "===== Installing Backend ====="
                helm upgrade --install backend ./backend-helm -n dev

                echo "Waiting for Backend Deployment..."
                kubectl rollout status deployment/backend -n dev --timeout=120s

                echo "===== Installing Frontend ====="
                helm upgrade --install frontend ./frontend-helm -n dev

                echo "Waiting for Frontend Deployment..."
                kubectl rollout status deployment/frontend -n dev --timeout=120s

                echo "===== Installing Ingress ALB  ====="
                helm upgrade --install ingress ./ingress-helm \
                --set rules.host=user-management \
                --set-string annotations.arn=arn:aws:acm:us-east-1:131676642204:certificate/67fa712b-0964-4ba2-af8e-b0cad6738e45

                echo "Waiting for Ingress..."
                kubectl wait --for=jsonpath='{.status.loadBalancer.ingress[0].hostname}' \
                    ingress/user-management-ingress \
                    -n dev \
                    --timeout=300s
                echo "Ingress Details:"
                kubectl get ingress -n dev
                echo "===== Deployment Complete ====="

                kubectl get pods -n dev
                kubectl get svc -n dev
                kubectl get deployments -n dev
                kubectl get statefulsets -n dev
                '''
            }
                }
            }
        }
    }
}
