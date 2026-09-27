# บทที่ 60: Message Queue กับ Redis

## บทนำ

Redis เป็น in-memory data structure store ที่นอกจากจะทำหน้าที่เป็น cache แล้ว ยังเป็น message broker ชั้นเยี่ยมอีกด้วย ใน Lua เราใช้ library เช่น `lua-resty-redis` (สำหรับ OpenResty) หรือ `redis-lua` (สำหรับ standalone Lua) เพื่อเชื่อมต่อกับ Redis

Message Queue ด้วย Redis ช่วยให้แอปพลิเคชันสื่อสารแบบ asynchronous ได้ ทำให้ decouple services, handle load spikes และรองรับ background job processing

> **บทก่อนหน้า**: [บทที่ 59: SQLite กับ Lua](part-59.md)

---

## 60.1 การเชื่อมต่อ Redis

### ตัวอย่างที่ 1: Setup และ Connection

```bash
# ติดตั้ง redis-lua
luarocks install redis-lua

# หรือสำหรับ OpenResty ใช้ lua-resty-redis (built-in)
# ตรวจสอบการติดตั้ง
lua -e "require('redis'); print('redis-lua OK')"

# ติดตั้ง Redis server
sudo apt-get install redis-server
# หรือด้วย Docker
docker run -d -p 6379:6379 redis:7-alpine
```

```lua
-- redis_connect.lua
-- ตัวอย่างใช้ redis-lua library

local redis = require("redis")

-- เชื่อมต่อ Redis
local client = redis.connect("127.0.0.1", 6379)

-- ทดสอบ connection
local pong = client:ping()
print("PING:", pong)  -- PONG

-- ข้อมูล server
local info = client:info("server")
-- แสดงเฉพาะ version
for line in info:gmatch("[^\r\n]+") do
    if line:match("^redis_version") then
        print("Redis:", line)
    end
end

-- Authentication (ถ้ามี password)
-- client:auth("your_password")

-- เลือก database (0-15)
client:select(0)  -- default database

-- ทดสอบ basic operations
client:set("hello", "world")
print("GET hello:", client:get("hello"))

-- ลบ key
client:del("hello")
print("After DEL:", client:get("hello"))  -- nil

-- Disconnect
client:quit()
print("Disconnected")
```

### ตัวอย่างที่ 2: Connection Pool

```lua
-- redis_pool.lua
-- Connection pool สำหรับ Redis

local redis = require("redis")

local RedisPool = {}
RedisPool.__index = RedisPool

function RedisPool.new(host, port, pool_size)
    local self = setmetatable({}, RedisPool)
    self.host = host or "127.0.0.1"
    self.port = port or 6379
    self.pool_size = pool_size or 10
    self.pool = {}
    self.available = {}
    
    for i = 1, pool_size do
        local client = redis.connect(self.host, self.port)
        self.pool[i] = client
        table.insert(self.available, i)
    end
    
    print(string.format("Redis pool: %d connections to %s:%d",
        pool_size, self.host, self.port))
    return self
end

function RedisPool:acquire()
    if #self.available == 0 then
        error("Redis pool exhausted!")
    end
    local idx = table.remove(self.available)
    return self.pool[idx], idx
end

function RedisPool:release(idx)
    table.insert(self.available, idx)
end

function RedisPool:execute(cmd, ...)
    local client, idx = self:acquire()
    local ok, result = pcall(function()
        return client[cmd](client, ...)
    end)
    self:release(idx)
    if not ok then error(result) end
    return result
end

function RedisPool:pipeline(commands)
    local client, idx = self:acquire()
    local results = {}
    
    local ok, err = pcall(function()
        client:pipeline(function(pipe)
            for _, cmd in ipairs(commands) do
                pipe[cmd[1]](pipe, table.unpack(cmd, 2))
            end
        end)
    end)
    
    self:release(idx)
    if not ok then error(err) end
    return results
end

function RedisPool:close()
    for _, client in ipairs(self.pool) do
        pcall(function() client:quit() end)
    end
    print("Redis pool closed")
end

-- ใช้งาน
local pool = RedisPool.new("127.0.0.1", 6379, 5)

-- Execute commands
pool:execute("set", "test_key", "test_value")
print("GET:", pool:execute("get", "test_key"))
pool:execute("del", "test_key")

print("Available connections:", #pool.available)
pool:close()
```

---

## 60.2 LPUSH/RPOP Queue

### ตัวอย่างที่ 3: Simple Queue ด้วย LPUSH/RPOP

```lua
-- simple_queue.lua
local redis = require("redis")
local json = require("dkjson")  -- หรือ cjson

local client = redis.connect("127.0.0.1", 6379)

-- Simple Queue: LPUSH เพิ่มหัว, RPOP ดึงท้าย (FIFO)
local QUEUE_KEY = "jobs:email"

-- Producer: เพิ่ม jobs เข้า queue
local function enqueue(job)
    local payload = json.encode(job)
    local len = client:lpush(QUEUE_KEY, payload)
    print(string.format("Enqueued job '%s' (queue length: %d)", 
        job.type, len))
    return len
end

-- Consumer: ดึง job จาก queue
local function dequeue()
    local payload = client:rpop(QUEUE_KEY)
    if not payload then return nil end
    return json.decode(payload)
end

-- Peek (ดูโดยไม่ลบ)
local function peek()
    local payload = client:lindex(QUEUE_KEY, -1)
    if not payload then return nil end
    return json.decode(payload)
end

-- Queue size
local function queue_size()
    return client:llen(QUEUE_KEY)
end

-- ล้าง queue
client:del(QUEUE_KEY)

-- Enqueue jobs
enqueue({type = "welcome_email", user_id = 101, email = "alice@example.com"})
enqueue({type = "password_reset", user_id = 102, email = "bob@example.com"})
enqueue({type = "order_confirm",  user_id = 103, email = "carol@example.com"})
enqueue({type = "newsletter",     user_id = 104, email = "dave@example.com"})

print("Queue size:", queue_size())
print("Next job:", peek() and peek().type or "empty")

-- Dequeue และ process
print("\nProcessing jobs:")
while true do
    local job = dequeue()
    if not job then break end
    print(string.format("  Processing: %s for user %d (%s)",
        job.type, job.user_id, job.email))
    -- ทำงานจริงที่นี่ เช่น send_email(job)
end

print("Queue empty:", queue_size() == 0)
client:quit()
```

### ตัวอย่างที่ 4: BRPOP - Blocking Queue

```lua
-- blocking_queue.lua
local redis = require("redis")
local json = require("dkjson")

-- BRPOP = Blocking RPOP: รอจนมี item ใน queue

-- Worker process
local function run_worker(worker_id, queues, timeout)
    local client = redis.connect("127.0.0.1", 6379)
    print(string.format("Worker %d started, watching: %s",
        worker_id, table.concat(queues, ", ")))
    
    local running = true
    local processed = 0
    
    while running do
        -- BRPOP รอสูงสุด timeout วินาที
        -- คืนค่า: {queue_name, value} หรือ nil ถ้า timeout
        local result = client:brpop(table.unpack(queues), timeout or 5)
        
        if result then
            local queue_name = result[1]
            local payload    = result[2]
            
            local ok, job = pcall(json.decode, payload)
            if ok and job then
                print(string.format("Worker %d: Processing from [%s]: %s",
                    worker_id, queue_name, job.type or "unknown"))
                
                -- simulate work
                -- process_job(job)
                processed = processed + 1
                
                -- หยุด worker เมื่อได้รับ stop signal
                if job.type == "STOP" then
                    running = false
                end
            else
                print(string.format("Worker %d: Invalid payload: %s", 
                    worker_id, payload))
            end
        else
            -- Timeout - ตรวจสอบสถานะ
            print(string.format("Worker %d: No jobs (timeout), processed=%d",
                worker_id, processed))
            -- สามารถ break ได้ถ้าต้องการ
            break  -- ออกจาก loop สำหรับตัวอย่างนี้
        end
    end
    
    print(string.format("Worker %d: Shutdown, processed %d jobs",
        worker_id, processed))
    client:quit()
end

-- Producer
local function produce_jobs()
    local client = redis.connect("127.0.0.1", 6379)
    
    local jobs = {
        {type = "email",  data = "Send welcome email"},
        {type = "report", data = "Generate monthly report"},
        {type = "image",  data = "Resize uploaded image"},
        {type = "STOP",   data = ""},
    }
    
    for _, job in ipairs(jobs) do
        client:lpush("jobs:default", json.encode(job))
        print("Produced:", job.type)
    end
    
    client:quit()
end

-- ในตัวอย่างนี้ run แบบ sequential (ใน production ใช้ separate processes)
produce_jobs()
run_worker(1, {"jobs:default"}, 2)
```

---

## 60.3 Pub/Sub Patterns

### ตัวอย่างที่ 5: Publisher

```lua
-- publisher.lua
local redis = require("redis")
local json = require("dkjson")

local client = redis.connect("127.0.0.1", 6379)

-- Publish messages ไปยัง channel
local function publish(channel, message)
    local payload
    if type(message) == "table" then
        payload = json.encode(message)
    else
        payload = tostring(message)
    end
    
    local subscribers = client:publish(channel, payload)
    return subscribers  -- จำนวน subscribers ที่ได้รับ
end

-- Publish ไปยัง channels ต่างๆ
local subs

subs = publish("notifications:user:101", {
    type = "like",
    from = "alice",
    post_id = 42
})
print("notification delivered to", subs, "subscribers")

subs = publish("chat:room:general", {
    user = "bob",
    text = "Hello everyone!",
    ts   = os.time()
})
print("chat message delivered to", subs, "subscribers")

subs = publish("system:alerts", {
    level   = "warning",
    message = "High CPU usage detected",
    server  = "web-01"
})
print("alert delivered to", subs, "subscribers")

-- Pattern-based channel (wildcard)
subs = publish("events:orders:created", {
    order_id = 12345,
    total    = 299.99,
    user_id  = 101
})
print("order event delivered to", subs, "subscribers")

client:quit()
```

### ตัวอย่างที่ 6: Subscriber

```lua
-- subscriber.lua
local redis = require("redis")
local json = require("dkjson")

local client = redis.connect("127.0.0.1", 6379)

-- Handler functions
local handlers = {}

handlers["notifications:user:101"] = function(channel, data)
    if data.type == "like" then
        print(string.format("  [NOTIFICATION] %s liked your post #%d",
            data.from, data.post_id))
    end
end

handlers["chat:room:general"] = function(channel, data)
    print(string.format("  [CHAT] %s: %s", data.user, data.text))
end

handlers["system:alerts"] = function(channel, data)
    print(string.format("  [ALERT:%s] %s: %s",
        data.level:upper(), data.server, data.message))
end

-- Subscribe to channels
local channels = {
    "notifications:user:101",
    "chat:room:general",
    "system:alerts"
}

print("Subscribing to:", table.concat(channels, ", "))

-- redis-lua subscribe callback
client:subscribe(table.unpack(channels))

-- Listen loop
local received = 0
local max_messages = 5  -- รับแค่ 5 messages แล้วหยุด (สำหรับตัวอย่าง)

for msg in client:receive() do
    if msg.kind == "message" then
        local ok, data = pcall(json.decode, msg.payload)
        if ok and type(data) == "table" then
            local handler = handlers[msg.channel]
            if handler then
                handler(msg.channel, data)
            else
                print(string.format("  [%s] %s", msg.channel, msg.payload))
            end
        end
        received = received + 1
        if received >= max_messages then break end
    elseif msg.kind == "subscribe" then
        print(string.format("  Subscribed to '%s' (total: %d)",
            msg.channel, msg.number))
    end
end

client:unsubscribe()
client:quit()
```

### ตัวอย่างที่ 7: Pattern Subscribe (PSUBSCRIBE)

```lua
-- pattern_subscribe.lua
local redis = require("redis")
local json = require("dkjson")

local client = redis.connect("127.0.0.1", 6379)

-- PSUBSCRIBE รองรับ wildcard patterns:
-- * = ตัวอักษรอะไรก็ได้
-- ? = ตัวอักษรหนึ่งตัว
-- [abc] = ตัวอักษรใดตัวหนึ่งใน set

-- Subscribe ด้วย patterns
client:psubscribe(
    "events:orders:*",      -- orders ทุกประเภท
    "notifications:user:*", -- notifications ทุก user
    "system:*"              -- system messages ทั้งหมด
)

print("Pattern subscribed")

-- Event router
local function route_event(pattern, channel, payload)
    local ok, data = pcall(json.decode, payload)
    if not ok then data = {raw = payload} end
    
    -- Extract event type from channel
    local parts = {}
    for p in channel:gmatch("[^:]+") do
        table.insert(parts, p)
    end
    
    local category = parts[1]
    
    if category == "events" then
        local resource = parts[2]
        local action   = parts[3]
        print(string.format("  [EVENT] %s.%s - %s",
            resource, action, json.encode(data)))
    elseif category == "notifications" then
        local user_id = parts[3]
        print(string.format("  [NOTIFY] User %s: %s",
            user_id, json.encode(data)))
    elseif category == "system" then
        print(string.format("  [SYSTEM] %s", json.encode(data)))
    end
end

-- Listen
local count = 0
for msg in client:receive() do
    if msg.kind == "pmessage" then
        route_event(msg.pattern, msg.channel, msg.payload)
        count = count + 1
        if count >= 3 then break end
    elseif msg.kind == "psubscribe" then
        print(string.format("  Psubscribed to '%s'", msg.channel))
    end
end

client:punsubscribe()
client:quit()
```

---

## 60.4 Redis Streams

### ตัวอย่างที่ 8: XADD - เพิ่ม Events ใน Stream

```lua
-- xadd_stream.lua
local redis = require("redis")
local json = require("dkjson")

local client = redis.connect("127.0.0.1", 6379)

local STREAM = "events:log"

-- ล้าง stream เก่า
client:del(STREAM)

-- XADD เพิ่ม event เข้า stream
-- syntax: XADD key [MAXLEN count] id field value [field value ...]
-- id = "*" ให้ Redis สร้างอัตโนมัติ

-- เพิ่ม events แบบธรรมดา
local id1 = client:xadd(STREAM, "*",
    "type",    "user.login",
    "user_id", "101",
    "ip",      "192.168.1.1",
    "ts",      tostring(os.time()))

print("Added event:", id1)  -- เช่น 1706000000000-0

-- เพิ่ม events ด้วย custom id prefix
local id2 = client:xadd(STREAM, "*",
    "type",      "order.created",
    "order_id",  "ORD-2025-001",
    "user_id",   "102",
    "amount",    "299.99")

print("Added event:", id2)

-- XADD พร้อม MAXLEN (จำกัดขนาด stream)
-- ~ = approximate trimming (เร็วกว่า)
local id3 = client:xadd(STREAM, "MAXLEN", "~", 1000, "*",
    "type",    "page.view",
    "user_id", "103",
    "page",    "/products/42")

print("Added with maxlen:", id3)

-- ดูขนาด stream
local len = client:xlen(STREAM)
print("Stream length:", len)

-- ดู range ของ ids
local first, last = client:xrange(STREAM, "-", "+", "COUNT", 1),
                    client:xrevrange(STREAM, "+", "-", "COUNT", 1)
if first and first[1] then
    print("First ID:", first[1][1])
end

client:quit()
```

### ตัวอย่างที่ 9: XREAD - อ่าน Stream

```lua
-- xread_stream.lua
local redis = require("redis")

local client = redis.connect("127.0.0.1", 6379)
local STREAM = "events:log"

-- Helper: parse stream entry
local function parse_entry(entry)
    local id = entry[1]
    local fields = entry[2]
    local data = {_id = id}
    for i = 1, #fields, 2 do
        data[fields[i]] = fields[i+1]
    end
    return data
end

-- XRANGE - ดึงทุก event ใน range
print("=== All events (XRANGE) ===")
local entries = client:xrange(STREAM, "-", "+")
for _, entry in ipairs(entries or {}) do
    local e = parse_entry(entry)
    print(string.format("  [%s] type=%s user=%s",
        e._id, e.type or "?", e.user_id or "?"))
end

-- XREAD - อ่าน events หลังจาก id ที่กำหนด
print("\n=== XREAD from beginning ===")
local result = client:xread("COUNT", 10, "STREAMS", STREAM, "0")
if result then
    for _, stream_data in ipairs(result) do
        local stream_name = stream_data[1]
        local events = stream_data[2]
        print("Stream:", stream_name)
        for _, entry in ipairs(events) do
            local e = parse_entry(entry)
            print(string.format("  [%s] %s", e._id, e.type or "?"))
        end
    end
end

-- XREVRANGE - ดึง events ล่าสุดก่อน
print("\n=== Latest 2 events (XREVRANGE) ===")
local latest = client:xrevrange(STREAM, "+", "-", "COUNT", 2)
for _, entry in ipairs(latest or {}) do
    local e = parse_entry(entry)
    print(string.format("  [%s] %s", e._id, e.type or "?"))
end

-- XREAD แบบ blocking (รอ events ใหม่)
-- local result = client:xread("COUNT", 10, "BLOCK", 5000, "STREAMS", STREAM, "$")
-- "$" = เริ่มจาก id ล่าสุด (เฉพาะ events ใหม่)

client:quit()
```

---

## 60.5 Consumer Groups

### ตัวอย่างที่ 10: สร้าง Consumer Group

```lua
-- consumer_group.lua
local redis = require("redis")

local client = redis.connect("127.0.0.1", 6379)
local STREAM = "jobs:processing"
local GROUP  = "workers"

-- ล้าง stream เก่า
client:del(STREAM)

-- สร้าง Consumer Group
-- XGROUP CREATE stream group $ MKSTREAM
-- $ = เริ่มจาก events ใหม่เท่านั้น
-- 0 = เริ่มจากต้น stream
local ok, err = pcall(function()
    client:xgroup("CREATE", STREAM, GROUP, "0", "MKSTREAM")
end)
if ok then
    print("Consumer group '" .. GROUP .. "' created")
else
    -- อาจมี group อยู่แล้ว
    print("Group exists or error:", err)
end

-- เพิ่ม jobs
for i = 1, 5 do
    client:xadd(STREAM, "*",
        "job_id",   tostring(i),
        "type",     "process_data",
        "payload",  "data_" .. i,
        "priority", tostring(math.random(1, 5)))
end

print("Added 5 jobs to stream")

-- ดู group info
local groups = client:xinfo("GROUPS", STREAM)
if groups then
    print("\nGroup info:")
    for _, g in ipairs(groups) do
        -- g คือ flat array: {key, val, key, val, ...}
        local info = {}
        for i = 1, #g, 2 do
            info[g[i]] = g[i+1]
        end
        print(string.format("  name=%s pending=%s consumers=%s",
            info["name"] or "?",
            info["pending"] or "?",
            info["consumers"] or "?"))
    end
end

client:quit()
```

### ตัวอย่างที่ 11: XREADGROUP - Consumer Worker

```lua
-- xreadgroup_worker.lua
local redis = require("redis")
local json = require("dkjson")

local STREAM   = "jobs:processing"
local GROUP    = "workers"

-- Helper: parse stream entry to table
local function parse_entry(entry)
    local id = entry[1]
    local fields = entry[2]
    local data = {_id = id}
    for i = 1, #fields, 2 do
        data[fields[i]] = fields[i+1]
    end
    return data
end

-- Worker function
local function run_consumer(consumer_id)
    local client = redis.connect("127.0.0.1", 6379)
    local consumer_name = "worker-" .. consumer_id
    
    print(string.format("Consumer '%s' started", consumer_name))
    
    local processed = 0
    local max_jobs = 3  -- process 3 jobs แล้วหยุด (สำหรับตัวอย่าง)
    
    while processed < max_jobs do
        -- XREADGROUP: ดึง jobs ที่ยังไม่มีใครเอา
        -- ">" = jobs ใหม่ที่ยังไม่ถูก deliver
        local result = client:xreadgroup(
            "GROUP", GROUP, consumer_name,
            "COUNT", 1,
            "BLOCK", 2000,  -- block 2 วินาที
            "STREAMS", STREAM, ">"
        )
        
        if result and result[1] then
            local stream_data = result[1]
            local events = stream_data[2]
            
            for _, entry in ipairs(events) do
                local job = parse_entry(entry)
                local job_id = job._id
                
                print(string.format("  %s: Processing job #%s (type=%s)",
                    consumer_name, job.job_id or "?", job.type or "?"))
                
                -- simulate processing
                local success = true
                -- ทำงานจริงที่นี่...
                
                if success then
                    -- ACK: ยืนยันว่า process สำเร็จ
                    client:xack(STREAM, GROUP, job_id)
                    print(string.format("    ACK: job #%s done", job_id))
                else
                    -- ไม่ ACK = job จะกลับไปอยู่ใน PEL
                    print(string.format("    NACK: job #%s failed", job_id))
                end
                
                processed = processed + 1
            end
        else
            print(string.format("%s: No jobs, timeout", consumer_name))
            break
        end
    end
    
    print(string.format("Consumer '%s' processed %d jobs",
        consumer_name, processed))
    client:quit()
end

-- รัน consumers (sequential สำหรับตัวอย่าง)
run_consumer(1)
run_consumer(2)
```

### ตัวอย่างที่ 12: Pending Entries List (PEL)

```lua
-- pending_entries.lua
local redis = require("redis")

local client = redis.connect("127.0.0.1", 6379)
local STREAM = "jobs:processing"
local GROUP  = "workers"

-- XPENDING - ดู pending messages
local pending = client:xpending(STREAM, GROUP, "-", "+", 10)

if pending and #pending > 0 then
    print("Pending messages:")
    for _, entry in ipairs(pending) do
        -- entry = {id, consumer, idle_ms, delivery_count}
        print(string.format("  ID: %s | Consumer: %s | Idle: %dms | Delivered: %dx",
            entry[1], entry[2],
            tonumber(entry[3]) or 0,
            tonumber(entry[4]) or 0))
    end
else
    print("No pending messages")
end

-- XCLAIM - รับ ownership ของ pending message ที่ค้างนาน
local IDLE_THRESHOLD = 30000  -- 30 seconds

local summary = client:xpending(STREAM, GROUP)
if summary and summary[1] then
    local pending_count = summary[1]
    print(string.format("\nTotal pending: %d", pending_count))
    
    -- ดู entries ที่ idle นาน
    local old_pending = client:xpending(STREAM, GROUP, "-", "+", 100)
    for _, entry in ipairs(old_pending or {}) do
        local msg_id       = entry[1]
        local consumer     = entry[2]
        local idle_ms      = tonumber(entry[3]) or 0
        local delivery_cnt = tonumber(entry[4]) or 0
        
        if idle_ms > IDLE_THRESHOLD then
            print(string.format("  Stale message %s from %s (idle=%ds, delivered=%dx)",
                msg_id, consumer, idle_ms // 1000, delivery_cnt))
            
            -- XCLAIM: โอน ownership ให้ recovery worker
            local claimed = client:xclaim(
                STREAM, GROUP, "recovery-worker", 
                IDLE_THRESHOLD, msg_id)
            if claimed and #claimed > 0 then
                print("  Claimed by recovery-worker")
            end
        end
    end
end

client:quit()
```

---

## 60.6 Reliable Queue with Acknowledgment

### ตัวอย่างที่ 13: Reliable Queue Pattern

```lua
-- reliable_queue.lua
local redis = require("redis")
local json = require("dkjson")

-- Reliable Queue ใช้ 2 lists:
-- jobs:pending  = รอ process
-- jobs:inflight = กำลัง process (ยังไม่ ACK)

local ReliableQueue = {}
ReliableQueue.__index = ReliableQueue

function ReliableQueue.new(name)
    local self = setmetatable({}, ReliableQueue)
    self.name = name
    self.pending_key  = "rq:" .. name .. ":pending"
    self.inflight_key = "rq:" .. name .. ":inflight"
    self.failed_key   = "rq:" .. name .. ":failed"
    self.client = redis.connect("127.0.0.1", 6379)
    return self
end

function ReliableQueue:enqueue(job)
    local payload = json.encode({
        id         = job.id or tostring(os.time()) .. math.random(1000),
        type       = job.type,
        data       = job.data,
        created_at = os.time(),
        attempts   = 0
    })
    self.client:lpush(self.pending_key, payload)
end

function ReliableQueue:dequeue(timeout)
    -- BRPOPLPUSH: atomic pop จาก pending, push ไปยัง inflight
    local payload = self.client:brpoplpush(
        self.pending_key, 
        self.inflight_key, 
        timeout or 5
    )
    if not payload then return nil end
    
    local ok, job = pcall(json.decode, payload)
    if not ok then return nil end
    
    job._payload = payload  -- เก็บ payload เดิมไว้สำหรับ ACK/NACK
    return job
end

function ReliableQueue:ack(job)
    -- ลบออกจาก inflight
    local count = self.client:lrem(self.inflight_key, 1, job._payload)
    return count > 0
end

function ReliableQueue:nack(job, reason)
    job.attempts = (job.attempts or 0) + 1
    job.last_error = reason
    job.failed_at  = os.time()
    
    -- ลบออกจาก inflight
    self.client:lrem(self.inflight_key, 1, job._payload)
    
    if job.attempts >= 3 then
        -- ย้ายไป dead letter queue
        self.client:lpush(self.failed_key, json.encode(job))
        print(string.format("  Job %s moved to failed queue (attempts=%d)",
            job.id, job.attempts))
    else
        -- ส่งกลับ pending สำหรับ retry
        job._payload = nil
        self.client:lpush(self.pending_key, json.encode(job))
        print(string.format("  Job %s requeued (attempt %d/3)",
            job.id, job.attempts))
    end
end

function ReliableQueue:recover_inflight()
    -- ย้าย inflight jobs กลับไป pending (เมื่อ worker crash)
    local recovered = 0
    while true do
        local payload = self.client:rpoplpush(
            self.inflight_key, self.pending_key)
        if not payload then break end
        recovered = recovered + 1
    end
    return recovered
end

function ReliableQueue:stats()
    return {
        pending  = self.client:llen(self.pending_key),
        inflight = self.client:llen(self.inflight_key),
        failed   = self.client:llen(self.failed_key),
    }
end

function ReliableQueue:close()
    self.client:quit()
end

-- ทดสอบ
local queue = ReliableQueue.new("emails")

-- ล้าง queue เก่า
queue.client:del(queue.pending_key, queue.inflight_key, queue.failed_key)

-- Enqueue jobs
for i = 1, 5 do
    queue:enqueue({
        type = "send_email",
        data = {
            to      = "user" .. i .. "@example.com",
            subject = "Test " .. i
        }
    })
end

print("Stats before:", queue:stats().pending, "pending")

-- Process jobs
for i = 1, 5 do
    local job = queue:dequeue(1)
    if job then
        print(string.format("Processing: %s to %s",
            job.type, job.data.to))
        
        -- Simulate: job 3 fails
        if i == 3 then
            queue:nack(job, "SMTP connection timeout")
        else
            queue:ack(job)
            print("  ACK: done")
        end
    end
end

local stats = queue:stats()
print(string.format("\nFinal stats: pending=%d inflight=%d failed=%d",
    stats.pending, stats.inflight, stats.failed))

queue:close()
```

---

## 60.7 Dead Letter Queue

### ตัวอย่างที่ 14: Dead Letter Queue System

```lua
-- dead_letter_queue.lua
local redis = require("redis")
local json = require("dkjson")

local DLQ_KEY     = "dlq:jobs"
local DLQ_LOG_KEY = "dlq:log"
local MAX_DLQ     = 1000  -- จำกัดขนาด DLQ

local client = redis.connect("127.0.0.1", 6379)
client:del(DLQ_KEY, DLQ_LOG_KEY)

-- ส่ง job ไป DLQ
local function send_to_dlq(job, reason)
    local dlq_entry = {
        original_job = job,
        reason       = reason,
        failed_at    = os.time(),
        attempts     = job.attempts or 0
    }
    
    -- เพิ่มใน DLQ พร้อม trim
    client:lpush(DLQ_KEY, json.encode(dlq_entry))
    client:ltrim(DLQ_KEY, 0, MAX_DLQ - 1)
    
    -- Log
    client:lpush(DLQ_LOG_KEY, string.format(
        "[%s] Job %s failed: %s",
        os.date("%Y-%m-%d %H:%M:%S"),
        job.id or "unknown",
        reason
    ))
    client:ltrim(DLQ_LOG_KEY, 0, 999)
end

-- ดู DLQ entries
local function inspect_dlq(limit)
    limit = limit or 10
    local entries = client:lrange(DLQ_KEY, 0, limit - 1)
    local results = {}
    for _, payload in ipairs(entries or {}) do
        local ok, entry = pcall(json.decode, payload)
        if ok then table.insert(results, entry) end
    end
    return results
end

-- Replay: ส่ง job กลับไป queue หลัก
local function replay_from_dlq(queue_key, count)
    count = count or 1
    local replayed = 0
    
    for _ = 1, count do
        local payload = client:rpop(DLQ_KEY)
        if not payload then break end
        
        local ok, entry = pcall(json.decode, payload)
        if ok and entry.original_job then
            local job = entry.original_job
            job.attempts = 0  -- reset attempts
            job.replayed_at = os.time()
            client:lpush(queue_key, json.encode(job))
            replayed = replayed + 1
        end
    end
    
    return replayed
end

-- ทดสอบ
local failed_jobs = {
    {id = "JOB-001", type = "email", attempts = 3, to = "bad@invalid"},
    {id = "JOB-002", type = "webhook", attempts = 3, url = "http://down.example.com"},
    {id = "JOB-003", type = "sms", attempts = 3, phone = "+invalid"},
}

print("Sending failed jobs to DLQ:")
send_to_dlq(failed_jobs[1], "Invalid email address")
send_to_dlq(failed_jobs[2], "Webhook endpoint timeout after 3 attempts")
send_to_dlq(failed_jobs[3], "Invalid phone number format")

print("DLQ size:", client:llen(DLQ_KEY))

-- Inspect
print("\nDLQ contents:")
for _, entry in ipairs(inspect_dlq()) do
    print(string.format("  [%s] Job %s: %s",
        os.date("%H:%M:%S", entry.failed_at),
        entry.original_job.id,
        entry.reason))
end

-- Replay ไป main queue
local main_queue = "jobs:main"
local n = replay_from_dlq(main_queue, 1)
print("\nReplayed", n, "job(s) from DLQ")
print("DLQ remaining:", client:llen(DLQ_KEY))
print("Main queue:", client:llen(main_queue))

-- ดู DLQ log
print("\nDLQ Log:")
local logs = client:lrange(DLQ_LOG_KEY, 0, 4)
for _, log in ipairs(logs or {}) do
    print("  " .. log)
end

client:quit()
```

---

## 60.8 Priority Queue ด้วย Sorted Sets

### ตัวอย่างที่ 15: Priority Queue

```lua
-- priority_queue.lua
local redis = require("redis")
local json = require("dkjson")

local PriorityQueue = {}
PriorityQueue.__index = PriorityQueue

function PriorityQueue.new(name)
    local self = setmetatable({}, PriorityQueue)
    self.key = "pq:" .. name
    self.client = redis.connect("127.0.0.1", 6379)
    return self
end

function PriorityQueue:enqueue(job, priority)
    -- score ต่ำ = priority สูง (ZRANGEBYSCORE ดึงน้อยสุดก่อน)
    -- ใช้ timestamp เพื่อให้ FIFO ภายใน priority เดียวกัน
    priority = priority or 5
    local score = priority * 1e13 + os.time() * 1000 + (os.clock() * 1000 % 1000)
    
    local payload = json.encode({
        id         = job.id or tostring(math.random(100000)),
        type       = job.type,
        data       = job.data,
        priority   = priority,
        created_at = os.time()
    })
    
    self.client:zadd(self.key, score, payload)
    return score
end

function PriorityQueue:dequeue()
    -- ZPOPMIN: ดึง element ที่มี score ต่ำสุด (priority สูงสุด)
    local result = self.client:zpopmin(self.key, 1)
    if not result or #result == 0 then return nil end
    
    local payload = result[1]
    local ok, job = pcall(json.decode, payload)
    if not ok then return nil end
    return job
end

function PriorityQueue:peek(count)
    count = count or 5
    local results = self.client:zrange(self.key, 0, count - 1, "WITHSCORES")
    local items = {}
    for i = 1, #(results or {}), 2 do
        local ok, job = pcall(json.decode, results[i])
        if ok then
            job._score = tonumber(results[i+1])
            table.insert(items, job)
        end
    end
    return items
end

function PriorityQueue:size()
    return self.client:zcard(self.key)
end

function PriorityQueue:close()
    self.client:quit()
end

-- ทดสอบ
local pq = PriorityQueue.new("tasks")
pq.client:del(pq.key)

-- เพิ่ม jobs ด้วย priorities ต่างกัน (1=highest, 5=lowest)
pq:enqueue({type = "maintenance",  data = "routine check"},      5)  -- low
pq:enqueue({type = "backup",       data = "daily backup"},       4)
pq:enqueue({type = "report",       data = "monthly report"},     3)
pq:enqueue({type = "payment",      data = "process payment"},    1)  -- critical
pq:enqueue({type = "notification", data = "send email"},         2)
pq:enqueue({type = "urgent_fix",   data = "production bug"},     1)  -- critical
pq:enqueue({type = "analytics",    data = "generate stats"},     5)  -- low

print("Priority Queue size:", pq:size())
print("\nQueue order (highest priority first):")
for _, item in ipairs(pq:peek(7)) do
    print(string.format("  P%d: %-15s %s",
        item.priority, item.type, item.data))
end

print("\nDequeuing in priority order:")
while pq:size() > 0 do
    local job = pq:dequeue()
    if job then
        print(string.format("  [P%d] %s: %s",
            job.priority, job.type, job.data))
    end
end

pq:close()
```

---

## 60.9 Job Retry กับ Exponential Backoff

### ตัวอย่างที่ 16: Retry Manager

```lua
-- retry_manager.lua
local redis = require("redis")
local json = require("dkjson")

local RetryManager = {}
RetryManager.__index = RetryManager

function RetryManager.new(name, opts)
    local self = setmetatable({}, RetryManager)
    self.name    = name
    self.queue   = "retry:" .. name .. ":queue"
    self.delayed = "retry:" .. name .. ":delayed"  -- sorted set
    self.dead    = "retry:" .. name .. ":dead"
    self.client  = redis.connect("127.0.0.1", 6379)
    
    self.max_attempts = opts and opts.max_attempts or 5
    self.base_delay   = opts and opts.base_delay or 1   -- seconds
    self.max_delay    = opts and opts.max_delay or 3600  -- 1 hour
    
    return self
end

-- Exponential backoff: delay = base * 2^(attempt-1) + jitter
function RetryManager:calc_delay(attempt)
    local delay = self.base_delay * (2 ^ (attempt - 1))
    -- เพิ่ม jitter เพื่อหลีกเลี่ยง thundering herd
    local jitter = math.random(0, math.floor(delay * 0.1))
    delay = math.min(delay + jitter, self.max_delay)
    return math.floor(delay)
end

function RetryManager:submit(job)
    job.attempts = job.attempts or 0
    job.id = job.id or tostring(os.time()) .. "_" .. math.random(9999)
    self.client:lpush(self.queue, json.encode(job))
end

function RetryManager:process_next()
    local payload = self.client:rpop(self.queue)
    if not payload then return nil end
    
    local ok, job = pcall(json.decode, payload)
    if not ok then return nil end
    
    return job
end

function RetryManager:retry(job, error_msg)
    job.attempts = (job.attempts or 0) + 1
    job.last_error = error_msg
    job.last_retry_at = os.time()
    
    if job.attempts >= self.max_attempts then
        -- ส่งไป dead letter queue
        job.dead_reason = "Max attempts reached"
        self.client:lpush(self.dead, json.encode(job))
        print(string.format("  Job %s dead after %d attempts: %s",
            job.id, job.attempts, error_msg))
        return false
    end
    
    -- คำนวณ delay
    local delay = self:calc_delay(job.attempts)
    local execute_at = os.time() + delay
    
    -- เก็บใน sorted set (score = execute_at timestamp)
    self.client:zadd(self.delayed, execute_at, json.encode(job))
    
    print(string.format("  Job %s scheduled for retry %d/%d in %ds",
        job.id, job.attempts, self.max_attempts, delay))
    return true
end

function RetryManager:move_ready_jobs()
    -- ย้าย delayed jobs ที่ถึงเวลาแล้วไปยัง queue
    local now = os.time()
    local ready = self.client:zrangebyscore(
        self.delayed, 0, now, "LIMIT", 0, 100)
    
    local moved = 0
    for _, payload in ipairs(ready or {}) do
        self.client:zrem(self.delayed, payload)
        self.client:lpush(self.queue, payload)
        moved = moved + 1
    end
    
    return moved
end

function RetryManager:stats()
    return {
        queued  = self.client:llen(self.queue),
        delayed = self.client:zcard(self.delayed),
        dead    = self.client:llen(self.dead),
    }
end

function RetryManager:close()
    self.client:quit()
end

-- ทดสอบ
local rm = RetryManager.new("api_calls", {
    max_attempts = 4,
    base_delay   = 1,
    max_delay    = 60
})

-- ล้างข้อมูลเก่า
rm.client:del(rm.queue, rm.delayed, rm.dead)

-- Submit jobs
rm:submit({type = "api_call", url = "https://api.example.com/users", id = "job-1"})
rm:submit({type = "api_call", url = "https://api.example.com/orders", id = "job-2"})

print("Initial stats:", rm:stats().queued, "queued")
print("\nSimulating failures:")

-- Simulate processing กับ failures
for round = 1, 4 do
    -- Move ready jobs ก่อน
    local moved = rm:move_ready_jobs()
    if moved > 0 then
        print(string.format("\nRound %d: Moved %d delayed jobs to queue", 
            round, moved))
    end
    
    local job = rm:process_next()
    if job then
        print(string.format("Round %d: Processing job '%s' (attempt %d)",
            round, job.id, (job.attempts or 0) + 1))
        
        -- Simulate failure
        local success = (round == 4)  -- สำเร็จใน round 4
        if success then
            print("  SUCCESS!")
        else
            rm:retry(job, "Connection refused (simulated)")
        end
    end
end

print("\nFinal stats:", 
    "queued=" .. rm:stats().queued,
    "delayed=" .. rm:stats().delayed,
    "dead=" .. rm:stats().dead)

rm:close()
```

---

## 60.10 Worker Pool Pattern

### ตัวอย่างที่ 17: Worker Pool

```lua
-- worker_pool.lua
local redis = require("redis")
local json = require("dkjson")

-- Worker Pool ด้วย Lua coroutines (single-threaded simulation)

local WorkerPool = {}
WorkerPool.__index = WorkerPool

function WorkerPool.new(queue_name, worker_count)
    local self = setmetatable({}, WorkerPool)
    self.queue_name   = queue_name
    self.worker_count = worker_count or 4
    self.workers      = {}
    self.running      = false
    
    -- สร้าง Redis connection สำหรับแต่ละ worker
    for i = 1, worker_count do
        self.workers[i] = {
            id         = i,
            client     = redis.connect("127.0.0.1", 6379),
            processed  = 0,
            errors     = 0,
            busy       = false
        }
    end
    
    return self
end

function WorkerPool:register_handler(job_type, handler_fn)
    if not self.handlers then self.handlers = {} end
    self.handlers[job_type] = handler_fn
end

function WorkerPool:process_job(worker, job)
    worker.busy = true
    local start_time = os.clock()
    
    local handler = self.handlers and self.handlers[job.type]
    
    local success, err
    if handler then
        success, err = pcall(handler, job)
    else
        success, err = false, "No handler for job type: " .. (job.type or "nil")
    end
    
    local elapsed = os.clock() - start_time
    
    if success then
        worker.processed = worker.processed + 1
    else
        worker.errors = worker.errors + 1
        -- Log error
        worker.client:lpush("errors:" .. self.queue_name,
            json.encode({
                job_id    = job.id,
                job_type  = job.type,
                error     = tostring(err),
                worker_id = worker.id,
                ts        = os.time()
            })
        )
    end
    
    worker.busy = false
    return success, elapsed
end

function WorkerPool:run_once()
    -- แต่ละ worker พยายามดึง job
    for _, worker in ipairs(self.workers) do
        if not worker.busy then
            local payload = worker.client:rpop(self.queue_name)
            if payload then
                local ok, job = pcall(json.decode, payload)
                if ok then
                    local success, elapsed = self:process_job(worker, job)
                    print(string.format(
                        "  Worker-%d: %s job '%s' in %.3fms",
                        worker.id,
                        success and "OK" or "FAIL",
                        job.type or "?",
                        elapsed * 1000))
                end
            end
        end
    end
end

function WorkerPool:stats()
    local total_processed = 0
    local total_errors = 0
    for _, w in ipairs(self.workers) do
        total_processed = total_processed + w.processed
        total_errors = total_errors + w.errors
    end
    return {
        workers   = self.worker_count,
        processed = total_processed,
        errors    = total_errors,
    }
end

function WorkerPool:close()
    for _, worker in ipairs(self.workers) do
        worker.client:quit()
    end
end

-- ทดสอบ
local pool = WorkerPool.new("jobs:worker", 3)

-- ล้าง queue
pool.workers[1].client:del("jobs:worker")

-- Register handlers
pool:register_handler("send_email", function(job)
    -- simulate email sending
    if math.random() < 0.1 then
        error("SMTP server unavailable")
    end
    return true
end)

pool:register_handler("resize_image", function(job)
    -- simulate image processing
    return true
end)

pool:register_handler("generate_pdf", function(job)
    if math.random() < 0.2 then
        error("PDF generation failed")
    end
    return true
end)

-- Enqueue jobs
local producer = redis.connect("127.0.0.1", 6379)
local job_types = {"send_email", "resize_image", "generate_pdf"}

for i = 1, 12 do
    producer:lpush("jobs:worker", json.encode({
        id   = "job-" .. i,
        type = job_types[((i-1) % 3) + 1],
        data = {n = i}
    }))
end
producer:quit()

print("Processing 12 jobs with 3 workers:")
-- Process ทุก jobs
for round = 1, 6 do
    print(string.format("\nRound %d:", round))
    pool:run_once()
end

local stats = pool:stats()
print(string.format("\nPool stats: %d processed, %d errors",
    stats.processed, stats.errors))

pool:close()
```

---

## 60.11 Rate Limiting

### ตัวอย่างที่ 18: Rate Limiter ด้วย Redis

```lua
-- rate_limiter.lua
local redis = require("redis")

local client = redis.connect("127.0.0.1", 6379)

-- Fixed Window Rate Limiter
local function rate_limit_fixed(user_id, max_requests, window_seconds)
    local key = string.format("rl:fixed:%s:%d",
        user_id, math.floor(os.time() / window_seconds))
    
    local count = client:incr(key)
    if count == 1 then
        client:expire(key, window_seconds)
    end
    
    return count <= max_requests, count, max_requests
end

-- Sliding Window Rate Limiter (accurate แต่ใช้ memory มากกว่า)
local function rate_limit_sliding(user_id, max_requests, window_seconds)
    local key = "rl:sliding:" .. user_id
    local now = os.time()
    local window_start = now - window_seconds
    
    -- ลบ entries เก่า
    client:zremrangebyscore(key, 0, window_start)
    
    -- นับ requests ใน window
    local count = client:zcard(key)
    
    if count < max_requests then
        -- เพิ่ม request นี้
        client:zadd(key, now, tostring(now) .. "_" .. math.random(99999))
        client:expire(key, window_seconds + 1)
        return true, count + 1, max_requests
    end
    
    return false, count, max_requests
end

-- Token Bucket Rate Limiter
local function rate_limit_token_bucket(user_id, capacity, refill_rate)
    local key_tokens = "rl:bucket:" .. user_id .. ":tokens"
    local key_last   = "rl:bucket:" .. user_id .. ":last"
    
    local now = os.time()
    local last_refill = tonumber(client:get(key_last)) or now
    local tokens = tonumber(client:get(key_tokens)) or capacity
    
    -- เติม tokens
    local elapsed = now - last_refill
    tokens = math.min(capacity, tokens + elapsed * refill_rate)
    
    if tokens >= 1 then
        tokens = tokens - 1
        client:set(key_tokens, tostring(tokens))
        client:set(key_last, tostring(now))
        client:expire(key_tokens, 3600)
        client:expire(key_last, 3600)
        return true, math.floor(tokens)
    end
    
    return false, 0
end

-- ทดสอบ Rate Limiters
print("=== Fixed Window (5 req/10s) ===")
client:del("rl:fixed:user123:" .. math.floor(os.time() / 10))
for i = 1, 7 do
    local allowed, count, max = rate_limit_fixed("user123", 5, 10)
    print(string.format("  Request %d: %s (%d/%d)",
        i, allowed and "ALLOWED" or "BLOCKED", count, max))
end

print("\n=== Sliding Window (5 req/10s) ===")
client:del("rl:sliding:user456")
for i = 1, 7 do
    local allowed, count, max = rate_limit_sliding("user456", 5, 10)
    print(string.format("  Request %d: %s (%d/%d)",
        i, allowed and "ALLOWED" or "BLOCKED", count, max))
end

print("\n=== Token Bucket (capacity=5, refill=1/s) ===")
client:del("rl:bucket:user789:tokens", "rl:bucket:user789:last")
for i = 1, 7 do
    local allowed, remaining = rate_limit_token_bucket("user789", 5, 1.0)
    print(string.format("  Request %d: %s (tokens remaining: %d)",
        i, allowed and "ALLOWED" or "BLOCKED", remaining))
end

client:quit()
```

---

## 60.12 Lua Scripts ใน Redis (EVAL)

### ตัวอย่างที่ 19: Atomic Operations ด้วย EVAL

```lua
-- redis_eval.lua
local redis = require("redis")
local client = redis.connect("127.0.0.1", 6379)

-- EVAL ให้เรา run Lua scripts บน Redis server แบบ atomic

-- Script 1: Atomic get-and-increment
local incr_script = [[
    local current = redis.call('GET', KEYS[1])
    current = tonumber(current) or 0
    local new_val = current + tonumber(ARGV[1])
    redis.call('SET', KEYS[1], new_val)
    return new_val
]]

client:set("counter", "10")
local result = client:eval(incr_script, 1, "counter", "5")
print("Atomic increment by 5:", result)  -- 15

-- Script 2: Conditional set (ถ้าค่าปัจจุบัน < threshold)
local cond_set_script = [[
    local current = tonumber(redis.call('GET', KEYS[1])) or 0
    local threshold = tonumber(ARGV[1])
    local new_val = tonumber(ARGV[2])
    
    if current < threshold then
        redis.call('SET', KEYS[1], new_val)
        return 1  -- สำเร็จ
    end
    return 0  -- ไม่ได้เซ็ต
]]

client:set("stock", "5")
local set_result = client:eval(cond_set_script, 1, "stock", "10", "100")
print("Set if < 10:", set_result == 1 and "SET" or "NOT SET")
print("Stock now:", client:get("stock"))  -- 100

-- Script 3: Distributed lock (Redlock simplified)
local acquire_lock_script = [[
    local key = KEYS[1]
    local token = ARGV[1]
    local ttl = tonumber(ARGV[2])
    
    if redis.call('EXISTS', key) == 0 then
        redis.call('SET', key, token, 'PX', ttl)
        return 1
    end
    return 0
]]

local release_lock_script = [[
    local key = KEYS[1]
    local token = ARGV[1]
    
    if redis.call('GET', key) == token then
        redis.call('DEL', key)
        return 1
    end
    return 0
]]

-- ใช้ distributed lock
local lock_key = "lock:resource"
local my_token = tostring(os.time()) .. "_" .. math.random(99999)

local acquired = client:eval(acquire_lock_script, 1, lock_key, my_token, 10000)
print("\nLock acquired:", acquired == 1)

-- ทำงานที่ต้องการ exclusive access
print("Doing critical work...")

-- Release lock
local released = client:eval(release_lock_script, 1, lock_key, my_token)
print("Lock released:", released == 1)

client:quit()
```

### ตัวอย่างที่ 20: EVALSHA - Script Caching

```lua
-- evalsha.lua
local redis = require("redis")
local client = redis.connect("127.0.0.1", 6379)

-- SCRIPT LOAD: โหลด script และได้ SHA1 กลับมา
-- EVALSHA: เรียก script ด้วย SHA1 (ประหยัด bandwidth)

local dequeue_script = [[
    -- Atomic dequeue: ดึง job และ track ใน inflight set
    local queue_key    = KEYS[1]
    local inflight_key = KEYS[2]
    local job_id_key   = KEYS[3]
    
    local payload = redis.call('RPOP', queue_key)
    if not payload then return nil end
    
    -- เพิ่มใน inflight hash
    local job_id = redis.call('INCR', job_id_key)
    redis.call('HSET', inflight_key, job_id, payload)
    
    return {job_id, payload}
]]

-- โหลด script
local sha = client:script("LOAD", dequeue_script)
print("Script SHA1:", sha)

-- ตรวจสอบว่า script อยู่ใน cache
local exists = client:script("EXISTS", sha)
print("Script cached:", exists[1] == 1)

-- สร้าง queue data
local QUEUE = "mq:test"
local INFLIGHT = "mq:test:inflight"
local JOB_ID = "mq:test:job_id"

client:del(QUEUE, INFLIGHT, JOB_ID)

-- Enqueue
for i = 1, 3 do
    client:lpush(QUEUE, string.format('{"type":"job","n":%d}', i))
end

-- Dequeue ด้วย EVALSHA
print("\nDequeuing with EVALSHA:")
for i = 1, 4 do
    local result = client:evalsha(sha, 3, QUEUE, INFLIGHT, JOB_ID)
    if result then
        print(string.format("  Job #%s: %s", result[1], result[2]))
    else
        print("  Queue empty")
    end
end

-- ดู inflight jobs
local inflight = client:hgetall(INFLIGHT)
print("\nInflight jobs:")
for k, v in pairs(inflight or {}) do
    print(string.format("  ID=%s Payload=%s", k, v))
end

-- SCRIPT FLUSH - ล้าง cache ทั้งหมด (ระวัง!)
-- client:script("FLUSH")

client:quit()
```

---

## 60.13 Monitoring และ Metrics

### ตัวอย่างที่ 21: Queue Metrics

```lua
-- queue_metrics.lua
local redis = require("redis")
local json = require("dkjson")

local client = redis.connect("127.0.0.1", 6379)

local Metrics = {}
Metrics.__index = Metrics

function Metrics.new(namespace)
    local self = setmetatable({}, Metrics)
    self.ns = namespace or "metrics"
    self.client = client
    return self
end

function Metrics:increment(name, value)
    value = value or 1
    return self.client:incrbyfloat(self.ns .. ":" .. name, value)
end

function Metrics:gauge(name, value)
    self.client:set(self.ns .. ":" .. name, value)
end

function Metrics:record_time(name, duration_ms)
    local key = self.ns .. ":timing:" .. name
    -- เก็บเป็น sorted set (score = timestamp, value = duration)
    self.client:zadd(key, os.time(), duration_ms)
    -- เก็บแค่ 1000 entries ล่าสุด
    self.client:zremrangebyrank(key, 0, -1001)
end

function Metrics:get_stats(name)
    local timings = self.client:zrange(
        self.ns .. ":timing:" .. name, 0, -1)
    if not timings or #timings == 0 then
        return nil
    end
    
    -- คำนวณ percentiles
    local values = {}
    for _, v in ipairs(timings) do
        table.insert(values, tonumber(v) or 0)
    end
    table.sort(values)
    
    local function percentile(sorted, p)
        local idx = math.ceil(#sorted * p / 100)
        return sorted[math.max(1, idx)]
    end
    
    local sum = 0
    for _, v in ipairs(values) do sum = sum + v end
    
    return {
        count = #values,
        avg   = sum / #values,
        min   = values[1],
        max   = values[#values],
        p50   = percentile(values, 50),
        p95   = percentile(values, 95),
        p99   = percentile(values, 99),
    }
end

-- simulate queue metrics
local metrics = Metrics.new("queue")

-- Simulate processing times
math.randomseed(42)
for i = 1, 100 do
    local duration = 10 + math.random(0, 90) + (math.random() < 0.05 and 500 or 0)
    metrics:record_time("job_duration", duration)
    
    if math.random() < 0.05 then
        metrics:increment("errors")
    else
        metrics:increment("processed")
    end
end

metrics:gauge("queue_depth", 42)
metrics:gauge("workers_active", 8)

-- แสดง stats
local stats = metrics:get_stats("job_duration")
if stats then
    print("Job Duration Stats:")
    print(string.format("  Count: %d", stats.count))
    print(string.format("  Avg:   %.1fms", stats.avg))
    print(string.format("  Min:   %.1fms", stats.min))
    print(string.format("  Max:   %.1fms", stats.max))
    print(string.format("  p50:   %.1fms", stats.p50))
    print(string.format("  p95:   %.1fms", stats.p95))
    print(string.format("  p99:   %.1fms", stats.p99))
end

print("\nCounters:")
print("  Processed:", client:get("queue:processed"))
print("  Errors:", client:get("queue:errors"))
print("  Queue depth:", client:get("queue:queue_depth"))

client:quit()
```

---

## 60.14 ตัวอย่างครบวงจร: Task Queue System

### ตัวอย่างที่ 22: Complete Task Queue

```lua
-- task_queue_system.lua
local redis = require("redis")
local json = require("dkjson")

local TaskQueue = {}
TaskQueue.__index = TaskQueue

function TaskQueue.new(name, opts)
    local self = setmetatable({}, TaskQueue)
    opts = opts or {}
    
    self.name        = name
    self.client      = redis.connect("127.0.0.1", 6379)
    self.max_retries = opts.max_retries or 3
    self.handlers    = {}
    
    -- Key names
    self.keys = {
        pending  = "tq:" .. name .. ":pending",
        delayed  = "tq:" .. name .. ":delayed",
        inflight = "tq:" .. name .. ":inflight",
        dead     = "tq:" .. name .. ":dead",
        stats    = "tq:" .. name .. ":stats",
    }
    
    return self
end

function TaskQueue:submit(job_type, data, opts)
    opts = opts or {}
    local job = {
        id         = tostring(os.time()) .. "_" .. math.random(99999),
        type       = job_type,
        data       = data,
        priority   = opts.priority or 5,
        attempts   = 0,
        max_retries = opts.max_retries or self.max_retries,
        created_at = os.time(),
        scheduled_at = opts.delay and (os.time() + opts.delay) or nil
    }
    
    if job.scheduled_at then
        -- Delayed job
        self.client:zadd(self.keys.delayed, job.scheduled_at, json.encode(job))
    else
        -- ใช้ priority score
        local score = job.priority * 1e13 + os.time()
        self.client:zadd(self.keys.pending, score, json.encode(job))
    end
    
    self.client:hincrby(self.keys.stats, "submitted", 1)
    return job.id
end

function TaskQueue:register(job_type, handler)
    self.handlers[job_type] = handler
end

function TaskQueue:_promote_delayed()
    local now = os.time()
    local ready = self.client:zrangebyscore(self.keys.delayed, 0, now, "LIMIT", 0, 50)
    for _, payload in ipairs(ready or {}) do
        self.client:zrem(self.keys.delayed, payload)
        local ok, job = pcall(json.decode, payload)
        if ok then
            local score = job.priority * 1e13 + os.time()
            self.client:zadd(self.keys.pending, score, json.encode(job))
        end
    end
end

function TaskQueue:process_next()
    self:_promote_delayed()
    
    -- ดึง job ที่ priority สูงสุด
    local result = self.client:zpopmin(self.keys.pending, 1)
    if not result or #result == 0 then return nil, "empty" end
    
    local payload = result[1]
    local ok, job = pcall(json.decode, payload)
    if not ok then return nil, "invalid" end
    
    -- Track inflight
    self.client:hset(self.keys.inflight, job.id, payload)
    
    -- Process
    local handler = self.handlers[job.type]
    local success, err
    
    if handler then
        success, err = pcall(handler, job.data, job)
    else
        success, err = false, "No handler for: " .. job.type
    end
    
    -- Remove from inflight
    self.client:hdel(self.keys.inflight, job.id)
    
    if success then
        self.client:hincrby(self.keys.stats, "completed", 1)
        return job, nil
    else
        job.attempts = job.attempts + 1
        job.last_error = tostring(err)
        
        if job.attempts <= job.max_retries then
            -- Retry with backoff
            local delay = 2 ^ (job.attempts - 1)
            local retry_at = os.time() + delay
            self.client:zadd(self.keys.delayed, retry_at, json.encode(job))
            self.client:hincrby(self.keys.stats, "retried", 1)
        else
            -- Dead letter
            self.client:lpush(self.keys.dead, json.encode(job))
            self.client:hincrby(self.keys.stats, "failed", 1)
        end
        
        return nil, err
    end
end

function TaskQueue:get_stats()
    local s = self.client:hgetall(self.keys.stats) or {}
    return {
        submitted = tonumber(s["submitted"]) or 0,
        completed = tonumber(s["completed"]) or 0,
        retried   = tonumber(s["retried"])   or 0,
        failed    = tonumber(s["failed"])    or 0,
        pending   = self.client:zcard(self.keys.pending),
        delayed   = self.client:zcard(self.keys.delayed),
        inflight  = self.client:hlen(self.keys.inflight),
        dead      = self.client:llen(self.keys.dead),
    }
end

function TaskQueue:flush()
    for _, key in pairs(self.keys) do
        self.client:del(key)
    end
end

function TaskQueue:close()
    self.client:quit()
end

-- ทดสอบ
local tq = TaskQueue.new("demo", {max_retries = 2})
tq:flush()

-- Register handlers
tq:register("email", function(data, job)
    if data.to:match("@invalid$") then
        error("Invalid email domain")
    end
    print(string.format("    Email sent to %s", data.to))
    return true
end)

tq:register("webhook", function(data, job)
    print(string.format("    Webhook: POST to %s", data.url))
    return true
end)

tq:register("report", function(data, job)
    print(string.format("    Report '%s' generated", data.title))
    return true
end)

-- Submit jobs
tq:submit("email", {to = "alice@example.com"}, {priority = 2})
tq:submit("webhook", {url = "https://hook.example.com/events"}, {priority = 1})
tq:submit("email", {to = "bad@invalid"}, {priority = 2})  -- จะ fail
tq:submit("report", {title = "Monthly Summary"}, {priority = 3})
tq:submit("email", {to = "bob@example.com"}, {priority = 2, delay = 1})  -- delayed

print("=== Processing Queue ===")
for round = 1, 8 do
    local job, err = tq:process_next()
    if job then
        print(string.format("Round %d: OK - %s", round, job.type))
    elseif err == "empty" then
        print(string.format("Round %d: Queue empty, waiting...", round))
        -- ในตัวอย่าง simulate delay ด้วยการรอ
        -- os.execute("sleep 1")
        break
    else
        print(string.format("Round %d: FAIL - %s", round, err))
    end
end

print("\n=== Final Stats ===")
local stats = tq:get_stats()
for k, v in pairs(stats) do
    print(string.format("  %-12s: %d", k, v))
end

tq:close()
```

---

## 60.15 ตัวอย่างที่ 23: Pub/Sub Event Bus

```lua
-- event_bus.lua
-- Event Bus สำหรับ microservices

local redis = require("redis")
local json = require("dkjson")

local EventBus = {}
EventBus.__index = EventBus

function EventBus.new()
    local self = setmetatable({}, EventBus)
    self.pub_client = redis.connect("127.0.0.1", 6379)
    self.sub_client = redis.connect("127.0.0.1", 6379)
    self.handlers   = {}
    self.middleware  = {}
    return self
end

function EventBus:use(middleware_fn)
    table.insert(self.middleware, middleware_fn)
end

function EventBus:on(event_pattern, handler)
    if not self.handlers[event_pattern] then
        self.handlers[event_pattern] = {}
    end
    table.insert(self.handlers[event_pattern], handler)
end

function EventBus:emit(event_name, data)
    local event = {
        name      = event_name,
        data      = data,
        timestamp = os.time(),
        id        = tostring(os.time()) .. "_" .. math.random(99999)
    }
    
    -- รัน middleware
    for _, mw in ipairs(self.middleware) do
        event = mw(event) or event
    end
    
    local payload = json.encode(event)
    local receivers = self.pub_client:publish("events:" .. event_name, payload)
    return receivers
end

function EventBus:listen(timeout)
    -- Subscribe to all events
    self.sub_client:psubscribe("events:*")
    
    local count = 0
    for msg in self.sub_client:receive() do
        if msg.kind == "pmessage" then
            local ok, event = pcall(json.decode, msg.payload)
            if ok then
                -- หา handlers ที่ match
                local event_name = event.name
                for pattern, handlers in pairs(self.handlers) do
                    -- simple pattern matching
                    local match = false
                    if pattern == event_name then
                        match = true
                    elseif pattern:sub(-1) == "*" then
                        local prefix = pattern:sub(1, -2)
                        match = event_name:sub(1, #prefix) == prefix
                    end
                    
                    if match then
                        for _, handler in ipairs(handlers) do
                            pcall(handler, event.data, event)
                        end
                    end
                end
                
                count = count + 1
                if count >= (timeout or 5) then break end
            end
        elseif msg.kind == "psubscribe" then
            print("Listening on pattern:", msg.channel)
        end
    end
    
    self.sub_client:punsubscribe()
end

function EventBus:close()
    self.pub_client:quit()
    self.sub_client:quit()
end

-- ใช้งาน
local bus = EventBus.new()

-- Middleware: เพิ่ม logging
bus:use(function(event)
    print(string.format("  [LOG] %s at %s",
        event.name, os.date("%H:%M:%S", event.timestamp)))
    return event
end)

-- Register event handlers
bus:on("user.created", function(data, event)
    print(string.format("    → Send welcome email to %s", data.email))
end)

bus:on("user.created", function(data, event)
    print(string.format("    → Create user profile for %s", data.username))
end)

bus:on("order.*", function(data, event)
    print(string.format("    → Order event: %s (order_id=%s)",
        event.name, data.order_id or "?"))
end)

bus:on("payment.completed", function(data, event)
    print(string.format("    → Update account balance: $%.2f", data.amount))
end)

-- Emit events (ต้องมี subscriber แล้ว)
-- ในตัวอย่างจริงจะรันใน process แยก
print("Event Bus example (emit side):")
local n
n = bus:emit("user.created", {username = "alice", email = "alice@example.com"})
print(string.format("user.created delivered to %d subscribers", n))

n = bus:emit("order.created", {order_id = "ORD-001", total = 99.99})
print(string.format("order.created delivered to %d subscribers", n))

n = bus:emit("payment.completed", {order_id = "ORD-001", amount = 99.99})
print(string.format("payment.completed delivered to %d subscribers", n))

bus:close()
```

---

## 60.16 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 23: Pipelines และ Batch Commands

```lua
-- redis_pipeline.lua
local redis = require("redis")
local client = redis.connect("127.0.0.1", 6379)

-- Pipeline: ส่ง commands หลายตัวพร้อมกัน (ลด network round-trips)

-- แบบไม่ใช้ pipeline (n round-trips)
local t1 = os.clock()
client:del("pl:test")
for i = 1, 1000 do
    client:lpush("pl:test", "item_" .. i)
end
local t_no_pipeline = os.clock() - t1
print(string.format("Without pipeline: %.4fs", t_no_pipeline))

-- แบบใช้ pipeline (1 round-trip)
client:del("pl:test2")
local t2 = os.clock()
local responses = client:pipeline(function(pipe)
    for i = 1, 1000 do
        pipe:lpush("pl:test2", "item_" .. i)
    end
end)
local t_pipeline = os.clock() - t2
print(string.format("With pipeline:    %.4fs (%.1fx faster)",
    t_pipeline, t_no_pipeline / math.max(t_pipeline, 0.0001)))

print("Pipeline responses:", #(responses or {}))
print("List length:", client:llen("pl:test2"))

-- Multi-Get ด้วย pipeline
client:mset("k1", "v1", "k2", "v2", "k3", "v3")

local values = {}
client:pipeline(function(pipe)
    pipe:get("k1")
    pipe:get("k2")
    pipe:get("k3")
    pipe:get("k_notexist")
end)

-- cleanup
client:del("pl:test", "pl:test2", "k1", "k2", "k3")
client:quit()
```

### ตัวอย่างที่ 24: Sorted Set สำหรับ Leaderboard

```lua
-- leaderboard.lua
local redis = require("redis")
local client = redis.connect("127.0.0.1", 6379)
local LB_KEY = "leaderboard:game1"
client:del(LB_KEY)

-- เพิ่มคะแนน
local players = {
    {"alice", 4500}, {"bob", 3200}, {"carol", 5800},
    {"dave", 2100},  {"eve", 4900}, {"frank", 3700},
    {"grace", 6200}, {"henry", 1500}
}
for _, p in ipairs(players) do
    client:zadd(LB_KEY, p[2], p[1])
end

-- Top 5 (score สูงสุดก่อน)
print("Top 5 Players:")
local top = client:zrevrange(LB_KEY, 0, 4, "WITHSCORES")
for i = 1, #(top or {}), 2 do
    local rank = (i + 1) // 2
    print(string.format("  #%d %-10s %s pts", rank, top[i], top[i+1]))
end

-- Rank ของ player
local alice_rank = client:zrevrank(LB_KEY, "alice")
print(string.format("\nAlice's rank: #%d", (alice_rank or 0) + 1))

-- Score ของ player
local alice_score = client:zscore(LB_KEY, "alice")
print(string.format("Alice's score: %s", alice_score))

-- อัปเดตคะแนน
client:zincrby(LB_KEY, 500, "alice")
print("Alice after +500:", client:zscore(LB_KEY, "alice"))

-- Players ใน score range
print("\nPlayers 3000-5000 pts:")
local mid = client:zrangebyscore(LB_KEY, 3000, 5000, "WITHSCORES")
for i = 1, #(mid or {}), 2 do
    print(string.format("  %-10s %s pts", mid[i], mid[i+1]))
end

client:del(LB_KEY)
client:quit()
```

### ตัวอย่างที่ 25: Session Management

```lua
-- session_manager.lua
local redis = require("redis")
local json = require("dkjson")

local SessionManager = {}
SessionManager.__index = SessionManager

function SessionManager.new(ttl_seconds)
    local self = setmetatable({}, SessionManager)
    self.client = redis.connect("127.0.0.1", 6379)
    self.ttl    = ttl_seconds or 3600  -- 1 hour default
    self.prefix = "session:"
    return self
end

function SessionManager:_gen_token()
    -- สร้าง random session token
    local chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    local token = {}
    for _ = 1, 32 do
        local idx = math.random(1, #chars)
        table.insert(token, chars:sub(idx, idx))
    end
    return table.concat(token)
end

function SessionManager:create(user_id, data)
    local token = self:_gen_token()
    local key = self.prefix .. token
    
    local session = {
        user_id    = user_id,
        data       = data or {},
        created_at = os.time(),
        last_seen  = os.time()
    }
    
    self.client:setex(key, self.ttl, json.encode(session))
    return token
end

function SessionManager:get(token)
    local key = self.prefix .. token
    local payload = self.client:get(key)
    if not payload then return nil end
    
    local ok, session = pcall(json.decode, payload)
    if not ok then return nil end
    
    -- Refresh TTL (sliding window)
    session.last_seen = os.time()
    self.client:setex(key, self.ttl, json.encode(session))
    return session
end

function SessionManager:destroy(token)
    return self.client:del(self.prefix .. token) > 0
end

function SessionManager:update(token, new_data)
    local session = self:get(token)
    if not session then return false end
    
    for k, v in pairs(new_data) do
        session.data[k] = v
    end
    
    local key = self.prefix .. token
    self.client:setex(key, self.ttl, json.encode(session))
    return true
end

function SessionManager:close()
    self.client:quit()
end

-- ทดสอบ
local sm = SessionManager.new(1800)
math.randomseed(os.time())

-- สร้าง session
local token = sm:create(101, {role = "admin", ip = "192.168.1.1"})
print("Session created:", token:sub(1,8) .. "...")

-- อ่าน session
local session = sm:get(token)
if session then
    print(string.format("User: %d, Role: %s",
        session.user_id, session.data.role))
end

-- อัปเดต session data
sm:update(token, {last_page = "/dashboard"})
local updated = sm:get(token)
print("Last page:", updated and updated.data.last_page or "nil")

-- ลบ session
local destroyed = sm:destroy(token)
print("Session destroyed:", destroyed)
print("Session after destroy:", sm:get(token) == nil and "nil" or "exists")

sm:close()
```

### ตัวอย่างที่ 26: Cache-Aside Pattern

```lua
-- cache_aside.lua
local redis = require("redis")
local json = require("dkjson")
local sqlite3 = require("lsqlite3")

-- Cache-Aside: ดึงจาก cache ก่อน ถ้าไม่มีค่อยดึงจาก database

local Cache = {}
Cache.__index = Cache

function Cache.new(ttl)
    local self = setmetatable({}, Cache)
    self.redis = redis.connect("127.0.0.1", 6379)
    self.db    = sqlite3.open(":memory:")
    self.ttl   = ttl or 300  -- 5 minutes
    self.hits  = 0
    self.misses = 0
    
    -- สร้าง test database
    self.db:exec([[
        CREATE TABLE users (
            id    INTEGER PRIMARY KEY,
            name  TEXT,
            email TEXT
        );
        INSERT INTO users VALUES
            (1, 'Alice', 'alice@example.com'),
            (2, 'Bob',   'bob@example.com'),
            (3, 'Carol', 'carol@example.com');
    ]])
    
    return self
end

function Cache:get_user(user_id)
    local cache_key = "user:" .. user_id
    
    -- 1. ลอง cache ก่อน
    local cached = self.redis:get(cache_key)
    if cached then
        self.hits = self.hits + 1
        local ok, data = pcall(json.decode, cached)
        if ok then return data, "cache" end
    end
    
    -- 2. Cache miss: ดึงจาก database
    self.misses = self.misses + 1
    local user
    local stmt = self.db:prepare("SELECT * FROM users WHERE id = ?")
    stmt:bind_values(user_id)
    if stmt:step() == sqlite3.ROW then
        user = {
            id    = stmt:get_value(0),
            name  = stmt:get_value(1),
            email = stmt:get_value(2)
        }
    end
    stmt:finalize()
    
    if user then
        -- 3. เก็บลง cache
        self.redis:setex(cache_key, self.ttl, json.encode(user))
    end
    
    return user, "db"
end

function Cache:invalidate(user_id)
    return self.redis:del("user:" .. user_id) > 0
end

function Cache:stats()
    local total = self.hits + self.misses
    return {
        hits     = self.hits,
        misses   = self.misses,
        hit_rate = total > 0 and (self.hits / total * 100) or 0
    }
end

function Cache:close()
    self.redis:quit()
    self.db:close()
end

-- ทดสอบ
local cache = Cache.new(60)

-- First access (cache miss)
for id = 1, 3 do
    local user, source = cache:get_user(id)
    if user then
        print(string.format("User %d from %s: %s", id, source, user.name))
    end
end

-- Second access (cache hit)
print("\nSecond access:")
for id = 1, 3 do
    local user, source = cache:get_user(id)
    if user then
        print(string.format("User %d from %s: %s", id, source, user.name))
    end
end

local stats = cache:stats()
print(string.format("\nCache stats: %d hits, %d misses, %.1f%% hit rate",
    stats.hits, stats.misses, stats.hit_rate))

-- Invalidate cache
cache:invalidate(1)
print("\nAfter invalidating user 1:")
local user, source = cache:get_user(1)
print(string.format("User 1 from %s: %s", source, user and user.name or "nil"))

cache:close()
```

### ตัวอย่างที่ 27: HyperLogLog (Approximate Counting)

```lua
-- hyperloglog.lua
-- HyperLogLog: นับ unique items แบบ approximate ใช้ memory น้อยมาก
local redis = require("redis")
local client = redis.connect("127.0.0.1", 6379)

local DAU_KEY = "hll:dau:"  -- Daily Active Users

-- Simulate user visits
local function record_visit(date, user_id)
    local key = DAU_KEY .. date
    client:pfadd(key, tostring(user_id))
    client:expire(key, 90 * 24 * 3600)  -- keep 90 days
end

-- สร้างข้อมูล test
math.randomseed(42)
local dates = {"2025-01-01", "2025-01-02", "2025-01-03"}
local user_pool = 1000  -- มี users ทั้งหมด 1000 คน

for _, date in ipairs(dates) do
    -- แต่ละวันมี users ประมาณ 300-500 คน visit
    local visits = math.random(300, 500)
    for _ = 1, visits do
        record_visit(date, math.random(1, user_pool))
    end
end

-- นับ DAU แต่ละวัน
print("Daily Active Users (approximate):")
for _, date in ipairs(dates) do
    local count = client:pfcount(DAU_KEY .. date)
    print(string.format("  %s: ~%d users", date, count))
end

-- Monthly Active Users (union ของทุกวัน)
local all_keys = {}
for _, date in ipairs(dates) do
    table.insert(all_keys, DAU_KEY .. date)
end

local mau_key = "hll:mau:2025-01"
-- PFMERGE: merge หลาย HLL เป็นหนึ่ง
client:pfmerge(mau_key, table.unpack(all_keys))
local mau = client:pfcount(mau_key)
print(string.format("\nMonthly Active Users: ~%d unique users", mau))
print(string.format("(from pool of %d users over %d days)", user_pool, #dates))

-- Memory usage comparison
-- HLL ใช้ ~12KB ต่อ counter, ไม่ว่าจะนับ 100 หรือ 1 billion items
print("\nHyperLogLog: accurate ~0.81%, uses max 12KB per counter")

-- Cleanup
for _, date in ipairs(dates) do client:del(DAU_KEY .. date) end
client:del(mau_key)
client:quit()
```

### ตัวอย่างที่ 28: Bloom Filter Pattern

```lua
-- bloom_filter.lua
-- Bloom Filter: ตรวจสอบว่า item "อาจ" มีหรือ "แน่นอนไม่มี"
-- ใช้ Redis SETBIT/GETBIT

local redis = require("redis")
local client = redis.connect("127.0.0.1", 6379)

local BloomFilter = {}
BloomFilter.__index = BloomFilter

function BloomFilter.new(name, size, hash_count)
    local self = setmetatable({}, BloomFilter)
    self.key        = "bloom:" .. name
    self.size       = size or 1000000  -- 1M bits = 128KB
    self.hash_count = hash_count or 5
    self.client     = client
    return self
end

-- Simple hash functions (ควรใช้ better hash functions ใน production)
function BloomFilter:_hash(item, seed)
    local h = seed * 0x5851f42d4c957f2d
    for i = 1, #item do
        h = h ~ (string.byte(item, i) * 0x14057b7ef767814f)
        h = ((h << 7) | (h >> 57)) * 0x3c6ef372fe94f82b
    end
    return math.abs(math.floor(h % self.size))
end

function BloomFilter:add(item)
    for i = 1, self.hash_count do
        local bit_pos = self:_hash(item, i * 0x9e3779b97f4a7c15)
        self.client:setbit(self.key, bit_pos, 1)
    end
end

function BloomFilter:might_contain(item)
    for i = 1, self.hash_count do
        local bit_pos = self:_hash(item, i * 0x9e3779b97f4a7c15)
        if self.client:getbit(self.key, bit_pos) == 0 then
            return false  -- แน่นอนไม่มี
        end
    end
    return true  -- อาจมี (false positive possible)
end

function BloomFilter:clear()
    self.client:del(self.key)
end

-- ทดสอบ
local bf = BloomFilter.new("emails", 100000, 5)
bf:clear()

-- เพิ่ม emails ที่ลงทะเบียนแล้ว
local registered = {
    "alice@example.com", "bob@example.com",
    "carol@example.com", "dave@example.com"
}
for _, email in ipairs(registered) do
    bf:add(email)
    print("Added:", email)
end

-- ตรวจสอบ
print("\nChecking emails:")
local test_emails = {
    "alice@example.com",    -- registered
    "eve@example.com",      -- not registered
    "bob@example.com",      -- registered
    "frank@example.com",    -- not registered
}
for _, email in ipairs(test_emails) do
    local result = bf:might_contain(email)
    print(string.format("  %-30s %s", email,
        result and "MIGHT EXIST (check DB)" or "DEFINITELY NOT EXISTS"))
end

bf:clear()
client:quit()
```

### ตัวอย่างที่ 29: Geo Location Queue

```lua
-- geo_queue.lua
-- ใช้ Redis GEO + Sorted Sets สำหรับ location-based queues

local redis = require("redis")
local json = require("dkjson")

local client = redis.connect("127.0.0.1", 6379)
local GEO_KEY = "geo:drivers"
local JOB_KEY = "geo:jobs"

client:del(GEO_KEY, JOB_KEY)

-- เพิ่ม driver locations (longitude, latitude, member)
local drivers = {
    {name = "driver:1", lon = 100.523, lat = 13.736},  -- Bangkok area
    {name = "driver:2", lon = 100.541, lat = 13.751},
    {name = "driver:3", lon = 100.498, lat = 13.722},
    {name = "driver:4", lon = 100.562, lat = 13.768},
    {name = "driver:5", lon = 100.510, lat = 13.740},
}

for _, d in ipairs(drivers) do
    client:geoadd(GEO_KEY, d.lon, d.lat, d.name)
    print(string.format("Added %s at (%.3f, %.3f)", d.name, d.lon, d.lat))
end

-- ผู้โดยสารต้องการ driver ที่ใกล้ที่สุด
local passenger_lon, passenger_lat = 100.530, 13.745

print(string.format("\nPassenger at (%.3f, %.3f)", passenger_lon, passenger_lat))
print("Nearby drivers (within 3km):")

-- GEORADIUS: หา members ใน radius
local nearby = client:georadius(
    GEO_KEY,
    passenger_lon, passenger_lat,
    3, "km",
    "WITHCOORD", "WITHDIST",
    "COUNT", 5,
    "ASC"  -- ใกล้สุดก่อน
)

for i, item in ipairs(nearby or {}) do
    local name = item[1]
    local dist = item[2]
    local coord = item[3]
    print(string.format("  %d. %s - %.3f km (%.3f, %.3f)",
        i, name, tonumber(dist),
        tonumber(coord[1]), tonumber(coord[2])))
end

-- คำนวณระยะทางระหว่าง 2 points
local dist = client:geodist(GEO_KEY, "driver:1", "driver:2", "km")
print(string.format("\nDistance driver:1 to driver:2: %.3f km",
    tonumber(dist or 0)))

client:del(GEO_KEY)
client:quit()
```

### ตัวอย่างที่ 30: Distributed Counter

```lua
-- distributed_counter.lua
local redis = require("redis")
local client = redis.connect("127.0.0.1", 6379)

-- Distributed counter ที่ accurate และ fast
local Counter = {}
Counter.__index = Counter

function Counter.new(name)
    local self = setmetatable({}, Counter)
    self.key = "counter:" .. name
    self.client = client
    return self
end

-- Atomic increment
function Counter:incr(amount)
    amount = amount or 1
    if amount == 1 then
        return self.client:incr(self.key)
    else
        return self.client:incrby(self.key, amount)
    end
end

-- Float increment
function Counter:incr_float(amount)
    return tonumber(self.client:incrbyfloat(self.key, amount))
end

-- Decrement
function Counter:decr(amount)
    amount = amount or 1
    return self.client:decrby(self.key, amount)
end

-- Get current value
function Counter:get()
    return tonumber(self.client:get(self.key)) or 0
end

-- Reset
function Counter:reset()
    self.client:set(self.key, 0)
end

-- Set with TTL
function Counter:set_expire(value, ttl)
    self.client:setex(self.key, ttl, tostring(value))
end

-- Time-window counter (count events per minute/hour)
function Counter:incr_window(window_seconds)
    local window_key = self.key .. ":" ..
        math.floor(os.time() / window_seconds)
    local count = self.client:incr(window_key)
    if count == 1 then
        self.client:expire(window_key, window_seconds * 2)
    end
    return count
end

function Counter:get_window(window_seconds)
    local window_key = self.key .. ":" ..
        math.floor(os.time() / window_seconds)
    return tonumber(self.client:get(window_key)) or 0
end

-- ทดสอบ
local page_views = Counter.new("page_views")
local api_calls  = Counter.new("api_calls")

-- Reset
page_views:reset()
api_calls:reset()

-- Simulate requests
for _ = 1, 1000 do
    page_views:incr()
end
for _ = 1, 250 do
    api_calls:incr()
end

print("Page views:", page_views:get())   -- 1000
print("API calls:", api_calls:get())     -- 250

-- Decrement (เมื่อ user logout)
local active_users = Counter.new("active_users")
active_users:reset()
for _ = 1, 50 do active_users:incr() end
active_users:decr(5)  -- 5 users logged out
print("Active users:", active_users:get())  -- 45

-- Time-window counter
local req_counter = Counter.new("requests")
for _ = 1, 100 do
    req_counter:incr_window(60)  -- นับใน 60-second window
end
print("Requests this minute:", req_counter:get_window(60))

-- Cleanup
client:del("counter:page_views", "counter:api_calls",
           "counter:active_users")
client:quit()
```

---

## แบบฝึกหัด

**ข้อ 1**: สร้าง distributed task scheduler ที่รองรับ:
- One-time tasks (run once at specific time)
- Recurring tasks (cron-like syntax)
- Task cancellation
ใช้ Redis Sorted Sets สำหรับ scheduling และ Streams สำหรับ execution log

**ข้อ 2**: Implement circuit breaker pattern บน Redis queue ที่จะ open circuit เมื่อ error rate สูงเกิน threshold (เช่น 50% ใน 1 นาที) และ half-open หลังจาก cooldown period

**ข้อ 3**: สร้าง message deduplication layer ที่ป้องกันการ process message ซ้ำ โดยใช้ Redis SET เพื่อ track message IDs ที่เคย process แล้ว พร้อม TTL สำหรับ cleanup อัตโนมัติ

**ข้อ 4**: เขียน fan-out system ที่เมื่อมี event "post.published" จะ deliver ไปยัง followers ทุกคนอย่าง efficient (hint: ใช้ Redis pipeline และ consumer groups)

**ข้อ 5**: สร้าง real-time dashboard ที่แสดง queue metrics ทุก 1 วินาที: จำนวน jobs pending, inflight, completed, failed และ throughput (jobs/second) โดยใช้ Redis Streams สำหรับ time-series data

---

> **บทถัดไป**: [บทที่ 61: LÖVE2D - 2D Game Development](part-61.md)
