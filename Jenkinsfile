pipeline {
  environment { // Declaration of environment variables
    DOCKER_ID = "slvdub" // replace this with your docker-id
    DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
    DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
    //BRANCH_NAME = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
    BRANCH_NAME = "${GIT_BRANCH}".replace("refs/heads/", "")
    KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
  }
  agent any
  stages {
      stage('Debug') {
      steps {
        script {
          echo "BRANCH_NAME: ${BRANCH_NAME}"
        }
      }
    }
    stage('Deploy_dev') {
      when {
        expression { BRANCH_NAME == 'dev'  }
      }
      steps {
        script {
          echo "Building Docker"
          build()
          echo "Deploying ${BRANCH_NAME}"
          delete_all_dev()
          deploy()          
          echo "Controling access from ${BRANCH_NAME}"
          verif_access_app()
          echo "Add data from ${BRANCH_NAME}"
          add_data()
          echo "Controling  ${BRANCH_NAME}"
          display_movie()            
        } 
      }
    }
    
    stage('Deploy') {
      when {
        expression { BRANCH_NAME == 'qa' || BRANCH_NAME == 'staging' }
      }
      steps {
        script {
          echo "Building Docker"
          build()
          echo "Deploying ${BRANCH_NAME}"
          deploy() 
          echo "Controling  ${BRANCH_NAME}"
          verif_access_app()         
          display_movie() 
        }         
      }
    }


    stage('Manual Approval for Master') {
      when {
        expression { BRANCH_NAME == 'master'  }
      }
      steps {
        script {
          echo "Deploying to PROD from ${BRANCH_NAME}"
          input message: "Approve deployment to PROD", ok: "Deploy"
          deploy()          
          echo "Controling PROD from ${BRANCH_NAME}"
          verif_access_app()
        }
      }
    }
  }
}

def verif_access_app(){

  echo '*** Verification acces application'
  sh 'sleep 20'
  sh 'curl -I http://localhost:9090/api/v1/movies/docs'
  sh 'curl -I http://localhost:9090/api/v1/casts/docs' 
}

def add_data(){  
  
  echo '*** Ajout de lignes '
  sh '''
     curl -X 'POST' 'http://localhost:9090/api/v1/casts/' -H 'accept: application/json' -H 'Content-Type: application/json' \
     -d '{ "name": "1", "nationality": "1" }'

     curl -X 'POST' 'http://localhost:9090/api/v1/movies/' -H 'accept: application/json' -H 'Content-Type: application/json' \
     -d '{  "name": "1",  "plot": "1",  "genres": [  "1"  ], "casts_id": [ 1 ] }'

  '''
}

def display_movie(){

  echo '*** affichage des lignes de movie'
  sh '''
     curl -X 'GET' 'http://localhost:9090/api/v1/movies/' -H 'accept: application/json'
  '''
  
}


def delete_all_dev(){
    sh "kubectl delete all --all -n ${BRANCH_NAME}"
    sh "kubectl delete pvc --all -n ${BRANCH_NAME}"
}

def deploy() {
    sh '''
    rm -Rf .kube
    mkdir .kube
    ls
    cat ${KUBECONFIG} > .kube/config
    cp app-movie/values.yaml values.yaml
    cat values.yaml | grep tag
    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yaml
    cat values.yaml | grep tag
    
    helm upgrade --install app-movie ./app-movie --values=values.yaml -n ${BRANCH_NAME}
    '''
}


def build() {
  echo '*** Creation du reseau'
  sh 'docker network create jenkins_cast_movie_network'
  
  echo '*** Creation des bases'
  sh 'docker volume create postgres_data_movie'
  sh '''
  docker run -d --name movie_db \
    --restart unless-stopped \
    -v postgres_data_movie:/var/lib/postgresql/data/ \
    --network jenkins_cast_movie_network \
    -e POSTGRES_USER=movie_db_username \
    -e POSTGRES_PASSWORD=movie_db_password \
    -e POSTGRES_DB=movie_db_dev postgres:12.1-alpine
  '''

  sh 'docker volume create postgres_data_cast'
  sh '''
  docker run -d --name cast_db \
    --restart unless-stopped \
    -v postgres_data_cast:/var/lib/postgresql/data/ \
    --network jenkins_cast_movie_network \
    -e POSTGRES_USER=cast_db_username \
    -e POSTGRES_PASSWORD=cast_db_password \
    -e POSTGRES_DB=cast_db_dev postgres:12.1-alpine
  '''
  
  
  echo '*** Creation du cast-service'
    sh 'docker build -t ${DOCKER_ID}/cast_service:${DOCKER_TAG} ./cast-service'
    sh '''
    docker run -d \
      --name cast_service \
      --restart unless-stopped \
      -p 8002:8000 \
      -v ./cast-service/:/app/ \
      -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev \
      --network jenkins_cast_movie_network \
      --link cast_db:cast_db \
      ${DOCKER_ID}/cast_service:${DOCKER_TAG} \
      uvicorn app.main:app --reload --host 0.0.0.0 --port 8000 
    '''
  
  echo '***Creation du movie-service'
    sh 'docker build -t ${DOCKER_ID}/movie_service:${DOCKER_TAG} ./movie-service'
    sh '''
    docker run -d \
      --name movie_service \
      --restart unless-stopped \
      -p 8001:8000 \
      -v ./movie-service/:/app/ \
      -e DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie_db/movie_db_dev \
      -e CAST_SERVICE_HOST_URL=http://cast_service:8000/api/v1/casts/ \
      --network jenkins_cast_movie_network \
      --link movie_db:movie_db \
      --link cast_service:cast_service \
      ${DOCKER_ID}/movie_service:${DOCKER_TAG} \
      uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
    '''
  
  echo '*** Creation de nginx'
    sh '''
    docker run -d \
      --name nginx \
      --restart unless-stopped \
      -p 9090:9090 \
      -v ./nginx_config.conf:/etc/nginx/conf.d/default.conf \
      --network jenkins_cast_movie_network \
      --link cast_service:cast_service \
      --link movie_service:movie_service \
      nginx:latest
    '''
  
  echo "*** Validation de l'application"
  sh "docker ps"
  sh "docker image ls" 
  sh 'curl -I http://localhost:9090/api/v1/movies/docs'
  sh 'curl -I http://localhost:9090/api/v1/casts/docs' 
  
  echo "*** Stop container"
  sh 'docker container stop cast_service'  
  sh 'docker container stop movie_service '  
  sh 'docker container stop cast_db'  
  sh 'docker container stop movie_db'  
  sh 'docker container stop nginx'  
  
  echo 'Push all images'

  def images = ["cast_service", "movie_service"]
  sh 'docker login -u $DOCKER_ID -p $DOCKER_PASS'
  
  images.each { image ->
    echo "${DOCKER_ID}/${image}:${DOCKER_TAG}"
    sh "docker push ${DOCKER_ID}/${image}:${DOCKER_TAG}"
  }
  
  sh "docker logout" // Logout from Docker Hub
  
  echo "*** Remove containers and images"
  sh 'docker container rm cast_service'  
  sh 'docker container rm movie_service '  
  sh 'docker container rm cast_db'  
  sh 'docker container rm movie_db'  
  sh 'docker container rm nginx' 
  
  images.each { image ->
    echo "${DOCKER_ID}/${image}:${DOCKER_TAG}"
    sh "docker image rm  ${DOCKER_ID}/${image}:${DOCKER_TAG}"
  }
  sh 'docker image rm postgres:12.1-alpine -f'
  sh 'docker image rm nginx -f'
  sh 'docker network rm jenkins_cast_movie_network'
 
  sh 'docker volume rm postgres_data_cast'
  sh 'docker volume rm postgres_data_movie'
 
  sh 'docker image ls'
}

