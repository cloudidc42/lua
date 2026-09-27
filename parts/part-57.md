# บทที่ 57: Redis + Lua Scripting

## บทนำ

Redis เป็น in-memory data structure store ที่รวดเร็วมาก Lua สามารถใช้งาน Redis ได้ 2 วิธี:

1. **Lua client** - เชื่อมต่อ Redis จาก Lua (lua-resty-redis, redis-lua)
2. **Redis Lua scripts** - รัน Lua scripts ภายใน Redis ด้วย EVAL command

---

## ตัวอย่างที่ 1: Redis Data Structures Overview

```lua
-- Redis data structures และการใช้งาน (conceptual)
local redis_structures = {
    -- STRING: ค่าเดี่ยว, counters, cached objects
    string_ops = {
        "SET key value",
        "GET key",
        "INCR counter",
        "INCRBY counter 5",
        "SETEX key 3600 value",  -- expire in 3600s
        "SETNX key value",       -- set if not exists
        "MSET k1 v1 k2 v2",
        "MGET k1 k2",
    },
    
    -- HASH: objects/records
    hash_ops = {
        "HSET user:1 name Alice email alice@example.com",
        "HGET user:1 name",
        "HMGET user:1 name email",
        "HGETALL user:1",
        "HINCRBY user:1 login_count 1",
        "HDEL user:1 field",
        "HEXISTS user:1 email",
    },
    
    -- LIST: queues, recent items
    list_ops = {
        "LPUSH queue task1 task2",  -- push left
        "RPUSH queue task3",        -- push right
        "LPOP queue",               -- pop left
        "RPOP queue",               -- pop right
        "LRANGE queue 0 -1",        -- get all
        "LLEN queue",               -- length
        "BRPOP queue 30",           -- blocking pop with timeout
    },
    
    -- SET: unique collections, tags
    set_ops = {
        "SADD tags:post:1 lua programming redis",
        "SMEMBERS tags:post:1",
        "SISMEMBER tags:post:1 lua",
        "SCARD tags:post:1",
        "SUNION tags:post:1 tags:post:2",
        "SINTER tags:post:1 tags:post:2",
    },
    
    -- SORTED SET: leaderboards, priority queues
    zset_ops = {
        "ZADD leaderboard 1000 player:1",
        "ZADD leaderboard 2500 player:2",
        "ZRANK leaderboard player:1",
        "ZREVRANK leaderboard player:1",
        "ZRANGE leaderboard 0 9 WITHSCORES",
        "ZINCRBY leaderboard 100 player:1",
    },
}

print("=== Redis Data Structures ===")
for structure, ops in pairs(redis_structures) do
    print("\n" .. structure:upper():gsub("_OPS", "") .. ":")
    for _, op in ipairs(ops) do
        print("  " .. op)
    end
end
```

---

## ตัวอย่างที่ 2: lua-resty-redis Connection (OpenResty)

```lua
-- การเชื่อมต่อ Redis ด้วย lua-resty-redis (OpenResty)
-- ไฟล์นี้ใช้สำหรับ Nginx/OpenResty context

local function create_redis_client(config)
    config = config or {}
    
    -- ตัวอย่าง mock Redis client
    local redis_mock = {
        host    = config.host or "127.0.0.1",
        port    = config.port or 6379,
        timeout = config.timeout or 1000,  -- ms
        pool_size = config.pool_size or 100,
        connected = false,
        data = {},  -- in-memory mock storage
    }
    
    -- Methods
    function redis_mock:connect()
        -- ในระบบจริง:
        -- local redis = require "resty.redis"
        -- local red = redis:new()
        -- red:set_timeouts(1000, 1000, 1000)
        -- local ok, err = red:connect(self.host, self.port)
        self.connected = true
        print(string.format("Connected to Redis %s:%d", self.host, self.port))
        return true, nil
    end
    
    function redis_mock:set(key, value, expire_seconds)
        if not self.connected then return nil, "Not connected" end
        self.data[key] = {
            value = value,
            expires_at = expire_seconds and (os.time() + expire_seconds) or nil,
        }
        return "OK"
    end
    
    function redis_mock:get(key)
        if not self.connected then return nil, "Not connected" end
        local entry = self.data[key]
        if not entry then return false end  -- Redis returns false for nil
        if entry.expires_at and os.time() > entry.expires_at then
            self.data[key] = nil
            return false
        end
        return entry.value
    end
    
    function redis_mock:del(...)
        local count = 0
        for _, key in ipairs({...}) do
            if self.data[key] then
                self.data[key] = nil
                count = count + 1
            end
        end
        return count
    end
    
    function redis_mock:incr(key)
        local entry = self.data[key]
        local val = entry and tonumber(entry.value) or 0
        val = val + 1
        self.data[key] = { value = tostring(val) }
        return val
    end
    
    function redis_mock:incrby(key, amount)
        local entry = self.data[key]
        local val = entry and tonumber(entry.value) or 0
        val = val + amount
        self.data[key] = { value = tostring(val) }
        return val
    end
    
    function redis_mock:expire(key, seconds)
        local entry = self.data[key]
        if not entry then return 0 end
        entry.expires_at = os.time() + seconds
        return 1
    end
    
    function redis_mock:ttl(key)
        local entry = self.data[key]
        if not entry then return -2 end
        if not entry.expires_at then return -1 end
        local remaining = entry.expires_at - os.time()
        return remaining > 0 and remaining or -2
    end
    
    function redis_mock:set_keepalive(max_idle_timeout, pool_size)
        -- ส่ง connection กลับ pool ในระบบจริง
        print(string.format("Connection returned to pool (idle: %dms, size: %d)",
            max_idle_timeout or 10000, pool_size or 100))
        self.connected = false
    end
    
    return redis_mock
end

-- ทดสอบ
local redis = create_redis_client({
    host = "127.0.0.1",
    port = 6379,
})

redis:connect()

-- Basic operations
redis:set("greeting", "สวัสดี Redis!")
local val = redis:get("greeting")
print("GET greeting:", val)

redis:set("counter", "0")
redis:incr("counter")
redis:incr("counter")
redis:incrby("counter", 5)
print("Counter:", redis:get("counter"))

-- Set with expiry
redis:set("temp_token", "abc123", 60)
print("TTL temp_token:", redis:ttl("temp_token"), "seconds")
print("TTL nonexistent:", redis:ttl("nonexistent"))

-- Delete
local deleted = redis:del("greeting", "counter")
print("Deleted:", deleted, "keys")
print("After delete:", redis:get("greeting"))

redis:set_keepalive(10000, 100)
```

---

## ตัวอย่างที่ 3: Redis Hash Operations

```lua
-- Redis Hash operations simulation
local RedisHash = {}
local hash_store = {}

function RedisHash.hset(key, field, value)
    if not hash_store[key] then
        hash_store[key] = {}
    end
    local is_new = hash_store[key][field] == nil and 1 or 0
    hash_store[key][field] = tostring(value)
    return is_new
end

function RedisHash.hmset(key, ...)
    local args = {...}
    if not hash_store[key] then hash_store[key] = {} end
    for i = 1, #args, 2 do
        hash_store[key][args[i]] = tostring(args[i+1])
    end
    return "OK"
end

function RedisHash.hget(key, field)
    local hash = hash_store[key]
    if not hash then return false end
    return hash[field] or false
end

function RedisHash.hmget(key, ...)
    local hash = hash_store[key]
    local result = {}
    for _, field in ipairs({...}) do
        table.insert(result, hash and hash[field] or false)
    end
    return result
end

function RedisHash.hgetall(key)
    local hash = hash_store[key]
    if not hash then return {} end
    local result = {}
    for field, value in pairs(hash) do
        table.insert(result, field)
        table.insert(result, value)
    end
    return result
end

function RedisHash.hincrby(key, field, amount)
    if not hash_store[key] then hash_store[key] = {} end
    local val = tonumber(hash_store[key][field]) or 0
    val = val + amount
    hash_store[key][field] = tostring(val)
    return val
end

function RedisHash.hdel(key, ...)
    local hash = hash_store[key]
    if not hash then return 0 end
    local count = 0
    for _, field in ipairs({...}) do
        if hash[field] then
            hash[field] = nil
            count = count + 1
        end
    end
    return count
end

function RedisHash.hexists(key, field)
    local hash = hash_store[key]
    return hash and hash[field] ~= nil and 1 or 0
end

function RedisHash.hlen(key)
    local hash = hash_store[key]
    if not hash then return 0 end
    local count = 0
    for _ in pairs(hash) do count = count + 1 end
    return count
end

-- ทดสอบ - User profile
print("=== Redis Hash Operations ===")

-- เก็บ user profile
RedisHash.hmset("user:1",
    "name", "Alice",
    "email", "alice@example.com",
    "age", "30",
    "role", "admin",
    "login_count", "0"
)

print("User name:", RedisHash.hget("user:1", "name"))
print("User email:", RedisHash.hget("user:1", "email"))

-- Get multiple fields
local fields = RedisHash.hmget("user:1", "name", "role", "nonexistent")
print("Multiple fields:", fields[1], fields[2], tostring(fields[3]))

-- Increment
RedisHash.hincrby("user:1", "login_count", 1)
RedisHash.hincrby("user:1", "login_count", 1)
print("Login count:", RedisHash.hget("user:1", "login_count"))

-- Get all
local all = RedisHash.hgetall("user:1")
print("\nAll user:1 fields:")
for i = 1, #all, 2 do
    print(string.format("  %s = %s", all[i], all[i+1]))
end

print("Field count:", RedisHash.hlen("user:1"))
print("Email exists:", RedisHash.hexists("user:1", "email"))
print("Phone exists:", RedisHash.hexists("user:1", "phone"))
```

---

## ตัวอย่างที่ 4: Redis List - Queue Operations

```lua
-- Redis List สำหรับทำ Queue
local RedisList = {}
local list_store = {}

function RedisList.lpush(key, ...)
    if not list_store[key] then list_store[key] = {} end
    for _, val in ipairs({...}) do
        table.insert(list_store[key], 1, tostring(val))
    end
    return #list_store[key]
end

function RedisList.rpush(key, ...)
    if not list_store[key] then list_store[key] = {} end
    for _, val in ipairs({...}) do
        table.insert(list_store[key], tostring(val))
    end
    return #list_store[key]
end

function RedisList.lpop(key)
    local list = list_store[key]
    if not list or #list == 0 then return false end
    return table.remove(list, 1)
end

function RedisList.rpop(key)
    local list = list_store[key]
    if not list or #list == 0 then return false end
    return table.remove(list)
end

function RedisList.lrange(key, start_idx, stop_idx)
    local list = list_store[key] or {}
    local len = #list
    
    -- Convert negative indices
    if start_idx < 0 then start_idx = len + start_idx + 1 end
    if stop_idx < 0 then stop_idx = len + stop_idx + 1 end
    
    start_idx = math.max(1, start_idx + 1)  -- Lua is 1-based
    stop_idx  = math.min(len, stop_idx + 1)
    
    local result = {}
    for i = start_idx, stop_idx do
        table.insert(result, list[i])
    end
    return result
end

function RedisList.llen(key)
    return #(list_store[key] or {})
end

function RedisList.lindex(key, index)
    local list = list_store[key]
    if not list then return false end
    if index < 0 then index = #list + index end
    return list[index + 1] or false
end

-- Simulate FIFO Queue
local function create_queue(name)
    return {
        name = name,
        enqueue = function(self, item)
            return RedisList.rpush(self.name, item)
        end,
        dequeue = function(self)
            return RedisList.lpop(self.name)
        end,
        peek = function(self)
            return RedisList.lindex(self.name, 0)
        end,
        size = function(self)
            return RedisList.llen(self.name)
        end,
        items = function(self)
            return RedisList.lrange(self.name, 0, -1)
        end,
    }
end

-- ทดสอบ Queue
print("=== Redis List Queue ===")

local queue = create_queue("task:queue")

-- Enqueue tasks
queue:enqueue('{"task":"send_email","to":"alice@example.com"}')
queue:enqueue('{"task":"resize_image","file":"photo.jpg"}')
queue:enqueue('{"task":"generate_report","type":"monthly"}')
queue:enqueue('{"task":"cleanup_temp_files"}')

print("Queue size:", queue:size())
print("Peek:", queue:peek())
print()

-- Dequeue and process
print("Processing tasks:")
while queue:size() > 0 do
    local task = queue:dequeue()
    print("  Processing:", task)
end

print("Queue empty:", queue:size() == 0)

-- Stack (LIFO) ใช้ LPUSH + LPOP
print("\n=== Redis List Stack ===")
local stack_key = "my:stack"
RedisList.lpush(stack_key, "first", "second", "third")
print("Stack after push:", table.concat(RedisList.lrange(stack_key, 0, -1), ", "))
print("Pop:", RedisList.lpop(stack_key))
print("Pop:", RedisList.lpop(stack_key))
```

---

## ตัวอย่างที่ 5: Redis Sorted Set - Leaderboard

```lua
-- Redis Sorted Set สำหรับทำ Leaderboard
local RedisSortedSet = {}
local zset_store = {}

local function get_or_create(key)
    if not zset_store[key] then
        zset_store[key] = {} -- { member -> score }
    end
    return zset_store[key]
end

function RedisSortedSet.zadd(key, score, member)
    local zset = get_or_create(key)
    local is_new = zset[member] == nil and 1 or 0
    zset[member] = tonumber(score)
    return is_new
end

function RedisSortedSet.zincrby(key, amount, member)
    local zset = get_or_create(key)
    zset[member] = (zset[member] or 0) + amount
    return zset[member]
end

function RedisSortedSet.zscore(key, member)
    local zset = zset_store[key]
    if not zset then return false end
    return zset[member] or false
end

-- Get sorted members (ascending)
local function get_sorted(key)
    local zset = zset_store[key] or {}
    local list = {}
    for member, score in pairs(zset) do
        table.insert(list, { member = member, score = score })
    end
    table.sort(list, function(a, b) return a.score < b.score end)
    return list
end

function RedisSortedSet.zrange(key, start_idx, stop_idx, withscores)
    local sorted = get_sorted(key)
    local len = #sorted
    if start_idx < 0 then start_idx = len + start_idx end
    if stop_idx < 0 then stop_idx = len + stop_idx end
    
    local result = {}
    for i = start_idx + 1, math.min(stop_idx + 1, len) do
        table.insert(result, sorted[i].member)
        if withscores then
            table.insert(result, sorted[i].score)
        end
    end
    return result
end

function RedisSortedSet.zrevrange(key, start_idx, stop_idx, withscores)
    local sorted = get_sorted(key)
    -- reverse
    local rev = {}
    for i = #sorted, 1, -1 do table.insert(rev, sorted[i]) end
    
    local len = #rev
    if start_idx < 0 then start_idx = len + start_idx end
    if stop_idx < 0 then stop_idx = len + stop_idx end
    
    local result = {}
    for i = start_idx + 1, math.min(stop_idx + 1, len) do
        table.insert(result, rev[i].member)
        if withscores then
            table.insert(result, rev[i].score)
        end
    end
    return result
end

function RedisSortedSet.zrank(key, member)
    local sorted = get_sorted(key)
    for i, item in ipairs(sorted) do
        if item.member == member then return i - 1 end
    end
    return false
end

function RedisSortedSet.zrevrank(key, member)
    local sorted = get_sorted(key)
    for i = #sorted, 1, -1 do
        if sorted[i].member == member then
            return #sorted - i
        end
    end
    return false
end

function RedisSortedSet.zcard(key)
    local zset = zset_store[key]
    if not zset then return 0 end
    local count = 0
    for _ in pairs(zset) do count = count + 1 end
    return count
end

-- Game Leaderboard
print("=== Redis Sorted Set Leaderboard ===")

local LB = "game:leaderboard"

-- เพิ่ม players
local players = {
    { name = "alice",   score = 8500  },
    { name = "bob",     score = 12000 },
    { name = "charlie", score = 9750  },
    { name = "diana",   score = 15000 },
    { name = "eve",     score = 7200  },
    { name = "frank",   score = 11500 },
}

for _, p in ipairs(players) do
    RedisSortedSet.zadd(LB, p.score, p.name)
end

-- Top 3 players
print("Top 3 Players:")
local top3 = RedisSortedSet.zrevrange(LB, 0, 2, true)
for i = 1, #top3, 2 do
    local rank = (i - 1) / 2 + 1
    print(string.format("  #%d %s: %s points", rank, top3[i], top3[i+1]))
end

-- Player rank
print("\nPlayer Rankings:")
local players_to_check = { "alice", "bob", "diana" }
for _, name in ipairs(players_to_check) do
    local rank = RedisSortedSet.zrevrank(LB, name)
    local score = RedisSortedSet.zscore(LB, name)
    print(string.format("  %s: rank #%d, score %s", name, rank + 1, tostring(score)))
end

-- Update score
print("\nAlice earns 500 more points:")
RedisSortedSet.zincrby(LB, 500, "alice")
print("Alice's new score:", RedisSortedSet.zscore(LB, "alice"))
print("Alice's new rank:", (RedisSortedSet.zrevrank(LB, "alice") or -1) + 1)
```

---

## ตัวอย่างที่ 6: Redis Pub/Sub

```lua
-- Redis Pub/Sub implementation
local PubSub = {}

local subscribers = {}  -- channel -> [callbacks]
local pattern_subs = {} -- pattern -> [callbacks]

-- Subscribe to channel
function PubSub.subscribe(channel, callback)
    if not subscribers[channel] then
        subscribers[channel] = {}
    end
    table.insert(subscribers[channel], callback)
    print(string.format("Subscribed to channel: %s (total: %d)", channel, #subscribers[channel]))
    return #subscribers[channel]
end

-- Unsubscribe
function PubSub.unsubscribe(channel, callback)
    if not subscribers[channel] then return 0 end
    if callback then
        for i, cb in ipairs(subscribers[channel]) do
            if cb == callback then
                table.remove(subscribers[channel], i)
                break
            end
        end
    else
        subscribers[channel] = {}
    end
    return #(subscribers[channel] or {})
end

-- Pattern subscribe
function PubSub.psubscribe(pattern, callback)
    if not pattern_subs[pattern] then
        pattern_subs[pattern] = {}
    end
    table.insert(pattern_subs[pattern], callback)
    print("Pattern subscribed:", pattern)
end

-- Publish message
function PubSub.publish(channel, message)
    local count = 0
    
    -- Direct subscribers
    local direct = subscribers[channel]
    if direct then
        for _, cb in ipairs(direct) do
            cb(channel, message)
            count = count + 1
        end
    end
    
    -- Pattern subscribers
    for pattern, callbacks in pairs(pattern_subs) do
        -- Convert Redis glob pattern to Lua pattern
        local lua_pattern = pattern:gsub("%*", ".*"):gsub("%?", ".")
        if channel:match("^" .. lua_pattern .. "$") then
            for _, cb in ipairs(callbacks) do
                cb(pattern, channel, message)
                count = count + 1
            end
        end
    end
    
    return count
end

-- ทดสอบ
print("=== Redis Pub/Sub ===")

-- Subscribe handlers
local received_messages = {}

PubSub.subscribe("notifications:user:1", function(channel, msg)
    table.insert(received_messages, { channel = channel, msg = msg })
    print("User 1 notification:", msg)
end)

PubSub.subscribe("system:alerts", function(channel, msg)
    print("ALERT [" .. channel .. "]:", msg)
end)

-- Pattern subscribe
PubSub.psubscribe("notifications:*", function(pattern, channel, msg)
    print(string.format("Pattern match (%s) on %s: %s", pattern, channel, msg))
end)

-- Publish
print()
local count1 = PubSub.publish("notifications:user:1", '{"type":"message","from":"bob","text":"สวัสดี!"}')
print("Receivers:", count1)

local count2 = PubSub.publish("system:alerts", "High memory usage: 95%")
print("Alert receivers:", count2)

local count3 = PubSub.publish("notifications:user:2", "You have a new follower")
print("User 2 receivers:", count3)

print("\nTotal messages received:", #received_messages)
```

---

## ตัวอย่างที่ 7: EVAL Command - Lua Scripts in Redis

```lua
-- Redis EVAL - รัน Lua script ภายใน Redis
-- นี่คือตัวอย่างของ scripts ที่จะรันใน Redis (server-side Lua)

local redis_scripts = {}

-- Script 1: Atomic GET and SET
redis_scripts.atomic_get_set = [[
-- KEYS[1] = key
-- ARGV[1] = new value
-- Returns old value
local old_val = redis.call('GET', KEYS[1])
redis.call('SET', KEYS[1], ARGV[1])
return old_val
]]

-- Script 2: Rate limiting
redis_scripts.rate_limit = [[
-- KEYS[1] = rate limit key (e.g., "ratelimit:ip:127.0.0.1")
-- ARGV[1] = max requests
-- ARGV[2] = window in seconds
-- Returns { current_count, is_allowed }

local key     = KEYS[1]
local max_req = tonumber(ARGV[1])
local window  = tonumber(ARGV[2])

local current = redis.call('GET', key)
current = current and tonumber(current) or 0

if current >= max_req then
    return {current, 0}  -- blocked
end

if current == 0 then
    redis.call('SET', key, 1)
    redis.call('EXPIRE', key, window)
else
    redis.call('INCR', key)
end

return {current + 1, 1}  -- allowed
]]

-- Script 3: Distributed lock (SET NX PX)
redis_scripts.acquire_lock = [[
-- KEYS[1] = lock key
-- ARGV[1] = lock value (unique identifier)
-- ARGV[2] = TTL in milliseconds
-- Returns 1 if acquired, 0 if not

local result = redis.call('SET', KEYS[1], ARGV[1], 'NX', 'PX', tonumber(ARGV[2]))
if result then
    return 1
else
    return 0
end
]]

-- Script 4: Release lock (only if owner)
redis_scripts.release_lock = [[
-- KEYS[1] = lock key
-- ARGV[1] = lock value (must match)
-- Returns 1 if released, 0 if not owner

local current = redis.call('GET', KEYS[1])
if current == ARGV[1] then
    redis.call('DEL', KEYS[1])
    return 1
else
    return 0
end
]]

-- Script 5: Atomic increment with limit
redis_scripts.increment_with_limit = [[
-- KEYS[1] = counter key
-- ARGV[1] = limit
-- ARGV[2] = expire seconds
-- Returns {new_value, is_over_limit}

local key    = KEYS[1]
local limit  = tonumber(ARGV[1])
local expire = tonumber(ARGV[2])

local val = redis.call('INCR', key)
if val == 1 and expire > 0 then
    redis.call('EXPIRE', key, expire)
end

return {val, val > limit and 1 or 0}
]]

-- Simulate EVAL execution
local function simulate_eval(script_name, keys, args, storage)
    print(string.format("\nEVAL script: %s", script_name))
    print("KEYS:", table.concat(keys, ", "))
    print("ARGS:", table.concat(args, ", "))
    
    -- Script simulation
    if script_name == "rate_limit" then
        local key = keys[1]
        local max_req = tonumber(args[1])
        local window = tonumber(args[2])
        
        local current = storage[key] and tonumber(storage[key].value) or 0
        if current >= max_req then
            return { current, 0 }
        end
        
        storage[key] = { value = tostring(current + 1), expires = os.time() + window }
        return { current + 1, 1 }
    end
    
    return nil
end

print("=== Redis EVAL Scripts ===")
print("\nScript: rate_limit")
print(redis_scripts.rate_limit)

-- Simulate running rate_limit script
local storage = {}
for i = 1, 7 do
    local result = simulate_eval("rate_limit",
        { "ratelimit:ip:192.168.1.1" },
        { "5", "60" },
        storage
    )
    print(string.format("Request %d: count=%s, allowed=%s",
        i, result[1], result[2] == 1 and "YES" or "NO"))
end
```

---

## ตัวอย่างที่ 8: EVALSHA - Caching Scripts

```lua
-- EVALSHA สำหรับ cache scripts ด้วย SHA1
local ScriptCache = {}

-- SHA1 simulation (ในระบบจริงใช้ OpenSSL)
local function sha1_mock(script)
    local hash = 0
    for i = 1, #script do
        hash = ((hash * 31) + string.byte(script, i)) % (2^31)
    end
    return string.format("%040x", hash)
end

local script_store = {} -- sha -> script

-- SCRIPT LOAD: โหลด script และได้ SHA1 กลับมา
function ScriptCache.script_load(script)
    local sha = sha1_mock(script)
    script_store[sha] = script
    return sha
end

-- EVALSHA: รัน script โดยใช้ SHA1
function ScriptCache.evalsha(sha, keys, args)
    local script = script_store[sha]
    if not script then
        return nil, "NOSCRIPT No matching script"
    end
    -- ในระบบจริง Redis จะรัน script นี้
    return "OK (simulated)", nil
end

-- ตรวจสอบว่า script มีใน cache ไหม
function ScriptCache.script_exists(sha)
    return script_store[sha] ~= nil and 1 or 0
end

-- SCRIPT FLUSH: ลบ scripts ทั้งหมด
function ScriptCache.script_flush()
    script_store = {}
    return "OK"
end

-- Rate limiting script
local RATE_LIMIT_SCRIPT = [[
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current = tonumber(redis.call('GET', key) or '0')
if current >= limit then
    return {current, 0}
end
if current == 0 then
    redis.call('SET', key, 1, 'EX', window)
else
    redis.call('INCR', key)
end
return {current + 1, 1}
]]

-- Leaderboard update script
local LEADERBOARD_SCRIPT = [[
local board = KEYS[1]
local player = ARGV[1]
local points = tonumber(ARGV[2])
redis.call('ZINCRBY', board, points, player)
local rank = redis.call('ZREVRANK', board, player)
local score = redis.call('ZSCORE', board, player)
return {rank + 1, score}
]]

-- ทดสอบ EVALSHA
print("=== EVALSHA Script Caching ===")

-- โหลด scripts
local rate_sha = ScriptCache.script_load(RATE_LIMIT_SCRIPT)
local lb_sha   = ScriptCache.script_load(LEADERBOARD_SCRIPT)

print("Rate limit SHA:", rate_sha:sub(1, 12) .. "...")
print("Leaderboard SHA:", lb_sha:sub(1, 12) .. "...")

-- ตรวจสอบว่ามีใน cache
print("Rate limit exists:", ScriptCache.script_exists(rate_sha))
print("Unknown SHA exists:", ScriptCache.script_exists("unknown_sha"))

-- Run via EVALSHA
local ok, err = ScriptCache.evalsha(rate_sha, {"ratelimit:test"}, {"10", "60"})
print("EVALSHA result:", ok, err)

-- ลอง SHA ที่ไม่มี
local ok2, err2 = ScriptCache.evalsha("nonexistent_sha", {}, {})
print("Bad SHA result:", ok2, err2)

-- Flush and verify
ScriptCache.script_flush()
print("After flush:", ScriptCache.script_exists(rate_sha))

-- Reload
rate_sha = ScriptCache.script_load(RATE_LIMIT_SCRIPT)
print("After reload:", ScriptCache.script_exists(rate_sha))

print("\nBenefit of EVALSHA:")
print("- ไม่ต้องส่ง script ทั้งหมดทุกครั้ง (bandwidth savings)")
print("- Redis cache script bytecode (performance)")
print("- ใช้ SHA เป็น identifier ที่ consistent")
```

---

## ตัวอย่างที่ 9: Atomic Operations with Lua

```lua
-- Atomic operations using Redis Lua scripts
local AtomicOps = {}
local mock_storage = {}

-- Atomic Compare-and-Swap
-- Script: if current value == expected, set new value
AtomicOps.cas_script = [[
local key = KEYS[1]
local expected = ARGV[1]
local new_val  = ARGV[2]
local current = redis.call('GET', key)
if current == expected then
    redis.call('SET', key, new_val)
    return 1  -- success
end
return 0  -- failed (value changed)
]]

function AtomicOps.cas(key, expected, new_val)
    local current = mock_storage[key]
    if current == expected then
        mock_storage[key] = new_val
        return 1
    end
    return 0
end

-- Atomic Pop N items from list
AtomicOps.pop_n_script = [[
local key = KEYS[1]
local n = tonumber(ARGV[1])
local result = {}
for i = 1, n do
    local item = redis.call('LPOP', key)
    if not item then break end
    result[i] = item
end
return result
]]

local list_store = {}
function AtomicOps.pop_n(key, n)
    local list = list_store[key] or {}
    local result = {}
    for _ = 1, n do
        if #list == 0 then break end
        table.insert(result, table.remove(list, 1))
    end
    list_store[key] = list
    return result
end

-- Atomic transfer between keys
AtomicOps.transfer_script = [[
local from_key = KEYS[1]
local to_key   = KEYS[2]
local amount   = tonumber(ARGV[1])

local from_val = tonumber(redis.call('GET', from_key) or '0')
if from_val < amount then
    return {0, from_val}  -- insufficient funds
end

redis.call('DECRBY', from_key, amount)
redis.call('INCRBY', to_key, amount)
return {1, from_val - amount}
]]

function AtomicOps.transfer(from_key, to_key, amount)
    local from_val = tonumber(mock_storage[from_key] or "0")
    if from_val < amount then
        return { 0, from_val }
    end
    mock_storage[from_key] = tostring(from_val - amount)
    mock_storage[to_key]   = tostring(tonumber(mock_storage[to_key] or "0") + amount)
    return { 1, from_val - amount }
end

-- ทดสอบ
print("=== Atomic Operations ===")

-- CAS test
mock_storage["version"] = "v1"
print("CAS v1 -> v2:", AtomicOps.cas("version", "v1", "v2"))
print("Current:", mock_storage["version"])
print("CAS v1 -> v3 (should fail):", AtomicOps.cas("version", "v1", "v3"))
print("Current:", mock_storage["version"])

-- Pop N
list_store["jobs"] = {"job1", "job2", "job3", "job4", "job5"}
local popped = AtomicOps.pop_n("jobs", 3)
print("\nPopped 3 jobs:", table.concat(popped, ", "))
print("Remaining:", #(list_store["jobs"] or {}))

-- Transfer
mock_storage["account:alice"] = "1000"
mock_storage["account:bob"]   = "500"
print("\nBefore transfer: alice=" .. mock_storage["account:alice"] ..
      ", bob=" .. mock_storage["account:bob"])

local result = AtomicOps.transfer("account:alice", "account:bob", 200)
print("Transfer 200: success=" .. result[1])
print("After transfer: alice=" .. mock_storage["account:alice"] ..
      ", bob=" .. mock_storage["account:bob"])

-- ลอง transfer มากกว่าที่มี
local result2 = AtomicOps.transfer("account:alice", "account:bob", 10000)
print("\nTransfer 10000 from alice (has 800): success=" .. result2[1])
print("Alice balance unchanged:", mock_storage["account:alice"])
```

---

## ตัวอย่างที่ 10: Rate Limiting with Redis + Lua

```lua
-- Advanced Rate Limiting patterns
local RateLimit = {}
local rl_storage = {}

-- Fixed Window Rate Limiter
function RateLimit.fixed_window(key, max_requests, window_seconds)
    local window_key = key .. ":" .. math.floor(os.time() / window_seconds)
    
    local entry = rl_storage[window_key]
    if not entry then
        entry = { count = 0, expires = os.time() + window_seconds }
        rl_storage[window_key] = entry
    end
    
    -- Cleanup expired
    if os.time() > entry.expires then
        entry = { count = 0, expires = os.time() + window_seconds }
        rl_storage[window_key] = entry
    end
    
    entry.count = entry.count + 1
    
    if entry.count > max_requests then
        return {
            allowed = false,
            count   = entry.count,
            reset   = entry.expires,
            retry_after = entry.expires - os.time(),
        }
    end
    
    return {
        allowed = true,
        count   = entry.count,
        remaining = max_requests - entry.count,
        reset   = entry.expires,
    }
end

-- Sliding Window Rate Limiter
local sliding_store = {}

function RateLimit.sliding_window(key, max_requests, window_seconds)
    local now = os.time()
    local window_start = now - window_seconds
    
    if not sliding_store[key] then
        sliding_store[key] = {}
    end
    
    local timestamps = sliding_store[key]
    
    -- Remove old timestamps
    local new_timestamps = {}
    for _, ts in ipairs(timestamps) do
        if ts > window_start then
            table.insert(new_timestamps, ts)
        end
    end
    
    local count = #new_timestamps
    
    if count >= max_requests then
        local oldest = new_timestamps[1]
        return {
            allowed = false,
            count   = count,
            retry_after = oldest + window_seconds - now,
        }
    end
    
    table.insert(new_timestamps, now)
    sliding_store[key] = new_timestamps
    
    return {
        allowed   = true,
        count     = count + 1,
        remaining = max_requests - count - 1,
    }
end

-- Token Bucket Rate Limiter
local token_buckets = {}

function RateLimit.token_bucket(key, capacity, refill_rate)
    local now = os.time()
    
    if not token_buckets[key] then
        token_buckets[key] = {
            tokens     = capacity,
            last_refill = now,
        }
    end
    
    local bucket = token_buckets[key]
    
    -- Refill tokens
    local elapsed = now - bucket.last_refill
    local new_tokens = elapsed * refill_rate
    bucket.tokens = math.min(capacity, bucket.tokens + new_tokens)
    bucket.last_refill = now
    
    if bucket.tokens >= 1 then
        bucket.tokens = bucket.tokens - 1
        return {
            allowed  = true,
            tokens   = math.floor(bucket.tokens),
            capacity = capacity,
        }
    end
    
    return {
        allowed  = false,
        tokens   = 0,
        retry_after = (1 - bucket.tokens) / refill_rate,
    }
end

-- ทดสอบ
print("=== Rate Limiting Patterns ===")

print("\n1. Fixed Window (5 requests per 60 seconds):")
for i = 1, 7 do
    local result = RateLimit.fixed_window("ip:127.0.0.1", 5, 60)
    if result.allowed then
        print(string.format("  Request %d: OK (remaining: %d)", i, result.remaining or 0))
    else
        print(string.format("  Request %d: BLOCKED (retry after: %ds)", i, result.retry_after or 0))
    end
end

print("\n2. Sliding Window (5 requests per 60 seconds):")
for i = 1, 7 do
    local result = RateLimit.sliding_window("api:user:1", 5, 60)
    if result.allowed then
        print(string.format("  Request %d: OK (remaining: %d)", i, result.remaining or 0))
    else
        print(string.format("  Request %d: BLOCKED (retry after: %.1fs)", i, result.retry_after or 0))
    end
end

print("\n3. Token Bucket (10 capacity, 1 token/second):")
-- Simulate burst
for i = 1, 5 do
    local result = RateLimit.token_bucket("user:2:api", 10, 1)
    print(string.format("  Request %d: %s (tokens: %d)",
        i, result.allowed and "OK" or "BLOCKED", result.tokens or 0))
end
```

---

## ตัวอย่างที่ 11: Distributed Lock

```lua
-- Distributed Lock implementation (Redlock concept)
local DistributedLock = {}
DistributedLock.__index = DistributedLock

local lock_store = {}

function DistributedLock.new(redis_client)
    local self = setmetatable({}, DistributedLock)
    self.redis = redis_client or lock_store  -- use shared storage
    self.default_ttl = 30000  -- 30 seconds in ms
    return self
end

-- สร้าง unique lock identifier
local function gen_lock_id()
    local id = ""
    for _ = 1, 16 do
        id = id .. string.format("%02x", math.random(0, 255))
    end
    return id
end

-- Acquire lock
function DistributedLock:acquire(resource, ttl_ms)
    ttl_ms = ttl_ms or self.default_ttl
    local lock_key = "lock:" .. resource
    local lock_id  = gen_lock_id()
    local expires_at = os.time() * 1000 + ttl_ms
    
    -- SET NX (set if not exists)
    local existing = lock_store[lock_key]
    
    -- Check if existing lock expired
    if existing and os.time() * 1000 > existing.expires_at then
        lock_store[lock_key] = nil
        existing = nil
    end
    
    if existing then
        return nil, string.format("Lock '%s' held by another process", resource)
    end
    
    lock_store[lock_key] = {
        id         = lock_id,
        resource   = resource,
        acquired_at = os.time(),
        expires_at = expires_at,
    }
    
    return {
        resource   = resource,
        lock_id    = lock_id,
        expires_at = expires_at,
        ttl_ms     = ttl_ms,
    }, nil
end

-- Release lock (only if we own it)
function DistributedLock:release(resource, lock_id)
    local lock_key = "lock:" .. resource
    local existing = lock_store[lock_key]
    
    if not existing then
        return false, "Lock not found"
    end
    
    if existing.id ~= lock_id then
        return false, "Lock owned by another process"
    end
    
    lock_store[lock_key] = nil
    return true, nil
end

-- Extend lock TTL
function DistributedLock:extend(resource, lock_id, extend_ms)
    local lock_key = "lock:" .. resource
    local existing = lock_store[lock_key]
    
    if not existing or existing.id ~= lock_id then
        return false, "Lock not owned"
    end
    
    existing.expires_at = existing.expires_at + extend_ms
    return true, nil
end

-- Execute with lock
function DistributedLock:with_lock(resource, ttl_ms, fn)
    local lock, err = self:acquire(resource, ttl_ms)
    if not lock then
        return nil, "Could not acquire lock: " .. err
    end
    
    local success, result = pcall(fn)
    
    self:release(resource, lock.lock_id)
    
    if not success then
        return nil, "Error in critical section: " .. tostring(result)
    end
    
    return result, nil
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Distributed Lock ===")

local dl = DistributedLock.new()

-- Acquire lock
local lock1, err1 = dl:acquire("shared-resource", 5000)
if lock1 then
    print("Lock acquired:", lock1.lock_id:sub(1, 8) .. "...")
    print("TTL:", lock1.ttl_ms, "ms")
else
    print("Failed:", err1)
end

-- Try to acquire same resource
local lock2, err2 = dl:acquire("shared-resource", 5000)
print("\nSecond acquire:", err2)

-- Release
local released, err3 = dl:release("shared-resource", lock1.lock_id)
print("\nRelease:", released, err3)

-- Now can acquire again
local lock3, _ = dl:acquire("shared-resource", 5000)
print("Third acquire:", lock3 and "OK" or "Failed")
if lock3 then dl:release("shared-resource", lock3.lock_id) end

-- with_lock usage
print("\nWith lock pattern:")
local result, err = dl:with_lock("db-migration", 10000, function()
    print("  Running critical section...")
    -- simulate work
    local data = "migration result"
    return data
end)
print("Result:", result, err)
```

---

## ตัวอย่างที่ 12: Session Storage ใน Redis

```lua
-- Redis Session Storage
local RedisSession = {}
RedisSession.__index = RedisSession

local session_store = {}

function RedisSession.new(options)
    local self = setmetatable({}, RedisSession)
    self.prefix  = options.prefix  or "sess:"
    self.ttl     = options.ttl     or 86400  -- 24 hours
    self.max_age = options.max_age or 86400
    return self
end

function RedisSession:_key(sid)
    return self.prefix .. sid
end

function RedisSession:create(data)
    local sid = string.format("%x%x%x%x",
        math.random(0xFFFF), math.random(0xFFFF),
        math.random(0xFFFF), math.random(0xFFFF))
    
    local key = self:_key(sid)
    local now = os.time()
    
    session_store[key] = {
        data       = data or {},
        created_at = now,
        updated_at = now,
        expires_at = now + self.ttl,
    }
    
    return sid
end

function RedisSession:get(sid)
    local key = self:_key(sid)
    local entry = session_store[key]
    if not entry then return nil end
    if os.time() > entry.expires_at then
        session_store[key] = nil
        return nil
    end
    -- Rolling TTL
    entry.expires_at = os.time() + self.ttl
    entry.updated_at = os.time()
    return entry.data
end

function RedisSession:set(sid, key, value)
    local full_key = self:_key(sid)
    local entry = session_store[full_key]
    if not entry then return false end
    entry.data[key] = value
    entry.updated_at = os.time()
    return true
end

function RedisSession:delete(sid)
    local key = self:_key(sid)
    if session_store[key] then
        session_store[key] = nil
        return true
    end
    return false
end

function RedisSession:touch(sid)
    local key = self:_key(sid)
    local entry = session_store[key]
    if entry then
        entry.expires_at = os.time() + self.ttl
        return true
    end
    return false
end

function RedisSession:serialize_data(data)
    -- ในระบบจริงใช้ MessagePack หรือ JSON
    local parts = {}
    for k, v in pairs(data) do
        table.insert(parts, k .. "=" .. tostring(v))
    end
    return table.concat(parts, "&")
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Redis Session Storage ===")

local sessions = RedisSession.new({
    prefix = "app:sess:",
    ttl    = 3600,
})

-- Create session
local sid = sessions:create({
    user_id  = 1,
    username = "alice",
    role     = "admin",
    cart     = {},
})
print("Session created:", sid)

-- Read
local data = sessions:get(sid)
if data then
    print("Username:", data.username)
    print("Role:", data.role)
end

-- Update
sessions:set(sid, "last_page", "/dashboard")
sessions:set(sid, "cart_count", 3)

local data2 = sessions:get(sid)
print("Last page:", data2.last_page)
print("Cart count:", data2.cart_count)

-- Delete (logout)
sessions:delete(sid)
local data3 = sessions:get(sid)
print("After delete:", tostring(data3))
```

---

## ตัวอย่างที่ 13: Caching Patterns

```lua
-- Redis Caching Patterns
local Cache = {}
local cache_store = {}
local cache_stats = { hits = 0, misses = 0, sets = 0 }

-- Cache-Aside (Lazy Loading)
function Cache.get_or_set(key, ttl, fetch_fn)
    local entry = cache_store[key]
    
    -- Check cache
    if entry then
        if os.time() < entry.expires_at then
            cache_stats.hits = cache_stats.hits + 1
            return entry.value, true  -- value, from_cache
        end
        cache_store[key] = nil
    end
    
    -- Cache miss - fetch from source
    cache_stats.misses = cache_stats.misses + 1
    local value, err = fetch_fn()
    if err then return nil, false, err end
    
    -- Store in cache
    cache_store[key] = {
        value      = value,
        created_at = os.time(),
        expires_at = os.time() + ttl,
    }
    cache_stats.sets = cache_stats.sets + 1
    
    return value, false
end

-- Write-Through Cache
function Cache.write_through(key, value, ttl, persist_fn)
    -- Save to persistent storage first
    local ok, err = persist_fn(key, value)
    if not ok then return false, err end
    
    -- Then update cache
    cache_store[key] = {
        value      = value,
        created_at = os.time(),
        expires_at = os.time() + ttl,
    }
    cache_stats.sets = cache_stats.sets + 1
    
    return true, nil
end

-- Write-Behind Cache (async)
local write_queue = {}

function Cache.write_behind(key, value, ttl)
    -- Update cache immediately
    cache_store[key] = {
        value      = value,
        created_at = os.time(),
        expires_at = os.time() + ttl,
        dirty      = true,
    }
    
    -- Queue for async persistence
    table.insert(write_queue, { key = key, value = value, queued_at = os.time() })
    return true
end

function Cache.flush_write_queue(persist_fn)
    local flushed = 0
    while #write_queue > 0 do
        local item = table.remove(write_queue, 1)
        persist_fn(item.key, item.value)
        flushed = flushed + 1
    end
    return flushed
end

-- Cache Invalidation
function Cache.invalidate(pattern)
    local count = 0
    local keys_to_delete = {}
    for key in pairs(cache_store) do
        if key:match(pattern) then
            table.insert(keys_to_delete, key)
        end
    end
    for _, key in ipairs(keys_to_delete) do
        cache_store[key] = nil
        count = count + 1
    end
    return count
end

-- Stats
function Cache.get_stats()
    local total = cache_stats.hits + cache_stats.misses
    local hit_rate = total > 0 and (cache_stats.hits / total * 100) or 0
    return {
        hits     = cache_stats.hits,
        misses   = cache_stats.misses,
        sets     = cache_stats.sets,
        hit_rate = string.format("%.1f%%", hit_rate),
        cached   = (function()
            local c = 0
            for _ in pairs(cache_store) do c = c + 1 end
            return c
        end)(),
    }
end

-- Mock database
local mock_db = {
    users = {
        [1] = { id = 1, name = "Alice", email = "alice@example.com" },
        [2] = { id = 2, name = "Bob",   email = "bob@example.com" },
    }
}

local db_queries = 0

local function fetch_user(user_id)
    db_queries = db_queries + 1
    print(string.format("  [DB Query #%d] Fetching user %d", db_queries, user_id))
    return mock_db.users[user_id], nil
end

-- ทดสอบ
print("=== Caching Patterns ===")
print("\nCache-Aside Pattern:")

for i = 1, 3 do
    print(string.format("\nAccess user 1 (iteration %d):", i))
    local user, from_cache, err = Cache.get_or_set(
        "user:1",
        300,  -- 5 minutes TTL
        function() return fetch_user(1) end
    )
    if user then
        print(string.format("  Name: %s (from_cache: %s)", user.name, tostring(from_cache)))
    end
end

print("\nCache Stats:", Cache.get_stats().hits, "hits,",
      Cache.get_stats().misses, "misses,",
      Cache.get_stats().hit_rate, "hit rate")

-- Invalidation
print("\nInvalidating user cache...")
local inv = Cache.invalidate("user:.*")
print("Invalidated:", inv, "entries")

-- Fetch again after invalidation
print("\nFetch after invalidation:")
local user2, from_cache2 = Cache.get_or_set("user:1", 300, function() return fetch_user(1) end)
print("Name:", user2.name, "from_cache:", from_cache2)
```

---

## ตัวอย่างที่ 14: Distributed Counter

```lua
-- Distributed Counter patterns
local DistCounter = {}
local counter_store = {}

-- Simple increment counter
function DistCounter.increment(key, amount)
    amount = amount or 1
    counter_store[key] = (counter_store[key] or 0) + amount
    return counter_store[key]
end

function DistCounter.decrement(key, amount)
    amount = amount or 1
    counter_store[key] = (counter_store[key] or 0) - amount
    return counter_store[key]
end

function DistCounter.get(key)
    return counter_store[key] or 0
end

function DistCounter.reset(key)
    counter_store[key] = 0
    return true
end

-- Time-windowed counter
local windowed_counters = {}

function DistCounter.increment_windowed(key, window_seconds)
    local window = math.floor(os.time() / window_seconds)
    local wkey = key .. ":" .. window
    
    if not windowed_counters[wkey] then
        windowed_counters[wkey] = {
            count = 0,
            expires = (window + 1) * window_seconds,
        }
    end
    
    windowed_counters[wkey].count = windowed_counters[wkey].count + 1
    return windowed_counters[wkey].count
end

function DistCounter.get_windowed(key, window_seconds)
    local window = math.floor(os.time() / window_seconds)
    local wkey = key .. ":" .. window
    local entry = windowed_counters[wkey]
    if not entry or os.time() > entry.expires then return 0 end
    return entry.count
end

-- Page view tracker
local page_views = {}
local daily_unique = {}

function DistCounter.track_pageview(page, user_id)
    -- Total views
    DistCounter.increment("pageview:total:" .. page)
    
    -- Today's views
    local today = os.date("%Y-%m-%d")
    DistCounter.increment("pageview:daily:" .. page .. ":" .. today)
    
    -- Unique visitors (using SET to track)
    local unique_key = "unique:" .. page .. ":" .. today
    if not daily_unique[unique_key] then
        daily_unique[unique_key] = {}
    end
    local is_new = not daily_unique[unique_key][user_id]
    daily_unique[unique_key][user_id] = true
    
    if is_new then
        DistCounter.increment("unique:" .. page .. ":" .. today)
    end
    
    return {
        total   = DistCounter.get("pageview:total:" .. page),
        today   = DistCounter.get("pageview:daily:" .. page .. ":" .. today),
        unique  = DistCounter.get("unique:" .. page .. ":" .. today),
    }
end

-- ทดสอบ
print("=== Distributed Counters ===")

-- Basic counter
DistCounter.increment("api:calls")
DistCounter.increment("api:calls")
DistCounter.increment("api:calls", 5)
print("API calls:", DistCounter.get("api:calls"))

DistCounter.decrement("api:calls", 2)
print("After decrement:", DistCounter.get("api:calls"))

-- Page view tracking
print("\nPage View Tracking:")
local pages = { "/home", "/about", "/home", "/products" }
local users = { 1, 2, 1, 3 }

for i, page in ipairs(pages) do
    local stats = DistCounter.track_pageview(page, users[i])
    print(string.format("  Page: %-12s total=%d today=%d unique=%d",
        page, stats.total, stats.today, stats.unique))
end
```

---

## ตัวอย่างที่ 15: Bloom Filter in Redis

```lua
-- Bloom Filter implementation (Redis BF simulation)
-- Redis มี RedisBloom module สำหรับ Bloom Filter จริงๆ

local BloomFilter = {}
BloomFilter.__index = BloomFilter

function BloomFilter.new(expected_items, false_positive_rate)
    local self = setmetatable({}, BloomFilter)
    expected_items = expected_items or 1000
    false_positive_rate = false_positive_rate or 0.01
    
    -- คำนวณขนาด bit array
    local m = math.ceil(-expected_items * math.log(false_positive_rate) / (math.log(2)^2))
    -- คำนวณจำนวน hash functions
    local k = math.ceil((m / expected_items) * math.log(2))
    
    self.size       = m
    self.num_hashes = k
    self.bits       = {}  -- bit array
    self.count      = 0
    
    print(string.format("Bloom Filter: size=%d bits, hash_functions=%d, FP_rate=%.2f%%",
        m, k, false_positive_rate * 100))
    
    return self
end

-- Hash functions (k different hash functions)
function BloomFilter:_hash(item, seed)
    local hash = seed * 2654435761
    for i = 1, #item do
        hash = ((hash * 31) + string.byte(item, i)) % (2^32)
        hash = hash ~ (hash >> 16)
        hash = (hash * 2246822519) % (2^32)
        hash = hash ~ (hash >> 13)
    end
    return hash % self.size
end

-- Add item
function BloomFilter:add(item)
    for i = 1, self.num_hashes do
        local pos = self:_hash(tostring(item), i * 1234567)
        self.bits[pos] = true
    end
    self.count = self.count + 1
end

-- Check if item might exist
function BloomFilter:contains(item)
    for i = 1, self.num_hashes do
        local pos = self:_hash(tostring(item), i * 1234567)
        if not self.bits[pos] then
            return false  -- Definitely NOT in set
        end
    end
    return true  -- PROBABLY in set (might be false positive)
end

-- Get fill ratio
function BloomFilter:fill_ratio()
    local set_bits = 0
    for _ in pairs(self.bits) do set_bits = set_bits + 1 end
    return set_bits / self.size
end

-- ใช้ Bloom Filter เพื่อตรวจสอบ email ที่ใช้แล้ว
print("=== Bloom Filter - Email Dedup ===")
local bf = BloomFilter.new(1000, 0.01)

-- เพิ่ม emails ที่ใช้แล้ว
local registered_emails = {
    "alice@example.com", "bob@example.com", "charlie@example.com",
    "diana@example.com", "eve@example.com", "frank@example.com",
}

print("\nAdding registered emails...")
for _, email in ipairs(registered_emails) do
    bf:add(email)
end
print("Items added:", bf.count)
print("Fill ratio:", string.format("%.2f%%", bf:fill_ratio() * 100))

-- ตรวจสอบ
local check_emails = {
    "alice@example.com",    -- ลงทะเบียนแล้ว
    "newuser@example.com",  -- ยังไม่ลงทะเบียน
    "bob@example.com",      -- ลงทะเบียนแล้ว
    "hacker@evil.com",      -- ยังไม่ลงทะเบียน
}

print("\nChecking emails:")
for _, email in ipairs(check_emails) do
    local might_exist = bf:contains(email)
    print(string.format("  %-30s -> %s",
        email,
        might_exist and "PROBABLY REGISTERED (check DB)" or "DEFINITELY NEW"))
end
```

---

## ตัวอย่างที่ 16: Priority Queue with Redis

```lua
-- Priority Queue using Redis Sorted Set
local PriorityQueue = {}
PriorityQueue.__index = PriorityQueue

function PriorityQueue.new(name)
    local self = setmetatable({}, PriorityQueue)
    self.key = "pq:" .. name
    self.store = {} -- { member -> score }
    return self
end

-- เพิ่ม item ด้วย priority (ต่ำ = priority สูง)
function PriorityQueue:push(item, priority)
    self.store[item] = priority
    return true
end

-- เอา item ที่มี priority สูงสุดออก (score ต่ำสุด)
function PriorityQueue:pop()
    local min_score = math.huge
    local min_member = nil
    
    for member, score in pairs(self.store) do
        if score < min_score then
            min_score = score
            min_member = member
        end
    end
    
    if min_member then
        self.store[min_member] = nil
        return min_member, min_score
    end
    return nil, nil
end

-- Peek ไม่เอาออก
function PriorityQueue:peek()
    local min_score = math.huge
    local min_member = nil
    
    for member, score in pairs(self.store) do
        if score < min_score then
            min_score = score
            min_member = member
        end
    end
    
    return min_member, min_score
end

function PriorityQueue:size()
    local count = 0
    for _ in pairs(self.store) do count = count + 1 end
    return count
end

-- ทดสอบ Priority Queue
print("=== Priority Queue ===")

local pq = PriorityQueue.new("tasks")

-- Task priorities: 1=critical, 5=low
local tasks = {
    { name = "send_invoice",    priority = 3 },
    { name = "backup_database", priority = 2 },
    { name = "generate_report", priority = 4 },
    { name = "security_patch",  priority = 1 },  -- CRITICAL
    { name = "cleanup_logs",    priority = 5 },
    { name = "update_users",    priority = 2 },
}

for _, task in ipairs(tasks) do
    pq:push(task.name, task.priority)
end

print(string.format("Queue size: %d", pq:size()))
print("\nProcessing by priority:")

while pq:size() > 0 do
    local task, priority = pq:pop()
    local labels = { "CRITICAL", "HIGH", "MEDIUM", "LOW", "VERY LOW" }
    print(string.format("  [P%d %s] %s", priority, labels[priority] or "?", task))
end
```

---

## ตัวอย่างที่ 17: Delayed Queue

```lua
-- Delayed Queue - Execute tasks at future time
local DelayedQueue = {}
local delayed_store = {}

function DelayedQueue.schedule(task_id, task_data, execute_at)
    -- ใน Redis ใช้ ZADD ด้วย score = timestamp
    delayed_store[task_id] = {
        id          = task_id,
        data        = task_data,
        execute_at  = execute_at,
        scheduled_at = os.time(),
    }
    return true
end

function DelayedQueue.schedule_after(task_id, task_data, delay_seconds)
    return DelayedQueue.schedule(task_id, task_data, os.time() + delay_seconds)
end

-- ดึง tasks ที่ถึงเวลาแล้ว
function DelayedQueue.get_due_tasks(limit)
    local now = os.time()
    local due = {}
    
    for id, task in pairs(delayed_store) do
        if task.execute_at <= now then
            table.insert(due, task)
        end
    end
    
    -- Sort by execute_at
    table.sort(due, function(a, b) return a.execute_at < b.execute_at end)
    
    if limit then
        local result = {}
        for i = 1, math.min(limit, #due) do
            table.insert(result, due[i])
        end
        return result
    end
    
    return due
end

-- Remove task (after processing)
function DelayedQueue.acknowledge(task_id)
    if delayed_store[task_id] then
        delayed_store[task_id] = nil
        return true
    end
    return false
end

function DelayedQueue.cancel(task_id)
    return DelayedQueue.acknowledge(task_id)
end

function DelayedQueue.size()
    local count = 0
    for _ in pairs(delayed_store) do count = count + 1 end
    return count
end

-- ทดสอบ
print("=== Delayed Queue ===")

local now = os.time()

-- Schedule tasks
DelayedQueue.schedule("task:001", { type = "email", to = "alice@example.com" }, now - 5)  -- due
DelayedQueue.schedule("task:002", { type = "report", format = "pdf" }, now - 2)           -- due
DelayedQueue.schedule("task:003", { type = "backup", db = "production" }, now + 300)     -- future
DelayedQueue.schedule("task:004", { type = "cleanup", target = "tmp" }, now + 3600)     -- future

print("Total scheduled:", DelayedQueue.size())

-- Get due tasks
local due = DelayedQueue.get_due_tasks()
print("Due tasks:", #due)

for _, task in ipairs(due) do
    print(string.format("  Processing: [%s] type=%s",
        task.id, task.data.type))
    DelayedQueue.acknowledge(task.id)
end

print("Remaining:", DelayedQueue.size())
```

---

## ตัวอย่างที่ 18: Complete Redis Integration Example

```lua
-- Complete example: Real-time analytics with Redis
local Analytics = {}
local analytics_store = {}

-- Track event
function Analytics.track(event_type, user_id, metadata)
    local now = os.time()
    local date = os.date("%Y-%m-%d", now)
    local hour = os.date("%H", now)
    
    -- Increment various counters
    local keys = {
        "events:total",
        "events:type:" .. event_type,
        "events:user:" .. user_id,
        "events:daily:" .. date,
        "events:hourly:" .. date .. ":" .. hour,
    }
    
    for _, key in ipairs(keys) do
        analytics_store[key] = (analytics_store[key] or 0) + 1
    end
    
    -- Track unique users per day
    local unique_key = "unique:users:" .. date
    if not analytics_store[unique_key] then
        analytics_store[unique_key] = {}
    end
    analytics_store[unique_key][user_id] = true
    
    -- Track unique users per event type
    local event_unique_key = "unique:event:" .. event_type .. ":" .. date
    if not analytics_store[event_unique_key] then
        analytics_store[event_unique_key] = {}
    end
    analytics_store[event_unique_key][user_id] = true
    
    return true
end

function Analytics.get_stats(date)
    date = date or os.date("%Y-%m-%d")
    
    local unique_users = 0
    local ukey = "unique:users:" .. date
    if analytics_store[ukey] then
        for _ in pairs(analytics_store[ukey]) do
            unique_users = unique_users + 1
        end
    end
    
    -- Collect hourly data
    local hourly = {}
    for h = 0, 23 do
        local hkey = string.format("events:hourly:%s:%02d", date, h)
        hourly[h+1] = analytics_store[hkey] or 0
    end
    
    return {
        total_events  = analytics_store["events:total"] or 0,
        daily_events  = analytics_store["events:daily:" .. date] or 0,
        unique_users  = unique_users,
        hourly        = hourly,
        page_views    = analytics_store["events:type:page_view"] or 0,
        clicks        = analytics_store["events:type:click"] or 0,
        conversions   = analytics_store["events:type:purchase"] or 0,
    }
end

-- ทดสอบ
print("=== Real-time Analytics ===")

-- Simulate events
local event_types = { "page_view", "page_view", "click", "page_view",
                      "click", "purchase", "page_view", "click" }
local user_ids    = { 1, 2, 1, 3, 2, 1, 4, 3 }

for i, event_type in ipairs(event_types) do
    Analytics.track(event_type, user_ids[i], {})
end

-- Get stats
local stats = Analytics.get_stats()
print("\nAnalytics Summary:")
print("  Total events:  ", stats.total_events)
print("  Daily events:  ", stats.daily_events)
print("  Unique users:  ", stats.unique_users)
print("  Page views:    ", stats.page_views)
print("  Clicks:        ", stats.clicks)
print("  Conversions:   ", stats.conversions)
print("  Conversion rate:", string.format("%.1f%%",
    stats.page_views > 0 and (stats.conversions / stats.page_views * 100) or 0))
```

---

## สรุปบทที่ 57

ในบทนี้เราได้เรียนรู้:

1. **Redis Data Structures** - String, Hash, List, Set, Sorted Set
2. **lua-resty-redis** - Connection, basic operations, connection pooling
3. **Hash operations** - User profiles, HGETALL, HINCRBY
4. **List operations** - Queue (FIFO), Stack (LIFO)
5. **Sorted Set** - Leaderboard, ranking
6. **Pub/Sub** - Event-driven messaging
7. **EVAL command** - Lua scripts in Redis
8. **EVALSHA** - Script caching ด้วย SHA1
9. **Atomic operations** - CAS, transfer, pop-n
10. **Rate limiting** - Fixed window, sliding window, token bucket
11. **Distributed Lock** - Redlock concepts
12. **Session storage** - Redis-backed sessions
13. **Caching patterns** - Cache-aside, write-through, write-behind
14. **Distributed counters** - Page views, windowed counters
15. **Bloom filter** - Probabilistic data structure
16. **Priority queue** - Task scheduling
17. **Delayed queue** - Future execution
18. **Analytics** - Real-time tracking

> **สำคัญ**: ในระบบ production ใช้ `lua-resty-redis` สำหรับ OpenResty หรือ `redis-lua` สำหรับ standalone Lua
