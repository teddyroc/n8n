# n8n

docker-compose.yml
version: "3.8"

services:
  nginx:
    image: nginx:alpine
    container_name: n8n-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - n8n

  n8n:
    image: n8nio/n8n
    dns:
      - 8.8.8.8
      - 1.1.1.1
    container_name: n8n
    restart: unless-stopped
    expose:
      - "5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=admin123

      - N8N_HOST=n8n.local
      - N8N_PORT=5678
      - N8N_PROTOCOL=https

      - WEBHOOK_URL=https://n8n.local/
      - N8N_EDITOR_BASE_URL=https://n8n.local/
      - GENERIC_TIMEZONE=Asia/Seoul
      - N8N_PUSH_BACKEND=sse
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:


nginx/nginx.conf
events {}

http {
  upstream n8n_upstream {
    server n8n:5678;
  }

  server {
    listen 80;
    server_name n8n.local;
    return 301 https://$host$request_uri;
  }

  server {
    listen 443 ssl;
    server_name n8n.local;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
      proxy_pass http://n8n_upstream;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-Proto https;
      proxy_set_header X-Forwarded-For $remote_addr;

      # --- Prevent Connection Timeout ---
      proxy_read_timeout 86400;
      proxy_send_timeout 86400;
    }
  }
}
