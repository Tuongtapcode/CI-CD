Bước 1: Cài đặt Jenkins với Docker
bat '''
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
'''
