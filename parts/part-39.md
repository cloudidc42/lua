# บทที่ 39: Event System และ Pub/Sub (ระบบ Event และการส่งข้อความ)

## บทนำ

Event-driven programming เป็น paradigm ที่การทำงานของโปรแกรมถูกขับเคลื่อนด้วย events เช่น user actions, messages, sensor outputs ฯลฯ รูปแบบนี้ช่วยให้ components ต่างๆ ทำงานได้อย่างอิสระและ loosely coupled

## 39.1 EventEmitter Class พื้นฐาน

```lua
-- ตัวอย่างที่ 1: EventEmitter พื้นฐาน
print("=== EventEmitter พื้นฐาน ===")

local EventEmitter = {}
EventEmitter.__index = EventEmitter

function EventEmitter.new()
    return setmetatable({
        _listeners = {},
        _maxListeners = 10,
    }, EventEmitter)
end

-- on(): ลงทะเบียน event handler
function EventEmitter:on(event, listener)
    if not self._listeners[event] then
        self._listeners[event] = {}
    end
    
    -- ตรวจสอบ max listeners
    if #self._listeners[event] >= self._maxListeners then
        print(string.format("Warning: Possible EventEmitter memory leak. %d listeners added for '%s'",
            #self._listeners[event] + 1, event))
    end
    
    table.insert(self._listeners[event], {
        fn = listener,
        once = false,
    })
    
    return self  -- chainable
end

-- once(): ลงทะเบียน handler ที่ทำงานครั้งเดียวแล้วลบออก
function EventEmitter:once(event, listener)
    if not self._listeners[event] then
        self._listeners[event] = {}
    end
    table.insert(self._listeners[event], {
        fn = listener,
        once = true,
    })
    return self
end

-- off(): ยกเลิก event handler
function EventEmitter:off(event, listener)
    if not self._listeners[event] then return self end
    
    local newListeners = {}
    for _, entry in ipairs(self._listeners[event]) do
        if entry.fn ~= listener then
            table.insert(newListeners, entry)
        end
    end
    self._listeners[event] = newListeners
    return self
end

-- emit(): ส่ง event
function EventEmitter:emit(event, ...)
    if not self._listeners[event] then return false end
    
    local toKeep = {}
    for _, entry in ipairs(self._listeners[event]) do
        local ok, err = pcall(entry.fn, ...)
        if not ok then
            print(string.format("EventEmitter error in '%s' handler: %s", event, err))
        end
        if not entry.once then
            table.insert(toKeep, entry)
        end
    end
    self._listeners[event] = toKeep
    return true
end

-- removeAllListeners(): ลบ listeners ทั้งหมด
function EventEmitter:removeAllListeners(event)
    if event then
        self._listeners[event] = nil
    else
        self._listeners = {}
    end
    return self
end

-- listenerCount(): นับ listeners
function EventEmitter:listenerCount(event)
    if not self._listeners[event] then return 0 end
    return #self._listeners[event]
end

-- eventNames(): ดู events ทั้งหมด
function EventEmitter:eventNames()
    local names = {}
    for name, _ in pairs(self._listeners) do
        table.insert(names, name)
    end
    table.sort(names)
    return names
end

-- ทดสอบ
local emitter = EventEmitter.new()

emitter:on("data", function(data)
    print("Handler 1 received: " .. tostring(data))
end)

emitter:on("data", function(data)
    print("Handler 2 received: " .. tostring(data))
end)

emitter:once("connect", function()
    print("Connected! (this fires only once)")
end)

emitter:emit("connect")
emitter:emit("connect")  -- ไม่มีผล
emitter:emit("data", "Hello World")
emitter:emit("data", 42)

print("Listener count for 'data': " .. emitter:listenerCount("data"))
print("Event names: " .. table.concat(emitter:eventNames(), ", "))
```

```lua
-- ตัวอย่างที่ 2: EventEmitter inheritance
print("=== EventEmitter Inheritance ===")

-- สร้าง class ที่ inherit จาก EventEmitter
local function createClass(base)
    local cls = {}
    cls.__index = cls
    if base then
        setmetatable(cls, {__index = base})
    end
    function cls.new(...)
        local instance = setmetatable({}, cls)
        if base and base.new then
            -- copy EventEmitter state
            instance._listeners = {}
            instance._maxListeners = 10
        end
        if instance.initialize then
            instance:initialize(...)
        end
        return instance
    end
    return cls
end

local Server = createClass(EventEmitter)

function Server:initialize(host, port)
    self.host = host
    self.port = port
    self.connections = {}
    self.running = false
end

function Server:start()
    self.running = true
    self:emit("start", {host = self.host, port = self.port})
    print(string.format("[Server] Listening on %s:%d", self.host, self.port))
end

function Server:connect(clientId, address)
    self.connections[clientId] = {id = clientId, address = address, time = os.time()}
    self:emit("connection", self.connections[clientId])
end

function Server:disconnect(clientId)
    local conn = self.connections[clientId]
    if conn then
        self.connections[clientId] = nil
        self:emit("disconnect", conn)
    end
end

function Server:stop()
    self.running = false
    self:emit("close")
end

local server = Server.new("0.0.0.0", 8080)

server:on("start", function(info)
    print(string.format("[Event] Server started on %s:%d", info.host, info.port))
end)

server:on("connection", function(conn)
    print(string.format("[Event] New connection: %s from %s", conn.id, conn.address))
end)

server:on("disconnect", function(conn)
    print(string.format("[Event] Disconnected: %s", conn.id))
end)

server:on("close", function()
    print("[Event] Server closed")
end)

server:start()
server:connect("client-1", "192.168.1.100")
server:connect("client-2", "192.168.1.101")
server:disconnect("client-1")
server:stop()
```

## 39.2 Event Namespacing

```lua
-- ตัวอย่างที่ 3: Event namespacing ด้วย dot notation
print("=== Event Namespacing ===")

local NamespacedEmitter = {}
NamespacedEmitter.__index = NamespacedEmitter

function NamespacedEmitter.new()
    return setmetatable({
        _listeners = {},
        _wildcards = {},
    }, NamespacedEmitter)
end

function NamespacedEmitter:_parseEvent(event)
    local parts = {}
    for part in event:gmatch("[^.]+") do
        table.insert(parts, part)
    end
    return parts
end

function NamespacedEmitter:on(event, listener)
    if not self._listeners[event] then
        self._listeners[event] = {}
    end
    table.insert(self._listeners[event], listener)
    return self
end

function NamespacedEmitter:emit(event, ...)
    local parts = self:_parseEvent(event)
    local fired = {}
    
    -- ตรง exact match
    if self._listeners[event] then
        for _, fn in ipairs(self._listeners[event]) do
            if not fired[fn] then
                fn(...)
                fired[fn] = true
            end
        end
    end
    
    -- Wildcard match: "user.*", "*.created", "*"
    for pattern, listeners in pairs(self._listeners) do
        if pattern:find("*") and not fired[pattern] then
            local regexPattern = "^" .. pattern:gsub("%.", "%%."):gsub("%*", ".*") .. "$"
            if event:match(regexPattern) then
                for _, fn in ipairs(listeners) do
                    if not fired[fn] then
                        fn(event, ...)
                        fired[fn] = true
                    end
                end
            end
        end
    end
end

local nsEmitter = NamespacedEmitter.new()

-- Specific handlers
nsEmitter:on("user.created", function(user)
    print(string.format("[user.created] New user: %s", user.name))
end)

nsEmitter:on("user.deleted", function(user)
    print(string.format("[user.deleted] Removed user: %s", user.name))
end)

nsEmitter:on("order.created", function(order)
    print(string.format("[order.created] New order: #%s", order.id))
end)

-- Wildcard handlers
nsEmitter:on("user.*", function(event, data)
    print(string.format("[user.*] Caught: %s", event))
end)

nsEmitter:on("*.created", function(event, data)
    print(string.format("[*.created] Something was created: %s", event))
end)

nsEmitter:on("*", function(event, data)
    print(string.format("[*] Global catch: %s", event))
end)

print("Emitting user.created:")
nsEmitter:emit("user.created", {name = "Alice"})

print("\nEmitting order.created:")
nsEmitter:emit("order.created", {id = "ORD-001"})

print("\nEmitting user.deleted:")
nsEmitter:emit("user.deleted", {name = "Bob"})
```

## 39.3 Wildcard Events

```lua
-- ตัวอย่างที่ 4: Wildcard event system
print("=== Wildcard Events ===")

local WildcardEmitter = {}
WildcardEmitter.__index = WildcardEmitter

function WildcardEmitter.new()
    return setmetatable({
        _handlers = {},
    }, WildcardEmitter)
end

function WildcardEmitter:_matchPattern(pattern, event)
    if pattern == "*" then return true end
    -- แปลง glob pattern เป็น lua pattern
    local p = "^" .. pattern:gsub("%.", "\\."):gsub("%*", "[^%.]*"):gsub("%?", ".") .. "$"
    return event:match(p) ~= nil
end

function WildcardEmitter:on(pattern, handler)
    if not self._handlers[pattern] then
        self._handlers[pattern] = {}
    end
    table.insert(self._handlers[pattern], handler)
    return self
end

function WildcardEmitter:emit(event, ...)
    local results = {}
    for pattern, handlers in pairs(self._handlers) do
        if self:_matchPattern(pattern, event) then
            for _, handler in ipairs(handlers) do
                local ok, result = pcall(handler, event, ...)
                table.insert(results, {pattern = pattern, ok = ok, result = result})
            end
        end
    end
    return results
end

local we = WildcardEmitter.new()

we:on("app.start",    function(e) print("[exact] " .. e) end)
we:on("app.*",        function(e) print("[app.*] " .. e) end)
we:on("*.error",      function(e) print("[*.error] " .. e) end)
we:on("*",            function(e) print("[*] " .. e) end)

print("Emitting: app.start")
we:emit("app.start")
print("\nEmitting: app.error")
we:emit("app.error")
print("\nEmitting: db.error")
we:emit("db.error")
print("\nEmitting: random.event")
we:emit("random.event")
```

## 39.4 Priority Queue for Events

```lua
-- ตัวอย่างที่ 5: Priority Event Queue
print("=== Priority Event Queue ===")

local PriorityQueue = {}
PriorityQueue.__index = PriorityQueue

function PriorityQueue.new()
    return setmetatable({_items = {}}, PriorityQueue)
end

function PriorityQueue:push(item, priority)
    table.insert(self._items, {item = item, priority = priority})
    -- Sort by priority (higher = first)
    table.sort(self._items, function(a, b)
        return a.priority > b.priority
    end)
end

function PriorityQueue:pop()
    if #self._items == 0 then return nil end
    return table.remove(self._items, 1).item
end

function PriorityQueue:peek()
    if #self._items == 0 then return nil end
    return self._items[1].item
end

function PriorityQueue:size()
    return #self._items
end

function PriorityQueue:isEmpty()
    return #self._items == 0
end

-- Priority Event Emitter
local PriorityEmitter = {}
PriorityEmitter.__index = PriorityEmitter

PriorityEmitter.CRITICAL = 100
PriorityEmitter.HIGH     = 75
PriorityEmitter.NORMAL   = 50
PriorityEmitter.LOW      = 25

function PriorityEmitter.new()
    return setmetatable({
        _listeners = {},
        _queue = PriorityQueue.new(),
        _processing = false,
    }, PriorityEmitter)
end

function PriorityEmitter:on(event, handler, priority)
    priority = priority or PriorityEmitter.NORMAL
    if not self._listeners[event] then
        self._listeners[event] = {}
    end
    table.insert(self._listeners[event], {
        fn = handler,
        priority = priority
    })
    -- Sort by priority
    table.sort(self._listeners[event], function(a, b)
        return a.priority > b.priority
    end)
    return self
end

function PriorityEmitter:emit(event, data, priority)
    priority = priority or PriorityEmitter.NORMAL
    self._queue:push({event = event, data = data}, priority)
    
    if not self._processing then
        self:_processQueue()
    end
end

function PriorityEmitter:_processQueue()
    self._processing = true
    while not self._queue:isEmpty() do
        local entry = self._queue:pop()
        local handlers = self._listeners[entry.event]
        if handlers then
            for _, h in ipairs(handlers) do
                pcall(h.fn, entry.data)
            end
        end
    end
    self._processing = false
end

local pe = PriorityEmitter.new()

-- Handlers with different priorities
pe:on("alert", function(data) 
    print("[CRITICAL Handler] " .. data.message) 
end, PriorityEmitter.CRITICAL)

pe:on("alert", function(data) 
    print("[NORMAL Handler] " .. data.message) 
end, PriorityEmitter.NORMAL)

pe:on("alert", function(data) 
    print("[LOW Handler] " .. data.message) 
end, PriorityEmitter.LOW)

pe:on("alert", function(data) 
    print("[HIGH Handler] " .. data.message) 
end, PriorityEmitter.HIGH)

-- Handlers run in priority order (CRITICAL > HIGH > NORMAL > LOW)
pe:emit("alert", {message = "System overload!"})
```

## 39.5 Event Propagation (Bubbling)

```lua
-- ตัวอย่างที่ 6: Event Bubbling
print("=== Event Bubbling ===")

local DOMNode = {}
DOMNode.__index = DOMNode

function DOMNode.new(name, parent)
    return setmetatable({
        name = name,
        parent = parent,
        children = {},
        _handlers = {},
    }, DOMNode)
end

function DOMNode:addChild(child)
    child.parent = self
    table.insert(self.children, child)
    return child
end

function DOMNode:on(event, handler, options)
    options = options or {}
    if not self._handlers[event] then
        self._handlers[event] = {}
    end
    table.insert(self._handlers[event], {
        fn = handler,
        capture = options.capture or false,  -- capture phase vs bubble phase
        once = options.once or false,
    })
    return self
end

function DOMNode:_trigger(event, eventObj, phase)
    local handlers = self._handlers[event]
    if not handlers then return end
    
    local toKeep = {}
    for _, h in ipairs(handlers) do
        local isCapture = h.capture
        -- Capture phase: handlers with capture=true
        -- Bubble phase: handlers with capture=false
        if (phase == "capture" and isCapture) or 
           (phase == "bubble" and not isCapture) then
            if not eventObj.stopped then
                h.fn(eventObj)
            end
        end
        if not h.once then
            table.insert(toKeep, h)
        end
    end
    self._handlers[event] = toKeep
end

function DOMNode:dispatchEvent(eventName, data)
    local eventObj = {
        type = eventName,
        target = self,
        currentTarget = nil,
        stopped = false,
        data = data,
        
        stopPropagation = function(self)
            self.stopped = true
        end,
        
        stopImmediatePropagation = function(self)
            self.stopped = true
        end
    }
    
    -- Build path from root to target
    local path = {}
    local node = self
    while node do
        table.insert(path, 1, node)
        node = node.parent
    end
    
    -- Capture phase (root -> target)
    for _, n in ipairs(path) do
        if eventObj.stopped then break end
        eventObj.currentTarget = n
        n:_trigger(eventName, eventObj, "capture")
    end
    
    -- Bubble phase (target -> root)
    for i = #path, 1, -1 do
        if eventObj.stopped then break end
        local n = path[i]
        eventObj.currentTarget = n
        n:_trigger(eventName, eventObj, "bubble")
    end
end

-- Build DOM tree
local document = DOMNode.new("document", nil)
local body = document:addChild(DOMNode.new("body"))
local div = body:addChild(DOMNode.new("div#container"))
local button = div:addChild(DOMNode.new("button#submit"))

-- Add event handlers
document:on("click", function(e)
    print(string.format("[document] Caught click (bubble), target: %s", e.target.name))
end)

body:on("click", function(e)
    print(string.format("[body] Caught click (capture), target: %s", e.target.name))
end, {capture = true})

div:on("click", function(e)
    print(string.format("[div] Caught click (bubble), target: %s", e.target.name))
end)

button:on("click", function(e)
    print(string.format("[button] Click handler, stopping propagation"))
    -- e:stopPropagation()  -- uncomment to stop bubbling
end)

print("Clicking the button:")
button:dispatchEvent("click", {x = 10, y = 20})
```

## 39.6 Middleware for Events

```lua
-- ตัวอย่างที่ 7: Event Middleware
print("=== Event Middleware ===")

local MiddlewareEmitter = {}
MiddlewareEmitter.__index = MiddlewareEmitter

function MiddlewareEmitter.new()
    return setmetatable({
        _listeners = {},
        _middleware = {},
    }, MiddlewareEmitter)
end

function MiddlewareEmitter:use(middleware)
    table.insert(self._middleware, middleware)
    return self
end

function MiddlewareEmitter:on(event, handler)
    if not self._listeners[event] then
        self._listeners[event] = {}
    end
    table.insert(self._listeners[event], handler)
    return self
end

function MiddlewareEmitter:emit(event, data)
    -- Run through middleware chain
    local idx = 0
    local middleware = self._middleware
    local listeners = self._listeners[event] or {}
    
    local function next(modifiedData)
        idx = idx + 1
        if idx <= #middleware then
            middleware[idx](event, modifiedData or data, next)
        else
            -- All middleware passed, call listeners
            for _, handler in ipairs(listeners) do
                handler(modifiedData or data)
            end
        end
    end
    
    next(data)
end

local mw = MiddlewareEmitter.new()

-- Logging middleware
mw:use(function(event, data, next)
    print(string.format("[Middleware:Log] Event: %s", event))
    next(data)
end)

-- Authentication middleware
mw:use(function(event, data, next)
    if event:match("^admin%.") and not data.isAdmin then
        print(string.format("[Middleware:Auth] BLOCKED admin event: %s", event))
        return  -- don't call next
    end
    next(data)
end)

-- Transform middleware
mw:use(function(event, data, next)
    if type(data) == "table" then
        data.processedAt = os.time()
        data.eventName = event
    end
    next(data)
end)

-- Rate limiting middleware
local _rateLimits = {}
mw:use(function(event, data, next)
    local now = os.time()
    local key = event
    if not _rateLimits[key] then _rateLimits[key] = {count = 0, resetAt = now + 60} end
    
    if now > _rateLimits[key].resetAt then
        _rateLimits[key] = {count = 0, resetAt = now + 60}
    end
    
    _rateLimits[key].count = _rateLimits[key].count + 1
    if _rateLimits[key].count > 5 then
        print(string.format("[Middleware:RateLimit] Rate limited: %s", event))
        return
    end
    
    next(data)
end)

-- Handlers
mw:on("user.action", function(data)
    print(string.format("[Handler] user.action: %s", data.action))
end)

mw:on("admin.delete", function(data)
    print(string.format("[Handler] admin.delete: %s", data.resource))
end)

mw:emit("user.action", {action = "view_profile"})
mw:emit("user.action", {action = "edit_profile"})
mw:emit("admin.delete", {resource = "user:123", isAdmin = true})
mw:emit("admin.delete", {resource = "post:456", isAdmin = false})  -- blocked
```

## 39.7 Event Store / History

```lua
-- ตัวอย่างที่ 8: Event Store
print("=== Event Store ===")

local EventStore = {}
EventStore.__index = EventStore

function EventStore.new(options)
    options = options or {}
    return setmetatable({
        _events = {},
        _maxSize = options.maxSize or 1000,
        _handlers = {},
        _snapshots = {},
        _version = 0,
    }, EventStore)
end

function EventStore:append(eventType, data, aggregateId)
    self._version = self._version + 1
    local event = {
        id = self._version,
        type = eventType,
        aggregateId = aggregateId,
        data = data,
        timestamp = os.time(),
        version = self._version,
    }
    
    table.insert(self._events, event)
    
    -- Trim if over max size
    while #self._events > self._maxSize do
        table.remove(self._events, 1)
    end
    
    -- Notify handlers
    if self._handlers[eventType] then
        for _, handler in ipairs(self._handlers[eventType]) do
            pcall(handler, event)
        end
    end
    
    return event
end

function EventStore:on(eventType, handler)
    if not self._handlers[eventType] then
        self._handlers[eventType] = {}
    end
    table.insert(self._handlers[eventType], handler)
end

function EventStore:getAll()
    return self._events
end

function EventStore:getByAggregate(aggregateId)
    local result = {}
    for _, event in ipairs(self._events) do
        if event.aggregateId == aggregateId then
            table.insert(result, event)
        end
    end
    return result
end

function EventStore:getByType(eventType)
    local result = {}
    for _, event in ipairs(self._events) do
        if event.type == eventType then
            table.insert(result, event)
        end
    end
    return result
end

function EventStore:replay(aggregateId, handler)
    local events = self:getByAggregate(aggregateId)
    table.sort(events, function(a, b) return a.version < b.version end)
    for _, event in ipairs(events) do
        handler(event)
    end
end

function EventStore:snapshot(aggregateId, state)
    self._snapshots[aggregateId] = {
        state = state,
        version = self._version,
        timestamp = os.time(),
    }
end

function EventStore:getSnapshot(aggregateId)
    return self._snapshots[aggregateId]
end

-- Event Sourced Bank Account
local BankAccount = {}
BankAccount.__index = BankAccount

function BankAccount.new(id, store)
    local account = setmetatable({
        id = id,
        balance = 0,
        owner = nil,
        transactions = {},
        store = store,
    }, BankAccount)
    return account
end

function BankAccount:apply(event)
    if event.type == "AccountCreated" then
        self.owner = event.data.owner
        self.balance = 0
    elseif event.type == "MoneyDeposited" then
        self.balance = self.balance + event.data.amount
        table.insert(self.transactions, {type = "deposit", amount = event.data.amount})
    elseif event.type == "MoneyWithdrawn" then
        self.balance = self.balance - event.data.amount
        table.insert(self.transactions, {type = "withdrawal", amount = event.data.amount})
    end
end

function BankAccount:create(owner)
    local event = self.store:append("AccountCreated", {owner = owner}, self.id)
    self:apply(event)
end

function BankAccount:deposit(amount)
    if amount <= 0 then error("Amount must be positive") end
    local event = self.store:append("MoneyDeposited", {amount = amount}, self.id)
    self:apply(event)
end

function BankAccount:withdraw(amount)
    if amount <= 0 then error("Amount must be positive") end
    if amount > self.balance then error("Insufficient funds") end
    local event = self.store:append("MoneyWithdrawn", {amount = amount}, self.id)
    self:apply(event)
end

function BankAccount:getBalance()
    return self.balance
end

local store = EventStore.new()

-- Subscribe to events
store:on("MoneyDeposited", function(e)
    print(string.format("[Notification] Deposit of %.2f to account %s", 
        e.data.amount, e.aggregateId))
end)

local account = BankAccount.new("ACC-001", store)
account:create("Alice Smith")
account:deposit(1000)
account:deposit(500)
account:withdraw(200)
account:deposit(750)

print(string.format("\nFinal balance: %.2f", account:getBalance()))

print("\nAccount history:")
for _, event in ipairs(store:getByAggregate("ACC-001")) do
    print(string.format("  [v%d] %s: %s", 
        event.version, event.type, 
        event.data.amount and tostring(event.data.amount) or event.data.owner))
end

-- Reconstruct from events
print("\nReconstructing account state from events:")
local rebulit = BankAccount.new("ACC-001", store)
store:replay("ACC-001", function(event)
    rebulit:apply(event)
end)
print(string.format("Reconstructed balance: %.2f", rebulit:getBalance()))
```

## 39.8 Async Events (Simulated)

```lua
-- ตัวอย่างที่ 9: Async Event Queue
print("=== Async Event Queue ===")

local AsyncEmitter = {}
AsyncEmitter.__index = AsyncEmitter

function AsyncEmitter.new()
    return setmetatable({
        _listeners = {},
        _queue = {},
        _running = false,
        _tickCount = 0,
    }, AsyncEmitter)
end

function AsyncEmitter:on(event, handler, async)
    if not self._listeners[event] then
        self._listeners[event] = {}
    end
    table.insert(self._listeners[event], {fn = handler, async = async or false})
    return self
end

function AsyncEmitter:emit(event, data)
    -- Add to queue instead of immediately calling
    table.insert(self._queue, {event = event, data = data})
end

function AsyncEmitter:emitSync(event, data)
    -- Immediate execution
    local handlers = self._listeners[event]
    if handlers then
        for _, h in ipairs(handlers) do
            pcall(h.fn, data)
        end
    end
end

function AsyncEmitter:tick()
    self._tickCount = self._tickCount + 1
    if #self._queue == 0 then return 0 end
    
    -- Process all queued events in this tick
    local processed = 0
    local toProcess = self._queue
    self._queue = {}
    
    for _, entry in ipairs(toProcess) do
        local handlers = self._listeners[entry.event]
        if handlers then
            for _, h in ipairs(handlers) do
                pcall(h.fn, entry.data)
                processed = processed + 1
            end
        end
    end
    
    return processed
end

function AsyncEmitter:processAll()
    local total = 0
    local iterations = 0
    repeat
        local count = self:tick()
        total = total + count
        iterations = iterations + 1
        if iterations > 100 then break end  -- safety limit
    until #self._queue == 0
    return total
end

local ae = AsyncEmitter.new()

ae:on("task.start", function(data)
    print(string.format("[Tick %d] Task started: %s", ae._tickCount, data.name))
    -- Queue a follow-up event
    ae:emit("task.progress", {name = data.name, percent = 50})
end)

ae:on("task.progress", function(data)
    print(string.format("[Tick %d] Task progress: %s - %d%%", 
        ae._tickCount, data.name, data.percent))
    if data.percent < 100 then
        ae:emit("task.progress", {name = data.name, percent = data.percent + 50})
    else
        ae:emit("task.complete", {name = data.name})
    end
end)

ae:on("task.complete", function(data)
    print(string.format("[Tick %d] Task complete: %s", ae._tickCount, data.name))
end)

-- Queue initial events
ae:emit("task.start", {name = "Data Import"})
ae:emit("task.start", {name = "Report Generation"})

-- Process event loop
print("Processing event loop:")
ae:processAll()
```

## 39.9 Pub/Sub Pattern

```lua
-- ตัวอย่างที่ 10: Pub/Sub System
print("=== Pub/Sub Pattern ===")

local PubSub = {}
PubSub.__index = PubSub

function PubSub.new()
    return setmetatable({
        _subscriptions = {},
        _messageCount = 0,
        _topics = {},
    }, PubSub)
end

function PubSub:subscribe(topic, subscriber, options)
    options = options or {}
    if not self._subscriptions[topic] then
        self._subscriptions[topic] = {}
        self._topics[topic] = {
            createdAt = os.time(),
            messageCount = 0,
        }
    end
    
    local sub = {
        id = string.format("sub_%d_%d", os.time(), math.random(1000)),
        topic = topic,
        fn = subscriber,
        filter = options.filter,
        transform = options.transform,
        active = true,
    }
    
    table.insert(self._subscriptions[topic], sub)
    print(string.format("[PubSub] Subscribed to '%s' (id: %s)", topic, sub.id))
    return sub.id
end

function PubSub:unsubscribe(subscriptionId)
    for topic, subs in pairs(self._subscriptions) do
        for i, sub in ipairs(subs) do
            if sub.id == subscriptionId then
                sub.active = false
                table.remove(subs, i)
                print(string.format("[PubSub] Unsubscribed: %s", subscriptionId))
                return true
            end
        end
    end
    return false
end

function PubSub:publish(topic, message)
    self._messageCount = self._messageCount + 1
    
    if self._topics[topic] then
        self._topics[topic].messageCount = self._topics[topic].messageCount + 1
    end
    
    local envelope = {
        id = string.format("msg_%d", self._messageCount),
        topic = topic,
        data = message,
        timestamp = os.time(),
        messageNumber = self._messageCount,
    }
    
    local delivered = 0
    local subs = self._subscriptions[topic]
    if subs then
        for _, sub in ipairs(subs) do
            if sub.active then
                -- Apply filter
                if sub.filter and not sub.filter(message) then
                    goto continue
                end
                
                -- Apply transform
                local data = message
                if sub.transform then
                    data = sub.transform(message)
                end
                
                local ok, err = pcall(sub.fn, data, envelope)
                if not ok then
                    print(string.format("[PubSub] Error in subscriber: %s", err))
                else
                    delivered = delivered + 1
                end
            end
            ::continue::
        end
    end
    
    return delivered
end

function PubSub:topicInfo()
    local info = {}
    for topic, meta in pairs(self._topics) do
        local subCount = self._subscriptions[topic] and #self._subscriptions[topic] or 0
        table.insert(info, string.format("  %s: %d subs, %d messages",
            topic, subCount, meta.messageCount))
    end
    table.sort(info)
    return info
end

-- ทดสอบ
local pubsub = PubSub.new()

-- Subscribe to news
local sub1 = pubsub:subscribe("news", function(msg, env)
    print(string.format("[Sub1] News received (#%d): %s", env.messageNumber, msg.headline))
end)

-- Subscribe with filter (only urgent news)
local sub2 = pubsub:subscribe("news", function(msg, env)
    print(string.format("[Sub2-Urgent] BREAKING: %s", msg.headline))
end, {
    filter = function(msg) return msg.urgent == true end
})

-- Subscribe with transform
local sub3 = pubsub:subscribe("news", function(msg, env)
    print(string.format("[Sub3] [%s] %s", msg.category, msg.headline))
end, {
    transform = function(msg)
        return {headline = msg.headline:upper(), category = msg.category}
    end
})

-- Publish messages
pubsub:publish("news", {headline = "Lua 5.5 announced", category = "tech", urgent = false})
pubsub:publish("news", {headline = "Major earthquake hits coast", category = "world", urgent = true})
pubsub:publish("news", {headline = "Stock market hits record", category = "finance", urgent = false})

print("\nTopic info:")
for _, info in ipairs(pubsub:topicInfo()) do
    print(info)
end
```

## 39.10 Reactive Streams Basics

```lua
-- ตัวอย่างที่ 11: Observable Pattern (RxLua concept)
print("=== Observable Pattern ===")

local Observable = {}
Observable.__index = Observable

function Observable.new(subscribe_fn)
    return setmetatable({_subscribe = subscribe_fn}, Observable)
end

-- สร้าง Observable จาก values
function Observable.from(values)
    return Observable.new(function(observer)
        for _, value in ipairs(values) do
            if observer.completed then break end
            observer:next(value)
        end
        observer:complete()
    end)
end

-- สร้าง Observable จาก range
function Observable.range(start, stop, step)
    step = step or 1
    return Observable.new(function(observer)
        local i = start
        while (step > 0 and i <= stop) or (step < 0 and i >= stop) do
            if observer.completed then break end
            observer:next(i)
            i = i + step
        end
        observer:complete()
    end)
end

-- สร้าง Observable ที่ emit error
function Observable.throw(err)
    return Observable.new(function(observer)
        observer:error(err)
    end)
end

-- Operators
function Observable:map(fn)
    local source = self
    return Observable.new(function(observer)
        source:subscribe({
            next = function(_, value) observer:next(fn(value)) end,
            error = function(_, err) observer:error(err) end,
            complete = function(_) observer:complete() end,
        })
    end)
end

function Observable:filter(predicate)
    local source = self
    return Observable.new(function(observer)
        source:subscribe({
            next = function(_, value)
                if predicate(value) then
                    observer:next(value)
                end
            end,
            error = function(_, err) observer:error(err) end,
            complete = function(_) observer:complete() end,
        })
    end)
end

function Observable:take(n)
    local source = self
    return Observable.new(function(observer)
        local count = 0
        source:subscribe({
            next = function(_, value)
                if count < n then
                    count = count + 1
                    observer:next(value)
                    if count == n then
                        observer:complete()
                    end
                end
            end,
            error = function(_, err) observer:error(err) end,
            complete = function(_) observer:complete() end,
        })
    end)
end

function Observable:reduce(fn, initial)
    local source = self
    return Observable.new(function(observer)
        local acc = initial
        local hasValue = initial ~= nil
        source:subscribe({
            next = function(_, value)
                if hasValue then
                    acc = fn(acc, value)
                else
                    acc = value
                    hasValue = true
                end
            end,
            error = function(_, err) observer:error(err) end,
            complete = function(_)
                if hasValue then observer:next(acc) end
                observer:complete()
            end,
        })
    end)
end

function Observable:toArray()
    local results = {}
    local completed = false
    self:subscribe({
        next = function(_, value) table.insert(results, value) end,
        error = function(_, err) error(err) end,
        complete = function(_) completed = true end,
    })
    return results
end

function Observable:subscribe(observer)
    if type(observer) == "function" then
        observer = {
            next = function(_, value) observer(value) end,
            error = function(_, err) error(err) end,
            complete = function(_) end,
        }
    end
    
    local obs = setmetatable({
        completed = false,
        _observer = observer,
        
        next = function(self, value)
            if not self.completed then
                pcall(self._observer.next, self._observer, value)
            end
        end,
        
        error = function(self, err)
            if not self.completed then
                self.completed = true
                pcall(self._observer.error, self._observer, err)
            end
        end,
        
        complete = function(self)
            if not self.completed then
                self.completed = true
                pcall(self._observer.complete, self._observer)
            end
        end,
    }, {__index = observer})
    
    self._subscribe(obs)
    return obs
end

-- ทดสอบ
print("Range 1-10, filter even, map *2, take 3:")
local result = Observable.range(1, 10)
    :filter(function(n) return n % 2 == 0 end)
    :map(function(n) return n * 2 end)
    :take(3)
    :toArray()
print("  Result: " .. table.concat(result, ", "))

print("\nFrom array, map to string, reduce to concat:")
local result2 = Observable.from({1, 2, 3, 4, 5})
    :map(function(n) return "[" .. n .. "]" end)
    :reduce(function(acc, s) return acc .. s end, "")
    :toArray()
print("  Result: " .. table.concat(result2, ""))

print("\nSum of squares from 1 to 5:")
local result3 = Observable.range(1, 5)
    :map(function(n) return n * n end)
    :reduce(function(acc, n) return acc + n end, 0)
    :toArray()
print("  Sum: " .. result3[1])
```

## 39.11 Real Example: Game Event System

```lua
-- ตัวอย่างที่ 12: Game Event System
print("=== Game Event System ===")

local GameEvents = {}
GameEvents.__index = GameEvents

-- Event types
GameEvents.PLAYER_MOVE     = "player.move"
GameEvents.PLAYER_ATTACK   = "player.attack"
GameEvents.PLAYER_DIED     = "player.died"
GameEvents.ENEMY_SPAWNED   = "enemy.spawned"
GameEvents.ENEMY_KILLED    = "enemy.killed"
GameEvents.ITEM_COLLECTED  = "item.collected"
GameEvents.LEVEL_COMPLETE  = "level.complete"
GameEvents.SCORE_CHANGED   = "score.changed"

function GameEvents.new()
    local ge = setmetatable({
        _handlers = {},
        _globalHandlers = {},
        _paused = false,
        _deferred = {},
        stats = {totalEvents = 0, byType = {}},
    }, GameEvents)
    return ge
end

function GameEvents:on(eventType, handler, priority)
    priority = priority or 0
    if not self._handlers[eventType] then
        self._handlers[eventType] = {}
    end
    table.insert(self._handlers[eventType], {fn = handler, priority = priority})
    table.sort(self._handlers[eventType], function(a, b)
        return a.priority > b.priority
    end)
    return self
end

function GameEvents:off(eventType, handler)
    if not self._handlers[eventType] then return end
    local new = {}
    for _, h in ipairs(self._handlers[eventType]) do
        if h.fn ~= handler then table.insert(new, h) end
    end
    self._handlers[eventType] = new
end

function GameEvents:emit(eventType, data)
    self.stats.totalEvents = self.stats.totalEvents + 1
    self.stats.byType[eventType] = (self.stats.byType[eventType] or 0) + 1
    
    if self._paused then
        table.insert(self._deferred, {type = eventType, data = data})
        return
    end
    
    local handlers = self._handlers[eventType]
    if not handlers then return end
    
    for _, h in ipairs(handlers) do
        local ok, err = pcall(h.fn, data)
        if not ok then
            print(string.format("[GameEvents] Error: %s", err))
        end
    end
end

function GameEvents:pause()
    self._paused = true
end

function GameEvents:resume()
    self._paused = false
    local deferred = self._deferred
    self._deferred = {}
    for _, e in ipairs(deferred) do
        self:emit(e.type, e.data)
    end
end

function GameEvents:getStats()
    return self.stats
end

-- Game components listening to events

-- Score system
local ScoreSystem = {}
function ScoreSystem.setup(events)
    local score = 0
    
    events:on(GameEvents.ENEMY_KILLED, function(data)
        local points = data.enemy.points or 10
        score = score + points
        events:emit(GameEvents.SCORE_CHANGED, {score = score, delta = points})
    end)
    
    events:on(GameEvents.ITEM_COLLECTED, function(data)
        local points = data.item.bonus or 0
        if points > 0 then
            score = score + points
            events:emit(GameEvents.SCORE_CHANGED, {score = score, delta = points})
        end
    end)
    
    events:on(GameEvents.SCORE_CHANGED, function(data)
        print(string.format("[Score] Score: %d (+%d)", data.score, data.delta))
    end)
    
    return {getScore = function() return score end}
end

-- UI system
local UISystem = {}
function UISystem.setup(events)
    events:on(GameEvents.PLAYER_DIED, function(data)
        print("[UI] Game Over screen shown!")
    end)
    
    events:on(GameEvents.LEVEL_COMPLETE, function(data)
        print(string.format("[UI] Level %d Complete! Showing results...", data.level))
    end)
    
    events:on(GameEvents.ITEM_COLLECTED, function(data)
        print(string.format("[UI] Collected: %s", data.item.name))
    end)
end

-- Audio system
local AudioSystem = {}
function AudioSystem.setup(events)
    events:on(GameEvents.PLAYER_ATTACK, function(data)
        print(string.format("[Audio] Playing attack sound: %s", data.weapon))
    end)
    
    events:on(GameEvents.ENEMY_KILLED, function(data)
        print("[Audio] Playing death sound")
    end)
    
    events:on(GameEvents.ITEM_COLLECTED, function(data)
        print("[Audio] Playing pickup sound")
    end)
end

-- Setup game
local gameEvents = GameEvents.new()
local scoreSystem = ScoreSystem.setup(gameEvents)
UISystem.setup(gameEvents)
AudioSystem.setup(gameEvents)

-- Simulate game events
print("-- Game simulation --")
gameEvents:emit(GameEvents.PLAYER_ATTACK, {weapon = "sword", damage = 25})
gameEvents:emit(GameEvents.ENEMY_KILLED, {enemy = {name = "Goblin", points = 50}})
gameEvents:emit(GameEvents.ITEM_COLLECTED, {item = {name = "Gold Coin", bonus = 10}})
gameEvents:emit(GameEvents.ITEM_COLLECTED, {item = {name = "Health Potion", bonus = 0}})
gameEvents:emit(GameEvents.ENEMY_KILLED, {enemy = {name = "Dragon", points = 500}})
gameEvents:emit(GameEvents.LEVEL_COMPLETE, {level = 1})

print(string.format("\nFinal score: %d", scoreSystem.getScore()))
local stats = gameEvents:getStats()
print(string.format("Total events: %d", stats.totalEvents))
```

## 39.12 Network Events

```lua
-- ตัวอย่างที่ 13: Network Event Simulation
print("=== Network Events ===")

local NetworkEmitter = {}
NetworkEmitter.__index = NetworkEmitter

function NetworkEmitter.new()
    local ne = setmetatable({
        _handlers = {},
        _connected = false,
        _reconnectAttempts = 0,
        _maxReconnectAttempts = 3,
        _reconnectDelay = 1,
        _messageQueue = {},
    }, NetworkEmitter)
    
    -- Mixin EventEmitter methods
    ne.on = function(self, event, handler)
        if not self._handlers[event] then self._handlers[event] = {} end
        table.insert(self._handlers[event], handler)
        return self
    end
    
    ne.emit = function(self, event, ...)
        if self._handlers[event] then
            for _, h in ipairs(self._handlers[event]) do
                pcall(h, ...)
            end
        end
    end
    
    return ne
end

function NetworkEmitter:connect(host, port)
    print(string.format("[Net] Connecting to %s:%d...", host, port))
    -- Simulate connection
    self._connected = true
    self._reconnectAttempts = 0
    self:emit("connect", {host = host, port = port})
    return true
end

function NetworkEmitter:disconnect()
    self._connected = false
    self:emit("disconnect", {reason = "manual"})
end

function NetworkEmitter:send(data)
    if not self._connected then
        table.insert(self._messageQueue, data)
        print("[Net] Not connected, queuing message")
        return false
    end
    self:emit("send", data)
    print(string.format("[Net] Sent: %s", tostring(data)))
    return true
end

function NetworkEmitter:simulateReceive(data)
    self:emit("data", data)
end

function NetworkEmitter:simulateError(errMsg)
    self._connected = false
    self:emit("error", errMsg)
    
    -- Auto-reconnect logic
    if self._reconnectAttempts < self._maxReconnectAttempts then
        self._reconnectAttempts = self._reconnectAttempts + 1
        print(string.format("[Net] Reconnecting... attempt %d/%d",
            self._reconnectAttempts, self._maxReconnectAttempts))
        self:emit("reconnecting", {attempt = self._reconnectAttempts})
        -- Simulate successful reconnect
        self._connected = true
        self:emit("reconnect", {attempt = self._reconnectAttempts})
        
        -- Flush queued messages
        if #self._messageQueue > 0 then
            print(string.format("[Net] Flushing %d queued messages", #self._messageQueue))
            local queue = self._messageQueue
            self._messageQueue = {}
            for _, msg in ipairs(queue) do
                self:send(msg)
            end
        end
    else
        self:emit("maxReconnectAttemptsReached")
    end
end

local net = NetworkEmitter.new()

net:on("connect",    function(info) print(string.format("[Event] Connected to %s:%d", info.host, info.port)) end)
net:on("disconnect", function(info) print("[Event] Disconnected: " .. info.reason) end)
net:on("data",       function(data) print(string.format("[Event] Received: %s", tostring(data))) end)
net:on("error",      function(err)  print(string.format("[Event] Error: %s", err)) end)
net:on("reconnecting", function(info) print(string.format("[Event] Reconnecting... (#%d)", info.attempt)) end)
net:on("reconnect",  function(info) print(string.format("[Event] Reconnected! (#%d)", info.attempt)) end)

net:connect("api.example.com", 443)
net:send({action = "login", user = "alice"})
net:simulateReceive({status = "ok", token = "abc123"})
net:simulateError("Connection timed out")
net:send({action = "getData"})
```

## 39.13 UI Events

```lua
-- ตัวอย่างที่ 14: UI Component Event System
print("=== UI Event System ===")

local UIComponent = {}
UIComponent.__index = UIComponent

function UIComponent.new(id, type)
    return setmetatable({
        id = id,
        type = type,
        children = {},
        parent = nil,
        _handlers = {},
        visible = true,
        enabled = true,
        props = {},
    }, UIComponent)
end

function UIComponent:on(event, handler)
    if not self._handlers[event] then
        self._handlers[event] = {}
    end
    table.insert(self._handlers[event], handler)
    return self
end

function UIComponent:trigger(event, data)
    data = data or {}
    data.target = self
    data.type = event
    
    -- Call own handlers
    if self._handlers[event] then
        for _, h in ipairs(self._handlers[event]) do
            local result = h(data)
            if result == false then
                return false  -- stop propagation
            end
        end
    end
    
    -- Bubble to parent
    if self.parent then
        return self.parent:trigger(event, data)
    end
    
    return true
end

function UIComponent:addChild(child)
    child.parent = self
    table.insert(self.children, child)
    return child
end

function UIComponent:click()
    if not self.enabled then return end
    self:trigger("click")
end

function UIComponent:change(value)
    if not self.enabled then return end
    self:trigger("change", {value = value, oldValue = self.value})
    self.value = value
end

function UIComponent:focus()
    self:trigger("focus")
end

function UIComponent:blur()
    self:trigger("blur")
end

-- Build a form
local form = UIComponent.new("form1", "form")
local nameField = form:addChild(UIComponent.new("nameField", "input"))
local emailField = form:addChild(UIComponent.new("emailField", "input"))
local submitBtn = form:addChild(UIComponent.new("submitBtn", "button"))

-- Form-level event capture
form:on("click", function(e)
    print(string.format("[Form] Click event bubbled from: %s", e.target.id))
end)

form:on("change", function(e)
    print(string.format("[Form] Field changed: %s = %s", e.target.id, tostring(e.value)))
end)

-- Button handler
submitBtn:on("click", function(e)
    print("[Button] Submit clicked!")
    -- Return false to prevent bubbling
    -- return false
end)

-- Field handlers
nameField:on("change", function(e)
    print(string.format("[NameField] Value changed to: %s", tostring(e.value)))
    -- Validate
    if type(e.value) == "string" and #e.value < 2 then
        print("[NameField] Validation error: too short")
    end
end)

nameField:on("focus", function(e)
    print("[NameField] Focused")
end)

-- Simulate interactions
nameField:focus()
nameField:change("Alice")
nameField:change("A")  -- triggers validation
emailField:change("alice@example.com")
submitBtn:click()
```

## 39.14 Event Debouncing and Throttling

```lua
-- ตัวอย่างที่ 15: Debounce and Throttle
print("=== Debounce and Throttle ===")

-- Debounce: รอจน event หยุดสักพักก่อน execute
function debounce(fn, delay)
    local timer = nil
    local callCount = 0
    
    return function(...)
        callCount = callCount + 1
        local args = {...}
        local thisCall = callCount
        
        -- Simulate delay by checking if we're the last call
        -- In real Lua with coroutines, this would use actual timers
        local function execute()
            if thisCall == callCount then
                fn(table.unpack(args))
            end
        end
        
        -- In a real system, we'd cancel the previous timer
        -- For simulation, just call immediately on "last" invocation
        timer = execute
        return timer
    end
end

-- Throttle: execute ไม่เกิน 1 ครั้งต่อ interval
function throttle(fn, interval)
    local lastCall = 0
    local callCount = 0
    
    return function(...)
        callCount = callCount + 1
        -- Simulate time by using call count as proxy
        if callCount % interval == 1 or callCount == 1 then
            fn(...)
            lastCall = callCount
        else
            print(string.format("[Throttle] Skipped call %d", callCount))
        end
    end
end

-- ตัวอย่าง search autocomplete (debounce)
local searchHandler = function(query)
    print(string.format("[Search] API call for: '%s'", query))
end

local debouncedSearch = debounce(searchHandler, 300)

print("Simulating fast typing:")
local searchCalls = {"h", "he", "hel", "hell", "hello"}
for _, query in ipairs(searchCalls) do
    local fn = debouncedSearch(query)
    print(string.format("  Typed: '%s'", query))
end
-- Only the last timer would fire in real implementation
print("  (In real usage, only 'hello' search fires after delay)")

-- ตัวอย่าง scroll handler (throttle)
print("\nSimulating scroll events (throttle every 3 calls):")
local scrollHandler = throttle(function(pos)
    print(string.format("  [Scroll] Loading data at position: %d", pos))
end, 3)

for i = 1, 9 do
    scrollHandler(i * 100)
end
```

## 39.15 EventEmitter สมบูรณ์พร้อม Metrics

```lua
-- ตัวอย่างที่ 16: Production-ready EventEmitter
print("=== Production EventEmitter ===")

local ProductionEmitter = {}
ProductionEmitter.__index = ProductionEmitter

function ProductionEmitter.new(options)
    options = options or {}
    return setmetatable({
        _listeners = {},
        _maxListeners = options.maxListeners or 20,
        _captureRejections = options.captureRejections or true,
        _metrics = {
            emits = {},
            errors = 0,
            listenerAdds = 0,
            listenerRemoves = 0,
        },
        _errorHandlers = {},
    }, ProductionEmitter)
end

function ProductionEmitter:on(event, fn, options)
    options = options or {}
    if not self._listeners[event] then
        self._listeners[event] = {}
    end
    
    local count = #self._listeners[event]
    if count >= self._maxListeners then
        print(string.format("MaxListenersExceededWarning: %d listeners for '%s'", count + 1, event))
    end
    
    local listener = {
        fn = fn,
        once = options.once or false,
        priority = options.priority or 0,
        addedAt = os.time(),
        callCount = 0,
    }
    
    if options.prepend then
        table.insert(self._listeners[event], 1, listener)
    else
        table.insert(self._listeners[event], listener)
    end
    
    -- Sort by priority if set
    if options.priority then
        table.sort(self._listeners[event], function(a, b)
            return a.priority > b.priority
        end)
    end
    
    self._metrics.listenerAdds = self._metrics.listenerAdds + 1
    self:emit("newListener", event, fn)
    return self
end

function ProductionEmitter:once(event, fn, options)
    options = options or {}
    options.once = true
    return self:on(event, fn, options)
end

function ProductionEmitter:prependListener(event, fn)
    return self:on(event, fn, {prepend = true})
end

function ProductionEmitter:off(event, fn)
    if not event then
        self._listeners = {}
        return self
    end
    if not self._listeners[event] then return self end
    
    local new = {}
    local removed = 0
    for _, l in ipairs(self._listeners[event]) do
        if l.fn ~= fn then
            table.insert(new, l)
        else
            removed = removed + 1
        end
    end
    self._listeners[event] = new
    self._metrics.listenerRemoves = self._metrics.listenerRemoves + removed
    
    if removed > 0 then
        self:emit("removeListener", event, fn)
    end
    return self
end

function ProductionEmitter:emit(event, ...)
    -- Track metrics
    if not self._metrics.emits[event] then
        self._metrics.emits[event] = 0
    end
    self._metrics.emits[event] = self._metrics.emits[event] + 1
    
    if not self._listeners[event] then return false end
    
    local toKeep = {}
    for _, listener in ipairs(self._listeners[event]) do
        listener.callCount = listener.callCount + 1
        
        local ok, err = pcall(listener.fn, ...)
        
        if not ok then
            self._metrics.errors = self._metrics.errors + 1
            if self._captureRejections then
                if #(self._errorHandlers) > 0 then
                    for _, eh in ipairs(self._errorHandlers) do
                        pcall(eh, err, event)
                    end
                else
                    print(string.format("[EmitterError] Event '%s': %s", event, err))
                end
            end
        end
        
        if not listener.once then
            table.insert(toKeep, listener)
        end
    end
    self._listeners[event] = toKeep
    
    return true
end

function ProductionEmitter:onError(handler)
    table.insert(self._errorHandlers, handler)
    return self
end

function ProductionEmitter:getMetrics()
    local total = 0
    for _, count in pairs(self._metrics.emits) do
        total = total + count
    end
    return {
        totalEmits = total,
        emitsByEvent = self._metrics.emits,
        errors = self._metrics.errors,
        listenerAdds = self._metrics.listenerAdds,
        listenerRemoves = self._metrics.listenerRemoves,
    }
end

function ProductionEmitter:listenerCount(event)
    return self._listeners[event] and #self._listeners[event] or 0
end

function ProductionEmitter:rawListeners(event)
    return self._listeners[event] or {}
end

-- ทดสอบ
local emitter = ProductionEmitter.new({maxListeners = 5})

emitter:onError(function(err, event)
    print(string.format("[ErrorHandler] Error in '%s': %s", event, err))
end)

emitter:on("data", function(d) print("Handler 1: " .. tostring(d)) end)
emitter:on("data", function(d) 
    if type(d) ~= "number" then error("Expected number!") end
    print("Handler 2 (number): " .. d)
end)
emitter:once("data", function(d) print("Once handler: " .. tostring(d)) end)

emitter:emit("data", 42)
emitter:emit("data", "not a number")  -- triggers error in handler 2
emitter:emit("data", 100)

local metrics = emitter:getMetrics()
print(string.format("\nMetrics: %d total emits, %d errors", 
    metrics.totalEmits, metrics.errors))
```

## 39.16 สรุปบทที่ 39

```lua
-- ตัวอย่างที่ 17: สรุป Event System patterns
print("=== สรุป Event System ===")

local patterns = {
    {
        name = "EventEmitter",
        methods = "on(), off(), emit(), once()",
        when = "Component communication, lifecycle hooks",
    },
    {
        name = "Pub/Sub",
        methods = "subscribe(), publish(), unsubscribe()",
        when = "Decoupled messaging, microservices",
    },
    {
        name = "Event Store",
        methods = "append(), replay(), getByType()",
        when = "Event sourcing, audit trails, time travel",
    },
    {
        name = "Observable",
        methods = "subscribe(), map(), filter(), reduce()",
        when = "Reactive streams, data transformation",
    },
    {
        name = "Event Bus",
        methods = "on(), emit(), off()",
        when = "Global event hub, cross-component communication",
    },
    {
        name = "Middleware",
        methods = "use(), emit(), next()",
        when = "Logging, auth, transformation before handling",
    },
}

for _, p in ipairs(patterns) do
    print(string.format("\n[%s]", p.name))
    print(string.format("  Methods: %s", p.methods))
    print(string.format("  Use when: %s", p.when))
end
```

---

## แบบฝึกหัดบทที่ 39

1. เขียน `TypedEventEmitter` ที่ตรวจสอบ type ของ event data ก่อน emit
2. สร้าง `EventBridge` ที่ bridge events ระหว่าง 2 EventEmitters
3. Implement `EventReplay` ที่สามารถ replay events ใน time window ที่กำหนด
4. เขียน `SharedEventBus` ที่หลาย modules สามารถใช้ร่วมกันผ่าน singleton
5. สร้าง `EventValidator` middleware ที่ validate event data ด้วย schema
