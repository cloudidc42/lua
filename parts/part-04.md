# บทที่ 4: String และการทำงานกับข้อความ

## บทนำ

String (สตริง) คือข้อมูลประเภทข้อความใน Lua ซึ่งเป็น immutable (ไม่สามารถเปลี่ยนแปลงได้โดยตรง) String ใน Lua เก็บข้อมูลแบบ byte sequence ซึ่งหมายความว่าสามารถเก็บข้อมูลไบนารีได้เช่นกัน

---

## 4.1 การสร้าง String

### ตัวอย่างที่ 1: การสร้าง String ด้วยวิธีต่างๆ

```lua
-- ใช้เครื่องหมาย single quote
local s1 = 'สวัสดี Lua'
print(s1)  -- สวัสดี Lua

-- ใช้เครื่องหมาย double quote
local s2 = "Hello, World!"
print(s2)  -- Hello, World!

-- ใช้ double square bracket (long string)
local s3 = [[
บรรทัดที่หนึ่ง
บรรทัดที่สอง
บรรทัดที่สาม
]]
print(s3)
-- (blank line)
-- บรรทัดที่หนึ่ง
-- บรรทัดที่สอง
-- บรรทัดที่สาม

-- Long string ระดับ 2 (มีเครื่องหมาย = คั่น)
local s4 = [==[
นี่คือ long string ระดับ 2
สามารถใส่ ]] ได้โดยไม่มีปัญหา
]==]
print(s4)
```

### ตัวอย่างที่ 2: Escape sequences

```lua
-- Escape sequences ที่ใช้บ่อย
print("Tab:\there")          -- Tab:	here
print("Newline:\nSecond line") 
-- Newline:
-- Second line
print("Quote: \"quoted\"")   -- Quote: "quoted"
print("Backslash: \\")       -- Backslash: \
print("Null: \0 end")        -- Null:  end (มี null byte)
print("\65\66\67")            -- ABC  (decimal escape)
print("\x41\x42\x43")        -- ABC  (hex escape)
print("\u{0E2A}")             -- ส    (Unicode escape - Lua 5.3+)
```

---

## 4.2 string.len() — ความยาวของ String

### ตัวอย่างที่ 3: การหาความยาว

```lua
local str = "Hello"
print(string.len(str))   -- 5
print(#str)              -- 5 (operator # เทียบเท่า string.len)

local thai = "สวัสดี"
print(#thai)             -- 18 (UTF-8: แต่ละตัวอักษรไทยใช้ 3 bytes)
-- หมายเหตุ: # นับ bytes ไม่ใช่ characters

local empty = ""
print(#empty)            -- 0

-- นับจำนวน bytes
local s = "abc\0def"
print(#s)  -- 7 (รวม null byte)
```

### ตัวอย่างที่ 4: ใช้ # กับ string ในเงื่อนไข

```lua
local function checkEmpty(s)
    if #s == 0 then
        return "ว่างเปล่า"
    elseif #s <= 5 then
        return "สั้น"
    else
        return "ยาว"
    end
end

print(checkEmpty(""))        -- ว่างเปล่า
print(checkEmpty("Hi"))      -- สั้น
print(checkEmpty("Hello World"))  -- ยาว
```

---

## 4.3 string.upper() และ string.lower()

### ตัวอย่างที่ 5: แปลง case

```lua
local str = "Hello World"
print(string.upper(str))   -- HELLO WORLD
print(string.lower(str))   -- hello world

-- ใช้ method syntax (OOP-style)
print(str:upper())   -- HELLO WORLD
print(str:lower())   -- hello world

-- ใช้กับตัวเลขและสัญลักษณ์ (ไม่เปลี่ยน)
print(string.upper("abc123!@#"))  -- ABC123!@#
print(string.lower("ABC123!@#"))  -- abc123!@#
```

### ตัวอย่างที่ 6: เปรียบเทียบ string แบบ case-insensitive

```lua
local function equalsIgnoreCase(a, b)
    return a:lower() == b:lower()
end

print(equalsIgnoreCase("Hello", "HELLO"))   -- true
print(equalsIgnoreCase("Lua", "lua"))       -- true
print(equalsIgnoreCase("Lua", "Python"))    -- false

-- Capitalize first letter
local function capitalize(s)
    if #s == 0 then return s end
    return s:sub(1,1):upper() .. s:sub(2):lower()
end

print(capitalize("hello"))   -- Hello
print(capitalize("WORLD"))   -- World
print(capitalize("lUA"))     -- Lua
```

---

## 4.4 string.sub() — การตัด Substring

### ตัวอย่างที่ 7: การใช้ sub() พื้นฐาน

```lua
local str = "Hello, World!"

-- sub(i, j) — ดึงตำแหน่ง i ถึง j
print(str:sub(1, 5))    -- Hello    (ตำแหน่ง 1-5)
print(str:sub(8, 13))   -- World!   (ตำแหน่ง 8-13)
print(str:sub(1, 1))    -- H        (ตัวแรก)

-- index ติดลบ = นับจากท้าย
print(str:sub(-6))       -- World!   (6 ตัวสุดท้าย)
print(str:sub(-6, -2))   -- World    (ตำแหน่ง -6 ถึง -2)
print(str:sub(-1))       -- !        (ตัวสุดท้าย)

-- ไม่ระบุ j = ถึงสุดท้าย
print(str:sub(8))        -- World!
```

### ตัวอย่างที่ 8: ดึงส่วนของ string

```lua
local filename = "document.txt"

-- ดึง extension
local function getExtension(fname)
    local dot = fname:find("%.")
    if dot then
        return fname:sub(dot)
    end
    return ""
end

-- ดึงชื่อไฟล์ไม่รวม extension
local function getBasename(fname)
    local dot = fname:find("%.")
    if dot then
        return fname:sub(1, dot - 1)
    end
    return fname
end

print(getExtension("document.txt"))   -- .txt
print(getExtension("image.png"))      -- .png
print(getBasename("document.txt"))    -- document
print(getBasename("image.png"))       -- image
```

---

## 4.5 string.rep() — การทำซ้ำ String

### ตัวอย่างที่ 9: การทำซ้ำ

```lua
-- rep(s, n) — ทำซ้ำ n ครั้ง
print(string.rep("abc", 3))       -- abcabcabc
print(string.rep("*", 10))        -- **********
print(string.rep("Lua", 0))       -- (ว่างเปล่า)

-- rep(s, n, sep) — ทำซ้ำพร้อม separator (Lua 5.2+)
print(string.rep("Ha", 3, "-"))   -- Ha-Ha-Ha
print(string.rep("*", 5, " "))    -- * * * * *

-- สร้าง separator line
local function makeLine(char, width)
    return string.rep(char, width)
end

print(makeLine("=", 40))
-- ========================================
print(makeLine("-", 20))
-- --------------------
```

### ตัวอย่างที่ 10: สร้าง Table border

```lua
local function printBox(text)
    local width = #text + 4
    local border = "+" .. string.rep("-", width - 2) .. "+"
    print(border)
    print("| " .. text .. " |")
    print(border)
end

printBox("Hello, Lua!")
-- +-------------+
-- | Hello, Lua! |
-- +-------------+

printBox("สวัสดี")
-- +----------+
-- | สวัสดี |
-- +----------+
```

---

## 4.6 string.reverse() — การกลับสตริง

### ตัวอย่างที่ 11: การกลับ String

```lua
print(string.reverse("Hello"))    -- olleH
print(string.reverse("12345"))    -- 54321
print(string.reverse("abcde"))    -- edcba
print(string.reverse("a"))        -- a
print(string.reverse(""))         -- (ว่างเปล่า)

-- ตรวจสอบ palindrome (ASCII เท่านั้น)
local function isPalindrome(s)
    s = s:lower()
    return s == s:reverse()
end

print(isPalindrome("racecar"))   -- true
print(isPalindrome("level"))     -- true
print(isPalindrome("hello"))     -- false
print(isPalindrome("A"))         -- true
```

---

## 4.7 string.format() — การจัดรูปแบบ String

### ตัวอย่างที่ 12: Format specifiers พื้นฐาน

```lua
-- %d — integer
print(string.format("%d", 42))        -- 42
print(string.format("%d", -100))      -- -100
print(string.format("%5d", 42))       -- "   42" (width 5, right-aligned)
print(string.format("%-5d|", 42))     -- "42   |" (left-aligned)
print(string.format("%05d", 42))      -- 00042 (zero-padded)

-- %i — integer (เหมือน %d)
print(string.format("%i", 255))       -- 255

-- %u — unsigned integer
print(string.format("%u", 42))        -- 42

-- %o — octal
print(string.format("%o", 8))         -- 10
print(string.format("%o", 255))       -- 377

-- %x — hexadecimal (lowercase)
print(string.format("%x", 255))       -- ff
print(string.format("%X", 255))       -- FF
print(string.format("%08x", 255))     -- 000000ff

-- %e — scientific notation (lowercase)
print(string.format("%e", 314.16e-2))  -- 3.141600e+00
print(string.format("%E", 314.16e-2))  -- 3.141600E+00

-- %f — floating point
print(string.format("%f", 3.14159))    -- 3.141590
print(string.format("%.2f", 3.14159)) -- 3.14
print(string.format("%8.2f", 3.14))   -- "    3.14"
print(string.format("%-8.2f|", 3.14)) -- "3.14    |"

-- %g — shorter of %e or %f
print(string.format("%g", 100.0))     -- 100
print(string.format("%g", 0.0001))    -- 0.0001
print(string.format("%g", 0.00001))   -- 1e-05
```

### ตัวอย่างที่ 13: Format specifiers สำหรับ String

```lua
-- %s — string
print(string.format("%s", "Hello"))       -- Hello
print(string.format("%10s", "Hello"))     -- "     Hello" (right-aligned)
print(string.format("%-10s|", "Hello"))   -- "Hello     |" (left-aligned)
print(string.format("%.3s", "Hello"))     -- Hel (truncate to 3 chars)

-- %q — quoted string (safe for Lua code)
print(string.format("%q", "Hello"))        -- "Hello"
print(string.format("%q", 'say "hi"'))     -- "say \"hi\""
print(string.format("%q", "line1\nline2")) -- "line1\nline2"

-- %c — character from ASCII code
print(string.format("%c", 65))    -- A
print(string.format("%c", 97))    -- a
print(string.format("%c%c%c", 72, 105, 33))  -- Hi!

-- %% — literal percent sign
print(string.format("%.1f%%", 99.9))  -- 99.9%
```

### ตัวอย่างที่ 14: การใช้ format สร้าง output ที่สวยงาม

```lua
-- สร้าง table report
local products = {
    {name = "Apple",  price = 25.50,  qty = 100},
    {name = "Banana", price = 8.75,   qty = 250},
    {name = "Cherry", price = 150.00, qty = 50},
}

print(string.format("%-10s %8s %6s %10s", "Product", "Price", "Qty", "Total"))
print(string.rep("-", 38))

for _, p in ipairs(products) do
    local total = p.price * p.qty
    print(string.format("%-10s %8.2f %6d %10.2f",
        p.name, p.price, p.qty, total))
end

print(string.rep("-", 38))
-- Product      Price    Qty      Total
-- --------------------------------------
-- Apple        25.50    100    2550.00
-- Banana        8.75    250    2187.50
-- Cherry      150.00     50    7500.00
-- --------------------------------------
```

### ตัวอย่างที่ 15: format สำหรับการ debug

```lua
local function debugVar(name, value)
    local t = type(value)
    if t == "number" then
        if math.type(value) == "integer" then
            print(string.format("[DEBUG] %s (int) = %d", name, value))
        else
            print(string.format("[DEBUG] %s (float) = %.6f", name, value))
        end
    elseif t == "string" then
        print(string.format("[DEBUG] %s (str) = %q", name, value))
    elseif t == "boolean" then
        print(string.format("[DEBUG] %s (bool) = %s", name, tostring(value)))
    else
        print(string.format("[DEBUG] %s (%s)", name, t))
    end
end

debugVar("count", 42)          -- [DEBUG] count (int) = 42
debugVar("pi", 3.14159)        -- [DEBUG] pi (float) = 3.141590
debugVar("greeting", "hello")  -- [DEBUG] greeting (str) = "hello"
debugVar("flag", true)         -- [DEBUG] flag (bool) = true
```

---

## 4.8 string.byte() และ string.char()

### ตัวอย่างที่ 16: การแปลงระหว่าง String และ byte

```lua
-- string.byte(s, i, j) — ดึง byte values
print(string.byte("A"))          -- 65
print(string.byte("Hello", 1))   -- 72 (H)
print(string.byte("Hello", 2))   -- 101 (e)
print(string.byte("Hello", -1))  -- 111 (o) (ตัวสุดท้าย)

-- ดึงหลาย bytes พร้อมกัน
print(string.byte("Hello", 1, 5))  -- 72  101  108  108  111

-- string.char(...) — สร้าง string จาก byte values
print(string.char(72, 101, 108, 108, 111))  -- Hello
print(string.char(65, 66, 67))              -- ABC
print(string.char(0x4C, 0x75, 0x61))        -- Lua
```

### ตัวอย่างที่ 17: การแปลง string เป็น byte array

```lua
-- แสดง bytes ของ string
local function showBytes(s)
    local bytes = {string.byte(s, 1, #s)}
    local parts = {}
    for _, b in ipairs(bytes) do
        table.insert(parts, string.format("%02x", b))
    end
    return table.concat(parts, " ")
end

print(showBytes("Hello"))   -- 48 65 6c 6c 6f
print(showBytes("ABC"))     -- 41 42 43
print(showBytes("Lua"))     -- 4c 75 61
```

### ตัวอย่างที่ 18: Simple XOR encryption

```lua
local function xorEncrypt(s, key)
    local result = {}
    local keyLen = #key
    for i = 1, #s do
        local sb = string.byte(s, i)
        local kb = string.byte(key, ((i-1) % keyLen) + 1)
        table.insert(result, string.char(sb ~ kb))  -- XOR (Lua 5.3+)
    end
    return table.concat(result)
end

local original = "Hello, World!"
local key = "secret"
local encrypted = xorEncrypt(original, key)
local decrypted = xorEncrypt(encrypted, key)  -- XOR ซ้ำกลับคืน

print("Original: ", original)   -- Hello, World!
print("Decrypted:", decrypted)  -- Hello, World!
```

---

## 4.9 string.find() — การค้นหาใน String

### ตัวอย่างที่ 19: การใช้ find() พื้นฐาน

```lua
local str = "Hello, World! Hello, Lua!"

-- find(s, pattern) — คืนค่า start, end positions
local s, e = string.find(str, "World")
print(s, e)   -- 8  12

-- find() คืนค่า nil ถ้าไม่พบ
local pos = string.find(str, "Python")
print(pos)    -- nil

-- ค้นหาจาก position ที่กำหนด
s, e = string.find(str, "Hello", 2)   -- เริ่มจากตำแหน่ง 2
print(s, e)   -- 15  19

-- plain = true: ค้นหาตัวอักษรตรงๆ (ไม่ใช่ pattern)
s, e = string.find("a+b=c", "+", 1, true)
print(s, e)   -- 2  2
```

### ตัวอย่างที่ 20: การใช้ find() กับ pattern

```lua
local str = "The price is $42.50 today"

-- ค้นหาตัวเลข
local s, e = str:find("%d+%.%d+")
print(s, e)                    -- 15  19
print(str:sub(s, e))           -- 42.50

-- ค้นหา email format
local email = "contact: user@example.com please"
s, e = email:find("[%w%.]+@[%w%.]+%.[%a]+")
if s then
    print(email:sub(s, e))  -- user@example.com
end

-- ค้นหาทุก occurrence
local text = "cat bat rat hat"
local pos = 1
while true do
    local s2, e2 = text:find("[cbr]at", pos)
    if not s2 then break end
    print(text:sub(s2, e2))
    pos = e2 + 1
end
-- cat
-- bat
-- rat
```

---

## 4.10 string.match() — Pattern Matching

### ตัวอย่างที่ 21: การใช้ match() พื้นฐาน

```lua
-- match(s, pattern) — คืนค่า captures หรือ match
local str = "Today is 2024-01-15"

-- Match ทั้งหมด
print(str:match("%d+%-%d+%-%d+"))  -- 2024-01-15

-- Match พร้อม capture groups
local year, month, day = str:match("(%d+)-(%d+)-(%d+)")
print(year, month, day)  -- 2024  01  15

-- Pattern classes
print(("Hello123"):match("%a+"))   -- Hello (letters)
print(("Hello123"):match("%d+"))   -- 123   (digits)
print(("  hello  "):match("%S+"))  -- hello (non-space)
```

### ตัวอย่างที่ 22: Pattern classes ใน Lua

```lua
-- Pattern classes
-- %a  = letters (a-z, A-Z)
-- %d  = digits (0-9)
-- %l  = lowercase letters
-- %u  = uppercase letters
-- %s  = whitespace
-- %w  = alphanumeric
-- %p  = punctuation
-- %c  = control characters
-- %x  = hexadecimal digits
-- .   = any character (except newline)

local test = "Hello, World! 123"

print(test:match("%u%l+"))   -- Hello (uppercase then lowercase)
print(test:match("%d+"))     -- 123
print(test:match("[%a%s]+")) -- Hello  World  (letters and spaces)

-- Anchors
print(("Hello"):match("^H"))    -- H (match at start)
print(("Hello"):match("o$"))    -- o (match at end)
print(("Hello"):match("^Hello$"))  -- Hello (exact match)

-- Quantifiers
-- *  = zero or more
-- +  = one or more
-- ?  = zero or one
-- -  = zero or more (lazy/shortest)

print(("color"):match("colo[u]?r"))   -- color
print(("colour"):match("colo[u]?r"))  -- colour
```

### ตัวอย่างที่ 23: match() สำหรับ parsing

```lua
-- Parse HTTP status line
local status = "HTTP/1.1 200 OK"
local version, code, msg = status:match("HTTP/(%S+) (%d+) (.*)")
print(version, code, msg)  -- 1.1  200  OK

-- Parse key=value
local function parseKeyValue(s)
    local key, value = s:match("^([%w_]+)%s*=%s*(.+)$")
    return key, value
end

local k, v = parseKeyValue("name = John Doe")
print(k, v)  -- name  John Doe

k, v = parseKeyValue("age=25")
print(k, v)  -- age  25

-- Parse IP address
local ip = "192.168.1.100"
local a, b, c, d = ip:match("(%d+)%.(%d+)%.(%d+)%.(%d+)")
print(a, b, c, d)  -- 192  168  1  100
```

---

## 4.11 string.gmatch() — Global Matching / Iteration

### ตัวอย่างที่ 24: การ iterate ด้วย gmatch()

```lua
-- gmatch(s, pattern) — คืน iterator

-- วน loop ผ่านคำทุกคำ
local sentence = "The quick brown fox jumps"
for word in sentence:gmatch("%a+") do
    io.write(word .. " ")
end
print()  -- The quick brown fox jumps

-- วน loop ผ่านตัวเลข
local numbers = "1, 2, 3, 4, 5"
local sum = 0
for n in numbers:gmatch("%d+") do
    sum = sum + tonumber(n)
end
print("Sum:", sum)  -- Sum: 15

-- ดึง key-value pairs
local config = "host=localhost port=8080 debug=true"
for key, val in config:gmatch("(%w+)=(%w+)") do
    print(key, "->", val)
end
-- host     -> localhost
-- port     -> 8080
-- debug    -> true
```

### ตัวอย่างที่ 25: สร้าง word counter

```lua
local function countWords(text)
    local counts = {}
    for word in text:lower():gmatch("[%a']+") do
        counts[word] = (counts[word] or 0) + 1
    end
    return counts
end

local text = "to be or not to be that is the question to be"
local wc = countWords(text)

-- เรียงตาม count
local words = {}
for w, c in pairs(wc) do
    table.insert(words, {word = w, count = c})
end
table.sort(words, function(a, b) return a.count > b.count end)

for _, item in ipairs(words) do
    print(string.format("%-10s: %d", item.word, item.count))
end
-- be        : 3
-- to        : 3
-- or        : 1
-- ...
```

---

## 4.12 string.gsub() — Global Substitution

### ตัวอย่างที่ 26: การใช้ gsub() พื้นฐาน

```lua
-- gsub(s, pattern, repl [, n]) — แทนที่ทุก match
-- คืนค่า: new_string, count

local str = "Hello, World! Hello, Lua!"

-- แทนที่ด้วย string
local new, n = str:gsub("Hello", "Hi")
print(new, n)  -- Hi, World! Hi, Lua!  2

-- จำกัดจำนวนครั้ง
new, n = str:gsub("Hello", "Hi", 1)
print(new, n)  -- Hi, World! Hello, Lua!  1

-- ลบ pattern
new = ("  hello  world  "):gsub("%s+", " ")
print(new)  -- " hello world "

-- แทนที่ด้วย pattern capture
new = ("2024-01-15"):gsub("(%d+)-(%d+)-(%d+)", "%3/%2/%1")
print(new)  -- 15/01/2024
```

### ตัวอย่างที่ 27: gsub() พร้อม function replacement

```lua
-- ใช้ function แทน replacement string
local function toUpperWords(s)
    return s:gsub("(%a+)", function(w)
        return w:upper()
    end)
end
print(toUpperWords("hello world"))  -- HELLO WORLD

-- HTML escape
local function htmlEscape(s)
    local escapes = {
        ["&"] = "&amp;",
        ["<"] = "&lt;",
        [">"] = "&gt;",
        ['"'] = "&quot;",
        ["'"] = "&#39;",
    }
    return s:gsub('[&<>"\'']', escapes)
end

print(htmlEscape('<script>alert("xss")</script>'))
-- &lt;script&gt;alert(&quot;xss&quot;)&lt;/script&gt;
```

### ตัวอย่างที่ 28: gsub() กับ table replacement

```lua
-- ใช้ table เป็น replacement
local vars = {name = "John", age = "30", city = "Bangkok"}
local template = "Name: {name}, Age: {age}, City: {city}"

local result = template:gsub("{(%w+)}", vars)
print(result)
-- Name: John, Age: 30, City: Bangkok

-- Simple template engine
local function render(tmpl, data)
    return tmpl:gsub("{{(%w+)}}", function(key)
        return tostring(data[key] or "")
    end)
end

local t = "Hello, {{name}}! You have {{count}} messages."
print(render(t, {name = "Alice", count = 5}))
-- Hello, Alice! You have 5 messages.
```

---

## 4.13 Long Strings และ Multi-line Strings

### ตัวอย่างที่ 29: Long strings

```lua
-- Long string ปกติ (ระดับ 0)
local sql = [[
    SELECT *
    FROM users
    WHERE active = true
    ORDER BY name
]]
print(sql)

-- Long string ระดับ 1 (มี = หนึ่งตัว) — ใช้เมื่อ content มี ]]
local html = [=[
<div class="container">
    <p>Hello [[World]]</p>
</div>
]=]
print(html)

-- Long string ระดับ 2
local code = [==[
local t = [=[
    nested string
]=]
print(t)
]==]
print(code)
```

### ตัวอย่างที่ 30: Multi-line string tricks

```lua
-- newline แรกถูกตัดทิ้งอัตโนมัติ
local s1 = [[
line 1
line 2]]
print(#s1)  -- ไม่มี \n นำหน้า

-- เพิ่ม newline เอง
local s2 = "\n" .. [[
line 1
line 2]]

-- Heredoc style
local function trim(s)
    return s:match("^%s*(.-)%s*$")
end

local text = trim([[
    This text has
    leading spaces
    that will be trimmed
]])
print(text)
-- This text has
--     leading spaces
--     that will be trimmed
-- (trim เฉพาะ leading/trailing ของ string ทั้งก้อน)
```

---

## 4.14 String Comparison

### ตัวอย่างที่ 31: การเปรียบเทียบ String

```lua
-- String comparison ใช้ lexicographic order
print("abc" == "abc")   -- true
print("abc" == "ABC")   -- false (case sensitive)
print("abc" ~= "def")   -- true

print("abc" < "abd")    -- true  (c < d)
print("abc" < "abcd")   -- true  (prefix is shorter)
print("B" < "a")        -- true  (uppercase < lowercase in ASCII)
print("z" > "A")        -- true

-- เรียงลำดับ
local names = {"Charlie", "Alice", "Bob", "David"}
table.sort(names)
print(table.concat(names, ", "))
-- Alice, Bob, Charlie, David

-- เรียงแบบ case-insensitive
table.sort(names, function(a, b)
    return a:lower() < b:lower()
end)
print(table.concat(names, ", "))
```

---

## 4.15 String Interpolation Patterns

### ตัวอย่างที่ 32: Template strings

```lua
-- Pattern 1: ใช้ gsub
local function interpolate(s, t)
    return (s:gsub("$(%w+)", function(k)
        return tostring(t[k] or "$" .. k)
    end))
end

local tmpl = "Hello $name, you are $age years old!"
local result = interpolate(tmpl, {name = "Alice", age = 30})
print(result)  -- Hello Alice, you are 30 years old!

-- Pattern 2: ใช้ string.format
local name = "Bob"
local score = 95.5
local message = string.format("Player %s scored %.1f%%", name, score)
print(message)  -- Player Bob scored 95.5%

-- Pattern 3: concatenation
local greeting = "Hello, " .. name .. "! Score: " .. score
print(greeting)  -- Hello, Bob! Score: 95.5
```

---

## 4.16 Building Strings Efficiently

### ตัวอย่างที่ 33: table.concat vs .. concatenation

```lua
-- วิธีที่ไม่ดี: ใช้ .. ใน loop (สร้าง string ใหม่ทุกครั้ง)
local function buildBad(n)
    local result = ""
    for i = 1, n do
        result = result .. tostring(i) .. ","
    end
    return result
end

-- วิธีที่ดี: ใช้ table.concat (สร้างครั้งเดียว)
local function buildGood(n)
    local parts = {}
    for i = 1, n do
        parts[i] = tostring(i)
    end
    return table.concat(parts, ",")
end

-- เปรียบเทียบ
local t1 = os.clock()
buildBad(10000)
local t2 = os.clock()
buildGood(10000)
local t3 = os.clock()

print(string.format("Concat: %.4f sec", t2 - t1))
print(string.format("Join:   %.4f sec", t3 - t2))
-- Join: เร็วกว่า Concat มาก
```

### ตัวอย่างที่ 34: String builder pattern

```lua
-- String builder class
local StringBuilder = {}
StringBuilder.__index = StringBuilder

function StringBuilder.new()
    return setmetatable({parts = {}, size = 0}, StringBuilder)
end

function StringBuilder:append(s)
    s = tostring(s)
    self.parts[#self.parts + 1] = s
    self.size = self.size + #s
    return self  -- สำหรับ chaining
end

function StringBuilder:toString()
    return table.concat(self.parts)
end

function StringBuilder:len()
    return self.size
end

-- ใช้งาน
local sb = StringBuilder.new()
sb:append("Hello"):append(", "):append("World"):append("!")
print(sb:toString())  -- Hello, World!
print(sb:len())       -- 13
```

---

## 4.17 String Splitting

### ตัวอย่างที่ 35: การแยก String

```lua
-- Split ด้วย separator
local function split(s, sep)
    local result = {}
    local pattern = "([^" .. sep .. "]+)"
    for part in s:gmatch(pattern) do
        table.insert(result, part)
    end
    return result
end

local csv = "apple,banana,cherry,date"
local fruits = split(csv, ",")
for i, v in ipairs(fruits) do
    print(i, v)
end
-- 1  apple
-- 2  banana
-- 3  cherry
-- 4  date

-- Split แบบ preserve empty fields
local function splitAll(s, sep)
    local result = {}
    local pat = "([^" .. sep .. "]*)" .. sep .. "?"
    for field in s:gmatch(pat) do
        table.insert(result, field)
    end
    -- ลบ field สุดท้ายที่ว่าง (artifact ของ pattern)
    if result[#result] == "" then
        result[#result] = nil
    end
    return result
end

local line = "a,,b,,c"
local parts = splitAll(line, ",")
for i, v in ipairs(parts) do
    print(i, string.format("%q", v))
end
-- 1  "a"
-- 2  ""
-- 3  "b"
-- 4  ""
-- 5  "c"
```

---

## 4.18 Trim Functions

### ตัวอย่างที่ 36: การตัด whitespace

```lua
-- ltrim: ตัด whitespace ด้านซ้าย
local function ltrim(s)
    return s:match("^%s*(.*)")
end

-- rtrim: ตัด whitespace ด้านขวา
local function rtrim(s)
    return s:match("(.-)%s*$")
end

-- trim: ตัดทั้งสองด้าน
local function trim(s)
    return s:match("^%s*(.-)%s*$")
end

local test = "   Hello, World!   "
print(string.format("[%s]", ltrim(test)))  -- [Hello, World!   ]
print(string.format("[%s]", rtrim(test)))  -- [   Hello, World!]
print(string.format("[%s]", trim(test)))   -- [Hello, World!]

-- trim_chars: ตัด characters ที่กำหนด
local function trimChars(s, chars)
    local escaped = chars:gsub("([%^%]%-%%])", "%%%1")
    local pattern = "^[" .. escaped .. "]*(.-)[" .. escaped .. "]*$"
    return s:match(pattern)
end

print(trimChars("***hello***", "*"))    -- hello
print(trimChars("...test...", "."))     -- test
```

---

## 4.19 Unicode Basics

### ตัวอย่างที่ 37: Unicode และ UTF-8 ใน Lua

```lua
-- Lua string ทำงานกับ bytes
-- ภาษาไทยใช้ UTF-8: แต่ละตัวอักษรใช้ 3 bytes

local thai = "ก"
print(#thai)              -- 3 (3 bytes)
print(string.byte(thai, 1, 3))  -- 224 184 129

-- ตรวจสอบ bytes ของ string
local function utf8Bytes(s)
    local result = {}
    for i = 1, #s do
        table.insert(result, string.format("%02x", string.byte(s, i)))
    end
    return table.concat(result, " ")
end

print(utf8Bytes("A"))      -- 41 (1 byte)
print(utf8Bytes("€"))      -- e2 82 ac (3 bytes)
print(utf8Bytes("ก"))     -- e0 b8 81 (3 bytes)
print(utf8Bytes("😀"))    -- f0 9f 98 80 (4 bytes)

-- Lua 5.3+ มี utf8 library
if utf8 then
    print(utf8.len("Hello"))    -- 5
    print(utf8.len("สวัสดี"))  -- 6 (6 codepoints)
    
    -- วน loop ผ่าน codepoints
    for pos, code in utf8.codes("Hi!") do
        print(pos, code, string.char(code))
    end
    -- 1  72   H
    -- 2  105  i
    -- 3  33   !
end
```

### ตัวอย่างที่ 38: UTF-8 helper functions

```lua
-- นับจำนวน characters (codepoints) ของ UTF-8 string
local function utf8Len(s)
    if utf8 then
        return utf8.len(s)
    end
    -- Fallback: นับด้วย pattern
    local _, count = s:gsub("[%z\1-\127\194-\253][\128-\191]*", "")
    return count
end

print(utf8Len("Hello"))    -- 5
print(utf8Len("สวัสดี"))  -- 6

-- ดึง character ที่ตำแหน่ง n (UTF-8 safe)
local function utf8Char(s, n)
    if utf8 then
        local i = 1
        for pos, code in utf8.codes(s) do
            if i == n then
                return utf8.char(code)
            end
            i = i + 1
        end
        return nil
    end
end

if utf8 then
    print(utf8Char("สวัสดี", 1))  -- ส
    print(utf8Char("สวัสดี", 3))  -- า
end
```

---

## 4.20 String เป็น Byte Arrays

### ตัวอย่างที่ 39: การทำงานกับ string เป็น binary

```lua
-- Lua string สามารถเก็บ arbitrary bytes ได้
local binary = string.char(0x00, 0x01, 0x02, 0xFF, 0xFE)
print(#binary)  -- 5

-- ตรวจสอบ bytes
for i = 1, #binary do
    local b = string.byte(binary, i)
    io.write(string.format("0x%02X ", b))
end
print()  -- 0x00 0x01 0x02 0xFF 0xFE

-- Pack/unpack integers (Lua 5.3+ string.pack/unpack)
if string.pack then
    -- Big-endian 32-bit integer
    local packed = string.pack(">I4", 1234567890)
    print(#packed)  -- 4
    print(utf8Bytes and utf8Bytes(packed) or "no utf8Bytes")
    
    local n = string.unpack(">I4", packed)
    print(n)  -- 1234567890
    
    -- Little-endian
    local le = string.pack("<I4", 100)
    local v = string.unpack("<I4", le)
    print(v)  -- 100
end
```

### ตัวอย่างที่ 40: Base64 encoding (ตัวอย่างการเขียน)

```lua
local b64chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"

local function base64Encode(data)
    local result = {}
    local padding = 0
    
    -- Process 3 bytes at a time
    for i = 1, #data, 3 do
        local b1 = string.byte(data, i) or 0
        local b2 = string.byte(data, i + 1) or (padding < 2 and 0)
        local b3 = string.byte(data, i + 2) or (padding < 1 and 0)
        
        if i + 2 > #data then
            padding = 3 - (#data - i + 1)
        end
        
        local n = (b1 << 16) | ((b2 or 0) << 8) | (b3 or 0)
        
        table.insert(result, b64chars:sub((n >> 18) + 1, (n >> 18) + 1))
        table.insert(result, b64chars:sub(((n >> 12) & 63) + 1, ((n >> 12) & 63) + 1))
        
        if padding < 2 then
            table.insert(result, b64chars:sub(((n >> 6) & 63) + 1, ((n >> 6) & 63) + 1))
        else
            table.insert(result, "=")
        end
        
        if padding < 1 then
            table.insert(result, b64chars:sub((n & 63) + 1, (n & 63) + 1))
        else
            table.insert(result, "=")
        end
    end
    
    return table.concat(result)
end

print(base64Encode("Hello"))     -- SGVsbG8=
print(base64Encode("Man"))       -- TWFu
print(base64Encode("Lua"))       -- THVh
```

---

## 4.21 string.dump()

### ตัวอย่างที่ 41: การใช้ string.dump()

```lua
-- string.dump(f [, strip]) — แปลง function เป็น binary string
-- ใช้สำหรับ serializing functions

local function add(a, b)
    return a + b
end

-- dump function เป็น binary
local binary = string.dump(add)
print(type(binary))     -- string
print(#binary)          -- (จำนวน bytes, แตกต่างกันตาม implementation)

-- โหลด function กลับ
local addLoaded = load(binary)
print(addLoaded(3, 4))  -- 7
print(addLoaded(10, 20)) -- 30

-- strip = true: ลบ debug info ออก (ขนาดเล็กลง)
local stripped = string.dump(add, true)
print(#stripped <= #binary)  -- true (ขนาดเล็กลงหรือเท่ากัน)
```

---

## 4.22 Advanced String Patterns

### ตัวอย่างที่ 42: Balanced match

```lua
-- %b() — match balanced parentheses
local s = "func(a, b(c, d), e)"
local inner = s:match("%b()")
print(inner)  -- (a, b(c, d), e)

-- Match nested brackets
local json_like = '{"key": {"nested": "value"}}'
local obj = json_like:match("%b{}")
print(obj)  -- {"key": {"nested": "value"}}

-- Frontier pattern %f[] (Lua 5.1+)
local text = "THE END"
-- Match word boundaries
for word in text:gmatch("%f[%a]%a+") do
    print(word)
end
-- THE
-- END
```

### ตัวอย่างที่ 43: Advanced gsub patterns

```lua
-- Camel case to snake_case
local function toSnakeCase(s)
    s = s:gsub("(%u)(%u%l)", "%1_%2")
    s = s:gsub("(%l)(%u)", "%1_%2")
    return s:lower()
end

print(toSnakeCase("camelCase"))        -- camel_case
print(toSnakeCase("MyVariableName"))   -- my_variable_name
print(toSnakeCase("HTMLParser"))       -- html_parser

-- snake_case to camelCase
local function toCamelCase(s)
    return s:gsub("_(%l)", function(c)
        return c:upper()
    end)
end

print(toCamelCase("snake_case"))         -- snakeCase
print(toCamelCase("my_variable_name"))   -- myVariableName
```

---

## 4.23 String Performance Tips

### ตัวอย่างที่ 44: Best practices

```lua
-- 1. ใช้ local string functions
local format = string.format
local find = string.find
local sub = string.sub

-- เร็วกว่าการเรียก string.format ทุกครั้ง
for i = 1, 1000 do
    local s = format("item_%04d", i)
end

-- 2. Pre-compile patterns (ไม่มีใน standard Lua แต่ใช้ local)
local pattern = "%d+"
local text = "abc 123 def 456"
for n in text:gmatch(pattern) do
    -- ใช้ pattern ซ้ำ
end

-- 3. ใช้ table.concat แทน ..
local parts = {}
for i = 1, 100 do
    parts[i] = tostring(i)
end
local result = table.concat(parts, ", ")
print(result:sub(1, 20) .. "...")  -- 1, 2, 3, 4, 5, 6, 7, 8,...
```

---

## แบบฝึกหัดบทที่ 4

### แบบฝึกหัดที่ 1: String manipulation
เขียนฟังก์ชัน `titleCase(s)` ที่แปลง string เป็น Title Case
```
"hello world" → "Hello World"
"the quick brown fox" → "The Quick Brown Fox"
```

### แบบฝึกหัดที่ 2: Email validator
เขียนฟังก์ชัน `isValidEmail(s)` ที่ตรวจสอบว่า string เป็น email ที่ถูกต้องหรือไม่
(ต้องมี `@` คั่นระหว่าง local part และ domain)

### แบบฝึกหัดที่ 3: CSV parser
เขียนฟังก์ชัน `parseCSV(line)` ที่แยก CSV line เป็น array
```
"John,Doe,30,Bangkok" → {"John", "Doe", "30", "Bangkok"}
```

### แบบฝึกหัดที่ 4: Word wrap
เขียนฟังก์ชัน `wordWrap(text, width)` ที่ตัด text ให้แต่ละบรรทัดไม่เกิน `width` characters

### แบบฝึกหัดที่ 5: String compression (Run-length encoding)
เขียนฟังก์ชัน `rleEncode(s)` และ `rleDecode(s)` สำหรับ RLE compression
```
"AAABBBCCDDDDEE" → "3A3B2C4D2E"
"3A3B2C4D2E" → "AAABBBCCDDDDEE"
```

### เฉลยแบบฝึกหัด

```lua
-- เฉลย 1: titleCase
local function titleCase(s)
    return s:gsub("(%a)([%w_']*)", function(first, rest)
        return first:upper() .. rest:lower()
    end)
end
print(titleCase("hello world"))          -- Hello World
print(titleCase("the quick brown fox"))  -- The Quick Brown Fox

-- เฉลย 2: isValidEmail
local function isValidEmail(s)
    return s:match("^[%w%._%+%-]+@[%w%.%-]+%.[%a]{2,}$") ~= nil
end
print(isValidEmail("user@example.com"))    -- true
print(isValidEmail("user.name+tag@example.co.th"))  -- true
print(isValidEmail("invalid"))             -- false
print(isValidEmail("@domain.com"))         -- false

-- เฉลย 3: parseCSV
local function parseCSV(line)
    local result = {}
    for field in (line .. ","):gmatch("([^,]*),") do
        table.insert(result, field)
    end
    return result
end

local row = parseCSV("John,Doe,30,Bangkok")
for i, v in ipairs(row) do
    print(i, v)
end
-- 1  John
-- 2  Doe
-- 3  30
-- 4  Bangkok

-- เฉลย 4: wordWrap
local function wordWrap(text, width)
    local lines = {}
    local line = ""
    for word in text:gmatch("%S+") do
        if #line == 0 then
            line = word
        elseif #line + 1 + #word <= width then
            line = line .. " " .. word
        else
            table.insert(lines, line)
            line = word
        end
    end
    if #line > 0 then
        table.insert(lines, line)
    end
    return table.concat(lines, "\n")
end

local lorem = "The quick brown fox jumps over the lazy dog"
print(wordWrap(lorem, 20))
-- The quick brown fox
-- jumps over the lazy
-- dog

-- เฉลย 5: RLE
local function rleEncode(s)
    return s:gsub("(.)%1*", function(c)
        local run = s:match(c:gsub("([%^%]%-%%%(%)%+%*%.%?])", "%%%1") .. "+", s:find(c, 1, true))
        local count = #run
        return count > 1 and count .. c or c
    end)
end

-- วิธีที่ชัดเจนกว่า
local function rleEncode2(s)
    local result = {}
    local i = 1
    while i <= #s do
        local c = s:sub(i, i)
        local j = i
        while j <= #s and s:sub(j, j) == c do
            j = j + 1
        end
        local count = j - i
        if count > 1 then
            table.insert(result, count .. c)
        else
            table.insert(result, c)
        end
        i = j
    end
    return table.concat(result)
end

local function rleDecode(s)
    return s:gsub("(%d+)(.)", function(n, c)
        return c:rep(tonumber(n))
    end)
end

print(rleEncode2("AAABBBCCDDDDEE"))  -- 3A3B2C4D2E
print(rleDecode("3A3B2C4D2E"))       -- AAABBBCCDDDDEE
```

---

## สรุปบทที่ 4

| Function | รายละเอียด |
|----------|------------|
| `string.len(s)` / `#s` | ความยาว (bytes) |
| `string.upper(s)` | แปลงเป็นตัวพิมพ์ใหญ่ |
| `string.lower(s)` | แปลงเป็นตัวพิมพ์เล็ก |
| `string.sub(s, i, j)` | ดึง substring |
| `string.rep(s, n, sep)` | ทำซ้ำ string |
| `string.reverse(s)` | กลับ string |
| `string.format(fmt, ...)` | จัดรูปแบบ string |
| `string.byte(s, i, j)` | แปลง string → bytes |
| `string.char(...)` | แปลง bytes → string |
| `string.find(s, pat, i, plain)` | ค้นหา pattern |
| `string.match(s, pat, i)` | Match pattern |
| `string.gmatch(s, pat)` | Iterator สำหรับ pattern |
| `string.gsub(s, pat, repl, n)` | แทนที่ pattern |
| `string.dump(f, strip)` | Serialize function |

**หลักการสำคัญ:**
- String ใน Lua เป็น immutable
- `#` นับ bytes ไม่ใช่ characters (สำคัญสำหรับ UTF-8)
- ใช้ `table.concat` แทน `..` ใน loop สำหรับ performance
- Pattern ใน Lua ไม่ใช่ regex แต่คล้ายกัน
