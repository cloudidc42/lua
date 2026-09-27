# บทที่ 20: Debugging และการแก้ไขข้อผิดพลาด

## บทนำ

Debugging คือกระบวนการค้นหาและแก้ไขข้อผิดพลาดในโปรแกรม Lua มี debug library ที่ทรงพลังและเทคนิคหลายอย่างสำหรับการ debug

ประเภทของ errors ใน Lua:
- **Syntax errors**: ข้อผิดพลาดในการเขียนโค้ด (แก้ไขโดย Lua compiler)
- **Runtime errors**: ข้อผิดพลาดขณะรันโปรแกรม
- **Logic errors**: โปรแกรมทำงานได้แต่ผลลัพธ์ไม่ถูกต้อง

---

## 20.1 Debug Library พื้นฐาน

```lua
-- ตัวอย่างที่ 1: debug.traceback()
local function level3()
    error("Something went wrong!")
end

local function level2()
    level3()
end

local function level1()
    level2()
end

-- ดู traceback
local ok, err = pcall(level1)
if not ok then
    print("Error caught:")
    print(err)
    print()
    print("Traceback:")
    print(debug.traceback("", 1))
end
```

```lua
-- ตัวอย่างที่ 2: debug.traceback() พร้อม custom message
local function divide(a, b)
    if b == 0 then
        error(debug.traceback("Division by zero!", 2))
    end
    return a / b
end

local function calculate(x)
    return divide(x, 0)
end

local ok, err = pcall(calculate, 10)
if not ok then
    print(err)
end
```

```lua
-- ตัวอย่างที่ 3: custom error handler
local function errorHandler(err)
    local msg = "FATAL ERROR: " .. tostring(err)
    local tb = debug.traceback("", 2)
    return msg .. "\n" .. tb
end

local function riskyOperation()
    local t = nil
    return t.field  -- attempt to index nil
end

local ok, err = xpcall(riskyOperation, errorHandler)
if not ok then
    print(err)
end
```

---

## 20.2 debug.getinfo()

```lua
-- ตัวอย่างที่ 4: debug.getinfo() พื้นฐาน
local function myFunction(a, b)
    local info = debug.getinfo(1, "nSl")
    print("Current function info:")
    print("  name:", info.name)
    print("  what:", info.what)     -- "Lua", "C", "main"
    print("  source:", info.source)
    print("  short_src:", info.short_src)
    print("  currentline:", info.currentline)
    print("  linedefined:", info.linedefined)
    print("  lastlinedefined:", info.lastlinedefined)
    return a + b
end

myFunction(10, 20)
```

```lua
-- ตัวอย่างที่ 5: ดู call stack ทั้งหมด
local function printStackTrace()
    print("=== Call Stack ===")
    local level = 1
    while true do
        local info = debug.getinfo(level, "nSl")
        if not info then break end
        
        local name = info.name or "?"
        local source = info.short_src
        local line = info.currentline
        
        print(string.format("  [%d] %s (%s:%d)",
            level, name, source, line))
        level = level + 1
    end
    print("=================")
end

local function c()
    printStackTrace()
end

local function b()
    c()
end

local function a()
    b()
end

a()
```

```lua
-- ตัวอย่างที่ 6: ข้อมูล function
local function getFunctionInfo(fn)
    local info = debug.getinfo(fn, "nSlu")
    return {
        name = info.name,
        source = info.short_src,
        lineDefined = info.linedefined,
        nParams = info.nparams,
        isVarArg = info.isvararg,
        nUpvalues = info.nups
    }
end

local function example(x, y, z)
    return x + y + z
end

local info = getFunctionInfo(example)
for k, v in pairs(info) do
    print(string.format("  %-15s = %s", k, tostring(v)))
end
```

---

## 20.3 debug.sethook() สำหรับ Profiling

```lua
-- ตัวอย่างที่ 7: Line-by-line profiling
local lineCount = {}

local function lineHook(event, line)
    local info = debug.getinfo(2, "S")
    local src = info.short_src
    local key = src .. ":" .. line
    lineCount[key] = (lineCount[key] or 0) + 1
end

-- เปิด line hook
debug.sethook(lineHook, "l")

-- โค้ดที่จะ profile
local function fibonacci(n)
    if n <= 1 then return n end
    return fibonacci(n-1) + fibonacci(n-2)
end

local result = fibonacci(10)

-- ปิด hook
debug.sethook()

print("Fibonacci(10) =", result)
print("\nLine execution counts:")
local lines = {}
for key, count in pairs(lineCount) do
    lines[#lines + 1] = {key = key, count = count}
end
table.sort(lines, function(a, b) return a.count > b.count end)
for _, item in ipairs(lines) do
    if item.count > 1 then  -- แสดงเฉพาะที่รันมากกว่า 1 ครั้ง
        print(string.format("  %-40s: %d times", item.key, item.count))
    end
end
```

```lua
-- ตัวอย่างที่ 8: Function call profiler
local callStats = {}

local function callHook(event)
    local info = debug.getinfo(2, "nS")
    local name = info.name or "(anonymous)"
    local source = info.short_src
    local key = source .. ":" .. name
    
    if not callStats[key] then
        callStats[key] = {name = name, calls = 0, source = source}
    end
    callStats[key].calls = callStats[key].calls + 1
end

debug.sethook(callHook, "c")  -- "c" = call events

-- Code to profile
local function helper(x)
    return x * 2
end

local function process(data)
    local sum = 0
    for _, v in ipairs(data) do
        sum = sum + helper(v)
    end
    return sum
end

local data = {}
for i = 1, 20 do data[i] = i end
process(data)

debug.sethook()

print("Function call statistics:")
local stats = {}
for _, s in pairs(callStats) do
    stats[#stats + 1] = s
end
table.sort(stats, function(a, b) return a.calls > b.calls end)
for _, s in ipairs(stats) do
    if s.calls > 0 then
        print(string.format("  %-20s: %5d calls", s.name, s.calls))
    end
end
```

```lua
-- ตัวอย่างที่ 9: Instruction count hook
local instrCount = 0

local function instrHook(event)
    instrCount = instrCount + 1
end

-- Profile specific function
local function profileFunction(fn, ...)
    instrCount = 0
    debug.sethook(instrHook, "", 1)  -- เรียกทุก 1 instruction
    local results = {fn(...)}
    debug.sethook()
    return instrCount, table.unpack(results)
end

-- Test different algorithms
local function bubbleSort(arr)
    local n = #arr
    for i = 1, n-1 do
        for j = 1, n-i do
            if arr[j] > arr[j+1] then
                arr[j], arr[j+1] = arr[j+1], arr[j]
            end
        end
    end
    return arr
end

local function quickSort(arr, low, high)
    low = low or 1
    high = high or #arr
    if low < high then
        local pivot = arr[high]
        local i = low - 1
        for j = low, high - 1 do
            if arr[j] <= pivot then
                i = i + 1
                arr[i], arr[j] = arr[j], arr[i]
            end
        end
        arr[i+1], arr[high] = arr[high], arr[i+1]
        local pi = i + 1
        quickSort(arr, low, pi - 1)
        quickSort(arr, pi + 1, high)
    end
    return arr
end

local function makeArray(n)
    local arr = {}
    math.randomseed(42)
    for i = 1, n do arr[i] = math.random(1, 100) end
    return arr
end

local arr1 = makeArray(20)
local arr2 = makeArray(20)

local count1 = profileFunction(bubbleSort, arr1)
local count2 = profileFunction(quickSort, arr2)

print(string.format("BubbleSort: %d instructions", count1))
print(string.format("QuickSort:  %d instructions", count2))
```

---

## 20.4 debug.getlocal() และ debug.setlocal()

```lua
-- ตัวอย่างที่ 10: debug.getlocal()
local function inspectLocals(level)
    level = level or 1
    print("Local variables at level " .. level .. ":")
    local i = 1
    while true do
        local name, value = debug.getlocal(level + 1, i)
        if not name then break end
        if name:sub(1, 1) ~= "(" then  -- ข้าม internal variables
            print(string.format("  [%d] %s = %s (%s)",
                i, name, tostring(value), type(value)))
        end
        i = i + 1
    end
end

local function myFunc()
    local x = 10
    local y = "hello"
    local z = {1, 2, 3}
    local w = true
    inspectLocals(1)  -- ดู locals ของ myFunc
end

myFunc()
```

```lua
-- ตัวอย่างที่ 11: debug.setlocal() - แก้ไขค่า local variable
local function modifyLocals()
    local x = 10
    local y = 20
    
    -- Hook ที่จะแก้ไข local variable
    debug.sethook(function(event)
        -- ตรวจสอบว่าอยู่ใน modifyLocals
        local info = debug.getinfo(2, "n")
        if info and info.name == "modifyLocals" then
            -- ค้นหา variable x
            local i = 1
            while true do
                local name, val = debug.getlocal(2, i)
                if not name then break end
                if name == "x" and val == 10 then
                    debug.setlocal(2, i, 999)  -- เปลี่ยน x เป็น 999
                end
                i = i + 1
            end
            debug.sethook()  -- ปิด hook หลังจากแก้แล้ว
        end
    end, "l")
    
    -- Trigger the hook
    local dummy = x + y  -- line ที่ hook จะทำงาน
    
    return x, y  -- x ควรเป็น 999 หรือ 10 ขึ้นกับ timing
end

local a, b = modifyLocals()
print("x =", a, "y =", b)
```

```lua
-- ตัวอย่างที่ 12: Variable inspector utility
local function inspect(level)
    level = level or 1
    local vars = {}
    
    -- Get local variables
    local i = 1
    while true do
        local name, value = debug.getlocal(level + 1, i)
        if not name then break end
        if name:sub(1, 1) ~= "(" then
            vars[name] = {
                value = value,
                varType = type(value),
                kind = "local"
            }
        end
        i = i + 1
    end
    
    -- Get upvalues of current function
    local info = debug.getinfo(level + 1, "f")
    if info and info.func then
        i = 1
        while true do
            local name, value = debug.getupvalue(info.func, i)
            if not name then break end
            vars[name] = {
                value = value,
                varType = type(value),
                kind = "upvalue"
            }
            i = i + 1
        end
    end
    
    return vars
end

-- Test
local upval = "I am an upvalue"

local function testInspect()
    local a = 42
    local b = "hello"
    local c = {x = 1, y = 2}
    
    local vars = inspect(1)
    print("Variables in scope:")
    for name, info in pairs(vars) do
        local valStr
        if type(info.value) == "table" then
            valStr = "{...}"
        else
            valStr = tostring(info.value)
        end
        print(string.format("  %-15s [%-8s] (%s) = %s",
            name, info.kind, info.varType, valStr))
    end
end

testInspect()
```

---

## 20.5 debug.getupvalue() และ debug.setupvalue()

```lua
-- ตัวอย่างที่ 13: Inspecting upvalues
local function makeCounter(start)
    local count = start or 0  -- upvalue
    
    return {
        inc = function() count = count + 1 end,
        dec = function() count = count - 1 end,
        get = function() return count end
    }
end

local counter = makeCounter(10)
counter.inc()
counter.inc()
print("Counter:", counter.get())  -- 12

-- Inspect upvalue ของ counter.get
local i = 1
while true do
    local name, value = debug.getupvalue(counter.get, i)
    if not name then break end
    print(string.format("Upvalue[%d]: %s = %s", i, name, tostring(value)))
    i = i + 1
end
```

```lua
-- ตัวอย่างที่ 14: Modifying upvalues (เทคนิคขั้นสูง)
local function makeAccumulator()
    local total = 0
    return function(n)
        total = total + n
        return total
    end
end

local acc = makeAccumulator()
print(acc(10))  -- 10
print(acc(20))  -- 30
print(acc(5))   -- 35

-- ค้นหาและแก้ไข upvalue
local function resetUpvalue(fn, name, newValue)
    local i = 1
    while true do
        local uvName, _ = debug.getupvalue(fn, i)
        if not uvName then break end
        if uvName == name then
            debug.setupvalue(fn, i, newValue)
            return true
        end
        i = i + 1
    end
    return false
end

resetUpvalue(acc, "total", 0)  -- reset total to 0
print(acc(50))  -- 50 (reset worked)
```

```lua
-- ตัวอย่างที่ 15: Upvalue sharing
local function createSharedState()
    local shared = {count = 0, data = {}}
    
    local function increment()
        shared.count = shared.count + 1
    end
    
    local function addData(val)
        shared.data[#shared.data + 1] = val
    end
    
    local function getState()
        return shared
    end
    
    -- ตรวจสอบว่า functions share upvalue เดียวกัน
    local _, uv1 = debug.getupvalue(increment, 1)
    local _, uv2 = debug.getupvalue(addData, 1)
    local _, uv3 = debug.getupvalue(getState, 1)
    
    print("Functions share same upvalue:", uv1 == uv2 and uv2 == uv3)
    
    return increment, addData, getState
end

local inc, add, get = createSharedState()
inc(); inc(); inc()
add("hello"); add("world")

local state = get()
print("Count:", state.count)           -- 3
print("Data:", table.concat(state.data, ", "))  -- hello, world
```

---

## 20.6 debug.getmetatable() และ debug.setmetatable()

```lua
-- ตัวอย่างที่ 16: debug.getmetatable() - bypass protection
-- ปกติ getmetatable() จะ return nil ถ้า __metatable ตั้งไว้
local protected = setmetatable({}, {
    __metatable = "protected",
    __index = function(_, k) return "default" end
})

-- getmetatable() ปกติ
print("getmetatable():", getmetatable(protected))  -- protected

-- debug.getmetatable() bypasses __metatable
local mt = debug.getmetatable(protected)
print("debug.getmetatable():", type(mt))  -- table
print("Has __index:", mt.__index ~= nil)  -- true

-- ตัวอย่างการใช้สำหรับ introspection
local function analyzeObject(obj)
    local mt = debug.getmetatable(obj)
    if not mt then
        print("No metatable")
        return
    end
    
    print("Metatable analysis:")
    local metamethods = {
        "__index", "__newindex", "__call", "__tostring",
        "__add", "__sub", "__mul", "__div", "__mod",
        "__eq", "__lt", "__le", "__len", "__concat",
        "__gc", "__close"
    }
    
    for _, mm in ipairs(metamethods) do
        if mt[mm] ~= nil then
            print(string.format("  %-15s: %s", mm, type(mt[mm])))
        end
    end
end

local MyClass = {}
MyClass.__index = MyClass
MyClass.__tostring = function(self) return "MyClass" end
MyClass.__add = function(a, b) return a end
MyClass.__len = function(self) return 0 end

local obj = setmetatable({}, MyClass)
analyzeObject(obj)
```

---

## 20.7 Print Debugging Techniques

```lua
-- ตัวอย่างที่ 17: Structured debug output
local debug_mode = true

local function dbg(...)
    if debug_mode then
        local info = debug.getinfo(2, "nSl")
        local name = info.name or "?"
        local line = info.currentline
        io.write(string.format("[DEBUG %s:%d] ", name, line))
        print(...)
    end
end

local function processData(data)
    dbg("Processing", #data, "items")
    local result = {}
    for i, v in ipairs(data) do
        dbg(string.format("Item %d: %s", i, tostring(v)))
        result[i] = v * 2
    end
    dbg("Done, result has", #result, "items")
    return result
end

local data = {1, 2, 3, 4, 5}
local result = processData(data)
print("Result:", table.concat(result, ", "))
```

```lua
-- ตัวอย่างที่ 18: Pretty printer สำหรับ tables
local function prettyPrint(value, indent, seen)
    indent = indent or ""
    seen = seen or {}
    
    local t = type(value)
    
    if t == "nil" then return "nil"
    elseif t == "boolean" then return tostring(value)
    elseif t == "number" then
        if math.type and math.type(value) == "float" then
            return string.format("%.6g", value)
        end
        return tostring(value)
    elseif t == "string" then
        return string.format("%q", value)
    elseif t == "function" then
        local info = debug.getinfo(value, "nS")
        return string.format("<function %s @%s:%d>",
            info.name or "?",
            info.short_src,
            info.linedefined)
    elseif t == "table" then
        if seen[value] then return "<circular>" end
        seen[value] = true
        
        local items = {}
        local nextIndent = indent .. "  "
        
        -- Check if array
        local isArray = #value > 0
        if isArray then
            for i, v in ipairs(value) do
                items[#items + 1] = nextIndent ..
                    "[" .. i .. "] = " ..
                    prettyPrint(v, nextIndent, seen)
            end
        else
            local keys = {}
            for k in pairs(value) do keys[#keys + 1] = k end
            table.sort(keys, function(a, b)
                return tostring(a) < tostring(b)
            end)
            for _, k in ipairs(keys) do
                items[#items + 1] = nextIndent ..
                    tostring(k) .. " = " ..
                    prettyPrint(value[k], nextIndent, seen)
            end
        end
        
        seen[value] = nil
        if #items == 0 then return "{}" end
        return "{\n" .. table.concat(items, ",\n") .. "\n" .. indent .. "}"
    else
        return string.format("<%s>", t)
    end
end

-- Test
local data = {
    name = "Alice",
    age = 30,
    scores = {95, 87, 92},
    address = {
        city = "Bangkok",
        zip = "10100"
    },
    active = true
}

print(prettyPrint(data))
```

---

## 20.8 Assertion-based Debugging

```lua
-- ตัวอย่างที่ 19: Enhanced assert
local function assertType(value, expectedType, name)
    if type(value) ~= expectedType then
        local info = debug.getinfo(2, "nSl")
        error(string.format(
            "Type assertion failed at %s:%d\n  %s: expected %s, got %s (%s)",
            info.short_src,
            info.currentline,
            name or "value",
            expectedType,
            type(value),
            tostring(value)
        ), 2)
    end
    return value
end

local function assertRange(value, min, max, name)
    assertType(value, "number", name)
    if value < min or value > max then
        local info = debug.getinfo(2, "nSl")
        error(string.format(
            "Range assertion failed at %s:%d\n  %s: %s not in [%s, %s]",
            info.short_src,
            info.currentline,
            name or "value",
            tostring(value),
            tostring(min),
            tostring(max)
        ), 2)
    end
    return value
end

local function assertNotNil(value, name)
    if value == nil then
        local info = debug.getinfo(2, "nSl")
        error(string.format(
            "Nil assertion failed at %s:%d\n  %s is nil",
            info.short_src,
            info.currentline,
            name or "value"
        ), 2)
    end
    return value
end

-- Test assertions
local function calculateScore(age, points, bonus)
    assertType(age, "number", "age")
    assertRange(age, 0, 150, "age")
    assertType(points, "number", "points")
    assertRange(points, 0, 1000, "points")
    bonus = bonus or 0
    assertType(bonus, "number", "bonus")
    
    return points * (1 + bonus/100) * (age > 18 and 1.1 or 1.0)
end

print(calculateScore(25, 500, 10))  -- ok

local ok, err = pcall(calculateScore, "twenty", 500, 10)
if not ok then print("Error:", err) end

ok, err = pcall(calculateScore, 200, 500, 10)
if not ok then print("Error:", err) end
```

```lua
-- ตัวอย่างที่ 20: Contract programming
local function contract(fn, preconditions, postconditions)
    return function(...)
        -- ตรวจ preconditions
        for _, pre in ipairs(preconditions or {}) do
            local ok, err = pcall(pre, ...)
            if not ok then
                error("Precondition failed: " .. tostring(err), 2)
            end
        end
        
        -- รัน function
        local results = {fn(...)}
        
        -- ตรวจ postconditions
        for _, post in ipairs(postconditions or {}) do
            local ok, err = pcall(post, results, ...)
            if not ok then
                error("Postcondition failed: " .. tostring(err), 2)
            end
        end
        
        return table.unpack(results)
    end
end

local sqrt = contract(
    math.sqrt,
    {
        function(x)
            assert(type(x) == "number", "x must be number")
            assert(x >= 0, "x must be non-negative, got " .. tostring(x))
        end
    },
    {
        function(results, x)
            local r = results[1]
            assert(math.abs(r * r - x) < 1e-10, "Result verification failed")
        end
    }
)

print(sqrt(16))   -- 4.0
print(sqrt(2))    -- 1.414...

local ok, err = pcall(sqrt, -1)
print("Error:", err)  -- Precondition failed

local ok2, err2 = pcall(sqrt, "hello")
print("Error:", err2)  -- Precondition failed
```

---

## 20.9 Logging Levels

```lua
-- ตัวอย่างที่ 21: Logging system สมบูรณ์
local Logger = {}
Logger.__index = Logger

-- Log levels
Logger.LEVELS = {
    TRACE = 1,
    DEBUG = 2,
    INFO  = 3,
    WARN  = 4,
    ERROR = 5,
    FATAL = 6,
    OFF   = 7
}

local LEVEL_NAMES = {}
for name, val in pairs(Logger.LEVELS) do
    LEVEL_NAMES[val] = name
end

local LEVEL_COLORS = {
    [1] = "\27[37m",   -- TRACE: white
    [2] = "\27[36m",   -- DEBUG: cyan
    [3] = "\27[32m",   -- INFO: green
    [4] = "\27[33m",   -- WARN: yellow
    [5] = "\27[31m",   -- ERROR: red
    [6] = "\27[35m",   -- FATAL: magenta
}
local RESET = "\27[0m"

function Logger.new(name, level, options)
    options = options or {}
    return setmetatable({
        name = name,
        level = level or Logger.LEVELS.INFO,
        useColor = options.useColor ~= false,
        showSource = options.showSource or false,
        handlers = {},
        formatter = options.formatter
    }, Logger)
end

function Logger:addHandler(handler)
    self.handlers[#self.handlers + 1] = handler
    return self
end

function Logger:setLevel(level)
    if type(level) == "string" then
        level = Logger.LEVELS[level:upper()]
    end
    self.level = level
    return self
end

function Logger:_shouldLog(level)
    return level >= self.level
end

function Logger:_format(level, msg, ...)
    if select('#', ...) > 0 then
        msg = string.format(msg, ...)
    end
    
    local timestamp = os.date("%Y-%m-%d %H:%M:%S")
    local levelName = LEVEL_NAMES[level] or "UNKNOWN"
    
    local source = ""
    if self.showSource then
        local info = debug.getinfo(4, "Sl")
        if info then
            source = string.format("[%s:%d] ",
                info.short_src:match("[^/\\]+$") or info.short_src,
                info.currentline)
        end
    end
    
    local formatted = string.format("[%s] [%-5s] [%s] %s%s",
        timestamp, levelName, self.name, source, msg)
    
    if self.useColor and LEVEL_COLORS[level] then
        return LEVEL_COLORS[level] .. formatted .. RESET
    end
    
    return formatted
end

function Logger:log(level, msg, ...)
    if not self:_shouldLog(level) then return end
    
    local formatted = self:_format(level, msg, ...)
    
    -- Default handler: print to stdout
    if #self.handlers == 0 then
        print(formatted)
    else
        for _, handler in ipairs(self.handlers) do
            handler(level, formatted, msg)
        end
    end
end

function Logger:trace(msg, ...) self:log(Logger.LEVELS.TRACE, msg, ...) end
function Logger:debug(msg, ...) self:log(Logger.LEVELS.DEBUG, msg, ...) end
function Logger:info(msg, ...)  self:log(Logger.LEVELS.INFO,  msg, ...) end
function Logger:warn(msg, ...)  self:log(Logger.LEVELS.WARN,  msg, ...) end
function Logger:error(msg, ...) self:log(Logger.LEVELS.ERROR, msg, ...) end
function Logger:fatal(msg, ...) self:log(Logger.LEVELS.FATAL, msg, ...) end

-- Test
local log = Logger.new("MyApp", Logger.LEVELS.DEBUG, {
    useColor = false,  -- disable color for clean output
    showSource = false
})

log:debug("Starting application")
log:info("Server listening on port %d", 8080)
log:warn("High memory usage: %.1f%%", 85.5)
log:error("Failed to connect to database: %s", "timeout")
```

```lua
-- ตัวอย่างที่ 22: File logger
local function createFileLogger(filename, minLevel)
    local f = io.open(filename, "a")
    if not f then
        error("Cannot open log file: " .. filename)
    end
    
    return function(level, formatted, original)
        f:write(formatted .. "\n")
        f:flush()
    end
end

-- ตัวอย่างการใช้:
local tmpLog = "/tmp/app_" .. os.time() .. ".log"
local log2 = Logger.new("FileApp", Logger.LEVELS.DEBUG, {useColor = false})
log2:addHandler(createFileLogger(tmpLog, Logger.LEVELS.INFO))

log2:info("This goes to file")
log2:error("This error goes to file too")

print("Log written to:", tmpLog)

-- อ่านไฟล์ log กลับมา
local f = io.open(tmpLog, "r")
if f then
    print("\nLog file contents:")
    for line in f:lines() do
        print(line)
    end
    f:close()
    os.remove(tmpLog)
end
```

---

## 20.10 Breakpoints ด้วย Debug Hooks

```lua
-- ตัวอย่างที่ 23: Simple breakpoint system
local Breakpoints = {}
Breakpoints._points = {}
Breakpoints._active = false

function Breakpoints.add(source, line, condition)
    Breakpoints._points[source .. ":" .. line] = {
        source = source,
        line = line,
        condition = condition,
        hitCount = 0
    }
end

function Breakpoints.enable()
    Breakpoints._active = true
    debug.sethook(function(event, line)
        if not Breakpoints._active then return end
        
        local info = debug.getinfo(2, "S")
        local src = info.short_src:match("[^/\\]+$") or info.short_src
        local key = src .. ":" .. line
        
        local bp = Breakpoints._points[key]
        if bp then
            bp.hitCount = bp.hitCount + 1
            
            -- Check condition
            if bp.condition then
                local env = {}
                -- Collect local variables
                local i = 1
                while true do
                    local name, val = debug.getlocal(2, i)
                    if not name then break end
                    env[name] = val
                    i = i + 1
                end
                
                local fn = load("return " .. bp.condition, "condition", "t", env)
                if fn then
                    local ok, result = pcall(fn)
                    if not ok or not result then return end
                end
            end
            
            print(string.format("\n[BREAKPOINT] %s:%d (hit #%d)",
                bp.source, bp.line, bp.hitCount))
            
            -- Show locals
            print("  Locals:")
            local i = 1
            while true do
                local name, val = debug.getlocal(2, i)
                if not name then break end
                if name:sub(1,1) ~= "(" then
                    print(string.format("    %s = %s", name, tostring(val)))
                end
                i = i + 1
            end
        end
    end, "l")
end

function Breakpoints.disable()
    Breakpoints._active = false
    debug.sethook()
end

-- เพิ่ม breakpoint
Breakpoints.add("stdin", 5)  -- ไฟล์ stdin หรือชื่อไฟล์จริง

-- การใช้งานจริงจะเป็น:
-- Breakpoints.add("myfile.lua", 42, "x > 10")
-- Breakpoints.enable()
-- ...โค้ดที่ต้องการ debug...
-- Breakpoints.disable()
```

---

## 20.11 Memory Usage และ collectgarbage()

```lua
-- ตัวอย่างที่ 24: Memory monitoring
local function getMemoryKB()
    return collectgarbage("count")
end

local function getMemoryMB()
    return collectgarbage("count") / 1024
end

local function measureMemory(fn, ...)
    collectgarbage("collect")
    local before = getMemoryKB()
    
    local results = {fn(...)}
    
    local after = getMemoryKB()
    local diff = after - before
    
    return diff, table.unpack(results)
end

print(string.format("Initial memory: %.2f KB", getMemoryKB()))

-- Test memory usage
local mem, _ = measureMemory(function()
    local t = {}
    for i = 1, 10000 do
        t[i] = {value = i, name = "item_" .. i}
    end
    return t
end)

print(string.format("Table allocation: %.2f KB", mem))

-- GC อัตโนมัติ
collectgarbage("collect")
print(string.format("After GC: %.2f KB", getMemoryKB()))
```

```lua
-- ตัวอย่างที่ 25: GC statistics
local function gcStats()
    return {
        totalKB = collectgarbage("count"),
        gcStep = function(n)
            collectgarbage("step", n or 100)
        end,
        stop = function() collectgarbage("stop") end,
        restart = function() collectgarbage("restart") end,
        collect = function() collectgarbage("collect") end,
        isRunning = function()
            return collectgarbage("isrunning")
        end
    }
end

local gc = gcStats()
print("Memory:", gc.totalKB, "KB")
print("GC running:", gc.isRunning())

-- สร้าง garbage
local function createGarbage()
    local t = {}
    for i = 1, 1000 do
        t[i] = string.rep("x", 100)
    end
    -- t จะ out of scope และกลายเป็น garbage
end

for i = 1, 5 do
    createGarbage()
end

print("Before GC:", collectgarbage("count"), "KB")
collectgarbage("collect")
print("After GC:", collectgarbage("count"), "KB")
```

```lua
-- ตัวอย่างที่ 26: Memory leak detection
local function detectLeaks(fn, threshold)
    threshold = threshold or 100  -- KB
    
    collectgarbage("collect")
    local baseline = collectgarbage("count")
    
    local iterations = 10
    local samples = {}
    
    for i = 1, iterations do
        fn()
        collectgarbage("collect")
        samples[i] = collectgarbage("count")
    end
    
    -- Check if memory grows consistently
    local growing = 0
    for i = 2, #samples do
        if samples[i] > samples[i-1] then
            growing = growing + 1
        end
    end
    
    local finalMem = samples[#samples]
    local growth = finalMem - baseline
    
    local result = {
        baseline = baseline,
        final = finalMem,
        growth = growth,
        samples = samples,
        hasLeak = growing >= iterations * 0.7 and growth > threshold
    }
    
    return result
end

-- Function ที่ไม่ leak
local function noLeak()
    local t = {}
    for i = 1, 1000 do t[i] = i end
    -- t จะถูก GC เมื่อออก scope
end

local result = detectLeaks(noLeak, 10)
print(string.format("Memory check: baseline=%.1fKB final=%.1fKB growth=%.1fKB leak=%s",
    result.baseline, result.final, result.growth,
    result.hasLeak and "POSSIBLE" or "NONE"))
```

---

## 20.12 Performance Measurement

```lua
-- ตัวอย่างที่ 27: Benchmark utility
local Benchmark = {}

function Benchmark.run(name, fn, iterations, warmup)
    iterations = iterations or 1000
    warmup = warmup or math.ceil(iterations * 0.1)
    
    -- Warmup
    for _ = 1, warmup do
        fn()
    end
    
    -- Actual measurement
    local times = {}
    for _ = 1, iterations do
        local start = os.clock()
        fn()
        local elapsed = os.clock() - start
        times[#times + 1] = elapsed
    end
    
    -- Statistics
    table.sort(times)
    
    local total = 0
    for _, t in ipairs(times) do total = total + t end
    local mean = total / #times
    
    local variance = 0
    for _, t in ipairs(times) do
        variance = variance + (t - mean)^2
    end
    variance = variance / #times
    local stddev = math.sqrt(variance)
    
    local median = times[math.ceil(#times / 2)]
    local p95 = times[math.ceil(#times * 0.95)]
    local p99 = times[math.ceil(#times * 0.99)]
    
    return {
        name = name,
        iterations = iterations,
        total = total,
        mean = mean,
        median = median,
        stddev = stddev,
        min = times[1],
        max = times[#times],
        p95 = p95,
        p99 = p99
    }
end

function Benchmark.report(result)
    print(string.format("Benchmark: %s (%d iterations)",
        result.name, result.iterations))
    print(string.format("  Mean:    %.6f ms", result.mean * 1000))
    print(string.format("  Median:  %.6f ms", result.median * 1000))
    print(string.format("  StdDev:  %.6f ms", result.stddev * 1000))
    print(string.format("  Min:     %.6f ms", result.min * 1000))
    print(string.format("  Max:     %.6f ms", result.max * 1000))
    print(string.format("  P95:     %.6f ms", result.p95 * 1000))
    print(string.format("  Ops/sec: %.0f", 1 / result.mean))
end

-- Test different implementations
local function concat_normal(n)
    local s = ""
    for i = 1, n do
        s = s .. tostring(i)
    end
    return s
end

local function concat_table(n)
    local parts = {}
    for i = 1, n do
        parts[i] = tostring(i)
    end
    return table.concat(parts)
end

print("Comparing string concatenation methods:")
local r1 = Benchmark.run(".. operator", function()
    concat_normal(100)
end, 100)

local r2 = Benchmark.run("table.concat", function()
    concat_table(100)
end, 100)

Benchmark.report(r1)
print()
Benchmark.report(r2)

if r1.mean > 0 then
    print(string.format("\ntable.concat is %.1fx faster",
        r1.mean / r2.mean))
end
```

```lua
-- ตัวอย่างที่ 28: os.clock() สำหรับ profiling
local function profile(label, fn, ...)
    local startClock = os.clock()
    local startTime = os.time()
    
    local results = {fn(...)}
    
    local clockElapsed = os.clock() - startClock
    local wallElapsed = os.time() - startTime
    
    print(string.format("[PROFILE] %s: CPU=%.4fs, Wall=%ds",
        label, clockElapsed, wallElapsed))
    
    return table.unpack(results)
end

-- Test
local function expensiveComputation(n)
    local sum = 0
    for i = 1, n do
        sum = sum + math.sqrt(i)
    end
    return sum
end

profile("sqrt sum", expensiveComputation, 1000000)
```

---

## 20.13 Stack Inspection

```lua
-- ตัวอย่างที่ 29: Full stack inspector
local function fullStackInspect()
    local stack = {}
    local level = 1
    
    while true do
        local info = debug.getinfo(level, "nSlftu")
        if not info then break end
        
        local frame = {
            level = level,
            name = info.name or (info.what == "main" and "[main]" or "?"),
            what = info.what,
            source = info.short_src,
            currentLine = info.currentline,
            lineDefined = info.linedefined,
            locals = {},
            upvalues = {}
        }
        
        -- Collect locals
        local li = 1
        while true do
            local name, val = debug.getlocal(level, li)
            if not name then break end
            if name:sub(1,1) ~= "(" then
                frame.locals[#frame.locals + 1] = {
                    name = name,
                    value = val,
                    type = type(val)
                }
            end
            li = li + 1
        end
        
        -- Collect upvalues
        if info.func then
            local ui = 1
            while true do
                local name, val = debug.getupvalue(info.func, ui)
                if not name then break end
                frame.upvalues[#frame.upvalues + 1] = {
                    name = name,
                    value = val,
                    type = type(val)
                }
                ui = ui + 1
            end
        end
        
        stack[#stack + 1] = frame
        level = level + 1
    end
    
    return stack
end

local function printStack(stack)
    print("=== Stack Trace ===")
    for i = #stack, 1, -1 do
        local frame = stack[i]
        print(string.format("\n[%d] %s @ %s:%d",
            frame.level,
            frame.name,
            frame.source,
            frame.currentLine))
        
        if #frame.locals > 0 then
            print("  Locals:")
            for _, v in ipairs(frame.locals) do
                local valStr = type(v.value) == "table" and "{...}" or tostring(v.value)
                print(string.format("    %-15s (%s) = %s", v.name, v.type, valStr))
            end
        end
    end
    print("===================")
end

-- Test
local function inner(x, y)
    local sum = x + y
    local stack = fullStackInspect()
    printStack(stack)
    return sum
end

local function middle(a)
    local doubled = a * 2
    return inner(doubled, 10)
end

local function outer()
    local val = 5
    return middle(val)
end

outer()
```

---

## 20.14 Simple Debugger

```lua
-- ตัวอย่างที่ 30: Simple interactive debugger
local SimpleDebugger = {}

function SimpleDebugger.attach(target_fn)
    local breakpoints = {}
    local stepping = false
    local paused = false
    
    local function getLocals(level)
        local vars = {}
        local i = 1
        while true do
            local name, val = debug.getlocal(level, i)
            if not name then break end
            if name:sub(1,1) ~= "(" then
                vars[name] = val
            end
            i = i + 1
        end
        return vars
    end
    
    local function debugPrompt(info)
        print(string.format("\n[DEBUGGER] Paused at %s:%d",
            info.short_src, info.currentline))
        
        local vars = getLocals(3)
        if next(vars) then
            print("  Variables:")
            for k, v in pairs(vars) do
                print(string.format("    %s = %s", k, tostring(v)))
            end
        end
        
        print("  Commands: c=continue, n=next, q=quit")
        io.write("  > ")
        local cmd = io.read()
        
        if cmd == "q" then
            error("Debugger: quit")
        elseif cmd == "n" then
            stepping = true
        else
            stepping = false
        end
    end
    
    local function hook(event, line)
        if event == "line" then
            local info = debug.getinfo(2, "S")
            
            -- Check breakpoints
            local key = info.short_src .. ":" .. line
            if breakpoints[key] or stepping then
                paused = true
                debugPrompt(info)
                paused = false
            end
        end
    end
    
    return {
        addBreakpoint = function(source, line)
            breakpoints[source .. ":" .. line] = true
        end,
        run = function(...)
            debug.sethook(hook, "l")
            local ok, err = pcall(target_fn, ...)
            debug.sethook()
            if not ok and not err:match("Debugger: quit") then
                error(err, 2)
            end
        end
    }
end

-- Demonstration (จะไม่รัน interactive ใน non-interactive mode)
print("SimpleDebugger defined - use in interactive mode")
print("Example:")
print("  local dbg = SimpleDebugger.attach(myFunction)")
print("  dbg.addBreakpoint('myfile.lua', 10)")
print("  dbg.run(arg1, arg2)")
```

---

## 20.15 ตัวอย่างรวม: Debugging Tools Suite

```lua
-- ตัวอย่างที่ 31: Comprehensive debug utilities
local DebugUtils = {}

-- 1. Function timer decorator
function DebugUtils.timed(fn, name)
    name = name or debug.getinfo(fn, "n").name or "unknown"
    return function(...)
        local start = os.clock()
        local results = {fn(...)}
        local elapsed = os.clock() - start
        print(string.format("[TIMER] %s: %.6f ms", name, elapsed * 1000))
        return table.unpack(results)
    end
end

-- 2. Function call tracer
function DebugUtils.trace(fn, name)
    name = name or "fn"
    return function(...)
        local args = {...}
        local argStr = {}
        for i, a in ipairs(args) do
            argStr[i] = tostring(a)
        end
        io.write(string.format("[TRACE] %s(%s) -> ", name, table.concat(argStr, ", ")))
        local results = {fn(...)}
        local retStr = {}
        for i, r in ipairs(results) do
            retStr[i] = tostring(r)
        end
        print(table.concat(retStr, ", "))
        return table.unpack(results)
    end
end

-- 3. Memoization with cache statistics
function DebugUtils.memoize(fn, name)
    local cache = {}
    local hits = 0
    local misses = 0
    
    local memoized = function(...)
        local key = table.concat({...}, "\0")
        if cache[key] ~= nil then
            hits = hits + 1
            return cache[key]
        end
        misses = misses + 1
        local result = fn(...)
        cache[key] = result
        return result
    end
    
    memoized.stats = function()
        local total = hits + misses
        print(string.format("[MEMO] %s: %d hits, %d misses (%.1f%% hit rate)",
            name or "fn", hits, misses,
            total > 0 and (hits/total*100) or 0))
    end
    
    return memoized
end

-- 4. Type checker
function DebugUtils.typed(fn, signature)
    return function(...)
        local args = {...}
        for i, expectedType in ipairs(signature.params or {}) do
            local actualType = type(args[i])
            if expectedType ~= "any" and actualType ~= expectedType then
                error(string.format(
                    "Argument %d: expected %s, got %s",
                    i, expectedType, actualType), 2)
            end
        end
        
        local results = {fn(...)}
        
        if signature.returns then
            for i, expectedType in ipairs(signature.returns) do
                local actualType = type(results[i])
                if expectedType ~= "any" and actualType ~= expectedType then
                    error(string.format(
                        "Return value %d: expected %s, got %s",
                        i, expectedType, actualType), 2)
                end
            end
        end
        
        return table.unpack(results)
    end
end

-- Test
local function slowFib(n)
    if n <= 1 then return n end
    return slowFib(n-1) + slowFib(n-2)
end

-- Add memoization
slowFib = DebugUtils.memoize(slowFib, "fibonacci")

-- Add timing
local timedFib = DebugUtils.timed(slowFib, "fibonacci")

print(timedFib(20))  -- Fast with memoization
slowFib.stats()

-- Type checking
local typedAdd = DebugUtils.typed(
    function(a, b) return a + b end,
    {params = {"number", "number"}, returns = {"number"}}
)

print(typedAdd(3, 4))  -- 7

local ok, err = pcall(typedAdd, "hello", 4)
print("Type error:", err)
```

```lua
-- ตัวอย่างที่ 32: Error context collector
local function withContext(context, fn, ...)
    local ok, err = xpcall(fn, function(e)
        local tb = debug.traceback("", 2)
        return {
            message = tostring(e),
            traceback = tb,
            context = context,
            time = os.time(),
            memory = collectgarbage("count")
        }
    end, ...)
    
    if not ok then
        -- err เป็น table ที่มีข้อมูล error
        print(string.format("[ERROR] Context: %s", err.context))
        print(string.format("  Message: %s", err.message))
        print(string.format("  Memory: %.2fKB", err.memory))
        print("  Traceback:")
        for line in err.traceback:gmatch("[^\n]+") do
            print("  " .. line)
        end
        return nil, err
    end
    
    return ok
end

-- Test
local function problematicFunction(data)
    for _, item in ipairs(data) do
        if item > 90 then
            error("Value too large: " .. item)
        end
    end
    return "OK"
end

local result = withContext(
    "Processing user data",
    problematicFunction,
    {10, 50, 95, 20}
)
```

```lua
-- ตัวอย่างที่ 33: Debug mode toggle
local DebugMode = (function()
    local _enabled = false
    local _level = "DEBUG"
    local _watchers = {}
    
    return {
        enable = function(level)
            _enabled = true
            _level = level or "DEBUG"
            print("[Debug] Debug mode enabled at level: " .. _level)
        end,
        
        disable = function()
            _enabled = false
            print("[Debug] Debug mode disabled")
        end,
        
        isEnabled = function() return _enabled end,
        
        assert = function(condition, msg)
            if _enabled and not condition then
                error("[Debug] Assertion failed: " .. (msg or "unknown"), 2)
            end
        end,
        
        print = function(msg, ...)
            if _enabled then
                local info = debug.getinfo(2, "Sl")
                print(string.format("[DEBUG %s:%d] " .. msg,
                    info.short_src:match("[^/\\]+$") or "",
                    info.currentline,
                    ...))
            end
        end,
        
        watch = function(name, getVal)
            _watchers[name] = getVal
        end,
        
        dumpWatchers = function()
            if not _enabled then return end
            print("[Debug] Watched variables:")
            for name, getter in pairs(_watchers) do
                print(string.format("  %s = %s", name, tostring(getter())))
            end
        end
    }
end)()

-- Test
DebugMode.enable("DEBUG")

local x = 42
local data = {1, 2, 3}

DebugMode.watch("x", function() return x end)
DebugMode.watch("data_len", function() return #data end)

DebugMode.print("Starting process with x=%d", x)
DebugMode.assert(x > 0, "x must be positive")
DebugMode.dumpWatchers()

DebugMode.disable()
DebugMode.print("This should not print")
```

---

## 20.16 ตัวอย่างรวม: Production Error Handling

```lua
-- ตัวอย่างที่ 34: Production-ready error handling
local ErrorHandler = {}

local _errorLog = {}
local _maxErrors = 1000

function ErrorHandler.capture(fn, context)
    return xpcall(fn, function(err)
        local info = debug.getinfo(3, "nSl")
        local errorEntry = {
            message = tostring(err),
            traceback = debug.traceback("", 2),
            context = context or {},
            location = {
                source = info and info.short_src or "unknown",
                line = info and info.currentline or 0,
                name = info and info.name or "unknown"
            },
            timestamp = os.time(),
            memory = collectgarbage("count")
        }
        
        -- Add to log (circular buffer)
        if #_errorLog >= _maxErrors then
            table.remove(_errorLog, 1)
        end
        _errorLog[#_errorLog + 1] = errorEntry
        
        return errorEntry
    end)
end

function ErrorHandler.getLastErrors(n)
    n = n or 10
    local start = math.max(1, #_errorLog - n + 1)
    local result = {}
    for i = start, #_errorLog do
        result[#result + 1] = _errorLog[i]
    end
    return result
end

function ErrorHandler.clearLog()
    _errorLog = {}
end

-- Test
local function riskyFn(x)
    if x < 0 then
        error("Negative value: " .. x)
    end
    return math.sqrt(x)
end

-- Capture errors
local ok1, result1 = ErrorHandler.capture(
    function() return riskyFn(16) end,
    {operation = "sqrt", input = 16}
)
print("Success:", ok1, result1)

local ok2, result2 = ErrorHandler.capture(
    function() return riskyFn(-1) end,
    {operation = "sqrt", input = -1}
)
print("Failed:", ok2, type(result2))

if not ok2 and type(result2) == "table" then
    print("Error message:", result2.message)
    print("Error location:", result2.location.source)
end
```

```lua
-- ตัวอย่างที่ 35: เครื่องมือ debug ที่ใช้ครั้งเดียวจบ
local function quickDebug(value, label)
    local info = debug.getinfo(2, "Sl")
    local location = string.format("%s:%d",
        info.short_src:match("[^/\\]+$") or "?",
        info.currentline)
    
    label = label or location
    
    local function fmt(v, depth)
        depth = depth or 0
        if depth > 3 then return "..." end
        local t = type(v)
        if t == "table" then
            local parts = {}
            local count = 0
            for k, val in pairs(v) do
                count = count + 1
                if count > 5 then
                    parts[#parts + 1] = "..."
                    break
                end
                parts[#parts + 1] = tostring(k) .. "=" .. fmt(val, depth+1)
            end
            return "{" .. table.concat(parts, ", ") .. "}"
        elseif t == "string" then
            if #v > 50 then
                return '"' .. v:sub(1,47) .. '..."'
            end
            return '"' .. v .. '"'
        else
            return tostring(v)
        end
    end
    
    print(string.format("[?] %s: %s (%s)", label, fmt(value), type(value)))
    return value  -- passthrough for inline debugging
end

-- ใช้งาน:
local x = quickDebug(42, "x value")
local y = quickDebug({1,2,3}, "array")
local z = quickDebug("hello world", "message")
```

---

## 20.17 Visual Debugging Output

```lua
-- ตัวอย่างที่ 36: ASCII visualization สำหรับ data structures
local Visualizer = {}

function Visualizer.box(title, content, width)
    width = width or 40
    local lines = {}
    local border = string.rep("─", width - 2)
    
    lines[#lines+1] = "┌" .. border .. "┐"
    
    -- Title
    local titlePad = math.max(0, width - 2 - #title)
    local leftPad = math.floor(titlePad / 2)
    local rightPad = titlePad - leftPad
    lines[#lines+1] = "│" .. string.rep(" ", leftPad) ..
        title .. string.rep(" ", rightPad) .. "│"
    
    lines[#lines+1] = "├" .. border .. "┤"
    
    -- Content
    if type(content) == "string" then
        for line in (content.."\n"):gmatch("(.-)\n") do
            local pad = math.max(0, width - 2 - #line)
            lines[#lines+1] = "│ " .. line .. string.rep(" ", pad - 1) .. "│"
        end
    elseif type(content) == "table" then
        for _, line in ipairs(content) do
            local pad = math.max(0, width - 2 - #line)
            lines[#lines+1] = "│ " .. line .. string.rep(" ", pad - 1) .. "│"
        end
    end
    
    lines[#lines+1] = "└" .. border .. "┘"
    return table.concat(lines, "\n")
end

function Visualizer.progressBar(current, total, width, label)
    width = width or 30
    local progress = current / total
    local filled = math.floor(progress * width)
    local empty = width - filled
    
    local bar = "[" .. string.rep("█", filled) .. string.rep("░", empty) .. "]"
    local pct = string.format(" %3.0f%%", progress * 100)
    
    if label then
        return label .. " " .. bar .. pct
    end
    return bar .. pct
end

function Visualizer.table(headers, rows, colWidths)
    local lines = {}
    
    -- Calculate column widths
    local widths = colWidths or {}
    for i, h in ipairs(headers) do
        widths[i] = math.max(widths[i] or 0, #tostring(h))
    end
    for _, row in ipairs(rows) do
        for i, cell in ipairs(row) do
            widths[i] = math.max(widths[i] or 0, #tostring(cell))
        end
    end
    
    local function makeSep(left, mid, right, fill)
        local parts = {}
        for _, w in ipairs(widths) do
            parts[#parts+1] = string.rep(fill, w + 2)
        end
        return left .. table.concat(parts, mid) .. right
    end
    
    local function makeRow(cells, left, mid, right)
        local parts = {}
        for i, cell in ipairs(cells) do
            local s = tostring(cell)
            local pad = widths[i] - #s
            parts[#parts+1] = " " .. s .. string.rep(" ", pad) .. " "
        end
        return left .. table.concat(parts, mid) .. right
    end
    
    lines[#lines+1] = makeSep("┌", "┬", "┐", "─")
    lines[#lines+1] = makeRow(headers, "│", "│", "│")
    lines[#lines+1] = makeSep("├", "┼", "┤", "─")
    for _, row in ipairs(rows) do
        lines[#lines+1] = makeRow(row, "│", "│", "│")
    end
    lines[#lines+1] = makeSep("└", "┴", "┘", "─")
    
    return table.concat(lines, "\n")
end

-- Test
print(Visualizer.box("Debug Info", {
    "Function: processData",
    "Memory: 128.5 KB",
    "Runtime: 0.0023s",
    "Items processed: 1000"
}))

print()
print(Visualizer.progressBar(75, 100, 30, "Processing"))
print(Visualizer.progressBar(30, 100, 30, "Loading   "))

print()
print(Visualizer.table(
    {"Variable", "Type", "Value"},
    {
        {"x", "number", "42"},
        {"name", "string", "Alice"},
        {"data", "table", "{...}"},
        {"active", "boolean", "true"}
    }
))
```

---

## 20.18 ตัวอย่าง: Debug สำหรับ Performance Issues

```lua
-- ตัวอย่างที่ 37: Hot path profiler
local HotPathProfiler = {}
HotPathProfiler._data = {}

function HotPathProfiler.start()
    HotPathProfiler._data = {}
    debug.sethook(function(event, line)
        local info = debug.getinfo(2, "Sl")
        if not info then return end
        
        local key = string.format("%s:%d",
            info.short_src:match("[^/\\]+$") or "?", line)
        HotPathProfiler._data[key] = (HotPathProfiler._data[key] or 0) + 1
    end, "l")
end

function HotPathProfiler.stop(topN)
    debug.sethook()
    topN = topN or 10
    
    local lines = {}
    for key, count in pairs(HotPathProfiler._data) do
        lines[#lines + 1] = {key = key, count = count}
    end
    table.sort(lines, function(a, b) return a.count > b.count end)
    
    print(string.format("Top %d hot lines:", topN))
    for i = 1, math.min(topN, #lines) do
        print(string.format("  %5d ×  %s", lines[i].count, lines[i].key))
    end
    
    return lines
end

-- Profile a computation
HotPathProfiler.start()

local function mergeSort(arr)
    if #arr <= 1 then return arr end
    local mid = math.floor(#arr / 2)
    local left = {}
    local right = {}
    for i = 1, mid do left[#left+1] = arr[i] end
    for i = mid+1, #arr do right[#right+1] = arr[i] end
    
    left = mergeSort(left)
    right = mergeSort(right)
    
    local result = {}
    local i, j = 1, 1
    while i <= #left and j <= #right do
        if left[i] <= right[j] then
            result[#result+1] = left[i]; i = i + 1
        else
            result[#result+1] = right[j]; j = j + 1
        end
    end
    while i <= #left do result[#result+1] = left[i]; i = i + 1 end
    while j <= #right do result[#result+1] = right[j]; j = j + 1 end
    
    return result
end

local arr = {}
math.randomseed(42)
for i = 1, 100 do arr[i] = math.random(1, 1000) end
local sorted = mergeSort(arr)

HotPathProfiler.stop(5)
print("\nFirst 5 sorted:", table.concat({table.unpack(sorted, 1, 5)}, ", "))
```

```lua
-- ตัวอย่างที่ 38: Memory allocation tracker
local AllocTracker = {}
AllocTracker._baseline = 0
AllocTracker._checkpoints = {}

function AllocTracker.baseline()
    collectgarbage("collect")
    AllocTracker._baseline = collectgarbage("count")
    AllocTracker._checkpoints = {}
    print(string.format("[Alloc] Baseline: %.2f KB", AllocTracker._baseline))
end

function AllocTracker.checkpoint(label)
    local current = collectgarbage("count")
    local diff = current - AllocTracker._baseline
    AllocTracker._checkpoints[#AllocTracker._checkpoints + 1] = {
        label = label,
        memory = current,
        diff = diff
    }
    print(string.format("[Alloc] %s: %.2f KB (+%.2f KB)",
        label, current, diff))
end

function AllocTracker.report()
    print("\n[Alloc] Full Report:")
    print(string.format("  %-30s %10s %10s", "Checkpoint", "Memory", "Delta"))
    print("  " .. string.rep("-", 52))
    for _, cp in ipairs(AllocTracker._checkpoints) do
        print(string.format("  %-30s %8.2fKB %+8.2fKB",
            cp.label, cp.memory, cp.diff))
    end
end

-- Test
AllocTracker.baseline()

local t1 = {}
for i = 1, 1000 do t1[i] = i end
AllocTracker.checkpoint("After 1000 ints")

local t2 = {}
for i = 1, 1000 do t2[i] = string.rep("x", 100) end
AllocTracker.checkpoint("After 1000 strings")

collectgarbage("collect")
AllocTracker.checkpoint("After GC")

t1, t2 = nil, nil
collectgarbage("collect")
AllocTracker.checkpoint("After nil + GC")

AllocTracker.report()
```

---

## 20.19 ตัวอย่างรวม: Debug Dashboard

```lua
-- ตัวอย่างที่ 39: Debug dashboard
local function createDashboard()
    local metrics = {
        memory = {},
        calls = {},
        errors = 0,
        startTime = os.clock()
    }
    
    local function snapshot()
        return {
            time = os.clock() - metrics.startTime,
            memory = collectgarbage("count"),
            errors = metrics.errors
        }
    end
    
    local function render()
        local snap = snapshot()
        local lines = {}
        
        lines[#lines+1] = "╔══════════════════════════════╗"
        lines[#lines+1] = "║      Debug Dashboard         ║"
        lines[#lines+1] = "╠══════════════════════════════╣"
        lines[#lines+1] = string.format("║ Runtime: %-19.3fs ║", snap.time)
        lines[#lines+1] = string.format("║ Memory:  %-18.1fKB ║", snap.memory)
        lines[#lines+1] = string.format("║ Errors:  %-20d ║", snap.errors)
        lines[#lines+1] = "╠══════════════════════════════╣"
        
        -- Recent memory samples
        if #metrics.memory > 0 then
            lines[#lines+1] = "║ Memory History:              ║"
            local recent = {}
            local start = math.max(1, #metrics.memory - 4)
            for i = start, #metrics.memory do
                recent[#recent+1] = string.format("%.0f", metrics.memory[i])
            end
            local hist = table.concat(recent, "→")
            lines[#lines+1] = string.format("║  %-28s ║", hist)
        end
        
        lines[#lines+1] = "╚══════════════════════════════╝"
        return table.concat(lines, "\n")
    end
    
    return {
        recordMemory = function()
            metrics.memory[#metrics.memory+1] = collectgarbage("count")
        end,
        recordError = function()
            metrics.errors = metrics.errors + 1
        end,
        render = render,
        snapshot = snapshot
    }
end

local dash = createDashboard()

-- Simulate some activity
for i = 1, 5 do
    local t = {}
    for j = 1, i * 100 do t[j] = j end
    dash.recordMemory()
end

print(dash.render())
```

```lua
-- ตัวอย่างที่ 40: watchdog timer
local function createWatchdog(timeout_seconds)
    local _startTime = nil
    local _timeout = timeout_seconds
    local _enabled = false
    
    local function checkTimeout()
        if not _enabled or not _startTime then return end
        local elapsed = os.clock() - _startTime
        if elapsed > _timeout then
            error(string.format("Watchdog: timeout after %.2fs (limit: %.2fs)",
                elapsed, _timeout), 2)
        end
    end
    
    local hook = nil
    
    return {
        start = function()
            _startTime = os.clock()
            _enabled = true
            hook = function(event)
                checkTimeout()
            end
            debug.sethook(hook, "", 1000)  -- check every 1000 instructions
        end,
        
        stop = function()
            _enabled = false
            _startTime = nil
            debug.sethook()
        end,
        
        reset = function()
            _startTime = os.clock()
        end,
        
        elapsed = function()
            if _startTime then
                return os.clock() - _startTime
            end
            return 0
        end
    }
end

local wd = createWatchdog(1.0)  -- 1 second timeout

wd.start()
-- Simulate work within timeout
local sum = 0
for i = 1, 100000 do sum = sum + i end
print("Sum:", sum)
print(string.format("Elapsed: %.4fs", wd.elapsed()))
wd.stop()
print("Watchdog stopped")
```

---

## แบบฝึกหัด

### ระดับพื้นฐาน

1. **Enhanced traceback**: เขียนฟังก์ชัน `getTraceback()` ที่:
   - แสดง call stack ทั้งหมด
   - แสดง local variables ของแต่ละ frame
   - กรอง internal Lua frames ออก
   - Format output ให้อ่านง่าย

2. **Function profiler**: เขียน decorator `profile(fn, name)` ที่:
   - วัดเวลาทุก call
   - นับจำนวน calls
   - คำนวณ min/max/average time
   - มี method `report()` แสดงสถิติ

3. **Memory tracker**: เขียน class ที่:
   - Track memory ก่อนและหลัง operations
   - แสดง memory leak warnings
   - มี baseline และ checkpoint
   - สร้าง report แบบ text

### ระดับกลาง

4. **Logger with file rotation**: เขียน Logger ที่:
   - มี levels: DEBUG, INFO, WARN, ERROR, FATAL
   - เขียนไปยังไฟล์
   - Rotate ไฟล์เมื่อ size เกิน limit
   - เก็บ backup ไว้หลายไฟล์
   - Support multiple handlers

5. **Assertion library**: เขียน assertion library ที่:
   - `assert.equal(a, b)`, `assert.notEqual(a, b)`
   - `assert.isType(val, type)`, `assert.isNil(val)`
   - `assert.throws(fn, errorMsg)` - ตรวจว่า fn throw error
   - `assert.doesNotThrow(fn)` - ตรวจว่าไม่ throw
   - แสดง helpful error messages พร้อม stack trace

6. **Performance regression detector**: เขียน tool ที่:
   - Benchmark functions หลายครั้ง
   - เปรียบเทียบผลกับ baseline
   - แจ้งเตือนเมื่อ performance แย่ลงเกิน threshold
   - สร้าง report แบบ table

### ระดับสูง

7. **Interactive REPL debugger**: สร้าง debugger ที่:
   - หยุดที่ breakpoints
   - แสดง local variables และ upvalues
   - รับคำสั่ง: print, set, step, continue, quit
   - Support conditional breakpoints
   - แสดง source code รอบๆ breakpoint

8. **Execution recorder**: สร้าง tool ที่:
   - Record การ execute ทุก line
   - Record ค่าของ variables ในแต่ละ step
   - Play back การ execute ย้อนหลังได้
   - Export เป็น JSON

9. **Production monitoring agent**: สร้าง monitoring system ที่:
   - Track error rate ตามเวลา
   - วัด memory usage trends
   - Detect memory leaks
   - Alert เมื่อ metrics เกิน threshold
   - มี dashboard แบบ text

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Debugging ใน Lua:

| เครื่องมือ | การใช้งาน |
|-----------|-----------|
| `debug.traceback()` | แสดง call stack |
| `debug.getinfo()` | ข้อมูล function/level |
| `debug.sethook()` | hook สำหรับ profiling |
| `debug.getlocal()` | อ่าน local variables |
| `debug.setlocal()` | แก้ไข local variables |
| `debug.getupvalue()` | อ่าน upvalues |
| `debug.setupvalue()` | แก้ไข upvalues |
| `debug.getmetatable()` | bypass __metatable |
| `collectgarbage()` | จัดการ GC และ memory |
| `os.clock()` | วัดเวลา CPU |

หลักการ debugging ที่ดี:
1. เริ่มจากข้อความ error และ traceback
2. ใช้ print debugging สำหรับปัญหาเล็กๆ
3. ใช้ assertions เพื่อ validate assumptions
4. Profile ก่อน optimize
5. วัด memory เพื่อหา leaks
6. เขียน test เพื่อป้องกัน regression
