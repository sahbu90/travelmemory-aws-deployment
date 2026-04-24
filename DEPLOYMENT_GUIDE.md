````markdown
# TravelMemory AWS Deployment Guide

This document explains how I deployed the TravelMemory MERN application on AWS using EC2, Nginx, PM2, Application Load Balancer, MongoDB Atlas, and CloudFront.

## Architecture

User → CloudFront HTTPS → Application Load Balancer → EC2 Instances → MongoDB Atlas

Setup used:

- 2 frontend EC2 instances
- 2 backend EC2 instances
- MongoDB Atlas database
- AWS Application Load Balancer
- AWS CloudFront for HTTPS

## Backend Deployment

The backend was deployed on two EC2 instances.

First, I installed the required packages on the backend servers:

```bash
sudo apt update
sudo apt install -y nginx git curl
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
````

Then I cloned the project and installed backend dependencies:

```bash
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
cd TravelMemory/backend
npm install
```

I created the backend `.env` file:

```env
MONGO_URI=your_mongodb_connection_string
PORT=3000
```

After that, I tested the backend locally:

```bash
node index.js
curl http://localhost:3000/hello
```

The expected response was:

```text
Hello World!
```

Then I used PM2 to keep the backend running:

```bash
sudo npm install -g pm2
pm2 start index.js --name travelmemory-backend --update-env
pm2 save
```

I configured Nginx as a reverse proxy for the backend:

```nginx
server {
    listen 80;
    server_name _;

    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Then I enabled the Nginx configuration:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo rm -f /etc/nginx/sites-available/default
sudo ln -sf /etc/nginx/sites-available/travelmemory-backend /etc/nginx/sites-enabled/travelmemory-backend
sudo nginx -t
sudo systemctl restart nginx
```

Backend verification:

```bash
curl http://localhost/api/hello
curl http://localhost/api/trip
```

## Frontend Deployment

The frontend was deployed on two EC2 instances.

First, I installed required packages:

```bash
sudo apt update
sudo apt install -y nginx git curl rsync
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Then I cloned the project and installed frontend dependencies:

```bash
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
cd TravelMemory/frontend
npm install
```

I configured the frontend API URL using `.env.production`:

```env
REACT_APP_BACKEND_URL=/api
```

This helped avoid CORS and mixed content issues because the frontend and backend were accessed through the same domain.

Then I created the production build:

```bash
rm -rf build
npm run build
```

I copied the build output to the Nginx web directory:

```bash
sudo mkdir -p /var/www/travelmemory
sudo rsync -av --delete build/ /var/www/travelmemory/
```

I configured Nginx for the React frontend:

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/travelmemory;
    index index.html;

    location /static/ {
        try_files $uri =404;
    }

    location = /favicon.ico {
        try_files /favicon.ico =404;
    }

    location = /logo192.png {
        try_files /logo192.png =404;
    }

    location = /logo512.png {
        try_files /logo512.png =404;
    }

    location = /manifest.json {
        try_files /manifest.json =404;
    }

    location = /robots.txt {
        try_files /robots.txt =404;
    }

    location / {
        try_files $uri /index.html;
    }
}
```

Then I enabled and restarted Nginx:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo rm -f /etc/nginx/sites-available/default
sudo ln -sf /etc/nginx/sites-available/travelmemory-frontend /etc/nginx/sites-enabled/travelmemory-frontend
sudo nginx -t
sudo systemctl restart nginx
```

Frontend verification:

```bash
curl -I http://localhost
curl -I http://localhost/static/js/main.js
```

## Load Balancer Setup

I created an AWS Application Load Balancer to distribute traffic.

Two target groups were created:

* `tg-frontend`
* `tg-backend`

Frontend target group:

* Protocol: HTTP
* Port: 80
* Health check path: `/`
* Registered targets:

  * frontend-1
  * frontend-2

Backend target group:

* Protocol: HTTP
* Port: 80
* Health check path: `/api/hello`
* Registered targets:

  * backend-1
  * backend-2

ALB listener rules:

```text
/api/*  → backend target group
/       → frontend target group
```

This allowed frontend and backend traffic to be routed properly through the same load balancer.

## CloudFront HTTPS Setup

Since I did not have a custom paid domain, I used CloudFront default domain for HTTPS.

CloudFront configuration:

* Origin: Application Load Balancer DNS
* Origin protocol policy: HTTP only
* Viewer protocol: HTTPS
* Default root object: index.html
* Allowed methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE

After changing CloudFront settings, I created invalidation:

```text
/*
```

Final HTTPS URL:

```text
https://d20ahthcxav8kv.cloudfront.net
```

## Issues Faced and Fixes

### CORS Issue

The frontend and backend were initially on different origins.

Fix:

```env
REACT_APP_BACKEND_URL=/api
```

### Mixed Content Error

The HTTPS frontend was calling HTTP backend URL.

Fix:

* Used relative API path `/api`
* Routed requests through CloudFront and ALB

### Blank Screen Issue

The frontend was showing a blank screen because different frontend servers had different build hashes.

Fix:

* Rebuilt frontend on both servers
* Ensured both frontend instances served the same build
* Corrected Nginx static file configuration

### Nginx 500 Error

Nginx had issues serving React build from the home directory.

Fix:

* Moved frontend build to `/var/www/travelmemory`

### CloudFront 504 Error

CloudFront was trying to connect to ALB using HTTPS, but ALB was configured only for HTTP.

Fix:

* Changed CloudFront origin protocol policy to HTTP only

### Static File Issue

CloudFront was receiving HTML instead of JavaScript/CSS files.

Fix:

* Corrected Nginx static route handling
* Cleared CloudFront cache using invalidation

##

The application was successfully deployed with:

* React frontend running on multiple EC2 instances
* Node.js backend running on multiple EC2 instances
* MongoDB Atlas connected
* Nginx reverse proxy configured
* PM2 process manager configured
* Load balancing using AWS ALB
* HTTPS enabled using CloudFront
* Add Experience feature working
* Trip listing working

## Live URL

```text
https://d20ahthcxav8kv.cloudfront.net
```

## Original Project Repository

```text
https://github.com/UnpredictablePrashant/TravelMemory
```

## What This Project Demonstrates

This deployment helped me understand how a full-stack MERN application can be deployed on AWS using a scalable architecture. I also handled real deployment issues like CORS, mixed content, Nginx static file problems, build mismatch, and CloudFront caching.

```
```
