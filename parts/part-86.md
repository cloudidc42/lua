# บทที่ 86: Concurrent Programming Patterns

## บทนำ: Concurrency ใน Lua

Lua เป็นภาษาที่ทำงานแบบ single-threaded โดยธรรมชาติ แต่นั่นไม่ได้หมายความว่าเราไม่สามารถเขียน concurrent code ได้ ในบทนี้เราจะสำรวจ patterns ต่าง ๆ ที่ทำให้ Lua จัดการกับงาน concurrent ได้อย่างมีประสิทธิภาพ

### ความแตกต่างระหว่าง Concurrency และ Parallelism

**Concurrency** คือการที่หลาย task ดำเนินการในช่วงเวลาที่ทับซ้อนกัน (overlapping time periods) แต่ไม่จำเป็นต้องทำงานพร้อมกันจริง ๆ

**Parallelism** คือการที่หลาย task ทำงานพร้อมกันจริง ๆ ในเวลาเดียวกัน (simultaneously)

```lua
-- ตัวอย่างที่ 1: แสดงให้เห็นความแตกต่าง
-- Concurrent (ใน Lua ด้วย coroutines)
local function task1()
    print("Task 1: เริ่มต้น")
    coroutine.yield()  -- หยุดชั่วคราว ให้ task อื่นทำงาน
    print("Task 1: ดำเนินต่อ")
    coroutine.yield()
    print("Task 1: สิ้นสุด")
end

local function task2()
    print("Task 2: เริ่มต้น")
    coroutine.yield()
    print("Task 2: ดำเนินต่อ")
    coroutine.yield()
    print("Task 2: สิ้นสุด")
end

local co1 = coroutine.create(task1)
local co2 = coroutine.create(task2)

-- สลับระหว่าง tasks (concurrent scheduling)
coroutine.resume(co1)  -- Task 1: เริ่มต้น
coroutine.resume(co2)  -- Task 2: เริ่มต้น
coroutine.resume(co1)  -- Task 1: ดำเนินต่อ
coroutine.resume(co2)  -- Task 2: ดำเนินต่อ
coroutine.resume(co1)  -- Task 1: สิ้นสุด
coroutine.resume(co2)  -- Task 2: สิ้นสุด
```

## Coroutine Scheduler พื้นฐาน

```lua
-- ตัวอย่างที่ 2: Simple Round-Robin Scheduler
local Scheduler = {}
Scheduler.__index = Scheduler

function Scheduler.new()
    return setmetatable({
        tasks = {},
        current = 0
    }, Scheduler)
end

function Scheduler:add(fn, ...)
    local co = coroutine.create(fn)
    local args = {...}
    table.insert(self.tasks, {co = co, args = args})
    return co
end

function Scheduler:run()
    while #self.tasks > 0 do
        self.current = (self.current % #self.tasks) + 1
        local task = self.tasks[self.current]
        
        local ok, err = coroutine.resume(task.co, table.unpack(task.args))
        task.args = {}  -- clear args after first resume
        
        if not ok then
            print("Error in task:", err)
            table.remove(self.tasks, self.current)
            self.current = self.current - 1
        elseif coroutine.status(task.co) == "dead" then
            table.remove(self.tasks, self.current)
            self.current = self.current - 1
        end
    end
end

-- ทดสอบ scheduler
local sched = Scheduler.new()

sched:add(function()
    for i = 1, 3 do
        print("Worker A:", i)
        coroutine.yield()
    end
end)

sched:add(function()
    for i = 1, 3 do
        print("Worker B:", i)
        coroutine.yield()
    end
end)

sched:run()
-- Output:
-- Worker A: 1
-- Worker B: 1
-- Worker A: 2
-- Worker B: 2
-- Worker A: 3
-- Worker B: 3
```

## Priority Scheduler

```lua
-- ตัวอย่างที่ 3: Priority-based Scheduler
local PriorityScheduler = {}
PriorityScheduler.__index = PriorityScheduler

function PriorityScheduler.new()
    return setmetatable({
        queues = {
            high = {},
            medium = {},
            low = {}
        }
    }, PriorityScheduler)
end

function PriorityScheduler:add(fn, priority)
    priority = priority or "medium"
    local co = coroutine.create(fn)
    table.insert(self.queues[priority], co)
    return co
end

function PriorityScheduler:step()
    -- ตรวจ high priority ก่อน
    for _, queue_name in ipairs({"high", "medium", "low"}) do
        local queue = self.queues[queue_name]
        if #queue > 0 then
            local co = table.remove(queue, 1)
            local ok, err = coroutine.resume(co)
            if not ok then
                print("Error:", err)
            elseif coroutine.status(co) ~= "dead" then
                table.insert(queue, co)
            end
            return true  -- ทำงาน 1 task
        end
    end
    return false  -- ไม่มี task แล้ว
end

function PriorityScheduler:run()
    while self:step() do end
end

local psched = PriorityScheduler.new()

psched:add(function()
    print("Low priority task: start")
    coroutine.yield()
    print("Low priority task: end")
end, "low")

psched:add(function()
    print("High priority task: start")
    coroutine.yield()
    print("High priority task: end")
end, "high")

psched:add(function()
    print("Medium priority task: start")
    coroutine.yield()
    print("Medium priority task: end")
end, "medium")

psched:run()
```

## Actor Model

Actor Model เป็น pattern ที่แต่ละ actor เป็น independent unit ที่สื่อสารกันผ่าน messages

```lua
-- ตัวอย่างที่ 4: Actor Model Implementation
local Actor = {}
Actor.__index = Actor

local actor_registry = {}
local actor_id_counter = 0

function Actor.new(behavior)
    actor_id_counter = actor_id_counter + 1
    local self = setmetatable({
        id = actor_id_counter,
        mailbox = {},
        behavior = behavior,
        running = true,
        co = nil
    }, Actor)
    
    self.co = coroutine.create(function()
        while self.running do
            if #self.mailbox > 0 then
                local msg = table.remove(self.mailbox, 1)
                self.behavior(self, msg)
            else
                coroutine.yield()
            end
        end
    end)
    
    actor_registry[self.id] = self
    return self
end

function Actor:send(message)
    table.insert(self.mailbox, message)
end

function Actor:stop()
    self.running = false
end

function Actor.spawn(behavior)
    return Actor.new(behavior)
end

-- Global scheduler for actors
local function run_actors()
    local has_work = true
    while has_work do
        has_work = false
        for id, actor in pairs(actor_registry) do
            if coroutine.status(actor.co) ~= "dead" then
                local ok, err = coroutine.resume(actor.co)
                if not ok then
                    print("Actor", id, "error:", err)
                    actor_registry[id] = nil
                elseif #actor.mailbox > 0 then
                    has_work = true
                end
            end
        end
    end
end

-- ตัวอย่าง: Counter Actor
local counter = Actor.spawn(function(self, msg)
    if msg.type == "increment" then
        self.count = (self.count or 0) + (msg.value or 1)
        print("Counter:", self.count)
    elseif msg.type == "get" then
        -- ส่งกลับไปยัง sender
        if msg.reply_to then
            actor_registry[msg.reply_to]:send({
                type = "count_reply",
                value = self.count or 0
            })
        end
    end
end)

-- Printer Actor  
local printer = Actor.spawn(function(self, msg)
    if msg.type == "count_reply" then
        print("Final count received:", msg.value)
    end
end)

-- ส่ง messages
counter:send({type = "increment", value = 5})
counter:send({type = "increment", value = 3})
counter:send({type = "get", reply_to = printer.id})

run_actors()
```

## CSP (Communicating Sequential Processes)

```lua
-- ตัวอย่างที่ 5: CSP Channel Implementation
local Channel = {}
Channel.__index = Channel

function Channel.new(capacity)
    return setmetatable({
        buffer = {},
        capacity = capacity or math.huge,
        senders = {},    -- blocked senders
        receivers = {}   -- blocked receivers
    }, Channel)
end

function Channel:send(value)
    if #self.receivers > 0 then
        -- มี receiver รออยู่ ส่งตรง ๆ ได้เลย
        local receiver = table.remove(self.receivers, 1)
        receiver.value = value
        receiver.ready = true
        coroutine.resume(receiver.co)
        return true
    elseif #self.buffer < self.capacity then
        -- buffer ยังมีที่ว่าง
        table.insert(self.buffer, value)
        return true
    else
        -- buffer เต็ม ต้อง block
        local sender = {
            co = coroutine.running(),
            value = value,
            sent = false
        }
        table.insert(self.senders, sender)
        coroutine.yield()
        return sender.sent
    end
end

function Channel:receive()
    if #self.buffer > 0 then
        local value = table.remove(self.buffer, 1)
        -- unblock sender ถ้ามี
        if #self.senders > 0 then
            local sender = table.remove(self.senders, 1)
            table.insert(self.buffer, sender.value)
            sender.sent = true
            coroutine.resume(sender.co)
        end
        return value
    elseif #self.senders > 0 then
        local sender = table.remove(self.senders, 1)
        sender.sent = true
        coroutine.resume(sender.co)
        return sender.value
    else
        -- block จนกว่าจะมีข้อมูล
        local receiver = {
            co = coroutine.running(),
            value = nil,
            ready = false
        }
        table.insert(self.receivers, receiver)
        coroutine.yield()
        return receiver.value
    end
end

function Channel:close()
    self.closed = true
    -- wake up all blocked receivers
    for _, receiver in ipairs(self.receivers) do
        receiver.value = nil
        receiver.ready = true
        coroutine.resume(receiver.co)
    end
    self.receivers = {}
end

-- ตัวอย่างการใช้งาน CSP
local function producer(ch, count)
    for i = 1, count do
        ch:send(i)
        print("Produced:", i)
    end
    ch:close()
end

local function consumer(ch, name)
    while true do
        local value = ch:receive()
        if value == nil then
            print(name .. " channel closed")
            break
        end
        print(name .. " consumed:", value)
    end
end

-- สร้าง buffered channel
local ch = Channel.new(2)

-- ทำงานใน coroutines
local prod = coroutine.create(function() producer(ch, 5) end)
local cons = coroutine.create(function() consumer(ch, "Consumer1") end)

-- Interleave execution
for _ = 1, 10 do
    if coroutine.status(prod) ~= "dead" then
        coroutine.resume(prod)
    end
    if coroutine.status(cons) ~= "dead" then
        coroutine.resume(cons)
    end
end
```

## Select Pattern

Select pattern ช่วยให้เราสามารถรอรับข้อมูลจากหลาย channels พร้อมกันได้

```lua
-- ตัวอย่างที่ 6: Select Pattern
local function select_channel(channels)
    -- ตรวจสอบว่า channel ไหนมีข้อมูลพร้อมก่อน
    for i, ch in ipairs(channels) do
        if #ch.buffer > 0 then
            return i, ch:receive()
        end
    end
    
    -- ถ้าไม่มีเลย ให้ register เป็น receiver ทุก channel
    local current_co = coroutine.running()
    local result_index = nil
    local result_value = nil
    
    local receivers = {}
    for i, ch in ipairs(channels) do
        local receiver = {
            co = current_co,
            channel_index = i,
            value = nil,
            ready = false,
            select_done = false
        }
        table.insert(receivers, receiver)
        table.insert(ch.receivers, receiver)
    end
    
    -- yield จนกว่าจะมี channel พร้อม
    coroutine.yield()
    
    -- cleanup และ return ผลลัพธ์
    for i, receiver in ipairs(receivers) do
        if receiver.ready then
            result_index = i
            result_value = receiver.value
        end
        -- remove from channel's receiver list
        for j, ch_receiver in ipairs(channels[i].receivers) do
            if ch_receiver == receiver then
                table.remove(channels[i].receivers, j)
                break
            end
        end
    end
    
    return result_index, result_value
end

-- ตัวอย่าง: Fan-in pattern ด้วย select
local function fanin(ch1, ch2)
    local merged = Channel.new(10)
    
    local co1 = coroutine.create(function()
        while true do
            local v = ch1:receive()
            if v == nil then break end
            merged:send({source = "ch1", value = v})
        end
    end)
    
    local co2 = coroutine.create(function()
        while true do
            local v = ch2:receive()
            if v == nil then break end
            merged:send({source = "ch2", value = v})
        end
    end)
    
    -- Run both forwarders
    coroutine.resume(co1)
    coroutine.resume(co2)
    
    return merged
end
```

## Work Stealing Queue

```lua
-- ตัวอย่างที่ 7: Work Stealing Queue
local WorkStealingQueue = {}
WorkStealingQueue.__index = WorkStealingQueue

function WorkStealingQueue.new(num_workers)
    local self = setmetatable({
        workers = {},
        num_workers = num_workers or 4
    }, WorkStealingQueue)
    
    for i = 1, num_workers do
        self.workers[i] = {
            deque = {},  -- double-ended queue
            id = i
        }
    end
    
    return self
end

-- Push task ไปที่ worker ของตัวเอง (push to top)
function WorkStealingQueue:push(worker_id, task)
    local worker = self.workers[worker_id]
    table.insert(worker.deque, task)
end

-- Pop task ของตัวเอง (pop from top - LIFO)
function WorkStealingQueue:pop(worker_id)
    local worker = self.workers[worker_id]
    if #worker.deque == 0 then
        return nil
    end
    return table.remove(worker.deque)
end

-- Steal task จาก worker อื่น (steal from bottom - FIFO)
function WorkStealingQueue:steal(thief_id)
    for i = 1, self.num_workers do
        if i ~= thief_id then
            local victim = self.workers[i]
            if #victim.deque > 0 then
                local stolen = table.remove(victim.deque, 1)
                print(string.format("Worker %d stealing from Worker %d", thief_id, i))
                return stolen
            end
        end
    end
    return nil
end

-- Worker function
function WorkStealingQueue:run_worker(worker_id)
    local worker = self.workers[worker_id]
    
    while true do
        -- ลองทำงานของตัวเองก่อน
        local task = self:pop(worker_id)
        
        if task == nil then
            -- ไม่มีงาน ลอง steal
            task = self:steal(worker_id)
        end
        
        if task == nil then
            -- ไม่มีงานเลย หยุด
            break
        end
        
        -- Execute task
        local ok, err = pcall(task)
        if not ok then
            print("Worker", worker_id, "task error:", err)
        end
        
        coroutine.yield()  -- ให้ worker อื่นทำงานบ้าง
    end
    
    print("Worker", worker_id, "finished")
end

-- ตัวอย่างการใช้งาน
local wsq = WorkStealingQueue.new(3)

-- เพิ่มงานไปที่ worker 1 มาก ๆ
for i = 1, 6 do
    local task_num = i
    wsq:push(1, function()
        print(string.format("Task %d executed", task_num))
    end)
end

-- สร้าง coroutines สำหรับ workers
local cos = {}
for i = 1, 3 do
    local worker_id = i
    cos[i] = coroutine.create(function()
        wsq:run_worker(worker_id)
    end)
end

-- Run workers
local running = true
while running do
    running = false
    for i = 1, 3 do
        if coroutine.status(cos[i]) ~= "dead" then
            coroutine.resume(cos[i])
            running = true
        end
    end
end
```

## Event-Driven Concurrency

```lua
-- ตัวอย่างที่ 8: Event Loop Implementation
local EventLoop = {}
EventLoop.__index = EventLoop

function EventLoop.new()
    return setmetatable({
        handlers = {},
        timers = {},
        io_handlers = {},
        running = false,
        tick = 0
    }, EventLoop)
end

function EventLoop:on(event, handler)
    if not self.handlers[event] then
        self.handlers[event] = {}
    end
    table.insert(self.handlers[event], handler)
end

function EventLoop:emit(event, ...)
    local args = {...}
    if self.handlers[event] then
        for _, handler in ipairs(self.handlers[event]) do
            handler(table.unpack(args))
        end
    end
end

function EventLoop:set_timeout(delay_ticks, callback)
    table.insert(self.timers, {
        fire_at = self.tick + delay_ticks,
        callback = callback,
        repeating = false
    })
end

function EventLoop:set_interval(interval_ticks, callback)
    table.insert(self.timers, {
        fire_at = self.tick + interval_ticks,
        interval = interval_ticks,
        callback = callback,
        repeating = true
    })
end

function EventLoop:step()
    self.tick = self.tick + 1
    
    -- Process timers
    local i = 1
    while i <= #self.timers do
        local timer = self.timers[i]
        if self.tick >= timer.fire_at then
            timer.callback()
            if timer.repeating then
                timer.fire_at = self.tick + timer.interval
                i = i + 1
            else
                table.remove(self.timers, i)
            end
        else
            i = i + 1
        end
    end
end

function EventLoop:run(max_ticks)
    self.running = true
    local tick = 0
    
    while self.running do
        tick = tick + 1
        if max_ticks and tick > max_ticks then
            break
        end
        self:step()
    end
end

function EventLoop:stop()
    self.running = false
end

-- ตัวอย่างการใช้งาน Event Loop
local loop = EventLoop.new()

-- Register event handlers
loop:on("data", function(data)
    print("Received data:", data)
end)

loop:on("error", function(err)
    print("Error occurred:", err)
end)

-- Set timers
loop:set_timeout(5, function()
    loop:emit("data", "First message")
end)

loop:set_timeout(10, function()
    loop:emit("data", "Second message")
end)

loop:set_interval(7, function()
    print("Interval tick:", loop.tick)
end)

loop:set_timeout(25, function()
    loop:stop()
end)

loop:run()
```

## Promise/Future Pattern

```lua
-- ตัวอย่างที่ 9: Promise Implementation
local Promise = {}
Promise.__index = Promise

Promise.PENDING = "pending"
Promise.FULFILLED = "fulfilled"
Promise.REJECTED = "rejected"

function Promise.new(executor)
    local self = setmetatable({
        state = Promise.PENDING,
        value = nil,
        handlers = {},
        rejection_handlers = {}
    }, Promise)
    
    local function resolve(value)
        if self.state == Promise.PENDING then
            self.state = Promise.FULFILLED
            self.value = value
            -- Execute all queued handlers
            for _, handler in ipairs(self.handlers) do
                handler(value)
            end
        end
    end
    
    local function reject(reason)
        if self.state == Promise.PENDING then
            self.state = Promise.REJECTED
            self.value = reason
            for _, handler in ipairs(self.rejection_handlers) do
                handler(reason)
            end
        end
    end
    
    local ok, err = pcall(executor, resolve, reject)
    if not ok then
        reject(err)
    end
    
    return self
end

function Promise:next(on_fulfilled, on_rejected)
    return Promise.new(function(resolve, reject)
        local function handle_fulfilled(value)
            if on_fulfilled then
                local ok, result = pcall(on_fulfilled, value)
                if ok then
                    resolve(result)
                else
                    reject(result)
                end
            else
                resolve(value)
            end
        end
        
        local function handle_rejected(reason)
            if on_rejected then
                local ok, result = pcall(on_rejected, reason)
                if ok then
                    resolve(result)
                else
                    reject(result)
                end
            else
                reject(reason)
            end
        end
        
        if self.state == Promise.FULFILLED then
            handle_fulfilled(self.value)
        elseif self.state == Promise.REJECTED then
            handle_rejected(self.value)
        else
            table.insert(self.handlers, handle_fulfilled)
            table.insert(self.rejection_handlers, handle_rejected)
        end
    end)
end

function Promise:catch(on_rejected)
    return self:next(nil, on_rejected)
end

function Promise.resolve(value)
    return Promise.new(function(resolve)
        resolve(value)
    end)
end

function Promise.reject(reason)
    return Promise.new(function(_, reject)
        reject(reason)
    end)
end

function Promise.all(promises)
    return Promise.new(function(resolve, reject)
        local results = {}
        local count = 0
        local total = #promises
        
        if total == 0 then
            resolve(results)
            return
        end
        
        for i, p in ipairs(promises) do
            p:next(function(value)
                results[i] = value
                count = count + 1
                if count == total then
                    resolve(results)
                end
            end, function(reason)
                reject(reason)
            end)
        end
    end)
end

function Promise.race(promises)
    return Promise.new(function(resolve, reject)
        for _, p in ipairs(promises) do
            p:next(resolve, reject)
        end
    end)
end

-- ตัวอย่างการใช้งาน Promise
local p1 = Promise.new(function(resolve, reject)
    -- Simulate async operation
    resolve(42)
end)

p1:next(function(value)
    print("Got value:", value)
    return value * 2
end):next(function(value)
    print("Doubled:", value)
end):catch(function(err)
    print("Error:", err)
end)

-- Promise.all example
local p2 = Promise.resolve(1)
local p3 = Promise.resolve(2)
local p4 = Promise.resolve(3)

Promise.all({p2, p3, p4}):next(function(results)
    print("All results:", table.concat(results, ", "))
end)
```

## Async/Await Pattern with Coroutines

```lua
-- ตัวอย่างที่ 10: Async/Await Simulation
local async_scheduler = {
    pending = {}
}

local function async(fn)
    return function(...)
        local args = {...}
        local co = coroutine.create(fn)
        
        local function step(...)
            local ok, result = coroutine.resume(co, ...)
            if not ok then
                error("Async error: " .. tostring(result))
            end
            
            if coroutine.status(co) ~= "dead" then
                -- result is a promise/future, wait for it
                if type(result) == "table" and result.next then
                    result:next(function(value)
                        step(value)
                    end, function(err)
                        -- handle rejection
                        coroutine.resume(co, nil, err)
                    end)
                end
            end
        end
        
        step(table.unpack(args))
    end
end

local function await(promise)
    return coroutine.yield(promise)
end

-- ตัวอย่าง async function
local function fetch_data(url)
    -- Simulate network request
    return Promise.new(function(resolve)
        -- In real code, this would be non-blocking I/O
        resolve({url = url, data = "response from " .. url})
    end)
end

local fetch_multiple = async(function()
    print("Starting async operations...")
    
    local result1 = await(fetch_data("https://api.example.com/users"))
    print("Got:", result1.data)
    
    local result2 = await(fetch_data("https://api.example.com/posts"))
    print("Got:", result2.data)
    
    print("All done!")
end)

fetch_multiple()
```

## Structured Concurrency

```lua
-- ตัวอย่างที่ 11: Structured Concurrency with Nursery
local Nursery = {}
Nursery.__index = Nursery

function Nursery.new()
    return setmetatable({
        tasks = {},
        errors = {},
        cancelled = false
    }, Nursery)
end

function Nursery:start(fn, ...)
    local args = {...}
    local co = coroutine.create(fn)
    table.insert(self.tasks, {
        co = co,
        args = args,
        name = tostring(fn)
    })
    return co
end

function Nursery:cancel()
    self.cancelled = true
end

function Nursery:wait()
    -- Run all tasks to completion
    while true do
        local all_done = true
        
        for _, task in ipairs(self.tasks) do
            if coroutine.status(task.co) ~= "dead" then
                all_done = false
                
                if not self.cancelled then
                    local ok, err = coroutine.resume(task.co, table.unpack(task.args))
                    task.args = {}
                    
                    if not ok then
                        table.insert(self.errors, err)
                        self:cancel()  -- Cancel nursery on first error
                    end
                end
            end
        end
        
        if all_done then break end
    end
    
    if #self.errors > 0 then
        error("Nursery failed with " .. #self.errors .. " error(s): " .. 
              table.concat(self.errors, "; "))
    end
end

-- Context manager style
local function with_nursery(fn)
    local nursery = Nursery.new()
    local ok, err = pcall(fn, nursery)
    
    if ok then
        nursery:wait()
    else
        nursery:cancel()
        error(err)
    end
end

-- ตัวอย่างการใช้งาน Structured Concurrency
with_nursery(function(nursery)
    nursery:start(function()
        for i = 1, 3 do
            print("Task A step", i)
            coroutine.yield()
        end
    end)
    
    nursery:start(function()
        for i = 1, 3 do
            print("Task B step", i)
            coroutine.yield()
        end
    end)
    
    -- nursery:wait() จะถูกเรียกอัตโนมัติเมื่อออกจาก scope
end)
```

## Timeout และ Cancellation

```lua
-- ตัวอย่างที่ 12: Timeout Mechanism
local function with_timeout(timeout_ticks, fn)
    local co = coroutine.create(fn)
    local tick = 0
    local result = nil
    local done = false
    
    while not done and tick < timeout_ticks do
        tick = tick + 1
        local ok, value = coroutine.resume(co)
        
        if not ok then
            return nil, "error: " .. tostring(value)
        end
        
        if coroutine.status(co) == "dead" then
            result = value
            done = true
        end
    end
    
    if not done then
        return nil, "timeout after " .. timeout_ticks .. " ticks"
    end
    
    return result, nil
end

-- ตัวอย่างที่ 13: Cancellation Token
local CancellationToken = {}
CancellationToken.__index = CancellationToken

function CancellationToken.new()
    return setmetatable({
        cancelled = false,
        callbacks = {}
    }, CancellationToken)
end

function CancellationToken:cancel()
    self.cancelled = true
    for _, cb in ipairs(self.callbacks) do
        cb()
    end
end

function CancellationToken:is_cancelled()
    return self.cancelled
end

function CancellationToken:on_cancel(callback)
    if self.cancelled then
        callback()
    else
        table.insert(self.callbacks, callback)
    end
end

-- ตัวอย่างการใช้งาน
local function cancellable_task(token, steps)
    for i = 1, steps do
        if token:is_cancelled() then
            print("Task cancelled at step", i)
            return false
        end
        print("Step", i, "of", steps)
        coroutine.yield()
    end
    return true
end

local token = CancellationToken.new()

local co = coroutine.create(function()
    return cancellable_task(token, 10)
end)

-- ทำงาน 5 steps แล้ว cancel
for i = 1, 5 do
    coroutine.resume(co)
end

token:cancel()

-- ทำงานต่อหลัง cancel
coroutine.resume(co)
```

## Deadlock Detection

```lua
-- ตัวอย่างที่ 14: Simple Deadlock Detector
local DeadlockDetector = {}
DeadlockDetector.__index = DeadlockDetector

function DeadlockDetector.new()
    return setmetatable({
        waiting_for = {},   -- task -> resource
        held_by = {},       -- resource -> task
        task_names = {}
    }, DeadlockDetector)
end

function DeadlockDetector:register_task(task_id, name)
    self.task_names[task_id] = name or tostring(task_id)
end

function DeadlockDetector:acquire(task_id, resource)
    if self.held_by[resource] then
        -- Resource held by another task
        self.waiting_for[task_id] = resource
        
        -- Check for cycle (deadlock)
        if self:detect_cycle(task_id) then
            return false, "DEADLOCK DETECTED"
        end
        return false, "blocked"
    end
    
    self.held_by[resource] = task_id
    return true
end

function DeadlockDetector:release(task_id, resource)
    if self.held_by[resource] == task_id then
        self.held_by[resource] = nil
        self.waiting_for[task_id] = nil
    end
end

function DeadlockDetector:detect_cycle(start_task)
    local visited = {}
    local task = start_task
    
    while task do
        if visited[task] then
            -- Found a cycle!
            self:report_deadlock(start_task)
            return true
        end
        visited[task] = true
        
        -- Follow the chain: task waits for resource, resource held by next task
        local resource = self.waiting_for[task]
        if resource then
            task = self.held_by[resource]
        else
            break
        end
    end
    
    return false
end

function DeadlockDetector:report_deadlock(start_task)
    print("DEADLOCK DETECTED! Cycle:")
    local task = start_task
    local path = {}
    
    while true do
        local name = self.task_names[task] or tostring(task)
        table.insert(path, name)
        
        local resource = self.waiting_for[task]
        if not resource then break end
        
        table.insert(path, "-> [" .. tostring(resource) .. "] ->")
        
        task = self.held_by[resource]
        if task == start_task then
            table.insert(path, self.task_names[task] or tostring(task))
            break
        end
    end
    
    print(table.concat(path, " "))
end

-- ตัวอย่าง: สร้าง deadlock
local detector = DeadlockDetector.new()

detector:register_task("T1", "Task-Alpha")
detector:register_task("T2", "Task-Beta")

-- T1 acquires Lock A
local ok1, _ = detector:acquire("T1", "LockA")
print("T1 acquired LockA:", ok1)

-- T2 acquires Lock B
local ok2, _ = detector:acquire("T2", "LockB")
print("T2 acquired LockB:", ok2)

-- T1 tries to acquire Lock B (blocked by T2)
local ok3, err3 = detector:acquire("T1", "LockB")
print("T1 tries LockB:", ok3, err3)

-- T2 tries to acquire Lock A -> DEADLOCK!
local ok4, err4 = detector:acquire("T2", "LockA")
print("T2 tries LockA:", ok4, err4)
```

## Livelock Prevention

```lua
-- ตัวอย่างที่ 15: Livelock Detection and Prevention
local function random_backoff(attempt)
    -- Exponential backoff with jitter
    math.randomseed(os.time() + attempt)
    local base = 2 ^ attempt
    local jitter = math.random(0, base)
    return base + jitter
end

local function try_acquire_with_backoff(resource, task_id, max_attempts)
    for attempt = 1, max_attempts do
        -- Simulate trying to acquire resource
        if math.random() > 0.3 then  -- 70% success rate
            print(string.format("Task %s acquired resource after %d attempt(s)", 
                               task_id, attempt))
            return true
        end
        
        -- Failed, backoff
        local wait = random_backoff(attempt)
        print(string.format("Task %s failed attempt %d, backing off for %d ticks", 
                           task_id, attempt, wait))
        
        -- In real code, this would be a timer/sleep
        -- Here we just simulate with a loop count
        for _ = 1, wait do
            coroutine.yield()
        end
    end
    
    return false, "max attempts exceeded"
end

-- ตัวอย่างที่ 16: Resource Ordering to Prevent Deadlock
local function acquire_ordered(resources, task_id)
    -- Sort resources by ID to ensure consistent ordering
    local sorted = {}
    for _, r in ipairs(resources) do
        table.insert(sorted, r)
    end
    table.sort(sorted)
    
    local acquired = {}
    for _, resource in ipairs(sorted) do
        print(string.format("Task %s acquiring %s", task_id, resource))
        table.insert(acquired, resource)
        coroutine.yield()
    end
    
    return acquired
end
```

## Pipeline Pattern

```lua
-- ตัวอย่างที่ 17: Pipeline with Coroutines
local function make_generator(items)
    return coroutine.wrap(function()
        for _, item in ipairs(items) do
            coroutine.yield(item)
        end
    end)
end

local function make_filter(source, predicate)
    return coroutine.wrap(function()
        for item in source do
            if predicate(item) then
                coroutine.yield(item)
            end
        end
    end)
end

local function make_mapper(source, transform)
    return coroutine.wrap(function()
        for item in source do
            coroutine.yield(transform(item))
        end
    end)
end

local function make_reducer(source, fn, initial)
    local acc = initial
    for item in source do
        acc = fn(acc, item)
    end
    return acc
end

-- ตัวอย่างการใช้งาน Pipeline
local numbers = make_generator({1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
local evens = make_filter(numbers, function(n) return n % 2 == 0 end)
local squared = make_mapper(evens, function(n) return n * n end)
local sum = make_reducer(squared, function(a, b) return a + b end, 0)

print("Sum of squares of even numbers 1-10:", sum)  -- 4+16+36+64+100 = 220
```

## Callback Hell Avoidance

```lua
-- ตัวอย่างที่ 18: Avoiding Callback Hell
-- BAD: Callback hell
local function bad_approach()
    fetch_user(1, function(user)
        fetch_posts(user.id, function(posts)
            fetch_comments(posts[1].id, function(comments)
                fetch_author(comments[1].author_id, function(author)
                    -- ลึกมาก ๆ และยากต่อการอ่าน
                    print("Author:", author.name)
                end)
            end)
        end)
    end)
end

-- GOOD: Using coroutines to flatten async code
local function good_approach()
    local co = coroutine.create(function()
        local user = yield_async(fetch_user_async, 1)
        local posts = yield_async(fetch_posts_async, user.id)
        local comments = yield_async(fetch_comments_async, posts[1].id)
        local author = yield_async(fetch_author_async, comments[1].author_id)
        print("Author:", author.name)
    end)
    
    -- Driver that handles the async operations
    drive_coroutine(co)
end

-- ตัวอย่างที่ 19: Continuation-Passing Style (CPS) Transformation
local function cps_transform(fn)
    return function(...)
        local args = {...}
        local callback = table.remove(args)
        
        local co = coroutine.create(function()
            return fn(table.unpack(args))
        end)
        
        local ok, result = coroutine.resume(co)
        if ok then
            callback(nil, result)
        else
            callback(result, nil)
        end
    end
end
```

## Advanced Channel Patterns

```lua
-- ตัวอย่างที่ 20: Multiplexer Channel
local function multiplex(channels)
    local output = Channel.new(100)
    local active = #channels
    
    for _, ch in ipairs(channels) do
        local source = ch
        coroutine.wrap(function()
            while true do
                local value = source:receive()
                if value == nil then
                    active = active - 1
                    if active == 0 then
                        output:close()
                    end
                    return
                end
                output:send(value)
            end
        end)()
    end
    
    return output
end

-- ตัวอย่างที่ 21: Fan-out Pattern
local function fanout(input, num_consumers)
    local outputs = {}
    for i = 1, num_consumers do
        outputs[i] = Channel.new(10)
    end
    
    coroutine.wrap(function()
        local idx = 1
        while true do
            local value = input:receive()
            if value == nil then
                for _, out in ipairs(outputs) do
                    out:close()
                end
                return
            end
            outputs[idx]:send(value)
            idx = (idx % num_consumers) + 1
        end
    end)()
    
    return outputs
end

-- ตัวอย่างที่ 22: Buffered Channel with Overflow Strategy
local OverflowChannel = {}
OverflowChannel.__index = OverflowChannel

OverflowChannel.DROP_OLDEST = "drop_oldest"
OverflowChannel.DROP_NEWEST = "drop_newest"
OverflowChannel.BLOCK = "block"

function OverflowChannel.new(capacity, overflow_strategy)
    return setmetatable({
        buffer = {},
        capacity = capacity,
        strategy = overflow_strategy or OverflowChannel.DROP_OLDEST,
        dropped = 0
    }, OverflowChannel)
end

function OverflowChannel:send(value)
    if #self.buffer < self.capacity then
        table.insert(self.buffer, value)
        return true
    end
    
    if self.strategy == OverflowChannel.DROP_OLDEST then
        table.remove(self.buffer, 1)
        table.insert(self.buffer, value)
        self.dropped = self.dropped + 1
        return true
    elseif self.strategy == OverflowChannel.DROP_NEWEST then
        self.dropped = self.dropped + 1
        return false  -- drop the new value
    end
    
    return false
end

function OverflowChannel:receive()
    return table.remove(self.buffer, 1)
end
```

## Reactive Streams

```lua
-- ตัวอย่างที่ 23: Observable/Reactive Pattern
local Observable = {}
Observable.__index = Observable

function Observable.new(subscribe_fn)
    return setmetatable({
        subscribe_fn = subscribe_fn
    }, Observable)
end

function Observable:subscribe(on_next, on_error, on_complete)
    local observer = {
        next = on_next or function() end,
        error = on_error or function(e) error(e) end,
        complete = on_complete or function() end,
        cancelled = false
    }
    
    self.subscribe_fn(observer)
    
    -- Return unsubscribe function
    return function()
        observer.cancelled = true
    end
end

function Observable:map(transform)
    local source = self
    return Observable.new(function(observer)
        source:subscribe(
            function(value)
                if not observer.cancelled then
                    local ok, result = pcall(transform, value)
                    if ok then
                        observer.next(result)
                    else
                        observer.error(result)
                    end
                end
            end,
            observer.error,
            observer.complete
        )
    end)
end

function Observable:filter(predicate)
    local source = self
    return Observable.new(function(observer)
        source:subscribe(
            function(value)
                if not observer.cancelled then
                    local ok, result = pcall(predicate, value)
                    if ok and result then
                        observer.next(value)
                    elseif not ok then
                        observer.error(result)
                    end
                end
            end,
            observer.error,
            observer.complete
        )
    end)
end

function Observable:take(n)
    local source = self
    return Observable.new(function(observer)
        local count = 0
        local unsubscribe
        
        unsubscribe = source:subscribe(
            function(value)
                if count < n then
                    count = count + 1
                    observer.next(value)
                    if count == n then
                        observer.complete()
                        if unsubscribe then unsubscribe() end
                    end
                end
            end,
            observer.error,
            observer.complete
        )
    end)
end

function Observable.from_array(arr)
    return Observable.new(function(observer)
        for _, v in ipairs(arr) do
            if not observer.cancelled then
                observer.next(v)
            end
        end
        observer.complete()
    end)
end

function Observable.interval(tick_interval, scheduler)
    return Observable.new(function(observer)
        local tick = 0
        scheduler:set_interval(tick_interval, function()
            tick = tick + 1
            if not observer.cancelled then
                observer.next(tick)
            end
        end)
    end)
end

-- ตัวอย่างการใช้งาน
local obs = Observable.from_array({1, 2, 3, 4, 5, 6, 7, 8, 9, 10})

obs
    :filter(function(n) return n % 2 == 0 end)
    :map(function(n) return n * n end)
    :take(3)
    :subscribe(
        function(value) print("Value:", value) end,
        function(err) print("Error:", err) end,
        function() print("Complete") end
    )
```

## Rate Limiter

```lua
-- ตัวอย่างที่ 24: Token Bucket Rate Limiter
local TokenBucket = {}
TokenBucket.__index = TokenBucket

function TokenBucket.new(rate, capacity)
    return setmetatable({
        rate = rate,          -- tokens per tick
        capacity = capacity,  -- max tokens
        tokens = capacity,    -- current tokens
        last_tick = 0
    }, TokenBucket)
end

function TokenBucket:refill(current_tick)
    local elapsed = current_tick - self.last_tick
    local new_tokens = elapsed * self.rate
    self.tokens = math.min(self.capacity, self.tokens + new_tokens)
    self.last_tick = current_tick
end

function TokenBucket:try_consume(tokens, current_tick)
    self:refill(current_tick)
    
    if self.tokens >= tokens then
        self.tokens = self.tokens - tokens
        return true
    end
    
    return false
end

function TokenBucket:consume(tokens, current_tick)
    self:refill(current_tick)
    
    if self.tokens >= tokens then
        self.tokens = self.tokens - tokens
        return 0  -- no wait needed
    end
    
    -- Calculate wait time
    local needed = tokens - self.tokens
    local wait_ticks = math.ceil(needed / self.rate)
    return wait_ticks
end

-- ตัวอย่างที่ 25: Sliding Window Rate Limiter
local SlidingWindowRateLimiter = {}
SlidingWindowRateLimiter.__index = SlidingWindowRateLimiter

function SlidingWindowRateLimiter.new(max_requests, window_size)
    return setmetatable({
        max_requests = max_requests,
        window_size = window_size,
        requests = {}  -- timestamps of recent requests
    }, SlidingWindowRateLimiter)
end

function SlidingWindowRateLimiter:allow(current_time)
    -- Remove expired requests
    local cutoff = current_time - self.window_size
    local i = 1
    while i <= #self.requests do
        if self.requests[i] < cutoff then
            table.remove(self.requests, i)
        else
            i = i + 1
        end
    end
    
    if #self.requests < self.max_requests then
        table.insert(self.requests, current_time)
        return true
    end
    
    return false
end

-- ตัวอย่างการใช้งาน Rate Limiter
local bucket = TokenBucket.new(1, 5)  -- 1 token/tick, max 5 tokens
local tick = 0

for _ = 1, 10 do
    tick = tick + 1
    if bucket:try_consume(1, tick) then
        print("Request allowed at tick", tick)
    else
        print("Request rejected at tick", tick)
    end
end
```

## Semaphore Implementation

```lua
-- ตัวอย่างที่ 26: Semaphore
local Semaphore = {}
Semaphore.__index = Semaphore

function Semaphore.new(permits)
    return setmetatable({
        permits = permits,
        waiters = {}
    }, Semaphore)
end

function Semaphore:acquire(count)
    count = count or 1
    
    if self.permits >= count then
        self.permits = self.permits - count
        return true
    end
    
    -- Block until permits available
    local waiter = {
        co = coroutine.running(),
        count = count
    }
    table.insert(self.waiters, waiter)
    coroutine.yield()
    return true
end

function Semaphore:release(count)
    count = count or 1
    self.permits = self.permits + count
    
    -- Wake up waiters
    local i = 1
    while i <= #self.waiters and self.permits > 0 do
        local waiter = self.waiters[i]
        if self.permits >= waiter.count then
            self.permits = self.permits - waiter.count
            table.remove(self.waiters, i)
            coroutine.resume(waiter.co)
        else
            i = i + 1
        end
    end
end

-- ตัวอย่างที่ 27: Read-Write Lock
local RWLock = {}
RWLock.__index = RWLock

function RWLock.new()
    return setmetatable({
        readers = 0,
        writer = false,
        write_waiters = {},
        read_waiters = {}
    }, RWLock)
end

function RWLock:read_lock()
    if not self.writer and #self.write_waiters == 0 then
        self.readers = self.readers + 1
        return
    end
    
    local waiter = {co = coroutine.running()}
    table.insert(self.read_waiters, waiter)
    coroutine.yield()
end

function RWLock:read_unlock()
    self.readers = self.readers - 1
    self:_wake_writer()
end

function RWLock:write_lock()
    if not self.writer and self.readers == 0 then
        self.writer = true
        return
    end
    
    local waiter = {co = coroutine.running()}
    table.insert(self.write_waiters, waiter)
    coroutine.yield()
end

function RWLock:write_unlock()
    self.writer = false
    
    if not self:_wake_writer() then
        self:_wake_readers()
    end
end

function RWLock:_wake_writer()
    if #self.write_waiters > 0 and self.readers == 0 then
        local waiter = table.remove(self.write_waiters, 1)
        self.writer = true
        coroutine.resume(waiter.co)
        return true
    end
    return false
end

function RWLock:_wake_readers()
    local readers_to_wake = {}
    for _, waiter in ipairs(self.read_waiters) do
        table.insert(readers_to_wake, waiter)
    end
    self.read_waiters = {}
    
    self.readers = self.readers + #readers_to_wake
    for _, waiter in ipairs(readers_to_wake) do
        coroutine.resume(waiter.co)
    end
end
```

## Message Passing Patterns

```lua
-- ตัวอย่างที่ 28: Request-Reply Pattern
local function make_request_reply_system()
    local pending_replies = {}
    local request_id = 0
    
    local function make_request(channel, payload)
        request_id = request_id + 1
        local id = request_id
        local reply_channel = Channel.new(1)
        pending_replies[id] = reply_channel
        
        channel:send({
            id = id,
            payload = payload,
            reply = reply_channel
        })
        
        -- Wait for reply
        local response = reply_channel:receive()
        pending_replies[id] = nil
        return response
    end
    
    local function handle_request(request, handler)
        local response = handler(request.payload)
        request.reply:send(response)
    end
    
    return make_request, handle_request
end

-- ตัวอย่างที่ 29: Publish-Subscribe Pattern
local PubSub = {}
PubSub.__index = PubSub

function PubSub.new()
    return setmetatable({
        subscribers = {}
    }, PubSub)
end

function PubSub:subscribe(topic, callback)
    if not self.subscribers[topic] then
        self.subscribers[topic] = {}
    end
    
    local sub_id = #self.subscribers[topic] + 1
    self.subscribers[topic][sub_id] = callback
    
    return function()
        self.subscribers[topic][sub_id] = nil
    end
end

function PubSub:publish(topic, message)
    if not self.subscribers[topic] then return end
    
    for _, callback in pairs(self.subscribers[topic]) do
        if callback then
            local ok, err = pcall(callback, message)
            if not ok then
                print("Subscriber error:", err)
            end
        end
    end
end

-- ตัวอย่างการใช้งาน PubSub
local bus = PubSub.new()

local unsub1 = bus:subscribe("user.created", function(user)
    print("Welcome email sent to:", user.email)
end)

local unsub2 = bus:subscribe("user.created", function(user)
    print("Analytics event for:", user.id)
end)

bus:subscribe("order.placed", function(order)
    print("Order processing:", order.id)
end)

bus:publish("user.created", {id = 1, email = "user@example.com"})
bus:publish("order.placed", {id = "ORD-001", amount = 99.99})

-- Unsubscribe
unsub1()
bus:publish("user.created", {id = 2, email = "user2@example.com"})
-- Only analytics handler will receive this
```

## Advanced Coroutine Patterns

```lua
-- ตัวอย่างที่ 30: Coroutine-based State Machine
local StateMachine = {}
StateMachine.__index = StateMachine

function StateMachine.new(initial_state, transitions)
    return setmetatable({
        state = initial_state,
        transitions = transitions,
        history = {initial_state},
        co = nil
    }, StateMachine)
end

function StateMachine:transition(event)
    local key = self.state .. ":" .. event
    local handler = self.transitions[key]
    
    if not handler then
        return false, "No transition from " .. self.state .. " on " .. event
    end
    
    local next_state = handler(self)
    if next_state then
        table.insert(self.history, next_state)
        self.state = next_state
        return true
    end
    
    return false, "Transition handler returned nil"
end

-- Traffic Light Example
local traffic_light = StateMachine.new("red", {
    ["red:go"] = function(sm)
        print("Changing from RED to GREEN")
        return "green"
    end,
    ["green:slow"] = function(sm)
        print("Changing from GREEN to YELLOW")
        return "yellow"
    end,
    ["yellow:stop"] = function(sm)
        print("Changing from YELLOW to RED")
        return "red"
    end
})

traffic_light:transition("go")
traffic_light:transition("slow")
traffic_light:transition("stop")
print("Final state:", traffic_light.state)
print("History:", table.concat(traffic_light.history, " -> "))

-- ตัวอย่างที่ 31: Generators
local function range(from, to, step)
    step = step or 1
    return coroutine.wrap(function()
        local i = from
        while i <= to do
            coroutine.yield(i)
            i = i + step
        end
    end)
end

local function zip(...)
    local iterators = {...}
    return coroutine.wrap(function()
        while true do
            local values = {}
            for _, iter in ipairs(iterators) do
                local v = iter()
                if v == nil then return end
                table.insert(values, v)
            end
            coroutine.yield(table.unpack(values))
        end
    end)
end

-- ใช้งาน generators
for i in range(1, 10, 2) do
    io.write(i .. " ")
end
print()

for a, b in zip(range(1, 3), range(10, 30, 10)) do
    print(a, b)
end
```

## Concurrent Data Structures

```lua
-- ตัวอย่างที่ 32: Thread-Safe Queue (Coroutine-safe)
local ConcurrentQueue = {}
ConcurrentQueue.__index = ConcurrentQueue

function ConcurrentQueue.new()
    return setmetatable({
        items = {},
        waiters = {}
    }, ConcurrentQueue)
end

function ConcurrentQueue:enqueue(item)
    if #self.waiters > 0 then
        local waiter = table.remove(self.waiters, 1)
        waiter.value = item
        coroutine.resume(waiter.co, item)
    else
        table.insert(self.items, item)
    end
end

function ConcurrentQueue:dequeue()
    if #self.items > 0 then
        return table.remove(self.items, 1)
    end
    
    -- Block until item available
    local waiter = {co = coroutine.running(), value = nil}
    table.insert(self.waiters, waiter)
    local item = coroutine.yield()
    return item
end

function ConcurrentQueue:size()
    return #self.items
end

-- ตัวอย่างที่ 33: Concurrent Map
local ConcurrentMap = {}
ConcurrentMap.__index = ConcurrentMap

function ConcurrentMap.new()
    return setmetatable({
        data = {},
        lock = RWLock and RWLock.new() or nil
    }, ConcurrentMap)
end

function ConcurrentMap:get(key)
    return self.data[key]
end

function ConcurrentMap:set(key, value)
    self.data[key] = value
end

function ConcurrentMap:delete(key)
    self.data[key] = nil
end

function ConcurrentMap:get_or_set(key, factory)
    local value = self.data[key]
    if value == nil then
        value = factory(key)
        self.data[key] = value
    end
    return value
end
```

## Event Sourcing Pattern

```lua
-- ตัวอย่างที่ 34: Event Sourcing
local EventStore = {}
EventStore.__index = EventStore

function EventStore.new()
    return setmetatable({
        events = {},
        handlers = {}
    }, EventStore)
end

function EventStore:register(event_type, handler)
    if not self.handlers[event_type] then
        self.handlers[event_type] = {}
    end
    table.insert(self.handlers[event_type], handler)
end

function EventStore:append(event)
    event.id = #self.events + 1
    event.timestamp = os.time()
    table.insert(self.events, event)
    
    -- Apply handlers
    if self.handlers[event.type] then
        for _, handler in ipairs(self.handlers[event.type]) do
            handler(event)
        end
    end
end

function EventStore:replay(from_id, to_id)
    from_id = from_id or 1
    to_id = to_id or #self.events
    
    for i = from_id, to_id do
        local event = self.events[i]
        if event and self.handlers[event.type] then
            for _, handler in ipairs(self.handlers[event.type]) do
                handler(event)
            end
        end
    end
end

-- ตัวอย่างการใช้งาน Event Sourcing
local store = EventStore.new()
local balance = 0

store:register("deposit", function(event)
    balance = balance + event.amount
    print(string.format("Deposit: +%d, Balance: %d", event.amount, balance))
end)

store:register("withdrawal", function(event)
    if balance >= event.amount then
        balance = balance - event.amount
        print(string.format("Withdrawal: -%d, Balance: %d", event.amount, balance))
    else
        print("Insufficient funds")
    end
end)

store:append({type = "deposit", amount = 1000})
store:append({type = "withdrawal", amount = 300})
store:append({type = "deposit", amount = 500})
store:append({type = "withdrawal", amount = 200})

print("Final balance:", balance)

-- Replay to reconstruct state
balance = 0
print("\nReplaying events...")
store:replay()
```

## Concurrent Testing Utilities

```lua
-- ตัวอย่างที่ 35: Race Condition Detector
local RaceDetector = {}
RaceDetector.__index = RaceDetector

function RaceDetector.new()
    return setmetatable({
        access_log = {},
        conflicts = {}
    }, RaceDetector)
end

function RaceDetector:log_access(resource, task_id, access_type)
    if not self.access_log[resource] then
        self.access_log[resource] = {}
    end
    
    local recent = self.access_log[resource]
    
    -- Check for concurrent write conflicts
    for _, entry in ipairs(recent) do
        if entry.task_id ~= task_id then
            if access_type == "write" or entry.access_type == "write" then
                table.insert(self.conflicts, {
                    resource = resource,
                    task1 = entry.task_id,
                    task2 = task_id,
                    types = entry.access_type .. "/" .. access_type
                })
            end
        end
    end
    
    table.insert(recent, {
        task_id = task_id,
        access_type = access_type,
        tick = os.clock()
    })
end

function RaceDetector:report()
    if #self.conflicts == 0 then
        print("No race conditions detected")
    else
        print(string.format("Found %d potential race condition(s):", #self.conflicts))
        for _, conflict in ipairs(self.conflicts) do
            print(string.format("  Resource '%s': Task %s (%s) vs Task %s (%s)",
                               conflict.resource,
                               conflict.task1, 
                               conflict.types:match("(.+)/"),
                               conflict.task2,
                               conflict.types:match("/(.+)")))
        end
    end
end
```

## Comprehensive Example: Concurrent Task Runner

```lua
-- ตัวอย่างที่ 36: Full Concurrent Task Runner
local TaskRunner = {}
TaskRunner.__index = TaskRunner

function TaskRunner.new(options)
    options = options or {}
    return setmetatable({
        max_concurrent = options.max_concurrent or 5,
        task_queue = {},
        running_tasks = {},
        completed_tasks = {},
        failed_tasks = {},
        on_complete = options.on_complete,
        on_error = options.on_error
    }, TaskRunner)
end

function TaskRunner:submit(name, fn, deps)
    local task = {
        name = name,
        fn = fn,
        deps = deps or {},
        status = "pending",
        result = nil,
        error = nil,
        co = nil
    }
    table.insert(self.task_queue, task)
    return task
end

function TaskRunner:_can_run(task)
    -- Check all deps are completed
    for _, dep_name in ipairs(task.deps) do
        local dep = self.completed_tasks[dep_name]
        if not dep then return false end
    end
    
    -- Check concurrency limit
    local running_count = 0
    for _ in pairs(self.running_tasks) do
        running_count = running_count + 1
    end
    
    return running_count < self.max_concurrent
end

function TaskRunner:_start_task(task)
    task.status = "running"
    task.co = coroutine.create(function()
        -- Gather dep results
        local dep_results = {}
        for _, dep_name in ipairs(task.deps) do
            dep_results[dep_name] = self.completed_tasks[dep_name].result
        end
        
        return task.fn(dep_results)
    end)
    
    self.running_tasks[task.name] = task
end

function TaskRunner:step()
    -- Try to start pending tasks
    local i = 1
    while i <= #self.task_queue do
        local task = self.task_queue[i]
        if self:_can_run(task) then
            self:_start_task(task)
            table.remove(self.task_queue, i)
        else
            i = i + 1
        end
    end
    
    -- Advance running tasks
    for name, task in pairs(self.running_tasks) do
        local ok, result = coroutine.resume(task.co)
        
        if not ok then
            task.status = "failed"
            task.error = result
            self.running_tasks[name] = nil
            self.failed_tasks[name] = task
            
            if self.on_error then
                self.on_error(task, result)
            end
        elseif coroutine.status(task.co) == "dead" then
            task.status = "completed"
            task.result = result
            self.running_tasks[name] = nil
            self.completed_tasks[name] = task
            
            if self.on_complete then
                self.on_complete(task, result)
            end
        end
    end
    
    return #self.task_queue > 0 or next(self.running_tasks) ~= nil
end

function TaskRunner:run()
    while self:step() do end
    
    -- Print summary
    local completed = 0
    local failed = 0
    for _ in pairs(self.completed_tasks) do completed = completed + 1 end
    for _ in pairs(self.failed_tasks) do failed = failed + 1 end
    
    print(string.format("Run complete: %d completed, %d failed", completed, failed))
end

-- ตัวอย่างการใช้งาน Task Runner
local runner = TaskRunner.new({
    max_concurrent = 3,
    on_complete = function(task, result)
        print(string.format("Task '%s' completed: %s", task.name, tostring(result)))
    end,
    on_error = function(task, err)
        print(string.format("Task '%s' failed: %s", task.name, err))
    end
})

runner:submit("fetch_config", function()
    coroutine.yield()
    return {db_host = "localhost", db_port = 5432}
end)

runner:submit("connect_db", function(deps)
    local config = deps["fetch_config"]
    coroutine.yield()
    return "connected to " .. config.db_host
end, {"fetch_config"})

runner:submit("fetch_users", function(deps)
    coroutine.yield()
    return {{id = 1, name = "Alice"}, {id = 2, name = "Bob"}}
end, {"connect_db"})

runner:submit("process_users", function(deps)
    local users = deps["fetch_users"]
    coroutine.yield()
    return "processed " .. #users .. " users"
end, {"fetch_users"})

runner:run()
```

## Backpressure Handling

```lua
-- ตัวอย่างที่ 37: Backpressure with Flow Control
local FlowControlledStream = {}
FlowControlledStream.__index = FlowControlledStream

function FlowControlledStream.new(high_watermark, low_watermark)
    return setmetatable({
        buffer = {},
        high_watermark = high_watermark or 100,
        low_watermark = low_watermark or 25,
        paused = false,
        on_drain = nil,
        on_data = nil
    }, FlowControlledStream)
end

function FlowControlledStream:write(data)
    table.insert(self.buffer, data)
    
    if self.on_data then
        self.on_data(data)
    end
    
    if #self.buffer >= self.high_watermark and not self.paused then
        self.paused = true
        print("Stream paused: buffer at", #self.buffer)
        return false  -- signal backpressure
    end
    
    return true  -- ok to continue
end

function FlowControlledStream:read()
    if #self.buffer == 0 then return nil end
    
    local data = table.remove(self.buffer, 1)
    
    if self.paused and #self.buffer <= self.low_watermark then
        self.paused = false
        print("Stream resumed: buffer at", #self.buffer)
        if self.on_drain then
            self.on_drain()
        end
    end
    
    return data
end

-- ตัวอย่างที่ 38: Circuit Breaker Pattern
local CircuitBreaker = {}
CircuitBreaker.__index = CircuitBreaker

CircuitBreaker.CLOSED = "closed"      -- normal operation
CircuitBreaker.OPEN = "open"          -- failing, reject requests
CircuitBreaker.HALF_OPEN = "half_open" -- testing recovery

function CircuitBreaker.new(options)
    options = options or {}
    return setmetatable({
        state = CircuitBreaker.CLOSED,
        failure_threshold = options.failure_threshold or 5,
        success_threshold = options.success_threshold or 2,
        timeout = options.timeout or 60,
        failure_count = 0,
        success_count = 0,
        last_failure_time = nil
    }, CircuitBreaker)
end

function CircuitBreaker:call(fn, ...)
    if self.state == CircuitBreaker.OPEN then
        -- Check if timeout has elapsed
        if os.time() - self.last_failure_time >= self.timeout then
            self.state = CircuitBreaker.HALF_OPEN
            self.success_count = 0
            print("Circuit breaker: HALF_OPEN, testing...")
        else
            return nil, "Circuit breaker OPEN"
        end
    end
    
    local ok, result = pcall(fn, ...)
    
    if ok then
        self:_on_success()
        return result
    else
        self:_on_failure()
        return nil, result
    end
end

function CircuitBreaker:_on_success()
    self.failure_count = 0
    
    if self.state == CircuitBreaker.HALF_OPEN then
        self.success_count = self.success_count + 1
        if self.success_count >= self.success_threshold then
            self.state = CircuitBreaker.CLOSED
            print("Circuit breaker: CLOSED (recovered)")
        end
    end
end

function CircuitBreaker:_on_failure()
    self.failure_count = self.failure_count + 1
    self.last_failure_time = os.time()
    
    if self.failure_count >= self.failure_threshold then
        if self.state ~= CircuitBreaker.OPEN then
            self.state = CircuitBreaker.OPEN
            print("Circuit breaker: OPEN (too many failures)")
        end
    end
end

-- ตัวอย่างการใช้งาน Circuit Breaker
local cb = CircuitBreaker.new({failure_threshold = 3})

local fail_count = 0
local function unreliable_service()
    fail_count = fail_count + 1
    if fail_count <= 4 then
        error("Service unavailable")
    end
    return "Success!"
end

for i = 1, 8 do
    local result, err = cb:call(unreliable_service)
    if result then
        print("Call", i, "succeeded:", result)
    else
        print("Call", i, "failed:", err)
    end
    
    -- Simulate time passing for half-open
    if i == 6 then
        cb.last_failure_time = os.time() - 61  -- force timeout
    end
end
```

## Comprehensive Testing for Concurrent Code

```lua
-- ตัวอย่างที่ 39: Concurrent Test Framework
local ConcurrentTest = {}
ConcurrentTest.__index = ConcurrentTest

function ConcurrentTest.new(name)
    return setmetatable({
        name = name,
        scenarios = {},
        results = {}
    }, ConcurrentTest)
end

function ConcurrentTest:scenario(name, setup_fn, workers)
    table.insert(self.scenarios, {
        name = name,
        setup = setup_fn,
        workers = workers
    })
end

function ConcurrentTest:run()
    print("\n=== Concurrent Test:", self.name, "===")
    
    for _, scenario in ipairs(self.scenarios) do
        print("\nScenario:", scenario.name)
        
        local shared_state = scenario.setup()
        local coroutines = {}
        
        for i, worker_fn in ipairs(scenario.workers) do
            local co = coroutine.create(function()
                worker_fn(shared_state, i)
            end)
            table.insert(coroutines, co)
        end
        
        -- Run all workers interleaved
        local running = true
        while running do
            running = false
            for _, co in ipairs(coroutines) do
                if coroutine.status(co) ~= "dead" then
                    running = true
                    local ok, err = coroutine.resume(co)
                    if not ok then
                        print("Worker error:", err)
                    end
                end
            end
        end
        
        print("Shared state after scenario:", require and require("inspect")(shared_state) 
              or tostring(shared_state))
    end
end

-- ตัวอย่างที่ 40: Property-Based Testing for Concurrent Code
local function concurrent_property_test(property_name, property_fn, num_trials)
    num_trials = num_trials or 100
    local failures = 0
    
    print("Testing property:", property_name)
    
    for trial = 1, num_trials do
        -- Generate random interleaving
        local num_tasks = math.random(2, 5)
        local tasks = {}
        
        for i = 1, num_tasks do
            tasks[i] = coroutine.create(function()
                property_fn(i, num_tasks)
            end)
        end
        
        -- Random interleaving
        while #tasks > 0 do
            local idx = math.random(1, #tasks)
            local co = tasks[idx]
            local ok, err = coroutine.resume(co)
            
            if not ok then
                failures = failures + 1
                print(string.format("Trial %d failed: %s", trial, err))
            end
            
            if coroutine.status(co) == "dead" then
                table.remove(tasks, idx)
            end
        end
    end
    
    if failures == 0 then
        print(string.format("Property holds for %d trials", num_trials))
    else
        print(string.format("Property VIOLATED in %d/%d trials", failures, num_trials))
    end
end
```

## สรุปบทที่ 86

ในบทนี้เราได้เรียนรู้ Concurrent Programming Patterns ใน Lua ครอบคลุม:

1. **Coroutine Scheduler** - Round-robin และ Priority-based scheduling
2. **Actor Model** - Independent actors communicating via messages
3. **CSP Channels** - Channel-based communication patterns
4. **Select Pattern** - Waiting on multiple channels
5. **Work Stealing** - Load balancing across workers
6. **Event-Driven Concurrency** - Event loop implementation
7. **Promise/Future** - Async operation abstractions
8. **Structured Concurrency** - Nursery pattern for safe concurrency
9. **Timeout/Cancellation** - Controlling async operations
10. **Deadlock Detection** - Detecting and preventing deadlocks
11. **Livelock Prevention** - Backoff strategies
12. **Pipeline Pattern** - Composable data transformation
13. **Rate Limiting** - Token bucket and sliding window
14. **Semaphore/RWLock** - Synchronization primitives
15. **Reactive Streams** - Observable pattern
16. **Circuit Breaker** - Fault tolerance pattern
17. **Backpressure** - Flow control mechanisms

Key takeaways:
- Lua's single-threaded nature ไม่ใช่ข้อจำกัด แต่เป็น feature ที่ทำให้ concurrency ง่ายต่อการ reason
- Coroutines เป็น building block หลักสำหรับ concurrent patterns
- Channels และ message passing ทำให้โค้ด thread-safe โดยธรรมชาติ
- Structured concurrency ช่วยป้องกัน resource leaks และ orphaned tasks
