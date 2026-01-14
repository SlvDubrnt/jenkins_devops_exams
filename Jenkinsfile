pipeline {
  environment { // Declaration of environment variables
    DOCKER_ID = "slvdub" // replace this with your docker-id
    DOCKER_TAG = "latest" // we will tag our images with the current build in order to increment the value by 1 with each new build
  }
  agent any
  stages {
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
            sh 'docker build -t cast_service ./cast-service'
            sh '''
            docker run -d \
              --name cast_service \
              --restart unless-stopped \
              -p 8002:8000 \
              -v ./cast-service/:/app/ \
              -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev \
              --network jenkins_cast_movie_network \
              --link cast_db:cast_db \
              cast_service uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
            '''
        }
      }
    }
    stage('Build and Run Movie Service') {
      steps {
        echo 'Creation du movie-service'
        script {
            sh 'docker build -t movie_service ./movie-service'
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
              movie_service uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
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
          sh 'docker image ls' 
          sh 'docker ps'  // To verify all containers are running
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
          sh 'DOCKER_IMAGE = "cast_service"'
          echo ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
          sh 'docker push ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}'
          sh '''          
          DOCKER_IMAGE = "movie_service" 
          echo ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
          docker push ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
          '''
          sh '''          
          DOCKER_IMAGE = "postgres:12.1-alpine" 
          echo ${DOCKER_ID/DOCKER_IMAGE}
          docker push ${DOCKER_ID/DOCKER_IMAGE}
          '''
          sh '''          
          DOCKER_IMAGE = "nginx" 
          echo ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
          docker push ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
          '''
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
