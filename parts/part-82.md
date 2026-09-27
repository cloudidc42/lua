# บทที่ 82: LuaJIT Deep Dive

## บทนำ

LuaJIT คือ Just-In-Time compiler สำหรับ Lua ที่พัฒนาโดย Mike Pall มันเป็นหนึ่งใน fastest dynamic language implementations ในโลก บางครั้ง LuaJIT ทำงานได้เร็วกว่า Lua 5.4 ถึง 10-100 เท่า และเร็วกว่า Python หลายสิบเท่า ในบทนี้เราจะดำดิ่งลึกเข้าไปในสถาปัตยกรรมของ LuaJIT และเรียนรู้วิธีเขียนโค้ดที่ใช้ประโยชน์จากมันได้อย่างเต็มที่

---

## 82.1 สถาปัตยกรรม LuaJIT

### Overview

```
Source Lua Code
       ↓
   Frontend (Parser) → AST
       ↓
   Bytecode (LuaJIT bytecode - ต่างจาก Lua 5.x)
       ↓
   ┌─────────────────────────────┐
   │      LuaJIT Runtime         │
   │  ┌──────────┐ ┌──────────┐  │
   │  │Interpreter│ │ JIT Core │  │
   │  │ (fast)   │ │          │  │
   │  │          │ │ Tracer   │  │
   │  │          │ │ IR Gen   │  │
   │  │          │ │ Backend  │  │
   │  └──────────┘ └──────────┘  │
   │  ┌──────────────────────┐   │
   │  │        FFI           │   │
   │  └──────────────────────┘   │
   └─────────────────────────────┘
```

LuaJIT มี 3 ส่วนหลัก:

1. **Interpreter**: Fast bytecode interpreter เขียนด้วย hand-optimized assembly
2. **JIT Compiler**: Tracing JIT ที่แปลง hot traces เป็น native machine code
3. **FFI**: Foreign Function Interface สำหรับ call C functions โดยตรง

### LuaJIT vs Lua 5.4

```lua
-- ตรวจสอบว่าเราใช้ LuaJIT หรือ Lua 5.4
if jit then
    print("LuaJIT version:", jit.version)
    print("Architecture:", jit.arch)
    print("OS:", jit.os)
    
    -- ดู JIT options
    print("JIT on:", jit.status())
else
    print("Standard Lua:", _VERSION)
end

-- Feature differences
local features = {
    ["goto"]          = true,    -- ทั้งคู่ support (Lua 5.2+)
    ["integers"]      = true,    -- Lua 5.3+ (LuaJIT ใช้ float internally)
    ["bitwise ops"]   = "different",  -- LuaJIT ใช้ bit.* library
    ["utf8"]          = true,    -- Lua 5.3+
    ["FFI"]           = "LuaJIT only",
    ["jit.* API"]     = "LuaJIT only",
    ["table.move"]    = "Lua 5.3+",
}

-- Integer handling difference (สำคัญ!)
-- Lua 5.3+: แยก integer และ float
-- LuaJIT 2.x: ทุกอย่างเป็น float (53-bit integer precision)
-- LuaJIT 2.1: มี 64-bit integer ผ่าน FFI

-- ตรวจสอบ integer support
print(type(1))    -- LuaJIT: number, Lua 5.3+: number (but integer subtype)
print(math.type and math.type(1) or "no math.type")  -- integer หรือ nil
```

---

## 82.2 Tracing JIT ทำงานอย่างไร

### Trace Recording

```lua
-- LuaJIT ทำงานแบบ tracing JIT ไม่ใช่ method JIT

-- ขั้นตอน:
-- 1. Interpreter ทำงานปกติ
-- 2. เมื่อ hotspot ถูกตรวจพบ (loop/function เรียกบ่อย)
-- 3. Tracer เริ่มบันทึก "trace" (sequence of instructions)
-- 4. Trace ถูก optimize และ compile เป็น machine code
-- 5. ครั้งต่อไปที่ hotspot นั้นทำงาน ใช้ compiled trace

-- ดู trace status
if jit then
    -- เปิด JIT verbose mode
    -- jit.on()
    -- require("jit.v").on()  -- verbose trace output
    
    -- ดู trace statistics
    -- require("jit.p").start("vl")  -- profiler
end

-- ตัวอย่าง hot loop ที่จะถูก trace
local function hot_loop()
    local sum = 0
    for i = 1, 1000000 do
        sum = sum + i  -- นี่จะถูก trace และ JIT compile
    end
    return sum
end

-- ครั้งแรก: interpreter
-- หลังจากนั้น: native code (เร็วกว่ามาก!)
print(hot_loop())
```

### Trace Tree vs Linear Trace

```lua
-- LuaJIT สร้าง "trace tree":
-- Root trace: loop แรกที่ถูก trace
-- Side traces: branches ที่ออกจาก root trace

local function trace_tree_demo(data)
    local sum = 0
    for i = 1, #data do
        -- conditional ทำให้มี branching
        if data[i] > 0 then
            sum = sum + data[i]    -- trace branch 1
        else
            sum = sum - data[i]    -- trace branch 2
        end
    end
    return sum
end

-- LuaJIT จะสร้าง:
-- Root trace: ทำ loop iteration ปกติ
-- Side trace 1: branch for positive numbers
-- Side trace 2: branch for negative numbers

-- เมื่อทุก branches ถูก JIT แล้ว: เร็วมาก
local data = {}
for i = 1, 1000000 do
    data[i] = (i % 2 == 0) and i or -i
end

local t1 = os.clock()
print(trace_tree_demo(data))
print("Time:", os.clock() - t1)
```

---

## 82.3 Intermediate Representation (IR)

LuaJIT ใช้ SSA-based IR ก่อน compile เป็น machine code:

```lua
-- ตัวอย่าง IR ที่ LuaJIT สร้าง
-- (ดูได้ด้วย -jdump หรือ require("jit.dump").on())

-- Source code:
local function ir_example(x, y)
    return x * y + x
end

-- IR ประมาณนี้ (simplified):
--[[
0001  SLOAD  #2    -- load y
0002  SLOAD  #1    -- load x
0003  MUL    0002, 0001  -- x * y
0004  ADD    0003, 0002  -- (x*y) + x
0005  RETF   0004        -- return
]]

-- IR optimization: CSE (Common Subexpression Elimination)
local function cse_demo(x)
    -- x*x ปรากฏ 2 ครั้ง
    return x*x + x*x + x*x
    -- IR จะ compute x*x เพียงครั้งเดียวและ reuse
end

-- IR optimization: Constant Folding
local function cf_demo()
    return 2 * 3 + 4  -- คำนวณ ณ compile time
end

-- IR optimization: Loop Invariant Code Motion (LICM)
local function licm_demo(t, n)
    local len = #t  -- ถ้า t ไม่เปลี่ยน ย้ายออกนอก loop
    local sum = 0
    for i = 1, n do
        sum = sum + len  -- len เป็น loop invariant
    end
    return sum
end
```

---

## 82.4 Guard Checks และ Side Exits

```lua
-- Guards คือ assumptions ที่ JIT compiler ทำ
-- ถ้า assumption ผิด → side exit → กลับไป interpreter

-- ตัวอย่าง guard types:

-- Type guard: ตรวจว่า value มี type ที่ถูกต้อง
local function type_guard_demo(x)
    return x + 1  -- JIT assume x เป็น number
    -- ถ้า x เป็น string ที่ coerce ได้ → side exit
end

-- Array guard: ตรวจว่า index อยู่ใน array bounds
local function array_guard_demo(t, i)
    return t[i]  -- JIT assume i ไม่ out of bounds
end

-- Nil guard: ตรวจว่า value ไม่เป็น nil
local function nil_guard_demo(t)
    return t.x  -- JIT assume t ไม่เป็น nil
end

-- ผลกระทบของ side exits:
-- มีจำนวนน้อย: ไม่เป็นไร
-- มีจำนวนมาก (>25% ของ loop): JIT ละทิ้ง trace นั้น

-- ตัวอย่างที่ทำให้ side exits เยอะ
local function bad_for_jit(data)
    local sum = 0
    for i = 1, #data do
        -- type เปลี่ยนทุก iteration → side exits ตลอด
        sum = sum + data[i]
    end
    return sum
end

-- Mixed types (ควรหลีกเลี่ยง)
local mixed = {1, 2.0, "3", 4, 5.0}  -- ทำให้ guards fail บ่อย

-- Uniform types (ดีกว่า)
local uniform = {1, 2, 3, 4, 5}  -- integer ทั้งหมด
```

---

## 82.5 NYI (Not Yet Implemented)

NYI คือ operations ที่ LuaJIT ยังไม่สามารถ JIT compile ได้:

```lua
-- NYI ทำให้ JIT หยุด trace และกลับไป interpreter
-- ต้องรู้ว่า operations ไหน NYI เพื่อหลีกเลี่ยง

-- Common NYIs ใน LuaJIT 2.x:
-- 1. string.format ที่มี %q
-- 2. table.sort (บางส่วน)
-- 3. math.random
-- 4. Certain metatables
-- 5. pcall/xpcall ใน certain contexts
-- 6. coroutine operations (บางส่วน)
-- 7. Raw string patterns (บางๆ)

-- ตรวจสอบ NYI ด้วย:
if jit then
    -- require("jit.v").on()  -- verbose output
    -- require("jit.p").start("F")  -- profile ดู NYI
end

-- ตัวอย่าง NYI ที่พบบ่อย
local function nyi_demo()
    local t = {3, 1, 4, 1, 5, 9, 2, 6}
    
    -- table.sort มี NYI ใน certain cases
    table.sort(t)  -- อาจทำให้ JIT ออก
    
    -- string.format บางรูปแบบ NYI
    local s = string.format("%q", "hello")  -- %q = NYI
    
    -- tostring ของบาง types
    local n = tostring(3.14)  -- อาจ NYI
    
    return t, s, n
end

-- วิธีตรวจสอบ NYI ใน production
local function check_jit_status(f)
    if not jit then return end
    
    local traces_before = {}
    -- ดู trace count ก่อน
    -- run function หลายครั้ง
    for i = 1, 1000 do f() end
    -- ถ้า JIT ทำงาน performance จะดีขึ้นมาก
end

-- Workaround สำหรับ NYI
-- แทนที่จะใช้ table.sort ใน hot loop:
-- ใช้ sorted insertion หรือ pre-sort ข้างนอก loop
local function sort_workaround(data)
    -- pre-sort ครั้งเดียว
    table.sort(data)
    
    -- hot loop ที่ไม่มี sort
    local sum = 0
    for i = 1, #data do
        sum = sum + data[i]
    end
    return sum
end
```

---

## 82.6 FFI (Foreign Function Interface)

FFI คือหนึ่งใน features ที่ทรงพลังที่สุดของ LuaJIT:

```lua
-- ต้องใช้ LuaJIT
if not jit then
    print("FFI requires LuaJIT")
    return
end

local ffi = require("ffi")

-- กำหนด C types และ functions
ffi.cdef[[
    // C standard library functions
    int printf(const char *fmt, ...);
    void *malloc(size_t size);
    void free(void *ptr);
    double sqrt(double x);
    
    // Custom struct
    typedef struct {
        double x;
        double y;
        double z;
    } Vec3;
    
    // Custom function declaration
    int memcmp(const void *s1, const void *s2, size_t n);
    void *memcpy(void *dest, const void *src, size_t n);
    void *memset(void *s, int c, size_t n);
]]

-- ใช้ sqrt โดยตรง (ไม่ต้องผ่าน Lua math library)
local sqrt = ffi.C.sqrt

local function ffi_sqrt_benchmark()
    local sum = 0.0
    for i = 1, 1000000 do
        sum = sum + sqrt(i)  -- C sqrt โดยตรง!
    end
    return sum
end

-- สร้าง C struct
local Vec3 = ffi.typeof("Vec3")

local function ffi_struct_demo()
    local v1 = Vec3(1.0, 2.0, 3.0)
    local v2 = Vec3(4.0, 5.0, 6.0)
    
    -- dot product
    local dot = v1.x*v2.x + v1.y*v2.y + v1.z*v2.z
    print("Dot product:", dot)
    
    -- magnitude
    local mag = sqrt(v1.x*v1.x + v1.y*v1.y + v1.z*v1.z)
    print("Magnitude:", mag)
end

ffi_struct_demo()
```

### FFI Array Operations

```lua
if not jit then return end

local ffi = require("ffi")

-- FFI arrays มีประสิทธิภาพสูงมาก
-- เพราะ JIT สามารถ access memory โดยตรงโดยไม่ผ่าน boxing

-- ตัวอย่าง: float array
ffi.cdef[[
    float *new_float_array(int n);
]]

-- สร้าง array ด้วย ffi.new
local function ffi_array_demo()
    local n = 1000000
    
    -- FFI array (เหมือน C array)
    local arr = ffi.new("double[?]", n)
    
    -- Fill array
    for i = 0, n-1 do  -- C-style index (0-based)
        arr[i] = i * 0.001
    end
    
    -- Sum array
    local sum = 0.0
    for i = 0, n-1 do
        sum = sum + arr[i]
    end
    
    return sum
end

-- เทียบกับ Lua table
local function lua_array_demo()
    local n = 1000000
    local arr = {}
    
    for i = 1, n do
        arr[i] = i * 0.001
    end
    
    local sum = 0.0
    for i = 1, n do
        sum = sum + arr[i]
    end
    
    return sum
end

-- FFI array เร็วกว่า Lua table มาก
-- เพราะ:
-- 1. ไม่มี boxing (ทุกค่าเป็น raw double)
-- 2. JIT สามารถ optimize memory access pattern
-- 3. Cache-friendly layout
```

### FFI Types สำหรับ Performance

```lua
if not jit then return end

local ffi = require("ffi")

ffi.cdef[[
    typedef struct {
        int32_t x, y;
    } Point2D;
    
    typedef struct {
        float data[4];  // SIMD-friendly
    } Vec4f;
    
    typedef uint8_t byte;
]]

-- Integer types สำหรับ fast integer ops
local int32_t = ffi.typeof("int32_t")
local uint64_t = ffi.typeof("uint64_t")

-- 64-bit integer arithmetic (ไม่มีใน standard Lua 5.x)
local function int64_demo()
    local big1 = ffi.new("int64_t", 9000000000)  -- 9 billion
    local big2 = ffi.new("int64_t", 1000000000)  -- 1 billion
    
    local result = big1 * big2  -- 9 * 10^18
    print("9B * 1B =", tostring(result))
end

int64_demo()

-- Struct of Arrays vs Array of Structs
-- SOA ดีกว่าสำหรับ SIMD

ffi.cdef[[
    // Array of Structs (AOS) - ไม่ดีสำหรับ SIMD
    typedef struct {
        float x, y, z, w;
    } ParticleAOS;
    
    // Struct of Arrays (SOA) - ดีสำหรับ SIMD  
    typedef struct {
        float *x;
        float *y;
        float *z;
        float *w;
        int count;
    } ParticleSOA;
]]

local function particle_benchmark(n)
    -- AOS approach
    local aos = ffi.new("ParticleAOS[?]", n)
    for i = 0, n-1 do
        aos[i].x = i * 0.1
        aos[i].y = i * 0.2
        aos[i].z = i * 0.3
        aos[i].w = 1.0
    end
    
    local t1 = os.clock()
    local sum = 0.0
    for i = 0, n-1 do
        -- Access x only (must skip y, z, w in memory)
        sum = sum + aos[i].x
    end
    print(string.format("AOS: %.4f sum=%.0f", os.clock()-t1, sum))
    
    -- SOA approach
    local x_arr = ffi.new("float[?]", n)
    for i = 0, n-1 do
        x_arr[i] = i * 0.1
    end
    
    local t2 = os.clock()
    sum = 0.0
    for i = 0, n-1 do
        -- Contiguous x access (cache friendly!)
        sum = sum + x_arr[i]
    end
    print(string.format("SOA: %.4f sum=%.0f", os.clock()-t3, sum))
end
```

---

## 82.7 Profile-Guided Optimization

```lua
-- LuaJIT ทำ profile-guided optimization โดยอัตโนมัติ
-- แต่เราสามารถช่วยได้ด้วยการ structure โค้ดให้ดี

-- ใช้ jit.p สำหรับ profiling
if jit then
    -- profiler
    local ok, prof = pcall(require, "jit.p")
    if ok then
        -- เริ่ม profiling
        -- prof.start("vl", "/tmp/profile.txt")
        
        -- run code
        local sum = 0
        for i = 1, 1000000 do
            sum = sum + math.sqrt(i)
        end
        
        -- หยุด profiling
        -- prof.stop()
    end
end

-- Manual profiling ด้วย os.clock
local function measure(f, name, iterations)
    iterations = iterations or 3
    local times = {}
    
    for i = 1, iterations do
        -- Warmup ก่อน (ให้ JIT compile)
        f()
        local t1 = os.clock()
        f()
        times[i] = os.clock() - t1
    end
    
    table.sort(times)
    local min = times[1]
    local sum = 0
    for _, t in ipairs(times) do sum = sum + t end
    
    print(string.format("%-30s min=%.4fs avg=%.4fs", 
        name, min, sum/#times))
    return min
end

-- Warmup ก่อน benchmark เสมอ!
local function proper_benchmark()
    local function target()
        local sum = 0
        for i = 1, 100000 do
            sum = sum + i
        end
        return sum
    end
    
    -- Warmup: ให้ JIT compile
    for i = 1, 10 do
        target()
    end
    
    -- วัดจริง
    measure(target, "sum loop")
end

proper_benchmark()
```

---

## 82.8 JIT-Specific Optimizations

### Avoiding Type Polymorphism

```lua
-- Type polymorphism ทำให้ JIT ต้องสร้าง specializations หลายอัน
-- ควรทำให้ function มี types ที่คาดเดาได้

-- Bad: polymorphic function
local function polymorphic_add(a, b)
    return a + b  -- อาจเป็น int+int, float+float, string+string
end

-- Good: monomorphic function
local function int_add(a, b)
    -- ใช้แค่กับ integers
    return a + b
end

-- ตัวอย่าง: function ที่ JIT ชอบ
local function jit_friendly_function(t, n)
    -- types ไม่เปลี่ยนระหว่าง iterations
    local sum = 0         -- integer
    local count = 0       -- integer
    
    for i = 1, n do
        local v = t[i]    -- consistent type
        if v > 0 then
            sum = sum + v
            count = count + 1
        end
    end
    
    return sum, count
end
```

### Avoiding Object Creation in Hot Paths

```lua
-- การสร้าง table/string ใน hot path ทำให้ GC pressure สูง
-- และ JIT ต้อง handle allocation ซึ่ง slower

-- Bad: allocate in loop
local function bad_alloc_loop(n)
    local results = {}
    for i = 1, n do
        -- สร้าง table ใหม่ทุก iteration!
        local temp = {x = i, y = i*2}  
        results[i] = temp.x + temp.y
    end
    return results
end

-- Good: avoid allocation
local function good_no_alloc_loop(n)
    local results = {}
    for i = 1, n do
        -- ไม่มี allocation ใน loop
        results[i] = i + i*2  -- คำนวณโดยตรง
    end
    return results
end

-- Good: reuse table
local function good_reuse_loop(n)
    local results = {}
    local temp_x, temp_y = 0, 0  -- ใช้ locals แทน table
    for i = 1, n do
        temp_x = i
        temp_y = i * 2
        results[i] = temp_x + temp_y
    end
    return results
end
```

### Loop Unrolling (Manual)

```lua
-- LuaJIT ทำ loop unrolling บางส่วน แต่เราช่วยได้

-- Normal loop
local function normal_sum(t, n)
    local sum = 0
    for i = 1, n do
        sum = sum + t[i]
    end
    return sum
end

-- Manually unrolled (4x)
local function unrolled_sum(t, n)
    local sum = 0
    local i = 1
    local limit = n - 3
    
    -- Process 4 elements at a time
    while i <= limit do
        sum = sum + t[i] + t[i+1] + t[i+2] + t[i+3]
        i = i + 4
    end
    
    -- Remaining elements
    while i <= n do
        sum = sum + t[i]
        i = i + 1
    end
    
    return sum
end

-- ในทางปฏิบัติ LuaJIT มักทำได้ดีพออยู่แล้ว
-- Manual unrolling อาจไม่ให้ประโยชน์เสมอไป
```

---

## 82.9 Benchmarking Methodology

```lua
-- Benchmarking ที่ถูกต้องสำหรับ LuaJIT

local function benchmark_suite()
    -- 1. Warmup phase (ให้ JIT compile)
    -- 2. Measurement phase (วัดหลาย rounds)
    -- 3. Statistical analysis
    
    local function run_benchmark(name, f, warmup_count, measure_count)
        warmup_count = warmup_count or 10
        measure_count = measure_count or 20
        
        -- Warmup
        for i = 1, warmup_count do
            f()
        end
        
        -- Measure
        local times = {}
        for i = 1, measure_count do
            local t1 = os.clock()
            f()
            times[i] = os.clock() - t1
        end
        
        -- Statistics
        table.sort(times)
        local n = #times
        local sum = 0
        for _, t in ipairs(times) do sum = sum + t end
        local avg = sum / n
        
        -- Median
        local median
        if n % 2 == 0 then
            median = (times[n/2] + times[n/2+1]) / 2
        else
            median = times[math.ceil(n/2)]
        end
        
        -- Standard deviation
        local sq_sum = 0
        for _, t in ipairs(times) do
            sq_sum = sq_sum + (t - avg)^2
        end
        local std = math.sqrt(sq_sum / n)
        
        -- Remove outliers (IQR method)
        local q1 = times[math.floor(n*0.25)]
        local q3 = times[math.floor(n*0.75)]
        local iqr = q3 - q1
        local cleaned = {}
        for _, t in ipairs(times) do
            if t >= q1 - 1.5*iqr and t <= q3 + 1.5*iqr then
                cleaned[#cleaned+1] = t
            end
        end
        
        local clean_sum = 0
        for _, t in ipairs(cleaned) do clean_sum = clean_sum + t end
        
        print(string.format("%-30s min=%.6f median=%.6f avg=%.6f std=%.6f",
            name,
            times[1],
            median,
            avg,
            std))
        
        return times[1]  -- return min time
    end
    
    return run_benchmark
end

local bench = benchmark_suite()

-- ตัวอย่าง benchmarks
local N = 100000

-- Test 1: Simple arithmetic
bench("arithmetic", function()
    local sum = 0
    for i = 1, N do
        sum = sum + i * 2 - i
    end
    return sum
end)

-- Test 2: Table access
local t = {}
for i = 1, N do t[i] = i end

bench("table array access", function()
    local sum = 0
    for i = 1, N do
        sum = sum + t[i]
    end
    return sum
end)

-- Test 3: Function call overhead
local function identity(x) return x end

bench("function calls", function()
    local sum = 0
    for i = 1, N do
        sum = sum + identity(i)
    end
    return sum
end)

-- Test 4: String operations
bench("string concat", function()
    local parts = {}
    for i = 1, 100 do
        parts[i] = tostring(i)
    end
    return table.concat(parts, ",")
end)
```

---

## 82.10 LuaJIT vs Lua 5.4 การเปรียบเทียบ

```lua
-- เปรียบเทียบ performance

local function create_benchmark(name, f)
    return {name = name, func = f}
end

local benchmarks = {
    create_benchmark("fibonacci", function()
        local function fib(n)
            if n < 2 then return n end
            return fib(n-1) + fib(n-2)
        end
        return fib(35)
    end),
    
    create_benchmark("bubble sort 1000", function()
        local t = {}
        for i = 1, 1000 do t[i] = 1001 - i end
        for i = 1, #t do
            for j = 1, #t - i do
                if t[j] > t[j+1] then
                    t[j], t[j+1] = t[j+1], t[j]
                end
            end
        end
        return t[#t]
    end),
    
    create_benchmark("sieve 10000", function()
        local sieve = {}
        local n = 10000
        for i = 2, n do sieve[i] = true end
        for i = 2, math.sqrt(n) do
            if sieve[i] then
                for j = i*i, n, i do
                    sieve[j] = false
                end
            end
        end
        local count = 0
        for i = 2, n do
            if sieve[i] then count = count + 1 end
        end
        return count
    end),
    
    create_benchmark("string ops", function()
        local s = string.rep("a", 100)
        local sum = 0
        for i = 1, 1000 do
            sum = sum + #s
            s = s:sub(1, 99) .. "b"
        end
        return sum
    end),
}

print("\n=== Benchmark Results ===")
print(string.format("%-20s %10s", "Test", "Time(ms)"))
print(string.rep("-", 35))

for _, b in ipairs(benchmarks) do
    -- Warmup
    for i = 1, 3 do b.func() end
    
    local t1 = os.clock()
    for i = 1, 5 do b.func() end
    local elapsed = (os.clock() - t1) * 1000 / 5
    
    print(string.format("%-20s %10.2f", b.name, elapsed))
end
```

---

## 82.11 Real Performance Numbers

```lua
-- ตัวเลข performance จริงๆ (ประมาณการ บน modern hardware)

--[[
Operation                | Lua 5.4    | LuaJIT 2.1
─────────────────────────┼────────────┼────────────
Simple for loop 1M iter  | 10-15ms    | 1-2ms
Fibonacci(35)            | 8000ms     | 200ms
Table access 1M          | 20ms       | 3ms
String concat 100k       | 50ms       | 10ms
Math operations 1M       | 15ms       | 2ms
FFI C call (vs C)        | N/A        | ~2x C speed
GC throughput            | baseline   | 2-3x faster
Memory overhead          | baseline   | lower (FFI)

LuaJIT speedup over Lua 5.4:
- Numeric loops: 5-20x faster
- Table operations: 3-8x faster  
- String ops: 2-4x faster
- Function calls: 2-5x faster
- C FFI calls: can match C speed
]]

-- ตัวอย่างวัดจริง
local function real_benchmark()
    local N = 1000000
    
    -- Test 1: Numeric loop
    local function num_loop()
        local sum = 0
        for i = 1, N do
            sum = sum + i
        end
        return sum
    end
    
    local warmup = 5
    for i = 1, warmup do num_loop() end
    
    local t1 = os.clock()
    local result = num_loop()
    local elapsed = os.clock() - t1
    
    print(string.format("Numeric loop %dM iter: %.3fms (result=%d)",
        N/1000000, elapsed*1000, result))
    
    -- Test 2: Table operations
    local function table_ops()
        local t = {}
        for i = 1, N//10 do
            t[i] = i * 2
        end
        local sum = 0
        for i = 1, #t do
            sum = sum + t[i]
        end
        return sum
    end
    
    for i = 1, warmup do table_ops() end
    t1 = os.clock()
    result = table_ops()
    elapsed = os.clock() - t1
    print(string.format("Table ops 100k: %.3fms (result=%d)",
        elapsed*1000, result))
end

real_benchmark()
```

---

## 82.12 Avoiding Deoptimization

```lua
-- Deoptimization เกิดเมื่อ JIT ทำ assumption ที่ผิด
-- และต้อง fallback ไป interpreter

-- Common deoptimization triggers:

-- 1. Type change
local function type_change_demo()
    local x = 1  -- JIT assume integer
    -- ...
    x = 1.5  -- Type change! JIT ต้อง deoptimize
    return x
end

-- 2. Metatable change
local t = {}
local mt1 = {__index = function(t,k) return k end}
setmetatable(t, mt1)

local function meta_demo()
    return t.x  -- JIT bake in mt1.__index
end

-- ถ้าเราเปลี่ยน metatable หลังจาก JIT compile:
-- setmetatable(t, mt2)  -- Invalidates all traces using t!

-- 3. Global variable change
local cached_math_sqrt = math.sqrt

-- ถ้า math.sqrt เปลี่ยน traces ที่ใช้มันจะ invalid

-- 4. Upvalue modification from C
-- C code ที่เปลี่ยน Lua upvalue โดยตรงอาจทำให้ traces invalid

-- Best practices เพื่อหลีกเลี่ยง deoptimization:
local function best_practices()
    -- 1. ใช้ local variables สำหรับ constants
    local MAX = 1000          -- ไม่เปลี่ยน
    local math_sqrt = math.sqrt  -- cache
    local table_insert = table.insert  -- cache
    
    -- 2. ไม่เปลี่ยน type ของ variables
    local count = 0  -- integer ตลอด
    -- count = 0.0  -- อย่าทำ!
    
    -- 3. ไม่เปลี่ยน metatable ของ objects ที่ใช้ใน hot path
    local data = {}  -- no metatable
    
    -- 4. ทำให้ table structure เสถียร
    local point = {x = 0, y = 0}  -- ไม่เพิ่ม/ลบ fields
    
    local sum = 0
    for i = 1, MAX do
        sum = sum + math_sqrt(i)
    end
    return sum
end
```

---

## 82.13 JIT Control API

```lua
if not jit then
    print("Requires LuaJIT")
else
    -- Enable/Disable JIT globally
    jit.on()   -- enable JIT
    jit.off()  -- disable JIT (interpreter only)
    
    -- Enable/Disable สำหรับ specific function
    local function my_func(x)
        return x * 2
    end
    
    jit.off(my_func)  -- disable JIT for this function
    jit.on(my_func)   -- re-enable
    
    -- ดู JIT status
    print("JIT enabled:", jit.status())
    
    -- Flush all compiled traces
    jit.flush()
    
    -- ดู JIT options
    local opts = {}
    -- jit.opt.start(...)  -- set optimization level
    
    -- Optimization levels:
    -- jit.opt.start(0)     -- no optimization
    -- jit.opt.start(1)     -- basic optimization
    -- jit.opt.start(2)     -- moderate (default)
    -- jit.opt.start(3)     -- aggressive
    -- jit.opt.start("hotloop=10")   -- trace after 10 iterations
    -- jit.opt.start("minstitch=0")  -- minimum trace stitching
    
    -- ตัวอย่าง: ปรับสำหรับ startup performance
    jit.opt.start("hotloop=1")  -- trace ทันที
    
    -- ตัวอย่าง: ปรับสำหรับ throughput
    jit.opt.start("hotloop=100")  -- trace หลังจาก 100 iterations
end
```

---

## 82.14 LuaJIT FFI Performance Patterns

```lua
if not jit then return end

local ffi = require("ffi")

-- Pattern 1: Batch operations ด้วย FFI
ffi.cdef[[
    // BLAS-like operations
    void daxpy(int n, double a, const double *x, double *y);
    double ddot(int n, const double *x, const double *y);
]]

-- เมื่อ C library ไม่มี implement ด้วย FFI เอง:
local function ffi_daxpy(n, a, x, y)
    for i = 0, n-1 do
        y[i] = y[i] + a * x[i]
    end
end

local function ffi_ddot(n, x, y)
    local sum = 0.0
    for i = 0, n-1 do
        sum = sum + x[i] * y[i]
    end
    return sum
end

-- สร้าง arrays
local n = 100000
local x = ffi.new("double[?]", n)
local y = ffi.new("double[?]", n)

for i = 0, n-1 do
    x[i] = i * 0.001
    y[i] = i * 0.002
end

-- Benchmark
local t1 = os.clock()
for iter = 1, 100 do
    ffi_daxpy(n, 2.0, x, y)
end
print("FFI daxpy:", os.clock()-t1)

-- Pattern 2: Callback functions
ffi.cdef[[
    typedef int (*compare_fn)(const void*, const void*);
    void qsort(void *base, size_t nmemb, size_t size, compare_fn compar);
]]

local function ffi_qsort_demo()
    local arr = ffi.new("int[10]", {5,3,1,4,1,5,9,2,6,5})
    
    -- สร้าง callback
    local compare = ffi.cast("compare_fn", function(a, b)
        local av = ffi.cast("int*", a)[0]
        local bv = ffi.cast("int*", b)[0]
        return av - bv
    end)
    
    ffi.C.qsort(arr, 10, ffi.sizeof("int"), compare)
    compare:free()
    
    local result = {}
    for i = 0, 9 do result[i+1] = arr[i] end
    print("Sorted:", table.concat(result, ","))
end

ffi_qsort_demo()
```

---

## 82.15 LuaJIT String Performance

```lua
-- String operations ใน LuaJIT

-- 1. String interning: same as Lua
-- 2. String comparison: O(1) for interned strings
-- 3. Pattern matching: เร็วกว่า Lua 5.x เล็กน้อย

-- ตัวอย่าง string benchmark
local function string_benchmark()
    local N = 100000
    
    -- String building (ควรใช้ table.concat)
    local t1 = os.clock()
    local parts = {}
    for i = 1, N do
        parts[i] = tostring(i)
    end
    local s = table.concat(parts)
    print(string.format("table.concat: %.3fs len=%d", os.clock()-t1, #s))
    
    -- String search
    local haystack = string.rep("abcdefghij", 1000)  -- 10000 chars
    t1 = os.clock()
    local count = 0
    for i = 1, 10000 do
        if haystack:find("hij", 1, true) then  -- plain find (เร็วกว่า)
            count = count + 1
        end
    end
    print(string.format("string.find plain: %.3fs count=%d", os.clock()-t1, count))
    
    -- Pattern matching
    t1 = os.clock()
    count = 0
    for i = 1, 10000 do
        if haystack:find("h.j") then  -- pattern find
            count = count + 1
        end
    end
    print(string.format("string.find pattern: %.3fs count=%d", os.clock()-t1, count))
end

string_benchmark()
```

---

## 82.16 ตัวอย่างโปรแกรมที่ Optimize สำหรับ LuaJIT

```lua
-- ตัวอย่าง: N-body simulation (JIT-friendly version)

local function nbody_simulation(n_bodies, steps)
    -- Initialize bodies
    local px = {}  -- positions x
    local py = {}
    local pz = {}
    local vx = {}  -- velocities
    local vy = {}
    local vz = {}
    local mass = {}
    
    for i = 1, n_bodies do
        px[i] = math.random() * 10 - 5
        py[i] = math.random() * 10 - 5
        pz[i] = math.random() * 10 - 5
        vx[i] = math.random() * 0.1
        vy[i] = math.random() * 0.1
        vz[i] = math.random() * 0.1
        mass[i] = math.random() * 10 + 1
    end
    
    local G = 6.674e-11
    local dt = 0.01
    
    -- SOA layout: JIT-friendly (vs AOS with struct fields)
    local t1 = os.clock()
    
    for step = 1, steps do
        -- Calculate forces
        for i = 1, n_bodies do
            local ax, ay, az = 0, 0, 0
            local pxi, pyi, pzi = px[i], py[i], pz[i]
            
            for j = 1, n_bodies do
                if i ~= j then
                    local dx = px[j] - pxi
                    local dy = py[j] - pyi
                    local dz = pz[j] - pzi
                    local dist_sq = dx*dx + dy*dy + dz*dz
                    local dist = math.sqrt(dist_sq)
                    local force = G * mass[i] * mass[j] / dist_sq
                    local fx = force * dx / dist
                    local fy = force * dy / dist
                    local fz = force * dz / dist
                    ax = ax + fx / mass[i]
                    ay = ay + fy / mass[i]
                    az = az + fz / mass[i]
                end
            end
            
            -- Update velocity
            vx[i] = vx[i] + ax * dt
            vy[i] = vy[i] + ay * dt
            vz[i] = vz[i] + az * dt
        end
        
        -- Update positions
        for i = 1, n_bodies do
            px[i] = px[i] + vx[i] * dt
            py[i] = py[i] + vy[i] * dt
            pz[i] = pz[i] + vz[i] * dt
        end
    end
    
    print(string.format("N-body %d bodies %d steps: %.3fs",
        n_bodies, steps, os.clock()-t1))
    
    return px[1], py[1], pz[1]  -- return first body position
end

math.randomseed(42)
nbody_simulation(20, 100)
```

---

## 82.17 ตัวอย่าง: Matrix Multiplication

```lua
-- Matrix multiplication - benchmark classic

local function create_matrix(rows, cols, fill)
    local m = {}
    for i = 1, rows do
        m[i] = {}
        for j = 1, cols do
            m[i][j] = fill and fill(i, j) or 0
        end
    end
    return m
end

local function matrix_multiply(A, B)
    local n = #A
    local m = #B[1]
    local p = #B
    
    local C = create_matrix(n, m)
    
    for i = 1, n do
        local Ai = A[i]
        local Ci = C[i]
        for k = 1, p do
            local Aik = Ai[k]
            local Bk = B[k]
            for j = 1, m do
                Ci[j] = Ci[j] + Aik * Bk[j]
            end
        end
    end
    
    return C
end

-- ขนาด matrix
local N = 100

local A = create_matrix(N, N, function(i, j) return i + j * 0.1 end)
local B = create_matrix(N, N, function(i, j) return i * 0.1 - j end)

-- Warmup
for i = 1, 3 do matrix_multiply(A, B) end

-- Measure
local t1 = os.clock()
local C = matrix_multiply(A, B)
local elapsed = os.clock() - t1

print(string.format("Matrix %dx%d multiply: %.3fms", N, N, elapsed*1000))
print(string.format("C[1][1] = %.4f", C[1][1]))

-- GFLOPS calculation
local ops = 2.0 * N^3  -- multiply-add per element pair
local gflops = ops / elapsed / 1e9
print(string.format("Performance: %.2f GFLOPS", gflops))
```

---

## 82.18 LuaJIT 2.1 ความแตกต่างจาก 2.0

```lua
-- LuaJIT 2.1 beta improvements:

-- 1. GC64: 64-bit GC (รองรับ memory > 4GB)
-- 2. DUALNUM: รองรับทั้ง integer และ float natively
-- 3. Better table.new
-- 4. Improved trace stitching
-- 5. Better ARM64 support

-- ตรวจสอบ version
if jit then
    local version = jit.version_num
    print(string.format("LuaJIT version: %d.%d.%d",
        version // 10000,
        (version // 100) % 100,
        version % 100))
    
    if version >= 20100 then
        print("LuaJIT 2.1+ features available")
        
        -- table.new: pre-allocate table
        local ok, table_new = pcall(require, "table.new")
        if ok then
            local t = table_new(1000, 0)  -- pre-alloc array part
            for i = 1, 1000 do
                t[i] = i
            end
            print("table.new available")
        end
        
        -- table.clear: fast table clearing
        local ok2, table_clear = pcall(require, "table.clear")
        if ok2 then
            local t = {1, 2, 3, 4, 5}
            table_clear(t)
            print("table.clear available, #t =", #t)
        end
    end
end
```

---

## 82.19 LuaJIT ใน Production

```lua
-- Best practices สำหรับ LuaJIT ใน production

-- 1. Version pinning: ใช้ specific version (2.0.5 stable)
-- 2. Error handling: pcall/xpcall ทำงานต่างกันใน JIT
-- 3. Memory limits: monitor memory usage
-- 4. Trace limits: จำนวน traces มีขีดจำกัด

-- Memory monitoring
local function monitor_memory()
    local kb = collectgarbage("count")
    print(string.format("Memory: %.1f KB (%.1f MB)", kb, kb/1024))
end

-- Trace limit monitoring (LuaJIT)
if jit then
    -- Default: 1024 traces max
    -- เมื่อเต็ม JIT จะ flush traces เก่า
    -- ซึ่งทำให้เกิด "JIT pause" ชั่วคราว
    
    -- Workaround: ลด trace size หรือเพิ่ม limit
    -- jit.opt.start("maxtrace=2000")
    
    print("JIT status:", jit.status())
end

-- Production monitoring pattern
local function production_monitor(f, name)
    local memory_before = collectgarbage("count")
    local t1 = os.clock()
    
    local ok, result = pcall(f)
    
    local elapsed = os.clock() - t1
    local memory_after = collectgarbage("count")
    
    if not ok then
        print(string.format("[ERROR] %s: %s", name, tostring(result)))
        return nil
    end
    
    print(string.format("[OK] %s: %.3fms, mem: %.1f KB",
        name, elapsed*1000, memory_after - memory_before))
    
    return result
end

-- ตัวอย่าง
production_monitor(function()
    local sum = 0
    for i = 1, 1000000 do sum = sum + i end
    return sum
end, "sum loop")
```

---

## 82.20 Debugging JIT Issues

```lua
-- เมื่อ JIT ไม่ทำงานตามที่คาด

-- 1. ตรวจสอบ NYI
if jit then
    -- เปิด verbose mode ชั่วคราว
    -- require("jit.v").on("-")  -- print to stderr
    
    -- หรือ profile
    -- local p = require("jit.p")
    -- p.start("vl5")  -- verbose, line info, 5 levels
end

-- 2. ตรวจสอบ trace aborts
if jit then
    local dumper = {
        on = function()
            -- jit.attach(function(tr, ...) 
            --    ... handle trace events
            -- end, "trace")
        end
    }
end

-- 3. Isolate problematic code
local function isolate_test()
    -- ทดสอบ function ที่สงสัย
    local function test1(x)
        return x * x  -- simple, should JIT
    end
    
    local function test2(x)
        -- มี NYI?
        return tostring(x)  -- string conversion
    end
    
    -- Warmup test1
    for i = 1, 1000 do test1(i) end
    
    -- Measure
    local t1 = os.clock()
    for i = 1, 1000000 do test1(i) end
    print("test1:", os.clock()-t1)
    
    -- Warmup test2
    for i = 1, 1000 do test2(i) end
    
    local t2 = os.clock()
    for i = 1, 1000000 do test2(i) end
    print("test2:", os.clock()-t2)
end

isolate_test()

-- 4. Check type stability
local function check_type_stability()
    local values = {1, 2, 3.0, 4, "5", 6}
    
    local function process(v)
        return v + 1  -- อาจ fail สำหรับ string
    end
    
    local types_seen = {}
    for _, v in ipairs(values) do
        local t = type(v)
        types_seen[t] = (types_seen[t] or 0) + 1
    end
    
    print("Types in data:")
    for t, count in pairs(types_seen) do
        print(string.format("  %s: %d", t, count))
    end
end

check_type_stability()
```

---

## 82.21 SIMD concepts ด้วย LuaJIT FFI

```lua
if not jit then return end

local ffi = require("ffi")

-- SIMD สำหรับ x86/x64 ผ่าน FFI
-- ต้องใช้ C library หรือ assembly

ffi.cdef[[
    // SSE2 types (ผ่าน GCC intrinsics หรือ inline assembly)
    typedef double __v2df __attribute__((__vector_size__(16)));
    typedef float __v4sf __attribute__((__vector_size__(16)));
    
    // หรือใช้ standard types สำหรับ vectorizable loops
    void add_arrays(const double *a, const double *b, double *c, int n);
    void dot_product(const double *a, const double *b, double *result, int n);
]]

-- LuaJIT สามารถ auto-vectorize บาง loops ได้เองด้วย JIT
-- ตัวอย่าง loop ที่ JIT อาจ vectorize:

local function vectorizable_loop(a, b, n)
    -- Loop ที่มี simple operations และไม่มี dependencies
    local c = ffi.new("double[?]", n)
    for i = 0, n-1 do
        c[i] = a[i] + b[i]  -- JIT อาจ vectorize นี้
    end
    return c
end

-- FFI ที่เรียก BLAS library (ถ้ามี)
-- BLAS operations เร็วมากเพราะใช้ SIMD จริงๆ
local function try_blas()
    local ok = pcall(function()
        ffi.load("blas")  -- ต้องมี libBLAS installed
    end)
    if ok then
        print("BLAS available - can use SIMD operations")
    else
        print("BLAS not available - using LuaJIT auto-vectorization")
    end
end

try_blas()
```

---

## 82.22 ตัวอย่างสมบูรณ์: JSON Parser ที่ JIT-Friendly

```lua
-- JSON parser ที่ออกแบบมาให้ JIT ทำงานได้ดี

local function create_json_parser()
    local parser = {}
    
    -- State machine แทน recursive parser
    -- ทำให้ JIT trace ได้ง่ายกว่า
    
    local function skip_whitespace(s, i)
        -- Tight loop ที่ JIT จะ compile
        while i <= #s do
            local c = s:byte(i)
            if c ~= 32 and c ~= 9 and c ~= 10 and c ~= 13 then
                break
            end
            i = i + 1
        end
        return i
    end
    
    local function parse_string(s, i)
        -- i ชี้หลัง opening quote
        local result = {}
        
        while i <= #s do
            local c = s:byte(i)
            if c == 34 then  -- closing "
                return table.concat(result), i + 1
            elseif c == 92 then  -- backslash
                i = i + 1
                local ec = s:byte(i)
                if ec == 110 then result[#result+1] = "\n"
                elseif ec == 116 then result[#result+1] = "\t"
                elseif ec == 114 then result[#result+1] = "\r"
                elseif ec == 34 then result[#result+1] = "\""
                elseif ec == 92 then result[#result+1] = "\\"
                end
            else
                result[#result+1] = string.char(c)
            end
            i = i + 1
        end
        
        error("Unterminated string")
    end
    
    local function parse_number(s, i)
        local start = i
        local c = s:byte(i)
        
        if c == 45 then i = i + 1 end  -- minus
        
        while i <= #s do
            c = s:byte(i)
            if c < 48 or c > 57 then break end
            i = i + 1
        end
        
        if i <= #s and s:byte(i) == 46 then  -- decimal point
            i = i + 1
            while i <= #s do
                c = s:byte(i)
                if c < 48 or c > 57 then break end
                i = i + 1
            end
        end
        
        return tonumber(s:sub(start, i-1)), i
    end
    
    function parser.parse(s)
        local i = skip_whitespace(s, 1)
        if i > #s then return nil, i end
        
        local c = s:byte(i)
        
        if c == 123 then  -- {
            -- parse object
            local obj = {}
            i = skip_whitespace(s, i + 1)
            
            while s:byte(i) ~= 125 do  -- }
                -- parse key
                if s:byte(i) == 34 then  -- "
                    local key, ni = parse_string(s, i + 1)
                    i = skip_whitespace(s, ni)
                    -- expect :
                    i = skip_whitespace(s, i + 1)
                    -- parse value
                    local val, nni = parser.parse(s:sub(i))
                    obj[key] = val
                    i = i + nni - 1
                    i = skip_whitespace(s, i)
                    if s:byte(i) == 44 then  -- ,
                        i = skip_whitespace(s, i + 1)
                    end
                end
            end
            
            return obj, i + 1
            
        elseif c == 91 then  -- [
            -- parse array
            local arr = {}
            i = skip_whitespace(s, i + 1)
            
            while s:byte(i) ~= 93 do  -- ]
                local val, ni = parser.parse(s:sub(i))
                arr[#arr+1] = val
                i = i + ni - 1
                i = skip_whitespace(s, i)
                if s:byte(i) == 44 then  -- ,
                    i = skip_whitespace(s, i + 1)
                end
            end
            
            return arr, i + 1
            
        elseif c == 34 then  -- "
            return parse_string(s, i + 1)
            
        elseif c >= 48 and c <= 57 or c == 45 then  -- number
            return parse_number(s, i)
            
        elseif s:sub(i, i+3) == "true" then
            return true, i + 4
            
        elseif s:sub(i, i+4) == "false" then
            return false, i + 5
            
        elseif s:sub(i, i+3) == "null" then
            return nil, i + 4
        end
    end
    
    return parser
end

-- ทดสอบ
local json = create_json_parser()
local result = json.parse('{"name":"test","value":42,"active":true}')
if result then
    print("Parsed JSON:", result.name, result.value, result.active)
end
```

---

## 82.23 สรุปและ Best Practices

```lua
-- สรุป LuaJIT best practices

local best_practices = [[
1. TYPE STABILITY
   - ใช้ variable กับ type เดียวตลอดอายุ
   - หลีกเลี่ยง type coercion ใน hot paths
   - ใช้ integers สำหรับ loop counters

2. TABLE STRUCTURE
   - สร้าง table ด้วย fields ที่ครบตั้งแต่ต้น
   - ไม่เพิ่ม/ลบ fields ใน hot paths
   - ใช้ array part สำหรับ sequential data

3. FUNCTION CALLS
   - Cache functions ใน locals
   - ใช้ local function แทน global
   - หลีกเลี่ยง vararg (...)  ใน hot functions

4. MEMORY
   - Pre-allocate arrays ด้วย table.new (LuaJIT 2.1)
   - ใช้ FFI arrays สำหรับ numeric data
   - Object pooling สำหรับ frequently-created objects

5. JIT CONTROL
   - Warmup ก่อน benchmark
   - ใช้ jit.off() สำหรับ debug code
   - Monitor trace count ใน production

6. FFI USAGE
   - ใช้ FFI สำหรับ number-crunching
   - SOA layout สำหรับ SIMD-friendly code
   - Cache ffi.typeof() ผลลัพธ์

7. AVOID NYIs
   - ทดสอบ functions ที่สำคัญด้วย jit.v
   - Workaround NYI operations
   - ใช้ alternative implementations

8. BENCHMARKING
   - Warmup 10+ iterations ก่อนวัด
   - วัดหลาย rounds และดู min/median
   - ระวัง garbage collection effects
]]

print(best_practices)
```

---

## แบบฝึกหัด

1. เขียน benchmark เพื่อเปรียบเทียบ Lua table กับ FFI array สำหรับการเก็บข้อมูล numeric 1 ล้านรายการ

2. ออกแบบ function ที่ type-stable สำหรับการประมวลผล array ของตัวเลขที่อาจมีทั้ง integer และ float

3. ใช้ `jit.v` module เพื่อดู trace output ของ fibonacci loop และอธิบายว่า JIT trace อะไรบ้าง

4. สร้าง object pool ที่ JIT-friendly สำหรับ particle system

5. เขียน matrix multiplication ที่ใช้ SOA layout และ compare กับ AOS version

6. ออกแบบ JSON serializer ที่หลีกเลี่ยง NYI operations

7. เปรียบเทียบ performance ของ `string.format` กับ `table.concat` สำหรับการสร้าง strings จำนวนมาก

---

## สรุป

LuaJIT เป็น powerful tool ที่ทำให้ Lua ทำงานได้เร็วขึ้นอย่างมาก แต่ต้องเข้าใจว่ามันทำงานอย่างไรเพื่อใช้ประโยชน์ได้เต็มที่ สิ่งสำคัญที่ต้องจำ:

- Tracing JIT ต้องการ hot paths ที่ stable เพื่อ compile
- Type polymorphism และ NYI operations ทำให้ JIT ไม่ทำงาน
- FFI เป็นวิธีที่ดีที่สุดสำหรับ performance-critical numeric code
- Benchmarking ต้องมี warmup และใช้ statistical analysis
- SOA layout ดีกว่า AOS สำหรับ SIMD

ในบทต่อไป (บทที่ 83) เราจะไปเรียนเรื่อง Compiler Construction - การสร้าง lexer และ parser ในภาษา Lua
