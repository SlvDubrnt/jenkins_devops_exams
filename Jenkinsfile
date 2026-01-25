pipeline {
  environment { // Declaration of environment variables
    DOCKER_ID = "slvdub" // replace this with your docker-id
    //DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
    DOCKER_TAG = "latest" // we will tag our images with the current build in order to increment the value by 1 with each new build
    //BRANCH_NAME = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
    BRANCH_NAME = "${GIT_BRANCH}".replace("refs/heads/", "")
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
        expression { BRANCH_NAME == 'dev' || BRANCH_NAME == 'qa' || BRANCH_NAME == 'staging' }
      }
      steps {
        
        
        //Build()
        // Ajoutez ici les étapes de déploiement spécifiques à votre projet
        script {
          echo "Deploying ${BRANCH_NAME}"
          if ( BRANCH_NAME == 'dev' ) {
            currentBuild.result = 'SUCCESS'
          }   
        }         
      }
    }

    stage('QA Approval') {
      when {
        expression { BRANCH_NAME == 'qa' && checkBranchSuccess('dev') } 
      }
      steps {
        script {
          echo "Deploying to STAGING ${BRANCH_NAME}"
        // Ajoutez ici les étapes de déploiement spécifiques à votre projet
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
        // Ajoutez ici les étapes de déploiement spécifiques à votre projet
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
          
          // Ajoutez ici les étapes de déploiement spécifiques à votre projet
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
  environment {
    KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
  }
  steps {
    script {
    sh '''
    rm -Rf .kube
    mkdir .kube
    ls
    cat $KUBECONFIG > .kube/config
    cp fastapi/values.yaml values.yml
    cat values.yml
    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
    helm upgrade --install app fastapi --values=values.yml --namespace dev
    '''
    }
  }
}


def build() {
  stage('Creation du reseau') {
    steps {
      echo 'Creation du reseau'
      script {
        sh 'docker network create jenkins_cast_movie_network'
      }
    }
  } 
  stage('Run Databases') {
    steps {
      echo 'Creation des bases'
      script {
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
      }
    }
  }
  stage('Build and Run Cast Service') {
    steps {
      echo 'Creation du cast-service'
      script {
          sh 'docker build -t $DOCKER_ID/cast_service:$DOCKER_TAG ./cast-service'
          sh '''
          docker run -d \
            --name cast_service \
            --restart unless-stopped \
            -p 8002:8000 \
            -v ./cast-service/:/app/ \
            -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev \
            --network jenkins_cast_movie_network \
            --link cast_db:cast_db \
            $DOCKER_ID/cast_service:$DOCKER_TAG \
            uvicorn app.main:app --reload --host 0.0.0.0 --port 8000 
          '''
      }
    }
  }
  stage('Build and Run Movie Service') {
    steps {
      echo 'Creation du movie-service'
      script {
          sh 'docker build -t $DOCKER_ID/movie_service:$DOCKER_TAG ./movie-service'
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
            $DOCKER_ID/movie_service:$DOCKER_TAG \
            uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
          '''
      }
    }
  }
  stage('Run Nginx') {
    steps {
      echo 'Creation de nginx'
      script {
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
      }
    }
  }
  stage("Validation de l'application") {
    steps {
      echo "Validation de l'application"
      script {
        sh "docker ps"
        sh "docker image ls" 
        sh 'curl -I http://localhost:9090/api/v1/movies/docs'
        sh 'curl -I http://localhost:9090/api/v1/casts/docs' 
      } 
    }
  }
  stage("Stop container") {
    steps {
      echo "Stop container"
      script {
        sh 'docker container stop cast_service'  
        sh 'docker container stop movie_service '  
        sh 'docker container stop cast_db'  
        sh 'docker container stop movie_db'  
        sh 'docker container stop nginx'  
      } 
    }
  }
  stage('Push des images sur dockerhub') {
    environment {
      DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
    }
    steps {
      script {
        echo 'Push all images'
        sh 'docker login -u $DOCKER_ID -p $DOCKER_PASS'

        def images = ["cast_service", "movie_service"]
        images.each { image ->
          echo "${DOCKER_ID}/${image}:${DOCKER_TAG}"
          sh "docker push ${DOCKER_ID}/${image}:${DOCKER_TAG}"
        }
      }
    } 
  } 
  stage("Remove containers and images") {
    steps {
      echo "Remove containers and images"
      script {
        sh 'docker container rm cast_service'  
        sh 'docker container rm movie_service '  
        sh 'docker container rm cast_db'  
        sh 'docker container rm movie_db'  
        sh 'docker container rm nginx' 
        
        def images = ["cast_service", "movie_service"]
        images.each { image ->
          echo "${DOCKER_ID}/${image}:${DOCKER_TAG}"
          sh "docker image rm -f ${DOCKER_ID}/${image}:${DOCKER_TAG}"
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
    }
  }
}
