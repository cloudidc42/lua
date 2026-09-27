# บทที่ 99: Real Project 3 - API Gateway

## บทนำ

บทนี้สร้าง **Production-Grade API Gateway** ด้วย OpenResty/Lua ที่รองรับ:
- Rate limiting (Token Bucket algorithm)
- Authentication & Authorization (JWT + API Keys)
- Request routing & load balancing
- Circuit breaker pattern
- Request/Response transformation
- Caching layer (Redis)
- Logging & monitoring
- Health checks

---

## 99.1 Architecture Overview

```
                    ┌─────────────────────────────┐
                    │         API Gateway          │
                    │                              │
Internet ──────────▶│  ┌──────────┐ ┌──────────┐  │
                    │  │  Auth    │ │  Rate    │  │
                    │  │  Layer   │ │  Limiter │  │
                    │  └──────────┘ └──────────┘  │
                    │  ┌──────────┐ ┌──────────┐  │
                    │  │ Router   │ │ Circuit  │  │
                    │  │         │ │ Breaker  │  │
                    │  └──────────┘ └──────────┘  │
                    │  ┌──────────┐ ┌──────────┐  │
                    │  │  Cache   │ │Transform │  │
                    │  │         │ │         │  │
                    │  └──────────┘ └──────────┘  │
                    └─────────────────────────────┘
                              │
                    ┌─────────┼────────────┐
                    ▼         ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Service A │ │ Service B │ │ Service C│
              │ :8081    │ │ :8082    │ │ :8083   │
              └──────────┘ └──────────┘ └──────────┘
```

---

## 99.2 Project Structure

```
api-gateway/
├── nginx.conf
├── conf.d/
│   ├── upstreams.conf
│   └── routes.conf
├── lua/
│   ├── gateway/
│   │   ├── init.lua
│   │   ├── auth.lua
│   │   ├── rate_limiter.lua
│   │   ├── router.lua
│   │   ├── circuit_breaker.lua
│   │   ├── cache.lua
│   │   ├── transform.lua
│   │   ├── logger.lua
│   │   └── health.lua
│   └── utils/
│       ├── redis.lua
│       ├── jwt.lua
│       └── config.lua
├── config/
│   ├── routes.json
│   └── services.json
└── docker-compose.yml
```

---

## 99.3 nginx.conf

```nginx
# nginx.conf
worker_processes auto;
error_log logs/error.log warn;
pid logs/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include mime.types;
    default_type application/octet-stream;
    
    # Lua package path
    lua_package_path "/usr/local/openresty/lua/?.lua;;";
    lua_package_cpath "/usr/local/openresty/lua/?.so;;";
    
    # Shared memory zones
    lua_shared_dict rate_limits 100m;
    lua_shared_dict circuit_breakers 10m;
    lua_shared_dict cache_store 200m;
    lua_shared_dict api_keys 50m;
    lua_shared_dict gateway_stats 10m;
    
    # DNS resolver
    resolver 8.8.8.8 valid=30s;
    
    # Gzip
    gzip on;
    gzip_types application/json text/plain application/xml;
    
    # Timeouts
    proxy_connect_timeout 5s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
    
    # Initialize gateway on startup
    init_by_lua_block {
        require("gateway.init").startup()
    }
    
    init_worker_by_lua_block {
        require("gateway.init").worker_init()
    }
    
    # Logging format
    log_format gateway_log '$remote_addr [$time_local] '
                           '"$request" $status $body_bytes_sent '
                           '"$http_referer" "$http_user_agent" '
                           '$request_time $upstream_response_time '
                           'route=$upstream_http_x_route '
                           'service=$upstream_http_x_service';
    
    access_log logs/gateway_access.log gateway_log;
    
    # Include route configs
    include conf.d/*.conf;
    
    server {
        listen 80;
        server_name api.example.com;
        
        # Health check (no auth needed)
        location /health {
            access_log off;
            content_by_lua_block {
                require("gateway.health").check()
            }
        }
        
        # Metrics endpoint
        location /metrics {
            allow 127.0.0.1;
            deny all;
            content_by_lua_block {
                require("gateway.health").metrics()
            }
        }
        
        # Main gateway entry point
        location / {
            # Phase 1: Access control
            access_by_lua_block {
                local gw = require("gateway.init")
                gw.access_phase()
            }
            
            # Phase 2: Content (proxy to upstream)
            content_by_lua_block {
                local gw = require("gateway.init")
                gw.proxy_phase()
            }
            
            # Phase 3: Header filter (transform response)
            header_filter_by_lua_block {
                local gw = require("gateway.init")
                gw.header_phase()
            }
            
            # Phase 4: Body filter (transform body)
            body_filter_by_lua_block {
                local gw = require("gateway.init")
                gw.body_phase()
            }
            
            # Phase 5: Log
            log_by_lua_block {
                local gw = require("gateway.init")
                gw.log_phase()
            }
        }
    }
}
```

---

## 99.4 Gateway Init (gateway/init.lua)

```lua
-- gateway/init.lua
local _M = {}

local auth = require("gateway.auth")
local rate_limiter = require("gateway.rate_limiter")
local router = require("gateway.router")
local circuit_breaker = require("gateway.circuit_breaker")
local cache = require("gateway.cache")
local transform = require("gateway.transform")
local logger = require("gateway.logger")
local config = require("utils.config")

-- Startup: runs once when nginx starts
function _M.startup()
    -- Load route configuration
    local ok, err = router.load_routes()
    if not ok then
        ngx.log(ngx.CRIT, "Failed to load routes: ", err)
        return
    end
    
    -- Warm up API key cache
    auth.load_api_keys()
    
    ngx.log(ngx.NOTICE, "API Gateway initialized")
end

-- Worker init: runs per nginx worker
function _M.worker_init()
    -- Start background jobs
    local ok, err = ngx.timer.at(0, function()
        -- Health check loop
        require("gateway.health").start_checks()
    end)
    if not ok then
        ngx.log(ngx.ERR, "Failed to start background jobs: ", err)
    end
end

-- Access phase: auth + rate limiting
function _M.access_phase()
    local ctx = ngx.ctx
    ctx.start_time = ngx.now()
    ctx.request_id = generate_request_id()
    
    -- Set request ID header
    ngx.req.set_header("X-Request-ID", ctx.request_id)
    
    -- Find matching route
    local route, err = router.match(ngx.var.uri, ngx.var.request_method)
    if not route then
        return ngx.exit(404)
    end
    ctx.route = route
    
    -- Authentication
    if route.auth_required ~= false then
        local identity, err = auth.authenticate()
        if not identity then
            ngx.status = 401
            ngx.header["Content-Type"] = "application/json"
            ngx.say('{"error":"' .. (err or "Unauthorized") .. '"}')
            return ngx.exit(ngx.HTTP_UNAUTHORIZED)
        end
        ctx.identity = identity
        
        -- Authorization
        if route.required_roles then
            if not auth.has_roles(identity, route.required_roles) then
                ngx.status = 403
                ngx.header["Content-Type"] = "application/json"
                ngx.say('{"error":"Forbidden"}')
                return ngx.exit(ngx.HTTP_FORBIDDEN)
            end
        end
    end
    
    -- Rate limiting
    local key = ctx.identity and ctx.identity.id or ngx.var.remote_addr
    local allowed, retry_after = rate_limiter.check(key, route)
    if not allowed then
        ngx.status = 429
        ngx.header["Content-Type"] = "application/json"
        ngx.header["Retry-After"] = retry_after
        ngx.say('{"error":"Rate limit exceeded","retry_after":' .. retry_after .. '}')
        return ngx.exit(429)
    end
    
    -- Check cache (GET requests only)
    if ngx.var.request_method == "GET" and route.cache_ttl then
        local cached = cache.get(ngx.var.request_uri, ctx.identity)
        if cached then
            ngx.header["Content-Type"] = "application/json"
            ngx.header["X-Cache"] = "HIT"
            ngx.say(cached)
            return ngx.exit(200)
        end
        ctx.should_cache = true
    end
end

-- Proxy phase: route to upstream
function _M.proxy_phase()
    local ctx = ngx.ctx
    local route = ctx.route
    
    -- Circuit breaker check
    local service = route.service
    local cb = circuit_breaker.get(service)
    if cb.state == "open" then
        -- Return cached stale response if available
        local stale = cache.get_stale(ngx.var.request_uri)
        if stale then
            ngx.header["X-Stale-Cache"] = "true"
            ngx.say(stale)
            return ngx.exit(200)
        end
        
        ngx.status = 503
        ngx.header["Content-Type"] = "application/json"
        ngx.say('{"error":"Service temporarily unavailable"}')
        return ngx.exit(503)
    end
    
    -- Get upstream URL
    local upstream = router.get_upstream(service)
    if not upstream then
        ngx.exit(502)
        return
    end
    
    -- Apply request transforms
    transform.request(route.request_transform)
    
    -- Proxy the request
    local res = ngx.location.capture("/proxy", {
        method = ngx.HTTP_GET,
        args = { 
            target = upstream .. ngx.var.request_uri,
        },
        vars = { upstream_url = upstream .. ngx.var.request_uri }
    })
    
    -- Alternative: use ngx.socket for direct proxy
    local ok, err = do_proxy(upstream, route)
    if not ok then
        circuit_breaker.record_failure(service)
        ngx.log(ngx.ERR, "Proxy error: ", err)
        ngx.exit(502)
        return
    end
    
    circuit_breaker.record_success(service)
end

-- Header filter phase
function _M.header_phase()
    local ctx = ngx.ctx
    local route = ctx.route
    
    if route and route.response_headers then
        for k, v in pairs(route.response_headers) do
            ngx.header[k] = v
        end
    end
    
    -- Security headers
    ngx.header["X-Content-Type-Options"] = "nosniff"
    ngx.header["X-Frame-Options"] = "DENY"
    ngx.header["X-XSS-Protection"] = "1; mode=block"
    ngx.header["X-Request-ID"] = ctx.request_id
end

-- Body filter phase  
function _M.body_phase()
    local ctx = ngx.ctx
    local chunk, eof = ngx.arg[1], ngx.arg[2]
    
    if ctx.should_cache and eof then
        -- Collect full body for caching
        ctx.response_body = (ctx.response_body or "") .. (chunk or "")
        
        if eof and ctx.response_body and ngx.status == 200 then
            local route = ctx.route
            cache.set(ngx.var.request_uri, ctx.response_body, 
                      ctx.identity, route.cache_ttl)
        end
    end
end

-- Log phase
function _M.log_phase()
    local ctx = ngx.ctx
    
    logger.log_request({
        request_id = ctx.request_id,
        method = ngx.var.request_method,
        uri = ngx.var.uri,
        status = ngx.status,
        latency = ngx.now() - (ctx.start_time or ngx.now()),
        identity = ctx.identity,
        route = ctx.route,
    })
end

-- Generate unique request ID
function generate_request_id()
    local t = ngx.now() * 1000
    local r = math.random(0, 0xFFFF)
    return string.format("%x%04x", t, r)
end

-- Direct proxy using cosocket
function do_proxy(upstream_host, route)
    local sock = ngx.socket.tcp()
    sock:settimeout(5000)
    
    -- Parse host and port
    local host, port = upstream_host:match("^(.-):(%d+)$")
    if not host then
        host = upstream_host
        port = 80
    end
    
    local ok, err = sock:connect(host, tonumber(port))
    if not ok then
        return nil, "Connect failed: " .. err
    end
    
    -- Build HTTP request
    local headers = {}
    local req_headers = ngx.req.get_headers()
    for k, v in pairs(req_headers) do
        if k:lower() ~= "host" then
            table.insert(headers, k .. ": " .. v)
        end
    end
    table.insert(headers, "Host: " .. host)
    table.insert(headers, "X-Forwarded-For: " .. ngx.var.remote_addr)
    table.insert(headers, "X-Gateway: OpenResty/1.0")
    
    ngx.req.read_body()
    local body = ngx.req.get_body_data()
    
    local request = string.format(
        "%s %s HTTP/1.1\r\n%s\r\nConnection: close\r\n\r\n%s",
        ngx.var.request_method,
        ngx.var.request_uri,
        table.concat(headers, "\r\n"),
        body or ""
    )
    
    local bytes, err = sock:send(request)
    if not bytes then
        return nil, "Send failed: " .. err
    end
    
    -- Read response
    local status_line, err = sock:receive("*l")
    if not status_line then
        return nil, "Receive failed: " .. err
    end
    
    local status = tonumber(status_line:match("HTTP/%d%.%d (%d+)"))
    ngx.status = status
    
    -- Read response headers
    local resp_headers = {}
    while true do
        local line = sock:receive("*l")
        if not line or line == "" then break end
        local name, value = line:match("^([^:]+):%s*(.+)$")
        if name then
            resp_headers[name:lower()] = value
            ngx.header[name] = value
        end
    end
    
    -- Read response body
    local chunks = {}
    while true do
        local chunk, err = sock:receive(8192)
        if chunk then
            table.insert(chunks, chunk)
        else
            break
        end
    end
    
    sock:close()
    
    local resp_body = table.concat(chunks)
    ngx.say(resp_body)
    
    -- Cache response if needed
    local ctx = ngx.ctx
    if ctx.should_cache and status == 200 then
        cache.set(ngx.var.request_uri, resp_body, ctx.identity, 
                  ctx.route and ctx.route.cache_ttl or 60)
    end
    
    return true
end

return _M
```

---

## 99.5 Router (gateway/router.lua)

```lua
-- gateway/router.lua
local _M = {}
local cjson = require("cjson")

local routes = {}
local services = {}

-- Route pattern to regex converter
local function pattern_to_regex(pattern)
    -- Convert :param to named capture
    local regex = pattern:gsub(":([%w_]+)", "([^/]+)")
    -- Convert * to wildcard
    regex = regex:gsub("%*", ".*")
    return "^" .. regex .. "$"
end

function _M.load_routes()
    -- Load from config file
    local file = io.open("/api-gateway/config/routes.json", "r")
    if not file then
        return nil, "Cannot open routes config"
    end
    
    local content = file:read("*a")
    file:close()
    
    local ok, data = pcall(cjson.decode, content)
    if not ok then
        return nil, "Invalid routes JSON: " .. data
    end
    
    -- Compile routes
    routes = {}
    for _, route in ipairs(data.routes or {}) do
        route._regex = pattern_to_regex(route.path)
        table.insert(routes, route)
    end
    
    services = data.services or {}
    
    ngx.log(ngx.NOTICE, "Loaded " .. #routes .. " routes")
    return true
end

function _M.match(uri, method)
    for _, route in ipairs(routes) do
        -- Check method
        if route.methods then
            local method_match = false
            for _, m in ipairs(route.methods) do
                if m == method or m == "*" then
                    method_match = true
                    break
                end
            end
            if not method_match then goto continue end
        end
        
        -- Check path
        local m = ngx.re.match(uri, route._regex, "jo")
        if m then
            -- Extract path params
            local params = {}
            local param_names = {}
            for name in route.path:gmatch(":([%w_]+)") do
                table.insert(param_names, name)
            end
            for i, name in ipairs(param_names) do
                params[name] = m[i]
            end
            
            return {
                id = route.id,
                service = route.service,
                path = route.path,
                strip_prefix = route.strip_prefix,
                add_prefix = route.add_prefix,
                auth_required = route.auth_required,
                required_roles = route.required_roles,
                rate_limit = route.rate_limit,
                cache_ttl = route.cache_ttl,
                request_transform = route.request_transform,
                response_headers = route.response_headers,
                params = params,
            }
        end
        
        ::continue::
    end
    
    return nil, "No route matched"
end

-- Round-robin load balancer
local upstream_indices = {}

function _M.get_upstream(service_name)
    local service = services[service_name]
    if not service then
        return nil, "Unknown service: " .. service_name
    end
    
    local upstreams = service.upstreams
    if not upstreams or #upstreams == 0 then
        return nil, "No upstreams for service: " .. service_name
    end
    
    -- Filter out unhealthy upstreams
    local healthy = {}
    for _, up in ipairs(upstreams) do
        if up.healthy ~= false then
            table.insert(healthy, up)
        end
    end
    
    if #healthy == 0 then
        return nil, "All upstreams unhealthy for: " .. service_name
    end
    
    -- Round-robin
    local idx = upstream_indices[service_name] or 0
    idx = (idx % #healthy) + 1
    upstream_indices[service_name] = idx
    
    return healthy[idx].url
end

return _M
```

---

## 99.6 Rate Limiter - Token Bucket (gateway/rate_limiter.lua)

```lua
-- gateway/rate_limiter.lua
-- Token Bucket algorithm
local _M = {}

local shared = ngx.shared.rate_limits

-- Token bucket: fills at rate tokens/second, max burst capacity
local function token_bucket_check(key, rate, burst)
    local now = ngx.now()
    
    -- Get current state
    local tokens = shared:get(key .. ":tokens")
    local last_time = shared:get(key .. ":time")
    
    if not tokens then
        -- First request: initialize bucket
        tokens = burst
        last_time = now
    end
    
    -- Refill tokens based on elapsed time
    local elapsed = now - last_time
    local new_tokens = math.min(burst, tokens + (elapsed * rate))
    
    if new_tokens < 1 then
        -- Not enough tokens
        local wait_time = math.ceil((1 - new_tokens) / rate)
        return false, wait_time
    end
    
    -- Consume one token
    new_tokens = new_tokens - 1
    
    -- Update state (expire after 2x refill time)
    local ttl = math.ceil((burst / rate) * 2)
    shared:set(key .. ":tokens", new_tokens, ttl)
    shared:set(key .. ":time", now, ttl)
    
    return true, 0
end

-- Sliding window counter
local function sliding_window_check(key, limit, window_seconds)
    local now = ngx.time()
    local window_start = now - window_seconds
    
    -- Use a sorted set simulation with shared dict
    -- Key format: key:TIMESTAMP = count
    local count = 0
    
    -- Count requests in window
    local current = shared:get(key .. ":count") or 0
    local window_key = key .. ":" .. math.floor(now / window_seconds)
    
    shared:incr(window_key, 1, 0, window_seconds * 2)
    
    -- Get counts for current and previous window
    local prev_key = key .. ":" .. (math.floor(now / window_seconds) - 1)
    local prev_count = shared:get(prev_key) or 0
    local curr_count = shared:get(window_key) or 0
    
    -- Weighted average
    local elapsed_in_window = (now % window_seconds) / window_seconds
    local weighted = prev_count * (1 - elapsed_in_window) + curr_count
    
    if weighted >= limit then
        return false, window_seconds - (now % window_seconds)
    end
    
    return true, 0
end

function _M.check(key, route)
    local rate_config = route.rate_limit
    if not rate_config then
        -- Default: 100 req/min
        rate_config = { requests = 100, window = 60 }
    end
    
    -- Determine algorithm
    if rate_config.burst then
        -- Token bucket
        return token_bucket_check(
            "tb:" .. key,
            rate_config.requests / rate_config.window,
            rate_config.burst
        )
    else
        -- Sliding window
        return sliding_window_check(
            "sw:" .. key,
            rate_config.requests,
            rate_config.window
        )
    end
end

-- Get current usage stats
function _M.get_stats(key)
    return {
        token_count = shared:get(key .. ":tokens"),
        last_time = shared:get(key .. ":time"),
    }
end

return _M
```

---

## 99.7 Circuit Breaker (gateway/circuit_breaker.lua)

```lua
-- gateway/circuit_breaker.lua
local _M = {}

local shared = ngx.shared.circuit_breakers

local STATES = {
    CLOSED = "closed",
    OPEN = "open",
    HALF_OPEN = "half_open"
}

local defaults = {
    failure_threshold = 5,
    success_threshold = 2,
    timeout = 30,      -- seconds to wait before trying again
    half_open_max = 3, -- max requests in half-open state
}

local function get_key(service, field)
    return service .. ":" .. field
end

function _M.get(service)
    local state = shared:get(get_key(service, "state")) or STATES.CLOSED
    local failures = shared:get(get_key(service, "failures")) or 0
    local successes = shared:get(get_key(service, "successes")) or 0
    local last_failure = shared:get(get_key(service, "last_failure")) or 0
    
    -- Check if open circuit should transition to half-open
    if state == STATES.OPEN then
        local now = ngx.time()
        if now - last_failure >= defaults.timeout then
            shared:set(get_key(service, "state"), STATES.HALF_OPEN)
            shared:set(get_key(service, "half_open_count"), 0)
            state = STATES.HALF_OPEN
        end
    end
    
    return {
        state = state,
        failures = failures,
        successes = successes,
        last_failure = last_failure,
    }
end

function _M.record_success(service)
    local state = shared:get(get_key(service, "state")) or STATES.CLOSED
    
    if state == STATES.HALF_OPEN then
        local successes = shared:incr(get_key(service, "successes"), 1, 0) or 1
        
        if successes >= defaults.success_threshold then
            -- Transition to closed
            shared:set(get_key(service, "state"), STATES.CLOSED)
            shared:set(get_key(service, "failures"), 0)
            shared:set(get_key(service, "successes"), 0)
            ngx.log(ngx.NOTICE, "Circuit breaker CLOSED for: ", service)
        end
    elseif state == STATES.CLOSED then
        -- Reset failure count on success
        shared:set(get_key(service, "failures"), 0)
    end
end

function _M.record_failure(service)
    local state = shared:get(get_key(service, "state")) or STATES.CLOSED
    
    if state == STATES.HALF_OPEN then
        -- Failure in half-open → back to open
        shared:set(get_key(service, "state"), STATES.OPEN)
        shared:set(get_key(service, "last_failure"), ngx.time())
        ngx.log(ngx.WARN, "Circuit breaker back to OPEN for: ", service)
        return
    end
    
    local failures = shared:incr(get_key(service, "failures"), 1, 0) or 1
    shared:set(get_key(service, "last_failure"), ngx.time())
    
    if failures >= defaults.failure_threshold then
        shared:set(get_key(service, "state"), STATES.OPEN)
        ngx.log(ngx.WARN, "Circuit breaker OPEN for: ", service, 
                " after ", failures, " failures")
    end
end

function _M.reset(service)
    shared:set(get_key(service, "state"), STATES.CLOSED)
    shared:set(get_key(service, "failures"), 0)
    shared:set(get_key(service, "successes"), 0)
end

function _M.get_all_states()
    local states = {}
    -- This would need service list from config
    return states
end

return _M
```

---

## 99.8 Authentication (gateway/auth.lua)

```lua
-- gateway/auth.lua
local _M = {}
local cjson = require("cjson")

local shared = ngx.shared.api_keys

-- JWT verification (simplified - use resty.jwt in production)
local function verify_jwt(token, secret)
    local parts = {}
    for part in token:gmatch("[^%.]+") do
        table.insert(parts, part)
    end
    
    if #parts ~= 3 then
        return nil, "Invalid JWT format"
    end
    
    -- Decode header and payload
    local function b64_decode(s)
        s = s:gsub("-", "+"):gsub("_", "/")
        local padding = 4 - (#s % 4)
        if padding < 4 then
            s = s .. string.rep("=", padding)
        end
        return ngx.decode_base64(s)
    end
    
    local header_json = b64_decode(parts[1])
    local payload_json = b64_decode(parts[2])
    
    if not header_json or not payload_json then
        return nil, "Invalid base64 encoding"
    end
    
    local ok, payload = pcall(cjson.decode, payload_json)
    if not ok then
        return nil, "Invalid payload JSON"
    end
    
    -- Check expiration
    if payload.exp and payload.exp < ngx.time() then
        return nil, "Token expired"
    end
    
    -- Verify signature (HMAC-SHA256)
    local data = parts[1] .. "." .. parts[2]
    local expected_sig = ngx.encode_base64(
        ngx.hmac_sha1(secret, data)  -- Use resty.hmac for SHA256
    ):gsub("+", "-"):gsub("/", "_"):gsub("=", "")
    
    if parts[3] ~= expected_sig then
        return nil, "Invalid signature"
    end
    
    return {
        id = payload.sub,
        email = payload.email,
        roles = payload.roles or {},
        metadata = payload,
    }
end

-- Load API keys from config into shared memory
function _M.load_api_keys()
    local file = io.open("/api-gateway/config/api_keys.json", "r")
    if not file then return end
    
    local content = file:read("*a")
    file:close()
    
    local ok, data = pcall(cjson.decode, content)
    if not ok then return end
    
    for _, key_info in ipairs(data.api_keys or {}) do
        shared:set("key:" .. key_info.key, cjson.encode(key_info))
    end
    
    ngx.log(ngx.NOTICE, "Loaded " .. #(data.api_keys or {}) .. " API keys")
end

function _M.authenticate()
    -- Check Authorization header
    local auth_header = ngx.req.get_headers()["Authorization"]
    
    if auth_header then
        -- Bearer token (JWT)
        local token = auth_header:match("^Bearer%s+(.+)$")
        if token then
            local jwt_secret = os.getenv("JWT_SECRET") or "change-me-in-production"
            local identity, err = verify_jwt(token, jwt_secret)
            if identity then
                return identity
            end
            return nil, err
        end
        
        -- Basic Auth
        local credentials = auth_header:match("^Basic%s+(.+)$")
        if credentials then
            local decoded = ngx.decode_base64(credentials)
            if decoded then
                local username, password = decoded:match("^([^:]+):(.+)$")
                return _M.verify_basic_auth(username, password)
            end
        end
    end
    
    -- Check X-API-Key header
    local api_key = ngx.req.get_headers()["X-API-Key"]
    if api_key then
        return _M.verify_api_key(api_key)
    end
    
    -- Check query parameter (less secure, for legacy)
    local key_param = ngx.var.arg_api_key
    if key_param then
        return _M.verify_api_key(key_param)
    end
    
    return nil, "No authentication credentials provided"
end

function _M.verify_api_key(key)
    local data = shared:get("key:" .. key)
    if not data then
        return nil, "Invalid API key"
    end
    
    local ok, key_info = pcall(cjson.decode, data)
    if not ok then
        return nil, "Key data corrupted"
    end
    
    -- Check if key is active
    if key_info.disabled then
        return nil, "API key disabled"
    end
    
    -- Check expiration
    if key_info.expires_at and key_info.expires_at < ngx.time() then
        return nil, "API key expired"
    end
    
    return {
        id = key_info.id,
        name = key_info.name,
        roles = key_info.roles or {},
        type = "api_key",
    }
end

function _M.verify_basic_auth(username, password)
    -- In production, verify against database
    -- This is a simple in-memory example
    local users = {
        admin = { password = "secret", roles = {"admin"} },
    }
    
    local user = users[username]
    if not user or user.password ~= password then
        return nil, "Invalid credentials"
    end
    
    return {
        id = username,
        name = username,
        roles = user.roles,
        type = "basic",
    }
end

function _M.has_roles(identity, required_roles)
    local user_roles = {}
    for _, role in ipairs(identity.roles or {}) do
        user_roles[role] = true
    end
    
    for _, required in ipairs(required_roles) do
        if not user_roles[required] and not user_roles["admin"] then
            return false
        end
    end
    
    return true
end

return _M
```

---

## 99.9 Cache Layer (gateway/cache.lua)

```lua
-- gateway/cache.lua
local _M = {}
local cjson = require("cjson")

local shared = ngx.shared.cache_store

local function make_key(uri, identity)
    local user_id = identity and identity.id or "anonymous"
    return "cache:" .. user_id .. ":" .. ngx.md5(uri)
end

local function make_stale_key(uri)
    return "stale:" .. ngx.md5(uri)
end

function _M.get(uri, identity)
    local key = make_key(uri, identity)
    local data = shared:get(key)
    if not data then return nil end
    
    local ok, entry = pcall(cjson.decode, data)
    if not ok then return nil end
    
    -- Update hit stats
    ngx.header["X-Cache"] = "HIT"
    ngx.header["X-Cache-Age"] = tostring(ngx.time() - entry.cached_at)
    
    return entry.body
end

function _M.get_stale(uri)
    local key = make_stale_key(uri)
    local data = shared:get(key)
    if not data then return nil end
    
    local ok, entry = pcall(cjson.decode, data)
    if not ok then return nil end
    
    return entry.body
end

function _M.set(uri, body, identity, ttl)
    ttl = ttl or 60
    
    local entry = cjson.encode({
        body = body,
        cached_at = ngx.time(),
        ttl = ttl,
    })
    
    -- Set main cache
    local key = make_key(uri, identity)
    shared:set(key, entry, ttl)
    
    -- Set stale cache (longer TTL for circuit breaker fallback)
    local stale_key = make_stale_key(uri)
    shared:set(stale_key, entry, ttl * 10)
    
    ngx.header["X-Cache"] = "MISS"
end

function _M.invalidate(uri, identity)
    local key = make_key(uri, identity)
    shared:delete(key)
end

function _M.flush()
    shared:flush_all()
    ngx.log(ngx.NOTICE, "Cache flushed")
end

function _M.get_stats()
    return {
        capacity = shared:capacity(),
        free_space = shared:free_space(),
    }
end

return _M
```

---

## 99.10 Request/Response Transformer (gateway/transform.lua)

```lua
-- gateway/transform.lua
local _M = {}
local cjson = require("cjson")

-- Apply request transformations
function _M.request(transforms)
    if not transforms then return end
    
    -- Add/modify headers
    if transforms.add_headers then
        for k, v in pairs(transforms.add_headers) do
            -- Support variable substitution
            v = _M.interpolate(v)
            ngx.req.set_header(k, v)
        end
    end
    
    -- Remove headers
    if transforms.remove_headers then
        for _, k in ipairs(transforms.remove_headers) do
            ngx.req.clear_header(k)
        end
    end
    
    -- Rewrite URL
    if transforms.rewrite then
        local uri = ngx.var.uri
        for pattern, replacement in pairs(transforms.rewrite) do
            uri = uri:gsub(pattern, replacement)
        end
        ngx.req.set_uri(uri, false)
    end
    
    -- Add query params
    if transforms.add_params then
        local args = ngx.req.get_uri_args()
        for k, v in pairs(transforms.add_params) do
            args[k] = _M.interpolate(v)
        end
        ngx.req.set_uri_args(args)
    end
    
    -- Transform JSON body
    if transforms.body_template then
        ngx.req.read_body()
        local body = ngx.req.get_body_data()
        if body then
            local ok, data = pcall(cjson.decode, body)
            if ok then
                local new_body = _M.transform_body(data, transforms.body_template)
                ngx.req.set_body_data(cjson.encode(new_body))
            end
        end
    end
end

-- Variable interpolation: replace ${var} with values
function _M.interpolate(str)
    return str:gsub("%${([^}]+)}", function(var)
        if var == "request_id" then
            return ngx.ctx.request_id or ""
        elseif var == "remote_addr" then
            return ngx.var.remote_addr or ""
        elseif var == "user_id" then
            local identity = ngx.ctx.identity
            return identity and identity.id or ""
        elseif var:sub(1, 4) == "env." then
            return os.getenv(var:sub(5)) or ""
        end
        return str
    end)
end

-- Transform body using template
function _M.transform_body(data, template)
    if type(template) == "table" then
        local result = {}
        for k, v in pairs(template) do
            if type(v) == "string" and v:sub(1, 1) == "$" then
                -- Reference to original field
                local field = v:sub(2)
                result[k] = data[field]
            elseif type(v) == "table" then
                result[k] = _M.transform_body(data, v)
            else
                result[k] = v
            end
        end
        return result
    end
    return data
end

return _M
```

---

## 99.11 Health Checks (gateway/health.lua)

```lua
-- gateway/health.lua
local _M = {}
local cjson = require("cjson")

local shared = ngx.shared.gateway_stats

-- Service health status
local health_status = {}

-- Background health check loop
function _M.start_checks()
    local router = require("gateway.router")
    
    local function check_loop()
        while true do
            _M.check_all_services()
            ngx.sleep(10)  -- Check every 10 seconds
        end
    end
    
    local ok, err = ngx.timer.at(0, check_loop)
    if not ok then
        ngx.log(ngx.ERR, "Failed to start health check loop: ", err)
    end
end

function _M.check_all_services()
    -- This would check each upstream
    local services = {
        { name = "user-service", url = "http://localhost:8081/health" },
        { name = "order-service", url = "http://localhost:8082/health" },
        { name = "product-service", url = "http://localhost:8083/health" },
    }
    
    for _, service in ipairs(services) do
        local ok, status = _M.ping_service(service.url)
        
        if health_status[service.name] ~= ok then
            if ok then
                ngx.log(ngx.NOTICE, "Service ", service.name, " is now UP")
            else
                ngx.log(ngx.WARN, "Service ", service.name, " is DOWN")
            end
        end
        
        health_status[service.name] = ok
        
        -- Update circuit breaker based on health
        if ok then
            require("gateway.circuit_breaker").record_success(service.name)
        end
    end
end

function _M.ping_service(url)
    local http = require("resty.http")
    local httpc = http.new()
    httpc:set_timeout(2000)
    
    local res, err = httpc:request_uri(url, {
        method = "GET",
        headers = { ["User-Agent"] = "Gateway-HealthCheck/1.0" }
    })
    
    if not res then
        return false, err
    end
    
    return res.status == 200, res.status
end

-- Health check endpoint
function _M.check()
    local circuit_breaker = require("gateway.circuit_breaker")
    local cache = require("gateway.cache")
    
    local overall_status = "healthy"
    local services_health = {}
    
    for name, is_healthy in pairs(health_status) do
        if not is_healthy then
            overall_status = "degraded"
        end
        services_health[name] = {
            status = is_healthy and "up" or "down",
        }
    end
    
    local response = {
        status = overall_status,
        timestamp = ngx.time(),
        version = "1.0.0",
        services = services_health,
        cache = cache.get_stats(),
    }
    
    ngx.header["Content-Type"] = "application/json"
    if overall_status == "healthy" then
        ngx.status = 200
    else
        ngx.status = 207
    end
    
    ngx.say(cjson.encode(response))
end

-- Prometheus metrics endpoint
function _M.metrics()
    local lines = {}
    
    -- Request counters
    local total_requests = shared:get("total_requests") or 0
    table.insert(lines, "# HELP gateway_requests_total Total requests processed")
    table.insert(lines, "# TYPE gateway_requests_total counter")
    table.insert(lines, "gateway_requests_total " .. total_requests)
    
    -- Error rate
    local total_errors = shared:get("total_errors") or 0
    table.insert(lines, "# HELP gateway_errors_total Total errors")
    table.insert(lines, "# TYPE gateway_errors_total counter")
    table.insert(lines, "gateway_errors_total " .. total_errors)
    
    -- Latency (would use histogram in production)
    local avg_latency = shared:get("avg_latency") or 0
    table.insert(lines, "# HELP gateway_latency_seconds Average request latency")
    table.insert(lines, "# TYPE gateway_latency_seconds gauge")
    table.insert(lines, "gateway_latency_seconds " .. avg_latency)
    
    -- Service health
    for name, is_healthy in pairs(health_status) do
        table.insert(lines, string.format(
            'gateway_service_up{service="%s"} %d',
            name, is_healthy and 1 or 0
        ))
    end
    
    ngx.header["Content-Type"] = "text/plain; version=0.0.4"
    ngx.say(table.concat(lines, "\n"))
end

return _M
```

---

## 99.12 Configuration Files

```json
// config/routes.json
{
  "routes": [
    {
      "id": "user-list",
      "path": "/api/v1/users",
      "methods": ["GET", "POST"],
      "service": "user-service",
      "auth_required": true,
      "rate_limit": {
        "requests": 100,
        "window": 60,
        "burst": 20
      },
      "cache_ttl": 30,
      "response_headers": {
        "X-API-Version": "v1"
      }
    },
    {
      "id": "user-detail",
      "path": "/api/v1/users/:id",
      "methods": ["GET", "PUT", "DELETE"],
      "service": "user-service",
      "auth_required": true,
      "required_roles": ["user", "admin"],
      "request_transform": {
        "add_headers": {
          "X-User-From-Gateway": "${user_id}",
          "X-Request-ID": "${request_id}"
        }
      }
    },
    {
      "id": "orders",
      "path": "/api/v1/orders",
      "methods": ["GET", "POST"],
      "service": "order-service",
      "auth_required": true,
      "rate_limit": {
        "requests": 50,
        "window": 60
      }
    },
    {
      "id": "products",
      "path": "/api/v1/products",
      "methods": ["GET"],
      "service": "product-service",
      "auth_required": false,
      "cache_ttl": 300
    },
    {
      "id": "admin",
      "path": "/api/v1/admin/*",
      "methods": ["*"],
      "service": "admin-service",
      "auth_required": true,
      "required_roles": ["admin"],
      "rate_limit": {
        "requests": 1000,
        "window": 60
      }
    }
  ],
  "services": {
    "user-service": {
      "upstreams": [
        { "url": "http://user-service-1:8081", "healthy": true },
        { "url": "http://user-service-2:8081", "healthy": true }
      ]
    },
    "order-service": {
      "upstreams": [
        { "url": "http://order-service:8082", "healthy": true }
      ]
    },
    "product-service": {
      "upstreams": [
        { "url": "http://product-service:8083", "healthy": true }
      ]
    },
    "admin-service": {
      "upstreams": [
        { "url": "http://admin-service:8084", "healthy": true }
      ]
    }
  }
}
```

---

## 99.13 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  gateway:
    image: openresty/openresty:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf
      - ./lua:/usr/local/openresty/lua
      - ./config:/api-gateway/config
      - ./logs:/usr/local/openresty/nginx/logs
    environment:
      - JWT_SECRET=your-super-secret-key-change-in-production
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - redis
    networks:
      - gateway-net

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    networks:
      - gateway-net

  user-service:
    image: your-user-service:latest
    networks:
      - gateway-net
    deploy:
      replicas: 2

  order-service:
    image: your-order-service:latest
    networks:
      - gateway-net

  product-service:
    image: your-product-service:latest
    networks:
      - gateway-net

volumes:
  redis-data:

networks:
  gateway-net:
    driver: bridge
```

---

## 99.14 Testing the Gateway

```bash
# Start the gateway
docker-compose up -d

# Test health check
curl http://localhost/health

# Test without auth (should return 401)
curl http://localhost/api/v1/users

# Test with API key
curl -H "X-API-Key: test-key-123" http://localhost/api/v1/users

# Test with JWT
TOKEN=$(curl -s -X POST http://localhost/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"secret"}' | jq -r '.token')

curl -H "Authorization: Bearer $TOKEN" http://localhost/api/v1/users

# Test rate limiting (run many times quickly)
for i in $(seq 1 120); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "X-API-Key: test-key-123" \
    http://localhost/api/v1/products
done

# Check metrics
curl http://localhost/metrics
```

---

## 99.15 Performance Tuning

```lua
-- Performance optimizations for production

-- 1. Connection pooling
local function get_upstream_with_pool(host, port)
    local sock = ngx.socket.tcp()
    sock:settimeout(1000)
    
    -- Use connection pool
    local ok, err = sock:connect(host, port)
    if not ok then return nil, err end
    
    -- Keep connection in pool after use
    -- sock:setkeepalive(60000, 100)  -- 60s timeout, 100 connections
    
    return sock
end

-- 2. LuaJIT optimization hints
local floor = math.floor      -- Localize for JIT
local format = string.format  -- Localize for JIT
local ngx_time = ngx.time     -- Localize ngx functions

-- 3. Avoid table creation in hot paths
local _reuse_table = {}

local function parse_headers_fast(raw)
    -- Reuse table instead of creating new one each time
    for k in pairs(_reuse_table) do _reuse_table[k] = nil end
    
    for line in raw:gmatch("[^\r\n]+") do
        local k, v = line:match("^([^:]+):%s*(.+)$")
        if k then _reuse_table[k:lower()] = v end
    end
    
    return _reuse_table
end

-- 4. Pre-compile patterns
local BEARER_PATTERN = "^Bearer%s+(.+)$"
local URI_PATTERN = "^/api/v(%d+)/(.+)$"

-- 5. Batch Redis operations
local function batch_rate_limit_check(keys, redis_client)
    local pipe = redis_client:pipeline()
    for _, key in ipairs(keys) do
        pipe:get(key)
        pipe:ttl(key)
    end
    return pipe:exec()
end
```

---

## แบบฝึกหัด

1. **Plugin System**: สร้างระบบ plugin ที่ให้เพิ่ม middleware แบบ dynamic ได้
2. **OAuth2**: เพิ่ม OAuth2 authentication flow
3. **GraphQL**: รองรับ GraphQL query routing
4. **WebSocket**: รองรับ WebSocket proxying
5. **Admin UI**: สร้าง dashboard สำหรับ manage routes และ monitor traffic

---

*ต่อไป: [Part 100 - Capstone Project & Career Path](part-100.md)*
