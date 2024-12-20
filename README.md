# NGINX with Tailscale Integration: Complete Guide

A production-ready Docker Compose implementation demonstrating secure NGINX deployment within a Tailscale network. This setup provides automated SSL certificate management, dynamic configuration, and secure networking capabilities through a custom Tailscale sidecar container.

[![Docker Compose](https://img.shields.io/badge/Docker%20Compose-supported-blue)](https://docs.docker.com/compose/)
[![Tailscale](https://img.shields.io/badge/Tailscale-integrated-success)](https://tailscale.com/)
[![NGINX](https://img.shields.io/badge/NGINX-latest-green)](https://nginx.org/)
[![Publish Docker Image](https://github.com/hhftechnology/tailscale-sidecar/actions/workflows/main.yml/badge.svg?branch=main)](https://github.com/hhftechnology/tailscale-sidecar/actions/workflows/main.yml)

## System Architecture

This implementation uses a two-container approach:
1. **Tailscale Sidecar**: Manages networking and certificates
2. **NGINX Server**: Handles web traffic with SSL termination

### Network Flow
```mermaid
graph LR
    A[Client] -->|HTTPS| B[Tailscale Network]
    B -->|Internal| C[NGINX Container]
    D[Tailscale Sidecar] -->|Certs| C
    D -->|Network| C
```

## Prerequisites

### Required Software
- Docker Engine (20.10.0+)
- Docker Compose (2.0.0+)
- Tailscale account with admin access

### System Requirements
- Linux-based OS (recommended)
- Minimum 1GB RAM
- 10GB available disk space

## Environment Configuration

### Required Variables
Create a `.env` file with these mandatory settings:

```env
# Tailscale Authentication
TS_AUTHKEY=tskey-auth-xxxxx-xxxxxxxxxxxxx  # Your Tailscale auth key
TS_HOSTNAME=web-server-prod                 # Internal Tailscale hostname
TS_HOST_FQDN=your.domain.com               # External FQDN for SSL

# Optional Performance Tuning
NGINX_WORKER_PROCESSES=auto                 # Defaults to auto
NGINX_WORKER_CONNECTIONS=1024               # Connection limit per worker
```

### Security Considerations
- Store `TS_AUTHKEY` securely
- Use ephemeral keys when possible
- Rotate keys periodically
- Implement least-privilege access

## Project Structure

```plaintext
.
├── docker-compose.yml          # Main service configuration
├── .env                        # Environment variables
├── nginx_conf/
│   ├── nginx-template.conf     # Base NGINX configuration
│   ├── security-headers.conf   # Security header settings
│   └── ssl-params.conf         # SSL optimization parameters
└── scripts/
    ├── healthcheck.sh         # Container health monitoring
    └── cert-validator.sh      # Certificate validation script
```

## Technical Implementation

### Docker Compose Configuration

```yaml
version: '3.8'

services:
  # Tailscale Service
  ts-nginx:
    image: hhftechnology/tailscale-sidecar:latest
    container_name: ts-${TS_HOSTNAME}
    restart: unless-stopped
    hostname: ${TS_HOSTNAME}
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_DOMAIN_NAME=${TS_HOST_FQDN}
    volumes:
      - ts-state:/var/lib/tailscale    # Persist Tailscale state
      - ts-certs:/certs                # Share certificates
      - /dev/net/tun:/dev/net/tun      # Required for VPN
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    healthcheck:
      test: ["CMD", "/scripts/healthcheck.sh"]
      interval: 30s
      timeout: 10s
      retries: 3

  # NGINX Service
  nginx:
    image: nginx:mainline-alpine
    container_name: ts-${TS_HOSTNAME}-nginx
    restart: unless-stopped
    network_mode: service:ts-nginx
    depends_on:
      ts-nginx:
        condition: service_healthy
    volumes:
      - ts-certs:/certs:ro
      - ./nginx_conf:/etc/nginx/conf.d:ro
      - web-content:/usr/share/nginx/html
    environment:
      - TS_HOST_FQDN=${TS_HOST_FQDN}
      - NGINX_WORKER_PROCESSES=${NGINX_WORKER_PROCESSES:-auto}
      - NGINX_WORKER_CONNECTIONS=${NGINX_WORKER_CONNECTIONS:-1024}
    entrypoint: [
      "sh", "-c",
      "envsubst < /etc/nginx/conf.d/nginx-template.conf > /etc/nginx/nginx.conf && 
       nginx -g 'daemon off;'"
    ]
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  ts-state:
    driver: local
  ts-certs:
    driver: local
  web-content:
    driver: local
```

### NGINX Configuration Template

```nginx
# nginx-template.conf
user nginx;
worker_processes ${NGINX_WORKER_PROCESSES};
worker_rlimit_nofile 65535;

events {
    worker_connections ${NGINX_WORKER_CONNECTIONS};
    multi_accept on;
    use epoll;
}

http {
    # Basic Settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 16M;

    # MIME Types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # SSL Configuration
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # Logging
    access_log /var/log/nginx/access.log combined buffer=512k flush=1m;
    error_log /var/log/nginx/error.log warn;

    # Security Headers
    include /etc/nginx/conf.d/security-headers.conf;

    # Virtual Host Configuration
    server {
        listen 80;
        server_name ${TS_HOST_FQDN};
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name ${TS_HOST_FQDN};

        # SSL Certificate Configuration
        ssl_certificate /certs/${TS_HOST_FQDN}.crt;
        ssl_certificate_key /certs/${TS_HOST_FQDN}.key;

        # Additional SSL Parameters
        include /etc/nginx/conf.d/ssl-params.conf;

        # Root Configuration
        root /usr/share/nginx/html;
        index index.html index.htm;

        # Location Blocks
        location / {
            try_files $uri $uri/ =404;
            add_header Cache-Control "public, no-transform";
        }

        # Error Pages
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
}
```

### Security Headers Configuration

```nginx
# security-headers.conf
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

## Deployment Guide

### Initial Setup

1. Clone this repository:
```bash
git clone https://github.com/hhftechnology/tailscaled-nginx.git
cd tailscaled-nginx
```

2. Create and configure `.env`:
```bash
cp .env.example .env
nano .env  # Add your configurations
```

3. Launch the stack:
```bash
docker-compose up -d
```

### Verification Steps

1. Check service status:
```bash
docker-compose ps
```

2. Verify Tailscale connection:
```bash
docker-compose exec ts-nginx tailscale status
```

3. Test NGINX configuration:
```bash
docker-compose exec nginx nginx -t
```

4. Validate SSL certificates:
```bash
docker-compose exec nginx /scripts/cert-validator.sh
```

## Performance Tuning

### NGINX Optimizations
- Adjust `worker_processes` based on CPU cores
- Configure `worker_connections` for concurrent connections
- Enable HTTP/2 for improved performance
- Implement proper caching strategies

### Monitoring

Monitor key metrics:
- Certificate expiration
- Connection status
- Error rates
- Response times

## Troubleshooting Guide

### Common Issues

1. **Certificate Problems**
```bash
# Check certificate validity
docker-compose exec nginx openssl x509 -in /certs/${TS_HOST_FQDN}.crt -text -noout
```

2. **Network Connectivity**
```bash
# Verify Tailscale connection
docker-compose exec ts-nginx tailscale ping your.domain.com
```

3. **NGINX Configuration**
```bash
# Test configuration
docker-compose exec nginx nginx -t
# Check logs
docker-compose logs nginx
```

### Debug Mode

Enable debug logging:
1. Add to nginx-template.conf:
```nginx
error_log /var/log/nginx/error.log debug;
```
2. Reload configuration:
```bash
docker-compose exec nginx nginx -s reload
```

## Maintenance

### Regular Tasks
- Monitor certificate expiration
- Review logs for issues
- Update Docker images
- Rotate Tailscale keys
- Backup configuration

### Backup Strategy
1. Backup volumes:
```bash
docker run --rm -v ts-state:/source:ro -v $(pwd)/backup:/backup alpine tar czf /backup/ts-state.tar.gz -C /source .
```

2. Backup configurations:
```bash
tar czf nginx-conf-backup.tar.gz nginx_conf/
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Submit pull request with:
   - Clear description
   - Tests if applicable
   - Updated documentation

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) file for details.

## Support

- Create an issue for bugs
- Submit feature requests
- Join our community discussions

## Acknowledgments

- Tailscale team for excellent documentation
- NGINX community for configuration examples
- Docker community for compose examples

---

**Note**: Always review security implications before deploying in production.
