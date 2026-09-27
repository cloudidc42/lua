# บทที่ 64: Rate Limiting

## บทนำ

Rate Limiting คือเทคนิคในการควบคุมจำนวน requests ที่ client สามารถส่งได้ในช่วงเวลาหนึ่ง เป็นการป้องกัน:
- DDoS attacks
- API abuse
- Resource exhaustion
- Ensuring fair usage

บทนี้จะครอบคลุม algorithms หลักและการ implement ใน Lua โดยเฉพาะใน OpenResty/NGINX

## 1. Token Bucket Algorithm

Token Bucket เป็น algorithm ที่นิยมมากที่สุด:
- Bucket มี capacity สูงสุด N tokens
- เติม tokens ด้วย rate R tokens/second
- แต่ละ request ใช้ 1 token (หรือมากกว่า)
- ถ้าไม่มี token -> reject หรือ queue

```lua
-- Example 1: Token Bucket implementation
local TokenBucket = {}
TokenBucket.__index = TokenBucket

function TokenBucket.new(capacity, refillRate)
    return setmetatable({
        capacity = capacity,      -- จำนวน token สูงสุด
        tokens = capacity,        -- token ปัจจุบัน
        refillRate = refillRate,  -- token/second
        lastRefill = os.clock(),
        totalAllowed = 0,
        totalDenied = 0
    }, TokenBucket)
end

function TokenBucket:refill()
    local now = os.clock()
    local elapsed = now - self.lastRefill
    local newTokens = elapsed * self.refillRate
    
    self.tokens = math.min(self.capacity, self.tokens + newTokens)
    self.lastRefill = now
end

function TokenBucket:consume(tokens)
    tokens = tokens or 1
    self:refill()
    
    if self.tokens >= tokens then
        self.tokens = self.tokens - tokens
        self.totalAllowed = self.totalAllowed + 1
        return true
    else
        self.totalDenied = self.totalDenied + 1
        return false
    end
end

function TokenBucket:getTokens()
    self:refill()
    return self.tokens
end

function TokenBucket:waitTime(tokens)
    tokens = tokens or 1
    self:refill()
    if self.tokens >= tokens then return 0 end
    return (tokens - self.tokens) / self.refillRate
end

function TokenBucket:stats()
    return {
        tokens = self.tokens,
        capacity = self.capacity,
        refillRate = self.refillRate,
        allowed = self.totalAllowed,
        denied = self.totalDenied,
        utilizationRate = self.totalAllowed > 0 and
            self.totalAllowed / (self.totalAllowed + self.totalDenied) or 0
    }
end

-- การใช้งาน
local bucket = TokenBucket.new(10, 2)  -- 10 tokens, refill 2/sec

print("=== Token Bucket Demo ===")
print(string.format("Capacity: %d, Refill rate: %d/sec", bucket.capacity, bucket.refillRate))
print()

-- Simulate burst
for i = 1, 15 do
    local allowed = bucket:consume(1)
    local tokens = bucket:getTokens()
    print(string.format("Request %2d: %s (tokens remaining: %.2f)",
        i,
        allowed and "ALLOWED" or "DENIED",
        tokens
    ))
end

print()
print("Stats:")
local stats = bucket:stats()
print(string.format("  Allowed: %d, Denied: %d (%.0f%% pass rate)",
    stats.allowed, stats.denied, stats.utilizationRate * 100))
```

## 2. Token Bucket สำหรับหลาย Clients

```lua
-- Example 2: Per-client token buckets
local RateLimiter = {}
RateLimiter.__index = RateLimiter

function RateLimiter.new(defaultConfig)
    return setmetatable({
        buckets = {},
        defaultConfig = defaultConfig or {capacity = 100, refillRate = 10},
        configs = {},   -- client-specific configs
        cleanupInterval = 300,  -- cleanup every 5 minutes
        lastCleanup = os.time()
    }, RateLimiter)
end

function RateLimiter:setConfig(key, capacity, refillRate)
    self.configs[key] = {capacity = capacity, refillRate = refillRate}
end

function RateLimiter:getBucket(clientId)
    if not self.buckets[clientId] then
        local config = self.configs[clientId] or self.defaultConfig
        self.buckets[clientId] = {
            tokens = config.capacity,
            capacity = config.capacity,
            refillRate = config.refillRate,
            lastRefill = os.clock(),
            lastAccess = os.time()
        }
    end
    return self.buckets[clientId]
end

function RateLimiter:check(clientId, tokens)
    tokens = tokens or 1
    local bucket = self:getBucket(clientId)
    bucket.lastAccess = os.time()
    
    -- Refill
    local now = os.clock()
    local elapsed = now - bucket.lastRefill
    local newTokens = elapsed * bucket.refillRate
    bucket.tokens = math.min(bucket.capacity, bucket.tokens + newTokens)
    bucket.lastRefill = now
    
    if bucket.tokens >= tokens then
        bucket.tokens = bucket.tokens - tokens
        return true, {
            remaining = math.floor(bucket.tokens),
            limit = bucket.capacity,
            reset = math.ceil((bucket.capacity - bucket.tokens) / bucket.refillRate)
        }
    else
        local retryAfter = math.ceil((tokens - bucket.tokens) / bucket.refillRate)
        return false, {
            remaining = 0,
            limit = bucket.capacity,
            retryAfter = retryAfter,
            reset = retryAfter
        }
    end
end

function RateLimiter:cleanup(maxAge)
    maxAge = maxAge or 3600  -- 1 hour
    local now = os.time()
    local removed = 0
    
    for clientId, bucket in pairs(self.buckets) do
        if now - bucket.lastAccess > maxAge then
            self.buckets[clientId] = nil
            removed = removed + 1
        end
    end
    
    self.lastCleanup = now
    return removed
end

-- ตัวอย่าง: API rate limiter
local apiLimiter = RateLimiter.new({capacity = 100, refillRate = 10})

-- Premium users get higher limits
apiLimiter:setConfig("user_premium_001", 1000, 100)
apiLimiter:setConfig("user_premium_002", 500, 50)

-- Test different clients
local clients = {
    "user_123",
    "user_456",
    "user_premium_001",
    "user_123",  -- same client again
    "user_123",
}

print("\n=== Per-Client Rate Limiting ===")
for _, clientId in ipairs(clients) do
    local allowed, info = apiLimiter:check(clientId)
    print(string.format("Client %-20s: %s (remaining: %d, limit: %d)",
        clientId,
        allowed and "ALLOWED" or "DENIED",
        info.remaining,
        info.limit
    ))
end

-- Drain one client
print("\nDraining user_123...")
local denied = 0
for i = 1, 120 do
    local allowed, info = apiLimiter:check("user_123")
    if not allowed then
        denied = denied + 1
        if denied == 1 then
            print(string.format("First denial at request %d, retry after %ds", i, info.retryAfter))
        end
    end
end
print("Total denied: " .. denied)
```

## 3. Leaky Bucket Algorithm

```lua
-- Example 3: Leaky Bucket
-- Requests เข้า bucket -> leak ออกด้วย rate คงที่
-- ต่างจาก Token Bucket ตรงที่ request rate output คงที่ (smoothing)

local LeakyBucket = {}
LeakyBucket.__index = LeakyBucket

function LeakyBucket.new(capacity, leakRate)
    return setmetatable({
        capacity = capacity,    -- max pending requests
        queue = {},             -- pending requests
        leakRate = leakRate,    -- requests/second
        lastLeak = os.clock(),
        totalQueued = 0,
        totalLeaked = 0,
        totalDropped = 0
    }, LeakyBucket)
end

function LeakyBucket:_leak()
    local now = os.clock()
    local elapsed = now - self.lastLeak
    local toLeak = math.floor(elapsed * self.leakRate)
    
    if toLeak > 0 then
        for i = 1, math.min(toLeak, #self.queue) do
            local req = table.remove(self.queue, 1)
            if req and req.callback then
                req.callback(req)
            end
            self.totalLeaked = self.totalLeaked + 1
        end
        self.lastLeak = now
    end
end

function LeakyBucket:add(request)
    self:_leak()
    
    if #self.queue < self.capacity then
        table.insert(self.queue, request)
        self.totalQueued = self.totalQueued + 1
        return true
    else
        self.totalDropped = self.totalDropped + 1
        return false
    end
end

function LeakyBucket:update(dt)
    self:_leak()
end

function LeakyBucket:queueSize()
    self:_leak()
    return #self.queue
end

function LeakyBucket:stats()
    return {
        queueSize = #self.queue,
        capacity = self.capacity,
        queued = self.totalQueued,
        leaked = self.totalLeaked,
        dropped = self.totalDropped
    }
end

-- การใช้งาน
local leaky = LeakyBucket.new(5, 2)  -- capacity 5, leak 2/sec

print("\n=== Leaky Bucket Demo ===")
local processed = {}

-- Burst of requests
for i = 1, 10 do
    local added = leaky:add({
        id = i,
        data = "request " .. i,
        callback = function(req)
            table.insert(processed, req.id)
        end
    })
    print(string.format("  Add request %2d: %s (queue: %d/%d)",
        i,
        added and "QUEUED" or "DROPPED",
        leaky:queueSize(),
        leaky.capacity
    ))
end

-- Simulate time passing (leak)
print("\nSimulating 3 seconds...")
-- In real code: wait and call update(dt)
-- Here we manually leak
for i = 1, 6 do  -- 6 leaks = 3 seconds * 2/sec
    leaky:_leak()
end

local s = leaky:stats()
print(string.format("Stats: queued=%d, leaked=%d, dropped=%d",
    s.queued, s.leaked, s.dropped))
print("Processed IDs:", table.concat(processed, ", "))
```

## 4. Fixed Window Counter

```lua
-- Example 4: Fixed Window Counter
-- แบ่งเวลาเป็น window คงที่ (เช่น ทุก 1 นาที)
-- นับ requests ใน window ปัจจุบัน

local FixedWindowLimiter = {}
FixedWindowLimiter.__index = FixedWindowLimiter

function FixedWindowLimiter.new(limit, windowSize)
    return setmetatable({
        limit = limit,          -- requests per window
        windowSize = windowSize, -- window size in seconds
        counters = {},          -- clientId -> {count, windowStart}
        totalRequests = 0,
        totalAllowed = 0,
        totalDenied = 0
    }, FixedWindowLimiter)
end

function FixedWindowLimiter:getWindowKey(clientId)
    local now = os.time()
    local windowStart = math.floor(now / self.windowSize) * self.windowSize
    return clientId, windowStart
end

function FixedWindowLimiter:check(clientId, cost)
    cost = cost or 1
    local now = os.time()
    local windowStart = math.floor(now / self.windowSize) * self.windowSize
    
    self.totalRequests = self.totalRequests + 1
    
    if not self.counters[clientId] then
        self.counters[clientId] = {count = 0, windowStart = windowStart}
    end
    
    local counter = self.counters[clientId]
    
    -- Reset if new window
    if counter.windowStart < windowStart then
        counter.count = 0
        counter.windowStart = windowStart
    end
    
    local remaining = self.limit - counter.count
    local resetIn = (windowStart + self.windowSize) - now
    
    if counter.count + cost <= self.limit then
        counter.count = counter.count + cost
        self.totalAllowed = self.totalAllowed + 1
        return true, {
            limit = self.limit,
            remaining = self.limit - counter.count,
            reset = resetIn,
            windowStart = windowStart
        }
    else
        self.totalDenied = self.totalDenied + 1
        return false, {
            limit = self.limit,
            remaining = 0,
            reset = resetIn,
            retryAfter = resetIn
        }
    end
end

function FixedWindowLimiter:reset(clientId)
    if clientId then
        self.counters[clientId] = nil
    else
        self.counters = {}
    end
end

function FixedWindowLimiter:cleanup()
    local now = os.time()
    local currentWindow = math.floor(now / self.windowSize) * self.windowSize
    local removed = 0
    
    for clientId, counter in pairs(self.counters) do
        if counter.windowStart < currentWindow - self.windowSize then
            self.counters[clientId] = nil
            removed = removed + 1
        end
    end
    return removed
end

-- ตัวอย่าง
local fwLimiter = FixedWindowLimiter.new(5, 60)  -- 5 requests per minute

print("\n=== Fixed Window Counter ===")
for i = 1, 8 do
    local allowed, info = fwLimiter:check("client_1")
    print(string.format("Request %d: %s (remaining: %d, reset in: %ds)",
        i,
        allowed and "ALLOWED" or "DENIED",
        info.remaining,
        info.reset
    ))
end

-- Problem: boundary burst
-- ช่วงท้าย window + ช่วงต้น window ถัดไป = 2x limit
print("\nFixed Window ข้อเสีย: boundary burst")
print("สามารถส่ง", fwLimiter.limit, "requests ก่อนสิ้น window")
print("และ", fwLimiter.limit, "requests อีกครั้งทันทีที่ขึ้น window ใหม่")
print("รวม", fwLimiter.limit * 2, "requests ใน", fwLimiter.windowSize, "seconds")
```

## 5. Sliding Window Log

```lua
-- Example 5: Sliding Window Log
-- เก็บ timestamp ของทุก request
-- ตรวจสอบว่าจำนวน requests ใน window ที่เลื่อนตามเวลาไม่เกิน limit

local SlidingWindowLog = {}
SlidingWindowLog.__index = SlidingWindowLog

function SlidingWindowLog.new(limit, windowSize)
    return setmetatable({
        limit = limit,
        windowSize = windowSize,
        logs = {},  -- clientId -> list of timestamps
        maxMemory = 10000  -- max total entries
    }, SlidingWindowLog)
end

function SlidingWindowLog:_cleanup(clientId)
    local now = os.time()
    local cutoff = now - self.windowSize
    local log = self.logs[clientId]
    
    if not log then return end
    
    -- Remove old entries
    local newLog = {}
    for _, timestamp in ipairs(log) do
        if timestamp > cutoff then
            table.insert(newLog, timestamp)
        end
    end
    self.logs[clientId] = newLog
end

function SlidingWindowLog:check(clientId)
    local now = os.time()
    
    if not self.logs[clientId] then
        self.logs[clientId] = {}
    end
    
    -- Remove expired entries
    self:_cleanup(clientId)
    
    local log = self.logs[clientId]
    local count = #log
    
    if count < self.limit then
        table.insert(log, now)
        return true, {
            limit = self.limit,
            remaining = self.limit - count - 1,
            reset = (count > 0) and (self.windowSize - (now - log[1])) or self.windowSize
        }
    else
        -- Calculate when oldest request expires
        local retryAfter = self.windowSize - (now - log[1])
        return false, {
            limit = self.limit,
            remaining = 0,
            retryAfter = math.ceil(retryAfter),
            reset = math.ceil(retryAfter)
        }
    end
end

function SlidingWindowLog:getCount(clientId)
    self:_cleanup(clientId)
    return #(self.logs[clientId] or {})
end

-- ข้อเสีย: memory usage สูง เพราะต้องเก็บทุก timestamp
-- ดีกว่า Fixed Window เรื่อง accuracy

local swLog = SlidingWindowLog.new(5, 60)

print("\n=== Sliding Window Log ===")
for i = 1, 8 do
    local allowed, info = swLog:check("client_1")
    print(string.format("Request %d: %s (remaining: %d, count in window: %d)",
        i,
        allowed and "ALLOWED" or "DENIED",
        info.remaining,
        swLog:getCount("client_1")
    ))
end
```

## 6. Sliding Window Counter

```lua
-- Example 6: Sliding Window Counter (memory efficient)
-- ผสมระหว่าง Fixed Window และ Sliding Window Log
-- ใช้ 2 windows แทนการเก็บทุก timestamp

local SlidingWindowCounter = {}
SlidingWindowCounter.__index = SlidingWindowCounter

function SlidingWindowCounter.new(limit, windowSize)
    return setmetatable({
        limit = limit,
        windowSize = windowSize,
        data = {}  -- clientId -> {current, previous, windowStart, prevWindowStart}
    }, SlidingWindowCounter)
end

function SlidingWindowCounter:check(clientId)
    local now = os.time()
    local windowStart = math.floor(now / self.windowSize) * self.windowSize
    
    if not self.data[clientId] then
        self.data[clientId] = {
            current = 0,
            previous = 0,
            windowStart = windowStart,
            prevWindowStart = windowStart - self.windowSize
        }
    end
    
    local d = self.data[clientId]
    
    -- Shift windows if needed
    if windowStart > d.windowStart then
        if windowStart == d.windowStart + self.windowSize then
            -- Advance one window
            d.previous = d.current
            d.prevWindowStart = d.windowStart
        else
            -- Skip multiple windows
            d.previous = 0
        end
        d.current = 0
        d.windowStart = windowStart
    end
    
    -- Calculate weighted count
    -- ส่วนที่เหลือของ window ก่อนหน้า (linear interpolation)
    local elapsed = now - windowStart
    local prevWeight = 1 - (elapsed / self.windowSize)
    local weightedCount = d.current + math.floor(d.previous * prevWeight)
    
    local remaining = self.limit - weightedCount
    
    if weightedCount < self.limit then
        d.current = d.current + 1
        return true, {
            limit = self.limit,
            remaining = remaining - 1,
            weighted = weightedCount,
            reset = self.windowSize - elapsed
        }
    else
        return false, {
            limit = self.limit,
            remaining = 0,
            retryAfter = math.ceil(1 / (d.previous / self.windowSize)),
            weighted = weightedCount
        }
    end
end

-- ตัวอย่าง
local swCounter = SlidingWindowCounter.new(10, 60)

print("\n=== Sliding Window Counter ===")
-- Simulate requests across time
local requests = {10, 8, 5, 3}  -- requests at different times
for window, count in ipairs(requests) do
    print(string.format("\nWindow %d:", window))
    for i = 1, count do
        local allowed, info = swCounter:check("client_1")
        if i <= 3 or not allowed then
            print(string.format("  Req %d: %s (weighted: %s, remaining: %d)",
                i,
                allowed and "ALLOWED" or "DENIED",
                tostring(info.weighted),
                info.remaining
            ))
        end
    end
end
```

## 7. Rate Limiting ใน OpenResty

```lua
-- Example 7: Rate limiting in OpenResty/NGINX
-- ใช้ ngx.shared.DICT สำหรับ shared memory

-- nginx.conf:
-- http {
--     lua_shared_dict rate_limit 10m;
--     lua_shared_dict rate_limit_counters 50m;
-- }

local RateLimitOpenResty = {}

-- Token Bucket ใน shared dict
function RateLimitOpenResty.tokenBucket(key, capacity, refillRate)
    local dict = ngx.shared.rate_limit
    
    local now = ngx.now()
    
    -- ใช้ Redis-like operations ด้วย dict
    local tokens_key = key .. ":tokens"
    local time_key = key .. ":time"
    
    -- Get current state
    local tokens = dict:get(tokens_key)
    local lastTime = dict:get(time_key)
    
    if not tokens or not lastTime then
        -- Initialize
        tokens = capacity
        lastTime = now
    end
    
    -- Refill tokens
    local elapsed = now - lastTime
    tokens = math.min(capacity, tokens + elapsed * refillRate)
    
    if tokens >= 1 then
        -- Allow request
        tokens = tokens - 1
        dict:set(tokens_key, tokens, 3600)  -- TTL 1 hour
        dict:set(time_key, now, 3600)
        
        return true, {
            remaining = math.floor(tokens),
            limit = capacity
        }
    else
        dict:set(time_key, now, 3600)
        dict:set(tokens_key, tokens, 3600)
        
        local retryAfter = math.ceil((1 - tokens) / refillRate)
        return false, {
            remaining = 0,
            limit = capacity,
            retryAfter = retryAfter
        }
    end
end

-- Sliding window counter ใน OpenResty
function RateLimitOpenResty.slidingWindow(key, limit, windowSize)
    local dict = ngx.shared.rate_limit_counters
    local now = ngx.now()
    
    local windowStart = math.floor(now / windowSize) * windowSize
    local prevWindowStart = windowStart - windowSize
    
    local currKey = key .. ":" .. windowStart
    local prevKey = key .. ":" .. prevWindowStart
    
    local currCount = dict:get(currKey) or 0
    local prevCount = dict:get(prevKey) or 0
    
    -- Weighted count
    local elapsed = now - windowStart
    local prevWeight = 1 - (elapsed / windowSize)
    local weightedCount = currCount + math.floor(prevCount * prevWeight)
    
    if weightedCount < limit then
        dict:incr(currKey, 1, 0, windowSize + 1)
        
        return true, {
            limit = limit,
            remaining = limit - weightedCount - 1,
            reset = math.ceil(windowSize - elapsed)
        }
    else
        return false, {
            limit = limit,
            remaining = 0,
            retryAfter = math.ceil(1 / (prevCount / windowSize or 0.001)),
            reset = math.ceil(windowSize - elapsed)
        }
    end
end

-- OpenResty middleware handler
function RateLimitOpenResty.handler(config)
    -- config = {
    --   keyFunc = function() ... end,  -- returns client key
    --   limit = 100,
    --   window = 60,
    --   algorithm = "sliding_window"  -- or "token_bucket"
    -- }
    
    local key = config.keyFunc and config.keyFunc() or
                ngx.var.remote_addr
    
    local allowed, info
    
    if config.algorithm == "token_bucket" then
        allowed, info = RateLimitOpenResty.tokenBucket(
            key,
            config.limit,
            config.limit / config.window
        )
    else
        allowed, info = RateLimitOpenResty.slidingWindow(
            key,
            config.limit,
            config.window
        )
    end
    
    -- Set rate limit headers
    ngx.header["X-RateLimit-Limit"] = config.limit
    ngx.header["X-RateLimit-Remaining"] = info.remaining
    ngx.header["X-RateLimit-Reset"] = (ngx.time() + (info.reset or 0))
    
    if not allowed then
        ngx.header["Retry-After"] = info.retryAfter or config.window
        ngx.header["X-RateLimit-Remaining"] = 0
        ngx.status = 429
        ngx.header["Content-Type"] = "application/json"
        ngx.say('{"error":"Too Many Requests","retryAfter":' .. (info.retryAfter or 0) .. '}')
        ngx.exit(429)
    end
end

-- จำลองสำหรับ non-OpenResty
print("\n=== OpenResty Rate Limit Simulation ===")
print("ngx.shared.rate_limit = {} (simulated)")

-- Mock ngx
local ngx_mock = {
    now = function() return os.time() end,
    time = function() return os.time() end,
    var = {remote_addr = "192.168.1.1"},
    header = {},
    status = 200,
    shared = {
        rate_limit = {
            data = {},
            get = function(self, k) return self.data[k] end,
            set = function(self, k, v, ttl) self.data[k] = v end,
        },
        rate_limit_counters = {
            data = {},
            counters = {},
            get = function(self, k) return self.data[k] end,
            set = function(self, k, v, ttl) self.data[k] = v end,
            incr = function(self, k, n, init, ttl)
                self.data[k] = (self.data[k] or init) + n
                return self.data[k]
            end,
        }
    },
    exit = function(code) end,
    say = function(msg) print("Response:", msg) end
}

print("OpenResty rate limiting configured (see code for production use)")
```

## 8. Distributed Rate Limiting ด้วย Redis

```lua
-- Example 8: Distributed rate limiting with Redis
-- ใช้ Redis เพราะ shared memory ระหว่าง multiple instances

-- Redis Lua script สำหรับ atomic token bucket
local TOKEN_BUCKET_SCRIPT = [[
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refillRate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4]) or 1

-- Get current state
local data = redis.call("HMGET", key, "tokens", "lastRefill")
local tokens = tonumber(data[1]) or capacity
local lastRefill = tonumber(data[2]) or now

-- Refill tokens
local elapsed = now - lastRefill
local newTokens = elapsed * refillRate
tokens = math.min(capacity, tokens + newTokens)

local allowed = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
end

-- Save state with TTL
redis.call("HMSET", key, "tokens", tokens, "lastRefill", now)
redis.call("EXPIRE", key, math.ceil(capacity / refillRate) + 10)

return {allowed, math.floor(tokens), math.ceil((capacity - tokens) / refillRate)}
]]

-- Redis Sliding Window script
local SLIDING_WINDOW_SCRIPT = [[
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local windowSize = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Remove expired entries
redis.call("ZREMRANGEBYSCORE", key, "-inf", now - windowSize)

-- Count current requests
local count = redis.call("ZCARD", key)

if count < limit then
    -- Add this request
    redis.call("ZADD", key, now, now .. math.random())
    redis.call("EXPIRE", key, windowSize + 1)
    return {1, limit - count - 1, 0}
else
    -- Get oldest timestamp to calculate retry-after
    local oldest = redis.call("ZRANGE", key, 0, 0, "WITHSCORES")
    local retryAfter = 0
    if oldest and oldest[2] then
        retryAfter = math.ceil(windowSize - (now - tonumber(oldest[2])))
    end
    return {0, 0, retryAfter}
end
]]

-- Lua Redis client wrapper (simplified)
local RedisRateLimiter = {}
RedisRateLimiter.__index = RedisRateLimiter

function RedisRateLimiter.new(redisClient)
    return setmetatable({
        redis = redisClient,
        scripts = {
            tokenBucket = nil,  -- SHA after SCRIPT LOAD
            slidingWindow = nil
        }
    }, RedisRateLimiter)
end

function RedisRateLimiter:tokenBucket(key, capacity, refillRate, requested)
    -- ใน production: ใช้ EVALSHA (cached script)
    -- result = self.redis:evalsha(sha, 1, key, capacity, refillRate, now, requested)
    
    -- Simplified simulation
    local now = os.time()
    requested = requested or 1
    
    print(string.format("[Redis TB] EVALSHA token_bucket %s cap=%d rate=%d req=%d",
        key, capacity, refillRate, requested))
    
    -- Simulate result
    local tokens = capacity - 3  -- mock remaining tokens
    local allowed = tokens >= requested
    
    return {
        allowed = allowed,
        remaining = allowed and tokens - requested or 0,
        retryAfter = allowed and 0 or math.ceil((requested - tokens) / refillRate)
    }
end

function RedisRateLimiter:slidingWindowCheck(key, limit, windowSize)
    local now = os.time()
    
    print(string.format("[Redis SW] EVALSHA sliding_window %s limit=%d window=%d",
        key, limit, windowSize))
    
    -- Simulate
    return {allowed = true, remaining = limit - 1, retryAfter = 0}
end

function RedisRateLimiter:checkMultipleKeys(checks)
    -- Pipeline multiple checks (efficient)
    local results = {}
    
    -- ใน production: ใช้ pipeline
    -- local pipe = redis.pipeline()
    -- for _, check in ipairs(checks) do
    --     pipe:evalsha(sha, ...)
    -- end
    -- results = pipe:execute()
    
    for i, check in ipairs(checks) do
        results[i] = {allowed = true, remaining = 99}
    end
    
    return results
end

-- Multi-dimensional rate limiting
local function checkMultiDimensional(limiter, request)
    local checks = {
        -- Per-IP limit
        {key = "ip:" .. request.ip, limit = 1000, window = 60},
        -- Per-user limit
        {key = "user:" .. request.userId, limit = 500, window = 60},
        -- Per-API-key limit
        {key = "apikey:" .. request.apiKey, limit = 10000, window = 3600},
        -- Per-endpoint limit
        {key = "endpoint:" .. request.endpoint, limit = 100, window = 10},
    }
    
    local results = limiter:checkMultipleKeys(checks)
    
    -- ต้องผ่านทุก check
    local lowestRemaining = math.huge
    for i, result in ipairs(results) do
        if not result.allowed then
            return false, {
                dimension = checks[i].key,
                retryAfter = result.retryAfter
            }
        end
        lowestRemaining = math.min(lowestRemaining, result.remaining)
    end
    
    return true, {remaining = lowestRemaining}
end

local mockRedis = {}
local redisLimiter = RedisRateLimiter.new(mockRedis)

print("\n=== Distributed Rate Limiting (Redis) ===")
local request = {
    ip = "203.0.113.42",
    userId = "user_789",
    apiKey = "ak_live_abc123",
    endpoint = "/api/search"
}

local allowed, info = checkMultiDimensional(redisLimiter, request)
print("Request allowed:", allowed)
if not allowed then
    print("Denied by:", info.dimension)
else
    print("Min remaining:", info.remaining)
end
```

## 9. Per-User, Per-IP, Per-API-Key Limits

```lua
-- Example 9: Tiered rate limiting by identity
local TieredRateLimiter = {}
TieredRateLimiter.__index = TieredRateLimiter

-- Tier definitions
local TIERS = {
    anonymous = {
        requestsPerMinute = 20,
        requestsPerHour = 200,
        requestsPerDay = 1000,
        burstSize = 5
    },
    free = {
        requestsPerMinute = 60,
        requestsPerHour = 1000,
        requestsPerDay = 10000,
        burstSize = 10
    },
    pro = {
        requestsPerMinute = 300,
        requestsPerHour = 10000,
        requestsPerDay = 100000,
        burstSize = 50
    },
    enterprise = {
        requestsPerMinute = 3000,
        requestsPerHour = 100000,
        requestsPerDay = 1000000,
        burstSize = 500
    },
    internal = {
        requestsPerMinute = math.huge,
        requestsPerHour = math.huge,
        requestsPerDay = math.huge,
        burstSize = math.huge
    }
}

function TieredRateLimiter.new()
    return setmetatable({
        userTiers = {},     -- userId -> tier
        ipLimits = {},      -- ip -> {tier, created}
        apiKeyTiers = {},   -- apiKey -> tier
        counters = {},      -- key -> {count, windowStart}
        whitelist = {},     -- whitelisted IPs/users
        blacklist = {},     -- blacklisted IPs/users
    }, TieredRateLimiter)
end

function TieredRateLimiter:setUserTier(userId, tier)
    self.userTiers[userId] = tier
end

function TieredRateLimiter:setApiKeyTier(apiKey, tier)
    self.apiKeyTiers[apiKey] = tier
end

function TieredRateLimiter:whitelist(identifier)
    self.whitelist[identifier] = true
end

function TieredRateLimiter:blacklist(identifier)
    self.blacklist[identifier] = true
end

function TieredRateLimiter:_getCounter(key, windowSize)
    local now = os.time()
    local windowStart = math.floor(now / windowSize) * windowSize
    
    if not self.counters[key] or self.counters[key].windowStart < windowStart then
        self.counters[key] = {count = 0, windowStart = windowStart}
    end
    
    return self.counters[key]
end

function TieredRateLimiter:_checkLimit(identifier, key, limit, windowSize)
    local counter = self:_getCounter(key, windowSize)
    
    if counter.count < limit then
        counter.count = counter.count + 1
        return true, limit - counter.count
    else
        return false, 0
    end
end

function TieredRateLimiter:check(request)
    -- Identify client
    local identifier = request.userId or request.apiKey or request.ip
    
    -- Blacklist check
    if self.blacklist[identifier] or self.blacklist[request.ip] then
        return false, {
            reason = "blacklisted",
            status = 403
        }
    end
    
    -- Whitelist bypass
    if self.whitelist[identifier] or self.whitelist[request.ip] then
        return true, {reason = "whitelisted", remaining = math.huge}
    end
    
    -- Determine tier
    local tier
    if request.apiKey then
        tier = TIERS[self.apiKeyTiers[request.apiKey] or "free"]
    elseif request.userId then
        tier = TIERS[self.userTiers[request.userId] or "free"]
    else
        tier = TIERS.anonymous
    end
    
    if not tier then
        return false, {reason = "invalid tier"}
    end
    
    -- Check minute limit
    local minKey = "min:" .. identifier
    local minOk, minRemaining = self:_checkLimit(
        identifier, minKey,
        tier.requestsPerMinute, 60
    )
    if not minOk then
        return false, {
            reason = "minute_limit_exceeded",
            limit = tier.requestsPerMinute,
            window = "1m",
            retryAfter = 60 - (os.time() % 60),
            status = 429
        }
    end
    
    -- Check hour limit
    local hrKey = "hr:" .. identifier
    local hrOk, hrRemaining = self:_checkLimit(
        identifier, hrKey,
        tier.requestsPerHour, 3600
    )
    if not hrOk then
        return false, {
            reason = "hourly_limit_exceeded",
            limit = tier.requestsPerHour,
            window = "1h",
            retryAfter = 3600 - (os.time() % 3600),
            status = 429
        }
    end
    
    -- Check daily limit
    local dayKey = "day:" .. identifier
    local dayOk, dayRemaining = self:_checkLimit(
        identifier, dayKey,
        tier.requestsPerDay, 86400
    )
    if not dayOk then
        return false, {
            reason = "daily_limit_exceeded",
            limit = tier.requestsPerDay,
            window = "1d",
            status = 429
        }
    end
    
    return true, {
        remaining = math.min(minRemaining, hrRemaining, dayRemaining),
        tier = tier
    }
end

-- การใช้งาน
local limiter = TieredRateLimiter.new()

-- Setup tiers
limiter:setUserTier("user_001", "pro")
limiter:setUserTier("user_002", "free")
limiter:setApiKeyTier("ak_enterprise_xyz", "enterprise")
limiter:whitelist("10.0.0.1")  -- internal load balancer
limiter:blacklist("1.2.3.4")   -- known bad actor

print("\n=== Tiered Rate Limiting ===")

local testRequests = {
    {ip = "203.0.113.1", userId = "user_001"},
    {ip = "203.0.113.2", userId = "user_002"},
    {ip = "203.0.113.3", apiKey = "ak_enterprise_xyz"},
    {ip = "203.0.113.4"},  -- anonymous
    {ip = "1.2.3.4"},      -- blacklisted
    {ip = "10.0.0.1"},     -- whitelisted
}

for _, req in ipairs(testRequests) do
    local allowed, info = limiter:check(req)
    local id = req.userId or req.apiKey or req.ip
    print(string.format("%-30s: %s (%s)",
        id,
        allowed and "ALLOWED" or "DENIED",
        info.reason or ("remaining: " .. tostring(info.remaining))
    ))
end
```

## 10. Rate Limit Headers

```lua
-- Example 10: Standard Rate Limit Headers (RFC 6585, Draft Headers)
local function setRateLimitHeaders(response, info)
    -- Standard headers (RateLimit draft - https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
    response.headers = response.headers or {}
    
    -- X-RateLimit-* (widely used de facto standard)
    response.headers["X-RateLimit-Limit"] = tostring(info.limit)
    response.headers["X-RateLimit-Remaining"] = tostring(math.max(0, info.remaining or 0))
    response.headers["X-RateLimit-Reset"] = tostring(info.resetTime or (os.time() + (info.reset or 0)))
    
    -- New standard (RateLimit header)
    response.headers["RateLimit-Limit"] = info.limit
    response.headers["RateLimit-Remaining"] = math.max(0, info.remaining or 0)
    response.headers["RateLimit-Reset"] = info.reset or 0  -- seconds until reset
    
    if info.policy then
        response.headers["RateLimit-Policy"] = info.policy
    end
    
    -- When rate limited
    if info.retryAfter then
        response.headers["Retry-After"] = tostring(math.ceil(info.retryAfter))
    end
    
    return response
end

-- Simulate HTTP response handling
local function handleRequest(req, rl)
    local allowed, info = rl:check(req.clientId)
    
    local response = {
        status = 200,
        headers = {},
        body = nil
    }
    
    -- Always set headers
    setRateLimitHeaders(response, {
        limit = 100,
        remaining = info.remaining or 0,
        reset = info.reset or 60,
        resetTime = os.time() + (info.reset or 60),
        retryAfter = not allowed and info.retryAfter or nil,
        policy = "100;w=60;comment=\"60 requests per minute\""
    })
    
    if not allowed then
        response.status = 429  -- Too Many Requests
        response.body = {
            error = "Too Many Requests",
            message = "Rate limit exceeded. Please retry after " .. (info.retryAfter or 60) .. " seconds.",
            retryAfter = info.retryAfter,
            documentation = "https://api.example.com/docs/rate-limiting"
        }
        return response
    end
    
    response.body = {data = "success", requestId = math.random(100000)}
    return response
end

-- Create simple rate limiter for demo
local SimpleRL = {}
SimpleRL.__index = SimpleRL

function SimpleRL.new(limit)
    return setmetatable({limit = limit, counters = {}}, SimpleRL)
end

function SimpleRL:check(key)
    local now = os.time()
    local window = math.floor(now / 60) * 60
    local k = key .. ":" .. window
    self.counters[k] = (self.counters[k] or 0) + 1
    local count = self.counters[k]
    local remaining = self.limit - count
    if remaining >= 0 then
        return true, {remaining = remaining, reset = 60 - (now % 60)}
    else
        return false, {remaining = 0, retryAfter = 60 - (now % 60), reset = 60 - (now % 60)}
    end
end

local rl = SimpleRL.new(5)

print("\n=== Rate Limit Headers Demo ===")
for i = 1, 8 do
    local response = handleRequest({clientId = "test_client"}, rl)
    print(string.format("Request %d: HTTP %d", i, response.status))
    print("  X-RateLimit-Limit:", response.headers["X-RateLimit-Limit"])
    print("  X-RateLimit-Remaining:", response.headers["X-RateLimit-Remaining"])
    if response.status == 429 then
        print("  Retry-After:", response.headers["Retry-After"])
        print("  Body:", response.body.message)
    end
end
```

## 11. Rate Limit Bypass Detection

```lua
-- Example 11: Detecting rate limit bypass attempts
local BypassDetector = {}
BypassDetector.__index = BypassDetector

function BypassDetector.new()
    return setmetatable({
        ipGroups = {},        -- Track IPs that seem coordinated
        fingerprints = {},    -- Browser/client fingerprints
        suspiciousIps = {},   -- IPs showing bypass behavior
        patterns = {},        -- Request patterns
        alerts = {}
    }, BypassDetector)
end

function BypassDetector:recordRequest(req)
    local now = os.time()
    
    -- Track by IP
    if not self.ipGroups[req.ip] then
        self.ipGroups[req.ip] = {
            requests = 0,
            userAgents = {},
            paths = {},
            firstSeen = now,
            lastSeen = now
        }
    end
    
    local ipData = self.ipGroups[req.ip]
    ipData.requests = ipData.requests + 1
    ipData.lastSeen = now
    ipData.userAgents[req.userAgent or "unknown"] = true
    ipData.paths[req.path or "/"] = (ipData.paths[req.path or "/"] or 0) + 1
    
    -- Check for distributed attack (many IPs, same pattern)
    self:checkDistributed(req)
    
    -- Check for rotating IPs
    self:checkIPRotation(req)
    
    return self:isSuspicious(req.ip)
end

function BypassDetector:checkDistributed(req)
    -- ตรวจสอบ requests จาก subnet เดียวกัน
    local subnet = req.ip:match("^(%d+%.%d+%.%d+)%.")
    if not subnet then return end
    
    if not self.patterns[subnet] then
        self.patterns[subnet] = {count = 0, ips = {}, firstSeen = os.time()}
    end
    
    local pattern = self.patterns[subnet]
    pattern.count = pattern.count + 1
    pattern.ips[req.ip] = true
    
    local uniqueIps = 0
    for _ in pairs(pattern.ips) do uniqueIps = uniqueIps + 1 end
    
    if uniqueIps > 20 and pattern.count > 1000 then
        self:alert("distributed_attack", {
            subnet = subnet,
            uniqueIps = uniqueIps,
            totalRequests = pattern.count
        })
    end
end

function BypassDetector:checkIPRotation(req)
    -- ตรวจสอบ user ที่สลับ IP บ่อย (Tor, VPN bypass)
    local fingerprint = req.fingerprint  -- browser fingerprint
    if not fingerprint then return end
    
    if not self.fingerprints[fingerprint] then
        self.fingerprints[fingerprint] = {ips = {}, count = 0}
    end
    
    local fp = self.fingerprints[fingerprint]
    fp.ips[req.ip] = true
    fp.count = fp.count + 1
    
    local uniqueIps = 0
    for _ in pairs(fp.ips) do uniqueIps = uniqueIps + 1 end
    
    if uniqueIps > 5 then
        self:alert("ip_rotation", {
            fingerprint = fingerprint,
            uniqueIps = uniqueIps
        })
        self:markSuspicious(req.ip, "ip_rotation")
    end
end

function BypassDetector:alert(type, data)
    local alert = {
        type = type,
        data = data,
        time = os.time()
    }
    table.insert(self.alerts, alert)
    print(string.format("[SECURITY ALERT] %s: %s", type, 
        type == "distributed_attack" and 
        string.format("subnet %s, %d IPs, %d requests", data.subnet, data.uniqueIps, data.totalRequests) or
        string.format("fingerprint %s from %d IPs", data.fingerprint, data.uniqueIps)))
end

function BypassDetector:markSuspicious(ip, reason)
    self.suspiciousIps[ip] = {
        reason = reason,
        markedAt = os.time(),
        count = (self.suspiciousIps[ip] and self.suspiciousIps[ip].count or 0) + 1
    }
end

function BypassDetector:isSuspicious(ip)
    return self.suspiciousIps[ip] ~= nil
end

function BypassDetector:getStats()
    local suspiciousCount = 0
    for _ in pairs(self.suspiciousIps) do suspiciousCount = suspiciousCount + 1 end
    
    return {
        trackedIPs = (function() local c = 0; for _ in pairs(self.ipGroups) do c = c + 1 end; return c end)(),
        suspiciousIPs = suspiciousCount,
        alerts = #self.alerts,
        recentAlerts = self.alerts
    }
end

-- Adaptive Rate Limiter (ปรับ limit ตาม behavior)
local AdaptiveRateLimiter = {}
AdaptiveRateLimiter.__index = AdaptiveRateLimiter

function AdaptiveRateLimiter.new(baseLimit)
    return setmetatable({
        baseLimit = baseLimit,
        adjustments = {},  -- clientId -> multiplier
        detector = BypassDetector.new(),
        normal = TokenBucket.new(baseLimit, baseLimit / 60)
    }, AdaptiveRateLimiter)
end

function AdaptiveRateLimiter:check(req)
    local suspicious = self.detector:recordRequest(req)
    
    -- Reduce limit for suspicious clients
    local multiplier = 1.0
    if suspicious then
        multiplier = 0.1  -- 10% of normal limit
    end
    
    local adjustedCapacity = math.max(1, math.floor(self.baseLimit * multiplier))
    local clientId = req.ip
    
    if not self.adjustments[clientId] then
        self.adjustments[clientId] = {
            bucket = TokenBucket.new(adjustedCapacity, adjustedCapacity / 60),
            multiplier = multiplier
        }
    end
    
    local adj = self.adjustments[clientId]
    if adj.multiplier ~= multiplier then
        adj.multiplier = multiplier
        -- Adjust existing bucket
        adj.bucket.capacity = adjustedCapacity
    end
    
    return adj.bucket:consume()
end

-- Demo
local bypassDetector = BypassDetector.new()

print("\n=== Bypass Detection Demo ===")

-- Simulate normal traffic
for i = 1, 5 do
    bypassDetector:recordRequest({
        ip = "203.0.113." .. i,
        path = "/api/products",
        userAgent = "Mozilla/5.0"
    })
end

-- Simulate IP rotation (same fingerprint, many IPs)
for i = 1, 10 do
    bypassDetector:recordRequest({
        ip = "10." .. i .. ".0.1",
        fingerprint = "fp_abcdef",
        path = "/api/search"
    })
end

local stats = bypassDetector:getStats()
print(string.format("\nStats: %d tracked IPs, %d suspicious, %d alerts",
    stats.trackedIPs, stats.suspiciousIPs, stats.alerts))
```

## 12. Whitelisting และ Blacklisting

```lua
-- Example 12: IP/User whitelist and blacklist management
local AccessControl = {}
AccessControl.__index = AccessControl

function AccessControl.new()
    return setmetatable({
        ipWhitelist = {},
        ipBlacklist = {},
        cidrRanges = {},     -- CIDR blocks
        userWhitelist = {},
        userBlacklist = {},
        apiKeyWhitelist = {},
        apiKeyBlacklist = {},
        expirations = {},    -- Temporary blocks
        history = {}
    }, AccessControl)
end

-- CIDR check (simplified, supports /8, /16, /24)
function AccessControl:_ipInCIDR(ip, cidr)
    local network, prefix = cidr:match("^(.+)/(%d+)$")
    if not network then return ip == cidr end
    
    prefix = tonumber(prefix)
    
    -- Convert IP to number parts
    local function ipParts(ipStr)
        local parts = {}
        for p in ipStr:gmatch("%d+") do
            table.insert(parts, tonumber(p))
        end
        return parts
    end
    
    local ipP = ipParts(ip)
    local netP = ipParts(network)
    
    if #ipP ~= 4 or #netP ~= 4 then return false end
    
    -- Check matching octets based on prefix
    local octets = math.floor(prefix / 8)
    local bits = prefix % 8
    
    for i = 1, octets do
        if ipP[i] ~= netP[i] then return false end
    end
    
    if bits > 0 and octets < 4 then
        local mask = (0xff << (8 - bits)) & 0xff
        if (ipP[octets + 1] & mask) ~= (netP[octets + 1] & mask) then
            return false
        end
    end
    
    return true
end

function AccessControl:_isExpired(key)
    local exp = self.expirations[key]
    if not exp then return false end
    if os.time() > exp then
        self.expirations[key] = nil
        return true  -- expired -> not blocked anymore
    end
    return false  -- still active
end

function AccessControl:whitelistIP(ip, reason)
    self.ipWhitelist[ip] = {added = os.time(), reason = reason or ""}
    table.insert(self.history, {action = "whitelist_ip", target = ip, time = os.time()})
end

function AccessControl:blacklistIP(ip, duration, reason)
    self.ipBlacklist[ip] = {added = os.time(), reason = reason or "", duration = duration}
    if duration then
        self.expirations["ip:" .. ip] = os.time() + duration
    end
    table.insert(self.history, {action = "blacklist_ip", target = ip, duration = duration, time = os.time()})
end

function AccessControl:allowCIDR(cidr)
    table.insert(self.cidrRanges, {type = "allow", cidr = cidr})
end

function AccessControl:denyCIDR(cidr)
    table.insert(self.cidrRanges, {type = "deny", cidr = cidr})
end

function AccessControl:whitelistUser(userId)
    self.userWhitelist[userId] = true
end

function AccessControl:blacklistUser(userId, duration, reason)
    self.userBlacklist[userId] = {reason = reason, added = os.time()}
    if duration then
        self.expirations["user:" .. userId] = os.time() + duration
    end
end

function AccessControl:check(request)
    local ip = request.ip
    local userId = request.userId
    
    -- IP whitelist (highest priority allow)
    if ip and self.ipWhitelist[ip] then
        return true, "ip_whitelisted"
    end
    
    -- User whitelist
    if userId and self.userWhitelist[userId] then
        return true, "user_whitelisted"
    end
    
    -- IP blacklist
    if ip then
        if self.ipBlacklist[ip] and not self:_isExpired("ip:" .. ip) then
            return false, "ip_blacklisted", {
                reason = self.ipBlacklist[ip].reason
            }
        end
    end
    
    -- User blacklist
    if userId then
        if self.userBlacklist[userId] and not self:_isExpired("user:" .. userId) then
            return false, "user_blacklisted", {
                reason = self.userBlacklist[userId].reason
            }
        end
    end
    
    -- CIDR rules (in order)
    if ip then
        for _, rule in ipairs(self.cidrRanges) do
            if self:_ipInCIDR(ip, rule.cidr) then
                if rule.type == "deny" then
                    return false, "cidr_denied", {cidr = rule.cidr}
                elseif rule.type == "allow" then
                    return true, "cidr_allowed"
                end
            end
        end
    end
    
    return true, "default_allow"
end

function AccessControl:getBlockedCount()
    local count = 0
    for ip, data in pairs(self.ipBlacklist) do
        if not self:_isExpired("ip:" .. ip) then count = count + 1 end
    end
    for userId, data in pairs(self.userBlacklist) do
        if not self:_isExpired("user:" .. userId) then count = count + 1 end
    end
    return count
end

-- การใช้งาน
local ac = AccessControl.new()

-- Setup rules
ac:whitelistIP("127.0.0.1")
ac:whitelistIP("10.0.0.1")
ac:blacklistIP("1.2.3.4", 3600, "DDoS attack")
ac:blacklistIP("5.6.7.8", nil, "Persistent spammer")
ac:blacklistUser("user_banned_001", nil, "Terms of service violation")
ac:denyCIDR("192.168.100.0/24")  -- Block test subnet
ac:allowCIDR("10.0.0.0/8")      -- Allow internal network

print("\n=== Access Control Demo ===")
local testCases = {
    {ip = "127.0.0.1",     userId = nil,             expected = "allow"},
    {ip = "1.2.3.4",       userId = nil,             expected = "deny (blacklisted)"},
    {ip = "203.0.113.5",   userId = "user_banned_001", expected = "deny (user)"},
    {ip = "192.168.100.50",userId = "user_ok",        expected = "deny (CIDR)"},
    {ip = "10.0.50.1",     userId = "user_ok",        expected = "allow (CIDR)"},
    {ip = "203.0.113.100", userId = "user_normal",    expected = "allow (default)"},
}

for _, tc in ipairs(testCases) do
    local allowed, reason, info = ac:check(tc)
    print(string.format("%-20s %-20s: %s [%s]",
        tc.ip or "-",
        tc.userId or "-",
        allowed and "ALLOWED" or "DENIED",
        reason
    ))
end
```

## 13. Implementation ใน OpenResty (Production)

```lua
-- Example 13: Production-ready OpenResty rate limiter
-- สำหรับ nginx.conf:
-- lua_shared_dict rate_limits 50m;
-- init_by_lua_block {
--     require "resty.core"
-- }

-- rate_limiter.lua (module)
local _M = {}
local ngx = ngx
local now = ngx.now
local shared = ngx.shared

_M._VERSION = "1.0.0"

local function log(level, msg)
    ngx.log(level, "[RateLimit] " .. msg)
end

-- Token bucket using shared dict (atomic with lua mutex or redis)
function _M.check(key, capacity, rate, window)
    window = window or 60
    local dict = shared.rate_limits
    
    local tokens_key = "tb:" .. key .. ":tokens"
    local time_key   = "tb:" .. key .. ":time"
    
    -- Use ngx.now() for millisecond precision
    local current_time = now()
    
    local tokens, _ = dict:get(tokens_key)
    local last_time, _ = dict:get(time_key)
    
    tokens = tokens or capacity
    last_time = last_time or current_time
    
    -- Refill
    local elapsed = current_time - last_time
    tokens = math.min(capacity, tokens + elapsed * rate)
    
    local allowed = tokens >= 1
    local new_tokens = allowed and tokens - 1 or tokens
    local reset_time = math.ceil(capacity / rate)
    
    dict:set(tokens_key, new_tokens, reset_time + 60)
    dict:set(time_key, current_time, reset_time + 60)
    
    return allowed, {
        remaining = math.max(0, math.floor(new_tokens)),
        limit = capacity,
        reset = math.ceil((capacity - new_tokens) / rate),
        retry_after = not allowed and math.ceil((1 - tokens) / rate) or nil
    }
end

-- Middleware function
function _M.limit_req(options)
    -- options = {
    --   key = string or function,  -- rate limit key
    --   limit = number,            -- requests per window
    --   window = number,           -- window in seconds
    --   burst = number,            -- burst allowance
    --   on_limit = function,       -- custom handler
    -- }
    
    local key
    if type(options.key) == "function" then
        key = options.key()
    else
        key = options.key or ngx.var.remote_addr
    end
    
    local limit = options.limit or 100
    local window = options.window or 60
    local burst = options.burst or 0
    local capacity = limit + burst
    local rate = limit / window
    
    local allowed, info = _M.check(key, capacity, rate, window)
    
    -- Always set headers
    ngx.header["X-RateLimit-Limit"] = limit
    ngx.header["X-RateLimit-Remaining"] = info.remaining
    ngx.header["X-RateLimit-Reset"] = ngx.time() + info.reset
    
    if not allowed then
        if options.on_limit then
            return options.on_limit(info)
        end
        
        ngx.header["Retry-After"] = info.retry_after
        ngx.header["Content-Type"] = "application/json; charset=utf-8"
        ngx.status = 429
        ngx.say('{"error":"Too Many Requests","message":"Rate limit exceeded","retryAfter":' ..
            (info.retry_after or window) .. '}')
        ngx.exit(429)
    end
end

-- Composite key functions
function _M.by_ip()
    return ngx.var.remote_addr
end

function _M.by_user()
    return ngx.ctx.user_id or ngx.var.remote_addr
end

function _M.by_api_key()
    return ngx.var.http_x_api_key or ngx.var.arg_api_key or ngx.var.remote_addr
end

function _M.composite(...)
    local parts = {...}
    local result = {}
    for _, fn in ipairs(parts) do
        if type(fn) == "function" then
            table.insert(result, fn())
        else
            table.insert(result, tostring(fn))
        end
    end
    return table.concat(result, ":")
end

-- Statistics
function _M.stats()
    local dict = shared.rate_limits
    if not dict then return {} end
    
    return {
        size = dict:capacity(),
        free = dict:free_space(),
        keys = 0  -- Cannot enumerate easily in shared dict
    }
end

-- Simulate the module
print("\n=== OpenResty Rate Limiter Module ===")
print("Module version:", _M._VERSION)
print("Functions: check, limit_req, by_ip, by_user, by_api_key, composite")
print()
print("Usage in nginx location block:")
print([[
  location /api/ {
    access_by_lua_block {
      local rl = require "rate_limiter"
      rl.limit_req({
        key = rl.composite(rl.by_ip, rl.by_api_key),
        limit = 100,
        window = 60,
        burst = 20,
      })
    }
    proxy_pass http://backend;
  }
]])
```

## สรุป Rate Limiting

Rate Limiting เป็น essential component ของ production API:

1. **Token Bucket** - ดีสำหรับ burst traffic, flexible
2. **Leaky Bucket** - ควบคุม output rate, smooth traffic
3. **Fixed Window** - ง่าย แต่มี boundary burst problem
4. **Sliding Window Log** - แม่นยำ แต่ memory สูง
5. **Sliding Window Counter** - ผสมความแม่นยำกับ memory efficiency

**Headers มาตรฐาน:**
- `X-RateLimit-Limit` - จำนวน requests ที่อนุญาตต่อ window
- `X-RateLimit-Remaining` - เหลืออีกเท่าไร
- `X-RateLimit-Reset` - Unix timestamp ที่ reset
- `Retry-After` - วินาทีที่ต้องรอ (เมื่อ rate limited)
- HTTP 429 Too Many Requests

**Best Practices:**
- ใช้ Redis สำหรับ distributed rate limiting
- Multi-dimensional limits (per-IP, per-user, per-endpoint)
- Bypass detection ป้องกัน evasion
- Whitelist สำหรับ internal services
- Adaptive limits ตาม behavior
- Rate limit headers ทุก response (ไม่ใช่แค่ตอน limit)
