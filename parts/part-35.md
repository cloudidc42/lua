# บทที่ 35: Memory Management และ Garbage Collection

## บทนำ

Lua ใช้ **automatic memory management** ผ่านระบบ Garbage Collection (GC) ทำให้นักพัฒนาไม่ต้องจัดการ memory ด้วยตนเอง อย่างไรก็ตาม การเข้าใจการทำงานของ GC ช่วยให้สามารถ tune performance ได้ดีขึ้น โดยเฉพาะในแอปพลิเคชันที่ต้องการ low latency เช่น เกมหรือ real-time systems

---

## 35.1 Garbage Collector ของ Lua

Lua ใช้ **Incremental Tri-color Mark-and-Sweep** garbage collector

### การทำงานของ GC

```
1. Mark Phase (สีขาว → สีเทา → สีดำ)
   - สีขาว: objects ที่ยังไม่ได้ตรวจสอบ (candidates for collection)
   - สีเทา: objects ที่ถูก reachable แต่ children ยังไม่ได้ scan
   - สีดำ: objects ที่ scan แล้วทั้ง object และ children

2. Sweep Phase
   - objects ที่ยังเป็นสีขาวหลัง mark phase = ไม่มี reference = garbage
   - ล้าง garbage objects
```

### ตัวอย่างที่ 1: ทำความเข้าใจ GC

```lua
-- example_01_gc_basics.lua

-- ดู memory usage ปัจจุบัน
local function memUsage()
    return collectgarbage("count")  -- returns KB
end

print("Initial memory:", memUsage(), "KB")

-- สร้าง objects จำนวนมาก
local t = {}
for i = 1, 100000 do
    t[i] = {x = i, y = i * 2, name = "obj" .. i}
end

print("After creating 100K objects:", memUsage(), "KB")

-- ลบ reference ทั้งหมด
t = nil

print("After removing reference:", memUsage(), "KB")

-- Force GC
collectgarbage("collect")

print("After GC:", memUsage(), "KB")
```

---

## 35.2 collectgarbage() Function

`collectgarbage()` เป็น interface หลักสำหรับควบคุม GC

### ตัวอย่างที่ 2: collectgarbage operations

```lua
-- example_02_collectgarbage.lua

-- ดู options ทั้งหมด
local function showGCInfo()
    -- count: คืน memory ที่ใช้อยู่ใน KB
    local mem = collectgarbage("count")
    print(string.format("Memory: %.2f KB (%.2f MB)", mem, mem/1024))
end

showGCInfo()

-- "collect": รัน full GC cycle
collectgarbage("collect")
print("After full collect:")
showGCInfo()

-- "stop": หยุด GC (ต้องระวัง!)
collectgarbage("stop")
print("\nGC stopped")

-- สร้าง objects โดยไม่มี GC
for i = 1, 10000 do
    local _ = {data = string.rep("x", 100)}
end

showGCInfo()

-- "restart": เริ่ม GC ใหม่
collectgarbage("restart")
print("GC restarted")
collectgarbage("collect")
showGCInfo()

-- "step": รัน GC step เดียว
collectgarbage("step", 100)  -- step size in KB

-- "isrunning": ตรวจสอบว่า GC กำลังทำงาน
print("\nGC running:", collectgarbage("isrunning"))

-- ดู generation (Lua 5.4)
-- print("Generation:", collectgarbage("generation"))
```

### ตัวอย่างที่ 3: GC Statistics

```lua
-- example_03_gc_stats.lua

local function gcStats()
    local count = collectgarbage("count")
    return {
        total_kb = count,
        total_mb = count / 1024,
        is_running = collectgarbage("isrunning")
    }
end

local function printStats(label)
    local s = gcStats()
    print(string.format("%-30s %8.2f KB %6.2f MB  GC: %s",
        label, s.total_kb, s.total_mb,
        s.is_running and "running" or "stopped"))
end

printStats("Start:")

-- Allocate memory
local data = {}
for i = 1, 50000 do
    data[i] = string.rep("a", 50)  -- 50 byte strings
end
printStats("After 50K strings:")

-- Release
data = nil
printStats("After release (before GC):")

collectgarbage("collect")
printStats("After GC:")
```

---

## 35.3 GC Modes: Incremental vs Generational

Lua 5.4 เพิ่ม **Generational GC** เป็นทางเลือกนอกจาก Incremental

### ตัวอย่างที่ 4: Incremental GC Mode

```lua
-- example_04_incremental_gc.lua

-- Incremental mode (default ใน Lua 5.4)
-- GC ทำงานทีละน้อย ระหว่าง allocation

collectgarbage("incremental")
print("GC mode: incremental")

-- ใน incremental mode, GC parameters:
-- pause: หยุดพัก GC ระหว่าง cycles (default 200 = 200%)
-- stepmul: speed ของ GC ต่อ KB ที่ allocate (default 100)
-- stepsize: amount of work ต่อ step (default 13 = 2^13 bytes)

-- ดู/ตั้งค่า parameters
-- collectgarbage("incremental", pause, stepmul, stepsize)

-- ตั้ง aggressive GC (ทำงานบ่อย)
collectgarbage("incremental", 100, 200)
print("Set aggressive GC")

-- ตั้ง lazy GC (ทำงานน้อย)
collectgarbage("incremental", 400, 50)
print("Set lazy GC")

-- reset to defaults
collectgarbage("incremental", 200, 100, 13)
print("Reset to defaults")
```

### ตัวอย่างที่ 5: Generational GC Mode (Lua 5.4)

```lua
-- example_05_generational_gc.lua
-- Generational GC ใน Lua 5.4

-- เปลี่ยนเป็น generational mode
-- collectgarbage("generational", minor_mul, major_mul)
-- minor_mul: ขยาย young generation (default 20)
-- major_mul: trigger major collection (default 100)

-- เปิด generational GC
collectgarbage("generational")
print("GC mode: generational")

-- Generational hypothesis:
-- "most objects die young"
-- Young generation: objects ที่เพิ่ง allocate
-- Old generation: objects ที่รอดจาก minor GC

-- ตัวอย่าง objects ที่อยู่ใน young generation
local function createTemporaryObjects()
    local result = 0
    for i = 1, 1000 do
        -- objects เหล่านี้มักจะ "die young"
        local temp = {x = i, y = i * 2}
        result = result + temp.x + temp.y
    end
    return result
end

-- เรียกหลายครั้ง
local start = os.clock()
for i = 1, 10000 do
    createTemporaryObjects()
end
print(string.format("Generational: %.4f sec", os.clock() - start))

-- กลับไป incremental
collectgarbage("incremental")
print("Switched back to incremental")

start = os.clock()
for i = 1, 10000 do
    createTemporaryObjects()
end
print(string.format("Incremental: %.4f sec", os.clock() - start))
```

---

## 35.4 Tuning GC Parameters

### ตัวอย่างที่ 6: Tuning สำหรับ Game Loop

```lua
-- example_06_game_gc_tuning.lua

-- สำหรับ game (ต้องการ smooth framerate)
-- ต้องการลด GC pauses

-- ลด GC pause time โดยรัน GC step เล็กๆ ทุก frame
local function gameLoop(frames)
    local frameTime = {}
    
    for frame = 1, frames do
        local frameStart = os.clock()
        
        -- game logic (simulate allocation)
        local particles = {}
        for i = 1, 100 do
            particles[i] = {x = math.random(800), y = math.random(600)}
        end
        
        -- update particles
        for _, p in ipairs(particles) do
            p.x = p.x + math.random(-5, 5)
            p.y = p.y + math.random(-5, 5)
        end
        
        -- Manual GC step ทุก frame (แทน automatic)
        -- ช่วยกระจาย GC work ไปในหลาย frames
        collectgarbage("step", 1)  -- small step
        
        frameTime[frame] = os.clock() - frameStart
    end
    
    -- วิเคราะห์ frame time
    local total = 0
    local maxTime = 0
    for _, t in ipairs(frameTime) do
        total = total + t
        if t > maxTime then maxTime = t end
    end
    
    return total / frames, maxTime
end

-- Game loop แบบ manual GC
collectgarbage("stop")  -- หยุด automatic GC
local avgTime, maxTime = gameLoop(1000)
collectgarbage("restart")

print(string.format("Manual GC - Avg: %.6f sec, Max: %.6f sec",
    avgTime, maxTime))
```

### ตัวอย่างที่ 7: Tuning สำหรับ Server

```lua
-- example_07_server_gc_tuning.lua

-- สำหรับ server (ต้องการ throughput สูง)
-- GC pause ยอมรับได้

-- ตั้ง GC ให้ทำงานน้อยลง (เพิ่ม throughput)
collectgarbage("incremental", 
    400,   -- pause: 400% = GC จะรอจนมี memory 4x ก่อน collect
    100,   -- stepmul: normal speed
    13     -- stepsize: normal
)

local function serverWorkload(requests)
    local processed = 0
    
    for i = 1, requests do
        -- simulate request processing
        local request = {
            id = i,
            data = string.rep("x", math.random(100, 1000)),
            timestamp = os.time()
        }
        
        -- process
        local response = {
            status = 200,
            body = "Processed " .. request.id,
            length = #request.data
        }
        
        processed = processed + 1
        
        -- request และ response จะ GC เอง
    end
    
    return processed
end

local start = os.clock()
local count = serverWorkload(100000)
local elapsed = os.clock() - start

print(string.format("Server workload: %d requests in %.4f sec",
    count, elapsed))
print(string.format("Throughput: %.0f req/sec", count / elapsed))

-- reset GC
collectgarbage("incremental", 200, 100, 13)
```

---

## 35.5 Weak Tables

Weak tables ช่วยจัดการ memory โดยไม่ป้องกัน GC จาก collect objects

### ตัวอย่างที่ 8: Weak Keys (__mode = "k")

```lua
-- example_08_weak_keys.lua

-- Weak key table: keys ไม่ป้องกัน GC
-- ใช้สำหรับ associating data กับ objects โดยไม่ป้องกัน GC

local weakKeyTable = setmetatable({}, {__mode = "k"})

-- สร้าง objects
local obj1 = {name = "Object 1"}
local obj2 = {name = "Object 2"}

-- เก็บ metadata
weakKeyTable[obj1] = {created = os.time(), tag = "first"}
weakKeyTable[obj2] = {created = os.time(), tag = "second"}

print("Before GC:")
local count = 0
for k, v in pairs(weakKeyTable) do
    count = count + 1
    print("  ", k.name, "->", v.tag)
end
print("Count:", count)

-- ลบ reference ไปที่ obj1
obj1 = nil

-- Force GC
collectgarbage("collect")

print("\nAfter GC (obj1 removed):")
count = 0
for k, v in pairs(weakKeyTable) do
    count = count + 1
    print("  ", k.name, "->", v.tag)
end
print("Count:", count)

-- ตัวอย่างใช้งานจริง: cache metadata ของ objects
local objectMetadata = setmetatable({}, {__mode = "k"})

local function setMetadata(obj, meta)
    objectMetadata[obj] = meta
end

local function getMetadata(obj)
    return objectMetadata[obj]
end

local myObj = {}
setMetadata(myObj, {version = "1.0", author = "test"})
print("\nMetadata:", getMetadata(myObj).version)

myObj = nil
collectgarbage("collect")
print("After GC, objectMetadata is empty:", next(objectMetadata) == nil)
```

### ตัวอย่างที่ 9: Weak Values (__mode = "v")

```lua
-- example_09_weak_values.lua

-- Weak value table: values ไม่ป้องกัน GC
-- ใช้สำหรับ cache

local cache = setmetatable({}, {__mode = "v"})

-- Cache expensive computations
local function getExpensiveData(key)
    -- ตรวจสอบ cache ก่อน
    if cache[key] then
        print("Cache hit:", key)
        return cache[key]
    end
    
    -- คำนวณ (simulate expensive operation)
    print("Cache miss - computing:", key)
    local result = {
        key = key,
        data = string.rep(tostring(key), 100),
        computed_at = os.time()
    }
    
    -- เก็บใน cache (weak reference)
    cache[key] = result
    
    return result
end

-- ใช้งาน cache
local data1 = getExpensiveData("config")
local data2 = getExpensiveData("config")  -- cache hit
local data3 = getExpensiveData("settings")

print("\nCache contents:")
for k, v in pairs(cache) do
    print("  " .. k .. ": " .. v.key)
end

-- ลบ reference ไปที่ data1 และ data3
data1 = nil
data3 = nil

collectgarbage("collect")

print("\nAfter releasing data1 and data3:")
print("Cache still has 'config':", cache["config"] ~= nil)
print("Cache still has 'settings':", cache["settings"] ~= nil)
print("data2 still valid:", data2 ~= nil)

-- ลบ data2 ด้วย
data2 = nil
collectgarbage("collect")

print("\nAfter releasing data2:")
for k, v in pairs(cache) do
    print("  Still in cache:", k)
end
print("Cache empty:", next(cache) == nil)
```

### ตัวอย่างที่ 10: Weak Keys และ Values (__mode = "kv")

```lua
-- example_10_weak_kv.lua

-- Weak key+value table: ทั้ง key และ value ไม่ป้องกัน GC

local weakBoth = setmetatable({}, {__mode = "kv"})

-- ใช้สำหรับ tracking relationships ระหว่าง objects
local source = {name = "source"}
local destination = {name = "destination"}

weakBoth[source] = destination

print("Before GC:")
print("  source -> destination:", weakBoth[source] ~= nil)

-- ลบ reference
destination = nil
collectgarbage("collect")

print("After removing destination:")
print("  source -> (nil):", weakBoth[source] == nil)

source = nil
collectgarbage("collect")

print("After removing source:")
print("  table empty:", next(weakBoth) == nil)
```

---

## 35.6 Ephemeron Tables

Ephemeron tables เป็น weak key tables ที่ value ก็จะถูก collect เมื่อ key ถูก collect

### ตัวอย่างที่ 11: Ephemeron Table

```lua
-- example_11_ephemeron.lua

-- ใน Lua, weak key table ที่มี value อ้างถึง key จะเป็น ephemeron
-- นี่คือพฤติกรรมปกติของ __mode = "k" ใน Lua 5.2+

-- ตัวอย่าง: event listener registry
local listeners = setmetatable({}, {__mode = "k"})

local function addListener(obj, callback)
    if not listeners[obj] then
        listeners[obj] = {}
    end
    table.insert(listeners[obj], callback)
end

local function triggerEvent(obj, event)
    if listeners[obj] then
        for _, callback in ipairs(listeners[obj]) do
            callback(event)
        end
    end
end

-- สร้าง objects
local button = {id = "submit_button"}
local textField = {id = "username_field"}

addListener(button, function(e)
    print("Button clicked:", e)
end)

addListener(textField, function(e)
    print("Text changed:", e)
end)

triggerEvent(button, "click")
triggerEvent(textField, "input")

print("\nListeners before GC:", (function()
    local c = 0
    for _ in pairs(listeners) do c = c + 1 end
    return c
end)())

-- ลบ objects
button = nil
collectgarbage("collect")

print("Listeners after button removed:", (function()
    local c = 0
    for _ in pairs(listeners) do c = c + 1 end
    return c
end)())

textField = nil
collectgarbage("collect")

print("Listeners after all removed:", (function()
    local c = 0
    for _ in pairs(listeners) do c = c + 1 end
    return c
end)())
```

---

## 35.7 Finalizers (__gc metamethod)

Finalizers รัน code ก่อน object ถูก garbage collected

### ตัวอย่างที่ 12: __gc Metamethod

```lua
-- example_12_finalizer.lua

-- __gc metamethod รันก่อน object ถูก collect

-- Resource wrapper ที่มี finalizer
local function createResource(name, cleanup)
    local resource = setmetatable({
        name = name,
        isOpen = true,
        _cleanup = cleanup
    }, {
        __gc = function(self)
            if self.isOpen then
                print(string.format("[GC] ปิด resource: %s", self.name))
                if self._cleanup then
                    self._cleanup(self)
                end
                self.isOpen = false
            end
        end,
        __tostring = function(self)
            return string.format("Resource(%s, open=%s)", 
                self.name, tostring(self.isOpen))
        end
    })
    
    return resource
end

-- ตัวอย่าง: file handle
local function openFile(path)
    print("Opening file:", path)
    return createResource("file:" .. path, function(self)
        print("Closing file:", self.name)
    end)
end

-- ตัวอย่าง: database connection
local function openConnection(host)
    print("Connecting to:", host)
    return createResource("db:" .. host, function(self)
        print("Disconnecting from:", self.name)
    end)
end

do
    -- สร้าง resources ใน scope
    local file = openFile("/tmp/test.txt")
    local db = openConnection("localhost")
    
    print("\nUsing resources...")
    print(tostring(file))
    print(tostring(db))
    
    -- เมื่อออกจาก scope, file และ db จะเป็น garbage
end

-- Force GC
collectgarbage("collect")
print("\nResources cleaned up by GC")
```

### ตัวอย่างที่ 13: Resource Management Pattern

```lua
-- example_13_resource_pattern.lua

-- Pattern ที่ดีกว่าสำหรับ resource management

local ManagedResource = {}
ManagedResource.__index = ManagedResource

-- สร้าง resource
function ManagedResource.new(name)
    local self = setmetatable({
        name = name,
        isOpen = true,
        _data = {}
    }, ManagedResource)
    
    -- ตั้ง finalizer
    -- ใน Lua 5.4 ใช้ to-be-closed variables แทน (ดูตัวอย่างที่ 14)
    return self
end

function ManagedResource:use(func)
    if not self.isOpen then
        error("Resource ปิดแล้ว: " .. self.name)
    end
    return func(self)
end

function ManagedResource:close()
    if self.isOpen then
        self.isOpen = false
        self._data = nil
        print("Closed:", self.name)
    end
end

function ManagedResource:write(data)
    if not self.isOpen then
        error("Cannot write to closed resource")
    end
    table.insert(self._data, data)
end

function ManagedResource:read()
    return table.concat(self._data, "\n")
end

-- try-finally pattern
local function withResource(factory, func)
    local resource = factory()
    local ok, err = pcall(func, resource)
    resource:close()
    if not ok then
        error(err)
    end
end

-- ใช้งาน
withResource(
    function() return ManagedResource.new("test_resource") end,
    function(res)
        res:write("line 1")
        res:write("line 2")
        res:write("line 3")
        print("Content:", res:read())
        -- error("simulate error")  -- ลองเปิด comment นี้
    end
)
print("Resource closed after use")
```

---

## 35.8 To-be-Closed Variables (Lua 5.4)

### ตัวอย่างที่ 14: To-be-Closed Variables

```lua
-- example_14_to_be_closed.lua
-- Lua 5.4 feature: <close> attribute

-- To-be-closed variables รัน __close method เมื่อออก scope
-- คล้าย defer ใน Go หรือ using ใน C#

-- สร้าง closable resource
local function newClosable(name)
    return setmetatable({name = name}, {
        __close = function(self, err)
            if err then
                print(string.format("[close] %s ปิดเนื่องจาก error: %s",
                    self.name, tostring(err)))
            else
                print(string.format("[close] %s ปิดปกติ", self.name))
            end
        end
    })
end

-- ตัวอย่างที่ 1: การปิดปกติ
print("Test 1: Normal close")
do
    local resource <close> = newClosable("Resource A")
    print("Using resource:", resource.name)
    -- resource.__close จะถูกเรียกเมื่อออก block
end

-- ตัวอย่างที่ 2: ปิดเมื่อ error
print("\nTest 2: Close on error")
local ok, err = pcall(function()
    local resource <close> = newClosable("Resource B")
    print("Using resource:", resource.name)
    error("Something went wrong!")
end)
print("Error caught:", err)

-- ตัวอย่างที่ 3: หลาย resources ปิดตามลำดับย้อนกลับ
print("\nTest 3: Multiple resources (LIFO order)")
do
    local r1 <close> = newClosable("Resource First")
    local r2 <close> = newClosable("Resource Second")
    local r3 <close> = newClosable("Resource Third")
    print("All resources open")
    -- r3, r2, r1 จะถูกปิดตามลำดับ (LIFO)
end
```

### ตัวอย่างที่ 15: File ด้วย To-be-Closed

```lua
-- example_15_file_tbc.lua

-- File handle ที่ปิดอัตโนมัติด้วย to-be-closed

local function openFileClosable(path, mode)
    local f, err = io.open(path, mode)
    if not f then
        return nil, err
    end
    
    return setmetatable({_file = f}, {
        __close = function(self)
            if self._file then
                self._file:close()
                self._file = nil
                print("File closed automatically")
            end
        end,
        __index = function(self, key)
            return self._file[key]
        end
    })
end

-- ใช้งาน (Lua 5.4)
do
    local file <close> = openFileClosable("/tmp/test_lua.txt", "w")
    if file then
        file:write("Hello, World!\n")
        file:write("This file will be closed automatically.\n")
        print("Written to file")
    end
    -- file จะถูกปิดอัตโนมัติที่นี่
end

print("After scope - file should be closed")
```

---

## 35.9 Memory Leak Patterns

### ตัวอย่างที่ 16: Global Variable Accumulation

```lua
-- example_16_global_leak.lua

-- Memory Leak Pattern 1: Global variables สะสม

-- BAD: เก็บ data ใน global
local _cache = {}  -- จริงๆ ควรเป็น module-local แต่ไม่มี cleanup

local function badCache(key, value)
    _cache[key] = value  -- เพิ่มแต่ไม่ลด!
end

-- GOOD: มี cleanup mechanism
local LRUCache = {}
LRUCache.__index = LRUCache

function LRUCache.new(maxSize)
    return setmetatable({
        _data = {},
        _order = {},  -- track insertion order
        _maxSize = maxSize or 100
    }, LRUCache)
end

function LRUCache:set(key, value)
    if #self._order >= self._maxSize and not self._data[key] then
        -- evict oldest
        local oldest = table.remove(self._order, 1)
        self._data[oldest] = nil
    end
    
    if not self._data[key] then
        table.insert(self._order, key)
    end
    
    self._data[key] = value
end

function LRUCache:get(key)
    return self._data[key]
end

function LRUCache:size()
    return #self._order
end

-- ทดสอบ
local cache = LRUCache.new(5)

for i = 1, 10 do
    cache:set("key" .. i, "value" .. i)
    print(string.format("Added key%d, cache size: %d", i, cache:size()))
end

-- ตรวจสอบ cache ไม่เกิน maxSize
assert(cache:size() <= 5, "Cache exceeded max size!")
print("Cache bounded correctly, size:", cache:size())
```

### ตัวอย่างที่ 17: Circular References

```lua
-- example_17_circular_refs.lua

-- Circular references ไม่ทำให้ memory leak ใน Lua
-- เพราะ mark-and-sweep จัดการได้

-- ตัวอย่าง: circular reference
local a = {}
local b = {}
a.other = b
b.other = a

print("Before GC:")
print("a:", tostring(a))
print("b:", tostring(b))

-- ลบ references
a = nil
b = nil

collectgarbage("collect")

-- a และ b ถูก collect แล้ว แม้จะมี circular reference
print("After GC: objects collected")

-- BUT: closure circular reference ต้องระวัง!
local function createCycle()
    local obj = {}
    
    -- function อ้างถึง obj, obj อ้างถึง function
    -- ทั้งคู่จะถูก collect เมื่อออก scope
    obj.method = function()
        return obj  -- closure captures obj
    end
    
    return obj
end

do
    local cycle = createCycle()
    print("Cycle created:", cycle ~= nil)
    -- cycle จะถูก collect เมื่อออก scope
end

collectgarbage("collect")
print("Circular closure collected")
```

### ตัวอย่างที่ 18: Event Listener Leak

```lua
-- example_18_event_leak.lua

-- Memory Leak Pattern: Event listeners ที่ไม่ถูก remove

-- BAD: leak pattern
local EventEmitter = {}
EventEmitter.__index = EventEmitter

function EventEmitter.new()
    return setmetatable({_listeners = {}}, EventEmitter)
end

function EventEmitter:on(event, callback)
    if not self._listeners[event] then
        self._listeners[event] = {}
    end
    table.insert(self._listeners[event], callback)
end

function EventEmitter:off(event, callback)
    if not self._listeners[event] then return end
    
    local listeners = self._listeners[event]
    for i = #listeners, 1, -1 do
        if listeners[i] == callback then
            table.remove(listeners, i)
        end
    end
end

function EventEmitter:emit(event, ...)
    if not self._listeners[event] then return end
    for _, cb in ipairs(self._listeners[event]) do
        cb(...)
    end
end

-- ใช้งานถูกต้อง: cleanup listeners
local emitter = EventEmitter.new()

local handler = function(msg)
    print("Received:", msg)
end

emitter:on("message", handler)
emitter:emit("message", "Hello!")

-- IMPORTANT: ลบ listener เมื่อไม่ต้องการ
emitter:off("message", handler)
handler = nil  -- ตอนนี้ callback ถูก GC ได้

print("Handler removed")

-- GOOD: ใช้ weak references สำหรับ event system
local WeakEventEmitter = {}
WeakEventEmitter.__index = WeakEventEmitter

function WeakEventEmitter.new()
    return setmetatable({
        _listeners = setmetatable({}, {__mode = "v"})
    }, WeakEventEmitter)
end

function WeakEventEmitter:on(event, callback)
    if not self._listeners[event] then
        self._listeners[event] = setmetatable({}, {__mode = "v"})
    end
    local id = tostring(#self._listeners[event] + 1)
    self._listeners[event][id] = callback
    return id
end

function WeakEventEmitter:emit(event, ...)
    if not self._listeners[event] then return end
    for _, cb in pairs(self._listeners[event]) do
        if cb then cb(...) end
    end
end

local weakEmitter = WeakEventEmitter.new()
local weakHandler = function(msg)
    print("Weak handler:", msg)
end

weakEmitter:on("test", weakHandler)
weakEmitter:emit("test", "World!")

weakHandler = nil
collectgarbage("collect")

print("After GC - emitting to collected handler:")
weakEmitter:emit("test", "Should be silent")
```

---

## 35.10 Large Data Handling

### ตัวอย่างที่ 19: ประมวลผล Large Data

```lua
-- example_19_large_data.lua

-- ประมวลผลข้อมูลขนาดใหญ่โดยไม่ load ทั้งหมดเข้า memory

-- Generator pattern
local function rangeGenerator(start, stop, step)
    step = step or 1
    local current = start - step
    
    return function()
        current = current + step
        if current <= stop then
            return current
        end
    end
end

-- ใช้ generator แทน table ขนาดใหญ่
local function sumRange(start, stop)
    local total = 0
    for n in rangeGenerator(start, stop) do
        total = total + n
    end
    return total
end

-- เปรียบเทียบ memory usage
local function sumWithTable(start, stop)
    -- สร้าง table ทั้งหมด (ใช้ memory มาก)
    local t = {}
    for i = start, stop do
        t[i - start + 1] = i
    end
    local total = 0
    for _, v in ipairs(t) do
        total = total + v
    end
    return total
end

local N = 100000

collectgarbage("collect")
local memBefore = collectgarbage("count")

local result1 = sumWithTable(1, N)
local memAfter = collectgarbage("count")
print(string.format("Table method: result=%d, memory=%+.0f KB",
    result1, memAfter - memBefore))

collectgarbage("collect")
memBefore = collectgarbage("count")

local result2 = sumRange(1, N)
memAfter = collectgarbage("count")
print(string.format("Generator method: result=%d, memory=%+.0f KB",
    result2, memAfter - memBefore))
```

### ตัวอย่างที่ 20: Streaming Processing

```lua
-- example_20_streaming.lua

-- Process ข้อมูลแบบ streaming เพื่อประหยัด memory

-- ตัวอย่าง: process CSV ทีละบรรทัด
local function processCSVLine(line)
    local fields = {}
    for field in (line .. ","):gmatch("([^,]*),") do
        table.insert(fields, field:match("^%s*(.-)%s*$"))
    end
    return fields
end

-- แทนการ load ทั้งไฟล์
local function processCSVStream(filename, processor)
    local f = io.open(filename, "r")
    if not f then
        -- simulate data
        local lines = {
            "name,age,city",
            "Alice,30,Bangkok",
            "Bob,25,Chiang Mai",
            "Charlie,35,Phuket",
        }
        for _, line in ipairs(lines) do
            processor(processCSVLine(line))
        end
        return
    end
    
    local lineCount = 0
    for line in f:lines() do
        lineCount = lineCount + 1
        processor(processCSVLine(line), lineCount)
    end
    
    f:close()
    return lineCount
end

-- ใช้งาน
local headerFields = nil
local dataCount = 0

processCSVStream(nil, function(fields, lineNum)
    if lineNum == 1 or not lineNum then
        if not lineNum then
            headerFields = fields
            print("Header:", table.concat(fields, " | "))
        else
            headerFields = fields
            print("Header:", table.concat(fields, " | "))
        end
    else
        dataCount = dataCount + 1
        print(string.format("Row %d: %s", dataCount, table.concat(fields, " | ")))
    end
end)

print(string.format("Processed %d data rows", dataCount))
```

---

## 35.11 String Interning

### ตัวอย่างที่ 21: String Memory

```lua
-- example_21_string_memory.lua

-- Lua intern ทุก short strings อัตโนมัติ
-- String เดียวกันจะใช้ memory เดียวกัน

-- ทดสอบ memory usage ของ strings
local function measureStringMemory()
    collectgarbage("collect")
    local before = collectgarbage("count")
    
    -- สร้าง strings ซ้ำๆ
    local strings = {}
    for i = 1, 10000 do
        -- string เดียวกัน Lua จะ intern
        strings[i] = "common_string"
    end
    
    collectgarbage("collect")
    local after = collectgarbage("count")
    
    print(string.format("10K identical strings: %+.2f KB", after - before))
    
    strings = nil
    collectgarbage("collect")
    before = collectgarbage("count")
    
    -- strings ที่ต่างกัน
    local differentStrings = {}
    for i = 1, 10000 do
        differentStrings[i] = "string_" .. i  -- unique strings
    end
    
    collectgarbage("collect")
    after = collectgarbage("count")
    
    print(string.format("10K unique strings:    %+.2f KB", after - before))
end

measureStringMemory()

-- String interning ช่วยให้ comparison เร็ว
local function compareInternedStrings(n)
    local s1 = "test_string_value"
    local s2 = "test_string_value"
    
    -- นี่คือ pointer comparison เพราะ interned
    local count = 0
    for i = 1, n do
        if s1 == s2 then count = count + 1 end
    end
    return count
end

local start = os.clock()
compareInternedStrings(10000000)
print(string.format("\nString comparison speed: %.4f sec for 10M comparisons",
    os.clock() - start))
```

---

## 35.12 GC Tuning Examples

### ตัวอย่างที่ 22: Real-time Application GC

```lua
-- example_22_realtime_gc.lua

-- สำหรับ real-time applications ที่ต้องการ predictable latency

local RealTimeGC = {}

-- ตั้งค่า GC สำหรับ real-time
function RealTimeGC.setup()
    -- หยุด automatic GC
    collectgarbage("stop")
    print("Automatic GC stopped")
    
    -- เก็บ budget สำหรับ GC work ต่อ frame
    RealTimeGC._budget = 1  -- 1 KB per step
    RealTimeGC._stepCount = 0
end

-- เรียกใน update loop
function RealTimeGC.step()
    -- รัน GC step เล็กๆ
    collectgarbage("step", RealTimeGC._budget)
    RealTimeGC._stepCount = RealTimeGC._stepCount + 1
end

-- เรียกเมื่อมีเวลาว่าง (เช่น หลัง render)
function RealTimeGC.idle()
    local mem = collectgarbage("count")
    
    -- ถ้า memory สูงเกินไป รัน GC เต็มที่
    if mem > 50000 then  -- > 50 MB
        collectgarbage("collect")
        print(string.format("Emergency GC: %.2f MB -> %.2f MB",
            mem / 1024, collectgarbage("count") / 1024))
    end
end

function RealTimeGC.stats()
    print(string.format("GC steps run: %d, Memory: %.2f KB",
        RealTimeGC._stepCount, collectgarbage("count")))
end

-- ตัวอย่าง game loop
RealTimeGC.setup()

for frame = 1, 100 do
    -- Update game state (allocate objects)
    local particles = {}
    for i = 1, 50 do
        particles[i] = {x = math.random(100), y = math.random(100)}
    end
    
    -- GC step ทุก frame
    RealTimeGC.step()
    
    -- Idle GC เป็นครั้งคราว
    if frame % 10 == 0 then
        RealTimeGC.idle()
    end
end

RealTimeGC.stats()
collectgarbage("restart")
```

### ตัวอย่างที่ 23: Batch Processing GC

```lua
-- example_23_batch_gc.lua

-- สำหรับ batch processing: ปล่อย GC ทำงานเต็มที่

local function processBatchWithGC(items, processor, batchSize)
    batchSize = batchSize or 1000
    local processed = 0
    local batchCount = 0
    
    local batch = {}
    
    for i, item in ipairs(items) do
        table.insert(batch, item)
        
        if #batch >= batchSize then
            -- process batch
            for _, bItem in ipairs(batch) do
                processor(bItem)
                processed = processed + 1
            end
            
            -- clear batch
            batch = {}
            batchCount = batchCount + 1
            
            -- GC หลังแต่ละ batch
            collectgarbage("collect")
            
            local mem = collectgarbage("count")
            print(string.format("Batch %d done: %d items, mem: %.0f KB",
                batchCount, processed, mem))
        end
    end
    
    -- process remaining
    for _, item in ipairs(batch) do
        processor(item)
        processed = processed + 1
    end
    
    return processed
end

-- สร้างข้อมูลทดสอบ
local testItems = {}
for i = 1, 10000 do
    testItems[i] = {
        id = i,
        data = string.rep("x", 100)
    }
end

local total = 0
local count = processBatchWithGC(testItems, function(item)
    total = total + item.id
end, 1000)

print(string.format("\nProcessed %d items, total sum = %d", count, total))
```

---

## 35.13 Memory Optimization Patterns

### ตัวอย่างที่ 24: Flyweight Pattern

```lua
-- example_24_flyweight.lua

-- Flyweight: share common state ระหว่าง objects
-- ลด memory usage เมื่อมี objects หลายตัวที่มี state เหมือนกัน

-- ไม่ดี: แต่ละ object เก็บข้อมูล texture ของตัวเอง
local function createSpriteNaive(textureData, x, y)
    return {
        textureData = textureData,  -- copy ข้อมูล!
        x = x,
        y = y
    }
end

-- ดี: share texture data (Flyweight)
local TextureCache = {}

local function getTexture(name)
    if not TextureCache[name] then
        -- load texture เพียงครั้งเดียว
        TextureCache[name] = {
            name = name,
            data = string.rep("pixel_data_" .. name, 100),
            width = 64,
            height = 64
        }
        print("Loaded texture:", name)
    end
    return TextureCache[name]
end

local function createSpriteFlyweight(textureName, x, y)
    return {
        texture = getTexture(textureName),  -- shared reference
        x = x,
        y = y
    }
end

-- สร้าง sprites จำนวนมาก
collectgarbage("collect")
local memBefore = collectgarbage("count")

local sprites = {}
for i = 1, 1000 do
    -- ใช้ textures ที่ซ้ำกัน
    local texName = "texture_" .. (i % 5 + 1)
    sprites[i] = createSpriteFlyweight(texName, i * 10, i * 5)
end

collectgarbage("collect")
local memAfter = collectgarbage("count")

print(string.format("\n1000 sprites with flyweight: %+.0f KB", 
    memAfter - memBefore))
print("Textures loaded:", (function()
    local c = 0
    for _ in pairs(TextureCache) do c = c + 1 end
    return c
end)())
```

### ตัวอย่างที่ 25: Memory Budget

```lua
-- example_25_memory_budget.lua

-- จัดการ memory budget สำหรับแอปพลิเคชัน

local MemoryManager = {}

function MemoryManager.new(maxKB)
    return {
        maxKB = maxKB,
        warnings = {},
        
        check = function(self, label)
            local current = collectgarbage("count")
            local usage = current / self.maxKB * 100
            
            if usage > 90 then
                -- Critical: force GC
                collectgarbage("collect")
                local after = collectgarbage("count")
                table.insert(self.warnings, {
                    level = "CRITICAL",
                    label = label,
                    before = current,
                    after = after
                })
            elseif usage > 75 then
                table.insert(self.warnings, {
                    level = "WARNING",
                    label = label,
                    usage = usage
                })
            end
            
            return current, usage
        end,
        
        report = function(self)
            local current = collectgarbage("count")
            print(string.format("\nMemory Report:"))
            print(string.format("  Current: %.2f KB / %d KB (%.1f%%)",
                current, self.maxKB, current / self.maxKB * 100))
            
            if #self.warnings > 0 then
                print("  Warnings:")
                for _, w in ipairs(self.warnings) do
                    if w.level == "CRITICAL" then
                        print(string.format("    [%s] %s: %.0f KB -> %.0f KB",
                            w.level, w.label, w.before, w.after))
                    else
                        print(string.format("    [%s] %s: %.1f%% usage",
                            w.level, w.label, w.usage))
                    end
                end
            end
        end
    }
end

-- ใช้งาน
local mm = MemoryManager.new(10000)  -- 10 MB budget

-- Simulate workload
for step = 1, 5 do
    -- allocate some memory
    local data = {}
    for i = 1, 10000 do
        data[i] = {value = i, str = "data_" .. i}
    end
    
    local mem, pct = mm:check("Step " .. step)
    print(string.format("Step %d: %.1f KB (%.1f%%)", step, mem, pct))
    
    -- sometimes release
    if step % 2 == 0 then
        data = nil
        collectgarbage("collect")
    end
end

mm:report()
```

---

## 35.14 Best Practices Summary

### ตัวอย่างที่ 26: Memory Best Practices

```lua
-- example_26_best_practices.lua

-- 1. Avoid unnecessary global variables
-- BAD:
globalData = {1, 2, 3}  -- ไม่ถูก collect เลย

-- GOOD:
local localData = {1, 2, 3}  -- ถูก collect เมื่อออก scope

-- 2. Clear large data structures
local function processLargeData()
    local bigData = {}
    for i = 1, 100000 do
        bigData[i] = i
    end
    
    -- process
    local sum = 0
    for _, v in ipairs(bigData) do
        sum = sum + v
    end
    
    -- clear explicitly ก่อน return
    bigData = nil  -- ช่วย GC รู้ว่าสามารถ collect ได้เร็วกว่า
    
    return sum
end

print("Large data sum:", processLargeData())

-- 3. Use weak tables for caches
local function createCache()
    return setmetatable({}, {__mode = "v"})
end

-- 4. Avoid closures ที่ capture ข้อมูลใหญ่
local function badClosure()
    local bigBuffer = string.rep("x", 1000000)  -- 1 MB
    
    -- closure นี้ capture bigBuffer ทั้งก้อน!
    return function()
        return #bigBuffer
    end
end

local function goodClosure()
    local bigBuffer = string.rep("x", 1000000)  -- 1 MB
    local bufferSize = #bigBuffer  -- capture เฉพาะที่ต้องการ
    bigBuffer = nil  -- release ทันที
    
    return function()
        return bufferSize  -- ไม่ capture bigBuffer แล้ว
    end
end

collectgarbage("collect")
local m1 = collectgarbage("count")
local f1 = badClosure()
local m2 = collectgarbage("count")
local f2 = goodClosure()
collectgarbage("collect")
local m3 = collectgarbage("count")

print(string.format("Bad closure size: %+.0f KB", m2 - m1))
print(string.format("Good closure result: %d", f2()))

-- 5. Use string.format instead of concatenation in loops
local function formatStrings()
    local results = {}
    for i = 1, 1000 do
        -- GOOD: string.format สร้าง string ใหม่ครั้งเดียว
        results[i] = string.format("Item #%04d: value=%d", i, i * 2)
    end
    return results
end

local strs = formatStrings()
print("First formatted string:", strs[1])
print("Total strings:", #strs)
```

### ตัวอย่างที่ 27: GC Event Monitoring

```lua
-- example_27_gc_monitor.lua

-- Monitor GC activity

local GCMonitor = {}

function GCMonitor.start()
    GCMonitor._startMem = collectgarbage("count")
    GCMonitor._startTime = os.clock()
    GCMonitor._samples = {}
    
    print(string.format("GC Monitor started - Initial memory: %.2f KB",
        GCMonitor._startMem))
end

function GCMonitor.sample(label)
    local mem = collectgarbage("count")
    local elapsed = os.clock() - GCMonitor._startTime
    
    table.insert(GCMonitor._samples, {
        label = label,
        memory = mem,
        time = elapsed,
        delta = mem - (GCMonitor._samples[#GCMonitor._samples] or 
            {memory = GCMonitor._startMem}).memory
    })
end

function GCMonitor.report()
    print("\n=== GC Monitor Report ===")
    print(string.format("%-30s %10s %10s %10s",
        "Label", "Memory(KB)", "Delta(KB)", "Time(s)"))
    print(string.rep("-", 62))
    
    for _, s in ipairs(GCMonitor._samples) do
        print(string.format("%-30s %10.2f %+10.2f %10.4f",
            s.label, s.memory, s.delta, s.time))
    end
    
    local totalDelta = collectgarbage("count") - GCMonitor._startMem
    print(string.rep("-", 62))
    print(string.format("Total delta: %+.2f KB", totalDelta))
end

-- ใช้งาน
GCMonitor.start()

-- Simulate work
GCMonitor.sample("Start")

local t1 = {}
for i = 1, 10000 do t1[i] = {data = string.rep("a", 50)} end
GCMonitor.sample("After 10K objects")

collectgarbage("collect")
GCMonitor.sample("After GC")

t1 = nil
GCMonitor.sample("After release")

collectgarbage("collect")
GCMonitor.sample("After final GC")

GCMonitor.report()
```

---

## 35.15 สรุป GC Configuration

### ตัวอย่างที่ 28: Configuration สำหรับ Use Cases ต่างๆ

```lua
-- example_28_gc_configs.lua

-- Configuration สำหรับ use cases ต่างๆ

local configs = {
    -- Game: smooth framerate, ลด stutter
    game = function()
        collectgarbage("stop")
        -- รัน GC ด้วยตนเองใน game loop
        return "Manual GC in game loop"
    end,
    
    -- Server: throughput สูง
    server = function()
        collectgarbage("incremental",
            400,   -- pause 400%: wait longer before GC
            100,   -- normal speed
            13     -- normal step
        )
        return "Lazy incremental GC"
    end,
    
    -- CLI tool: memory efficient
    cli = function()
        collectgarbage("incremental",
            100,   -- pause 100%: aggressive GC
            400,   -- faster GC
            13
        )
        return "Aggressive incremental GC"
    end,
    
    -- Generational (Lua 5.4): most objects die young
    generational = function()
        collectgarbage("generational",
            20,    -- minor_mul default
            100    -- major_mul default
        )
        return "Generational GC"
    end,
}

-- แสดง configurations
print("GC Configurations:")
for name, config in pairs(configs) do
    local desc = config()
    print(string.format("  %-20s %s", name .. ":", desc))
end

-- Reset to default
collectgarbage("incremental", 200, 100, 13)
print("\nReset to defaults")
print(string.format("Memory: %.2f KB", collectgarbage("count")))
```

### ตัวอย่างที่ 29: ตรวจสอบ Memory Pressure

```lua
-- example_29_memory_pressure.lua

-- ตรวจสอบ memory pressure และปรับ GC

local function getMemoryPressure()
    local mem = collectgarbage("count")
    
    -- กำหนด thresholds
    local LOW = 10 * 1024    -- 10 MB
    local MEDIUM = 50 * 1024  -- 50 MB
    local HIGH = 100 * 1024   -- 100 MB
    
    if mem < LOW then
        return "low", mem
    elseif mem < MEDIUM then
        return "medium", mem
    elseif mem < HIGH then
        return "high", mem
    else
        return "critical", mem
    end
end

local function adaptiveGC()
    local pressure, mem = getMemoryPressure()
    
    if pressure == "critical" then
        -- Force full collection
        collectgarbage("collect")
        collectgarbage("collect")  -- สองรอบ (ล้าง finalizers ด้วย)
        print(string.format("CRITICAL GC: forced collection, mem: %.0f KB", mem))
    elseif pressure == "high" then
        -- Incremental step ขนาดใหญ่
        collectgarbage("step", 100)
        print(string.format("HIGH pressure: large step, mem: %.0f KB", mem))
    elseif pressure == "medium" then
        -- Normal step
        collectgarbage("step", 10)
    end
    -- low pressure: ทำ nothing
end

-- Simulate varying load
for i = 1, 5 do
    -- allocate
    local _ = {}
    for j = 1, 10000 do
        _[j] = {data = string.rep("x", 50)}
    end
    
    adaptiveGC()
    
    -- release
    _ = nil
end

print("\nFinal memory:", collectgarbage("count"), "KB")
```

### ตัวอย่างที่ 30: Memory Pool for Performance

```lua
-- example_30_memory_pool_advanced.lua

-- Advanced memory pool สำหรับ high-performance applications

local Pool = {}
Pool.__index = Pool

function Pool.new(template, initSize)
    initSize = initSize or 10
    
    local pool = setmetatable({
        _free = {},
        _inUse = {},
        _template = template,
        _created = 0,
        _reused = 0
    }, Pool)
    
    -- pre-allocate
    for i = 1, initSize do
        pool._free[i] = pool:_createNew()
    end
    
    return pool
end

function Pool:_createNew()
    self._created = self._created + 1
    
    -- deep copy template
    local obj = {}
    if self._template then
        for k, v in pairs(self._template) do
            if type(v) ~= "function" then
                obj[k] = v
            end
        end
    end
    
    return obj
end

function Pool:acquire()
    local obj
    
    if #self._free > 0 then
        self._reused = self._reused + 1
        obj = table.remove(self._free)
    else
        obj = self:_createNew()
    end
    
    self._inUse[obj] = true
    return obj
end

function Pool:release(obj)
    if not self._inUse[obj] then
        error("Object not from this pool!")
    end
    
    self._inUse[obj] = nil
    
    -- reset to template values
    if self._template then
        for k, v in pairs(self._template) do
            if type(v) ~= "function" then
                obj[k] = v
            end
        end
    end
    
    table.insert(self._free, obj)
end

function Pool:stats()
    return {
        free = #self._free,
        inUse = (function()
            local c = 0
            for _ in pairs(self._inUse) do c = c + 1 end
            return c
        end)(),
        created = self._created,
        reused = self._reused,
        reuseRate = self._reused > 0 and 
            self._reused / (self._created + self._reused) or 0
    }
end

-- ใช้งาน
local vectorPool = Pool.new({x = 0, y = 0, z = 0}, 100)

-- simulate heavy usage
local start = os.clock()
for i = 1, 100000 do
    local v = vectorPool:acquire()
    v.x = math.random()
    v.y = math.random()
    v.z = math.random()
    
    -- use vector
    local _ = math.sqrt(v.x*v.x + v.y*v.y + v.z*v.z)
    
    vectorPool:release(v)
end

local elapsed = os.clock() - start
local stats = vectorPool:stats()

print(string.format("Pool benchmark: %.4f sec", elapsed))
print(string.format("Stats: created=%d, reused=%d, rate=%.1f%%",
    stats.created, stats.reused, stats.reuseRate * 100))
```

---

## สรุปบทที่ 35

### GC Control Functions

| Function | การใช้งาน |
|----------|----------|
| `collectgarbage("collect")` | Force full GC cycle |
| `collectgarbage("stop")` | หยุด automatic GC |
| `collectgarbage("restart")` | เริ่ม GC ใหม่ |
| `collectgarbage("step", n)` | รัน GC step ขนาด n KB |
| `collectgarbage("count")` | คืน memory ที่ใช้ (KB) |
| `collectgarbage("isrunning")` | ตรวจสอบ GC state |
| `collectgarbage("incremental", p, s, z)` | ตั้ง incremental mode |
| `collectgarbage("generational", m, M)` | ตั้ง generational mode (5.4) |

### Weak Table Modes

| Mode | Key | Value | ใช้สำหรับ |
|------|-----|-------|----------|
| `"k"` | weak | strong | Object metadata |
| `"v"` | strong | weak | Cache |
| `"kv"` | weak | weak | Relationship tracking |

### Memory Management Best Practices

1. **ใช้ local variables** - garbage collect ได้เร็วกว่า globals
2. **Clear ข้อมูลใหญ่** - ตั้งเป็น nil หลังใช้งาน
3. **Object pools** - สำหรับ objects ที่สร้าง/ลบบ่อย
4. **Weak tables** - สำหรับ caches และ event listeners
5. **table.concat** แทน string concatenation
6. **Generator/Iterator** แทน large tables
7. **Monitor memory** - ตรวจสอบ memory usage ใน production
8. **Tune GC** ตาม use case (game vs server vs CLI)

---

*จบบทที่ 35 - Memory Management และ Garbage Collection*

*จบ Series: Lua Programming Tutorial (บทที่ 31-35)*
