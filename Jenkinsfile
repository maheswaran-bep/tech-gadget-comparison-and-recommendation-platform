pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        COMPOSE_PROJECT_NAME = 'tech-gadget-comparison-and-recommendation-platform'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git rev-parse --short HEAD'
            }
        }

        stage('Verify Docker') {
            steps {
                sh '''
                    set -eu
                    docker version
                    docker compose version
                '''
            }
        }

        stage('Prepare Environment') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'gadget-db-password',
                        variable: 'DB_PASSWORD'
                    ),
                    string(
                        credentialsId: 'gadget-secret-key',
                        variable: 'SECRET_KEY'
                    )
                ]) {
                    sh '''
                        set +x
                        umask 077

                        cat > .env <<EOF_ENV
DB_PASSWORD=${DB_PASSWORD}
SECRET_KEY=${SECRET_KEY}
DATABASE_URL=mysql+pymysql://root:${DB_PASSWORD}@mysql:3306/tech_gadgets
FRONTEND_URL=http://localhost:3015
EOF_ENV

                        chmod 600 .env
                        test -s .env
                    '''
                }
            }
        }

        stage('Validate Compose') {
            steps {
                sh '''
                    set -eu
                    docker compose config -q
                    echo "Docker Compose configuration is valid."
                '''
            }
        }

        stage('Build Images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Backend Validation') {
            steps {
                sh '''
                    set -eu
                    docker compose run --rm --no-deps backend \
                      python -c "from app.main import app; print(app.title)"
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -eu
                    docker compose up -d --remove-orphans
                    docker compose ps
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "Waiting for the backend health check..."

                    healthy=0
                    for attempt in $(seq 1 30); do
                        if docker compose exec -T backend python -c \
                          "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=5).read().decode())"
                        then
                            healthy=1
                            break
                        fi
                        sleep 2
                    done

                    if [ "$healthy" -ne 1 ]; then
                        echo "Backend health check failed."
                        docker compose logs --tail 100 backend
                        exit 1
                    fi

                    echo "Checking the frontend..."
                    docker compose exec -T frontend \
                      wget -qO- http://127.0.0.1/ > /dev/null

                    echo "Checking the frontend API proxy..."
                    docker compose exec -T frontend \
                      wget -qO- http://127.0.0.1/api/gadgets/test

                    echo
                    docker compose ps
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment successful. Frontend: http://localhost:3015'
        }

        failure {
            echo 'Pipeline failed. Review the failed stage and its console output.'
        }

        always {
            sh 'rm -f .env'
        }
    }
}
