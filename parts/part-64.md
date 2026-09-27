# บทที่ 64: Docker กับ Lua Applications

## บทนำ

ในยุคของ Cloud-Native Development การ containerize แอปพลิเคชันเป็นสิ่งสำคัญมาก Docker ช่วยให้เราสามารถ package แอปพลิเคชัน Lua พร้อมกับ dependencies ทั้งหมดได้ใน container เดียว ทำให้การ deploy มีความสม่ำเสมอในทุก environment ตั้งแต่ development จนถึง production

บทนี้จะครอบคลุมการใช้ Docker กับ Lua applications โดยเฉพาะกับ OpenResty ซึ่งเป็น nginx-based web platform ที่รัน Lua code ได้อย่างมีประสิทธิภาพ

---

## 64.1 Dockerfile สำหรับ Lua/OpenResty Applications

### 64.1.1 Dockerfile พื้นฐานสำหรับ OpenResty

OpenResty คือ web server ที่รวม nginx กับ LuaJIT เข้าด้วยกัน เหมาะสำหรับการสร้าง high-performance web applications ด้วย Lua

```dockerfile
# ตัวอย่างที่ 1: Dockerfile พื้นฐานสำหรับ OpenResty
FROM openresty/openresty:1.25.3.1-alpine

# ติดตั้ง dependencies เพิ่มเติม
RUN apk add --no-cache \
    lua5.1 \
    lua5.1-dev \
    luarocks \
    curl \
    git

# ติดตั้ง Lua packages ผ่าน LuaRocks
RUN luarocks install lua-resty-http && \
    luarocks install lua-resty-redis && \
    luarocks install inspect

# กำหนด working directory
WORKDIR /app

# คัดลอก nginx configuration
COPY nginx.conf /usr/local/openresty/nginx/conf/nginx.conf

# คัดลอก Lua application code
COPY lua/ /app/lua/
COPY html/ /usr/local/openresty/nginx/html/

# เปิด port 80 และ 443
EXPOSE 80 443

# รัน OpenResty
CMD ["/usr/local/openresty/nginx/sbin/nginx", "-g", "daemon off;"]
```

### 64.1.2 nginx.conf สำหรับ OpenResty

```nginx
# ตัวอย่างที่ 2: nginx.conf สำหรับ OpenResty Application
worker_processes auto;
error_log /var/log/openresty/error.log warn;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    # Lua package path
    lua_package_path '/app/lua/?.lua;/usr/local/openresty/lualib/?.lua;;';
    lua_package_cpath '/usr/local/openresty/lualib/?.so;;';

    # Lua code cache (เปิดใน production)
    lua_code_cache on;

    # Shared memory zones
    lua_shared_dict my_cache 10m;
    lua_shared_dict rate_limit 5m;

    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';

    access_log /var/log/openresty/access.log main;

    server {
        listen 80;
        server_name localhost;

        # Health check endpoint
        location /health {
            access_log off;
            content_by_lua_block {
                ngx.header.content_type = 'application/json'
                ngx.say('{"status":"ok","service":"lua-app"}')
            }
        }

        # Main API endpoint
        location /api/ {
            content_by_lua_file /app/lua/api.lua;
        }

        # Static files
        location / {
            root /usr/local/openresty/nginx/html;
            index index.html;
        }
    }
}
```

### 64.1.3 Lua Application Code

```lua
-- ตัวอย่างที่ 3: /app/lua/api.lua - Main API handler
local cjson = require "cjson"
local redis = require "resty.redis"

local M = {}

-- ดึง request method และ path
local method = ngx.req.get_method()
local uri = ngx.var.uri

-- Router table
local routes = {
    ["/api/users"] = {
        GET = function()
            -- ดึงข้อมูลจาก Redis cache
            local red = redis:new()
            red:set_timeouts(1000, 1000, 1000)
            
            local ok, err = red:connect("redis", 6379)
            if not ok then
                ngx.log(ngx.ERR, "Cannot connect to Redis: ", err)
                ngx.status = 500
                ngx.say(cjson.encode({error = "Database connection failed"}))
                return
            end
            
            local data, err = red:get("users:list")
            if data == ngx.null or not data then
                data = cjson.encode({
                    users = {
                        {id = 1, name = "สมชาย", email = "somchai@example.com"},
                        {id = 2, name = "สมหญิง", email = "somying@example.com"}
                    }
                })
                -- Cache เป็นเวลา 60 วินาที
                red:setex("users:list", 60, data)
            end
            
            ngx.header.content_type = "application/json"
            ngx.say(data)
        end,
        POST = function()
            ngx.req.read_body()
            local body = ngx.req.get_body_data()
            local ok, data = pcall(cjson.decode, body)
            
            if not ok then
                ngx.status = 400
                ngx.say(cjson.encode({error = "Invalid JSON"}))
                return
            end
            
            -- Validate ข้อมูล
            if not data.name or not data.email then
                ngx.status = 422
                ngx.say(cjson.encode({error = "name and email are required"}))
                return
            end
            
            ngx.status = 201
            ngx.header.content_type = "application/json"
            ngx.say(cjson.encode({
                message = "User created",
                user = data
            }))
        end
    }
}

-- Route handler
local route = routes[uri]
if route then
    local handler = route[method]
    if handler then
        handler()
    else
        ngx.status = 405
        ngx.say(cjson.encode({error = "Method not allowed"}))
    end
else
    ngx.status = 404
    ngx.say(cjson.encode({error = "Not found"}))
end
```

---

## 64.2 Multi-Stage Builds

Multi-stage builds ช่วยลดขนาด Docker image สุดท้ายโดยแยก build environment ออกจาก runtime environment

### 64.2.1 Multi-Stage Dockerfile สำหรับ Lua Application

```dockerfile
# ตัวอย่างที่ 4: Multi-Stage Dockerfile
# Stage 1: Builder - ติดตั้ง dependencies และ build artifacts
FROM openresty/openresty:1.25.3.1-alpine AS builder

WORKDIR /build

# ติดตั้ง build tools
RUN apk add --no-cache \
    build-base \
    lua5.1-dev \
    luarocks \
    git \
    openssl-dev \
    libffi-dev

# คัดลอก rockspec file
COPY *.rockspec ./
COPY .luarocks/ ./.luarocks/

# ติดตั้ง Lua dependencies
RUN luarocks install --tree /build/lua_modules lua-resty-http && \
    luarocks install --tree /build/lua_modules lua-resty-jwt && \
    luarocks install --tree /build/lua_modules lua-resty-validation && \
    luarocks install --tree /build/lua_modules inspect

# Stage 2: Test runner - รัน tests ก่อน build
FROM builder AS tester

COPY . .

# ติดตั้ง test dependencies
RUN luarocks install --tree /build/lua_modules busted && \
    luarocks install --tree /build/lua_modules luacov

# รัน unit tests
RUN cd /build && \
    lua_modules/.bin/busted spec/ --coverage && \
    lua lua_modules/.bin/luacov

# Stage 3: Production image - เฉพาะสิ่งที่จำเป็น
FROM openresty/openresty:1.25.3.1-alpine AS production

LABEL maintainer="team@example.com"
LABEL version="1.0.0"
LABEL description="Lua/OpenResty Production Application"

# ติดตั้ง runtime dependencies เท่านั้น
RUN apk add --no-cache \
    libssl1.1 \
    pcre \
    tzdata && \
    cp /usr/share/zoneinfo/Asia/Bangkok /etc/localtime && \
    echo "Asia/Bangkok" > /etc/timezone

WORKDIR /app

# คัดลอก Lua modules จาก builder stage
COPY --from=builder /build/lua_modules /app/lua_modules

# คัดลอก application code
COPY --chown=nobody:nobody nginx.conf /usr/local/openresty/nginx/conf/nginx.conf
COPY --chown=nobody:nobody lua/ /app/lua/
COPY --chown=nobody:nobody html/ /usr/local/openresty/nginx/html/
COPY --chown=nobody:nobody config/ /app/config/

# สร้าง directories ที่จำเป็น
RUN mkdir -p /var/log/openresty && \
    chown -R nobody:nobody /var/log/openresty && \
    mkdir -p /var/run/openresty && \
    chown -R nobody:nobody /var/run/openresty

# Security: รันด้วย user ที่ไม่ใช่ root
USER nobody

EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

CMD ["/usr/local/openresty/nginx/sbin/nginx", "-g", "daemon off;"]
```

### 64.2.2 .dockerignore ลดขนาด Context

```dockerfile
# ตัวอย่างที่ 5: .dockerignore file
# Version control
.git
.gitignore
.gitattributes

# Documentation
*.md
docs/
README*

# Tests (ไม่รวมใน production image)
spec/
test/
*.spec.lua

# Development files
.env.local
.env.development
docker-compose.dev.yml

# Editor files
.vscode/
.idea/
*.swp
*.swo

# Coverage reports
luacov.report.out
luacov.stats.out

# CI/CD files
.github/
.travis.yml
Jenkinsfile

# Temporary files
tmp/
*.tmp

# Lua compiled files (ถ้ามี)
*.lc
```

---

## 64.3 Docker Compose สำหรับ Lua Services

Docker Compose ช่วยจัดการ multi-container applications ได้อย่างง่ายดาย

### 64.3.1 docker-compose.yml พื้นฐาน

```yaml
# ตัวอย่างที่ 6: docker-compose.yml สำหรับ Lua Application Stack
version: '3.8'

services:
  # OpenResty/Lua Application
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    image: myapp/lua-app:latest
    container_name: lua-app
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      - APP_ENV=production
      - LOG_LEVEL=warn
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=myapp
      - DB_USER=appuser
      - DB_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
      - jwt_secret
    volumes:
      - app_logs:/var/log/openresty
      - ./config:/app/config:ro
      - ssl_certs:/etc/ssl/certs:ro
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: lua-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: lua-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Nginx Reverse Proxy (สำหรับ SSL termination)
  nginx-proxy:
    image: nginx:alpine
    container_name: lua-proxy
    restart: unless-stopped
    ports:
      - "443:443"
    volumes:
      - ./nginx/proxy.conf:/etc/nginx/nginx.conf:ro
      - ssl_certs:/etc/ssl/certs:ro
    depends_on:
      - app
    networks:
      - frontend

# Networks
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true

# Volumes
volumes:
  app_logs:
    driver: local
  redis_data:
    driver: local
  postgres_data:
    driver: local
  ssl_certs:
    driver: local

# Docker Secrets
secrets:
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
```

### 64.3.2 docker-compose.dev.yml สำหรับ Development

```yaml
# ตัวอย่างที่ 7: docker-compose.dev.yml - Development override
version: '3.8'

services:
  app:
    build:
      target: builder    # ใช้ builder stage แทน production
    ports:
      - "8080:80"       # Port ต่างกันเพื่อไม่ขัดแย้ง
    environment:
      - APP_ENV=development
      - LOG_LEVEL=debug
      - LUA_CODE_CACHE=off    # ปิด cache เพื่อ reload ได้ทันที
    volumes:
      # Mount source code เพื่อ hot reload
      - ./lua:/app/lua
      - ./nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf
      - ./html:/usr/local/openresty/nginx/html
    command: >
      sh -c "while true; do 
        openresty -t && openresty -s reload; 
        inotifywait -r -e modify,create,delete /app/lua /usr/local/openresty/nginx/conf;
      done"

  # Development tools container
  dev-tools:
    image: openresty/openresty:1.25.3.1-alpine
    container_name: lua-dev-tools
    volumes:
      - .:/workspace
    working_dir: /workspace
    command: sleep infinity
    profiles:
      - dev
```

---

## 64.4 Environment Variables และ Configuration Management

### 64.4.1 การจัดการ Configuration ด้วย Environment Variables

```lua
-- ตัวอย่างที่ 8: /app/lua/config.lua - Configuration management
local os = require "os"
local cjson = require "cjson"

local Config = {}

-- Default values
local defaults = {
    app_env = "development",
    log_level = "info",
    redis_host = "localhost",
    redis_port = "6379",
    redis_db = "0",
    db_host = "localhost",
    db_port = "5432",
    db_name = "myapp",
    db_pool_size = "10",
    request_timeout = "30",
    max_body_size = "1048576",  -- 1MB
    jwt_expiry = "3600",        -- 1 hour
    rate_limit_rps = "100"      -- requests per second
}

-- อ่านค่าจาก Environment Variable พร้อม default value
function Config.get(key, default_value)
    local env_key = key:upper()
    local value = os.getenv(env_key)
    
    if value == nil or value == "" then
        return default_value or defaults[key]
    end
    
    return value
end

-- แปลงค่าเป็น number
function Config.get_number(key, default_value)
    local value = Config.get(key, tostring(default_value or 0))
    return tonumber(value) or default_value or 0
end

-- แปลงค่าเป็น boolean
function Config.get_bool(key, default_value)
    local value = Config.get(key)
    if value == nil then return default_value end
    return value:lower() == "true" or value == "1"
end

-- อ่าน secret จาก file (Docker secrets)
function Config.get_secret(key)
    local file_path = os.getenv(key .. "_FILE")
    if file_path then
        local file = io.open(file_path, "r")
        if file then
            local content = file:read("*all")
            file:close()
            return content:gsub("%s+$", "")  -- trim whitespace
        end
    end
    return os.getenv(key)
end

-- Export configuration object
Config.app = {
    env = Config.get("app_env"),
    is_production = Config.get("app_env") == "production",
    is_development = Config.get("app_env") == "development",
    log_level = Config.get("log_level"),
}

Config.redis = {
    host = Config.get("redis_host"),
    port = Config.get_number("redis_port", 6379),
    db = Config.get_number("redis_db", 0),
    password = Config.get_secret("redis_password"),
    timeout = Config.get_number("redis_timeout", 1000),
    pool_size = Config.get_number("redis_pool_size", 10),
}

Config.db = {
    host = Config.get("db_host"),
    port = Config.get_number("db_port", 5432),
    name = Config.get("db_name"),
    user = Config.get("db_user"),
    password = Config.get_secret("db_password"),
    pool_size = Config.get_number("db_pool_size", 10),
}

Config.jwt = {
    secret = Config.get_secret("jwt_secret"),
    expiry = Config.get_number("jwt_expiry", 3600),
}

Config.rate_limit = {
    rps = Config.get_number("rate_limit_rps", 100),
    burst = Config.get_number("rate_limit_burst", 200),
}

return Config
```

### 64.4.2 .env file สำหรับ Docker Compose

```bash
# ตัวอย่างที่ 9: .env file สำหรับ Docker Compose
# Application Settings
APP_ENV=production
LOG_LEVEL=warn
APP_VERSION=1.0.0

# Redis Configuration
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=your_redis_password_here
REDIS_POOL_SIZE=20

# Database Configuration
DB_HOST=postgres
DB_PORT=5432
DB_NAME=myapp
DB_USER=appuser
# DB_PASSWORD ใช้ Docker secrets แทน

# JWT Configuration
JWT_EXPIRY=3600

# Rate Limiting
RATE_LIMIT_RPS=100
RATE_LIMIT_BURST=200

# OpenResty Performance
NGINX_WORKER_PROCESSES=auto
NGINX_WORKER_CONNECTIONS=1024

# Timezone
TZ=Asia/Bangkok
```

---

## 64.5 Health Check Endpoints

Health check เป็นสิ่งสำคัญสำหรับ container orchestration เพื่อให้ระบบรู้ว่า container พร้อมให้บริการหรือไม่

### 64.5.1 Health Check Endpoint แบบละเอียด

```lua
-- ตัวอย่างที่ 10: /app/lua/health.lua - Comprehensive health check
local cjson = require "cjson"
local redis = require "resty.redis"

local function check_redis()
    local red = redis:new()
    red:set_timeouts(500, 500, 500)
    
    local ok, err = red:connect("redis", 6379)
    if not ok then
        return false, "Cannot connect: " .. (err or "unknown")
    end
    
    local pong, err = red:ping()
    red:set_keepalive(10000, 100)
    
    if pong == "PONG" then
        return true, "ok"
    end
    return false, err or "No PONG response"
end

local function check_disk_space()
    local handle = io.popen("df -h /app 2>&1 | tail -1 | awk '{print $5}'")
    if not handle then
        return true, "unknown"  -- ถ้าไม่สามารถตรวจสอบได้ ถือว่า ok
    end
    
    local usage = handle:read("*l")
    handle:close()
    
    if usage then
        local percent = tonumber(usage:match("(%d+)%%"))
        if percent and percent > 90 then
            return false, "Disk usage critical: " .. usage
        end
    end
    return true, usage or "ok"
end

-- ดึงข้อมูล system stats จาก shared memory
local function get_stats()
    local cache = ngx.shared.my_cache
    if not cache then return {} end
    
    return {
        requests_total = cache:get("stats:requests_total") or 0,
        errors_total = cache:get("stats:errors_total") or 0,
        uptime_seconds = ngx.now() - (cache:get("stats:start_time") or ngx.now())
    }
end

-- Main health check handler
local health = {
    status = "ok",
    timestamp = ngx.now(),
    version = os.getenv("APP_VERSION") or "unknown",
    checks = {}
}

-- ตรวจสอบ Redis
local redis_ok, redis_msg = check_redis()
health.checks.redis = {
    status = redis_ok and "ok" or "error",
    message = redis_msg
}
if not redis_ok then health.status = "degraded" end

-- ตรวจสอบ disk space
local disk_ok, disk_msg = check_disk_space()
health.checks.disk = {
    status = disk_ok and "ok" or "warning",
    message = disk_msg
}

-- เพิ่ม stats
health.stats = get_stats()

-- กำหนด HTTP status code
if health.status == "ok" then
    ngx.status = 200
elseif health.status == "degraded" then
    ngx.status = 503
else
    ngx.status = 500
end

ngx.header.content_type = "application/json"
ngx.say(cjson.encode(health))
```

### 64.5.2 Readiness vs Liveness Probes

```lua
-- ตัวอย่างที่ 11: /app/lua/probes.lua - Kubernetes probes
local cjson = require "cjson"

local uri = ngx.var.uri

-- Liveness probe: แอปยังทำงานอยู่ไหม?
if uri == "/health/live" then
    ngx.header.content_type = "application/json"
    ngx.status = 200
    ngx.say(cjson.encode({status = "alive"}))
    return
end

-- Readiness probe: แอปพร้อมรับ traffic ไหม?
if uri == "/health/ready" then
    -- ตรวจสอบว่า dependencies พร้อมไหม
    local ready = true
    local reasons = {}
    
    -- ตรวจสอบ cache warm-up
    local cache = ngx.shared.my_cache
    if cache then
        local warmed = cache:get("cache:warmed")
        if not warmed then
            ready = false
            table.insert(reasons, "Cache not warmed up")
        end
    end
    
    ngx.header.content_type = "application/json"
    
    if ready then
        ngx.status = 200
        ngx.say(cjson.encode({status = "ready"}))
    else
        ngx.status = 503
        ngx.say(cjson.encode({
            status = "not_ready",
            reasons = reasons
        }))
    end
    return
end

-- Startup probe: แอป initialize เสร็จแล้วไหม?
if uri == "/health/startup" then
    local startup_complete = ngx.shared.my_cache:get("startup:complete")
    
    ngx.header.content_type = "application/json"
    if startup_complete then
        ngx.status = 200
        ngx.say(cjson.encode({status = "started"}))
    else
        ngx.status = 503
        ngx.say(cjson.encode({status = "starting"}))
    end
    return
end
```

---

## 64.6 Container Networking

### 64.6.1 Docker Network Configuration

```yaml
# ตัวอย่างที่ 12: docker-compose.yml with advanced networking
version: '3.8'

services:
  app:
    networks:
      frontend:
        aliases:
          - lua-app
          - api
      backend:
        aliases:
          - app-internal
    # กำหนด DNS search domains
    dns:
      - 8.8.8.8
      - 8.8.4.4
    dns_search:
      - internal.example.com

  redis:
    networks:
      backend:
        ipv4_address: 172.20.0.10  # กำหนด IP แบบ static
        aliases:
          - cache
          - redis-primary

  postgres:
    networks:
      backend:
        ipv4_address: 172.20.0.20
        aliases:
          - database
          - db

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.19.0.0/16
    driver_opts:
      com.docker.network.bridge.name: lua-frontend

  backend:
    driver: bridge
    internal: true    # ไม่มี internet access
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/16
    driver_opts:
      com.docker.network.bridge.name: lua-backend
```

### 64.6.2 Inter-Service Communication ใน Lua

```lua
-- ตัวอย่างที่ 13: /app/lua/service_client.lua - HTTP client สำหรับ inter-service communication
local http = require "resty.http"
local cjson = require "cjson"

local ServiceClient = {}

-- Base URLs ของ services ต่าง ๆ (ใช้ Docker DNS)
local services = {
    auth    = "http://auth-service:3000",
    user    = "http://user-service:3001",
    payment = "http://payment-service:3002",
    email   = "http://email-service:3003"
}

function ServiceClient.call(service_name, method, path, options)
    local base_url = services[service_name]
    if not base_url then
        return nil, "Unknown service: " .. service_name
    end
    
    local httpc = http.new()
    httpc:set_timeouts(
        (options and options.connect_timeout) or 2000,
        (options and options.send_timeout) or 5000,
        (options and options.recv_timeout) or 10000
    )
    
    local headers = {
        ["Content-Type"] = "application/json",
        ["X-Request-ID"] = ngx.var.request_id or "",
        ["X-Service-Name"] = "lua-app"
    }
    
    -- เพิ่ม headers จาก options
    if options and options.headers then
        for k, v in pairs(options.headers) do
            headers[k] = v
        end
    end
    
    local body = nil
    if options and options.body then
        if type(options.body) == "table" then
            body = cjson.encode(options.body)
        else
            body = options.body
        end
    end
    
    local res, err = httpc:request_uri(base_url .. path, {
        method = method,
        headers = headers,
        body = body,
        ssl_verify = false,
        keepalive_timeout = 60000,
        keepalive_pool = 10
    })
    
    if not res then
        ngx.log(ngx.ERR, "Service call failed [", service_name, "]: ", err)
        return nil, err
    end
    
    -- Parse JSON response
    if res.headers["content-type"] and 
       res.headers["content-type"]:find("application/json") then
        local ok, data = pcall(cjson.decode, res.body)
        if ok then
            return data, nil, res.status
        end
    end
    
    return res.body, nil, res.status
end

-- Convenience methods
function ServiceClient.get(service, path, options)
    return ServiceClient.call(service, "GET", path, options)
end

function ServiceClient.post(service, path, body, options)
    options = options or {}
    options.body = body
    return ServiceClient.call(service, "POST", path, options)
end

return ServiceClient
```

---

## 64.7 Volume Management สำหรับ Lua Apps

### 64.7.1 Volume Strategies

```yaml
# ตัวอย่างที่ 14: Volume management สำหรับ Lua applications
version: '3.8'

services:
  app:
    volumes:
      # Named volumes สำหรับ persistent data
      - app_logs:/var/log/openresty
      - app_uploads:/app/uploads
      
      # Bind mounts สำหรับ configuration (read-only)
      - ./config/production.lua:/app/config/env.lua:ro
      - ./ssl:/etc/ssl/app:ro
      
      # tmpfs สำหรับ temporary files (เร็วและไม่ persist)
      - type: tmpfs
        target: /tmp/app
        tmpfs:
          size: 104857600  # 100MB

  # Nginx สำหรับ serve static files จาก volume
  nginx:
    volumes:
      - app_uploads:/usr/share/nginx/html/uploads:ro  # share volume แบบ read-only

volumes:
  app_logs:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /var/log/lua-app  # map ไปยัง host path จริง
  
  app_uploads:
    driver: local
  
  # NFS volume สำหรับ shared storage ใน cluster
  shared_assets:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=nfs-server.internal,nolock,soft,rw"
      device: ":/exports/lua-app-assets"
```

### 64.7.2 File Upload Handler ใน Lua

```lua
-- ตัวอย่างที่ 15: /app/lua/upload.lua - File upload handler
local cjson = require "cjson"

local UPLOAD_DIR = "/app/uploads"
local MAX_FILE_SIZE = 10 * 1024 * 1024  -- 10MB
local ALLOWED_TYPES = {
    ["image/jpeg"] = "jpg",
    ["image/png"] = "png",
    ["image/gif"] = "gif",
    ["application/pdf"] = "pdf"
}

local function generate_filename(content_type)
    local ext = ALLOWED_TYPES[content_type] or "bin"
    local timestamp = ngx.now()
    local random = math.random(100000, 999999)
    return string.format("%d_%d.%s", timestamp, random, ext)
end

local function save_file(data, filename)
    local path = UPLOAD_DIR .. "/" .. filename
    local file = io.open(path, "wb")
    if not file then
        return nil, "Cannot open file for writing"
    end
    file:write(data)
    file:close()
    return path
end

-- อ่าน request body
ngx.req.read_body()
local body = ngx.req.get_body_data()

if not body then
    ngx.status = 400
    ngx.say(cjson.encode({error = "No file data"}))
    return
end

-- ตรวจสอบขนาดไฟล์
if #body > MAX_FILE_SIZE then
    ngx.status = 413
    ngx.say(cjson.encode({error = "File too large"}))
    return
end

-- ตรวจสอบ content type
local content_type = ngx.req.get_headers()["content-type"] or ""
-- Extract MIME type จาก content-type header
local mime_type = content_type:match("^([^;]+)")
if mime_type then
    mime_type = mime_type:gsub("%s+", "")
end

if not ALLOWED_TYPES[mime_type] then
    ngx.status = 415
    ngx.say(cjson.encode({
        error = "Unsupported media type",
        allowed = vim.tbl_keys(ALLOWED_TYPES)
    }))
    return
end

local filename = generate_filename(mime_type)
local path, err = save_file(body, filename)

if not path then
    ngx.status = 500
    ngx.say(cjson.encode({error = err}))
    return
end

ngx.status = 201
ngx.header.content_type = "application/json"
ngx.say(cjson.encode({
    success = true,
    filename = filename,
    url = "/uploads/" .. filename,
    size = #body,
    content_type = mime_type
}))
```

---

## 64.8 Kubernetes Basics สำหรับ Lua Services

### 64.8.1 Kubernetes Deployment

```yaml
# ตัวอย่างที่ 16: kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lua-app
  namespace: production
  labels:
    app: lua-app
    version: "1.0.0"
    component: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lua-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0      # Zero downtime deployment
  template:
    metadata:
      labels:
        app: lua-app
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9145"
        prometheus.io/path: "/metrics"
    spec:
      # Pod anti-affinity เพื่อกระจาย pods ไปยัง nodes ต่าง ๆ
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["lua-app"]
              topologyKey: kubernetes.io/hostname

      # ใช้ service account ที่มีสิทธิ์จำกัด
      serviceAccountName: lua-app-sa

      # Init container สำหรับ warm-up
      initContainers:
      - name: wait-for-redis
        image: redis:7-alpine
        command: ['sh', '-c', 'until redis-cli -h redis -p 6379 ping; do echo waiting for redis; sleep 2; done']

      containers:
      - name: lua-app
        image: myregistry.io/lua-app:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 80
          name: http
        
        # Environment variables จาก ConfigMap และ Secrets
        env:
        - name: APP_ENV
          value: "production"
        - name: REDIS_HOST
          value: "redis-service"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: lua-app-config
              key: db_host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: lua-app-secrets
              key: db_password
        
        # Resource limits
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /health/live
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness probe
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Startup probe
        startupProbe:
          httpGet:
            path: /health/startup
            port: 80
          failureThreshold: 30
          periodSeconds: 10
        
        # Volume mounts
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
        - name: logs-volume
          mountPath: /var/log/openresty

      volumes:
      - name: config-volume
        configMap:
          name: lua-app-config
      - name: logs-volume
        emptyDir: {}

      # Image pull secret
      imagePullSecrets:
      - name: registry-credentials
```

### 64.8.2 Kubernetes Service และ Ingress

```yaml
# ตัวอย่างที่ 17: kubernetes/service-ingress.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: lua-app-service
  namespace: production
  labels:
    app: lua-app
spec:
  type: ClusterIP
  selector:
    app: lua-app
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lua-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: lua-app-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: lua-app-service
            port:
              number: 80

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: lua-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: lua-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 64.8.3 ConfigMap และ Secrets

```yaml
# ตัวอย่างที่ 18: kubernetes/configmap-secrets.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: lua-app-config
  namespace: production
data:
  db_host: "postgres-service"
  db_port: "5432"
  db_name: "myapp"
  redis_host: "redis-service"
  redis_port: "6379"
  log_level: "warn"
  rate_limit_rps: "100"
  nginx.conf: |
    worker_processes auto;
    events { worker_connections 1024; }
    http {
      lua_package_path '/app/lua/?.lua;;';
      server {
        listen 80;
        location /health/live { content_by_lua_file /app/lua/probes.lua; }
        location /api/ { content_by_lua_file /app/lua/api.lua; }
      }
    }

---
apiVersion: v1
kind: Secret
metadata:
  name: lua-app-secrets
  namespace: production
type: Opaque
# ค่าเหล่านี้ต้อง base64 encoded
# echo -n "your_password" | base64
stringData:
  db_password: "your_db_password_here"
  redis_password: "your_redis_password_here"
  jwt_secret: "your_jwt_secret_here_minimum_32_chars"
```

---

## 64.9 Service Mesh Concepts

### 64.9.1 Istio Service Mesh กับ Lua Services

```yaml
# ตัวอย่างที่ 19: kubernetes/istio-config.yaml - Service mesh configuration
---
# Virtual Service สำหรับ traffic routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: lua-app-vs
  namespace: production
spec:
  hosts:
  - lua-app-service
  - api.example.com
  gateways:
  - lua-app-gateway
  http:
  # Canary deployment: 10% traffic ไป v2
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: lua-app-service
        subset: v2
  
  # Traffic splitting
  - route:
    - destination:
        host: lua-app-service
        subset: v1
      weight: 90
    - destination:
        host: lua-app-service
        subset: v2
      weight: 10
    
    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 5s
      retryOn: "5xx,reset,connect-failure"
    
    # Timeout
    timeout: 30s
    
    # Fault injection (สำหรับ testing)
    fault:
      delay:
        percentage:
          value: 0.1  # 0.1% requests ที่จะได้รับ delay
        fixedDelay: 5s

---
# Destination Rule สำหรับ load balancing และ circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: lua-app-dr
  namespace: production
spec:
  host: lua-app-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN  # Least connection load balancing
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
    # Circuit breaker
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: "1.0.0"
  - name: v2
    labels:
      version: "2.0.0"
```

### 64.9.2 Circuit Breaker Pattern ใน Lua

```lua
-- ตัวอย่างที่ 20: /app/lua/circuit_breaker.lua - Circuit breaker implementation
local cjson = require "cjson"

local CircuitBreaker = {}
CircuitBreaker.__index = CircuitBreaker

-- States
local CLOSED = "closed"      -- ทำงานปกติ
local OPEN = "open"          -- หยุดรับ request
local HALF_OPEN = "half_open" -- ทดสอบว่าระบบกลับมาแล้วหรือยัง

function CircuitBreaker.new(name, options)
    local cb = setmetatable({}, CircuitBreaker)
    cb.name = name
    cb.threshold = options.threshold or 5        -- จำนวน failures ก่อน open
    cb.timeout = options.timeout or 60           -- วินาทีก่อนลอง half-open
    cb.half_open_max = options.half_open_max or 3
    
    -- ใช้ shared memory สำหรับ state ที่ share ระหว่าง workers
    cb.cache = ngx.shared.circuit_breakers or ngx.shared.my_cache
    
    return cb
end

function CircuitBreaker:get_state()
    local state = self.cache:get("cb:" .. self.name .. ":state") or CLOSED
    
    if state == OPEN then
        local opened_at = self.cache:get("cb:" .. self.name .. ":opened_at") or 0
        if ngx.now() - opened_at > self.timeout then
            -- เปลี่ยนเป็น HALF_OPEN หลังจาก timeout
            self.cache:set("cb:" .. self.name .. ":state", HALF_OPEN)
            self.cache:set("cb:" .. self.name .. ":half_open_count", 0)
            return HALF_OPEN
        end
    end
    
    return state
end

function CircuitBreaker:call(fn)
    local state = self:get_state()
    
    if state == OPEN then
        return nil, "Circuit breaker is OPEN"
    end
    
    -- Execute the function
    local ok, result = pcall(fn)
    
    if ok then
        -- Success: reset failure count
        if state == HALF_OPEN then
            -- กลับมา CLOSED หลังจาก success ใน HALF_OPEN
            self.cache:set("cb:" .. self.name .. ":state", CLOSED)
            self.cache:set("cb:" .. self.name .. ":failures", 0)
            ngx.log(ngx.INFO, "Circuit breaker [", self.name, "] closed")
        else
            self.cache:set("cb:" .. self.name .. ":failures", 0)
        end
        return result
    else
        -- Failure: increment counter
        local failures = (self.cache:get("cb:" .. self.name .. ":failures") or 0) + 1
        self.cache:set("cb:" .. self.name .. ":failures", failures)
        
        ngx.log(ngx.WARN, "Circuit breaker [", self.name, "] failure #", failures)
        
        if failures >= self.threshold then
            self.cache:set("cb:" .. self.name .. ":state", OPEN)
            self.cache:set("cb:" .. self.name .. ":opened_at", ngx.now())
            ngx.log(ngx.ERR, "Circuit breaker [", self.name, "] OPENED after ", failures, " failures")
        end
        
        return nil, result
    end
end

return CircuitBreaker
```

---

## 64.10 Production Deployment Checklist

### 64.10.1 Pre-Deployment Checklist

```lua
-- ตัวอย่างที่ 21: /app/lua/startup_checks.lua - Production startup checks
local config = require "config"
local redis = require "resty.redis"
local cjson = require "cjson"

local checks = {}
local all_passed = true

-- Check 1: ตรวจสอบ environment variables ที่จำเป็น
local required_env = {
    "APP_ENV",
    "REDIS_HOST", 
    "DB_HOST",
    "DB_NAME",
    "DB_USER"
}

for _, env_name in ipairs(required_env) do
    local value = os.getenv(env_name)
    if not value or value == "" then
        checks[env_name] = {passed = false, message = "Missing required env var"}
        all_passed = false
    else
        checks[env_name] = {passed = true, message = "OK"}
    end
end

-- Check 2: ตรวจสอบ secrets
local required_secrets = {"db_password", "jwt_secret"}
for _, secret_name in ipairs(required_secrets) do
    local secret = config.get_secret(secret_name:upper())
    if not secret or secret == "" then
        checks["secret:" .. secret_name] = {
            passed = false, 
            message = "Missing required secret"
        }
        all_passed = false
    else
        checks["secret:" .. secret_name] = {passed = true, message = "OK"}
    end
end

-- Check 3: ตรวจสอบ JWT secret ความยาวขั้นต่ำ
local jwt_secret = config.get_secret("JWT_SECRET")
if jwt_secret and #jwt_secret < 32 then
    checks["jwt_secret_strength"] = {
        passed = false,
        message = "JWT secret must be at least 32 characters"
    }
    all_passed = false
end

-- ถ้า check ไม่ผ่านใน production ให้หยุดทำงาน
if not all_passed and config.app.is_production then
    ngx.log(ngx.EMERG, "Startup checks failed: ", cjson.encode(checks))
    os.exit(1)
end

-- Mark startup as complete
ngx.shared.my_cache:set("startup:complete", true)
ngx.log(ngx.INFO, "Startup checks passed")
```

### 64.10.2 Docker Production Checklist Script

```bash
# ตัวอย่างที่ 22: scripts/production-checklist.sh - Production deployment checklist
#!/bin/bash
set -e

echo "=== Production Deployment Checklist ==="
PASS=0
FAIL=0

check() {
    local name="$1"
    local cmd="$2"
    
    if eval "$cmd" > /dev/null 2>&1; then
        echo "  [PASS] $name"
        ((PASS++))
    else
        echo "  [FAIL] $name"
        ((FAIL++))
    fi
}

echo ""
echo "1. Docker Image Checks"
check "Image exists" "docker image inspect myapp/lua-app:latest"
check "Image is recent (< 7 days)" "[ $(( $(date +%s) - $(docker image inspect myapp/lua-app:latest --format '{{.Created}}' | xargs date +%s -d) )) -lt 604800 ]"
check "No known vulnerabilities" "docker scout cves myapp/lua-app:latest --exit-code 0"

echo ""
echo "2. Configuration Checks"  
check ".env.production exists" "[ -f .env.production ]"
check "All secrets files exist" "ls secrets/*.txt > /dev/null"
check "SSL certificates exist" "ls ssl/*.pem > /dev/null"
check "No dev passwords in config" "! grep -r 'password123\|admin\|test123' config/"

echo ""
echo "3. Database Checks"
check "Migrations are up to date" "docker-compose run --rm app lua /app/scripts/check-migrations.lua"
check "Database backup is recent" "[ $(find /backups -name 'db_*.sql.gz' -mtime -1 | wc -l) -gt 0 ]"

echo ""
echo "4. Service Health Checks"
check "App responds to health check" "curl -sf http://localhost/health"
check "Redis is accessible" "redis-cli -h redis ping | grep PONG"

echo ""
echo "5. Monitoring Checks"
check "Prometheus metrics endpoint" "curl -sf http://localhost:9145/metrics"
check "Log aggregation configured" "[ -f /etc/fluentd/fluent.conf ]"

echo ""
echo "=== Summary ==="
echo "Passed: $PASS"
echo "Failed: $FAIL"

if [ $FAIL -gt 0 ]; then
    echo "DEPLOYMENT BLOCKED: $FAIL checks failed"
    exit 1
fi

echo "All checks passed! Safe to deploy."
```

### 64.10.3 Docker Swarm Deployment สำหรับ Production

```yaml
# ตัวอย่างที่ 23: docker-stack.yml - Docker Swarm production stack
version: '3.8'

services:
  app:
    image: myregistry.io/lua-app:${APP_VERSION:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        monitor: 30s
        max_failure_ratio: 0.3
      rollback_config:
        parallelism: 1
        delay: 5s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.10'
          memory: 128M
      placement:
        constraints:
          - node.role == worker
          - node.labels.environment == production
    secrets:
      - db_password
      - jwt_secret
    networks:
      - frontend
      - backend

  nginx:
    image: nginx:alpine
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: tcp
        mode: host
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.role == manager
    networks:
      - frontend

networks:
  frontend:
    driver: overlay
    attachable: true
  backend:
    driver: overlay
    internal: true

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true
```

---

## 64.11 Monitoring และ Logging

### 64.11.1 Structured Logging ใน Lua

```lua
-- ตัวอย่างที่ 24: /app/lua/logger.lua - Structured logging
local cjson = require "cjson"

local Logger = {}

local LOG_LEVELS = {
    debug = 0,
    info = 1,
    warn = 2,
    error = 3,
    fatal = 4
}

local current_level = LOG_LEVELS[os.getenv("LOG_LEVEL") or "info"] or 1

function Logger.log(level, message, data)
    local level_num = LOG_LEVELS[level] or 1
    if level_num < current_level then return end
    
    local log_entry = {
        timestamp = ngx.now(),
        level = level:upper(),
        message = message,
        service = "lua-app",
        request_id = ngx.var.request_id,
        remote_addr = ngx.var.remote_addr,
        uri = ngx.var.uri,
        method = ngx.req.get_method and ngx.req.get_method() or nil
    }
    
    if data then
        for k, v in pairs(data) do
            log_entry[k] = v
        end
    end
    
    -- เขียน JSON log ไปยัง stderr (Docker จะ capture ไว้)
    io.stderr:write(cjson.encode(log_entry) .. "\n")
end

-- Convenience methods
function Logger.debug(msg, data) Logger.log("debug", msg, data) end
function Logger.info(msg, data)  Logger.log("info", msg, data) end
function Logger.warn(msg, data)  Logger.log("warn", msg, data) end
function Logger.error(msg, data) Logger.log("error", msg, data) end
function Logger.fatal(msg, data) Logger.log("fatal", msg, data) end

return Logger
```

### 64.11.2 Prometheus Metrics

```lua
-- ตัวอย่างที่ 25: /app/lua/metrics.lua - Prometheus metrics endpoint
local cache = ngx.shared.metrics

-- Helper function สำหรับ increment counter
local function increment(key, value)
    cache:incr(key, value or 1, 0)
end

-- Helper function สำหรับ set gauge
local function set_gauge(key, value)
    cache:set(key, value)
end

-- Export metrics ในรูปแบบ Prometheus text format
local function export_metrics()
    local output = {}
    
    -- HTTP request metrics
    local total = cache:get("http_requests_total") or 0
    table.insert(output, "# HELP http_requests_total Total HTTP requests")
    table.insert(output, "# TYPE http_requests_total counter")
    table.insert(output, string.format("http_requests_total %d", total))
    
    -- Response time histogram
    table.insert(output, "# HELP http_request_duration_seconds HTTP request duration")
    table.insert(output, "# TYPE http_request_duration_seconds histogram")
    
    for _, bucket in ipairs({0.01, 0.05, 0.1, 0.5, 1.0, 5.0}) do
        local count = cache:get("duration_bucket_" .. bucket) or 0
        table.insert(output, string.format(
            'http_request_duration_seconds_bucket{le="%s"} %d',
            bucket, count
        ))
    end
    
    -- Active connections
    local active = cache:get("active_connections") or 0
    table.insert(output, "# HELP active_connections Current active connections")
    table.insert(output, "# TYPE active_connections gauge")
    table.insert(output, string.format("active_connections %d", active))
    
    -- Cache hit rate
    local hits = cache:get("cache_hits") or 0
    local misses = cache:get("cache_misses") or 0
    table.insert(output, "# HELP cache_hit_ratio Cache hit ratio")
    table.insert(output, "# TYPE cache_hit_ratio gauge")
    local ratio = hits + misses > 0 and hits / (hits + misses) or 0
    table.insert(output, string.format("cache_hit_ratio %.4f", ratio))
    
    return table.concat(output, "\n") .. "\n"
end

-- Handle /metrics endpoint
if ngx.var.uri == "/metrics" then
    ngx.header.content_type = "text/plain; version=0.0.4"
    ngx.say(export_metrics())
end

return {
    increment = increment,
    set_gauge = set_gauge
}
```

---

## 64.12 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Containerization
สร้าง Dockerfile สำหรับ Lua web application ที่รับ query parameter `?name=xxx` และตอบกลับ `Hello, {name}!` โดยต้องรวม:
- Multi-stage build
- Health check endpoint ที่ `/health`
- ใช้ non-root user
- `.dockerignore` ที่เหมาะสม

### แบบฝึกหัดที่ 2: Docker Compose Stack
สร้าง docker-compose.yml สำหรับ blog application ที่มี:
- OpenResty สำหรับ API
- Redis สำหรับ session และ cache
- PostgreSQL สำหรับ database
- กำหนด network แยกระหว่าง frontend และ backend
- Health checks สำหรับทุก services

### แบบฝึกหัดที่ 3: Kubernetes Deployment
แปลง docker-compose.yml จากแบบฝึกหัดที่ 2 เป็น Kubernetes manifests ประกอบด้วย:
- Deployment สำหรับ application
- Services
- ConfigMap สำหรับ configuration
- Secrets สำหรับ sensitive data
- HorizontalPodAutoscaler

### แบบฝึกหัดที่ 4: Production Checklist
เพิ่มสิ่งต่อไปนี้ให้ application ของคุณ:
- Structured JSON logging
- Prometheus metrics endpoint
- Circuit breaker สำหรับ Redis connection
- Graceful shutdown handling

### แบบฝึกหัดที่ 5: Service Mesh
ออกแบบ Istio configuration สำหรับ canary deployment โดย:
- Version 1 รับ 90% ของ traffic
- Version 2 รับ 10% ของ traffic
- มี retry policy สำหรับ 5xx errors
- มี circuit breaker ที่ 5 consecutive errors

---

## สรุป

ในบทนี้เราได้เรียนรู้การใช้ Docker กับ Lua/OpenResty applications ครอบคลุม:

- **Dockerfile** สำหรับ OpenResty พร้อม multi-stage builds เพื่อลดขนาด image
- **Docker Compose** สำหรับจัดการ multi-container stack ใน development และ production
- **Environment variables** และ Docker secrets สำหรับ configuration management
- **Health checks** ทั้ง liveness, readiness, และ startup probes
- **Container networking** สำหรับ isolation ระหว่าง services
- **Volume management** สำหรับ persistent data และ configuration
- **Kubernetes** deployment, services, ConfigMaps, Secrets และ HPA
- **Service mesh** ด้วย Istio สำหรับ advanced traffic management
- **Production checklist** เพื่อให้การ deploy มีความปลอดภัยและน่าเชื่อถือ

บทต่อไปเราจะเรียนรู้เกี่ยวกับ CI/CD Pipeline สำหรับ Lua projects โดยใช้ GitHub Actions
