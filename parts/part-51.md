# บทที่ 51: OpenResty - Nginx + Lua Web Server

## บทนำ

OpenResty คือ web platform ที่รวม Nginx กับ LuaJIT เข้าด้วยกัน ทำให้สามารถเขียน server-side logic ด้วย Lua ได้โดยตรงภายใน Nginx ซึ่งทำให้ได้ performance สูงมากเพราะไม่ต้องผ่าน process fork หรือ network overhead แบบ CGI/FastCGI

### ข้อดีของ OpenResty
- Performance สูงมาก (non-blocking I/O)
- Lua เป็นภาษาที่ lightweight และเร็ว
- Ecosystem ของ modules มากมาย (lua-resty-*)
- รองรับ cosocket สำหรับ async networking
- ใช้งานร่วมกับ Nginx ได้ทั้งหมด

---

## 51.1 การติดตั้ง OpenResty

```bash
# Ubuntu/Debian
sudo apt-get install -y software-properties-common
sudo add-apt-repository -y "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main"
wget -qO - https://openresty.org/package/pub.gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install openresty

# CentOS/RHEL
sudo yum install yum-utils
sudo yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo
sudo yum install openresty

# macOS (Homebrew)
brew install openresty/brew/openresty

# ตรวจสอบการติดตั้ง
openresty -v
# openresty/1.25.3.1

# รัน OpenResty
sudo openresty
# หรือระบุ config
sudo openresty -c /etc/openresty/nginx.conf
```

---

## 51.2 โครงสร้าง nginx.conf พื้นฐาน

```nginx
# /etc/openresty/nginx.conf หรือ /usr/local/openresty/nginx/conf/nginx.conf

worker_processes  auto;
error_log  logs/error.log;
pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

http {
    # กำหนด lua package path
    lua_package_path '/usr/local/openresty/lualib/?.lua;;';
    lua_package_cpath '/usr/local/openresty/lualib/?.so;;';
    
    # Shared memory dictionary
    lua_shared_dict my_cache 10m;
    lua_shared_dict counters 1m;
    
    server {
        listen 80;
        server_name localhost;
        
        # ตัวอย่าง location พื้นฐาน
        location / {
            content_by_lua_block {
                ngx.say("Hello from OpenResty!")
            }
        }
    }
}
```

---

## 51.3 Nginx Phases และ Lua Directives

OpenResty เพิ่ม Lua directives เข้าไปใน Nginx phases ต่างๆ:

```
1. set_by_lua           - Set phase: กำหนดค่า variables
2. rewrite_by_lua       - Rewrite phase: redirect, rewrite URLs  
3. access_by_lua        - Access phase: authentication, authorization
4. content_by_lua       - Content phase: สร้าง response body (หลักๆ ใช้นี้)
5. header_filter_by_lua - Header filter phase: แก้ไข response headers
6. body_filter_by_lua   - Body filter phase: แก้ไข response body
7. log_by_lua           - Log phase: custom logging
8. init_by_lua          - Initialization phase: ทำงานตอนเริ่ม worker
9. init_worker_by_lua   - Worker initialization
```

```nginx
server {
    listen 8080;
    
    # Phase 1: set_by_lua_block
    location /set-example {
        set_by_lua_block $my_var {
            local val = ngx.var.arg_name or "World"
            return "Hello, " .. val .. "!"
        }
        echo $my_var;
    }
    
    # Phase 2: rewrite_by_lua_block
    location /rewrite-example {
        rewrite_by_lua_block {
            local uri = ngx.var.uri
            if uri == "/rewrite-example" then
                -- redirect ไปที่อื่น
                ngx.redirect("/new-location", 301)
            end
        }
        content_by_lua_block {
            ngx.say("This won't run if redirected")
        }
    }
    
    # Phase 3: access_by_lua_block
    location /protected {
        access_by_lua_block {
            local token = ngx.var.http_authorization
            if not token or token ~= "Bearer secret123" then
                ngx.status = 401
                ngx.say('{"error": "Unauthorized"}')
                ngx.exit(401)
            end
        }
        content_by_lua_block {
            ngx.say("Welcome to protected resource!")
        }
    }
}
```

---

## 51.4 content_by_lua_block - สร้าง Response

```nginx
location /hello {
    content_by_lua_block {
        -- ส่ง response ง่ายๆ
        ngx.say("Hello, World!")
    }
}

location /json {
    content_by_lua_block {
        -- ส่ง JSON response
        ngx.header["Content-Type"] = "application/json"
        local cjson = require "cjson"
        local data = {
            message = "Hello from OpenResty",
            timestamp = ngx.time(),
            version = "1.0"
        }
        ngx.say(cjson.encode(data))
    }
}

location /multi-line {
    content_by_lua_block {
        -- ส่ง response หลาย lines
        ngx.print("<html><body>")
        ngx.print("<h1>Hello from Lua!</h1>")
        ngx.print("<p>Time: " .. ngx.now() .. "</p>")
        ngx.say("</body></html>")
        -- ngx.say() เพิ่ม newline ท้าย, ngx.print() ไม่เพิ่ม
    }
}
```

---

## 51.5 ngx API พื้นฐาน

```lua
-- ตัวอย่างการใช้ ngx API ใน content_by_lua_block

-- ngx.say() และ ngx.print()
ngx.say("Hello")          -- พิมพ์ + newline
ngx.print("World")        -- พิมพ์ไม่มี newline
ngx.say("Line 1", "Line 2", "Line 3")  -- รับหลาย arguments

-- ngx.log() - logging
ngx.log(ngx.ERR, "This is an error")
ngx.log(ngx.WARN, "This is a warning")
ngx.log(ngx.INFO, "This is info")
ngx.log(ngx.DEBUG, "This is debug")
ngx.log(ngx.NOTICE, "Custom: ", ngx.var.remote_addr)

-- ngx.time() - timestamp
local t = ngx.time()           -- Unix timestamp (วินาที)
local t_ms = ngx.now()         -- Timestamp ทศนิยม (milliseconds)
local today = ngx.today()      -- "YYYY-MM-DD"
local http_time = ngx.http_time(t)  -- HTTP date format

-- ngx.exit() - จบ request
ngx.exit(200)    -- จบด้วย status 200
ngx.exit(404)    -- จบด้วย status 404
ngx.exit(ngx.HTTP_OK)
ngx.exit(ngx.HTTP_NOT_FOUND)

-- ngx.redirect() - redirect
ngx.redirect("/new-page")           -- 302 redirect
ngx.redirect("/new-page", 301)     -- 301 redirect

-- ngx.status - set status code
ngx.status = 404
ngx.say("Not Found")
```

---

## 51.6 ngx.var - Nginx Variables

```nginx
location /vars-demo {
    content_by_lua_block {
        -- อ่าน Nginx built-in variables
        ngx.say("URI: "         .. ngx.var.uri)
        ngx.say("Method: "      .. ngx.var.request_method)
        ngx.say("Remote IP: "   .. ngx.var.remote_addr)
        ngx.say("Host: "        .. (ngx.var.http_host or ""))
        ngx.say("User-Agent: "  .. (ngx.var.http_user_agent or ""))
        ngx.say("Query: "       .. (ngx.var.query_string or ""))
        ngx.say("Server Port: " .. ngx.var.server_port)
        ngx.say("Request: "     .. ngx.var.request)
        
        -- Query string parameters
        -- URL: /vars-demo?name=John&age=30
        ngx.say("name: "        .. (ngx.var.arg_name or ""))
        ngx.say("age: "         .. (ngx.var.arg_age or ""))
        
        -- Set custom Nginx variable
        ngx.var.my_custom_var = "custom value"
    }
}
```

```nginx
# ต้องประกาศ variable ก่อนถึงจะ set ได้
location /set-var {
    set $my_custom_var "";   # ประกาศตรงนี้
    content_by_lua_block {
        ngx.var.my_custom_var = "Hello!"
        ngx.say(ngx.var.my_custom_var)
    }
}
```

---

## 51.7 ngx.req - Request Object

```lua
-- ใน content_by_lua_block

-- อ่าน request headers
local headers = ngx.req.get_headers()
for k, v in pairs(headers) do
    ngx.say(k .. ": " .. tostring(v))
end

-- อ่าน header เฉพาะตัว
local content_type = ngx.req.get_headers()["content-type"]
local auth = ngx.req.get_headers()["authorization"]

-- อ่าน query parameters
local args = ngx.req.get_uri_args()
-- URL: /api?name=John&tags=lua&tags=nginx
-- args = {name="John", tags={"lua","nginx"}}

-- อ่าน POST body
ngx.req.read_body()
local body = ngx.req.get_body_data()

-- อ่าน POST form data
local post_args = ngx.req.get_post_args()
-- form: name=John&email=john@example.com
-- post_args = {name="John", email="john@example.com"}

-- Set request headers (สำหรับ proxy)
ngx.req.set_header("X-Custom-Header", "value")
ngx.req.clear_header("User-Agent")

-- HTTP method
local method = ngx.req.get_method()  -- "GET", "POST", etc.

-- Request body file
ngx.req.read_body()
local body_file = ngx.req.get_body_file()
```

---

## 51.8 ngx.header - Response Headers

```nginx
location /headers-demo {
    content_by_lua_block {
        -- Set response headers
        ngx.header["Content-Type"] = "application/json"
        ngx.header["X-Powered-By"] = "OpenResty/Lua"
        ngx.header["Cache-Control"] = "no-cache"
        ngx.header["X-Request-Id"] = ngx.md5(tostring(ngx.now()))
        
        -- Set multiple values (List header)
        ngx.header["Set-Cookie"] = {
            "session=abc123; Path=/; HttpOnly",
            "preferences=dark; Path=/"
        }
        
        -- อ่าน response header ที่ set แล้ว
        local ct = ngx.header["Content-Type"]
        ngx.log(ngx.INFO, "Content-Type: " .. (ct or ""))
        
        -- ลบ header
        ngx.header["X-Unnecessary"] = nil
        
        ngx.say('{"status": "ok"}')
    }
}
```

---

## 51.9 Shared Memory (ngx.shared)

```nginx
http {
    # ประกาศ shared dict ใน http block
    lua_shared_dict page_cache  20m;
    lua_shared_dict rate_limits 5m;
    lua_shared_dict counters    1m;
    
    server {
        listen 8080;
        
        location /cache-demo {
            content_by_lua_block {
                local cache = ngx.shared.page_cache
                local key = "greeting"
                
                -- ลอง get จาก cache ก่อน
                local val = cache:get(key)
                if val then
                    ngx.say("From cache: " .. val)
                    return
                end
                
                -- ไม่มีใน cache, สร้างใหม่
                local new_val = "Hello at " .. ngx.now()
                
                -- เก็บลง cache 10 วินาที
                local ok, err = cache:set(key, new_val, 10)
                if not ok then
                    ngx.log(ngx.ERR, "Failed to cache: " .. err)
                end
                
                ngx.say("Computed: " .. new_val)
            }
        }
        
        location /counter {
            content_by_lua_block {
                local counters = ngx.shared.counters
                
                -- Atomic increment
                local count, err = counters:incr("visits", 1, 0)
                if not count then
                    ngx.log(ngx.ERR, "Counter error: " .. (err or ""))
                    count = 0
                end
                
                ngx.say("Visit #" .. count)
            }
        }
        
        location /rate-limit {
            access_by_lua_block {
                local limits = ngx.shared.rate_limits
                local ip = ngx.var.remote_addr
                local key = "limit:" .. ip
                local limit = 10  -- requests per minute
                
                local count, err = limits:incr(key, 1, 0, 60)  -- TTL = 60s
                if not count then
                    count = 1
                end
                
                if count > limit then
                    ngx.status = 429
                    ngx.header["Retry-After"] = "60"
                    ngx.say('{"error": "Too Many Requests"}')
                    ngx.exit(429)
                end
                
                ngx.header["X-RateLimit-Remaining"] = tostring(limit - count)
            }
            content_by_lua_block {
                ngx.say("Request processed successfully")
            }
        }
    }
}
```

---

## 51.10 Shared Memory - Methods ต่างๆ

```lua
local dict = ngx.shared.my_dict

-- set(key, value, exptime, flags)
dict:set("key1", "value1")          -- ไม่หมดอายุ
dict:set("key2", "value2", 60)      -- หมดอายุใน 60 วินาที
dict:set("key3", 42, 0, 100)        -- value=42, ไม่หมดอายุ, flags=100

-- get(key)
local val, flags = dict:get("key1")

-- add(key, value) - เพิ่มเฉพาะถ้า key ไม่มีอยู่
local ok, err = dict:add("new_key", "new_value")
-- ok=false ถ้า key มีอยู่แล้ว

-- replace(key, value) - แก้ไขเฉพาะถ้า key มีอยู่
local ok, err = dict:replace("existing_key", "updated_value")

-- delete(key)
dict:delete("key1")

-- incr(key, value, init, init_ttl) - atomic increment
local count = dict:incr("counter", 1, 0)     -- เริ่มจาก 0 ถ้าไม่มี
local count = dict:incr("counter", 5, 100)   -- เพิ่ม 5, เริ่มจาก 100

-- flush_all() - ลบทั้งหมด
dict:flush_all()

-- flush_expired() - ลบที่หมดอายุ
local flushed = dict:flush_expired()

-- keys() - ดึง keys ทั้งหมด
local keys = dict:keys()     -- อาจช้าถ้ามีข้อมูลเยอะ
local keys = dict:keys(100)  -- ดึงแค่ 100 keys แรก

-- stats
local capacity, free = dict:capacity(), dict:free_space()
ngx.say("Used: " .. (capacity - free) .. " bytes")
```

---

## 51.11 Worker Processes และ init_worker_by_lua

```nginx
http {
    lua_shared_dict worker_data 1m;
    
    # ทำงานครั้งเดียวตอน Nginx start (master process)
    init_by_lua_block {
        -- โหลด module ล่วงหน้า
        cjson = require "cjson"
        -- กำหนดค่า global
        MY_APP_VERSION = "1.0.0"
        ngx.log(ngx.NOTICE, "App initialized, version: " .. MY_APP_VERSION)
    }
    
    # ทำงานใน worker แต่ละตัวตอนเริ่ม
    init_worker_by_lua_block {
        local worker_id = ngx.worker.id()
        local pid = ngx.worker.pid()
        ngx.log(ngx.NOTICE, "Worker " .. worker_id .. " started, PID: " .. pid)
        
        -- สามารถตั้ง timer ใน worker ได้
        local function background_job(premature)
            if premature then return end
            ngx.log(ngx.INFO, "Background job running in worker " .. ngx.worker.id())
            -- ทำงานอะไรบางอย่าง...
            
            -- ตั้ง timer ครั้งต่อไป
            local ok, err = ngx.timer.at(30, background_job)
            if not ok then
                ngx.log(ngx.ERR, "Timer error: " .. err)
            end
        end
        
        -- เริ่ม background job ทุก 30 วินาที
        local ok, err = ngx.timer.at(0, background_job)
        if not ok then
            ngx.log(ngx.ERR, "Failed to start background job: " .. err)
        end
    }
    
    server {
        listen 8080;
        
        location /worker-info {
            content_by_lua_block {
                ngx.say("Worker ID: " .. ngx.worker.id())
                ngx.say("Worker PID: " .. ngx.worker.pid())
                ngx.say("Worker Count: " .. ngx.worker.count())
            }
        }
    }
}
```

---

## 51.12 cosocket - Async I/O

cosocket ทำให้ Lua code ใน OpenResty สามารถทำ non-blocking network calls ได้

```lua
-- ตัวอย่าง TCP connection
local sock = ngx.socket.tcp()

-- connect to external service
local ok, err = sock:connect("127.0.0.1", 6379)  -- Redis port
if not ok then
    ngx.say("Connect failed: " .. err)
    return
end

-- ส่งและรับข้อมูล
local bytes, err = sock:send("PING\r\n")
if not bytes then
    ngx.say("Send failed: " .. err)
    return
end

local data, err = sock:receive("*l")  -- อ่านทีละบรรทัด
if not data then
    ngx.say("Receive failed: " .. err)
    return
end

ngx.say("Redis response: " .. data)  -- "+PONG"
sock:close()
```

```lua
-- HTTP request ด้วย cosocket
local function http_get(host, path)
    local sock = ngx.socket.tcp()
    
    local ok, err = sock:connect(host, 80)
    if not ok then
        return nil, err
    end
    
    -- ส่ง HTTP request
    local req = "GET " .. path .. " HTTP/1.1\r\n"
               .. "Host: " .. host .. "\r\n"
               .. "Connection: close\r\n"
               .. "\r\n"
    
    sock:send(req)
    
    -- อ่าน response
    local resp = {}
    while true do
        local line, err = sock:receive("*l")
        if not line then break end
        table.insert(resp, line)
    end
    
    sock:close()
    return table.concat(resp, "\n")
end

-- ใช้งาน
local response = http_get("example.com", "/")
ngx.say(response)
```

---

## 51.13 lua-resty-http - HTTP Client

```nginx
location /fetch-api {
    content_by_lua_block {
        local http = require "resty.http"
        local httpc = http.new()
        
        -- Simple GET
        local res, err = httpc:request_uri("https://api.example.com/data", {
            method = "GET",
            headers = {
                ["Authorization"] = "Bearer mytoken",
                ["Accept"] = "application/json"
            },
            ssl_verify = false,
            timeout = 5000  -- 5 seconds
        })
        
        if not res then
            ngx.status = 502
            ngx.say("Upstream error: " .. (err or "unknown"))
            return
        end
        
        -- Forward status
        ngx.status = res.status
        
        -- Forward headers
        for k, v in pairs(res.headers) do
            ngx.header[k] = v
        end
        
        -- Send body
        ngx.say(res.body)
    }
}

location /post-api {
    content_by_lua_block {
        local http = require "resty.http"
        local cjson = require "cjson"
        
        local httpc = http.new()
        
        local payload = cjson.encode({
            name = "test",
            value = 42
        })
        
        local res, err = httpc:request_uri("https://httpbin.org/post", {
            method = "POST",
            headers = {
                ["Content-Type"] = "application/json",
                ["Content-Length"] = #payload
            },
            body = payload
        })
        
        if not res then
            ngx.status = 502
            ngx.say("Error: " .. err)
            return
        end
        
        ngx.status = res.status
        ngx.header["Content-Type"] = "application/json"
        ngx.say(res.body)
    }
}
```

---

## 51.14 access_by_lua_block - Authentication

```nginx
# JWT Authentication Middleware
location /api/ {
    access_by_lua_block {
        local cjson = require "cjson"
        
        -- ดึง token จาก Authorization header
        local auth = ngx.var.http_authorization
        if not auth then
            ngx.status = 401
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({error = "Missing authorization header"}))
            ngx.exit(401)
        end
        
        -- ตรวจสอบ Bearer format
        local token = auth:match("^Bearer%s+(.+)$")
        if not token then
            ngx.status = 401
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({error = "Invalid authorization format"}))
            ngx.exit(401)
        end
        
        -- TODO: ตรวจสอบ JWT token จริงๆ ด้วย lua-resty-jwt
        -- สมมติตรวจสอบง่ายๆ
        if token == "valid-token-123" then
            -- ผ่าน! ตั้งค่า user info ใน header สำหรับ upstream
            ngx.req.set_header("X-User-Id", "user_1")
            ngx.req.set_header("X-User-Role", "admin")
        else
            ngx.status = 401
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({error = "Invalid token"}))
            ngx.exit(401)
        end
    }
    
    content_by_lua_block {
        local user_id = ngx.req.get_headers()["X-User-Id"]
        ngx.say("Hello, user " .. (user_id or "unknown") .. "!")
    }
}
```

---

## 51.15 header_filter_by_lua_block

```nginx
location /api {
    # เพิ่ม headers ใน response ทุกตัวที่ผ่าน location นี้
    header_filter_by_lua_block {
        -- เพิ่ม security headers
        ngx.header["X-Content-Type-Options"] = "nosniff"
        ngx.header["X-Frame-Options"] = "DENY"
        ngx.header["X-XSS-Protection"] = "1; mode=block"
        ngx.header["Strict-Transport-Security"] = "max-age=31536000"
        
        -- เพิ่ม CORS headers
        local origin = ngx.var.http_origin
        if origin then
            ngx.header["Access-Control-Allow-Origin"] = origin
            ngx.header["Access-Control-Allow-Credentials"] = "true"
        end
        
        -- เพิ่ม custom header
        ngx.header["X-Server-Time"] = ngx.http_time(ngx.time())
        ngx.header["X-Request-Id"] = ngx.md5(
            ngx.var.remote_addr .. tostring(ngx.now())
        )
        
        -- ลบ header ที่ไม่ต้องการ
        ngx.header["Server"] = nil
        ngx.header["X-Powered-By"] = nil
    }
    
    content_by_lua_block {
        ngx.say("Response with security headers")
    }
}
```

---

## 51.16 body_filter_by_lua_block

```nginx
location /transform {
    body_filter_by_lua_block {
        -- ngx.arg[1] คือ chunk ของ body
        -- ngx.arg[2] คือ bool ว่าเป็น last chunk ไหม
        
        local chunk = ngx.arg[1]
        local is_last = ngx.arg[2]
        
        if chunk then
            -- แปลง response body (เช่น inject tracking script)
            chunk = chunk:gsub("</body>", 
                '<script>console.log("Tracked!");</script></body>')
            ngx.arg[1] = chunk
        end
        
        -- ถ้า last chunk ก็ไม่ต้องทำอะไรพิเศษ
    }
    
    content_by_lua_block {
        ngx.header["Content-Type"] = "text/html"
        ngx.say("<html><body><h1>Hello</h1></body></html>")
    }
}
```

```nginx
location /compress-demo {
    # กรองและแปลง JSON response
    body_filter_by_lua_block {
        local cjson = require "cjson"
        local chunk = ngx.arg[1]
        local is_last = ngx.arg[2]
        
        -- รวม chunks ด้วย context variable
        if chunk and chunk ~= "" then
            ngx.ctx.body = (ngx.ctx.body or "") .. chunk
            ngx.arg[1] = ""  -- clear chunk (จะส่งใน last chunk)
        end
        
        if is_last then
            local body = ngx.ctx.body or ""
            -- พยายาม parse JSON และ pretty print
            local ok, data = pcall(cjson.decode, body)
            if ok then
                ngx.arg[1] = cjson.encode(data)  -- re-encode (compact)
            else
                ngx.arg[1] = body  -- ส่ง original ถ้า parse ไม่ได้
            end
        end
    }
    
    content_by_lua_block {
        ngx.header["Content-Type"] = "application/json"
        -- ส่ง JSON แบบ pretty (มี spaces)
        ngx.say('{ "name" : "John" , "age" : 30 }')
    }
}
```

---

## 51.17 log_by_lua_block - Custom Logging

```nginx
http {
    lua_shared_dict access_logs 50m;
    
    server {
        listen 8080;
        
        location /api/ {
            log_by_lua_block {
                local cjson = require "cjson"
                
                -- สร้าง structured log
                local log_entry = {
                    timestamp = ngx.now(),
                    method    = ngx.var.request_method,
                    uri       = ngx.var.uri,
                    status    = ngx.status,
                    duration  = tonumber(ngx.var.request_time) * 1000,  -- ms
                    bytes     = ngx.var.bytes_sent,
                    ip        = ngx.var.remote_addr,
                    user_agent = ngx.var.http_user_agent
                }
                
                -- log ลงไฟล์ผ่าน ngx.log
                ngx.log(ngx.INFO, "ACCESS: " .. cjson.encode(log_entry))
                
                -- หรือเก็บลง shared dict
                local logs = ngx.shared.access_logs
                local count = logs:incr("total_requests", 1, 0)
                
                -- เก็บ slow requests
                if log_entry.duration > 1000 then  -- > 1 second
                    ngx.log(ngx.WARN, "SLOW REQUEST: " .. cjson.encode(log_entry))
                end
            }
            
            content_by_lua_block {
                -- simulate some processing time
                ngx.sleep(0.01)  -- 10ms
                ngx.say("OK")
            }
        }
        
        location /stats {
            content_by_lua_block {
                local logs = ngx.shared.access_logs
                ngx.say("Total requests: " .. (logs:get("total_requests") or 0))
            }
        }
    }
}
```

---

## 51.18 Building a Simple HTTP Router

```nginx
http {
    server {
        listen 8080;
        
        location / {
            content_by_lua_block {
                local cjson = require "cjson"
                
                -- Simple router
                local method = ngx.req.get_method()
                local uri = ngx.var.uri
                
                -- Route table
                local routes = {
                    {method="GET",  pattern="^/api/users$",          handler="list_users"},
                    {method="GET",  pattern="^/api/users/(%d+)$",    handler="get_user"},
                    {method="POST", pattern="^/api/users$",          handler="create_user"},
                    {method="PUT",  pattern="^/api/users/(%d+)$",    handler="update_user"},
                    {method="DELETE", pattern="^/api/users/(%d+)$",  handler="delete_user"},
                    {method="GET",  pattern="^/health$",             handler="health_check"},
                }
                
                -- Handlers
                local handlers = {}
                
                handlers.list_users = function(params)
                    return 200, {users = {
                        {id=1, name="Alice"},
                        {id=2, name="Bob"}
                    }}
                end
                
                handlers.get_user = function(params)
                    local user_id = tonumber(params[1])
                    if user_id == 1 then
                        return 200, {id=1, name="Alice", email="alice@example.com"}
                    end
                    return 404, {error = "User not found"}
                end
                
                handlers.create_user = function(params)
                    ngx.req.read_body()
                    local body = ngx.req.get_body_data()
                    local ok, data = pcall(cjson.decode, body or "")
                    if not ok then
                        return 400, {error = "Invalid JSON"}
                    end
                    -- สร้าง user ใหม่ (mock)
                    return 201, {id=3, name=data.name, created=true}
                end
                
                handlers.update_user = function(params)
                    local user_id = tonumber(params[1])
                    return 200, {id=user_id, updated=true}
                end
                
                handlers.delete_user = function(params)
                    return 204, nil
                end
                
                handlers.health_check = function(params)
                    return 200, {status="ok", time=ngx.time()}
                end
                
                -- Match route
                local matched = false
                for _, route in ipairs(routes) do
                    if method == route.method then
                        local captures = {uri:match(route.pattern)}
                        if #captures > 0 or uri:match(route.pattern) then
                            matched = true
                            local handler = handlers[route.handler]
                            if handler then
                                local status, body = handler(captures)
                                ngx.status = status
                                ngx.header["Content-Type"] = "application/json"
                                if body then
                                    ngx.say(cjson.encode(body))
                                end
                            end
                            break
                        end
                    end
                end
                
                if not matched then
                    ngx.status = 404
                    ngx.header["Content-Type"] = "application/json"
                    ngx.say(cjson.encode({error = "Not Found"}))
                end
            }
        }
    }
}
```

---

## 51.19 Static File Serving with Cache Headers

```nginx
server {
    listen 8080;
    root /var/www/html;
    
    location /static/ {
        # Static files with Lua cache headers
        header_filter_by_lua_block {
            local uri = ngx.var.uri
            
            -- Set cache headers based on file type
            if uri:match("%.(%w+)$") then
                local ext = uri:match("%.(%w+)$"):lower()
                
                local cache_times = {
                    png  = 2592000,  -- 30 days
                    jpg  = 2592000,
                    jpeg = 2592000,
                    gif  = 2592000,
                    css  = 604800,   -- 7 days
                    js   = 604800,
                    ico  = 86400,    -- 1 day
                    html = 3600,     -- 1 hour
                }
                
                local cache_time = cache_times[ext]
                if cache_time then
                    ngx.header["Cache-Control"] = "public, max-age=" .. cache_time
                    ngx.header["Expires"] = ngx.http_time(ngx.time() + cache_time)
                end
            end
        }
        
        try_files $uri $uri/ =404;
    }
    
    location /api/ {
        content_by_lua_block {
            -- ไม่ cache API responses
            ngx.header["Cache-Control"] = "no-store, no-cache, must-revalidate"
            ngx.header["Pragma"] = "no-cache"
            ngx.say('{"api": "response"}')
        }
    }
}
```

---

## 51.20 Connection Pooling กับ Redis

```nginx
location /redis-demo {
    content_by_lua_block {
        local redis = require "resty.redis"
        local red = redis:new()
        
        -- Set timeout
        red:set_timeouts(1000, 1000, 1000)  -- connect, send, read
        
        -- Connect
        local ok, err = red:connect("127.0.0.1", 6379)
        if not ok then
            ngx.status = 500
            ngx.say("Redis connect error: " .. err)
            return
        end
        
        -- Auth (ถ้ามี password)
        -- red:auth("password")
        
        -- SET
        local ok, err = red:set("hello", "world")
        if not ok then
            ngx.say("SET failed: " .. err)
            return
        end
        
        -- GET
        local val, err = red:get("hello")
        if not val then
            ngx.say("GET failed: " .. err)
            return
        end
        if val == ngx.null then
            ngx.say("Key not found")
            return
        end
        
        ngx.say("hello = " .. val)
        
        -- INCR
        local count = red:incr("visit_count")
        ngx.say("Visits: " .. count)
        
        -- EXPIRE
        red:expire("hello", 60)
        
        -- Connection pooling - สำคัญมาก!
        -- แทน red:close() ให้ใช้ set_keepalive
        local ok, err = red:set_keepalive(10000, 100)
        -- 10000 = idle timeout (ms), 100 = max connections in pool
        if not ok then
            ngx.log(ngx.ERR, "Failed to set keepalive: " .. err)
        end
    }
}
```

---

## 51.21 Caching ด้วย Redis

```nginx
location /cached-api {
    content_by_lua_block {
        local redis = require "resty.redis"
        local cjson = require "cjson"
        
        local red = redis:new()
        red:set_timeouts(1000, 1000, 1000)
        
        local ok, err = red:connect("127.0.0.1", 6379)
        if not ok then
            -- Redis ไม่ available, ไปดึงข้อมูลจริงเลย
            ngx.say(cjson.encode(fetch_data()))
            return
        end
        
        local cache_key = "api:users:list"
        
        -- ลอง cache ก่อน
        local cached, err = red:get(cache_key)
        if cached and cached ~= ngx.null then
            ngx.header["X-Cache"] = "HIT"
            ngx.header["Content-Type"] = "application/json"
            red:set_keepalive(10000, 100)
            ngx.say(cached)
            return
        end
        
        -- Cache miss - ดึงข้อมูลจริง
        local data = {
            users = {
                {id=1, name="Alice", email="alice@example.com"},
                {id=2, name="Bob", email="bob@example.com"},
            },
            total = 2,
            cached_at = ngx.time()
        }
        
        local json_data = cjson.encode(data)
        
        -- เก็บลง cache 60 วินาที
        red:setex(cache_key, 60, json_data)
        
        red:set_keepalive(10000, 100)
        
        ngx.header["X-Cache"] = "MISS"
        ngx.header["Content-Type"] = "application/json"
        ngx.say(json_data)
    }
}
```

---

## 51.22 Complete API Server Example

```nginx
-- /etc/openresty/conf.d/api.conf

lua_shared_dict api_cache 20m;
lua_shared_dict rate_limits 5m;

server {
    listen 8080;
    server_name api.example.com;
    
    # CORS preflight
    location / {
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization';
            add_header 'Content-Length' '0';
            return 204;
        }
        
        content_by_lua_file /etc/openresty/lua/router.lua;
    }
    
    # Health check endpoint (no auth needed)
    location = /health {
        content_by_lua_block {
            ngx.header["Content-Type"] = "application/json"
            ngx.say('{"status":"ok","time":' .. ngx.time() .. '}')
        }
    }
    
    # Metrics endpoint
    location = /metrics {
        content_by_lua_block {
            local cache = ngx.shared.api_cache
            local limits = ngx.shared.rate_limits
            
            ngx.header["Content-Type"] = "text/plain"
            ngx.say("# TYPE api_requests_total counter")
            ngx.say("api_requests_total " .. (cache:get("total_requests") or 0))
        }
    }
}
```

```lua
-- /etc/openresty/lua/router.lua

local cjson = require "cjson"
local cache = ngx.shared.api_cache

-- Rate limiting
local ip = ngx.var.remote_addr
local rate_key = "rate:" .. ip
local rate_limits = ngx.shared.rate_limits
local count = rate_limits:incr(rate_key, 1, 0, 60)

if count > 100 then  -- 100 req/min
    ngx.status = 429
    ngx.header["Content-Type"] = "application/json"
    ngx.say(cjson.encode({error="Too Many Requests", retry_after=60}))
    return
end

-- Track total requests
cache:incr("total_requests", 1, 0)

-- Router
local method = ngx.req.get_method()
local uri = ngx.var.uri

-- Dispatch
if method == "GET" and uri == "/api/v1/posts" then
    local posts = {
        {id=1, title="Hello World", author="Alice"},
        {id=2, title="OpenResty Guide", author="Bob"},
    }
    ngx.status = 200
    ngx.header["Content-Type"] = "application/json"
    ngx.say(cjson.encode({data=posts, total=#posts}))
    
elseif method == "GET" and uri:match("^/api/v1/posts/(%d+)$") then
    local post_id = tonumber(uri:match("^/api/v1/posts/(%d+)$"))
    if post_id == 1 then
        ngx.status = 200
        ngx.header["Content-Type"] = "application/json"
        ngx.say(cjson.encode({
            data = {id=1, title="Hello World", author="Alice", 
                   content="Full content here..."}
        }))
    else
        ngx.status = 404
        ngx.header["Content-Type"] = "application/json"
        ngx.say(cjson.encode({error="Post not found"}))
    end
    
elseif method == "POST" and uri == "/api/v1/posts" then
    ngx.req.read_body()
    local body = ngx.req.get_body_data()
    local ok, data = pcall(cjson.decode, body or "")
    
    if not ok then
        ngx.status = 400
        ngx.header["Content-Type"] = "application/json"
        ngx.say(cjson.encode({error="Invalid JSON body"}))
        return
    end
    
    if not data.title then
        ngx.status = 422
        ngx.header["Content-Type"] = "application/json"
        ngx.say(cjson.encode({error="title is required"}))
        return
    end
    
    ngx.status = 201
    ngx.header["Content-Type"] = "application/json"
    ngx.say(cjson.encode({
        data = {id=3, title=data.title, created=true}
    }))
    
else
    ngx.status = 404
    ngx.header["Content-Type"] = "application/json"
    ngx.say(cjson.encode({error="Route not found"}))
end
```

---

## 51.23 WebSocket ด้วย OpenResty

```nginx
location /ws {
    content_by_lua_block {
        local server = require "resty.websocket.server"
        
        local wb, err = server:new{
            timeout = 5000,
            max_payload_len = 65535
        }
        
        if not wb then
            ngx.log(ngx.ERR, "WebSocket upgrade failed: " .. err)
            return ngx.exit(444)
        end
        
        ngx.log(ngx.INFO, "WebSocket connection from: " .. ngx.var.remote_addr)
        
        while true do
            local data, typ, err = wb:recv_frame()
            
            if wb.fatal then
                ngx.log(ngx.ERR, "Fatal error: " .. (err or "unknown"))
                break
            end
            
            if not data then
                -- Timeout หรือ error
                local ok, err = wb:send_ping()
                if not ok then break end
            elseif typ == "close" then
                wb:send_close()
                break
            elseif typ == "ping" then
                wb:send_pong()
            elseif typ == "text" then
                -- Echo back
                local ok, err = wb:send_text("Echo: " .. data)
                if not ok then
                    ngx.log(ngx.ERR, "Send error: " .. (err or "unknown"))
                    break
                end
            end
        end
        
        wb:close()
    }
}
```

---

## 51.24 Error Handling Best Practices

```lua
-- Pattern 1: pcall สำหรับ catching errors
local ok, result = pcall(function()
    -- code ที่อาจ error
    local data = cjson.decode(invalid_json)
    return data
end)

if not ok then
    ngx.log(ngx.ERR, "Error: " .. tostring(result))
    ngx.status = 500
    ngx.say('{"error": "Internal Server Error"}')
    return
end

-- Pattern 2: Custom error handler
local function safe_json_decode(str)
    if not str or str == "" then
        return nil, "empty input"
    end
    local ok, result = pcall(cjson.decode, str)
    if not ok then
        return nil, "invalid JSON: " .. tostring(result)
    end
    return result, nil
end

ngx.req.read_body()
local body = ngx.req.get_body_data()
local data, err = safe_json_decode(body)
if not data then
    ngx.status = 400
    ngx.say('{"error": "' .. err .. '"}')
    return
end

-- Pattern 3: Error pages
local function send_error(status, message)
    ngx.status = status
    ngx.header["Content-Type"] = "application/json"
    ngx.say(cjson.encode({
        error = message,
        status = status,
        timestamp = ngx.time()
    }))
    ngx.exit(status)
end

-- ใช้งาน
if not user_id then
    send_error(400, "Missing user_id parameter")
end
```

---

## 51.25 Performance Tips

```lua
-- 1. ใช้ local variables สำหรับ globals ที่ใช้บ่อย
local ngx_say = ngx.say
local ngx_time = ngx.time
local ngx_log = ngx.log

-- 2. ใช้ table.concat แทน string concatenation ใน loop
local parts = {}
for i = 1, 100 do
    parts[i] = "item" .. i
end
ngx.say(table.concat(parts, ", "))

-- 3. Preload modules ใน init_by_lua_block
init_by_lua_block {
    cjson = require "cjson"
    -- cjson ถูกโหลดครั้งเดียว share ระหว่าง requests
}

-- 4. ใช้ ngx.shared dict แทน database สำหรับ hot data
local cache = ngx.shared.my_cache
local val = cache:get("frequent_key")

-- 5. Connection pooling เสมอ
local ok, err = red:set_keepalive(10000, 100)

-- 6. ใช้ ngx.ctx สำหรับ per-request state
ngx.ctx.user_id = "123"
ngx.ctx.start_time = ngx.now()

-- 7. Avoid global variable usage ใน request handler
-- BAD:
MY_DATA = {}  -- shared ระหว่าง requests!

-- GOOD:
local my_data = {}  -- local ต่อ request
```

---

## 51.26 SSL/TLS Configuration

```nginx
server {
    listen 443 ssl;
    server_name example.com;
    
    ssl_certificate     /etc/ssl/certs/example.crt;
    ssl_certificate_key /etc/ssl/private/example.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    
    location / {
        content_by_lua_block {
            -- ตรวจสอบ SSL info
            local ssl_protocol = ngx.var.ssl_protocol
            local ssl_cipher = ngx.var.ssl_cipher
            ngx.say("Protocol: " .. (ssl_protocol or "none"))
            ngx.say("Cipher: " .. (ssl_cipher or "none"))
            
            -- ตรวจว่า request มาจาก HTTPS
            local scheme = ngx.var.scheme
            if scheme ~= "https" then
                ngx.redirect("https://" .. ngx.var.host .. ngx.var.uri, 301)
                return
            end
            
            ngx.say("Secure connection!")
        }
    }
}
```

---

## 51.27 Lua Modules Organization

```nginx
# ใน nginx.conf
http {
    lua_package_path '/opt/myapp/lua/?.lua;/opt/myapp/lua/?/init.lua;;';
    
    server {
        location /app {
            content_by_lua_file /opt/myapp/lua/main.lua;
        }
    }
}
```

```lua
-- /opt/myapp/lua/main.lua
local router = require "app.router"
local middleware = require "app.middleware"

middleware.apply()
router.dispatch()
```

```lua
-- /opt/myapp/lua/app/router.lua
local M = {}
local cjson = require "cjson"

function M.dispatch()
    local uri = ngx.var.uri
    local method = ngx.req.get_method()
    
    -- Route matching logic
    ngx.say("Dispatching: " .. method .. " " .. uri)
end

return M
```

```lua
-- /opt/myapp/lua/app/middleware.lua
local M = {}

function M.apply()
    -- Apply global middlewares
    local ip = ngx.var.remote_addr
    ngx.header["X-Served-By"] = "openresty"
    ngx.log(ngx.INFO, "Request from: " .. ip)
end

return M
```

---

## 51.28 Timer และ Background Tasks

```nginx
init_worker_by_lua_block {
    -- Background task: cleanup expired cache
    local function cleanup_cache(premature)
        if premature then return end
        
        local cache = ngx.shared.page_cache
        local flushed = cache:flush_expired()
        
        if flushed > 0 then
            ngx.log(ngx.INFO, "Flushed " .. flushed .. " expired cache entries")
        end
        
        -- Schedule next run in 5 minutes
        ngx.timer.at(300, cleanup_cache)
    end
    
    -- Background task: health check upstream
    local function health_check(premature)
        if premature then return end
        
        local http = require "resty.http"
        local httpc = http.new()
        httpc:set_timeout(3000)
        
        local res, err = httpc:request_uri("http://upstream:8080/health")
        
        local status = ngx.shared.counters
        if res and res.status == 200 then
            status:set("upstream_healthy", 1)
        else
            status:set("upstream_healthy", 0)
            ngx.log(ngx.ERR, "Upstream unhealthy!")
        end
        
        -- Repeat every 30 seconds
        ngx.timer.at(30, health_check)
    end
    
    -- Start both background tasks
    ngx.timer.at(0, cleanup_cache)
    ngx.timer.at(10, health_check)  -- Start after 10s delay
}
```

---

## 51.29 Request Context (ngx.ctx)

```lua
-- ngx.ctx เป็น table ที่ share ระหว่าง phases ในแต่ละ request
-- มีชีวิตอยู่แค่ตาม request (ไม่ persist)

-- ใน access_by_lua_block:
ngx.ctx.start_time = ngx.now()
ngx.ctx.request_id = ngx.md5(ngx.var.remote_addr .. tostring(ngx.now()))
ngx.ctx.user = {id=1, name="Alice", role="admin"}

-- ใน content_by_lua_block:
local elapsed = ngx.now() - ngx.ctx.start_time
local user = ngx.ctx.user
ngx.say("User: " .. user.name)
ngx.say("Request took: " .. (elapsed * 1000) .. "ms")

-- ใน log_by_lua_block:
ngx.log(ngx.INFO, "Request " .. ngx.ctx.request_id 
    .. " duration: " .. (ngx.now() - ngx.ctx.start_time) .. "s")
```

---

## 51.30 Subrequests

```nginx
location /internal-api {
    internal;  -- เฉพาะ internal subrequests
    content_by_lua_block {
        ngx.say('{"data": "internal data"}')
    }
}

location /main {
    content_by_lua_block {
        -- ทำ subrequest ไปยัง internal location
        local res = ngx.location.capture("/internal-api")
        
        ngx.say("Status: " .. res.status)
        ngx.say("Body: " .. res.body)
        
        -- Parallel subrequests
        local res1, res2 = ngx.location.capture_multi({
            {"/api/users"},
            {"/api/posts"}
        })
        
        ngx.say("Users status: " .. res1.status)
        ngx.say("Posts status: " .. res2.status)
    }
}
```

---

## 51.31 Proxy Pass ด้วย Lua

```nginx
location /proxy {
    rewrite_by_lua_block {
        -- ปรับ request ก่อน proxy
        ngx.req.set_header("X-Forwarded-By", "openresty")
        ngx.req.set_header("X-Real-IP", ngx.var.remote_addr)
    }
    
    proxy_pass http://upstream_backend;
    
    header_filter_by_lua_block {
        -- ปรับ response headers หลัง proxy
        ngx.header["X-Processed-By"] = "openresty"
    }
}

upstream upstream_backend {
    server backend1:8080;
    server backend2:8080;
    keepalive 10;
}
```

---

## 51.32 Lua Nginx Context Summary

```lua
-- สรุป context ที่ใช้ได้ในแต่ละ phase

-- init_by_lua_block: ทุก Lua API ยกเว้น cosocket, sleep
-- init_worker_by_lua_block: ทุก API + timer, ไม่มี request context
-- set_by_lua_block: ngx.var, ngx.log, shared dict
-- rewrite_by_lua_block: ทุก API รวม cosocket
-- access_by_lua_block: ทุก API รวม cosocket  
-- content_by_lua_block: ทุก API
-- header_filter_by_lua_block: ส่วนใหญ่ แต่ไม่มี response body
-- body_filter_by_lua_block: ส่วนใหญ่ แต่ไม่มี cosocket
-- log_by_lua_block: ส่วนใหญ่ ไม่มี cosocket

-- ngx.ctx - request-scoped table
ngx.ctx.my_key = "my_value"

-- ngx.var - nginx variables
ngx.var.uri           -- "/api/users"
ngx.var.remote_addr   -- "192.168.1.1"
ngx.var.request_method -- "GET"

-- ngx.req - request helpers
ngx.req.get_headers()
ngx.req.get_uri_args()
ngx.req.get_body_data()

-- ngx.header - response headers
ngx.header["Content-Type"] = "application/json"
```

---

## 51.33 Complete Production-Ready Example

```nginx
# Production nginx.conf สำหรับ API server

worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    lua_package_path '/opt/myapp/lib/?.lua;;';
    
    # Shared memory
    lua_shared_dict rate_limits 10m;
    lua_shared_dict api_cache   50m;
    lua_shared_dict counters    5m;
    
    # Init
    init_by_lua_block {
        require "resty.core"
        cjson = require "cjson"
        cjson.encode_empty_table_as_array(false)
    }
    
    # Logging format
    log_format json_log escape=json
        '{"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"method":"$request_method",'
        '"uri":"$uri",'
        '"status":$status,'
        '"bytes_sent":$bytes_sent,'
        '"request_time":$request_time}';
    
    access_log /var/log/openresty/access.log json_log;
    
    # Gzip
    gzip on;
    gzip_types application/json text/plain;
    
    server {
        listen 8080;
        
        # Security headers (ใช้ทุก location)
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options DENY;
        
        location /api/ {
            access_by_lua_file  /opt/myapp/lua/auth.lua;
            content_by_lua_file /opt/myapp/lua/api.lua;
            log_by_lua_file     /opt/myapp/lua/logger.lua;
        }
        
        location /health {
            content_by_lua_block {
                ngx.header["Content-Type"] = "application/json"
                ngx.say('{"status":"ok"}')
            }
        }
    }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **OpenResty** คืออะไรและทำงานอย่างไร
2. **Nginx phases** และ Lua directives ต่างๆ (set, rewrite, access, content, header_filter, body_filter, log)
3. **ngx API** หลักๆ: ngx.say, ngx.print, ngx.header, ngx.var, ngx.req, ngx.status
4. **Shared memory** ด้วย ngx.shared สำหรับ cache และ rate limiting
5. **Worker processes** และ init_worker_by_lua
6. **cosocket** สำหรับ async I/O
7. **Connection pooling** กับ Redis/databases
8. การสร้าง **HTTP API server** และ routing
9. **Background tasks** ด้วย ngx.timer
10. **Best practices** สำหรับ production

OpenResty เป็นเครื่องมือที่ทรงพลังมากสำหรับสร้าง high-performance web services ด้วยความเรียบง่ายของ Lua!
