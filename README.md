Bước 1: Cài đặt Jenkins với Docker
```bash
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    environment:
      - JENKINS_OPTS=--httpPort=8080
    restart: unless-stopped

  # Database cho testing
  mysql:
    image: mysql:8.0
    container_name: mysql-jenkins
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: english_vocab_test
    ports:
      - "3307:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    restart: unless-stopped

  # Redis cho caching
  redis:
    image: redis:7.0
    container_name: redis-jenkins
    ports:
      - "6380:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

  # Docker Registry (optional - để lưu images)
  registry:
    image: registry:2
    container_name: docker-registry
    ports:
      - "5000:5000"
    volumes:
      - registry_data:/var/lib/registry
    restart: unless-stopped

volumes:
  jenkins_home:
  mysql_data:
  redis_data:
  registry_data:
```
Bước 2: Tạo Dockerfile cho Backend

```bash
# Multi-stage build
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app
COPY pom.xml .
COPY src ./src

# Build application
RUN mvn clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Create user for security
RUN addgroup -g 1000 appgroup && adduser -u 1000 -G appgroup -s /bin/sh -D appuser

# Copy jar file from builder stage
COPY --from=builder /app/target/*.jar app.jar

# Change ownership
RUN chown appuser:appgroup app.jar

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

Bước 3: Tạo Jenkinsfile cho Pipeline
```bash
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'localhost:5000'
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/english-vocab-backend"
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/english-vocab-frontend"
        MYSQL_ROOT_PASSWORD = 'root'
        MYSQL_DATABASE = 'english_vocab_test'
    }
    
    tools {
        maven 'Maven-3.9'
        nodejs 'NodeJS-18'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }
        
        stage('Build & Test Backend') {
            steps {
                script {
                    dir('backend') {
                        // Start test databases
                        sh '''
                            docker-compose -f ../docker-compose.test.yml up -d mysql redis
                            sleep 30
                        '''
                        
                        try {
                            // Run tests
                            sh '''
                                export SPRING_PROFILES_ACTIVE=test
                                export SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3307/english_vocab_test
                                export SPRING_DATASOURCE_USERNAME=root
                                export SPRING_DATASOURCE_PASSWORD=root
                                export SPRING_REDIS_HOST=localhost
                                export SPRING_REDIS_PORT=6380
                                mvn clean test
                            '''
                            
                            // Build application
                            sh 'mvn clean package -DskipTests'
                            
                        } finally {
                            // Cleanup test databases
                            sh 'docker-compose -f ../docker-compose.test.yml down'
                        }
                    }
                }
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'backend/target/surefire-reports/*.xml'
                    publishCoverage adapters: [jacocoAdapter('backend/target/site/jacoco/jacoco.xml')]
                }
            }
        }
        
        stage('Build & Test Frontend') {
            when {
                changeset "frontend/**"
            }
            steps {
                dir('frontend') {
                    sh '''
                        npm ci
                        npm run test -- --coverage --watchAll=false
                        npm run build
                    '''
                }
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'frontend/test-results.xml'
                }
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('Backend Security') {
                    steps {
                        dir('backend') {
                            sh '''
                                mvn org.owasp:dependency-check-maven:check
                            '''
                        }
                    }
                }
                stage('Frontend Security') {
                    when {
                        changeset "frontend/**"
                    }
                    steps {
                        dir('frontend') {
                            sh '''
                                npm audit --audit-level high
                                npx safety-cli scan
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Backend Image') {
                    steps {
                        dir('backend') {
                            script {
                                def backendImage = docker.build("${BACKEND_IMAGE}:${env.GIT_COMMIT_SHORT}")
                                docker.withRegistry("http://${DOCKER_REGISTRY}") {
                                    backendImage.push()
                                    backendImage.push('latest')
                                }
                            }
                        }
                    }
                }
                stage('Frontend Image') {
                    when {
                        changeset "frontend/**"
                    }
                    steps {
                        dir('frontend') {
                            script {
                                def frontendImage = docker.build("${FRONTEND_IMAGE}:${env.GIT_COMMIT_SHORT}")
                                docker.withRegistry("http://${DOCKER_REGISTRY}") {
                                    frontendImage.push()
                                    frontendImage.push('latest')
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    sh '''
                        docker-compose -f docker-compose.staging.yml down
                        docker-compose -f docker-compose.staging.yml up -d
                    '''
                }
            }
        }
        
        stage('Integration Tests') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    sh '''
                        sleep 60  # Wait for services to start
                        cd integration-tests
                        npm ci
                        npm run test:staging
                    '''
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    input message: 'Deploy to production?', ok: 'Deploy'
                    
                    sh '''
                        docker-compose -f docker-compose.prod.yml pull
                        docker-compose -f docker-compose.prod.yml up -d --no-deps backend
                        sleep 30
                        # Health check
                        curl -f http://localhost:8080/actuator/health
                    '''
                }
            }
        }
        
        stage('Mobile App Build') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    changeset "mobile/**"
                }
            }
            parallel {
                stage('Android Build') {
                    agent {
                        label 'android'  // Cần agent có Android SDK
                    }
                    steps {
                        dir('mobile') {
                            sh '''
                                npm ci
                                cd android
                                ./gradlew assembleRelease
                            '''
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'mobile/android/app/build/outputs/apk/release/*.apk'
                        }
                    }
                }
                stage('iOS Build') {
                    agent {
                        label 'macos'  // Cần macOS agent cho iOS
                    }
                    when {
                        branch 'main'
                    }
                    steps {
                        dir('mobile') {
                            sh '''
                                npm ci
                                cd ios
                                xcodebuild -workspace EnglishVocabApp.xcworkspace \
                                          -scheme EnglishVocabApp \
                                          -configuration Release \
                                          archive -archivePath EnglishVocabApp.xcarchive
                            '''
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ Build SUCCESS: ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ Build FAILED: ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
    }
}
```
Bước 4: Tạo Docker Compose cho Testing
```bash
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql-test
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: english_vocab_test
      MYSQL_USER: test_user
      MYSQL_PASSWORD: test_password
    ports:
      - "3307:3306"
    volumes:
      - mysql_test_data:/var/lib/mysql
      - ./backend/src/test/resources/sql:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10
    networks:
      - test-network

  redis:
    image: redis:7.0-alpine
    container_name: redis-test
    ports:
      - "6380:6379"
    volumes:
      - redis_test_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      timeout: 10s
      retries: 5
    networks:
      - test-network

volumes:
  mysql_test_data:
  redis_test_data:

networks:
  test-network:
    driver: bridge
```
Bước 5: Script khởi động Jenkins
```bash
#!/bin/bash

echo "🚀 Setting up Jenkins CI/CD for English Vocabulary System"

# Tạo thư mục cho Jenkins
mkdir -p jenkins_home
sudo chown -R 1000:1000 jenkins_home

# Start Jenkins và services
echo "📦 Starting Jenkins services..."
docker-compose up -d

echo "⏳ Waiting for Jenkins to start..."
sleep 60

# Get initial admin password
echo "🔑 Jenkins Admin Password:"
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Install Docker in Jenkins container
echo "🐳 Installing Docker in Jenkins container..."
docker exec -u root jenkins bash -c "
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    echo 'deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian bullseye stable' | tee /etc/apt/sources.list.d/docker.list > /dev/null
    apt-get update
    apt-get install -y docker-ce-cli
"

# Restart Jenkins to apply changes
echo "🔄 Restarting Jenkins..."
docker restart jenkins
sleep 30

echo "✅ Jenkins setup completed!"
echo ""
echo "📋 Next steps:"
echo "1. Open http://localhost:8080"
echo "2. Use the admin password above to login"
echo "3. Install suggested plugins + additional plugins:"
echo "   - Docker Pipeline"
echo "   - Pipeline Stage View"
echo "   - Blue Ocean"
echo "   - NodeJS"
echo "   - Maven Integration"
echo "   - Coverage"
echo "   - Slack Notification (optional)"
echo ""
echo "4. Configure Global Tools:"
echo "   - Java: OpenJDK 17"
echo "   - Maven: 3.9.x"
echo "   - NodeJS: 18.x"
echo ""
echo "5. Create Pipeline Job:"
echo "   - New Item -> Pipeline"
echo "   - Pipeline from SCM -> Git"
echo "   - Repository URL: https://github.com/Tuongtapcode/EnglishVocabSystem"
echo "   - Script Path: Jenkinsfile"

# Create .gitignore for Jenkins files
cat > .gitignore << EOF
# Jenkins
jenkins_home/
!jenkins_home/.gitkeep

# Docker volumes
mysql_data/
redis_data/
registry_data/

# Logs
*.log
logs/

# Test reports
target/
node_modules/
coverage/
test-results/
EOF

echo ""
echo "🎉 Setup script completed successfully!"
echo "💡 Don't forget to:"
echo "   - Add webhooks in GitHub: http://your-server:8080/github-webhook/"
echo "   - Configure Docker registry credentials if using external registry"
echo "   - Set up Slack notifications if needed"
```

