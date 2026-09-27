# บทที่ 85: High-Performance Lua

## บทนำ

การเขียน Lua ที่มีประสิทธิภาพสูงต้องการความเข้าใจทั้งในระดับ language semantics, VM internals, hardware characteristics และ algorithmic complexity บทนี้จะครอบคลุมเทคนิค optimization ทุกระดับ ตั้งแต่ micro-optimizations ที่ให้ผล 2-5% ไปจนถึง architectural decisions ที่ให้ผล 100x

---

## 85.1 CPU Cache Effects

Memory hierarchy มีผลอย่างมากต่อ performance:

```
Register:     0.25ns    ~1 cycle
L1 cache:     1ns       4 cycles     (32-64 KB)
L2 cache:     3-4ns     12 cycles    (256 KB - 1 MB)
L3 cache:     12-30ns   40-100 cycles (8-64 MB)
RAM:          50-100ns  200 cycles   (GBs)
SSD:          100μs     
HDD:          5-10ms
```

```lua
-- ตัวอย่าง: Cache-friendly vs Cache-unfriendly access patterns

-- Cache-friendly: sequential access
local function cache_friendly_sum(data, n)
    local sum = 0
    for i = 1, n do
        sum = sum + data[i]  -- sequential, cache-friendly
    end
    return sum
end

-- Cache-unfriendly: random access
local function cache_unfriendly_sum(data, n, indices)
    local sum = 0
    for i = 1, n do
        sum = sum + data[indices[i]]  -- random, cache-unfriendly
    end
    return sum
end

-- ทดสอบ cache effects
local function test_cache_effects()
    local N = 1000000
    local data = {}
    for i = 1, N do data[i] = i end
    
    -- Sequential indices
    local seq_indices = {}
    for i = 1, N do seq_indices[i] = i end
    
    -- Random indices
    local rand_indices = {}
    for i = 1, N do rand_indices[i] = math.random(1, N) end
    
    -- Warmup
    for _ = 1, 3 do cache_friendly_sum(data, N) end
    
    local t1 = os.clock()
    local s1 = cache_friendly_sum(data, N)
    local t2 = os.clock()
    
    -- Warmup random
    for _ = 1, 3 do cache_unfriendly_sum(data, N, rand_indices) end
    
    local t3 = os.clock()
    local s2 = cache_unfriendly_sum(data, N, rand_indices)
    local t4 = os.clock()
    
    print(string.format("Sequential: %.3fms", (t2-t1)*1000))
    print(string.format("Random:     %.3fms", (t4-t3)*1000))
    print(string.format("Speedup:    %.1fx", (t4-t3)/(t2-t1)))
end

test_cache_effects()

-- Cache line สำคัญ: 64 bytes ใน x86/x64
-- การ access ข้าม cache lines บ่อยๆ = slow

-- ตัวอย่าง: False Sharing (ใน multi-thread context)
-- Thread A เขียน x, Thread B เขียน y
-- ถ้า x และ y อยู่ใน cache line เดียวกัน = false sharing
-- Lua เป็น single-threaded ดังนั้นไม่มี false sharing
-- แต่ data layout ยังคงสำคัญสำหรับ cache efficiency
```

---

## 85.2 Memory Layout Optimization

```lua
-- Array of Structs (AOS) vs Struct of Arrays (SOA)

-- AOS: แต่ละ object มีทุก fields
local function create_aos_particles(n)
    local particles = {}
    for i = 1, n do
        particles[i] = {
            x = math.random() * 100,
            y = math.random() * 100,
            z = math.random() * 100,
            vx = math.random() - 0.5,
            vy = math.random() - 0.5,
            vz = math.random() - 0.5,
            mass = math.random() * 10 + 1,
            alive = true,
        }
    end
    return particles
end

-- SOA: แยก field ออกมาเป็น arrays
local function create_soa_particles(n)
    local x  = {}; local y  = {}; local z  = {}
    local vx = {}; local vy = {}; local vz = {}
    local mass  = {}; local alive = {}
    
    for i = 1, n do
        x[i]     = math.random() * 100
        y[i]     = math.random() * 100
        z[i]     = math.random() * 100
        vx[i]    = math.random() - 0.5
        vy[i]    = math.random() - 0.5
        vz[i]    = math.random() - 0.5
        mass[i]  = math.random() * 10 + 1
        alive[i] = true
    end
    
    return {x=x, y=y, z=z, vx=vx, vy=vy, vz=vz, mass=mass, alive=alive}
end

-- Update particles (only need x, y, z, vx, vy, vz)
local function update_aos(particles, dt)
    for i = 1, #particles do
        local p = particles[i]
        if p.alive then
            p.x = p.x + p.vx * dt
            p.y = p.y + p.vy * dt
            p.z = p.z + p.vz * dt
        end
    end
end

local function update_soa(p, n, dt)
    -- ใช้แค่ x, y, z, vx, vy, vz
    -- mass และ alive ไม่อยู่ใกล้กัน (cache friendly!)
    local x = p.x; local y = p.y; local z = p.z
    local vx = p.vx; local vy = p.vy; local vz = p.vz
    
    for i = 1, n do
        x[i] = x[i] + vx[i] * dt
        y[i] = y[i] + vy[i] * dt
        z[i] = z[i] + vz[i] * dt
    end
end

-- Benchmark
local N_PARTICLES = 100000
local DT = 0.016  -- 60 FPS

math.randomseed(42)
local aos = create_aos_particles(N_PARTICLES)

math.randomseed(42)
local soa = create_soa_particles(N_PARTICLES)

-- Warmup
for _ = 1, 5 do update_aos(aos, DT) end
for _ = 1, 5 do update_soa(soa, N_PARTICLES, DT) end

local t1 = os.clock()
for _ = 1, 100 do update_aos(aos, DT) end
local t2 = os.clock()

local t3 = os.clock()
for _ = 1, 100 do update_soa(soa, N_PARTICLES, DT) end
local t4 = os.clock()

print(string.format("\nAOS update 100x: %.3fms", (t2-t1)*1000))
print(string.format("SOA update 100x: %.3fms", (t4-t3)*1000))
print(string.format("SOA speedup: %.2fx", (t2-t1)/(t4-t3)))
```

---

## 85.3 String Interning และ String Operations

```lua
-- String Interning: Lua intern short strings โดยอัตโนมัติ

-- เข้าใจ string memory model
local function string_memory_demo()
    -- Short strings (< ~40 bytes): automatically interned
    local s1 = "hello"
    local s2 = "hello"
    -- s1 และ s2 ชี้ไปที่เดียวกัน
    
    -- Long strings: ไม่ intern โดยอัตโนมัติ (Lua 5.4)
    local long = string.rep("a", 100)
    
    -- String comparison
    -- Short strings: O(1) เพราะ pointer comparison
    -- Long strings: O(n) หรือ O(1) ถ้า interned
    
    print("s1 == s2:", s1 == s2)  -- true
    
    -- String building patterns
    local parts = {}
    for i = 1, 1000 do
        parts[i] = tostring(i)
    end
    
    -- GOOD: table.concat (เร็วมาก)
    local t1 = os.clock()
    local result1 = table.concat(parts, ",")
    local t2 = os.clock()
    
    -- BAD: concatenation ใน loop (สร้าง string ใหม่ทุกครั้ง)
    local t3 = os.clock()
    local result2 = ""
    for _, p in ipairs(parts) do
        result2 = result2 .. p .. ","  -- O(n²) time!
    end
    local t4 = os.clock()
    
    print(string.format("table.concat: %.4fms", (t2-t1)*1000))
    print(string.format("string concat loop: %.4fms", (t4-t3)*1000))
    print(string.format("concat speedup: %.1fx", (t4-t3)/(t2-t1)))
end

string_memory_demo()

-- String formatting ที่มีประสิทธิภาพ
local function string_format_benchmark()
    local N = 100000
    
    -- Method 1: string.format
    local t1 = os.clock()
    local s = {}
    for i = 1, N do
        s[i] = string.format("x=%d y=%d", i, i*2)
    end
    local t2 = os.clock()
    
    -- Method 2: table.concat
    local t3 = os.clock()
    for i = 1, N do
        s[i] = "x=" .. i .. " y=" .. (i*2)
    end
    local t4 = os.clock()
    
    print(string.format("\nstring.format: %.3fms", (t2-t1)*1000))
    print(string.format("concat:        %.3fms", (t4-t3)*1000))
    
    -- Method 3: pre-computed format string (ถ้า format ซ้ำ)
    local fmt = "x=%d y=%d"
    local t5 = os.clock()
    for i = 1, N do
        s[i] = fmt:format(i, i*2)  -- method syntax
    end
    local t6 = os.clock()
    print(string.format("cached format: %.3fms", (t6-t5)*1000))
end

string_format_benchmark()
```

---

## 85.4 Table Pre-allocation

```lua
-- Table pre-allocation ลด rehashing overhead

-- SLOW: ไม่ pre-allocate
local function slow_table_build(n)
    local t = {}  -- starts empty
    for i = 1, n do
        t[i] = i  -- อาจ trigger resizing หลายครั้ง
    end
    return t
end

-- FAST: pre-allocate ด้วย table.new (LuaJIT) หรือ hints
local function fast_table_build(n)
    -- LuaJIT: ใช้ table.new
    local t
    if package.loaded.jit then
        local ok, new = pcall(require, "table.new")
        if ok then
            t = new(n, 0)  -- pre-alloc n array slots
        end
    end
    
    if not t then
        t = {}
    end
    
    for i = 1, n do
        t[i] = i
    end
    return t
end

-- Hash table pre-allocation
local function hash_prealloc_demo()
    local n = 10000
    
    -- Hash table ที่ไม่ pre-allocate
    local t1 = os.clock()
    local ht = {}
    for i = 1, n do
        ht["key" .. i] = i
    end
    local t2 = os.clock()
    
    print(string.format("\nHash table (no prealloc): %.4fms", (t2-t1)*1000))
    
    -- ในทางปฏิบัติ: กำหนด size hint ด้วย initial values
    -- หรือใช้ pre-computed keys
    local keys = {}
    for i = 1, n do keys[i] = "key" .. i end  -- pre-compute keys
    
    local t3 = os.clock()
    local ht2 = {}
    for i = 1, n do
        ht2[keys[i]] = i  -- use pre-computed keys
    end
    local t4 = os.clock()
    
    print(string.format("Hash table (pre-computed keys): %.4fms", (t4-t3)*1000))
end

hash_prealloc_demo()

-- Table growth pattern ใน Lua
-- Array part: 1, 2, 4, 8, 16, 32, ... (powers of 2)
-- Hash part: similar doubling

-- ตรวจสอบ table size หลังจาก operations
local function table_size_demo()
    local t = {}
    
    local function print_size(label)
        -- ใน Lua 5.4 ไม่มี direct way ดู allocated size
        -- ใช้ collectgarbage count เป็น proxy
        local kb = collectgarbage("count")
        print(string.format("  %s: GC used = %.1f KB", label, kb))
    end
    
    print_size("empty table")
    
    for i = 1, 100 do t[i] = i end
    print_size("after 100 elements")
    
    for i = 1, 1000 do t[i] = i end
    print_size("after 1000 elements")
    
    for i = 1, 10000 do t[i] = i end
    print_size("after 10000 elements")
end

table_size_demo()
```

---

## 85.5 Avoiding Allocation in Hot Paths

```lua
-- การ allocate memory ใน hot path ทำให้ GC ทำงานบ่อย

-- PROBLEM: allocate ใน loop
local function allocate_in_loop(n)
    local results = {}
    for i = 1, n do
        -- สร้าง table ใหม่ทุก iteration!
        local point = {x = i, y = i * 2}
        results[i] = point.x + point.y
    end
    return results
end

-- SOLUTION: ใช้ local variables แทน
local function no_allocate_in_loop(n)
    local results = {}
    for i = 1, n do
        -- ไม่มี allocation
        results[i] = i + i * 2
    end
    return results
end

-- Benchmark
local N = 1000000

-- Warmup
for _ = 1, 3 do allocate_in_loop(1000) end
for _ = 1, 3 do no_allocate_in_loop(1000) end

local before_gc = collectgarbage("count")
local t1 = os.clock()
allocate_in_loop(N)
local t2 = os.clock()
collectgarbage("collect")
local after_gc = collectgarbage("count")

print(string.format("\nWith allocation: %.3fms, GC diff=%.1f KB",
    (t2-t1)*1000, before_gc - after_gc))

before_gc = collectgarbage("count")
local t3 = os.clock()
no_allocate_in_loop(N)
local t4 = os.clock()
collectgarbage("collect")
after_gc = collectgarbage("count")

print(string.format("Without allocation: %.3fms, GC diff=%.1f KB",
    (t4-t3)*1000, before_gc - after_gc))

-- ตัวอย่างขั้นสูง: Reusing complex objects
local function reuse_demo(n)
    -- Pre-allocate vector
    local vec = {x=0, y=0, z=0}
    
    local total = 0
    for i = 1, n do
        -- Reuse vec (ไม่สร้างใหม่)
        vec.x = i
        vec.y = i * 2
        vec.z = i * 3
        
        -- Use vec
        total = total + vec.x + vec.y + vec.z
    end
    return total
end
```

---

## 85.6 Object Pooling

```lua
-- Object Pool: reuse objects แทนที่จะสร้างใหม่ทุกครั้ง

local ObjectPool = {}
ObjectPool.__index = ObjectPool

function ObjectPool.new(constructor, reset_fn, initial_size)
    local self = setmetatable({}, ObjectPool)
    self.constructor = constructor
    self.reset = reset_fn or function(obj) end
    self.pool = {}
    self.created = 0
    self.reused = 0
    self.active = 0
    
    -- Pre-fill pool
    for i = 1, (initial_size or 0) do
        self.pool[#self.pool+1] = constructor()
        self.created = self.created + 1
    end
    
    return self
end

function ObjectPool:acquire()
    local obj
    if #self.pool > 0 then
        obj = table.remove(self.pool)
        self.reused = self.reused + 1
    else
        obj = self.constructor()
        self.created = self.created + 1
    end
    self.active = self.active + 1
    return obj
end

function ObjectPool:release(obj)
    self.reset(obj)
    self.pool[#self.pool+1] = obj
    self.active = self.active - 1
end

function ObjectPool:stats()
    return {
        pool_size = #self.pool,
        active = self.active,
        created = self.created,
        reused = self.reused,
        reuse_rate = self.created > 0 and 
            (self.reused / (self.created + self.reused) * 100) or 0,
    }
end

-- ตัวอย่าง: Particle system ด้วย object pool
local particle_pool = ObjectPool.new(
    function()
        return {x=0, y=0, vx=0, vy=0, life=0, alive=false}
    end,
    function(p)
        p.x = 0; p.y = 0
        p.vx = 0; p.vy = 0
        p.life = 0; p.alive = false
    end,
    100  -- pre-allocate 100 particles
)

local function spawn_particle()
    local p = particle_pool:acquire()
    p.x = math.random() * 100
    p.y = math.random() * 100
    p.vx = (math.random() - 0.5) * 10
    p.vy = (math.random() - 0.5) * 10
    p.life = math.random(60, 120)  -- frames
    p.alive = true
    return p
end

local active_particles = {}

local function particle_simulation_pooled(frames)
    local total_spawned = 0
    local total_recycled = 0
    
    for frame = 1, frames do
        -- Spawn 5 particles per frame
        for i = 1, 5 do
            local p = spawn_particle()
            active_particles[#active_particles+1] = p
            total_spawned = total_spawned + 1
        end
        
        -- Update and kill
        local alive = {}
        for _, p in ipairs(active_particles) do
            p.x = p.x + p.vx
            p.y = p.y + p.vy
            p.life = p.life - 1
            
            if p.life > 0 then
                alive[#alive+1] = p
            else
                particle_pool:release(p)
                total_recycled = total_recycled + 1
            end
        end
        active_particles = alive
    end
    
    return total_spawned, total_recycled
end

-- Benchmark
local t1 = os.clock()
local spawned, recycled = particle_simulation_pooled(1000)
local t2 = os.clock()

print(string.format("\nParticle simulation (1000 frames)"))
print(string.format("  Time: %.3fms", (t2-t1)*1000))
print(string.format("  Spawned: %d, Recycled: %d", spawned, recycled))

local stats = particle_pool:stats()
print(string.format("  Pool: %d available, %d active", stats.pool_size, stats.active))
print(string.format("  Reuse rate: %.1f%%", stats.reuse_rate))
```

---

## 85.7 Avoiding GC Pressure

```lua
-- GC pressure: การสร้าง objects มากเกินไปทำให้ GC ทำงานบ่อย

-- ดู GC metrics
local function gc_metrics()
    return {
        used_kb = collectgarbage("count"),
        collections = 0,  -- Lua ไม่ expose collection count โดยตรง
    }
end

-- Pattern 1: ใช้ upvalues แทน locals ใน hot functions
local function avoid_gc_with_upvalues()
    -- Pre-allocate reusable objects
    local result_buffer = {x=0, y=0, z=0}  -- reusable
    local temp = {0, 0, 0}  -- reusable array
    
    return function(a, b, c)
        -- Modify in-place แทนการสร้างใหม่
        result_buffer.x = a
        result_buffer.y = b
        result_buffer.z = c
        return result_buffer  -- WARNING: caller must not store this!
    end
end

local get_vec = avoid_gc_with_upvalues()

-- Pattern 2: String interning สำหรับ repeated strings
local intern_cache = {}
local function intern(s)
    if intern_cache[s] then
        return intern_cache[s]
    end
    intern_cache[s] = s
    return s
end

-- Pattern 3: ใช้ integer keys แทน string keys (ประหยัด memory)
local function integer_key_demo()
    local n = 100000
    
    -- String keys: มาก string objects
    local before = collectgarbage("count")
    local t_str = {}
    for i = 1, n do
        t_str["key" .. i] = i  -- สร้าง string "key1", "key2", ... ทุกครั้ง
    end
    local str_mem = collectgarbage("count") - before
    
    -- Integer keys: ไม่มี string objects
    before = collectgarbage("count")
    local t_int = {}
    for i = 1, n do
        t_int[i] = i
    end
    local int_mem = collectgarbage("count") - before
    
    print(string.format("\nString keys memory: %.1f KB", str_mem))
    print(string.format("Integer keys memory: %.1f KB", int_mem))
    print(string.format("String/Integer ratio: %.1fx", str_mem / math.max(int_mem, 0.001)))
end

integer_key_demo()

-- Pattern 4: GC Tuning
local function gc_tuning_demo()
    -- Default settings
    -- pause = 200 (เริ่ม GC เมื่อ memory เพิ่มขึ้น 200% จากจุดก่อนหน้า)
    -- stepmul = 100 (GC ทำงานเร็ว 100x เร็วกว่า allocation rate)
    
    -- สำหรับ low-latency applications:
    collectgarbage("setpause", 110)     -- GC บ่อยขึ้น
    collectgarbage("setstepmul", 200)   -- GC ทำงานเร็วขึ้นต่อ step
    
    -- สำหรับ throughput:
    -- collectgarbage("setpause", 300)   -- GC น้อยลง
    -- collectgarbage("setstepmul", 50)  -- GC ช้าลง แต่รบกวนน้อยลง
    
    -- Manual GC ในช่วงที่ไม่ busy
    local function idle_callback()
        collectgarbage("step", 100)  -- GC step เล็กๆ
    end
    
    -- Generational GC (Lua 5.4)
    -- collectgarbage("generational")  -- switch to gen GC
    -- ดีสำหรับ applications ที่ objects ส่วนมาก short-lived
    
    print("GC tuned for low latency")
end

gc_tuning_demo()
```

---

## 85.8 Avoiding Closure Overhead ใน Hot Loops

```lua
-- Closures มี overhead: upvalue access

-- SLOW: closure ใน hot loop
local function slow_with_closure(t, n)
    local multiplier = 3  -- upvalue
    local result = {}
    
    for i = 1, n do
        result[i] = (function(x) return x * multiplier end)(t[i])
        -- สร้าง closure ใหม่ทุก iteration!
    end
    return result
end

-- FAST: ไม่มี closure
local function fast_no_closure(t, n, multiplier)
    local result = {}
    for i = 1, n do
        result[i] = t[i] * multiplier
    end
    return result
end

-- FAST: closure สร้างครั้งเดียว
local function fast_cached_closure(t, n)
    local multiplier = 3
    local mul = function(x) return x * multiplier end  -- สร้างครั้งเดียว
    
    local result = {}
    for i = 1, n do
        result[i] = mul(t[i])
    end
    return result
end

local N = 100000
local data = {}
for i = 1, N do data[i] = i end

-- Warmup
for _ = 1, 3 do fast_no_closure(data, N, 3) end

local t1 = os.clock()
slow_with_closure(data, N)
local t2 = os.clock()

local t3 = os.clock()
fast_no_closure(data, N, 3)
local t4 = os.clock()

local t5 = os.clock()
fast_cached_closure(data, N)
local t6 = os.clock()

print(string.format("\nClosure in loop: %.3fms", (t2-t1)*1000))
print(string.format("No closure:       %.3fms", (t4-t3)*1000))
print(string.format("Cached closure:   %.3fms", (t6-t5)*1000))
```

---

## 85.9 Local Variable Optimization

```lua
-- Local variables เป็น register access (เร็วที่สุด)
-- Global variables ต้องผ่าน GETTABUP (ช้ากว่า)

-- SLOW: global access
local function slow_global_access(n)
    local sum = 0
    for i = 1, n do
        -- math.sqrt เป็น global: GETTABUP ทุก iteration
        sum = sum + math.sqrt(i) + math.floor(i * 0.5) + math.abs(i - 5000)
    end
    return sum
end

-- FAST: local references
local function fast_local_access(n)
    -- Cache ใน locals
    local sqrt = math.sqrt
    local floor = math.floor
    local abs = math.abs
    
    local sum = 0
    for i = 1, n do
        -- ทุกอย่างเป็น register access
        sum = sum + sqrt(i) + floor(i * 0.5) + abs(i - 5000)
    end
    return sum
end

local N = 1000000

local t1 = os.clock()
slow_global_access(N)
local t2 = os.clock()

local t3 = os.clock()
fast_local_access(N)
local t4 = os.clock()

print(string.format("\nGlobal access: %.3fms", (t2-t1)*1000))
print(string.format("Local access:  %.3fms", (t4-t3)*1000))
print(string.format("Speedup:       %.2fx", (t2-t1)/(t4-t3)))

-- Nested function access optimization
local outer_value = 100

local function make_inner()
    -- ถ้า outer_value ถูก access บ่อยใน inner
    -- cache ไว้ใน parameter
    return function(x)
        return x + outer_value  -- upvalue access ทุกครั้ง
    end
end

local function make_inner_optimized()
    local cached = outer_value  -- copy ครั้งเดียว (local)
    return function(x)
        return x + cached  -- local access (เร็วกว่า upvalue)
    end
end

-- Note: ถ้า outer_value เปลี่ยนค่า make_inner ยังเห็นการเปลี่ยนแปลง
-- แต่ make_inner_optimized ไม่เห็น (เพราะ copy แล้ว)
```

---

## 85.10 Integer vs Float Performance

```lua
-- Lua 5.3+ แยก integer และ float
-- Integer operations ไม่ใช้ FPU = เร็วกว่า

-- ตรวจสอบว่าเราใช้ integer หรือ float
local function check_type(v)
    if math.type then
        return math.type(v)  -- "integer" หรือ "float"
    end
    return type(v)
end

print("\n-- Type check --")
print("1:", check_type(1))        -- integer
print("1.0:", check_type(1.0))    -- float
print("1/1:", check_type(1//1))   -- integer (integer division)
print("1.0//1:", check_type(1.0//1))  -- float

-- Integer arithmetic benchmark
local function int_arithmetic(n)
    local sum = 0
    for i = 1, n do
        sum = sum + i
        sum = sum * 2
        sum = sum // 3  -- integer division
        sum = sum % 1000
    end
    return sum
end

-- Float arithmetic benchmark
local function float_arithmetic(n)
    local sum = 0.0
    for i = 1, n do
        sum = sum + i
        sum = sum * 2.0
        sum = sum / 3.0  -- float division
        sum = sum - math.floor(sum / 1000) * 1000
    end
    return sum
end

local N = 5000000

local t1 = os.clock()
int_arithmetic(N)
local t2 = os.clock()

local t3 = os.clock()
float_arithmetic(N)
local t4 = os.clock()

print(string.format("\nInteger arithmetic: %.3fms", (t2-t1)*1000))
print(string.format("Float arithmetic:   %.3fms", (t4-t3)*1000))

-- ตัวอย่าง: เมื่อต้องเลือก integer vs float
local function choose_int_or_float()
    -- ใช้ integer เมื่อ:
    -- - ค่าเป็น whole numbers
    -- - ต้องการ bitwise operations
    -- - ต้องการ exact arithmetic (ไม่มี floating point error)
    
    -- ใช้ float เมื่อ:
    -- - ต้องการ decimal precision
    -- - ใช้ transcendental functions (sin, cos, sqrt)
    -- - interop กับ C API ที่รับ double
    
    -- CAREFUL: การผสม int/float ทำให้ได้ float
    local i = 10      -- integer
    local f = 3.14    -- float
    local mixed = i + f  -- float! (implicit conversion)
    
    print(string.format("\n%d (int) + %g (float) = %g (%s)",
        i, f, mixed, check_type(mixed)))
end

choose_int_or_float()
```

---

## 85.11 Profiling ด้วยเทคนิคต่างๆ

```lua
-- Manual micro-benchmarking

local function microperf(name, f, n, warmup)
    n = n or 10
    warmup = warmup or 5
    
    -- Warmup
    for i = 1, warmup do f() end
    
    -- Collect samples
    local times = {}
    for i = 1, n do
        local t1 = os.clock()
        f()
        times[i] = os.clock() - t1
    end
    
    -- Statistics
    table.sort(times)
    local sum = 0
    for _, t in ipairs(times) do sum = sum + t end
    
    local min = times[1]
    local max = times[n]
    local avg = sum / n
    local median = times[math.ceil(n/2)]
    
    -- Variance
    local var_sum = 0
    for _, t in ipairs(times) do
        var_sum = var_sum + (t - avg)^2
    end
    local std = math.sqrt(var_sum / n)
    
    -- Remove outliers (trim 10% from each end)
    local trim = math.floor(n * 0.1)
    local trimmed_sum = 0
    local trimmed_count = 0
    for i = trim+1, n-trim do
        trimmed_sum = trimmed_sum + times[i]
        trimmed_count = trimmed_count + 1
    end
    local trimmed_mean = trimmed_count > 0 and trimmed_sum / trimmed_count or avg
    
    print(string.format("%-30s min=%7.4f avg=%7.4f median=%7.4f std=%7.4f trimmed=%7.4f (ms)",
        name,
        min * 1000,
        avg * 1000,
        median * 1000,
        std * 1000,
        trimmed_mean * 1000))
    
    return min, avg, median
end

print("\n=== Micro Benchmarks ===")

local N = 100000
local data = {}
for i = 1, N do data[i] = math.random() end

-- Test 1: ipairs vs numeric for
microperf("ipairs", function()
    local sum = 0
    for i, v in ipairs(data) do sum = sum + v end
end, 10)

microperf("numeric for", function()
    local sum = 0
    for i = 1, N do sum = sum + data[i] end
end, 10)

-- Test 2: string operations
local strs = {}
for i = 1, 1000 do strs[i] = tostring(i) end

microperf("string.len", function()
    local sum = 0
    for _, s in ipairs(strs) do sum = sum + #s end
end, 10)

microperf("string.format %d", function()
    local parts = {}
    for i = 1, 100 do parts[i] = string.format("%d", i) end
end, 10)

-- Test 3: Table operations
microperf("table.insert", function()
    local t = {}
    for i = 1, 1000 do table.insert(t, i) end
end, 10)

microperf("direct index", function()
    local t = {}
    for i = 1, 1000 do t[#t+1] = i end
end, 10)

microperf("pre-indexed", function()
    local t = {}
    for i = 1, 1000 do t[i] = i end
end, 10)
```

---

## 85.12 Lock-Free Data Structures Concepts

```lua
-- Lua เป็น single-threaded ดังนั้น lock-free ไม่จำเป็น
-- แต่ถ้าใช้ coroutines หรือ callback-based code
-- ต้องระวัง "re-entrancy"

-- Re-entrant queue
local ReentrantQueue = {}
ReentrantQueue.__index = ReentrantQueue

function ReentrantQueue.new()
    return setmetatable({
        items = {},
        processing = false,
        deferred = {},
    }, ReentrantQueue)
end

function ReentrantQueue:push(item)
    if self.processing then
        -- ถ้า push ระหว่าง processing: defer
        self.deferred[#self.deferred+1] = item
    else
        self.items[#self.items+1] = item
    end
end

function ReentrantQueue:process(handler)
    if self.processing then return end
    
    self.processing = true
    
    while #self.items > 0 do
        local item = table.remove(self.items, 1)
        handler(item)
        
        -- Process any deferred items
        while #self.deferred > 0 do
            self.items[#self.items+1] = table.remove(self.deferred, 1)
        end
    end
    
    self.processing = false
end

-- ทดสอบ re-entrant queue
local function test_reentrant()
    print("\n=== Re-entrant Queue Demo ===")
    
    local q = ReentrantQueue.new()
    local processed = {}
    
    local function handler(item)
        processed[#processed+1] = item
        
        -- ในขณะ processing ลอง push อีก
        if item == 2 then
            q:push(10)  -- deferred!
            q:push(11)  -- deferred!
        end
    end
    
    q:push(1)
    q:push(2)
    q:push(3)
    
    q:process(handler)
    
    print("Processed order:", table.concat(processed, ", "))
    -- ควรได้: 1, 2, 3, 10, 11 (deferred items หลัง original items)
end

test_reentrant()

-- Atomic counter simulation (สำหรับ metrics ที่ thread-safe concept)
-- ใน Lua เดี่ยว ไม่ต้อง atomic แต่ concept ยังมีประโยชน์

local AtomicCounter = {}
AtomicCounter.__index = AtomicCounter

function AtomicCounter.new(initial)
    return setmetatable({value = initial or 0}, AtomicCounter)
end

function AtomicCounter:increment(by)
    self.value = self.value + (by or 1)
    return self.value
end

function AtomicCounter:decrement(by)
    self.value = self.value - (by or 1)
    return self.value
end

function AtomicCounter:get()
    return self.value
end

function AtomicCounter:compare_and_swap(expected, new_value)
    if self.value == expected then
        self.value = new_value
        return true
    end
    return false
end
```

---

## 85.13 Flamegraph Analysis Concepts

```lua
-- Flamegraph analysis: visual profiling

-- สร้าง profiler ที่ generate flamegraph data

local FlameProfiler = {}
FlameProfiler.__index = FlameProfiler

function FlameProfiler.new()
    local self = setmetatable({}, FlameProfiler)
    self.samples = {}
    self.call_stack = {}
    self.call_counts = {}
    self.sampling = false
    return self
end

function FlameProfiler:start(interval_ms)
    interval_ms = interval_ms or 1
    self.sampling = true
    self.start_time = os.clock()
    
    -- ใช้ debug hook เพื่อ sample call stack
    local function hook(event)
        if event == "call" then
            local info = debug.getinfo(2, "Sn")
            local name = (info.name or "?") .. "@" .. (info.short_src or "?")
            self.call_stack[#self.call_stack+1] = name
        elseif event == "return" then
            if #self.call_stack > 0 then
                table.remove(self.call_stack)
            end
        end
        
        -- Sample current stack (every N calls)
        if #self.samples < 10000 then  -- limit samples
            local stack_key = table.concat(self.call_stack, ";")
            if stack_key ~= "" then
                self.call_counts[stack_key] = 
                    (self.call_counts[stack_key] or 0) + 1
            end
        end
    end
    
    debug.sethook(hook, "cr", 100)  -- sample every 100 instructions
end

function FlameProfiler:stop()
    debug.sethook()
    self.sampling = false
    self.elapsed = os.clock() - (self.start_time or 0)
end

function FlameProfiler:generate_flamegraph_data()
    -- Output in collapsed format สำหรับ flamegraph.pl
    local lines = {}
    for stack, count in pairs(self.call_counts) do
        lines[#lines+1] = stack .. " " .. count
    end
    table.sort(lines)
    return table.concat(lines, "\n")
end

function FlameProfiler:top_functions()
    -- รวม counts ต่อ function
    local func_counts = {}
    for stack, count in pairs(self.call_counts) do
        -- Function ล่าสุดใน stack
        local func = stack:match("([^;]+)$") or stack
        func_counts[func] = (func_counts[func] or 0) + count
    end
    
    local sorted = {}
    for func, count in pairs(func_counts) do
        sorted[#sorted+1] = {func=func, count=count}
    end
    table.sort(sorted, function(a, b) return a.count > b.count end)
    
    return sorted
end

-- Simple call profiler (เร็วกว่า FlameProfiler)
local CallProfiler = {}
CallProfiler.__index = CallProfiler

function CallProfiler.new()
    return setmetatable({
        data = {},
        stack = {},
    }, CallProfiler)
end

function CallProfiler:start()
    local self = self
    local function hook(event)
        local info = debug.getinfo(2, "Sn")
        if not info then return end
        
        local key = (info.name or "?") .. "@" .. (info.short_src or "?") .. ":" .. 
                    (info.linedefined or 0)
        
        if event == "call" then
            self.stack[#self.stack+1] = {key=key, t=os.clock()}
            
        elseif event == "return" then
            if #self.stack > 0 then
                local frame = table.remove(self.stack)
                local elapsed = os.clock() - frame.t
                
                local entry = self.data[frame.key]
                if not entry then
                    entry = {count=0, total=0, self_time=0}
                    self.data[frame.key] = entry
                end
                entry.count = entry.count + 1
                entry.total = entry.total + elapsed
                
                -- self time (ลบ child time)
                -- TODO: implement properly
                entry.self_time = entry.self_time + elapsed
            end
        end
    end
    
    debug.sethook(hook, "cr")
end

function CallProfiler:stop()
    debug.sethook()
end

function CallProfiler:report(top_n)
    top_n = top_n or 20
    
    local sorted = {}
    for key, entry in pairs(self.data) do
        sorted[#sorted+1] = {
            key = key,
            count = entry.count,
            total = entry.total,
            avg = entry.total / entry.count,
        }
    end
    
    table.sort(sorted, function(a, b) return a.total > b.total end)
    
    print(string.format("\n%-40s %8s %8s %10s", 
        "Function", "Count", "Total(ms)", "Avg(ms)"))
    print(string.rep("-", 70))
    
    for i = 1, math.min(top_n, #sorted) do
        local e = sorted[i]
        -- Show only non-trivial functions
        if e.total > 0.0001 then
            local short_key = e.key:sub(1, 38)  -- truncate
            print(string.format("%-40s %8d %8.2f %10.4f",
                short_key, e.count, e.total*1000, e.avg*1000))
        end
    end
end
```

---

## 85.14 NUMA Awareness Concepts

```lua
-- NUMA (Non-Uniform Memory Access) ใน modern servers
-- Memory access ถูกกว่าถ้า access memory บน NUMA node เดียวกัน

-- ใน Lua ไม่สามารถควบคุม NUMA โดยตรง
-- แต่ใน multi-process Lua (nginx+lua, etc.) ต้องคำนึงถึง

-- Concept:
--[[
NUMA Node 0: CPU0-CPU15 + Memory0 (fast)
NUMA Node 1: CPU16-CPU31 + Memory1 (fast)

CPU0 access Memory0: 100ns (local)
CPU0 access Memory1: 150ns (remote - 1.5x slower)
]]

-- สำหรับ single-process Lua:
-- Pin process ไปยัง NUMA node ด้วย numactl (Linux)
-- numactl --cpunodebind=0 --membind=0 lua program.lua

-- ใน Lua: จัดการ memory locality ด้วย data structures

-- ตัวอย่าง: จัดกลุ่ม data ที่ access พร้อมกัน

-- BAD: data ที่ใช้พร้อมกันกระจายอยู่ใน heap
local function scattered_access(n)
    local a = {}
    local b = {}
    local c = {}
    
    -- สร้าง a, b, c แยกกัน (อาจกระจายใน memory)
    for i = 1, n do a[i] = i end
    for i = 1, n do b[i] = i * 2 end
    for i = 1, n do c[i] = i * 3 end
    
    -- Access พร้อมกัน
    local sum = 0
    for i = 1, n do
        sum = sum + a[i] + b[i] + c[i]
    end
    return sum
end

-- GOOD: data ที่ใช้พร้อมกันอยู่ใกล้กัน
local function locality_access(n)
    -- เก็บ a, b, c ไว้ใกล้กัน ใน flat array
    local data = {}
    for i = 1, n do
        local base = (i-1) * 3 + 1
        data[base]   = i        -- a
        data[base+1] = i * 2    -- b
        data[base+2] = i * 3    -- c
    end
    
    local sum = 0
    for i = 1, n do
        local base = (i-1) * 3 + 1
        sum = sum + data[base] + data[base+1] + data[base+2]
    end
    return sum
end

print("\n=== Memory Locality Test ===")
local N = 100000

local t1 = os.clock()
scattered_access(N)
local t2 = os.clock()

local t3 = os.clock()
locality_access(N)
local t4 = os.clock()

print(string.format("Scattered: %.3fms", (t2-t1)*1000))
print(string.format("Locality:  %.3fms", (t4-t3)*1000))
```

---

## 85.15 Micro-Benchmarking Methodology

```lua
-- หลักการ benchmarking ที่ถูกต้อง

local Benchmark = {}
Benchmark.__index = Benchmark

function Benchmark.new()
    return setmetatable({
        results = {},
    }, Benchmark)
end

function Benchmark:add(name, f, options)
    options = options or {}
    
    local result = {
        name = name,
        func = f,
        warmup = options.warmup or 10,
        iterations = options.iterations or 100,
        times = {},
    }
    
    self.results[#self.results+1] = result
    return self
end

function Benchmark:run()
    for _, bench in ipairs(self.results) do
        -- Warmup
        for i = 1, bench.warmup do
            bench.func()
        end
        
        -- Force GC before measurement
        collectgarbage("collect")
        collectgarbage("collect")
        
        -- Measure
        for i = 1, bench.iterations do
            local t1 = os.clock()
            bench.func()
            bench.times[i] = os.clock() - t1
        end
    end
end

function Benchmark:report()
    print(string.format("\n%-35s %8s %8s %8s %8s %8s",
        "Name", "Min", "P25", "P50", "P75", "P95"))
    print(string.rep("-", 80))
    
    for _, bench in ipairs(self.results) do
        table.sort(bench.times)
        local n = #bench.times
        
        local function percentile(p)
            local idx = math.max(1, math.ceil(n * p / 100))
            return bench.times[idx] * 1000000  -- microseconds
        end
        
        print(string.format("%-35s %8.1f %8.1f %8.1f %8.1f %8.1f μs",
            bench.name:sub(1, 35),
            percentile(0),
            percentile(25),
            percentile(50),
            percentile(75),
            percentile(95)))
    end
end

function Benchmark:compare(base_name, compare_name)
    local base, comp
    for _, b in ipairs(self.results) do
        if b.name == base_name then base = b end
        if b.name == compare_name then comp = b end
    end
    
    if not base or not comp then return end
    
    table.sort(base.times)
    table.sort(comp.times)
    
    local base_median = base.times[math.ceil(#base.times/2)]
    local comp_median = comp.times[math.ceil(#comp.times/2)]
    
    local speedup = base_median / comp_median
    
    print(string.format("\n%s vs %s: %.2fx %s",
        compare_name, base_name,
        speedup > 1 and speedup or 1/speedup,
        speedup > 1 and "faster" or "slower"))
end

-- ตัวอย่างการใช้ Benchmark suite
local bench = Benchmark.new()

local N = 50000
local data = {}
for i = 1, N do data[i] = math.random() * 100 end

bench:add("ipairs sum", function()
    local sum = 0
    for _, v in ipairs(data) do sum = sum + v end
end, {warmup=20, iterations=50})

bench:add("numeric for sum", function()
    local sum = 0
    for i = 1, N do sum = sum + data[i] end
end, {warmup=20, iterations=50})

bench:add("while loop sum", function()
    local sum = 0
    local i = 1
    while i <= N do
        sum = sum + data[i]
        i = i + 1
    end
end, {warmup=20, iterations=50})

bench:run()
bench:report()
bench:compare("ipairs sum", "numeric for sum")
```

---

## 85.16 Advanced Performance Patterns

```lua
-- Pattern 1: Lazy evaluation
local function lazy(compute_fn)
    local computed = false
    local value = nil
    
    return function()
        if not computed then
            value = compute_fn()
            computed = true
        end
        return value
    end
end

local expensive_result = lazy(function()
    -- Simulate expensive computation
    local sum = 0
    for i = 1, 1000000 do sum = sum + i end
    return sum
end)

-- ไม่ compute จนกว่าจะ access
-- expensive_result()  -- ครั้งแรก: compute
-- expensive_result()  -- ครั้งหลัง: return cached

-- Pattern 2: Memoization
local function memoize(f, key_fn)
    local cache = {}
    key_fn = key_fn or function(...) 
        local parts = {...}
        for i, v in ipairs(parts) do parts[i] = tostring(v) end
        return table.concat(parts, ",")
    end
    
    return function(...)
        local key = key_fn(...)
        if cache[key] == nil then
            cache[key] = f(...)
        end
        return cache[key]
    end
end

-- Memoized fibonacci
local fib
fib = memoize(function(n)
    if n <= 1 then return n end
    return fib(n-1) + fib(n-2)
end)

local t1 = os.clock()
print("\nfib(40):", fib(40))
print("Time:", (os.clock()-t1)*1000, "ms")

t1 = os.clock()
print("fib(40) cached:", fib(40))
print("Time:", (os.clock()-t1)*1000000, "μs")  -- microseconds

-- Pattern 3: Batch processing
local function batch_processor(batch_size, process_fn)
    local batch = {}
    local total_processed = 0
    
    return {
        add = function(item)
            batch[#batch+1] = item
            if #batch >= batch_size then
                process_fn(batch)
                total_processed = total_processed + #batch
                batch = {}
            end
        end,
        flush = function()
            if #batch > 0 then
                process_fn(batch)
                total_processed = total_processed + #batch
                batch = {}
            end
        end,
        stats = function()
            return {processed = total_processed, pending = #batch}
        end,
    }
end

-- ใช้ batch processor
local db_batch = batch_processor(100, function(items)
    -- Simulate DB bulk insert
    -- ดีกว่า insert ทีละรายการ
    -- print(string.format("Bulk insert %d items", #items))
end)

for i = 1, 1000 do
    db_batch.add({id=i, value=i*2})
end
db_batch.flush()

local stats = db_batch.stats()
print(string.format("\nBatch processor: %d processed, %d pending",
    stats.processed, stats.pending))
```

---

## 85.17 Compile-Time vs Runtime Computations

```lua
-- ย้าย computation จาก runtime ไปยัง "compile time" (load time)

-- BAD: compute every call
local function sin_table_runtime(angle_deg)
    return math.sin(angle_deg * math.pi / 180)
end

-- GOOD: precompute table ณ load time
local SIN_TABLE = {}
local COS_TABLE = {}
for i = 0, 360 do
    SIN_TABLE[i] = math.sin(i * math.pi / 180)
    COS_TABLE[i] = math.cos(i * math.pi / 180)
end

local function sin_table_lookup(angle_deg)
    return SIN_TABLE[angle_deg % 361]
end

-- Benchmark
local N = 1000000

local t1 = os.clock()
local sum1 = 0
for i = 1, N do
    sum1 = sum1 + sin_table_runtime(i % 360)
end
local t2 = os.clock()

local t3 = os.clock()
local sum2 = 0
for i = 1, N do
    sum2 = sum2 + sin_table_lookup(i % 360)
end
local t4 = os.clock()

print(string.format("\nmath.sin call: %.3fms", (t2-t1)*1000))
print(string.format("Table lookup:  %.3fms", (t4-t3)*1000))
print(string.format("Speedup:       %.1fx", (t2-t1)/(t4-t3)))

-- ตัวอย่าง: Precomputed hash values
local function build_keyword_lookup()
    local keywords = {
        "if", "then", "else", "end", "while", "do",
        "for", "function", "return", "local", "nil",
        "true", "false", "and", "or", "not", "break",
        "repeat", "until", "goto", "in",
    }
    
    -- สร้าง hash set ณ load time
    local lookup = {}
    for _, kw in ipairs(keywords) do
        lookup[kw] = true
    end
    
    return lookup
end

local KEYWORDS = build_keyword_lookup()

local function is_keyword(word)
    return KEYWORDS[word] == true
end

print("\nis_keyword tests:")
print("if:", is_keyword("if"))        -- true
print("foo:", is_keyword("foo"))      -- false
print("while:", is_keyword("while"))  -- true
```

---

## 85.18 Bit Manipulation สำหรับ Performance

```lua
-- Bitwise operations สำหรับ fast math

-- ใน Lua 5.3+: &, |, ~, <<, >>
-- ใน LuaJIT: bit.band, bit.bor, bit.bxor, bit.bnot, bit.lshift, bit.rshift

-- Fast power of 2 check
local function is_power_of_two(n)
    return n > 0 and (n & (n - 1)) == 0
end

print("\n=== Bit Manipulation ===")
for _, n in ipairs({1, 2, 3, 4, 7, 8, 15, 16}) do
    print(string.format("  %3d: power of 2 = %s", n, tostring(is_power_of_two(n))))
end

-- Fast integer log2
local function ilog2(n)
    local result = 0
    while n > 1 do
        n = n >> 1
        result = result + 1
    end
    return result
end

print("\nlog2 examples:")
for _, n in ipairs({1, 2, 4, 8, 16, 32, 64, 128, 256}) do
    print(string.format("  log2(%d) = %d", n, ilog2(n)))
end

-- Fast modulo (power of 2)
local function fast_mod_power2(n, m)
    -- Works only when m is power of 2
    return n & (m - 1)
end

-- Verify
print("\nFast mod:")
for i = 0, 10 do
    local fast = fast_mod_power2(i, 8)
    local slow = i % 8
    assert(fast == slow, "mismatch!")
    print(string.format("  %d %% 8 = %d", i, fast))
end

-- Bit flags
local FLAGS = {
    ALIVE   = 1 << 0,  -- bit 0
    VISIBLE = 1 << 1,  -- bit 1
    MOVING  = 1 << 2,  -- bit 2
    SOLID   = 1 << 3,  -- bit 3
}

local function create_entity(alive, visible, moving, solid)
    local flags = 0
    if alive   then flags = flags | FLAGS.ALIVE end
    if visible then flags = flags | FLAGS.VISIBLE end
    if moving  then flags = flags | FLAGS.MOVING end
    if solid   then flags = flags | FLAGS.SOLID end
    return flags
end

local function has_flag(entity, flag)
    return (entity & flag) ~= 0
end

local function set_flag(entity, flag)
    return entity | flag
end

local function clear_flag(entity, flag)
    return entity & ~flag
end

-- ใช้ flags
local e1 = create_entity(true, true, false, true)
print(string.format("\nEntity flags: 0x%x", e1))
print("  Alive:", has_flag(e1, FLAGS.ALIVE))
print("  Moving:", has_flag(e1, FLAGS.MOVING))

e1 = set_flag(e1, FLAGS.MOVING)
print("After set MOVING:", has_flag(e1, FLAGS.MOVING))

e1 = clear_flag(e1, FLAGS.ALIVE)
print("After clear ALIVE:", has_flag(e1, FLAGS.ALIVE))
```

---

## 85.19 ตัวอย่างสมบูรณ์: High-Performance Game Loop

```lua
-- High-performance game loop ที่ใช้เทคนิคทั้งหมด

local GameEngine = {}
GameEngine.__index = GameEngine

function GameEngine.new()
    local self = setmetatable({}, GameEngine)
    
    -- Pre-allocate systems
    self.entity_count = 0
    self.max_entities = 10000
    
    -- SOA layout สำหรับ entities
    self.pos_x    = {}
    self.pos_y    = {}
    self.vel_x    = {}
    self.vel_y    = {}
    self.health   = {}
    self.flags    = {}
    self.alive    = {}
    
    -- Pre-allocate arrays
    for i = 1, self.max_entities do
        self.pos_x[i] = 0
        self.pos_y[i] = 0
        self.vel_x[i] = 0
        self.vel_y[i] = 0
        self.health[i] = 0
        self.flags[i]  = 0
        self.alive[i]  = false
    end
    
    -- Free entity list (สำหรับ O(1) allocation)
    self.free_list = {}
    for i = self.max_entities, 1, -1 do
        self.free_list[#self.free_list+1] = i
    end
    
    -- Systems (functions ที่ cache ไว้)
    self.sqrt   = math.sqrt
    self.floor  = math.floor
    self.random = math.random
    
    -- Stats
    self.frame = 0
    self.entity_updates = 0
    
    return self
end

function GameEngine:create_entity(x, y, vx, vy, health)
    if #self.free_list == 0 then
        return nil, "Max entities reached"
    end
    
    local id = table.remove(self.free_list)
    
    self.pos_x[id]  = x or 0
    self.pos_y[id]  = y or 0
    self.vel_x[id]  = vx or 0
    self.vel_y[id]  = vy or 0
    self.health[id] = health or 100
    self.flags[id]  = 0x01  -- ALIVE flag
    self.alive[id]  = true
    self.entity_count = self.entity_count + 1
    
    return id
end

function GameEngine:destroy_entity(id)
    if not self.alive[id] then return end
    
    self.alive[id]  = false
    self.flags[id]  = 0
    self.entity_count = self.entity_count - 1
    
    -- Return to free list
    self.free_list[#self.free_list+1] = id
end

-- Update systems (ใช้ SOA สำหรับ cache efficiency)
function GameEngine:update_physics(dt)
    local px = self.pos_x
    local py = self.pos_y
    local vx = self.vel_x
    local vy = self.vel_y
    local alive = self.alive
    local max = self.max_entities
    
    -- Tight loop: ไม่มี table creation, ไม่มี function calls
    for i = 1, max do
        if alive[i] then
            px[i] = px[i] + vx[i] * dt
            py[i] = py[i] + vy[i] * dt
            
            -- Boundary check
            if px[i] < 0 then
                px[i] = 0
                vx[i] = -vx[i] * 0.8  -- bounce
            elseif px[i] > 1000 then
                px[i] = 1000
                vx[i] = -vx[i] * 0.8
            end
            
            if py[i] < 0 then
                py[i] = 0
                vy[i] = -vy[i] * 0.8
            elseif py[i] > 1000 then
                py[i] = 1000
                vy[i] = -vy[i] * 0.8
            end
        end
    end
    
    self.entity_updates = self.entity_updates + self.entity_count
end

function GameEngine:update_health(dt)
    local health = self.health
    local alive = self.alive
    local max = self.max_entities
    
    for i = 1, max do
        if alive[i] then
            health[i] = health[i] - dt * 5  -- drain 5 HP/s
            if health[i] <= 0 then
                self:destroy_entity(i)
            end
        end
    end
end

function GameEngine:run_simulation(entity_count, frames)
    math.randomseed(42)
    
    -- Spawn entities
    for i = 1, entity_count do
        self:create_entity(
            math.random(0, 1000),
            math.random(0, 1000),
            (math.random() - 0.5) * 100,
            (math.random() - 0.5) * 100,
            math.random(50, 200)
        )
    end
    
    local dt = 1/60  -- 60 FPS
    local t1 = os.clock()
    
    for frame = 1, frames do
        self.frame = frame
        self:update_physics(dt)
        self:update_health(dt)
        
        -- Respawn dead entities
        if self.entity_count < entity_count // 2 then
            local to_spawn = math.min(100, entity_count - self.entity_count)
            for i = 1, to_spawn do
                self:create_entity(
                    math.random(0, 1000),
                    math.random(0, 1000),
                    (math.random() - 0.5) * 100,
                    (math.random() - 0.5) * 100,
                    math.random(50, 200)
                )
            end
        end
    end
    
    local elapsed = os.clock() - t1
    
    print(string.format("\n=== Game Engine Results ==="))
    print(string.format("Entities: %d (max: %d)", self.entity_count, entity_count))
    print(string.format("Frames: %d", frames))
    print(string.format("Total entity updates: %d", self.entity_updates))
    print(string.format("Time: %.3fs (%.1f FPS)", elapsed, frames/elapsed))
    print(string.format("Entity-updates/sec: %.0fK",
        self.entity_updates/elapsed/1000))
end

-- Run
local engine = GameEngine.new()
engine:run_simulation(5000, 600)  -- 5000 entities, 10 seconds at 60fps
```

---

## 85.20 Summary of Performance Techniques

```lua
-- สรุปเทคนิค performance ทั้งหมด

local techniques = {
    {
        category = "Memory Layout",
        techniques = {
            "SOA instead of AOS for bulk data",
            "Keep hot data in same cache line",
            "Integer keys > string keys",
            "Pre-allocate arrays and tables",
        }
    },
    {
        category = "Variable Access",
        techniques = {
            "Cache globals as locals",
            "Cache method references (math.sqrt → sqrt)",
            "Use parameters vs upvalues in hot functions",
            "Minimize variable count in tight loops",
        }
    },
    {
        category = "Allocation",
        techniques = {
            "Avoid allocating in hot paths",
            "Use object pools for frequently created objects",
            "Reuse tables by clearing fields instead of recreating",
            "Use local vars instead of temporary tables",
        }
    },
    {
        category = "String Operations",
        techniques = {
            "Use table.concat instead of .. in loops",
            "Pre-compute strings (tostring, format) if repeated",
            "Use plain string.find for literal searches",
            "Avoid large string operations in hot paths",
        }
    },
    {
        category = "Loop Optimization",
        techniques = {
            "Numeric for > generic for (ipairs/pairs)",
            "Hoist invariants out of loops",
            "Minimize function calls in tight loops",
            "Pre-compute length (#t) before loops if constant",
        }
    },
    {
        category = "Integer vs Float",
        techniques = {
            "Use integer arithmetic when possible",
            "Avoid mixing integer and float (implicit conversion)",
            "Use // for integer division",
            "Bitwise ops for power-of-2 operations",
        }
    },
    {
        category = "GC Control",
        techniques = {
            "Minimize allocations (fewer GC pauses)",
            "Tune GC pause/stepmul for latency vs throughput",
            "Force GC before latency-sensitive sections",
            "Use generational GC for short-lived objects",
        }
    },
    {
        category = "Compile-Time Work",
        techniques = {
            "Precompute lookup tables at load time",
            "Memoize pure functions",
            "Use lazy evaluation for expensive computations",
            "Pre-compile patterns and regexes",
        }
    },
    {
        category = "LuaJIT Specific",
        techniques = {
            "Warmup before benchmarking",
            "Maintain type stability in hot functions",
            "Use FFI for numeric-heavy operations",
            "Avoid NYI operations in critical paths",
        }
    },
}

print("\n========================================")
print("  HIGH-PERFORMANCE LUA TECHNIQUES")
print("========================================")

for _, cat in ipairs(techniques) do
    print(string.format("\n[%s]", cat.category))
    for i, tech in ipairs(cat.techniques) do
        print(string.format("  %d. %s", i, tech))
    end
end

print("\n\nGeneral Rules:")
print("1. Measure first, optimize second")
print("2. Focus on algorithmic improvements (O(n²) → O(n log n))")
print("3. Profile to find ACTUAL bottlenecks")
print("4. Test after each optimization (correctness!)")
print("5. Document why you made the optimization")
```

---

## แบบฝึกหัด

1. Profile ฟังก์ชัน Fibonacci แบบ recursive และ iterative เปรียบเทียบ performance

2. สร้าง string builder class ที่มีประสิทธิภาพสูง รองรับ `append`, `prepend`, `insert`, และ `build`

3. Implement LRU Cache ที่มีประสิทธิภาพสูงด้วย doubly-linked list และ hash map

4. เขียน particle system ที่ทำงานได้ที่ 60fps กับ 10,000+ particles

5. สร้าง memory-efficient sparse matrix implementation

6. Implement quicksort ใน Lua ที่เร็วกว่า `table.sort` ด้วย custom comparator

7. เขียน JSON parser ที่ใช้ทั้ง SOA layout และ object pooling

8. ออกแบบ cache-oblivious matrix multiplication algorithm

9. Implement skip list ที่มี O(log n) operations

10. สร้าง micro-benchmark suite ที่วัดผลได้แม่นยำโดยคำนึงถึง JIT warmup, GC pauses, และ statistical significance

---

## สรุป

ในบทนี้เราได้เรียนรู้ high-performance Lua อย่างครอบคลุม:

- **CPU Cache Effects**: ทำความเข้าใจ memory hierarchy และ access patterns
- **Memory Layout**: SOA vs AOS, cache-friendly data structures
- **String Optimization**: interning, building strategies
- **Table Pre-allocation**: ลด rehashing overhead
- **Avoiding Allocation**: object pools, reuse patterns
- **GC Control**: tuning, reducing pressure
- **Closure Overhead**: เมื่อไหร่ต้องระวัง
- **Local Variables**: cache globals, register vs upvalue
- **Integer vs Float**: type-specific optimizations
- **Profiling**: micro-benchmarking methodology
- **Lock-free Concepts**: re-entrancy, atomic operations
- **Bit Manipulation**: fast math tricks
- **Game Loop**: integrated example สมบูรณ์

Performance optimization เป็น iterative process ที่ต้องการ:
1. **Measure**: รู้ว่าอะไรช้า
2. **Understand**: รู้ว่าทำไมถึงช้า
3. **Optimize**: แก้ปัญหา
4. **Verify**: ตรวจสอบว่าเร็วขึ้นและยังถูกต้อง
5. **Repeat**: ทำซ้ำสำหรับ next bottleneck

จำไว้ว่า "Premature optimization is the root of all evil" - Donald Knuth
แต่ "We should forget about small efficiencies, say about 97% of the time; premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%."
