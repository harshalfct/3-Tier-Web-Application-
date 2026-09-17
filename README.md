# 3-Tier Registration Project
## React + NGINX + Flask + MySQL + Docker Compose

This project is a simplified production-style 3-tier web application showing how to integrate and secure React, Flask, and MySQL inside Docker.

## Architecture & Network Isolation

```
           [ Browser ]
                |
                v  (port 80)
      [ NGINX Reverse Proxy ]   <--- (web-network)
         |               |
         |               v
         |       [ React Frontend ]  <--- (web-network)
         v
  [ Flask Backend ]                  <--- (web-network & db-network)
         |
         v  (port 3306)
     [ MySQL ]                       <--- (db-network)
```

For security, the containers are separated into two isolated networks:
1. **`web-network`**: Connects the `proxy`, `frontend`, and `backend`.
2. **`db-network`**: Connects the `backend` and `db` (MySQL).

*The Nginx proxy and MySQL database are on different networks and cannot communicate directly with each other.*

---

## Directory Structure

```text
3-Tier-Web-Application/
│
├── .env                  # DB Credentials & Configuration (Git-ignored)
├── .env.example          # Sample environment variables template
├── docker-compose.yml    # Orchestrates services and isolated networks
│
├── k8s/                  # Kubernetes (Kubeadm) production manifests & guide
├── proxy/                # NGINX reverse proxy configuration
│   ├── Dockerfile
│   └── nginx.conf
│
├── frontend/             # React (Vite) application
│   ├── Dockerfile
│   ├── package.json
│   ├── index.html
│   ├── vite.config.js
│   └── src/
│       ├── main.jsx
│       └── style.css
│
├── backend/              # Python Flask API backend
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
│       ├── __init__.py
│       ├── __main__.py
│       └── main.py
│
└── db/                   # Database initialization
    └── init.sql          # Creates users table
```

---

## Kubernetes Deployment (Kubeadm)

To deploy this application on a production-ready Kubernetes cluster (Kubeadm), refer to the dedicated guide in [`k8s/README.md`](file:///e:/3-Tier-Web-Application-/k8s/README.md).

Quick start:
```bash
# Apply all manifests using Kustomize:
kubectl apply -k k8s/
```

---

## Setup & Getting Started

1. **Environment Variables**:
   Copy `.env.example` to a new file named `.env` and configure your credentials:
   ```bash
   cp .env.example .env
   ```

2. **Start the Application**:
   Run the following command to build and launch all services in detached mode:
   ```bash
   docker compose up -d --build
   ```

3. **Verify Status**:
   Ensure all containers are running and healthy:
   ```bash
   docker compose ps
   ```

4. **Access the App**:
   Open your browser and navigate to:
   ```text
   http://localhost
   ```

---

## Authentication Mechanism
This application uses a simplified authentication system:
* Upon successful login, the server returns the user's database `ID`.
* The frontend stores the ID inside local storage as `userId`.
* Subsequent requests send the header `Authorization: Bearer <userId>`.
* The backend parses this header and fetches details directly from the database, eliminating JWT overhead.
