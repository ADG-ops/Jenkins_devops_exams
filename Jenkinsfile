pipeline {
    agent any
    environment {
        DOCKER_ID = "adgops"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }
    stages {

        stage('Docker Build') {
            steps {
                script {
                    sh '''
                    docker rm -f cast_service_test movie_service_test || true
                    docker build -t $DOCKER_ID/cast-service:$DOCKER_TAG ./cast-service
                    docker build -t $DOCKER_ID/movie-service:$DOCKER_TAG ./movie-service
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                    docker run -d -p 8002:8000 --name cast_service_test $DOCKER_ID/cast-service:$DOCKER_TAG
                    docker run -d -p 8001:8000 --name movie_service_test $DOCKER_ID/movie-service:$DOCKER_TAG
                    sleep 10
                    curl -f localhost:8002/api/v1/casts/docs || exit 1
                    curl -f localhost:8001/api/v1/movies/docs || exit 1
                    docker rm -f cast_service_test movie_service_test
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/cast-service:$DOCKER_TAG
                    docker push $DOCKER_ID/movie-service:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deploiement en dev') {
            environment { KUBECONFIG = credentials("config") }
            steps {
                script {
                    sh '''
                    rm -Rf ~/.kube/ && mkdir ~/.kube/
                    cat $KUBECONFIG > ~/.kube/config
                    helm upgrade --install cast-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/cast-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30001 --set imagePullSecrets=null --namespace dev
                    helm upgrade --install movie-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/movie-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30002 --set imagePullSecrets=null --namespace dev
                    '''
                }
            }
        }

        stage('Deploiement en qa') {
            environment { KUBECONFIG = credentials("config") }
            steps {
                script {
                    sh '''
                    rm -Rf ~/.kube/ && mkdir ~/.kube/
                    cat $KUBECONFIG > ~/.kube/config
                    helm upgrade --install cast-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/cast-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30003 --set imagePullSecrets=null --namespace qa
                    helm upgrade --install movie-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/movie-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30004 --set imagePullSecrets=null --namespace qa
                    '''
                }
            }
        }

        stage('Deploiement en staging') {
            environment { KUBECONFIG = credentials("config") }
            steps {
                script {
                    sh '''
                    rm -Rf ~/.kube/ && mkdir ~/.kube/
                    cat $KUBECONFIG > ~/.kube/config
                    helm upgrade --install cast-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/cast-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30005 --set imagePullSecrets=null --namespace staging
                    helm upgrade --install movie-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/movie-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30006 --set imagePullSecrets=null --namespace staging
                    '''
                }
            }
        }

        stage('Deploiement en prod') {
            when {
                expression { env.GIT_BRANCH == 'origin/master' || env.GIT_BRANCH == 'master' }
            }
            environment { KUBECONFIG = credentials("config") }
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                }
                script {
                    sh '''
                    rm -Rf ~/.kube/ && mkdir ~/.kube/
                    cat $KUBECONFIG > ~/.kube/config
                    helm upgrade --install cast-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/cast-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30007 --set imagePullSecrets=null --namespace prod
                    helm upgrade --install movie-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/movie-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30008 --set imagePullSecrets=null --namespace prod
                    '''
                }
            }
        }
    }
}
