

# ✅ **iMac 2013 (macOS 10.15) — Perfect Guide for Installing Dify**

### *dify-deployment-on-macos-10.15*

## **Introduction**

Due to macOS 10.15 system limitations and compatibility issues with Docker 4.15
(such as unsupported latest Compose syntax, port 80 bridge bug, Nginx freeze, OpenDAL storage configuration issues, and the need for manual database initialization),
this guide is specifically designed and tailored for **macOS 10.15 + Docker 4.15 hardware environments**.

It allows older macOS devices to successfully run the Dify AI Knowledge Base, even though the official default installation fails on older Mac systems.

You only need to copy and run the commands from top to bottom — **no need to think about the underlying logic.**
(Local deployment with direct connection mode.)

---

## **Requirements**

* **System:** macOS Catalina 10.15.7
* **Docker:** Must lock version to **Docker Desktop 4.15.0**
  *(Do NOT upgrade. Newer versions do not support this OS.)*
* **Installation Path:** Everything will be installed in your **Documents** folder.
  The installation script uses `cd ~/Documents`, so nothing will clutter your Desktop.

Think of this folder as:
👉 the macOS equivalent of *C:\Program Files* on Windows.

**Just leave it untouched. Do not rename or move it.**

---

# 🥇 **Phase 1 — Environment Cleanup (Reset to Zero)**

To prevent previous misconfigurations from causing issues, we start with a full clean-up.

Open **Terminal**, then copy and paste the entire block below:

```bash
# 1. Enter Documents directory
cd ~/Documents

# 2. Stop and remove old containers if exist
if [ -d "dify/docker" ]; then
  cd dify/docker
  docker-compose down --remove-orphans
fi

# 3. Remove old folders (data will be cleared)
cd ~/Documents
rm -rf dify
rm -rf imac  # Clean up folders created by mistake

# 4. Clean Docker cache
docker container prune -f
```

---

# 🥇 **Phase 2 — Download the Official Program**

Run the following in Terminal:

```bash
cd ~/Documents

# Clone official repository
git clone https://github.com/langgenius/dify.git

# Enter docker folder
cd dify/docker

# Copy base environment file
cp .env.example .env
```

---

# 🥇 **Phase 3 — Write the “Direct-Connection Edition” Config (Key Step)**

This creates a config optimized for macOS 10.15 + Docker 4.15:

* Avoids Nginx (direct UI access via port 3000)
* Fixes OpenDAL storage issues
* Fixes missing keys preventing API startup
* Adapts URL settings to local access
* Removes unsupported Compose syntax for older Docker

Copy **everything** from `cat` to `EOF`:

```bash
cat > docker-compose.yaml << 'EOF'
version: '3'

x-shared-env: &shared-api-worker-env
  CONSOLE_API_URL: http://localhost:5001
  CONSOLE_WEB_URL: http://localhost:3000
  SERVICE_API_URL: http://localhost:5001
  APP_API_URL: http://localhost:5001
  APP_WEB_URL: http://localhost:3000
  FILES_URL: http://localhost:5001
  DB_USERNAME: postgres
  DB_PASSWORD: difyai123456
  DB_HOST: db_postgres
  DB_PORT: 5432
  DB_DATABASE: dify
  REDIS_HOST: redis
  REDIS_PORT: 6379
  REDIS_PASSWORD: difyai123456
  CELERY_BROKER_URL: redis://:difyai123456@redis:6379/1
  STORAGE_TYPE: local
  OPENDAL_SCHEME: fs
  OPENDAL_FS_ROOT: /app/api/storage
  VECTOR_STORE: weaviate
  WEAVIATE_ENDPOINT: http://weaviate:8080
  WEAVIATE_API_KEY: WVF5YThaHlkYwhGUSmCRgsX3tD5ngdN8pkih
  CODE_EXECUTION_ENDPOINT: http://sandbox:8194
  CODE_EXECUTION_API_KEY: dify-sandbox
  SECRET_KEY: sk-DifyAiDefaultSecretKey123456

services:
  api:
    image: langgenius/dify-api:1.10.1
    restart: always
    environment:
      <<: *shared-api-worker-env
      MODE: api
    depends_on:
      - db_postgres
      - redis
    volumes:
      - ./volumes/app/storage:/app/api/storage
    ports:
      - "5001:5001"
    networks:
      - default

  worker:
    image: langgenius/dify-api:1.10.1
    restart: always
    environment:
      <<: *shared-api-worker-env
      MODE: worker
    depends_on:
      - db_postgres
      - redis
    volumes:
      - ./volumes/app/storage:/app/api/storage
    networks:
      - default

  web:
    image: langgenius/dify-web:1.10.1
    restart: always
    environment:
      CONSOLE_API_URL: http://localhost:5001
      APP_API_URL: http://localhost:5001
      MARKETPLACE_API_URL: https://marketplace.dify.ai
    ports:
      - "3000:3000"

  db_postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_PASSWORD: difyai123456
      POSTGRES_DB: dify
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - ./volumes/db/data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 1s
      timeout: 3s
      retries: 30

  redis:
    image: redis:6-alpine
    restart: always
    volumes:
      - ./volumes/redis/data:/data
    command: redis-server --requirepass difyai123456
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

  weaviate:
    image: semitechnologies/weaviate:1.19.0
    restart: always
    volumes:
      - ./volumes/weaviate:/var/lib/weaviate
    environment:
      PERSISTENCE_DATA_PATH: /var/lib/weaviate
      QUERY_DEFAULTS_LIMIT: 25
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'false'
      DEFAULT_VECTORIZER_MODULE: 'none'
      AUTHENTICATION_APIKEY_ENABLED: 'true'
      AUTHENTICATION_APIKEY_ALLOWED_KEYS: WVF5YThaHlkYwhGUSmCRgsX3tD5ngdN8pkih
      AUTHENTICATION_APIKEY_USERS: hello@dify.ai

  sandbox:
    image: langgenius/dify-sandbox:0.2.1
    restart: always
    environment:
      API_KEY: dify-sandbox
    networks:
      - default

  ssrf_proxy:
    image: ubuntu/squid:latest
    restart: always
    ports:
      - "3128:3128"
    networks:
      - default

networks:
  default:
    driver: bridge
EOF

```

---

### **Why this is necessary**

Old HDD / SATA SSD + old Intel CPUs run database migrations very slowly.
Dify must create hundreds of tables during the first launch.

* On M1/M2/M3/M4 Macs: ~10 seconds
* On an iMac 2013: **3–5 minutes**

During this period the backend keeps failing, causing “Internal Server Error.”

By manually running migrations, we “finish the house interior decoration” once and for all.

---

# 🥇 **Phase 4 — Start Dify**

```bash
docker-compose up -d
```

Wait until you see all containers show **Done** or **Running**.

---

# 🥇 **Phase 5 — Force Database Initialization (Required for Old Macs)**

Run:

```bash
docker exec docker-api-1 flask db upgrade
```

You'll see a large amount of output — wait patiently until it stops.

Then restart API:

```bash
docker restart docker-api-1
```

---

# 🥇 **Phase 6 — Access Dify**

Wait **30–60 seconds**, then open:

👉 **[http://localhost:3000/install](http://localhost:3000/install)**

⚠️ *Always use port 3000.
Do NOT use port 80 or 8080.*

---

## (Optional) Create a Desktop App Shortcut

In Chrome:
`Menu ⋮ → Save & Share → Create Shortcut → Open in Window`

---

# **Daily Tips**

* Dify folder is located at:
  `~/Documents/dify`
  Never delete it — it contains your data.
* If it feels laggy, run:

  ```bash
  cd ~/Documents/dify/docker
  docker-compose stop
  ```


