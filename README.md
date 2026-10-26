# Fullstack Deployment Guide: Django Backend on AWS EC2 with Gunicorn, Nginx & Cloudflare

<div align="center">

[![GitHub Release](https://img.shields.io/github/v/release/codewithmuh/backend-django?color=blue&logo=github)](https://github.com/codewithmuh/backend-django/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub Stars](https://img.shields.io/github/stars/codewithmuh/backend-django?style=flat&color=gold)](https://github.com/codewithmuh/backend-django/stargazers)
[![GitHub Forks](https://img.shields.io/github/forks/codewithmuh/backend-django?style=flat&color=teal)](https://github.com/codewithmuh/backend-django/network/members)
[![Django](https://img.shields.io/badge/Backend-Django_DRF-092E20?logo=django)](https://www.djangoproject.com/)
[![Gunicorn](https://img.shields.io/badge/WSGI-Gunicorn-499848?logo=gunicorn)](https://gunicorn.org/)
[![AWS EC2](https://img.shields.io/badge/Compute-AWS_EC2_Ubuntu-FF9900?logo=amazon-aws)](https://aws.amazon.com/ec2/)
[![Nginx](https://img.shields.io/badge/Reverse_Proxy-Nginx-009639?logo=nginx)](https://nginx.org/)
[![Frontend: React](https://img.shields.io/badge/Paired_Frontend-React_SPA-61DAFB?logo=react)](https://github.com/codewithmuh/frontend-react)
[![YouTube: @codewithmuh](https://img.shields.io/badge/YouTube-codewithmuh-FF0000?logo=youtube&logoColor=white)](https://www.youtube.com/@codewithmuh)

**Complete deployment architecture and production guide for hosting a Django REST Framework backend alongside a React SPA on a single AWS EC2 server using Gunicorn, Systemd, Nginx, and Cloudflare.**

[Architecture](#system-architecture) • [EC2 Setup](#1-aws-ec2-instance-setup) • [Backend Setup](#3-django-backend-environment-setup) • [Gunicorn Service](#4-gunicorn-systemd-service-configuration) • [Nginx Setup](#5-nginx-unified-reverse-proxy) • [Paired Frontend](#paired-frontend-repository)

</div>

---

![Deployment Architecture Banner](https://github.com/user-attachments/assets/0d84814f-8f0e-4f8f-b3fd-6fd696cbbe15)

## Overview

This repository houses the **Django REST Framework (DRF)** backend featuring JWT authentication, automated Swagger/OpenAPI documentation, and user/blog account APIs. 

Engineered to run on the same AWS EC2 virtual machine as its paired [`frontend-react`](https://github.com/codewithmuh/frontend-react) single-page app, this production design eliminates CORS overhead and saves server costs while delivering enterprise-grade security and reliability via Systemd and Nginx reverse proxying.

---

## System Architecture

```mermaid
flowchart TD
    subgraph Clients["Users & Browsers"]
        Browser["User Web Browser"]
    end

    subgraph Edge["Edge Network & CDN"]
        Cloudflare["Cloudflare (Universal SSL + DNS Proxy)"]
    end

    subgraph AWSCloud["AWS Infrastructure (EC2 Ubuntu 22.04 LTS)"]
        EIP["AWS Elastic IP (Static Public IPv4)"]
        
        subgraph ServerHost["Single Host Nginx Server"]
            Nginx["Nginx Web Server / Reverse Proxy (Port 80 / 443)"]
            StaticFiles["React Production Build (/var/www/frontend/build)"]
            
            subgraph BackendService["Django Gunicorn Systemd Service"]
                Gunicorn["Gunicorn WSGI Workers (127.0.0.1:8000)"]
                Django["Django 4.x + REST Framework + SimpleJWT"]
                DB[("PostgreSQL / SQLite Database")]
            end
        end
    end

    Browser -->|HTTPS Request| Cloudflare
    Cloudflare -->|Proxied Traffic| EIP
    EIP --> Nginx
    Nginx -->|Route / (Static HTML/JS)| StaticFiles
    Nginx -->|Route /api/* & /admin/*| Gunicorn
    Gunicorn --> Django
    Django --> DB
```

---

## Step-by-Step Production Deployment Guide

### 1. AWS EC2 Instance Setup
1. Launch an **Ubuntu Server 22.04 LTS** instance on AWS EC2 (`t2.micro` or `t3.small`).
2. Attach an **AWS Elastic IP** to ensure persistent static IP addressing across restarts.
3. Add an `A` record in **Cloudflare DNS** pointing your domain to the Elastic IP address.
4. Set Cloudflare SSL mode to **Full**.

### 2. System Dependencies Installation
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-venv python3-dev nginx git curl
```

### 3. Django Backend Environment Setup
Clone the repository and prepare Python virtual environment:
```bash
sudo mkdir -p /var/www/backend
sudo chown -R $USER:$USER /var/www/backend
git clone https://github.com/codewithmuh/backend-django.git /var/www/backend

cd /var/www/backend
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

# Run migrations and collect static files
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
```

### 4. Gunicorn Systemd Service Configuration
Create a background systemd service file to manage the Gunicorn WSGI processes automatically on boot:

```bash
sudo nano /etc/systemd/system/gunicorn.service
```

Paste the following service configuration:

```ini
[Unit]
Description=Gunicorn daemon for Django Backend
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/var/www/backend
ExecStart=/var/www/backend/venv/bin/gunicorn \
          --workers 3 \
          --bind 127.0.0.1:8000 \
          blogapp.wsgi:application

Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Enable and start Gunicorn:
```bash
sudo systemctl daemon-reload
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
sudo systemctl status gunicorn
```

### 5. Nginx Unified Reverse Proxy
Configure Nginx to serve the paired React build from `/var/www/frontend/build` and proxy API requests to Gunicorn:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    # Frontend React Static Files
    root /var/www/frontend/build;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Django API Reverse Proxy
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Django Admin Dashboard
    location /admin/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Django Static Assets
    location /static/ {
        alias /var/www/backend/static/;
    }

    # User Uploaded Media
    location /media/ {
        alias /var/www/backend/media/;
    }
}
```

Reload Nginx:
```bash
sudo nginx -t && sudo systemctl restart nginx
```

---

## Paired Frontend Repository

For frontend React application code and build instructions, visit the companion repository:
👉 [**codewithmuh/frontend-react**](https://github.com/codewithmuh/frontend-react)

---

## Community & Video Walkthrough

- 📺 **YouTube Channel:** [CodeWithMuh](https://www.youtube.com/@codewithmuh)
- 🌐 **Connect on LinkedIn:** [Muhammad Rashid](https://www.linkedin.com/in/codewithmuh)

---

## License

This project is licensed under the [MIT License](LICENSE).
