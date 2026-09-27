# บทที่ 45: LuaJIT - Just-In-Time Compilation

## บทนำ

LuaJIT คือ implementation ของ Lua ที่มี Just-In-Time (JIT) compiler ทำให้โค้ด Lua ทำงานได้เร็วขึ้นอย่างมากในบางกรณี LuaJIT ถูกสร้างโดย Mike Pall และเป็นหนึ่งใน dynamic language implementations ที่เร็วที่สุดในโลก

---

## 45.1 LuaJIT คืออะไร

### ตัวอย่างที่ 1: การตรวจสอบ LuaJIT

```lua
-- check_luajit.lua

-- ตรวจสอบว่ากำลังใช้ LuaJIT หรือไม่
if jit then
    print("Running on LuaJIT")
    print("Version:", jit.version)
    print("Architecture:", jit.arch)
    print("OS:", jit.os)
    
    -- ข้อมูล JIT
    local status = jit.status()
    print("JIT enabled:", tostring(status))
    
    -- ดู JIT options
    print("\nJIT configuration:")
    for k, v in pairs(jit.opt and {} or {}) do
        print(string.format("  %s = %s", k, tostring(v)))
    end
else
    print("Running on standard Lua:", _VERSION)
    print("LuaJIT features not available")
end

-- ความแตกต่างระหว่าง LuaJIT และ Lua 5.4
print("\n=== Feature Differences ===")
print("Lua version:", _VERSION)

-- goto (LuaJIT + Lua 5.2+)
local hasGoto = pcall(load, "::label:: goto label")
print("goto support:", hasGoto)

-- integer types
print("integer type support:", math.type ~= nil)

-- bit32 library
print("bit32:", bit32 ~= nil)

-- bit library (LuaJIT specific)
print("bit (LuaJIT):", bit ~= nil)

-- ffi library (LuaJIT specific)
print("ffi (LuaJIT):", type(package and package.preload and package.preload.ffi) ~= "nil"
    or pcall(require, "ffi"))
```

### ตัวอย่างที่ 2: LuaJIT vs Standard Lua Differences

```lua
-- luajit_differences.lua

print("=== LuaJIT vs Lua 5.4 Differences ===\n")

-- 1. goto statement (LuaJIT, Lua 5.2+)
print("1. goto statement:")
local i = 0
::continue::
i = i + 1
if i <= 3 then
    io.write(i .. " ")
    goto continue
end
print("")

-- 2. Number representation
print("\n2. Number types:")
if jit then
    -- LuaJIT: default float (double), integer ต้องใช้ tonumber
    print("LuaJIT uses double as default number type")
    print("type(1):", type(1))  -- number
    print("1 == 1.0:", 1 == 1.0)  -- true
else
    -- Lua 5.3+: แยก integer และ float
    print("Lua 5.3+ has separate integer and float")
    print("math.type(1):", math.type(1))      -- integer
    print("math.type(1.0):", math.type(1.0))  -- float
end

-- 3. Bit operations
print("\n3. Bit operations:")
if bit then
    -- LuaJIT: bit library
    print("LuaJIT bit library:")
    print("  bit.band(0xFF, 0x0F):", bit.band(0xFF, 0x0F))
    print("  bit.bor(0x10, 0x01):", bit.bor(0x10, 0x01))
    print("  bit.lshift(1, 8):", bit.lshift(1, 8))
elseif bit32 then
    -- Lua 5.2: bit32 library
    print("Lua 5.2 bit32 library")
else
    -- Lua 5.3+: native bitwise operators
    print("Lua 5.3+ native bitwise:")
    print("  0xFF & 0x0F:", 0xFF & 0x0F)
    print("  0x10 | 0x01:", 0x10 | 0x01)
    print("  1 << 8:", 1 << 8)
end

-- 4. String internalization
print("\n4. String handling:")
local s1 = "hello"
local s2 = "hello"
print("String identity (s1 == s2):", s1 == s2)  -- true in both
print("rawequal:", rawequal(s1, s2))  -- true (same interned string)
```

---

## 45.2 Performance Differences

### ตัวอย่างที่ 3: Benchmark - Numeric Loop

```lua
-- benchmark_numeric.lua

local function benchmark(name, fn, iterations)
    iterations = iterations or 1000000
    
    -- Warmup
    fn(math.min(iterations, 1000))
    
    local start = os.clock()
    fn(iterations)
    local elapsed = os.clock() - start
    
    local rate = iterations / math.max(elapsed, 0.0001)
    print(string.format("%-30s: %.4fs (%s iter/s)",
        name, elapsed, formatNum(rate)))
end

local function formatNum(n)
    if n >= 1e9 then return string.format("%.1fG", n / 1e9)
    elseif n >= 1e6 then return string.format("%.1fM", n / 1e6)
    elseif n >= 1e3 then return string.format("%.1fK", n / 1e3)
    else return string.format("%.0f", n) end
end

print("=== Benchmark Suite ===")
print("Lua version:", _VERSION)
if jit then print("LuaJIT:", jit.version) end
print()

-- Test 1: Simple counter loop
benchmark("Counter loop (1M iterations)", function(n)
    local sum = 0
    for i = 1, n do
        sum = sum + i
    end
    return sum
end)

-- Test 2: Math operations
benchmark("Math operations (1M)", function(n)
    local result = 0
    for i = 1, n do
        result = result + math.sqrt(i)
    end
    return result
end)

-- Test 3: String operations
benchmark("String concat (100K)", function(n)
    n = math.min(n, 100000)
    local parts = {}
    for i = 1, n do
        parts[i] = tostring(i)
    end
    return table.concat(parts)
end, 100000)

-- Test 4: Table operations
benchmark("Table insert/read (1M)", function(n)
    local t = {}
    for i = 1, n do
        t[i] = i
    end
    local sum = 0
    for i = 1, n do
        sum = sum + t[i]
    end
    return sum
end)

-- Test 5: Function calls
benchmark("Function calls (1M)", function(n)
    local function double(x) return x * 2 end
    local sum = 0
    for i = 1, n do
        sum = sum + double(i)
    end
    return sum
end)

print("\nNote: LuaJIT typically runs 5-100x faster than standard Lua")
print("depending on the operation type")
```

### ตัวอย่างที่ 4: JIT-Friendly vs JIT-Unfriendly Code

```lua
-- jit_friendly.lua

print("=== JIT-Friendly Code Patterns ===\n")

-- Pattern 1: ใช้ local variables (ดีกว่า global)
local function jitFriendly_locals()
    -- GOOD: local variables
    local sum = 0
    local math_sqrt = math.sqrt  -- cache library function
    for i = 1, 1000000 do
        sum = sum + math_sqrt(i)
    end
    return sum
end

local function jitUnfriendly_globals()
    -- BAD: global variable access (ช้ากว่า)
    sum = 0  -- global
    for i = 1, 1000000 do
        sum = sum + math.sqrt(i)  -- non-cached
    end
    return sum
end

-- Test speed
local start = os.clock()
local r1 = jitFriendly_locals()
local t1 = os.clock() - start

start = os.clock()
local r2 = jitUnfriendly_globals()
local t2 = os.clock() - start

print(string.format("Local variables:  %.4fs (result=%.2f)", t1, r1))
print(string.format("Global variables: %.4fs (result=%.2f)", t2, r2))
print(string.format("Speedup: %.1fx", t2 / math.max(t1, 0.0001)))

-- Pattern 2: ประเภทข้อมูลที่สม่ำเสมอ
print("\n--- Type Stability ---")
local function stableTypes()
    local sum = 0.0  -- จะเป็น float เสมอ
    for i = 1, 1000000 do
        sum = sum + i * 1.0  -- consistent float
    end
    return sum
end

local function unstableTypes()
    local result = 0  -- จะสลับระหว่าง int และ float
    for i = 1, 1000000 do
        if i % 2 == 0 then
            result = result + i      -- int
        else
            result = result + i / 2  -- float
        end
    end
    return result
end

start = os.clock()
stableTypes()
local ts = os.clock() - start

start = os.clock()
unstableTypes()
local tu = os.clock() - start

print(string.format("Stable types:   %.4fs", ts))
print(string.format("Unstable types: %.4fs", tu))
```

---

## 45.3 JIT Compilation Process

### ตัวอย่างที่ 5: Understanding the JIT Pipeline

```lua
-- jit_pipeline.lua

--[[
LuaJIT JIT Compilation Pipeline:
1. Interpreter (cold code)
2. Recording (hotspot detection, default: 56 iterations)
3. Compilation (trace compilation)
4. Native code execution (hot paths)

Tracing JIT:
- Record execution trace (linear path)
- Compile trace to native machine code
- Re-enter trace when same path is taken
- "Bail out" when trace mismatch
--]]

if not jit then
    print("LuaJIT not available, showing conceptual demo")
    
    -- จำลอง JIT behavior concept
    local function demonstrateTracing()
        local THRESHOLD = 56  -- iterations before JIT kicks in
        local traces = {}
        
        local function hotLoop(n)
            local sum = 0
            for i = 1, n do
                -- JIT ตรวจสอบ: ถ้าเจอ path เดิมซ้ำๆ จะ compile เป็น native code
                sum = sum + i * i
            end
            return sum
        end
        
        -- ครั้งแรกๆ: interpreted
        print("Phase 1: Interpreted (warm-up)")
        for i = 1, THRESHOLD do
            hotLoop(100)
        end
        
        -- หลัง threshold: "compiled" (simulation)
        print("Phase 2: JIT Compiled (fast)")
        local start = os.clock()
        for i = 1, 1000 do
            hotLoop(1000)
        end
        print(string.format("Time: %.4fs", os.clock() - start))
    end
    
    demonstrateTracing()
    return
end

-- ถ้ามี LuaJIT จริง
print("LuaJIT version:", jit.version)
print("JIT status:", jit.status())

-- Control JIT compilation
print("\n--- JIT Control ---")

-- ปิด JIT
jit.off()
print("JIT disabled")
local start = os.clock()
local sum = 0
for i = 1, 1000000 do sum = sum + i end
local tInterp = os.clock() - start
print(string.format("Interpreted: %.4fs (sum=%d)", tInterp, sum))

-- เปิด JIT
jit.on()
print("\nJIT enabled")
start = os.clock()
sum = 0
for i = 1, 1000000 do sum = sum + i end
local tJIT = os.clock() - start
print(string.format("JIT compiled: %.4fs (sum=%d)", tJIT, sum))
print(string.format("Speedup: %.1fx", tInterp / math.max(tJIT, 0.0001)))
```

### ตัวอย่างที่ 6: JIT Flushing and Optimization

```lua
-- jit_optimization.lua

if not jit then
    print("LuaJIT required for this example")
    print("Showing optimization concepts instead:")
    
    -- Optimization patterns ที่ทำงานได้ทั้ง Lua และ LuaJIT
    
    -- 1. ลด table lookups
    print("\n1. Reduce table lookups:")
    local function slow()
        local sum = 0
        local t = {1, 2, 3, 4, 5}
        for i = 1, 1000000 do
            sum = sum + t[i % 5 + 1]
        end
        return sum
    end
    
    -- Cache table reference
    local function fast()
        local sum = 0
        local t = {1, 2, 3, 4, 5}
        local t1, t2, t3, t4, t5 = t[1], t[2], t[3], t[4], t[5]
        local vals = {t1, t2, t3, t4, t5}
        for i = 1, 1000000 do
            sum = sum + vals[i % 5 + 1]
        end
        return sum
    end
    
    local s1 = os.clock()
    slow()
    s1 = os.clock() - s1
    
    local s2 = os.clock()
    fast()
    s2 = os.clock() - s2
    
    print(string.format("  Standard: %.4fs", s1))
    print(string.format("  Cached:   %.4fs", s2))
    return
end

-- LuaJIT specific optimizations
print("=== LuaJIT Optimizations ===\n")

-- 1. JIT flush: รีเซ็ต compiled code
print("1. JIT flush (reset compiled traces):")
jit.flush()
print("   All JIT traces cleared")

-- 2. ดู JIT statistics
if jit.util then
    print("\n2. JIT statistics:")
    local stats = {}
    jit.util.stats(function(k, v) stats[k] = v end)
    for k, v in pairs(stats) do
        print(string.format("   %s = %s", k, tostring(v)))
    end
end

-- 3. Optimize specific function
print("\n3. Function-level JIT control:")

local function criticalFn(n)
    local sum = 0
    for i = 1, n do
        sum = sum + i * i
    end
    return sum
end

-- Force compile critical function
jit.compile(criticalFn)
print("   criticalFn compiled")

-- 4. Optimization options
print("\n4. JIT optimization options:")
jit.opt.start(2)  -- optimization level 2
print("   Optimization level 2 set")

-- Test
local start = os.clock()
for _ = 1, 1000 do
    criticalFn(10000)
end
print(string.format("   Time with opt level 2: %.4fs", os.clock() - start))
```

---

## 45.4 NYI (Not Yet Implemented)

### ตัวอย่างที่ 7: NYI Functions ที่ต้องระวัง

```lua
-- nyi_list.lua
-- NYI = สิ่งที่ LuaJIT ยังไม่ compile เป็น native code

--[[
NYI Functions/Operations (LuaJIT 2.1):
1. string.format with %q, some formats
2. pairs() (use ipairs หรือ next ด้วยตนเอง)
3. pcall/xpcall ใน JIT traces (causes bailout)
4. table.sort (ไม่ JIT compiled)
5. math.random/randomseed
6. os.time, os.clock ใน trace
7. tostring() for numbers in some cases
8. string.gsub ที่ซับซ้อน
9. coroutine operations ใน traces
10. debug library functions
--]]

print("=== NYI Awareness Demo ===\n")

-- Detect JIT availability
local hasJIT = jit ~= nil

local function timeIt(name, fn, n)
    n = n or 1000000
    local start = os.clock()
    local result = fn(n)
    local elapsed = os.clock() - start
    print(string.format("%-35s %.4fs", name .. ":", elapsed))
    return elapsed
end

-- 1. pairs vs ipairs (ipairs is JIT-friendly)
print("1. Iteration methods:")
local arr = {}
for i = 1, 10000 do arr[i] = i end

timeIt("ipairs (JIT-friendly)", function(n)
    local sum = 0
    for _ = 1, n // 10000 do
        for _, v in ipairs(arr) do sum = sum + v end
    end
    return sum
end)

timeIt("pairs (may not JIT)", function(n)
    local sum = 0
    for _ = 1, n // 10000 do
        for _, v in pairs(arr) do sum = sum + v end
    end
    return sum
end)

timeIt("numeric for (best)", function(n)
    local sum = 0
    local len = #arr
    for _ = 1, n // 10000 do
        for i = 1, len do sum = sum + arr[i] end
    end
    return sum
end)

-- 2. pcall overhead
print("\n2. pcall vs direct call:")

local function safeMath(x)
    if x <= 0 then error("must be positive") end
    return math.sqrt(x)
end

timeIt("direct call", function(n)
    local sum = 0
    for i = 1, n do
        sum = sum + math.sqrt(i)
    end
    return sum
end)

timeIt("pcall wrapped", function(n)
    local sum = 0
    for i = 1, n do
        local ok, result = pcall(safeMath, i)
        if ok then sum = sum + result end
    end
    return sum
end, 100000)

-- 3. String operations
print("\n3. String operations:")

timeIt("tostring(number)", function(n)
    local result
    for i = 1, n do
        result = tostring(i)
    end
    return result
end, 100000)

timeIt("string.format(%d)", function(n)
    local result
    for i = 1, n do
        result = string.format("%d", i)
    end
    return result
end, 100000)
```

---

## 45.5 Tracing JIT

### ตัวอย่างที่ 8: Understanding Traces

```lua
-- understanding_traces.lua

--[[
Tracing JIT คือ:
1. Observer execution path (trace)
2. เมื่อ path ถูกเดินซ้ำหลายครั้ง (hot)
3. Compile path นั้นเป็น native code
4. Side exits: ถ้า condition แตกต่าง -> bailout

Linear Trace: if/else จะมีหลาย traces
Loop Trace: inner loops ที่ tight คือ ideal target
--]]

print("=== Tracing JIT Concepts ===\n")

-- 1. Loop trace (ideal for JIT)
print("1. Tight numeric loop - ideal JIT target:")
local function tightLoop()
    local sum = 0
    -- JIT trace จะ: 
    --   - ตรวจ i <= 1000000
    --   - บวก sum
    --   - increment i
    --   - jump back (single trace)
    for i = 1, 1000000 do
        sum = sum + i
    end
    return sum
end

local t1 = os.clock()
local r1 = tightLoop()
print(string.format("  Result: %d, Time: %.4fs", r1, os.clock() - t1))

-- 2. Loop with branches (multiple traces)
print("\n2. Loop with conditional branches:")
local function branchedLoop()
    local sum = 0
    -- JIT จะสร้าง trace แยกสำหรับแต่ละ branch
    for i = 1, 1000000 do
        if i % 3 == 0 then
            sum = sum + i * 3
        elseif i % 2 == 0 then
            sum = sum + i * 2
        else
            sum = sum + i
        end
    end
    return sum
end

local t2 = os.clock()
local r2 = branchedLoop()
print(string.format("  Result: %d, Time: %.4fs", r2, os.clock() - t2))

-- 3. Loop with function calls (inline vs non-inline)
print("\n3. Function call optimization:")

local function square(x) return x * x end  -- likely inlined

local function withInlineable()
    local sum = 0
    for i = 1, 1000000 do
        sum = sum + square(i)  -- JIT may inline this
    end
    return sum
end

-- Function that prevents inlining (complex or NYI)
local function noInline(x)
    if x <= 0 then error("bad") end
    return x * x
end

local function withNonInlineable()
    local sum = 0
    for i = 1, 1000000 do
        local ok, v = pcall(noInline, i)  -- pcall prevents inlining
        if ok then sum = sum + v end
    end
    return sum
end

t1 = os.clock()
local ri = withInlineable()
ti = os.clock() - t1

t1 = os.clock()
local rn = withNonInlineable()
tn = os.clock() - t1

print(string.format("  Inlineable:     %.4fs", ti))
print(string.format("  Non-inlineable: %.4fs", tn))
print(string.format("  Overhead: %.1fx", tn / math.max(ti, 0.0001)))
```

---

## 45.6 Optimization Tips

### ตัวอย่างที่ 9: Top Optimization Patterns

```lua
-- optimization_tips.lua

print("=== LuaJIT Optimization Tips ===\n")

-- TIP 1: Cache frequently accessed globals
print("TIP 1: Cache globals and module functions")
do
    local function bad()
        local sum = 0
        for i = 1, 1000000 do
            sum = sum + math.sin(i) + math.cos(i)
        end
        return sum
    end
    
    local function good()
        local sin = math.sin  -- cache
        local cos = math.cos  -- cache
        local sum = 0
        for i = 1, 1000000 do
            sum = sum + sin(i) + cos(i)
        end
        return sum
    end
    
    local t1 = os.clock(); bad(); t1 = os.clock() - t1
    local t2 = os.clock(); good(); t2 = os.clock() - t2
    print(string.format("  Without cache: %.4fs", t1))
    print(string.format("  With cache:    %.4fs", t2))
    print(string.format("  Improvement:   %.1fx", t1 / math.max(t2, 0.0001)))
end

-- TIP 2: Avoid type coercion
print("\nTIP 2: Maintain consistent types")
do
    local function typeCoercion()
        local sum = 0  -- int
        for i = 1, 1000000 do
            sum = sum + i / 3  -- coerces to float each time!
        end
        return sum
    end
    
    local function consistentFloat()
        local sum = 0.0  -- float
        local div = 1 / 3  -- precompute
        for i = 1, 1000000 do
            sum = sum + i * div  -- stays float
        end
        return sum
    end
    
    local t1 = os.clock(); typeCoercion(); t1 = os.clock() - t1
    local t2 = os.clock(); consistentFloat(); t2 = os.clock() - t2
    print(string.format("  With coercion: %.4fs", t1))
    print(string.format("  Consistent:    %.4fs", t2))
end

-- TIP 3: Array access pattern
print("\nTIP 3: Array access patterns")
do
    local SIZE = 1000
    local arr = {}
    for i = 1, SIZE do arr[i] = i end
    
    -- Forward sequential: cache-friendly
    local function forwardAccess()
        local sum = 0
        for _ = 1, 1000 do
            for i = 1, SIZE do
                sum = sum + arr[i]
            end
        end
        return sum
    end
    
    -- Random access: cache-unfriendly
    local indices = {}
    math.randomseed(42)
    for i = 1, SIZE do indices[i] = math.random(1, SIZE) end
    
    local function randomAccess()
        local sum = 0
        for _ = 1, 1000 do
            for i = 1, SIZE do
                sum = sum + arr[indices[i]]
            end
        end
        return sum
    end
    
    local t1 = os.clock(); forwardAccess(); t1 = os.clock() - t1
    local t2 = os.clock(); randomAccess(); t2 = os.clock() - t2
    print(string.format("  Sequential:    %.4fs", t1))
    print(string.format("  Random:        %.4fs", t2))
end

-- TIP 4: String building
print("\nTIP 4: String concatenation")
do
    local function badConcat(n)
        local s = ""
        for i = 1, n do
            s = s .. tostring(i) .. ","  -- O(n^2) !!!
        end
        return s
    end
    
    local function goodConcat(n)
        local parts = {}
        for i = 1, n do
            parts[i] = tostring(i)
        end
        return table.concat(parts, ",")  -- O(n)
    end
    
    local N = 10000
    local t1 = os.clock(); badConcat(N); t1 = os.clock() - t1
    local t2 = os.clock(); goodConcat(N); t2 = os.clock() - t2
    print(string.format("  String concat:  %.4fs", t1))
    print(string.format("  table.concat:   %.4fs", t2))
    print(string.format("  Improvement:    %.0fx", t1 / math.max(t2, 0.0001)))
end

-- TIP 5: Avoid creating objects in hot loops
print("\nTIP 5: Avoid allocations in hot paths")
do
    local function withAllocation()
        local sum = 0
        for i = 1, 1000000 do
            local point = {x = i, y = i}  -- allocation every iteration!
            sum = sum + point.x + point.y
        end
        return sum
    end
    
    local function withoutAllocation()
        local sum = 0
        local px, py = 0, 0  -- reuse variables
        for i = 1, 1000000 do
            px, py = i, i
            sum = sum + px + py
        end
        return sum
    end
    
    local t1 = os.clock(); withAllocation(); t1 = os.clock() - t1
    local t2 = os.clock(); withoutAllocation(); t2 = os.clock() - t2
    print(string.format("  With alloc:    %.4fs", t1))
    print(string.format("  Without alloc: %.4fs", t2))
end
```

---

## 45.7 FFI Library

### ตัวอย่างที่ 10: FFI Basics

```lua
-- ffi_basics.lua
-- LuaJIT FFI (Foreign Function Interface)
-- ช่วยให้ call C functions โดยตรงโดยไม่ต้องเขียน C extension

if not jit then
    print("LuaJIT FFI requires LuaJIT")
    print("Showing FFI concepts:")
    
    --[[
    FFI ช่วยให้:
    1. เรียก C library functions โดยตรง
    2. ใช้ C structs
    3. ไม่ต้องผ่าน C extension boilerplate
    4. เร็วมาก (JIT compiles FFI calls)
    
    ตัวอย่าง FFI code (ต้องการ LuaJIT):
    
    local ffi = require("ffi")
    
    -- Declare C functions
    ffi.cdef[[
      double sqrt(double x);
      int printf(const char *fmt, ...);
      void *malloc(size_t size);
      void free(void *ptr);
    ]]
    
    -- Call C math
    print(ffi.C.sqrt(2.0))  -- 1.4142...
    ffi.C.printf("Hello from C! Pi = %.4f\n", math.pi)
    ]]
    
    print("FFI concept shown above")
    return
end

-- LuaJIT FFI example
local ffi = require("ffi")

-- Declare C standard library functions
ffi.cdef[[
    double sqrt(double x);
    double pow(double base, double exp);
    double fabs(double x);
    
    typedef struct {
        double x;
        double y;
    } Point2D;
    
    typedef struct {
        int width;
        int height;
        unsigned char *data;
    } Image;
    
    void *malloc(size_t size);
    void free(void *ptr);
    void *memset(void *s, int c, size_t n);
]]

print("=== FFI Basics ===\n")

-- 1. Call C math functions
print("1. C math functions via FFI:")
print("sqrt(2):", ffi.C.sqrt(2.0))
print("pow(2, 10):", ffi.C.pow(2.0, 10.0))
print("fabs(-3.14):", ffi.C.fabs(-3.14))

-- 2. FFI struct
print("\n2. FFI struct:")
local p = ffi.new("Point2D", {x = 3.0, y = 4.0})
print(string.format("Point: x=%.1f, y=%.1f", p.x, p.y))

-- Calculate distance using FFI sqrt
local dist = ffi.C.sqrt(p.x * p.x + p.y * p.y)
print(string.format("Distance from origin: %.1f", dist))

-- 3. FFI array
print("\n3. FFI array:")
local arr = ffi.new("double[10]")
for i = 0, 9 do  -- C arrays start at 0!
    arr[i] = (i + 1) * 1.5
end
print("FFI array values:")
for i = 0, 9 do
    io.write(string.format("%.1f ", arr[i]))
end
print()

-- 4. C memory allocation
print("\n4. Manual C memory:")
local buf = ffi.cast("char *", ffi.C.malloc(64))
ffi.C.memset(buf, 0, 64)
-- เขียนข้อมูล
local msg = "Hello FFI!"
for i = 0, #msg - 1 do
    buf[i] = string.byte(msg, i + 1)
end
print("C buffer:", ffi.string(buf, #msg))
ffi.C.free(buf)
```

### ตัวอย่างที่ 11: FFI with C Libraries

```lua
-- ffi_c_libraries.lua

if not jit then
    print("LuaJIT FFI required")
    print([[
    FFI Example (LuaJIT only):
    
    local ffi = require("ffi")
    
    -- ใช้ libc string functions
    ffi.cdef[[
        size_t strlen(const char *s);
        int strcmp(const char *s1, const char *s2);
        char *strstr(const char *haystack, const char *needle);
        char *strcpy(char *dest, const char *src);
    ]]
    
    print(ffi.C.strlen("Hello"))  -- 5
    print(ffi.C.strcmp("abc", "abd"))  -- negative number
    
    -- ใช้ libc math
    ffi.cdef[[
        double sin(double x);
        double cos(double x);
        double log(double x);
    ]]
    
    print(ffi.C.sin(math.pi / 2))  -- 1.0
    print(ffi.C.cos(0))  -- 1.0
    print(ffi.C.log(math.exp(1)))  -- 1.0
    ]])
    return
end

local ffi = require("ffi")

-- String functions
ffi.cdef[[
    size_t strlen(const char *s);
    int strcmp(const char *s1, const char *s2);
    int strncmp(const char *s1, const char *s2, size_t n);
    char *strstr(const char *haystack, const char *needle);
    char *strchr(const char *s, int c);
    int toupper(int c);
    int tolower(int c);
    int isdigit(int c);
    int isalpha(int c);
]]

-- Math functions (different from Lua's math)
ffi.cdef[[
    double sin(double x);
    double cos(double x);
    double exp(double x);
    double log(double x);
    double log2(double x);
    double log10(double x);
    double ceil(double x);
    double floor(double x);
    double round(double x);
]]

print("=== FFI C Library Access ===\n")

-- String operations via FFI
print("1. String functions:")
local s = "Hello, LuaJIT World!"
print("strlen:", ffi.C.strlen(s))
print("strcmp('abc','abd'):", ffi.C.strcmp("abc", "abd"))

-- ค้นหา substring
local found = ffi.C.strstr(s, "LuaJIT")
if found ~= nil then
    print("strstr found: offset", ffi.cast("intptr_t", found) - ffi.cast("intptr_t", ffi.cast("char*", s)))
end

-- Character classification
print("\n2. Character classification:")
for _, c in ipairs({"A", "z", "5", "!", " "}) do
    local b = string.byte(c)
    print(string.format("  '%s': isdigit=%d isalpha=%d",
        c, ffi.C.isdigit(b), ffi.C.isalpha(b)))
end

-- Math via FFI
print("\n3. Math functions:")
print("sin(pi/2):", ffi.C.sin(math.pi / 2))
print("log2(1024):", ffi.C.log2(1024))
print("round(3.7):", ffi.C.round(3.7))

-- FFI Performance comparison
print("\n4. Performance: Lua math vs FFI math:")
local N = 10000000
local sum = 0

local start = os.clock()
for i = 1, N do
    sum = sum + math.sin(i * 0.001)
end
local t_lua = os.clock() - start
print(string.format("Lua math.sin: %.4fs", t_lua))

sum = 0
local ffi_sin = ffi.C.sin
start = os.clock()
for i = 1, N do
    sum = sum + ffi_sin(i * 0.001)
end
local t_ffi = os.clock() - start
print(string.format("FFI C sin:    %.4fs", t_ffi))
print(string.format("Difference:   %.1fx", t_lua / math.max(t_ffi, 0.0001)))
```

### ตัวอย่างที่ 12: FFI Structs และ Pointers

```lua
-- ffi_structs.lua

if not jit then
    print([[
    FFI Struct Example (LuaJIT only):
    
    local ffi = require("ffi")
    
    ffi.cdef[[
        typedef struct {
            float r, g, b, a;
        } Color;
        
        typedef struct {
            int x, y, w, h;
        } Rect;
        
        // Array of structs
        typedef struct {
            Color pixels[256];
            int count;
        } Palette;
    ]]
    
    local red = ffi.new("Color", {r=1.0, g=0.0, b=0.0, a=1.0})
    print(red.r, red.g, red.b)  -- 1.0  0.0  0.0
    
    -- Modify fields
    red.b = 0.5
    print("Modified:", red.r, red.g, red.b)  -- 1.0  0.0  0.5
    
    -- Create array of structs
    local palette = ffi.new("Color[256]")
    for i = 0, 255 do
        palette[i].r = i / 255.0
        palette[i].g = (255 - i) / 255.0
        palette[i].b = 0.5
        palette[i].a = 1.0
    end
    print("Palette[0]:", palette[0].r, palette[0].g)
    print("Palette[128]:", palette[128].r, palette[128].g)
    ]])
    return
end

local ffi = require("ffi")

ffi.cdef[[
    typedef struct {
        float x, y, z;
    } Vec3;
    
    typedef struct {
        float m[16];  // 4x4 matrix
    } Mat4;
    
    typedef struct {
        Vec3 position;
        Vec3 normal;
        float u, v;
    } Vertex;
]]

print("=== FFI Structs ===\n")

-- สร้าง Vec3
local v1 = ffi.new("Vec3", {x=1.0, y=2.0, z=3.0})
local v2 = ffi.new("Vec3", {x=4.0, y=5.0, z=6.0})

print("v1:", v1.x, v1.y, v1.z)
print("v2:", v2.x, v2.y, v2.z)

-- Vector operations
local function vec3_add(a, b)
    return ffi.new("Vec3", {x=a.x+b.x, y=a.y+b.y, z=a.z+b.z})
end

local function vec3_dot(a, b)
    return a.x*b.x + a.y*b.y + a.z*b.z
end

local function vec3_length(v)
    return math.sqrt(v.x*v.x + v.y*v.y + v.z*v.z)
end

local v3 = vec3_add(v1, v2)
print("\nv1 + v2:", v3.x, v3.y, v3.z)
print("dot product:", vec3_dot(v1, v2))
print("|v1|:", vec3_length(v1))

-- Array of structs: Vertex buffer
print("\n--- Vertex Buffer ---")
local N = 1000000
local vertices = ffi.new("Vertex[?]", N)  -- array of N vertices

-- Initialize
local start = os.clock()
for i = 0, N - 1 do
    vertices[i].position.x = i * 0.001
    vertices[i].position.y = i * 0.002
    vertices[i].position.z = 0
    vertices[i].normal.x = 0
    vertices[i].normal.y = 0
    vertices[i].normal.z = 1
    vertices[i].u = (i % 1000) / 1000.0
    vertices[i].v = (i // 1000) / 1000.0
end
local t = os.clock() - start

print(string.format("Initialized %d vertices in %.4fs", N, t))
print(string.format("Memory: %.1f MB", ffi.sizeof("Vertex") * N / 1024 / 1024))
print("First vertex:", vertices[0].position.x, vertices[0].position.y)
print("Last vertex:", vertices[N-1].position.x, vertices[N-1].position.y)
```

---

## 45.8 When to Use LuaJIT vs Lua 5.4

### ตัวอย่างที่ 13: Decision Guide

```lua
-- decision_guide.lua

print("=== LuaJIT vs Lua 5.4 Decision Guide ===\n")

local function showDecision(scenario, luajitPros, lua54Pros, recommendation)
    print("Scenario: " .. scenario)
    print("  LuaJIT pros: " .. luajitPros)
    print("  Lua 5.4 pros: " .. lua54Pros)
    print("  Recommendation: " .. recommendation)
    print()
end

showDecision(
    "High-performance numerical computation",
    "JIT compilation, 5-50x faster for numeric loops",
    "Integers are native (no float overhead), Lua 5.3+",
    "USE LuaJIT if maximum speed needed"
)

showDecision(
    "Systems that need C interop (FFI)",
    "FFI: call C without writing wrappers, very fast",
    "Standard C extension API, more portable",
    "USE LuaJIT for rapid C integration"
)

showDecision(
    "Long-running server/daemon",
    "JIT warm-up, memory usage similar",
    "More predictable GC, stable long-term",
    "Either works; LuaJIT for CPU-bound, Lua 5.4 for I/O-bound"
)

showDecision(
    "Embedded in C application",
    "Faster execution, but more complex build",
    "Simpler embedding, standard API, Lua 5.4 features",
    "Lua 5.4 for simplicity, LuaJIT for performance"
)

showDecision(
    "Maximum compatibility",
    "Based on Lua 5.1 (some 5.2 features)",
    "Latest features: integers, bitwise, utf8, etc.",
    "USE Lua 5.4 for latest language features"
)

showDecision(
    "Game development",
    "Excellent - widely used in games (Roblox etc.)",
    "Growing support, better integer math",
    "LuaJIT for established game engines"
)

-- Feature comparison table
print("=== Feature Comparison ===")
print(string.format("%-30s %-12s %-12s", "Feature", "LuaJIT 2.1", "Lua 5.4"))
print(string.rep("-", 56))

local features = {
    {"JIT compilation",         "YES",    "NO"},
    {"FFI library",             "YES",    "NO (C ext)"},
    {"Integers (math.type)",    "NO*",    "YES"},
    {"Bitwise ops (<<, &)",     "NO*",    "YES"},
    {"Integer size",            "32/64b", "64-bit"},
    {"goto statement",          "YES",    "YES"},
    {"Generalized for",         "NO",     "YES"},
    {"to-be-closed vars",       "NO",     "YES"},
    {"UTF-8 library",           "Limited","YES"},
    {"debug.getuservalue",      "YES*",   "YES (v2)"},
    {"string.pack/unpack",      "YES*",   "YES"},
    {"Coroutine C-API",         "YES",    "YES"},
    {"ARM64 support",           "YES",    "YES"},
    {"RISC-V support",          "Limited","YES"},
}

for _, f in ipairs(features) do
    print(string.format("%-30s %-12s %-12s", f[1], f[2], f[3]))
end

print("\n* = partially supported or requires workaround")
```

---

## 45.9 Benchmarks

### ตัวอย่างที่ 14: Comprehensive Benchmark Suite

```lua
-- benchmark_suite.lua

local ITERATIONS = 1000000

local function run(name, fn)
    -- warmup
    for _ = 1, 100 do fn() end
    
    local start = os.clock()
    for _ = 1, ITERATIONS do
        fn()
    end
    local elapsed = os.clock() - start
    local mops = ITERATIONS / elapsed / 1e6  -- million ops per second
    
    print(string.format("  %-30s %8.2f Mops/s  (%6.4fs)", name, mops, elapsed))
end

print("=== Comprehensive Benchmark ===")
print(string.format("Iterations: %d\n", ITERATIONS))
print("Platform: " .. _VERSION .. (jit and (" / " .. jit.version) or ""))
print()

-- Arithmetic
print("ARITHMETIC:")
run("integer add", function() return 1 + 2 end)
run("float add", function() return 1.5 + 2.5 end)
run("integer mul", function() return 7 * 13 end)
run("float mul", function() return 1.5 * 2.5 end)
run("float div", function() return 7.0 / 3.0 end)
run("math.sqrt", function() return math.sqrt(2.0) end)
run("math.sin", function() return math.sin(0.5) end)
run("math.floor", function() return math.floor(3.7) end)

-- String
print("\nSTRING:")
local s = "Hello, World! This is a test string."
run("string.len", function() return #s end)
run("string.sub", function() return s:sub(1, 5) end)
run("string.upper", function() return s:upper() end)
run("string.byte", function() return s:byte(1) end)
run("string.find", function() return s:find("World") end)
run("string.format %d", function() return string.format("%d", 42) end)
run("string.format %s", function() return string.format("%s", "test") end)

-- Table
print("\nTABLE:")
local arr = {}
for i = 1, 100 do arr[i] = i end

run("table read [i]", function()
    local sum = 0
    for i = 1, 100 do sum = sum + arr[i] end
    return sum
end)
run("table write [i]", function()
    for i = 1, 100 do arr[i] = i * 2 end
end)
run("table.insert + remove", function()
    local t = {}
    table.insert(t, 1)
    table.remove(t, 1)
end)

-- Function calls
print("\nFUNCTION CALLS:")
local function id(x) return x end
local function add(a, b) return a + b end
local function fib(n) return n <= 1 and n or fib(n-1) + fib(n-2) end

run("identity call", function() return id(42) end)
run("two args", function() return add(1, 2) end)
run("fib(10)", function() return fib(10) end)

-- Object-oriented
print("\nOOP:")
local Counter = {}
Counter.__index = Counter
function Counter.new(n) return setmetatable({n=n}, Counter) end
function Counter:increment() self.n = self.n + 1 end
function Counter:get() return self.n end

run("method call", function()
    local c = Counter.new(0)
    c:increment()
    return c:get()
end)

-- Closure
print("\nCLOSURES:")
local function makeAdder(x)
    return function(y) return x + y end
end
local add5 = makeAdder(5)

run("closure call", function() return add5(3) end)
run("create closure", function() return makeAdder(42) end)

print("\nNote: Results vary by CPU, OS, and Lua version")
if jit then
    print("JIT acceleration active")
else
    print("Standard interpreter (no JIT)")
end
```

---

## 45.10 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 15: LuaJIT FFI Performance

```lua
-- ffi_performance.lua

if not jit then
    print("FFI performance demo (LuaJIT required)")
    
    -- แสดง equivalent operations ใน pure Lua
    print("\nEquivalent pure Lua implementations:")
    
    -- Image processing simulation
    local function luaBlur(pixels, width, height)
        local result = {}
        for y = 1, height do
            result[y] = {}
            for x = 1, width do
                local sum = 0
                local count = 0
                for dy = -1, 1 do
                    for dx = -1, 1 do
                        local nx = x + dx
                        local ny = y + dy
                        if nx >= 1 and nx <= width and ny >= 1 and ny <= height then
                            sum = sum + pixels[ny][nx]
                            count = count + 1
                        end
                    end
                end
                result[y][x] = sum / count
            end
        end
        return result
    end
    
    -- Create test image
    local W, H = 100, 100
    local img = {}
    for y = 1, H do
        img[y] = {}
        for x = 1, W do
            img[y][x] = math.random(0, 255)
        end
    end
    
    local start = os.clock()
    luaBlur(img, W, H)
    local t = os.clock() - start
    print(string.format("Lua image blur %dx%d: %.4fs", W, H, t))
    print("FFI equivalent would be ~10-50x faster")
    return
end

local ffi = require("ffi")

-- FFI image processing
ffi.cdef[[
    typedef unsigned char uint8_t;
    typedef unsigned int uint32_t;
    
    void *malloc(size_t size);
    void free(void *ptr);
    void *memcpy(void *dest, const void *src, size_t n);
    void *memset(void *s, int c, size_t n);
]]

print("=== FFI Performance Demo ===\n")

local W, H = 200, 200
local SIZE = W * H

-- Allocate image buffers using FFI
local imgBuf    = ffi.cast("uint8_t *", ffi.C.malloc(SIZE))
local resultBuf = ffi.cast("uint8_t *", ffi.C.malloc(SIZE))

-- Initialize with random data
math.randomseed(42)
for i = 0, SIZE - 1 do
    imgBuf[i] = math.random(0, 255)
end

-- FFI blur implementation
local function ffiBlur(src, dst, w, h)
    for y = 0, h - 1 do
        for x = 0, w - 1 do
            local sum = 0
            local count = 0
            for dy = -1, 1 do
                for dx = -1, 1 do
                    local nx = x + dx
                    local ny = y + dy
                    if nx >= 0 and nx < w and ny >= 0 and ny < h then
                        sum = sum + src[ny * w + nx]
                        count = count + 1
                    end
                end
            end
            dst[y * w + x] = sum / count
        end
    end
end

-- Benchmark
local RUNS = 10
local start = os.clock()
for _ = 1, RUNS do
    ffiBlur(imgBuf, resultBuf, W, H)
end
local t_ffi = (os.clock() - start) / RUNS

-- Pure Lua comparison
local luaImg = {}
for i = 0, SIZE - 1 do
    luaImg[i] = imgBuf[i]
end

local function luaBlur(src, dst, w, h)
    for y = 0, h - 1 do
        for x = 0, w - 1 do
            local sum = 0
            local count = 0
            for dy = -1, 1 do
                for dx = -1, 1 do
                    local nx = x + dx
                    local ny = y + dy
                    if nx >= 0 and nx < w and ny >= 0 and ny < h then
                        sum = sum + src[ny * w + nx]
                        count = count + 1
                    end
                end
            end
            dst[y * w + x] = sum / count
        end
    end
end

local luaDst = {}
start = os.clock()
for _ = 1, RUNS do
    luaBlur(luaImg, luaDst, W, H)
end
local t_lua = (os.clock() - start) / RUNS

print(string.format("Image size: %dx%d = %d pixels", W, H, SIZE))
print(string.format("FFI blur:      %.4fs per run", t_ffi))
print(string.format("Lua table:     %.4fs per run", t_lua))
print(string.format("FFI speedup:   %.1fx", t_lua / math.max(t_ffi, 0.0001)))

-- Cleanup
ffi.C.free(imgBuf)
ffi.C.free(resultBuf)
```

### ตัวอย่างที่ 16: LuaJIT Bit Operations

```lua
-- luajit_bit.lua

print("=== Bit Operations ===\n")

-- LuaJIT มี bit library
-- Lua 5.3+ มี native bitwise operators

if bit then
    print("Using LuaJIT bit library:")
    local bit = bit  -- require('bit') ใน LuaJIT
    
    print(string.format("band(0xFF, 0x0F)   = 0x%02X", bit.band(0xFF, 0x0F)))
    print(string.format("bor(0x10, 0x01)    = 0x%02X", bit.bor(0x10, 0x01)))
    print(string.format("bxor(0xFF, 0x0F)   = 0x%02X", bit.bxor(0xFF, 0x0F)))
    print(string.format("bnot(0xFF)         = 0x%08X", bit.bnot(0xFF)))
    print(string.format("lshift(1, 8)       = %d", bit.lshift(1, 8)))
    print(string.format("rshift(256, 4)     = %d", bit.rshift(256, 4)))
    print(string.format("arshift(-1, 8)     = %d", bit.arshift(-1, 8)))
    print(string.format("rol(0x12345678, 4) = 0x%08X", bit.rol(0x12345678, 4)))
    print(string.format("ror(0x12345678, 4) = 0x%08X", bit.ror(0x12345678, 4)))
    print(string.format("bswap(0x12345678)  = 0x%08X", bit.bswap(0x12345678)))
    print(string.format("tobit(2^33)        = %d", bit.tobit(2^33)))
    print(string.format("tohex(0xDEADBEEF)  = %s", bit.tohex(0xDEADBEEF)))
else
    -- Lua 5.3+ native operators
    print("Using Lua 5.3+ native bitwise operators:")
    print(string.format("0xFF & 0x0F   = 0x%02X", 0xFF & 0x0F))
    print(string.format("0x10 | 0x01   = 0x%02X", 0x10 | 0x01))
    print(string.format("0xFF ~ 0x0F   = 0x%02X", 0xFF ~ 0x0F))  -- xor
    print(string.format("~0xFF         = %d", ~0xFF))
    print(string.format("1 << 8        = %d", 1 << 8))
    print(string.format("256 >> 4      = %d", 256 >> 4))
end

-- Practical bit operations
print("\n--- Practical Examples ---")

-- Pack/unpack RGB color
local function packRGB(r, g, b)
    if bit then
        return bit.bor(bit.lshift(r, 16), bit.lshift(g, 8), b)
    else
        return (r << 16) | (g << 8) | b
    end
end

local function unpackRGB(color)
    if bit then
        return bit.rshift(bit.band(color, 0xFF0000), 16),
               bit.rshift(bit.band(color, 0x00FF00), 8),
               bit.band(color, 0x0000FF)
    else
        return (color >> 16) & 0xFF,
               (color >> 8) & 0xFF,
               color & 0xFF
    end
end

local red   = packRGB(255, 0, 0)
local green = packRGB(0, 255, 0)
local blue  = packRGB(0, 0, 255)
local white = packRGB(255, 255, 255)

print(string.format("Red:   #%06X", red))
print(string.format("Green: #%06X", green))
print(string.format("Blue:  #%06X", blue))
print(string.format("White: #%06X", white))

local r, g, b = unpackRGB(0xAB3F7C)
print(string.format("Unpack #AB3F7C: r=%d g=%d b=%d", r, g, b))

-- Flags/bitmask
print("\n--- Bitmask Flags ---")
local FLAGS = {
    VISIBLE  = 1,    -- bit 0
    COLLIDABLE = 2,  -- bit 1
    ACTIVE   = 4,    -- bit 2
    ENEMY    = 8,    -- bit 3
    BOSS     = 16,   -- bit 4
}

local function hasFlag(flags, flag)
    if bit then return bit.band(flags, flag) ~= 0
    else return (flags & flag) ~= 0 end
end

local function setFlag(flags, flag)
    if bit then return bit.bor(flags, flag)
    else return flags | flag end
end

local function clearFlag(flags, flag)
    if bit then return bit.band(flags, bit.bnot(flag))
    else return flags & ~flag end
end

-- สร้าง entity flags
local entity = setFlag(0, FLAGS.VISIBLE)
entity = setFlag(entity, FLAGS.ACTIVE)
entity = setFlag(entity, FLAGS.ENEMY)

print(string.format("Entity flags: 0x%02X = %d", entity, entity))
print("Is visible:", hasFlag(entity, FLAGS.VISIBLE))
print("Is boss:", hasFlag(entity, FLAGS.BOSS))
print("Is enemy:", hasFlag(entity, FLAGS.ENEMY))

-- Remove ENEMY flag
entity = clearFlag(entity, FLAGS.ENEMY)
print("After removing ENEMY:", hasFlag(entity, FLAGS.ENEMY))
```

### ตัวอย่างที่ 17: LuaJIT string.buffer

```lua
-- luajit_buffer.lua

-- LuaJIT มี string.buffer สำหรับ efficient string building

if not jit then
    print("LuaJIT string.buffer not available")
    print("Using table.concat alternative:\n")
    
    -- Alternative สำหรับ standard Lua
    local Buffer = {}
    Buffer.__index = Buffer
    
    function Buffer.new()
        return setmetatable({parts = {}, size = 0}, Buffer)
    end
    
    function Buffer:put(s)
        s = tostring(s)
        table.insert(self.parts, s)
        self.size = self.size + #s
        return self
    end
    
    function Buffer:putf(fmt, ...)
        return self:put(string.format(fmt, ...))
    end
    
    function Buffer:tostring()
        return table.concat(self.parts)
    end
    
    function Buffer:reset()
        self.parts = {}
        self.size = 0
    end
    
    -- ทดสอบ
    local buf = Buffer.new()
    buf:put("Hello")
    buf:put(", ")
    buf:put("World")
    buf:put("!")
    print("Buffer result:", buf:tostring())
    print("Buffer size:", buf.size)
    
    -- Performance test
    local start = os.clock()
    local b = Buffer.new()
    for i = 1, 100000 do
        b:putf("%d,", i)
    end
    local result = b:tostring()
    print(string.format("100K format ops: %.4fs, size=%d", os.clock() - start, #result))
    return
end

-- LuaJIT string.buffer
local buffer = require("string.buffer")

print("=== LuaJIT string.buffer ===\n")

-- สร้าง buffer
local buf = buffer.new()

-- put: append string
buf:put("Hello")
buf:put(", ")
buf:put("World")
buf:put("!")

-- tostring: get result
print("Basic:", tostring(buf))

-- putf: formatted put
buf:reset()
buf:putf("Pi = %.4f\n", math.pi)
buf:putf("E  = %.4f\n", math.exp(1))
print("Formatted:\n" .. tostring(buf))

-- Performance: buffer vs concat
local N = 100000

print("--- Performance Comparison ---")

-- String buffer
local start = os.clock()
local b = buffer.new()
for i = 1, N do
    b:putf("%d,", i)
end
local result1 = tostring(b)
local t1 = os.clock() - start

-- table.concat
start = os.clock()
local parts = {}
for i = 1, N do
    parts[i] = string.format("%d,", i)
end
local result2 = table.concat(parts)
local t2 = os.clock() - start

-- string concat (slow)
start = os.clock()
local s = ""
for i = 1, math.min(N, 10000) do
    s = s .. string.format("%d,", i)
end
local t3 = os.clock() - start

print(string.format("string.buffer: %.4fs (%d bytes)", t1, #result1))
print(string.format("table.concat:  %.4fs (%d bytes)", t2, #result2))
print(string.format("string concat: %.4fs (10K only)", t3))
print(string.format("buffer vs table.concat: %.1fx", t2 / math.max(t1, 0.0001)))
```

### ตัวอย่างที่ 18: Profiling LuaJIT Code

```lua
-- profiling.lua

-- Simple profiler สำหรับ LuaJIT และ standard Lua

local Profiler = {}
Profiler.__index = Profiler

function Profiler.new()
    local self = setmetatable({}, Profiler)
    self.data = {}
    self.callStack = {}
    self.active = false
    return self
end

function Profiler:start()
    self.active = true
    debug.sethook(function(event)
        local info = debug.getinfo(2, "Sn")
        if not info then return end
        
        local name = (info.name or "?") .. "@" .. 
                     (info.short_src or "?") .. ":" ..
                     (info.linedefined or 0)
        
        local t = os.clock()
        
        if event == "call" then
            table.insert(self.callStack, {name = name, startTime = t})
        elseif event == "return" then
            if #self.callStack > 0 then
                local frame = table.remove(self.callStack)
                local elapsed = t - frame.startTime
                
                if not self.data[frame.name] then
                    self.data[frame.name] = {calls = 0, totalTime = 0, selfTime = 0}
                end
                local d = self.data[frame.name]
                d.calls = d.calls + 1
                d.totalTime = d.totalTime + elapsed
                d.selfTime = d.selfTime + elapsed
                
                -- Subtract from parent
                if #self.callStack > 0 then
                    local parent = self.callStack[#self.callStack]
                    if not self.data[parent.name] then
                        self.data[parent.name] = {calls = 0, totalTime = 0, selfTime = 0}
                    end
                    self.data[parent.name].selfTime = 
                        self.data[parent.name].selfTime - elapsed
                end
            end
        end
    end, "cr")
end

function Profiler:stop()
    self.active = false
    debug.sethook()
end

function Profiler:report(topN)
    topN = topN or 10
    
    -- Sort by total time
    local entries = {}
    for name, data in pairs(self.data) do
        table.insert(entries, {
            name = name,
            calls = data.calls,
            totalTime = data.totalTime,
            selfTime = math.max(data.selfTime, 0)
        })
    end
    
    table.sort(entries, function(a, b)
        return a.totalTime > b.totalTime
    end)
    
    print(string.format("\n%-40s %8s %10s %10s",
        "Function", "Calls", "Total(ms)", "Self(ms)"))
    print(string.rep("-", 72))
    
    for i = 1, math.min(topN, #entries) do
        local e = entries[i]
        -- ตัดชื่อให้สั้นลง
        local name = e.name
        if #name > 38 then
            name = ".." .. name:sub(-36)
        end
        print(string.format("%-40s %8d %10.2f %10.2f",
            name, e.calls,
            e.totalTime * 1000,
            e.selfTime * 1000))
    end
end

-- ทดสอบ profiler
print("=== Profiling Demo ===\n")

-- Functions to profile
local function fibonacci(n)
    if n <= 1 then return n end
    return fibonacci(n-1) + fibonacci(n-2)
end

local function sumArray(arr)
    local sum = 0
    for i = 1, #arr do sum = sum + arr[i] end
    return sum
end

local function processData(n)
    local arr = {}
    for i = 1, n do arr[i] = i * i end
    return sumArray(arr), fibonacci(math.min(n, 15))
end

-- Profile
local profiler = Profiler.new()
profiler:start()

for _ = 1, 100 do
    processData(100)
end

profiler:stop()
profiler:report(10)

-- Alternative: simple timing profiler
print("\n=== Simple Timer Profiler ===")
local timers = {}

local function timed(name, fn, ...)
    local start = os.clock()
    local results = {fn(...)}
    local elapsed = os.clock() - start
    timers[name] = (timers[name] or 0) + elapsed
    return table.unpack(results)
end

-- ใช้งาน
for _ = 1, 1000 do
    timed("fibonacci(20)", fibonacci, 20)
    timed("sumArray", sumArray, {1,2,3,4,5,6,7,8,9,10})
end

print("\nTimer results:")
local timerEntries = {}
for k, v in pairs(timers) do
    table.insert(timerEntries, {name = k, time = v})
end
table.sort(timerEntries, function(a, b) return a.time > b.time end)

for _, e in ipairs(timerEntries) do
    print(string.format("  %-30s %.4fs", e.name, e.time))
end
```

### ตัวอย่างที่ 19: Hot Path Optimization

```lua
-- hot_path_optimization.lua

print("=== Hot Path Optimization ===\n")

--[[
Hot path = ส่วนของโค้ดที่ execute บ่อยที่สุด
Optimization principle: 80% ของเวลา อยู่ใน 20% ของโค้ด
Focus optimization on the hot path
--]]

-- Example: Particle system
local ParticleSystem = {}
ParticleSystem.__index = ParticleSystem

function ParticleSystem.new(maxParticles)
    local self = setmetatable({}, ParticleSystem)
    self.maxParticles = maxParticles
    self.count = 0
    
    -- สร้าง arrays แบบ parallel (structure of arrays)
    -- ดีกว่า array of structs สำหรับ cache locality
    self.x  = {}
    self.y  = {}
    self.vx = {}
    self.vy = {}
    self.life = {}
    
    return self
end

function ParticleSystem:spawn(x, y, vx, vy, life)
    if self.count >= self.maxParticles then return end
    self.count = self.count + 1
    local i = self.count
    self.x[i]    = x
    self.y[i]    = y
    self.vx[i]   = vx
    self.vy[i]   = vy
    self.life[i] = life
end

-- Hot path: update เรียกทุก frame สำหรับทุก particle
function ParticleSystem:update(dt)
    -- Cache fields เป็น locals เพื่อ JIT friendliness
    local x    = self.x
    local y    = self.y
    local vx   = self.vx
    local vy   = self.vy
    local life = self.life
    local count = self.count
    local gravity = -9.8 * dt
    
    local newCount = 0
    
    -- การทำงาน O(n) ใน hot loop
    for i = 1, count do
        -- Update physics
        vx[i] = vx[i] * 0.999
        vy[i] = vy[i] + gravity
        x[i]  = x[i] + vx[i] * dt
        y[i]  = y[i] + vy[i] * dt
        life[i] = life[i] - dt
        
        -- Remove dead particles (swap with last)
        if life[i] > 0 then
            newCount = newCount + 1
            if newCount ~= i then
                x[newCount]    = x[i]
                y[newCount]    = y[i]
                vx[newCount]   = vx[i]
                vy[newCount]   = vy[i]
                life[newCount] = life[i]
            end
        end
    end
    
    self.count = newCount
end

-- ทดสอบ
math.randomseed(42)
local ps = ParticleSystem.new(10000)

-- Spawn particles
for _ = 1, 5000 do
    ps:spawn(
        math.random(-100, 100),
        math.random(0, 200),
        math.random(-50, 50) * 0.1,
        math.random(20, 100) * 0.1,
        math.random(1, 5) + math.random()
    )
end

print(string.format("Initial particles: %d", ps.count))

-- Simulate
local FRAMES = 1000
local DT = 0.016  -- 60 fps

local start = os.clock()
for _ = 1, FRAMES do
    ps:update(DT)
    
    -- Respawn particles that die
    while ps.count < 3000 do
        ps:spawn(0, 0, math.random(-50, 50) * 0.1,
            math.random(50, 150) * 0.1, 3 + math.random())
    end
end
local elapsed = os.clock() - start

print(string.format("Simulated %d frames: %.4fs", FRAMES, elapsed))
print(string.format("Average FPS: %.1f", FRAMES / elapsed))
print(string.format("Final particles: %d", ps.count))
print(string.format("Particles/sec: %.1fM",
    ps.count * FRAMES / elapsed / 1e6))
```

### ตัวอย่างที่ 20: LuaJIT và Garbage Collector

```lua
-- luajit_gc.lua

print("=== GC Tuning for LuaJIT ===\n")

-- ดู GC stats
local function gcStats()
    local mem = collectgarbage("count")  -- KB
    print(string.format("Memory: %.2f KB", mem))
end

print("Initial:")
gcStats()

-- สร้าง objects
local function createObjects(n)
    local objs = {}
    for i = 1, n do
        objs[i] = {
            id = i,
            name = "object_" .. i,
            data = string.rep("x", 100)
        }
    end
    return objs
end

print("\nAfter creating 10000 objects:")
local objects = createObjects(10000)
gcStats()

print("\nAfter GC collect:")
objects = nil
collectgarbage("collect")
gcStats()

-- GC Tuning
print("\n--- GC Settings ---")

-- ดู current settings
local gcPause = 200     -- default: 200%
local gcMultiplier = 200  -- default: 200%

-- collectgarbage("setpause"): กำหนด pause threshold
-- ค่าสูงขึ้น = GC ทำงานน้อยลง แต่ใช้ memory มากขึ้น
collectgarbage("setpause", gcPause)
print(string.format("GC pause: %d%%", gcPause))

-- collectgarbage("setstepmul"): กำหนด step multiplier
-- ค่าสูงขึ้น = GC เร็วขึ้นแต่ใช้ CPU มากขึ้น
collectgarbage("setstepmul", gcMultiplier)
print(string.format("GC step multiplier: %d%%", gcMultiplier))

-- สำหรับ game/realtime: ปิด auto GC, GC เองในช่วงที่เหมาะสม
print("\n--- Manual GC Control ---")

-- หยุด automatic GC
collectgarbage("stop")
print("Automatic GC stopped")

-- สร้าง objects โดยไม่มี auto GC
for i = 1, 1000 do
    local _ = {data = string.rep("x", 1000)}
end
print("After creating objects (no auto GC):")
gcStats()

-- GC manually
collectgarbage("collect")
print("After manual collect:")
gcStats()

-- เปิด automatic GC กลับ
collectgarbage("restart")
print("\nAutomatic GC restarted")

-- incremental GC: ทำงานทีละนิด
print("\n--- Incremental GC ---")
for i = 1, 100 do
    local _ = {string.rep("y", 100)}
    if i % 10 == 0 then
        collectgarbage("step", 100)  -- run GC for 100 steps
    end
end
print("After incremental GC:")
gcStats()

-- LuaJIT-specific GC info
if jit then
    print("\n--- LuaJIT GC info ---")
    -- LuaJIT ใช้ generational GC (different from Lua 5.4)
    print("LuaJIT uses a non-generational, non-incremental GC")
    print("GC is stop-the-world but very fast for small heaps")
    
    -- จำลอง GC pressure test
    local gcTimes = {}
    for trial = 1, 5 do
        -- สร้าง garbage
        for _ = 1, 10000 do
            local _ = {string.rep("z", 50)}
        end
        
        local start = os.clock()
        collectgarbage("collect")
        local gcTime = os.clock() - start
        table.insert(gcTimes, gcTime)
    end
    
    local totalGC = 0
    for _, t in ipairs(gcTimes) do totalGC = totalGC + t end
    print(string.format("Average GC time: %.4fs", totalGC / #gcTimes))
end
```

---

## 45.11 Practical LuaJIT Examples

### ตัวอย่างที่ 21: LuaJIT Data Processing

```lua
-- data_processing.lua

print("=== Data Processing Benchmark ===\n")

-- สร้าง dataset
local function generateData(n)
    math.randomseed(42)
    local data = {}
    for i = 1, n do
        data[i] = {
            id = i,
            value = math.random() * 1000,
            category = math.random(1, 5),
            active = math.random() > 0.3
        }
    end
    return data
end

local N = 100000
local data = generateData(N)
print(string.format("Dataset: %d records\n", N))

-- Operation 1: Filter
local function filterActive(data)
    local result = {}
    for _, item in ipairs(data) do
        if item.active then
            result[#result + 1] = item
        end
    end
    return result
end

-- Operation 2: Map
local function mapValues(data, fn)
    local result = {}
    for i, item in ipairs(data) do
        result[i] = fn(item)
    end
    return result
end

-- Operation 3: Reduce
local function reduce(data, fn, init)
    local acc = init
    for _, item in ipairs(data) do
        acc = fn(acc, item)
    end
    return acc
end

-- Operation 4: Group by
local function groupBy(data, keyFn)
    local groups = {}
    for _, item in ipairs(data) do
        local key = keyFn(item)
        if not groups[key] then groups[key] = {} end
        groups[key][#groups[key] + 1] = item
    end
    return groups
end

-- Operation 5: Sort
local function sortBy(data, compareFn)
    local copy = {}
    for i, v in ipairs(data) do copy[i] = v end
    table.sort(copy, compareFn)
    return copy
end

-- Benchmark each operation
local function bench(name, fn)
    local start = os.clock()
    local result = fn()
    local elapsed = os.clock() - start
    local count = type(result) == "table" and #result or result
    print(string.format("  %-20s %.4fs  result=%s", name, elapsed, tostring(count)))
    return result
end

print("Operations:")
local active = bench("filter(active)", function()
    return filterActive(data)
end)

local values = bench("map(value*2)", function()
    return mapValues(data, function(item) return item.value * 2 end)
end)

local total = bench("reduce(sum)", function()
    return reduce(data, function(acc, item) return acc + item.value end, 0)
end)

local groups = bench("groupBy(category)", function()
    return groupBy(data, function(item) return item.category end)
end)

local sorted = bench("sort(value desc)", function()
    return sortBy(data, function(a, b) return a.value > b.value end)
end)

-- Pipeline
print("\nPipeline (filter -> map -> reduce):")
local start = os.clock()

local filteredData = filterActive(data)
local mappedData = mapValues(filteredData, function(item)
    return item.value * (1 + item.category * 0.1)
end)
local pipelineResult = reduce(mappedData, function(acc, v) return acc + v end, 0)

local pipelineTime = os.clock() - start
print(string.format("  Pipeline: %.4fs, result=%.2f", pipelineTime, pipelineResult))
print(string.format("  Active records: %d / %d", #filteredData, N))

-- Category statistics
print("\nCategory statistics:")
for cat = 1, 5 do
    local g = groups[cat]
    if g then
        local catTotal = 0
        for _, item in ipairs(g) do catTotal = catTotal + item.value end
        print(string.format("  Category %d: %d records, avg=%.2f",
            cat, #g, catTotal / #g))
    end
end

print(string.format("\nTop 3 by value: %.2f, %.2f, %.2f",
    sorted[1].value, sorted[2].value, sorted[3].value))
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ LuaJIT:

1. **LuaJIT คืออะไร** - Just-In-Time compiler สำหรับ Lua
2. **ความแตกต่างจาก Lua 5.4** - features, limitations, compatibility
3. **Performance** - benchmarks และ performance characteristics
4. **JIT Compilation Process** - tracing, recording, native code
5. **NYI (Not Yet Implemented)** - สิ่งที่ LuaJIT ยังไม่ JIT compile
6. **Tracing JIT** - how traces work, side exits, bailouts
7. **Optimization Tips** - cache globals, type stability, avoid allocations
8. **FFI Library** - call C without writing wrappers
9. **FFI Structs** - C data structures in Lua
10. **When to Use LuaJIT vs Lua 5.4** - decision guide
11. **Benchmarks** - comprehensive performance comparison
12. **GC Tuning** - garbage collector settings for LuaJIT
13. **Data Processing** - practical performance patterns

LuaJIT เป็นเครื่องมือที่ทรงพลังสำหรับงานที่ต้องการ performance สูง โดยเฉพาะงานคำนวณตัวเลขหนัก การเข้าถึง C libraries ผ่าน FFI และการประมวลผลข้อมูลขนาดใหญ่ อย่างไรก็ตาม ควรพิจารณา Lua 5.4 สำหรับโปรเจกต์ที่ต้องการ features ล่าสุดหรือ maximum compatibility
