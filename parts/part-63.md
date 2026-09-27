# บทที่ 63: Microservices Architecture

## บทนำ

Microservices Architecture คือรูปแบบการออกแบบซอฟต์แวร์ที่แบ่ง application ออกเป็น services เล็กๆ ที่ทำงานอิสระ สื่อสารผ่าน API (HTTP/gRPC/message queue) บทนี้จะครอบคลุมการนำ Lua ไปใช้ใน microservices context โดยใช้ OpenResty/NGINX หรือ frameworks อื่นๆ

## 1. Monolith vs Microservices

```lua
-- Example 1: Understanding the difference conceptually

-- Monolith: ทุกอย่างอยู่ใน single process
local MonolithApp = {
    -- User module
    users = {
        create = function(data) end,
        getById = function(id) end,
    },
    -- Order module
    orders = {
        create = function(userId, items) end,
        getByUser = function(userId) end,
    },
    -- Payment module
    payments = {
        process = function(orderId, amount) end,
        refund = function(paymentId) end,
    },
    -- Notification module
    notifications = {
        sendEmail = function(to, subject, body) end,
        sendSMS = function(to, message) end,
    }
}

-- Microservices: แต่ละส่วนเป็น service แยกกัน
-- user-service       -> :3001
-- order-service      -> :3002
-- payment-service    -> :3003
-- notification-service -> :3004
-- api-gateway        -> :8080

-- ข้อดีของ Microservices:
-- 1. Scale แต่ละ service ได้อิสระ
-- 2. Deploy แยกกัน (CI/CD per service)
-- 3. ใช้ technology ที่เหมาะสมกับแต่ละ service
-- 4. Fault isolation (service หนึ่ง fail ไม่กระทบทั้งหมด)
-- 5. Team autonomy

-- ข้อเสีย:
-- 1. Distributed systems complexity
-- 2. Network latency
-- 3. Data consistency challenges
-- 4. More infrastructure overhead

print("Microservices vs Monolith: tradeoffs!")
```

## 2. HTTP Client สำหรับ Service Communication

```lua
-- Example 2: HTTP client for inter-service communication
-- ใช้ socket.http (LuaSocket) หรือ lua-resty-http ใน OpenResty

-- LuaSocket HTTP client
local http = require("socket.http")
local ltn12 = require("ltn12")
local json = require("dkjson")  -- หรือ cjson

local HttpClient = {}
HttpClient.__index = HttpClient

function HttpClient.new(baseUrl, timeout)
    return setmetatable({
        baseUrl = baseUrl,
        timeout = timeout or 5,
        defaultHeaders = {
            ["Content-Type"] = "application/json",
            ["Accept"] = "application/json",
        }
    }, HttpClient)
end

function HttpClient:request(method, path, body, headers)
    local url = self.baseUrl .. path
    local responseBody = {}
    local requestHeaders = {}
    
    -- Merge headers
    for k, v in pairs(self.defaultHeaders) do
        requestHeaders[k] = v
    end
    if headers then
        for k, v in pairs(headers) do
            requestHeaders[k] = v
        end
    end
    
    local bodyStr
    if body then
        bodyStr = json.encode(body)
        requestHeaders["Content-Length"] = #bodyStr
    end
    
    local result, statusCode, responseHeaders, statusLine = http.request({
        url = url,
        method = method,
        headers = requestHeaders,
        source = bodyStr and ltn12.source.string(bodyStr),
        sink = ltn12.sink.table(responseBody),
    })
    
    local responseStr = table.concat(responseBody)
    local responseData
    
    if responseStr and #responseStr > 0 then
        local ok, decoded = pcall(json.decode, responseStr)
        if ok then responseData = decoded end
    end
    
    return {
        status = statusCode,
        body = responseData,
        rawBody = responseStr,
        headers = responseHeaders,
        ok = statusCode and statusCode >= 200 and statusCode < 300
    }
end

function HttpClient:get(path, headers)
    return self:request("GET", path, nil, headers)
end

function HttpClient:post(path, body, headers)
    return self:request("POST", path, body, headers)
end

function HttpClient:put(path, body, headers)
    return self:request("PUT", path, body, headers)
end

function HttpClient:delete(path, headers)
    return self:request("DELETE", path, nil, headers)
end

-- การใช้งาน
local userService = HttpClient.new("http://user-service:3001")
local orderService = HttpClient.new("http://order-service:3002")

-- ดึงข้อมูล user
local response = userService:get("/api/users/123")
if response.ok then
    print("User:", response.body.name)
else
    print("Error:", response.status, response.rawBody)
end

-- สร้าง order
local orderResp = orderService:post("/api/orders", {
    userId = "123",
    items = {
        {productId = "456", quantity = 2, price = 100}
    }
})

if orderResp.ok then
    print("Order created:", orderResp.body.orderId)
end
```

## 3. Simple HTTP Server ด้วย LuaSocket

```lua
-- Example 3: Simple HTTP server in Lua
local socket = require("socket")
local json = require("dkjson")

local Server = {}
Server.__index = Server

function Server.new(host, port)
    local self = setmetatable({}, Server)
    self.host = host or "0.0.0.0"
    self.port = port or 8080
    self.routes = {}
    self.middleware = {}
    return self
end

function Server:use(fn)
    table.insert(self.middleware, fn)
end

function Server:addRoute(method, path, handler)
    local pattern = "^" .. path:gsub(":([%w_]+)", "([^/]+)") .. "$"
    local params = {}
    for param in path:gmatch(":([%w_]+)") do
        table.insert(params, param)
    end
    table.insert(self.routes, {
        method = method:upper(),
        pattern = pattern,
        params = params,
        handler = handler
    })
end

function Server:get(path, handler) self:addRoute("GET", path, handler) end
function Server:post(path, handler) self:addRoute("POST", path, handler) end
function Server:put(path, handler) self:addRoute("PUT", path, handler) end
function Server:delete(path, handler) self:addRoute("DELETE", path, handler) end

function Server:parseRequest(client)
    local request = {}
    local headers = {}
    
    -- Read first line
    local line = client:receive("*l")
    if not line then return nil end
    
    local method, path, version = line:match("^(%w+)%s+(/[^%s]*)%s+(HTTP/%d+%.%d+)$")
    if not method then return nil end
    
    -- Parse query string
    local queryString = path:match("%?(.+)$")
    path = path:match("^([^?]+)")
    
    request.method = method
    request.path = path
    request.query = {}
    
    if queryString then
        for key, val in queryString:gmatch("([^&=]+)=([^&]*)") do
            request.query[key] = val
        end
    end
    
    -- Read headers
    while true do
        line = client:receive("*l")
        if not line or line == "" then break end
        local key, val = line:match("^([^:]+):%s*(.+)$")
        if key then headers[key:lower()] = val end
    end
    
    request.headers = headers
    
    -- Read body
    local contentLength = tonumber(headers["content-length"])
    if contentLength and contentLength > 0 then
        local body = client:receive(contentLength)
        request.rawBody = body
        
        local ct = headers["content-type"] or ""
        if ct:find("application/json") then
            local ok, decoded = pcall(json.decode, body)
            if ok then request.body = decoded end
        end
    end
    
    return request
end

function Server:sendResponse(client, response)
    local statusMessages = {
        [200] = "OK",
        [201] = "Created",
        [204] = "No Content",
        [400] = "Bad Request",
        [401] = "Unauthorized",
        [403] = "Forbidden",
        [404] = "Not Found",
        [429] = "Too Many Requests",
        [500] = "Internal Server Error",
        [503] = "Service Unavailable",
    }
    
    local statusCode = response.status or 200
    local statusMsg = statusMessages[statusCode] or "OK"
    local body = response.body
    
    if type(body) == "table" then
        body = json.encode(body)
    elseif body == nil then
        body = ""
    end
    
    local headers = response.headers or {}
    headers["Content-Length"] = #body
    headers["Content-Type"] = headers["Content-Type"] or "application/json"
    headers["Connection"] = "close"
    
    client:send("HTTP/1.1 " .. statusCode .. " " .. statusMsg .. "\r\n")
    for k, v in pairs(headers) do
        client:send(k .. ": " .. v .. "\r\n")
    end
    client:send("\r\n")
    if #body > 0 then
        client:send(body)
    end
end

function Server:handleRequest(client)
    local req = self:parseRequest(client)
    if not req then
        self:sendResponse(client, {status = 400, body = {error = "Bad Request"}})
        return
    end
    
    -- Run middleware
    local ctx = {req = req, res = {status = 200, headers = {}}}
    for _, mw in ipairs(self.middleware) do
        local stop = mw(ctx)
        if stop then
            self:sendResponse(client, ctx.res)
            return
        end
    end
    
    -- Match route
    for _, route in ipairs(self.routes) do
        if route.method == req.method then
            local matches = {req.path:match(route.pattern)}
            if #matches > 0 then
                req.params = {}
                for i, paramName in ipairs(route.params) do
                    req.params[paramName] = matches[i]
                end
                
                local ok, response = pcall(route.handler, req)
                if ok then
                    self:sendResponse(client, response)
                else
                    print("Handler error: " .. tostring(response))
                    self:sendResponse(client, {
                        status = 500,
                        body = {error = "Internal Server Error"}
                    })
                end
                return
            end
        end
    end
    
    self:sendResponse(client, {status = 404, body = {error = "Not Found", path = req.path}})
end

function Server:start()
    local server = socket.bind(self.host, self.port)
    server:settimeout(0)
    
    print("Server listening on " .. self.host .. ":" .. self.port)
    
    while true do
        local client = server:accept()
        if client then
            client:settimeout(5)
            local ok, err = pcall(function()
                self:handleRequest(client)
            end)
            if not ok then print("Request error: " .. tostring(err)) end
            client:close()
        end
        socket.sleep(0.001)
    end
end

-- User Service Example
local app = Server.new("0.0.0.0", 3001)

-- Logging middleware
app:use(function(ctx)
    local start = os.time()
    print(string.format("[%s] %s %s",
        os.date("%H:%M:%S"),
        ctx.req.method,
        ctx.req.path
    ))
end)

-- In-memory data store (ใน production ใช้ database จริง)
local usersDb = {
    ["1"] = {id = "1", name = "Alice", email = "alice@example.com"},
    ["2"] = {id = "2", name = "Bob",   email = "bob@example.com"},
}
local nextUserId = 3

-- Routes
app:get("/health", function(req)
    return {status = 200, body = {status = "healthy", service = "user-service"}}
end)

app:get("/api/users", function(req)
    local users = {}
    for _, u in pairs(usersDb) do
        table.insert(users, u)
    end
    return {status = 200, body = {users = users, total = #users}}
end)

app:get("/api/users/:id", function(req)
    local user = usersDb[req.params.id]
    if not user then
        return {status = 404, body = {error = "User not found"}}
    end
    return {status = 200, body = user}
end)

app:post("/api/users", function(req)
    if not req.body or not req.body.name or not req.body.email then
        return {status = 400, body = {error = "name and email required"}}
    end
    
    local id = tostring(nextUserId)
    nextUserId = nextUserId + 1
    
    local user = {
        id = id,
        name = req.body.name,
        email = req.body.email,
        createdAt = os.time()
    }
    usersDb[id] = user
    
    return {status = 201, body = user}
end)

app:put("/api/users/:id", function(req)
    local user = usersDb[req.params.id]
    if not user then
        return {status = 404, body = {error = "User not found"}}
    end
    
    if req.body then
        if req.body.name then user.name = req.body.name end
        if req.body.email then user.email = req.body.email end
        user.updatedAt = os.time()
    end
    
    return {status = 200, body = user}
end)

app:delete("/api/users/:id", function(req)
    if not usersDb[req.params.id] then
        return {status = 404, body = {error = "User not found"}}
    end
    usersDb[req.params.id] = nil
    return {status = 204}
end)

-- app:start()  -- uncomment to run
print("User service defined on port 3001")
```

## 4. API Gateway Pattern

```lua
-- Example 4: API Gateway
-- API Gateway เป็น single entry point สำหรับ clients
-- ทำหน้าที่: routing, auth, rate limiting, logging, circuit breaking

local Gateway = {}
Gateway.__index = Gateway

function Gateway.new()
    local self = setmetatable({}, Gateway)
    self.routes = {}
    self.services = {}
    self.circuitBreakers = {}
    return self
end

-- Register backend service
function Gateway:registerService(name, config)
    self.services[name] = {
        name = name,
        url = config.url,
        timeout = config.timeout or 5,
        healthPath = config.healthPath or "/health",
        healthy = true,
        lastCheck = 0
    }
    self.circuitBreakers[name] = {
        failures = 0,
        threshold = config.cbThreshold or 5,
        timeout = config.cbTimeout or 30,
        state = "closed",  -- closed, open, half-open
        lastFailure = 0
    }
end

-- Add route mapping
function Gateway:route(pattern, serviceName, options)
    table.insert(self.routes, {
        pattern = pattern,
        service = serviceName,
        stripPrefix = options and options.stripPrefix,
        addPrefix = options and options.addPrefix,
        requireAuth = options and options.requireAuth,
        rateLimit = options and options.rateLimit
    })
end

function Gateway:getCircuitBreaker(serviceName)
    return self.circuitBreakers[serviceName]
end

function Gateway:isCircuitOpen(serviceName)
    local cb = self.circuitBreakers[serviceName]
    if not cb then return false end
    
    if cb.state == "open" then
        -- Check if timeout elapsed
        if os.time() - cb.lastFailure > cb.timeout then
            cb.state = "half-open"
            print("[CB] Circuit half-open for " .. serviceName)
            return false
        end
        return true
    end
    return false
end

function Gateway:recordSuccess(serviceName)
    local cb = self.circuitBreakers[serviceName]
    if not cb then return end
    
    if cb.state == "half-open" then
        cb.state = "closed"
        cb.failures = 0
        print("[CB] Circuit closed for " .. serviceName)
    end
    cb.failures = 0
end

function Gateway:recordFailure(serviceName)
    local cb = self.circuitBreakers[serviceName]
    if not cb then return end
    
    cb.failures = cb.failures + 1
    cb.lastFailure = os.time()
    
    if cb.failures >= cb.threshold then
        cb.state = "open"
        print("[CB] Circuit OPEN for " .. serviceName .. " after " .. cb.failures .. " failures")
    end
end

function Gateway:proxyRequest(route, req)
    local serviceName = route.service
    
    -- Circuit breaker check
    if self:isCircuitOpen(serviceName) then
        return {
            status = 503,
            body = {
                error = "Service Unavailable",
                reason = "Circuit breaker open",
                service = serviceName
            }
        }
    end
    
    local service = self.services[serviceName]
    if not service then
        return {status = 404, body = {error = "Service not configured: " .. serviceName}}
    end
    
    -- Construct target path
    local targetPath = req.path
    if route.stripPrefix then
        targetPath = targetPath:gsub("^" .. route.stripPrefix, "")
        if targetPath == "" then targetPath = "/" end
    end
    if route.addPrefix then
        targetPath = route.addPrefix .. targetPath
    end
    
    -- Forward request (สมมุติ)
    local targetUrl = service.url .. targetPath
    
    -- In real implementation ใช้ http client ส่ง request ไปยัง service
    print(string.format("[Gateway] %s %s -> %s", req.method, req.path, targetUrl))
    
    -- Simulate response
    local success = math.random() > 0.1  -- 90% success rate
    
    if success then
        self:recordSuccess(serviceName)
        return {
            status = 200,
            body = {
                proxied = true,
                service = serviceName,
                path = targetPath
            }
        }
    else
        self:recordFailure(serviceName)
        return {
            status = 502,
            body = {error = "Bad Gateway", service = serviceName}
        }
    end
end

-- ตัวอย่างการตั้งค่า Gateway
local gw = Gateway.new()

gw:registerService("users", {
    url = "http://user-service:3001",
    timeout = 3,
    cbThreshold = 5,
    cbTimeout = 30
})

gw:registerService("orders", {
    url = "http://order-service:3002",
    timeout = 5,
    cbThreshold = 3,
    cbTimeout = 60
})

gw:registerService("payments", {
    url = "http://payment-service:3003",
    timeout = 10,
    cbThreshold = 3,
    cbTimeout = 120
})

-- Route configuration
gw:route("/api/users", "users", {requireAuth = true})
gw:route("/api/users/*", "users", {requireAuth = true})
gw:route("/api/orders", "orders", {requireAuth = true, rateLimit = "100/min"})
gw:route("/api/payments", "payments", {requireAuth = true, rateLimit = "20/min"})

print("API Gateway configured")
print("Services: users, orders, payments")
```

## 5. Service Discovery

```lua
-- Example 5: Service discovery (simple in-memory registry)
local ServiceRegistry = {}
ServiceRegistry.__index = ServiceRegistry

function ServiceRegistry.new()
    return setmetatable({
        services = {},  -- name -> list of instances
        heartbeatTimeout = 30  -- seconds
    }, ServiceRegistry)
end

-- Instance registration
function ServiceRegistry:register(name, instance)
    if not self.services[name] then
        self.services[name] = {}
    end
    
    -- Check if already registered
    for i, inst in ipairs(self.services[name]) do
        if inst.host == instance.host and inst.port == instance.port then
            -- Update heartbeat
            inst.lastHeartbeat = os.time()
            inst.healthy = true
            return inst.id
        end
    end
    
    local id = name .. "-" .. instance.host .. "-" .. instance.port
    table.insert(self.services[name], {
        id = id,
        name = name,
        host = instance.host,
        port = instance.port,
        metadata = instance.metadata or {},
        registeredAt = os.time(),
        lastHeartbeat = os.time(),
        healthy = true,
        weight = instance.weight or 1
    })
    
    print("[Registry] Registered: " .. id)
    return id
end

-- Deregister service
function ServiceRegistry:deregister(name, host, port)
    if not self.services[name] then return end
    
    for i, inst in ipairs(self.services[name]) do
        if inst.host == host and inst.port == port then
            table.remove(self.services[name], i)
            print("[Registry] Deregistered: " .. name .. " @ " .. host .. ":" .. port)
            return
        end
    end
end

-- Heartbeat
function ServiceRegistry:heartbeat(name, host, port)
    if not self.services[name] then return false end
    
    for _, inst in ipairs(self.services[name]) do
        if inst.host == host and inst.port == port then
            inst.lastHeartbeat = os.time()
            inst.healthy = true
            return true
        end
    end
    return false
end

-- Get service instances (load balancing)
function ServiceRegistry:getInstances(name)
    local all = self.services[name] or {}
    local healthy = {}
    local now = os.time()
    
    for _, inst in ipairs(all) do
        -- Check heartbeat timeout
        if now - inst.lastHeartbeat <= self.heartbeatTimeout then
            table.insert(healthy, inst)
        else
            inst.healthy = false
        end
    end
    
    return healthy
end

-- Round-robin load balancing
local rrCounters = {}

function ServiceRegistry:getOne(name)
    local instances = self:getInstances(name)
    if #instances == 0 then return nil end
    
    if #instances == 1 then return instances[1] end
    
    -- Round-robin
    rrCounters[name] = rrCounters[name] or 0
    rrCounters[name] = (rrCounters[name] % #instances) + 1
    return instances[rrCounters[name]]
end

-- Weighted load balancing
function ServiceRegistry:getOneWeighted(name)
    local instances = self:getInstances(name)
    if #instances == 0 then return nil end
    
    local totalWeight = 0
    for _, inst in ipairs(instances) do
        totalWeight = totalWeight + inst.weight
    end
    
    local rand = math.random() * totalWeight
    local cumulative = 0
    
    for _, inst in ipairs(instances) do
        cumulative = cumulative + inst.weight
        if rand <= cumulative then
            return inst
        end
    end
    
    return instances[#instances]
end

-- Clean up dead instances
function ServiceRegistry:cleanup()
    local now = os.time()
    local removed = 0
    
    for name, instances in pairs(self.services) do
        for i = #instances, 1, -1 do
            if now - instances[i].lastHeartbeat > self.heartbeatTimeout * 2 then
                print("[Registry] Removing dead instance: " .. instances[i].id)
                table.remove(instances, i)
                removed = removed + 1
            end
        end
    end
    
    return removed
end

function ServiceRegistry:listAll()
    local result = {}
    for name, instances in pairs(self.services) do
        result[name] = {
            total = #instances,
            healthy = #self:getInstances(name),
            instances = instances
        }
    end
    return result
end

-- การใช้งาน
local registry = ServiceRegistry.new()

-- Services register themselves on startup
registry:register("user-service", {host = "10.0.0.1", port = 3001, weight = 2})
registry:register("user-service", {host = "10.0.0.2", port = 3001, weight = 1})
registry:register("order-service", {host = "10.0.0.3", port = 3002})
registry:register("payment-service", {host = "10.0.0.4", port = 3003})

-- ค้นหา service
local userInstance = registry:getOne("user-service")
if userInstance then
    print("Calling user service at: " .. userInstance.host .. ":" .. userInstance.port)
end

-- Simulated heartbeats
for i = 1, 5 do
    registry:heartbeat("user-service", "10.0.0.1", 3001)
end

print("\nService Registry:")
for name, info in pairs(registry:listAll()) do
    print(string.format("  %s: %d total, %d healthy", name, info.total, info.healthy))
end
```

## 6. Health Checks

```lua
-- Example 6: Health check system
local HealthCheck = {}
HealthCheck.__index = HealthCheck

function HealthCheck.new()
    return setmetatable({
        checks = {},
        results = {},
        lastRun = 0
    }, HealthCheck)
end

-- เพิ่ม health check
function HealthCheck:addCheck(name, checkFn, critical)
    self.checks[name] = {
        fn = checkFn,
        critical = critical or false  -- ถ้า critical=true และ fail -> unhealthy
    }
end

function HealthCheck:run()
    local allHealthy = true
    self.results = {}
    self.lastRun = os.time()
    
    for name, check in pairs(self.checks) do
        local start = os.clock()
        local ok, result = pcall(check.fn)
        local elapsed = (os.clock() - start) * 1000  -- ms
        
        if ok and result.healthy then
            self.results[name] = {
                status = "healthy",
                message = result.message or "OK",
                latency = math.floor(elapsed)
            }
        else
            self.results[name] = {
                status = "unhealthy",
                message = (not ok) and tostring(result) or (result.message or "Check failed"),
                latency = math.floor(elapsed)
            }
            if check.critical then
                allHealthy = false
            end
        end
    end
    
    return allHealthy, self.results
end

function HealthCheck:toResponse()
    local healthy, results = self:run()
    
    local checks = {}
    for name, result in pairs(results) do
        checks[name] = result
    end
    
    return {
        status = healthy and "healthy" or "unhealthy",
        checks = checks,
        timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
    }
end

-- ตัวอย่าง health checks
local hc = HealthCheck.new()

-- Database check
hc:addCheck("database", function()
    -- สมมุติเชื่อมต่อ database
    local ok = true  -- mysql.ping() หรือ similar
    return {
        healthy = ok,
        message = ok and "Connected" or "Connection failed"
    }
end, true)  -- critical

-- Redis check
hc:addCheck("redis", function()
    local ok = true  -- redis:ping() == "PONG"
    return {
        healthy = ok,
        message = ok and "Connected" or "Connection failed"
    }
end, true)

-- Disk space check
hc:addCheck("disk", function()
    -- ตรวจสอบ disk space
    local usedPercent = 65  -- จาก os command
    local healthy = usedPercent < 90
    return {
        healthy = healthy,
        message = string.format("Disk usage: %d%%", usedPercent)
    }
end, false)

-- Memory check
hc:addCheck("memory", function()
    local memInfo = {used = 512, total = 2048}  -- MB
    local usedPercent = (memInfo.used / memInfo.total) * 100
    local healthy = usedPercent < 85
    return {
        healthy = healthy,
        message = string.format("Memory: %d/%d MB (%.0f%%)", memInfo.used, memInfo.total, usedPercent)
    }
end, false)

-- External service check
hc:addCheck("payment_api", function()
    -- สมมุติเรียก payment provider API
    local reachable = true
    return {
        healthy = reachable,
        message = reachable and "Payment API reachable" or "Payment API unreachable"
    }
end, false)

-- Run all checks
local healthy, results = hc:run()
print("\nHealth Check Results:")
print("Overall:", healthy and "HEALTHY" or "UNHEALTHY")
for name, result in pairs(results) do
    print(string.format("  [%s] %s: %s (%dms)",
        result.status == "healthy" and "OK" or "FAIL",
        name,
        result.message,
        result.latency
    ))
end
```

## 7. Circuit Breaker Pattern

```lua
-- Example 7: Circuit Breaker implementation
local CircuitBreaker = {}
CircuitBreaker.__index = CircuitBreaker

-- States: CLOSED (normal), OPEN (failing), HALF_OPEN (testing)
local STATE = {CLOSED = "closed", OPEN = "open", HALF_OPEN = "half_open"}

function CircuitBreaker.new(options)
    return setmetatable({
        name = options.name or "unknown",
        failureThreshold = options.failureThreshold or 5,
        successThreshold = options.successThreshold or 2,  -- for half-open -> closed
        timeout = options.timeout or 30,    -- seconds before OPEN -> HALF_OPEN
        halfOpenMaxCalls = options.halfOpenMaxCalls or 1,
        
        state = STATE.CLOSED,
        failures = 0,
        successes = 0,
        lastFailureTime = 0,
        halfOpenCalls = 0,
        
        onStateChange = options.onStateChange,
        metrics = {
            totalCalls = 0,
            successCalls = 0,
            failureCalls = 0,
            rejectedCalls = 0,
            latencySum = 0
        }
    }, CircuitBreaker)
end

function CircuitBreaker:getState()
    if self.state == STATE.OPEN then
        local now = os.time()
        if now - self.lastFailureTime >= self.timeout then
            self:_transition(STATE.HALF_OPEN)
        end
    end
    return self.state
end

function CircuitBreaker:_transition(newState)
    local oldState = self.state
    self.state = newState
    self.halfOpenCalls = 0
    self.successes = 0
    
    print(string.format("[CB:%s] State: %s -> %s", self.name, oldState, newState))
    
    if self.onStateChange then
        self.onStateChange(oldState, newState)
    end
end

function CircuitBreaker:call(fn)
    local state = self:getState()
    self.metrics.totalCalls = self.metrics.totalCalls + 1
    
    -- Circuit OPEN -> reject
    if state == STATE.OPEN then
        self.metrics.rejectedCalls = self.metrics.rejectedCalls + 1
        return nil, "Circuit breaker is OPEN for " .. self.name
    end
    
    -- HALF_OPEN -> allow limited calls
    if state == STATE.HALF_OPEN then
        self.halfOpenCalls = self.halfOpenCalls + 1
        if self.halfOpenCalls > self.halfOpenMaxCalls then
            self.metrics.rejectedCalls = self.metrics.rejectedCalls + 1
            return nil, "Circuit breaker is HALF_OPEN (at capacity)"
        end
    end
    
    -- Execute function
    local startTime = os.clock()
    local ok, result = pcall(fn)
    local elapsed = os.clock() - startTime
    self.metrics.latencySum = self.metrics.latencySum + elapsed
    
    if ok then
        self:_onSuccess()
        self.metrics.successCalls = self.metrics.successCalls + 1
        return result, nil
    else
        self:_onFailure()
        self.metrics.failureCalls = self.metrics.failureCalls + 1
        return nil, result
    end
end

function CircuitBreaker:_onSuccess()
    if self.state == STATE.HALF_OPEN then
        self.successes = self.successes + 1
        if self.successes >= self.successThreshold then
            self.failures = 0
            self:_transition(STATE.CLOSED)
        end
    else
        self.failures = math.max(0, self.failures - 1)
    end
end

function CircuitBreaker:_onFailure()
    self.failures = self.failures + 1
    self.lastFailureTime = os.time()
    
    if self.state == STATE.HALF_OPEN then
        self:_transition(STATE.OPEN)
    elseif self.failures >= self.failureThreshold then
        self:_transition(STATE.OPEN)
    end
end

function CircuitBreaker:getMetrics()
    local m = self.metrics
    local avgLatency = m.totalCalls > 0 and (m.latencySum / m.totalCalls * 1000) or 0
    return {
        state = self.state,
        totalCalls = m.totalCalls,
        successRate = m.totalCalls > 0 and (m.successCalls / m.totalCalls * 100) or 0,
        failureRate = m.totalCalls > 0 and (m.failureCalls / m.totalCalls * 100) or 0,
        rejectedCalls = m.rejectedCalls,
        avgLatencyMs = avgLatency
    }
end

-- การใช้งาน
local paymentCB = CircuitBreaker.new({
    name = "payment-service",
    failureThreshold = 3,
    timeout = 10,
    successThreshold = 2,
    onStateChange = function(from, to)
        -- ส่ง alert ไปยัง monitoring system
        print("ALERT: Circuit breaker state changed: " .. from .. " -> " .. to)
    end
})

-- Simulated payment service calls
local function callPaymentService(amount)
    -- สมมุติ 30% fail rate
    if math.random() < 0.3 then
        error("Payment service timeout")
    end
    return {transactionId = "txn_" .. math.random(10000), amount = amount}
end

-- Test circuit breaker
math.randomseed(42)
for i = 1, 15 do
    local result, err = paymentCB:call(function()
        return callPaymentService(100 * i)
    end)
    
    if result then
        print(string.format("Call %d: SUCCESS - txn: %s", i, result.transactionId))
    else
        print(string.format("Call %d: FAILED - %s", i, err))
    end
end

local metrics = paymentCB:getMetrics()
print("\nCircuit Breaker Metrics:")
print("  State:", metrics.state)
print("  Total calls:", metrics.totalCalls)
print(string.format("  Success rate: %.1f%%", metrics.successRate))
print(string.format("  Failure rate: %.1f%%", metrics.failureRate))
print("  Rejected calls:", metrics.rejectedCalls)
```

## 8. Retry Pattern

```lua
-- Example 8: Retry with exponential backoff
local function sleep(seconds)
    -- In real Lua: socket.sleep, ngx.sleep, etc.
    local t = os.clock()
    while os.clock() - t < seconds do end
end

local Retry = {}
Retry.__index = Retry

function Retry.new(options)
    return setmetatable({
        maxAttempts = options.maxAttempts or 3,
        baseDelay = options.baseDelay or 0.1,   -- seconds
        maxDelay = options.maxDelay or 30,
        multiplier = options.multiplier or 2,
        jitter = options.jitter or true,
        retryOn = options.retryOn,  -- function(err) -> bool
    }, Retry)
end

function Retry:execute(fn)
    local attempt = 0
    local lastErr
    
    while attempt < self.maxAttempts do
        attempt = attempt + 1
        
        local ok, result = pcall(fn)
        
        if ok then
            if attempt > 1 then
                print(string.format("[Retry] Succeeded on attempt %d", attempt))
            end
            return result, nil
        end
        
        lastErr = result
        
        -- Check if should retry this error
        if self.retryOn and not self.retryOn(result) then
            return nil, result
        end
        
        if attempt < self.maxAttempts then
            -- Calculate delay with exponential backoff
            local delay = math.min(
                self.baseDelay * (self.multiplier ^ (attempt - 1)),
                self.maxDelay
            )
            
            -- Add jitter
            if self.jitter then
                delay = delay * (0.5 + math.random())
            end
            
            print(string.format("[Retry] Attempt %d failed: %s. Retrying in %.2fs...",
                attempt, tostring(result), delay))
            
            sleep(delay)
        end
    end
    
    return nil, string.format("All %d attempts failed. Last error: %s",
        self.maxAttempts, tostring(lastErr))
end

-- Retry with timeout
function Retry:executeWithTimeout(fn, totalTimeout)
    local startTime = os.time()
    local attempt = 0
    local lastErr
    
    while attempt < self.maxAttempts do
        if os.time() - startTime > totalTimeout then
            return nil, "Operation timed out after " .. totalTimeout .. "s"
        end
        
        attempt = attempt + 1
        local ok, result = pcall(fn)
        
        if ok then return result, nil end
        lastErr = result
        
        if attempt < self.maxAttempts then
            local delay = math.min(
                self.baseDelay * (self.multiplier ^ (attempt - 1)),
                self.maxDelay
            )
            if self.jitter then delay = delay * (0.5 + math.random()) end
            
            local remaining = totalTimeout - (os.time() - startTime)
            delay = math.min(delay, remaining)
            
            print(string.format("[Retry] Attempt %d failed. Waiting %.2fs...", attempt, delay))
            sleep(delay)
        end
    end
    
    return nil, lastErr
end

-- การใช้งาน
local retrier = Retry.new({
    maxAttempts = 4,
    baseDelay = 0.05,
    maxDelay = 1,
    multiplier = 2,
    jitter = false,
    retryOn = function(err)
        -- เฉพาะ retry สำหรับ transient errors
        return tostring(err):find("timeout") or
               tostring(err):find("connection") or
               tostring(err):find("503")
    end
})

local callCount = 0
local result, err = retrier:execute(function()
    callCount = callCount + 1
    print("Calling service... attempt " .. callCount)
    if callCount < 3 then
        error("connection timeout")
    end
    return {data = "success!"}
end)

if result then
    print("Result:", result.data)
else
    print("Failed:", err)
end
```

## 9. Saga Pattern สำหรับ Distributed Transactions

```lua
-- Example 9: Saga pattern for distributed transactions
-- แก้ปัญหา distributed transaction ที่ 2PC ซับซ้อนเกินไป
-- Saga ทำงานเป็น sequence ของ local transactions
-- ถ้า step ใด fail -> ทำ compensating transactions ย้อนกลับ

local Saga = {}
Saga.__index = Saga

function Saga.new(name)
    return setmetatable({
        name = name,
        steps = {},
        completedSteps = {},
        status = "pending"
    }, Saga)
end

function Saga:addStep(name, action, compensation)
    table.insert(self.steps, {
        name = name,
        action = action,          -- forward action
        compensation = compensation  -- rollback action
    })
end

function Saga:execute(context)
    context = context or {}
    self.status = "running"
    self.completedSteps = {}
    
    print(string.format("[Saga:%s] Starting execution with %d steps", self.name, #self.steps))
    
    for i, step in ipairs(self.steps) do
        print(string.format("[Saga:%s] Executing step %d/%d: %s", self.name, i, #self.steps, step.name))
        
        local ok, result = pcall(step.action, context)
        
        if ok then
            table.insert(self.completedSteps, {
                step = step,
                result = result,
                index = i
            })
            -- Pass result to context for next steps
            context["step_" .. i] = result
            print(string.format("[Saga:%s] Step '%s' completed", self.name, step.name))
        else
            print(string.format("[Saga:%s] Step '%s' FAILED: %s", self.name, step.name, tostring(result)))
            
            -- Run compensating transactions in reverse order
            self:compensate(context, result)
            self.status = "failed"
            return false, result
        end
    end
    
    self.status = "completed"
    print(string.format("[Saga:%s] Completed successfully!", self.name))
    return true, context
end

function Saga:compensate(context, originalError)
    print(string.format("[Saga:%s] Starting compensation for %d completed steps...", 
        self.name, #self.completedSteps))
    
    -- Reverse order compensation
    for i = #self.completedSteps, 1, -1 do
        local completed = self.completedSteps[i]
        
        if completed.step.compensation then
            print(string.format("[Saga:%s] Compensating: %s", self.name, completed.step.name))
            
            local ok, compErr = pcall(completed.step.compensation, context, completed.result)
            
            if ok then
                print(string.format("[Saga:%s] Compensation '%s' succeeded", self.name, completed.step.name))
            else
                print(string.format("[Saga:%s] Compensation '%s' FAILED: %s (manual intervention needed)",
                    self.name, completed.step.name, tostring(compErr)))
            end
        else
            print(string.format("[Saga:%s] No compensation defined for '%s'", self.name, completed.step.name))
        end
    end
end

-- ตัวอย่าง: Order Saga
-- 1. Reserve inventory
-- 2. Charge payment
-- 3. Create order
-- 4. Send notification

local orderSaga = Saga.new("create-order")

orderSaga:addStep(
    "reserve-inventory",
    function(ctx)
        print("  Reserving inventory for product " .. ctx.productId)
        -- inventoryService:reserve(ctx.productId, ctx.quantity)
        if ctx.productId == "SOLD_OUT" then
            error("Product out of stock")
        end
        return {reservationId = "res_" .. math.random(1000)}
    end,
    function(ctx, result)
        print("  COMPENSATE: Releasing reservation " .. result.reservationId)
        -- inventoryService:release(result.reservationId)
    end
)

orderSaga:addStep(
    "charge-payment",
    function(ctx)
        print("  Charging $" .. ctx.amount .. " from card " .. ctx.cardId)
        -- paymentService:charge(ctx.cardId, ctx.amount)
        if ctx.cardId == "DECLINED" then
            error("Payment declined")
        end
        return {chargeId = "chg_" .. math.random(10000), amount = ctx.amount}
    end,
    function(ctx, result)
        print("  COMPENSATE: Refunding charge " .. result.chargeId)
        -- paymentService:refund(result.chargeId)
    end
)

orderSaga:addStep(
    "create-order",
    function(ctx)
        print("  Creating order record...")
        return {orderId = "ord_" .. math.random(100000)}
    end,
    function(ctx, result)
        print("  COMPENSATE: Canceling order " .. result.orderId)
        -- orderService:cancel(result.orderId)
    end
)

orderSaga:addStep(
    "send-notification",
    function(ctx)
        print("  Sending order confirmation email to " .. ctx.email)
        -- notificationService:sendEmail(ctx.email, "Order confirmed", ...)
        return {notificationId = "notif_" .. math.random(1000)}
    end,
    nil  -- No compensation needed (email already sent)
)

-- Test successful order
print("\n=== Test 1: Successful order ===")
local success, result = orderSaga:execute({
    productId = "PROD-123",
    quantity = 2,
    amount = 299.99,
    cardId = "card_visa_4242",
    email = "customer@example.com"
})
print("Success:", success)

-- Test failed order (payment declined)
print("\n=== Test 2: Payment declined ===")
local saga2 = Saga.new("create-order-2")
-- (copy steps สำหรับ clean state)
saga2:addStep("reserve-inventory",
    function(ctx) return {reservationId = "res_999"} end,
    function(ctx, result) print("  Releasing reservation " .. result.reservationId) end)
saga2:addStep("charge-payment",
    function(ctx) error("Payment declined by bank") end,
    function(ctx, result) print("  Refunding " .. tostring(result)) end)

local success2, err2 = saga2:execute({cardId = "DECLINED", amount = 100})
print("Success:", success2, "Error:", err2)
```

## 10. Event-Driven Microservices

```lua
-- Example 10: Event-driven architecture with message queue
-- ใช้ Redis Pub/Sub หรือ message broker สำหรับ async communication

local EventBus = {}
EventBus.__index = EventBus

function EventBus.new()
    return setmetatable({
        handlers = {},
        deadLetterQueue = {},
        maxRetries = 3,
        published = 0,
        consumed = 0
    }, EventBus)
end

function EventBus:subscribe(eventType, handler, options)
    if not self.handlers[eventType] then
        self.handlers[eventType] = {}
    end
    table.insert(self.handlers[eventType], {
        fn = handler,
        service = options and options.service or "unknown",
        retries = options and options.maxRetries or self.maxRetries
    })
end

function EventBus:publish(eventType, payload, options)
    local event = {
        id = string.format("%x%x%x", math.random(0xffff), math.random(0xffff), math.random(0xffff)),
        type = eventType,
        payload = payload,
        publishedAt = os.time(),
        source = options and options.source or "unknown",
        version = options and options.version or "1.0",
        correlationId = options and options.correlationId,
        attempts = 0
    }
    
    self.published = self.published + 1
    print(string.format("[EventBus] Published: %s (id: %s)", eventType, event.id))
    
    -- Dispatch to handlers
    self:dispatch(event)
    
    return event.id
end

function EventBus:dispatch(event)
    local handlers = self.handlers[event.type]
    if not handlers or #handlers == 0 then
        print(string.format("[EventBus] No handlers for: %s", event.type))
        return
    end
    
    for _, handler in ipairs(handlers) do
        event.attempts = event.attempts + 1
        
        local ok, err = pcall(handler.fn, event)
        
        if ok then
            self.consumed = self.consumed + 1
            print(string.format("[EventBus] [%s] Handled: %s", handler.service, event.type))
        else
            print(string.format("[EventBus] [%s] Failed to handle: %s - %s",
                handler.service, event.type, tostring(err)))
            
            if event.attempts <= handler.retries then
                print(string.format("[EventBus] Retrying (%d/%d)...", event.attempts, handler.retries))
                self:dispatch(event)
            else
                print("[EventBus] Max retries reached, moving to dead letter queue")
                table.insert(self.deadLetterQueue, {
                    event = event,
                    error = err,
                    failedAt = os.time()
                })
            end
        end
    end
end

function EventBus:processDeadLetters(handler)
    while #self.deadLetterQueue > 0 do
        local item = table.remove(self.deadLetterQueue, 1)
        handler(item.event, item.error)
    end
end

-- Domain Events
local function defineEvent(type, schema)
    return {
        type = type,
        schema = schema,
        create = function(data)
            -- Validate against schema
            for field, required in pairs(schema) do
                if required and data[field] == nil then
                    error("Missing required field: " .. field)
                end
            end
            return data
        end
    }
end

-- Event definitions
local Events = {
    UserCreated = defineEvent("user.created", {
        userId = true, email = true, name = true
    }),
    OrderPlaced = defineEvent("order.placed", {
        orderId = true, userId = true, items = true, totalAmount = true
    }),
    PaymentProcessed = defineEvent("payment.processed", {
        paymentId = true, orderId = true, amount = true, status = true
    }),
    OrderShipped = defineEvent("order.shipped", {
        orderId = true, trackingNumber = true
    }),
}

-- Service Handlers
local bus = EventBus.new()

-- Email service listens to multiple events
bus:subscribe("user.created", function(event)
    local p = event.payload
    print(string.format("  [email-service] Welcome email to %s (%s)", p.name, p.email))
end, {service = "email-service"})

bus:subscribe("order.placed", function(event)
    local p = event.payload
    print(string.format("  [email-service] Order confirmation for order %s", p.orderId))
end, {service = "email-service"})

-- Order service listens to payment
bus:subscribe("payment.processed", function(event)
    local p = event.payload
    if p.status == "success" then
        print(string.format("  [order-service] Payment confirmed for order %s, proceeding to fulfillment", p.orderId))
        -- Update order status to "paid"
    else
        print(string.format("  [order-service] Payment failed for order %s, canceling order", p.orderId))
    end
end, {service = "order-service"})

-- Inventory service listens to order placed
bus:subscribe("order.placed", function(event)
    local p = event.payload
    print(string.format("  [inventory-service] Reserve items for order %s", p.orderId))
    for _, item in ipairs(p.items) do
        print(string.format("    Reserve: %s x%d", item.productId, item.quantity))
    end
end, {service = "inventory-service"})

-- Analytics service
bus:subscribe("order.placed", function(event)
    print(string.format("  [analytics] Recording order event: $%.2f", event.payload.totalAmount))
end, {service = "analytics"})

-- Publish events
print("\n=== Event Flow Demo ===")
bus:publish("user.created", Events.UserCreated.create({
    userId = "usr_001",
    email = "john@example.com",
    name = "John Doe"
}), {source = "user-service"})

print()
bus:publish("order.placed", Events.OrderPlaced.create({
    orderId = "ord_001",
    userId = "usr_001",
    items = {
        {productId = "prod_123", quantity = 2, price = 50},
        {productId = "prod_456", quantity = 1, price = 199},
    },
    totalAmount = 299
}), {source = "order-service"})

print()
bus:publish("payment.processed", Events.PaymentProcessed.create({
    paymentId = "pay_001",
    orderId = "ord_001",
    amount = 299,
    status = "success"
}), {source = "payment-service"})

print("\nEvent Bus Stats:")
print("  Published:", bus.published)
print("  Consumed:", bus.consumed)
print("  Dead letters:", #bus.deadLetterQueue)
```

## 11. gRPC vs REST Concepts

```lua
-- Example 11: Implementing a simple RPC framework (inspired by gRPC)
-- (Simplified conceptual implementation)

local RPC = {}
RPC.__index = RPC

function RPC.new()
    return setmetatable({
        services = {},
        interceptors = {}
    }, RPC)
end

-- Define service methods (like proto definition)
function RPC:defineService(name, methods)
    self.services[name] = {
        name = name,
        methods = methods,
        handlers = {}
    }
end

-- Register handler
function RPC:registerHandler(serviceName, methodName, handler)
    if not self.services[serviceName] then
        error("Service not defined: " .. serviceName)
    end
    self.services[serviceName].handlers[methodName] = handler
end

-- Add interceptor (middleware)
function RPC:addInterceptor(fn)
    table.insert(self.interceptors, fn)
end

-- Call service method
function RPC:call(serviceName, methodName, request)
    local service = self.services[serviceName]
    if not service then
        return nil, {code = 12, message = "Service not found: " .. serviceName}
    end
    
    local methodDef = service.methods[methodName]
    if not methodDef then
        return nil, {code = 12, message = "Method not found: " .. methodName}
    end
    
    local handler = service.handlers[methodName]
    if not handler then
        return nil, {code = 12, message = "Handler not registered: " .. methodName}
    end
    
    -- Validate request
    if methodDef.input then
        for _, required in ipairs(methodDef.input) do
            if request[required] == nil then
                return nil, {code = 3, message = "Missing field: " .. required}
            end
        end
    end
    
    -- Run interceptors
    local ctx = {service = serviceName, method = methodName, request = request}
    for _, interceptor in ipairs(self.interceptors) do
        local stop, err = interceptor(ctx)
        if stop then
            return nil, err
        end
    end
    
    -- Execute handler
    local ok, result = pcall(handler, request, ctx)
    
    if ok then
        return result, nil
    else
        return nil, {code = 2, message = "Internal error: " .. tostring(result)}
    end
end

-- Proto-like service definitions
local rpc = RPC.new()

rpc:defineService("UserService", {
    GetUser = {input = {"userId"}, output = {"id", "name", "email"}},
    CreateUser = {input = {"name", "email"}, output = {"id", "name", "email"}},
    ListUsers = {input = {}, output = {"users", "total"}},
    DeleteUser = {input = {"userId"}, output = {}},
})

rpc:defineService("OrderService", {
    CreateOrder = {input = {"userId", "items"}, output = {"orderId", "status"}},
    GetOrder = {input = {"orderId"}, output = {"orderId", "userId", "items", "status"}},
})

-- Logging interceptor
rpc:addInterceptor(function(ctx)
    print(string.format("[RPC] %s.%s called", ctx.service, ctx.method))
end)

-- Auth interceptor
rpc:addInterceptor(function(ctx)
    -- Check auth token
    if not ctx.request._token then
        -- Non-auth methods
        local publicMethods = {"ListUsers", "GetUser"}
        for _, m in ipairs(publicMethods) do
            if m == ctx.method then return false end
        end
        return true, {code = 16, message = "Unauthenticated"}
    end
end)

-- Register handlers
local usersStore = {
    ["u1"] = {id = "u1", name = "Alice", email = "alice@example.com"},
    ["u2"] = {id = "u2", name = "Bob",   email = "bob@example.com"},
}
local nextId = 3

rpc:registerHandler("UserService", "GetUser", function(req)
    local user = usersStore[req.userId]
    if not user then
        error({code = 5, message = "User not found: " .. req.userId})
    end
    return user
end)

rpc:registerHandler("UserService", "ListUsers", function(req)
    local list = {}
    for _, u in pairs(usersStore) do
        table.insert(list, u)
    end
    return {users = list, total = #list}
end)

rpc:registerHandler("UserService", "CreateUser", function(req)
    local id = "u" .. nextId
    nextId = nextId + 1
    local user = {id = id, name = req.name, email = req.email}
    usersStore[id] = user
    return user
end)

rpc:registerHandler("UserService", "DeleteUser", function(req)
    if not usersStore[req.userId] then
        error({code = 5, message = "User not found"})
    end
    usersStore[req.userId] = nil
    return {}
end)

-- Test RPC calls
print("\n=== RPC Demo ===")

local users, err = rpc:call("UserService", "ListUsers", {})
if users then
    print("Users:", users.total)
    for _, u in ipairs(users.users) do
        print("  -", u.name, u.email)
    end
else
    print("Error:", err.message)
end

local user, err2 = rpc:call("UserService", "GetUser", {userId = "u1"})
if user then
    print("\nFound user:", user.name, user.email)
else
    print("Error:", err2.message)
end

local _, err3 = rpc:call("UserService", "CreateUser", {name = "Charlie", email = "charlie@example.com"})
-- This will fail because no token
print("\nCreate without auth:", err3 and err3.message or "success")
```

## 12. Docker Compose สำหรับ Local Development

```lua
-- Example 12: Generate Docker Compose configuration
-- Lua สามารถใช้ generate infrastructure config ได้

local yaml = {}

function yaml.dump(data, indent)
    indent = indent or 0
    local result = ""
    local spaces = string.rep("  ", indent)
    
    if type(data) == "table" then
        -- Check if array or object
        local isArray = #data > 0
        
        if isArray then
            for _, v in ipairs(data) do
                result = result .. spaces .. "- "
                if type(v) == "table" then
                    result = result .. "\n" .. yaml.dump(v, indent + 1)
                else
                    result = result .. tostring(v) .. "\n"
                end
            end
        else
            for k, v in pairs(data) do
                result = result .. spaces .. k .. ":"
                if type(v) == "table" then
                    result = result .. "\n" .. yaml.dump(v, indent + 1)
                elseif type(v) == "boolean" then
                    result = result .. " " .. tostring(v) .. "\n"
                elseif type(v) == "number" then
                    result = result .. " " .. tostring(v) .. "\n"
                else
                    result = result .. " " .. tostring(v) .. "\n"
                end
            end
        end
    end
    
    return result
end

-- Docker Compose generator
local function generateDockerCompose(services, networks)
    local compose = {
        version = "3.8",
        services = {},
        networks = networks or {}
    }
    
    for name, svc in pairs(services) do
        compose.services[name] = svc
    end
    
    return "version: '3.8'\n\nservices:\n" .. yaml.dump(compose.services, 1)
end

-- Define microservices
local services = {
    ["api-gateway"] = {
        image = "nginx:alpine",
        ports = {"8080:80"},
        volumes = {"./nginx.conf:/etc/nginx/nginx.conf:ro"},
        depends_on = {"user-service", "order-service"},
        networks = {"frontend", "backend"},
        restart = "unless-stopped"
    },
    
    ["user-service"] = {
        build = {context = "./services/user", dockerfile = "Dockerfile"},
        environment = {
            "PORT=3001",
            "DB_HOST=postgres",
            "DB_NAME=users_db",
            "REDIS_HOST=redis",
            "JWT_SECRET=secret"
        },
        depends_on = {"postgres", "redis"},
        networks = {"backend"},
        healthcheck = {
            test = {"CMD", "curl", "-f", "http://localhost:3001/health"},
            interval = "30s",
            timeout = "10s",
            retries = 3
        },
        restart = "unless-stopped",
        deploy = {
            replicas = 2,
            resources = {
                limits = {cpus = "0.5", memory = "256M"}
            }
        }
    },
    
    ["order-service"] = {
        build = {context = "./services/order"},
        environment = {
            "PORT=3002",
            "DB_HOST=postgres",
            "DB_NAME=orders_db",
            "REDIS_HOST=redis",
            "USER_SERVICE_URL=http://user-service:3001"
        },
        depends_on = {"postgres", "redis", "user-service"},
        networks = {"backend"},
        restart = "unless-stopped"
    },
    
    ["payment-service"] = {
        build = {context = "./services/payment"},
        environment = {
            "PORT=3003",
            "STRIPE_KEY=sk_test_...",
            "DB_HOST=postgres",
            "DB_NAME=payments_db"
        },
        depends_on = {"postgres"},
        networks = {"backend"},
        restart = "unless-stopped"
    },
    
    ["postgres"] = {
        image = "postgres:15-alpine",
        environment = {
            "POSTGRES_USER=admin",
            "POSTGRES_PASSWORD=secret",
            "POSTGRES_MULTIPLE_DATABASES=users_db,orders_db,payments_db"
        },
        volumes = {
            "postgres_data:/var/lib/postgresql/data",
            "./init-db.sh:/docker-entrypoint-initdb.d/init.sh"
        },
        networks = {"backend"},
        restart = "unless-stopped"
    },
    
    ["redis"] = {
        image = "redis:7-alpine",
        command = "redis-server --requirepass secret --appendonly yes",
        volumes = {"redis_data:/data"},
        networks = {"backend"},
        restart = "unless-stopped"
    },
    
    ["jaeger"] = {
        image = "jaegertracing/all-in-one:latest",
        ports = {"16686:16686", "6831:6831/udp"},
        networks = {"backend", "monitoring"},
        restart = "unless-stopped"
    },
    
    ["prometheus"] = {
        image = "prom/prometheus:latest",
        volumes = {"./prometheus.yml:/etc/prometheus/prometheus.yml:ro"},
        ports = {"9090:9090"},
        networks = {"monitoring"},
        restart = "unless-stopped"
    }
}

print("=== Docker Compose Configuration ===")
print(generateDockerCompose(services))

-- NGINX config generator
local function generateNginxConfig(upstreams, locations)
    local config = "events { worker_connections 1024; }\n\nhttp {\n"
    
    -- Upstreams
    for name, servers in pairs(upstreams) do
        config = config .. "  upstream " .. name .. " {\n"
        config = config .. "    least_conn;\n"
        for _, server in ipairs(servers) do
            config = config .. "    server " .. server .. ";\n"
        end
        config = config .. "  }\n\n"
    end
    
    config = config .. "  server {\n"
    config = config .. "    listen 80;\n\n"
    
    for path, upstream in pairs(locations) do
        config = config .. "    location " .. path .. " {\n"
        config = config .. "      proxy_pass http://" .. upstream .. ";\n"
        config = config .. "      proxy_set_header Host $host;\n"
        config = config .. "      proxy_set_header X-Real-IP $remote_addr;\n"
        config = config .. "    }\n\n"
    end
    
    config = config .. "  }\n}"
    return config
end

local nginxConf = generateNginxConfig(
    {
        ["user_service"] = {"user-service:3001", "user-service:3001"},
        ["order_service"] = {"order-service:3002"},
        ["payment_service"] = {"payment-service:3003"},
    },
    {
        ["/api/users"] = "user_service",
        ["/api/orders"] = "order_service",
        ["/api/payments"] = "payment_service",
    }
)

print("\n=== NGINX Configuration ===")
print(nginxConf)
```

## 13. Distributed Tracing Concepts

```lua
-- Example 13: Distributed tracing (OpenTelemetry inspired)
local Tracer = {}
Tracer.__index = Tracer

function Tracer.new(serviceName)
    return setmetatable({
        serviceName = serviceName,
        spans = {}
    }, Tracer)
end

local Span = {}
Span.__index = Span

function Span.new(tracer, name, parentId)
    local traceId = parentId and parentId:match("^([^:]+)") or
        string.format("%016x", math.random(0xffffffffffff))
    local spanId = string.format("%016x", math.random(0xffffffffffff))
    
    return setmetatable({
        traceId = traceId,
        spanId = spanId,
        parentId = parentId,
        name = name,
        service = tracer.serviceName,
        startTime = os.time() * 1000,  -- ms
        endTime = nil,
        attributes = {},
        events = {},
        status = "ok",
        tracer = tracer
    }, Span)
end

function Span:setAttr(key, value)
    self.attributes[key] = value
    return self
end

function Span:addEvent(name, attrs)
    table.insert(self.events, {
        name = name,
        time = os.time() * 1000,
        attributes = attrs or {}
    })
    return self
end

function Span:setStatus(status, message)
    self.status = status
    if message then self.statusMessage = message end
    return self
end

function Span:finish()
    self.endTime = os.time() * 1000
    self.duration = self.endTime - self.startTime
    table.insert(self.tracer.spans, self)
    return self
end

function Span:getContext()
    return self.traceId .. ":" .. self.spanId
end

function Tracer:startSpan(name, parentContext)
    return Span.new(self, name, parentContext)
end

function Tracer:getSpans()
    return self.spans
end

function Tracer:export()
    local result = {}
    for _, span in ipairs(self.spans) do
        table.insert(result, {
            traceId = span.traceId,
            spanId = span.spanId,
            parentId = span.parentId,
            name = span.name,
            service = span.service,
            startTime = span.startTime,
            duration = span.duration or 0,
            attributes = span.attributes,
            events = span.events,
            status = span.status
        })
    end
    return result
end

-- ตัวอย่าง: Traced request flow
local gatewayTracer = Tracer.new("api-gateway")
local userTracer = Tracer.new("user-service")
local orderTracer = Tracer.new("order-service")
local paymentTracer = Tracer.new("payment-service")

-- Simulate traced request
local function simulateRequest(userId, orderData)
    -- Gateway span
    local gwSpan = gatewayTracer:startSpan("POST /api/orders")
    gwSpan:setAttr("http.method", "POST")
    gwSpan:setAttr("http.url", "/api/orders")
    gwSpan:setAttr("user.id", userId)
    
    local ctx = gwSpan:getContext()
    
    -- User service span
    local userSpan = userTracer:startSpan("GetUser", ctx)
    userSpan:setAttr("db.type", "postgresql")
    userSpan:setAttr("db.statement", "SELECT * FROM users WHERE id=$1")
    userSpan:addEvent("db_query_start")
    -- Simulate DB call
    userSpan:addEvent("db_query_end", {rows = 1})
    userSpan:finish()
    
    -- Order service span
    local orderSpan = orderTracer:startSpan("CreateOrder", ctx)
    orderSpan:setAttr("order.items_count", #orderData.items)
    orderSpan:setAttr("order.total", orderData.total)
    
    -- Payment span (child of order)
    local paymentCtx = orderSpan:getContext()
    local paySpan = paymentTracer:startSpan("ProcessPayment", paymentCtx)
    paySpan:setAttr("payment.amount", orderData.total)
    paySpan:setAttr("payment.method", "credit_card")
    paySpan:addEvent("payment_initiated")
    
    -- Simulate payment processing
    paySpan:addEvent("payment_completed", {status = "success"})
    paySpan:setAttr("payment.transaction_id", "txn_" .. math.random(100000))
    paySpan:finish()
    
    orderSpan:addEvent("order_created", {orderId = "ord_" .. math.random(100000)})
    orderSpan:finish()
    
    gwSpan:setAttr("http.status_code", 201)
    gwSpan:finish()
end

simulateRequest("usr_001", {
    items = {{productId = "p1", qty = 2}, {productId = "p2", qty = 1}},
    total = 299.99
})

-- Print trace
local allSpans = {}
for _, s in ipairs(gatewayTracer:getSpans()) do table.insert(allSpans, s) end
for _, s in ipairs(userTracer:getSpans()) do table.insert(allSpans, s) end
for _, s in ipairs(orderTracer:getSpans()) do table.insert(allSpans, s) end
for _, s in ipairs(paymentTracer:getSpans()) do table.insert(allSpans, s) end

print("\n=== Distributed Trace ===")
for _, span in ipairs(allSpans) do
    print(string.format("[%s] %s.%s (trace:%s, span:%s, %dms)",
        span.service,
        span.service,
        span.name,
        span.traceId:sub(1, 8),
        span.spanId:sub(1, 8),
        span.duration or 0
    ))
    for k, v in pairs(span.attributes) do
        print(string.format("  attr: %s=%s", k, tostring(v)))
    end
end
```

## สรุป Microservices Architecture

Microservices ใน Lua/OpenResty มีข้อดีสำหรับ:

1. **API Gateway** - OpenResty เหมาะมากสำหรับ high-performance gateway
2. **Sidecar Proxy** - Lua scripts ใน NGINX/OpenResty สำหรับ service mesh
3. **Rate Limiting** - Efficient rate limiting ด้วย `ngx.shared`
4. **Service Discovery** - Client-side discovery ด้วย Lua
5. **Circuit Breaker** - Protection pattern สำหรับ fault tolerance
6. **Event-Driven** - Pub/Sub patterns ด้วย Redis

Design Principles:
- Single Responsibility ต่อ service
- Loose coupling, high cohesion
- Failure isolation
- Observability (logs, metrics, traces)
- Automation (CI/CD, health checks)
