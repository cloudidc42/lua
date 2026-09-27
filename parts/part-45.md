# บทที่ 45: LuaJIT - Just-In-Time Compiler

## บทนำ

LuaJIT เป็น implementation ของ Lua ที่มาพร้อมกับ Just-In-Time (JIT) compiler ทำให้โค้ด Lua ทำงานได้เร็วกว่า standard Lua interpreter มาก โดยเฉพาะโค้ดที่เกี่ยวกับการคำนวณตัวเลข สามารถเร็วได้ 2-5 เท่าหรือมากกว่า บางกรณีเร็วกว่า C ที่ไม่ได้ optimize ด้วย

LuaJIT พัฒนาโดย Mike Pall และ compatible กับ Lua 5.1 เป็นหลัก แต่มี extension หลายอย่างที่ทำให้ทรงพลังมากขึ้น โดยเฉพาะ **FFI library** ที่ทำให้เรียก C functions ได้โดยตรงโดยไม่ต้องเขียน C binding

> **บทก่อนหน้า**: [บทที่ 44: Embedding Lua in C Applications](part-44.md)

---

## 45.1 LuaJIT vs Standard Lua

### ตัวอย่างที่ 1: ความแตกต่างระหว่าง LuaJIT กับ Lua 5.4

```lua
-- compare_versions.lua
-- ตรวจสอบว่ากำลังรัน LuaJIT หรือ standard Lua

-- ตรวจสอบ version
print("Lua version:", _VERSION)

-- ตรวจสอบว่ามี jit module หรือเปล่า (เฉพาะ LuaJIT)
if jit then
    print("Running on LuaJIT:", jit.version)
    print("JIT status:", jit.status())
    print("Architecture:", jit.arch)
    print("OS:", jit.os)
else
    print("Running on standard Lua")
end

-- ตัวอย่างผลลัพธ์:
-- Running on LuaJIT: LuaJIT 2.1.0-beta3
-- JIT status: true  SSE2 SSE3 SSE4.1 BMI2 fold cse dce fwd dse narrow loop abc sink fuse
-- Architecture: x64
-- OS: Linux
```

### ตัวอย่างที่ 2: Benchmark เปรียบเทียบความเร็ว

```lua
-- benchmark.lua
-- เปรียบเทียบความเร็วของ LuaJIT vs standard Lua

local function fibonacci(n)
    if n < 2 then return n end
    return fibonacci(n - 1) + fibonacci(n - 2)
end

local function benchmark(name, func, ...)
    local start = os.clock()
    local result = func(...)
    local elapsed = os.clock() - start
    print(string.format("%s: %.4f seconds (result=%s)", name, elapsed, tostring(result)))
    return elapsed
end

-- Fibonacci แบบ recursive (เหมาะกับ JIT optimization)
benchmark("fibonacci(35)", fibonacci, 35)

-- Loop นับล้านครั้ง
local function count_loop(n)
    local sum = 0
    for i = 1, n do
        sum = sum + i
    end
    return sum
end

benchmark("count_loop(10M)", count_loop, 10000000)

-- Floating point operations
local function fp_ops(n)
    local x = 1.0
    for i = 1, n do
        x = x * 1.0000001 + 0.000001
    end
    return x
end

benchmark("fp_ops(10M)", fp_ops, 10000000)
```

---

## 45.2 การติดตั้ง LuaJIT

### ตัวอย่างที่ 3: วิธีติดตั้ง LuaJIT บน Ubuntu/Debian

```bash
# ติดตั้งผ่าน package manager
sudo apt-get update
sudo apt-get install luajit

# ตรวจสอบ version
luajit -v
# LuaJIT 2.1.0-beta3 -- Copyright (C) 2005-2017 Mike Pall.

# ติดตั้งจาก source (version ล่าสุด)
git clone https://github.com/LuaJIT/LuaJIT.git
cd LuaJIT
make
sudo make install

# ทดสอบ
luajit -e "print('Hello from LuaJIT!')"
```

### ตัวอย่างที่ 4: ติดตั้งบน macOS และ Windows

```bash
# macOS ด้วย Homebrew
brew install luajit

# Windows ด้วย Scoop
scoop install luajit

# Windows ด้วย Chocolatey
choco install luajit

# ตรวจสอบการติดตั้ง
luajit -v
luajit -e "print(jit.version)"
```

### ตัวอย่างที่ 5: การรัน scripts ด้วย LuaJIT

```bash
# รัน script ธรรมดา
luajit myscript.lua

# รันพร้อมเปิด JIT verbosely
luajit -jv myscript.lua

# รันพร้อม dump JIT trace
luajit -jdump myscript.lua

# รันโดยปิด JIT (เพื่อ debug)
luajit -joff myscript.lua

# Interactive mode
luajit
```

---

## 45.3 FFI Library พื้นฐาน

FFI (Foreign Function Interface) เป็น library ที่ทรงพลังที่สุดของ LuaJIT ทำให้เรียก C functions และใช้ C data types ได้โดยตรง

### ตัวอย่างที่ 6: ffi.cdef - ประกาศ C declarations

```lua
-- ffi_basics.lua
local ffi = require("ffi")

-- ประกาศ C functions และ types ที่จะใช้
ffi.cdef[[
    /* Standard C library functions */
    int printf(const char *fmt, ...);
    void *malloc(size_t size);
    void free(void *ptr);
    size_t strlen(const char *s);
    char *strcpy(char *dest, const char *src);
    
    /* Math functions */
    double sin(double x);
    double cos(double x);
    double sqrt(double x);
    double pow(double x, double y);
    
    /* Time functions */
    typedef long time_t;
    time_t time(time_t *t);
    
    /* Custom struct */
    typedef struct {
        int x;
        int y;
        int width;
        int height;
    } Rect;
]]

-- เรียกใช้ printf ผ่าน FFI
ffi.C.printf("Hello from FFI! Pi = %.6f\n", math.pi)

-- เรียก math functions
local angle = math.pi / 4
print(string.format("sin(45°) = %.6f", ffi.C.sin(angle)))
print(string.format("cos(45°) = %.6f", ffi.C.cos(angle)))
print(string.format("sqrt(2) = %.6f", ffi.C.sqrt(2.0)))
```

### ตัวอย่างที่ 7: ffi.load - โหลด shared library

```lua
-- ffi_load.lua
local ffi = require("ffi")

-- โหลด math library
local m = ffi.load("m")  -- libm.so บน Linux

ffi.cdef[[
    double sin(double x);
    double cos(double x);
    double tan(double x);
    double atan2(double y, double x);
    double exp(double x);
    double log(double x);
]]

-- ใช้งาน math functions
print("sin(PI/2) =", m.sin(math.pi / 2))    -- 1.0
print("cos(0) =", m.cos(0))                  -- 1.0
print("atan2(1,1) =", m.atan2(1, 1))         -- 0.785... (pi/4)

-- โหลด zlib (compression library)
local ok, zlib = pcall(ffi.load, "z")
if ok then
    ffi.cdef[[
        const char *zlibVersion(void);
    ]]
    print("zlib version:", ffi.string(zlib.zlibVersion()))
else
    print("zlib not available:", zlib)
end
```

### ตัวอย่างที่ 8: ffi.new - สร้าง C data types

```lua
-- ffi_new.lua
local ffi = require("ffi")

ffi.cdef[[
    typedef struct {
        float x, y, z;
    } Vec3;
    
    typedef struct {
        uint8_t r, g, b, a;
    } Color;
    
    typedef struct Node {
        int value;
        struct Node *next;
    } Node;
]]

-- สร้าง struct
local v = ffi.new("Vec3", {x = 1.0, y = 2.0, z = 3.0})
print(string.format("Vector: (%.1f, %.1f, %.1f)", v.x, v.y, v.z))

-- แก้ไขค่า
v.x = v.x + 1
v.y = v.y * 2
print(string.format("Modified: (%.1f, %.1f, %.1f)", v.x, v.y, v.z))

-- สร้าง Color
local red = ffi.new("Color", {r = 255, g = 0, b = 0, a = 255})
print(string.format("Color: RGB(%d, %d, %d)", red.r, red.g, red.b))

-- สร้าง array
local arr = ffi.new("int[10]")
for i = 0, 9 do  -- C arrays เริ่มที่ index 0
    arr[i] = i * i
end

io.write("Squares: ")
for i = 0, 9 do
    io.write(arr[i] .. " ")
end
print()

-- sizeof
print("sizeof Vec3:", ffi.sizeof("Vec3"))
print("sizeof Color:", ffi.sizeof("Color"))
print("sizeof int[10]:", ffi.sizeof("int[10]"))
```

### ตัวอย่างที่ 9: ffi.string - แปลง C strings

```lua
-- ffi_string.lua
local ffi = require("ffi")

ffi.cdef[[
    size_t strlen(const char *s);
    char *strcat(char *dest, const char *src);
    int sprintf(char *str, const char *format, ...);
]]

-- แปลง Lua string เป็น C string และกลับ
local lua_str = "Hello, World!"

-- strlen
local len = ffi.C.strlen(lua_str)
print("Length:", len)  -- 13

-- สร้าง C char array
local buf = ffi.new("char[100]")
ffi.copy(buf, lua_str)
print("From C buffer:", ffi.string(buf))

-- ffi.string กับ length
local data = ffi.new("uint8_t[5]", {72, 101, 108, 108, 111})  -- "Hello"
local s = ffi.string(data, 5)
print("From bytes:", s)  -- Hello

-- sprintf ผ่าน FFI
local result_buf = ffi.new("char[256]")
ffi.C.sprintf(result_buf, "Pi = %.4f, e = %.4f", math.pi, math.exp(1))
print(ffi.string(result_buf))
```

---

## 45.4 การเรียก C Functions ผ่าน FFI

### ตัวอย่างที่ 10: เรียก Standard Library functions

```lua
-- call_c_functions.lua
local ffi = require("ffi")

ffi.cdef[[
    /* String functions */
    int strcmp(const char *s1, const char *s2);
    int strncmp(const char *s1, const char *s2, size_t n);
    char *strstr(const char *haystack, const char *needle);
    char *strtok(char *str, const char *delim);
    
    /* Memory functions */
    void *memcpy(void *dest, const void *src, size_t n);
    void *memset(void *s, int c, size_t n);
    int memcmp(const void *s1, const void *s2, size_t n);
    
    /* Stdlib */
    int atoi(const char *nptr);
    double atof(const char *nptr);
    void qsort(void *base, size_t nmemb, size_t size,
               int (*compar)(const void *, const void *));
]]

-- strcmp
local r1 = ffi.C.strcmp("apple", "banana")
local r2 = ffi.C.strcmp("apple", "apple")
local r3 = ffi.C.strcmp("banana", "apple")
print("apple vs banana:", r1 < 0 and "less" or r1 == 0 and "equal" or "greater")
print("apple vs apple:", r2 == 0 and "equal" or "not equal")
print("banana vs apple:", r3 > 0 and "greater" or "less")

-- memset
local buf = ffi.new("char[10]")
ffi.C.memset(buf, string.byte('A'), 10)
print("After memset:", ffi.string(buf, 10))  -- AAAAAAAAAA

-- atoi, atof
print("atoi('42'):", ffi.C.atoi("42"))        -- 42
print("atof('3.14'):", ffi.C.atof("3.14"))    -- 3.14
```

### ตัวอย่างที่ 11: เรียก POSIX functions

```lua
-- posix_functions.lua
local ffi = require("ffi")

ffi.cdef[[
    /* File operations */
    typedef struct FILE FILE;
    FILE *fopen(const char *path, const char *mode);
    int fclose(FILE *stream);
    size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
    size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
    int feof(FILE *stream);
    
    /* Process */
    int getpid(void);
    int getppid(void);
    
    /* Environment */
    char *getenv(const char *name);
    
    /* System info */
    typedef struct {
        long uptime;
        unsigned long loads[3];
        unsigned long totalram;
        unsigned long freeram;
        unsigned long sharedram;
        unsigned long bufferram;
        unsigned long totalswap;
        unsigned long freeswap;
        unsigned short procs;
        unsigned long totalhigh;
        unsigned long freehigh;
        unsigned int mem_unit;
    } sysinfo_t;
    
    int sysinfo(sysinfo_t *info);
]]

-- getpid
print("Current PID:", ffi.C.getpid())
print("Parent PID:", ffi.C.getppid())

-- getenv
local path = ffi.C.getenv("PATH")
if path ~= nil then
    print("PATH:", ffi.string(path):sub(1, 50) .. "...")
end

-- sysinfo (Linux only)
if ffi.os == "Linux" then
    local info = ffi.new("sysinfo_t")
    if ffi.C.sysinfo(info) == 0 then
        print(string.format("Total RAM: %.1f MB", 
            tonumber(info.totalram) * tonumber(info.mem_unit) / 1024 / 1024))
        print(string.format("Free RAM: %.1f MB",
            tonumber(info.freeram) * tonumber(info.mem_unit) / 1024 / 1024))
        print("Running processes:", info.procs)
    end
end
```

---

## 45.5 JIT Optimization Hints

### ตัวอย่างที่ 12: jit.on และ jit.off

```lua
-- jit_control.lua

-- ตรวจสอบว่ามี jit module
if not jit then
    print("Not running on LuaJIT")
    os.exit(1)
end

print("JIT enabled:", jit.status())

-- ปิด JIT สำหรับ function ที่ไม่ได้รับประโยชน์จาก JIT
local function debug_heavy_function()
    -- ฟังก์ชันที่ใช้ debug หรือ coroutines มาก
    local t = {}
    for i = 1, 100 do
        t[i] = {x = i, y = i * 2}
    end
    return t
end

-- ปิด JIT เฉพาะ function นี้
jit.off(debug_heavy_function)

-- เปิด JIT กลับ (default)
jit.on()

-- ปิด JIT ทั้งหมด
-- jit.off()

-- เปิดอีกครั้ง
-- jit.on()

-- ตรวจสอบ status
local status, flags = jit.status()
print("JIT active:", status)
print("JIT flags:", flags)
```

### ตัวอย่างที่ 13: jit.flush - ล้าง JIT cache

```lua
-- jit_flush.lua

if not jit then error("Requires LuaJIT") end

-- jit.flush() ล้าง compiled traces ทั้งหมด
-- ใช้เมื่อ:
-- 1. เปลี่ยน code hotly (hot reload)
-- 2. Debug JIT compilation
-- 3. ลด memory footprint

local function heavy_computation(n)
    local sum = 0
    for i = 1, n do
        sum = sum + math.sqrt(i)
    end
    return sum
end

-- รันครั้งแรก (JIT จะ compile)
local t1 = os.clock()
heavy_computation(1000000)
print(string.format("First run: %.4fs", os.clock() - t1))

-- รันครั้งที่สอง (ใช้ compiled code)
local t2 = os.clock()
heavy_computation(1000000)
print(string.format("Second run (JIT): %.4fs", os.clock() - t2))

-- Flush แล้วรันใหม่
jit.flush()

local t3 = os.clock()
heavy_computation(1000000)
print(string.format("After flush: %.4fs", os.clock() - t3))

-- jit.flush(func) - flush เฉพาะ function
jit.flush(heavy_computation)
```

### ตัวอย่างที่ 14: jit.opt - ตั้งค่า optimization

```lua
-- jit_optimize.lua

if not jit then error("Requires LuaJIT") end

-- jit.opt.start ตั้งค่า optimization level
require("jit.opt").start(3)  -- optimization level 0-4

-- ตั้งค่า specific optimizations
require("jit.opt").start(
    "hotloop=10",      -- จำนวน iterations ก่อน JIT compile (default 56)
    "hotexit=2",       -- จำนวน exits ก่อน compile side trace
    "maxrecord=4000",  -- จำนวน bytecodes สูงสุดที่ record
    "maxirpatch=500",  -- จำนวน IR instructions สูงสุดต่อ patch
    "loopunroll=15",   -- จำนวน loop unroll iterations
    "maxside=100",     -- จำนวน side traces สูงสุด
    "maxtrace=1000",   -- จำนวน traces สูงสุด
    "maxmcode=512"     -- ขนาด machine code สูงสุด (KB)
)

-- ใช้ jit.v สำหรับ verbose output
-- luajit -jv script.lua

-- ใช้ jit.dump สำหรับ detailed dump
-- luajit -jdump script.lua

print("JIT optimization configured")
```

---

## 45.6 NYI (Not Yet Implemented) Gotchas

NYI คือสิ่งที่ JIT compiler ยังไม่รองรับ ทำให้ต้อง fallback ไปใช้ interpreter ซึ่งช้ากว่า

### ตัวอย่างที่ 15: สิ่งที่ทำให้ JIT deoptimize

```lua
-- nyi_examples.lua

-- NYI #1: pcall/xpcall ใน hot loop (แก้ได้ใน LuaJIT 2.1+)
-- ควรย้าย error handling ออกจาก hot loop
local function bad_pattern()
    local sum = 0
    for i = 1, 1000000 do
        local ok, val = pcall(function() return i * 2 end)
        if ok then sum = sum + val end
    end
    return sum
end

local function good_pattern()
    local function double(i) return i * 2 end
    local sum = 0
    for i = 1, 1000000 do
        sum = sum + double(i)
    end
    return sum
end

-- NYI #2: coroutine.wrap ใน hot loop
-- ใช้ state machine แทน

-- NYI #3: string.format ใน hot loop (บางรูปแบบ)
-- buffer ผลลัพธ์แล้วค่อย format ทีเดียว

-- NYI #4: table.sort ด้วย custom comparator บางแบบ
-- ใช้ sort แบบธรรมดาหรือ sort ด้วย index

-- NYI #5: ipairs บน non-contiguous tables (บางกรณี)
-- ใช้ numeric for loop แทน

-- ตรวจสอบ NYI ด้วย jit.dump
-- luajit -jdump=r script.lua 2>&1 | grep "NYI"

print("NYI examples loaded")
print("Use: luajit -jdump=r nyi_examples.lua 2>&1 | grep NYI")
```

### ตัวอย่างที่ 16: วิธี workaround NYI

```lua
-- nyi_workaround.lua

-- Workaround: แทนที่ type() ใน hot loop
-- type() เป็น NYI ในบาง context

-- Bad:
local function check_types_bad(t)
    local count = 0
    for i = 1, #t do
        if type(t[i]) == "number" then
            count = count + 1
        end
    end
    return count
end

-- Good: ใช้ tonumber แทน
local function check_types_good(t)
    local count = 0
    for i = 1, #t do
        if tonumber(t[i]) then
            count = count + 1
        end
    end
    return count
end

-- Workaround: string.byte แทน pattern matching ใน hot loop
local function count_spaces_bad(s)
    local count = 0
    for c in s:gmatch(" ") do
        count = count + 1
    end
    return count
end

local function count_spaces_good(s)
    local count = 0
    local space = string.byte(" ")
    for i = 1, #s do
        if string.byte(s, i) == space then
            count = count + 1
        end
    end
    return count
end

-- Test
local data = {}
for i = 1, 1000 do data[i] = i end
print("Type check result:", check_types_good(data))

local text = "Hello World This Is A Test String"
print("Spaces in text:", count_spaces_good(text))
```

---

## 45.7 Performance Comparison

### ตัวอย่างที่ 17: Benchmark ครบถ้วน

```lua
-- full_benchmark.lua
-- รัน: luajit full_benchmark.lua (LuaJIT)
-- เปรียบเทียบกับ: lua full_benchmark.lua (standard Lua)

local function bench(name, iterations, func)
    -- Warmup
    for _ = 1, math.min(iterations // 10, 100) do
        func()
    end
    
    local start = os.clock()
    for _ = 1, iterations do
        func()
    end
    local elapsed = os.clock() - start
    
    local ops_per_sec = iterations / elapsed
    print(string.format("%-30s: %8.3f ms (%s ops/sec)",
        name, elapsed * 1000, 
        ops_per_sec >= 1e6 and string.format("%.1fM", ops_per_sec/1e6)
            or string.format("%.1fK", ops_per_sec/1e3)
    ))
end

print("=== Performance Benchmark ===")
print(string.format("Running on: %s", jit and jit.version or _VERSION))
print()

-- Test 1: Integer arithmetic
bench("integer_add", 1000000, function()
    local x = 0
    for i = 1, 100 do x = x + i end
    return x
end)

-- Test 2: Float arithmetic
bench("float_mul", 1000000, function()
    local x = 1.0
    for i = 1, 100 do x = x * 1.001 end
    return x
end)

-- Test 3: Table access
local t = {}; for i = 1, 100 do t[i] = i end
bench("table_read", 1000000, function()
    local sum = 0
    for i = 1, 100 do sum = sum + t[i] end
    return sum
end)

-- Test 4: String operations
bench("string_concat", 100000, function()
    local parts = {}
    for i = 1, 20 do parts[i] = tostring(i) end
    return table.concat(parts)
end)

-- Test 5: Function call overhead
local function add(a, b) return a + b end
bench("function_call", 1000000, function()
    local x = 0
    for i = 1, 100 do x = add(x, i) end
    return x
end)

-- Test 6: Math operations
bench("math_sqrt", 1000000, function()
    local x = 0
    for i = 1, 100 do x = x + math.sqrt(i) end
    return x
end)
```

---

## 45.8 FFI Struct และ Array

### ตัวอย่างที่ 18: Complex Struct

```lua
-- ffi_struct.lua
local ffi = require("ffi")

ffi.cdef[[
    typedef struct {
        double x, y;
    } Point2D;
    
    typedef struct {
        double x, y, z;
    } Point3D;
    
    typedef struct {
        Point2D min;
        Point2D max;
    } BoundingBox;
    
    typedef struct {
        char name[64];
        int id;
        float score;
        bool active;
    } Player;
    
    typedef struct {
        uint32_t count;
        Player players[100];
    } GameState;
]]

-- สร้าง Point
local p1 = ffi.new("Point2D", {x = 3.0, y = 4.0})
local p2 = ffi.new("Point2D", {x = 0.0, y = 0.0})

-- คำนวณ distance
local function distance(a, b)
    local dx = a.x - b.x
    local dy = a.y - b.y
    return math.sqrt(dx*dx + dy*dy)
end

print(string.format("Distance: %.4f", distance(p1, p2)))  -- 5.0000

-- BoundingBox
local bbox = ffi.new("BoundingBox")
bbox.min.x, bbox.min.y = -10.0, -10.0
bbox.max.x, bbox.max.y = 10.0, 10.0

local function contains(box, point)
    return point.x >= box.min.x and point.x <= box.max.x
       and point.y >= box.min.y and point.y <= box.max.y
end

print("Contains p1:", contains(bbox, p1))  -- true
print("Contains (20,0):", contains(bbox, ffi.new("Point2D", {x=20, y=0})))  -- false

-- GameState
local state = ffi.new("GameState")
state.count = 3

for i = 0, 2 do
    local name = string.format("Player%d", i+1)
    ffi.copy(state.players[i].name, name)
    state.players[i].id = i + 1
    state.players[i].score = (i + 1) * 100.5
    state.players[i].active = true
end

for i = 0, tonumber(state.count) - 1 do
    print(string.format("Player: %s (ID=%d, Score=%.1f)",
        ffi.string(state.players[i].name),
        state.players[i].id,
        state.players[i].score))
end
```

### ตัวอย่างที่ 19: C Arrays กับ FFI

```lua
-- ffi_arrays.lua
local ffi = require("ffi")

ffi.cdef[[
    void *malloc(size_t size);
    void free(void *ptr);
    void *memset(void *s, int c, size_t n);
    void qsort(void *base, size_t nmemb, size_t size,
               int (*compar)(const void *, const void *));
]]

-- Static arrays
local static_arr = ffi.new("int[10]", {5, 3, 8, 1, 9, 2, 7, 4, 6, 0})
print("Original:")
for i = 0, 9 do io.write(static_arr[i] .. " ") end
print()

-- qsort กับ C comparator
local compare_int = ffi.cast("int (*)(const void *, const void *)",
    function(a, b)
        local ia = ffi.cast("const int *", a)[0]
        local ib = ffi.cast("const int *", b)[0]
        return ia < ib and -1 or ia > ib and 1 or 0
    end
)

ffi.C.qsort(static_arr, 10, ffi.sizeof("int"), compare_int)
print("Sorted:")
for i = 0, 9 do io.write(static_arr[i] .. " ") end
print()

-- Dynamic arrays ด้วย malloc
local size = 1000
local dyn_arr = ffi.cast("float *", ffi.C.malloc(size * ffi.sizeof("float")))

-- เติมข้อมูล
for i = 0, size - 1 do
    dyn_arr[i] = math.sin(i * 0.01) * 100
end

-- หา max
local max_val = dyn_arr[0]
for i = 1, size - 1 do
    if dyn_arr[i] > max_val then max_val = dyn_arr[i] end
end
print(string.format("Max value: %.4f", max_val))

-- Free memory
ffi.C.free(dyn_arr)
print("Memory freed")

-- 2D arrays
local rows, cols = 4, 4
local matrix = ffi.new("float[4][4]")
for i = 0, rows - 1 do
    for j = 0, cols - 1 do
        matrix[i][j] = i == j and 1.0 or 0.0  -- identity matrix
    end
end

print("Identity Matrix:")
for i = 0, rows - 1 do
    for j = 0, cols - 1 do
        io.write(string.format("%4.1f ", matrix[i][j]))
    end
    print()
end
```

### ตัวอย่างที่ 20: FFI Pointers

```lua
-- ffi_pointers.lua
local ffi = require("ffi")

ffi.cdef[[
    void *malloc(size_t size);
    void free(void *ptr);
    
    typedef struct LinkedNode {
        int value;
        struct LinkedNode *next;
    } LinkedNode;
]]

-- สร้าง linked list
local function create_node(value)
    local node = ffi.cast("LinkedNode *", 
        ffi.C.malloc(ffi.sizeof("LinkedNode")))
    node.value = value
    node.next = nil
    return node
end

local function insert_front(head, value)
    local node = create_node(value)
    node.next = head
    return node
end

local function print_list(head)
    local current = head
    local values = {}
    while current ~= nil do
        table.insert(values, tostring(current.value))
        current = current.next
    end
    print("List: " .. table.concat(values, " -> "))
end

local function free_list(head)
    local current = head
    while current ~= nil do
        local next = current.next
        ffi.C.free(current)
        current = next
    end
end

-- ใช้งาน
local head = nil
for i = 1, 5 do
    head = insert_front(head, i * 10)
end

print_list(head)  -- 50 -> 40 -> 30 -> 20 -> 10
free_list(head)
print("Linked list freed")
```

---

## 45.9 Practical Example: Fast Image Processing

### ตัวอย่างที่ 21: Image Processing พื้นฐาน

```lua
-- fast_image.lua
-- Fast image processing using FFI

local ffi = require("ffi")

ffi.cdef[[
    void *malloc(size_t size);
    void *calloc(size_t nmemb, size_t size);
    void free(void *ptr);
    void *memcpy(void *dest, const void *src, size_t n);
    
    typedef struct {
        uint32_t width;
        uint32_t height;
        uint8_t  channels;  // 1=gray, 3=RGB, 4=RGBA
        uint8_t  *data;
    } Image;
]]

-- สร้าง Image
local function create_image(w, h, channels)
    channels = channels or 3
    local img = ffi.new("Image")
    img.width = w
    img.height = h
    img.channels = channels
    img.data = ffi.cast("uint8_t *", 
        ffi.C.calloc(w * h * channels, 1))
    return img
end

-- ลบ Image
local function free_image(img)
    ffi.C.free(img.data)
end

-- Get pixel
local function get_pixel(img, x, y)
    local idx = (y * img.width + x) * img.channels
    if img.channels == 3 then
        return img.data[idx], img.data[idx+1], img.data[idx+2]
    else
        return img.data[idx]
    end
end

-- Set pixel
local function set_pixel(img, x, y, r, g, b)
    local idx = (y * img.width + x) * img.channels
    img.data[idx]   = r
    img.data[idx+1] = g
    img.data[idx+2] = b
end

-- สร้าง gradient image
local function create_gradient(w, h)
    local img = create_image(w, h, 3)
    for y = 0, h - 1 do
        for x = 0, w - 1 do
            set_pixel(img, x, y,
                math.floor(x / w * 255),  -- R
                math.floor(y / h * 255),  -- G
                128)                       -- B
        end
    end
    return img
end

local img = create_gradient(256, 256)
print(string.format("Created %dx%d image", img.width, img.height))

local r, g, b = get_pixel(img, 128, 128)
print(string.format("Center pixel: RGB(%d, %d, %d)", r, g, b))

free_image(img)
```

### ตัวอย่างที่ 22: Image Filters ด้วย FFI

```lua
-- image_filters.lua
local ffi = require("ffi")

ffi.cdef[[
    void *malloc(size_t size);
    void *calloc(size_t nmemb, size_t size);
    void free(void *ptr);
]]

-- Grayscale conversion
local function to_grayscale(src)
    local w, h = src.width, src.height
    local dst = {
        width = w, height = h, channels = 1,
        data = ffi.cast("uint8_t *", ffi.C.calloc(w * h, 1))
    }
    
    for y = 0, h - 1 do
        for x = 0, w - 1 do
            local si = (y * w + x) * 3
            -- BT.601 luma formula
            local gray = math.floor(
                0.299 * src.data[si] +
                0.587 * src.data[si+1] +
                0.114 * src.data[si+2]
            )
            dst.data[y * w + x] = gray
        end
    end
    return dst
end

-- Brightness adjustment (FFI array access เร็วมาก)
local function adjust_brightness(img, delta)
    local total = img.width * img.height * img.channels
    for i = 0, total - 1 do
        local val = img.data[i] + delta
        -- clamp ให้อยู่ใน [0, 255]
        img.data[i] = val < 0 and 0 or val > 255 and 255 or val
    end
end

-- Horizontal flip
local function flip_horizontal(img)
    local w, h, c = img.width, img.height, img.channels
    local row_size = w * c
    local row_buf = ffi.new("uint8_t[?]", row_size)
    
    for y = 0, h - 1 do
        local row_start = y * row_size
        -- copy row to buffer
        ffi.copy(row_buf, img.data + row_start, row_size)
        -- write back reversed
        for x = 0, w - 1 do
            for ch = 0, c - 1 do
                img.data[row_start + x*c + ch] = 
                    row_buf[(w-1-x)*c + ch]
            end
        end
    end
end

-- Threshold (binary)
local function threshold(img, thresh)
    local total = img.width * img.height * img.channels
    for i = 0, total - 1 do
        img.data[i] = img.data[i] >= thresh and 255 or 0
    end
end

print("Image filter functions defined")
print("Functions: to_grayscale, adjust_brightness, flip_horizontal, threshold")
```

### ตัวอย่างที่ 23: Fast Convolution Filter

```lua
-- convolution.lua
-- 2D convolution ด้วย FFI (เร็วกว่า pure Lua มาก)
local ffi = require("ffi")

ffi.cdef[[
    void *calloc(size_t nmemb, size_t size);
    void free(void *ptr);
]]

-- Gaussian blur kernel 3x3
local BLUR_3x3 = {
    {1/16, 2/16, 1/16},
    {2/16, 4/16, 2/16},
    {1/16, 2/16, 1/16},
}

-- Sharpen kernel
local SHARPEN = {
    { 0, -1,  0},
    {-1,  5, -1},
    { 0, -1,  0},
}

-- Edge detection (Sobel X)
local SOBEL_X = {
    {-1,  0,  1},
    {-2,  0,  2},
    {-1,  0,  1},
}

local function apply_kernel(src_data, dst_data, w, h, kernel, ksize)
    local half = math.floor(ksize / 2)
    
    for y = half, h - 1 - half do
        for x = half, w - 1 - half do
            local sum = 0.0
            for ky = 0, ksize - 1 do
                for kx = 0, ksize - 1 do
                    local sy = y + ky - half
                    local sx = x + kx - half
                    sum = sum + src_data[sy * w + sx] * kernel[ky+1][kx+1]
                end
            end
            -- Clamp to [0, 255]
            sum = math.max(0, math.min(255, sum))
            dst_data[y * w + x] = math.floor(sum)
        end
    end
end

-- สร้าง test grayscale image (256x256)
local W, H = 256, 256
local src = ffi.cast("uint8_t *", ffi.C.calloc(W * H, 1))
local dst = ffi.cast("uint8_t *", ffi.C.calloc(W * H, 1))

-- สร้าง test pattern
for y = 0, H - 1 do
    for x = 0, W - 1 do
        -- checkerboard pattern
        src[y * W + x] = ((x // 32 + y // 32) % 2 == 0) and 255 or 0
    end
end

-- วัดเวลา blur
local t = os.clock()
apply_kernel(src, dst, W, H, BLUR_3x3, 3)
print(string.format("Blur: %.4fs for %dx%d image", os.clock() - t, W, H))

-- วัดเวลา sharpen
t = os.clock()
apply_kernel(src, dst, W, H, SHARPEN, 3)
print(string.format("Sharpen: %.4fs for %dx%d image", os.clock() - t, W, H))

-- ตรวจสอบผลลัพธ์
print("Center pixel after blur:", dst[128 * W + 128])

ffi.C.free(src)
ffi.C.free(dst)
```

### ตัวอย่างที่ 24: PNG Writing ด้วย FFI (ผ่าน libpng)

```lua
-- ffi_libpng.lua
-- ตัวอย่างการใช้ libpng ผ่าน FFI

local ffi = require("ffi")

-- ลองโหลด libpng
local ok, png = pcall(ffi.load, "png")
if not ok then
    print("libpng not available, showing API only")
    -- แสดง API ที่จะใช้
    ffi.cdef[[
        typedef struct png_struct png_struct;
        typedef png_struct *png_structp;
        typedef struct png_info png_info;
        typedef png_info *png_infop;
        typedef unsigned char png_byte;
        typedef unsigned int png_uint_32;
        
        png_structp png_create_write_struct(
            const char *user_png_ver,
            void *error_ptr, void *error_fn, void *warn_fn);
        png_infop png_create_info_struct(png_structp png_ptr);
        void png_destroy_write_struct(
            png_structp *png_ptr_ptr, png_infop *info_ptr_ptr);
        void png_set_IHDR(
            png_structp png_ptr, png_infop info_ptr,
            png_uint_32 width, png_uint_32 height,
            int bit_depth, int color_type, int interlace_method,
            int compression_method, int filter_method);
        void png_write_row(png_structp png_ptr, png_byte *row);
        void png_write_end(png_structp png_ptr, png_infop info_ptr);
    ]]
    print("PNG API declarations loaded (library not available)")
    return
end

print("libpng loaded successfully")
-- การใช้งานจริงจะ initialize png_struct, set IHDR, แล้ว write rows
```

---

## 45.10 FFI Callbacks

### ตัวอย่างที่ 25: C Callbacks จาก Lua

```lua
-- ffi_callbacks.lua
local ffi = require("ffi")

ffi.cdef[[
    void qsort(void *base, size_t nmemb, size_t size,
               int (*compar)(const void *, const void *));
    void *malloc(size_t size);
    void free(void *ptr);
]]

-- สร้าง C callback จาก Lua function
local compare_asc = ffi.cast("int (*)(const void *, const void *)",
    function(a, b)
        local ia = ffi.cast("const int *", a)[0]
        local ib = ffi.cast("const int *", b)[0]
        return ia - ib
    end
)

local compare_desc = ffi.cast("int (*)(const void *, const void *)",
    function(a, b)
        local ia = ffi.cast("const int *", a)[0]
        local ib = ffi.cast("const int *", b)[0]
        return ib - ia
    end
)

-- สร้าง array
local n = 10
local arr = ffi.new("int[10]", {64, 25, 12, 22, 11, 90, 45, 3, 78, 56})

print("Original:")
for i = 0, n-1 do io.write(arr[i] .. " ") end; print()

-- Sort ascending
ffi.C.qsort(arr, n, ffi.sizeof("int"), compare_asc)
print("Sorted ascending:")
for i = 0, n-1 do io.write(arr[i] .. " ") end; print()

-- Sort descending
ffi.C.qsort(arr, n, ffi.sizeof("int"), compare_desc)
print("Sorted descending:")
for i = 0, n-1 do io.write(arr[i] .. " ") end; print()

-- cleanup callbacks
compare_asc:free()
compare_desc:free()
```

### ตัวอย่างที่ 26: Metatypes - OOP กับ FFI

```lua
-- ffi_metatype.lua
local ffi = require("ffi")

ffi.cdef[[
    typedef struct {
        double x, y;
    } Vec2;
]]

-- ใส่ methods บน C struct ด้วย metatype
local Vec2MT = {
    __add = function(a, b)
        return ffi.new("Vec2", {x = a.x + b.x, y = a.y + b.y})
    end,
    __sub = function(a, b)
        return ffi.new("Vec2", {x = a.x - b.x, y = a.y - b.y})
    end,
    __mul = function(a, s)
        return ffi.new("Vec2", {x = a.x * s, y = a.y * s})
    end,
    __tostring = function(v)
        return string.format("Vec2(%.3f, %.3f)", v.x, v.y)
    end,
    __index = {
        length = function(self)
            return math.sqrt(self.x * self.x + self.y * self.y)
        end,
        normalize = function(self)
            local len = self:length()
            if len == 0 then return ffi.new("Vec2", {x=0, y=0}) end
            return ffi.new("Vec2", {x = self.x/len, y = self.y/len})
        end,
        dot = function(self, other)
            return self.x * other.x + self.y * other.y
        end,
    }
}

ffi.metatype("Vec2", Vec2MT)

-- ใช้งาน
local a = ffi.new("Vec2", {x = 3.0, y = 4.0})
local b = ffi.new("Vec2", {x = 1.0, y = 2.0})

print("a:", tostring(a))
print("b:", tostring(b))
print("a + b:", tostring(a + b))
print("a - b:", tostring(a - b))
print("a * 2:", tostring(a * 2))
print("|a|:", a:length())      -- 5.0
print("norm(a):", tostring(a:normalize()))
print("a · b:", a:dot(b))
```

---

## 45.11 ตัวอย่างขั้นสูง: Fast Hash Table

### ตัวอย่างที่ 27: Hash Table ด้วย FFI

```lua
-- ffi_hashtable.lua
local ffi = require("ffi")

ffi.cdef[[
    void *calloc(size_t nmemb, size_t size);
    void free(void *ptr);
    
    typedef struct {
        uint64_t key;
        int64_t  value;
        uint8_t  occupied;
    } HEntry;
    
    typedef struct {
        HEntry  *entries;
        uint32_t capacity;
        uint32_t count;
    } HashTable;
]]

local HashTable = {}
HashTable.__index = HashTable

function HashTable.new(capacity)
    capacity = capacity or 1024
    local ht = setmetatable({}, HashTable)
    ht._c = ffi.new("HashTable")
    ht._c.capacity = capacity
    ht._c.count = 0
    ht._c.entries = ffi.cast("HEntry *",
        ffi.C.calloc(capacity, ffi.sizeof("HEntry")))
    return ht
end

function HashTable:_hash(key)
    -- FNV-1a hash
    local h = 0xcbf29ce484222325ULL
    local fnv = 0x100000001b3ULL
    for i = 1, #key do
        h = ffi.cast("uint64_t", 
            bit.bxor(tonumber(h), string.byte(key, i)))
        h = h * fnv
    end
    return tonumber(h % self._c.capacity)
end

function HashTable:set(key, value)
    local hash = self:_hash(tostring(key))
    local cap = tonumber(self._c.capacity)
    local idx = hash
    
    for _ = 1, cap do
        local entry = self._c.entries[idx]
        if entry.occupied == 0 or entry.key == hash then
            entry.key = hash
            entry.value = value
            if entry.occupied == 0 then
                entry.occupied = 1
                self._c.count = self._c.count + 1
            end
            return
        end
        idx = (idx + 1) % cap
    end
    error("Hash table full!")
end

function HashTable:get(key)
    local hash = self:_hash(tostring(key))
    local cap = tonumber(self._c.capacity)
    local idx = hash
    
    for _ = 1, cap do
        local entry = self._c.entries[idx]
        if entry.occupied == 0 then return nil end
        if entry.key == hash then
            return tonumber(entry.value)
        end
        idx = (idx + 1) % cap
    end
    return nil
end

function HashTable:destroy()
    ffi.C.free(self._c.entries)
end

-- ทดสอบ
local ht = HashTable.new(1024)

-- Insert
for i = 1, 100 do
    ht:set("key_" .. i, i * 10)
end

-- Lookup
print("key_1 =", ht:get("key_1"))    -- 10
print("key_50 =", ht:get("key_50"))  -- 500
print("key_100 =", ht:get("key_100")) -- 1000
print("Count:", tonumber(ht._c.count))

-- Benchmark
local t = os.clock()
for i = 1, 100000 do
    ht:set("k" .. i, i)
end
print(string.format("100K inserts: %.4fs", os.clock() - t))

ht:destroy()
```

---

## 45.12 การจัดการ Memory ใน FFI

### ตัวอย่างที่ 28: GC Integration

```lua
-- ffi_gc.lua
local ffi = require("ffi")

ffi.cdef[[
    void *malloc(size_t size);
    void free(void *ptr);
    
    typedef struct {
        int width, height;
        uint8_t *pixels;
    } Bitmap;
]]

-- วิธีที่ 1: ffi.gc - auto cleanup
local function create_buffer(size)
    local ptr = ffi.C.malloc(size)
    if ptr == nil then error("malloc failed") end
    -- ffi.gc จะเรียก free เมื่อ ptr ถูก GC
    return ffi.gc(ptr, ffi.C.free)
end

local buf = create_buffer(1024)
print("Buffer allocated:", buf ~= nil)
-- buf จะถูก free อัตโนมัติเมื่อ garbage collected

-- วิธีที่ 2: Wrapper object พร้อม destructor
local Bitmap = {}
Bitmap.__index = Bitmap

function Bitmap.new(w, h)
    local self = setmetatable({}, Bitmap)
    self.width = w
    self.height = h
    self.pixels = ffi.gc(
        ffi.cast("uint8_t *", ffi.C.malloc(w * h)),
        ffi.C.free
    )
    return self
end

function Bitmap:pixel(x, y)
    return self.pixels[y * self.width + x]
end

function Bitmap:set_pixel(x, y, value)
    self.pixels[y * self.width + x] = value
end

-- ใช้งาน
local bmp = Bitmap.new(100, 100)
bmp:set_pixel(50, 50, 255)
print("Pixel at (50,50):", bmp:pixel(50, 50))
-- memory จะถูก free อัตโนมัติเมื่อ bmp ถูก GC

print("Bitmap created with auto-cleanup")
```

### ตัวอย่างที่ 29: Memory Pool

```lua
-- memory_pool.lua
local ffi = require("ffi")

ffi.cdef[[
    void *malloc(size_t size);
    void free(void *ptr);
    void *memset(void *s, int c, size_t n);
]]

-- Simple memory pool สำหรับ fixed-size allocations
local Pool = {}
Pool.__index = Pool

function Pool.new(item_size, capacity)
    local self = setmetatable({}, Pool)
    self.item_size = item_size
    self.capacity = capacity
    self.count = 0
    
    -- Allocate pool memory
    self._data = ffi.gc(
        ffi.cast("uint8_t *", ffi.C.malloc(item_size * capacity)),
        ffi.C.free
    )
    
    -- Free list (stack of available indices)
    self._free = ffi.new("int[?]", capacity)
    for i = 0, capacity - 1 do
        self._free[i] = i
    end
    self._free_count = capacity
    
    return self
end

function Pool:alloc()
    if self._free_count == 0 then
        error("Pool exhausted!")
    end
    self._free_count = self._free_count - 1
    local idx = self._free[self._free_count]
    self.count = self.count + 1
    return self._data + idx * self.item_size, idx
end

function Pool:free_item(idx)
    self._free[self._free_count] = idx
    self._free_count = self._free_count + 1
    self.count = self.count - 1
end

-- ทดสอบ
local ITEM_SIZE = 64  -- 64 bytes per item
local pool = Pool.new(ITEM_SIZE, 1000)

print("Pool created:", pool.capacity, "items of", ITEM_SIZE, "bytes")

-- Allocate
local ptr1, idx1 = pool:alloc()
local ptr2, idx2 = pool:alloc()
print(string.format("Allocated items at indices %d and %d", idx1, idx2))
print("Pool usage:", pool.count, "/", pool.capacity)

-- Free
pool:free_item(idx1)
print("After free, usage:", pool.count, "/", pool.capacity)

-- Benchmark
local t = os.clock()
for _ = 1, 10000 do
    local p, idx = pool:alloc()
    pool:free_item(idx)
end
print(string.format("10K alloc/free cycles: %.4fs", os.clock() - t))
```

---

## 45.13 LuaJIT Extensions

### ตัวอย่างที่ 30: bit library (Bitwise Operations)

```lua
-- luajit_bit.lua
-- LuaJIT มี bit library (ใน Lua 5.3+ มี ~ << >> &)

-- LuaJIT ใช้ require("bit") หรือ bit.*
local bit = require("bit")

local function demo_bitops()
    local a = 0xFF00
    local b = 0x0F0F
    
    print(string.format("a = 0x%04X (%d)", a, a))
    print(string.format("b = 0x%04X (%d)", b, b))
    print()
    
    -- AND
    print(string.format("a & b  = 0x%04X", bit.band(a, b)))   -- 0x0F00
    -- OR
    print(string.format("a | b  = 0x%04X", bit.bor(a, b)))    -- 0xFF0F
    -- XOR
    print(string.format("a ^ b  = 0x%04X", bit.bxor(a, b)))   -- 0xF00F
    -- NOT
    print(string.format("~a     = 0x%08X", bit.bnot(a)))
    -- Left shift
    print(string.format("a << 4 = 0x%05X", bit.lshift(a, 4))) -- 0xFF000
    -- Right shift (logical)
    print(string.format("a >> 4 = 0x%03X", bit.rshift(a, 4))) -- 0xFF0
    -- Arithmetic right shift
    print(string.format("a >>> 4 = 0x%03X", bit.arshift(a, 4)))
    -- Rotate left/right
    print(string.format("rol(a,8) = 0x%08X", bit.rol(a, 8)))
    print(string.format("ror(a,8) = 0x%08X", bit.ror(a, 8)))
end

demo_bitops()

-- ตัวอย่าง: pack/unpack integers
local function pack_rgba(r, g, b, a)
    return bit.bor(
        bit.lshift(r, 24),
        bit.lshift(g, 16),
        bit.lshift(b, 8),
        a
    )
end

local function unpack_rgba(packed)
    return
        bit.rshift(bit.band(packed, 0xFF000000), 24),
        bit.rshift(bit.band(packed, 0x00FF0000), 16),
        bit.rshift(bit.band(packed, 0x0000FF00), 8),
        bit.band(packed, 0x000000FF)
end

local packed = pack_rgba(255, 128, 64, 200)
print(string.format("\nPacked RGBA: 0x%08X", bit.tobit(packed)))
local r, g, b, a = unpack_rgba(bit.tobit(packed))
print(string.format("Unpacked: R=%d G=%d B=%d A=%d", r, g, b, a))
```

---

## 45.14 สรุป

### ตัวอย่างที่ 31: เมื่อใดควรใช้ LuaJIT

```lua
-- when_to_use.lua
-- แนวทางเลือก LuaJIT vs standard Lua

--[[
ใช้ LuaJIT เมื่อ:
1. ต้องการ performance สูงสำหรับ numeric computation
2. ต้องการ FFI เพื่อเรียก C libraries
3. ใช้ OpenResty (nginx + LuaJIT)
4. ต้องการ bit operations (require "bit")
5. ทำ game development หรือ real-time processing

ใช้ Standard Lua 5.4 เมื่อ:
1. ต้องการ integer type (Lua 5.3+)
2. ต้องการ to-be-closed variables (<close>)
3. Compatibility กับ Lua 5.4 features
4. ใช้ rock packages ที่ต้อง Lua 5.4

สิ่งที่ต้องระวังใน LuaJIT:
- LuaJIT compatible กับ Lua 5.1 (ไม่ใช่ 5.4)
- บาง LuaRocks packages อาจไม่รองรับ
- NYI items ทำให้ fallback ไป interpreter
- Memory limit 1-2GB (เพราะใช้ 32-bit pointers ใน 64-bit mode)
]]

-- ตรวจสอบ compatibility
local version_info = {}
version_info.lua_version = _VERSION
version_info.is_luajit = jit ~= nil
if jit then
    version_info.luajit_version = jit.version
    version_info.luajit_arch = jit.arch
    version_info.jit_on = jit.status()
end

for k, v in pairs(version_info) do
    print(string.format("  %-20s = %s", k, tostring(v)))
end
```

---

## แบบฝึกหัด

**ข้อ 1**: เขียนโปรแกรมใช้ FFI เรียก `clock_gettime` บน Linux เพื่อวัดเวลาแบบ nanosecond precision เปรียบเทียบกับ `os.clock()`

**ข้อ 2**: สร้าง FFI wrapper สำหรับ `zlib` ที่รองรับ `compress` และ `decompress` ข้อมูล ทดสอบกับ string ขนาดต่างๆ

**ข้อ 3**: เขียน matrix multiplication 100x100 สามวิธี: (1) pure Lua, (2) FFI arrays, (3) FFI + BLAS ถ้ามี แล้ว benchmark เปรียบเทียบ

**ข้อ 4**: สร้าง `ffi.metatype` สำหรับ `Mat3x3` (3x3 float matrix) พร้อม methods: `mul`, `transpose`, `determinant`, `inverse`

**ข้อ 5**: ใช้ LuaJIT FFI เพื่อสร้าง simple ring buffer สำหรับ audio samples (float array) ที่รองรับ `push`, `pop`, `size` และ lock-free thread safety ด้วย memory barriers

---

> **บทถัดไป**: [บทที่ 46: FFI - Foreign Function Interface](part-46.md)
