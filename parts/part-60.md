# บทที่ 60: Message Queue

## บทนำ

Message Queue คือกลไกสำคัญในการออกแบบระบบ distributed ที่ช่วยให้ components สื่อสารกันแบบ asynchronous ลด coupling และเพิ่ม resilience

---

## ตัวอย่างที่ 1: ทำไมต้องใช้ Message Queue?

```lua
-- Why Message Queues?
local problems_without_queue = {
    "1. Tight coupling: caller ต้องรอ callee ตลอดเวลา",
    "2. Single point of failure: ถ้า service ล่มงานหายหมด",
    "3. Overwhelm: traffic spike ทำให้ service ล่ม",
    "4. Retry logic: ต้อง implement เอง",
    "5. No priority: งานทุกชิ้นเท่ากันหมด",
}

local benefits_with_queue = {
    "1. Decoupling: producer ไม่ต้องรู้จัก consumer",
    "2. Resilience: consumer ล่มแต่ message ยังอยู่",
    "3. Load leveling: buffer traffic spikes",
    "4. Retry built-in: dead letter queue สำหรับ failures",
    "5. Priority queues: ทำงานสำคัญก่อน",
    "6. Async processing: user ไม่ต้องรองาน heavy",
}

print("=== Why Message Queues? ===")
print("\nProblems WITHOUT queue:")
for _, p in ipairs(problems_without_queue) do print("  " .. p) end
print("\nBenefits WITH queue:")
for _, b in ipairs(benefits_with_queue) do print("  " .. b) end

-- Common use cases
local use_cases = {
    { name = "Email sending",         pattern = "Work Queue" },
    { name = "Image resizing",        pattern = "Work Queue" },
    { name = "Report generation",     pattern = "Delay Queue" },
    { name = "Real-time chat",        pattern = "Pub/Sub" },
    { name = "Stock price updates",   pattern = "Pub/Sub" },
    { name = "Order processing",      pattern = "Priority Queue" },
    { name = "Log aggregation",       pattern = "Fan-out" },
}

print("\nCommon Use Cases:")
for _, uc in ipairs(use_cases) do
    print(string.format("  %-30s -> %s", uc.name, uc.pattern))
end
```

---

## ตัวอย่างที่ 2: Queue Patterns Overview

```lua
-- Queue patterns explanation
local QueuePatterns = {}

-- 1. FIFO (First In, First Out) - Queue ธรรมดา
QueuePatterns.FIFO = {
    description = "งานออกตามลำดับที่เข้ามา",
    pros = { "Simple", "Fair ordering", "Easy to implement" },
    cons = { "Head-of-line blocking", "No priority" },
    use_cases = { "Email queue", "Log processing", "Sequential tasks" },
}

-- 2. LIFO (Last In, First Out) - Stack
QueuePatterns.LIFO = {
    description = "งานล่าสุดออกก่อน",
    pros = { "Recent work first", "Cache-friendly" },
    cons = { "Starvation of old messages" },
    use_cases = { "Undo operations", "Browser history" },
}

-- 3. Priority Queue
QueuePatterns.PRIORITY = {
    description = "งานที่มี priority สูงออกก่อน",
    pros = { "Critical tasks first", "Flexible ordering" },
    cons = { "Complexity", "Priority inversion possible" },
    use_cases = { "Order processing", "Incident response", "VIP users" },
}

-- 4. Delay Queue
QueuePatterns.DELAY = {
    description = "งานรอเวลาที่กำหนดก่อนถูก process",
    pros = { "Scheduled execution", "Retry with backoff" },
    cons = { "Time-based, not event-based" },
    use_cases = { "Scheduled emails", "Retry failed jobs", "Reminders" },
}

-- 5. Dead Letter Queue
QueuePatterns.DLQ = {
    description = "เก็บ messages ที่ process ไม่ผ่าน",
    pros = { "No message loss", "Debugging failed messages" },
    cons = { "Requires separate monitoring" },
    use_cases = { "Failed payments", "Invalid data", "System errors" },
}

print("=== Queue Patterns ===")
for pattern_name, pattern in pairs(QueuePatterns) do
    print(string.format("\n[%s] %s", pattern_name, pattern.description))
    print("  Use cases: " .. table.concat(pattern.use_cases, ", "))
end
```

---

## ตัวอย่างที่ 3: In-Memory FIFO Queue

```lua
-- Simple In-Memory FIFO Queue
local Queue = {}
Queue.__index = Queue

function Queue.new(options)
    local self = setmetatable({}, Queue)
    self.name      = options and options.name or "default"
    self.items     = {}
    self.head      = 1
    self.tail      = 0
    self.max_size  = options and options.max_size or math.huge
    self.stats     = { enqueued = 0, dequeued = 0, failed = 0 }
    return self
end

-- Enqueue
function Queue:push(item, metadata)
    if self:size() >= self.max_size then
        return false, "Queue is full (max=" .. self.max_size .. ")"
    end
    
    self.tail = self.tail + 1
    self.items[self.tail] = {
        id          = self.stats.enqueued + 1,
        data        = item,
        enqueued_at = os.time(),
        attempts    = 0,
        metadata    = metadata or {},
    }
    self.stats.enqueued = self.stats.enqueued + 1
    return true, self.tail
end

-- Dequeue
function Queue:pop()
    if self:is_empty() then return nil end
    
    local item = self.items[self.head]
    self.items[self.head] = nil
    self.head = self.head + 1
    self.stats.dequeued = self.stats.dequeued + 1
    return item
end

-- Peek without removing
function Queue:peek()
    return self.items[self.head]
end

-- Peek last item
function Queue:peek_last()
    return self.items[self.tail]
end

function Queue:size()
    return self.tail - self.head + 1
end

function Queue:is_empty()
    return self.head > self.tail
end

function Queue:clear()
    self.items = {}
    self.head  = 1
    self.tail  = 0
end

-- Iterate all items without dequeuing
function Queue:each(fn)
    for i = self.head, self.tail do
        if self.items[i] then
            fn(self.items[i], i)
        end
    end
end

-- Stats
function Queue:get_stats()
    return {
        name      = self.name,
        size      = self:size(),
        enqueued  = self.stats.enqueued,
        dequeued  = self.stats.dequeued,
        failed    = self.stats.failed,
    }
end

-- ทดสอบ
print("=== In-Memory FIFO Queue ===")
local q = Queue.new({ name = "email_queue", max_size = 100 })

-- Enqueue tasks
local tasks = {
    { type = "welcome_email",  to = "alice@example.com", template = "welcome" },
    { type = "order_confirm",  to = "bob@example.com",   order_id = 1001 },
    { type = "password_reset", to = "charlie@example.com", token = "abc123" },
    { type = "weekly_digest",  to = "diana@example.com" },
}

for _, task in ipairs(tasks) do
    local ok, id = q:push(task)
    print(string.format("Enqueued [%d]: %s -> %s", id, task.type, task.to))
end

print("\nQueue size:", q:size())
print("Peek:", q:peek().data.type)

-- Process
print("\nProcessing:")
while not q:is_empty() do
    local item = q:pop()
    print(string.format("  Processing [%d] %s (attempt %d)",
        item.id, item.data.type, item.attempts))
end

print("\nStats:", q:get_stats().enqueued, "enqueued,", q:get_stats().dequeued, "dequeued")
```

---

## ตัวอย่างที่ 4: Priority Queue

```lua
-- Priority Queue implementation
local PriorityQueue = {}
PriorityQueue.__index = PriorityQueue

function PriorityQueue.new(options)
    local self = setmetatable({}, PriorityQueue)
    self.heap     = {}
    self.options  = options or {}
    self.min_heap = self.options.min_heap ~= false  -- true = min priority value = highest priority
    self.stats    = { enqueued = 0, dequeued = 0 }
    return self
end

-- Heap operations
local function parent(i) return math.floor(i / 2) end
local function left(i)   return 2 * i end
local function right(i)  return 2 * i + 1 end

function PriorityQueue:_swap(i, j)
    self.heap[i], self.heap[j] = self.heap[j], self.heap[i]
end

function PriorityQueue:_compare(i, j)
    local pi = self.heap[i].priority
    local pj = self.heap[j].priority
    return self.min_heap and pi < pj or pi > pj
end

function PriorityQueue:_heapify_up(i)
    while i > 1 and self:_compare(i, parent(i)) do
        self:_swap(i, parent(i))
        i = parent(i)
    end
end

function PriorityQueue:_heapify_down(i)
    local n = #self.heap
    while true do
        local target = i
        local l, r   = left(i), right(i)
        
        if l <= n and self:_compare(l, target) then target = l end
        if r <= n and self:_compare(r, target) then target = r end
        
        if target == i then break end
        self:_swap(i, target)
        i = target
    end
end

-- Enqueue with priority
function PriorityQueue:push(data, priority)
    local item = {
        data        = data,
        priority    = priority,
        id          = self.stats.enqueued + 1,
        enqueued_at = os.time(),
    }
    table.insert(self.heap, item)
    self:_heapify_up(#self.heap)
    self.stats.enqueued = self.stats.enqueued + 1
    return item.id
end

-- Dequeue highest priority item
function PriorityQueue:pop()
    if #self.heap == 0 then return nil end
    
    local top = self.heap[1]
    local last = table.remove(self.heap)
    
    if #self.heap > 0 then
        self.heap[1] = last
        self:_heapify_down(1)
    end
    
    self.stats.dequeued = self.stats.dequeued + 1
    return top
end

function PriorityQueue:peek()
    return self.heap[1]
end

function PriorityQueue:size()
    return #self.heap
end

function PriorityQueue:is_empty()
    return #self.heap == 0
end

-- Priority constants
local Priority = {
    CRITICAL = 1,
    HIGH     = 2,
    NORMAL   = 3,
    LOW      = 4,
    BULK     = 5,
}

-- ทดสอบ
print("=== Priority Queue ===")
local pq = PriorityQueue.new()

-- Add tasks with different priorities
local job_list = {
    { name = "Send marketing email",   priority = Priority.BULK     },
    { name = "Database backup",        priority = Priority.HIGH     },
    { name = "Security alert response", priority = Priority.CRITICAL },
    { name = "Generate monthly report", priority = Priority.NORMAL   },
    { name = "Update search index",    priority = Priority.LOW      },
    { name = "Process payment",        priority = Priority.CRITICAL },
    { name = "Resize uploaded image",  priority = Priority.NORMAL   },
    { name = "Send order confirmation", priority = Priority.HIGH    },
}

for _, job in ipairs(job_list) do
    pq:push(job, job.priority)
end

print(string.format("Queue has %d jobs", pq:size()))
print("\nProcessing by priority:")

local labels = { [1]="CRITICAL", [2]="HIGH", [3]="NORMAL", [4]="LOW", [5]="BULK" }
while not pq:is_empty() do
    local item = pq:pop()
    print(string.format("  [%s] %s", labels[item.priority] or "?", item.data.name))
end
```

---

## ตัวอย่างที่ 5: Delay Queue

```lua
-- Delay Queue - execute messages at specific time
local DelayQueue = {}
DelayQueue.__index = DelayQueue

function DelayQueue.new()
    local self = setmetatable({}, DelayQueue)
    self.items  = {}
    self.id_seq = 0
    return self
end

-- Schedule message for future execution
function DelayQueue:schedule(data, delay_seconds)
    self.id_seq = self.id_seq + 1
    local item = {
        id           = self.id_seq,
        data         = data,
        scheduled_at = os.time(),
        execute_at   = os.time() + delay_seconds,
        delay        = delay_seconds,
    }
    table.insert(self.items, item)
    -- Keep sorted by execute_at
    table.sort(self.items, function(a, b)
        return a.execute_at < b.execute_at
    end)
    return item.id
end

-- Schedule at specific timestamp
function DelayQueue:schedule_at(data, timestamp)
    return self:schedule(data, timestamp - os.time())
end

-- Get messages that are ready to execute
function DelayQueue:poll(max_items)
    max_items = max_items or 10
    local now  = os.time()
    local ready = {}
    
    for i = #self.items, 1, -1 do
        if self.items[i].execute_at <= now then
            table.insert(ready, 1, table.remove(self.items, i))
            if #ready >= max_items then break end
        end
    end
    
    return ready
end

-- Cancel scheduled message
function DelayQueue:cancel(id)
    for i, item in ipairs(self.items) do
        if item.id == id then
            table.remove(self.items, i)
            return true
        end
    end
    return false
end

-- Get next execution time
function DelayQueue:next_at()
    if #self.items == 0 then return nil end
    return self.items[1].execute_at
end

function DelayQueue:size()
    return #self.items
end

-- Exponential backoff delay
function DelayQueue.calc_backoff(attempt, base_delay, max_delay)
    base_delay = base_delay or 5
    max_delay  = max_delay or 3600
    local delay = base_delay * (2 ^ (attempt - 1))
    -- Add jitter
    delay = delay + math.random(0, math.floor(delay * 0.1))
    return math.min(delay, max_delay)
end

-- ทดสอบ
print("=== Delay Queue ===")
local dq = DelayQueue.new()

-- Schedule various tasks
local id1 = dq:schedule({ type = "send_email", to = "alice@example.com" }, -2)  -- past (ready)
local id2 = dq:schedule({ type = "reminder",    id = 1 }, -1)                    -- past (ready)
local id3 = dq:schedule({ type = "backup",      db = "prod" }, 3600)             -- 1 hour
local id4 = dq:schedule({ type = "report",      period = "monthly" }, 86400)     -- 1 day

print(string.format("Scheduled %d tasks", dq:size()))
print("Next execution in:", (dq:next_at() or os.time()) - os.time(), "seconds")

-- Poll ready items
local ready = dq:poll()
print(string.format("\nReady to process: %d items", #ready))
for _, item in ipairs(ready) do
    print(string.format("  [%d] %s (was delayed %ds)",
        item.id, item.data.type, item.delay))
end

print("Remaining in queue:", dq:size())

-- Backoff calculation
print("\nExponential backoff for retries:")
for i = 1, 6 do
    local delay = DelayQueue.calc_backoff(i, 5, 300)
    print(string.format("  Attempt %d: retry after %d seconds", i, delay))
end
```

---

## ตัวอย่างที่ 6: Dead Letter Queue

```lua
-- Dead Letter Queue (DLQ) implementation
local DLQManager = {}
DLQManager.__index = DLQManager

function DLQManager.new(config)
    local self = setmetatable({}, DLQManager)
    self.config = {
        max_retries   = config.max_retries   or 3,
        retry_delay   = config.retry_delay   or 60,
        dlq_retention = config.dlq_retention or 7 * 86400, -- 7 days
    }
    self.main_queue = {}
    self.retry_queue = {}
    self.dlq = {}
    self.stats = { processed = 0, failed = 0, dlq_count = 0, retried = 0 }
    return self
end

-- Enqueue message
function DLQManager:enqueue(message)
    local entry = {
        id          = os.time() .. "_" .. math.random(1000, 9999),
        data        = message,
        attempts    = 0,
        enqueued_at = os.time(),
        errors      = {},
    }
    table.insert(self.main_queue, entry)
    return entry.id
end

-- Process next message
function DLQManager:process_next(processor_fn)
    if #self.main_queue == 0 then
        -- Check retry queue
        local now = os.time()
        for i = #self.retry_queue, 1, -1 do
            local item = self.retry_queue[i]
            if now >= item.retry_at then
                table.remove(self.retry_queue, i)
                table.insert(self.main_queue, 1, item)
            end
        end
        if #self.main_queue == 0 then return nil, "Queue empty" end
    end
    
    local message = table.remove(self.main_queue, 1)
    message.attempts = message.attempts + 1
    message.last_attempted = os.time()
    
    -- Try processing
    local ok, err = pcall(processor_fn, message.data)
    
    if ok then
        self.stats.processed = self.stats.processed + 1
        return true, message
    else
        -- Failed
        table.insert(message.errors, {
            attempt = message.attempts,
            error   = tostring(err),
            time    = os.time(),
        })
        
        if message.attempts < self.config.max_retries then
            -- Schedule retry with exponential backoff
            local delay = self.config.retry_delay * (2 ^ (message.attempts - 1))
            message.retry_at = os.time() + delay
            table.insert(self.retry_queue, message)
            self.stats.retried = self.stats.retried + 1
            print(string.format("[DLQ] Message %s retry #%d in %ds",
                message.id, message.attempts, delay))
        else
            -- Move to DLQ
            message.dlq_reason = "Max retries (" .. self.config.max_retries .. ") exceeded"
            message.dlq_at     = os.time()
            table.insert(self.dlq, message)
            self.stats.failed    = self.stats.failed + 1
            self.stats.dlq_count = self.stats.dlq_count + 1
            print(string.format("[DLQ] Message %s moved to DLQ: %s",
                message.id, message.dlq_reason))
        end
        
        return false, message
    end
end

-- Process DLQ messages (manual retry)
function DLQManager:reprocess_dlq(processor_fn, max_messages)
    max_messages = max_messages or 10
    local reprocessed = 0
    
    for i = 1, math.min(max_messages, #self.dlq) do
        local message = table.remove(self.dlq, 1)
        message.attempts = 0
        message.errors   = {}
        message.dlq_reason = nil
        
        local ok, _ = pcall(processor_fn, message.data)
        if ok then
            reprocessed = reprocessed + 1
            self.stats.dlq_count = self.stats.dlq_count - 1
        else
            -- Re-add to DLQ
            message.dlq_at = os.time()
            table.insert(self.dlq, message)
        end
    end
    
    return reprocessed
end

function DLQManager:stats_summary()
    return {
        main_queue   = #self.main_queue,
        retry_queue  = #self.retry_queue,
        dlq_count    = #self.dlq,
        processed    = self.stats.processed,
        failed       = self.stats.failed,
        retried      = self.stats.retried,
    }
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Dead Letter Queue ===")

local dlq_manager = DLQManager.new({
    max_retries  = 3,
    retry_delay  = 1,
})

-- Enqueue messages
local msg_ids = {}
for i = 1, 5 do
    local id = dlq_manager:enqueue({
        type    = "process_payment",
        order_id = i,
        amount  = math.random(100, 5000),
    })
    table.insert(msg_ids, id)
end

print(string.format("Enqueued %d messages", #msg_ids))

-- Processor that fails 40% of the time
local process_count = 0
local function flaky_processor(data)
    process_count = process_count + 1
    if process_count % 3 == 0 then  -- fail every 3rd
        error("Payment gateway timeout")
    end
    print(string.format("  ✓ Processed order #%d (%.0f THB)",
        data.order_id, data.amount))
end

-- Process all messages
print("\nProcessing messages:")
for i = 1, #msg_ids do
    dlq_manager:process_next(flaky_processor)
end

-- Process retries
print("\nProcessing retries:")
for i = 1, #msg_ids do
    dlq_manager:process_next(flaky_processor)
end

local stats = dlq_manager:stats_summary()
print("\nFinal Stats:")
print("  Processed:", stats.processed)
print("  In retry queue:", stats.retry_queue)
print("  In DLQ:", stats.dlq_count)
```

---

## ตัวอย่างที่ 7: Redis-Based Queue

```lua
-- Redis-based Queue implementation
local RedisQueue = {}
RedisQueue.__index = RedisQueue

-- Mock Redis for simulation
local redis_storage = {}

local function redis_rpush(key, value)
    if not redis_storage[key] then redis_storage[key] = {} end
    table.insert(redis_storage[key], value)
    return #redis_storage[key]
end

local function redis_lpop(key)
    if not redis_storage[key] or #redis_storage[key] == 0 then return nil end
    return table.remove(redis_storage[key], 1)
end

local function redis_brpoplpush(source, dest, timeout)
    -- Block right pop + left push (atomic)
    if not redis_storage[source] or #redis_storage[source] == 0 then
        return nil  -- timeout
    end
    local val = table.remove(redis_storage[source])  -- right pop
    if not redis_storage[dest] then redis_storage[dest] = {} end
    table.insert(redis_storage[dest], 1, val)  -- left push to dest
    return val
end

local function redis_llen(key)
    return #(redis_storage[key] or {})
end

local function redis_lrange(key, start, stop)
    local list = redis_storage[key] or {}
    local result = {}
    local len = #list
    if start < 0 then start = len + start + 1 end
    if stop < 0 then stop = len + stop + 1 end
    for i = start + 1, math.min(stop + 1, len) do
        table.insert(result, list[i])
    end
    return result
end

local function redis_lrem(key, count, value)
    local list = redis_storage[key]
    if not list then return 0 end
    local removed = 0
    for i = #list, 1, -1 do
        if list[i] == value then
            table.remove(list, i)
            removed = removed + 1
            if count > 0 and removed >= count then break end
        end
    end
    return removed
end

-- JSON serialize/deserialize (simple)
local function serialize(data)
    if type(data) == "string" then return data end
    local parts = {}
    for k, v in pairs(data) do
        local val_str
        if type(v) == "string" then
            val_str = '"' .. v:gsub('"', '\\"') .. '"'
        elseif type(v) == "number" or type(v) == "boolean" then
            val_str = tostring(v)
        else
            val_str = '"' .. tostring(v) .. '"'
        end
        table.insert(parts, '"' .. k .. '":' .. val_str)
    end
    return "{" .. table.concat(parts, ",") .. "}"
end

local function deserialize(str)
    -- Simple JSON parser for flat objects
    local obj = {}
    for k, v in str:gmatch('"(%w+)"%s*:%s*"([^"]*)"') do
        obj[k] = v
    end
    for k, v in str:gmatch('"(%w+)"%s*:%s*(%d+)') do
        obj[k] = tonumber(v)
    end
    for k, v in str:gmatch('"(%w+)"%s*:%s*(true)') do
        obj[k] = true
    end
    for k, v in str:gmatch('"(%w+)"%s*:%s*(false)') do
        obj[k] = false
    end
    return obj
end

function RedisQueue.new(config)
    local self = setmetatable({}, RedisQueue)
    self.name         = config.name or "default"
    self.processing   = config.name .. ":processing"
    self.failed       = config.name .. ":failed"
    self.max_retries  = config.max_retries or 3
    return self
end

-- Enqueue (RPUSH to end)
function RedisQueue:push(job_data)
    local job = {
        id          = tostring(os.time()) .. "_" .. tostring(math.random(10000, 99999)),
        data        = job_data,
        attempts    = 0,
        created_at  = os.time(),
    }
    local serialized = serialize(job)
    redis_rpush(self.name, serialized)
    return job.id
end

-- Dequeue (LPOP from front, atomically move to processing)
function RedisQueue:pop()
    local raw = redis_brpoplpush(self.name, self.processing, 0)
    if not raw then return nil end
    
    -- Parse
    local id = raw:match('"id":"([^"]+)"')
    local data_str = raw:match('"data":{([^}]+)}')
    
    return {
        id       = id,
        raw      = raw,
        data     = data_str and deserialize("{" .. data_str .. "}") or {},
        attempts = tonumber(raw:match('"attempts":(%d+)') or "0"),
    }
end

-- Acknowledge: remove from processing
function RedisQueue:ack(job)
    redis_lrem(self.processing, 1, job.raw)
end

-- Nack: move back to main queue or DLQ
function RedisQueue:nack(job, error_msg)
    redis_lrem(self.processing, 1, job.raw)
    
    local new_attempts = (job.attempts or 0) + 1
    
    if new_attempts <= self.max_retries then
        -- Re-queue for retry
        local retry_job = job.raw:gsub('"attempts":(%d+)', '"attempts":' .. new_attempts)
        redis_rpush(self.name, retry_job)
        print(string.format("[Queue] Retry %d/%d for job %s",
            new_attempts, self.max_retries, job.id or "?"))
    else
        -- Move to DLQ
        local dlq_entry = job.raw .. ',"error":"' .. (error_msg or "unknown") .. '"'
        redis_rpush(self.failed, dlq_entry)
        print(string.format("[Queue] Job %s moved to DLQ after %d attempts",
            job.id or "?", new_attempts))
    end
end

function RedisQueue:length()
    return redis_llen(self.name)
end

function RedisQueue:processing_count()
    return redis_llen(self.processing)
end

function RedisQueue:dlq_count()
    return redis_llen(self.failed)
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Redis-Based Queue ===")

local queue = RedisQueue.new({
    name         = "jobs:email",
    max_retries  = 2,
})

-- Push jobs
local job_types = {
    { type = "welcome",      recipient = "alice@example.com" },
    { type = "order_confirm", recipient = "bob@example.com",  order_id = 42 },
    { type = "password_reset", recipient = "charlie@example.com", token = "xyz" },
}

for _, job in ipairs(job_types) do
    local id = queue:push(job)
    print(string.format("Pushed job %s", id:sub(1, 15) .. "..."))
end

print(string.format("Queue length: %d", queue:length()))

-- Process with some failures
local fail_count = 0
print("\nProcessing:")
for i = 1, #job_types do
    local job = queue:pop()
    if job then
        print(string.format("  Got job [%s] attempts=%d",
            job.id and job.id:sub(1, 12) or "?", job.attempts))
        
        fail_count = fail_count + 1
        if fail_count == 2 then
            -- Simulate failure
            queue:nack(job, "SMTP connection refused")
        else
            queue:ack(job)
            print("  ✓ Acknowledged")
        end
    end
end

print(string.format("\nQueue: %d, Processing: %d, DLQ: %d",
    queue:length(), queue:processing_count(), queue:dlq_count()))
```

---

## ตัวอย่างที่ 8: Publish/Subscribe

```lua
-- Pub/Sub Message Broker
local PubSubBroker = {}
PubSubBroker.__index = PubSubBroker

function PubSubBroker.new(config)
    local self = setmetatable({}, PubSubBroker)
    self.subscribers   = {}   -- topic -> [{ id, callback, filter }]
    self.message_log   = {}   -- history for replay
    self.max_history   = config and config.max_history or 100
    self.stats         = { published = 0, delivered = 0 }
    return self
end

-- Subscribe to topic
function PubSubBroker:subscribe(topic, callback, options)
    if not self.subscribers[topic] then
        self.subscribers[topic] = {}
    end
    
    local sub = {
        id       = tostring(#self.subscribers[topic] + 1) .. "_" .. topic,
        callback = callback,
        filter   = options and options.filter,
        group    = options and options.group,
        created  = os.time(),
    }
    
    table.insert(self.subscribers[topic], sub)
    return sub.id
end

-- Unsubscribe
function PubSubBroker:unsubscribe(sub_id)
    for topic, subs in pairs(self.subscribers) do
        for i, sub in ipairs(subs) do
            if sub.id == sub_id then
                table.remove(subs, i)
                print("[PubSub] Unsubscribed: " .. sub_id)
                return true
            end
        end
    end
    return false
end

-- Publish to topic
function PubSubBroker:publish(topic, message, metadata)
    local msg = {
        id        = os.time() .. "_" .. math.random(1000, 9999),
        topic     = topic,
        data      = message,
        metadata  = metadata or {},
        timestamp = os.time(),
    }
    
    self.stats.published = self.stats.published + 1
    
    -- Store in history
    table.insert(self.message_log, msg)
    if #self.message_log > self.max_history then
        table.remove(self.message_log, 1)
    end
    
    -- Deliver to subscribers
    local subs = self.subscribers[topic] or {}
    local delivered = 0
    
    -- Consumer group: only ONE consumer in group gets the message
    local group_seen = {}
    
    for _, sub in ipairs(subs) do
        -- Check filter
        if sub.filter and not sub.filter(msg) then
            goto continue
        end
        
        -- Consumer group deduplication
        if sub.group then
            if group_seen[sub.group] then goto continue end
            group_seen[sub.group] = true
        end
        
        local ok, err = pcall(sub.callback, msg)
        if ok then
            delivered = delivered + 1
            self.stats.delivered = self.stats.delivered + 1
        else
            print(string.format("[PubSub] Delivery failed to %s: %s", sub.id, err))
        end
        
        ::continue::
    end
    
    return msg.id, delivered
end

-- Replay messages from specific time
function PubSubBroker:replay(topic, from_timestamp, callback)
    local replayed = 0
    for _, msg in ipairs(self.message_log) do
        if msg.topic == topic and msg.timestamp >= from_timestamp then
            callback(msg)
            replayed = replayed + 1
        end
    end
    return replayed
end

-- Wildcard subscription (pattern matching)
function PubSubBroker:subscribe_pattern(pattern, callback)
    -- Store pattern subscriber
    if not self.subscribers["__patterns__"] then
        self.subscribers["__patterns__"] = {}
    end
    local sub = {
        id       = "pattern_" .. pattern,
        pattern  = pattern,
        callback = callback,
    }
    table.insert(self.subscribers["__patterns__"], sub)
    return sub.id
end

-- Override publish to check patterns
local orig_publish = PubSubBroker.publish
PubSubBroker.publish = function(self, topic, message, metadata)
    local msg_id, delivered = orig_publish(self, topic, message, metadata)
    
    -- Check pattern subscribers
    local patterns = self.subscribers["__patterns__"] or {}
    for _, sub in ipairs(patterns) do
        local lua_pattern = sub.pattern:gsub("%*", ".*"):gsub("%?", ".")
        if topic:match("^" .. lua_pattern .. "$") then
            pcall(sub.callback, {
                id = msg_id, topic = topic, data = message,
                metadata = metadata, timestamp = os.time()
            })
            delivered = delivered + 1
        end
    end
    
    return msg_id, delivered
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Pub/Sub Message Broker ===")

local broker = PubSubBroker.new({ max_history = 50 })

-- Subscribe
broker:subscribe("orders.created", function(msg)
    print(string.format("  [Order Handler] New order: %s",
        type(msg.data) == "table" and msg.data.id or tostring(msg.data)))
end)

broker:subscribe("orders.created", function(msg)
    print(string.format("  [Inventory Handler] Reserve stock for order: %s",
        type(msg.data) == "table" and (msg.data.product or "?") or "?"))
end)

broker:subscribe("orders.created", function(msg)
    print(string.format("  [Email Handler] Send confirmation for order %s",
        type(msg.data) == "table" and msg.data.id or "?"))
end, { group = "notification-group" })

broker:subscribe("orders.updated", function(msg)
    print(string.format("  [Audit Handler] Order updated: %s -> %s",
        type(msg.data) == "table" and (msg.data.id or "?") or "?",
        type(msg.data) == "table" and (msg.data.status or "?") or "?"))
end)

-- Pattern subscriber
broker:subscribe_pattern("orders.*", function(msg)
    print(string.format("  [Analytics] Event on topic '%s'", msg.topic))
end)

-- Publish
print("\nPublishing events:")
local id1, cnt1 = broker:publish("orders.created", {
    id      = 1001,
    product = "Laptop",
    amount  = 35000,
    user_id = 1,
})
print(string.format("Published to orders.created: %d receivers", cnt1))

local id2, cnt2 = broker:publish("orders.updated", {
    id     = 1001,
    status = "shipped",
})
print(string.format("Published to orders.updated: %d receivers", cnt2))

print("\nBroker stats:", broker.stats.published, "published,",
      broker.stats.delivered, "total deliveries")
```

---

## ตัวอย่างที่ 9: Topic Exchange (RabbitMQ-style)

```lua
-- Topic Exchange - Routes messages based on routing keys
local TopicExchange = {}
TopicExchange.__index = TopicExchange

function TopicExchange.new(name)
    local self = setmetatable({}, TopicExchange)
    self.name    = name
    self.queues  = {}   -- queue_name -> { messages }
    self.bindings = {}  -- { queue, pattern }
    return self
end

-- Bind queue to exchange with routing key pattern
function TopicExchange:bind(queue_name, pattern)
    if not self.queues[queue_name] then
        self.queues[queue_name] = {}
    end
    table.insert(self.bindings, { queue = queue_name, pattern = pattern })
    print(string.format("[Exchange:%s] Bound queue '%s' with pattern '%s'",
        self.name, queue_name, pattern))
end

-- Match routing key against pattern
-- * matches one word, # matches zero or more words
local function routing_key_matches(pattern, routing_key)
    -- Convert AMQP pattern to Lua pattern
    local lua_pat = "^"
    
    for part in (pattern .. "."):gmatch("([^%.]*%.?)") do
        local word = part:gsub("%.$", "")
        local has_dot = part:match("%.$") and "%" .. "." or ""
        
        if word == "#" then
            lua_pat = lua_pat .. "([%w_]+%.)*([%w_]+)?"
        elseif word == "*" then
            lua_pat = lua_pat .. "[%w_]+" .. has_dot
        else
            lua_pat = lua_pat .. word:gsub("%.", "%%.") .. has_dot
        end
    end
    
    lua_pat = lua_pat .. "$"
    
    return routing_key:match(lua_pat) ~= nil
end

-- Publish to exchange
function TopicExchange:publish(routing_key, message)
    local routed_to = {}
    
    for _, binding in ipairs(self.bindings) do
        if routing_key_matches(binding.pattern, routing_key) then
            if not self.queues[binding.queue] then
                self.queues[binding.queue] = {}
            end
            table.insert(self.queues[binding.queue], {
                routing_key = routing_key,
                data        = message,
                timestamp   = os.time(),
            })
            table.insert(routed_to, binding.queue)
        end
    end
    
    print(string.format("[Exchange:%s] Routing '%s' -> [%s]",
        self.name, routing_key, table.concat(routed_to, ", ")))
    
    return #routed_to
end

-- Consume from queue
function TopicExchange:consume(queue_name)
    if not self.queues[queue_name] or #self.queues[queue_name] == 0 then
        return nil
    end
    return table.remove(self.queues[queue_name], 1)
end

-- Queue stats
function TopicExchange:queue_stats()
    local stats = {}
    for name, queue in pairs(self.queues) do
        stats[name] = #queue
    end
    return stats
end

-- ทดสอบ
print("=== Topic Exchange (AMQP-style) ===")

local exchange = TopicExchange.new("events")

-- Bind queues with patterns
-- Syntax: * = one word, # = zero or more words
exchange:bind("all_orders",        "order.#")         -- all order events
exchange:bind("order_payments",    "order.*.payment") -- payment events
exchange:bind("critical_events",   "*.critical.*")    -- all critical events
exchange:bind("user_notifications", "user.#")         -- user events
exchange:bind("analytics",          "#")              -- everything

-- Publish messages
print("\nPublishing events:")
local routing_keys = {
    "order.created",
    "order.1001.payment",
    "order.cancelled",
    "user.login",
    "user.critical.breach",
    "system.health",
    "system.critical.downtime",
}

for _, key in ipairs(routing_keys) do
    local count = exchange:publish(key, { event = key, timestamp = os.time() })
    print(string.format("  Routed to %d queues", count))
end

-- Queue sizes
print("\nQueue sizes:")
local stats = exchange:queue_stats()
for queue, count in pairs(stats) do
    print(string.format("  %-25s: %d messages", queue, count))
end

-- Consume from analytics
print("\nConsuming from 'all_orders':")
local msg
repeat
    msg = exchange:consume("all_orders")
    if msg then
        print(string.format("  [%s] %s", msg.routing_key, tostring(msg.data.event)))
    end
until not msg
```

---

## ตัวอย่างที่ 10: Message Retry with Backoff

```lua
-- Message Retry with Exponential Backoff
local RetryManager = {}
RetryManager.__index = RetryManager

function RetryManager.new(config)
    local self = setmetatable({}, RetryManager)
    self.config = {
        max_attempts = config.max_attempts or 5,
        base_delay   = config.base_delay   or 5,    -- seconds
        max_delay    = config.max_delay    or 3600, -- 1 hour
        jitter       = config.jitter       ~= false, -- add randomness
        backoff_type = config.backoff_type or "exponential",
    }
    self.pending  = {}
    self.history  = {}
    return self
end

-- Calculate delay for next retry
function RetryManager:calc_delay(attempt)
    local base  = self.config.base_delay
    local max_d = self.config.max_delay
    local delay
    
    if self.config.backoff_type == "exponential" then
        delay = base * (2 ^ (attempt - 1))
    elseif self.config.backoff_type == "linear" then
        delay = base * attempt
    elseif self.config.backoff_type == "constant" then
        delay = base
    else
        delay = base
    end
    
    -- Cap at max delay
    delay = math.min(delay, max_d)
    
    -- Add jitter: ±10% random
    if self.config.jitter then
        local jitter_range = math.floor(delay * 0.1)
        if jitter_range > 0 then
            delay = delay + math.random(-jitter_range, jitter_range)
        end
    end
    
    return math.max(1, delay)
end

-- Submit job with retry logic
function RetryManager:submit(job_id, job_data, process_fn)
    local attempts = 0
    local last_error = nil
    
    while attempts < self.config.max_attempts do
        attempts = attempts + 1
        print(string.format("[Retry] Job %s: attempt %d/%d",
            job_id, attempts, self.config.max_attempts))
        
        local ok, err = pcall(process_fn, job_data)
        
        if ok then
            table.insert(self.history, {
                job_id   = job_id,
                attempts = attempts,
                success  = true,
                time     = os.time(),
            })
            print(string.format("[Retry] Job %s: SUCCESS after %d attempts",
                job_id, attempts))
            return true, attempts
        else
            last_error = tostring(err)
            
            if attempts < self.config.max_attempts then
                local delay = self:calc_delay(attempts)
                print(string.format("[Retry] Job %s: FAILED (%s), retry in %ds",
                    job_id, last_error, delay))
                -- ในระบบจริงจะ sleep หรือใส่ delay queue
                -- os.execute("sleep " .. delay)
            end
        end
    end
    
    -- All attempts exhausted
    table.insert(self.history, {
        job_id    = job_id,
        attempts  = attempts,
        success   = false,
        error     = last_error,
        time      = os.time(),
    })
    print(string.format("[Retry] Job %s: EXHAUSTED (%d attempts)",
        job_id, attempts))
    return false, attempts
end

-- Show delay schedule
function RetryManager:show_schedule(attempts)
    attempts = attempts or self.config.max_attempts
    print(string.format("\nRetry schedule (%s backoff, base=%ds, max=%ds):",
        self.config.backoff_type, self.config.base_delay, self.config.max_delay))
    
    local total = 0
    for i = 1, attempts do
        local delay = self:calc_delay(i)
        total = total + delay
        print(string.format("  Attempt %d: wait %4ds (total: %ds)",
            i, delay, total))
    end
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Message Retry with Backoff ===")

local retry_mgr = RetryManager.new({
    max_attempts = 5,
    base_delay   = 2,
    max_delay    = 60,
    backoff_type = "exponential",
    jitter       = false,
})

retry_mgr:show_schedule()

-- Test job that fails first N times
local attempts_before_success = 3
local call_count = 0

local function unstable_processor(data)
    call_count = call_count + 1
    if call_count < attempts_before_success then
        error("Connection timeout: " .. call_count)
    end
    print("  ✓ Processed: " .. tostring(data.message))
end

print("\nRunning job with flaky processor (fails first 2 attempts):")
local success, total_attempts = retry_mgr:submit(
    "job_001",
    { message = "Process this data" },
    unstable_processor
)

print(string.format("\nResult: success=%s, total_attempts=%d",
    tostring(success), total_attempts))
```

---

## ตัวอย่างที่ 11: Idempotency

```lua
-- Idempotent Message Processing
local IdempotencyManager = {}
IdempotencyManager.__index = IdempotencyManager

function IdempotencyManager.new(config)
    local self = setmetatable({}, IdempotencyManager)
    self.processed_keys = {}  -- idempotency_key -> result
    self.key_ttl = config and config.ttl or 86400  -- 24 hours
    self.stats = { processed = 0, skipped = 0 }
    return self
end

-- Generate idempotency key
function IdempotencyManager.generate_key(data)
    if type(data) == "string" then return data end
    
    -- Create deterministic key from data
    local parts = {}
    local keys = {}
    for k in pairs(data) do table.insert(keys, k) end
    table.sort(keys)
    
    for _, k in ipairs(keys) do
        table.insert(parts, k .. "=" .. tostring(data[k]))
    end
    
    local combined = table.concat(parts, "&")
    
    -- Simple hash
    local hash = 0
    for i = 1, #combined do
        hash = ((hash * 31) + string.byte(combined, i)) % (2^32)
    end
    
    return string.format("idem_%08x", hash)
end

-- Process with idempotency check
function IdempotencyManager:process(idempotency_key, job_data, process_fn)
    -- Check if already processed
    local existing = self.processed_keys[idempotency_key]
    if existing then
        if os.time() < existing.expires_at then
            self.stats.skipped = self.stats.skipped + 1
            print(string.format("[Idempotent] SKIP (already processed): %s", idempotency_key))
            return existing.result, true  -- result, was_duplicate
        else
            self.processed_keys[idempotency_key] = nil
        end
    end
    
    -- Process for first time
    local ok, result = pcall(process_fn, job_data)
    
    if ok then
        self.processed_keys[idempotency_key] = {
            result     = result,
            processed_at = os.time(),
            expires_at = os.time() + self.key_ttl,
        }
        self.stats.processed = self.stats.processed + 1
        return result, false  -- result, was_duplicate
    else
        return nil, false, tostring(result)  -- error
    end
end

-- Cleanup expired keys
function IdempotencyManager:cleanup()
    local now = os.time()
    local removed = 0
    for key, entry in pairs(self.processed_keys) do
        if now > entry.expires_at then
            self.processed_keys[key] = nil
            removed = removed + 1
        end
    end
    return removed
end

-- ทดสอบ
print("=== Idempotency ===")

local idem = IdempotencyManager.new({ ttl = 300 })

-- Simulate duplicate webhook deliveries
local webhook_events = {
    { id = "evt_001", type = "payment.succeeded", amount = 1500 },
    { id = "evt_002", type = "order.created",     order_id = 42 },
    { id = "evt_001", type = "payment.succeeded", amount = 1500 },  -- duplicate!
    { id = "evt_003", type = "user.verified",     user_id = 5 },
    { id = "evt_002", type = "order.created",     order_id = 42 },  -- duplicate!
    { id = "evt_001", type = "payment.succeeded", amount = 1500 },  -- duplicate!
}

print("Processing webhook events:")
for _, event in ipairs(webhook_events) do
    local key = "webhook_" .. event.id
    
    local result, was_dup = idem:process(key, event, function(data)
        -- Process the event
        if data.type == "payment.succeeded" then
            print(string.format("  [NEW] Recording payment of %.0f THB", data.amount))
            return { recorded = true, amount = data.amount }
        elseif data.type == "order.created" then
            print(string.format("  [NEW] Creating order #%d", data.order_id))
            return { created = true, order_id = data.order_id }
        elseif data.type == "user.verified" then
            print(string.format("  [NEW] Verifying user #%d", data.user_id))
            return { verified = true }
        end
    end)
    
    if was_dup then
        print(string.format("  [DUP] Event %s skipped (idempotent)", event.id))
    end
end

print(string.format("\nStats: %d processed, %d duplicates skipped",
    idem.stats.processed, idem.stats.skipped))
```

---

## ตัวอย่างที่ 12: Consumer Groups

```lua
-- Consumer Groups for parallel processing
local ConsumerGroup = {}
ConsumerGroup.__index = ConsumerGroup

function ConsumerGroup.new(queue_name, group_name, config)
    local self = setmetatable({}, ConsumerGroup)
    self.queue_name  = queue_name
    self.group_name  = group_name
    self.config      = config or {}
    self.consumers   = {}
    self.pending     = {}  -- pending ACKs
    self.messages    = {}  -- shared message store
    self.offset      = 0   -- current processing position
    return self
end

-- Simulate shared message stream (Redis Stream-like)
local message_streams = {}

function ConsumerGroup:write(data)
    if not message_streams[self.queue_name] then
        message_streams[self.queue_name] = {}
    end
    local id = tostring(os.time()) .. "-" .. tostring(#message_streams[self.queue_name] + 1)
    table.insert(message_streams[self.queue_name], {
        id   = id,
        data = data,
        time = os.time(),
    })
    return id
end

-- Register consumer in group
function ConsumerGroup:add_consumer(consumer_id, process_fn)
    self.consumers[consumer_id] = {
        id         = consumer_id,
        process_fn = process_fn,
        processed  = 0,
        errors     = 0,
        active     = true,
    }
    print(string.format("[CG:%s] Consumer '%s' joined group '%s'",
        self.queue_name, consumer_id, self.group_name))
end

-- Read next message for consumer
function ConsumerGroup:read_next(consumer_id)
    local consumer = self.consumers[consumer_id]
    if not consumer then return nil, "Consumer not found" end
    
    local stream = message_streams[self.queue_name]
    if not stream then return nil end
    
    self.offset = self.offset + 1
    local msg = stream[self.offset]
    if not msg then
        self.offset = self.offset - 1
        return nil
    end
    
    -- Track pending
    self.pending[msg.id] = {
        consumer_id  = consumer_id,
        message      = msg,
        delivered_at = os.time(),
    }
    
    return msg
end

-- Acknowledge message
function ConsumerGroup:ack(consumer_id, message_id)
    local pending = self.pending[message_id]
    if pending and pending.consumer_id == consumer_id then
        self.pending[message_id] = nil
        local consumer = self.consumers[consumer_id]
        if consumer then consumer.processed = consumer.processed + 1 end
        return true
    end
    return false, "Not owner or not found"
end

-- Process all pending for consumer
function ConsumerGroup:process_consumer(consumer_id)
    local consumer = self.consumers[consumer_id]
    if not consumer then return 0, "Consumer not found" end
    
    local processed = 0
    
    while true do
        local msg = self:read_next(consumer_id)
        if not msg then break end
        
        local ok, err = pcall(consumer.process_fn, msg.data)
        
        if ok then
            self:ack(consumer_id, msg.id)
            processed = processed + 1
        else
            consumer.errors = consumer.errors + 1
            print(string.format("[CG] Consumer %s failed on msg %s: %s",
                consumer_id, msg.id, err))
            -- In real system: retry or move to DLQ
            self.pending[msg.id] = nil
        end
    end
    
    return processed
end

-- Stats
function ConsumerGroup:stats()
    local consumer_stats = {}
    for id, c in pairs(self.consumers) do
        consumer_stats[id] = { processed = c.processed, errors = c.errors }
    end
    return {
        queue       = self.queue_name,
        group       = self.group_name,
        total_msgs  = #(message_streams[self.queue_name] or {}),
        pending     = (function()
            local c = 0; for _ in pairs(self.pending) do c = c + 1 end; return c
        end)(),
        consumers   = consumer_stats,
    }
end

-- ทดสอบ
print("=== Consumer Groups ===")

local cg = ConsumerGroup.new("orders", "processors")

-- Write messages to stream
print("Writing messages:")
for i = 1, 6 do
    local id = cg:write({
        order_id = 1000 + i,
        product  = "Product " .. i,
        amount   = (i * 100),
    })
    print(string.format("  Written message %s", id))
end

-- Add consumers
cg:add_consumer("worker-1", function(data)
    print(string.format("    [W1] Processing order #%d (%.0f THB)",
        data.order_id, data.amount))
end)

cg:add_consumer("worker-2", function(data)
    print(string.format("    [W2] Processing order #%d (%.0f THB)",
        data.order_id, data.amount))
end)

-- Each consumer processes some messages
print("\nWorker-1 processing:")
local w1 = cg:process_consumer("worker-1")
print(string.format("Worker-1 processed: %d messages", w1))

print("\nWorker-2 processing:")
local w2 = cg:process_consumer("worker-2")
print(string.format("Worker-2 processed: %d messages", w2))

-- Final stats
local stats = cg:stats()
print(string.format("\nGroup Stats: %d total, %d pending",
    stats.total_msgs, stats.pending))
for id, cs in pairs(stats.consumers) do
    print(string.format("  %s: %d processed, %d errors", id, cs.processed, cs.errors))
end
```

---

## ตัวอย่างที่ 13: Background Job Processor

```lua
-- Complete Background Job Processor
local JobProcessor = {}
JobProcessor.__index = JobProcessor

function JobProcessor.new(config)
    local self = setmetatable({}, JobProcessor)
    self.config = {
        concurrency = config.concurrency or 3,
        max_retries = config.max_retries or 3,
        retry_delay = config.retry_delay or 5,
        timeout     = config.timeout     or 30,
    }
    self.queues       = {}  -- { name -> Queue }
    self.handlers     = {}  -- { job_type -> handler_fn }
    self.middlewares  = {}  -- before/after hooks
    self.stats        = { total = 0, success = 0, failed = 0, retried = 0 }
    self.running      = false
    self.job_log      = {}
    return self
end

-- Register job handler
function JobProcessor:register(job_type, handler, options)
    self.handlers[job_type] = {
        fn      = handler,
        options = options or {},
        name    = job_type,
    }
    print("[Processor] Registered handler: " .. job_type)
end

-- Add middleware
function JobProcessor:use(middleware)
    table.insert(self.middlewares, middleware)
end

-- Enqueue job
function JobProcessor:enqueue(queue_name, job_type, data, options)
    if not self.queues[queue_name] then
        self.queues[queue_name] = {}
    end
    
    options = options or {}
    local job = {
        id          = string.format("job_%d_%d", os.time(), math.random(1000, 9999)),
        type        = job_type,
        queue       = queue_name,
        data        = data,
        priority    = options.priority or 3,
        attempts    = 0,
        max_retries = options.max_retries or self.config.max_retries,
        created_at  = os.time(),
        scheduled_at = options.delay and (os.time() + options.delay) or os.time(),
        status      = "pending",
    }
    
    table.insert(self.queues[queue_name], job)
    table.sort(self.queues[queue_name], function(a, b)
        if a.priority ~= b.priority then return a.priority < b.priority end
        return a.created_at < b.created_at
    end)
    
    print(string.format("[Processor] Enqueued: [%s] %s (priority=%d)",
        job.id, job_type, job.priority))
    
    return job.id
end

-- Process single job
function JobProcessor:_process_job(job)
    local handler_info = self.handlers[job.type]
    if not handler_info then
        return false, "No handler for job type: " .. job.type
    end
    
    job.status   = "processing"
    job.attempts = job.attempts + 1
    job.started_at = os.time()
    
    -- Run before middlewares
    local ctx = { job = job, start_time = os.time() }
    for _, mw in ipairs(self.middlewares) do
        if mw.before then
            pcall(mw.before, ctx)
        end
    end
    
    -- Execute handler
    local ok, result = pcall(handler_info.fn, job.data, ctx)
    
    job.finished_at = os.time()
    job.duration    = job.finished_at - job.started_at
    
    -- Run after middlewares
    ctx.result = result
    ctx.error  = not ok and result or nil
    for _, mw in ipairs(self.middlewares) do
        if mw.after then
            pcall(mw.after, ctx)
        end
    end
    
    if ok then
        job.status = "completed"
        job.result = result
        self.stats.success = self.stats.success + 1
        table.insert(self.job_log, {
            id = job.id, type = job.type,
            success = true, duration = job.duration,
        })
        return true, result
    else
        local error_msg = tostring(result)
        job.last_error = error_msg
        
        if job.attempts < job.max_retries then
            job.status = "retry"
            local delay = self.config.retry_delay * (2 ^ (job.attempts - 1))
            job.retry_at = os.time() + delay
            self.stats.retried = self.stats.retried + 1
            return false, error_msg, "retry"
        else
            job.status = "failed"
            self.stats.failed = self.stats.failed + 1
            table.insert(self.job_log, {
                id = job.id, type = job.type,
                success = false, error = error_msg,
            })
            return false, error_msg, "failed"
        end
    end
end

-- Process all jobs in queue
function JobProcessor:process_queue(queue_name)
    local queue = self.queues[queue_name]
    if not queue or #queue == 0 then
        return 0
    end
    
    local processed = 0
    local retry_jobs = {}
    
    while #queue > 0 do
        local job = table.remove(queue, 1)
        
        -- Skip if not yet scheduled
        if job.scheduled_at > os.time() then
            table.insert(retry_jobs, job)
            goto continue
        end
        
        self.stats.total = self.stats.total + 1
        print(string.format("\n[Processor] Running job [%s] %s (attempt %d)",
            job.id, job.type, job.attempts + 1))
        
        local ok, result, disposition = self:_process_job(job)
        
        if ok then
            print(string.format("[Processor] ✓ Completed [%s] in %dms",
                job.id, (job.duration or 0) * 1000))
            processed = processed + 1
        elseif disposition == "retry" then
            print(string.format("[Processor] ↩ Retry [%s] (attempt %d/%d)",
                job.id, job.attempts, job.max_retries))
            table.insert(retry_jobs, job)
        else
            print(string.format("[Processor] ✗ Failed [%s]: %s",
                job.id, result))
        end
        
        ::continue::
    end
    
    -- Re-add retry jobs
    for _, job in ipairs(retry_jobs) do
        table.insert(queue, job)
    end
    
    return processed
end

-- Get stats
function JobProcessor:get_stats()
    local queue_sizes = {}
    for name, q in pairs(self.queues) do
        queue_sizes[name] = #q
    end
    return {
        total      = self.stats.total,
        success    = self.stats.success,
        failed     = self.stats.failed,
        retried    = self.stats.retried,
        queues     = queue_sizes,
        success_rate = self.stats.total > 0 and
            string.format("%.1f%%", self.stats.success / self.stats.total * 100) or "N/A",
    }
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Background Job Processor ===")

local processor = JobProcessor.new({
    concurrency = 3,
    max_retries = 3,
    retry_delay = 1,
})

-- Add logging middleware
processor:use({
    before = function(ctx)
        print(string.format("  → Start: %s", ctx.job.type))
    end,
    after = function(ctx)
        if ctx.error then
            print(string.format("  ← Error: %s", ctx.error))
        end
    end,
})

-- Register handlers
processor:register("send_email", function(data, ctx)
    -- Simulate occasional failure
    if math.random() < 0.3 then
        error("SMTP connection refused")
    end
    print(string.format("  Email sent to: %s", data.to))
    return { delivered = true }
end)

processor:register("resize_image", function(data, ctx)
    print(string.format("  Resized: %s to %dx%d",
        data.filename, data.width, data.height))
    return { url = "/resized/" .. data.filename }
end)

processor:register("generate_report", function(data, ctx)
    print(string.format("  Report generated: %s for %s",
        data.type, data.period))
    return { filename = data.type .. "_" .. data.period .. ".pdf" }
end)

-- Enqueue jobs
print("\nEnqueueing jobs:")
processor:enqueue("default", "send_email",      { to = "alice@example.com", subject = "Welcome!" }, { priority = 2 })
processor:enqueue("default", "send_email",      { to = "bob@example.com",   subject = "Invoice"  }, { priority = 2 })
processor:enqueue("default", "resize_image",    { filename = "photo.jpg",  width = 800, height = 600 })
processor:enqueue("default", "generate_report", { type = "sales", period = "2024-03" }, { priority = 4 })
processor:enqueue("default", "send_email",      { to = "charlie@example.com", subject = "Alert!" }, { priority = 1 })

-- Process
print("\nProcessing queue:")
processor:process_queue("default")

-- Second pass for retries
print("\nSecond pass (retries):")
processor:process_queue("default")

-- Final stats
local final_stats = processor:get_stats()
print(string.format("\nFinal Stats:"))
print(string.format("  Total:   %d", final_stats.total))
print(string.format("  Success: %d (%s)", final_stats.success, final_stats.success_rate))
print(string.format("  Failed:  %d", final_stats.failed))
print(string.format("  Retried: %d", final_stats.retried))
```

---

## ตัวอย่างที่ 14: Exactly-Once Delivery

```lua
-- Exactly-Once Delivery guarantee
local ExactlyOnce = {}
ExactlyOnce.__index = ExactlyOnce

function ExactlyOnce.new(config)
    local self = setmetatable({}, ExactlyOnce)
    self.dedup_window  = config and config.dedup_window or 3600
    self.processed_ids = {}  -- message_id -> timestamp
    self.stats = { received = 0, processed = 0, duplicates = 0 }
    return self
end

-- Message fingerprint for dedup
local function message_fingerprint(message)
    local parts = {}
    if type(message) == "table" then
        for k, v in pairs(message) do
            table.insert(parts, tostring(k) .. ":" .. tostring(v))
        end
        table.sort(parts)
    else
        table.insert(parts, tostring(message))
    end
    
    local combined = table.concat(parts, "|")
    local hash = 0
    for i = 1, #combined do
        hash = ((hash * 31) + string.byte(combined, i)) % (2^32)
    end
    return string.format("msg_%08x", hash)
end

-- Process with exactly-once guarantee
function ExactlyOnce:process(message_id, message, handler)
    self.stats.received = self.stats.received + 1
    
    -- Auto-generate ID if not provided
    if not message_id then
        message_id = message_fingerprint(message)
    end
    
    -- Cleanup expired IDs
    local now = os.time()
    local to_remove = {}
    for id, ts in pairs(self.processed_ids) do
        if now - ts > self.dedup_window then
            table.insert(to_remove, id)
        end
    end
    for _, id in ipairs(to_remove) do
        self.processed_ids[id] = nil
    end
    
    -- Check duplicate
    if self.processed_ids[message_id] then
        self.stats.duplicates = self.stats.duplicates + 1
        print(string.format("[EO] DUPLICATE message %s (originally processed %ds ago)",
            message_id, now - self.processed_ids[message_id]))
        return true, nil, true  -- success, result, was_duplicate
    end
    
    -- Process the message
    local ok, result = pcall(handler, message)
    
    if ok then
        self.processed_ids[message_id] = now
        self.stats.processed = self.stats.processed + 1
        return true, result, false
    else
        return false, nil, false, tostring(result)
    end
end

-- ทดสอบ
print("=== Exactly-Once Delivery ===")

local eo = ExactlyOnce.new({ dedup_window = 60 })

-- Simulate network duplicates
local events = {
    { id = "msg_001", data = { type = "payment", amount = 500 } },
    { id = "msg_002", data = { type = "order",   id = 42 } },
    { id = "msg_001", data = { type = "payment", amount = 500 } },  -- duplicate
    { id = "msg_003", data = { type = "refund",  amount = 200 } },
    { id = "msg_002", data = { type = "order",   id = 42 } },  -- duplicate
    { id = "msg_001", data = { type = "payment", amount = 500 } },  -- duplicate
}

local processed_payments = 0
local processed_orders   = 0

print("Processing events (with duplicates):")
for _, event in ipairs(events) do
    local ok, result, was_dup = eo:process(event.id, event.data, function(msg)
        if msg.type == "payment" then
            processed_payments = processed_payments + 1
            print(string.format("  [NEW] Processing payment of %.0f THB", msg.amount))
            return { status = "charged" }
        elseif msg.type == "order" then
            processed_orders = processed_orders + 1
            print(string.format("  [NEW] Creating order #%d", msg.id))
            return { status = "created" }
        elseif msg.type == "refund" then
            print(string.format("  [NEW] Processing refund of %.0f THB", msg.amount))
            return { status = "refunded" }
        end
    end)
    
    if was_dup then
        print("  [DUP] Skipped")
    end
end

print(string.format("\nResults: payments=%d, orders=%d (should be 1 each)",
    processed_payments, processed_orders))

local stats = eo.stats
print(string.format("Stats: received=%d, processed=%d, duplicates=%d",
    stats.received, stats.processed, stats.duplicates))
```

---

## สรุปบทที่ 60

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องใช้ Message Queue** - Decoupling, resilience, load leveling
2. **Queue Patterns** - FIFO, LIFO, Priority, Delay, Dead Letter Queue
3. **In-Memory FIFO Queue** - Simple queue implementation
4. **Priority Queue** - Heap-based priority processing
5. **Delay Queue** - Scheduled execution, exponential backoff
6. **Dead Letter Queue** - Failed message handling
7. **Redis-Based Queue** - RPUSH/LPOP, processing queue pattern
8. **Pub/Sub** - Message broker, topic-based delivery
9. **Topic Exchange** - AMQP-style routing keys with wildcards
10. **Message Retry** - Exponential backoff, max retries
11. **Idempotency** - Prevent duplicate processing
12. **Consumer Groups** - Parallel processing, message distribution
13. **Background Job Processor** - Complete job system with middleware
14. **Exactly-Once Delivery** - Deduplication guarantee

### Key Principles

- **At-most-once**: ส่งครั้งเดียว อาจหาย (fast but lossy)
- **At-least-once**: ส่งซ้ำได้ ต้องจัดการ duplicates (most common)
- **Exactly-once**: ส่งครั้งเดียวแน่นอน (hardest, slowest)

### Technology Recommendations

| Use Case | Technology |
|----------|------------|
| Simple queue | Redis Lists |
| Reliable messaging | RabbitMQ, AWS SQS |
| Event streaming | Apache Kafka, Redis Streams |
| Scheduled jobs | Redis + Sorted Set |
| Priority queue | Redis Sorted Set |

> **สำคัญ**: ใน production ใช้ message broker จริงเช่น RabbitMQ, Kafka, หรือ Redis Streams เพื่อความน่าเชื่อถือและ persistence
