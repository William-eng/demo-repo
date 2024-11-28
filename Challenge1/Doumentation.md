# Challenge Overview
- The tasks for the challenge are outlined below:

- Dockerize a full-stack application with a FastAPI backend and a React frontend.

- Set up and deploy monitoring and logging tools, including Prometheus, Grafana, Loki, Promtail, and cAdvisor.

- Configure a reverse proxy for application routing.

- Deploy the application to a cloud platform with the correct domain setup.



  ## Project Implementation
This article will briefly cover deployment but will focus more on monitoring. For a step-by-step guide on local and production deployment, click here.

Technologies Used:

- Docker: For containerizing applications.

- AWS EC2: Infrastructure for deploying applications.

- PostgreSQL: Serves as the database.

- cAdvisor: Exposes container metrics.

- Promtail: Collects logs from different sources on the server and sends them to Loki.

- Loki: Aggregates and stores logs received from Promtail.

- Prometheus: Collects metrics from cAdvisor and sends them to Grafana.

- Grafana: Visualizes metrics and logs.



## Containerization
Let's start by cloning the repository.


         git clone https://github.com/DrInTech22/cv-challenge01.git
         cd cv-challenge01
 
Here are the Dockerfiles for the frontend (React) and the backend (FastAPI) applications.

**Frontend Dockerfile**

            # Stage 1: Build the Node.js frontend application
            FROM node:20.18-alpine3.19 AS builder
            
            # Set working directory inside the container
            WORKDIR /app
            
            # Copy package.json and package-lock.json files for dependency installation
            COPY package*.json ./
            RUN npm cache clean --force
            # Install dependencies
            RUN npm install
            
            # Copy the rest of the application's code
            COPY . .
            
            # Build the application (output will go to the 'build' directory for React)
            RUN npm run build
            
            # Stage 2: Set up Nginx to serve the static files
            FROM nginx:1.10.1-alpine
            
            # Copy a custom Nginx configuration file for reverse proxy
            COPY nginx.conf /etc/nginx/nginx.conf
            
            # Copy built application from the previous stage to Nginx html directory
            COPY --from=builder /app/dist /usr/share/nginx/html
            
            # Expose port 80 to be accessible outside of the container
            EXPOSE 80
            
            # Start Nginx
            CMD ["nginx", "-g", "daemon off;"]


**Backend Dockerfile**

            # Use the latest official Python image as a base
            FROM python:3.12-bookworm
            
            # Install Node.js and npm
            RUN apt-get update && apt-get install -y \
                nodejs \
                npm
            
            # Install Poetry using pip
            RUN pip install poetry
            
            # Set the working directory
            WORKDIR /app
            
            # Copy the application files
            COPY . .
            
            # Install dependencies using Poetry
            RUN poetry install
            
            # Expose the port FastAPI runs on
            EXPOSE 8000
            
            # Run the prestart script and start the server
            CMD ["sh", "-c", "poetry run bash ./prestart.sh && poetry run uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"]



## Docker Compose Setup & Monitoring Configurations
The monitoring folder contains configuration files for Promtail, Loki, and Prometheus.

**promtail-config.yml**

                server:
                  http_listen_port: 9080
                  grpc_listen_port: 0
                clients:
                  - url: http://loki:3100/loki/api/v1/push
                positions:
                  filename: /tmp/positions.yaml
                scrape_configs:
                  - job_name: system
                    static_configs:
                      - targets:
                          - localhost
                        labels:
                          job: varlogs
                          __path__: /var/log/*.log



The promtail configuration sets up a server to listen on port 9080, defines where to store log positions, specifies the Loki endpoint for log shipping, and configures log scraping from the server and Docker log files with appropriate labels.

**loki-config.yml**

                auth_enabled: false
                server:
                  http_listen_port: 3100
                ingester:
                  lifecycler:
                    ring:
                      kvstore:
                        store: inmemory
                      replication_factor: 1
                  chunk_idle_period: 5m
                  max_chunk_age: 1h
                  chunk_target_size: 1048576
                schema_config:
                  configs:
                    - from: 2020-10-24
                      store: boltdb-shipper
                      object_store: filesystem
                      schema: v11
                      index:
                        prefix: index_
                        period: 168h
                storage_config:
                  boltdb_shipper:
                    active_index_directory: /loki/index
                    shared_store: filesystem
                    cache_location: /loki/cache
                  filesystem:
                    directory: /loki/chunks
                limits_config:
                  enforce_metric_name: false
                  reject_old_samples: true
                  reject_old_samples_max_age: 168h


This configuration sets up Loki to run on specific ports, use in-memory storage, enable caching, and define the schema for storing and indexing log data.

**prometheus.yml**

          global:
            scrape_interval: 15s
          scrape_configs:
            - job_name: "prometheus"
              metrics_path: "/prometheus/metrics"
              static_configs:
                - targets: ["prometheus:9090"]
            - job_name: "cadvisor"
              static_configs:
                - targets: ["cadvisor:8080"]


**Application stack: Compose.yml**

                        version: '3.9'
                        
                        services:
                          backend:
                            build:
                              context: ./backend
                            ports:
                              - "8000:8000"
                            depends_on:
                              db:
                                condition: service_healthy
                            env_file:
                              - ./backend/.env
                            networks:
                              - chall_network
                        
                          frontend:
                            build:
                              context: ./frontend
                            ports:
                              - "80:80"
                            env_file:
                              - ./frontend/.env
                            depends_on:
                              - backend
                            networks:
                              - chall_network
                        
                          db:
                            image: postgres:15.9-alpine3.20
                            restart: unless-stopped
                            container_name: postgres_db
                            ports:
                              - "5432:5432"
                            volumes:
                              - postgres_data:/var/lib/postgresql/data
                            env_file:
                              - ./backend/.env
                            networks:
                              - chall_network
                            healthcheck:
                              test: ["CMD", "pg_isready", "-U", "app", "-h", "localhost"]
                              interval: 10s
                              timeout: 5s
                              retries: 5
                              start_period: 30s
                        
                          adminer:
                            image: adminer
                            container_name: adminer
                            ports:
                              - "8081:8080"
                            networks:
                              chall_network:
                        
                        # docker-compose-monitoring.yml
                          prometheus:
                            image: prom/prometheus
                            container_name: prometheus
                            volumes:
                              - ./prometheus.yml:/etc/prometheus/prometheus.yml
                            ports:
                              - "9090:9090"
                            networks:
                              - chall_network
                        
                          grafana:
                            image: grafana/grafana
                            container_name: grafana
                            ports:
                              - "3000:3000"
                            networks:
                              - chall_network
                        
                          loki:
                            image: grafana/loki
                            container_name: loki
                            ports:
                              - "3100:3100"
                            volumes:
                              - ./loki-config.yml:/etc/loki/loki-config.yml
                            networks:
                              - chall_network
                        
                          promtail:
                            image: grafana/promtail
                            container_name: promtail
                            volumes:
                              - ./promtail-config.yml:/etc/promtail/promtail-config.yml
                              - /var/log:/var/log
                            command:
                              -config.file=/etc/promtail/promtail-config.yml
                            networks:
                              - chall_network
                        
                          cadvisor:
                            image: gcr.io/cadvisor/cadvisor:latest
                            container_name: cadvisor
                            ports:
                              - "8082:8080"
                            volumes:
                                - /:/rootfs:ro
                                - /var/run:/var/run:rw
                                - /sys:/sys:ro
                                - /var/lib/docker/:/var/lib/docker:ro
                            networks:
                              - chall_network
                        
                        volumes:
                          postgres_data:
                        
                        networks:
                          chall_network:

Open the nginx.conf file and make sure the domain names match your configured domain names. Then, add the SSL paths for each domain in their respective SSL server blocks.

Generating SSL certificate

*.key for the private key and *.crt for public
- ![Screenshot from 2024-11-28 01-21-42](https://github.com/user-attachments/assets/982af7ba-a039-4f12-be5d-6bf077023fe6)




**Nginx.conf**


            version: '3.9'
            
            services:
              backend:
                build:
                  context: ./backend
                ports:
                  - "8000:8000"
                depends_on:
                  db:
                    condition: service_healthy
                env_file:
                  - ./backend/.env
                networks:
                  - chall_network
            
              frontend:
                build:
                  context: ./frontend
                ports:
                  - "80:80"
                  - "443:443"
                env_file:
                  - ./frontend/.env
                depends_on:
                  - backend
                networks:
                  - chall_network
            
              db:
                image: postgres:15.9-alpine3.20
                restart: unless-stopped
                container_name: postgres_db
                ports:
                  - "5432:5432"
                volumes:
                  - postgres_data:/var/lib/postgresql/data
                env_file:
                  - ./backend/.env
                networks:
                  - chall_network
                healthcheck:
                  test: ["CMD", "pg_isready", "-U", "app", "-h", "localhost"]
                  interval: 10s
                  timeout: 5s
                  retries: 5
                  start_period: 30s
            
              adminer:
                image: adminer
                container_name: adminer
                ports:
                  - "8081:8080"
                networks:
                  chall_network:
            
            # docker-compose-monitoring.yml
              prometheus:
                image: prom/prometheus
                container_name: prometheus
                volumes:
                  - ./prometheus.yml:/etc/prometheus/prometheus.yml
                ports:
                  - "9090:9090"
                networks:
                  - chall_network
            
              grafana:
                image: grafana/grafana
                container_name: grafana
                ports:
                  - "3000:3000"
                networks:
                  - chall_network
            
              loki:
                image: grafana/loki
                container_name: loki
                ports:
                  - "3100:3100"
                volumes:
                  - ./loki-config.yml:/etc/loki/loki-config.yml
                networks:
                  - chall_network
            
              promtail:
                image: grafana/promtail
                container_name: promtail
                volumes:
                  - ./promtail-config.yml:/etc/promtail/promtail-config.yml
                  - /var/log:/var/log
                command:
                  -config.file=/etc/promtail/promtail-config.yml
                networks:
                  - chall_network
            
              cadvisor:
                image: gcr.io/cadvisor/cadvisor:latest
                container_name: cadvisor
                ports:
                  - "8082:8080"
                volumes:
                    - /:/rootfs:ro
                    - /var/run:/var/run:rw
                    - /sys:/sys:ro
                    - /var/lib/docker/:/var/lib/docker:ro
                networks:
                  - chall_network
            
            volumes:
              postgres_data:
            
            networks:
              chall_network:



























































































































































































































































