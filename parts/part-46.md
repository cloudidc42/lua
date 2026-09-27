# บทที่ 46: FFI - Foreign Function Interface

## บทนำ

FFI (Foreign Function Interface) เป็นกลไกที่ช่วยให้ LuaJIT สามารถเรียกใช้ฟังก์ชัน C และเข้าถึง data structures ของ C ได้โดยตรง โดยไม่ต้องเขียน C extension แยกต่างหาก นี่เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ LuaJIT ที่ทำให้สามารถเชื่อมต่อกับ native libraries ได้อย่างมีประสิทธิภาพสูง

## ทำความเข้าใจ FFI

FFI ใน LuaJIT ทำงานโดยการแปลง C declarations เป็น Lua objects ที่สามารถใช้งานได้โดยตรง ซึ่งแตกต่างจากการเขียน binding ด้วย lua_CFunction แบบดั้งเดิม

### ข้อดีของ FFI
- เร็วกว่า Lua/C API ดั้งเดิมมาก
- ไม่ต้องคอมไพล์ C code แยกต่างหาก
- เข้าถึง C structs ได้โดยตรง
- รองรับ pointers, arrays, callbacks

---

## ตัวอย่างที่ 1: การเริ่มต้นใช้งาน FFI

```lua
-- ต้องใช้ LuaJIT เท่านั้น
local ffi = require("ffi")

-- ประกาศ C functions ที่ต้องการใช้
ffi.cdef[[
    int printf(const char *fmt, ...);
    double sqrt(double x);
    double sin(double x);
    double cos(double x);
]]

-- เรียกใช้ printf โดยตรง
ffi.C.printf("Hello from C printf!\n")
ffi.C.printf("Value: %d\n", 42)

-- เรียกใช้ math functions
local result = ffi.C.sqrt(16.0)
print("sqrt(16) =", result)  -- 4.0

local s = ffi.C.sin(math.pi / 2)
print("sin(pi/2) =", s)  -- 1.0
```

---

## ตัวอย่างที่ 2: การโหลด Shared Library ด้วย ffi.load()

```lua
local ffi = require("ffi")

-- โหลด libm (math library)
local libm = ffi.load("m")

ffi.cdef[[
    double pow(double x, double y);
    double log(double x);
    double exp(double x);
    double floor(double x);
    double ceil(double x);
    double fabs(double x);
]]

-- ใช้งานฟังก์ชัน
print("pow(2, 10) =", libm.pow(2, 10))    -- 1024.0
print("log(e) =", libm.log(math.exp(1)))   -- 1.0
print("exp(1) =", libm.exp(1))             -- 2.718...
print("floor(3.7) =", libm.floor(3.7))     -- 3.0
print("ceil(3.2) =", libm.ceil(3.2))       -- 4.0
print("fabs(-5.5) =", libm.fabs(-5.5))     -- 5.5
```

---

## ตัวอย่างที่ 3: ffi.cdef() สำหรับประกาศ Types

```lua
local ffi = require("ffi")

-- ประกาศ struct แบบง่าย
ffi.cdef[[
    typedef struct {
        int x;
        int y;
    } Point;

    typedef struct {
        float r;
        float g;
        float b;
        float a;
    } Color;

    typedef struct {
        Point position;
        Color color;
        float size;
    } Particle;
]]

-- สร้าง instance ของ struct
local p = ffi.new("Point")
p.x = 10
p.y = 20
print(string.format("Point: (%d, %d)", p.x, p.y))

-- สร้าง Color
local c = ffi.new("Color")
c.r = 1.0
c.g = 0.5
c.b = 0.0
c.a = 1.0
print(string.format("Color: RGBA(%.1f, %.1f, %.1f, %.1f)", c.r, c.g, c.b, c.a))

-- สร้าง Particle ที่ซ้อน struct
local particle = ffi.new("Particle")
particle.position.x = 100
particle.position.y = 200
particle.color.r = 1.0
particle.color.g = 0.0
particle.color.b = 0.0
particle.color.a = 1.0
particle.size = 5.0
print(string.format("Particle at (%d, %d) size=%.1f",
    particle.position.x, particle.position.y, particle.size))
```

---

## ตัวอย่างที่ 4: ffi.new() และการ Initialize Values

```lua
local ffi = require("ffi")

ffi.cdef[[
    typedef struct {
        int width;
        int height;
        char name[64];
    } Window;
]]

-- สร้างด้วยค่าเริ่มต้น
local w1 = ffi.new("Window")
print("Default width:", w1.width)   -- 0
print("Default height:", w1.height) -- 0

-- สร้างพร้อม initialize
local w2 = ffi.new("Window", {800, 600})
print("w2 width:", w2.width)   -- 800
print("w2 height:", w2.height) -- 600

-- กำหนดค่า string ใน C array
ffi.copy(w2.name, "Main Window")
print("Window name:", ffi.string(w2.name))

-- ใช้ sizeof
print("Size of Window:", ffi.sizeof("Window"), "bytes")
print("Size of int:", ffi.sizeof("int"), "bytes")
print("Size of double:", ffi.sizeof("double"), "bytes")
```

---

## ตัวอย่างที่ 5: C Arrays ใน FFI

```lua
local ffi = require("ffi")

-- สร้าง array ของ integers
local arr = ffi.new("int[10]")
for i = 0, 9 do
    arr[i] = i * i  -- 0-based indexing!
end

print("Array contents:")
for i = 0, 9 do
    io.write(arr[i] .. " ")
end
print()

-- Array ของ doubles
local doubles = ffi.new("double[5]", {1.1, 2.2, 3.3, 4.4, 5.5})
print("\nDouble array:")
for i = 0, 4 do
    io.write(string.format("%.1f ", doubles[i]))
end
print()

-- Array ของ structs
ffi.cdef[[
    typedef struct { float x, y, z; } Vec3;
]]

local verts = ffi.new("Vec3[3]")
verts[0].x, verts[0].y, verts[0].z = 0, 0, 0
verts[1].x, verts[1].y, verts[1].z = 1, 0, 0
verts[2].x, verts[2].y, verts[2].z = 0, 1, 0

for i = 0, 2 do
    print(string.format("Vertex %d: (%.1f, %.1f, %.1f)",
        i, verts[i].x, verts[i].y, verts[i].z))
end
```

---

## ตัวอย่างที่ 6: Pointers ใน FFI

```lua
local ffi = require("ffi")

ffi.cdef[[
    void *malloc(size_t size);
    void free(void *ptr);
    void *memset(void *s, int c, size_t n);
    void *memcpy(void *dest, const void *src, size_t n);
]]

-- การใช้ pointer แบบง่าย
local x = ffi.new("int[1]", {42})
local ptr = ffi.cast("int*", x)
print("Value via pointer:", ptr[0])  -- 42
ptr[0] = 100
print("Modified value:", x[0])  -- 100

-- malloc และ free
local buf = ffi.C.malloc(256)
ffi.C.memset(buf, 0, 256)

-- Cast เป็น char pointer
local cbuf = ffi.cast("char*", buf)
ffi.copy(cbuf, "Hello from malloc!")
print("String from malloc:", ffi.string(cbuf))

ffi.C.free(buf)
print("Memory freed successfully")

-- Pointer arithmetic
local nums = ffi.new("int[5]", {10, 20, 30, 40, 50})
local p = ffi.cast("int*", nums)
print("\nPointer arithmetic:")
for i = 0, 4 do
    print(string.format("  p[%d] = %d", i, p[i]))
end
```

---

## ตัวอย่างที่ 7: String Handling ใน FFI

```lua
local ffi = require("ffi")

ffi.cdef[[
    size_t strlen(const char *s);
    char *strcpy(char *dest, const char *src);
    char *strcat(char *dest, const char *src);
    int strcmp(const char *s1, const char *s2);
    char *strstr(const char *haystack, const char *needle);
    char *strchr(const char *s, int c);
    char *strdup(const char *s);
    void free(void *ptr);
]]

-- ใช้ string functions ของ C
local s = "Hello, World!"
print("Length:", ffi.C.strlen(s))

-- สร้าง buffer สำหรับ string
local buf = ffi.new("char[128]")
ffi.copy(buf, "Hello")
ffi.C.strcat(buf, ", ")
ffi.C.strcat(buf, "World!")
print("Concatenated:", ffi.string(buf))

-- เปรียบเทียบ strings
local a, b = "apple", "banana"
local cmp = ffi.C.strcmp(a, b)
if cmp < 0 then
    print(a .. " comes before " .. b)
elseif cmp > 0 then
    print(a .. " comes after " .. b)
else
    print(a .. " equals " .. b)
end

-- หา substring
local text = "The quick brown fox"
local pos = ffi.C.strstr(text, "brown")
if pos ~= nil then
    print("Found 'brown' at:", ffi.string(pos))
end

-- ffi.string() แปลง C string เป็น Lua string
local cstr = ffi.new("char[20]", "Test String")
local luastr = ffi.string(cstr)
print("Lua string:", luastr, type(luastr))
```

---

## ตัวอย่างที่ 8: Bitfields และ Union ใน FFI

```lua
local ffi = require("ffi")

-- Union
ffi.cdef[[
    typedef union {
        int i;
        float f;
        unsigned char bytes[4];
    } IntFloat;
]]

local uf = ffi.new("IntFloat")
uf.f = 3.14159

print(string.format("As float: %f", uf.f))
print(string.format("As int: %d (0x%08X)", uf.i, uf.i))
print("As bytes:")
for i = 0, 3 do
    io.write(string.format("  [%d] = 0x%02X", i, uf.bytes[i]))
end
print()

-- Packed struct (bitfields)
ffi.cdef[[
    typedef struct {
        unsigned int red   : 5;
        unsigned int green : 6;
        unsigned int blue  : 5;
    } RGB565;
]]

local color = ffi.new("RGB565")
color.red   = 31   -- max 5 bits
color.green = 63   -- max 6 bits
color.blue  = 31   -- max 5 bits

print(string.format("\nRGB565: R=%d G=%d B=%d",
    color.red, color.green, color.blue))
print("Size:", ffi.sizeof("RGB565"), "bytes")
```

---

## ตัวอย่างที่ 9: Callbacks จาก C ไป Lua

```lua
local ffi = require("ffi")

ffi.cdef[[
    typedef int (*compare_func)(const void*, const void*);
    void qsort(void *base, size_t nmemb, size_t size,
               compare_func compar);
]]

-- สร้าง callback function
local compare_int = ffi.cast("compare_func", function(a, b)
    local ia = ffi.cast("int*", a)[0]
    local ib = ffi.cast("int*", b)[0]
    if ia < ib then return -1
    elseif ia > ib then return 1
    else return 0 end
end)

-- Array ที่ต้องการ sort
local arr = ffi.new("int[8]", {5, 2, 8, 1, 9, 3, 7, 4})
print("Before sort:")
for i = 0, 7 do io.write(arr[i] .. " ") end
print()

-- เรียก qsort พร้อม callback
ffi.C.qsort(arr, 8, ffi.sizeof("int"), compare_int)

print("After sort:")
for i = 0, 7 do io.write(arr[i] .. " ") end
print()

-- ต้องเคลียร์ callback เมื่อไม่ใช้แล้ว
compare_int:free()
```

---

## ตัวอย่างที่ 10: การใช้ libc Functions

```lua
local ffi = require("ffi")

ffi.cdef[[
    // Time functions
    typedef long time_t;
    typedef struct {
        int tm_sec;
        int tm_min;
        int tm_hour;
        int tm_mday;
        int tm_mon;
        int tm_year;
        int tm_wday;
        int tm_yday;
        int tm_isdst;
    } tm;

    time_t time(time_t *tloc);
    struct tm *localtime(const time_t *timer);
    char *ctime(const time_t *timer);
    size_t strftime(char *s, size_t max, const char *format,
                    const struct tm *tm);
]]

-- รับเวลาปัจจุบัน
local t = ffi.new("time_t[1]")
ffi.C.time(t)
print("Unix timestamp:", tonumber(t[0]))

-- แปลงเป็น local time struct
local tm = ffi.C.localtime(t)
print(string.format("Date: %04d-%02d-%02d",
    tm.tm_year + 1900, tm.tm_mon + 1, tm.tm_mday))
print(string.format("Time: %02d:%02d:%02d",
    tm.tm_hour, tm.tm_min, tm.tm_sec))

-- ใช้ strftime
local buf = ffi.new("char[64]")
ffi.C.strftime(buf, 64, "%Y-%m-%d %H:%M:%S", tm)
print("Formatted:", ffi.string(buf))

-- ctime
local timestr = ffi.C.ctime(t)
print("ctime:", ffi.string(timestr):gsub("\n", ""))
```

---

## ตัวอย่างที่ 11: File I/O ผ่าน FFI

```lua
local ffi = require("ffi")

ffi.cdef[[
    typedef struct FILE FILE;
    FILE *fopen(const char *path, const char *mode);
    int fclose(FILE *stream);
    size_t fwrite(const void *ptr, size_t size,
                  size_t nmemb, FILE *stream);
    size_t fread(void *ptr, size_t size,
                 size_t nmemb, FILE *stream);
    int feof(FILE *stream);
    int fprintf(FILE *stream, const char *format, ...);
    int fflush(FILE *stream);
    long ftell(FILE *stream);
    int fseek(FILE *stream, long offset, int whence);
]]

-- สร้างและเขียน file
local path = "/tmp/ffi_test.txt"
local f = ffi.C.fopen(path, "w")
if f == nil then
    error("ไม่สามารถเปิดไฟล์ได้")
end

local content = "Hello from FFI file I/O!\nLine 2\nLine 3\n"
ffi.C.fwrite(content, 1, #content, f)
ffi.C.fclose(f)
print("เขียนไฟล์สำเร็จ:", path)

-- อ่าน file กลับมา
f = ffi.C.fopen(path, "r")
if f ~= nil then
    local buf = ffi.new("char[1024]")
    local n = ffi.C.fread(buf, 1, 1023, f)
    buf[n] = 0  -- null terminate
    print("อ่านได้", tonumber(n), "bytes:")
    print(ffi.string(buf, tonumber(n)))
    ffi.C.fclose(f)
end

-- Seek และ Tell
f = ffi.C.fopen(path, "r")
ffi.C.fseek(f, 0, 2)  -- SEEK_END = 2
local size = ffi.C.ftell(f)
print("File size:", tonumber(size), "bytes")
ffi.C.fclose(f)
```

---

## ตัวอย่างที่ 12: System Calls ผ่าน FFI (Linux)

```lua
local ffi = require("ffi")

ffi.cdef[[
    // Process information
    typedef int pid_t;
    typedef unsigned int uid_t;
    typedef unsigned int gid_t;

    pid_t getpid(void);
    pid_t getppid(void);
    uid_t getuid(void);
    gid_t getgid(void);

    // Environment
    char *getenv(const char *name);

    // System info
    typedef struct {
        char sysname[65];
        char nodename[65];
        char release[65];
        char version[65];
        char machine[65];
    } utsname;

    int uname(utsname *buf);
]]

print("Process ID:", tonumber(ffi.C.getpid()))
print("Parent PID:", tonumber(ffi.C.getppid()))
print("User ID:", tonumber(ffi.C.getuid()))
print("Group ID:", tonumber(ffi.C.getgid()))

-- Environment variables
local path = ffi.C.getenv("PATH")
if path ~= nil then
    print("PATH:", ffi.string(path):sub(1, 60) .. "...")
end

local home = ffi.C.getenv("HOME")
if home ~= nil then
    print("HOME:", ffi.string(home))
end

-- System information
local uts = ffi.new("utsname")
if ffi.C.uname(uts) == 0 then
    print("\nSystem Information:")
    print("  OS:", ffi.string(uts.sysname))
    print("  Node:", ffi.string(uts.nodename))
    print("  Release:", ffi.string(uts.release))
    print("  Machine:", ffi.string(uts.machine))
end
```

---

## ตัวอย่างที่ 13: Performance Comparison - FFI vs Pure Lua

```lua
local ffi = require("ffi")

ffi.cdef[[
    double sqrt(double x);
    double sin(double x);
    double cos(double x);
]]

local libm = ffi.load("m")

-- Benchmark function
local function benchmark(name, iterations, func)
    local start = os.clock()
    local result = func(iterations)
    local elapsed = os.clock() - start
    print(string.format("%s: %.4f seconds (result=%.6f)",
        name, elapsed, result))
    return elapsed
end

local N = 10000000

-- Pure Lua math
local t1 = benchmark("Pure Lua math.sqrt", N, function(n)
    local sum = 0
    for i = 1, n do
        sum = sum + math.sqrt(i)
    end
    return sum
end)

-- FFI C sqrt
local t2 = benchmark("FFI ffi.C.sqrt", N, function(n)
    local sum = 0
    for i = 1, n do
        sum = sum + ffi.C.sqrt(i)
    end
    return sum
end)

-- FFI libm sqrt
local t3 = benchmark("FFI libm.sqrt", N, function(n)
    local sum = 0
    for i = 1, n do
        sum = sum + libm.pow(i, 0.5)
    end
    return sum
end)

print(string.format("\nSpeedup (Lua vs FFI C): %.2fx", t1/t2))
```

---

## ตัวอย่างที่ 14: Memory Management ใน FFI

```lua
local ffi = require("ffi")

ffi.cdef[[
    void *malloc(size_t size);
    void *calloc(size_t nmemb, size_t size);
    void *realloc(void *ptr, size_t size);
    void free(void *ptr);
]]

-- GC-managed memory (อัตโนมัติ)
local gc_buf = ffi.new("uint8_t[1024]")
-- จะถูก free โดย GC อัตโนมัติ

-- Manual memory management
local ptr = ffi.C.malloc(256)
if ptr == nil then
    error("malloc failed!")
end
print("Allocated 256 bytes at:", tostring(ptr))

-- ใช้ ffi.gc() เพื่อ attach finalizer
local managed = ffi.gc(
    ffi.cast("uint8_t*", ffi.C.malloc(512)),
    function(p)
        ffi.C.free(p)
        print("Auto-freed memory")
    end
)

ffi.C.free(ptr)
print("Manually freed memory")

-- calloc (zero-initialized)
local zeroed = ffi.C.calloc(10, ffi.sizeof("int"))
local iptr = ffi.cast("int*", zeroed)
print("\nZero-initialized array:")
for i = 0, 9 do
    io.write(iptr[i] .. " ")
end
print()
ffi.C.free(zeroed)

-- realloc
local buf = ffi.C.malloc(64)
print("\nInitial size: 64")
buf = ffi.C.realloc(buf, 256)
print("Reallocated to: 256")
ffi.C.free(buf)
```

---

## ตัวอย่างที่ 15: FFI Metatypes

```lua
local ffi = require("ffi")

ffi.cdef[[
    typedef struct {
        double x;
        double y;
    } Vec2;
]]

-- เพิ่ม methods ให้กับ C struct ผ่าน metatype
local Vec2_mt = {
    __index = {
        length = function(self)
            return math.sqrt(self.x^2 + self.y^2)
        end,
        normalize = function(self)
            local len = math.sqrt(self.x^2 + self.y^2)
            return ffi.new("Vec2", {self.x/len, self.y/len})
        end,
        dot = function(self, other)
            return self.x * other.x + self.y * other.y
        end,
    },
    __add = function(a, b)
        return ffi.new("Vec2", {a.x + b.x, a.y + b.y})
    end,
    __sub = function(a, b)
        return ffi.new("Vec2", {a.x - b.x, a.y - b.y})
    end,
    __mul = function(a, b)
        if type(a) == "number" then
            return ffi.new("Vec2", {a * b.x, a * b.y})
        elseif type(b) == "number" then
            return ffi.new("Vec2", {a.x * b, a.y * b})
        end
    end,
    __tostring = function(self)
        return string.format("Vec2(%.3f, %.3f)", self.x, self.y)
    end,
}

ffi.metatype("Vec2", Vec2_mt)

local v1 = ffi.new("Vec2", {3, 4})
local v2 = ffi.new("Vec2", {1, 2})

print("v1:", tostring(v1))
print("v2:", tostring(v2))
print("v1 + v2:", tostring(v1 + v2))
print("v1 - v2:", tostring(v1 - v2))
print("v1 * 2:", tostring(v1 * 2))
print("|v1|:", v1:length())
print("v1 normalized:", tostring(v1:normalize()))
print("v1 · v2:", v1:dot(v2))
```

---

## ตัวอย่างที่ 16: FFI กับ Enums และ Constants

```lua
local ffi = require("ffi")

ffi.cdef[[
    // Unix file permissions
    static const int S_IRUSR = 0400;
    static const int S_IWUSR = 0200;
    static const int S_IXUSR = 0100;
    static const int S_IRGRP = 040;
    static const int S_IWGRP = 020;

    // Enum-like constants
    enum {
        SEEK_SET = 0,
        SEEK_CUR = 1,
        SEEK_END = 2
    };

    // Open flags
    enum {
        O_RDONLY = 0,
        O_WRONLY = 1,
        O_RDWR   = 2,
        O_CREAT  = 64,
        O_TRUNC  = 512
    };
]]

-- ใช้ constants
print("SEEK_SET:", ffi.C.SEEK_SET)
print("SEEK_CUR:", ffi.C.SEEK_CUR)
print("SEEK_END:", ffi.C.SEEK_END)
print("O_RDONLY:", ffi.C.O_RDONLY)
print("O_WRONLY:", ffi.C.O_WRONLY)
print("O_RDWR:", ffi.C.O_RDWR)

-- สร้าง permission flags
local perms = bit.bor(ffi.C.S_IRUSR, ffi.C.S_IWUSR)
print(string.format("Read+Write permission: 0%o", perms))
```

---

## ตัวอย่างที่ 17: Error Handling ใน FFI

```lua
local ffi = require("ffi")

ffi.cdef[[
    int *__errno_location(void);
    char *strerror(int errnum);

    int open(const char *pathname, int flags);
    int close(int fd);
    ssize_t read(int fd, void *buf, size_t count);
]]

-- Helper function สำหรับดึง errno
local function get_errno()
    return ffi.C.__errno_location()[0]
end

-- Helper function สำหรับ error message
local function strerror(err)
    return ffi.string(ffi.C.strerror(err))
end

-- ลองเปิดไฟล์ที่ไม่มีอยู่
local O_RDONLY = 0
local fd = ffi.C.open("/nonexistent/file.txt", O_RDONLY)
if fd == -1 then
    local err = get_errno()
    print(string.format("Error opening file: errno=%d (%s)",
        err, strerror(err)))
end

-- Wrapper function ที่จัดการ errors
local function safe_open(path, flags)
    local fd = ffi.C.open(path, flags)
    if fd == -1 then
        local err = get_errno()
        return nil, string.format("Failed to open '%s': %s",
            path, strerror(err))
    end
    return fd, nil
end

local fd2, err = safe_open("/tmp/test_ffi.txt", O_RDONLY)
if err then
    print("Safe open error:", err)
else
    print("Opened successfully, fd =", fd2)
    ffi.C.close(fd2)
end
```

---

## ตัวอย่างที่ 18: Complex Data Structures

```lua
local ffi = require("ffi")

-- Linked list node
ffi.cdef[[
    typedef struct Node {
        int value;
        struct Node *next;
    } Node;

    void *malloc(size_t size);
    void free(void *ptr);
]]

-- สร้าง linked list
local function new_node(value)
    local node = ffi.cast("Node*", ffi.C.malloc(ffi.sizeof("Node")))
    node.value = value
    node.next = nil
    return node
end

local function free_list(head)
    while head ~= nil do
        local next = head.next
        ffi.C.free(head)
        head = next
    end
end

-- สร้าง list
local head = new_node(1)
local current = head
for i = 2, 5 do
    local node = new_node(i * 10)
    current.next = node
    current = node
end

-- traverse
print("Linked List:")
current = head
while current ~= nil do
    io.write(current.value .. " -> ")
    current = current.next
end
print("nil")

-- free memory
free_list(head)
print("List freed")
```

---

## ตัวอย่างที่ 19: FFI กับ Varargs Functions

```lua
local ffi = require("ffi")

ffi.cdef[[
    int printf(const char *fmt, ...);
    int sprintf(char *str, const char *fmt, ...);
    int snprintf(char *str, size_t size, const char *fmt, ...);
    int sscanf(const char *str, const char *fmt, ...);
]]

-- ใช้ printf
ffi.C.printf("Hello, %s! You are %d years old.\n", "World", 25)
ffi.C.printf("Pi is approximately %.4f\n", math.pi)
ffi.C.printf("Hex: 0x%08X\n", 255)

-- ใช้ sprintf
local buf = ffi.new("char[256]")
ffi.C.sprintf(buf, "Value: %d, Float: %.2f", 42, 3.14)
print("sprintf result:", ffi.string(buf))

-- ใช้ snprintf (safe)
ffi.C.snprintf(buf, 256, "Safe format: %s %d", "test", 123)
print("snprintf result:", ffi.string(buf))

-- ใช้ sscanf สำหรับ parsing
local input = "42 3.14 hello"
local i_val = ffi.new("int[1]")
local f_val = ffi.new("double[1]")
local s_val = ffi.new("char[64]")

local n = ffi.C.sscanf(input, "%d %lf %s", i_val, f_val, s_val)
print(string.format("\nsscanf parsed %d values:", n))
print("  int:", i_val[0])
print("  double:", f_val[0])
print("  string:", ffi.string(s_val))
```

---

## ตัวอย่างที่ 20: Platform Detection และ Conditional FFI

```lua
local ffi = require("ffi")

-- ตรวจสอบ OS
local function get_os()
    if ffi.os == "Windows" then
        return "Windows"
    elseif ffi.os == "Linux" then
        return "Linux"
    elseif ffi.os == "OSX" then
        return "macOS"
    else
        return ffi.os
    end
end

-- ตรวจสอบ Architecture
local function get_arch()
    return ffi.arch  -- "x64", "x86", "arm", "arm64", etc.
end

print("OS:", get_os())
print("Architecture:", get_arch())
print("Is 64-bit:", ffi.abi("64bit"))
print("Is little-endian:", ffi.abi("le"))
print("sizeof(pointer):", ffi.sizeof("void*"), "bytes")

-- Conditional loading ตาม platform
if ffi.os == "Windows" then
    ffi.cdef[[
        int GetLastError(void);
        int Sleep(unsigned long dwMilliseconds);
    ]]
    print("Windows-specific functions loaded")
elseif ffi.os == "Linux" or ffi.os == "OSX" then
    ffi.cdef[[
        unsigned int sleep(unsigned int seconds);
        int usleep(unsigned int usec);
    ]]
    print("Unix-specific functions loaded")
end

-- Type sizes vary by platform
print("\nType sizes:")
for _, t in ipairs({"char", "short", "int", "long", "float", "double"}) do
    print(string.format("  %s: %d bytes", t, ffi.sizeof(t)))
end
```

---

## ตัวอย่างที่ 21: FFI กับ OpenGL (Concept Example)

```lua
local ffi = require("ffi")

-- ตัวอย่าง conceptual สำหรับ OpenGL-style bindings
ffi.cdef[[
    typedef unsigned int GLenum;
    typedef unsigned int GLuint;
    typedef int GLint;
    typedef float GLfloat;
    typedef int GLsizei;
    typedef unsigned char GLboolean;
    typedef void GLvoid;

    // Vertex structure
    typedef struct {
        GLfloat x, y, z;
        GLfloat r, g, b, a;
        GLfloat u, v;
    } Vertex;

    typedef struct {
        Vertex vertices[4];
        GLuint indices[6];
    } Quad;
]]

-- สร้าง quad สำหรับ rendering
local function create_quad(x, y, w, h, r, g, b)
    local quad = ffi.new("Quad")

    -- Top-left
    quad.vertices[0].x = x
    quad.vertices[0].y = y
    quad.vertices[0].z = 0
    quad.vertices[0].r = r
    quad.vertices[0].g = g
    quad.vertices[0].b = b
    quad.vertices[0].a = 1.0
    quad.vertices[0].u = 0
    quad.vertices[0].v = 0

    -- Top-right
    quad.vertices[1].x = x + w
    quad.vertices[1].y = y
    quad.vertices[1].z = 0
    quad.vertices[1].r = r
    quad.vertices[1].g = g
    quad.vertices[1].b = b
    quad.vertices[1].a = 1.0
    quad.vertices[1].u = 1
    quad.vertices[1].v = 0

    -- ตั้ง indices
    quad.indices[0] = 0
    quad.indices[1] = 1
    quad.indices[2] = 2
    quad.indices[3] = 2
    quad.indices[4] = 3
    quad.indices[5] = 0

    return quad
end

local q = create_quad(0, 0, 100, 100, 1.0, 0.5, 0.0)
print("Quad created:")
print(string.format("  Vertex 0: (%.1f, %.1f, %.1f)",
    q.vertices[0].x, q.vertices[0].y, q.vertices[0].z))
print(string.format("  Color: (%.1f, %.1f, %.1f)",
    q.vertices[0].r, q.vertices[0].g, q.vertices[0].b))
print("Size of Quad:", ffi.sizeof("Quad"), "bytes")
```

---

## ตัวอย่างที่ 22: FFI vs lua_CFunction Comparison

```lua
-- บทสรุปการเปรียบเทียบ
-- FFI vs Lua/C API (lua_CFunction)
--
-- FFI (LuaJIT):
-- + ง่ายกว่า ไม่ต้องเขียน C code
-- + ไม่ต้องคอมไพล์ extension
-- + JIT-compiled ได้
-- + เข้าถึง C structs โดยตรง
-- - ใช้ได้เฉพาะ LuaJIT
-- - FFI overhead บางกรณี
-- - Type safety น้อยกว่า
--
-- lua_CFunction (C extension):
-- + รองรับ Lua 5.x ทุกเวอร์ชัน
-- + Type safety สูง
-- + เหมาะกับ complex logic
-- - ต้องเขียนและคอมไพล์ C code
-- - API ซับซ้อนกว่า
-- - ต้องจัดการ stack ด้วยตนเอง

local ffi = require("ffi")

-- ตัวอย่าง FFI binding สำหรับ string operation
ffi.cdef[[
    int toupper(int c);
    int tolower(int c);
    int isalpha(int c);
    int isdigit(int c);
    int isspace(int c);
]]

-- ฟังก์ชัน Lua ที่ใช้ FFI
local function to_upper(s)
    local result = {}
    for i = 1, #s do
        result[i] = string.char(ffi.C.toupper(s:byte(i)))
    end
    return table.concat(result)
end

local function count_digits(s)
    local count = 0
    for i = 1, #s do
        if ffi.C.isdigit(s:byte(i)) ~= 0 then
            count = count + 1
        end
    end
    return count
end

print(to_upper("hello world 123"))  -- HELLO WORLD 123
print("Digits in 'abc123xyz789':", count_digits("abc123xyz789"))  -- 6
```

---

## ตัวอย่างที่ 23: Struct Packing และ Alignment

```lua
local ffi = require("ffi")

-- Default alignment
ffi.cdef[[
    typedef struct {
        char   a;    // 1 byte + 3 padding
        int    b;    // 4 bytes
        char   c;    // 1 byte + 7 padding
        double d;    // 8 bytes
    } Aligned;
]]

-- Packed struct (__attribute__((packed)) equivalent)
ffi.cdef[[
    #pragma pack(1)
    typedef struct {
        char   a;
        int    b;
        char   c;
        double d;
    } Packed;
    #pragma pack()
]]

print("Aligned struct size:", ffi.sizeof("Aligned"), "bytes")
print("Packed struct size:", ffi.sizeof("Packed"), "bytes")

-- Offset ของแต่ละ field
print("\nAligned field offsets:")
print("  a:", ffi.offsetof("Aligned", "a"))
print("  b:", ffi.offsetof("Aligned", "b"))
print("  c:", ffi.offsetof("Aligned", "c"))
print("  d:", ffi.offsetof("Aligned", "d"))

print("\nPacked field offsets:")
print("  a:", ffi.offsetof("Packed", "a"))
print("  b:", ffi.offsetof("Packed", "b"))
print("  c:", ffi.offsetof("Packed", "c"))
print("  d:", ffi.offsetof("Packed", "d"))
```

---

## ตัวอย่างที่ 24: Dynamic Library Loading

```lua
local ffi = require("ffi")

-- Helper ที่จัดการ library loading แบบ cross-platform
local function load_library(name)
    local lib = nil
    local err_msgs = {}

    -- ลอง platform-specific names
    local names = {}
    if ffi.os == "Windows" then
        names = {name .. ".dll", "lib" .. name .. ".dll"}
    elseif ffi.os == "OSX" then
        names = {"lib" .. name .. ".dylib", "lib" .. name .. ".so"}
    else
        names = {"lib" .. name .. ".so", "lib" .. name .. ".so.0"}
    end

    for _, libname in ipairs(names) do
        local ok, result = pcall(ffi.load, libname)
        if ok then
            return result, nil
        else
            table.insert(err_msgs, libname .. ": " .. tostring(result))
        end
    end

    return nil, "ไม่พบ library: " .. table.concat(err_msgs, "; ")
end

-- ลองโหลด libz (zlib)
local zlib, err = load_library("z")
if zlib then
    print("โหลด zlib สำเร็จ")

    ffi.cdef[[
        unsigned long adler32(unsigned long adler,
                              const unsigned char *buf,
                              unsigned int len);
        unsigned long crc32(unsigned long crc,
                            const unsigned char *buf,
                            unsigned int len);
    ]]

    local data = "Hello, World!"
    local crc = zlib.crc32(0, data, #data)
    print(string.format("CRC32 of '%s': 0x%08X", data, crc))

    local adler = zlib.adler32(1, data, #data)
    print(string.format("Adler32 of '%s': 0x%08X", data, adler))
else
    print("ไม่สามารถโหลด zlib:", err)
    print("(ปกติ zlib มักจะมีอยู่บน Linux/Mac)")
end
```

---

## ตัวอย่างที่ 25: FFI กับ Regex (POSIX)

```lua
local ffi = require("ffi")

-- POSIX regex
ffi.cdef[[
    typedef struct {
        size_t re_nsub;
        void  *re_comp;
        int    re_cflags;
        size_t re_endp;
        size_t re_len;
    } regex_t;

    typedef struct {
        int rm_so;
        int rm_eo;
    } regmatch_t;

    int regcomp(regex_t *preg, const char *regex, int cflags);
    int regexec(const regex_t *preg, const char *string,
                size_t nmatch, regmatch_t pmatch[], int eflags);
    void regfree(regex_t *preg);
    size_t regerror(int errcode, const regex_t *preg,
                    char *errbuf, size_t errbuf_size);

    static const int REG_EXTENDED  = 1;
    static const int REG_ICASE     = 2;
    static const int REG_NEWLINE   = 4;
    static const int REG_NOSUB     = 8;
]]

local function regex_match(pattern, text)
    local preg = ffi.new("regex_t")
    local ret = ffi.C.regcomp(preg, pattern, ffi.C.REG_EXTENDED)
    if ret ~= 0 then
        local errbuf = ffi.new("char[256]")
        ffi.C.regerror(ret, preg, errbuf, 256)
        return nil, ffi.string(errbuf)
    end

    local nmatch = 10
    local pmatch = ffi.new("regmatch_t[10]")
    ret = ffi.C.regexec(preg, text, nmatch, pmatch, 0)
    ffi.C.regfree(preg)

    if ret ~= 0 then
        return nil, "ไม่พบ match"
    end

    -- ดึง matches
    local matches = {}
    for i = 0, nmatch - 1 do
        if pmatch[i].rm_so == -1 then break end
        local s = tonumber(pmatch[i].rm_so) + 1
        local e = tonumber(pmatch[i].rm_eo)
        table.insert(matches, text:sub(s, e))
    end

    return matches
end

local text = "Email: user@example.com and admin@test.org"
local matches, err = regex_match(
    "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}",
    text
)

if matches then
    print("Found email:", matches[1])
else
    print("Error:", err)
end
```

---

## ตัวอย่างที่ 26: FFI Numeric Types ทั้งหมด

```lua
local ffi = require("ffi")

-- ทดสอบ numeric types ทั้งหมด
ffi.cdef[[
    typedef signed char        int8_t;
    typedef unsigned char      uint8_t;
    typedef signed short       int16_t;
    typedef unsigned short     uint16_t;
    typedef signed int         int32_t;
    typedef unsigned int       uint32_t;
    typedef signed long long   int64_t;
    typedef unsigned long long uint64_t;
]]

-- Integer ranges
print("Type ranges:")
print(string.format("int8_t:  %d to %d", -128, 127))
print(string.format("uint8_t: 0 to %d", 255))
print(string.format("int16_t: %d to %d", -32768, 32767))
print(string.format("uint16_t: 0 to %d", 65535))

-- สร้างค่าแต่ละ type
local vals = {
    int8  = ffi.new("int8_t", 100),
    uint8 = ffi.new("uint8_t", 255),
    int16 = ffi.new("int16_t", 32000),
    int32 = ffi.new("int32_t", 2000000000),
    int64 = ffi.new("int64_t", 9000000000000LL),
}

print("\nValues:")
for name, val in pairs(vals) do
    print(string.format("  %s = %s", name, tostring(val)))
end

-- Overflow behavior
local b = ffi.new("uint8_t", 255)
b = b + 1  -- wraps to 0
print("\nOverflow test (uint8 255+1):", tostring(b))

-- 64-bit integers
local big = ffi.new("int64_t", 2^53)
print("2^53 as int64:", tostring(big))

local max64 = ffi.new("int64_t", "9223372036854775807")
print("MAX int64:", tostring(max64))
```

---

## ตัวอย่างที่ 27: FFI กับ Audio Buffer Processing

```lua
local ffi = require("ffi")

-- จำลองการประมวลผล audio buffer
ffi.cdef[[
    typedef struct {
        float *samples;
        int    length;
        int    sample_rate;
        int    channels;
    } AudioBuffer;

    void *malloc(size_t size);
    void free(void *ptr);
]]

local function create_audio_buffer(length, sample_rate, channels)
    local buf = ffi.new("AudioBuffer")
    local samples = ffi.cast("float*",
        ffi.C.malloc(length * channels * ffi.sizeof("float")))

    buf.samples = samples
    buf.length = length
    buf.sample_rate = sample_rate
    buf.channels = channels

    return buf
end

local function free_audio_buffer(buf)
    ffi.C.free(buf.samples)
end

-- สร้าง sine wave
local function generate_sine(buf, freq, amplitude)
    local dt = 1.0 / buf.sample_rate
    for i = 0, buf.length - 1 do
        local t = i * dt
        buf.samples[i] = amplitude * math.sin(2 * math.pi * freq * t)
    end
end

-- Apply gain
local function apply_gain(buf, gain)
    for i = 0, buf.length - 1 do
        buf.samples[i] = buf.samples[i] * gain
    end
end

-- Calculate RMS
local function calc_rms(buf)
    local sum = 0
    for i = 0, buf.length - 1 do
        sum = sum + buf.samples[i] ^ 2
    end
    return math.sqrt(sum / buf.length)
end

local audio = create_audio_buffer(44100, 44100, 1)
generate_sine(audio, 440, 0.5)  -- 440 Hz sine wave
print(string.format("Generated %d samples at %d Hz",
    audio.length, audio.sample_rate))
print(string.format("RMS level: %.4f", calc_rms(audio)))

apply_gain(audio, 0.5)
print(string.format("After 50%% gain: %.4f", calc_rms(audio)))

free_audio_buffer(audio)
print("Audio buffer freed")
```

---

## ตัวอย่างที่ 28: Real Example - SQLite via FFI

```lua
local ffi = require("ffi")

-- SQLite3 bindings
ffi.cdef[[
    typedef struct sqlite3 sqlite3;
    typedef struct sqlite3_stmt sqlite3_stmt;

    int sqlite3_open(const char *filename, sqlite3 **ppDb);
    int sqlite3_close(sqlite3 *db);
    int sqlite3_exec(sqlite3 *db, const char *sql,
                     void *callback, void *arg, char **errmsg);
    int sqlite3_prepare_v2(sqlite3 *db, const char *zSql,
                           int nByte, sqlite3_stmt **ppStmt,
                           const char **pzTail);
    int sqlite3_step(sqlite3_stmt *stmt);
    int sqlite3_finalize(sqlite3_stmt *stmt);
    int sqlite3_column_count(sqlite3_stmt *stmt);
    int sqlite3_column_type(sqlite3_stmt *stmt, int iCol);
    int sqlite3_column_int(sqlite3_stmt *stmt, int iCol);
    double sqlite3_column_double(sqlite3_stmt *stmt, int iCol);
    const unsigned char *sqlite3_column_text(sqlite3_stmt *stmt,
                                              int iCol);
    void sqlite3_free(void *ptr);
    const char *sqlite3_errmsg(sqlite3 *db);

    static const int SQLITE_OK   = 0;
    static const int SQLITE_ROW  = 100;
    static const int SQLITE_DONE = 101;
    static const int SQLITE_INTEGER = 1;
    static const int SQLITE_FLOAT   = 2;
    static const int SQLITE_TEXT    = 3;
    static const int SQLITE_BLOB    = 4;
    static const int SQLITE_NULL    = 5;
]]

-- ลองโหลด SQLite3
local ok, sqlite3 = pcall(ffi.load, "sqlite3")
if not ok then
    print("SQLite3 ไม่พบใน system (ต้องติดตั้ง libsqlite3)")
    print("Example code shown but not executed")
else
    -- เปิด in-memory database
    local db_ptr = ffi.new("sqlite3*[1]")
    local rc = sqlite3.sqlite3_open(":memory:", db_ptr)
    local db = db_ptr[0]

    if rc ~= sqlite3.SQLITE_OK then
        print("ไม่สามารถเปิด database")
    else
        print("SQLite3 database opened (in-memory)")

        -- สร้าง table
        local create_sql = [[
            CREATE TABLE users (
                id INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                age INTEGER,
                email TEXT
            )
        ]]
        local errmsg = ffi.new("char*[1]")
        rc = sqlite3.sqlite3_exec(db, create_sql, nil, nil, errmsg)
        if rc == sqlite3.SQLITE_OK then
            print("Table 'users' created")
        end

        -- Insert data
        local inserts = {
            "INSERT INTO users VALUES (1, 'Alice', 30, 'alice@example.com')",
            "INSERT INTO users VALUES (2, 'Bob', 25, 'bob@example.com')",
            "INSERT INTO users VALUES (3, 'Charlie', 35, 'charlie@example.com')",
        }
        for _, sql in ipairs(inserts) do
            sqlite3.sqlite3_exec(db, sql, nil, nil, nil)
        end
        print("Inserted 3 rows")

        -- Query data
        local stmt_ptr = ffi.new("sqlite3_stmt*[1]")
        local query = "SELECT * FROM users ORDER BY age"
        rc = sqlite3.sqlite3_prepare_v2(db, query, -1, stmt_ptr, nil)
        local stmt = stmt_ptr[0]

        if rc == sqlite3.SQLITE_OK then
            print("\nAll users:")
            print(string.rep("-", 50))
            while sqlite3.sqlite3_step(stmt) == sqlite3.SQLITE_ROW do
                local id = sqlite3.sqlite3_column_int(stmt, 0)
                local name = ffi.string(sqlite3.sqlite3_column_text(stmt, 1))
                local age = sqlite3.sqlite3_column_int(stmt, 2)
                local email = ffi.string(sqlite3.sqlite3_column_text(stmt, 3))
                print(string.format("  %d | %-10s | %3d | %s",
                    id, name, age, email))
            end
            sqlite3.sqlite3_finalize(stmt)
        end

        sqlite3.sqlite3_close(db)
        print("\nDatabase closed")
    end
end
```

---

## ตัวอย่างที่ 29: FFI กับ Hash Functions (via libssl)

```lua
local ffi = require("ffi")

-- MD5 via libcrypto (OpenSSL)
ffi.cdef[[
    typedef struct env_md_ctx_st EVP_MD_CTX;
    typedef struct env_md_st EVP_MD;

    EVP_MD_CTX *EVP_MD_CTX_new(void);
    void EVP_MD_CTX_free(EVP_MD_CTX *ctx);
    const EVP_MD *EVP_md5(void);
    const EVP_MD *EVP_sha1(void);
    const EVP_MD *EVP_sha256(void);
    int EVP_DigestInit_ex(EVP_MD_CTX *ctx, const EVP_MD *type,
                          void *impl);
    int EVP_DigestUpdate(EVP_MD_CTX *ctx, const void *d, size_t cnt);
    int EVP_DigestFinal_ex(EVP_MD_CTX *ctx, unsigned char *md,
                           unsigned int *s);

    unsigned char *MD5(const unsigned char *d, size_t n,
                       unsigned char *md);
    unsigned char *SHA256(const unsigned char *d, size_t n,
                          unsigned char *md);
]]

local function bytes_to_hex(bytes, len)
    local hex = {}
    for i = 0, len - 1 do
        hex[i+1] = string.format("%02x", bytes[i])
    end
    return table.concat(hex)
end

local ok, crypto = pcall(ffi.load, "crypto")
if ok then
    local data = "Hello, World!"
    local digest = ffi.new("unsigned char[32]")

    -- MD5
    crypto.MD5(data, #data, digest)
    print("MD5:", bytes_to_hex(digest, 16))

    -- SHA256
    crypto.SHA256(data, #data, digest)
    print("SHA256:", bytes_to_hex(digest, 32))
else
    print("libcrypto (OpenSSL) ไม่พบ")
    print("ติดตั้งด้วย: apt-get install libssl-dev")
end
```

---

## ตัวอย่างที่ 30: High-Performance Vector Operations

```lua
local ffi = require("ffi")

ffi.cdef[[
    typedef struct {
        float data[4];
    } Vec4;

    typedef struct {
        float data[16];
    } Mat4;
]]

-- Vector 4D operations ที่ใช้ FFI array
local function vec4_add(result, a, b)
    for i = 0, 3 do
        result.data[i] = a.data[i] + b.data[i]
    end
end

local function vec4_dot(a, b)
    local sum = 0
    for i = 0, 3 do
        sum = sum + a.data[i] * b.data[i]
    end
    return sum
end

local function mat4_identity(m)
    for i = 0, 15 do m.data[i] = 0 end
    m.data[0]  = 1
    m.data[5]  = 1
    m.data[10] = 1
    m.data[15] = 1
end

local function mat4_mul_vec4(result, m, v)
    for row = 0, 3 do
        local sum = 0
        for col = 0, 3 do
            sum = sum + m.data[row * 4 + col] * v.data[col]
        end
        result.data[row] = sum
    end
end

-- Test
local v1 = ffi.new("Vec4", {{1, 2, 3, 4}})
local v2 = ffi.new("Vec4", {{5, 6, 7, 8}})
local v3 = ffi.new("Vec4")

vec4_add(v3, v1, v2)
print(string.format("v1 + v2 = (%.0f, %.0f, %.0f, %.0f)",
    v3.data[0], v3.data[1], v3.data[2], v3.data[3]))

print("v1 · v2 =", vec4_dot(v1, v2))

local m = ffi.new("Mat4")
mat4_identity(m)
local result = ffi.new("Vec4")
mat4_mul_vec4(result, m, v1)
print(string.format("Identity * v1 = (%.0f, %.0f, %.0f, %.0f)",
    result.data[0], result.data[1], result.data[2], result.data[3]))
```

---

## ตัวอย่างที่ 31: FFI Struct Iteration

```lua
local ffi = require("ffi")

ffi.cdef[[
    typedef struct {
        char symbol[8];
        double price;
        long   volume;
        double change_pct;
    } StockTick;
]]

-- จำลอง stock data
local ticks = ffi.new("StockTick[5]")

local data = {
    {"AAPL", 182.50, 45000000, 1.25},
    {"GOOGL", 140.20, 22000000, -0.80},
    {"MSFT", 375.10, 31000000, 2.10},
    {"AMZN", 185.30, 38000000, 0.55},
    {"META", 485.90, 18000000, 3.20},
}

for i, d in ipairs(data) do
    ffi.copy(ticks[i-1].symbol, d[1])
    ticks[i-1].price      = d[2]
    ticks[i-1].volume     = d[3]
    ticks[i-1].change_pct = d[4]
end

-- แสดง stock data
print(string.format("%-8s %10s %12s %8s",
    "Symbol", "Price", "Volume", "Change%"))
print(string.rep("-", 44))

for i = 0, 4 do
    local t = ticks[i]
    local arrow = t.change_pct >= 0 and "▲" or "▼"
    print(string.format("%-8s %10.2f %12d  %s%.2f%%",
        ffi.string(t.symbol), t.price,
        tonumber(t.volume), arrow, math.abs(t.change_pct)))
end

-- หา best gainer
local best_idx = 0
for i = 1, 4 do
    if ticks[i].change_pct > ticks[best_idx].change_pct then
        best_idx = i
    end
end
print("\nBest gainer:", ffi.string(ticks[best_idx].symbol),
    string.format("(+%.2f%%)", ticks[best_idx].change_pct))
```

---

## ตัวอย่างที่ 32: Buffer Protocol และ Binary Data

```lua
local ffi = require("ffi")

-- Binary data processing
ffi.cdef[[
    typedef struct {
        uint16_t magic;
        uint8_t  version;
        uint8_t  flags;
        uint32_t size;
        uint8_t  data[1];
    } PacketHeader;
]]

local MAGIC = 0xCAFE
local VERSION = 1

-- Encode binary packet
local function encode_packet(data, flags)
    flags = flags or 0
    local total_size = ffi.sizeof("PacketHeader") - 1 + #data
    local buf = ffi.new("uint8_t[?]", total_size)
    local header = ffi.cast("PacketHeader*", buf)

    header.magic   = MAGIC
    header.version = VERSION
    header.flags   = flags
    header.size    = #data
    ffi.copy(header.data, data, #data)

    return ffi.string(buf, total_size)
end

-- Decode binary packet
local function decode_packet(raw)
    local buf = ffi.new("uint8_t[?]", #raw)
    ffi.copy(buf, raw, #raw)
    local header = ffi.cast("PacketHeader*", buf)

    if header.magic ~= MAGIC then
        return nil, "Invalid magic bytes"
    end
    if header.version ~= VERSION then
        return nil, "Version mismatch"
    end

    local size = tonumber(header.size)
    local data = ffi.string(header.data, size)

    return {
        version = tonumber(header.version),
        flags   = tonumber(header.flags),
        size    = size,
        data    = data,
    }
end

-- Test encoding/decoding
local original = "Hello, Binary World!"
local packet = encode_packet(original, 0x01)
print(string.format("Encoded packet: %d bytes", #packet))

local decoded, err = decode_packet(packet)
if decoded then
    print("Decoded successfully:")
    print("  Version:", decoded.version)
    print("  Flags:", string.format("0x%02X", decoded.flags))
    print("  Data:", decoded.data)
else
    print("Error:", err)
end
```

---

## ตัวอย่างที่ 33: FFI กับ Socket Programming

```lua
local ffi = require("ffi")

-- POSIX socket definitions
ffi.cdef[[
    typedef int socklen_t;

    struct sockaddr {
        uint16_t sa_family;
        char     sa_data[14];
    };

    struct sockaddr_in {
        uint16_t sin_family;
        uint16_t sin_port;
        uint32_t sin_addr;
        uint8_t  sin_zero[8];
    };

    int    socket(int domain, int type, int protocol);
    int    connect(int sockfd, const struct sockaddr *addr,
                   socklen_t addrlen);
    int    close(int fd);
    ssize_t send(int sockfd, const void *buf, size_t len, int flags);
    ssize_t recv(int sockfd, void *buf, size_t len, int flags);
    uint16_t htons(uint16_t hostshort);
    uint32_t htonl(uint32_t hostlong);
    int    inet_pton(int af, const char *src, void *dst);
    char  *inet_ntop(int af, const void *src, char *dst, socklen_t size);

    static const int AF_INET     = 2;
    static const int SOCK_STREAM = 1;
    static const int SOCK_DGRAM  = 2;
    static const int IPPROTO_TCP = 6;
]]

-- สร้าง TCP connection แบบ conceptual
local function tcp_connect(host_ip, port)
    local sock = ffi.C.socket(ffi.C.AF_INET, ffi.C.SOCK_STREAM, 0)
    if sock == -1 then
        return nil, "socket() failed"
    end

    local addr = ffi.new("struct sockaddr_in")
    addr.sin_family = ffi.C.AF_INET
    addr.sin_port   = ffi.C.htons(port)

    local ret = ffi.C.inet_pton(ffi.C.AF_INET, host_ip,
        ffi.cast("void*", ffi.addressof(addr, "sin_addr")))
    if ret ~= 1 then
        ffi.C.close(sock)
        return nil, "invalid IP address"
    end

    ret = ffi.C.connect(sock,
        ffi.cast("struct sockaddr*", addr),
        ffi.sizeof("struct sockaddr_in"))

    if ret == -1 then
        ffi.C.close(sock)
        return nil, "connect() failed"
    end

    return sock, nil
end

print("TCP connect function defined")
print("(ต้องการ network access จริงๆ ในการทดสอบ)")
print("socket() system call:", ffi.C.AF_INET)
```

---

## ตัวอย่างที่ 34: Advanced Callback Patterns

```lua
local ffi = require("ffi")

-- Event system ที่ใช้ callbacks
ffi.cdef[[
    typedef void (*EventCallback)(int event_type, void *user_data);

    typedef struct {
        EventCallback callback;
        void         *user_data;
        int           event_type;
    } EventListener;
]]

-- Lua-side event dispatcher
local listeners = {}
local callback_refs = {}  -- เก็บ reference ไม่ให้ถูก GC

local EVENT_CLICK    = 1
local EVENT_KEYPRESS = 2
local EVENT_RESIZE   = 3

local function add_listener(event_type, func)
    local cb = ffi.cast("EventCallback", function(etype, udata)
        func(tonumber(etype))
    end)

    -- เก็บ reference
    table.insert(callback_refs, cb)

    local listener = ffi.new("EventListener")
    listener.callback   = cb
    listener.user_data  = nil
    listener.event_type = event_type

    table.insert(listeners, listener)
    return #listeners
end

local function dispatch_event(event_type)
    for _, listener in ipairs(listeners) do
        if tonumber(listener.event_type) == event_type then
            listener.callback(event_type, listener.user_data)
        end
    end
end

-- Register listeners
add_listener(EVENT_CLICK, function(etype)
    print("  [Click Handler] Received event:", etype)
end)

add_listener(EVENT_KEYPRESS, function(etype)
    print("  [KeyPress Handler] Received event:", etype)
end)

add_listener(EVENT_CLICK, function(etype)
    print("  [Second Click Handler] Also received:", etype)
end)

-- Dispatch events
print("Dispatching CLICK event:")
dispatch_event(EVENT_CLICK)

print("\nDispatching KEYPRESS event:")
dispatch_event(EVENT_KEYPRESS)

print("\nDispatching RESIZE event (no listeners):")
dispatch_event(EVENT_RESIZE)

-- Cleanup
for _, cb in ipairs(callback_refs) do
    cb:free()
end
print("\nAll callbacks freed")
```

---

## ตัวอย่างที่ 35: FFI Module Pattern

```lua
local ffi = require("ffi")

-- สร้าง module สำหรับ Math operations ผ่าน FFI
local mathffi = {}

ffi.cdef[[
    double sqrt(double x);
    double pow(double x, double y);
    double fabs(double x);
    double floor(double x);
    double ceil(double x);
    double round(double x);
    double fmod(double x, double y);
    double log(double x);
    double log2(double x);
    double log10(double x);
    double exp(double x);
    double sin(double x);
    double cos(double x);
    double tan(double x);
    double asin(double x);
    double acos(double x);
    double atan(double x);
    double atan2(double y, double x);
    double hypot(double x, double y);
]]

local libm = ffi.load("m")

-- Wrap ฟังก์ชันทั้งหมด
local math_funcs = {
    "sqrt", "pow", "fabs", "floor", "ceil", "round", "fmod",
    "log", "log2", "log10", "exp",
    "sin", "cos", "tan", "asin", "acos", "atan", "atan2", "hypot"
}

for _, name in ipairs(math_funcs) do
    mathffi[name] = libm[name]
end

-- Test
print("FFI Math Module:")
print("  sqrt(2):", mathffi.sqrt(2))
print("  pow(2,10):", mathffi.pow(2, 10))
print("  sin(pi/2):", mathffi.sin(math.pi/2))
print("  atan2(1,1)*4:", mathffi.atan2(1,1)*4)  -- pi
print("  hypot(3,4):", mathffi.hypot(3, 4))

-- Benchmark
local N = 5000000
local start = os.clock()
local s = 0
for i = 1, N do s = s + mathffi.sqrt(i) end
print(string.format("\nFFI sqrt %dM times: %.3fs", N/1e6, os.clock()-start))

start = os.clock()
s = 0
for i = 1, N do s = s + math.sqrt(i) end
print(string.format("Lua sqrt %dM times: %.3fs", N/1e6, os.clock()-start))
```

---

## ตัวอย่างที่ 36: Memory Pool ด้วย FFI

```lua
local ffi = require("ffi")

ffi.cdef[[
    void *malloc(size_t size);
    void free(void *ptr);
    void *memset(void *s, int c, size_t n);
]]

-- Memory Pool implementation
local MemPool = {}
MemPool.__index = MemPool

function MemPool.new(block_size, capacity)
    local pool = setmetatable({}, MemPool)
    pool.block_size = block_size
    pool.capacity   = capacity
    pool.used       = 0

    -- Allocate pool
    pool.memory = ffi.cast("uint8_t*",
        ffi.C.malloc(block_size * capacity))
    ffi.C.memset(pool.memory, 0, block_size * capacity)

    -- Free list (stack of available blocks)
    pool.free_list = {}
    for i = 0, capacity - 1 do
        table.insert(pool.free_list, i)
    end

    print(string.format("MemPool created: %d blocks x %d bytes = %d bytes total",
        capacity, block_size, block_size * capacity))

    return pool
end

function MemPool:alloc()
    if #self.free_list == 0 then
        return nil, "pool exhausted"
    end
    local idx = table.remove(self.free_list)
    self.used = self.used + 1
    return self.memory + (idx * self.block_size), idx
end

function MemPool:free_block(idx)
    -- Zero the block
    ffi.C.memset(self.memory + (idx * self.block_size), 0, self.block_size)
    table.insert(self.free_list, idx)
    self.used = self.used - 1
end

function MemPool:destroy()
    ffi.C.free(self.memory)
    self.memory = nil
end

function MemPool:stats()
    return {
        capacity  = self.capacity,
        used      = self.used,
        available = #self.free_list,
    }
end

-- Test
local pool = MemPool.new(64, 16)

local blocks = {}
for i = 1, 5 do
    local ptr, idx = pool:alloc()
    if ptr then
        ffi.copy(ptr, string.format("Block #%d data", i))
        blocks[i] = idx
        print(string.format("Allocated block %d", idx))
    end
end

local stats = pool:stats()
print(string.format("Stats: %d/%d used", stats.used, stats.capacity))

-- Free some blocks
pool:free_block(blocks[2])
pool:free_block(blocks[4])
stats = pool:stats()
print(string.format("After free: %d/%d used", stats.used, stats.capacity))

pool:destroy()
print("Pool destroyed")
```

---

## สรุป FFI

FFI ใน LuaJIT เป็นเครื่องมือที่ทรงพลังสำหรับ:

1. **ความเร็ว**: เรียก C functions ได้เร็วเกือบเท่า C native call
2. **ความสะดวก**: ไม่ต้องเขียน C wrapper code
3. **การเข้าถึง**: ใช้ system libraries ได้ทันที
4. **Memory Control**: จัดการ memory ระดับ C ได้โดยตรง

### Best Practices

```lua
-- 1. เก็บ cdata ที่ต้องใช้บ่อยไว้ใน local variable
local libm = ffi.load("m")  -- โหลดครั้งเดียว
local sqrt = libm.sqrt      -- cache function reference

-- 2. ใช้ ffi.gc() สำหรับ manual memory
local buf = ffi.gc(
    ffi.cast("void*", ffi.C.malloc(1024)),
    ffi.C.free
)

-- 3. ระวัง null pointer
local ptr = some_c_function()
if ptr == nil then
    error("got null pointer")
end

-- 4. ใช้ ffi.string() สำหรับแปลง C string
local lua_str = ffi.string(c_str_ptr)

-- 5. ใช้ tonumber() เพื่อแปลง cdata เป็น Lua number
local n = tonumber(some_int64_value)
```

### ข้อควรระวัง

- FFI ใช้ได้เฉพาะ **LuaJIT** เท่านั้น ไม่รองรับ Lua 5.x ดั้งเดิม
- Memory leaks เกิดขึ้นง่ายถ้าไม่ระวัง
- Type errors จาก FFI อาจทำให้ program crash ได้
- ต้องระวัง platform differences (Windows vs Linux vs macOS)
