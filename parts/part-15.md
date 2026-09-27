# บทที่ 15: Scope, Variables, และ Upvalues

## บทนำ

Scope (ขอบเขต) คือแนวคิดพื้นฐานสำคัญในการเขียนโปรแกรม บทนี้จะอธิบาย **Lexical Scoping** ใน Lua ซึ่งหมายความว่า scope ของตัวแปรถูกกำหนดโดยตำแหน่งในซอร์สโค้ด ไม่ใช่ขณะ runtime

เราจะเรียนรู้เรื่อง local variables, global variables, upvalues, _ENV table และ การสร้าง sandbox สำหรับการรันโค้ดอย่างปลอดภัย

---

## 15.1 Lexical Scoping ใน Lua

```lua
-- ตัวอย่างที่ 1: Lexical scope พื้นฐาน
local x = 10  -- x อยู่ใน scope ของ chunk นี้

local function outer()
    local y = 20  -- y อยู่ใน scope ของ outer

    local function inner()
        local z = 30  -- z อยู่ใน scope ของ inner
        print(x, y, z)  -- inner เห็น x, y, z ทั้งหมด
    end

    inner()
    -- print(z)  -- ERROR: z ไม่อยู่ใน scope ของ outer
end

outer()  -- 10  20  30
-- print(y)  -- ERROR: y ไม่อยู่ใน scope ของ chunk
```

```lua
-- ตัวอย่างที่ 2: scope กำหนดโดย "lexical position" ไม่ใช่ call stack
local a = "outer a"

local function show_a()
    print(a)  -- เห็น a ของ scope ที่ฟังก์ชันถูกนิยาม (lexical scope)
end

local function wrapper()
    local a = "inner a"  -- a ใหม่ใน scope ของ wrapper
    show_a()  -- ยังคงเห็น "outer a" เพราะ show_a ถูกนิยามใน outer scope
end

wrapper()  -- outer a  (ไม่ใช่ "inner a")
```

---

## 15.2 Block Scope ด้วย do...end

```lua
-- ตัวอย่างที่ 3: สร้าง block scope ด้วย do...end
do
    local temp = "ฉันอยู่แค่ใน block นี้"
    print(temp)  -- ฉันอยู่แค่ใน block นี้
end

-- print(temp)  -- ERROR: temp ไม่อยู่ใน scope แล้ว
```

```lua
-- ตัวอย่างที่ 4: ใช้ do...end เพื่อจำกัด scope ของตัวแปรชั่วคราว
local result

do
    local raw_data = {3, 1, 4, 1, 5, 9, 2, 6}
    local sum = 0
    for _, v in ipairs(raw_data) do
        sum = sum + v
    end
    result = sum / #raw_data  -- เฉลี่ย
end
-- raw_data และ sum ถูก garbage collected แล้ว
print("average:", result)  -- average: 3.875
```

```lua
-- ตัวอย่างที่ 5: do...end สำหรับ initialization ที่ซับซ้อน
local config
do
    local base = "/usr/local"
    local app = "myapp"
    local version = "1.0"
    config = {
        base_dir = base,
        app_dir = base .. "/" .. app,
        log_dir = base .. "/" .. app .. "/logs",
        version = version,
        full_name = app .. "-" .. version
    }
end
print(config.full_name)   -- myapp-1.0
print(config.log_dir)     -- /usr/local/myapp/logs
```

```lua
-- ตัวอย่างที่ 6: nested do blocks
do
    local x = "outer block"
    print(x)  -- outer block
    do
        local x = "inner block"
        print(x)  -- inner block  (shadow)
    end
    print(x)  -- outer block  (กลับมาเห็น x ของ outer)
end
```

---

## 15.3 Local vs Global Variables

### Global Variables

```lua
-- ตัวอย่างที่ 7: global variable (ไม่มี local keyword)
function set_global()
    g_value = 42  -- global! (ระวัง)
end

set_global()
print(g_value)  -- 42  (accessible จากทุกที่)

-- global อยู่ใน _G table
print(_G.g_value)  -- 42
```

```lua
-- ตัวอย่างที่ 8: local variable
function set_local()
    local l_value = 42  -- local! เห็นแค่ใน function นี้
end

set_local()
print(l_value)  -- nil  (ไม่มีใน global scope)
```

```lua
-- ตัวอย่างที่ 9: ความแตกต่างที่สำคัญ
-- WRONG: ลืม local
function bad_counter()
    count = count and count + 1 or 1  -- count เป็น global!
end

-- RIGHT: ใช้ local
local good_count = 0
function good_counter()
    good_count = good_count + 1  -- upvalue
end

bad_counter()
bad_counter()
bad_counter()
print(count)       -- 3  (global ที่ไม่ได้ตั้งใจ)

good_counter()
good_counter()
print(good_count)  -- 2
```

---

## 15.4 Variable Shadowing

```lua
-- ตัวอย่างที่ 10: shadowing พื้นฐาน
local x = "global x"

local function demo()
    print(x)      -- global x (ก่อน shadow)
    local x = "local x"  -- shadow!
    print(x)      -- local x
    do
        local x = "inner x"  -- shadow อีกครั้ง
        print(x)  -- inner x
    end
    print(x)      -- local x (กลับมา)
end

demo()
print(x)  -- global x
```

```lua
-- ตัวอย่างที่ 11: shadowing ใน for loop
local i = 100  -- i ใน outer scope

for i = 1, 3 do  -- i ใน for loop เป็นคนละตัว
    print("loop i:", i)
end

print("outer i:", i)  -- outer i: 100  (ไม่เปลี่ยน)
```

```lua
-- ตัวอย่างที่ 12: shadowing อาจทำให้ code สับสน
local value = 10

local function confusing()
    print(value)       -- 10 (ยังไม่ถูก shadow)
    if true then
        local value = 20   -- shadow
        print(value)   -- 20
    end
    print(value)       -- 10 (หลัง block)
end

confusing()
```

```lua
-- ตัวอย่างที่ 13: ใช้ shadowing อย่างตั้งใจเพื่อ refine
local print = print  -- shadow global print ด้วย local (เร็วกว่า)

local function inner_work()
    print("fast access to print")
end

-- ตัวอย่างจริง: localize functions ที่ใช้บ่อยๆ
local floor = math.floor
local sqrt = math.sqrt
local format = string.format

local function process(n)
    return format("floor(sqrt(%d)) = %d", n, floor(sqrt(n)))
end

print(process(100))  -- floor(sqrt(100)) = 10
```

---

## 15.5 Upvalues

Upvalue คือ local variable ที่ถูก "capture" โดย closure (ฟังก์ชันที่ซ้อนอยู่ภายใน)

```lua
-- ตัวอย่างที่ 14: upvalue พื้นฐาน
local function make_counter()
    local count = 0  -- นี่คือ upvalue สำหรับ increment

    local function increment()
        count = count + 1  -- count เป็น upvalue
        return count
    end

    return increment
end

local counter = make_counter()
print(counter())  -- 1
print(counter())  -- 2
print(counter())  -- 3
```

```lua
-- ตัวอย่างที่ 15: หลาย closures share upvalue เดียวกัน
local function make_pair()
    local value = 0

    local function get() return value end
    local function set(v) value = v end

    return get, set
end

local get, set = make_pair()
print(get())   -- 0
set(42)
print(get())   -- 42  (ทั้ง get และ set share value เดียวกัน)
```

```lua
-- ตัวอย่างที่ 16: upvalue ถูก share ผ่าน closures
local function make_stack()
    local items = {}

    return {
        push = function(v) items[#items+1] = v end,
        pop = function()
            local v = items[#items]
            items[#items] = nil
            return v
        end,
        peek = function() return items[#items] end,
        size = function() return #items end,
        is_empty = function() return #items == 0 end
    }
end

local stack = make_stack()
stack.push(1)
stack.push(2)
stack.push(3)
print(stack.size())  -- 3
print(stack.pop())   -- 3
print(stack.peek())  -- 2
print(stack.size())  -- 2
```

```lua
-- ตัวอย่างที่ 17: upvalue มีชีวิตนานกว่า scope ที่สร้างมัน
local function outer()
    local x = 10
    return function()
        x = x + 1
        return x
    end
end

local fn = outer()  -- outer() return แล้ว แต่ x ยังมีชีวิต
print(fn())  -- 11
print(fn())  -- 12
print(fn())  -- 13
-- x ยังคงอยู่เพราะ closure ยังอ้างถึงมัน
```

---

## 15.6 The _ENV Table

ใน Lua 5.2+ global variables ทั้งหมดเก็บใน table ชื่อ `_ENV`

```lua
-- ตัวอย่างที่ 18: _ENV คือที่เก็บ globals
x = 42  -- เหมือนกับ _ENV.x = 42
print(x)       -- 42
print(_ENV.x)  -- 42

-- อ่านจาก global
y = 100
print(_ENV.y)  -- 100

-- เขียนผ่าน _ENV
_ENV.z = 999
print(z)       -- 999
```

```lua
-- ตัวอย่างที่ 19: ตรวจสอบ globals ที่มีอยู่
-- _G เป็น reference ไปยัง global environment
for name, value in pairs(_G) do
    if type(value) ~= "function" and type(value) ~= "table" then
        print(name, "=", value)
    end
end
```

```lua
-- ตัวอย่างที่ 20: _ENV เป็น local variable!
-- ทุก chunk ได้รับ _ENV เป็น upvalue
print(type(_ENV))  -- table

-- เราสามารถสร้าง local _ENV เพื่อเปลี่ยน environment ของ code block
do
    local _ENV = {print = print, math = math}  -- สร้าง environment จำกัด
    -- ในนี้ globals อื่นๆ ไม่สามารถเข้าถึงได้
    print(math.pi)  -- 3.1415926535898
    -- io.write("test")  -- ERROR: io ไม่มีใน _ENV ใหม่
end
```

---

## 15.7 Performance: Local vs Global

```lua
-- ตัวอย่างที่ 21: วัดความเร็ว local vs global
local iterations = 10000000

-- Test global
local t1 = os.clock()
for i = 1, iterations do
    math.sin(1.0)  -- เข้าถึง math (global) แล้วเรียก sin
end
local global_time = os.clock() - t1

-- Test local
local sin = math.sin  -- localize
local t2 = os.clock()
for i = 1, iterations do
    sin(1.0)  -- เข้าถึง sin (local)
end
local local_time = os.clock() - t2

print(string.format("global: %.3f sec", global_time))
print(string.format("local:  %.3f sec", local_time))
print(string.format("speedup: %.1fx", global_time / local_time))
```

```lua
-- ตัวอย่างที่ 22: best practice - localize ที่ top ของ module
-- ทำให้ทั้งเร็วกว่าและชัดเจนกว่า
local abs    = math.abs
local ceil   = math.ceil
local floor  = math.floor
local max    = math.max
local min    = math.min
local sqrt   = math.sqrt

local concat = table.concat
local insert = table.insert
local remove = table.remove
local sort   = table.sort

local format = string.format
local find   = string.find
local sub    = string.sub

-- ใช้งาน
local function normalize(x, lo, hi)
    return (x - lo) / (hi - lo)
end

print(format("%.4f", normalize(75, 0, 100)))  -- 0.7500
```

---

## 15.8 Module Pattern ด้วย Locals

```lua
-- ตัวอย่างที่ 23: module pattern พื้นฐาน
local M = {}  -- module table (public interface)

-- private state (ไม่ expose ออกไป)
local _count = 0
local _data = {}

-- private functions
local function _validate(v)
    return type(v) == "number" and v >= 0
end

-- public functions (เพิ่มใน M)
function M.add(v)
    if not _validate(v) then
        return false, "invalid value: " .. tostring(v)
    end
    _count = _count + 1
    _data[_count] = v
    return true
end

function M.sum()
    local total = 0
    for _, v in ipairs(_data) do
        total = total + v
    end
    return total
end

function M.count() return _count end

function M.reset()
    _count = 0
    _data = {}
end

-- ใช้งาน
M.add(10)
M.add(20)
M.add(30)
print(M.sum())    -- 60
print(M.count())  -- 3
-- print(_count)  -- ERROR หรือ nil (private)
```

```lua
-- ตัวอย่างที่ 24: module ที่มี private helper functions
local StringUtils = {}

-- private
local function _trim_left(s)
    return s:match("^%s*(.*)")
end

local function _trim_right(s)
    return s:match("(.-)%s*$")
end

-- public
function StringUtils.trim(s)
    return _trim_right(_trim_left(s))
end

function StringUtils.split(s, sep)
    local parts = {}
    local pattern = "([^" .. sep .. "]*)" .. sep .. "?"
    for part in s:gmatch(pattern) do
        if part ~= "" then
            parts[#parts+1] = part
        end
    end
    return parts
end

function StringUtils.title_case(s)
    return s:gsub("(%a)([%w]*)", function(first, rest)
        return first:upper() .. rest:lower()
    end)
end

print(StringUtils.trim("  hello world  "))  -- hello world
local parts = StringUtils.split("a,b,c,d", ",")
print(table.concat(parts, "|"))             -- a|b|c|d
print(StringUtils.title_case("hello world"))  -- Hello World
```

---

## 15.9 Accessing Globals via _G

```lua
-- ตัวอย่างที่ 25: dynamic global access
local function get_global(name)
    return _G[name]
end

local function set_global(name, value)
    _G[name] = value
end

set_global("dynamic_var", 42)
print(get_global("dynamic_var"))  -- 42
print(dynamic_var)               -- 42  (เข้าถึงได้เหมือน global ปกติ)
```

```lua
-- ตัวอย่างที่ 26: ตรวจสอบว่า global มีอยู่หรือไม่
local function global_exists(name)
    return _G[name] ~= nil
end

print(global_exists("print"))   -- true
print(global_exists("math"))    -- true
print(global_exists("xyz123"))  -- false
```

```lua
-- ตัวอย่างที่ 27: list global functions
local function list_global_functions()
    local funcs = {}
    for name, value in pairs(_G) do
        if type(value) == "function" then
            funcs[#funcs+1] = name
        end
    end
    table.sort(funcs)
    return funcs
end

local global_funcs = list_global_functions()
print("Global functions:")
for _, name in ipairs(global_funcs) do
    io.write(name .. " ")
end
print()
```

```lua
-- ตัวอย่างที่ 28: ตรวจจับการเข้าถึง undefined globals (debugging)
local original_env = _ENV

local strict_env = setmetatable({}, {
    __index = function(t, k)
        local v = original_env[k]
        if v == nil then
            error("accessing undefined global: " .. tostring(k), 2)
        end
        return v
    end,
    __newindex = function(t, k, v)
        error("setting global variable: " .. tostring(k), 2)
    end
})

-- ใช้ใน production เพื่อ catch bugs
-- _ENV = strict_env  -- uncomment เพื่อเปิดใช้งาน
```

---

## 15.10 load() และ loadstring() กับ Environment

```lua
-- ตัวอย่างที่ 29: load() พื้นฐาน
local code = "return 1 + 2"
local fn = load(code)
print(fn())  -- 3
```

```lua
-- ตัวอย่างที่ 30: load() กับ custom environment
local sandbox_env = {
    print = print,
    math = {
        sin = math.sin,
        cos = math.cos,
        pi = math.pi
    },
    -- ไม่มี io, os, load, etc.
}
sandbox_env._ENV = sandbox_env  -- ให้ sandbox เข้าถึง _ENV ของตัวเองได้

local code = [[
    local x = math.sin(math.pi / 2)
    print("sin(pi/2) =", x)
    return x
]]

local fn = load(code, "sandbox", "t", sandbox_env)
if fn then
    local result = fn()
    print("returned:", result)
else
    print("compile error")
end
```

```lua
-- ตัวอย่างที่ 31: sandboxed execution ที่ปลอดภัยกว่า
function safe_eval(code, allowed_globals)
    -- สร้าง environment ที่จำกัด
    local env = allowed_globals or {}
    env._ENV = env

    -- compile
    local fn, err = load(code, "safe_code", "t", env)
    if not fn then
        return nil, "compile error: " .. err
    end

    -- run ใน pcall
    local ok, result = pcall(fn)
    if not ok then
        return nil, "runtime error: " .. tostring(result)
    end

    return result
end

-- ให้แค่ math functions
local allowed = {
    math = math,
    print = print,
    tostring = tostring,
    tonumber = tonumber
}

local result, err = safe_eval("return math.sqrt(144)", allowed)
print(result)  -- 12.0

local result2, err2 = safe_eval("io.read()", allowed)
print(result2, err2)  -- nil  runtime error: ...
```

---

## 15.11 Variable Lifetime

```lua
-- ตัวอย่างที่ 32: lifetime ของ local variable
do
    local x = {}  -- x ถูก allocate
    for i = 1, 1000 do
        x[i] = i * i
    end
    print("sum:", (function()
        local s = 0
        for _, v in ipairs(x) do s = s + v end
        return s
    end)())
end
-- x ออก scope และจะถูก GC
-- collectgarbage() จะคืน memory

print("memory before GC:", collectgarbage("count"), "KB")
collectgarbage("collect")
print("memory after GC:", collectgarbage("count"), "KB")
```

```lua
-- ตัวอย่างที่ 33: closure ยืด lifetime ของ upvalue
local function create_resource()
    local resource = {
        data = string.rep("x", 1000),  -- ข้อมูลขนาดใหญ่
        close = false
    }

    local function use()
        if resource.close then
            error("resource already closed")
        end
        return #resource.data
    end

    local function close()
        resource.close = true
        resource.data = nil  -- ปล่อย memory
    end

    return use, close
end

local use, close = create_resource()
print(use())   -- 1000  (resource ยังมีชีวิต)
close()
-- print(use())  -- ERROR: resource already closed
```

---

## 15.12 Loop Variable Scope

```lua
-- ตัวอย่างที่ 34: loop variable scope ใน numeric for
for i = 1, 5 do
    -- i เป็น local ของ loop body
    print(i)
end
-- print(i)  -- nil หรือ error  (i ออก scope แล้ว)
```

```lua
-- ตัวอย่างที่ 35: loop variable scope ใน generic for
local fruits = {"apple", "banana", "cherry"}
for idx, fruit in ipairs(fruits) do
    print(idx, fruit)
end
-- print(idx, fruit)  -- nil  (ออก scope แล้ว)
```

```lua
-- ตัวอย่างที่ 36: while loop ไม่สร้าง scope ใหม่
local j = 0
while j < 3 do
    local temp = j * j  -- temp อยู่ใน scope ของแต่ละ iteration
    j = j + 1
    print(j, temp)
end
-- print(temp)  -- nil  (ออก scope แล้ว)
```

---

## 15.13 Pitfall: Closures ใน Loops

นี่เป็น bug ที่พบบ่อยมากใน Lua!

```lua
-- ตัวอย่างที่ 37: BUG คลาสสิก - closures share upvalue
local funcs = {}
for i = 1, 5 do
    funcs[i] = function() return i end  -- ทุก closure share i เดียวกัน!
end

-- ใน numeric for, i เป็น loop variable ที่ Lua จัดการพิเศษ
-- แต่มาดู generic for หรือ while loop ก่อน
```

```lua
-- ตัวอย่างที่ 38: BUG ใน while loop
local funcs2 = {}
local k = 1
while k <= 5 do
    local captured_wrong = k  -- ดูเหมือนถูก แต่ต้องทำในแต่ละ iteration
    funcs2[k] = function() return k end  -- k เป็น upvalue เดียวกัน!
    k = k + 1
end

print("--- wrong version (k shared) ---")
for i = 1, 5 do
    print(funcs2[i]())  -- พิมพ์ 6 ทุกตัว! (k = 6 ตอน loop จบ)
end
```

```lua
-- ตัวอย่างที่ 39: FIX - ใช้ local copy
local funcs3 = {}
local m = 1
while m <= 5 do
    local local_m = m  -- สร้าง local copy ใน iteration นี้
    funcs3[m] = function() return local_m end  -- capture local_m
    m = m + 1
end

print("--- correct version (local copy) ---")
for i = 1, 5 do
    print(funcs3[i]())  -- 1  2  3  4  5  (ถูกต้อง!)
end
```

```lua
-- ตัวอย่างที่ 40: numeric for ไม่มีปัญหานี้ (Lua ทำให้ถูกอัตโนมัติ)
local funcs4 = {}
for i = 1, 5 do
    funcs4[i] = function() return i end
    -- Lua สร้าง i ใหม่ทุก iteration สำหรับ numeric for
end

print("--- numeric for (safe) ---")
for i = 1, 5 do
    print(funcs4[i]())  -- 1  2  3  4  5  (ถูกต้อง!)
end
```

```lua
-- ตัวอย่างที่ 41: generic for ก็ปลอดภัย
local funcs5 = {}
local items = {"a", "b", "c", "d", "e"}
for idx, val in ipairs(items) do
    funcs5[idx] = function() return val end
    -- val เป็น local ใหม่ทุก iteration
end

print("--- generic for (safe) ---")
for i = 1, 5 do
    print(funcs5[i]())  -- a  b  c  d  e
end
```

---

## 15.14 Why Always Use Local

```lua
-- ตัวอย่างที่ 42: เหตุผลที่ 1 - Performance
-- global lookup: ค้น _ENV ทุกครั้ง O(1) แต่ overhead มากกว่า
-- local lookup: direct register/stack access, เร็วที่สุด

local function perf_test()
    local n = 10000000
    local t1, t2

    -- Global
    t1 = os.clock()
    for i = 1, n do
        local x = math.pi  -- global lookup
    end
    t2 = os.clock()
    print(string.format("global math.pi: %.3f sec", t2 - t1))

    -- Local
    local pi = math.pi  -- localize ครั้งเดียว
    t1 = os.clock()
    for i = 1, n do
        local x = pi     -- local lookup
    end
    t2 = os.clock()
    print(string.format("local pi:       %.3f sec", t2 - t1))
end

perf_test()
```

```lua
-- ตัวอย่างที่ 43: เหตุผลที่ 2 - No side effects
-- Global variables สร้าง side effects ที่ไม่ตั้งใจ

-- ตัวอย่าง BUG จาก global:
function buggy_sum(t)
    total = 0  -- global! ทับค่า global อื่น
    for _, v in ipairs(t) do
        total = total + v
    end
    return total
end

total = 999  -- global ที่ใช้งานอยู่

buggy_sum({1, 2, 3})
print(total)  -- 6  (ถูกทับ!)

-- FIX:
function safe_sum(t)
    local total = 0  -- local, ปลอดภัย
    for _, v in ipairs(t) do
        total = total + v
    end
    return total
end

total = 999
safe_sum({1, 2, 3})
print(total)  -- 999  (ไม่ถูกแก้)
```

```lua
-- ตัวอย่างที่ 44: เหตุผลที่ 3 - Garbage Collection
-- Globals อยู่ใน _G ตลอดชีวิตของโปรแกรม
-- Locals ถูก GC เมื่อออก scope

local function memory_efficient()
    local large_data = {}
    for i = 1, 10000 do
        large_data[i] = string.rep("x", 100)
    end
    -- ประมวลผล large_data
    local sum = 0
    for _, s in ipairs(large_data) do
        sum = sum + #s
    end
    return sum
    -- large_data ถูก GC หลัง function return
end

print(memory_efficient())  -- 1000000
-- ถ้าใช้ global, large_data จะอยู่ตลอดไป
```

```lua
-- ตัวอย่างที่ 45: เหตุผลที่ 4 - Name Collision
-- module A
do
    local count = 0  -- ไม่ขัดกับ module อื่น
    function module_a_inc() count = count + 1 end
    function module_a_get() return count end
end

-- module B  
do
    local count = 0  -- count แยกจาก module A
    function module_b_inc() count = count + 1 end
    function module_b_get() return count end
end

module_a_inc()
module_a_inc()
module_b_inc()

print("A:", module_a_get())  -- A: 2
print("B:", module_b_get())  -- B: 1
```

---

## 15.15 Advanced: Upvalue Manipulation ด้วย debug Library

```lua
-- ตัวอย่างที่ 46: ดู upvalues ด้วย debug library
local function make_adder(n)
    return function(x)
        return x + n  -- n คือ upvalue
    end
end

local add5 = make_adder(5)
print(add5(10))  -- 15

-- ดู upvalue
local i = 1
while true do
    local name, value = debug.getupvalue(add5, i)
    if not name then break end
    print(string.format("upvalue[%d]: %s = %s", i, name, tostring(value)))
    i = i + 1
end
-- upvalue[1]: n = 5
```

```lua
-- ตัวอย่างที่ 47: แก้ไข upvalue ด้วย debug.setupvalue
local function make_counter_obj()
    local n = 0
    return {
        inc = function() n = n + 1 end,
        get = function() return n end
    }
end

local c = make_counter_obj()
c.inc()
c.inc()
print(c.get())  -- 2

-- เข้าถึงและแก้ไข upvalue โดยตรง (ระวัง: เป็น advanced technique)
debug.setupvalue(c.get, 1, 100)  -- reset n = 100
print(c.get())  -- 100
```

```lua
-- ตัวอย่างที่ 48: share upvalue ระหว่าง closures
-- ใช้ debug.upvalueid และ debug.upvaluejoin
local function make_shared()
    local shared = 0

    local function reader() return shared end
    local function writer(v) shared = v end

    return reader, writer
end

local r1, w1 = make_shared()
local r2, w2 = make_shared()

-- ตอนนี้ r1/w1 share กัน และ r2/w2 share กัน แต่สองคู่นี้แยกกัน
w1(42)
print(r1())  -- 42
print(r2())  -- 0  (แยก shared value)

-- ทำให้ r2 อ่าน shared value ของ r1/w1
debug.upvaluejoin(r2, 1, r1, 1)  -- join upvalue ที่ 1 ของ r2 เข้ากับ r1
print(r2())  -- 42  (ตอนนี้เห็น shared ของ r1)
```

---

## 15.16 Sandboxing ขั้นสูง

```lua
-- ตัวอย่างที่ 49: sandbox ที่ปลอดภัยสำหรับ user scripts
local function create_sandbox(allowed_modules)
    allowed_modules = allowed_modules or {}

    -- สร้าง safe environment
    local env = {
        -- ฟังก์ชัน safe ที่อนุญาต
        print = print,
        tostring = tostring,
        tonumber = tonumber,
        type = type,
        pairs = pairs,
        ipairs = ipairs,
        next = next,
        select = select,
        unpack = table.unpack,
        pcall = pcall,
        xpcall = xpcall,
        error = error,
        assert = assert,

        -- ข้อมูล safe
        math = {
            pi = math.pi, e = math.exp(1),
            sin = math.sin, cos = math.cos, tan = math.tan,
            floor = math.floor, ceil = math.ceil,
            sqrt = math.sqrt, abs = math.abs,
            max = math.max, min = math.min,
            random = math.random
        },

        string = {
            format = string.format,
            len = string.len,
            sub = string.sub,
            upper = string.upper,
            lower = string.lower,
            rep = string.rep,
            find = string.find,
            match = string.match,
            gmatch = string.gmatch,
            gsub = string.gsub
        },

        table = {
            concat = table.concat,
            insert = table.insert,
            remove = table.remove,
            sort = table.sort,
            unpack = table.unpack,
            pack = table.pack
        }
    }

    -- เพิ่ม modules ที่อนุญาตพิเศษ
    for _, mod in ipairs(allowed_modules) do
        env[mod] = _G[mod]
    end

    env._ENV = env

    return function(code)
        local fn, err = load(code, "sandbox", "t", env)
        if not fn then
            return nil, "compile error: " .. err
        end

        local ok, result = pcall(fn)
        if not ok then
            return nil, "runtime error: " .. tostring(result)
        end

        return result
    end
end

-- สร้าง sandbox
local run = create_sandbox()

-- รัน code อย่างปลอดภัย
local r1 = run("return math.sqrt(16) + math.pi")
print("result:", r1)  -- result: 7.1415926535898

-- พยายามเข้าถึง io (ไม่ได้รับอนุญาต)
local r2, err = run("return io.read()")
print("error:", err)  -- error: runtime error: ...

-- รัน code ปกติ
local r3 = run([[
    local total = 0
    for i = 1, 100 do
        total = total + i
    end
    return total
]])
print("sum 1..100:", r3)  -- sum 1..100: 5050
```

```lua
-- ตัวอย่างที่ 50: resource-limited sandbox
local function create_limited_sandbox(max_instructions)
    local instruction_count = 0

    local function count_hook()
        instruction_count = instruction_count + 1
        if instruction_count > max_instructions then
            error("instruction limit exceeded", 2)
        end
    end

    return function(code)
        instruction_count = 0

        local env = {
            print = print,
            math = math,
            string = string,
            table = table,
            tostring = tostring,
            tonumber = tonumber,
            type = type,
            ipairs = ipairs,
            pairs = pairs,
            select = select
        }
        env._ENV = env

        local fn, err = load(code, "limited", "t", env)
        if not fn then return nil, err end

        -- Set debug hook เพื่อนับ instructions
        debug.sethook(count_hook, "", 1)
        local ok, result = pcall(fn)
        debug.sethook()  -- remove hook

        if not ok then
            return nil, tostring(result)
        end
        return result, instruction_count
    end
end

local run_limited = create_limited_sandbox(10000)

-- Code ปกติ
local r, count = run_limited("local x = 0; for i=1,100 do x=x+i end; return x")
print(string.format("result=%s, instructions=%s", tostring(r), tostring(count)))

-- Code ที่ใช้ instructions เกินกำหนด
local r2, err2 = run_limited("while true do end")
print("error:", err2)
```

---

## แบบฝึกหัด (Exercises)

### ระดับพื้นฐาน

**แบบฝึกหัดที่ 1:** เขียน module `Counter` ที่มี state private และ methods: `increment()`, `decrement()`, `reset()`, `get()`

```lua
-- Template:
local Counter = {}
-- TODO: implement Counter module
-- Counter.new() -> สร้าง counter ใหม่
-- :increment(n) -> เพิ่มด้วย n (default 1)
-- :decrement(n) -> ลดด้วย n (default 1)
-- :reset() -> reset เป็น 0
-- :get() -> คืนค่าปัจจุบัน

local c = Counter.new()
c:increment()
c:increment()
c:increment(5)
c:decrement(2)
print(c:get())  -- ควรได้ 5
```

**แบบฝึกหัดที่ 2:** แก้ bug closure-in-loop ต่อไปนี้:

```lua
-- BUGGY CODE:
local handlers = {}
for i = 1, 5 do
    handlers[i] = function()
        print("handler", i)
    end
end

-- ตอนนี้ทุกตัวพิมพ์ "handler 5" แต่เราต้องการ 1-5
-- TODO: แก้ไขให้ถูกต้อง
```

**แบบฝึกหัดที่ 3:** เขียนฟังก์ชัน `make_logger(prefix)` ที่คืน logger function

```lua
-- Template:
function make_logger(prefix)
    -- TODO: คืน function ที่พิมพ์ prefix + message
end

local info = make_logger("[INFO]")
local warn = make_logger("[WARN]")
info("Server started")   -- [INFO] Server started
warn("Low memory")       -- [WARN] Low memory
```

### ระดับกลาง

**แบบฝึกหัดที่ 4:** สร้าง `Namespace` system

```lua
-- Template:
local Namespace = {}
-- Namespace.create(name) -> สร้าง namespace
-- Namespace.get(name) -> ดึง namespace

-- ใช้งาน:
local ns = Namespace.create("myapp")
ns.version = "1.0"
ns.debug = true

print(Namespace.get("myapp").version)  -- 1.0
```

**แบบฝึกหัดที่ 5:** เขียน `with_scope(vars, fn)` ที่ตั้งค่า vars ชั่วคราวแล้วเรียก fn

```lua
-- Template:
function with_scope(vars, fn)
    -- TODO: ตั้ง globals ชั่วคราว, เรียก fn, แล้ว restore
end

-- ใช้งาน:
x = 0
with_scope({x = 100}, function()
    print(x)  -- 100  (ชั่วคราว)
end)
print(x)  -- 0  (กลับมาแล้ว)
```

**แบบฝึกหัดที่ 6:** เขียน restricted environment สำหรับ math-only scripts

### ระดับยาก

**แบบฝึกหัดที่ 7:** เขียน `require` จำลองที่ cache modules

```lua
-- Template:
local loaded = {}
function my_require(name, loader)
    if not loaded[name] then
        loaded[name] = loader()
    end
    return loaded[name]
end
```

**แบบฝึกหัดที่ 8:** เขียน `strict` mode ที่ error เมื่อ access undefined globals

**แบบฝึกหัดที่ 9:** เขียน Proxy ที่ track การเข้าถึง globals และนับความถี่

**แบบฝึกหัดที่ 10:** สร้าง scoped variable system คล้าย dynamic binding ของ Common Lisp

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Lexical Scoping** - scope ถูกกำหนดโดยตำแหน่งในโค้ด ไม่ใช่ call stack
2. **Block Scope** - ใช้ `do...end` สร้าง scope พิเศษ
3. **Local vs Global** - local เร็วกว่า, ปลอดภัยกว่า, GC ได้
4. **Variable Shadowing** - local ในขอบเขตใกล้กว่าซ่อน variable ที่อยู่ไกลกว่า
5. **Upvalues** - local variables ที่ถูก capture โดย closures
6. **_ENV** - table ที่เก็บ global environment, สามารถแก้ไขได้
7. **Performance** - local access เร็วกว่า global อย่างมีนัยสำคัญ
8. **Module Pattern** - ใช้ local variables เพื่อ private state
9. **Loop Closure Bug** - closures ใน while loop share upvalue ต้องใช้ local copy
10. **Sandboxing** - ใช้ `load()` กับ custom `_ENV` เพื่อรัน code อย่างปลอดภัย

การเข้าใจ scope และ upvalues อย่างลึกซึ้งจะช่วยให้คุณเขียนโค้ดที่ถูกต้อง มีประสิทธิภาพ และปลอดภัยมากขึ้น
