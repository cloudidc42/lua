# บทที่ 65: Caching Strategies

## บทนำ

Caching เป็นหนึ่งในเทคนิคที่สำคัญที่สุดในการเพิ่มประสิทธิภาพระบบ โดยการเก็บข้อมูลที่ใช้บ่อยไว้ใน storage ที่เร็วกว่า บทนี้ครอบคลุม cache types, patterns, algorithms และ implementation ใน Lua/OpenResty

## 1. Cache Types

```lua
-- Example 1: Understanding cache hierarchy
local CacheTypes = {}

-- L1 Cache: In-process memory (fastest, smallest)
-- - Lua table ใน single process
-- - ngx.ctx (per-request, OpenResty)
-- - เข้าถึงได้แค่ process เดียว

-- L2 Cache: Shared memory
-- - ngx.shared.DICT (OpenResty)
-- - shared memory ระหว่าง workers ใน same machine

-- L3 Cache: Distributed cache
-- - Redis, Memcached
-- - ใช้งานร่วมกันได้ทุก process/machine

-- CDN Cache: Edge cache
-- - Cloudflare, CloudFront, Fastly
-- - Geographic distribution

-- Application Cache: Custom cache layer
-- - Database query cache
-- - Computed result cache
-- - API response cache

-- Performance comparison (approximate latency):
local latency = {
    l1_memory  = "< 1ns",     -- L1 CPU cache
    l2_memory  = "< 5ns",     -- L2/L3 CPU cache
    ram        = "~100ns",    -- RAM (Lua table)
    ssd        = "~100μs",    -- SSD
    redis_local = "< 1ms",   -- Redis local network
    redis_remote = "< 5ms",  -- Redis remote
    database   = "1-100ms",  -- Database query
    http_api   = "10-500ms", -- External API
    cdn_hit    = "< 10ms",   -- CDN cache hit
}

print("=== Cache Hierarchy ===")
print("Cache Type       | Latency       | Scope")
print(string.rep("-", 50))
print(string.format("%-16s | %-13s | %s", "L1 (Lua table)", "< 1μs", "Process"))
print(string.format("%-16s | %-13s | %s", "L2 (ngx.shared)", "< 1μs", "Machine"))
print(string.format("%-16s | %-13s | %s", "Redis (local)", "< 1ms", "Cluster"))
print(string.format("%-16s | %-13s | %s", "Database", "1-100ms", "Datacenter"))
print(string.format("%-16s | %-13s | %s", "CDN", "< 10ms", "Global"))

-- Simple L1 cache
local L1Cache = {}
L1Cache.__index = L1Cache

function L1Cache.new(maxSize, defaultTTL)
    return setmetatable({
        data = {},
        maxSize = maxSize or 1000,
        defaultTTL = defaultTTL or 300,
        hits = 0,
        misses = 0,
        evictions = 0,
        order = {}  -- for LRU tracking
    }, L1Cache)
end

function L1Cache:get(key)
    local entry = self.data[key]
    if not entry then
        self.misses = self.misses + 1
        return nil
    end
    
    -- Check TTL
    if entry.expiresAt and os.time() > entry.expiresAt then
        self.data[key] = nil
        self.misses = self.misses + 1
        return nil
    end
    
    self.hits = self.hits + 1
    entry.lastAccess = os.time()
    return entry.value
end

function L1Cache:set(key, value, ttl)
    -- Evict if full
    local size = 0
    for _ in pairs(self.data) do size = size + 1 end
    
    if size >= self.maxSize and not self.data[key] then
        self:evictLRU()
    end
    
    self.data[key] = {
        value = value,
        createdAt = os.time(),
        lastAccess = os.time(),
        expiresAt = ttl and (os.time() + ttl) or (os.time() + self.defaultTTL)
    }
end

function L1Cache:evictLRU()
    local oldestTime = math.huge
    local oldestKey = nil
    
    for key, entry in pairs(self.data) do
        if entry.lastAccess < oldestTime then
            oldestTime = entry.lastAccess
            oldestKey = key
        end
    end
    
    if oldestKey then
        self.data[oldestKey] = nil
        self.evictions = self.evictions + 1
    end
end

function L1Cache:delete(key)
    if self.data[key] then
        self.data[key] = nil
        return true
    end
    return false
end

function L1Cache:stats()
    local size = 0
    for _ in pairs(self.data) do size = size + 1 end
    local total = self.hits + self.misses
    return {
        size = size,
        maxSize = self.maxSize,
        hits = self.hits,
        misses = self.misses,
        hitRate = total > 0 and (self.hits / total * 100) or 0,
        evictions = self.evictions
    }
end

local cache = L1Cache.new(100, 60)
cache:set("user:1", {name = "Alice", email = "alice@example.com"})
cache:set("user:2", {name = "Bob",   email = "bob@example.com"})

local u1 = cache:get("user:1")
local u2 = cache:get("user:3")  -- miss

local stats = cache:stats()
print(string.format("\nL1 Cache: hits=%d, misses=%d, hit rate=%.0f%%",
    stats.hits, stats.misses, stats.hitRate))
```

## 2. Cache-Aside Pattern

```lua
-- Example 2: Cache-Aside (Lazy Loading)
-- Application ควบคุมการโหลดข้อมูลเข้า cache เอง
-- 1. ตรวจ cache ก่อน
-- 2. Cache miss -> โหลดจาก database
-- 3. บันทึกลง cache
-- 4. Return data

local CacheAside = {}
CacheAside.__index = CacheAside

function CacheAside.new(cache, dataSource)
    return setmetatable({
        cache = cache,
        dataSource = dataSource,
        loadTimes = {},
        totalLoads = 0
    }, CacheAside)
end

function CacheAside:get(key, ttl)
    -- Step 1: Check cache
    local cached = self.cache:get(key)
    if cached ~= nil then
        return cached, "hit"
    end
    
    -- Step 2: Load from data source
    self.totalLoads = self.totalLoads + 1
    local start = os.clock()
    
    local data, err = self.dataSource:load(key)
    
    local elapsed = os.clock() - start
    self.loadTimes[key] = elapsed
    
    if err then
        return nil, "error", err
    end
    
    if data == nil then
        return nil, "not_found"
    end
    
    -- Step 3: Store in cache
    self.cache:set(key, data, ttl)
    
    return data, "miss"
end

function CacheAside:invalidate(key)
    self.cache:delete(key)
end

function CacheAside:refresh(key, ttl)
    -- Force reload from data source
    self.cache:delete(key)
    return self:get(key, ttl)
end

-- Simulated database
local Database = {}
function Database.new()
    return {
        store = {
            ["user:1"] = {id = 1, name = "Alice", email = "alice@example.com", role = "admin"},
            ["user:2"] = {id = 2, name = "Bob",   email = "bob@example.com",   role = "user"},
            ["user:3"] = {id = 3, name = "Carol", email = "carol@example.com", role = "user"},
            ["product:100"] = {id = 100, name = "Laptop", price = 999.99, stock = 50},
            ["product:101"] = {id = 101, name = "Phone",  price = 599.99, stock = 100},
        },
        queryCount = 0,
        avgLatency = 0.05  -- 50ms simulated
    }
end

function Database:load(key)
    self.queryCount = self.queryCount + 1
    -- Simulate latency
    -- In real code: socket.sleep(self.avgLatency)
    return self.store[key], nil
end

function Database:save(key, data)
    self.store[key] = data
end

-- Test Cache-Aside
local db = Database.new()
local l1 = L1Cache.new(100, 300)
local cacheAside = CacheAside.new(l1, db)

print("\n=== Cache-Aside Pattern ===")

-- First access (miss)
local user, status = cacheAside:get("user:1", 60)
print(string.format("get user:1 -> %s (%s)", user and user.name or "nil", status))

-- Second access (hit)
user, status = cacheAside:get("user:1", 60)
print(string.format("get user:1 -> %s (%s)", user and user.name or "nil", status))

-- Unknown key
user, status = cacheAside:get("user:999")
print(string.format("get user:999 -> %s (%s)", tostring(user), status))

-- Multiple keys
for _, key in ipairs({"user:1", "user:2", "user:3", "user:1", "user:2"}) do
    local data, s = cacheAside:get(key, 60)
    print(string.format("  %s: %s (%s)", key, data and data.name or "nil", s))
end

print(string.format("\nDB queries: %d, Cache stats: hits=%d, misses=%d",
    db.queryCount,
    l1:stats().hits,
    l1:stats().misses
))
```

## 3. Write-Through Pattern

```lua
-- Example 3: Write-Through Cache
-- เขียนข้อมูลพร้อมกันทั้ง cache และ database
-- ข้อดี: cache ไม่มี stale data
-- ข้อเสีย: write latency สูงขึ้น

local WriteThrough = {}
WriteThrough.__index = WriteThrough

function WriteThrough.new(cache, dataStore)
    return setmetatable({
        cache = cache,
        dataStore = dataStore,
        writeCount = 0,
        writeLatency = {}
    }, WriteThrough)
end

function WriteThrough:write(key, data, ttl)
    local start = os.clock()
    
    -- Write to database FIRST
    local ok, err = pcall(function()
        self.dataStore:save(key, data)
    end)
    
    if not ok then
        return false, "Database write failed: " .. tostring(err)
    end
    
    -- Then write to cache
    self.cache:set(key, data, ttl)
    
    self.writeCount = self.writeCount + 1
    self.writeLatency[#self.writeLatency + 1] = os.clock() - start
    
    return true
end

function WriteThrough:read(key)
    -- Read from cache first
    local cached = self.cache:get(key)
    if cached then return cached, "hit" end
    
    -- Load from DB on miss
    local data = self.dataStore:load(key)
    if data then
        self.cache:set(key, data)
    end
    return data, "miss"
end

function WriteThrough:delete(key)
    -- Delete from both
    self.dataStore:store[key] = nil
    self.cache:delete(key)
end

function WriteThrough:avgWriteLatency()
    if #self.writeLatency == 0 then return 0 end
    local sum = 0
    for _, l in ipairs(self.writeLatency) do sum = sum + l end
    return sum / #self.writeLatency * 1000  -- ms
end

-- Test
local db2 = Database.new()
local l1_2 = L1Cache.new(100, 300)
local wt = WriteThrough.new(l1_2, db2)

print("\n=== Write-Through Pattern ===")

-- Write through
local ok = wt:write("user:10", {id = 10, name = "Dave", email = "dave@test.com"}, 120)
print("Write user:10:", ok)

-- Immediate read (should hit cache)
local user, status = wt:read("user:10")
print(string.format("Read user:10: %s (%s)", user and user.name or "nil", status))

-- Verify also in DB
local inDb = db2:load("user:10")
print("In DB:", inDb and inDb.name or "nil")
print(string.format("Avg write latency: %.2fms", wt:avgWriteLatency()))
```

## 4. Write-Behind (Write-Back) Pattern

```lua
-- Example 4: Write-Behind Cache
-- เขียนลง cache ก่อน แล้ว async flush ลง database ทีหลัง
-- ข้อดี: write performance สูงมาก
-- ข้อเสีย: risk of data loss ถ้า crash ก่อน flush

local WriteBehind = {}
WriteBehind.__index = WriteBehind

function WriteBehind.new(cache, dataStore, options)
    return setmetatable({
        cache = cache,
        dataStore = dataStore,
        dirtyKeys = {},      -- keys ที่ยังไม่ได้ flush
        writeQueue = {},     -- queue สำหรับ batch writes
        flushInterval = options and options.flushInterval or 5,  -- seconds
        maxQueueSize = options and options.maxQueueSize or 100,
        lastFlush = os.time(),
        flushedCount = 0,
        lostOnCrash = 0  -- hypothetical
    }, WriteBehind)
end

function WriteBehind:write(key, data, ttl)
    -- Write to cache immediately
    self.cache:set(key, data, ttl)
    
    -- Mark as dirty
    self.dirtyKeys[key] = {
        data = data,
        writtenAt = os.time(),
        attempts = 0
    }
    
    -- Add to write queue
    table.insert(self.writeQueue, {key = key, data = data, time = os.time()})
    
    -- Check if should flush (queue full)
    if #self.writeQueue >= self.maxQueueSize then
        print("[WriteBehind] Queue full, flushing...")
        self:flush()
    end
    
    return true
end

function WriteBehind:read(key)
    -- Check cache first
    local cached = self.cache:get(key)
    if cached then return cached, "cache_hit" end
    
    -- Check dirty queue (not yet in DB)
    if self.dirtyKeys[key] then
        return self.dirtyKeys[key].data, "dirty_hit"
    end
    
    -- Load from DB
    local data = self.dataStore:load(key)
    if data then self.cache:set(key, data) end
    return data, "db_hit"
end

function WriteBehind:flush()
    if #self.writeQueue == 0 then return 0 end
    
    local batch = {}
    local count = #self.writeQueue
    
    -- Dequeue all
    for _, item in ipairs(self.writeQueue) do
        batch[item.key] = item.data
    end
    self.writeQueue = {}
    
    -- Batch write to database
    local success = true
    local ok, err = pcall(function()
        for key, data in pairs(batch) do
            self.dataStore:save(key, data)
            self.dirtyKeys[key] = nil
        end
    end)
    
    if not ok then
        print("[WriteBehind] Flush failed: " .. tostring(err))
        -- Re-queue failed items
        for key, data in pairs(batch) do
            if self.dirtyKeys[key] then  -- still dirty
                table.insert(self.writeQueue, {key = key, data = data})
            end
        end
        success = false
    else
        self.flushedCount = self.flushedCount + count
    end
    
    self.lastFlush = os.time()
    return success and count or 0
end

function WriteBehind:update(dt)
    -- Call in game loop or timer
    if os.time() - self.lastFlush >= self.flushInterval then
        local flushed = self:flush()
        if flushed > 0 then
            print(string.format("[WriteBehind] Flushed %d entries to database", flushed))
        end
    end
end

function WriteBehind:stats()
    return {
        dirtyKeys = (function() local c = 0; for _ in pairs(self.dirtyKeys) do c = c + 1 end; return c end)(),
        queueSize = #self.writeQueue,
        flushed = self.flushedCount,
        timeSinceFlush = os.time() - self.lastFlush
    }
end

-- Test
local db3 = Database.new()
local l1_3 = L1Cache.new(100, 300)
local wb = WriteBehind.new(l1_3, db3, {flushInterval = 2, maxQueueSize = 5})

print("\n=== Write-Behind Pattern ===")

-- Write multiple items quickly
for i = 1, 8 do
    wb:write("item:" .. i, {id = i, value = math.random(100)})
    print(string.format("  Wrote item:%d (dirty: %d, queue: %d)",
        i, wb:stats().dirtyKeys, wb:stats().queueSize))
end

-- Flush
local flushed = wb:flush()
print(string.format("Flushed: %d items", flushed))
print("Stats:", wb:stats().dirtyKeys, "dirty keys remaining")
```

## 5. Read-Through Pattern

```lua
-- Example 5: Read-Through Cache
-- Cache ตัวเองรับผิดชอบโหลดข้อมูลจาก DB
-- Application ไม่รู้จัก DB โดยตรง

local ReadThrough = {}
ReadThrough.__index = ReadThrough

function ReadThrough.new(dataLoader, options)
    return setmetatable({
        data = {},              -- cache store
        loader = dataLoader,   -- function to load data
        ttl = options and options.ttl or 300,
        maxSize = options and options.maxSize or 10000,
        stats = {hits = 0, misses = 0, loads = 0, errors = 0}
    }, ReadThrough)
end

function ReadThrough:get(key, options)
    options = options or {}
    local ttl = options.ttl or self.ttl
    
    -- Check cache
    local entry = self.data[key]
    local now = os.time()
    
    if entry and (not entry.expiresAt or now < entry.expiresAt) then
        self.stats.hits = self.stats.hits + 1
        return entry.value, nil
    end
    
    -- Load through cache
    self.stats.misses = self.stats.misses + 1
    self.stats.loads = self.stats.loads + 1
    
    local ok, value = pcall(self.loader, key)
    
    if not ok then
        self.stats.errors = self.stats.errors + 1
        
        -- Return stale data if available (stale-while-revalidate)
        if entry and options.staleOnError then
            print(string.format("[ReadThrough] Error loading %s, returning stale data", key))
            return entry.value, "stale"
        end
        
        return nil, value  -- value is error message
    end
    
    if value ~= nil then
        self:_store(key, value, ttl)
    end
    
    return value, nil
end

function ReadThrough:_store(key, value, ttl)
    -- Evict if needed (simple LRU)
    local size = 0
    for _ in pairs(self.data) do size = size + 1 end
    
    if size >= self.maxSize then
        -- Remove oldest
        local oldest, oldestKey = math.huge, nil
        for k, e in pairs(self.data) do
            if e.loadedAt < oldest then
                oldest = e.loadedAt
                oldestKey = k
            end
        end
        if oldestKey then self.data[oldestKey] = nil end
    end
    
    self.data[key] = {
        value = value,
        loadedAt = os.time(),
        expiresAt = os.time() + ttl
    }
end

function ReadThrough:invalidate(key)
    self.data[key] = nil
end

function ReadThrough:invalidatePattern(pattern)
    local removed = 0
    for key in pairs(self.data) do
        if key:match(pattern) then
            self.data[key] = nil
            removed = removed + 1
        end
    end
    return removed
end

function ReadThrough:getStats()
    local total = self.stats.hits + self.stats.misses
    return {
        hits = self.stats.hits,
        misses = self.stats.misses,
        hitRate = total > 0 and (self.stats.hits / total * 100) or 0,
        loads = self.stats.loads,
        errors = self.stats.errors,
        size = (function() local c = 0; for _ in pairs(self.data) do c = c + 1 end; return c end)()
    }
end

-- ตัวอย่าง: User profile cache
local db4 = Database.new()

local userCache = ReadThrough.new(function(key)
    -- This is the loader function - called on cache miss
    print("  [DB] Loading: " .. key)
    local data = db4:load(key)
    if not data then error("Key not found: " .. key) end
    return data
end, {ttl = 120, maxSize = 1000})

print("\n=== Read-Through Pattern ===")

-- Access data
local keys = {"user:1", "user:2", "user:1", "user:3", "user:2", "user:1"}
for _, key in ipairs(keys) do
    local data, err = userCache:get(key)
    if data then
        print(string.format("  %s: %s (loaded: %s)", key, data.name, err or "fresh"))
    else
        print(string.format("  %s: ERROR - %s", key, tostring(err)))
    end
end

local st = userCache:getStats()
print(string.format("\nStats: hits=%d, misses=%d, hit rate=%.0f%%",
    st.hits, st.misses, st.hitRate))
```

## 6. Cache Invalidation

```lua
-- Example 6: Cache invalidation strategies
-- "There are only two hard things in Computer Science:
--  cache invalidation and naming things" - Phil Karlton

local CacheInvalidator = {}
CacheInvalidator.__index = CacheInvalidator

function CacheInvalidator.new(cache)
    return setmetatable({
        cache = cache,
        tags = {},          -- tag -> set of cache keys
        keyTags = {},       -- key -> set of tags
        versionCounters = {},  -- key prefix -> version
    }, CacheInvalidator)
end

-- Tag-based invalidation
function CacheInvalidator:setWithTags(key, value, ttl, tags)
    self.cache:set(key, value, ttl)
    
    for _, tag in ipairs(tags) do
        if not self.tags[tag] then
            self.tags[tag] = {}
        end
        self.tags[tag][key] = true
    end
    
    self.keyTags[key] = tags
end

function CacheInvalidator:invalidateByTag(tag)
    local keys = self.tags[tag]
    if not keys then return 0 end
    
    local count = 0
    for key in pairs(keys) do
        self.cache:delete(key)
        -- Remove from keyTags
        if self.keyTags[key] then
            for i, t in ipairs(self.keyTags[key]) do
                if t == tag then
                    -- Don't need to remove from other tag sets here
                    break
                end
            end
        end
        count = count + 1
    end
    
    self.tags[tag] = {}
    return count
end

-- Version-based invalidation (namespace versioning)
function CacheInvalidator:getVersionedKey(prefix, id)
    local version = self.versionCounters[prefix] or 1
    return prefix .. ":" .. version .. ":" .. id
end

function CacheInvalidator:incrementVersion(prefix)
    self.versionCounters[prefix] = (self.versionCounters[prefix] or 1) + 1
    print(string.format("[Invalidation] Version bumped for '%s': v%d",
        prefix, self.versionCounters[prefix]))
    return self.versionCounters[prefix]
end

function CacheInvalidator:setVersioned(prefix, id, value, ttl)
    local key = self:getVersionedKey(prefix, id)
    self.cache:set(key, value, ttl)
    return key
end

function CacheInvalidator:getVersioned(prefix, id)
    local key = self:getVersionedKey(prefix, id)
    return self.cache:get(key)
end

-- Invalidate all keys with prefix (by incrementing version)
function CacheInvalidator:invalidateNamespace(prefix)
    return self:incrementVersion(prefix)
end

-- Dependency-based invalidation
local DependencyCache = {}
DependencyCache.__index = DependencyCache

function DependencyCache.new(cache)
    return setmetatable({
        cache = cache,
        dependencies = {}  -- key -> list of dependent keys
    }, DependencyCache)
end

function DependencyCache:set(key, value, ttl, dependsOn)
    self.cache:set(key, value, ttl)
    
    if dependsOn then
        for _, depKey in ipairs(dependsOn) do
            if not self.dependencies[depKey] then
                self.dependencies[depKey] = {}
            end
            self.dependencies[depKey][key] = true
        end
    end
end

function DependencyCache:invalidate(key)
    self.cache:delete(key)
    
    -- Cascade invalidation
    if self.dependencies[key] then
        for depKey in pairs(self.dependencies[key]) do
            print(string.format("  Cascade invalidating: %s (depends on %s)", depKey, key))
            self:invalidate(depKey)  -- Recursive
        end
        self.dependencies[key] = nil
    end
end

-- ตัวอย่างการใช้งาน
local baseCache = L1Cache.new(1000, 300)
local invalidator = CacheInvalidator.new(baseCache)
local depCache = DependencyCache.new(L1Cache.new(1000, 300))

print("\n=== Cache Invalidation Strategies ===")

-- 1. Tag-based
print("\n1. Tag-based invalidation:")
invalidator:setWithTags("user:1:profile", {name = "Alice"}, 300, {"user:1", "users"})
invalidator:setWithTags("user:1:orders", {orders = 5}, 300, {"user:1", "orders"})
invalidator:setWithTags("user:2:profile", {name = "Bob"}, 300, {"user:2", "users"})

print("Invalidating tag 'user:1':")
local count = invalidator:invalidateByTag("user:1")
print("  Invalidated", count, "keys")

-- 2. Version-based (namespace invalidation)
print("\n2. Version-based invalidation:")
invalidator:setVersioned("products", "100", {name = "Laptop", price = 999})
invalidator:setVersioned("products", "101", {name = "Phone",  price = 599})

local p = invalidator:getVersioned("products", "100")
print("Get product:100:", p and p.name or "nil")

-- Invalidate ALL products at once (just bump version)
invalidator:invalidateNamespace("products")
local p2 = invalidator:getVersioned("products", "100")
print("After namespace invalidation:", p2 and p2.name or "nil (gone)")

-- 3. Dependency-based
print("\n3. Dependency-based invalidation:")
depCache:set("user:1", {name = "Alice"}, 300)
depCache:set("user:1:posts", {posts = 10}, 300, {"user:1"})
depCache:set("user:1:summary", {total = 15}, 300, {"user:1", "user:1:posts"})

print("Cache has user:1:", depCache.cache:get("user:1") ~= nil)
print("Cache has user:1:summary:", depCache.cache:get("user:1:summary") ~= nil)

print("Invalidating user:1 (cascade):")
depCache:invalidate("user:1")
print("Cache has user:1:summary after:", depCache.cache:get("user:1:summary") ~= nil)
```

## 7. TTL Strategies

```lua
-- Example 7: TTL (Time-To-Live) strategies
local TTLStrategy = {}

-- Static TTL: คงที่ ง่ายที่สุด
function TTLStrategy.static(seconds)
    return function(key, data) return seconds end
end

-- Dynamic TTL: คำนวณจากข้อมูล
function TTLStrategy.dynamic(fn)
    return function(key, data) return fn(key, data) end
end

-- Jittered TTL: สุ่ม offset เพื่อกระจาย expiration
-- ป้องกัน thundering herd เมื่อ cache expire พร้อมกัน
function TTLStrategy.jittered(baseTTL, jitterPercent)
    jitterPercent = jitterPercent or 0.1  -- 10% jitter
    return function(key, data)
        local jitter = baseTTL * jitterPercent
        return baseTTL + (math.random() * jitter * 2 - jitter)
    end
end

-- Sliding TTL: ต่ออายุเมื่อมีการเข้าถึง
local SlidingTTLCache = {}
SlidingTTLCache.__index = SlidingTTLCache

function SlidingTTLCache.new(maxIdleTime)
    return setmetatable({
        data = {},
        maxIdleTime = maxIdleTime or 300,  -- reset TTL on access
        absoluteTTL = maxIdleTime * 10     -- absolute max TTL
    }, SlidingTTLCache)
end

function SlidingTTLCache:get(key)
    local entry = self.data[key]
    if not entry then return nil end
    
    local now = os.time()
    
    -- Check absolute TTL
    if now > entry.createdAt + self.absoluteTTL then
        self.data[key] = nil
        return nil
    end
    
    -- Check idle TTL
    if now > entry.lastAccess + self.maxIdleTime then
        self.data[key] = nil
        return nil
    end
    
    -- Extend TTL on access (sliding window)
    entry.lastAccess = now
    return entry.value
end

function SlidingTTLCache:set(key, value)
    self.data[key] = {
        value = value,
        createdAt = os.time(),
        lastAccess = os.time()
    }
end

-- Stale-While-Revalidate
local StaleWhileRevalidate = {}
StaleWhileRevalidate.__index = StaleWhileRevalidate

function StaleWhileRevalidate.new(loader, freshTTL, staleTTL)
    return setmetatable({
        data = {},
        loader = loader,
        freshTTL = freshTTL or 60,    -- fresh for 60s
        staleTTL = staleTTL or 3600,  -- stale ok for 1 hour
        revalidating = {}              -- keys being revalidated
    }, StaleWhileRevalidate)
end

function StaleWhileRevalidate:get(key)
    local entry = self.data[key]
    local now = os.time()
    
    if not entry then
        -- No cache, load synchronously
        local value = self.loader(key)
        if value then
            self.data[key] = {
                value = value,
                loadedAt = now
            }
        end
        return value, "miss"
    end
    
    local age = now - entry.loadedAt
    
    if age < self.freshTTL then
        -- Fresh: return immediately
        return entry.value, "fresh"
    elseif age < self.staleTTL then
        -- Stale: return stale data, revalidate in background
        if not self.revalidating[key] then
            self.revalidating[key] = true
            print(string.format("[SWR] Returning stale data for '%s', revalidating async...", key))
            
            -- Async revalidation (simulate)
            -- In real code: use coroutine, ngx.timer, or goroutine
            local newValue = self.loader(key)
            if newValue then
                self.data[key] = {value = newValue, loadedAt = os.time()}
                print(string.format("[SWR] '%s' revalidated", key))
            end
            self.revalidating[key] = nil
        end
        return entry.value, "stale"
    else
        -- Too stale: must reload
        self.data[key] = nil
        local value = self.loader(key)
        if value then
            self.data[key] = {value = value, loadedAt = now}
        end
        return value, "expired"
    end
end

-- ตัวอย่าง TTL strategies
print("\n=== TTL Strategies Demo ===")

-- Jittered TTL
local jitter = TTLStrategy.jittered(300, 0.2)
print("Jittered TTL samples:")
for i = 1, 5 do
    local ttl = jitter("key", {})
    print(string.format("  %.0fs", ttl))
end

-- Sliding TTL cache
local sliding = SlidingTTLCache.new(10)  -- 10s idle timeout
sliding:set("session:abc123", {userId = 1, data = "..."})
print("\nSliding TTL - get session:", sliding:get("session:abc123") ~= nil)

-- SWR
local counter = 0
local swrCache = StaleWhileRevalidate.new(
    function(key)
        counter = counter + 1
        print(string.format("  [Loader] Loading %s (call #%d)", key, counter))
        return {value = counter, key = key}
    end,
    5,   -- fresh for 5 seconds
    60   -- stale ok for 60 seconds
)

print("\nStale-While-Revalidate:")
local v, status = swrCache:get("data:1")
print("First get:", v and v.value, status)
v, status = swrCache:get("data:1")
print("Second get:", v and v.value, status)
```

## 8. Cache Stampede Prevention

```lua
-- Example 8: Cache stampede / thundering herd prevention
-- เกิดเมื่อ cache expire และ requests หลายตัวพยายาม rebuild พร้อมกัน

local StampedeProtection = {}
StampedeProtection.__index = StampedeProtection

-- Probabilistic early expiration (XFetch algorithm)
-- Cache entry "pretends" to expire early with increasing probability
-- เพื่อให้ refresh เกิดก่อน expire จริง
function StampedeProtection.xFetch(entry, beta)
    if not entry then return true end  -- definitely expired
    
    beta = beta or 1.0  -- higher = earlier refresh
    
    local now = os.time()
    local ttl = entry.expiresAt - now
    
    if ttl <= 0 then return true end
    
    -- Simulate recomputation time (use actual time in prod)
    local delta = entry.computeTime or 0.1
    
    -- XFetch formula: expire early with probability
    local shouldRefresh = -delta * beta * math.log(math.random()) > ttl
    return shouldRefresh
end

-- Mutex-based protection (single reload)
local MutexCache = {}
MutexCache.__index = MutexCache

function MutexCache.new(loader, ttl)
    return setmetatable({
        cache = {},
        loader = loader,
        ttl = ttl or 300,
        loading = {},   -- keys currently being loaded
        waiters = {}    -- waiters per key
    }, MutexCache)
end

function MutexCache:get(key)
    local entry = self.cache[key]
    local now = os.time()
    
    if entry and now < entry.expiresAt then
        return entry.value, "hit"
    end
    
    -- Check if already loading (prevent stampede)
    if self.loading[key] then
        -- Wait for the loading to complete (simplified)
        print(string.format("  [MutexCache] %s: another thread is loading, waiting...", key))
        -- In real code: use semaphore, channel, or condition variable
        -- Here we just return stale or nil
        if entry then
            return entry.value, "stale_wait"
        end
        return nil, "loading"
    end
    
    -- Start loading
    self.loading[key] = true
    local start = os.clock()
    
    local ok, value = pcall(self.loader, key)
    
    local computeTime = os.clock() - start
    self.loading[key] = nil
    
    if ok and value ~= nil then
        self.cache[key] = {
            value = value,
            expiresAt = now + self.ttl,
            computeTime = computeTime
        }
        return value, "loaded"
    else
        return nil, "error"
    end
end

-- Staggered invalidation (distribute expiration times)
local function staggeredCache(items, baseTTL, spreadFactor)
    local cache = {}
    spreadFactor = spreadFactor or 0.3
    
    for i, item in ipairs(items) do
        -- Spread expiration over a range
        local ttlSpread = baseTTL * spreadFactor
        local ttl = baseTTL + (i / #items * ttlSpread)
        cache[item.key] = {
            value = item.value,
            expiresAt = os.time() + ttl
        }
    end
    
    return cache
end

-- Test stampede protection
print("\n=== Cache Stampede Prevention ===")

-- XFetch early expiration
print("XFetch early expiration (beta=1.5, 100 simulations):")
local refreshCount = 0
for i = 1, 100 do
    local entry = {
        value = "data",
        expiresAt = os.time() + 5,  -- 5 seconds left
        computeTime = 0.5  -- 500ms to recompute
    }
    if StampedeProtection.xFetch(entry, 1.5) then
        refreshCount = refreshCount + 1
    end
end
print(string.format("  Would refresh early: %d/100 times", refreshCount))

-- Mutex cache
local loads = 0
local mutex = MutexCache.new(function(key)
    loads = loads + 1
    return {data = key, loadedAt = os.time()}
end, 30)

-- Simulate multiple requests
print("\nMutex cache:")
for i = 1, 5 do
    local val, status = mutex:get("product:100")
    print(string.format("  Request %d: %s [%s]", i, val and "got data" or "nil", status))
end
print("  Total DB loads:", loads)

-- Staggered cache
print("\nStaggered expiration (to prevent mass expiry):")
local items = {}
for i = 1, 10 do
    table.insert(items, {key = "item:" .. i, value = "data " .. i})
end
local staggered = staggeredCache(items, 300, 0.2)
local minTTL, maxTTL = math.huge, 0
local now = os.time()
for key, entry in pairs(staggered) do
    local ttl = entry.expiresAt - now
    minTTL = math.min(minTTL, ttl)
    maxTTL = math.max(maxTTL, ttl)
end
print(string.format("  TTL spread: %ds - %ds (spread: %ds)",
    math.floor(minTTL), math.floor(maxTTL), math.floor(maxTTL - minTTL)))
```

## 9. Consistent Hashing สำหรับ Distributed Cache

```lua
-- Example 9: Consistent hashing for distributed cache
-- ใช้เมื่อมี cache nodes หลายตัว และต้องการกระจาย keys อย่างสม่ำเสมอ
-- ข้อดี: เมื่อเพิ่ม/ลบ node, กระทบแค่ส่วนหนึ่งของ keys

local ConsistentHash = {}
ConsistentHash.__index = ConsistentHash

-- Simple hash function (djb2)
local function hash(str)
    local h = 5381
    for i = 1, #str do
        h = ((h * 33) ~ string.byte(str, i)) & 0x7FFFFFFF
    end
    return h
end

function ConsistentHash.new(virtualNodes)
    return setmetatable({
        ring = {},           -- sorted list of {hash, node}
        nodes = {},          -- active nodes
        virtualNodes = virtualNodes or 150  -- virtual replicas per node
    }, ConsistentHash)
end

function ConsistentHash:addNode(node)
    self.nodes[node] = true
    
    -- Add virtual nodes
    for i = 1, self.virtualNodes do
        local virtualKey = node .. "#" .. i
        local h = hash(virtualKey)
        table.insert(self.ring, {hash = h, node = node})
    end
    
    -- Keep ring sorted
    table.sort(self.ring, function(a, b) return a.hash < b.hash end)
    
    print(string.format("[CH] Added node '%s' (%d virtual nodes, ring size: %d)",
        node, self.virtualNodes, #self.ring))
end

function ConsistentHash:removeNode(node)
    self.nodes[node] = nil
    
    -- Remove all virtual nodes for this physical node
    local newRing = {}
    for _, entry in ipairs(self.ring) do
        if entry.node ~= node then
            table.insert(newRing, entry)
        end
    end
    self.ring = newRing
    
    print(string.format("[CH] Removed node '%s' (ring size: %d)", node, #self.ring))
end

function ConsistentHash:getNode(key)
    if #self.ring == 0 then return nil end
    
    local h = hash(key)
    
    -- Binary search for first node >= h
    local lo, hi = 1, #self.ring
    while lo < hi do
        local mid = math.floor((lo + hi) / 2)
        if self.ring[mid].hash < h then
            lo = mid + 1
        else
            hi = mid
        end
    end
    
    -- Wrap around
    if lo > #self.ring then lo = 1 end
    
    return self.ring[lo].node
end

function ConsistentHash:getNodes(key, count)
    -- Get multiple nodes (for replication)
    local nodes = {}
    local seen = {}
    
    local h = hash(key)
    local lo = 1
    
    -- Find start position
    for i, entry in ipairs(self.ring) do
        if entry.hash >= h then
            lo = i
            break
        end
    end
    
    local pos = lo
    while #nodes < count do
        local entry = self.ring[pos]
        if not seen[entry.node] then
            table.insert(nodes, entry.node)
            seen[entry.node] = true
        end
        pos = (pos % #self.ring) + 1
        if pos == lo then break end
    end
    
    return nodes
end

function ConsistentHash:getDistribution(keys)
    local dist = {}
    for node in pairs(self.nodes) do
        dist[node] = 0
    end
    
    for _, key in ipairs(keys) do
        local node = self:getNode(key)
        if node then
            dist[node] = (dist[node] or 0) + 1
        end
    end
    
    return dist
end

-- ตัวอย่าง
local ch = ConsistentHash.new(150)

ch:addNode("cache-1:6379")
ch:addNode("cache-2:6379")
ch:addNode("cache-3:6379")

print("\n=== Consistent Hashing ===")

-- Distribute 1000 test keys
local testKeys = {}
for i = 1, 1000 do
    table.insert(testKeys, "user:" .. i)
end

local dist = ch:getDistribution(testKeys)
print("\nKey distribution (3 nodes, 1000 keys):")
for node, count in pairs(dist) do
    print(string.format("  %s: %d keys (%.1f%%)", node, count, count/10))
end

-- Add a node
print("\nAdding cache-4:")
ch:addNode("cache-4:6379")

-- Check redistribution
local dist2 = ch:getDistribution(testKeys)
local moved = 0
for i, key in ipairs(testKeys) do
    local newNode = ch:getNode(key)
    if newNode ~= ch:getNode(key) then  -- simplified
        -- In real comparison would track original assignment
    end
end

print("Distribution after adding node:")
for node, count in pairs(dist2) do
    print(string.format("  %s: %d keys (%.1f%%)", node, count, count/10))
end

-- Test replication (get 2 nodes for each key)
local key = "user:12345"
local replicas = ch:getNodes(key, 2)
print(string.format("\nKey '%s' -> nodes: %s", key, table.concat(replicas, ", ")))
```

## 10. ngx.shared สำหรับ OpenResty Caching

```lua
-- Example 10: OpenResty shared memory cache
-- nginx.conf:
-- lua_shared_dict my_cache 10m;
-- lua_shared_dict cache_locks 1m;

local SharedCache = {}
SharedCache._VERSION = "1.0"

-- ใน OpenResty จริง:
-- local dict = ngx.shared.my_cache

-- Simulate ngx.shared.DICT for testing
local function makeSharedDict()
    local d = {
        _data = {},
        _expiry = {},
        capacity = function(self) return 10 * 1024 * 1024 end,  -- 10MB
        free_space = function(self)
            local used = 0
            for k, v in pairs(self._data) do
                used = used + #k + #tostring(v)
            end
            return 10 * 1024 * 1024 - used
        end
    }
    
    function d:get(key)
        if self._expiry[key] and os.time() > self._expiry[key] then
            self._data[key] = nil
            self._expiry[key] = nil
            return nil, nil  -- (value, flags)
        end
        return self._data[key], 0
    end
    
    function d:set(key, value, exptime, flags)
        self._data[key] = value
        if exptime and exptime > 0 then
            self._expiry[key] = os.time() + exptime
        else
            self._expiry[key] = nil
        end
        return true, nil, false
    end
    
    function d:add(key, value, exptime)
        if self._data[key] then return false, "exists", false end
        return self:set(key, value, exptime)
    end
    
    function d:incr(key, value, init, init_ttl)
        local current = tonumber(self._data[key])
        if not current then
            if init then
                self._data[key] = init + value
                if init_ttl then self._expiry[key] = os.time() + init_ttl end
                return init + value
            end
            return nil, "not found"
        end
        self._data[key] = current + value
        return current + value
    end
    
    function d:delete(key)
        self._data[key] = nil
        self._expiry[key] = nil
    end
    
    function d:flush_expired()
        local count = 0
        for k in pairs(self._expiry) do
            if os.time() > self._expiry[k] then
                self._data[k] = nil
                self._expiry[k] = nil
                count = count + 1
            end
        end
        return count
    end
    
    function d:get_keys(max)
        local keys = {}
        for k in pairs(self._data) do
            table.insert(keys, k)
            if max and #keys >= max then break end
        end
        return keys
    end
    
    return d
end

local ngx_shared = {my_cache = makeSharedDict()}

-- Cache wrapper
local function createSharedCache(dictName)
    local dict = ngx_shared[dictName]
    
    return {
        get = function(key)
            local value, flags = dict:get(key)
            if value then
                -- Deserialize if JSON
                if type(value) == "string" and value:sub(1,1) == "{" then
                    local ok, decoded = pcall(function()
                        -- Simple JSON decode simulation
                        return value  -- In real: require("cjson").decode(value)
                    end)
                    if ok then return decoded end
                end
            end
            return value
        end,
        
        set = function(key, value, ttl)
            local encoded = value
            if type(value) == "table" then
                -- Serialize: encoded = cjson.encode(value)
                encoded = tostring(value)
            end
            return dict:set(key, encoded, ttl or 300)
        end,
        
        delete = function(key)
            dict:delete(key)
        end,
        
        incr = function(key, step, init, ttl)
            return dict:incr(key, step or 1, init or 0, ttl)
        end,
        
        exists = function(key)
            local v = dict:get(key)
            return v ~= nil
        end,
        
        stats = function()
            return {
                free = dict:free_space(),
                total = dict:capacity()
            }
        end
    }
end

local sharedCache = createSharedCache("my_cache")

print("\n=== ngx.shared Cache Demo ===")

-- Basic operations
sharedCache.set("config:app_version", "2.1.0", 3600)
sharedCache.set("config:feature_flags", {darkMode = true, betaUI = false}, 600)
sharedCache.set("counter:page_views", 0)

print("App version:", sharedCache.get("config:app_version"))

-- Counter
for i = 1, 5 do
    local val = ngx_shared.my_cache:incr("counter:page_views", 1, 0)
    print(string.format("  Page views: %d", val))
end

-- Cache miss
local missing = sharedCache.get("nonexistent:key")
print("Missing key:", missing)

local stats = sharedCache.stats()
print(string.format("Cache stats: %.1fKB free / %.1fKB total",
    stats.free / 1024, stats.total / 1024))

-- Distributed lock using shared dict
local function acquireLock(key, timeout)
    local lockKey = "lock:" .. key
    local ok = ngx_shared.my_cache:add(lockKey, 1, timeout or 5)
    return ok
end

local function releaseLock(key)
    ngx_shared.my_cache:delete("lock:" .. key)
end

print("\nDistributed lock test:")
local locked = acquireLock("resource:1", 10)
print("Lock acquired:", locked)
local locked2 = acquireLock("resource:1", 10)
print("Second lock attempt:", locked2)
releaseLock("resource:1")
local locked3 = acquireLock("resource:1", 10)
print("After release:", locked3)
```

## 11. Redis Caching Layer

```lua
-- Example 11: Redis as caching layer
local RedisCache = {}
RedisCache.__index = RedisCache

function RedisCache.new(host, port, password, db)
    return setmetatable({
        host = host or "127.0.0.1",
        port = port or 6379,
        password = password,
        db = db or 0,
        connected = false,
        ops = 0,
        hitCount = 0,
        missCount = 0
    }, RedisCache)
end

-- Mock Redis operations (replace with actual redis library)
local mockRedisData = {}
local mockRedisExpiry = {}

function RedisCache:get(key)
    self.ops = self.ops + 1
    -- Check expiry
    if mockRedisExpiry[key] and os.time() > mockRedisExpiry[key] then
        mockRedisData[key] = nil
        mockRedisExpiry[key] = nil
        self.missCount = self.missCount + 1
        return nil
    end
    local v = mockRedisData[key]
    if v then self.hitCount = self.hitCount + 1 else self.missCount = self.missCount + 1 end
    return v
end

function RedisCache:set(key, value, ttl)
    self.ops = self.ops + 1
    mockRedisData[key] = value
    if ttl then mockRedisExpiry[key] = os.time() + ttl end
    return "OK"
end

function RedisCache:del(key)
    mockRedisData[key] = nil
    mockRedisExpiry[key] = nil
    return 1
end

function RedisCache:expire(key, ttl)
    if mockRedisData[key] then
        mockRedisExpiry[key] = os.time() + ttl
        return 1
    end
    return 0
end

function RedisCache:incr(key)
    mockRedisData[key] = (tonumber(mockRedisData[key]) or 0) + 1
    return mockRedisData[key]
end

function RedisCache:hset(key, field, value)
    if not mockRedisData[key] then mockRedisData[key] = {} end
    mockRedisData[key][field] = value
    return 1
end

function RedisCache:hget(key, field)
    if not mockRedisData[key] then return nil end
    return mockRedisData[key][field]
end

function RedisCache:hgetall(key)
    return mockRedisData[key]
end

function RedisCache:zadd(key, score, member)
    if not mockRedisData[key] then mockRedisData[key] = {} end
    table.insert(mockRedisData[key], {score = score, member = member})
    table.sort(mockRedisData[key], function(a, b) return a.score < b.score end)
    return 1
end

function RedisCache:zrangebyscore(key, min, max)
    local result = {}
    for _, entry in ipairs(mockRedisData[key] or {}) do
        if entry.score >= min and entry.score <= max then
            table.insert(result, entry.member)
        end
    end
    return result
end

function RedisCache:pipeline(fn)
    -- Simplified pipeline
    local cmds = {}
    local pipe = {
        get = function(p, k) table.insert(cmds, {"get", k}) end,
        set = function(p, k, v, t) table.insert(cmds, {"set", k, v, t}) end,
        del = function(p, k) table.insert(cmds, {"del", k}) end,
        execute = function(p)
            local results = {}
            for _, cmd in ipairs(cmds) do
                if cmd[1] == "get" then table.insert(results, self:get(cmd[2]))
                elseif cmd[1] == "set" then table.insert(results, self:set(cmd[2], cmd[3], cmd[4]))
                elseif cmd[1] == "del" then table.insert(results, self:del(cmd[2]))
                end
            end
            return results
        end
    }
    if fn then fn(pipe) end
    return pipe
end

function RedisCache:stats()
    local total = self.hitCount + self.missCount
    return {
        hits = self.hitCount,
        misses = self.missCount,
        hitRate = total > 0 and (self.hitCount / total * 100) or 0,
        totalOps = self.ops
    }
end

-- High-level cache service using Redis
local CacheService = {}
CacheService.__index = CacheService

function CacheService.new(redis)
    return setmetatable({redis = redis}, CacheService)
end

-- Cache user session
function CacheService:setSession(sessionId, data, ttl)
    local key = "session:" .. sessionId
    -- Store as hash
    for field, value in pairs(data) do
        self.redis:hset(key, field, tostring(value))
    end
    self.redis:expire(key, ttl or 3600)
end

function CacheService:getSession(sessionId)
    return self.redis:hgetall("session:" .. sessionId)
end

-- Cache product with TTL
function CacheService:cacheProduct(productId, data, ttl)
    self.redis:set("product:" .. productId, data, ttl or 300)
end

function CacheService:getProduct(productId)
    return self.redis:get("product:" .. productId)
end

-- Leaderboard (sorted set)
function CacheService:updateScore(userId, score)
    self.redis:zadd("leaderboard", score, userId)
end

function CacheService:getTopScores(min, max)
    return self.redis:zrangebyscore("leaderboard", min, max)
end

-- Cache counter
function CacheService:incrementPageView(page)
    return self.redis:incr("pageview:" .. page)
end

-- ตัวอย่าง
local redis = RedisCache.new()
local cacheService = CacheService.new(redis)

print("\n=== Redis Cache Layer Demo ===")

-- Session caching
cacheService:setSession("sess_abc123", {
    userId = "user_001",
    role = "admin",
    loginAt = os.time()
}, 3600)

local session = cacheService:getSession("sess_abc123")
print("Session user:", session and session.userId or "nil")

-- Product cache
cacheService:cacheProduct("prod_100", "Laptop Pro - $999", 300)
local product = cacheService:getProduct("prod_100")
print("Product:", product or "nil")

-- Leaderboard
for i = 1, 5 do
    cacheService:updateScore("user_" .. i, math.random(1000, 9999))
end

local top = cacheService:getTopScores(0, math.huge)
print("\nLeaderboard entries:", #top)

-- Page views
for i = 1, 10 do
    cacheService:incrementPageView("/home")
end
local views = redis:get("pageview:/home")
print("Page views /home:", views)

-- Stats
local st = redis:stats()
print(string.format("\nRedis stats: %d hits, %d misses (%.0f%% hit rate)",
    st.hits, st.misses, st.hitRate))
```

## 12. HTTP Caching Headers

```lua
-- Example 12: HTTP cache control headers
local HTTPCache = {}

-- Generate Cache-Control header
function HTTPCache.cacheControl(options)
    local directives = {}
    
    if options.noStore then
        return "no-store"
    end
    
    if options.noCache then
        table.insert(directives, "no-cache")
    end
    
    if options.private then
        table.insert(directives, "private")
    elseif options.public then
        table.insert(directives, "public")
    end
    
    if options.maxAge then
        table.insert(directives, "max-age=" .. options.maxAge)
    end
    
    if options.sMaxAge then
        table.insert(directives, "s-maxage=" .. options.sMaxAge)
    end
    
    if options.mustRevalidate then
        table.insert(directives, "must-revalidate")
    end
    
    if options.proxyRevalidate then
        table.insert(directives, "proxy-revalidate")
    end
    
    if options.immutable then
        table.insert(directives, "immutable")
    end
    
    if options.staleWhileRevalidate then
        table.insert(directives, "stale-while-revalidate=" .. options.staleWhileRevalidate)
    end
    
    if options.staleIfError then
        table.insert(directives, "stale-if-error=" .. options.staleIfError)
    end
    
    return table.concat(directives, ", ")
end

-- ETag generation
function HTTPCache.generateETag(content, weak)
    local hash = 0
    for i = 1, #content do
        hash = ((hash * 31) + string.byte(content, i)) & 0xFFFFFFFF
    end
    local etag = string.format('"%x"', hash)
    return weak and 'W/' .. etag or etag
end

-- Last-Modified header
function HTTPCache.lastModified(timestamp)
    -- RFC 7231 date format
    local days = {"Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"}
    local months = {"Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"}
    
    local t = os.date("*t", timestamp)
    return string.format("%s, %02d %s %04d %02d:%02d:%02d GMT",
        days[t.wday], t.day, months[t.month], t.year,
        t.hour, t.min, t.sec
    )
end

-- Check conditional request
function HTTPCache.checkConditional(request, response)
    -- If-None-Match: ETag comparison
    local ifNoneMatch = request.headers and request.headers["if-none-match"]
    if ifNoneMatch and response.etag then
        if ifNoneMatch == response.etag or ifNoneMatch == "*" then
            return 304, nil  -- Not Modified
        end
    end
    
    -- If-Modified-Since comparison
    local ifModifiedSince = request.headers and request.headers["if-modified-since"]
    if ifModifiedSince and response.lastModifiedTimestamp then
        -- Parse the date (simplified)
        local reqTime = 0  -- Would parse HTTP date
        if response.lastModifiedTimestamp <= reqTime then
            return 304, nil
        end
    end
    
    return 200, response
end

-- Vary header for content negotiation
function HTTPCache.varyHeader(fields)
    return table.concat(fields, ", ")
end

-- Cache rules for different resource types
local cacheRules = {
    -- Static assets (immutable with hash in filename)
    static_immutable = function()
        return HTTPCache.cacheControl({
            public = true,
            maxAge = 31536000,  -- 1 year
            immutable = true
        })
    end,
    
    -- Static assets (no hash, can change)
    static_versioned = function()
        return HTTPCache.cacheControl({
            public = true,
            maxAge = 86400,  -- 1 day
            mustRevalidate = true
        })
    end,
    
    -- API responses (cacheable)
    api_cacheable = function(ttl)
        return HTTPCache.cacheControl({
            public = true,
            maxAge = ttl or 60,
            sMaxAge = ttl and ttl * 2 or 120,
            staleWhileRevalidate = 30,
            staleIfError = 3600
        })
    end,
    
    -- API responses (private user data)
    api_private = function(ttl)
        return HTTPCache.cacheControl({
            private = true,
            maxAge = ttl or 300
        })
    end,
    
    -- No caching
    no_cache = function()
        return HTTPCache.cacheControl({noStore = true})
    end,
    
    -- Always validate
    always_validate = function()
        return HTTPCache.cacheControl({noCache = true})
    end
}

-- Test
print("\n=== HTTP Cache Headers Demo ===")

print("Cache-Control values:")
for name, fn in pairs(cacheRules) do
    print(string.format("  %-20s: %s", name, fn()))
end

print("\nETag examples:")
local content = '{"id":1,"name":"Alice","email":"alice@example.com"}'
local etag = HTTPCache.generateETag(content)
local weakEtag = HTTPCache.generateETag(content, true)
print("  Strong ETag:", etag)
print("  Weak ETag:", weakEtag)

print("\nLast-Modified:")
print("  " .. HTTPCache.lastModified(os.time() - 3600))

print("\nVary headers:")
print("  API (Accept):", HTTPCache.varyHeader({"Accept"}))
print("  CDN (encoding+lang):", HTTPCache.varyHeader({"Accept-Encoding", "Accept-Language"}))
```

## 13. Cache Penetration, Breakdown, Avalanche

```lua
-- Example 13: Cache failure modes and solutions

-- 1. Cache Penetration (cache miss + DB miss)
-- เกิดเมื่อ query keys ที่ไม่มีใน DB ทำให้ทุก request ผ่าน cache ไปถึง DB

local BloomFilter = {}
BloomFilter.__index = BloomFilter

function BloomFilter.new(size, hashCount)
    local self = setmetatable({}, BloomFilter)
    self.size = size or 1000
    self.hashCount = hashCount or 3
    self.bits = {}
    for i = 1, self.size do self.bits[i] = false end
    self.count = 0
    return self
end

function BloomFilter:_hashes(key)
    local hashes = {}
    local h = 5381
    for i = 1, self.hashCount do
        for j = 1, #key do
            h = ((h * 33) ~ string.byte(key, j)) & 0x7FFFFFFF
        end
        h = h ~ (i * 0x9e3779b9)
        table.insert(hashes, (h % self.size) + 1)
    end
    return hashes
end

function BloomFilter:add(key)
    for _, pos in ipairs(self:_hashes(key)) do
        self.bits[pos] = true
    end
    self.count = self.count + 1
end

function BloomFilter:mightContain(key)
    for _, pos in ipairs(self:_hashes(key)) do
        if not self.bits[pos] then return false end
    end
    return true  -- might be true (false positives possible)
end

function BloomFilter:falsePositiveRate()
    local setBits = 0
    for _, b in ipairs(self.bits) do
        if b then setBits = setBits + 1 end
    end
    -- Approximation
    return (setBits / self.size) ^ self.hashCount
end

-- Null cache (cache miss results)
local NullCache = {}
NullCache.__index = NullCache

function NullCache.new(ttl)
    return setmetatable({
        data = {},
        ttl = ttl or 60  -- short TTL for null values
    }, NullCache)
end

function NullCache:setNull(key)
    self.data[key] = {isNull = true, expiresAt = os.time() + self.ttl}
end

function NullCache:isNull(key)
    local entry = self.data[key]
    if not entry then return false end
    if os.time() > entry.expiresAt then
        self.data[key] = nil
        return false
    end
    return entry.isNull
end

-- 2. Cache Breakdown (hot key expires)
-- Hot key expire ทำให้ requests จำนวนมากตีไป DB พร้อมกัน
-- Solution: Mutex lock, warm cache before expiry

local HotKeyProtection = {}

function HotKeyProtection.warmBeforeExpiry(cache, key, loader, renewThreshold)
    renewThreshold = renewThreshold or 30  -- seconds before expiry
    
    local entry = cache.data[key]
    if not entry then return end
    
    local timeLeft = entry.expiresAt - os.time()
    
    if timeLeft <= renewThreshold then
        -- Schedule background renewal
        print(string.format("[HotKey] Key '%s' expiring in %ds, renewing...", key, timeLeft))
        local newValue = loader(key)
        if newValue then
            cache:set(key, newValue, cache.defaultTTL)
        end
    end
end

-- 3. Cache Avalanche (many keys expire at same time)
-- Solution: Jitter TTL, multi-level cache, circuit breaker

local AvalancheProtection = {}

function AvalancheProtection.jitterTTL(baseTTL, jitterRange)
    jitterRange = jitterRange or math.floor(baseTTL * 0.1)
    return baseTTL + math.random(-jitterRange, jitterRange)
end

function AvalancheProtection.multiLevelCache(l1, l2, key)
    -- Check L1 (fast, in-process)
    local v = l1:get(key)
    if v then return v, "l1" end
    
    -- Check L2 (shared/Redis)
    v = l2:get(key)
    if v then
        l1:set(key, v, 30)  -- Short TTL in L1
        return v, "l2"
    end
    
    return nil, "miss"
end

-- 综合 demonstration
print("\n=== Cache Failure Modes ===")

-- 1. Cache Penetration Prevention
print("\n1. Bloom Filter (Penetration Prevention):")
local bf = BloomFilter.new(1000, 4)

-- Pre-populate with valid IDs
for i = 1, 100 do
    bf:add("user:" .. i)
end

local testIds = {"user:50", "user:101", "user:999", "user:1"}
for _, id in ipairs(testIds) do
    local exists = bf:mightContain(id)
    print(string.format("  %s: %s", id, exists and "might exist (check DB)" or "definitely not in DB"))
end
print(string.format("  Bloom filter false positive rate: ~%.1f%%", bf:falsePositiveRate() * 100))

-- Null cache
print("\n  Null Cache (cache miss results):")
local nullCache = NullCache.new(60)
nullCache:setNull("user:99999")
print("  user:99999 is null:", nullCache:isNull("user:99999"))
print("  user:1 is null:", nullCache:isNull("user:1"))

-- 2. Cache Breakdown
print("\n2. Cache Breakdown Prevention:")
print("  Solution: Distributed lock + XFetch early renewal")
print("  XFetch: refresh cache before expiry with probability")
print("  Lock: only one request reloads, others wait for result")

-- 3. Cache Avalanche
print("\n3. Cache Avalanche Prevention:")
local baseCache = L1Cache.new(1000, 0)
print("  Loading 10 items with jittered TTL (base=300s, jitter=±30s):")
for i = 1, 10 do
    local ttl = AvalancheProtection.jitterTTL(300, 30)
    baseCache:set("item:" .. i, {id = i}, ttl)
    print(string.format("    item:%d TTL: %ds", i, ttl))
end

-- Multi-level cache
print("\n  Multi-level cache:")
local l1_test = L1Cache.new(100, 30)
local l2_test = L1Cache.new(10000, 300)

l2_test:set("config:theme", "dark", 300)
local v, level = AvalancheProtection.multiLevelCache(l1_test, l2_test, "config:theme")
print(string.format("    First get from level: %s, value: %s", level, tostring(v)))
v, level = AvalancheProtection.multiLevelCache(l1_test, l2_test, "config:theme")
print(string.format("    Second get from level: %s, value: %s", level, tostring(v)))
```

## 14. Cache Keys Design

```lua
-- Example 14: Cache key design best practices
local CacheKeyBuilder = {}
CacheKeyBuilder.__index = CacheKeyBuilder

function CacheKeyBuilder.new(prefix, separator)
    return setmetatable({
        prefix = prefix or "app",
        sep = separator or ":"
    }, CacheKeyBuilder)
end

function CacheKeyBuilder:build(...)
    local parts = {self.prefix}
    for _, part in ipairs({...}) do
        if part ~= nil then
            table.insert(parts, tostring(part))
        end
    end
    return table.concat(parts, self.sep)
end

-- Version-aware keys
function CacheKeyBuilder:versioned(version, ...)
    local parts = {self.prefix, "v" .. version}
    for _, p in ipairs({...}) do
        table.insert(parts, tostring(p))
    end
    return table.concat(parts, self.sep)
end

-- Hash key (for long keys)
function CacheKeyBuilder:hash(...)
    local key = self:build(...)
    if #key > 128 then
        -- Hash the key
        local h = 5381
        for i = 1, #key do
            h = ((h * 33) ~ string.byte(key, i)) & 0x7FFFFFFF
        end
        return self.prefix .. self.sep .. string.format("%x", h)
    end
    return key
end

-- Key with locale
function CacheKeyBuilder:localized(locale, ...)
    return self:build(locale, ...)
end

-- Parameterized key (for query results)
local function hashParams(params)
    if type(params) ~= "table" then return tostring(params) end
    
    -- Sort keys for consistency
    local keys = {}
    for k in pairs(params) do table.insert(keys, k) end
    table.sort(keys)
    
    local parts = {}
    for _, k in ipairs(keys) do
        table.insert(parts, k .. "=" .. tostring(params[k]))
    end
    return table.concat(parts, "&")
end

function CacheKeyBuilder:query(entity, params)
    local paramStr = hashParams(params)
    return self:build(entity, "query", paramStr)
end

-- Tag-aware key builder
function CacheKeyBuilder:withTags(key, tags)
    return {
        key = key,
        tags = tags,
        build = function(self) return key end
    }
end

local kb = CacheKeyBuilder.new("myapp")

print("\n=== Cache Key Design ===")
print("Key examples:")
print("  User profile:", kb:build("user", 123, "profile"))
print("  User orders:", kb:build("user", 123, "orders", "page", 1))
print("  Product:", kb:build("product", "PRD-456"))
print("  Config:", kb:build("config", "features"))
print("  Versioned:", kb:versioned(2, "user", 123))
print("  Localized:", kb:localized("th", "i18n", "home_title"))
print("  Query:", kb:query("products", {category = "electronics", sort = "price", page = 1}))

print("\nKey naming conventions:")
local examples = {
    "app:user:{id}",
    "app:user:{id}:session",
    "app:product:{id}:detail",
    "app:category:{slug}:products",
    "app:search:{query_hash}",
    "app:config:{key}",
    "app:counter:{entity}:{id}",
    "app:lock:{resource}",
    "app:ratelimit:{ip}:{window}",
}
for _, pattern in ipairs(examples) do
    print("  " .. pattern)
end
```

## สรุป Caching Strategies

Caching เป็นหัวใจสำคัญของระบบ high-performance:

**Cache Patterns:**
1. **Cache-Aside** (Lazy Loading) - Application จัดการ cache เอง, ง่ายสุด
2. **Write-Through** - เขียนพร้อมกันทั้ง cache และ DB, consistent
3. **Write-Behind** - เขียน cache ก่อน, flush DB ทีหลัง, fast writes
4. **Read-Through** - Cache จัดการ load เอง, transparent

**Cache Invalidation:**
- Tag-based, Version-based, Dependency-based
- Stale-While-Revalidate, Stale-If-Error
- Cascade invalidation

**TTL Strategies:**
- Static TTL, Dynamic TTL
- Jittered TTL (ป้องกัน avalanche)
- Sliding TTL (ต่ออายุเมื่อใช้งาน)

**Failure Modes:**
- **Cache Penetration** → Bloom Filter, Null Cache
- **Cache Breakdown** → Mutex Lock, Early Renewal (XFetch)
- **Cache Avalanche** → Jitter TTL, Multi-level Cache, Circuit Breaker

**OpenResty/NGINX:**
- `ngx.shared.DICT` สำหรับ shared memory cache
- `ngx.ctx` สำหรับ per-request cache
- `ngx.var` สำหรับ NGINX variable caching

**HTTP Caching:**
- `Cache-Control`: max-age, s-maxage, public/private, immutable
- `ETag`: content hash validation
- `Last-Modified`: timestamp validation
- `Vary`: content negotiation

**Distributed Cache:**
- Redis/Memcached สำหรับ multi-instance
- Consistent Hashing สำหรับ cache cluster
- Pipeline/Batch สำหรับ efficiency
