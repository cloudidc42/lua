# บทที่ 2: ตัวแปรและชนิดข้อมูล

## สารบัญ

1. [ชนิดข้อมูล 8 ชนิดใน Lua](#ชนิดข้อมูล-8-ชนิดใน-lua)
2. [ฟังก์ชัน type()](#ฟังก์ชัน-type)
3. [กฎการตั้งชื่อตัวแปร](#กฎการตั้งชื่อตัวแปร)
4. [Global vs Local Variables](#global-vs-local-variables)
5. [ชนิด nil](#ชนิด-nil)
6. [ชนิด boolean](#ชนิด-boolean)
7. [ชนิด number](#ชนิด-number)
8. [ชนิด string](#ชนิด-string)
9. [ชนิด table](#ชนิด-table)
10. [ชนิด function](#ชนิด-function)
11. [ชนิด userdata](#ชนิด-userdata)
12. [ชนิด thread (Coroutine)](#ชนิด-thread-coroutine)
13. [Type Coercion อัตโนมัติ](#type-coercion-อัตโนมัติ)
14. [การแปลงชนิดอย่างชัดเจน](#การแปลงชนิดอย่างชัดเจน)
15. [Pattern สำหรับ Constants](#pattern-สำหรับ-constants)
16. [_G Global Table](#_g-global-table)
17. [แบบฝึกหัด](#แบบฝึกหัด)

---

## ชนิดข้อมูล 8 ชนิดใน Lua

Lua มีชนิดข้อมูลเพียง **8 ชนิด** เท่านั้น ซึ่งน้อยมากเมื่อเทียบกับภาษาอื่น:

| ชนิด       | คำอธิบาย                              | ตัวอย่าง                        |
|-----------|---------------------------------------|--------------------------------|
| `nil`      | ไม่มีค่า, ไม่มีตัวตน                  | `nil`                          |
| `boolean`  | ค่าตรรกศาสตร์                          | `true`, `false`                |
| `number`   | ตัวเลข (integer และ float)            | `42`, `3.14`, `0xFF`           |
| `string`   | ข้อความ                               | `"hello"`, `'world'`           |
| `table`    | โครงสร้างข้อมูล (array, dict, object) | `{1, 2, 3}`, `{name="Lua"}`    |
| `function` | ฟังก์ชัน/ขั้นตอนการทำงาน             | `function() end`               |
| `userdata` | C data pointer (ใช้จาก C API)        | `io.stdin`, `io.stdout`        |
| `thread`   | coroutine thread                      | `coroutine.create(f)`          |

```lua
-- ตัวอย่างที่ 1: แสดงชนิดข้อมูลทั้ง 8
print("=== ชนิดข้อมูลทั้ง 8 ใน Lua ===")
print(type(nil))            -- nil
print(type(true))           -- boolean
print(type(false))          -- boolean
print(type(42))             -- number
print(type(3.14))           -- number
print(type("hello"))        -- string
print(type({}))             -- table
print(type(print))          -- function
print(type(io.stdin))       -- userdata
print(type(coroutine.create(function() end)))  -- thread
```

---

## ฟังก์ชัน type()

### การใช้งาน type() พื้นฐาน

```lua
-- ตัวอย่างที่ 2: type() function
-- type() รับค่าใดๆ และคืน string ที่บอกชนิด

local values = {
    nil,          -- ต้องระวัง: nil ใน table จะ terminate array
    true,
    42,
    3.14,
    "hello",
    {},
    print,
}

-- วิธีที่ถูกต้องในการใช้ type() กับ nil
print(type(nil))     -- nil
print(type(true))    -- boolean
print(type(42))      -- number
print(type(3.14))    -- number
print(type("hello")) -- string
print(type({}))      -- table
print(type(print))   -- function
```

### ใช้ type() ในการตรวจสอบ

```lua
-- ตัวอย่างที่ 3: ตรวจสอบชนิดก่อนใช้งาน
local function processValue(value)
    local t = type(value)
    
    if t == "number" then
        print("ตัวเลข: " .. value)
        print("กำลังสอง: " .. value^2)
    elseif t == "string" then
        print("ข้อความ: " .. value)
        print("ความยาว: " .. #value)
    elseif t == "boolean" then
        print("บูลีน: " .. tostring(value))
    elseif t == "nil" then
        print("ค่าว่าง (nil)")
    elseif t == "table" then
        print("table ที่มี " .. #value .. " elements")
    elseif t == "function" then
        print("ฟังก์ชัน")
        value()  -- เรียกใช้งาน
    else
        print("ชนิด: " .. t)
    end
end

processValue(42)
processValue("สวัสดี")
processValue(true)
processValue(nil)
processValue({1, 2, 3})
processValue(function() print("ถูกเรียก!") end)
```

### type() กับตัวแปรที่เปลี่ยนชนิด

```lua
-- ตัวอย่างที่ 4: ตัวแปร dynamic typing
local x = 10
print(type(x))   -- number

x = "hello"
print(type(x))   -- string

x = true
print(type(x))   -- boolean

x = {}
print(type(x))   -- table

x = nil
print(type(x))   -- nil

-- type() เป็น function ที่ตรวจสอบ ณ ขณะที่เรียก (runtime)
print("type() คืนค่าเป็น: " .. type(type(x)))  -- string
```

---

## กฎการตั้งชื่อตัวแปร

### กฎพื้นฐาน

```lua
-- ตัวอย่างที่ 5: กฎการตั้งชื่อตัวแปร

-- ถูกต้อง: เริ่มด้วยตัวอักษรหรือ underscore
local myVar = 1
local _myVar = 2
local MY_CONSTANT = 3
local camelCaseVar = 4
local snake_case_var = 5
local PascalCaseClass = 6
local _1 = 7          -- underscore แล้วตามด้วยตัวเลขได้
local abc123 = 8      -- ตัวอักษรแล้วตามด้วยตัวเลขได้
local _ = 9           -- underscore ตัวเดียว (ใช้สำหรับ "don't care")
local __ = 10         -- underscore สองตัว

-- ผิด: ขึ้นต้นด้วยตัวเลข
-- local 1abc = "error"    -- SyntaxError

-- ผิด: ใช้ special characters
-- local my-var = "error"  -- SyntaxError
-- local my.var = "error"  -- SyntaxError
-- local my var = "error"  -- SyntaxError

print(myVar, _myVar, MY_CONSTANT)
print(camelCaseVar, snake_case_var)
```

### Keywords ที่ใช้เป็นชื่อไม่ได้

```lua
-- ตัวอย่างที่ 6: Keywords ของ Lua
-- ห้ามใช้เป็นชื่อตัวแปร!

--[[
Keywords ทั้ง 22 คำ:
and       break     do        else      elseif
end       false     for       function  goto
if        in        local     nil       not
or        repeat    return    then      true
until     while
]]

-- ต่อไปนี้จะ error:
-- local and = 1      -- SyntaxError: 'and' is keyword
-- local if = 2       -- SyntaxError: 'if' is keyword
-- local nil = 3      -- SyntaxError: 'nil' is keyword

-- แต่ชื่อที่มี keyword เป็นส่วนหนึ่งได้:
local andValue = 1    -- ถูกต้อง
local iffy = 2        -- ถูกต้อง
local nilCheck = 3    -- ถูกต้อง

print(andValue, iffy, nilCheck)
```

### Conventions การตั้งชื่อ

```lua
-- ตัวอย่างที่ 7: Naming conventions ที่นิยมใช้

-- 1. Constants: UPPER_SNAKE_CASE
local MAX_SPEED = 300
local MIN_TEMP = -273.15
local PI = 3.14159265358979
local EARTH_GRAVITY = 9.81

-- 2. Variables: camelCase หรือ snake_case
local userName = "สมชาย"
local user_age = 25
local itemCount = 0
local is_active = true

-- 3. Functions: camelCase
local function getUserName() return userName end
local function calculate_area(r) return PI * r * r end

-- 4. Classes/Modules: PascalCase
local MyClass = {}
local HttpClient = {}

-- 5. Private (by convention): ขึ้นต้นด้วย underscore
local _privateVar = "ใช้ภายใน"
local function _internalHelper() end

-- 6. Throwaway values: _ (underscore)
local _, err = pcall(function() error("test") end)
print("Error:", err)

for _, v in ipairs({10, 20, 30}) do  -- ไม่สนใจ index
    print(v)
end
```

---

## Global vs Local Variables

### Global Variables

```lua
-- ตัวอย่างที่ 8: Global variables
-- ตัวแปร global ใช้ได้ทุกที่ แต่ช้ากว่า local

globalCounter = 0  -- global (ไม่มี local keyword)

local function increment()
    globalCounter = globalCounter + 1  -- แก้ไข global ได้
    print("Counter: " .. globalCounter)
end

increment()  -- Counter: 1
increment()  -- Counter: 2
increment()  -- Counter: 3

print("Final: " .. globalCounter)  -- Final: 3

-- Global เข้าถึงได้จากทุก scope
local function showGlobal()
    local function innerFn()
        local function deepFn()
            print("Deep inside: " .. globalCounter)
        end
        deepFn()
    end
    innerFn()
end

showGlobal()  -- Deep inside: 3
```

### Local Variables และ Scope

```lua
-- ตัวอย่างที่ 9: Local variables และ scope
local outer = "outer"

do
    local inner = "inner"
    print(outer)  -- outer (เข้าถึง outer scope ได้)
    print(inner)  -- inner
    
    do
        local deepest = "deepest"
        print(outer)    -- outer
        print(inner)    -- inner
        print(deepest)  -- deepest
    end
    
    -- print(deepest)  -- nil หรือ error (deepest อยู่นอก scope)
    print(inner)  -- inner (ยังอยู่ใน scope)
end

-- print(inner)  -- nil หรือ error (inner อยู่นอก scope)
print(outer)    -- outer (ยังอยู่ใน scope)
```

### Shadowing (การซ่อนตัวแปร)

```lua
-- ตัวอย่างที่ 10: Variable shadowing
local x = "global x"
print(x)  -- global x

do
    local x = "local x"  -- ซ่อน outer x
    print(x)  -- local x
    
    do
        local x = "deep x"  -- ซ่อน x อีกครั้ง
        print(x)  -- deep x
    end
    
    print(x)  -- local x (กลับมา)
end

print(x)  -- global x (กลับมา)
```

### ทำไม Local ดีกว่า Global?

```lua
-- ตัวอย่างที่ 11: Local เร็วกว่า Global (benchmark แบบง่าย)
-- Lua เก็บ local ใน register (เร็วกว่า)
-- Global ต้องค้นหาใน _G table (ช้ากว่า)

local start = os.clock()

-- ทดสอบ local
local sum_local = 0
local n = 1000000
local i = 0  -- local variable สำหรับ loop

for i = 1, n do
    sum_local = sum_local + i
end

local time_local = os.clock() - start
print(string.format("Local: %.4f วินาที, sum = %d", time_local, sum_local))

-- แนะนำ: ใช้ local เสมอ เว้นแต่ต้องการ global จริงๆ
-- สาเหตุ:
-- 1. เร็วกว่า (2-10x ในบางกรณี)
-- 2. ป้องกันการ conflict ของชื่อ
-- 3. Code อ่านง่ายขึ้น
-- 4. ป้องกัน memory leak
```

---

## ชนิด nil

### nil คืออะไร?

```lua
-- ตัวอย่างที่ 12: nil type
-- nil แทน "ไม่มีค่า" หรือ "ไม่มีตัวตน"

-- ตัวแปรที่ยังไม่ได้กำหนดค่า = nil
local undeclared
print(undeclared)         -- nil
print(type(undeclared))   -- nil

-- กำหนดค่า nil ให้ตัวแปร (เหมือนลบตัวแปรออก)
local x = 42
print(x)  -- 42
x = nil
print(x)  -- nil

-- nil ใน table = ลบ key ออก
local t = {a = 1, b = 2, c = 3}
print(t.a)  -- 1
t.a = nil   -- ลบ key 'a' ออกจาก table
print(t.a)  -- nil

-- ตรวจสอบ nil
if x == nil then
    print("x เป็น nil")
end

-- วิธีสั้น
if not x then
    print("x เป็น nil หรือ false")
end
```

### nil เป็น "falsy" value

```lua
-- ตัวอย่างที่ 13: nil ใน boolean context
-- ใน Lua มีแค่ nil และ false เป็น falsy
-- ทุกอย่างอื่นเป็น truthy (รวมถึง 0 และ "")

local values = {nil, false, true, 0, "", "0", {}, print}

for i, v in ipairs({false, true, 0, "", "0", {}, print}) do
    if v then
        print(tostring(v) .. " เป็น truthy")
    else
        print(tostring(v) .. " เป็น falsy")
    end
end

-- nil เป็น falsy ด้วย
local n = nil
if n then
    print("truthy")
else
    print("nil เป็น falsy")  -- จะแสดงอันนี้
end
```

### การใช้ nil เพื่อลบ elements

```lua
-- ตัวอย่างที่ 14: ใช้ nil ลบ table entries
local config = {
    host = "localhost",
    port = 8080,
    debug = true,
    secret = "my-secret-key"
}

print("ก่อนลบ:")
for k, v in pairs(config) do
    print("  " .. k .. " = " .. tostring(v))
end

-- ลบ secret key
config.secret = nil

print("\nหลังลบ secret:")
for k, v in pairs(config) do
    print("  " .. k .. " = " .. tostring(v))
end
```

---

## ชนิด boolean

### Boolean พื้นฐาน

```lua
-- ตัวอย่างที่ 15: boolean type
local isActive = true
local isDeleted = false

print(type(isActive))   -- boolean
print(type(isDeleted))  -- boolean

-- boolean ค่าที่เป็นไปได้มีแค่ 2 ค่า: true และ false
print(isActive)    -- true
print(isDeleted)   -- false
print(not isActive)   -- false
print(not isDeleted)  -- true
```

### Boolean Operations

```lua
-- ตัวอย่างที่ 16: Boolean operations
local a = true
local b = false

-- and: คืน true ถ้าทั้งคู่เป็น true
print(a and b)   -- false
print(a and a)   -- true
print(b and b)   -- false

-- or: คืน true ถ้าอย่างน้อยหนึ่งตัวเป็น true
print(a or b)    -- true
print(a or a)    -- true
print(b or b)    -- false

-- not: กลับค่า
print(not a)     -- false
print(not b)     -- true
print(not nil)   -- true
print(not 0)     -- false (0 เป็น truthy ใน Lua!)
print(not "")    -- false ("" เป็น truthy ใน Lua!)
```

### ค่า Truthy และ Falsy ใน Lua

```lua
-- ตัวอย่างที่ 17: Truthy vs Falsy
-- ใน Lua มีแค่ 2 ค่าที่เป็น falsy: nil และ false
-- ทุกอย่างอื่นเป็น truthy (รวมถึง 0 และ string ว่าง!)

local function checkTruthy(value, name)
    if value then
        print(name .. " เป็น TRUTHY")
    else
        print(name .. " เป็น FALSY")
    end
end

checkTruthy(nil, "nil")         -- FALSY
checkTruthy(false, "false")     -- FALSY
checkTruthy(true, "true")       -- TRUTHY
checkTruthy(0, "0")             -- TRUTHY (ต่างจาก C/Python!)
checkTruthy("", "\"\"")         -- TRUTHY (ต่างจาก Python!)
checkTruthy(0.0, "0.0")         -- TRUTHY
checkTruthy({}, "{}")           -- TRUTHY
checkTruthy(print, "print")     -- TRUTHY
```

---

## ชนิด number

### Integer vs Float (Lua 5.3+)

```lua
-- ตัวอย่างที่ 18: Integer และ Float ใน Lua 5.3+
-- ตั้งแต่ Lua 5.3 มีการแยก integer subtype

local i = 42          -- integer
local f = 42.0        -- float
local f2 = 3.14       -- float

print(type(i))   -- number
print(type(f))   -- number
-- type() คืน "number" ทั้งคู่ แต่มีความแตกต่าง

-- ตรวจสอบว่าเป็น integer หรือ float
print(math.type(i))   -- integer
print(math.type(f))   -- float
print(math.type(f2))  -- float

-- Integer range (64-bit)
print(math.maxinteger)   -- 9223372036854775807
print(math.mininteger)   -- -9223372036854775808

-- Float (double precision)
print(math.huge)         -- inf
print(-math.huge)        -- -inf
print(0/0)               -- -nan หรือ nan (Not a Number)
```

### Hex Numbers

```lua
-- ตัวอย่างที่ 19: เลขฐาน 16 (Hexadecimal)
-- ขึ้นต้นด้วย 0x หรือ 0X

local hex1 = 0xFF      -- 255
local hex2 = 0x1A      -- 26
local hex3 = 0xDEAD    -- 57005
local hex4 = 0xCAFE    -- 51966

print(hex1)  -- 255
print(hex2)  -- 26
print(hex3)  -- 57005
print(hex4)  -- 51966

-- Hex float (Lua 5.3+)
local hexFloat = 0x1.fp10  -- 1984.0
print(hexFloat)

-- แสดงเลขในรูปแบบ hex
print(string.format("0x%X", 255))   -- 0xFF
print(string.format("0x%x", 255))   -- 0xff
print(string.format("%d", 0xFF))    -- 255
```

### Scientific Notation

```lua
-- ตัวอย่างที่ 20: Scientific notation
local n1 = 1e3      -- 1000.0
local n2 = 1.5e2    -- 150.0
local n3 = 2.5e-3   -- 0.0025
local n4 = 6.022e23 -- Avogadro's number
local n5 = 1.6e-19  -- Elementary charge

print(n1)   -- 1000.0
print(n2)   -- 150.0
print(n3)   -- 0.0025
print(string.format("%.3e", n4))  -- 6.022e+23
print(string.format("%.2e", n5))  -- 1.60e-19

-- ใช้ในการคำนวณ
local lightSpeed = 3e8   -- 300,000,000 m/s
local distance = 1.5e11  -- ระยะ Earth-Sun (150 ล้านกิโลเมตร)
local time = distance / lightSpeed
print(string.format("แสงใช้เวลา %.1f วินาทีจากดวงอาทิตย์ถึงโลก", time))
-- ~499 วินาที (~8.3 นาที)
```

### Math Library

```lua
-- ตัวอย่างที่ 21: Math library
print("=== Math Library ===")

-- ค่าคงที่
print("pi = " .. math.pi)                    -- 3.1415926535898

-- ฟังก์ชันคณิตศาสตร์พื้นฐาน
print("abs(-5) = " .. math.abs(-5))          -- 5
print("ceil(3.2) = " .. math.ceil(3.2))      -- 4
print("floor(3.8) = " .. math.floor(3.8))    -- 3
print("sqrt(16) = " .. math.sqrt(16))        -- 4.0
print("pow(2,10) = " .. 2^10)                -- 1024.0

-- ฟังก์ชัน log
print("log(100) = " .. math.log(100))        -- 4.605...
print("log10 = " .. math.log(100, 10))       -- 2.0
print("log2 = " .. math.log(8, 2))           -- 3.0

-- ฟังก์ชัน trig
print("sin(pi/2) = " .. math.sin(math.pi/2))  -- 1.0
print("cos(pi) = " .. math.cos(math.pi))       -- -1.0
print("tan(pi/4) = " .. math.tan(math.pi/4))   -- ~1.0

-- ฟังก์ชัน min/max
print("min = " .. math.min(5, 3, 8, 1, 9))    -- 1
print("max = " .. math.max(5, 3, 8, 1, 9))    -- 9

-- ฟังก์ชัน random
math.randomseed(os.time())  -- seed ด้วยเวลาปัจจุบัน
print("random() = " .. math.random())           -- 0 ถึง 1
print("random(10) = " .. math.random(10))       -- 1 ถึง 10
print("random(5,10) = " .. math.random(5, 10))  -- 5 ถึง 10

-- แปลงระหว่าง integer/float
print("tointeger(3.0) = " .. tostring(math.tointeger(3.0)))  -- 3
print("type: " .. math.type(math.tointeger(3.0)))  -- integer
```

### Integer Arithmetic ใน Lua 5.4

```lua
-- ตัวอย่างที่ 22: Integer arithmetic
-- Lua 5.4 แยก integer และ float อย่างชัดเจน

local a = 10    -- integer
local b = 3     -- integer

-- การหารแบบ integer (//)
print(a // b)   -- 3 (integer floor division)
print(10 // 3)  -- 3
print(-10 // 3) -- -4 (floor หาค่า ไม่ใช่ truncate)

-- การหารแบบ float (/)
print(a / b)    -- 3.3333... (เสมอเป็น float)

-- เมื่อ integer ผสมกับ float ผลลัพธ์เป็น float
print(a + 1.0)  -- 11.0 (float)
print(math.type(a + 1.0))  -- float

-- Integer overflow wraps around
local max = math.maxinteger
print(max)           -- 9223372036854775807
print(max + 1)       -- -9223372036854775808 (overflow!)
print(math.type(max))  -- integer
```

---

## ชนิด string

### String Literals

```lua
-- ตัวอย่างที่ 23: String literals
-- Lua รองรับหลายรูปแบบ

-- Double quotes
local s1 = "Hello, World!"

-- Single quotes (เหมือนกัน)
local s2 = 'Hello, World!'

-- Long strings (multiline)
local s3 = [[
    นี่คือ
    string
    หลายบรรทัด
]]

-- Long strings ระดับต่างๆ
local s4 = [==[
    long string ระดับ 2
    สามารถมี [[ ]] ข้างใน
]==]

print(s1)
print(s2)
print(s3)
print(s4)

-- String เหมือนกันไม่ว่าจะใช้ ' หรือ "
print(s1 == s2)  -- true
```

### String Escape Sequences

```lua
-- ตัวอย่างที่ 24: Escape sequences
print("tab:\there")           -- tab character
print("newline:\nhere")       -- newline
print("carriage return:\r")   -- carriage return
print("backslash: \\")        -- backslash
print("double quote: \"")     -- double quote
print('single quote: \'')     -- single quote
print("null: \0 here")        -- null character
print("bell: \a")             -- bell (ASCII 7)
print("backspace: \b")        -- backspace
print("form feed: \f")        -- form feed
print("vertical tab: \v")     -- vertical tab

-- Numeric escapes
print("\65")    -- A (ASCII 65)
print("\66")    -- B (ASCII 66)
print("\x41")   -- A (hex 41)
print("\u{0E2A}") -- ส (Unicode Thai สระ ส)

-- ตัวอย่างใน string จริง
local text = "ชื่อ:\tสมชาย\nอายุ:\t25\nเมือง:\tกรุงเทพ"
print(text)
```

### String Length

```lua
-- ตัวอย่างที่ 25: ความยาว string ด้วย #
local s = "Hello"
print(#s)  -- 5

-- ภาษาไทย (UTF-8): # นับ bytes ไม่ใช่ characters
local thai = "สวัสดี"
print(#thai)  -- 18 (ภาษาไทย 1 ตัว = 3 bytes ใน UTF-8)
-- "สวัสดี" = 6 ตัวอักษร x 3 bytes = 18 bytes

-- English
local eng = "Hello"
print(#eng)   -- 5 (ASCII = 1 byte ต่อตัว)

-- Mixed
local mixed = "Hi สวัสดี"
print(#mixed)  -- 2 + 1 (space) + 18 = 21 bytes

-- String ว่าง
print(#"")   -- 0

-- ใช้ # กับตัวแปร
local words = {"apple", "banana", "cherry"}
for _, word in ipairs(words) do
    print(word .. " มีความยาว " .. #word .. " characters")
end
```

### String Methods

```lua
-- ตัวอย่างที่ 26: String methods (string library)
local s = "Hello, World!"

-- ความยาว
print(#s)                           -- 13
print(string.len(s))                -- 13

-- แปลงตัวพิมพ์
print(string.upper(s))              -- HELLO, WORLD!
print(string.lower(s))              -- hello, world!
print(s:upper())                    -- HELLO, WORLD! (OOP style)
print(s:lower())                    -- hello, world!

-- ค้นหา
local start, finish = s:find("World")
print("พบที่: " .. start .. "-" .. finish)  -- พบที่: 8-12

-- แทนที่
local replaced = s:gsub("World", "Lua")
print(replaced)   -- Hello, Lua!

-- ตัด
print(s:sub(1, 5))    -- Hello
print(s:sub(8))       -- World!
print(s:sub(-6))      -- orld! (นับจากท้าย)

-- ซ้ำ
print(string.rep("ab", 3))         -- ababab
print(string.rep("ab", 3, ","))    -- ab,ab,ab

-- ย้อนกลับ
print(string.reverse("hello"))    -- olleh
```

### String Pattern Matching

```lua
-- ตัวอย่างที่ 27: Pattern matching (Lua regex)
local email = "user@example.com"
local date = "2025-01-15"
local phone = "02-123-4567"

-- ตรวจสอบ email อย่างง่าย
if email:match("[%w%.]+@[%w%.]+%.[%a]+") then
    print("Email ถูกต้อง: " .. email)
end

-- ดึงส่วนประกอบของวันที่
local year, month, day = date:match("(%d%d%d%d)-(%d%d)-(%d%d)")
print("ปี: " .. year .. ", เดือน: " .. month .. ", วัน: " .. day)

-- ดึงเบอร์โทร
local area, prefix, number = phone:match("(%d+)-(%d+)-(%d+)")
print("เขต: " .. area .. ", Prefix: " .. prefix .. ", เบอร์: " .. number)

-- gmatch: วนซ้ำค้นหาทุกที่
local text = "apple 5, banana 10, cherry 3"
for word, num in text:gmatch("(%a+) (%d+)") do
    print(word .. ": " .. num)
end
```

### String Formatting

```lua
-- ตัวอย่างที่ 28: string.format() อย่างละเอียด
-- Format specifiers:

-- %d - decimal integer
print(string.format("%d", 42))           -- 42
print(string.format("%10d", 42))         -- จัดขวา 10 ช่อง
print(string.format("%-10d", 42))        -- จัดซ้าย 10 ช่อง
print(string.format("%010d", 42))        -- เติมศูนย์

-- %f - floating point
print(string.format("%f", 3.14159))      -- 3.141590
print(string.format("%.2f", 3.14159))   -- 3.14
print(string.format("%8.2f", 3.14159))  -- จัดขวา 8 ช่อง

-- %s - string
print(string.format("%s", "hello"))      -- hello
print(string.format("%10s", "hello"))    -- จัดขวา
print(string.format("%-10s", "hello"))   -- จัดซ้าย

-- %x, %X - hex
print(string.format("%x", 255))          -- ff
print(string.format("%X", 255))          -- FF
print(string.format("%08X", 255))        -- 000000FF

-- %o - octal
print(string.format("%o", 8))            -- 10

-- %e - scientific
print(string.format("%e", 12345.678))    -- 1.234568e+04
print(string.format("%.2e", 12345.678)) -- 1.23e+04

-- %q - quoted string
print(string.format("%q", 'He said "hi"'))  -- "He said \"hi\""

-- %% - literal %
print(string.format("%.1f%%", 75.5))     -- 75.5%

-- Multiple values
print(string.format("%-10s: %6.2f%%", "ภาษาไทย", 85.5))
print(string.format("%-10s: %6.2f%%", "คณิตศาสตร์", 92.3))
print(string.format("%-10s: %6.2f%%", "ภาษาอังกฤษ", 78.8))
```

---

## ชนิด table

### Table พื้นฐาน

```lua
-- ตัวอย่างที่ 29: Table - โครงสร้างข้อมูลหลักของ Lua
-- Table ทำหน้าที่เป็นได้ทั้ง array, dictionary, object, set

-- Array (sequence)
local fruits = {"apple", "banana", "cherry"}
print(fruits[1])  -- apple (index เริ่มที่ 1!)
print(fruits[2])  -- banana
print(fruits[3])  -- cherry
print(#fruits)    -- 3

-- Dictionary (hash table)
local person = {
    name = "สมชาย",
    age = 25,
    city = "กรุงเทพ"
}
print(person.name)       -- สมชาย
print(person["age"])     -- 25 (เข้าถึงด้วย bracket notation)
print(person.city)       -- กรุงเทพ

-- Mixed table
local mixed = {
    "first",      -- index 1
    "second",     -- index 2
    key = "value",
    100,          -- index 3
    nested = {1, 2, 3}
}
print(mixed[1])       -- first
print(mixed[2])       -- second
print(mixed[3])       -- 100
print(mixed.key)      -- value
print(mixed.nested[2]) -- 2
```

### Table Operations

```lua
-- ตัวอย่างที่ 30: Table operations
local t = {10, 20, 30, 40, 50}

-- เพิ่ม element
table.insert(t, 60)         -- เพิ่มที่ท้าย
table.insert(t, 2, 15)      -- เพิ่มที่ index 2

-- ลบ element
table.remove(t)             -- ลบตัวสุดท้าย
table.remove(t, 1)          -- ลบที่ index 1

-- เรียง
table.sort(t)               -- เรียงน้อยไปมาก
print(table.concat(t, ", ")) -- เชื่อมด้วย separator

-- เรียงแบบกำหนดเอง
table.sort(t, function(a, b) return a > b end)  -- มากไปน้อย
print(table.concat(t, ", "))

-- ความยาว
print(#t)

-- วนซ้ำ
for i, v in ipairs(t) do
    print(i .. ": " .. v)
end
```

---

## ชนิด function

### Function เป็น First-class Value

```lua
-- ตัวอย่างที่ 31: Function เป็น first-class value
-- ใน Lua function เป็น value เหมือนกับตัวเลขหรือ string

-- สร้าง function และเก็บใน variable
local greet = function(name)
    return "สวัสดี " .. name .. "!"
end

print(greet("สมชาย"))  -- สวัสดี สมชาย!
print(type(greet))     -- function

-- ส่ง function เป็น argument
local function apply(fn, value)
    return fn(value)
end

local double = function(n) return n * 2 end
print(apply(double, 5))  -- 10

-- เก็บ function ใน table
local math_ops = {
    add = function(a, b) return a + b end,
    sub = function(a, b) return a - b end,
    mul = function(a, b) return a * b end,
    div = function(a, b) 
        if b == 0 then return nil, "หารด้วยศูนย์ไม่ได้" end
        return a / b 
    end
}

print(math_ops.add(3, 4))  -- 7
print(math_ops.mul(5, 6))  -- 30
local result, err = math_ops.div(10, 0)
print(result, err)         -- nil    หารด้วยศูนย์ไม่ได้
```

### Closures

```lua
-- ตัวอย่างที่ 32: Closures (ฟังก์ชันที่จำค่าได้)
local function makeCounter(start)
    local count = start or 0  -- upvalue
    
    return {
        increment = function()
            count = count + 1
        end,
        decrement = function()
            count = count - 1
        end,
        get = function()
            return count
        end,
        reset = function()
            count = start or 0
        end
    }
end

local counter1 = makeCounter(0)
local counter2 = makeCounter(10)

counter1.increment()
counter1.increment()
counter1.increment()
print("counter1: " .. counter1.get())  -- 3

counter2.increment()
counter2.decrement()
print("counter2: " .. counter2.get())  -- 10

counter1.reset()
print("counter1 หลัง reset: " .. counter1.get())  -- 0
```

---

## ชนิด userdata

```lua
-- ตัวอย่างที่ 33: userdata type
-- userdata เป็น C data pointer ที่จัดการโดย C API
-- ไม่สามารถสร้างเองใน pure Lua ได้

-- ตัวอย่าง userdata ที่เห็นบ่อย
local f = io.open("test_file.txt", "w")
if f then
    print(type(f))    -- file (userdata)
    print(type(io.stdin))   -- file (userdata)
    print(type(io.stdout))  -- file (userdata)
    
    f:write("Hello from userdata!\n")
    f:close()
    
    -- ลบไฟล์ทดสอบ
    os.remove("test_file.txt")
end

-- userdata มักใช้ใน:
-- 1. File handles (io.open)
-- 2. Database connections
-- 3. OpenGL/graphics objects
-- 4. Network sockets
-- 5. Custom C structures

-- ตรวจสอบ
print(type(io.stdin))    -- file (ชนิดที่แม่นยำขึ้นอยู่กับ implementation)
```

---

## ชนิด thread (Coroutine)

### Coroutine พื้นฐาน

```lua
-- ตัวอย่างที่ 34: thread type (coroutines)
-- thread ใน Lua = coroutine (ไม่ใช่ OS thread)

local function producer()
    local items = {"apple", "banana", "cherry", "date"}
    for _, item in ipairs(items) do
        print("ผลิต: " .. item)
        coroutine.yield(item)  -- หยุดชั่วคราวและส่งค่า
    end
end

-- สร้าง coroutine
local co = coroutine.create(producer)
print(type(co))  -- thread

-- เรียกใช้ coroutine
local ok, value = coroutine.resume(co)
print("ได้รับ: " .. (value or "nil"))  -- apple

ok, value = coroutine.resume(co)
print("ได้รับ: " .. (value or "nil"))  -- banana

ok, value = coroutine.resume(co)
print("ได้รับ: " .. (value or "nil"))  -- cherry

-- ตรวจสอบ status
print("Status: " .. coroutine.status(co))  -- suspended หรือ dead

-- วนจนครบ
while coroutine.status(co) ~= "dead" do
    ok, value = coroutine.resume(co)
    if value then
        print("ได้รับ: " .. value)
    end
end
```

### Coroutine ใช้ทำ Iterator

```lua
-- ตัวอย่างที่ 35: Coroutine เป็น custom iterator
local function range(from, to, step)
    step = step or 1
    return coroutine.wrap(function()
        for i = from, to, step do
            coroutine.yield(i)
        end
    end)
end

-- ใช้งาน
print("เลข 1 ถึง 10:")
for n in range(1, 10) do
    io.write(n .. " ")
end
print()

print("เลขคู่ 2 ถึง 20:")
for n in range(2, 20, 2) do
    io.write(n .. " ")
end
print()

print("เลข 10 ถึง 1 (ลดลง):")
for n in range(10, 1, -1) do
    io.write(n .. " ")
end
print()
```

---

## Type Coercion อัตโนมัติ

### Number ↔ String Coercion

```lua
-- ตัวอย่างที่ 36: Automatic type coercion
-- Lua แปลง string ↔ number อัตโนมัติในบางกรณี

-- String ถูกแปลงเป็น number ในนิพจน์คณิตศาสตร์
print("10" + 5)      -- 15 (string "10" แปลงเป็น number)
print("3.14" * 2)    -- 6.28
print("10" - "3")    -- 7
print("2" ^ 8)       -- 256.0

-- Number ถูกแปลงเป็น string ใน concatenation
print(10 .. 20)      -- 1020 (number แปลงเป็น string)
print(3.14 .. "!")   -- 3.14!

-- แต่ไม่แปลงในทุกกรณี
-- print("10" == 10)  -- false! (ไม่แปลงใน comparison)
print("10" == 10)    -- false
print(10 == 10)      -- true
print("10" == "10")  -- true

-- แปลงไม่ได้จะ error
-- local x = "hello" + 1  -- Error: attempt to perform arithmetic on a string value
```

### Boolean ไม่มี Coercion

```lua
-- ตัวอย่างที่ 37: Boolean ไม่มี automatic coercion
-- ต่างจาก Python/JavaScript!

local x = true

-- ไม่สามารถใช้ boolean ในนิพจน์คณิตศาสตร์โดยตรง
-- print(true + 1)   -- Error!
-- print(false - 0)  -- Error!

-- ต้องแปลงเอง
local num = x and 1 or 0  -- ternary idiom
print("true as number: " .. num)  -- 1

-- Boolean เปรียบเทียบ
print(true == true)   -- true
print(false == false) -- true
print(true == false)  -- false
print(true ~= false)  -- true

-- ไม่สามารถเปรียบเทียบกับ number
print(true == 1)   -- false (ไม่เท่ากัน!)
print(false == 0)  -- false (ไม่เท่ากัน!)
```

---

## การแปลงชนิดอย่างชัดเจน

### tostring() ทุกชนิด

```lua
-- ตัวอย่างที่ 38: tostring() กับทุกชนิด
print(tostring(nil))          -- nil
print(tostring(true))         -- true
print(tostring(false))        -- false
print(tostring(42))           -- 42
print(tostring(3.14))         -- 3.14
print(tostring(math.huge))    -- inf
print(tostring(-math.huge))   -- -inf
print(tostring(0/0))          -- -nan

-- ตัวเลขแบบต่างๆ
print(tostring(1000000))      -- 1000000
print(tostring(1e10))         -- 10000000000.0
print(tostring(1e100))        -- 1e+100
print(tostring(0.001))        -- 0.001
print(tostring(0.0001))       -- 0.0001
print(tostring(0.00001))      -- 1e-05

-- Table และ Function แสดง address
local t = {}
print(tostring(t))     -- table: 0x... (address)
print(tostring(print)) -- function: 0x... (address)
```

### tonumber() อย่างละเอียด

```lua
-- ตัวอย่างที่ 39: tonumber() อย่างละเอียด
-- String ที่แปลงได้
print(tonumber("42"))       -- 42
print(tonumber("42.5"))     -- 42.5
print(tonumber("  42  "))   -- 42 (trim spaces)
print(tonumber("0xff"))     -- 255
print(tonumber("1e3"))      -- 1000.0
print(tonumber("-42"))      -- -42
print(tonumber("+42"))      -- 42

-- String ที่แปลงไม่ได้
print(tonumber("hello"))    -- nil
print(tonumber("42abc"))    -- nil
print(tonumber(""))         -- nil
print(tonumber(" "))        -- nil

-- ระบุฐาน
print(tonumber("ff", 16))   -- 255
print(tonumber("1010", 2))  -- 10
print(tonumber("17", 8))    -- 15
print(tonumber("ZZ", 36))   -- 1295 (base-36)

-- Number คืนค่าเดิม
print(tonumber(42))         -- 42
print(tonumber(3.14))       -- 3.14

-- Boolean และ nil คืน nil
print(tonumber(true))       -- nil
print(tonumber(nil))        -- nil
```

### การแปลงชนิดปลอดภัย

```lua
-- ตัวอย่างที่ 40: Safe type conversion
local function safeToNumber(value, default)
    local n = tonumber(value)
    return n or (default or 0)
end

local function safeToString(value)
    if value == nil then
        return "nil"
    elseif type(value) == "boolean" then
        return value and "true" or "false"
    elseif type(value) == "table" then
        return "{table}"
    else
        return tostring(value)
    end
end

-- ทดสอบ
print(safeToNumber("42"))       -- 42
print(safeToNumber("hello"))    -- 0 (default)
print(safeToNumber("hello", -1)) -- -1 (custom default)
print(safeToNumber(nil))         -- 0

print(safeToString(42))          -- 42
print(safeToString(true))        -- true
print(safeToString(nil))         -- nil
print(safeToString({1,2,3}))    -- {table}
```

---

## Pattern สำหรับ Constants

### Constants ด้วย Local Variables

```lua
-- ตัวอย่างที่ 41: Constants pattern ใน Lua
-- Lua ไม่มี const keyword แต่ใช้ convention UPPER_CASE

-- Physics constants
local SPEED_OF_LIGHT = 299792458    -- m/s
local PLANCK_CONSTANT = 6.626e-34   -- J·s
local GRAVITY = 9.80665             -- m/s²
local AVOGADRO = 6.022e23           -- mol⁻¹

-- Math constants
local PI = math.pi
local E = math.exp(1)
local TAU = 2 * PI

-- Application constants
local MAX_RETRIES = 3
local TIMEOUT_SECONDS = 30
local DEFAULT_PORT = 8080
local APP_VERSION = "1.0.0"

-- HTTP status codes
local HTTP_OK = 200
local HTTP_NOT_FOUND = 404
local HTTP_INTERNAL_ERROR = 500

print("ความเร็วแสง: " .. SPEED_OF_LIGHT .. " m/s")
print("Pi: " .. PI)
print("Timeout: " .. TIMEOUT_SECONDS .. " วินาที")
```

### Constants ใน Table (Enum Pattern)

```lua
-- ตัวอย่างที่ 42: Enum pattern ใน Lua
-- ใช้ table เพื่อจัดกลุ่ม constants

local Direction = {
    NORTH = "north",
    SOUTH = "south",
    EAST = "east",
    WEST = "west"
}

local Color = {
    RED = 1,
    GREEN = 2,
    BLUE = 3,
    ALPHA = 4
}

local Status = {
    PENDING = 0,
    ACTIVE = 1,
    INACTIVE = 2,
    DELETED = 3
}

-- ใช้งาน
local playerDir = Direction.NORTH
print("ทิศ: " .. playerDir)

local bgColor = Color.BLUE
print("สี: " .. bgColor)

local userStatus = Status.ACTIVE
if userStatus == Status.ACTIVE then
    print("ผู้ใช้ active")
end

-- ป้องกันการแก้ไข (read-only table)
-- ใช้ metatable ทำ read-only
local function makeReadOnly(t)
    local proxy = {}
    local mt = {
        __index = t,
        __newindex = function(_, k, _)
            error("พยายามแก้ไข constant: " .. tostring(k))
        end
    }
    setmetatable(proxy, mt)
    return proxy
end

local ReadOnlyDirection = makeReadOnly(Direction)
print(ReadOnlyDirection.NORTH)  -- north

-- ReadOnlyDirection.NORTH = "south"  -- Error!
```

### Lua 5.4 <const> attribute

```lua
-- ตัวอย่างที่ 43: <const> attribute ใน Lua 5.4
-- Lua 5.4 เพิ่ม <const> สำหรับ local constants

local PI <const> = 3.14159265358979
local MAX_SIZE <const> = 1000
local APP_NAME <const> = "MyApp"

print(PI)        -- 3.14159265358979
print(MAX_SIZE)  -- 1000
print(APP_NAME)  -- MyApp

-- พยายามแก้ไขจะ error ตอน compile time!
-- PI = 3.14  -- Error: attempt to assign to const variable 'PI'

-- ประโยชน์: Lua optimizer สามารถ inline ค่าได้
-- ทำให้ code เร็วขึ้น
```

---

## _G Global Table

### _G คืออะไร?

```lua
-- ตัวอย่างที่ 44: _G - Global environment table
-- ตัวแปร global ทั้งหมดถูกเก็บใน _G table

myGlobal = "hello"
counter = 0

-- เข้าถึงผ่าน _G
print(_G["myGlobal"])  -- hello
print(_G["counter"])   -- 0
print(_G["print"])     -- function: 0x...

-- _G เป็น table ที่เก็บ global environment
print(type(_G))        -- table

-- แก้ไขผ่าน _G (เหมือนกับแก้ไข global)
_G["newVar"] = 42
print(newVar)          -- 42

-- ลบ global ผ่าน _G
_G["myGlobal"] = nil
print(myGlobal)        -- nil
```

### ดูตัวแปร Global ทั้งหมด

```lua
-- ตัวอย่างที่ 45: แสดง global variables ทั้งหมด
-- (เฉพาะที่สร้างเอง ไม่รวม built-ins)

-- สร้างตัวแปร global ทดสอบ
myVar1 = "test1"
myVar2 = 42
myVar3 = true

-- รวบรวม global ที่ไม่ใช่ built-in
local builtins = {
    print=true, type=true, pairs=true, ipairs=true,
    next=true, select=true, unpack=true, tostring=true,
    tonumber=true, rawget=true, rawset=true, rawequal=true,
    rawlen=true, setmetatable=true, getmetatable=true,
    require=true, dofile=true, load=true, loadfile=true,
    pcall=true, xpcall=true, error=true, assert=true,
    collectgarbage=true, gcinfo=true, newproxy=true,
    _VERSION=true, _G=true,
    string=true, table=true, math=true, io=true,
    os=true, coroutine=true, package=true, utf8=true,
    debug=true, arg=true
}

print("=== ตัวแปร Global ที่สร้างเอง ===")
local customGlobals = {}
for k, v in pairs(_G) do
    if not builtins[k] then
        table.insert(customGlobals, 
            string.format("  %-15s = %s (%s)", k, tostring(v), type(v)))
    end
end
table.sort(customGlobals)
for _, line in ipairs(customGlobals) do
    print(line)
end
```

### Dynamic Variable Names

```lua
-- ตัวอย่างที่ 46: สร้างตัวแปรแบบ dynamic ผ่าน _G
-- (ไม่แนะนำในงาน production แต่มีประโยชน์บางกรณี)

-- สร้างตัวแปร global แบบ dynamic
for i = 1, 5 do
    _G["item" .. i] = i * 10
end

-- เข้าถึง
for i = 1, 5 do
    print("item" .. i .. " = " .. _G["item" .. i])
end

-- ทางที่ดีกว่า: ใช้ table แทน
local items = {}
for i = 1, 5 do
    items[i] = i * 10
end

for i, v in ipairs(items) do
    print("items[" .. i .. "] = " .. v)
end
```

---

## Variable Scoping อย่างละเอียด

```lua
-- ตัวอย่างที่ 47: Scope ที่ซับซ้อน
local x = 1

local function outer()
    local x = 2  -- ซ่อน outer x
    
    local function middle()
        local x = 3  -- ซ่อน middle x
        
        local function inner()
            -- x ที่นี่คืออะไร?
            print("inner x = " .. x)   -- 3 (จาก middle)
        end
        
        inner()
        print("middle x = " .. x)  -- 3
    end
    
    middle()
    print("outer x = " .. x)  -- 2
end

outer()
print("global x = " .. x)  -- 1
```

```lua
-- ตัวอย่างที่ 48: Upvalues และ Closures
local function createAdder(n)
    -- n เป็น upvalue ของ function ที่คืนกลับมา
    return function(x)
        return x + n
    end
end

local add5 = createAdder(5)
local add10 = createAdder(10)
local add100 = createAdder(100)

print(add5(3))    -- 8
print(add10(3))   -- 13
print(add100(3))  -- 103

-- แต่ละ closure มี upvalue ของตัวเอง
print(add5(0))    -- 5
print(add10(0))   -- 10
```

---

## โปรแกรมตัวอย่างรวม

### โปรแกรมตรวจสอบชนิดข้อมูล

```lua
-- ตัวอย่างที่ 49: โปรแกรมวิเคราะห์ชนิดข้อมูล
local function analyzeValue(value, name)
    name = name or "value"
    local t = type(value)
    
    print("=== วิเคราะห์ " .. name .. " ===")
    print("ค่า: " .. tostring(value))
    print("ชนิด: " .. t)
    
    if t == "number" then
        print("  Integer/Float: " .. math.type(value))
        if math.type(value) == "integer" then
            print("  Hex: 0x" .. string.format("%X", value))
            print("  Binary: ..." )  -- simplified
        else
            print("  ทศนิยม: " .. string.format("%.6f", value))
        end
    elseif t == "string" then
        print("  ความยาว: " .. #value .. " bytes")
        print("  Upper: " .. value:upper())
        print("  Lower: " .. value:lower())
    elseif t == "boolean" then
        print("  เป็น truthy: " .. tostring(value))
        print("  not: " .. tostring(not value))
    elseif t == "table" then
        local count = 0
        for _ in pairs(value) do count = count + 1 end
        print("  จำนวน entries: " .. count)
        print("  Array length: " .. #value)
    elseif t == "function" then
        print("  เรียกใช้ได้: ใช่")
    end
    print()
end

analyzeValue(42, "integer")
analyzeValue(3.14, "float")
analyzeValue("Hello World", "string")
analyzeValue(true, "boolean")
analyzeValue({1, 2, 3, name="Lua"}, "table")
analyzeValue(print, "function")
analyzeValue(nil, "nil_value")
```

### โปรแกรมสถิติ

```lua
-- ตัวอย่างที่ 50: โปรแกรมคำนวณสถิติ
local function statistics(data)
    if type(data) ~= "table" or #data == 0 then
        return nil, "ต้องการ table ที่ไม่ว่าง"
    end
    
    -- ตรวจสอบว่าทุก element เป็น number
    for i, v in ipairs(data) do
        if type(v) ~= "number" then
            return nil, "element ที่ " .. i .. " ไม่ใช่ตัวเลข"
        end
    end
    
    local n = #data
    local sum = 0
    local min = data[1]
    local max = data[1]
    
    for _, v in ipairs(data) do
        sum = sum + v
        if v < min then min = v end
        if v > max then max = v end
    end
    
    local mean = sum / n
    
    -- คำนวณ variance
    local variance = 0
    for _, v in ipairs(data) do
        variance = variance + (v - mean)^2
    end
    variance = variance / n
    local stddev = math.sqrt(variance)
    
    -- median
    local sorted = {}
    for i, v in ipairs(data) do sorted[i] = v end
    table.sort(sorted)
    local median
    if n % 2 == 0 then
        median = (sorted[n//2] + sorted[n//2 + 1]) / 2
    else
        median = sorted[(n+1)//2]
    end
    
    return {
        count = n,
        sum = sum,
        min = min,
        max = max,
        mean = mean,
        median = median,
        variance = variance,
        stddev = stddev,
        range = max - min
    }
end

-- ทดสอบ
local scores = {85, 92, 78, 95, 88, 76, 90, 83, 91, 87}

local stats, err = statistics(scores)
if err then
    print("Error: " .. err)
else
    print("=== สถิติคะแนน ===")
    print("จำนวน: " .. stats.count)
    print("รวม: " .. stats.sum)
    print("ต่ำสุด: " .. stats.min)
    print("สูงสุด: " .. stats.max)
    print("ช่วง: " .. stats.range)
    print("ค่าเฉลี่ย: " .. string.format("%.2f", stats.mean))
    print("มัธยฐาน: " .. string.format("%.2f", stats.median))
    print("ส่วนเบี่ยงเบนมาตรฐาน: " .. string.format("%.2f", stats.stddev))
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ตรวจสอบชนิดข้อมูล

เขียนฟังก์ชัน `checkType(value)` ที่แสดงข้อมูลเกี่ยวกับค่าที่ส่งเข้ามา:

```lua
-- แบบฝึกหัดที่ 1 - เฉลย
local function checkType(value)
    local t = type(value)
    io.write("ค่า: " .. tostring(value))
    io.write("  | ชนิด: " .. t)
    
    if t == "number" then
        io.write("  | subtype: " .. math.type(value))
    elseif t == "string" then
        io.write("  | ความยาว: " .. #value)
    end
    
    io.write("  | truthy: " .. tostring(value and true or false))
    print()
end

checkType(nil)
checkType(true)
checkType(false)
checkType(0)
checkType(42)
checkType(3.14)
checkType("")
checkType("hello")
checkType({})
checkType({1,2,3})
checkType(print)
```

### แบบฝึกหัดที่ 2: ระบบ Inventory

สร้างระบบ inventory อย่างง่ายที่ใช้ table เก็บสินค้า:

```lua
-- แบบฝึกหัดที่ 2 - เฉลย
local inventory = {}

-- เพิ่มสินค้า
local function addItem(name, quantity, price)
    if type(name) ~= "string" then
        error("ชื่อสินค้าต้องเป็น string")
    end
    if type(quantity) ~= "number" or quantity < 0 then
        error("จำนวนต้องเป็นตัวเลขที่ไม่ติดลบ")
    end
    if type(price) ~= "number" or price < 0 then
        error("ราคาต้องเป็นตัวเลขที่ไม่ติดลบ")
    end
    
    if inventory[name] then
        inventory[name].quantity = inventory[name].quantity + quantity
    else
        inventory[name] = {
            quantity = quantity,
            price = price,
            total = 0
        }
    end
    inventory[name].total = inventory[name].quantity * inventory[name].price
end

-- แสดง inventory
local function showInventory()
    print("=== รายการสินค้า ===")
    print(string.format("%-15s %8s %10s %12s", "สินค้า", "จำนวน", "ราคา/หน่วย", "รวม"))
    print(string.rep("-", 50))
    
    local grandTotal = 0
    for name, item in pairs(inventory) do
        print(string.format("%-15s %8d %10.2f %12.2f",
            name, item.quantity, item.price, item.total))
        grandTotal = grandTotal + item.total
    end
    
    print(string.rep("-", 50))
    print(string.format("%-15s %8s %10s %12.2f", "รวมทั้งหมด", "", "", grandTotal))
end

-- ทดสอบ
addItem("แอปเปิ้ล", 50, 15.0)
addItem("กล้วย", 100, 5.0)
addItem("ส้ม", 75, 12.0)
addItem("มะม่วง", 30, 25.0)
addItem("แอปเปิ้ล", 20, 15.0)  -- เพิ่มเข้าไปอีก

showInventory()
```

### แบบฝึกหัดที่ 3: ตัวแปลงหน่วย Universal

สร้างโปรแกรมที่แปลงหน่วยต่างๆ โดยใช้ table เก็บ conversion factors:

```lua
-- แบบฝึกหัดที่ 3 - เฉลย
local conversions = {
    length = {
        base = "meter",
        units = {
            meter = 1,
            kilometer = 0.001,
            centimeter = 100,
            millimeter = 1000,
            inch = 39.3701,
            foot = 3.28084,
            yard = 1.09361,
            mile = 0.000621371
        }
    },
    weight = {
        base = "kilogram",
        units = {
            kilogram = 1,
            gram = 1000,
            pound = 2.20462,
            ounce = 35.274,
            ton = 0.001
        }
    },
    temperature = {
        -- อุณหภูมิต้องใช้ formula ไม่ใช่ factor
    }
}

local function convert(category, value, fromUnit, toUnit)
    local cat = conversions[category]
    if not cat then
        return nil, "ไม่รู้จักหมวดหมู่: " .. category
    end
    
    local fromFactor = cat.units[fromUnit]
    local toFactor = cat.units[toUnit]
    
    if not fromFactor then
        return nil, "ไม่รู้จักหน่วย: " .. fromUnit
    end
    if not toFactor then
        return nil, "ไม่รู้จักหน่วย: " .. toUnit
    end
    
    -- แปลงเป็น base unit ก่อน แล้วแปลงเป็น target unit
    local baseValue = value / fromFactor
    local result = baseValue * toFactor
    
    return result
end

-- ทดสอบ
local tests = {
    {"length", 1, "kilometer", "meter"},
    {"length", 100, "centimeter", "inch"},
    {"length", 1, "mile", "kilometer"},
    {"weight", 1, "kilogram", "pound"},
    {"weight", 500, "gram", "ounce"},
}

print("=== ผลการแปลงหน่วย ===")
for _, test in ipairs(tests) do
    local result, err = convert(table.unpack(test))
    if err then
        print("Error: " .. err)
    else
        print(string.format("%.4g %s = %.4g %s",
            test[2], test[3], result, test[4]))
    end
end
```

### แบบฝึกหัดที่ 4: String Processing

เขียนโปรแกรมที่ประมวลผล string และแสดงสถิติ:

```lua
-- แบบฝึกหัดที่ 4 - เฉลย
local function analyzeText(text)
    if type(text) ~= "string" then
        return nil, "ต้องการ string"
    end
    
    local stats = {
        length = #text,
        words = 0,
        sentences = 0,
        letters = 0,
        digits = 0,
        spaces = 0,
        uppercase = 0,
        lowercase = 0,
        paragraphs = 0
    }
    
    -- นับตัวอักษรแต่ละชนิด
    for c in text:gmatch(".") do
        if c:match("[%a]") then
            stats.letters = stats.letters + 1
            if c:match("[%u]") then
                stats.uppercase = stats.uppercase + 1
            else
                stats.lowercase = stats.lowercase + 1
            end
        elseif c:match("[%d]") then
            stats.digits = stats.digits + 1
        elseif c:match("[%s]") then
            stats.spaces = stats.spaces + 1
        end
    end
    
    -- นับคำ
    for _ in text:gmatch("%S+") do
        stats.words = stats.words + 1
    end
    
    -- นับประโยค
    for _ in text:gmatch("[%.!%?]+") do
        stats.sentences = stats.sentences + 1
    end
    
    -- นับย่อหน้า
    stats.paragraphs = 1
    for _ in text:gmatch("\n\n") do
        stats.paragraphs = stats.paragraphs + 1
    end
    
    return stats
end

local sampleText = [[
Hello, World! This is a test text.
It has multiple sentences. Some have numbers like 42 and 3.14.
This is the second paragraph with UPPERCASE letters.
]]

local stats, err = analyzeText(sampleText)
if err then
    print("Error: " .. err)
else
    print("=== สถิติข้อความ ===")
    for k, v in pairs(stats) do
        print(string.format("  %-15s: %d", k, v))
    end
end
```

### แบบฝึกหัดที่ 5: Type Checker Function

สร้าง function ที่ตรวจสอบ argument types:

```lua
-- แบบฝึกหัดที่ 5 - เฉลย
-- สร้าง type checking system

local function typeCheck(funcName, args, expectedTypes)
    for i, expected in ipairs(expectedTypes) do
        local actual = type(args[i])
        
        -- รองรับ multiple types ด้วย | เช่น "number|string"
        local valid = false
        for t in expected:gmatch("[^|]+") do
            if actual == t or t == "any" then
                valid = true
                break
            end
        end
        
        if not valid then
            error(string.format(
                "%s: argument %d ต้องเป็น %s แต่ได้รับ %s",
                funcName, i, expected, actual
            ), 2)
        end
    end
end

-- ตัวอย่างการใช้งาน
local function power(base, exp)
    typeCheck("power", {base, exp}, {"number", "number"})
    return base ^ exp
end

local function repeat_string(s, n)
    typeCheck("repeat_string", {s, n}, {"string", "number"})
    return s:rep(n)
end

local function formatTable(t, sep)
    typeCheck("formatTable", {t, sep}, {"table", "string|nil"})
    sep = sep or ", "
    local parts = {}
    for i, v in ipairs(t) do
        parts[i] = tostring(v)
    end
    return table.concat(parts, sep)
end

-- ทดสอบ
print(power(2, 10))                    -- 1024.0
print(repeat_string("abc", 3))         -- abcabcabc
print(formatTable({1,2,3,4,5}))        -- 1, 2, 3, 4, 5
print(formatTable({1,2,3,4,5}, " | ")) -- 1 | 2 | 3 | 4 | 5

-- ทดสอบ error handling
local ok, err = pcall(power, "hello", 2)
print("Error caught: " .. err)

ok, err = pcall(repeat_string, 42, 3)
print("Error caught: " .. err)
```

---

## สรุปบทที่ 2

ในบทนี้เราได้เรียนรู้:

1. **8 ชนิดข้อมูลของ Lua**: nil, boolean, number, string, table, function, userdata, thread
2. **type() function**: ตรวจสอบชนิดข้อมูล ณ runtime
3. **กฎการตั้งชื่อ**: ขึ้นต้นด้วยตัวอักษรหรือ `_`, ไม่ใช้ keywords
4. **Global vs Local**: local เร็วกว่าและปลอดภัยกว่า
5. **nil**: ค่าที่แทน "ไม่มีตัวตน", เป็น falsy
6. **boolean**: true/false, เฉพาะ nil และ false เป็น falsy
7. **number**: integer และ float, รองรับ hex, scientific notation
8. **string**: single/double quote, long strings, escape sequences
9. **table**: โครงสร้างข้อมูลหลัก ทำได้ทั้ง array/dict/object
10. **function**: first-class value, closures, upvalues
11. **userdata**: C data pointer
12. **thread**: coroutines
13. **Type coercion**: string ↔ number อัตโนมัติ
14. **Constants**: UPPER_CASE convention, Lua 5.4 `<const>`
15. **_G**: global environment table

### ตาราง type() Values

```
type(nil)       → "nil"
type(true)      → "boolean"
type(false)     → "boolean"
type(42)        → "number"
type(3.14)      → "number"
type("hello")   → "string"
type({})        → "table"
type(print)     → "function"
type(io.stdin)  → "file" (userdata)
type(co)        → "thread"
```

**บทถัดไป:** บทที่ 3 - ตัวดำเนินการและนิพจน์ จะครอบคลุม arithmetic, relational, logical operators, bitwise operators (5.3+) และ operator precedence
