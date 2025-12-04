# dify-deployment-on-macos-10.15
A guide for installing Docker 4.15+ and the latest Dify on macOS 10.15.7.

简介
由于 macOS 10.15 系统限制及 Docker 4.15 的兼容性问题（不支持最新的 Compose 语法、80 端口网络桥接 Bug、Nginx 假死、修复 OpenDAL 存储配置、手动初始化数据库）。
这份文档是专门针对macOS 10.15 + Docker 4.15 硬件环境量身定制编写的。成功复活老macos系统运行 Dify AI 知识库
你只需要从头到尾复制命令执行即可，不需要思考中间的逻辑。（直连模式+本地部署方案）
成功复活老macos系统运行 Dify AI 知识库，解决官方的标准安装步骤在旧款 Mac macOS上会失败。

<img width="1382" height="1368" alt="截屏2025-12-02 上午7 04 31" src="https://github.com/user-attachments/assets/5973795c-3272-4ee6-bb7f-9dddfc7e52d7" />
<img width="1382" height="1368" alt="截屏2025-12-02 上午7 04 38" src="https://github.com/user-attachments/assets/441f05a7-c1e1-4c9a-828d-676b35c62ddc" />
<img width="972" height="1172" alt="截屏2025-12-02 上午7 04 52" src="https://github.com/user-attachments/assets/451f22cf-b35a-4668-ab54-dec943abcd57" />
<img width="2266" height="1275" alt="截屏2025-12-02 上午7 05 12" src="https://github.com/user-attachments/assets/772ef9f3-bc0b-4c84-853e-aea1d5ba32b0" />



✅ 核心前提
系统：macOS Catalina 10.15.7
Docker：必须锁定使用 Docker Desktop 4.15.0（千万不要点击升级，新版不支持你的系统）
安装位置：所有文件将安装在你的 【文稿 (Documents)】 目录下，请勿随意移动或删除。
脚本里写的是 cd ~/Documents，所以它其实是下载到了你的 【文稿】(Documents) 文件夹里，不会弄乱你的桌面。
它的角色：你可以把它理解为安装在 Windows C:\Program Files 里的软件目录。
如何对待：就把它安安静静地放在【文稿】里，不要去动它，也不要改名。


第一阶段：环境清理 (重置到零点)

为了防止之前的错误配置干扰，我们先执行一次彻底的清理。
打开 终端 (Terminal)。

完整复制下面这块代码，粘贴进去并回车

# 1. 进入文档目录
cd ~/Documents

# 2. 如果存在旧容器，停止并移除
if [ -d "dify/docker" ]; then
  cd dify/docker
  docker-compose down --remove-orphans
fi

# 3. 删除旧的代码文件夹 (数据会清空)
cd ~/Documents
rm -rf dify
rm -rf imac  # 清理可能因误操作生成的文件夹

# 4. 清理 Docker 缓存 (输入 y 确认)
docker container prune -f


第二阶段：下载官方程序
这一步从 GitHub 拉取 Dify 的最新代码。

在终端复制并执行：

# 1. 进入文档目录
cd ~/Documents

# 2. 克隆官方代码
git clone https://github.com/langgenius/dify.git

# 3. 进入安装目录
cd dify/docker

# 4. 复制基础环境配置
cp .env.example .env

第三步：写入“直连版”配置文件（核心步骤）
这一步将生成一个专门适配 Docker 4.15 和老款 macOS 的配置文件。 
特点：
避开 Nginx：直接通过 3000 端口访问前端，避开 macOS 10.15 的 80 端口 Bug。
修复了存储配置：防止出现 OpenDAL 报错。
修复了密钥缺失：防止 API 启动失败。
适配了 Web/API 地址：指向本地端口。
适配老版 Docker：移除不兼容的语法。

在终端完整复制下面整块代码（从 cat 开始到 EOF 结束），粘贴并回车：

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


这段代码的用途：
机械硬盘（或者 SATA SSD）加上旧 CPU，在处理 数据库迁移 (Database Migrations) 时会比 M1-M4 慢很多。 
Dify 第一次启动时，需要向数据库里写入几百张表。这个过程在 M4 上可能只要 10 秒，但在 iMac 上可能需要 3-5 分钟。 
在此期间，后端会一直报错或拒绝服务，网页就会转圈。
做的操作（flask db upgrade）相当于手动完成了房子的“装修”（建表）。
装修只要做一次：现在表已经建好了，数据已经永久写入你的硬盘里了（就在 dify/docker/volumes 文件夹下）。
以后不用再做：不管你重启电脑多少次，Dify 启动时一看：“哟，表都在，数据也在”，它就会直接运行，不会再卡住了。


第四步：启动 Dify
执行启动命令：

docker-compose up -d

现象：你会看到一系列 Creating... 或 Starting...。
等待：直到所有行都显示 Done 或 Running，回到命令行提示符。

第五步：强制初始化数据库（老电脑必做）
因为旧Mac/iMac性能有限，自动建表大概率会超时导致网页报错 Internal Server Error 或转圈。我们需要手动执行这一步。

在终端执行：

docker exec docker-api-1 flask db upgrade

现象：你会看到很多代码滚动。
操作：耐心等待，直到滚动停止，回到命令行提示符。
这一步完成后，你的数据库就永久准备好了。

重启一下 API 以确保生效：

在终端执行：

docker restart docker-api-1

第六步：访问与使用
等待 30-60秒 让服务完全启动。

打开 Chrome 或 Safari 浏览器，访问： 👉 http://localhost:3000/install

(注意：以后所有的访问地址都是 端口 3000，不要用 80 或 8080)

制作成 APP (可选，推荐使用 Chrome)：


日常使用小贴士：
开机后：如果你设置了 Docker 开机自启，直接访问 http://localhost:3000 即可。
在 Chrome 中打开上面的网址。
点击右上角菜单 ⋮ -> 保存并分享 -> 创建快捷方式。
勾选【在窗口中打开】 -> 点击创建。
在 Dock 栏右键新出现的图标 -> 在程序坞中保留。

觉得卡顿：不用的时候，可以去终端输入 cd ~/Documents/dify/docker 然后 docker-compose stop 暂停服务。
关于文件夹：~/Documents/dify 是你的程序和数据存放地，千万不要删除。
