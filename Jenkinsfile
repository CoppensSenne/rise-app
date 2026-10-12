properties([pipelineTriggers([pollSCM("* * * * *")])])

node {
    stage('Checkout') {
        checkout scm
    }
    stage('Preparation') {
        sh 'docker network create rise-net || true'
        sh 'docker network connect rise-net jenkins_server || true'
        sh 'docker stop riseapp || true'
        sh 'docker rm riseapp || true'
    }
    stage('Database') {
        sh '''
        if [ -z "$(docker ps -aq -f name=^risedb$)" ]; then
          docker run -d --name risedb --network rise-net \
            -e MARIADB_ROOT_PASSWORD=sekrit \
            -e MARIADB_DATABASE=rise_db \
            -e MARIADB_USER=rise_usr \
            -e MARIADB_PASSWORD=letmeinplz \
            -v rise-db-data:/var/lib/mysql \
            mariadb:11.4
        else
          docker start risedb || true
        fi
        for i in $(seq 1 30); do
          docker exec risedb healthcheck.sh --connect --innodb_initialized && break
          sleep 2
        done
        '''
    }
    stage('Build') {
        sh 'docker build -t riseapp:latest .'
    }
    stage('Deploy') {
        sh '''
        docker run -d --name riseapp --network rise-net -p 8082:8080 \
          -e "ConnectionStrings__DatabaseConnection=Server=risedb;Port=3306;Database=rise_db;User=rise_usr;Password=letmeinplz;" \
          -e ASPNETCORE_ENVIRONMENT=Development \
          riseapp:latest
        '''
    }
    stage('Test') {
        sh '''
        for i in $(seq 1 30); do
          curl -sf http://riseapp:8080/ > /dev/null && echo "App OK" && exit 0
          sleep 3
        done
        docker logs riseapp
        exit 1
        '''
    }
}
