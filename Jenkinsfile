pipeline {
    agent any
    stage('Creation du reseau') {
            steps {
                echo 'Creation du reseau'
                script {
                    sh 'docker network create jenkins_cast_movie_network'
                }
    }    
    stage('Run Databases') {
            steps {
                echo 'Creation des bases'
                script {
                    sh 'docker volume create postgres_data_movie'
                    sh 'docker run -d --name movie_db --restart unless-stopped -v postgres_data_movie:/var/lib/postgresql/data/ --network jenkins_cast_movie_network -e POSTGRES_USER=movie_db_username -e POSTGRES_PASSWORD=movie_db_password -e POSTGRES_DB=movie_db_dev postgres:12.1-alpine'

                    sh 'docker volume create postgres_data_cast'
                    sh 'docker run -d --name cast_db --restart unless-stopped -v postgres_data_cast:/var/lib/postgresql/data/ --network jenkins_cast_movie_network -e POSTGRES_USER=cast_db_username -e POSTGRES_PASSWORD=cast_db_password -e POSTGRES_DB=cast_db_dev postgres:12.1-alpine'
                }
            }
    }
}
