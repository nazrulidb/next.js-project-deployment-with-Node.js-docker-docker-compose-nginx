# next.js-project-deployment-with-Node.js-docker-docker-compose-nginx

1. Create a non-root user:
   
          adduser ubuntu
3. add the user to sudo group:
   
       vi /etc/sudoers
       add the line
        username ALL=(ALL) NOPASSWD:ALL
3. Install Docker, Docker compose on the ubuntu server:
   
      # Add Docker's official GPG key:
      	sudo apt update
      	sudo apt install ca-certificates curl
      	sudo install -m 0755 -d /etc/apt/keyrings
      	sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
      	sudo chmod a+r /etc/apt/keyrings/docker.asc
      
      	# Add the repository to Apt sources:
      	sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
      	Types: deb
      	URIs: https://download.docker.com/linux/ubuntu
      	Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
      	Components: stable
      	Architectures: $(dpkg --print-architecture)
      	Signed-By: /etc/apt/keyrings/docker.asc
      	EOF
      
      	sudo apt update
      
      	sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
      	
      	sudo apt  install docker-compose
4. To run docker command without sudo:

         Create the docker group:
         sudo groupadd docker
                  
         Add your user to the docker group:
         sudo usermod -aG docker $USER

         You can also run the following command to activate the changes to groups:
         newgrp docker

5. Create a Dockerfile

        # Stage 1: Install dependencies
      	FROM node:20-alpine AS deps
      	RUN apk add --no-cache libc6-compat
      	WORKDIR /app
      
      	COPY package.json package-lock.json* ./
      	RUN npm ci
      
      	# Stage 2: Build the application
      	FROM node:20-alpine AS builder
      	WORKDIR /app
      	COPY --from=deps /app/node_modules ./node_modules
      	COPY . .
      	RUN npm run build

        # Stage 3: Runner
      	FROM node:20-alpine AS runner
      	WORKDIR /app
      
      	ENV NODE_ENV=production
      
      	RUN addgroup --system --gid 1001 nodejs
      	RUN adduser --system --uid 1001 nextjs
      
      	# 1. Copy files
      	COPY --from=builder /app/public ./public
      	COPY --from=builder /app/.next ./.next
      	COPY --from=builder /app/node_modules ./node_modules
      	COPY --from=builder /app/package.json ./package.json
      
      	# 2. FIX PERMISSIONS: This is the crucial missing step
      	# We make sure the nextjs user owns the cache directory so it can write to it
      	RUN chown -R nextjs:nodejs /app/.next
      
      	USER nextjs
      
      	EXPOSE 3000
      	ENV PORT=3000
      
      	CMD ["npm", "start"]
   

6. Build image:
   
         docker build -t my-nextjs-app .

7. Create docker compose file. docker-compose.yaml
         version: '3.8'
      
      		services:
      		  nextjs_app:
      			build: .
      			container_name: nextjs_container
      			restart: always
      			expose:
      			  - "3000"
      
      		  nginx:
      			image: nginx:alpine
      			container_name: nginx_proxy
      			restart: always
      			ports:
      			  - "80:80"
      			  - "443:443"
      			volumes:
      			  - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      			  - ./.next/static:/app/.next/static:ro # Mount static files for Nginx to serve
      			  - ./certbot/conf:/etc/letsencrypt:ro
      			  - ./certbot/www:/var/www/certbot:ro
      			depends_on:
      			  - nextjs_app
      		   
      		   
      		  certbot:
      			image: certbot/certbot
      			volumes:
      			  - ./certbot/conf:/etc/letsencrypt
      			  - ./certbot/www:/var/www/certbot

8. First nginx conf before let's encrypt certbot run:

           server {
             listen 80;
             server_name next.lemle.online;
   
            location /.well-known/acme-challenge/ {
            root /var/www/certbot;
            }
       
            # Only one location / block here!
          }

9. Run container:

         docker-compose up -d
            
         Check docker container is running:
            
         docker ps

10. Run certbot command to generate Let's Encrypt SSL certificates

        docker-compose run --rm certbot certonly --webroot --webroot-path /var/www/certbot -d next.lemle.online

11. Nginx conf file after SSL generate: edit nginx.conf file which is mounted to nginx container

          server {
      	 	listen 80;
      	 	server_name next.lemle.online;
          
      		location /.well-known/acme-challenge/ {
      			root /var/www/certbot;
      		}
      	 	
      	 	# Only one location / block here!
      	 	location / {
      	 		return 301 https://$host$request_uri;
      	 	}
      	  }
           
      	 server {
      		listen 443 ssl;
      		server_name next.lemle.online; # Change this!
      
      		ssl_certificate /etc/letsencrypt/live/next.lemle.online/fullchain.pem;
      		ssl_certificate_key /etc/letsencrypt/live/next.lemle.online/privkey.pem;
      
      		location / {
      			proxy_pass http://nextjs_app:3000;
      			# Standard headers
      			proxy_set_header Host $host;
      			proxy_set_header X-Real-IP $remote_addr;
      		 }
      	   
      	  }
    

      

12. Restart nginx container:

        docker restart nginx_container_name

         


   
