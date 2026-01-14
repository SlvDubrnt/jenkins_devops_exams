pipeline {
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
  }
  post {
    always {
      script {
        sh 'docker ps'  // To verify all containers are running
        sh 'curl -I http://localhost/api/v1/movies/docsi'
        sh 'curl -I http://localhost/api/v1/casts/docs' 
      }
    }
  }
}
