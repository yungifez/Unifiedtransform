version: '3'

#Docker Networks
networks:
  app-network:
    driver: bridge

#Volumes
volumes:
  dbdata:
    driver: local

services:

  #Nginx Service
  nginx:
    build:
      context: .
      dockerfile: ./docker/nginx.dockerfile
    container_name: nginx
    ports:
      - "${DOCKER_WEBSERVER_HOST}:80"
    volumes:
      - .:/var/www/html
    depends_on:
      - php
      - db
    networks:
      - app-network

  #MySQL Service
  db:
    image: mysql:latest
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    volumes:
      - dbdata:/var/lib/mysql
    networks:
      - app-network

  #PHP Service
  php:
    build:
      context: .
      dockerfile: ./docker/php.dockerfile
    container_name: php
    volumes:
      - .:/var/www/html
    ports:
      - "9000:9000"
    networks:
      - app-network

  #Composer Service
  composer:
    build:
      context: .
      dockerfile: ./docker/composer.dockerfile
    container_name: composer
    volumes:
      - .:/var/www/html
    working_dir: /var/www/html
    depends_on:
      - php
    user: laravel
    networks:
      - app-network
    entrypoint: [ 'composer', '--ignore-platform-reqs' ]

  #Artisan Service
  artisan:
    build:
      context: .
      dockerfile: ./docker/php.dockerfile
    container_name: artisan
    volumes:
      - .:/var/www/html
    depends_on:
      - db
    working_dir: /var/www/html
    user: laravel
    entrypoint: [ 'php', '/var/www/html/artisan' ]
    networks:
      - app-network

  #PHPMyAdmin Service
  phpmyadmin:
    depends_on:
      - db
    image: phpmyadmin/phpmyadmin:latest
    container_name: phpmyadmin
    restart: always
    ports:
      - '${DOCKER_PHPMYADMIN_HOST}:80'
    environment:
      PMA_HOST: db
      UPLOAD_LIMIT: 3000000000
    networks:
      - app-network