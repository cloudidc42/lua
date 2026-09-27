# บทที่ 80: Advanced Coroutines - Async Patterns ใน Lua

## บทนำ

Coroutines ใน Lua เป็น feature ที่ทรงพลังมาก สามารถใช้สร้าง async patterns ได้อย่างยืดหยุ่น บทนี้จะสำรวจการสร้าง async/await, Promise, Future, และ event loop ด้วย coroutines

---

## 1. พื้นฐาน Coroutine Recap

```lua
-- Coroutine basics ก่อนเข้าสู่ async patterns

-- coroutine.create - สร้าง coroutine
local co = coroutine.create(function(x)
    print("Started with", x)
    local y = coroutine.yield(x + 1)
    print("Resumed with", y)
    return y * 2
end)

print(coroutine.status(co))  -- suspended

local ok, val = coroutine.resume(co, 10)
print("First resume:", ok, val)   -- true 11

local ok2, val2 = coroutine.resume(co, 20)
print("Second resume:", ok2, val2)  -- true 40

print(coroutine.status(co))  -- dead

-- coroutine.wrap - simpler interface
local gen = coroutine.wrap(function()
    for i = 1, 5 do
        coroutine.yield(i)
    end
end)

io.write("Generator: ")
for val in gen do
    io.write(val .. " ")
end
print()
```

---

## 2. Async/Await Simulation

```lua
-- Simulate async/await ด้วย coroutines
-- Pattern: function ที่เป็น "async" คือ coroutine
-- "await" คือ yield ที่รอผลจาก async operation

-- Async scheduler
local Scheduler = {}
Scheduler.__index = Scheduler

function Scheduler.new()
    return setmetatable({
        _queue = {},
        _running = false
    }, Scheduler)
end

function Scheduler:schedule(fn, ...)
    local args = { ... }
    local co = coroutine.create(function()
        fn(table.unpack(args))
    end)
    table.insert(self._queue, co)
    return co
end

function Scheduler:run()
    self._running = true
    
    while #self._queue > 0 do
        local nextQueue = {}
        
        for _, co in ipairs(self._queue) do
            if coroutine.status(co) == "suspended" then
                local ok, err = coroutine.resume(co)
                if not ok then
                    print("[Scheduler] Error:", err)
                end
                
                if coroutine.status(co) == "suspended" then
                    table.insert(nextQueue, co)
                end
            end
        end
        
        self._queue = nextQueue
    end
    
    self._running = false
end

-- Global scheduler instance
local scheduler = Scheduler.new()

-- "await" function - yield until value is ready
local function await(asyncFn, ...)
    local result = nil
    local done = false
    
    asyncFn(function(value)
        result = value
        done = true
    end, ...)
    
    while not done do
        coroutine.yield()
    end
    
    return result
end

-- Simulated async operations
local function asyncSleep(ms, callback)
    -- In real implementation, would use timer
    -- Here we just simulate by yielding once
    scheduler:schedule(function()
        coroutine.yield()
        if callback then callback() end
    end)
end

local function asyncFetch(url, callback)
    scheduler:schedule(function()
        -- Simulate network delay
        coroutine.yield()
        local data = { url = url, body = "Response from " .. url, status = 200 }
        if callback then callback(data) end
    end)
end

-- Async function using callbacks
local function fetchUser(userId, callback)
    asyncFetch("https://api.example.com/users/" .. userId, function(response)
        callback({ id = userId, name = "User " .. userId })
    end)
end

-- Test basic scheduling
print("Scheduling async tasks:")
scheduler:schedule(function()
    print("  Task 1 start")
    coroutine.yield()
    print("  Task 1 end")
end)

scheduler:schedule(function()
    print("  Task 2 start")
    coroutine.yield()
    print("  Task 2 end")
end)

scheduler:run()
```

---

## 3. Promise Implementation

```lua
-- Promise implementation ใน Lua
local Promise = {}
Promise.__index = Promise

local PENDING = "pending"
local FULFILLED = "fulfilled"
local REJECTED = "rejected"

function Promise.new(executor)
    local promise = setmetatable({
        _state = PENDING,
        _value = nil,
        _reason = nil,
        _resolveHandlers = {},
        _rejectHandlers = {}
    }, Promise)
    
    local function resolve(value)
        if promise._state ~= PENDING then return end
        promise._state = FULFILLED
        promise._value = value
        
        for _, handler in ipairs(promise._resolveHandlers) do
            local ok, err = pcall(handler, value)
            if not ok then print("[Promise] Handler error:", err) end
        end
    end
    
    local function reject(reason)
        if promise._state ~= PENDING then return end
        promise._state = REJECTED
        promise._reason = reason
        
        for _, handler in ipairs(promise._rejectHandlers) do
            local ok, err = pcall(handler, reason)
            if not ok then print("[Promise] Handler error:", err) end
        end
    end
    
    if executor then
        local ok, err = pcall(executor, resolve, reject)
        if not ok then
            reject(err)
        end
    end
    
    return promise, resolve, reject
end

function Promise:andThen(onFulfilled, onRejected)
    local nextPromise, resolve, reject = Promise.new()
    
    local function handleFulfilled(value)
        if type(onFulfilled) == "function" then
            local ok, result = pcall(onFulfilled, value)
            if ok then
                if type(result) == "table" and result._state then
                    -- result is a Promise
                    result:andThen(resolve, reject)
                else
                    resolve(result)
                end
            else
                reject(result)
            end
        else
            resolve(value)
        end
    end
    
    local function handleRejected(reason)
        if type(onRejected) == "function" then
            local ok, result = pcall(onRejected, reason)
            if ok then
                resolve(result)
            else
                reject(result)
            end
        else
            reject(reason)
        end
    end
    
    if self._state == FULFILLED then
        handleFulfilled(self._value)
    elseif self._state == REJECTED then
        handleRejected(self._reason)
    else
        -- Pending: register handlers
        table.insert(self._resolveHandlers, handleFulfilled)
        table.insert(self._rejectHandlers, handleRejected)
    end
    
    return nextPromise
end

function Promise:catch(onRejected)
    return self:andThen(nil, onRejected)
end

function Promise:finally(fn)
    return self:andThen(
        function(value)
            fn()
            return value
        end,
        function(reason)
            fn()
            error(reason)
        end
    )
end

-- Static methods
function Promise.resolve(value)
    return Promise.new(function(resolve) resolve(value) end)
end

function Promise.reject(reason)
    return Promise.new(function(_, reject) reject(reason) end)
end

function Promise.all(promises)
    return Promise.new(function(resolve, reject)
        local results = {}
        local remaining = #promises
        
        if remaining == 0 then
            resolve(results)
            return
        end
        
        for i, p in ipairs(promises) do
            p:andThen(
                function(value)
                    results[i] = value
                    remaining = remaining - 1
                    if remaining == 0 then
                        resolve(results)
                    end
                end,
                reject
            )
        end
    end)
end

function Promise.race(promises)
    return Promise.new(function(resolve, reject)
        for _, p in ipairs(promises) do
            p:andThen(resolve, reject)
        end
    end)
end

-- Test Promise
print("Testing Promise:")

-- Immediate resolution
local p1 = Promise.resolve(42)
p1:andThen(function(v)
    print("  Resolved:", v)
end)

-- Promise chain
Promise.resolve(10)
    :andThen(function(v)
        print("  Step 1:", v)
        return v * 2
    end)
    :andThen(function(v)
        print("  Step 2:", v)
        return v + 5
    end)
    :andThen(function(v)
        print("  Step 3:", v)
    end)

-- Error handling
Promise.reject("Something went wrong")
    :andThen(function(v)
        print("  This won't run")
    end)
    :catch(function(err)
        print("  Caught error:", err)
    end)

-- Promise.all
local p2 = Promise.resolve(1)
local p3 = Promise.resolve(2)
local p4 = Promise.resolve(3)

Promise.all({ p2, p3, p4 }):andThen(function(results)
    print("  All resolved:", table.concat(results, ", "))
end)
```

---

## 4. Future/Promise Chain

```lua
-- Future pattern - lazy evaluation
local Future = {}
Future.__index = Future

function Future.new(computation)
    return setmetatable({
        _computation = computation,
        _result = nil,
        _computed = false,
        _error = nil
    }, Future)
end

function Future:force()
    if not self._computed then
        local ok, result = pcall(self._computation)
        if ok then
            self._result = result
        else
            self._error = result
        end
        self._computed = true
    end
    
    if self._error then
        error(self._error)
    end
    
    return self._result
end

function Future:map(fn)
    return Future.new(function()
        return fn(self:force())
    end)
end

function Future:flatMap(fn)
    return Future.new(function()
        local future = fn(self:force())
        return future:force()
    end)
end

function Future:recover(fn)
    return Future.new(function()
        local ok, result = pcall(function() return self:force() end)
        if ok then
            return result
        else
            return fn(result)
        end
    end)
end

function Future.sequence(futures)
    return Future.new(function()
        local results = {}
        for i, f in ipairs(futures) do
            results[i] = f:force()
        end
        return results
    end)
end

-- Lazy computation example
local function expensiveComputation(n)
    print("  Computing factorial of " .. n)
    local result = 1
    for i = 1, n do result = result * i end
    return result
end

-- Futures are lazy - computation doesn't run until forced
print("Creating futures (no computation yet):")
local f1 = Future.new(function() return expensiveComputation(5) end)
local f2 = Future.new(function() return expensiveComputation(10) end)
local f3 = f1:map(function(v) return v * 2 end)

print("Forcing f1:")
print("  Result:", f1:force())

print("Forcing f1 again (cached):")
print("  Result:", f1:force())

print("Forcing f3 (depends on f1):")
print("  Result:", f3:force())

-- Chain of futures
local chain = Future.new(function() return 10 end)
    :map(function(v) return v + 5 end)
    :map(function(v) return v * 2 end)
    :flatMap(function(v)
        return Future.new(function() return v .. " items" end)
    end)

print("Chain result:", chain:force())

-- Error recovery
local errorFuture = Future.new(function()
    error("Something failed")
end):recover(function(err)
    print("  Recovering from:", err)
    return "default value"
end)

print("Error recovery:", errorFuture:force())
```

---

## 5. Parallel Async Operations

```lua
-- Parallel async operations ด้วย coroutines
local Async = {}

-- Task system
local TaskQueue = {}
TaskQueue.__index = TaskQueue

function TaskQueue.new()
    return setmetatable({
        _tasks = {},
        _results = {},
        _errors = {}
    }, TaskQueue)
end

function TaskQueue:add(name, fn)
    table.insert(self._tasks, { name = name, fn = fn })
    return self
end

function TaskQueue:runParallel()
    local pending = #self._tasks
    local results = {}
    local errors = {}
    
    if pending == 0 then
        return results, errors
    end
    
    -- Create coroutines for each task
    local coroutines = {}
    for _, task in ipairs(self._tasks) do
        local co = coroutine.create(task.fn)
        table.insert(coroutines, { name = task.name, co = co })
    end
    
    -- Round-robin execution
    local maxIterations = 1000
    local iteration = 0
    
    while #coroutines > 0 and iteration < maxIterations do
        iteration = iteration + 1
        local remaining = {}
        
        for _, entry in ipairs(coroutines) do
            if coroutine.status(entry.co) == "suspended" then
                local ok, value = coroutine.resume(entry.co)
                
                if coroutine.status(entry.co) == "dead" then
                    -- Task completed
                    if ok then
                        results[entry.name] = value
                    else
                        errors[entry.name] = value
                    end
                else
                    table.insert(remaining, entry)
                end
            end
        end
        
        coroutines = remaining
    end
    
    return results, errors
end

-- Async.parallel - Run multiple async functions in parallel
function Async.parallel(tasks)
    local queue = TaskQueue.new()
    
    for name, fn in pairs(tasks) do
        queue:add(name, fn)
    end
    
    return queue:runParallel()
end

-- Async.series - Run tasks in sequence
function Async.series(tasks)
    local results = {}
    local errors = {}
    
    for name, fn in pairs(tasks) do
        local co = coroutine.create(fn)
        
        while coroutine.status(co) == "suspended" do
            local ok, value = coroutine.resume(co)
            if coroutine.status(co) == "dead" then
                if ok then
                    results[name] = value
                else
                    errors[name] = value
                    return results, errors  -- Stop on error in series
                end
            end
        end
    end
    
    return results, errors
end

-- Async.waterfall - Pass results from one task to next
function Async.waterfall(tasks, initialValue)
    local currentValue = initialValue
    
    for i, task in ipairs(tasks) do
        local co = coroutine.create(function()
            return task(currentValue)
        end)
        
        while coroutine.status(co) == "suspended" do
            local ok, value = coroutine.resume(co)
            if coroutine.status(co) == "dead" then
                if ok then
                    currentValue = value
                else
                    error("Waterfall step " .. i .. " failed: " .. tostring(value))
                end
            end
        end
    end
    
    return currentValue
end

-- Test parallel operations
print("Running parallel tasks:")
local results, errors = Async.parallel({
    fetchUser = function()
        coroutine.yield()  -- Simulate async
        return { id = 1, name = "Alice" }
    end,
    fetchPosts = function()
        coroutine.yield()  -- Simulate async
        coroutine.yield()  -- Simulate longer operation
        return { { id = 1, title = "Post 1" }, { id = 2, title = "Post 2" } }
    end,
    fetchConfig = function()
        return { theme = "dark", language = "th" }
    end
})

print("  fetchUser:", results.fetchUser and results.fetchUser.name or "nil")
print("  fetchPosts:", results.fetchPosts and #results.fetchPosts or "nil")
print("  fetchConfig:", results.fetchConfig and results.fetchConfig.theme or "nil")

-- Test waterfall
print("\nRunning waterfall:")
local finalResult = Async.waterfall({
    function(input)
        print("  Step 1: got", input)
        return input .. "_step1"
    end,
    function(input)
        print("  Step 2: got", input)
        return input .. "_step2"
    end,
    function(input)
        print("  Step 3: got", input)
        return input .. "_done"
    end
}, "start")

print("  Final result:", finalResult)
```

---

## 6. Error Propagation ใน Async

```lua
-- Error Propagation ในระบบ async
local AsyncError = {}

-- Custom error types
local function AsyncException(type_, message, cause)
    return {
        type = type_,
        message = message,
        cause = cause,
        stackTrace = debug.traceback("", 2),
        isAsyncError = true
    }
end

local function TimeoutError(message, timeout)
    return AsyncException("TimeoutError", message or "Operation timed out", { timeout = timeout })
end

local function NetworkError(message, statusCode)
    return AsyncException("NetworkError", message or "Network error", { statusCode = statusCode })
end

local function CancellationError(message)
    return AsyncException("CancellationError", message or "Operation cancelled")
end

-- Async function with proper error handling
local function asyncWithError(fn, onError)
    local co = coroutine.create(fn)
    
    local function step(...)
        local ok, result = coroutine.resume(co, ...)
        
        if not ok then
            if onError then
                onError(result)
            else
                error("Async error: " .. tostring(result))
            end
            return
        end
        
        if coroutine.status(co) == "dead" then
            return result
        end
    end
    
    return step
end

-- Retry with exponential backoff
local function withRetry(fn, maxRetries, baseDelay)
    maxRetries = maxRetries or 3
    baseDelay = baseDelay or 100  -- ms
    
    local retries = 0
    
    local function attempt()
        local ok, result = pcall(fn)
        
        if ok then
            return result
        end
        
        if retries >= maxRetries then
            error("Max retries exceeded: " .. tostring(result))
        end
        
        retries = retries + 1
        local delay = baseDelay * (2 ^ (retries - 1))
        print(string.format("  [Retry] Attempt %d failed, retrying in %dms: %s",
            retries, delay, tostring(result)))
        
        -- In real implementation, would wait
        coroutine.yield()
        
        return attempt()
    end
    
    return attempt()
end

-- Test error propagation
print("Testing error handling:")

-- Successful operation
local function successfulOp()
    print("  Success operation running...")
    return "success"
end

local function failingOp()
    print("  Failing operation running...")
    error("Something went wrong")
end

-- pcall wrapper for async
local function safeAsync(fn)
    local ok, result = pcall(fn)
    if ok then
        return result, nil
    else
        return nil, result
    end
end

local result, err = safeAsync(successfulOp)
print("  Result:", result, "Error:", err)

local result2, err2 = safeAsync(failingOp)
print("  Result:", result2, "Error:", err2)

-- Retry example
local attemptCount = 0
local result3 = withRetry(function()
    attemptCount = attemptCount + 1
    if attemptCount < 3 then
        error("Temporary failure #" .. attemptCount)
    end
    return "Success after " .. attemptCount .. " attempts"
end, 5, 10)

print("  Retry result:", result3)
```

---

## 7. Timeout สำหรับ Async Operations

```lua
-- Timeout mechanism สำหรับ async operations
local TimeoutManager = {}
TimeoutManager.__index = TimeoutManager

function TimeoutManager.new()
    return setmetatable({
        _timers = {},
        _nextId = 0
    }, TimeoutManager)
end

-- Timeout wrapper สำหรับ coroutine-based operations
local function withTimeout(fn, timeoutMs)
    local startTime = os.clock()
    local timeoutSec = timeoutMs / 1000
    
    local co = coroutine.create(fn)
    local result = nil
    local timedOut = false
    
    while coroutine.status(co) == "suspended" do
        -- Check timeout
        if os.clock() - startTime > timeoutSec then
            timedOut = true
            break
        end
        
        local ok, val = coroutine.resume(co)
        if not ok then
            error("Operation failed: " .. tostring(val))
        end
        
        if coroutine.status(co) == "dead" then
            result = val
        end
    end
    
    if timedOut then
        error(string.format("Operation timed out after %dms", timeoutMs))
    end
    
    return result
end

-- Deadline-based timeout
local function withDeadline(fn, deadlineSec)
    local startTime = os.clock()
    
    return function(...)
        local elapsed = os.clock() - startTime
        if elapsed > deadlineSec then
            error("Deadline exceeded")
        end
        return fn(...)
    end
end

-- Race condition: first to complete wins
local function race(operations, timeoutMs)
    local startTime = os.clock()
    local timeoutSec = (timeoutMs or 5000) / 1000
    
    -- Create coroutines for all operations
    local coroutines = {}
    for i, op in ipairs(operations) do
        table.insert(coroutines, {
            id = i,
            co = coroutine.create(op)
        })
    end
    
    -- Run until one completes or timeout
    while #coroutines > 0 do
        if os.clock() - startTime > timeoutSec then
            error("Race timed out")
        end
        
        local remaining = {}
        for _, entry in ipairs(coroutines) do
            if coroutine.status(entry.co) == "suspended" then
                local ok, value = coroutine.resume(entry.co)
                
                if coroutine.status(entry.co) == "dead" then
                    if ok then
                        -- First to complete wins
                        return value, entry.id
                    end
                else
                    table.insert(remaining, entry)
                end
            end
        end
        coroutines = remaining
    end
    
    return nil
end

-- Test timeout
print("Testing timeout:")

-- Successful within timeout
local ok, result = pcall(withTimeout, function()
    coroutine.yield()
    return "fast result"
end, 1000)  -- 1 second timeout

print("  Success:", result)

-- Operation that finishes quickly
local ok2, result2 = pcall(withTimeout, function()
    -- No yielding = completes immediately
    return "immediate result"
end, 100)

print("  Immediate:", result2)

-- Test race condition
print("\nTesting race:")
local winner, id = race({
    function()
        -- Fast operation
        return "fast"
    end,
    function()
        -- Slower operation
        coroutine.yield()
        return "slow"
    end
}, 2000)

print("  Winner:", winner, "ID:", id)
```

---

## 8. Cancellation

```lua
-- Cancellation Token pattern
local CancellationToken = {}
CancellationToken.__index = CancellationToken

function CancellationToken.new()
    local token = setmetatable({
        _cancelled = false,
        _reason = nil,
        _handlers = {}
    }, CancellationToken)
    
    -- Return token and cancel function
    local cancel = function(reason)
        if token._cancelled then return end
        token._cancelled = true
        token._reason = reason or "Cancelled"
        
        for _, handler in ipairs(token._handlers) do
            pcall(handler, token._reason)
        end
    end
    
    return token, cancel
end

function CancellationToken:isCancelled()
    return self._cancelled
end

function CancellationToken:throwIfCancelled()
    if self._cancelled then
        error("CancellationError: " .. (self._reason or "Cancelled"))
    end
end

function CancellationToken:onCancelled(handler)
    if self._cancelled then
        handler(self._reason)
    else
        table.insert(self._handlers, handler)
    end
    return self
end

-- Cancellable async operation
local function cancellableOperation(token, fn)
    return coroutine.create(function()
        local co = coroutine.create(fn)
        
        while coroutine.status(co) == "suspended" do
            -- Check cancellation before each step
            if token:isCancelled() then
                error("Operation cancelled: " .. (token._reason or ""))
            end
            
            local ok, val = coroutine.resume(co)
            if not ok then error(val) end
        end
        
        return coroutine.resume(co)
    end)
end

-- CancellationTokenSource - can create linked tokens
local function createLinkedTokens(parentToken)
    local childToken, cancel = CancellationToken.new()
    
    if parentToken then
        parentToken:onCancelled(function(reason)
            cancel("Parent cancelled: " .. (reason or ""))
        end)
    end
    
    return childToken, cancel
end

-- Test cancellation
print("Testing cancellation:")

local token, cancel = CancellationToken.new()

-- Register cancellation handler
token:onCancelled(function(reason)
    print("  [Cancelled]", reason)
end)

-- Simulate long operation
local step = 0
local function longOperation()
    while step < 10 do
        step = step + 1
        print("  Step " .. step)
        
        token:throwIfCancelled()
        
        coroutine.yield()
    end
    return "completed"
end

local co = coroutine.create(longOperation)

-- Run for a few steps then cancel
for i = 1, 3 do
    coroutine.resume(co)
end

print("  Cancelling after 3 steps...")
cancel("User requested cancellation")

-- Try to continue after cancellation
local ok, err = coroutine.resume(co)
print("  After cancel - ok:", ok, "err:", err and err:sub(1, 40))

-- Linked tokens
print("\nLinked cancellation tokens:")
local parentToken, cancelParent = CancellationToken.new()
local childToken, cancelChild = createLinkedTokens(parentToken)

childToken:onCancelled(function(reason)
    print("  Child cancelled:", reason)
end)

cancelParent("Parent stopped")  -- This should also cancel child
print("  Parent cancelled:", parentToken:isCancelled())
print("  Child cancelled:", childToken:isCancelled())
```

---

## 9. Async Iterators

```lua
-- Async Iterator - iterate ข้อมูลที่ได้มาแบบ async
local AsyncIterator = {}
AsyncIterator.__index = AsyncIterator

function AsyncIterator.new(generator)
    return setmetatable({
        _generator = generator,
        _co = nil,
        _done = false
    }, AsyncIterator)
end

function AsyncIterator:start()
    self._co = coroutine.create(self._generator)
    return self
end

function AsyncIterator:next()
    if self._done then
        return nil, true  -- value, done
    end
    
    if not self._co then
        self:start()
    end
    
    if coroutine.status(self._co) == "dead" then
        self._done = true
        return nil, true
    end
    
    local ok, value = coroutine.resume(self._co)
    
    if not ok then
        self._done = true
        error("Iterator error: " .. tostring(value))
    end
    
    if coroutine.status(self._co) == "dead" then
        self._done = true
        return value, true
    end
    
    return value, false
end

function AsyncIterator:toArray()
    local results = {}
    while true do
        local value, done = self:next()
        if done then break end
        if value ~= nil then
            table.insert(results, value)
        end
    end
    return results
end

function AsyncIterator:map(fn)
    local source = self
    return AsyncIterator.new(function()
        while true do
            local value, done = source:next()
            if done then return end
            coroutine.yield(fn(value))
        end
    end)
end

function AsyncIterator:filter(predicate)
    local source = self
    return AsyncIterator.new(function()
        while true do
            local value, done = source:next()
            if done then return end
            if predicate(value) then
                coroutine.yield(value)
            end
        end
    end)
end

function AsyncIterator:take(n)
    local source = self
    local count = 0
    return AsyncIterator.new(function()
        while count < n do
            local value, done = source:next()
            if done then return end
            count = count + 1
            coroutine.yield(value)
        end
    end)
end

function AsyncIterator:forEach(fn)
    while true do
        local value, done = self:next()
        if done then break end
        if value ~= nil then
            fn(value)
        end
    end
end

-- Generators
local function range(start, stop, step)
    step = step or 1
    return AsyncIterator.new(function()
        local i = start
        while i <= stop do
            coroutine.yield(i)
            i = i + step
        end
    end)
end

local function infiniteCounter(start)
    start = start or 0
    return AsyncIterator.new(function()
        local n = start
        while true do
            coroutine.yield(n)
            n = n + 1
        end
    end)
end

local function fromArray(arr)
    return AsyncIterator.new(function()
        for _, v in ipairs(arr) do
            coroutine.yield(v)
        end
    end)
end

-- Test async iterators
print("Async Iterator examples:")

print("Range 1-10:")
local sum = 0
range(1, 10):forEach(function(n)
    sum = sum + n
end)
print("  Sum:", sum)

print("\nFilter even, map *2, take 5:")
local results = infiniteCounter(1)
    :filter(function(n) return n % 2 == 0 end)
    :map(function(n) return n * 2 end)
    :take(5)
    :toArray()

print("  Result:", table.concat(results, ", "))

print("\nFrom array:")
local arr = fromArray({ "apple", "banana", "cherry", "date" })
arr:filter(function(s) return #s > 5 end)
   :forEach(function(s)
       print("  Long fruit:", s)
   end)
```

---

## 10. Reactive Streams ด้วย Coroutines

```lua
-- Reactive Streams - Observable pattern ด้วย coroutines
local Observable = {}
Observable.__index = Observable

function Observable.new(subscribe)
    return setmetatable({
        _subscribe = subscribe
    }, Observable)
end

-- Create observers
local function createObserver(onNext, onError, onComplete)
    return {
        next = onNext or function() end,
        error = onError or function(err) error(err) end,
        complete = onComplete or function() end
    }
end

function Observable:subscribe(onNext, onError, onComplete)
    local observer
    
    if type(onNext) == "table" then
        observer = onNext
    else
        observer = createObserver(onNext, onError, onComplete)
    end
    
    local co = coroutine.create(function()
        self._subscribe(observer)
    end)
    
    -- Start the coroutine
    local function run()
        while coroutine.status(co) == "suspended" do
            local ok, err = coroutine.resume(co)
            if not ok then
                observer.error(err)
                return
            end
        end
    end
    
    run()
    
    -- Return subscription (unsubscribe)
    return {
        unsubscribe = function()
            -- In real implementation, signal cancellation
        end
    }
end

function Observable:map(fn)
    local source = self
    return Observable.new(function(observer)
        source:subscribe(
            function(value)
                local ok, result = pcall(fn, value)
                if ok then
                    observer.next(result)
                else
                    observer.error(result)
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
                local ok, shouldPass = pcall(predicate, value)
                if ok and shouldPass then
                    observer.next(value)
                elseif not ok then
                    observer.error(shouldPass)
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
        source:subscribe(
            function(value)
                if count < n then
                    count = count + 1
                    observer.next(value)
                    if count >= n then
                        observer.complete()
                    end
                end
            end,
            observer.error,
            observer.complete
        )
    end)
end

function Observable:reduce(fn, initial)
    local source = self
    return Observable.new(function(observer)
        local acc = initial
        source:subscribe(
            function(value)
                if acc == nil then
                    acc = value
                else
                    local ok, result = pcall(fn, acc, value)
                    if ok then acc = result
                    else observer.error(result) end
                end
            end,
            observer.error,
            function()
                observer.next(acc)
                observer.complete()
            end
        )
    end)
end

-- Static factories
function Observable.from(array)
    return Observable.new(function(observer)
        for _, v in ipairs(array) do
            observer.next(v)
        end
        observer.complete()
    end)
end

function Observable.range(start, stop)
    return Observable.new(function(observer)
        for i = start, stop do
            observer.next(i)
            coroutine.yield()  -- Allow cancellation
        end
        observer.complete()
    end)
end

function Observable.interval(ms)
    return Observable.new(function(observer)
        local i = 0
        while true do
            i = i + 1
            observer.next(i)
            coroutine.yield()  -- Each tick yields
        end
    end)
end

function Observable.merge(...)
    local sources = { ... }
    return Observable.new(function(observer)
        local completed = 0
        for _, source in ipairs(sources) do
            source:subscribe(
                observer.next,
                observer.error,
                function()
                    completed = completed + 1
                    if completed == #sources then
                        observer.complete()
                    end
                end
            )
        end
    end)
end

-- Test observables
print("Observable examples:")

print("\nBasic observable:")
Observable.from({ 1, 2, 3, 4, 5 })
    :map(function(n) return n * 2 end)
    :filter(function(n) return n > 4 end)
    :subscribe(
        function(v) print("  next:", v) end,
        nil,
        function() print("  complete!") end
    )

print("\nRange with reduce:")
Observable.range(1, 5)
    :reduce(function(acc, n) return acc + n end, 0)
    :subscribe(function(sum)
        print("  Sum:", sum)
    end)

print("\nMerge streams:")
local s1 = Observable.from({ "a", "b", "c" })
local s2 = Observable.from({ 1, 2, 3 })
Observable.merge(s1, s2):subscribe(function(v)
    io.write(tostring(v) .. " ")
end)
print()
```

---

## 11. Event Loop ใน Lua

```lua
-- Simple Event Loop implementation
local EventLoop = {}
EventLoop.__index = EventLoop

function EventLoop.new()
    return setmetatable({
        _callbacks = {},    -- Next tick callbacks
        _timers = {},       -- Timer queue
        _ioQueue = {},      -- IO event queue
        _running = false,
        _tick = 0
    }, EventLoop)
end

-- Next tick (schedule for next iteration)
function EventLoop:nextTick(fn)
    table.insert(self._callbacks, fn)
end

-- Simulated timer
function EventLoop:setTimeout(fn, delayMs)
    local targetTick = self._tick + math.ceil(delayMs / 16)  -- ~60fps
    table.insert(self._timers, {
        fn = fn,
        targetTick = targetTick
    })
    table.sort(self._timers, function(a, b) return a.targetTick < b.targetTick end)
end

-- Simulated setInterval
function EventLoop:setInterval(fn, intervalMs)
    local id = {}  -- Unique ID as table reference
    local interval = math.ceil(intervalMs / 16)
    
    local function schedule()
        local targetTick = self._tick + interval
        table.insert(self._timers, {
            fn = function()
                if not id._cancelled then
                    fn()
                    schedule()
                end
            end,
            targetTick = targetTick
        })
    end
    
    schedule()
    
    return function()
        id._cancelled = true
    end
end

-- Add IO event
function EventLoop:queueIO(event, data)
    table.insert(self._ioQueue, { event = event, data = data })
end

-- Run event loop for N ticks
function EventLoop:run(maxTicks)
    self._running = true
    maxTicks = maxTicks or 100
    
    while self._running and self._tick < maxTicks do
        self._tick = self._tick + 1
        
        -- Process callbacks (nextTick)
        local callbacks = self._callbacks
        self._callbacks = {}
        for _, fn in ipairs(callbacks) do
            local ok, err = pcall(fn)
            if not ok then
                print("[EventLoop] Callback error:", err)
            end
        end
        
        -- Process timers
        local i = 1
        while i <= #self._timers do
            local timer = self._timers[i]
            if timer.targetTick <= self._tick then
                table.remove(self._timers, i)
                local ok, err = pcall(timer.fn)
                if not ok then
                    print("[EventLoop] Timer error:", err)
                end
            else
                i = i + 1
            end
        end
        
        -- Process IO events
        local ioQueue = self._ioQueue
        self._ioQueue = {}
        for _, event in ipairs(ioQueue) do
            -- Handle IO events
            if self._ioHandlers and self._ioHandlers[event.event] then
                for _, handler in ipairs(self._ioHandlers[event.event]) do
                    pcall(handler, event.data)
                end
            end
        end
        
        -- Stop if nothing pending
        if #self._callbacks == 0 and #self._timers == 0 and #self._ioQueue == 0 then
            self._running = false
        end
    end
end

function EventLoop:stop()
    self._running = false
end

-- Test event loop
print("Event Loop simulation:")

local loop = EventLoop.new()
local log = {}

loop:nextTick(function()
    table.insert(log, "nextTick 1")
end)

loop:nextTick(function()
    table.insert(log, "nextTick 2")
end)

loop:setTimeout(function()
    table.insert(log, "setTimeout 50ms")
end, 50)

loop:setTimeout(function()
    table.insert(log, "setTimeout 100ms")
end, 100)

local cancelInterval = loop:setInterval(function()
    table.insert(log, "interval tick")
end, 100)

-- Cancel interval after a few ticks (simulated)
loop:setTimeout(function()
    cancelInterval()
    table.insert(log, "interval cancelled")
end, 350)

loop:run(50)

print("Event log:")
for _, entry in ipairs(log) do
    print("  -", entry)
end
```

---

## 12. Non-Blocking I/O Patterns

```lua
-- Non-blocking I/O Patterns
-- Simulate async file reading ด้วย coroutines

local IOManager = {}
IOManager.__index = IOManager

function IOManager.new()
    return setmetatable({
        _pendingOps = {},
        _completedOps = {}
    }, IOManager)
end

-- Simulate async file read
function IOManager:readFileAsync(path, callback)
    table.insert(self._pendingOps, {
        type = "read",
        path = path,
        callback = callback,
        co = coroutine.running()
    })
    coroutine.yield()
end

-- Process pending IO operations
function IOManager:processIO()
    local pending = self._pendingOps
    self._pendingOps = {}
    
    for _, op in ipairs(pending) do
        if op.type == "read" then
            -- Simulate file read
            local content = "Content of file: " .. op.path
            local ok, err = pcall(op.callback, nil, content)
            if not ok then
                pcall(op.callback, err, nil)
            end
        end
    end
end

-- Coroutine-based async read
local function readFile(path)
    local result = nil
    local error_ = nil
    local done = false
    
    -- Start async operation
    local function startRead()
        -- Simulate async read
        coroutine.yield()  -- Pretend to wait for IO
        result = "Content: " .. path
        done = true
    end
    
    local co = coroutine.create(startRead)
    coroutine.resume(co)
    
    -- Process IO
    if coroutine.status(co) == "suspended" then
        coroutine.resume(co)
    end
    
    return result
end

-- Stream-based file reading
local function createFileStream(content, chunkSize)
    chunkSize = chunkSize or 64
    local pos = 1
    
    return {
        read = function()
            if pos > #content then return nil end
            local chunk = content:sub(pos, pos + chunkSize - 1)
            pos = pos + chunkSize
            coroutine.yield(chunk)
            return chunk
        end,
        
        readAll = coroutine.wrap(function()
            while pos <= #content do
                local chunk = content:sub(pos, pos + chunkSize - 1)
                pos = pos + chunkSize
                coroutine.yield(chunk)
            end
        end)
    }
end

-- Buffer management
local Buffer = {}
Buffer.__index = Buffer

function Buffer.new(size)
    return setmetatable({
        _data = {},
        _size = size or 4096,
        _pos = 0
    }, Buffer)
end

function Buffer:write(data)
    if self._pos + #data > self._size then
        -- Buffer flush needed
        return false, "Buffer full"
    end
    table.insert(self._data, data)
    self._pos = self._pos + #data
    return true
end

function Buffer:flush()
    local data = table.concat(self._data)
    self._data = {}
    self._pos = 0
    return data
end

function Buffer:size()
    return self._pos
end

-- Pipeline for processing file data
local function processFileInChunks(content, processor)
    local results = {}
    local buffer = Buffer.new(128)
    
    -- Stream through content in chunks
    local gen = coroutine.wrap(function()
        local pos = 1
        while pos <= #content do
            local chunk = content:sub(pos, pos + 31)  -- 32 byte chunks
            pos = pos + 32
            coroutine.yield(chunk)
        end
    end)
    
    for chunk in gen do
        local processed = processor(chunk)
        table.insert(results, processed)
    end
    
    return table.concat(results)
end

-- Test non-blocking patterns
print("Non-blocking I/O patterns:")

local fileContent = "Hello, World! This is a test file content for streaming example."

print("\nChunked file processing:")
local processed = processFileInChunks(fileContent, function(chunk)
    return "[" .. chunk .. "]"
end)
print("Processed:", processed:sub(1, 80) .. "...")

print("\nBuffer usage:")
local buf = Buffer.new(100)
local ok1 = buf:write("Hello ")
local ok2 = buf:write("World!")
print("Write 1:", ok1)
print("Write 2:", ok2)
print("Buffer size:", buf:size())
print("Flushed:", buf:flush())
```

---

## 13. Producer-Consumer Pattern

```lua
-- Producer-Consumer pattern ด้วย coroutines
local function createChannel(capacity)
    capacity = capacity or 10
    
    local buffer = {}
    local producers = {}
    local consumers = {}
    
    local channel = {
        -- Send value to channel
        send = function(value)
            if #buffer < capacity then
                table.insert(buffer, value)
                
                -- Resume waiting consumer
                if #consumers > 0 then
                    local consumer = table.remove(consumers, 1)
                    coroutine.resume(consumer, table.remove(buffer, 1))
                end
            else
                -- Buffer full, yield
                table.insert(producers, coroutine.running())
                coroutine.yield()
                table.insert(buffer, value)
            end
        end,
        
        -- Receive value from channel
        receive = function()
            if #buffer > 0 then
                local value = table.remove(buffer, 1)
                
                -- Resume waiting producer
                if #producers > 0 then
                    local producer = table.remove(producers, 1)
                    coroutine.resume(producer)
                end
                
                return value
            else
                -- Buffer empty, yield
                table.insert(consumers, coroutine.running())
                return coroutine.yield()
            end
        end,
        
        -- Check if has data
        isEmpty = function()
            return #buffer == 0
        end,
        
        size = function()
            return #buffer
        end
    }
    
    return channel
end

-- Producer-Consumer simulation
local function runProducerConsumer()
    local channel = createChannel(5)
    local log = {}
    
    -- Producer coroutine
    local producer = coroutine.create(function()
        for i = 1, 8 do
            table.insert(log, "Producing: " .. i)
            channel.send(i)
            coroutine.yield()  -- Give consumer a turn
        end
        table.insert(log, "Producer done")
    end)
    
    -- Consumer coroutine
    local consumer = coroutine.create(function()
        local received = 0
        while received < 8 do
            coroutine.yield()  -- Give producer a turn
            if not channel.isEmpty() then
                local value = channel.receive()
                received = received + 1
                table.insert(log, "Consumed: " .. value)
            end
        end
        table.insert(log, "Consumer done")
    end)
    
    -- Interleave producer and consumer
    for i = 1, 20 do
        if coroutine.status(producer) == "suspended" then
            coroutine.resume(producer)
        end
        if coroutine.status(consumer) == "suspended" then
            coroutine.resume(consumer)
        end
        
        if coroutine.status(producer) == "dead" 
           and coroutine.status(consumer) == "dead" then
            break
        end
    end
    
    return log
end

print("Producer-Consumer Pattern:")
local log = runProducerConsumer()
for _, entry in ipairs(log) do
    print("  " .. entry)
end
```

---

## 14. Async Pipeline ด้วย Coroutines

```lua
-- Async Pipeline: compose async operations
local AsyncPipeline = {}
AsyncPipeline.__index = AsyncPipeline

function AsyncPipeline.new(source)
    return setmetatable({
        _source = source,
        _stages = {}
    }, AsyncPipeline)
end

function AsyncPipeline:pipe(fn)
    table.insert(self._stages, { type = "transform", fn = fn })
    return self
end

function AsyncPipeline:filter(pred)
    table.insert(self._stages, { type = "filter", fn = pred })
    return self
end

function AsyncPipeline:batch(size)
    table.insert(self._stages, { type = "batch", size = size })
    return self
end

function AsyncPipeline:async(fn)
    -- Mark as async stage (would use real async in production)
    table.insert(self._stages, { type = "async", fn = fn })
    return self
end

function AsyncPipeline:execute()
    local results = {}
    local currentData = {}
    
    -- Collect source data
    if type(self._source) == "table" then
        currentData = self._source
    elseif type(self._source) == "function" then
        local gen = coroutine.wrap(self._source)
        for v in gen do
            table.insert(currentData, v)
        end
    end
    
    -- Process through each stage
    for _, stage in ipairs(self._stages) do
        local nextData = {}
        
        if stage.type == "transform" then
            for _, item in ipairs(currentData) do
                local ok, result = pcall(stage.fn, item)
                if ok then
                    table.insert(nextData, result)
                end
            end
            
        elseif stage.type == "filter" then
            for _, item in ipairs(currentData) do
                local ok, pass = pcall(stage.fn, item)
                if ok and pass then
                    table.insert(nextData, item)
                end
            end
            
        elseif stage.type == "batch" then
            local batch = {}
            for _, item in ipairs(currentData) do
                table.insert(batch, item)
                if #batch >= stage.size then
                    table.insert(nextData, batch)
                    batch = {}
                end
            end
            if #batch > 0 then
                table.insert(nextData, batch)
            end
            
        elseif stage.type == "async" then
            for _, item in ipairs(currentData) do
                local result = nil
                local co = coroutine.create(function()
                    result = stage.fn(item)
                end)
                coroutine.resume(co)
                if result ~= nil then
                    table.insert(nextData, result)
                end
            end
        end
        
        currentData = nextData
    end
    
    return currentData
end

-- Test async pipeline
print("Async Pipeline:")

local numbers = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 }

local result = AsyncPipeline.new(numbers)
    :filter(function(n) return n % 2 == 0 end)  -- Keep evens
    :pipe(function(n) return n * n end)          -- Square them
    :batch(2)                                    -- Group into pairs
    :pipe(function(batch)                        -- Sum each pair
        local sum = 0
        for _, v in ipairs(batch) do sum = sum + v end
        return sum
    end)
    :execute()

print("Pipeline result:")
for i, v in ipairs(result) do
    print("  Batch " .. i .. ": " .. v)
end
```

---

## 15. Complete Async Runtime

```lua
-- Complete Async Runtime สำหรับ Lua
-- Combines: coroutines + event loop + promises + channels

local Runtime = {}
Runtime.__index = Runtime

function Runtime.new()
    local rt = setmetatable({
        _microtasks = {},    -- High priority (promises)
        _macrotasks = {},    -- Normal priority (timers, IO)
        _coroutines = {},    -- Running coroutines
        _tick = 0,
        _running = false
    }, Runtime)
    
    return rt
end

-- Microtask (runs before next macrotask)
function Runtime:queueMicrotask(fn)
    table.insert(self._microtasks, fn)
end

-- Macrotask (runs in next tick)
function Runtime:queueMacrotask(fn, delay)
    table.insert(self._macrotasks, {
        fn = fn,
        executeAt = self._tick + (delay or 0)
    })
    table.sort(self._macrotasks, function(a, b)
        return a.executeAt < b.executeAt
    end)
end

-- Spawn a coroutine
function Runtime:spawn(fn, ...)
    local args = { ... }
    local co = coroutine.create(function()
        return fn(table.unpack(args))
    end)
    table.insert(self._coroutines, co)
    return co
end

-- Run for specified ticks
function Runtime:run(maxTicks)
    self._running = true
    maxTicks = maxTicks or 50
    
    while self._running and self._tick < maxTicks do
        self._tick = self._tick + 1
        
        -- 1. Process microtasks (all of them)
        while #self._microtasks > 0 do
            local task = table.remove(self._microtasks, 1)
            pcall(task)
        end
        
        -- 2. Process one macrotask
        if #self._macrotasks > 0 and 
           self._macrotasks[1].executeAt <= self._tick then
            local task = table.remove(self._macrotasks, 1)
            pcall(task.fn)
        end
        
        -- 3. Resume suspended coroutines
        local remaining = {}
        for _, co in ipairs(self._coroutines) do
            if coroutine.status(co) == "suspended" then
                local ok, err = coroutine.resume(co)
                if not ok then
                    print("[Runtime] Coroutine error:", err)
                end
            end
            if coroutine.status(co) == "suspended" then
                table.insert(remaining, co)
            end
        end
        self._coroutines = remaining
        
        -- Stop if nothing more to do
        if #self._microtasks == 0 and 
           #self._macrotasks == 0 and 
           #self._coroutines == 0 then
            self._running = false
        end
    end
end

-- Async/await built on top of runtime
local function async(runtime, fn)
    return function(...)
        local args = { ... }
        local co = coroutine.create(function()
            fn(table.unpack(args))
        end)
        table.insert(runtime._coroutines, co)
    end
end

local function await_(runtime, asyncOp)
    local result = nil
    local done = false
    
    asyncOp(function(value)
        result = value
        done = true
    end)
    
    while not done do
        coroutine.yield()
    end
    
    return result
end

-- Test the complete runtime
print("Complete Async Runtime:")

local rt = Runtime.new()

-- Schedule some tasks
rt:queueMacrotask(function()
    print("  [Macro] Timer fired at tick", rt._tick)
end, 5)

rt:queueMicrotask(function()
    print("  [Micro] Microtask 1 at tick", rt._tick)
end)

rt:queueMicrotask(function()
    print("  [Micro] Microtask 2 at tick", rt._tick)
end)

-- Spawn coroutines
rt:spawn(function()
    print("  [Coroutine A] Start")
    coroutine.yield()
    print("  [Coroutine A] After yield 1")
    coroutine.yield()
    print("  [Coroutine A] After yield 2")
    print("  [Coroutine A] Done")
end)

rt:spawn(function()
    print("  [Coroutine B] Start")
    coroutine.yield()
    print("  [Coroutine B] After yield")
    print("  [Coroutine B] Done")
end)

rt:run(20)

print("\nRuntime completed at tick:", rt._tick)
```

---

## 16. Async HTTP Client (Simulation)

```lua
-- Simulated Async HTTP Client
local AsyncHTTP = {}
AsyncHTTP.__index = AsyncHTTP

function AsyncHTTP.new()
    return setmetatable({
        _pendingRequests = {},
        _baseURL = "https://api.example.com"
    }, AsyncHTTP)
end

function AsyncHTTP:get(path, options)
    options = options or {}
    
    -- Simulate async HTTP request using coroutine
    local co = coroutine.running()
    
    -- Would normally use actual socket/HTTP library
    local result = nil
    
    -- Simulate network delay via coroutine yield
    local function simulateRequest()
        coroutine.yield()  -- Simulate network time
        
        -- Simulate response
        local statusCode = options.mockStatus or 200
        local body = options.mockBody or 
            string.format('{"path":"%s","method":"GET"}', path)
        
        return {
            status = statusCode,
            headers = {
                ["Content-Type"] = "application/json",
                ["X-Request-ID"] = tostring(math.random(10000, 99999))
            },
            body = body,
            ok = statusCode >= 200 and statusCode < 300
        }
    end
    
    local simCo = coroutine.create(simulateRequest)
    
    -- Step through simulation
    while coroutine.status(simCo) == "suspended" do
        coroutine.resume(simCo)
    end
    
    local ok, response = coroutine.resume(simCo)
    
    if not ok then
        error("HTTP request failed: " .. tostring(response))
    end
    
    return response
end

function AsyncHTTP:post(path, body, options)
    options = options or {}
    options.mockBody = options.mockBody or 
        string.format('{"path":"%s","method":"POST","created":true}', path)
    return self:get(path, options)
end

-- API Client built on AsyncHTTP
local APIClient = {}
APIClient.__index = APIClient

function APIClient.new(baseURL)
    return setmetatable({
        _http = AsyncHTTP.new(),
        _baseURL = baseURL,
        _token = nil,
        _cache = {}
    }, APIClient)
end

function APIClient:auth(token)
    self._token = token
    return self
end

function APIClient:getUsers(options)
    options = options or {}
    options.mockBody = '[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]'
    local response = self._http:get("/users", options)
    return response
end

function APIClient:getUser(id)
    local cacheKey = "user:" .. id
    if self._cache[cacheKey] then
        print("  [Cache] Cache hit for user:" .. id)
        return self._cache[cacheKey]
    end
    
    local response = self._http:get("/users/" .. id, {
        mockBody = string.format('{"id":%d,"name":"User %d"}', id, id)
    })
    
    if response.ok then
        self._cache[cacheKey] = response
    end
    
    return response
end

function APIClient:createUser(data)
    local response = self._http:post("/users", data, {
        mockBody = '{"id":42,"name":"' .. (data.name or "New User") .. '"}'
    })
    return response
end

-- Test async HTTP client
print("Async HTTP Client:")

local client = APIClient.new("https://api.example.com")
    :auth("my-api-token")

-- Simulate concurrent requests using coroutines
local results = {}

local requests = {
    coroutine.create(function()
        local r = client:getUsers()
        table.insert(results, "Users: " .. r.body:sub(1, 50))
    end),
    coroutine.create(function()
        local r = client:getUser(1)
        table.insert(results, "User 1: " .. r.body)
    end),
    coroutine.create(function()
        local r = client:getUser(1)  -- Should hit cache
        table.insert(results, "User 1 (cached): " .. r.body)
    end),
    coroutine.create(function()
        local r = client:createUser({ name = "Charlie" })
        table.insert(results, "Created: " .. r.body)
    end)
}

-- Run all requests
local pending = requests
while #pending > 0 do
    local next = {}
    for _, co in ipairs(pending) do
        if coroutine.status(co) == "suspended" then
            coroutine.resume(co)
        end
        if coroutine.status(co) == "suspended" then
            table.insert(next, co)
        end
    end
    pending = next
end

print("Results:")
for _, r in ipairs(results) do
    print("  " .. r)
end
```

---

## 17. Async Error Handling และ Recovery

```lua
-- Async error handling patterns
local AsyncResult = {}
AsyncResult.__index = AsyncResult

function AsyncResult.ok(value)
    return setmetatable({ _ok = true, _value = value }, AsyncResult)
end

function AsyncResult.err(error_)
    return setmetatable({ _ok = false, _error = error_ }, AsyncResult)
end

function AsyncResult:isOk()
    return self._ok
end

function AsyncResult:isErr()
    return not self._ok
end

function AsyncResult:unwrap()
    if not self._ok then
        error("Called unwrap on Err: " .. tostring(self._error))
    end
    return self._value
end

function AsyncResult:unwrapOr(default)
    return self._ok and self._value or default
end

function AsyncResult:map(fn)
    if self._ok then
        local ok, result = pcall(fn, self._value)
        if ok then
            return AsyncResult.ok(result)
        else
            return AsyncResult.err(result)
        end
    end
    return self
end

function AsyncResult:mapErr(fn)
    if not self._ok then
        local ok, result = pcall(fn, self._error)
        if ok then
            return AsyncResult.err(result)
        end
    end
    return self
end

function AsyncResult:andThen(fn)
    if self._ok then
        local ok, result = pcall(fn, self._value)
        if ok then
            if type(result) == "table" and result._ok ~= nil then
                return result  -- Already an AsyncResult
            end
            return AsyncResult.ok(result)
        else
            return AsyncResult.err(result)
        end
    end
    return self
end

function AsyncResult:orElse(fn)
    if not self._ok then
        return fn(self._error)
    end
    return self
end

-- Use AsyncResult in async operations
local function fetchData(id)
    if id <= 0 then
        return AsyncResult.err("Invalid ID: " .. id)
    end
    return AsyncResult.ok({ id = id, data = "Data for " .. id })
end

local function processData(record)
    if not record.data then
        return AsyncResult.err("No data in record")
    end
    return AsyncResult.ok(record.data:upper())
end

-- Chain operations
print("AsyncResult chain:")

local result1 = fetchData(5)
    :andThen(processData)
    :map(function(data)
        return "Processed: " .. data
    end)

if result1:isOk() then
    print("  Success:", result1:unwrap())
end

local result2 = fetchData(-1)
    :andThen(processData)
    :orElse(function(err)
        print("  Recovering from:", err)
        return AsyncResult.ok("default_value")
    end)
    :map(function(v)
        return v:upper()
    end)

print("  Result:", result2:unwrap())

-- Combining multiple results
local function allOk(results)
    local values = {}
    for _, r in ipairs(results) do
        if r:isErr() then
            return r  -- Return first error
        end
        table.insert(values, r:unwrap())
    end
    return AsyncResult.ok(values)
end

local combined = allOk({
    fetchData(1),
    fetchData(2),
    fetchData(3)
})

print("  Combined:", combined:isOk() and "all ok" or "some error")
if combined:isOk() then
    print("  Count:", #combined:unwrap())
end
```

---

## 18. Semaphore และ Mutex สำหรับ Coroutines

```lua
-- Semaphore - จำกัดจำนวน concurrent operations
local Semaphore = {}
Semaphore.__index = Semaphore

function Semaphore.new(limit)
    return setmetatable({
        _limit = limit or 1,
        _count = 0,
        _waiting = {}
    }, Semaphore)
end

function Semaphore:acquire()
    if self._count < self._limit then
        self._count = self._count + 1
        return true
    end
    
    -- Wait for permit
    table.insert(self._waiting, coroutine.running())
    coroutine.yield()
    return true
end

function Semaphore:release()
    self._count = self._count - 1
    
    -- Wake a waiting coroutine
    if #self._waiting > 0 then
        local co = table.remove(self._waiting, 1)
        self._count = self._count + 1
        coroutine.resume(co)
    end
end

-- Mutex (Semaphore with limit 1)
local function Mutex()
    return Semaphore.new(1)
end

-- Rate Limiter ด้วย Semaphore
local RateLimiter = {}
RateLimiter.__index = RateLimiter

function RateLimiter.new(maxConcurrent, maxPerSecond)
    return setmetatable({
        _semaphore = Semaphore.new(maxConcurrent),
        _maxPerSec = maxPerSecond,
        _requestCount = 0,
        _windowStart = os.clock()
    }, RateLimiter)
end

function RateLimiter:acquire()
    -- Check rate limit
    local now = os.clock()
    if now - self._windowStart >= 1.0 then
        self._windowStart = now
        self._requestCount = 0
    end
    
    if self._requestCount >= self._maxPerSec then
        return false, "Rate limit exceeded"
    end
    
    self._requestCount = self._requestCount + 1
    self._semaphore:acquire()
    return true
end

function RateLimiter:release()
    self._semaphore:release()
end

-- Test Semaphore
print("Semaphore demonstration:")

local sem = Semaphore.new(2)  -- Allow 2 concurrent
local results = {}

local function worker(id)
    return coroutine.create(function()
        local acquired = sem:acquire()
        if acquired then
            table.insert(results, "Worker " .. id .. " acquired")
            coroutine.yield()  -- Simulate work
            table.insert(results, "Worker " .. id .. " releasing")
            sem:release()
        end
    end)
end

-- Create 4 workers, only 2 can run at once
local workers = {}
for i = 1, 4 do
    table.insert(workers, worker(i))
end

-- Run workers
local running = workers
for _ = 1, 20 do
    local next = {}
    for _, co in ipairs(running) do
        if coroutine.status(co) == "suspended" then
            coroutine.resume(co)
        end
        if coroutine.status(co) == "suspended" then
            table.insert(next, co)
        end
    end
    running = next
    if #running == 0 then break end
end

for _, entry in ipairs(results) do
    print("  " .. entry)
end
```

---

## 19. Generator Functions

```lua
-- Generator Functions - คล้าย Python generators
local function generator(fn)
    return coroutine.wrap(fn)
end

-- Infinite generators
local function naturals(start)
    return generator(function()
        local n = start or 0
        while true do
            coroutine.yield(n)
            n = n + 1
        end
    end)
end

local function fibonacci()
    return generator(function()
        local a, b = 0, 1
        while true do
            coroutine.yield(a)
            a, b = b, a + b
        end
    end)
end

local function primes()
    return generator(function()
        local function isPrime(n)
            if n < 2 then return false end
            for i = 2, math.sqrt(n) do
                if n % i == 0 then return false end
            end
            return true
        end
        
        local n = 2
        while true do
            if isPrime(n) then
                coroutine.yield(n)
            end
            n = n + 1
        end
    end)
end

-- Finite generators
local function readLines(text)
    return generator(function()
        for line in (text .. "\n"):gmatch("([^\n]*)\n") do
            coroutine.yield(line)
        end
    end)
end

local function enumerate(iter)
    return generator(function()
        local i = 0
        for v in iter do
            i = i + 1
            coroutine.yield(i, v)
        end
    end)
end

-- Generator combinator
local function zip(...)
    local iters = { ... }
    return generator(function()
        while true do
            local values = {}
            for _, iter in ipairs(iters) do
                local v = iter()
                if v == nil then return end
                table.insert(values, v)
            end
            coroutine.yield(table.unpack(values))
        end
    end)
end

local function take(n, iter)
    return generator(function()
        local count = 0
        for v in iter do
            count = count + 1
            coroutine.yield(v)
            if count >= n then return end
        end
    end)
end

local function map(fn, iter)
    return generator(function()
        for v in iter do
            coroutine.yield(fn(v))
        end
    end)
end

local function filter(pred, iter)
    return generator(function()
        for v in iter do
            if pred(v) then
                coroutine.yield(v)
            end
        end
    end)
end

-- Test generators
print("Generator examples:")

print("\nFirst 10 naturals:")
local nat = take(10, naturals(1))
local natArr = {}
for n in nat do table.insert(natArr, n) end
print("  " .. table.concat(natArr, ", "))

print("\nFirst 10 Fibonacci:")
local fib = take(10, fibonacci())
local fibArr = {}
for n in fib do table.insert(fibArr, n) end
print("  " .. table.concat(fibArr, ", "))

print("\nFirst 10 Primes:")
local primesGen = take(10, primes())
local primesArr = {}
for n in primesGen do table.insert(primesArr, n) end
print("  " .. table.concat(primesArr, ", "))

print("\nEnumerate:")
local fruits = {"apple", "banana", "cherry"}
local fruitsIter = (function()
    local i = 0
    return function()
        i = i + 1
        return fruits[i]
    end
end)()

for i, v in enumerate(fruitsIter) do
    print(string.format("  %d: %s", i, v))
end

print("\nZip generators:")
local nums = take(5, naturals(1))
local sqrs = take(5, map(function(n) return n*n end, naturals(1)))

local zipped = zip(
    (function()
        local i, data = 0, {1,2,3,4,5}
        return function() i=i+1; return data[i] end
    end)(),
    (function()
        local i, data = 0, {1,4,9,16,25}
        return function() i=i+1; return data[i] end
    end)()
)

for n, sq in zipped do
    print(string.format("  %d^2 = %d", n, sq))
end
```

---

## 20. Complete Async Server Runtime

```lua
-- Complete simulation ของ async server runtime
local AsyncServer = {}
AsyncServer.__index = AsyncServer

function AsyncServer.new(options)
    options = options or {}
    
    return setmetatable({
        _handlers = {},
        _middleware = {},
        _connections = {},
        _requestCount = 0,
        _errorCount = 0,
        _startTime = nil,
        _config = {
            maxConnections = options.maxConnections or 100,
            timeout = options.timeout or 30,
            keepAlive = options.keepAlive ~= false
        }
    }, AsyncServer)
end

function AsyncServer:use(middleware)
    table.insert(self._middleware, middleware)
end

function AsyncServer:route(path, method, handler)
    local key = method:upper() .. ":" .. path
    self._handlers[key] = handler
end

function AsyncServer:get(path, handler)
    self:route(path, "GET", handler)
end

function AsyncServer:post(path, handler)
    self:route(path, "POST", handler)
end

function AsyncServer:handleRequest(request)
    self._requestCount = self._requestCount + 1
    
    local ctx = {
        request = request,
        response = {
            status = 200,
            headers = { ["Content-Type"] = "application/json" },
            body = ""
        },
        params = {},
        state = {}
    }
    
    -- Run middleware chain
    local mwIndex = 0
    
    local function next()
        mwIndex = mwIndex + 1
        local mw = self._middleware[mwIndex]
        if mw then
            local ok, err = pcall(mw, ctx, next)
            if not ok then
                ctx.response.status = 500
                ctx.response.body = '{"error":"' .. tostring(err) .. '"}'
                self._errorCount = self._errorCount + 1
            end
        else
            -- Route handler
            local key = request.method:upper() .. ":" .. request.path
            local handler = self._handlers[key]
            
            if handler then
                local ok, err = pcall(handler, ctx)
                if not ok then
                    ctx.response.status = 500
                    ctx.response.body = '{"error":"Handler error"}'
                    self._errorCount = self._errorCount + 1
                end
            else
                ctx.response.status = 404
                ctx.response.body = '{"error":"Not Found","path":"' .. request.path .. '"}'
            end
        end
    end
    
    next()
    return ctx.response
end

function AsyncServer:simulateLoad(requests)
    self._startTime = os.clock()
    local responses = {}
    
    -- Process requests as coroutines
    local coroutines = {}
    for _, req in ipairs(requests) do
        local co = coroutine.create(function()
            coroutine.yield()  -- Simulate IO wait
            return self:handleRequest(req)
        end)
        table.insert(coroutines, { co = co, req = req })
    end
    
    -- Run all coroutines
    while #coroutines > 0 do
        local remaining = {}
        for _, entry in ipairs(coroutines) do
            if coroutine.status(entry.co) == "suspended" then
                local ok, response = coroutine.resume(entry.co)
                if coroutine.status(entry.co) == "dead" then
                    if ok and response then
                        table.insert(responses, {
                            request = entry.req,
                            response = response
                        })
                    end
                else
                    table.insert(remaining, entry)
                end
            end
        end
        coroutines = remaining
    end
    
    local duration = os.clock() - self._startTime
    return responses, duration
end

function AsyncServer:stats()
    return {
        requests = self._requestCount,
        errors = self._errorCount,
        handlers = (function()
            local n = 0
            for _ in pairs(self._handlers) do n = n + 1 end
            return n
        end)()
    }
end

-- Build a complete async server
local server = AsyncServer.new({ maxConnections = 50, timeout = 10 })

-- Middleware
server:use(function(ctx, next)
    ctx.state.startTime = os.clock()
    next()
    local duration = os.clock() - ctx.state.startTime
    ctx.response.headers["X-Response-Time"] = string.format("%.3fms", duration * 1000)
end)

server:use(function(ctx, next)
    ctx.response.headers["X-Request-ID"] = tostring(math.random(100000, 999999))
    next()
end)

-- Routes
server:get("/health", function(ctx)
    ctx.response.body = '{"status":"ok","uptime":100}'
end)

server:get("/users", function(ctx)
    ctx.response.body = '[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]'
end)

server:post("/users", function(ctx)
    ctx.response.status = 201
    ctx.response.body = '{"id":3,"name":"New User"}'
end)

server:get("/error", function(ctx)
    error("Intentional error for testing")
end)

-- Simulate load
local testRequests = {
    { method = "GET", path = "/health", headers = {} },
    { method = "GET", path = "/users", headers = {} },
    { method = "POST", path = "/users", headers = {}, body = '{"name":"Charlie"}' },
    { method = "GET", path = "/notfound", headers = {} },
    { method = "GET", path = "/error", headers = {} },
    { method = "GET", path = "/users", headers = {} },
    { method = "GET", path = "/health", headers = {} },
}

print("Running async server simulation:")
local responses, duration = server:simulateLoad(testRequests)

for _, entry in ipairs(responses) do
    print(string.format("  %s %s -> %d",
        entry.request.method,
        entry.request.path,
        entry.response.status))
end

local stats = server:stats()
print(string.format("\nServer stats: %d requests, %d errors, %d routes",
    stats.requests, stats.errors, stats.handlers))
print(string.format("Simulated in %.4f seconds", duration))
```

---

## สรุป

บทนี้ครอบคลุม:

1. **Coroutine Basics** - recap สำหรับ async patterns
2. **Async/Await Simulation** - simulate async/await ด้วย coroutines
3. **Promise Implementation** - then/catch/finally/all/race
4. **Future Pattern** - Lazy evaluation
5. **Parallel Operations** - async.parallel, series, waterfall
6. **Error Propagation** - Error handling ใน async context
7. **Timeout** - จำกัดเวลาของ async operations
8. **Cancellation** - CancellationToken pattern
9. **Async Iterators** - Iterate over async data streams
10. **Reactive Streams** - Observable pattern
11. **Event Loop** - Simple event loop implementation
12. **Non-blocking I/O** - Buffer, streams, chunked processing
13. **Producer-Consumer** - Channel-based communication
14. **Async Pipeline** - Compose async operations
15. **Complete Runtime** - microtask/macrotask/coroutine management
16. **Async HTTP Client** - Simulated async HTTP
17. **AsyncResult** - Rust-style Result type for async
18. **Semaphore/Mutex** - Concurrency control
19. **Generator Functions** - Infinite sequences
20. **Async Server Runtime** - Complete server simulation

Coroutines ใน Lua ทรงพลังมากสำหรับ async patterns เพราะ:
- **Cooperative multitasking** - coroutines yield control explicitly
- **Low overhead** - เบากว่า threads มาก
- **Composable** - สามารถ compose ได้หลายรูปแบบ
- **No callback hell** - code ดูเหมือน synchronous แต่ทำงาน async

Lua เหมาะอย่างมากสำหรับ embedded scripting ใน async server frameworks เช่น OpenResty (Nginx + Lua)
