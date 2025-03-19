## demo app - for docker-compose with load balancer 


### With Docker

## Project Structure

.
├── docker-compose.yml
├── nginx
│   └── nginx.conf
├── app
│   └── main.py
├── Dockerfile

## Step 1: FastAPI App Code (app/main.py)

# This app identifies which container instance is responding. app/main.py

from fastapi import FastAPI
import socket

app = FastAPI()

@app.get("/")
async def read_root():
    hostname = socket.gethostname()
    return {"message": f"Hello from {hostname}!"}

## Step 2: Dockerfile (Dockerfile)
# A single Dockerfile to build and scale multiple app instances. Dockerfile

FROM python:3.10-slim
WORKDIR /app
COPY ./app /app
RUN pip install fastapi uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5000"]

## Step 3: Nginx Configuration with Load Balancing, round robin Sessions, and Security

This setup includes:

✅ Sticky sessions for consistent user experience
✅ Rate limiting to prevent abuse
✅ Optimized timeout settings for stability
✅ Detailed logging for better observability

nginx.conf

worker_processes auto;

events {
    worker_connections 1024;
}

http {
    # Rate limiting zone - allows 10 requests per second with a burst of 20
    limit_req_zone $binary_remote_addr zone=rate_limit_zone:10m rate=10r/s;

    upstream backend {
        ip_hash;  # Sticky sessions (based on client IP)
        server app:5000 max_fails=3 fail_timeout=10s;
        keepalive 32;  # Maintain up to 32 persistent connections
    }

    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'request_time=$request_time';

    server {
        listen 80;

        access_log /var/log/nginx/access.log main;

        location / {
            limit_req zone=rate_limit_zone burst=20 nodelay;  # Rate limit rule
            proxy_pass http://backend;
            
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            # Optimized timeout settings
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
        }
    }
}

## Step 4: Docker Compose Configuration (docker-compose.yml)
This enables:

✅ Dynamic scaling
✅ Health checks for improved reliability
✅ Automatic container recovery in case of failures

docker-compose.yml

version: '3.8'

services:
  app:
    build: .
    deploy:
      replicas: 3  # Scale app instances dynamically
    networks:
      - app_network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000"]
      interval: 5s
      retries: 5
      timeout: 3s

  nginx:
    image: nginx:latest
    container_name: nginx
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "8080:80"
    networks:
      - app_network
    depends_on:
      - app

networks:
  app_network:
    driver: bridge

##  Step 5: Build and Run

Build the containers:
docker-compose up --build

Scale the FastAPI app dynamically:
docker-compose up --build --scale app=3


##  Step 6: Testing

Run the following command multiple times to see responses from different instances:

curl localhost:8080

Expected output (alternating container responses):
{"message": "Hello from app_1!"}
{"message": "Hello from app_2!"}
{"message": "Hello from app_3!"}


##  Step 7: Delete the stack
docker-compose down






