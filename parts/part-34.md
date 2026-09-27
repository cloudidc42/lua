# บทที่ 34: Performance Optimization

## บทนำ

การ optimize performance ใน Lua เป็นทักษะสำคัญสำหรับแอปพลิเคชันที่ต้องการความเร็วสูง เช่น เกม, server software, หรือ embedded systems บทนี้จะครอบคลุมเทคนิคการ profiling, การหาจุดช้า และวิธีปรับปรุงโค้ดให้เร็วขึ้น

---

## 34.1 การวัดเวลาด้วย os.clock()

`os.clock()` วัด CPU time ที่โปรแกรมใช้ไป (ไม่รวม sleep หรือ I/O wait)

### ตัวอย่างที่ 1: Timer พื้นฐาน

```lua
-- example_01_timer.lua

-- วัดเวลาแบบง่าย
local function measureTime(func, ...)
    local start = os.clock()
    local result = {func(...)}
    local elapsed = os.clock() - start
    return elapsed, table.unpack(result)
end

-- ฟังก์ชันที่ต้องการวัด
local function sumNumbers(n)
    local total = 0
    for i = 1, n do
        total = total + i
    end
    return total
end

local elapsed, result = measureTime(sumNumbers, 1000000)
print(string.format("Sum = %d, Time = %.6f seconds", result, elapsed))
```

### ตัวอย่างที่ 2: Benchmark Framework

```lua
-- example_02_benchmark.lua

local Benchmark = {}

-- รัน function หลายครั้งและวัดเวลาเฉลี่ย
function Benchmark.run(name, func, iterations)
    iterations = iterations or 1000
    
    -- Warmup
    for i = 1, math.min(10, iterations // 10) do
        func()
    end
    
    -- Actual measurement
    local start = os.clock()
    for i = 1, iterations do
        func()
    end
    local elapsed = os.clock() - start
    
    local perIteration = elapsed / iterations
    print(string.format(
        "%-30s %10.6f sec total | %12.9f sec/iter | %10.0f iter/sec",
        name,
        elapsed,
        perIteration,
        1 / perIteration
    ))
    
    return elapsed, perIteration
end

-- เปรียบเทียบหลาย implementations
function Benchmark.compare(tests, iterations)
    print(string.rep("-", 80))
    print(string.format("%-30s %11s %17s %15s", 
        "Name", "Total Time", "Per Iteration", "Iterations/sec"))
    print(string.rep("-", 80))
    
    local results = {}
    for _, test in ipairs(tests) do
        local elapsed, perIter = Benchmark.run(
            test.name, test.func, iterations
        )
        table.insert(results, {
            name = test.name,
            elapsed = elapsed,
            perIter = perIter
        })
    end
    
    print(string.rep("-", 80))
    
    -- หา fastest
    table.sort(results, function(a, b)
        return a.elapsed < b.elapsed
    end)
    print(string.format("Fastest: %s", results[1].name))
    
    return results
end

-- ตัวอย่างการใช้
Benchmark.compare({
    {
        name = "string concat with ..",
        func = function()
            local s = ""
            for i = 1, 100 do
                s = s .. "x"
            end
        end
    },
    {
        name = "table.concat",
        func = function()
            local t = {}
            for i = 1, 100 do
                t[i] = "x"
            end
            local s = table.concat(t)
        end
    }
}, 10000)
```

---

## 34.2 String Concatenation

การ concatenate string ด้วย `..` ใน loop เป็น anti-pattern ที่พบบ่อย

### ตัวอย่างที่ 3: เปรียบเทียบ String Concatenation

```lua
-- example_03_string_concat.lua

local function measureTime(func, iterations)
    local start = os.clock()
    for i = 1, iterations do
        func()
    end
    return os.clock() - start
end

local N = 1000  -- จำนวน elements ใน string

-- วิธีที่ 1: .. operator (ช้า!)
local function concatOperator()
    local s = ""
    for i = 1, N do
        s = s .. tostring(i) .. ","
    end
    return s
end

-- วิธีที่ 2: table.concat (เร็ว!)
local function concatTable()
    local parts = {}
    for i = 1, N do
        parts[#parts + 1] = tostring(i)
    end
    return table.concat(parts, ",")
end

-- วิธีที่ 3: pre-allocated table (เร็วที่สุด)
local function concatTablePrealloc()
    local parts = {}
    for i = 1, N do
        parts[i] = tostring(i)  -- ใช้ index โดยตรงแทน #parts+1
    end
    return table.concat(parts, ",")
end

local iterations = 1000

local t1 = measureTime(concatOperator, iterations)
local t2 = measureTime(concatTable, iterations)
local t3 = measureTime(concatTablePrealloc, iterations)

print(string.format(".. operator:        %.4f sec", t1))
print(string.format("table.concat:       %.4f sec", t2))
print(string.format("pre-alloc table:    %.4f sec", t3))
print(string.format("Speedup (2 vs 1):   %.1fx", t1/t2))
print(string.format("Speedup (3 vs 1):   %.1fx", t1/t3))
```

### ตัวอย่างที่ 4: String Builder Pattern

```lua
-- example_04_string_builder.lua

-- StringBuilder class
local StringBuilder = {}
StringBuilder.__index = StringBuilder

function StringBuilder.new()
    return setmetatable({
        _parts = {},
        _length = 0
    }, StringBuilder)
end

function StringBuilder:append(s)
    s = tostring(s)
    self._parts[#self._parts + 1] = s
    self._length = self._length + #s
    return self  -- สำหรับ method chaining
end

function StringBuilder:appendLine(s)
    return self:append(s):append("\n")
end

function StringBuilder:toString()
    return table.concat(self._parts)
end

function StringBuilder:length()
    return self._length
end

function StringBuilder:clear()
    self._parts = {}
    self._length = 0
    return self
end

-- ใช้งาน
local sb = StringBuilder.new()

sb:append("Hello"):append(", "):append("World"):appendLine("!")
sb:appendLine("สวัสดีชาว Lua")

for i = 1, 5 do
    sb:append("Line "):append(i):appendLine()
end

print(sb:toString())
print("Length:", sb:length())
```

---

## 34.3 Global vs Local Variables

### ตัวอย่างที่ 5: Cache Global ลงใน Local

```lua
-- example_05_local_cache.lua

-- ทุกครั้งที่เข้าถึง global variable Lua ต้องค้นหาใน _ENV table
-- การ cache ลงใน local variable จะเร็วกว่ามาก

local function measureTime(func, n)
    local start = os.clock()
    func(n)
    return os.clock() - start
end

-- ช้า: ใช้ math.sin โดยตรง
local function slowVersion(n)
    local sum = 0
    for i = 1, n do
        sum = sum + math.sin(i)
    end
    return sum
end

-- เร็ว: cache math.sin เป็น local
local function fastVersion(n)
    local sin = math.sin  -- cache ไว้ก่อน
    local sum = 0
    for i = 1, n do
        sum = sum + sin(i)
    end
    return sum
end

-- เร็วยิ่งขึ้น: cache ทุก function ที่ใช้
local function fastestVersion(n)
    -- cache locals
    local sin = math.sin
    local floor = math.floor
    local sum = 0
    for i = 1, n do
        sum = sum + sin(floor(i * 0.1))
    end
    return sum
end

local N = 1000000
print(string.format("Slow (global):   %.4f sec", measureTime(slowVersion, N)))
print(string.format("Fast (local):    %.4f sec", measureTime(fastVersion, N)))
print(string.format("Fastest:         %.4f sec", measureTime(fastestVersion, N)))
```

### ตัวอย่างที่ 6: Local ใน Module

```lua
-- example_06_module_locals.lua

-- ตัวอย่าง: module ที่ optimize ด้วย local variables

-- ไม่ดี: เข้าถึง global ตลอด
local BadModule = {}

function BadModule.process(data)
    local result = {}
    for i = 1, #data do
        -- ค้นหา math.sqrt ใน _ENV ทุกครั้ง
        result[i] = math.sqrt(math.abs(data[i])) * math.pi
    end
    return result
end

-- ดี: cache globals เป็น module-level locals
local sqrt = math.sqrt
local abs = math.abs
local pi = math.pi

local GoodModule = {}

function GoodModule.process(data)
    local result = {}
    for i = 1, #data do
        -- เข้าถึง local variable เร็วกว่า
        result[i] = sqrt(abs(data[i])) * pi
    end
    return result
end

-- ทดสอบ
local testData = {}
for i = 1, 10000 do
    testData[i] = (math.random() - 0.5) * 1000
end

local function benchmark(name, func, data)
    local start = os.clock()
    for i = 1, 100 do
        func(data)
    end
    local elapsed = os.clock() - start
    print(string.format("%-20s %.4f sec", name, elapsed))
end

benchmark("Bad module:", BadModule.process, testData)
benchmark("Good module:", GoodModule.process, testData)
```

---

## 34.4 Table Pre-allocation

### ตัวอย่างที่ 7: Pre-allocate Tables

```lua
-- example_07_prealloc.lua

-- Lua จัดการหน่วยความจำของ table แบบ dynamic
-- การ pre-allocate จะลด re-allocation

local function measureTime(func, n, iter)
    local start = os.clock()
    for i = 1, iter do
        func(n)
    end
    return os.clock() - start
end

-- วิธีที่ 1: ไม่ pre-allocate (ช้า)
local function noPrealloc(n)
    local t = {}
    for i = 1, n do
        t[#t + 1] = i
    end
    return t
end

-- วิธีที่ 2: pre-allocate ด้วย table.move หรือ index โดยตรง
local function withIndex(n)
    local t = {}
    for i = 1, n do
        t[i] = i  -- กำหนด index โดยตรง
    end
    return t
end

-- วิธีที่ 3: reuse table (ดีที่สุดถ้าเป็นไปได้)
local sharedTable = {}
local function reuseTable(n)
    -- clear แทนสร้างใหม่
    for i = 1, #sharedTable do
        sharedTable[i] = nil
    end
    for i = 1, n do
        sharedTable[i] = i
    end
    return sharedTable
end

local N = 1000
local ITER = 10000

print(string.format("No prealloc:     %.4f sec", measureTime(noPrealloc, N, ITER)))
print(string.format("With index:      %.4f sec", measureTime(withIndex, N, ITER)))
print(string.format("Reuse table:     %.4f sec", measureTime(reuseTable, N, ITER)))
```

### ตัวอย่างที่ 8: Object Pool Pattern

```lua
-- example_08_object_pool.lua

-- Object pool: reuse objects แทนสร้างใหม่

local ObjectPool = {}
ObjectPool.__index = ObjectPool

function ObjectPool.new(factory, maxSize)
    return setmetatable({
        _factory = factory,
        _pool = {},
        _size = 0,
        _maxSize = maxSize or 100,
        -- สถิติ
        _hits = 0,
        _misses = 0
    }, ObjectPool)
end

function ObjectPool:acquire()
    if self._size > 0 then
        self._hits = self._hits + 1
        local obj = self._pool[self._size]
        self._pool[self._size] = nil
        self._size = self._size - 1
        return obj
    else
        self._misses = self._misses + 1
        return self._factory()
    end
end

function ObjectPool:release(obj)
    if self._size < self._maxSize then
        -- reset object ก่อน return ไป pool
        if obj.reset then
            obj:reset()
        end
        self._size = self._size + 1
        self._pool[self._size] = obj
        return true
    end
    return false
end

function ObjectPool:stats()
    local total = self._hits + self._misses
    return {
        hits = self._hits,
        misses = self._misses,
        hitRate = total > 0 and self._hits / total or 0,
        poolSize = self._size
    }
end

-- ตัวอย่าง: Pool ของ Particle objects
local function createParticle()
    return {
        x = 0, y = 0,
        vx = 0, vy = 0,
        life = 0,
        active = false,
        reset = function(self)
            self.x = 0
            self.y = 0
            self.vx = 0
            self.vy = 0
            self.life = 0
            self.active = false
        end
    }
end

local particlePool = ObjectPool.new(createParticle, 1000)

-- simulation
local activeParticles = {}

local function spawnParticle(x, y, vx, vy)
    local p = particlePool:acquire()
    p.x = x
    p.y = y
    p.vx = vx
    p.vy = vy
    p.life = 100
    p.active = true
    table.insert(activeParticles, p)
end

local function updateParticles(dt)
    local toRemove = {}
    for i, p in ipairs(activeParticles) do
        p.x = p.x + p.vx * dt
        p.y = p.y + p.vy * dt
        p.life = p.life - dt * 10
        
        if p.life <= 0 then
            p.active = false
            table.insert(toRemove, i)
        end
    end
    
    -- remove dead particles (ต้องทำย้อนกลับ)
    for i = #toRemove, 1, -1 do
        local p = table.remove(activeParticles, toRemove[i])
        particlePool:release(p)
    end
end

-- benchmark
local start = os.clock()
for frame = 1, 1000 do
    -- spawn particles
    for i = 1, 10 do
        spawnParticle(
            math.random(100), math.random(100),
            (math.random() - 0.5) * 10,
            (math.random() - 0.5) * 10
        )
    end
    
    -- update
    updateParticles(0.016)  -- 60fps
end

local elapsed = os.clock() - start
local stats = particlePool:stats()

print(string.format("Simulation: %.4f sec", elapsed))
print(string.format("Pool hits: %d (%.1f%%)", 
    stats.hits, stats.hitRate * 100))
print(string.format("Pool misses: %d", stats.misses))
print(string.format("Pool size: %d", stats.poolSize))
```

---

## 34.5 Avoid Unnecessary Function Calls

### ตัวอย่างที่ 9: Inline vs Function Call

```lua
-- example_09_inline.lua

local function measureTime(func, n)
    local start = os.clock()
    func(n)
    return os.clock() - start
end

-- ช้า: เรียก function ทุก iteration
local function square(x)
    return x * x
end

local function withFunctionCall(n)
    local sum = 0
    for i = 1, n do
        sum = sum + square(i)
    end
    return sum
end

-- เร็ว: inline calculation
local function withInline(n)
    local sum = 0
    for i = 1, n do
        sum = sum + i * i  -- inline แทน function call
    end
    return sum
end

local N = 5000000
print(string.format("With function call: %.4f sec", measureTime(withFunctionCall, N)))
print(string.format("Inlined:            %.4f sec", measureTime(withInline, N)))
```

### ตัวอย่างที่ 10: ลด Method Lookup

```lua
-- example_10_method_lookup.lua

-- ช้า: ค้นหา method ทุก iteration
local function slowLoop(obj, n)
    for i = 1, n do
        obj:process(i)
    end
end

-- เร็ว: cache method reference
local function fastLoop(obj, n)
    local process = obj.process  -- cache method
    for i = 1, n do
        process(obj, i)  -- เรียกโดยตรง
    end
end

local myObj = {
    _count = 0,
    process = function(self, value)
        self._count = self._count + value
    end
}

local function measureObjTime(func, n)
    myObj._count = 0
    local start = os.clock()
    func(myObj, n)
    return os.clock() - start
end

local N = 1000000
print(string.format("Slow (method lookup): %.4f sec", measureObjTime(slowLoop, N)))
print(string.format("Fast (cached method): %.4f sec", measureObjTime(fastLoop, N)))
```

---

## 34.6 Tail Call Optimization

### ตัวอย่างที่ 11: Tail Calls

```lua
-- example_11_tailcall.lua

-- Lua รองรับ Proper Tail Call (PTC)
-- ทำให้ recursive function ไม่ stack overflow

-- Regular recursion (จะ stack overflow สำหรับ n ใหญ่)
local function sumRegular(n, acc)
    acc = acc or 0
    if n == 0 then
        return acc
    end
    return sumRegular(n - 1, acc + n)  -- tail call!
end

-- Iterative version (เร็วที่สุด)
local function sumIterative(n)
    local total = 0
    for i = 1, n do
        total = total + i
    end
    return total
end

-- ทดสอบ
print("Tail recursive (n=100000):", sumRegular(100000))
print("Iterative (n=100000):", sumIterative(100000))

-- Fibonacci แบบ tail recursive
local function fibTail(n, a, b)
    a = a or 0
    b = b or 1
    if n == 0 then return a end
    if n == 1 then return b end
    return fibTail(n - 1, b, a + b)  -- tail call
end

-- Fibonacci แบบ iterative
local function fibIterative(n)
    if n <= 1 then return n end
    local a, b = 0, 1
    for i = 2, n do
        a, b = b, a + b
    end
    return b
end

print("Fibonacci(40) tail:", fibTail(40))
print("Fibonacci(40) iter:", fibIterative(40))

-- วัดเวลา
local function bench(name, func, n)
    local start = os.clock()
    for i = 1, 10000 do
        func(n)
    end
    print(string.format("%-25s %.4f sec", name, os.clock() - start))
end

bench("Fibonacci tail (n=30):", fibTail, 30)
bench("Fibonacci iter (n=30):", fibIterative, 30)
```

---

## 34.7 Integer vs Float Performance

### ตัวอย่างที่ 12: Integer Operations

```lua
-- example_12_integer_float.lua
-- Lua 5.3+ แยก integer และ float subtype

-- ตรวจสอบ type
print(math.type(1))      -- "integer"
print(math.type(1.0))    -- "float"
print(math.type(1/1))    -- "float" (division always returns float)
print(math.type(1//1))   -- "integer" (floor division)

local function measureTime(func, n)
    local start = os.clock()
    func(n)
    return os.clock() - start
end

-- Integer operations
local function intOps(n)
    local sum = 0
    for i = 1, n do
        sum = sum + i  -- integer + integer = integer
    end
    return sum
end

-- Float operations
local function floatOps(n)
    local sum = 0.0
    for i = 1, n do
        sum = sum + i  -- float + integer = float
    end
    return sum
end

-- Mixed operations
local function mixedOps(n)
    local sum = 0
    local factor = 1.5  -- float
    for i = 1, n do
        sum = sum + i * factor  -- promotes to float
    end
    return sum
end

local N = 10000000
print(string.format("Integer ops: %.4f sec", measureTime(intOps, N)))
print(string.format("Float ops:   %.4f sec", measureTime(floatOps, N)))
print(string.format("Mixed ops:   %.4f sec", measureTime(mixedOps, N)))

-- ใช้ integer division แทน float division
local function fastDiv(n)
    local result = 0
    for i = 1, n do
        result = result + i // 2  -- integer division
    end
    return result
end

local function slowDiv(n)
    local result = 0
    for i = 1, n do
        result = result + i / 2  -- float division
    end
    return result
end

print(string.format("\nInteger div: %.4f sec", measureTime(fastDiv, N)))
print(string.format("Float div:   %.4f sec", measureTime(slowDiv, N)))
```

---

## 34.8 Benchmarking Framework

### ตัวอย่างที่ 13: Framework สมบูรณ์

```lua
-- example_13_benchmark_framework.lua

local BenchmarkSuite = {}
BenchmarkSuite.__index = BenchmarkSuite

function BenchmarkSuite.new(name)
    return setmetatable({
        name = name,
        benchmarks = {},
        results = {}
    }, BenchmarkSuite)
end

function BenchmarkSuite:add(name, func, setup, teardown)
    table.insert(self.benchmarks, {
        name = name,
        func = func,
        setup = setup,
        teardown = teardown
    })
end

function BenchmarkSuite:run(iterations, warmupIter)
    iterations = iterations or 1000
    warmupIter = warmupIter or math.max(10, iterations // 10)
    
    print(string.format("\n=== %s ===", self.name))
    print(string.format("Iterations: %d, Warmup: %d\n", iterations, warmupIter))
    print(string.format("%-35s %10s %12s %12s %10s",
        "Name", "Total(s)", "Mean(ms)", "Min(ms)", "Max(ms)"))
    print(string.rep("-", 82))
    
    self.results = {}
    
    for _, bench in ipairs(self.benchmarks) do
        -- setup
        local ctx = {}
        if bench.setup then
            ctx = bench.setup() or ctx
        end
        
        -- warmup
        for i = 1, warmupIter do
            bench.func(ctx)
        end
        
        -- measure
        local times = {}
        for i = 1, iterations do
            local s = os.clock()
            bench.func(ctx)
            times[i] = os.clock() - s
        end
        
        -- teardown
        if bench.teardown then
            bench.teardown(ctx)
        end
        
        -- statistics
        local total = 0
        local min = math.huge
        local max = -math.huge
        
        for _, t in ipairs(times) do
            total = total + t
            if t < min then min = t end
            if t > max then max = t end
        end
        
        local mean = total / iterations
        
        print(string.format("%-35s %10.4f %12.6f %12.6f %10.6f",
            bench.name,
            total,
            mean * 1000,
            min * 1000,
            max * 1000))
        
        table.insert(self.results, {
            name = bench.name,
            total = total,
            mean = mean,
            min = min,
            max = max
        })
    end
    
    print(string.rep("-", 82))
    
    -- find fastest
    table.sort(self.results, function(a, b) return a.mean < b.mean end)
    print(string.format("\nFastest: %s", self.results[1].name))
    
    if #self.results > 1 then
        local baseline = self.results[1].mean
        print("\nRelative performance:")
        for _, r in ipairs(self.results) do
            print(string.format("  %-35s %.2fx", 
                r.name, r.mean / baseline))
        end
    end
end

-- ตัวอย่างการใช้
local suite = BenchmarkSuite.new("String Concatenation Benchmark")

suite:add(".. operator", function()
    local s = ""
    for i = 1, 100 do
        s = s .. "x"
    end
end)

suite:add("table.concat", function()
    local t = {}
    for i = 1, 100 do
        t[i] = "x"
    end
    table.concat(t)
end)

suite:add("string.format", function()
    local parts = {}
    for i = 1, 10 do
        parts[i] = string.format("item%d", i)
    end
    table.concat(parts, ",")
end)

suite:run(5000, 100)
```

---

## 34.9 Memory Profiling

### ตัวอย่างที่ 14: วัด Memory Usage

```lua
-- example_14_memory.lua

-- collectgarbage("count") คืน KB ที่ใช้อยู่

local function getMemoryKB()
    return collectgarbage("count")
end

local function measureMemory(func, label)
    collectgarbage("collect")  -- force GC ก่อน
    local before = getMemoryKB()
    
    local result = func()
    
    local after = getMemoryKB()
    
    print(string.format("%-30s before: %8.2f KB, after: %8.2f KB, diff: %+8.2f KB",
        label, before, after, after - before))
    
    return result
end

-- ทดสอบ memory usage
print("Memory Usage Tests:\n")

-- สร้าง strings จำนวนมาก
measureMemory(function()
    local t = {}
    for i = 1, 10000 do
        t[i] = "string_" .. i
    end
    return t
end, "10000 strings:")

-- สร้าง tables จำนวนมาก
measureMemory(function()
    local t = {}
    for i = 1, 1000 do
        t[i] = {x = i, y = i * 2, name = "obj" .. i}
    end
    return t
end, "1000 tables:")

-- สร้าง functions จำนวนมาก
measureMemory(function()
    local t = {}
    for i = 1, 1000 do
        local n = i  -- capture
        t[i] = function() return n end
    end
    return t
end, "1000 closures:")

-- หลัง GC
print("\nAfter GC:")
collectgarbage("collect")
print(string.format("Memory: %.2f KB", getMemoryKB()))
```

### ตัวอย่างที่ 15: Memory Leak Detection

```lua
-- example_15_memory_leak.lua

-- ตรวจจับ memory leak แบบง่าย

local function checkForLeak(func, iterations, label)
    local samples = {}
    
    for i = 1, iterations do
        func()
        
        if i % (iterations // 10) == 0 then
            collectgarbage("collect")
            table.insert(samples, collectgarbage("count"))
        end
    end
    
    -- ตรวจสอบ trend
    local first = samples[1]
    local last = samples[#samples]
    local growth = last - first
    
    print(string.format("%-30s Start: %8.1f KB, End: %8.1f KB, Growth: %+8.1f KB %s",
        label,
        first, last, growth,
        math.abs(growth) > 10 and "⚠ POSSIBLE LEAK!" or "✓ OK"))
end

-- Leak: global variable ที่สะสม
local globalAccumulator = {}
local function leakyFunction()
    -- ลืม cleanup!
    table.insert(globalAccumulator, {
        data = string.rep("x", 100),
        timestamp = os.time()
    })
end

-- No leak: cleanup properly
local localBuffer = {}
local function cleanFunction()
    -- ใช้แล้วล้าง
    for i = 1, 10 do
        localBuffer[i] = {data = string.rep("y", 100)}
    end
    -- cleanup
    for i = 1, #localBuffer do
        localBuffer[i] = nil
    end
end

print("Memory Leak Detection:\n")
checkForLeak(leakyFunction, 1000, "Leaky function:")
checkForLeak(cleanFunction, 1000, "Clean function:")
```

---

## 34.10 Common Performance Patterns

### ตัวอย่างที่ 16: Memoization

```lua
-- example_16_memoization.lua

-- Memoization: cache ผลลัพธ์ที่คำนวณแล้ว

local function memoize(func)
    local cache = {}
    return function(...)
        local key = table.concat({...}, ",")
        
        if cache[key] == nil then
            cache[key] = func(...)
        end
        
        return cache[key]
    end
end

-- Fibonacci ช้า (exponential)
local function slowFib(n)
    if n <= 1 then return n end
    return slowFib(n - 1) + slowFib(n - 2)
end

-- Fibonacci เร็ว (memoized)
local fastFib
fastFib = memoize(function(n)
    if n <= 1 then return n end
    return fastFib(n - 1) + fastFib(n - 2)
end)

-- วัดเวลา
local function bench(name, func, n)
    local start = os.clock()
    local result = func(n)
    local elapsed = os.clock() - start
    print(string.format("%-25s fib(%d) = %d, time: %.6f sec",
        name, n, result, elapsed))
end

print("Fibonacci Performance:")
bench("Slow (no cache):", slowFib, 30)
bench("Fast (memoized):", fastFib, 30)

-- Memoization สำหรับ expensive computation
local expensiveComputation = memoize(function(x, y)
    -- simulate expensive work
    local result = 0
    for i = 1, 10000 do
        result = result + math.sqrt(x * x + y * y + i)
    end
    return result
end)

print("\nExpensive computation:")
local s1 = os.clock()
expensiveComputation(3, 4)  -- คำนวณจริง
print(string.format("First call: %.4f sec", os.clock() - s1))

local s2 = os.clock()
expensiveComputation(3, 4)  -- ใช้ cache
print(string.format("Cached call: %.6f sec", os.clock() - s2))
```

### ตัวอย่างที่ 17: Batch Processing

```lua
-- example_17_batch.lua

-- Batch processing: ประมวลผลเป็นกลุ่ม

-- ช้า: process ทีละ item
local function processOne(item)
    -- simulate work
    return item * 2 + 1
end

local function processSingly(items)
    local results = {}
    for _, item in ipairs(items) do
        results[#results + 1] = processOne(item)
    end
    return results
end

-- เร็ว: batch process
local function processBatch(items, batchSize)
    batchSize = batchSize or 100
    local results = {}
    local total = #items
    
    for i = 1, total, batchSize do
        local batchEnd = math.min(i + batchSize - 1, total)
        
        -- ประมวลผล batch
        for j = i, batchEnd do
            results[j] = items[j] * 2 + 1
        end
    end
    
    return results
end

-- สร้างข้อมูลทดสอบ
local testData = {}
for i = 1, 100000 do
    testData[i] = i
end

local function bench(name, func)
    local start = os.clock()
    local result = func(testData)
    print(string.format("%-25s %.4f sec, items: %d",
        name, os.clock() - start, #result))
end

bench("Single process:", processSingly)
bench("Batch process:", processBatch)
```

---

## 34.11 LuaJIT Profiling

### ตัวอย่างที่ 18: ใช้ LuaJIT Profiler

```lua
-- example_18_luajit_profiler.lua
-- (ใช้ได้เฉพาะ LuaJIT)

-- ตรวจสอบว่าใช้ LuaJIT
if jit then
    print("Running on LuaJIT", jit.version)
    
    -- เปิด JIT profiler
    require("jit.p").start("fv", "/tmp/profile.txt")
    
    -- code ที่ต้องการ profile
    local sum = 0
    for i = 1, 1000000 do
        sum = sum + math.sqrt(i)
    end
    print("Sum:", sum)
    
    -- หยุด profiler
    require("jit.p").stop()
    
    -- อ่านผลลัพธ์
    print("\nProfile saved to /tmp/profile.txt")
    print("Use: luajit -jp=fv myscript.lua")
else
    print("Not running on LuaJIT - using standard profiler")
    
    -- ใช้ debug.sethook สำหรับ profiling ใน standard Lua
    local callCounts = {}
    local callTime = {}
    local callStack = {}
    
    debug.sethook(function(event)
        local info = debug.getinfo(2, "nS")
        if not info then return end
        
        local name = info.name or info.source .. ":" .. (info.currentline or 0)
        
        if event == "call" then
            callCounts[name] = (callCounts[name] or 0) + 1
            table.insert(callStack, {name = name, start = os.clock()})
        elseif event == "return" and #callStack > 0 then
            local frame = table.remove(callStack)
            if frame.name == name then
                callTime[name] = (callTime[name] or 0) + 
                    (os.clock() - frame.start)
            end
        end
    end, "cr")
    
    -- code ที่ต้องการ profile
    local function target()
        local sum = 0
        for i = 1, 100000 do
            sum = sum + math.sqrt(i)
        end
        return sum
    end
    
    target()
    
    debug.sethook()  -- หยุด profiling
    
    -- แสดงผล
    print("\nFunction call counts (top 5):")
    local sorted = {}
    for name, count in pairs(callCounts) do
        table.insert(sorted, {name = name, count = count})
    end
    table.sort(sorted, function(a, b) return a.count > b.count end)
    
    for i = 1, math.min(5, #sorted) do
        print(string.format("  %-40s %d calls", 
            sorted[i].name, sorted[i].count))
    end
end
```

---

## 34.12 Algorithm Optimization

### ตัวอย่างที่ 19: เลือก Algorithm ที่เหมาะสม

```lua
-- example_19_algorithms.lua

-- เปรียบเทียบ algorithms ต่างๆ

-- Linear search: O(n)
local function linearSearch(t, value)
    for i, v in ipairs(t) do
        if v == value then
            return i
        end
    end
    return nil
end

-- Binary search: O(log n) - ต้องการ sorted array
local function binarySearch(t, value)
    local low, high = 1, #t
    while low <= high do
        local mid = (low + high) // 2
        if t[mid] == value then
            return mid
        elseif t[mid] < value then
            low = mid + 1
        else
            high = mid - 1
        end
    end
    return nil
end

-- Hash lookup: O(1)
local function buildHashLookup(t)
    local hash = {}
    for i, v in ipairs(t) do
        hash[v] = i
    end
    return hash
end

-- สร้างข้อมูลทดสอบ
local SIZE = 10000
local sortedData = {}
for i = 1, SIZE do
    sortedData[i] = i
end

local hashLookup = buildHashLookup(sortedData)

local target = SIZE // 2  -- ค้นหาตรงกลาง

local function benchSearch(name, func, data, val, iters)
    local start = os.clock()
    for i = 1, iters do
        func(data, val)
    end
    local elapsed = os.clock() - start
    print(string.format("%-20s %.4f sec", name, elapsed))
end

local function benchHash(iters)
    local start = os.clock()
    for i = 1, iters do
        local _ = hashLookup[target]
    end
    print(string.format("%-20s %.4f sec", "Hash lookup:", os.clock() - start))
end

local ITERS = 100000
print("Search Algorithm Comparison (100K searches):")
benchSearch("Linear search:", linearSearch, sortedData, target, ITERS)
benchSearch("Binary search:", binarySearch, sortedData, target, ITERS)
benchHash(ITERS)
```

### ตัวอย่างที่ 20: Sorting Algorithms

```lua
-- example_20_sorting.lua

-- Bubble Sort: O(n²) - ช้ามาก
local function bubbleSort(arr)
    local n = #arr
    local t = {table.unpack(arr)}  -- copy
    for i = 1, n do
        for j = 1, n - i do
            if t[j] > t[j+1] then
                t[j], t[j+1] = t[j+1], t[j]
            end
        end
    end
    return t
end

-- Quick Sort: O(n log n) average
local function quickSort(arr, low, high)
    low = low or 1
    high = high or #arr
    
    if low < high then
        -- partition
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

-- Built-in table.sort (ใช้ C implementation)
local function luaSort(arr)
    local t = {table.unpack(arr)}
    table.sort(t)
    return t
end

-- สร้างข้อมูลทดสอบ
math.randomseed(42)
local function makeData(n)
    local t = {}
    for i = 1, n do
        t[i] = math.random(1, n)
    end
    return t
end

local N = 1000

local function benchSort(name, func, data)
    local start = os.clock()
    for i = 1, 100 do
        func({table.unpack(data)})
    end
    print(string.format("%-20s %.4f sec", name, os.clock() - start))
end

local data = makeData(N)
print(string.format("Sorting %d elements (100 runs):", N))
benchSort("Bubble Sort:", bubbleSort, data)
benchSort("Quick Sort:", quickSort, data)
benchSort("table.sort:", luaSort, data)
```

---

## 34.13 String Interning

### ตัวอย่างที่ 21: String Interning ใน Lua

```lua
-- example_21_string_interning.lua

-- Lua intern strings อัตโนมัติ (short strings)
-- strings ที่เหมือนกันจะเป็น object เดียวกัน

-- ตรวจสอบว่า strings เป็น object เดียวกัน
local s1 = "hello"
local s2 = "hello"
local s3 = "hel" .. "lo"

-- ใน Lua string comparison ใช้ pointer เปรียบเทียบ (เร็ว)
print(s1 == s2)  -- true
print(s1 == s3)  -- true (interned)

-- String comparison เร็วมากเพราะ interning
local function benchStringComp(n)
    local s = "testing_string_value"
    local count = 0
    for i = 1, n do
        if s == "testing_string_value" then
            count = count + 1
        end
    end
    return count
end

-- Integer comparison
local function benchIntComp(n)
    local x = 42
    local count = 0
    for i = 1, n do
        if x == 42 then
            count = count + 1
        end
    end
    return count
end

local N = 10000000
local start = os.clock()
benchStringComp(N)
print(string.format("String comparison: %.4f sec", os.clock() - start))

start = os.clock()
benchIntComp(N)
print(string.format("Integer comparison: %.4f sec", os.clock() - start))
```

---

## 34.14 Bytecode Cache

### ตัวอย่างที่ 22: Bytecode Caching

```lua
-- example_22_bytecode.lua

-- Lua compile source code เป็น bytecode ก่อน execute
-- สามารถ save bytecode เพื่อ load เร็วขึ้นครั้งต่อไป

-- Dump bytecode
local sourceCode = [[
local function add(a, b)
    return a + b
end
return add
]]

local chunk = load(sourceCode)
if chunk then
    -- dump bytecode
    local bytecode = string.dump(chunk)
    print(string.format("Source size: %d bytes", #sourceCode))
    print(string.format("Bytecode size: %d bytes", #bytecode))
    
    -- load จาก bytecode (เร็วกว่า compile ใหม่)
    local loadedChunk = load(bytecode)
    if loadedChunk then
        local add = loadedChunk()
        print("Test add(3, 4):", add(3, 4))
    end
end

-- Save bytecode to file
local function saveCompiledModule(sourcePath, outputPath)
    local f = io.open(sourcePath, "r")
    if not f then
        return false, "Cannot open source file"
    end
    
    local source = f:read("*a")
    f:close()
    
    local chunk, err = load(source, "@" .. sourcePath)
    if not chunk then
        return false, "Compile error: " .. tostring(err)
    end
    
    local bytecode = string.dump(chunk)
    
    local out = io.open(outputPath, "wb")
    if not out then
        return false, "Cannot write output file"
    end
    
    out:write(bytecode)
    out:close()
    
    return true
end

print("\nBytecode caching example:")
print("Use: luac -o script.luac script.lua")
print("Then: lua script.luac")
```

---

## 34.15 Real-world Optimization Examples

### ตัวอย่างที่ 23: JSON Serialization Optimization

```lua
-- example_23_json_opt.lua

-- JSON encoder ที่ optimize แล้ว

local function encodeValue(val, parts)
    local t = type(val)
    
    if t == "string" then
        -- escape special characters
        local escaped = val:gsub('[\\"/\n\r\t]', function(c)
            local escapes = {
                ['\\'] = '\\\\',
                ['"'] = '\\"',
                ['/'] = '\\/',
                ['\n'] = '\\n',
                ['\r'] = '\\r',
                ['\t'] = '\\t',
            }
            return escapes[c] or c
        end)
        parts[#parts + 1] = '"'
        parts[#parts + 1] = escaped
        parts[#parts + 1] = '"'
    elseif t == "number" then
        if math.type(val) == "integer" then
            parts[#parts + 1] = tostring(val)
        else
            parts[#parts + 1] = string.format("%.14g", val)
        end
    elseif t == "boolean" then
        parts[#parts + 1] = val and "true" or "false"
    elseif t == "nil" then
        parts[#parts + 1] = "null"
    elseif t == "table" then
        -- ตรวจสอบว่าเป็น array หรือ object
        local isArray = true
        local maxN = 0
        for k, _ in pairs(val) do
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                isArray = false
                break
            end
            if k > maxN then maxN = k end
        end
        isArray = isArray and maxN == #val
        
        if isArray then
            parts[#parts + 1] = "["
            for i, v in ipairs(val) do
                if i > 1 then parts[#parts + 1] = "," end
                encodeValue(v, parts)
            end
            parts[#parts + 1] = "]"
        else
            parts[#parts + 1] = "{"
            local first = true
            for k, v in pairs(val) do
                if not first then parts[#parts + 1] = "," end
                first = false
                encodeValue(tostring(k), parts)
                parts[#parts + 1] = ":"
                encodeValue(v, parts)
            end
            parts[#parts + 1] = "}"
        end
    end
end

local function jsonEncode(val)
    local parts = {}
    encodeValue(val, parts)
    return table.concat(parts)
end

-- ทดสอบ
local data = {
    name = "Test User",
    age = 25,
    scores = {90, 85, 92, 88},
    address = {
        city = "Bangkok",
        country = "Thailand"
    },
    active = true,
    note = nil
}

local json = jsonEncode(data)
print("JSON output:", json)

-- Benchmark
local start = os.clock()
for i = 1, 100000 do
    jsonEncode(data)
end
print(string.format("100K encodes: %.4f sec", os.clock() - start))
```

### ตัวอย่างที่ 24: Matrix Operations

```lua
-- example_24_matrix.lua

-- Matrix multiplication - ตัวอย่าง optimization

-- สร้าง matrix
local function newMatrix(rows, cols, fill)
    local m = {rows = rows, cols = cols, data = {}}
    fill = fill or 0
    for i = 1, rows * cols do
        m.data[i] = fill
    end
    return m
end

-- Access: row-major order
local function get(m, r, c)
    return m.data[(r-1) * m.cols + c]
end

local function set(m, r, c, v)
    m.data[(r-1) * m.cols + c] = v
end

-- ช้า: naive multiplication
local function matMulSlow(a, b)
    assert(a.cols == b.rows)
    local result = newMatrix(a.rows, b.cols)
    
    for i = 1, a.rows do
        for j = 1, b.cols do
            local sum = 0
            for k = 1, a.cols do
                sum = sum + get(a, i, k) * get(b, k, j)
            end
            set(result, i, j, sum)
        end
    end
    
    return result
end

-- เร็ว: cache locals, minimize function calls
local function matMulFast(a, b)
    assert(a.cols == b.rows)
    local aRows, aCols, bCols = a.rows, a.cols, b.cols
    local aData, bData = a.data, b.data
    
    local result = newMatrix(aRows, bCols)
    local rData = result.data
    
    for i = 1, aRows do
        local iOffset = (i-1) * aCols
        local riOffset = (i-1) * bCols
        
        for k = 1, aCols do
            local aik = aData[iOffset + k]
            local kOffset = (k-1) * bCols
            
            for j = 1, bCols do
                rData[riOffset + j] = rData[riOffset + j] + 
                    aik * bData[kOffset + j]
            end
        end
    end
    
    return result
end

-- สร้าง test matrices
local N = 50
local a = newMatrix(N, N)
local b = newMatrix(N, N)

-- fill with random values
math.randomseed(42)
for i = 1, N * N do
    a.data[i] = math.random()
    b.data[i] = math.random()
end

-- Benchmark
local function benchMat(name, func)
    local start = os.clock()
    for i = 1, 10 do
        func(a, b)
    end
    print(string.format("%-20s %.4f sec", name, os.clock() - start))
end

print(string.format("Matrix multiplication (%dx%d), 10 runs:", N, N))
benchMat("Slow (get/set):", matMulSlow)
benchMat("Fast (direct):", matMulFast)
```

---

## 34.16 Profiling Workflow

### ตัวอย่างที่ 25: Complete Profiling Workflow

```lua
-- example_25_profiling_workflow.lua

-- ขั้นตอนการ optimize:
-- 1. วัดก่อน (baseline)
-- 2. หา bottleneck
-- 3. Optimize
-- 4. วัดอีกครั้ง (verify improvement)

print("=== Performance Optimization Workflow ===\n")

-- Step 1: สร้าง code ที่ต้องการ optimize
local function processData_v1(data)
    -- Version 1: ไม่ optimize
    local result = ""
    for _, item in ipairs(data) do
        if item.active then
            result = result .. item.name .. "," .. item.value .. ";"
        end
    end
    return result
end

local function processData_v2(data)
    -- Version 2: ใช้ table.concat
    local parts = {}
    for _, item in ipairs(data) do
        if item.active then
            parts[#parts + 1] = item.name
            parts[#parts + 1] = ","
            parts[#parts + 1] = tostring(item.value)
            parts[#parts + 1] = ";"
        end
    end
    return table.concat(parts)
end

local function processData_v3(data)
    -- Version 3: pre-allocate + local cache
    local name_key = "name"
    local value_key = "value"
    local active_key = "active"
    local parts = {}
    local n = 0
    
    for i = 1, #data do
        local item = data[i]
        if item[active_key] then
            n = n + 1
            parts[n] = item[name_key]
            n = n + 1
            parts[n] = ","
            n = n + 1
            parts[n] = tostring(item[value_key])
            n = n + 1
            parts[n] = ";"
        end
    end
    
    return table.concat(parts)
end

-- Step 2: สร้าง test data
local testData = {}
for i = 1, 1000 do
    testData[i] = {
        name = "item" .. i,
        value = i * 1.5,
        active = i % 3 ~= 0  -- 2/3 active
    }
end

-- Step 3: Benchmark
local RUNS = 1000

local function bench(name, func)
    local start = os.clock()
    for i = 1, RUNS do
        func(testData)
    end
    local elapsed = os.clock() - start
    print(string.format("%-25s %.4f sec (%.2f ms avg)",
        name, elapsed, elapsed / RUNS * 1000))
    return elapsed
end

local t1 = bench("v1 (string concat):", processData_v1)
local t2 = bench("v2 (table.concat):", processData_v2)
local t3 = bench("v3 (pre-alloc):", processData_v3)

print(string.format("\nSpeedup v2 vs v1: %.1fx", t1/t2))
print(string.format("Speedup v3 vs v1: %.1fx", t1/t3))
print(string.format("Speedup v3 vs v2: %.1fx", t2/t3))
```

---

## สรุปบทที่ 34

### เทคนิค Performance Optimization หลัก

| เทคนิค | ผลลัพธ์ | เมื่อไหร่ใช้ |
|--------|---------|------------|
| Cache globals เป็น locals | 10-30% เร็วขึ้น | ใน hot loops |
| table.concat แทน .. | 5-50x เร็วขึ้น | สร้าง string ใน loop |
| Pre-allocate tables | 20-50% เร็วขึ้น | สร้าง large tables |
| Object pools | ลด GC pressure | spawn objects บ่อย |
| Memoization | ขึ้นอยู่กับ cache hit rate | expensive pure functions |
| Integer แทน float | 10-20% เร็วขึ้น | math operations |
| Inline calculations | 5-20% เร็วขึ้น | simple operations ใน hot loops |
| Binary search | O(log n) vs O(n) | sorted data |

### กฎ Performance Optimization

1. **วัดก่อน optimize** - อย่า guess ว่าส่วนไหนช้า
2. **Optimize hot paths เท่านั้น** - 80% ของเวลาอยู่ใน 20% ของ code
3. **อ่านง่ายก่อน เร็วทีหลัง** - optimize เมื่อจำเป็น
4. **ทดสอบหลัง optimize** - อย่าให้ correctness เสีย

---

*บทต่อไป: บทที่ 35 - Memory Management และ Garbage Collection*
