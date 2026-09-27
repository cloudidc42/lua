# บทที่ 66: Load Balancing

## บทนำ

Load Balancing คือเทคนิคการกระจายโหลด (traffic) ไปยังเซิร์ฟเวอร์หลายตัวเพื่อให้ระบบรองรับผู้ใช้งานได้มากขึ้นและมีความเสถียรสูง ในบทนี้เราจะเรียนรู้อัลกอริทึมต่างๆ และการ implement Load Balancer ด้วย Lua

---

## 66.1 Round Robin Algorithm

Round Robin เป็นอัลกอริทึมพื้นฐานที่กระจาย request ไปยังเซิร์ฟเวอร์แต่ละตัวตามลำดับวน

```lua
-- ตัวอย่างที่ 1: Round Robin พื้นฐาน
local RoundRobin = {}
RoundRobin.__index = RoundRobin

function RoundRobin.new(servers)
    local self = setmetatable({}, RoundRobin)
    self.servers = servers
    self.current = 0
    return self
end

function RoundRobin:next()
    self.current = (self.current % #self.servers) + 1
    return self.servers[self.current]
end

-- ทดสอบ
local lb = RoundRobin.new({
    {host = "10.0.0.1", port = 8080},
    {host = "10.0.0.2", port = 8080},
    {host = "10.0.0.3", port = 8080},
})

for i = 1, 9 do
    local server = lb:next()
    print(string.format("Request %d -> %s:%d", i, server.host, server.port))
end
```

```lua
-- ตัวอย่างที่ 2: Round Robin พร้อม Thread Safety
local RoundRobinSafe = {}
RoundRobinSafe.__index = RoundRobinSafe

function RoundRobinSafe.new(servers)
    local self = setmetatable({}, RoundRobinSafe)
    self.servers = {}
    for i, s in ipairs(servers) do
        self.servers[i] = {
            host    = s.host,
            port    = s.port,
            active  = true,
            requests = 0,
        }
    end
    self.current = 0
    self.total   = #servers
    return self
end

function RoundRobinSafe:next()
    local tried = 0
    while tried < self.total do
        self.current = (self.current % self.total) + 1
        local server = self.servers[self.current]
        if server.active then
            server.requests = server.requests + 1
            return server
        end
        tried = tried + 1
    end
    return nil -- ไม่มีเซิร์ฟเวอร์ว่าง
end

function RoundRobinSafe:markDown(host)
    for _, s in ipairs(self.servers) do
        if s.host == host then
            s.active = false
            print("Server " .. host .. " marked as DOWN")
            return
        end
    end
end

function RoundRobinSafe:markUp(host)
    for _, s in ipairs(self.servers) do
        if s.host == host then
            s.active = true
            print("Server " .. host .. " marked as UP")
            return
        end
    end
end

local lb = RoundRobinSafe.new({
    {host = "10.0.0.1", port = 8080},
    {host = "10.0.0.2", port = 8080},
    {host = "10.0.0.3", port = 8080},
})

lb:markDown("10.0.0.2")

for i = 1, 6 do
    local server = lb:next()
    if server then
        print(string.format("Request %d -> %s:%d", i, server.host, server.port))
    else
        print("No available server!")
    end
end
```

---

## 66.2 Weighted Round Robin

Weighted Round Robin กำหนดน้ำหนักให้แต่ละเซิร์ฟเวอร์เพื่อกระจาย traffic ตามความสามารถ

```lua
-- ตัวอย่างที่ 3: Weighted Round Robin
local WeightedRoundRobin = {}
WeightedRoundRobin.__index = WeightedRoundRobin

function WeightedRoundRobin.new(servers)
    local self = setmetatable({}, WeightedRoundRobin)
    self.servers = servers
    -- สร้างลิสต์ตามน้ำหนัก
    self.pool = {}
    for _, s in ipairs(servers) do
        for _ = 1, s.weight do
            table.insert(self.pool, s)
        end
    end
    self.current = 0
    return self
end

function WeightedRoundRobin:next()
    self.current = (self.current % #self.pool) + 1
    return self.pool[self.current]
end

local lb = WeightedRoundRobin.new({
    {host = "10.0.0.1", port = 8080, weight = 5},
    {host = "10.0.0.2", port = 8080, weight = 3},
    {host = "10.0.0.3", port = 8080, weight = 2},
})

local count = {}
for i = 1, 100 do
    local s = lb:next()
    count[s.host] = (count[s.host] or 0) + 1
end

print("Distribution after 100 requests:")
for host, c in pairs(count) do
    print(string.format("  %s: %d requests (%.1f%%)", host, c, c))
end
```

```lua
-- ตัวอย่างที่ 4: Smooth Weighted Round Robin (Nginx style)
local SmoothWRR = {}
SmoothWRR.__index = SmoothWRR

function SmoothWRR.new(servers)
    local self = setmetatable({}, SmoothWRR)
    self.servers = {}
    for _, s in ipairs(servers) do
        table.insert(self.servers, {
            host            = s.host,
            port            = s.port,
            weight          = s.weight,
            effective_weight = s.weight,
            current_weight  = 0,
        })
    end
    return self
end

function SmoothWRR:next()
    local total = 0
    local best  = nil

    for _, s in ipairs(self.servers) do
        s.current_weight = s.current_weight + s.effective_weight
        total = total + s.effective_weight
        if best == nil or s.current_weight > best.current_weight then
            best = s
        end
    end

    best.current_weight = best.current_weight - total
    return best
end

local lb = SmoothWRR.new({
    {host = "a", port = 80, weight = 5},
    {host = "b", port = 80, weight = 2},
    {host = "c", port = 80, weight = 3},
})

print("Smooth WRR sequence (10 requests):")
for i = 1, 10 do
    local s = lb:next()
    io.write(s.host .. " ")
end
print()
```

---

## 66.3 Least Connections Algorithm

Least Connections ส่ง request ไปยังเซิร์ฟเวอร์ที่มี active connections น้อยที่สุด

```lua
-- ตัวอย่างที่ 5: Least Connections
local LeastConnections = {}
LeastConnections.__index = LeastConnections

function LeastConnections.new(servers)
    local self = setmetatable({}, LeastConnections)
    self.servers = {}
    for _, s in ipairs(servers) do
        table.insert(self.servers, {
            host        = s.host,
            port        = s.port,
            connections = 0,
            active      = true,
        })
    end
    return self
end

function LeastConnections:acquire()
    local best = nil
    for _, s in ipairs(self.servers) do
        if s.active then
            if best == nil or s.connections < best.connections then
                best = s
            end
        end
    end
    if best then
        best.connections = best.connections + 1
    end
    return best
end

function LeastConnections:release(server)
    for _, s in ipairs(self.servers) do
        if s.host == server.host then
            if s.connections > 0 then
                s.connections = s.connections - 1
            end
            return
        end
    end
end

function LeastConnections:status()
    print("Server Status:")
    for _, s in ipairs(self.servers) do
        print(string.format("  %s:%d -> connections: %d, active: %s",
            s.host, s.port, s.connections, tostring(s.active)))
    end
end

local lb = LeastConnections.new({
    {host = "10.0.0.1", port = 8080},
    {host = "10.0.0.2", port = 8080},
    {host = "10.0.0.3", port = 8080},
})

-- จำลองการรับ requests
local active = {}
for i = 1, 5 do
    local s = lb:acquire()
    table.insert(active, s)
    print(string.format("Acquired connection to %s", s.host))
end

lb:status()

-- คืน connection บางส่วน
lb:release(active[1])
lb:release(active[2])

print("\nAfter releasing 2 connections:")
lb:status()
```

```lua
-- ตัวอย่างที่ 6: Weighted Least Connections
local WeightedLC = {}
WeightedLC.__index = WeightedLC

function WeightedLC.new(servers)
    local self = setmetatable({}, WeightedLC)
    self.servers = {}
    for _, s in ipairs(servers) do
        table.insert(self.servers, {
            host        = s.host,
            port        = s.port,
            weight      = s.weight or 1,
            connections = 0,
            active      = true,
        })
    end
    return self
end

function WeightedLC:score(server)
    -- score ต่ำ = ดีกว่า
    return server.connections / server.weight
end

function WeightedLC:next()
    local best = nil
    for _, s in ipairs(self.servers) do
        if s.active then
            if best == nil or self:score(s) < self:score(best) then
                best = s
            end
        end
    end
    if best then
        best.connections = best.connections + 1
    end
    return best
end

local lb = WeightedLC.new({
    {host = "big",    port = 80, weight = 4},
    {host = "medium", port = 80, weight = 2},
    {host = "small",  port = 80, weight = 1},
})

for i = 1, 7 do
    local s = lb:next()
    print(string.format("Request %d -> %s (connections: %d, weight: %d)",
        i, s.host, s.connections, s.weight))
end
```

---

## 66.4 IP Hash Algorithm

IP Hash กำหนดให้ client IP เดิมไปยังเซิร์ฟเวอร์เดิมเสมอ (Sticky Sessions แบบ stateless)

```lua
-- ตัวอย่างที่ 7: IP Hash
local IPHash = {}
IPHash.__index = IPHash

function IPHash.new(servers)
    local self = setmetatable({}, IPHash)
    self.servers = {}
    for _, s in ipairs(servers) do
        if s.active ~= false then
            table.insert(self.servers, s)
        end
    end
    return self
end

function IPHash:hashIP(ip)
    -- Simple hash function สำหรับ IP address
    local hash = 0
    for part in ip:gmatch("%d+") do
        hash = hash * 256 + tonumber(part)
    end
    return hash
end

function IPHash:next(clientIP)
    if #self.servers == 0 then return nil end
    local hash  = self:hashIP(clientIP)
    local index = (hash % #self.servers) + 1
    return self.servers[index]
end

local lb = IPHash.new({
    {host = "10.0.0.1", port = 8080},
    {host = "10.0.0.2", port = 8080},
    {host = "10.0.0.3", port = 8080},
})

local clients = {
    "192.168.1.100",
    "192.168.1.101",
    "10.0.1.50",
    "172.16.0.1",
    "192.168.1.100",  -- same as first
}

print("IP Hash routing:")
for _, ip in ipairs(clients) do
    local s = lb:next(ip)
    print(string.format("  %s -> %s", ip, s.host))
end
```

```lua
-- ตัวอย่างที่ 8: IP Hash พร้อม consistent hashing
local function djb2hash(str)
    local hash = 5381
    for i = 1, #str do
        hash = ((hash * 33) + str:byte(i)) % (2^32)
    end
    return hash
end

local IPHashConsistent = {}
IPHashConsistent.__index = IPHashConsistent

function IPHashConsistent.new(servers)
    local self = setmetatable({}, IPHashConsistent)
    self.servers = servers
    return self
end

function IPHashConsistent:next(clientIP)
    local hash  = djb2hash(clientIP)
    local index = (hash % #self.servers) + 1
    return self.servers[index]
end

local lb = IPHashConsistent.new({
    {host = "s1", port = 80},
    {host = "s2", port = 80},
    {host = "s3", port = 80},
    {host = "s4", port = 80},
})

print("Consistent IP Hash:")
local testIPs = {"1.2.3.4", "5.6.7.8", "9.10.11.12", "1.2.3.4"}
for _, ip in ipairs(testIPs) do
    local s = lb:next(ip)
    print(string.format("  %s -> %s", ip, s.host))
end
```

---

## 66.5 Random Algorithm

```lua
-- ตัวอย่างที่ 9: Random Load Balancing
math.randomseed(os.time())

local RandomLB = {}
RandomLB.__index = RandomLB

function RandomLB.new(servers)
    local self = setmetatable({}, RandomLB)
    self.servers = {}
    for _, s in ipairs(servers) do
        if s.active ~= false then
            table.insert(self.servers, s)
        end
    end
    return self
end

function RandomLB:next()
    if #self.servers == 0 then return nil end
    return self.servers[math.random(#self.servers)]
end

local lb = RandomLB.new({
    {host = "s1", port = 80},
    {host = "s2", port = 80},
    {host = "s3", port = 80},
})

local count = {}
for i = 1, 300 do
    local s = lb:next()
    count[s.host] = (count[s.host] or 0) + 1
end

print("Random distribution (300 requests):")
for host, c in pairs(count) do
    print(string.format("  %s: %d (%.1f%%)", host, c, c/3))
end
```

```lua
-- ตัวอย่างที่ 10: Power of Two Choices (Random สองตัวเลือกที่ดีกว่า)
local PowerOfTwo = {}
PowerOfTwo.__index = PowerOfTwo

function PowerOfTwo.new(servers)
    local self = setmetatable({}, PowerOfTwo)
    self.servers = {}
    for _, s in ipairs(servers) do
        table.insert(self.servers, {
            host        = s.host,
            port        = s.port,
            connections = 0,
            active      = true,
        })
    end
    return self
end

function PowerOfTwo:next()
    local active = {}
    for _, s in ipairs(self.servers) do
        if s.active then table.insert(active, s) end
    end
    if #active == 0 then return nil end
    if #active == 1 then return active[1] end

    -- สุ่มสองตัว แล้วเลือกตัวที่ connections น้อยกว่า
    local i1 = math.random(#active)
    local i2
    repeat i2 = math.random(#active) until i2 ~= i1
    local s1, s2 = active[i1], active[i2]
    local chosen = s1.connections <= s2.connections and s1 or s2
    chosen.connections = chosen.connections + 1
    return chosen
end

local lb = PowerOfTwo.new({
    {host = "s1", port = 80},
    {host = "s2", port = 80},
    {host = "s3", port = 80},
    {host = "s4", port = 80},
})

print("Power of Two Choices:")
for i = 1, 8 do
    local s = lb:next()
    print(string.format("  Request %d -> %s (connections=%d)", i, s.host, s.connections))
end
```

---

## 66.6 Consistent Hashing

Consistent Hashing ใช้สำหรับ stateful services เพื่อลดการ remapping เมื่อเพิ่ม/ลดเซิร์ฟเวอร์

```lua
-- ตัวอย่างที่ 11: Consistent Hashing Ring
local ConsistentHash = {}
ConsistentHash.__index = ConsistentHash

local function hash(key)
    local h = 0
    for i = 1, #key do
        h = (h * 31 + key:byte(i)) % (2^32)
    end
    return h
end

function ConsistentHash.new(replicas)
    local self       = setmetatable({}, ConsistentHash)
    self.replicas    = replicas or 100
    self.ring        = {}   -- {hash -> node}
    self.sorted_keys = {}
    return self
end

function ConsistentHash:addNode(node)
    for i = 1, self.replicas do
        local key = hash(node .. "#" .. i)
        self.ring[key] = node
        table.insert(self.sorted_keys, key)
    end
    table.sort(self.sorted_keys)
end

function ConsistentHash:removeNode(node)
    for i = 1, self.replicas do
        local key = hash(node .. "#" .. i)
        self.ring[key] = nil
    end
    local new_keys = {}
    for _, k in ipairs(self.sorted_keys) do
        if self.ring[k] ~= nil then
            table.insert(new_keys, k)
        end
    end
    self.sorted_keys = new_keys
end

function ConsistentHash:getNode(key)
    if #self.sorted_keys == 0 then return nil end
    local h = hash(key)
    -- Binary search สำหรับ key ที่ >= h
    local lo, hi = 1, #self.sorted_keys
    local idx = 1
    while lo <= hi do
        local mid = math.floor((lo + hi) / 2)
        if self.sorted_keys[mid] >= h then
            idx = mid
            hi  = mid - 1
        else
            lo = mid + 1
        end
    end
    -- Wrap around
    if self.ring[self.sorted_keys[idx]] == nil then
        idx = 1
    end
    return self.ring[self.sorted_keys[idx]]
end

-- ทดสอบ
local ch = ConsistentHash.new(50)
ch:addNode("server-A")
ch:addNode("server-B")
ch:addNode("server-C")

local keys = {"user:1001", "user:1002", "user:1003", "session:abc", "session:xyz"}
print("Before removing server-B:")
for _, k in ipairs(keys) do
    print(string.format("  %s -> %s", k, ch:getNode(k)))
end

ch:removeNode("server-B")
print("\nAfter removing server-B:")
for _, k in ipairs(keys) do
    print(string.format("  %s -> %s", k, ch:getNode(k)))
end
```

```lua
-- ตัวอย่างที่ 12: Consistent Hashing with Virtual Nodes และ weighted distribution
local WeightedConsistentHash = {}
WeightedConsistentHash.__index = WeightedConsistentHash

local function murmur32(key)
    -- Simplified murmur-like hash
    local h = 0x9747b28c
    for i = 1, #key do
        h = h ~ (key:byte(i) * 0xcc9e2d51)
        h = ((h << 15) | (h >> 17)) & 0xFFFFFFFF
        h = (h * 0x1b873593) & 0xFFFFFFFF
    end
    h = h ~ (h >> 16)
    h = (h * 0x85ebca6b) & 0xFFFFFFFF
    h = h ~ (h >> 13)
    h = (h * 0xc2b2ae35) & 0xFFFFFFFF
    h = h ~ (h >> 16)
    return h
end

function WeightedConsistentHash.new()
    local self       = setmetatable({}, WeightedConsistentHash)
    self.ring        = {}
    self.sorted_keys = {}
    return self
end

function WeightedConsistentHash:addNode(name, weight)
    local replicas = weight * 10
    for i = 1, replicas do
        local k = murmur32(name .. ":" .. i)
        self.ring[k] = name
        table.insert(self.sorted_keys, k)
    end
    table.sort(self.sorted_keys)
end

function WeightedConsistentHash:get(key)
    if #self.sorted_keys == 0 then return nil end
    local h  = murmur32(key)
    for _, k in ipairs(self.sorted_keys) do
        if k >= h then return self.ring[k] end
    end
    return self.ring[self.sorted_keys[1]]
end

local wch = WeightedConsistentHash.new()
wch:addNode("heavy", 5)
wch:addNode("medium", 3)
wch:addNode("light", 1)

local dist = {}
for i = 1, 900 do
    local node = wch:get("key-" .. i)
    dist[node] = (dist[node] or 0) + 1
end

print("Weighted Consistent Hash distribution (900 keys):")
for node, c in pairs(dist) do
    print(string.format("  %s: %d keys", node, c))
end
```

---

## 66.7 Health Check Integration

```lua
-- ตัวอย่างที่ 13: Health Check Manager
local HealthChecker = {}
HealthChecker.__index = HealthChecker

function HealthChecker.new(config)
    local self = setmetatable({}, HealthChecker)
    self.interval   = config.interval   or 10   -- seconds
    self.timeout    = config.timeout    or 2
    self.threshold  = config.threshold  or 3    -- failures before mark down
    self.servers    = {}
    return self
end

function HealthChecker:addServer(host, port, path)
    table.insert(self.servers, {
        host     = host,
        port     = port,
        path     = path or "/health",
        healthy  = true,
        failures = 0,
        successes = 0,
        last_check = 0,
    })
end

-- Simulate HTTP health check (ใน production ใช้ socket library จริง)
function HealthChecker:checkServer(server)
    -- จำลองผลลัพธ์ (random สำหรับ demo)
    local ok = math.random() > 0.2
    return ok, ok and 200 or 503
end

function HealthChecker:runChecks()
    for _, s in ipairs(self.servers) do
        local ok, status = self:checkServer(s)
        s.last_check = os.time()

        if ok then
            s.failures  = 0
            s.successes = s.successes + 1
            if not s.healthy then
                s.healthy = true
                print(string.format("[HEALTH] %s:%d is UP (HTTP %d)", s.host, s.port, status))
            end
        else
            s.failures  = s.failures + 1
            s.successes = 0
            if s.failures >= self.threshold and s.healthy then
                s.healthy = false
                print(string.format("[HEALTH] %s:%d is DOWN after %d failures",
                    s.host, s.port, s.failures))
            end
        end
    end
end

function HealthChecker:healthyServers()
    local result = {}
    for _, s in ipairs(self.servers) do
        if s.healthy then table.insert(result, s) end
    end
    return result
end

math.randomseed(42)
local hc = HealthChecker.new({interval = 10, threshold = 2})
hc:addServer("10.0.0.1", 8080)
hc:addServer("10.0.0.2", 8080)
hc:addServer("10.0.0.3", 8080)

print("Running 3 health check rounds:")
for round = 1, 3 do
    print("\n-- Round " .. round .. " --")
    hc:runChecks()
    local healthy = hc:healthyServers()
    print(string.format("Healthy servers: %d", #healthy))
end
```

```lua
-- ตัวอย่างที่ 14: HTTP Health Check endpoint parser
local HealthStatus = {}
HealthStatus.__index = HealthStatus

function HealthStatus.new()
    local self    = setmetatable({}, HealthStatus)
    self.checks   = {}
    self.status   = "healthy"
    return self
end

function HealthStatus:addCheck(name, fn)
    table.insert(self.checks, {name = name, fn = fn})
end

function HealthStatus:run()
    local results = {}
    local overall = "healthy"

    for _, check in ipairs(self.checks) do
        local ok, detail = pcall(check.fn)
        local status
        if ok and detail ~= false then
            status = "pass"
        else
            status  = "fail"
            overall = "unhealthy"
        end
        results[check.name] = {status = status, detail = tostring(detail)}
    end

    self.status = overall
    return {
        status = overall,
        checks = results,
        timestamp = os.time(),
    }
end

function HealthStatus:toJSON(report)
    local lines = {}
    table.insert(lines, '{"status":"' .. report.status .. '","checks":{')
    local checkLines = {}
    for name, v in pairs(report.checks) do
        table.insert(checkLines, string.format(
            '"%s":{"status":"%s","detail":"%s"}',
            name, v.status, v.detail))
    end
    table.insert(lines, table.concat(checkLines, ","))
    table.insert(lines, '},"timestamp":' .. report.timestamp .. '}')
    return table.concat(lines)
end

local hs = HealthStatus.new()

hs:addCheck("database", function()
    -- จำลองการ check database
    return true, "connected"
end)

hs:addCheck("cache", function()
    return true, "connected"
end)

hs:addCheck("disk", function()
    -- จำลองการ check disk space
    local freePercent = 45
    if freePercent < 10 then
        return false, "disk usage critical"
    end
    return true, string.format("%.1f%% free", freePercent)
end)

local report = hs:run()
print(hs:toJSON(report))
```

---

## 66.8 Sticky Sessions

Sticky Sessions ทำให้ client เดิมถูก route ไปยังเซิร์ฟเวอร์เดิมตลอดเวลา

```lua
-- ตัวอย่างที่ 15: Cookie-based Sticky Sessions
local StickySessions = {}
StickySessions.__index = StickySessions

function StickySessions.new(servers, cookieName)
    local self        = setmetatable({}, StickySessions)
    self.servers      = {}
    self.cookieName   = cookieName or "SERVERID"
    self.sessions     = {}   -- sessionId -> serverIndex
    self.ttl          = 3600 -- 1 hour

    for i, s in ipairs(servers) do
        self.servers[i] = {
            id   = "s" .. i,
            host = s.host,
            port = s.port,
            active = true,
        }
    end
    return self
end

function StickySessions:getServerId(cookieValue)
    -- ดึง server id จาก cookie
    return cookieValue
end

function StickySessions:findServer(serverId)
    for _, s in ipairs(self.servers) do
        if s.id == serverId and s.active then
            return s
        end
    end
    return nil
end

function StickySessions:selectServer()
    -- Round robin เลือกเซิร์ฟเวอร์ใหม่
    self._rr = ((self._rr or 0) % #self.servers) + 1
    return self.servers[self._rr]
end

function StickySessions:route(request)
    local cookie = request.cookies and request.cookies[self.cookieName]
    local server

    if cookie then
        local serverId = self:getServerId(cookie)
        server = self:findServer(serverId)
    end

    if not server then
        server = self:selectServer()
    end

    return server, server.id  -- return server and new cookie value
end

local lb = StickySessions.new({
    {host = "10.0.0.1", port = 8080},
    {host = "10.0.0.2", port = 8080},
    {host = "10.0.0.3", port = 8080},
}, "BACKEND")

-- Client 1: ไม่มี cookie (request แรก)
local s1, cookie1 = lb:route({cookies = {}})
print(string.format("Client-1 first request -> %s (cookie: %s)", s1.host, cookie1))

-- Client 1: มี cookie แล้ว
local s2, cookie2 = lb:route({cookies = {BACKEND = cookie1}})
print(string.format("Client-1 second request -> %s (sticky!)", s2.host))

-- Client 2: ไม่มี cookie
local s3, cookie3 = lb:route({cookies = {}})
print(string.format("Client-2 first request -> %s (cookie: %s)", s3.host, cookie3))
```

```lua
-- ตัวอย่างที่ 16: Session Store สำหรับ Sticky Sessions
local SessionStore = {}
SessionStore.__index = SessionStore

function SessionStore.new(ttl)
    local self     = setmetatable({}, SessionStore)
    self.store     = {}
    self.ttl       = ttl or 3600
    return self
end

function SessionStore:set(sessionId, serverId)
    self.store[sessionId] = {
        serverId  = serverId,
        createdAt = os.time(),
        expiresAt = os.time() + self.ttl,
    }
end

function SessionStore:get(sessionId)
    local entry = self.store[sessionId]
    if not entry then return nil end
    if os.time() > entry.expiresAt then
        self.store[sessionId] = nil
        return nil
    end
    return entry.serverId
end

function SessionStore:delete(sessionId)
    self.store[sessionId] = nil
end

function SessionStore:cleanup()
    local now     = os.time()
    local removed = 0
    for id, entry in pairs(self.store) do
        if now > entry.expiresAt then
            self.store[id] = nil
            removed = removed + 1
        end
    end
    return removed
end

function SessionStore:count()
    local n = 0
    for _ in pairs(self.store) do n = n + 1 end
    return n
end

local store = SessionStore.new(60)
store:set("sess-001", "server-A")
store:set("sess-002", "server-B")
store:set("sess-003", "server-A")

print("Session store (3 entries):")
print("sess-001 -> " .. (store:get("sess-001") or "expired"))
print("sess-002 -> " .. (store:get("sess-002") or "expired"))
print("Total sessions: " .. store:count())

store:delete("sess-002")
print("After delete: Total sessions = " .. store:count())
```

---

## 66.9 Connection Draining

Connection Draining ช่วยให้เซิร์ฟเวอร์ที่กำลังจะถอดออกสามารถ serve existing connections ให้เสร็จก่อน

```lua
-- ตัวอย่างที่ 17: Connection Draining
local ConnectionDrainer = {}
ConnectionDrainer.__index = ConnectionDrainer

local STATE = {
    ACTIVE   = "active",
    DRAINING = "draining",
    DRAINED  = "drained",
}

function ConnectionDrainer.new(servers, drainTimeout)
    local self         = setmetatable({}, ConnectionDrainer)
    self.drainTimeout  = drainTimeout or 30
    self.servers       = {}
    for _, s in ipairs(servers) do
        table.insert(self.servers, {
            host        = s.host,
            port        = s.port,
            state       = STATE.ACTIVE,
            connections = 0,
            drainStart  = nil,
        })
    end
    return self
end

function ConnectionDrainer:startDrain(host)
    for _, s in ipairs(self.servers) do
        if s.host == host then
            if s.state == STATE.ACTIVE then
                s.state     = STATE.DRAINING
                s.drainStart = os.time()
                print(string.format("[DRAIN] Starting drain for %s (connections: %d)",
                    host, s.connections))
            end
            return
        end
    end
end

function ConnectionDrainer:acquire(host)
    for _, s in ipairs(self.servers) do
        if s.host == host and s.state == STATE.ACTIVE then
            s.connections = s.connections + 1
            return true
        end
    end
    return false  -- server is draining or not found
end

function ConnectionDrainer:release(host)
    for _, s in ipairs(self.servers) do
        if s.host == host then
            if s.connections > 0 then
                s.connections = s.connections - 1
            end
            -- Check if drain is complete
            if s.state == STATE.DRAINING and s.connections == 0 then
                s.state = STATE.DRAINED
                print(string.format("[DRAIN] %s is fully drained!", host))
            end
            return
        end
    end
end

function ConnectionDrainer:canReceive(host)
    for _, s in ipairs(self.servers) do
        if s.host == host then
            return s.state == STATE.ACTIVE
        end
    end
    return false
end

function ConnectionDrainer:status()
    for _, s in ipairs(self.servers) do
        print(string.format("  %s: state=%s, connections=%d",
            s.host, s.state, s.connections))
    end
end

local drainer = ConnectionDrainer.new({
    {host = "10.0.0.1", port = 8080},
    {host = "10.0.0.2", port = 8080},
})

-- Simulate active connections
drainer:acquire("10.0.0.1")
drainer:acquire("10.0.0.1")
drainer:acquire("10.0.0.1")
drainer:acquire("10.0.0.2")

print("Initial state:")
drainer:status()

-- Start draining server 1
drainer:startDrain("10.0.0.1")
print("\nCan server 1 receive new connections? " ..
    tostring(drainer:canReceive("10.0.0.1")))

-- Release connections gradually
drainer:release("10.0.0.1")
drainer:release("10.0.0.1")
drainer:release("10.0.0.1")

print("\nFinal state:")
drainer:status()
```

---

## 66.10 Blue-Green Deployment

```lua
-- ตัวอย่างที่ 18: Blue-Green Deployment
local BlueGreen = {}
BlueGreen.__index = BlueGreen

function BlueGreen.new()
    local self    = setmetatable({}, BlueGreen)
    self.blue     = {}
    self.green    = {}
    self.active   = "blue"  -- traffic goes to blue by default
    self.lb_index = 0
    return self
end

function BlueGreen:setBlue(servers)
    self.blue = servers
end

function BlueGreen:setGreen(servers)
    self.green = servers
end

function BlueGreen:activePool()
    return self.active == "blue" and self.blue or self.green
end

function BlueGreen:inactivePool()
    return self.active == "blue" and self.green or self.blue
end

function BlueGreen:next()
    local pool = self:activePool()
    if #pool == 0 then return nil end
    self.lb_index = (self.lb_index % #pool) + 1
    return pool[self.lb_index]
end

function BlueGreen:switchTraffic()
    local from = self.active
    self.active = self.active == "blue" and "green" or "blue"
    self.lb_index = 0
    print(string.format("[DEPLOY] Traffic switched from %s to %s", from, self.active))
end

function BlueGreen:rollback()
    self:switchTraffic()
    print("[DEPLOY] Rollback complete!")
end

function BlueGreen:status()
    print(string.format("Active environment: %s", self.active))
    print(string.format("Blue servers: %d, Green servers: %d",
        #self.blue, #self.green))
end

local bg = BlueGreen.new()

-- Setup environments
bg:setBlue({
    {host = "blue-1", port = 8080, version = "v1.0"},
    {host = "blue-2", port = 8080, version = "v1.0"},
})
bg:setGreen({
    {host = "green-1", port = 8080, version = "v2.0"},
    {host = "green-2", port = 8080, version = "v2.0"},
})

print("=== Blue-Green Deployment Demo ===")
bg:status()

print("\nServing traffic (v1.0 - Blue):")
for i = 1, 4 do
    local s = bg:next()
    print(string.format("  Request %d -> %s (%s)", i, s.host, s.version))
end

print("\nSwitching to Green (v2.0)...")
bg:switchTraffic()

print("Serving traffic (v2.0 - Green):")
for i = 1, 4 do
    local s = bg:next()
    print(string.format("  Request %d -> %s (%s)", i, s.host, s.version))
end

print("\nSimulating rollback...")
bg:rollback()

print("After rollback:")
for i = 1, 2 do
    local s = bg:next()
    print(string.format("  Request %d -> %s (%s)", i, s.host, s.version))
end
```

---

## 66.11 Canary Releases

```lua
-- ตัวอย่างที่ 19: Canary Release
local CanaryRelease = {}
CanaryRelease.__index = CanaryRelease

function CanaryRelease.new(stable, canary, canaryPercent)
    local self           = setmetatable({}, CanaryRelease)
    self.stable          = stable
    self.canary          = canary
    self.canaryPercent   = canaryPercent or 5  -- 5% default
    self.stableIndex     = 0
    self.canaryIndex     = 0
    self.stats           = {stable = 0, canary = 0}
    return self
end

function CanaryRelease:next()
    local roll = math.random(100)
    if roll <= self.canaryPercent then
        -- Route to canary
        self.canaryIndex = (self.canaryIndex % #self.canary) + 1
        self.stats.canary = self.stats.canary + 1
        return self.canary[self.canaryIndex], "canary"
    else
        -- Route to stable
        self.stableIndex = (self.stableIndex % #self.stable) + 1
        self.stats.stable = self.stats.stable + 1
        return self.stable[self.stableIndex], "stable"
    end
end

function CanaryRelease:setCanaryPercent(pct)
    self.canaryPercent = math.max(0, math.min(100, pct))
    print(string.format("[CANARY] Traffic to canary: %d%%", self.canaryPercent))
end

function CanaryRelease:promote()
    -- Promote canary to stable
    self.stable        = self.canary
    self.canary        = {}
    self.canaryPercent = 0
    print("[CANARY] Promoted! Canary is now stable.")
end

function CanaryRelease:report()
    local total = self.stats.stable + self.stats.canary
    if total == 0 then return end
    print(string.format("Canary report: stable=%d (%.1f%%), canary=%d (%.1f%%)",
        self.stats.stable, self.stats.stable/total*100,
        self.stats.canary, self.stats.canary/total*100))
end

math.randomseed(42)

local cr = CanaryRelease.new(
    {
        {host = "stable-1", version = "v1.5"},
        {host = "stable-2", version = "v1.5"},
    },
    {
        {host = "canary-1", version = "v2.0-beta"},
    },
    10  -- 10% canary
)

print("=== Canary Release (10% traffic) ===")
for i = 1, 20 do
    local s, env = cr:next()
    if env == "canary" then
        print(string.format("  Request %d -> [CANARY] %s (%s)", i, s.host, s.version))
    end
end

cr:report()

print("\nGradually increasing canary traffic...")
cr:setCanaryPercent(25)
cr:setCanaryPercent(50)
cr:setCanaryPercent(100)
cr:promote()
```

---

## 66.12 A/B Testing Routing

```lua
-- ตัวอย่างที่ 20: A/B Testing Router
local ABTestRouter = {}
ABTestRouter.__index = ABTestRouter

function ABTestRouter.new()
    local self   = setmetatable({}, ABTestRouter)
    self.tests   = {}
    self.results = {}
    return self
end

function ABTestRouter:addTest(name, config)
    self.tests[name] = {
        name     = name,
        groups   = config.groups,  -- [{id, servers, weight}]
        active   = true,
    }
    self.results[name] = {}
    for _, g in ipairs(config.groups) do
        self.results[name][g.id] = {requests = 0, conversions = 0}
    end
end

function ABTestRouter:route(testName, userId)
    local test = self.tests[testName]
    if not test or not test.active then return nil, nil end

    -- Deterministic assignment based on userId
    local hash = 0
    for c in userId:gmatch(".") do
        hash = (hash * 31 + c:byte()) % 100
    end

    local cumulative = 0
    for _, group in ipairs(test.groups) do
        cumulative = cumulative + group.weight
        if hash < cumulative then
            self.results[testName][group.id].requests =
                self.results[testName][group.id].requests + 1
            -- Round-robin within group
            group._idx = ((group._idx or 0) % #group.servers) + 1
            return group.servers[group._idx], group.id
        end
    end

    -- Fallback to last group
    local last = test.groups[#test.groups]
    return last.servers[1], last.id
end

function ABTestRouter:recordConversion(testName, groupId)
    if self.results[testName] and self.results[testName][groupId] then
        self.results[testName][groupId].conversions =
            self.results[testName][groupId].conversions + 1
    end
end

function ABTestRouter:report(testName)
    print(string.format("=== A/B Test Report: %s ===", testName))
    for groupId, stats in pairs(self.results[testName]) do
        local rate = 0
        if stats.requests > 0 then
            rate = stats.conversions / stats.requests * 100
        end
        print(string.format("  Group %s: %d requests, %d conversions, %.1f%% CVR",
            groupId, stats.requests, stats.conversions, rate))
    end
end

local router = ABTestRouter.new()

router:addTest("homepage_redesign", {
    groups = {
        {id = "control", servers = {{host = "app-v1"}}, weight = 50},
        {id = "variant", servers = {{host = "app-v2"}}, weight = 50},
    }
})

local users = {"u001","u002","u003","u004","u005","u006","u007","u008","u009","u010"}

print("A/B Test routing:")
for _, uid in ipairs(users) do
    local server, group = router:route("homepage_redesign", uid)
    print(string.format("  user %s -> [%s] %s", uid, group, server.host))
    -- Simulate some conversions
    if math.random() > 0.7 then
        router:recordConversion("homepage_redesign", group)
    end
end

router:report("homepage_redesign")
```

---

## 66.13 Load Balancer แบบ Pure Lua

```lua
-- ตัวอย่างที่ 21: Full-featured Load Balancer
local LoadBalancer = {}
LoadBalancer.__index = LoadBalancer

function LoadBalancer.new(config)
    local self         = setmetatable({}, LoadBalancer)
    self.algorithm     = config.algorithm or "round_robin"
    self.servers       = {}
    self.healthCheck   = config.healthCheck or false
    self._rrIndex      = 0
    self.stats         = {
        total    = 0,
        errors   = 0,
        byServer = {}
    }
    return self
end

function LoadBalancer:addServer(server)
    local s = {
        host        = server.host,
        port        = server.port or 80,
        weight      = server.weight or 1,
        active      = true,
        connections = 0,
        requests    = 0,
        errors      = 0,
    }
    table.insert(self.servers, s)
    self.stats.byServer[s.host] = {requests = 0, errors = 0}
end

function LoadBalancer:activeServers()
    local result = {}
    for _, s in ipairs(self.servers) do
        if s.active then table.insert(result, s) end
    end
    return result
end

function LoadBalancer:_roundRobin()
    local active = self:activeServers()
    if #active == 0 then return nil end
    self._rrIndex = (self._rrIndex % #active) + 1
    return active[self._rrIndex]
end

function LoadBalancer:_leastConn()
    local active = self:activeServers()
    if #active == 0 then return nil end
    local best = active[1]
    for _, s in ipairs(active) do
        if s.connections < best.connections then best = s end
    end
    return best
end

function LoadBalancer:_random()
    local active = self:activeServers()
    if #active == 0 then return nil end
    return active[math.random(#active)]
end

function LoadBalancer:_weightedRR()
    local active = self:activeServers()
    if #active == 0 then return nil end
    local pool = {}
    for _, s in ipairs(active) do
        for _ = 1, s.weight do table.insert(pool, s) end
    end
    self._wrrIdx = ((self._wrrIdx or 0) % #pool) + 1
    return pool[self._wrrIdx]
end

function LoadBalancer:select()
    local algo = self.algorithm
    local server
    if algo == "round_robin" then
        server = self:_roundRobin()
    elseif algo == "least_conn" then
        server = self:_leastConn()
    elseif algo == "random" then
        server = self:_random()
    elseif algo == "weighted" then
        server = self:_weightedRR()
    end
    return server
end

function LoadBalancer:handleRequest(requestId)
    local server = self:select()
    if not server then
        self.stats.errors = self.stats.errors + 1
        return nil, "no available server"
    end

    server.connections = server.connections + 1
    server.requests    = server.requests + 1
    self.stats.total   = self.stats.total + 1
    self.stats.byServer[server.host].requests =
        self.stats.byServer[server.host].requests + 1

    return server, nil
end

function LoadBalancer:completeRequest(server, success)
    if server.connections > 0 then
        server.connections = server.connections - 1
    end
    if not success then
        server.errors = server.errors + 1
        self.stats.byServer[server.host].errors =
            self.stats.byServer[server.host].errors + 1
    end
end

function LoadBalancer:printStats()
    print(string.format("Total requests: %d, Errors: %d",
        self.stats.total, self.stats.errors))
    for host, s in pairs(self.stats.byServer) do
        print(string.format("  %s: requests=%d, errors=%d",
            host, s.requests, s.errors))
    end
end

-- ทดสอบ
math.randomseed(os.time())

local lb = LoadBalancer.new({algorithm = "least_conn"})
lb:addServer({host = "app-1", port = 8080, weight = 2})
lb:addServer({host = "app-2", port = 8080, weight = 1})
lb:addServer({host = "app-3", port = 8080, weight = 3})

print("=== Full Load Balancer Demo ===")
local active_reqs = {}
for i = 1, 10 do
    local server, err = lb:handleRequest("req-" .. i)
    if server then
        print(string.format("Request %d -> %s (connections: %d)",
            i, server.host, server.connections))
        table.insert(active_reqs, server)
    else
        print("Error: " .. err)
    end
end

-- Complete requests
for _, s in ipairs(active_reqs) do
    lb:completeRequest(s, math.random() > 0.1)
end

print("\nFinal statistics:")
lb:printStats()
```

---

## 66.14 OpenResty / Nginx Upstream Configuration

```lua
-- ตัวอย่างที่ 22: Nginx upstream config generator
local NginxConfig = {}
NginxConfig.__index = NginxConfig

function NginxConfig.new()
    local self    = setmetatable({}, NginxConfig)
    self.upstream = {}
    return self
end

function NginxConfig:addUpstream(name, config)
    self.upstream[name] = {
        name     = name,
        servers  = config.servers or {},
        method   = config.method or "round_robin",
        keepalive = config.keepalive or 32,
    }
end

function NginxConfig:generateUpstream(name)
    local up = self.upstream[name]
    if not up then return "" end

    local lines = {}
    table.insert(lines, "upstream " .. name .. " {")

    if up.method == "least_conn" then
        table.insert(lines, "    least_conn;")
    elseif up.method == "ip_hash" then
        table.insert(lines, "    ip_hash;")
    end

    for _, s in ipairs(up.servers) do
        local line = string.format("    server %s:%d", s.host, s.port)
        if s.weight and s.weight > 1 then
            line = line .. " weight=" .. s.weight
        end
        if s.max_fails then
            line = line .. " max_fails=" .. s.max_fails
        end
        if s.fail_timeout then
            line = line .. " fail_timeout=" .. s.fail_timeout .. "s"
        end
        if s.backup then
            line = line .. " backup"
        end
        if s.down then
            line = line .. " down"
        end
        line = line .. ";"
        table.insert(lines, line)
    end

    table.insert(lines, string.format("    keepalive %d;", up.keepalive))
    table.insert(lines, "}")
    return table.concat(lines, "\n")
end

local cfg = NginxConfig.new()

cfg:addUpstream("backend", {
    method   = "least_conn",
    keepalive = 64,
    servers  = {
        {host = "10.0.0.1", port = 8080, weight = 3, max_fails = 3, fail_timeout = 30},
        {host = "10.0.0.2", port = 8080, weight = 2, max_fails = 3, fail_timeout = 30},
        {host = "10.0.0.3", port = 8080, weight = 1, max_fails = 3, fail_timeout = 30},
        {host = "10.0.0.4", port = 8080, backup = true},
    }
})

print(cfg:generateUpstream("backend"))
```

```lua
-- ตัวอย่างที่ 23: OpenResty Dynamic Upstream (Lua script สำหรับ nginx.conf)
-- หมายเหตุ: โค้ดนี้ทำงานใน OpenResty (nginx + LuaJIT)

--[[
-- nginx.conf snippet:
http {
    lua_shared_dict upstreams 1m;

    init_by_lua_block {
        -- Load upstream list at startup
        local upstreams = ngx.shared.upstreams
        upstreams:set("backend:servers", cjson.encode({
            {host="10.0.0.1", port=8080, weight=3},
            {host="10.0.0.2", port=8080, weight=2},
        }))
    }

    upstream dynamic_backend {
        server 0.0.0.1;  -- placeholder
        balancer_by_lua_block {
            local balancer = require "ngx.balancer"
            local cjson    = require "cjson"
            local shared   = ngx.shared.upstreams
            local servers_json = shared:get("backend:servers")
            local servers  = cjson.decode(servers_json)
            -- Simple round-robin using shared dict counter
            local idx_key  = "backend:index"
            local idx      = (shared:get(idx_key) or 0) % #servers + 1
            shared:set(idx_key, idx)
            local s = servers[idx]
            local ok, err = balancer.set_current_peer(s.host, s.port)
            if not ok then
                ngx.log(ngx.ERR, "Failed to set peer: ", err)
                return ngx.exit(500)
            end
        }
    }
}
--]]

-- Standalone simulation of the logic above:
local function simulateDynamicUpstream()
    local cjson_sim = {
        encode = function(t)
            -- minimal JSON encoder for demo
            local parts = {}
            for _, v in ipairs(t) do
                table.insert(parts, string.format(
                    '{"host":"%s","port":%d,"weight":%d}',
                    v.host, v.port, v.weight))
            end
            return "[" .. table.concat(parts, ",") .. "]"
        end,
        decode = function(s)
            -- Use load() as a simple deserializer placeholder
            local result = {}
            for host, port, weight in s:gmatch('"host":"([^"]+)","port":(%d+),"weight":(%d+)') do
                table.insert(result, {host=host, port=tonumber(port), weight=tonumber(weight)})
            end
            return result
        end,
    }

    local servers_json = cjson_sim.encode({
        {host="10.0.0.1", port=8080, weight=3},
        {host="10.0.0.2", port=8080, weight=2},
    })

    print("Stored upstream JSON:")
    print(servers_json)

    local servers = cjson_sim.decode(servers_json)
    print("\nParsed servers:")
    for i, s in ipairs(servers) do
        print(string.format("  [%d] %s:%d weight=%d", i, s.host, s.port, s.weight))
    end
end

simulateDynamicUpstream()
```

---

## 66.15 Dynamic Upstream Updates

```lua
-- ตัวอย่างที่ 24: Dynamic Upstream Manager with hot-reload
local DynamicUpstream = {}
DynamicUpstream.__index = DynamicUpstream

function DynamicUpstream.new()
    local self    = setmetatable({}, DynamicUpstream)
    self.pools    = {}
    self.version  = 0
    self.listeners = {}
    return self
end

function DynamicUpstream:setPool(name, servers)
    local oldPool = self.pools[name]
    self.pools[name] = {
        servers  = servers,
        version  = self.version + 1,
        updatedAt = os.time(),
    }
    self.version = self.version + 1

    -- Notify listeners
    for _, fn in ipairs(self.listeners) do
        fn(name, oldPool, self.pools[name])
    end
end

function DynamicUpstream:getPool(name)
    return self.pools[name]
end

function DynamicUpstream:addServer(poolName, server)
    local pool = self.pools[poolName]
    if not pool then
        self:setPool(poolName, {server})
        return
    end
    -- Check for duplicates
    for _, s in ipairs(pool.servers) do
        if s.host == server.host and s.port == server.port then
            print(string.format("[UPSTREAM] %s:%d already exists in pool %s",
                server.host, server.port, poolName))
            return
        end
    end
    table.insert(pool.servers, server)
    pool.version  = pool.version + 1
    pool.updatedAt = os.time()
    print(string.format("[UPSTREAM] Added %s:%d to pool '%s'",
        server.host, server.port, poolName))
end

function DynamicUpstream:removeServer(poolName, host, port)
    local pool = self.pools[poolName]
    if not pool then return end
    local new = {}
    for _, s in ipairs(pool.servers) do
        if not (s.host == host and s.port == port) then
            table.insert(new, s)
        end
    end
    pool.servers   = new
    pool.version   = pool.version + 1
    pool.updatedAt  = os.time()
    print(string.format("[UPSTREAM] Removed %s:%d from pool '%s'",
        host, port, poolName))
end

function DynamicUpstream:onUpdate(fn)
    table.insert(self.listeners, fn)
end

function DynamicUpstream:listPools()
    print("Upstream pools:")
    for name, pool in pairs(self.pools) do
        print(string.format("  %s (v%d, %d servers):",
            name, pool.version, #pool.servers))
        for _, s in ipairs(pool.servers) do
            print(string.format("    - %s:%d weight=%d active=%s",
                s.host, s.port, s.weight or 1, tostring(s.active ~= false)))
        end
    end
end

local upstream = DynamicUpstream.new()

-- Listen for changes
upstream:onUpdate(function(name, old, new)
    print(string.format("[EVENT] Pool '%s' updated to v%d", name, new.version))
end)

upstream:setPool("api", {
    {host = "10.0.0.1", port = 8080, weight = 2},
    {host = "10.0.0.2", port = 8080, weight = 2},
})

upstream:addServer("api", {host = "10.0.0.3", port = 8080, weight = 1})
upstream:removeServer("api", "10.0.0.2", 8080)

upstream:listPools()
```

---

## 66.16 Rate Limiting per Backend

```lua
-- ตัวอย่างที่ 25: Rate Limiter สำหรับ backend requests
local RateLimiter = {}
RateLimiter.__index = RateLimiter

function RateLimiter.new(rps, burst)
    local self    = setmetatable({}, RateLimiter)
    self.rps      = rps    -- requests per second
    self.burst    = burst  -- max burst
    self.tokens   = burst
    self.lastTime = os.clock()
    return self
end

function RateLimiter:allow()
    local now    = os.clock()
    local elapsed = now - self.lastTime
    self.lastTime = now

    -- Refill tokens
    self.tokens = math.min(self.burst, self.tokens + elapsed * self.rps)

    if self.tokens >= 1 then
        self.tokens = self.tokens - 1
        return true
    end
    return false
end

-- Per-server rate limiters
local ServerRateLimiter = {}
ServerRateLimiter.__index = ServerRateLimiter

function ServerRateLimiter.new()
    local self    = setmetatable({}, ServerRateLimiter)
    self.limiters = {}
    return self
end

function ServerRateLimiter:setLimit(host, rps, burst)
    self.limiters[host] = RateLimiter.new(rps, burst)
end

function ServerRateLimiter:allow(host)
    local limiter = self.limiters[host]
    if not limiter then return true end  -- no limit set
    return limiter:allow()
end

local srl = ServerRateLimiter.new()
srl:setLimit("app-1", 100, 10)   -- 100 rps, burst 10
srl:setLimit("app-2", 50, 5)     -- 50 rps, burst 5

print("Rate limiter test (burst phase):")
local allowed, denied = 0, 0
for i = 1, 15 do
    if srl:allow("app-1") then
        allowed = allowed + 1
    else
        denied = denied + 1
    end
end
print(string.format("  app-1: allowed=%d, denied=%d", allowed, denied))
```

---

## 66.17 Circuit Breaker Pattern

```lua
-- ตัวอย่างที่ 26: Circuit Breaker
local CircuitBreaker = {}
CircuitBreaker.__index = CircuitBreaker

local CB_STATE = {
    CLOSED   = "CLOSED",
    OPEN     = "OPEN",
    HALF_OPEN = "HALF_OPEN",
}

function CircuitBreaker.new(config)
    local self            = setmetatable({}, CircuitBreaker)
    self.threshold        = config.threshold or 5
    self.timeout          = config.timeout   or 30
    self.halfOpenRequests = config.halfOpen  or 1
    self.state            = CB_STATE.CLOSED
    self.failures         = 0
    self.lastFailure      = 0
    self.halfOpenAttempts = 0
    return self
end

function CircuitBreaker:canRequest()
    if self.state == CB_STATE.CLOSED then
        return true
    elseif self.state == CB_STATE.OPEN then
        if os.time() - self.lastFailure >= self.timeout then
            self.state            = CB_STATE.HALF_OPEN
            self.halfOpenAttempts = 0
            print("[CB] State: OPEN -> HALF_OPEN")
            return true
        end
        return false
    elseif self.state == CB_STATE.HALF_OPEN then
        return self.halfOpenAttempts < self.halfOpenRequests
    end
    return false
end

function CircuitBreaker:onSuccess()
    if self.state == CB_STATE.HALF_OPEN then
        self.state    = CB_STATE.CLOSED
        self.failures = 0
        print("[CB] State: HALF_OPEN -> CLOSED (recovered)")
    else
        self.failures = math.max(0, self.failures - 1)
    end
end

function CircuitBreaker:onFailure()
    self.failures    = self.failures + 1
    self.lastFailure = os.time()

    if self.state == CB_STATE.HALF_OPEN then
        self.state = CB_STATE.OPEN
        print("[CB] State: HALF_OPEN -> OPEN (still failing)")
    elseif self.failures >= self.threshold then
        self.state = CB_STATE.OPEN
        print(string.format("[CB] State: CLOSED -> OPEN (failures: %d)", self.failures))
    end
end

function CircuitBreaker:call(fn)
    if not self:canRequest() then
        return nil, "circuit breaker OPEN"
    end

    if self.state == CB_STATE.HALF_OPEN then
        self.halfOpenAttempts = self.halfOpenAttempts + 1
    end

    local ok, result = pcall(fn)
    if ok then
        self:onSuccess()
        return result, nil
    else
        self:onFailure()
        return nil, result
    end
end

-- ทดสอบ
local cb = CircuitBreaker.new({threshold = 3, timeout = 1})
local callCount = 0

local function simulateRequest()
    callCount = callCount + 1
    if callCount <= 4 then
        error("Connection refused")
    end
    return "OK"
end

print("=== Circuit Breaker Demo ===")
for i = 1, 8 do
    local result, err = cb:call(simulateRequest)
    if result then
        print(string.format("Request %d: SUCCESS (%s) [CB: %s]", i, result, cb.state))
    else
        print(string.format("Request %d: FAIL (%s) [CB: %s]", i, err, cb.state))
    end
end
```

---

## 66.18 Load Balancer Metrics

```lua
-- ตัวอย่างที่ 27: LB Metrics Collector
local LBMetrics = {}
LBMetrics.__index = LBMetrics

function LBMetrics.new()
    local self = setmetatable({}, LBMetrics)
    self.counters   = {}
    self.histograms = {}
    self.startTime  = os.time()
    return self
end

function LBMetrics:inc(name, labels, value)
    local key = name
    if labels then
        local parts = {}
        for k, v in pairs(labels) do
            table.insert(parts, k .. "=" .. v)
        end
        table.sort(parts)
        key = name .. "{" .. table.concat(parts, ",") .. "}"
    end
    self.counters[key] = (self.counters[key] or 0) + (value or 1)
end

function LBMetrics:observe(name, value)
    if not self.histograms[name] then
        self.histograms[name] = {
            sum   = 0,
            count = 0,
            buckets = {10, 50, 100, 250, 500, 1000, 2500, 5000},
            values  = {},
        }
    end
    local h = self.histograms[name]
    h.sum   = h.sum + value
    h.count = h.count + 1
    table.insert(h.values, value)
end

function LBMetrics:percentile(name, p)
    local h = self.histograms[name]
    if not h or h.count == 0 then return 0 end
    local sorted = {}
    for _, v in ipairs(h.values) do table.insert(sorted, v) end
    table.sort(sorted)
    local idx = math.ceil(p / 100 * #sorted)
    return sorted[idx] or sorted[#sorted]
end

function LBMetrics:report()
    print("=== Load Balancer Metrics ===")
    print("\nCounters:")
    for k, v in pairs(self.counters) do
        print(string.format("  %s = %d", k, v))
    end
    print("\nHistograms:")
    for name, h in pairs(self.histograms) do
        local avg = h.count > 0 and (h.sum / h.count) or 0
        print(string.format("  %s: count=%d, avg=%.2fms, p50=%dms, p95=%dms, p99=%dms",
            name, h.count, avg,
            self:percentile(name, 50),
            self:percentile(name, 95),
            self:percentile(name, 99)))
    end
end

local metrics = LBMetrics.new()

-- Simulate requests
math.randomseed(42)
local servers = {"app-1", "app-2", "app-3"}
for i = 1, 100 do
    local server = servers[math.random(#servers)]
    local latency = math.random(5, 500)
    local success = math.random() > 0.05

    metrics:inc("lb_requests_total", {server = server, status = success and "200" or "500"})
    metrics:observe("lb_request_duration_ms", latency)

    if not success then
        metrics:inc("lb_errors_total", {server = server})
    end
end

metrics:report()
```

---

## 66.19 Failover และ Retry

```lua
-- ตัวอย่างที่ 28: Retry with Exponential Backoff
local RetryLB = {}
RetryLB.__index = RetryLB

function RetryLB.new(lb, maxRetries, baseDelay)
    local self        = setmetatable({}, RetryLB)
    self.lb           = lb
    self.maxRetries   = maxRetries or 3
    self.baseDelay    = baseDelay  or 0.1  -- seconds
    self.usedServers  = {}
    return self
end

function RetryLB:execute(requestFn)
    local lastError
    self.usedServers = {}

    for attempt = 1, self.maxRetries do
        local server = self.lb:next()
        if not server then
            return nil, "no available servers"
        end

        -- ป้องกันใช้เซิร์ฟเวอร์เดิมซ้ำ
        self.usedServers[server.host] = true

        local ok, result = pcall(requestFn, server)
        if ok then
            return result, nil
        else
            lastError = result
            print(string.format("[RETRY] Attempt %d/%d failed for %s: %s",
                attempt, self.maxRetries, server.host, lastError))

            if attempt < self.maxRetries then
                -- Exponential backoff (simulated)
                local delay = self.baseDelay * (2 ^ (attempt - 1))
                print(string.format("[RETRY] Waiting %.2fs before next attempt", delay))
                -- os.execute("sleep " .. delay)  -- ไม่ใช้ใน production
            end
        end
    end

    return nil, "max retries exceeded: " .. (lastError or "unknown")
end

-- Simple LB for retry test
local SimpleLB = {}
SimpleLB.__index = SimpleLB
function SimpleLB.new(servers)
    local self = setmetatable({}, SimpleLB)
    self.servers = servers
    self.idx = 0
    return self
end
function SimpleLB:next()
    self.idx = (self.idx % #self.servers) + 1
    return self.servers[self.idx]
end

local lb = SimpleLB.new({
    {host = "s1", port = 80},
    {host = "s2", port = 80},
    {host = "s3", port = 80},
})

local retryLB = RetryLB.new(lb, 3)

local callCount = 0
local result, err = retryLB:execute(function(server)
    callCount = callCount + 1
    if callCount < 3 then
        error(string.format("Connection timeout to %s", server.host))
    end
    return string.format("Success from %s", server.host)
end)

if result then
    print("Final result: " .. result)
else
    print("Failed: " .. err)
end
```

---

## 66.20 Load Balancer Configuration File

```lua
-- ตัวอย่างที่ 29: Config-driven Load Balancer
local ConfigLB = {}
ConfigLB.__index = ConfigLB

function ConfigLB.loadFromTable(config)
    local self = setmetatable({}, ConfigLB)
    self.name      = config.name or "default"
    self.algorithm = config.algorithm or "round_robin"
    self.timeout   = config.timeout or 30
    self.pools     = {}
    self.routes    = config.routes or {}

    for poolName, poolConfig in pairs(config.pools or {}) do
        self.pools[poolName] = {
            servers  = poolConfig.servers or {},
            lb       = nil,  -- LB instance per pool
        }
    end
    return self
end

function ConfigLB:routeRequest(path, method)
    for _, route in ipairs(self.routes) do
        if path:match(route.path) then
            if not route.method or route.method == method then
                return route.pool
            end
        end
    end
    return "default"
end

function ConfigLB:getServer(path, method)
    local poolName = self:routeRequest(path, method)
    local pool     = self.pools[poolName]
    if not pool or #pool.servers == 0 then
        return nil, "pool not found: " .. tostring(poolName)
    end
    -- Simple round-robin per pool
    pool._idx = ((pool._idx or 0) % #pool.servers) + 1
    return pool.servers[pool._idx], nil
end

local config = {
    name      = "api-gateway",
    algorithm = "round_robin",
    timeout   = 30,
    pools = {
        api = {
            servers = {
                {host = "api-1", port = 8080},
                {host = "api-2", port = 8080},
            }
        },
        static = {
            servers = {
                {host = "cdn-1", port = 80},
                {host = "cdn-2", port = 80},
            }
        },
        default = {
            servers = {
                {host = "web-1", port = 80},
            }
        },
    },
    routes = {
        {path = "^/api/",     pool = "api"},
        {path = "^/static/",  pool = "static"},
        {path = "^/",         pool = "default"},
    },
}

local lb = ConfigLB.loadFromTable(config)

local requests = {
    {path = "/api/users",    method = "GET"},
    {path = "/api/orders",   method = "POST"},
    {path = "/static/app.js", method = "GET"},
    {path = "/",             method = "GET"},
    {path = "/api/products", method = "GET"},
}

print("=== Config-driven LB routing ===")
for _, req in ipairs(requests) do
    local server, err = lb:getServer(req.path, req.method)
    if server then
        print(string.format("  %s %s -> %s:%d", req.method, req.path, server.host, server.port))
    else
        print(string.format("  %s %s -> ERROR: %s", req.method, req.path, err))
    end
end
```

---

## 66.21 Geolocation-based Routing

```lua
-- ตัวอย่างที่ 30: Geo-based Load Balancing
local GeoLB = {}
GeoLB.__index = GeoLB

function GeoLB.new()
    local self    = setmetatable({}, GeoLB)
    self.regions  = {}
    self.fallback = nil
    return self
end

function GeoLB:addRegion(name, config)
    self.regions[name] = {
        servers   = config.servers or {},
        countries = config.countries or {},
        _idx      = 0,
    }
end

function GeoLB:setFallback(regionName)
    self.fallback = regionName
end

function GeoLB:getRegion(country)
    for regionName, region in pairs(self.regions) do
        for _, c in ipairs(region.countries) do
            if c == country then return regionName end
        end
    end
    return self.fallback
end

function GeoLB:next(country)
    local regionName = self:getRegion(country)
    local region     = self.regions[regionName]
    if not region or #region.servers == 0 then return nil end
    region._idx = (region._idx % #region.servers) + 1
    return region.servers[region._idx], regionName
end

local geoLB = GeoLB.new()

geoLB:addRegion("ap-southeast", {
    countries = {"TH", "SG", "MY", "ID", "PH"},
    servers   = {
        {host = "sg-app-1", port = 443},
        {host = "sg-app-2", port = 443},
    }
})

geoLB:addRegion("us-east", {
    countries = {"US", "CA"},
    servers   = {
        {host = "us-app-1", port = 443},
        {host = "us-app-2", port = 443},
    }
})

geoLB:addRegion("eu-west", {
    countries = {"GB", "DE", "FR", "NL"},
    servers   = {
        {host = "eu-app-1", port = 443},
        {host = "eu-app-2", port = 443},
    }
})

geoLB:setFallback("us-east")

local visitors = {
    {country = "TH", ip = "1.2.3.4"},
    {country = "US", ip = "5.6.7.8"},
    {country = "DE", ip = "9.10.11.12"},
    {country = "AU", ip = "13.14.15.16"},  -- no region, fallback
}

print("Geo-based routing:")
for _, v in ipairs(visitors) do
    local s, region = geoLB:next(v.country)
    if s then
        print(string.format("  Country=%s, IP=%s -> [%s] %s",
            v.country, v.ip, region, s.host))
    end
end
```

---

## 66.22 สรุป Load Balancing Patterns

```lua
-- ตัวอย่างที่ 31: Strategy Pattern สำหรับ LB
local LBStrategy = {}

-- Factory สร้าง load balancer ตาม algorithm
function LBStrategy.create(algorithm, servers, options)
    options = options or {}
    if algorithm == "round_robin" then
        return RoundRobin.new(servers)
    elseif algorithm == "least_conn" then
        return LeastConnections.new(servers)
    elseif algorithm == "ip_hash" then
        return IPHash.new(servers)
    elseif algorithm == "random" then
        return RandomLB.new(servers)
    elseif algorithm == "weighted" then
        return WeightedRoundRobin.new(servers)
    elseif algorithm == "smooth_weighted" then
        return SmoothWRR.new(servers)
    else
        error("Unknown algorithm: " .. algorithm)
    end
end

-- Demo: เปรียบเทียบอัลกอริทึม
print("=== Algorithm Comparison ===")
local servers = {
    {host = "s1", port = 80, weight = 3},
    {host = "s2", port = 80, weight = 2},
    {host = "s3", port = 80, weight = 1},
}

local algos = {"round_robin", "weighted", "random"}
for _, algo in ipairs(algos) do
    local dist = {}
    -- Create simple test
    local pool
    if algo == "round_robin" then
        pool = RoundRobin.new(servers)
    elseif algo == "weighted" then
        pool = WeightedRoundRobin.new(servers)
    elseif algo == "random" then
        pool = RandomLB.new(servers)
    end

    for _ = 1, 60 do
        local s = pool:next()
        dist[s.host] = (dist[s.host] or 0) + 1
    end

    io.write(string.format("  [%s] ", algo))
    for _, s in ipairs(servers) do
        io.write(string.format("%s:%d ", s.host, dist[s.host] or 0))
    end
    print()
end
```

```lua
-- ตัวอย่างที่ 32: Load Balancer Benchmark
local function benchmark(name, lb, iterations)
    local t0 = os.clock()
    for i = 1, iterations do
        lb:next()
    end
    local t1    = os.clock()
    local elapsed = t1 - t0
    local rps   = iterations / elapsed
    print(string.format("  %s: %d ops in %.4fs (%.0f ops/sec)",
        name, iterations, elapsed, rps))
end

print("\n=== Load Balancer Benchmark ===")
local benchServers = {
    {host = "s1", port = 80, weight = 3},
    {host = "s2", port = 80, weight = 2},
    {host = "s3", port = 80, weight = 1},
}
local ITERS = 100000

benchmark("RoundRobin",       RoundRobin.new(benchServers),        ITERS)
benchmark("WeightedRR",       WeightedRoundRobin.new(benchServers), ITERS)
benchmark("SmoothWRR",        SmoothWRR.new(benchServers),          ITERS)
benchmark("Random",           RandomLB.new(benchServers),           ITERS)
```

```lua
-- ตัวอย่างที่ 33: Complete Health-aware Round Robin
local HealthAwareLB = {}
HealthAwareLB.__index = HealthAwareLB

function HealthAwareLB.new(servers)
    local self = setmetatable({}, HealthAwareLB)
    self.servers = {}
    for _, s in ipairs(servers) do
        table.insert(self.servers, {
            host         = s.host,
            port         = s.port,
            weight       = s.weight or 1,
            healthy      = true,
            failures     = 0,
            maxFailures  = s.maxFailures or 3,
            lastCheck    = os.time(),
            requests     = 0,
        })
    end
    self.current = 0
    return self
end

function HealthAwareLB:next()
    local tried = 0
    local n     = #self.servers
    while tried < n do
        self.current = (self.current % n) + 1
        local s = self.servers[self.current]
        if s.healthy then
            s.requests = s.requests + 1
            return s
        end
        tried = tried + 1
    end
    return nil
end

function HealthAwareLB:reportSuccess(host)
    for _, s in ipairs(self.servers) do
        if s.host == host then
            s.failures = math.max(0, s.failures - 1)
            if not s.healthy and s.failures == 0 then
                s.healthy = true
                print(string.format("[LB] %s recovered", host))
            end
            return
        end
    end
end

function HealthAwareLB:reportFailure(host)
    for _, s in ipairs(self.servers) do
        if s.host == host then
            s.failures = s.failures + 1
            if s.failures >= s.maxFailures and s.healthy then
                s.healthy = false
                print(string.format("[LB] %s marked unhealthy (failures: %d)",
                    host, s.failures))
            end
            return
        end
    end
end

local lb = HealthAwareLB.new({
    {host = "a", port = 80, maxFailures = 2},
    {host = "b", port = 80, maxFailures = 2},
    {host = "c", port = 80, maxFailures = 2},
})

print("=== Health-aware LB ===")
for i = 1, 9 do
    local s = lb:next()
    if s then
        -- Simulate server 'b' failing
        if s.host == "b" and i <= 4 then
            lb:reportFailure("b")
            print(string.format("Request %d -> %s [FAIL]", i, s.host))
        else
            lb:reportSuccess(s.host)
            print(string.format("Request %d -> %s [OK]", i, s.host))
        end
    end
end
```

```lua
-- ตัวอย่างที่ 34: LB with request queuing
local QueuedLB = {}
QueuedLB.__index = QueuedLB

function QueuedLB.new(servers, maxQueue)
    local self      = setmetatable({}, QueuedLB)
    self.servers    = servers
    self.maxQueue   = maxQueue or 100
    self.queue      = {}
    self.processing = 0
    self._idx       = 0
    return self
end

function QueuedLB:enqueue(request)
    if #self.queue >= self.maxQueue then
        return false, "queue full"
    end
    table.insert(self.queue, request)
    return true, nil
end

function QueuedLB:process()
    if #self.queue == 0 then return nil end
    local req = table.remove(self.queue, 1)
    self._idx = (self._idx % #self.servers) + 1
    local server = self.servers[self._idx]
    self.processing = self.processing + 1
    return {request = req, server = server}
end

function QueuedLB:complete()
    if self.processing > 0 then
        self.processing = self.processing - 1
    end
end

function QueuedLB:stats()
    return {
        queued     = #self.queue,
        processing = self.processing,
        capacity   = self.maxQueue,
    }
end

local qlb = QueuedLB.new(
    {{host = "w1"}, {host = "w2"}},
    5
)

-- Enqueue requests
for i = 1, 7 do
    local ok, err = qlb:enqueue({id = i, path = "/api/" .. i})
    if ok then
        print(string.format("Enqueued request %d", i))
    else
        print(string.format("Request %d rejected: %s", i, err))
    end
end

-- Process
print("\nProcessing:")
while true do
    local job = qlb:process()
    if not job then break end
    print(string.format("  Processing req %d on %s",
        job.request.id, job.server.host))
    qlb:complete()
end

local s = qlb:stats()
print(string.format("\nFinal stats: queued=%d, processing=%d",
    s.queued, s.processing))
```

```lua
-- ตัวอย่างที่ 35: Nginx upstream_hash สำหรับ consistent routing
-- ใช้ใน OpenResty environment
--[[
-- nginx.conf:
upstream backend {
    hash $request_uri consistent;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
    keepalive 32;
}
--]]

-- Simulate upstream_hash behavior in pure Lua
local function nginxHashLB(servers, requestUri)
    local function crc32(s)
        local crc = 0xFFFFFFFF
        for i = 1, #s do
            crc = crc ~ s:byte(i)
            for _ = 1, 8 do
                if crc & 1 == 1 then
                    crc = (crc >> 1) ~ 0xEDB88320
                else
                    crc = crc >> 1
                end
            end
        end
        return crc ~ 0xFFFFFFFF
    end

    local hash = crc32(requestUri) % #servers
    return servers[hash + 1]
end

local servers = {
    {host = "10.0.0.1", port = 8080},
    {host = "10.0.0.2", port = 8080},
    {host = "10.0.0.3", port = 8080},
}

local uris = {
    "/api/user/1001",
    "/api/user/1002",
    "/api/product/5555",
    "/api/user/1001",  -- same as first
    "/api/order/9999",
}

print("Nginx-style hash routing:")
for _, uri in ipairs(uris) do
    local s = nginxHashLB(servers, uri)
    print(string.format("  %s -> %s", uri, s.host))
end
```

---

## สรุปบทที่ 66

ในบทนี้เราได้เรียนรู้:

1. **Round Robin** - กระจาย request แบบ sequential
2. **Weighted Round Robin** - กระจายตามน้ำหนัก (Smooth WRR แบบ Nginx)
3. **Least Connections** - เลือกเซิร์ฟเวอร์ที่ว่างที่สุด
4. **IP Hash** - Client เดิมไปเซิร์ฟเวอร์เดิม
5. **Random / Power of Two** - สุ่มเลือกที่ดีกว่า
6. **Consistent Hashing** - สำหรับ stateful services
7. **Health Check** - ตรวจสอบสถานะเซิร์ฟเวอร์
8. **Sticky Sessions** - Cookie-based session persistence
9. **Connection Draining** - ถอดเซิร์ฟเวอร์อย่างนุ่มนวล
10. **Blue-Green Deployment** - Zero-downtime deployment
11. **Canary Releases** - Gradual rollout
12. **A/B Testing** - User-based routing
13. **Circuit Breaker** - ป้องกัน cascade failures
14. **Retry with Backoff** - Fault tolerance
15. **Geo-based Routing** - Route ตาม geography

---

*จบบทที่ 66 - Load Balancing*
