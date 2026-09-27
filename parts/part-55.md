# บทที่ 55: Middleware Architecture ใน Lua

## บทนำ

Middleware คือ software component ที่อยู่ระหว่าง request และ response ทำหน้าที่ประมวลผล, แปลง, หรือตัดสินใจเกี่ยวกับ request/response ก่อนที่จะส่งต่อไปยัง handler หรือ client

### Request Pipeline ทำงานอย่างไร
```
Client Request
       ↓
[Middleware 1: Logging]        ← ทำงานก่อน
       ↓
[Middleware 2: Auth]           ← ตรวจสอบ token
       ↓
[Middleware 3: Rate Limiting]  ← จำกัด requests
       ↓
[Middleware 4: CORS]           ← เพิ่ม CORS headers
       ↓
[Route Handler]                ← Business logic
       ↑
[Middleware 4: CORS]           ← (ถ้า process response ด้วย)
       ↑
[Middleware 3: Cache]          ← cache response
       ↑
[Middleware 2: Compression]    ← compress response
       ↑
[Middleware 1: Logging]        ← log response
       ↑
Client Response
```

---

## 55.1 Middleware Concept พื้นฐาน

```lua
-- Middleware คือ function ที่รับ handler และ return handler ใหม่
-- Type signature: (handler) -> handler
-- handler คือ function(request) -> response

-- Simple middleware pattern
local function timing_middleware(next_handler)
    return function(request)
        local start = ngx.now()
        
        local response = next_handler(request)  -- เรียก handler ถัดไป
        
        local elapsed = (ngx.now() - start) * 1000
        ngx.log(ngx.INFO, string.format("Request took %.2fms", elapsed))
        
        return response
    end
end

-- Compose middlewares
local function compose(handler, ...)
    local middlewares = {...}
    
    -- Apply middlewares in reverse order
    for i = #middlewares, 1, -1 do
        handler = middlewares[i](handler)
    end
    
    return handler
end

-- Handler
local function my_handler(request)
    return {body = "Hello World!", status = 200}
end

-- Apply middlewares
local composed = compose(my_handler, 
    timing_middleware,
    -- auth_middleware,
    -- rate_limit_middleware
)

-- ใช้งาน
-- composed({method="GET", path="/", headers={}})
```

---

## 55.2 Building Middleware Framework

```lua
-- middleware_framework.lua
-- Framework พื้นฐานสำหรับ middleware chain

local Framework = {}
Framework.__index = Framework

function Framework.new()
    local self = setmetatable({}, Framework)
    self.middlewares = {}
    self.routes      = {}
    return self
end

-- เพิ่ม global middleware
function Framework:use(middleware)
    table.insert(self.middlewares, middleware)
    return self  -- chainable
end

-- เพิ่ม route
function Framework:route(method, path, ...)
    local handlers = {...}
    -- handlers อาจเป็น middleware + final handler
    table.insert(self.routes, {
        method   = method:upper(),
        path     = path,
        handlers = handlers
    })
    return self
end

-- Shorthand methods
function Framework:get(path, ...)
    return self:route("GET", path, ...)
end

function Framework:post(path, ...)
    return self:route("POST", path, ...)
end

function Framework:put(path, ...)
    return self:route("PUT", path, ...)
end

function Framework:delete(path, ...)
    return self:route("DELETE", path, ...)
end

-- Match route
function Framework:match_route(method, path)
    for _, route in ipairs(self.routes) do
        if route.method == method then
            -- Simple exact match
            if route.path == path then
                return route, {}
            end
            
            -- Pattern match with params
            local pattern = route.path:gsub(":([%w_]+)", "([^/]+)")
            local param_names = {}
            for name in route.path:gmatch(":([%w_]+)") do
                table.insert(param_names, name)
            end
            
            local captures = {path:match("^" .. pattern .. "$")}
            if #captures > 0 then
                local params = {}
                for i, name in ipairs(param_names) do
                    params[name] = captures[i]
                end
                return route, params
            end
        end
    end
    return nil, {}
end

-- Process request
function Framework:handle(ctx)
    ctx = ctx or {}
    ctx.method  = ctx.method  or ngx.req.get_method()
    ctx.path    = ctx.path    or ngx.var.uri
    ctx.params  = ctx.params  or {}
    ctx.headers = ctx.headers or ngx.req.get_headers()
    ctx.status  = ctx.status  or 200
    
    -- Find matching route
    local route, path_params = self:match_route(ctx.method, ctx.path)
    
    if not route then
        ctx.status = 404
        ngx.status = 404
        ngx.header["Content-Type"] = "application/json"
        ngx.say('{"error": "Route not found"}')
        return
    end
    
    -- Merge path params
    for k, v in pairs(path_params) do
        ctx.params[k] = v
    end
    
    ctx.route = route
    
    -- Build handler chain: global middlewares + route handlers
    local all_handlers = {}
    for _, mw in ipairs(self.middlewares) do
        table.insert(all_handlers, mw)
    end
    for _, h in ipairs(route.handlers) do
        table.insert(all_handlers, h)
    end
    
    -- Execute chain
    local index = 0
    
    local function next_fn(err)
        if err then
            ctx.error = err
            ctx.status = 500
            ngx.status = 500
            ngx.header["Content-Type"] = "application/json"
            ngx.say('{"error": "' .. tostring(err) .. '"}')
            return
        end
        
        index = index + 1
        local handler = all_handlers[index]
        
        if handler then
            local ok, err = pcall(handler, ctx, next_fn)
            if not ok then
                ngx.log(ngx.ERR, "Handler error: " .. tostring(err))
                ngx.status = 500
                ngx.header["Content-Type"] = "application/json"
                ngx.say('{"error": "Internal server error"}')
            end
        end
    end
    
    next_fn()
end

return Framework

-- ===== ตัวอย่างการใช้งาน =====
local app = Framework.new()

-- เพิ่ม global middlewares
app:use(function(ctx, next)
    -- Timing middleware
    ctx.start_time = ngx.now()
    next()
    ngx.log(ngx.INFO, string.format("%.2fms %s %s %d",
        (ngx.now() - ctx.start_time) * 1000,
        ctx.method, ctx.path, ctx.status))
end)

app:use(function(ctx, next)
    -- Security headers middleware
    ngx.header["X-Content-Type-Options"] = "nosniff"
    ngx.header["X-Frame-Options"]        = "DENY"
    next()
end)

-- Routes
app:get("/", function(ctx, next)
    ngx.header["Content-Type"] = "application/json"
    ngx.say('{"message": "Hello World!"}')
end)

app:get("/users/:id", function(ctx, next)
    ngx.header["Content-Type"] = "application/json"
    ngx.say('{"user_id": "' .. ctx.params.id .. '"}')
end)
```

---

## 55.3 Authentication Middleware

```lua
-- auth_middleware.lua

local cjson = require "cjson"

-- Basic token authentication
local function token_auth_middleware(options)
    options = options or {}
    local excluded_paths = options.exclude or {}
    local token_store    = options.tokens or {}   -- ใช้ static tokens
    
    return function(ctx, next)
        -- ตรวจสอบว่า path นี้ต้องการ auth ไหม
        local path = ctx.path
        for _, excluded in ipairs(excluded_paths) do
            if path == excluded or (excluded:sub(-1) == "*" and 
                path:sub(1, #excluded-1) == excluded:sub(1, -2)) then
                return next()
            end
        end
        
        -- ดึง token จาก header
        local auth_header = ngx.var.http_authorization or ""
        local token       = auth_header:match("^Bearer%s+(.+)$")
        
        if not token then
            ctx.status = 401
            ngx.status = 401
            ngx.header["Content-Type"]     = "application/json"
            ngx.header["WWW-Authenticate"] = 'Bearer realm="API"'
            ngx.say(cjson.encode({
                error = {code = "AUTH_REQUIRED", message = "Authentication required"}
            }))
            return  -- ไม่ call next()
        end
        
        -- ตรวจสอบ token
        local user = token_store[token]
        if not user then
            ctx.status = 401
            ngx.status = 401
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({
                error = {code = "INVALID_TOKEN", message = "Invalid or expired token"}
            }))
            return
        end
        
        -- เก็บ user info ใน context
        ctx.current_user = user
        ctx.authenticated = true
        
        next()
    end
end

-- JWT Authentication middleware
local function jwt_middleware(options)
    options = options or {}
    local secret   = options.secret or error("JWT secret required")
    local excluded = options.exclude or {}
    
    return function(ctx, next)
        -- ตรวจสอบ excluded paths
        for _, path in ipairs(excluded) do
            if ctx.path == path then return next() end
        end
        
        local auth  = ngx.var.http_authorization or ""
        local token = auth:match("^Bearer%s+(.+)$")
        
        if not token then
            ctx.status = 401
            ngx.status = 401
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({error={code="AUTH_REQUIRED"}}))
            return
        end
        
        -- Verify JWT (ต้องมี lua-resty-jwt)
        local jwt = require "resty.jwt"
        local jwt_obj = jwt:verify(secret, token)
        
        if not jwt_obj.verified then
            ctx.status = 401
            ngx.status = 401
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({error={
                code    = "INVALID_TOKEN",
                message = jwt_obj.reason
            }}))
            return
        end
        
        -- ตรวจสอบ expiry
        local payload = jwt_obj.payload
        if payload.exp and payload.exp < ngx.time() then
            ctx.status = 401
            ngx.status = 401
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({error={code="TOKEN_EXPIRED"}}))
            return
        end
        
        ctx.current_user = {
            id   = tonumber(payload.sub),
            role = payload.role
        }
        
        next()
    end
end

-- Role-based authorization middleware
local function require_role(role)
    return function(ctx, next)
        if not ctx.current_user then
            ctx.status = 401
            ngx.status = 401
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({error={code="AUTH_REQUIRED"}}))
            return
        end
        
        local user_role = ctx.current_user.role or "user"
        
        -- Simple role hierarchy: admin > moderator > user
        local role_levels = {user=1, moderator=2, admin=3}
        local required    = role_levels[role] or 1
        local current     = role_levels[user_role] or 1
        
        if current < required then
            ctx.status = 403
            ngx.status = 403
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({error={
                code    = "PERMISSION_DENIED",
                message = "Requires role: " .. role
            }}))
            return
        end
        
        next()
    end
end

return {
    token_auth   = token_auth_middleware,
    jwt          = jwt_middleware,
    require_role = require_role
}
```

---

## 55.4 Logging Middleware

```lua
-- logging_middleware.lua

local cjson = require "cjson"

-- Structured logging middleware
local function logging_middleware(options)
    options = options or {}
    local log_level    = options.level    or ngx.INFO
    local log_body     = options.log_body or false
    local max_body_len = options.max_body or 1024
    
    return function(ctx, next)
        -- Before request
        ctx.log_start_time  = ngx.now()
        ctx.log_request_id  = ngx.md5(ngx.var.remote_addr .. tostring(ngx.now()) .. math.random(100000))
        
        -- Set request ID header
        ngx.header["X-Request-ID"] = ctx.log_request_id
        
        -- Log request
        local req_log = {
            event      = "request_start",
            request_id = ctx.log_request_id,
            method     = ngx.req.get_method(),
            path       = ngx.var.uri,
            query      = ngx.var.query_string,
            ip         = ngx.var.remote_addr,
            user_agent = ngx.var.http_user_agent,
            timestamp  = ngx.time()
        }
        
        if ctx.current_user then
            req_log.user_id = ctx.current_user.id
        end
        
        ngx.log(log_level, "REQ: " .. cjson.encode(req_log))
        
        -- Execute next handlers
        next()
        
        -- After request (log response)
        local duration = (ngx.now() - ctx.log_start_time) * 1000
        
        local res_log = {
            event      = "request_end",
            request_id = ctx.log_request_id,
            method     = ngx.req.get_method(),
            path       = ngx.var.uri,
            status     = ngx.status,
            duration   = string.format("%.2f", duration),
            bytes_sent = ngx.var.bytes_sent
        }
        
        -- Log level based on status
        local level = ngx.INFO
        if ngx.status >= 500 then
            level = ngx.ERR
        elseif ngx.status >= 400 then
            level = ngx.WARN
        end
        
        ngx.log(level, "RES: " .. cjson.encode(res_log))
        
        -- Track slow requests
        if duration > 1000 then  -- > 1 second
            ngx.log(ngx.WARN, "SLOW: " .. cjson.encode({
                request_id = ctx.log_request_id,
                path       = ngx.var.uri,
                duration   = duration
            }))
        end
    end
end

-- Access log middleware (ใช้ shared dict)
local function access_log_middleware(options)
    options = options or {}
    local shared_dict = options.shared_dict or "access_logs"
    
    return function(ctx, next)
        ctx.req_start = ngx.now()
        
        next()
        
        local dict = ngx.shared[shared_dict]
        if not dict then return end
        
        -- Increment counters
        dict:incr("total_requests", 1, 0)
        
        local status = tostring(ngx.status or 200)
        dict:incr("status_" .. status, 1, 0)
        
        -- Track per-path
        local path_key = "path:" .. (ngx.var.uri or "/"):gsub("[^%w]", "_")
        dict:incr(path_key, 1, 0)
        
        -- Track slow requests
        local duration = (ngx.now() - ctx.req_start) * 1000
        if duration > 500 then
            dict:incr("slow_requests", 1, 0)
        end
    end
end

return {
    logging    = logging_middleware,
    access_log = access_log_middleware
}
```

---

## 55.5 Rate Limiting Middleware

```lua
-- rate_limit_middleware.lua

local cjson = require "cjson"

-- Fixed window rate limiter
local function fixed_window_limiter(options)
    options    = options or {}
    local limit      = options.limit  or 100
    local window     = options.window or 60  -- seconds
    local key_prefix = options.prefix or "rl:"
    local shared     = options.shared or "rate_limits"
    
    return function(ctx, next)
        local dict  = ngx.shared[shared]
        if not dict then return next() end  -- ถ้าไม่มี shared dict, ข้าม
        
        -- Key based on IP or user
        local identifier
        if ctx.current_user then
            identifier = "user:" .. tostring(ctx.current_user.id)
        else
            identifier = "ip:" .. ngx.var.remote_addr
        end
        
        local key = key_prefix .. identifier
        
        -- Increment counter
        local count, err = dict:incr(key, 1, 0, window)
        if not count then
            ngx.log(ngx.ERR, "Rate limit counter error: " .. (err or ""))
            return next()  -- ถ้า error ให้ผ่านไป
        end
        
        -- Set headers
        ngx.header["X-RateLimit-Limit"]     = limit
        ngx.header["X-RateLimit-Remaining"] = math.max(0, limit - count)
        ngx.header["X-RateLimit-Reset"]     = ngx.time() + window
        
        if count > limit then
            ctx.status = 429
            ngx.status = 429
            ngx.header["Content-Type"] = "application/json"
            ngx.header["Retry-After"]  = window
            ngx.say(cjson.encode({
                error = {
                    code        = "RATE_LIMIT_EXCEEDED",
                    message     = "Too many requests",
                    retry_after = window
                }
            }))
            return  -- ไม่ call next()
        end
        
        next()
    end
end

-- Sliding window rate limiter (ใช้ Redis)
local function sliding_window_limiter(options)
    options    = options or {}
    local limit      = options.limit  or 100
    local window     = options.window or 60
    local key_prefix = options.prefix or "slrl:"
    
    return function(ctx, next)
        local redis = require "resty.redis"
        local red   = redis:new()
        red:set_timeouts(100, 100, 100)
        
        local ok, err = red:connect("127.0.0.1", 6379)
        if not ok then
            ngx.log(ngx.WARN, "Rate limiter: Redis unavailable: " .. (err or ""))
            return next()
        end
        
        local identifier = ctx.current_user 
            and ("user:" .. tostring(ctx.current_user.id))
            or  ("ip:" .. ngx.var.remote_addr)
        
        local key     = key_prefix .. identifier
        local now     = ngx.time()
        local win_start = now - window
        
        -- Sorted set sliding window
        local unique_id = now .. ":" .. math.random(1000000)
        
        red:multi()
        red:zadd(key, now, unique_id)
        red:zremrangebyscore(key, "-inf", win_start)
        red:zcard(key)
        red:expire(key, window + 1)
        local results = red:exec()
        
        red:set_keepalive(10000, 100)
        
        local count = results and results[3] or 0
        
        ngx.header["X-RateLimit-Limit"]     = limit
        ngx.header["X-RateLimit-Remaining"] = math.max(0, limit - count)
        ngx.header["X-RateLimit-Reset"]     = now + window
        
        if count > limit then
            ctx.status = 429
            ngx.status = 429
            ngx.header["Content-Type"] = "application/json"
            ngx.header["Retry-After"]  = window
            ngx.say(cjson.encode({
                error = {
                    code        = "RATE_LIMIT_EXCEEDED",
                    message     = "Rate limit exceeded. Try again in " .. window .. " seconds.",
                    retry_after = window,
                    limit       = limit
                }
            }))
            return
        end
        
        next()
    end
end

-- Different limits for different endpoints
local function tiered_rate_limiter(tiers)
    -- tiers = {
    --   {path="/api/v1/auth/login", limit=5, window=60},
    --   {path="/api/v1/",           limit=1000, window=3600},
    --   {path="*",                  limit=100, window=60}   -- default
    -- }
    
    return function(ctx, next)
        local path  = ctx.path or ngx.var.uri
        local tier  = nil
        
        -- Find matching tier
        for _, t in ipairs(tiers) do
            if t.path == "*" or path == t.path or 
               (t.path:sub(-1) == "/" and path:sub(1, #t.path) == t.path) then
                tier = t
                break
            end
        end
        
        if not tier then return next() end
        
        -- Apply rate limit for this tier
        local limiter = fixed_window_limiter({
            limit  = tier.limit,
            window = tier.window,
            prefix = "tier:" .. (tier.name or tier.path) .. ":"
        })
        
        limiter(ctx, next)
    end
end

return {
    fixed_window   = fixed_window_limiter,
    sliding_window = sliding_window_limiter,
    tiered         = tiered_rate_limiter
}
```

---

## 55.6 CORS Middleware

```lua
-- cors_middleware.lua

local function cors_middleware(options)
    options = options or {}
    
    local allowed_origins = options.origins or {"*"}
    local allowed_methods = options.methods or 
        "GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD"
    local allowed_headers = options.headers or 
        "Content-Type, Authorization, X-Requested-With, X-API-Key, Accept"
    local expose_headers  = options.expose or
        "X-RateLimit-Limit, X-RateLimit-Remaining, X-Request-ID, X-Total-Count"
    local max_age         = options.max_age or 86400
    local allow_credentials = options.credentials
    if allow_credentials == nil then allow_credentials = true end
    
    -- Normalize origins to set for O(1) lookup
    local origin_set = {}
    local has_wildcard = false
    for _, origin in ipairs(allowed_origins) do
        if origin == "*" then
            has_wildcard = true
        else
            origin_set[origin] = true
        end
    end
    
    return function(ctx, next)
        local origin = ngx.var.http_origin
        
        if origin then
            local origin_allowed = has_wildcard or origin_set[origin]
            
            -- Check wildcard subdomain patterns
            if not origin_allowed then
                for _, allowed in ipairs(allowed_origins) do
                    if allowed:match("^%*%.") then
                        local domain = allowed:sub(3)
                        if origin:match(domain:gsub("([%.%-])", "%%%1") .. "$") then
                            origin_allowed = true
                            break
                        end
                    end
                end
            end
            
            if origin_allowed then
                -- Set CORS headers
                if has_wildcard then
                    ngx.header["Access-Control-Allow-Origin"] = "*"
                else
                    ngx.header["Access-Control-Allow-Origin"] = origin
                    ngx.header["Vary"] = "Origin"
                end
                
                if allow_credentials and not has_wildcard then
                    ngx.header["Access-Control-Allow-Credentials"] = "true"
                end
                
                if expose_headers ~= "" then
                    ngx.header["Access-Control-Expose-Headers"] = expose_headers
                end
            end
        end
        
        -- Handle preflight
        if ngx.req.get_method() == "OPTIONS" then
            if origin then
                ngx.header["Access-Control-Allow-Methods"] = allowed_methods
                ngx.header["Access-Control-Allow-Headers"] = allowed_headers
                ngx.header["Access-Control-Max-Age"]       = max_age
            end
            
            ctx.status = 204
            ngx.status = 204
            ngx.header["Content-Length"] = "0"
            return  -- ไม่ต้อง call next() สำหรับ preflight
        end
        
        next()
    end
end

return cors_middleware
```

---

## 55.7 Compression Middleware

```lua
-- compression_middleware.lua
-- (OpenResty มี gzip built-in, แต่นี่เป็น Lua implementation)

local function compression_middleware(options)
    options = options or {}
    local min_size = options.min_size or 1024  -- ไม่ compress ถ้า < 1KB
    local level    = options.level    or 6     -- compression level 1-9
    local types    = options.types    or {
        "application/json",
        "text/html",
        "text/plain",
        "text/css",
        "application/javascript",
        "text/xml",
        "application/xml"
    }
    
    -- Convert types array to set
    local type_set = {}
    for _, t in ipairs(types) do
        type_set[t] = true
    end
    
    return function(ctx, next)
        -- ตรวจสอบว่า client รองรับ gzip
        local accept_enc = ngx.var.http_accept_encoding or ""
        local supports_gzip = accept_enc:match("gzip")
        
        if not supports_gzip then
            return next()  -- ไม่ compress
        end
        
        -- เพิ่ม Vary header
        ngx.header["Vary"] = "Accept-Encoding"
        
        -- ใน OpenResty ใช้ nginx gzip module แทนดีกว่า
        -- แต่สามารถตั้ง flag ให้ nginx กด gzip ได้
        ngx.ctx.compress_response = true
        
        next()
        
        -- หลัง next() ตรวจสอบ content type
        local ct = ngx.header["Content-Type"] or ""
        local base_type = ct:match("^([^;]+)"):match("^%s*(.-)%s*$")
        
        if ngx.ctx.compress_response and type_set[base_type] then
            -- Enable gzip (ใน OpenResty ต้องใช้ nginx directive)
            -- ngx.header["Content-Encoding"] = "gzip"
            -- (actual compression done by nginx)
        end
    end
end

return compression_middleware
```

---

## 55.8 Error Handling Middleware

```lua
-- error_middleware.lua

local cjson = require "cjson"

-- Global error handler middleware
local function error_middleware(options)
    options = options or {}
    local include_trace = options.include_trace or false
    local log_errors    = options.log_errors    ~= false  -- default true
    
    return function(ctx, next)
        -- Wrap next() ด้วย error handling
        local ok, err = pcall(next)
        
        if not ok then
            local error_msg = tostring(err)
            
            if log_errors then
                ngx.log(ngx.ERR, "Unhandled error: " .. error_msg)
                ngx.log(ngx.ERR, "Stack trace: " .. debug.traceback())
            end
            
            -- ตรวจสอบ error type
            local status  = 500
            local code    = "INTERNAL_ERROR"
            local message = "An internal server error occurred"
            
            -- Custom error types
            if type(err) == "table" then
                status  = err.status  or 500
                code    = err.code    or "ERROR"
                message = err.message or "Unknown error"
            elseif error_msg:match("not found") or error_msg:match("404") then
                status  = 404
                code    = "NOT_FOUND"
                message = "Resource not found"
            elseif error_msg:match("unauthorized") or error_msg:match("401") then
                status  = 401
                code    = "UNAUTHORIZED"
                message = "Unauthorized"
            end
            
            ctx.status = status
            ngx.status = status
            ngx.header["Content-Type"] = "application/json"
            
            local error_body = {
                error = {
                    code    = code,
                    message = message
                }
            }
            
            if include_trace and ngx.config.debug then
                error_body.error.trace = error_msg
            end
            
            ngx.say(cjson.encode(error_body))
        end
    end
end

-- Custom error class
local function create_error(status, code, message)
    return {
        status  = status,
        code    = code,
        message = message
    }
end

-- Error helpers
local Errors = {
    NotFound    = function(msg) return create_error(404, "NOT_FOUND", msg or "Not found") end,
    Unauthorized = function(msg) return create_error(401, "UNAUTHORIZED", msg or "Unauthorized") end,
    Forbidden   = function(msg) return create_error(403, "FORBIDDEN", msg or "Forbidden") end,
    BadRequest  = function(msg) return create_error(400, "BAD_REQUEST", msg or "Bad request") end,
    Conflict    = function(msg) return create_error(409, "CONFLICT", msg or "Conflict") end,
    Internal    = function(msg) return create_error(500, "INTERNAL_ERROR", msg or "Internal error") end,
}

return {
    middleware = error_middleware,
    Errors     = Errors
}
```

---

## 55.9 Cache Middleware

```lua
-- cache_middleware.lua

local cjson = require "cjson"

-- Response caching middleware
local function cache_middleware(options)
    options = options or {}
    local shared_name = options.shared or "response_cache"
    local default_ttl = options.ttl    or 60  -- seconds
    local cache_get   = options.get_methods  or {"GET", "HEAD"}
    local vary_by     = options.vary_by      or {}  -- headers to vary by
    
    -- Convert to set
    local method_set  = {}
    for _, m in ipairs(cache_get) do method_set[m] = true end
    
    return function(ctx, next)
        local cache = ngx.shared[shared_name]
        if not cache then return next() end
        
        local method = ngx.req.get_method()
        
        -- Only cache GET/HEAD
        if not method_set[method] then
            -- Non-GET request: invalidate related cache
            if method == "POST" or method == "PUT" or 
               method == "PATCH" or method == "DELETE" then
                -- Optionally invalidate cache for this path
                local uri    = ngx.var.uri
                local prefix = "cache:" .. uri
                -- Simple: delete exact match
                cache:delete(prefix)
            end
            return next()
        end
        
        -- Build cache key
        local uri          = ngx.var.uri
        local query        = ngx.var.query_string or ""
        local vary_parts   = {uri .. "?" .. query}
        
        for _, header_name in ipairs(vary_by) do
            local hval = ngx.var["http_" .. header_name:lower():gsub("-", "_")] or ""
            table.insert(vary_parts, header_name .. "=" .. hval)
        end
        
        -- Include user context in key (private caches)
        if options.private and ctx.current_user then
            table.insert(vary_parts, "user=" .. tostring(ctx.current_user.id))
        end
        
        local cache_key = "cache:" .. table.concat(vary_parts, "|")
        
        -- Try to get from cache
        local cached = cache:get(cache_key)
        if cached then
            local ok, cached_data = pcall(cjson.decode, cached)
            if ok then
                ngx.status = cached_data.status or 200
                
                -- Restore headers
                if cached_data.headers then
                    for k, v in pairs(cached_data.headers) do
                        ngx.header[k] = v
                    end
                end
                
                ngx.header["X-Cache"]   = "HIT"
                ngx.header["Age"]       = tostring(ngx.time() - (cached_data.cached_at or 0))
                ngx.say(cached_data.body)
                return
            end
        end
        
        -- Cache miss - need to capture response
        -- (ใน OpenResty ต้องใช้ body_filter_by_lua หรือ technique อื่น)
        ngx.header["X-Cache"] = "MISS"
        
        -- Simple approach: mark for caching in context
        ctx.should_cache  = true
        ctx.cache_key     = cache_key
        ctx.cache_ttl     = options.ttl or default_ttl
        
        next()
    end
end

-- Cache invalidation helper
local function invalidate_cache(pattern, shared_name)
    shared_name = shared_name or "response_cache"
    local cache = ngx.shared[shared_name]
    if not cache then return 0 end
    
    local keys   = cache:keys()
    local count  = 0
    
    for _, key in ipairs(keys) do
        if key:match(pattern) then
            cache:delete(key)
            count = count + 1
        end
    end
    
    return count
end

return {
    middleware = cache_middleware,
    invalidate = invalidate_cache
}
```

---

## 55.10 Request Validation Middleware

```lua
-- validation_middleware.lua

local cjson = require "cjson"

-- Schema-based validation middleware
local function validation_middleware(schema)
    return function(ctx, next)
        local errors = {}
        local params = ctx.params or {}
        
        for field, rules in pairs(schema) do
            local value = params[field]
            
            -- Required check
            if rules.required and (value == nil or value == "") then
                errors[field] = errors[field] or {}
                table.insert(errors[field], "is required")
                goto continue_field
            end
            
            -- Skip other validations if no value and not required
            if value == nil or value == "" then
                goto continue_field
            end
            
            -- Type check
            if rules.type == "number" then
                local num = tonumber(value)
                if not num then
                    errors[field] = errors[field] or {}
                    table.insert(errors[field], "must be a number")
                else
                    -- Coerce to number
                    params[field] = num
                    
                    if rules.min and num < rules.min then
                        errors[field] = errors[field] or {}
                        table.insert(errors[field], "must be >= " .. rules.min)
                    end
                    if rules.max and num > rules.max then
                        errors[field] = errors[field] or {}
                        table.insert(errors[field], "must be <= " .. rules.max)
                    end
                end
                
            elseif rules.type == "boolean" then
                if value == "true" or value == "1" or value == true then
                    params[field] = true
                elseif value == "false" or value == "0" or value == false then
                    params[field] = false
                else
                    errors[field] = errors[field] or {}
                    table.insert(errors[field], "must be a boolean")
                end
                
            elseif rules.type == "string" or not rules.type then
                local str = tostring(value)
                
                if rules.min_length and #str < rules.min_length then
                    errors[field] = errors[field] or {}
                    table.insert(errors[field], 
                        "must be at least " .. rules.min_length .. " characters")
                end
                
                if rules.max_length and #str > rules.max_length then
                    errors[field] = errors[field] or {}
                    table.insert(errors[field],
                        "must be at most " .. rules.max_length .. " characters")
                end
                
                if rules.pattern and not str:match(rules.pattern) then
                    errors[field] = errors[field] or {}
                    table.insert(errors[field], rules.pattern_msg or "format is invalid")
                end
                
                if rules.enum then
                    local valid = false
                    for _, v in ipairs(rules.enum) do
                        if v == str then valid = true break end
                    end
                    if not valid then
                        errors[field] = errors[field] or {}
                        table.insert(errors[field],
                            "must be one of: " .. table.concat(rules.enum, ", "))
                    end
                end
                
            elseif rules.type == "email" then
                if not value:match("^[^@%s]+@[^@%s]+%.[^@%s]+$") then
                    errors[field] = errors[field] or {}
                    table.insert(errors[field], "must be a valid email address")
                end
            end
            
            -- Custom validator
            if rules.validate and type(rules.validate) == "function" then
                local valid, msg = rules.validate(value, params)
                if not valid then
                    errors[field] = errors[field] or {}
                    table.insert(errors[field], msg or "is invalid")
                end
            end
            
            ::continue_field::
        end
        
        -- Return errors if any
        if next(errors) then
            ctx.status = 422
            ngx.status = 422
            ngx.header["Content-Type"] = "application/json"
            ngx.say(cjson.encode({
                error = {
                    code    = "VALIDATION_FAILED",
                    message = "Validation failed",
                    details = errors
                }
            }))
            return  -- ไม่ call next()
        end
        
        ctx.params = params
        next()
    end
end

return validation_middleware

-- ===== ตัวอย่างการใช้งาน =====
local validate = require "validation_middleware"

-- สร้าง schema สำหรับ create user
local create_user_schema = {
    name = {
        required   = true,
        type       = "string",
        min_length = 2,
        max_length = 100
    },
    email = {
        required = true,
        type     = "email"
    },
    age = {
        type = "number",
        min  = 0,
        max  = 150
    },
    role = {
        type = "string",
        enum = {"admin", "user", "moderator"}
    },
    password = {
        required   = true,
        min_length = 8,
        validate   = function(val, params)
            if not val:match("%d") then
                return false, "must contain at least one number"
            end
            if not val:match("%u") then
                return false, "must contain at least one uppercase letter"
            end
            return true
        end
    }
}

app:post("/api/v1/users",
    validate(create_user_schema),
    function(ctx, next)
        -- params ถูก validate แล้ว
        local Users = require "models.users"
        local user  = Users:create(ctx.params)
        
        ngx.status = 201
        ngx.header["Content-Type"] = "application/json"
        ngx.say(cjson.encode({data = user}))
    end
)
```

---

## 55.11 Response Transformation Middleware

```lua
-- response_transform_middleware.lua

local cjson = require "cjson"

-- Transform response data
local function response_transform_middleware(options)
    options = options or {}
    
    return function(ctx, next)
        next()
        
        -- ตรวจสอบ content type
        local ct = ngx.header["Content-Type"] or ""
        if not ct:match("application/json") then return end
        
        -- Wrap response ใน standard format (ถ้ายังไม่ wrap)
        -- ยากที่จะทำใน OpenResty เพราะ body ถูกส่งไปแล้ว
        -- ต้องใช้ body_filter_by_lua ในระดับ nginx
        
        -- บันทึก response info ไว้ใน context
        ctx.response_sent_at = ngx.time()
    end
end

-- Body filter สำหรับ transform JSON responses
-- ใช้ใน nginx.conf: body_filter_by_lua_block { ... }
local function json_transform_filter()
    local chunk   = ngx.arg[1]
    local is_last = ngx.arg[2]
    
    local ct = ngx.header["Content-Type"] or ""
    if not ct:match("application/json") then return end
    
    -- สะสม chunks
    if chunk and chunk ~= "" then
        ngx.ctx.body_buf = (ngx.ctx.body_buf or "") .. chunk
        ngx.arg[1] = ""  -- clear chunk
    end
    
    if is_last then
        local body = ngx.ctx.body_buf or ""
        
        if body ~= "" then
            local ok, data = pcall(cjson.decode, body)
            
            if ok then
                -- เพิ่ม metadata
                local transformed = {
                    success   = ngx.status < 400,
                    data      = data.data or data,
                    meta      = data.meta,
                    timestamp = ngx.time(),
                    request_id = ngx.ctx.request_id
                }
                
                -- Remove nil values
                if not transformed.meta then transformed.meta = nil end
                
                ngx.arg[1] = cjson.encode(transformed)
            else
                ngx.arg[1] = body  -- return as-is
            end
        end
    end
end

return {
    middleware = response_transform_middleware,
    filter     = json_transform_filter
}
```

---

## 55.12 Chaining Middlewares

```lua
-- middleware_chain.lua
-- ตัวอย่างการ chain middlewares หลายตัวเข้าด้วยกัน

local logging   = require "middleware.logging"
local auth      = require "middleware.auth"
local rate_limit = require "middleware.rate_limit"
local cors      = require "middleware.cors"
local validate  = require "middleware.validation"
local cache     = require "middleware.cache"

-- Build middleware stack
local function create_api(config)
    local middlewares = {}
    
    -- 1. CORS (ต้องเป็นลำดับแรกสุด)
    if config.cors then
        table.insert(middlewares, cors(config.cors))
    end
    
    -- 2. Logging
    if config.logging ~= false then
        table.insert(middlewares, logging.middleware({
            level    = ngx.INFO,
            log_body = config.log_body or false
        }))
    end
    
    -- 3. Error handling
    table.insert(middlewares, require("middleware.error").middleware({
        include_trace = config.debug or false
    }))
    
    -- 4. Rate limiting
    if config.rate_limit then
        table.insert(middlewares, rate_limit.fixed_window(config.rate_limit))
    end
    
    -- 5. Authentication
    if config.auth then
        if config.auth.type == "jwt" then
            table.insert(middlewares, auth.jwt(config.auth))
        elseif config.auth.type == "token" then
            table.insert(middlewares, auth.token_auth(config.auth))
        end
    end
    
    -- 6. Caching (หลัง auth เพื่อ cache per-user ได้)
    if config.cache then
        table.insert(middlewares, cache.middleware(config.cache))
    end
    
    return middlewares
end

-- ตัวอย่างการใช้งาน
local api_config = {
    cors = {
        origins     = {"https://app.example.com"},
        credentials = true
    },
    rate_limit = {
        limit  = 100,
        window = 60
    },
    auth = {
        type   = "jwt",
        secret = os.getenv("JWT_SECRET"),
        exclude = {"/api/v1/auth/login", "/api/v1/auth/register", "/api/v1/health"}
    },
    cache = {
        shared = "response_cache",
        ttl    = 60
    }
}

local stack = create_api(api_config)

-- Apply stack to request
local function handle_request()
    local ctx = {
        method = ngx.req.get_method(),
        path   = ngx.var.uri,
        params = {}
    }
    
    -- Merge query params
    local args = ngx.req.get_uri_args()
    for k, v in pairs(args) do
        ctx.params[k] = v
    end
    
    -- Build execution chain
    local index = 0
    local function execute_next()
        index = index + 1
        local mw = stack[index]
        if mw then
            mw(ctx, execute_next)
        end
        -- If no more middlewares, return 404
    end
    
    execute_next()
end
```

---

## 55.13 Conditional Middleware

```lua
-- conditional_middleware.lua
-- Middleware ที่ทำงานเฉพาะเงื่อนไขบางอย่าง

-- Apply middleware เฉพาะ paths บางอย่าง
local function path_middleware(patterns, middleware_fn)
    return function(ctx, next)
        local path = ctx.path or ngx.var.uri
        local matches = false
        
        for _, pattern in ipairs(patterns) do
            if type(pattern) == "string" then
                -- Exact match หรือ prefix
                if pattern:sub(-1) == "*" then
                    local prefix = pattern:sub(1, -2)
                    if path:sub(1, #prefix) == prefix then
                        matches = true
                        break
                    end
                elseif path == pattern then
                    matches = true
                    break
                end
            elseif type(pattern) == "function" then
                if pattern(path) then
                    matches = true
                    break
                end
            end
        end
        
        if matches then
            middleware_fn(ctx, next)
        else
            next()
        end
    end
end

-- Apply middleware เฉพาะ methods บางอย่าง
local function method_middleware(methods, middleware_fn)
    local method_set = {}
    for _, m in ipairs(methods) do method_set[m:upper()] = true end
    
    return function(ctx, next)
        local method = ngx.req.get_method():upper()
        
        if method_set[method] then
            middleware_fn(ctx, next)
        else
            next()
        end
    end
end

-- Apply middleware ตาม content type
local function content_type_middleware(types, middleware_fn)
    local type_set = {}
    for _, t in ipairs(types) do type_set[t] = true end
    
    return function(ctx, next)
        local ct = ngx.var.http_content_type or ""
        local base_type = ct:match("^([^;]+)")
        if base_type then base_type = base_type:match("^%s*(.-)%s*$") end
        
        if type_set[base_type] then
            middleware_fn(ctx, next)
        else
            next()
        end
    end
end

-- ตัวอย่างการใช้งาน
local cond = require "conditional_middleware"
local logging = require "middleware.logging"
local cache   = require "middleware.cache"

-- เฉพาะ API paths
local api_logging = path_middleware(
    {"/api/*", "/v1/*"},
    logging.middleware({level = ngx.INFO})
)

-- Cache เฉพาะ GET requests
local get_cache = method_middleware(
    {"GET", "HEAD"},
    cache.middleware({ttl = 60})
)

return {
    path         = path_middleware,
    method       = method_middleware,
    content_type = content_type_middleware
}
```

---

## 55.14 Per-route Middleware

```lua
-- per_route_middleware.lua
-- Middleware ที่ apply เฉพาะ route บางตัว

local auth      = require "middleware.auth"
local rate_limit = require "middleware.rate_limit"
local validate  = require "middleware.validation"

-- Route definitions พร้อม middlewares
local routes = {
    -- Public routes (ไม่ต้องการ auth)
    {
        method   = "GET",
        path     = "/api/v1/health",
        middlewares = {},
        handler  = function(ctx)
            ngx.status = 200
            ngx.header["Content-Type"] = "application/json"
            ngx.say('{"status":"ok"}')
        end
    },
    
    -- Login (rate limited, no auth)
    {
        method = "POST",
        path   = "/api/v1/auth/login",
        middlewares = {
            rate_limit.fixed_window({limit=5, window=60, prefix="login:"})
        },
        handler = function(ctx)
            -- Login logic
        end
    },
    
    -- Protected routes (ต้องการ auth)
    {
        method = "GET",
        path   = "/api/v1/users",
        middlewares = {
            auth.jwt({secret = os.getenv("JWT_SECRET")}),
            auth.require_role("admin")
        },
        handler = function(ctx)
            -- List users (admin only)
        end
    },
    
    -- Create user with validation
    {
        method = "POST",
        path   = "/api/v1/users",
        middlewares = {
            auth.jwt({secret = os.getenv("JWT_SECRET")}),
            validate({
                name  = {required=true, min_length=2},
                email = {required=true, type="email"}
            })
        },
        handler = function(ctx)
            -- Create user
        end
    }
}

-- Router with per-route middleware
local function route_handler()
    local method = ngx.req.get_method():upper()
    local path   = ngx.var.uri
    
    -- Read body for POST/PUT/PATCH
    if method == "POST" or method == "PUT" or method == "PATCH" then
        ngx.req.read_body()
        local body = ngx.req.get_body_data()
        if body and body ~= "" then
            local cjson = require "cjson"
            local ok, data = pcall(cjson.decode, body)
            if ok and type(data) == "table" then
                ngx.ctx.json_body = data
            end
        end
    end
    
    -- Find matching route
    for _, route in ipairs(routes) do
        if route.method == method and route.path == path then
            -- Build context
            local ctx = {
                method  = method,
                path    = path,
                params  = ngx.req.get_uri_args() or {}
            }
            
            -- Merge JSON body
            if ngx.ctx.json_body then
                for k, v in pairs(ngx.ctx.json_body) do
                    ctx.params[k] = v
                end
            end
            
            -- Build middleware chain
            local handlers = {}
            for _, mw in ipairs(route.middlewares) do
                table.insert(handlers, mw)
            end
            table.insert(handlers, function(c, n) route.handler(c) end)
            
            -- Execute chain
            local index = 0
            local function next_fn()
                index = index + 1
                local h = handlers[index]
                if h then h(ctx, next_fn) end
            end
            
            next_fn()
            return
        end
    end
    
    -- No route matched
    ngx.status = 404
    ngx.header["Content-Type"] = "application/json"
    local cjson = require "cjson"
    ngx.say(cjson.encode({error={code="NOT_FOUND", message="Route not found"}}))
end

return route_handler
```

---

## 55.15 Benchmarking Middleware Overhead

```lua
-- benchmark_middleware.lua
-- เครื่องมือ benchmark middleware stack

local function benchmark_middleware(options)
    options = options or {}
    local threshold_ms = options.threshold or 10  -- log ถ้า middleware ใช้เวลา > 10ms
    local name         = options.name or "unknown"
    
    return function(fn)
        return function(ctx, next)
            local start = ngx.now()
            
            fn(ctx, function()
                local mw_time = (ngx.now() - start) * 1000
                
                if mw_time > threshold_ms then
                    ngx.log(ngx.WARN, string.format(
                        "Slow middleware [%s]: %.2fms",
                        name, mw_time
                    ))
                end
                
                next()
                
                local total_time = (ngx.now() - start) * 1000
                ngx.log(ngx.DEBUG, string.format(
                    "Middleware [%s] total: %.2fms",
                    name, total_time
                ))
            end)
        end
    end
end

-- ใช้ wrap middleware เพื่อ benchmark
local function wrap_with_timing(name, middleware_fn)
    return function(ctx, next)
        local start = ngx.now()
        
        middleware_fn(ctx, function()
            local mw_duration = (ngx.now() - start) * 1000
            ngx.log(ngx.DEBUG, name .. " overhead: " .. string.format("%.3f", mw_duration) .. "ms")
            
            local inner_start = ngx.now()
            next()
            local inner_duration = (ngx.now() - inner_start) * 1000
            ngx.log(ngx.DEBUG, name .. " inner: " .. string.format("%.3f", inner_duration) .. "ms")
        end)
        
        local total = (ngx.now() - start) * 1000
        ngx.log(ngx.DEBUG, name .. " total: " .. string.format("%.3f", total) .. "ms")
    end
end

-- Performance stats middleware
local function perf_stats_middleware()
    local stats = ngx.shared.perf_stats
    if not stats then
        return function(ctx, next) next() end
    end
    
    return function(ctx, next)
        local start = ngx.now()
        
        next()
        
        local duration = (ngx.now() - start) * 1000
        
        -- Update stats
        stats:incr("total_requests", 1, 0)
        
        -- Track P50, P95, P99 (simplified with buckets)
        if duration < 10 then
            stats:incr("bucket_10ms", 1, 0)
        elseif duration < 50 then
            stats:incr("bucket_50ms", 1, 0)
        elseif duration < 100 then
            stats:incr("bucket_100ms", 1, 0)
        elseif duration < 500 then
            stats:incr("bucket_500ms", 1, 0)
        elseif duration < 1000 then
            stats:incr("bucket_1000ms", 1, 0)
        else
            stats:incr("bucket_slow", 1, 0)
        end
        
        -- Cumulative duration (สำหรับ average)
        -- (ใช้ float ไม่ได้ใน incr, ต้องแปลง)
        stats:incr("total_duration_ms", math.floor(duration), 0)
    end
end

-- ดู stats
-- location /perf-stats {
--   content_by_lua_block {
--     local stats = ngx.shared.perf_stats
--     local total = stats:get("total_requests") or 0
--     local total_ms = stats:get("total_duration_ms") or 0
--     local avg = total > 0 and (total_ms / total) or 0
--     
--     ngx.say("Total requests: " .. total)
--     ngx.say("Average response: " .. string.format("%.2f", avg) .. "ms")
--     ngx.say("< 10ms:   " .. (stats:get("bucket_10ms") or 0))
--     ngx.say("< 50ms:   " .. (stats:get("bucket_50ms") or 0))
--     ngx.say("< 100ms:  " .. (stats:get("bucket_100ms") or 0))
--     ngx.say("< 500ms:  " .. (stats:get("bucket_500ms") or 0))
--     ngx.say("< 1000ms: " .. (stats:get("bucket_1000ms") or 0))
--     ngx.say(">= 1000ms:" .. (stats:get("bucket_slow") or 0))
--   }
-- }

return {
    benchmark  = benchmark_middleware,
    wrap_timing = wrap_with_timing,
    perf_stats = perf_stats_middleware
}
```

---

## 55.16 Complete Middleware Stack Example

```nginx
-- nginx.conf สำหรับ production API กับ middleware stack ครบ

http {
    lua_shared_dict rate_limits  10m;
    lua_shared_dict response_cache 50m;
    lua_shared_dict access_logs  20m;
    lua_shared_dict perf_stats   5m;
    
    lua_package_path '/opt/api/lib/?.lua;;';
    
    init_by_lua_block {
        -- Preload modules
        require "resty.core"
        cjson = require "cjson"
        
        -- Load middleware stack
        middleware_stack = require "api.middleware_stack"
    }
    
    server {
        listen 8080;
        server_name api.example.com;
        
        location / {
            content_by_lua_block {
                middleware_stack.handle()
            }
            
            log_by_lua_block {
                middleware_stack.log()
            }
        }
    }
}
```

```lua
-- /opt/api/lib/api/middleware_stack.lua
-- Assembled middleware stack

local cjson      = require "cjson"
local logging    = require "middleware.logging"
local auth       = require "middleware.auth"
local rate_limit = require "middleware.rate_limit"
local cors_mw    = require "middleware.cors"
local validate   = require "middleware.validation"
local error_mw   = require "middleware.error"
local perf       = require "middleware.benchmark"

-- Configuration
local config = {
    jwt_secret = os.getenv("JWT_SECRET") or "dev-secret",
    rate_limit_per_min = 100,
    allowed_origins = {
        "https://app.example.com",
        "https://admin.example.com"
    }
}

-- Global middleware stack
local global_stack = {
    -- 1. CORS
    cors_mw({
        origins     = config.allowed_origins,
        credentials = true
    }),
    
    -- 2. Performance stats
    perf.perf_stats(),
    
    -- 3. Logging
    logging.middleware({level = ngx.INFO}),
    
    -- 4. Error handling
    error_mw.middleware({
        include_trace = ngx.config.debug
    }),
    
    -- 5. Rate limiting
    rate_limit.fixed_window({
        limit  = config.rate_limit_per_min,
        window = 60,
        shared = "rate_limits"
    }),
    
    -- 6. Authentication (optional per route)
    auth.jwt({
        secret  = config.jwt_secret,
        exclude = {
            "/api/v1/health",
            "/api/v1/auth/login",
            "/api/v1/auth/register"
        }
    }),
}

-- Route registry
local routes = {}

local function register(method, path, specific_middlewares, handler)
    table.insert(routes, {
        method      = method:upper(),
        path        = path,
        middlewares = specific_middlewares,
        handler     = handler
    })
end

-- Route definitions
register("GET", "/api/v1/health", {}, function(ctx)
    ngx.status = 200
    ngx.header["Content-Type"] = "application/json"
    ngx.say(cjson.encode({
        status    = "ok",
        version   = "1.0.0",
        timestamp = ngx.time()
    }))
end)

register("POST", "/api/v1/auth/login",
    {rate_limit.fixed_window({limit=5, window=60, prefix="login:"})},
    function(ctx)
        -- Login logic
        ngx.status = 200
        ngx.header["Content-Type"] = "application/json"
        ngx.say(cjson.encode({
            data = {
                access_token  = "jwt-token-here",
                expires_in    = 3600,
                token_type    = "Bearer"
            }
        }))
    end
)

register("GET", "/api/v1/users",
    {auth.require_role("admin")},
    function(ctx)
        ngx.status = 200
        ngx.header["Content-Type"] = "application/json"
        ngx.say(cjson.encode({
            data = {},
            meta = {total = 0}
        }))
    end
)

-- Handle incoming request
local function handle()
    -- Parse request
    local method = ngx.req.get_method():upper()
    local path   = ngx.var.uri
    
    -- Read body
    ngx.req.read_body()
    local body  = ngx.req.get_body_data()
    local params = ngx.req.get_uri_args() or {}
    
    -- Merge JSON body
    if body and body ~= "" then
        local ok, data = pcall(cjson.decode, body)
        if ok and type(data) == "table" then
            for k, v in pairs(data) do params[k] = v end
        end
    end
    
    -- Build context
    local ctx = {
        method  = method,
        path    = path,
        params  = params,
        headers = ngx.req.get_headers()
    }
    
    -- Find route
    local matched_route = nil
    local path_params   = {}
    
    for _, route in ipairs(routes) do
        if route.method == method then
            if route.path == path then
                matched_route = route
                break
            end
            
            -- Pattern matching
            local pattern = route.path:gsub(":([%w_]+)", "([^/]+)")
            local names   = {}
            for name in route.path:gmatch(":([%w_]+)") do
                table.insert(names, name)
            end
            
            local caps = {path:match("^" .. pattern .. "$")}
            if #caps > 0 then
                matched_route = route
                for i, name in ipairs(names) do
                    path_params[name] = caps[i]
                end
                break
            end
        end
    end
    
    if not matched_route then
        ngx.status = 404
        ngx.header["Content-Type"] = "application/json"
        ngx.say(cjson.encode({error={code="NOT_FOUND", message="Route not found"}}))
        return
    end
    
    -- Merge path params
    for k, v in pairs(path_params) do ctx.params[k] = v end
    ctx.route = matched_route
    
    -- Build full chain: global + route-specific + handler
    local chain = {}
    for _, mw in ipairs(global_stack) do table.insert(chain, mw) end
    for _, mw in ipairs(matched_route.middlewares) do table.insert(chain, mw) end
    table.insert(chain, function(c, n) matched_route.handler(c) end)
    
    -- Execute
    local idx = 0
    local function exec_next()
        idx = idx + 1
        if chain[idx] then chain[idx](ctx, exec_next) end
    end
    
    exec_next()
end

-- Log phase
local function log()
    local dict = ngx.shared.access_logs
    if dict then
        dict:incr("total", 1, 0)
        dict:incr("status_" .. ngx.status, 1, 0)
    end
end

return {
    handle   = handle,
    log      = log,
    register = register
}
```

---

## สรุป

ในบทนี้เราเรียนรู้ Middleware Architecture ครบถ้วน:

1. **Middleware Concept** - หลักการและ request pipeline
2. **Framework Building** - สร้าง middleware framework เอง
3. **Authentication Middleware** - JWT, token-based auth, role-based access
4. **Logging Middleware** - structured logging, access logs
5. **Rate Limiting Middleware** - fixed window, sliding window
6. **CORS Middleware** - handle cross-origin requests
7. **Compression Middleware** - compress responses
8. **Error Handling Middleware** - centralized error handling
9. **Cache Middleware** - response caching
10. **Validation Middleware** - schema-based input validation
11. **Response Transform** - transform response format
12. **Chaining Middlewares** - compose middleware stack
13. **Conditional Middleware** - apply by path/method/content-type
14. **Per-route Middleware** - route-specific middlewares
15. **Benchmarking** - measure middleware overhead
16. **Complete Stack** - production-ready middleware stack

Middleware architecture เป็นหัวใจของ web framework ที่ดี ทำให้ code reusable, testable และ maintainable!
