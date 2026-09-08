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
                    docker rm -f cast_service_test movie_service_test cast_db_test movie_db_test || true
                    docker network rm test_network || true
                    docker network create test_network

                    docker run -d --name cast_db_test --network test_network \
                     -e POSTGRES_USER=cast_db_username -e POSTGRES_PASSWORD=cast_db_password -e POSTGRES_DB=cast_db_dev \
                    postgres:12.1-alpine

            docker run -d --name movie_db_test --network test_network \
              -e POSTGRES_USER=movie_db_username -e POSTGRES_PASSWORD=movie_db_password -e POSTGRES_DB=movie_db_dev \
              postgres:12.1-alpine

            sleep 10

            docker run -d --name cast_service_test --network test_network -p 8002:8000 \
              -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db_test/cast_db_dev \
              $DOCKER_ID/cast-service:$DOCKER_TAG \
              uvicorn app.main:app --host 0.0.0.0 --port 8000 --loop asyncio

            docker run -d --name movie_service_test --network test_network -p 8001:8000 \
              -e DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie_db_test/movie_db_dev \
              -e CAST_SERVICE_HOST_URL=http://cast_service_test:8000/api/v1/casts/ \
              $DOCKER_ID/movie-service:$DOCKER_TAG \
              uvicorn app.main:app --host 0.0.0.0 --port 8000 --loop asyncio

            sleep 15

            curl -f localhost:8002/api/v1/casts/docs || (docker logs cast_service_test; exit 1)
            curl -f localhost:8001/api/v1/movies/docs || (docker logs movie_service_test; exit 1)

            docker rm -f cast_service_test movie_service_test cast_db_test movie_db_test
            docker network rm test_network
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
                      --set service.nodePort=30011 --set imagePullSecrets=null --namespace dev
                    helm upgrade --install movie-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/movie-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30012 --set imagePullSecrets=null --namespace dev
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
                      --set service.nodePort=30013 --set imagePullSecrets=null --namespace qa
                    helm upgrade --install movie-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/movie-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30014 --set imagePullSecrets=null --namespace qa
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
                      --set service.nodePort=30015 --set imagePullSecrets=null --namespace staging
                    helm upgrade --install movie-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/movie-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30016 --set imagePullSecrets=null --namespace staging
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
                      --set service.nodePort=30017 --set imagePullSecrets=null --namespace prod
                    helm upgrade --install movie-service charts --values=charts/values.yaml \
                      --set image.repository=$DOCKER_ID/movie-service --set image.tag=$DOCKER_TAG \
                      --set service.nodePort=30018 --set imagePullSecrets=null --namespace prod
                    '''
                }
            }
        }
    }
}
