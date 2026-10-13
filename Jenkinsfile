pipeline {
    agent { label 'git-test' }

    stages {
        stage('Checkout Backend') {
            steps {
                dir('backend') {
                    git branch: 'homemall/dockerization',
                        url: 'https://github.com/chweca/mall.git'
                }
                stash name: 'helm-chart',
                      includes: 'backend/deploy/helm/homemall-chart/**'
            }
        }

        stage('Check Backend') {
            steps {
                dir('backend') {
                    sh 'pwd'
                    sh 'git branch --show-current'
                    sh 'git log -1 --oneline'
                    sh 'ls -la'
                }
            }
        }

        stage('Build Backend') {
            steps {
                dir('backend') {
                    sh '''
                        mvn clean package \
                          -pl mall-admin \
                          -am \
                          -DskipTests \
                          -Ddocker.skip=true
                    '''
                }
            }
        }

        stage('Check Backend Artifact') {
            steps {
                dir('backend') {
                    sh 'ls -lh mall-admin/target/mall-admin-1.0-SNAPSHOT.jar'
                }
            }
        }

        stage('Build Backend Image') {
            steps {
                dir('backend/mall-admin') {
                    sh 'docker build -t homemall-mall-admin:${BUILD_NUMBER} .'
                }
            }
        }

        stage('Tag Backend Image') {
            steps {
                sh '''
                    docker tag \
                      homemall-mall-admin:${BUILD_NUMBER} \
                      10.10.10.10/homemall/mall-admin:${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Backend Image') {
            steps {
                sh 'docker push 10.10.10.10/homemall/mall-admin:${BUILD_NUMBER}'
            }
        }

        stage('Checkout Frontend') {
            steps {
                dir('frontend') {
                    git branch: 'homemall/dockerization',
                        url: 'https://github.com/chweca/mall-admin-web.git'
                }
            }
        }

        stage('Check Frontend') {
            steps {
                dir('frontend') {
                    sh 'pwd'
                    sh 'git branch --show-current'
                    sh 'git log -1 --oneline'
                    sh 'ls -la'
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh '''
                        npm install
                        npm run build
                    '''
                }
            }
        }

        stage('Check Frontend Artifact') {
            steps {
                dir('frontend') {
                    sh 'ls -lh dist'
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                dir('frontend') {
                    sh 'docker build -t homemall-admin-web:${BUILD_NUMBER} .'
                }
            }
        }

        stage('Tag Frontend Image') {
            steps {
                sh '''
                   docker tag \
                    homemall-admin-web:${BUILD_NUMBER} \
                    10.10.10.10/homemall/admin-web:${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Frontend Image') {
            steps {
                sh 'docker push 10.10.10.10/homemall/admin-web:${BUILD_NUMBER}'
            }
        }

        stage('Deploy to Kubernetes') {
            agent { label 'k8s-master' }

            steps {
                unstash 'helm-chart'

                sh '''
                     helm upgrade homemall backend/deploy/helm/homemall-chart \
                     -n homemall-dev \
                     --set-string adminWeb.image.tag=${BUILD_NUMBER} \
                     --set-string mallAdmin.image.tag=${BUILD_NUMBER}

                     kubectl rollout status deployment/admin-web \
                     -n homemall-dev --timeout=120s

                     kubectl rollout status deployment/mall-admin \
                     -n homemall-dev --timeout=120s
                 '''
            }
        }
    }
}
