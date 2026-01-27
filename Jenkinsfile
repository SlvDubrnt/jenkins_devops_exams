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
          // Ajoutez d'autres débogages si nécessaire
        }
      }
    }
    stage('Deploy') {
      when {
        expression { BRANCH_NAME == 'dev'  }
      }
      steps {
        script {
          echo "Building Docker"
          build()
          echo "Deploying ${BRANCH_NAME}"
          deploy(BRANCH_NAME)          
        }         
      }
    }

    stage('QA Approval') {
      when {
        expression { BRANCH_NAME == 'qa' && checkBranchSuccess('dev') } 
      }
      steps {
        script {
          echo "Deploying to QA ${BRANCH_NAME}"
          echo "Deploying ${BRANCH_NAME}"
          deploy(BRANCH_NAME)          
        }
      }
    }

    stage('Staging Approval') {
      when {
        expression { BRANCH_NAME == 'staging' && checkBranchSuccess('qa') }
      }
      steps {
       script {
          echo "Deploying to STAGING ${BRANCH_NAME}"
          echo "Deploying ${BRANCH_NAME}"
          deploy(BRANCH_NAME)          
        }
      }
    }

    stage('Manual Approval for Master') {
      when {
        expression { BRANCH_NAME == 'master'   && checkBranchSuccess('staging')}
      }
      steps {
        script {
          echo "Deploying to PROD from ${BRANCH_NAME}"
          input message: "Approve deployment to PROD", ok: "Deploy"
          deploy(BRANCH_NAME)          
          
        }
      }
    }
  }
  post {
    always {
      script {
        sh "docker logout" // Logout from Docker Hub
      } 
    }
  } 
}

// Fonction pour vérifier si une branche spécifique a réussi
def checkBranchSuccess(String branch) {
  def lastBuild = currentBuild.raw.builds.find { it.getEnvironment("BRANCH_NAME") == previousBranch }
    if (lastBuild && lastBuild.result == 'SUCCESS') {
      return true
    } else {
      return false
    }
}
def branchSuccess(String branch) {
    // Implémentez votre logique de vérification ici
  currentBuild.rawBuild.getPreviousBuilds().find { previousBuild ->
    previousBuild.displayName == branch && previousBuild.result == 'SUCCESS'
  } != null
}

def deploy(String branch) {
    sh '''
    rm -Rf .kube
    mkdir .kube
    ls
    cat ${KUBECONFIG} > .kube/config
    cp app-movie/values.yaml values.yaml
    cat values.yaml
    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yaml
    cat values.yaml
    helm upgrade --install app-movie app-movie --values=values.yaml -- branch
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
  sh 'docker login -u $DOCKER_ID -p $DOCKER_PASS'

  def images = ["cast_service", "movie_service"]
  
  images.each { image ->
    echo "${DOCKER_ID}/${image}:${DOCKER_TAG}"
    sh "docker push ${DOCKER_ID}/${image}:${DOCKER_TAG}"
  }
  
  echo "*** Remove containers and images"
  sh 'docker container rm cast_service'  
  sh 'docker container rm movie_service '  
  sh 'docker container rm cast_db'  
  sh 'docker container rm movie_db'  
  sh 'docker container rm nginx' 
  
  images.each { image ->
    echo "${DOCKER_ID}/${image}:${DOCKER_TAG}"
    sh "docker push ${DOCKER_ID}/${image}:${DOCKER_TAG}"
  }
  sh 'docker image rm postgres:12.1-alpine -f'
  sh 'docker image rm nginx -f'
  sh 'docker network rm jenkins_cast_movie_network'
 
  sh 'docker volume rm postgres_data_cast'
  sh 'docker volume rm postgres_data_movie'
 
  sh 'docker image ls'
  sh 'docker network ls'
  sh 'docker container ls'
  sh 'docker volume ls'
}
