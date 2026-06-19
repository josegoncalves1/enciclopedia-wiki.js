# 1️⃣-3️⃣ WIKI.JS: INSTALAÇÃO, GITLAB INTEGRATION & ORGANIZAÇÃO

---

# 1️⃣ INSTALAÇÃO & SETUP

## 📦 EXEMPLO 1: Docker Compose (5 minutos)

**Arquivo:** `docker-compose.yml`

```yaml
version: '3.8'

services:
  wiki:
    image: requarks/wiki:latest
    hostname: wiki.example.com
    container_name: wiki-js
    restart: always
    
    depends_on:
      - postgres
      - redis
    
    ports:
      - "80:3000"
    
    environment:
      DB_TYPE: postgres
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: wiki
      DB_PASS: wiki_password
      DB_NAME: wiki
      
      REDIS_HOST: redis
      REDIS_PORT: 6379
      
      # SSL
      SSL_ENABLED: "false"
      # Configure SSL_CERT e SSL_KEY para HTTPS
      
      # Authentication
      JWT_SECRET: your-secret-key-here
      JWT_EXPIRATION: 7d
      
      # Email
      MAIL_HOST: smtp.gmail.com
      MAIL_PORT: 587
      MAIL_USER: your-email@gmail.com
      MAIL_PASS: your-password
      MAIL_FROM_NAME: "Wiki Notifications"
      
      # Site
      SITE_TITLE: "My Wiki"
      SITE_LANG: en
      SITE_URL: "http://wiki.example.com"
    
    volumes:
      - ./wiki/data:/var/lib/wiki
      - ./wiki/config:/etc/wiki
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: wiki-postgres
    restart: always
    
    environment:
      POSTGRES_DB: wiki
      POSTGRES_USER: wiki
      POSTGRES_PASSWORD: wiki_password
    
    volumes:
      - ./postgres/data:/var/lib/postgresql/data
    
    ports:
      - "5432:5432"
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wiki"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: wiki-redis
    restart: always
    
    command: redis-server --appendonly yes --requirepass redis_password
    
    volumes:
      - ./redis/data:/data
    
    ports:
      - "6379:6379"
    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # (Opcional) Elasticsearch para search avançado
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
    container_name: wiki-elasticsearch
    restart: always
    
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    
    volumes:
      - ./elasticsearch/data:/usr/share/elasticsearch/data
    
    ports:
      - "9200:9200"
    
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health | grep -q 'green'"]
      interval: 30s
      timeout: 10s
      retries: 5

networks:
  default:
    name: wiki-network
```

**Como usar:**

```bash
# Create directories
mkdir -p {wiki,postgres,redis,elasticsearch}/{data,config}

# Start
docker-compose up -d

# Verify
docker-compose logs -f wiki

# First access
open http://localhost:3000

# Admin setup (seguirá wizard)
# Email: admin@example.com
# Password: (setup na UI)

# Logs
docker-compose logs wiki
docker-compose logs postgres

# Stop
docker-compose down

# Backup
docker-compose exec postgres pg_dump -U wiki wiki > wiki_backup.sql
```

---

## 📦 EXEMPLO 2: Kubernetes + Helm

**Arquivo:** `helm-values.yml`

```yaml
replicaCount: 3

image:
  repository: requarks/wiki
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: wiki.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: wiki-tls
      hosts:
        - wiki.example.com

postgres:
  enabled: true
  auth:
    username: wiki
    password: secure_password
    database: wiki
  primary:
    persistence:
      enabled: true
      size: 10Gi

redis:
  enabled: true
  auth:
    enabled: true
    password: secure_password
  master:
    persistence:
      enabled: true
      size: 5Gi

elasticsearch:
  enabled: true
  replicas: 1
  resources:
    requests:
      memory: 512Mi
      cpu: 250m

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

**Deploy:**

```bash
# Add Helm repo
helm repo add wiki https://requarks.github.io/wiki
helm repo update

# Create namespace
kubectl create namespace wiki

# Install
helm install wiki wiki/wiki \
  -n wiki \
  -f helm-values.yml

# Monitor
kubectl get pods -n wiki -w
kubectl logs -n wiki deployment/wiki -f

# Access
kubectl port-forward -n wiki svc/wiki 3000:80
open http://localhost:3000
```

---

## 📦 EXEMPLO 3: Bare Metal Installation

**Arquivo:** `install.sh`

```bash
#!/bin/bash
set -euo pipefail

echo "🚀 Installing Wiki.js (Bare Metal)..."

# 1. Install dependencies
sudo apt update
sudo apt install -y curl nodejs npm postgresql postgresql-contrib redis-server nginx

# 2. Setup PostgreSQL
sudo systemctl start postgresql
sudo sudo -u postgres createdb wiki
sudo -u postgres createuser wiki
sudo -u postgres psql -c "ALTER USER wiki WITH PASSWORD 'secure_password';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE wiki TO wiki;"

# 3. Create wiki user
sudo adduser --disabled-login --gecos 'Wiki.js' wiki || true
sudo mkdir -p /opt/wiki
sudo chown -R wiki:wiki /opt/wiki

# 4. Download Wiki.js
cd /opt/wiki
sudo -u wiki wget https://github.com/Requarks/wiki/releases/download/v2.5.x/wiki-js.tar.gz
sudo -u wiki tar xzf wiki-js.tar.gz
sudo -u wiki rm wiki-js.tar.gz

# 5. Install Node dependencies
sudo -u wiki npm install --production

# 6. Configure Wiki.js
sudo -u wiki tee /opt/wiki/config.yml > /dev/null <<'EOF'
port: 3000
db:
  type: postgres
  host: localhost
  port: 5432
  user: wiki
  password: secure_password
  db: wiki
redis:
  host: localhost
  port: 6379
site:
  title: "My Wiki"
  url: "https://wiki.example.com"
  lang: en
mail:
  host: smtp.gmail.com
  port: 587
  secure: false
  user: your-email@gmail.com
  pass: your-password
  from: "Wiki <noreply@example.com>"
auth:
  jwt:
    secret: your-secret-key-here
    expiration: 7d
EOF

# 7. Create systemd service
sudo tee /etc/systemd/system/wiki.service > /dev/null <<'EOF'
[Unit]
Description=Wiki.js
After=network.target postgresql.service redis.service

[Service]
Type=simple
User=wiki
WorkingDirectory=/opt/wiki
ExecStart=/usr/bin/node server
Restart=always
Environment="NODE_ENV=production"

[Install]
WantedBy=multi-user.target
EOF

# 8. Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable wiki
sudo systemctl start wiki

# 9. Configure Nginx
sudo tee /etc/nginx/sites-available/wiki > /dev/null <<'EOF'
upstream wiki {
  server 127.0.0.1:3000;
}

server {
  listen 80;
  server_name wiki.example.com;
  
  location / {
    proxy_pass http://wiki;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
EOF

sudo ln -s /etc/nginx/sites-available/wiki /etc/nginx/sites-enabled/wiki
sudo nginx -t
sudo systemctl restart nginx

# 10. SSL with Let's Encrypt
sudo apt install -y certbot python3-certbot-nginx
sudo certbot certonly --nginx -d wiki.example.com -n --agree-tos --email admin@example.com

echo "✅ Wiki.js installed!"
echo "Access: https://wiki.example.com"
echo "Logs: sudo journalctl -u wiki -f"
```

**Como executar:**

```bash
chmod +x install.sh
sudo ./install.sh

# Verify
curl http://localhost:3000
```

---

# 2️⃣ GITLAB INTEGRATION

## 📦 EXEMPLO 4: OAuth2 Setup

**No GitLab (Admin → Applications):**

```bash
# 1. Create OAuth Application
Name: Wiki.js
Scopes: 
  ☑ read_user
  ☑ read_repository
Redirect URI: http://wiki.example.com/login/oauth2/callback

# Copy Client ID e Client Secret
```

**No Wiki.js (Admin → Authentication):**

```yaml
providers:
  - id: oauth2-gitlab
    type: oauth2
    title: "GitLab"
    isEnabled: true
    
    oauth2:
      clientId: YOUR_CLIENT_ID
      clientSecret: YOUR_CLIENT_SECRET
      authorizationURL: https://gitlab.example.com/oauth/authorize
      tokenURL: https://gitlab.example.com/oauth/token
      userProfileURL: https://gitlab.example.com/api/v4/user
      userNameField: username
      userEmailField: email
      userIdField: id
      
    mapping:
      email: email
      name: name
      avatar: avatar_url
```

---

## 📦 EXEMPLO 5: Git Sync (Bi-directional)

**No Wiki.js (Admin → Storage):**

```yaml
storage:
  target: git
  git:
    enabled: true
    url: https://gitlab.example.com/your-org/wiki-content.git
    branch: main
    path: wiki/
    
    # Authentication
    authType: oauth
    authToken: YOUR_GITLAB_TOKEN
    
    # Sync Settings
    syncInterval: 60  # minutes
    pullOnBoot: true
    pushOnSave: true
    commitMessage: "docs: update wiki content"
    committerName: "Wiki.js"
    committerEmail: "wiki@example.com"
```

**Estrutura no GitLab:**

```
wiki-content/
├── .gitignore
├── README.md
├── wiki/
│   ├── home/
│   │   └── index.md
│   ├── docs/
│   │   ├── getting-started/
│   │   │   ├── index.md
│   │   │   └── installation.md
│   │   ├── guides/
│   │   └── reference/
│   ├── assets/
│   │   ├── images/
│   │   └── files/
│   └── locales/
│       ├── en/
│       └── pt-br/
└── .gitlab-ci.yml
```

**CI/CD Pipeline:**

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - deploy

validate:markdown:
  stage: validate
  image: node:18
  script:
    - npm install -g markdownlint-cli
    - markdownlint "wiki/**/*.md"
  allow_failure: false

validate:links:
  stage: validate
  image: node:18
  script:
    - npm install -g markdown-link-check
    - find wiki -name "*.md" -exec markdown-link-check {} \;
  allow_failure: true

validate:spelling:
  stage: validate
  image: node:18
  script:
    - npm install -g cspell
    - cspell "wiki/**/*.md"
  allow_failure: true

deploy:wiki:
  stage: deploy
  image: curlimages/curl:latest
  script:
    - |
      curl -X POST \
        -H "Authorization: Bearer $WIKI_API_TOKEN" \
        https://wiki.example.com/api/pages/rebuild
  only:
    - main
  when: manual
```

---

# 3️⃣ ORGANIZAÇÃO & ESTRUTURA

## 📦 EXEMPLO 6: Information Architecture

**Estrutura Recomendada:**

```
📚 Wiki Root
├── 🏠 Home (Dashboard)
│
├── 📖 Getting Started
│   ├── Overview
│   ├── Installation Guide
│   ├── Quick Start
│   └── FAQ
│
├── 📚 Documentation
│   ├── Architecture
│   │   ├── System Overview
│   │   ├── Components
│   │   ├── Data Flow
│   │   └── Diagrams
│   │
│   ├── Development
│   │   ├── Setup Development Environment
│   │   ├── Project Structure
│   │   ├── Coding Standards
│   │   ├── Testing Guide
│   │   └── Debugging
│   │
│   ├── API Reference
│   │   ├── REST API
│   │   │   ├── Authentication
│   │   │   ├── Endpoints
│   │   │   └── Examples
│   │   └── GraphQL
│   │       ├── Schema
│   │       ├── Queries
│   │       └── Mutations
│   │
│   └── Guides
│       ├── How to Deploy
│       ├── How to Monitor
│       ├── How to Backup
│       └── How to Troubleshoot
│
├── 🔧 Operations
│   ├── Infrastructure
│   │   ├── Cloud Setup (AWS/Azure/GCP)
│   │   ├── Kubernetes
│   │   ├── Networking
│   │   └── Security
│   │
│   ├── Processes
│   │   ├── Release Process
│   │   ├── Incident Response
│   │   ├── On-Call Procedures
│   │   └── Change Management
│   │
│   └── Runbooks
│       ├── Application Down
│       ├── Database Issues
│       ├── Performance Problems
│       └── Security Incident
│
├── 👥 Team
│   ├── Organizational Structure
│   ├── Team Contacts
│   ├── Meeting Notes
│   └── Decision Log
│
└── 🌍 Resources
    ├── External Links
    ├── Tools & Services
    ├── Templates
    └── Learning Resources
```

---

## 📦 EXEMPLO 7: Page Templates

**Arquivo:** `templates/api-endpoint.md`

```markdown
# {{ endpoint }} Endpoint

## Overview
{{ brief description of what this endpoint does }}

## Request

### URL
```
{{ method }} /api/{{ path }}
```

### Authentication
Bearer {{ token_type }}

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| {{ name }} | {{ type }} | {{ yes/no }} | {{ description }} |

### Example Request
```bash
curl -X {{ method }} \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{{ example_payload }}' \
  https://api.example.com/api/{{ path }}
```

## Response

### Success (200)
```json
{{ example_response }}
```

### Error (4xx/5xx)
```json
{
  "error": "{{ error_code }}",
  "message": "{{ error_message }}"
}
```

### Status Codes
- `200` - Success
- `400` - Bad Request
- `401` - Unauthorized
- `403` - Forbidden
- `404` - Not Found
- `500` - Server Error

## Rate Limiting
- {{ rate_limit }} requests per {{ period }}

## Related
- [{{ related_endpoint }}](#{{ related_link }})
```

**Arquivo:** `templates/process.md`

```markdown
# {{ process_name }} Process

## Overview
{{ brief description }}

## Participants
- {{ role1 }}: {{ responsibility1 }}
- {{ role2 }}: {{ responsibility2 }}

## Prerequisites
- [ ] {{ prerequisite1 }}
- [ ] {{ prerequisite2 }}

## Steps

### Phase 1: Planning
1. {{ step1 }}
2. {{ step2 }}
   - Details: {{ details }}

### Phase 2: Execution
1. {{ step1 }}
2. {{ step2 }}

### Phase 3: Validation
- [ ] {{ check1 }}
- [ ] {{ check2 }}

## Rollback
In case of failure:
1. {{ rollback_step1 }}
2. {{ rollback_step2 }}

## Success Criteria
- [ ] {{ criteria1 }}
- [ ] {{ criteria2 }}

## Timeline
- Planning: {{ duration }}
- Execution: {{ duration }}
- Validation: {{ duration }}

## Risk Assessment
| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| {{ risk }} | {{ prob }} | {{ impact }} | {{ mitigation }} |

## Related Documentation
- [Related Process 1](#)
- [Related Process 2](#)

## Change History
{{ history of changes to this process }}
```

---

## 📦 EXEMPLO 8: Localization Setup

**Arquivo:** `locales/setup.json`

```json
{
  "locales": [
    {
      "code": "en",
      "name": "English",
      "isDefault": true,
      "isRTL": false,
      "namespacePrefix": "en/"
    },
    {
      "code": "pt-br",
      "name": "Português (Brasil)",
      "isDefault": false,
      "isRTL": false,
      "namespacePrefix": "pt-br/"
    },
    {
      "code": "es",
      "name": "Español",
      "isDefault": false,
      "isRTL": false,
      "namespacePrefix": "es/"
    },
    {
      "code": "fr",
      "name": "Français",
      "isDefault": false,
      "isRTL": false,
      "namespacePrefix": "fr/"
    }
  ],
  "translationThreshold": 80
}
```

**Estrutura de Pages:**

```
docs/
├── getting-started/
│   ├── index.md (English - default)
│   ├── [pt-br]/index.md (Portuguese)
│   ├── [es]/index.md (Spanish)
│   └── [fr]/index.md (French)
│
└── guides/
    ├── api-guide.md (English)
    ├── [pt-br]/api-guide.md
    ├── [es]/api-guide.md
    └── [fr]/api-guide.md
```

---

## RESUMO CATEGORIAS 1-3

✅ **Instalação:**
- Docker Compose (rápido)
- Kubernetes + Helm (enterprise)
- Bare Metal (performance)

✅ **GitLab Integration:**
- OAuth2 auth
- Bi-directional Git sync
- CI/CD validation
- Auto-commit on save

✅ **Organização:**
- Information Architecture
- Page Templates
- Localization setup
- Navigation structure
- Taxonomia clara

Próximas: Workflows, Segurança, Customização, Content Management, Performance, Automação, Operações
