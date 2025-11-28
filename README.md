## MEAN CRUD Application – Dockerized Deployment with CI/CD (AWS EC2 + Docker + Nginx)

This project is a fully containerized MEAN Stack CRUD Application deployed on AWS EC2 using Docker, Docker Compose, Nginx reverse proxy, MongoDB, and automated via GitHub Actions CI/CD.

## Project Structure

```bash

crud-dd-task-mean-app/
│
├── backend/                         # Node.js + Express API
│   ├── Dockerfile
│   └── src/...
│
├── frontend/                        # Angular Application
│   ├── Dockerfile
│   ├── nginx.conf
│   └── src/...
│
├── docker-compose.yml               # Multi-service deployment
└── README.md

```

## 1. Docker Setup

- Install Docker & Docker Compose plugin

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu
```

- 1.1 Backend Dockerfile

```bash

FROM node:18-alpine
WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install

COPY . .
ENV PORT=3000
EXPOSE 3000

CMD ["npm", "start"]

```

- 1.2 Frontend Dockerfile

```bash 
# Stage 1: Build Angular App
FROM node:18-alpine AS build
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build -- --configuration=production

# Stage 2: Serve with Nginx
FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy Angular build output (folder name must match dist output)
COPY --from=build /app/dist/angular-15-crud/ /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

```

- 1.3 Nginx Reverse Proxy Configuration

```bash

server {
    listen 80;

    root /usr/share/nginx/html/angular-15-crud;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

```

## 2. Build & push Docker images

- Log in to Docker Hub

```bash

docker login

```
- Build backend image

```bash

docker build -t <your-dockerhub-username>/mean-backend:latest ./backend
docker push <your-dockerhub-username>/mean-backend:latest

```
- Build frontend image

```bash

docker build -t <your-dockerhub-username>/mean-frontend:latest ./frontend
docker push <your-dockerhub-username>/mean-frontend:latest

```
![Screenshot](https://drive.google.com/uc?export=view&id=16_NSPAhnwMRIBi8TL_R57-Um7WFdkx7V)

[My Docker Hub Profile](https://hub.docker.com/u/hemanth173)


## 3. Docker Compose Configuration

```bash

version: "3.8"

services:
  mongo:
    image: mongo:7
    container_name: mongo
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: rootpassword
    volumes:
      - mongo_data:/data/db
    networks:
      - mean-net

  backend:
    image: hemanth173/mean-backend:latest
    container_name: backend
    restart: always
    environment:
      MONGO_URI: "mongodb://mongo:27017/meanapp"
      PORT: 3000
    depends_on:
      - mongo
    networks:
      - mean-net

  frontend:
    image: hemanth173/mean-frontend:latest
    container_name: frontend
    restart: always
    depends_on:
      - backend
    ports:
      - "80:80"
    networks:
      - mean-net
    volumes:
      - ./frontend/nginx.conf:/etc/nginx/conf.d/default.conf:ro

networks:
  mean-net:

volumes:
  mongo_data:

```
- Test locally

Security Group: allow HTTP (80) and SSH (22)

```bash

docker compose up -d

open http://localhost

```

![Screenshot](https://drive.google.com/uc?export=view&id=1x-VRUS9Vk2WMB79Wxh5Hs8XZzvy6e6JZ)


## Set up CI/CD with GitHub Actions

- Add secrets in GitHub

Go to your GitHub repo → Settings → Secrets and variables → Actions → New repository secret:

Add these:

DOCKERHUB_USERNAME → your Docker Hub username

DOCKERHUB_TOKEN → Docker Hub access token

SSH_HOST → your EC2 public IP

SSH_USER → ubuntu

SSH_PRIVATE_KEY → contents of your .pem file

 SSH_PORT → 22


- Create workflow file

```bash

name: CI/CD - MEAN App

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build & push backend image
        uses: docker/build-push-action@v6
        with:
          context: ./backend
          file: ./backend/Dockerfile
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/mean-backend:latest

      - name: Build & push frontend image
        uses: docker/build-push-action@v6
        with:
          context: ./frontend
          file: ./frontend/Dockerfile
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/mean-frontend:latest

      - name: SSH into VM and deploy
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT || 22 }}
          script: |
            cd /home/${{ secrets.SSH_USER }}/<your-repo>
            git pull origin main
            docker compose pull
            docker compose up -d

```

- Commit and push workflow

```bash

git add .
git commit -m "Add Dockerfiles, compose, and CI/CD workflow"
git push origin main

```
![Screenshot](https://drive.google.com/uc?export=view&id=14I2PVMSj2euK1D_MLuHgEsErqfL5gv30)


![Screenshot](https://drive.google.com/uc?export=view&id=1zLZumSIp_qt_YaVtLGXiCXAS0LqvWl4S)


Go to GitHub → Actions tab → you should see the pipeline running:

Build backend & frontend images

Push to Docker Hub

SSH to EC2 and run docker compose pull & docker compose up -d

After it finishes, open again:
http://<EC2-PUBLIC-IP>/ and confirm it works.

## Out Put

- Home

![Screenshot](https://drive.google.com/uc?export=view&id=1PfugZ0FV2H2-GltPj9Dqyg4P_2VdZds8)

- Tutorials

![Screenshot](https://drive.google.com/uc?export=view&id=1wtP0JpXPEQ93Raw9rEYKmpHIgezD5YVV)

- Add

![Screenshot](https://drive.google.com/uc?export=view&id=1fgvuHHmEE4bF7oOTCnRNSMfwQIdR65_1)


