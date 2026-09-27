# บทที่ 12: Pattern Matching และ Regular Expressions

## สารบัญ
1. [บทนำ Pattern Matching](#บทนำ)
2. [Lua Patterns vs POSIX Regex](#lua-patterns-vs-posix-regex)
3. [Pattern Classes](#pattern-classes)
4. [Quantifiers](#quantifiers)
5. [Anchors](#anchors)
6. [Character Sets](#character-sets)
7. [Captures](#captures)
8. [string.find()](#stringfind)
9. [string.match()](#stringmatch)
10. [string.gmatch()](#stringgmatch)
11. [string.gsub()](#stringgsub)
12. [Balanced Match %b](#balanced-match-b)
13. [Frontier Pattern %f](#frontier-pattern-f)
14. [Named Captures Pattern](#named-captures-pattern)
15. [Parsing Structured Text](#parsing-structured-text)
16. [URL Parser](#url-parser)
17. [Date Parser](#date-parser)
18. [CSV Parser with Patterns](#csv-parser-with-patterns)
19. [Template Substitution](#template-substitution)
20. [Tokenizer](#tokenizer)

---

## บทนำ

Lua ไม่ใช้ POSIX regex มาตรฐาน แต่มีระบบ pattern matching ของตัวเอง ที่เบากว่าและเพียงพอสำหรับงานส่วนใหญ่ Pattern ใน Lua เป็น "magic patterns" ที่มีความสามารถเฉพาะตัว

```lua
-- ตัวอย่างที่ 1: Pattern Matching functions ใน Lua
print("=== String Pattern Functions ===")
print("string.find()   - ค้นหาและคืน start, end position")
print("string.match()  - ค้นหาและคืน matched string")
print("string.gmatch() - iterator สำหรับค้นหาทั้งหมด")
print("string.gsub()   - แทนที่ด้วย pattern")
print("")
print("สามารถเรียกได้สองแบบ:")
print("  string.find(s, pattern) หรือ")
print('  s:find(pattern)         (method syntax)')
```

```lua
-- ตัวอย่างที่ 2: ทำไม Lua ใช้ patterns แทน regex
-- Lua patterns มีขนาดเล็กกว่าและ embed ง่ายกว่า
-- Lua source: ประมาณ 500 บรรทัดสำหรับ pattern engine
-- POSIX regex: ใหญ่กว่ามาก

-- ข้อจำกัดของ Lua patterns:
-- 1. ไม่มี alternation (|) อย่าง regex
-- 2. ไม่มี backreferences
-- 3. ไม่มี lookahead/lookbehind
-- แต่มี:
-- 1. %b สำหรับ balanced matching
-- 2. %f สำหรับ frontier patterns
-- 3. Captures ด้วย ()

local text = "Hello, World! 42 items at $15.99"
print("Text:", text)
print("Find 'World':", string.find(text, "World"))
print("Match number:", string.match(text, "%d+"))
print("Match price:", string.match(text, "%$(%d+%.%d+)"))
```

---

## Lua Patterns vs POSIX Regex

```lua
-- ตัวอย่างที่ 3: เปรียบเทียบ Lua patterns กับ POSIX regex

-- POSIX regex -> Lua pattern
local comparisons = {
    {".", ".", "จุดทุกอักขระ (ยกเว้น newline ใน POSIX)"},
    {"[a-z]", "%l", "ตัวอักษรพิมพ์เล็ก"},
    {"[A-Z]", "%u", "ตัวอักษรพิมพ์ใหญ่"},
    {"[0-9]", "%d", "ตัวเลข"},
    {"[a-zA-Z]", "%a", "ตัวอักษร"},
    {"[a-zA-Z0-9]", "%w", "ตัวอักษรหรือตัวเลข"},
    {"\\s", "%s", "whitespace"},
    {"\\p", "%p", "punctuation"},
    {"[^\\s]", "%S", "non-whitespace (uppercase class)"},
    {"^", "^", "ต้นบรรทัด"},
    {"$", "$", "ท้ายบรรทัด"},
    {"*", "*", "0 หรือมากกว่า (greedy)"},
    {"+", "+", "1 หรือมากกว่า (greedy)"},
    {"?", "?", "0 หรือ 1"},
    {"*? (lazy)", "-", "0 หรือมากกว่า (lazy/shortest)"},
    {"(...)", "(...)", "capture group"},
}

print(string.format("%-20s %-15s %s", "POSIX", "Lua Pattern", "ความหมาย"))
print(string.rep("-", 60))
for _, row in ipairs(comparisons) do
    print(string.format("%-20s %-15s %s", row[1], row[2], row[3]))
end
```

---

## Pattern Classes

```lua
-- ตัวอย่างที่ 4: Character classes พื้นฐาน
local test_cases = {
    {"%a", "Hello123 World!", "ตัวอักษร (a-z, A-Z)"},
    {"%A", "Hello123 World!", "NON-ตัวอักษร"},
    {"%d", "Phone: 02-123-4567", "ตัวเลข"},
    {"%D", "Phone: 02-123-4567", "NON-ตัวเลข"},
    {"%l", "Hello World", "ตัวพิมพ์เล็ก"},
    {"%u", "Hello World", "ตัวพิมพ์ใหญ่"},
    {"%s", "Hello World\tTab", "whitespace"},
    {"%S", "Hello World", "NON-whitespace"},
    {"%p", "Hello, World!", "punctuation"},
    {"%w", "Hello_123", "alphanumeric"},
    {"%x", "Color: #FF5733", "hexadecimal"},
    {"%c", "A\0B\nC\t", "control characters"},
}

print("=== Pattern Classes ===")
for _, tc in ipairs(test_cases) do
    local pattern, text, desc = tc[1], tc[2], tc[3]
    local matches = {}
    for m in text:gmatch(pattern) do
        table.insert(matches, m == " " and "SPACE" or 
                              m == "\t" and "TAB" or
                              m == "\n" and "NL" or
                              m == "\0" and "NULL" or m)
    end
    print(string.format("  %-5s : %-30s -> [%s]", 
          pattern, "'" .. text:sub(1, 20) .. "'", 
          table.concat(matches, "|"):sub(1, 40)))
end
```

```lua
-- ตัวอย่างที่ 5: Magic characters ที่ต้อง escape
-- Magic chars: ( ) . % + - * ? [ ^ $
-- ใช้ % นำหน้าเพื่อ escape

local function escape_pattern(s)
    return s:gsub("([%(%)%.%%%+%-%*%?%[%^%$])", "%%%1")
end

local magic_chars = {".", "*", "+", "?", "-", "^", "$", "(", ")", "[", "%"}

print("=== Magic Characters ===")
for _, c in ipairs(magic_chars) do
    print(string.format("  '%s' -> escape เป็น '%%%s'", c, c))
end

-- ทดสอบการ escape
local text = "Price: $15.99 (tax included)"
local search = "$15.99"  -- มี magic chars
local escaped = escape_pattern(search)
print("\nค้นหา '" .. search .. "' ใน '" .. text .. "'")
print("Escaped pattern:", escaped)
print("Found:", text:find(escaped) ~= nil)
```

---

## Quantifiers

```lua
-- ตัวอย่างที่ 6: Quantifiers
local text = "aaabbbcccc"

print("=== Quantifiers ===")
-- * = 0 หรือมากกว่า (greedy)
print("a*:", text:match("a*"))      -- "aaa"
print("x*:", text:match("x*"))      -- "" (0 matches OK)

-- + = 1 หรือมากกว่า (greedy)
print("a+:", text:match("a+"))      -- "aaa"
print("x+:", tostring(text:match("x+")))  -- nil (ต้องมีอย่างน้อย 1)

-- ? = 0 หรือ 1
print("a?:", text:match("a?"))      -- "a" (1 match)
print("x?:", text:match("x?"))      -- "" (0 match OK)

-- - = 0 หรือมากกว่า (lazy/shortest)
print("a-:", text:match("a-"))      -- "" (lazy, matches minimum)
```

```lua
-- ตัวอย่างที่ 7: Greedy vs Lazy matching
local html = "<b>bold</b> and <i>italic</i>"

-- Greedy: จับ string ยาวที่สุด
local greedy = html:match("<(.+)>")
print("=== Greedy vs Lazy ===")
print("Greedy <(.+)>:", greedy)   -- "b>bold</b> and <i>italic</i" (ยาวสุด)

-- Lazy: จับ string สั้นที่สุด
local lazy = html:match("<(.-)>")
print("Lazy   <(.-)>:", lazy)     -- "b" (สั้นสุด)

-- ตัวอย่างจริง: extract HTML tags
print("\nAll tags:")
for tag in html:gmatch("<([^>]+)>") do
    print("  Tag:", tag)
end
```

```lua
-- ตัวอย่างที่ 8: Quantifiers กับ character classes
local text = "  Hello   World   Lua  "

-- จับคำ
for word in text:gmatch("%S+") do
    io.write("[" .. word .. "] ")
end
print()

-- จับตัวเลข
local data = "x=10, y=200, z=3000"
for num in data:gmatch("%d+") do
    io.write(num .. " ")
end
print()

-- จับ identifier (ขึ้นต้นด้วยตัวอักษร/_, ตามด้วย alphanumeric/_)
local code = "local my_var = func_name(arg1, _private)"
for ident in code:gmatch("[%a_][%w_]*") do
    io.write(ident .. " ")
end
print()
```

---

## Anchors

```lua
-- ตัวอย่างที่ 9: Anchors ^ และ $
local lines = {
    "Hello World",
    "  Hello World",
    "Hello World  ",
    "World Hello",
}

print("=== Anchors ===")
for _, line in ipairs(lines) do
    local starts_hello = line:match("^Hello") ~= nil
    local ends_world = line:match("World$") ~= nil
    print(string.format("  '%-20s' start=%-5s end=%s", 
          line, starts_hello, ends_world))
end
```

```lua
-- ตัวอย่างที่ 10: ตรวจสอบ string ทั้งหมดด้วย ^ และ $
local function is_integer(s)
    return s:match("^%-?%d+$") ~= nil
end

local function is_float(s)
    return s:match("^%-?%d+%.?%d*$") ~= nil
end

local function is_identifier(s)
    return s:match("^[%a_][%w_]*$") ~= nil
end

local function is_email_simple(s)
    return s:match("^[%w%.]+@[%w%.]+%.[%a]+$") ~= nil
end

local tests = {
    {is_integer, {"42", "-5", "3.14", "abc", ""}},
    {is_float, {"3.14", "-2.5", "100", "abc", ".5"}},
    {is_identifier, {"hello", "_var", "my_func", "123abc", ""}},
    {is_email_simple, {"user@example.com", "bad.email", "a@b.c", "no@"}},
}

local names = {"is_integer", "is_float", "is_identifier", "is_email"}
print("=== Validation ===")
for i, test in ipairs(tests) do
    print("\n" .. names[i] .. "():")
    local func = test[1]
    for _, val in ipairs(test[2]) do
        print(string.format("  '%-20s' -> %s", val, func(val)))
    end
end
```

---

## Character Sets

```lua
-- ตัวอย่างที่ 11: Custom character sets [...]
local text = "Hello, World! 12345"

-- [abc] = a หรือ b หรือ c
print("[aeiou]:", text:match("[aeiou]+"))

-- [a-z] = range
print("[a-z]+:", text:match("[a-z]+"))

-- [^abc] = NOT a, b, หรือ c
print("[^%a%s]+:", text:match("[^%a%s]+"))  -- ไม่ใช่ตัวอักษรหรือ space

-- รวมหลาย range
print("[a-zA-Z]+:", text:match("[a-zA-Z]+"))
print("[a-z0-9]+:", text:match("[a-z0-9]+"))
```

```lua
-- ตัวอย่างที่ 12: ตรวจสอบข้อมูลด้วย character sets
local function validate_date(s)
    -- รูปแบบ DD/MM/YYYY หรือ DD-MM-YYYY
    return s:match("^%d%d[%-%/]%d%d[%-%/]%d%d%d%d$") ~= nil
end

local function validate_ip(s)
    local function valid_octet(n)
        n = tonumber(n)
        return n and n >= 0 and n <= 255
    end
    
    local a, b, c, d = s:match("^(%d+)%.(%d+)%.(%d+)%.(%d+)$")
    return a and valid_octet(a) and valid_octet(b) and 
           valid_octet(c) and valid_octet(d)
end

local function validate_phone_th(s)
    -- เบอร์โทรศัพท์ไทย: 0X-XXXX-XXXX หรือ 0XXXXXXXXX
    s = s:gsub("[%-%s]", "")  -- ลบ - และ space
    return s:match("^0[689]%d%d%d%d%d%d%d%d$") ~= nil or
           s:match("^0[23457]%d%d%d%d%d%d%d$") ~= nil
end

print("=== Input Validation ===")
local dates = {"25/12/2024", "25-12-2024", "2024/12/25", "32/13/2024"}
for _, d in ipairs(dates) do
    print(string.format("  Date %-15s : %s", d, validate_date(d)))
end

local ips = {"192.168.1.1", "10.0.0.256", "not.an.ip", "0.0.0.0"}
for _, ip in ipairs(ips) do
    print(string.format("  IP   %-20s : %s", ip, validate_ip(ip)))
end

local phones = {"081-234-5678", "0812345678", "02-123-4567", "0999999"}
for _, p in ipairs(phones) do
    print(string.format("  Phone %-15s : %s", p, validate_phone_th(p)))
end
```

---

## Captures

```lua
-- ตัวอย่างที่ 13: Captures พื้นฐาน
local date_str = "2024-12-25"

-- capture เดียว
local year = date_str:match("(%d%d%d%d)")
print("Year:", year)

-- หลาย captures
local y, m, d = date_str:match("(%d%d%d%d)-(%d%d)-(%d%d)")
print("Year:", y, "Month:", m, "Day:", d)

-- Capture กับ position (ใน string.find)
local s, e, cap = string.find("Hello World", "(%a+)%s")
print("Find with capture:", s, e, cap)
```

```lua
-- ตัวอย่างที่ 14: Captures ใน gsub
local text = "John Smith, Jane Doe, Bob Wilson"

-- สลับ first และ last name
local swapped = text:gsub("(%a+)%s(%a+)", "%2, %1")
print("Swapped:", swapped)

-- ใส่วงเล็บรอบตัวเลข
local code = "item_1 cost 100 baht and item_2 cost 200 baht"
local bracketed = code:gsub("(%d+)", "[%1]")
print("Bracketed:", bracketed)
```

```lua
-- ตัวอย่างที่ 15: Nested captures
local text = "2024-12-25 14:30:45"

-- Capture ซ้อนกัน
local datetime, date_part, time_part = 
    text:match("((%d%d%d%d%-%d%d%-%d%d) (%d%d:%d%d:%d%d))")

print("Full datetime:", datetime)
print("Date part:", date_part)
print("Time part:", time_part)
```

```lua
-- ตัวอย่างที่ 16: Position captures ()
-- () capture position ไม่ใช่ string
local text = "Hello World Lua"

local positions = {}
for pos in text:gmatch("()%a+") do
    table.insert(positions, pos)
end
print("Word positions:", table.concat(positions, ", "))

-- หา position ของทุก space
for pos in text:gmatch("()%s") do
    print("Space at position:", pos)
end
```

```lua
-- ตัวอย่างที่ 17: Multiple captures กับ gmatch
local data = "name=Alice;age=30;city=Bangkok;job=Developer"

print("=== Multiple Captures in gmatch ===")
for key, value in data:gmatch("([^=;]+)=([^;]+)") do
    print(string.format("  %-10s = %s", key, value))
end
```

```lua
-- ตัวอย่างที่ 18: Capture ทั้งหมด vs ส่วนที่ต้องการ
local html = '<a href="https://example.com" class="link">Click here</a>'

-- Capture URL เท่านั้น
local url = html:match('href="([^"]+)"')
print("URL:", url)

-- Capture หลาย attributes
local href, class = html:match('href="([^"]+)"%s+class="([^"]+)"')
print("href:", href)
print("class:", class)

-- Capture link text
local link_text = html:match(">([^<]+)<")
print("Text:", link_text)
```

---

## string.find()

```lua
-- ตัวอย่างที่ 19: string.find() รายละเอียด
local text = "The quick brown fox jumps over the lazy dog"

-- string.find(s, pattern, init, plain)
-- คืนค่า: start_pos, end_pos [, captures...]
-- init: เริ่มค้นหาจาก position นี้
-- plain: true = ค้นหาตามตัวอักษรจริง (no pattern)

local s, e = text:find("fox")
print("Find 'fox':", s, e)  -- 17, 19
print("Substring:", text:sub(s, e))

-- ค้นหาจาก position ที่กำหนด
s, e = text:find("the", 20)  -- ค้นจาก pos 20
print("Find 'the' from 20:", s, e)  -- lowercase 'the' only

-- ค้นหาแบบ plain (ปิด pattern)
s, e = text:find("o", 1, true)  -- plain search
print("Plain find 'o':", s, e)

-- ค้นหาด้วย pattern
s, e = text:find("%a+")  -- หา word แรก
print("First word:", text:sub(s, e))
```

```lua
-- ตัวอย่างที่ 20: วนหาทุก occurrence ด้วย string.find
local text = "the cat and the dog and the bird"
local search = "the"
local count = 0
local pos = 1

print("=== Find all occurrences ===")
while true do
    local s, e = text:find(search, pos, true)
    if not s then break end
    count = count + 1
    print(string.format("  Found '%s' at positions %d-%d", search, s, e))
    pos = e + 1
end
print("Total:", count)
```

```lua
-- ตัวอย่างที่ 21: string.find() กับ captures
local log_line = "[2024-12-25 14:30:45] ERROR: Connection failed"

local s, e, level, message = 
    log_line:find("%[%d%d%d%d%-%d%d%-%d%d %d%d:%d%d:%d%d%] (%u+): (.+)")

if s then
    print("Log level:", level)
    print("Message:", message)
end
```

---

## string.match()

```lua
-- ตัวอย่างที่ 22: string.match() รายละเอียด
-- คืน captures ถ้ามี ไม่งั้นคืน matched string ทั้งหมด
-- คืน nil ถ้าไม่พบ

local text = "Today is 2024-12-25"

-- ไม่มี capture - คืน matched string
local date = text:match("%d%d%d%d%-%d%d%-%d%d")
print("Date:", date)

-- มี captures - คืน captures
local y, m, d = text:match("(%d%d%d%d)-(%d%d)-(%d%d)")
print("Year:", y, "Month:", m, "Day:", d)

-- ไม่พบ
local result = text:match("%d%d%d%d%d")
print("5-digit number:", tostring(result))  -- nil
```

```lua
-- ตัวอย่างที่ 23: ดึงข้อมูลจาก string
-- Parse HTTP response header
local response = [[HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 1234
Server: nginx/1.21.0
X-Request-Id: abc-123-def]]

-- ดึง status code
local version, status_code, status_text = 
    response:match("HTTP/(%d+%.%d+) (%d+) (.+)\n")
print("Version:", version)
print("Status:", status_code, status_text)

-- ดึง header ที่กำหนด
local function get_header(headers, name)
    return headers:match(name .. ":%s*([^\n]+)")
end

print("Content-Type:", get_header(response, "Content%-Type"))
print("Server:", get_header(response, "Server"))
print("X-Request-Id:", get_header(response, "X%-Request%-Id"))
```

```lua
-- ตัวอย่างที่ 24: match กับ init parameter
local text = "ID:001 ID:002 ID:003"

-- match แรก
print("First match:", text:match("ID:(%d+)"))  -- 001

-- match จาก position ที่กำหนด (ผ่าน find+match)
local pos = 1
print("All matches:")
while true do
    local s, e, id = text:find("ID:(%d+)", pos)
    if not s then break end
    print("  ID:", id)
    pos = e + 1
end
```

---

## string.gmatch()

```lua
-- ตัวอย่างที่ 25: string.gmatch() Iterator
local text = "one two three four five"

-- วนหา words
print("=== gmatch words ===")
for word in text:gmatch("%a+") do
    io.write(word .. " ")
end
print()

-- วนหา numbers
local numbers_str = "a1 b2 c3 d4 e5"
print("Numbers:")
for n in numbers_str:gmatch("%d+") do
    io.write(n .. " ")
end
print()
```

```lua
-- ตัวอย่างที่ 26: gmatch กับ captures
local data = "Alice:90, Bob:85, Charlie:92, Diana:78"

print("=== gmatch with captures ===")
local total, count = 0, 0
for name, score in data:gmatch("(%a+):(%d+)") do
    score = tonumber(score)
    print(string.format("  %-10s : %d", name, score))
    total = total + score
    count = count + 1
end
print(string.format("Average: %.1f", total/count))
```

```lua
-- ตัวอย่างที่ 27: gmatch สำหรับ tokenizer
local function tokenize(text)
    local tokens = {}
    for token in text:gmatch("[^%s,;]+") do
        table.insert(tokens, token)
    end
    return tokens
end

local input = "hello, world; lua is great"
local tokens = tokenize(input)
print("=== Tokenizer ===")
for i, t in ipairs(tokens) do
    print(string.format("  [%d] '%s'", i, t))
end
```

```lua
-- ตัวอย่างที่ 28: gmatch สำหรับ parsing config
local config_str = [[
# Server config
host = localhost
port = 8080
debug = true
name = "My Server"
]]

local config = {}
for key, value in config_str:gmatch("\n([%w_]+)%s*=%s*(.-)%s*\n") do
    -- ลบ quotes ถ้ามี
    value = value:match('^"(.-)"$') or value
    config[key] = value
end

print("=== Config Parse ===")
for k, v in pairs(config) do
    print(string.format("  %-10s = %s", k, v))
end
```

```lua
-- ตัวอย่างที่ 29: gmatch สร้าง table จาก string
local function split(s, sep)
    sep = sep or "%s"
    local parts = {}
    local pattern = string.format("([^%s]+)", sep)
    for part in s:gmatch(pattern) do
        table.insert(parts, part)
    end
    return parts
end

local function split_any(s, seps)
    local parts = {}
    local pattern = "[^" .. seps .. "]+"
    for part in s:gmatch(pattern) do
        table.insert(parts, part)
    end
    return parts
end

-- ทดสอบ
print("=== Split ===")
local csv = "a,b,c,d,e"
print("Split by ',':", table.concat(split(csv, ","), " | "))

local path = "/home/user/documents/file.txt"
print("Split path:", table.concat(split(path, "/"), " -> "))

local text = "hello.world,test;data"
print("Split multi:", table.concat(split_any(text, ".,;"), " | "))
```

---

## string.gsub()

```lua
-- ตัวอย่างที่ 30: string.gsub() พื้นฐาน
-- string.gsub(s, pattern, repl [, n])
-- คืน: new_string, count

-- แทนที่ด้วย string
local text = "Hello World Hello Lua"
local result, count = text:gsub("Hello", "Hi")
print("gsub:", result, "(replaced:", count, "times)")

-- จำกัดจำนวนครั้ง (n=1)
result, count = text:gsub("Hello", "Hi", 1)
print("gsub n=1:", result, "(replaced:", count, "times)")

-- ลบ pattern
local no_vowels = ("Hello World"):gsub("[aeiouAEIOU]", "")
print("No vowels:", no_vowels)
```

```lua
-- ตัวอย่างที่ 31: gsub กับ captures (%1, %2, ...)
local text = "John Smith and Jane Doe"

-- สลับ first/last name
local swapped = text:gsub("(%a+) (%a+)", "%2 %1")
print("Swapped:", swapped)

-- เพิ่ม prefix
local prefixed = text:gsub("(%a+)", "Mr/Ms. %1")
print("Prefixed:", prefixed)

-- ห่อ HTML
local html = text:gsub("(%a+)", "<span>%1</span>")
print("HTML:", html)
```

```lua
-- ตัวอย่างที่ 32: gsub กับ function
-- function รับ captures แล้วคืน replacement string
local text = "price: $100 and $200 and $50"

-- แปลง price เป็น THB
local result = text:gsub("%$(%d+)", function(amount)
    return amount .. " USD (" .. tonumber(amount) * 35 .. " THB)"
end)
print("Converted:", result)

-- ทำให้ตัวแรกเป็นพิมพ์ใหญ่
local sentence = "hello world lua programming"
local capitalized = sentence:gsub("(%a)([%a]*)", function(first, rest)
    return first:upper() .. rest
end)
print("Capitalized:", capitalized)
```

```lua
-- ตัวอย่างที่ 33: gsub กับ table (lookup replacement)
-- ถ้า replacement เป็น table, ใช้ capture เป็น key
local abbreviations = {
    HTML = "HyperText Markup Language",
    CSS  = "Cascading Style Sheets",
    JS   = "JavaScript",
    API  = "Application Programming Interface",
    URL  = "Uniform Resource Locator",
}

local text = "Build an HTML page with CSS and JS to call an API via URL"
local expanded = text:gsub("%u+", abbreviations)
print("Expanded:", expanded)
```

```lua
-- ตัวอย่างที่ 34: gsub สำหรับ template engine
local function render_template(template, vars)
    return (template:gsub("{{(%w+)}}", function(key)
        return tostring(vars[key] or "{{" .. key .. "}}")
    end))
end

local template = [[
Hello, {{name}}!
Your account: {{email}}
Balance: {{balance}} THB
Status: {{status}}
Unknown: {{missing}}
]]

local vars = {
    name = "สมชาย ใจดี",
    email = "somchai@example.com",
    balance = "1500.00",
    status = "Active",
}

print("=== Template Rendering ===")
print(render_template(template, vars))
```

```lua
-- ตัวอย่างที่ 35: gsub สำหรับ sanitize input
local function sanitize_html(text)
    return text
        :gsub("&", "&amp;")
        :gsub("<", "&lt;")
        :gsub(">", "&gt;")
        :gsub('"', "&quot;")
        :gsub("'", "&#039;")
end

local function sanitize_sql_like(text)
    -- เฉพาะสำหรับ LIKE pattern (ไม่ใช่ SQL injection prevention จริงๆ)
    return text:gsub("([%%_])", "\\%1")
end

local function trim(s)
    return s:match("^%s*(.-)%s*$")
end

local function trim_all(s)
    -- ลบ whitespace ทั้งหมด
    return s:gsub("%s+", "")
end

-- ทดสอบ
print("=== Sanitize ===")
local html_input = '<script>alert("XSS")</script>'
print("Original:", html_input)
print("Sanitized:", sanitize_html(html_input))

print("\nTrim:")
print("'" .. trim("  hello world  ") .. "'")
print("'" .. trim_all("  hello world  ") .. "'")
```

---

## Balanced Match %b

```lua
-- ตัวอย่างที่ 36: %b สำหรับ balanced matching
-- %bxy: match เริ่มด้วย x และจบด้วย y (balanced)

local text = "func(arg1, (nested), arg3)"

-- จับ parentheses แบบ balanced
local content = text:match("%b()")
print("Balanced ():", content)

-- จับ brackets
local json = '{"key": "value", "nested": {"a": 1}}'
local obj = json:match("%b{}")
print("Balanced {}:", obj)

-- จับ square brackets
local array_str = "[1, [2, 3], [4, [5, 6]]]"
local arr = array_str:match("%b[]")
print("Balanced []:", arr)
```

```lua
-- ตัวอย่างที่ 37: %b สำหรับ parsing code
local function extract_function_body(code)
    -- หา function body ใน {}
    local body = code:match("function%s+%w+%s*%(.-%)%s*(%b{})")
    return body
end

local code = [[
function hello(name) {
    if (name) {
        return "Hello, " + name;
    }
    return "Hello, World";
}
]]

local body = extract_function_body(code)
print("=== Function Body ===")
print(body)
```

```lua
-- ตัวอย่างที่ 38: %b สำหรับ extract comments
local function remove_block_comments(code)
    -- ลบ /* ... */ comments
    return code:gsub("/%*.-%*/", "")
end

local function extract_strings(code)
    local strings = {}
    for s in code:gmatch('"[^"]*"') do
        table.insert(strings, s)
    end
    return strings
end

local c_code = [[
/* This is a comment */
int main() {
    printf("Hello"); /* inline comment */
    char *name = "World";
    return 0; /* final comment */
}
]]

print("=== Remove Comments ===")
print(remove_block_comments(c_code))

print("=== Extract Strings ===")
local strings = extract_strings(c_code)
for _, s in ipairs(strings) do
    print("  " .. s)
end
```

---

## Frontier Pattern %f

```lua
-- ตัวอย่างที่ 39: %f Frontier Pattern
-- %f[set] matches empty string at transition เข้า/ออกจาก character class

local text = "Hello World Lua"

-- จับ word boundaries (ขอบของคำ)
-- %f[%a] = frontier ก่อนตัวอักษร (เริ่มคำ)
for word in text:gmatch("%f[%a]%a+") do
    io.write("[" .. word .. "]")
end
print()
```

```lua
-- ตัวอย่างที่ 40: %f สำหรับ word boundary replacement
local text = "cat concatenate category catfish"

-- แทนที่ "cat" เฉพาะที่เป็น whole word
-- ไม่ใช้ %f:
local with_simple = text:gsub("cat", "DOG")
print("Without frontier:", with_simple)  -- เปลี่ยนทั้งหมด!

-- ใช้ %f:
local with_frontier = text:gsub("%f[%a]cat%f[%A]", "DOG")
print("With frontier:", with_frontier)  -- เปลี่ยนเฉพาะ "cat" เดี่ยวๆ
```

```lua
-- ตัวอย่างที่ 41: %f สำหรับ highlight keywords
local function highlight_keywords(code, keywords)
    local kw_set = {}
    for _, kw in ipairs(keywords) do kw_set[kw] = true end
    
    return code:gsub("%f[%a_][%a_][%w_]*%f[%W]", function(word)
        if kw_set[word] then
            return "**" .. word .. "**"
        end
        return word
    end)
end

local code = "local function hello(name) return name end"
local keywords = {"local", "function", "return", "end", "if", "then"}
print("=== Highlight Keywords ===")
print(highlight_keywords(code, keywords))
```

---

## Named Captures Pattern

```lua
-- ตัวอย่างที่ 42: Simulated Named Captures
-- Lua ไม่มี named captures โดยตรง แต่สามารถ simulate ได้

local function match_named(s, pattern, names)
    local results = {}
    local captures = {s:match(pattern)}
    for i, name in ipairs(names) do
        results[name] = captures[i]
    end
    return results
end

local date = "2024-12-25"
local result = match_named(date, "(%d%d%d%d)-(%d%d)-(%d%d)", 
                           {"year", "month", "day"})
print("=== Simulated Named Captures ===")
print("Year:", result.year)
print("Month:", result.month)
print("Day:", result.day)
```

```lua
-- ตัวอย่างที่ 43: Pattern builder สำหรับ named captures
local PatternCapture = {}
PatternCapture.__index = PatternCapture

function PatternCapture.new(pattern, names)
    return setmetatable({
        pattern = pattern,
        names = names,
    }, PatternCapture)
end

function PatternCapture:match(s)
    local captures = {s:match(self.pattern)}
    if #captures == 0 then return nil end
    
    local result = {}
    for i, name in ipairs(self.names) do
        result[name] = captures[i]
    end
    return result
end

function PatternCapture:gmatch(s)
    local results = {}
    for ... in s:gmatch(self.pattern) do
        local captures = {...}
        local row = {}
        for i, name in ipairs(self.names) do
            row[name] = captures[i]
        end
        table.insert(results, row)
    end
    return results
end

-- ทดสอบ
local date_pattern = PatternCapture.new(
    "(%d%d%d%d)-(%d%d)-(%d%d)",
    {"year", "month", "day"}
)

local d = date_pattern:match("Today is 2024-12-25")
if d then
    print(string.format("Date: %s/%s/%s", d.day, d.month, d.year))
end

local log_pattern = PatternCapture.new(
    "(%d%d%d%d%-%d%d%-%d%d) (%d%d:%d%d:%d%d) (%u+): (.+)",
    {"date", "time", "level", "message"}
)

local logs = [[
2024-01-15 10:30:00 ERROR: Connection failed
2024-01-15 10:30:01 INFO: Retrying...
2024-01-15 10:30:05 DEBUG: Connected to server
]]

local entries = log_pattern:gmatch(logs)
print("\n=== Log Entries ===")
for _, entry in ipairs(entries) do
    print(string.format("[%s %s] %-7s %s", 
          entry.date, entry.time, entry.level, entry.message))
end
```

---

## Parsing Structured Text

```lua
-- ตัวอย่างที่ 44: Parse key-value pairs
local function parse_kv(text, pair_sep, kv_sep)
    pair_sep = pair_sep or "[,;]"
    kv_sep = kv_sep or "="
    local result = {}
    
    -- แยก pairs
    for pair in (text .. ","):gmatch("([^" .. pair_sep:sub(2,-2) .. "]+)") do
        pair = pair:match("^%s*(.-)%s*$")  -- trim
        if pair ~= "" then
            local k, v = pair:match("([^" .. kv_sep .. "]+)" .. kv_sep .. "(.*)")
            if k then
                k = k:match("^%s*(.-)%s*$")
                v = v and v:match("^%s*(.-)%s*$") or ""
                result[k] = v
            end
        end
    end
    
    return result
end

local query_string = "name=Alice&age=30&city=Bangkok&active=true"
-- ปรับ separator สำหรับ URL query string
local params = {}
for k, v in query_string:gmatch("([^&=]+)=([^&]*)") do
    params[k] = v
end

print("=== URL Query Parameters ===")
for k, v in pairs(params) do
    print(string.format("  %-10s = %s", k, v))
end

-- Cookie string
local cookie = "session=abc123; user=john; expires=2024-12-31; secure"
local cookie_parts = parse_kv(cookie, "[;]", "=")
print("\n=== Cookie ===")
for k, v in pairs(cookie_parts) do
    print(string.format("  %-10s = %s", k, v))
end
```

```lua
-- ตัวอย่างที่ 45: Parse Markdown links
local function extract_links(markdown)
    local links = {}
    
    -- [text](url) format
    for text, url in markdown:gmatch("%[([^%]]+)%]%(([^%)]+)%)") do
        table.insert(links, {type = "inline", text = text, url = url})
    end
    
    -- [text][ref] format
    for text, ref in markdown:gmatch("%[([^%]]+)%]%[([^%]]*)%]") do
        table.insert(links, {type = "ref", text = text, ref = ref ~= "" and ref or text})
    end
    
    return links
end

local markdown = [[
Visit [Google](https://google.com) for search.
See [Lua manual](https://lua.org/manual/5.4) for docs.
Check [this][ref1] link.
Also [another][ref2].
]]

print("=== Markdown Links ===")
local links = extract_links(markdown)
for _, link in ipairs(links) do
    if link.type == "inline" then
        print(string.format("  [%s] -> %s", link.text, link.url))
    else
        print(string.format("  [%s][%s]", link.text, link.ref))
    end
end
```

---

## URL Parser

```lua
-- ตัวอย่างที่ 46: Full URL Parser
local function parse_url(url)
    local result = {}
    
    -- Protocol
    result.scheme, url = url:match("^([%a][%a%d%+%-%.]*):(.*)")
    if not result.scheme then
        return nil, "Invalid URL: no scheme"
    end
    
    -- Fragment
    local fragment_pos = url:find("#")
    if fragment_pos then
        result.fragment = url:sub(fragment_pos + 1)
        url = url:sub(1, fragment_pos - 1)
    end
    
    -- Query
    local query_pos = url:find("?")
    if query_pos then
        result.query = url:sub(query_pos + 1)
        url = url:sub(1, query_pos - 1)
    end
    
    -- Authority (host, port, userinfo)
    if url:sub(1, 2) == "//" then
        url = url:sub(3)
        local path_start = url:find("/")
        local authority
        if path_start then
            authority = url:sub(1, path_start - 1)
            result.path = url:sub(path_start)
        else
            authority = url
            result.path = "/"
        end
        
        -- User info
        local userinfo_end = authority:find("@")
        if userinfo_end then
            local userinfo = authority:sub(1, userinfo_end - 1)
            authority = authority:sub(userinfo_end + 1)
            result.username, result.password = userinfo:match("([^:]+):?(.*)")
        end
        
        -- Host and port
        result.host, result.port = authority:match("^(.-):?(%d*)$")
        if result.port == "" then result.port = nil end
        if result.port then result.port = tonumber(result.port) end
    else
        result.path = url
    end
    
    -- Parse query string
    if result.query then
        result.params = {}
        for k, v in result.query:gmatch("([^&=]+)=?([^&]*)") do
            result.params[k] = v ~= "" and v or true
        end
    end
    
    return result
end

local urls = {
    "https://user:pass@api.example.com:8080/v1/users?page=1&limit=10#section",
    "http://localhost/path/to/file",
    "ftp://files.example.com/pub/data.zip",
    "https://example.com?search=hello+world&lang=th",
}

print("=== URL Parser ===")
for _, url in ipairs(urls) do
    print("\nURL: " .. url)
    local parsed, err = parse_url(url)
    if parsed then
        for k, v in pairs(parsed) do
            if type(v) ~= "table" then
                print(string.format("  %-10s: %s", k, tostring(v)))
            end
        end
        if parsed.params then
            print("  params:")
            for k, v in pairs(parsed.params) do
                print(string.format("    %s = %s", k, tostring(v)))
            end
        end
    else
        print("  Error:", err)
    end
end
```

---

## Date Parser

```lua
-- ตัวอย่างที่ 47: Date Parser หลายรูปแบบ
local DateParser = {}

local months = {
    Jan=1, Feb=2, Mar=3, Apr=4, May=5, Jun=6,
    Jul=7, Aug=8, Sep=9, Oct=10, Nov=11, Dec=12,
    January=1, February=2, March=3, April=4, June=6,
    July=7, August=8, September=9, October=10, November=11, December=12
}

function DateParser.parse(s)
    local result = {}
    
    -- ISO 8601: YYYY-MM-DD
    local y, m, d = s:match("^(%d%d%d%d)-(%d%d)-(%d%d)$")
    if y then
        return {year=tonumber(y), month=tonumber(m), day=tonumber(d), format="ISO"}
    end
    
    -- DD/MM/YYYY หรือ MM/DD/YYYY
    local a, b, c = s:match("^(%d%d?)/(%d%d?)/(%d%d%d%d)$")
    if a then
        -- สมมติว่าเป็น DD/MM/YYYY
        return {year=tonumber(c), month=tonumber(b), day=tonumber(a), format="EU"}
    end
    
    -- "25 December 2024" หรือ "December 25, 2024"
    d, m, y = s:match("^(%d+)%s+(%a+)%s+(%d%d%d%d)$")
    if d and months[m] then
        return {year=tonumber(y), month=months[m], day=tonumber(d), format="Long"}
    end
    
    m, d, y = s:match("^(%a+)%s+(%d+),%s*(%d%d%d%d)$")
    if m and months[m] then
        return {year=tonumber(y), month=months[m], day=tonumber(d), format="US"}
    end
    
    -- Compact: YYYYMMDD
    y, m, d = s:match("^(%d%d%d%d)(%d%d)(%d%d)$")
    if y then
        return {year=tonumber(y), month=tonumber(m), day=tonumber(d), format="Compact"}
    end
    
    return nil, "Cannot parse date: " .. s
end

function DateParser.format(date, fmt)
    fmt = fmt or "%Y-%m-%d"
    return os.date(fmt, os.time(date))
end

local date_strings = {
    "2024-12-25",
    "25/12/2024",
    "25 December 2024",
    "December 25, 2024",
    "20241225",
}

print("=== Date Parser ===")
for _, ds in ipairs(date_strings) do
    local date, err = DateParser.parse(ds)
    if date then
        print(string.format("  %-25s -> %s (format: %s)", 
              ds, DateParser.format(date), date.format))
    else
        print(string.format("  %-25s -> ERROR: %s", ds, err))
    end
end
```

---

## CSV Parser with Patterns

```lua
-- ตัวอย่างที่ 48: CSV Parser ด้วย Patterns
local function parse_csv_pattern(line, delimiter)
    delimiter = delimiter or ","
    local fields = {}
    
    -- Pattern สำหรับ field ที่อาจมี quotes
    local pattern = string.format(
        '"([^"]*)"[%s]?|([^%s]*)[%s]?',
        delimiter, delimiter, delimiter
    )
    
    local pos = 1
    while pos <= #line do
        -- ลอง match quoted field ก่อน
        if line:sub(pos, pos) == '"' then
            local content, end_pos = line:match('^"(.-)"', pos)
            if content then
                table.insert(fields, content)
                pos = pos + #content + 2  -- skip quotes
                if line:sub(pos, pos) == delimiter then
                    pos = pos + 1
                end
            else
                pos = pos + 1
            end
        else
            -- Unquoted field
            local field, sep = line:match("([^" .. delimiter .. "]*)(.)?" , pos)
            table.insert(fields, field or "")
            pos = pos + (field and #field or 0) + (sep == delimiter and 1 or 0)
            if pos > #line and sep ~= delimiter then break end
        end
    end
    
    return fields
end

-- Simple แต่ใช้งานได้จริงมากกว่า
local function parse_csv_simple(line)
    local fields = {}
    line = line .. ","  -- sentinel
    local i = 1
    
    while i <= #line do
        if line:sub(i,i) == '"' then
            -- Quoted
            local j = i + 1
            while j <= #line do
                if line:sub(j,j) == '"' then
                    if line:sub(j+1,j+1) == '"' then
                        j = j + 2  -- escaped quote
                    else
                        break
                    end
                else
                    j = j + 1
                end
            end
            local content = line:sub(i+1, j-1):gsub('""', '"')
            table.insert(fields, content)
            i = j + 2  -- skip closing quote and comma
        else
            -- Unquoted
            local j = line:find(",", i, true) or #line + 1
            table.insert(fields, line:sub(i, j-1))
            i = j + 1
        end
    end
    
    return fields
end

local csv_lines = {
    'Alice,30,Bangkok,"Software Engineer"',
    '"Smith, John",25,"New York","Has ""quotes"" here"',
    'Bob,,"",Simple',
}

print("=== CSV Parser with Patterns ===")
for _, line in ipairs(csv_lines) do
    print("Input: " .. line)
    local fields = parse_csv_simple(line)
    for i, f in ipairs(fields) do
        print(string.format("  [%d] '%s'", i, f))
    end
    print()
end
```

---

## Template Substitution

```lua
-- ตัวอย่างที่ 49: Advanced Template Engine
local Template = {}
Template.__index = Template

function Template.new(text)
    return setmetatable({text = text}, Template)
end

function Template:render(vars, filters)
    filters = filters or {}
    
    -- เพิ่ม default filters
    filters.upper = string.upper
    filters.lower = string.lower
    filters.len = tostring
    filters.trim = function(s) return s:match("^%s*(.-)%s*$") end
    
    local result = self.text
    
    -- {{var}} - simple substitution
    result = result:gsub("{{%s*(%w+)%s*}}", function(key)
        return tostring(vars[key] or "")
    end)
    
    -- {{var|filter}} - with filter
    result = result:gsub("{{%s*(%w+)|(%w+)%s*}}", function(key, filter)
        local val = tostring(vars[key] or "")
        if filters[filter] then
            return filters[filter](val)
        end
        return val
    end)
    
    -- {{#if var}}...{{/if}} - conditional
    result = result:gsub("{{#if%s+(%w+)}}(.-){{/if}}", function(key, content)
        if vars[key] and vars[key] ~= false and vars[key] ~= "" then
            return content
        end
        return ""
    end)
    
    return result
end

-- ทดสอบ
local tmpl = Template.new([[
Dear {{name|upper}},

{{#if premium}}
You are a PREMIUM member!
{{/if}}
Your account: {{email}}
Member since: {{since}}
Status: {{status}}

Best regards,
The Team
]])

local vars = {
    name = "สมชาย ใจดี",
    email = "somchai@example.com",
    since = "2023-01-15",
    status = "Active",
    premium = true,
}

print("=== Template Engine ===")
print(tmpl:render(vars))
```

---

## Tokenizer

```lua
-- ตัวอย่างที่ 50: Lexer/Tokenizer สำหรับ simple language
local function tokenize_lua_like(code)
    local tokens = {}
    local i = 1
    
    local keywords = {
        ["local"] = true, ["function"] = true, ["return"] = true,
        ["if"] = true, ["then"] = true, ["else"] = true, ["end"] = true,
        ["while"] = true, ["do"] = true, ["for"] = true, ["in"] = true,
        ["true"] = true, ["false"] = true, ["nil"] = true, ["not"] = true,
        ["and"] = true, ["or"] = true,
    }
    
    while i <= #code do
        -- Skip whitespace
        local ws = code:match("^%s+", i)
        if ws then
            i = i + #ws
        -- Comment
        elseif code:match("^%-%-", i) then
            local comment = code:match("^%-%-[^\n]*", i)
            table.insert(tokens, {type = "COMMENT", value = comment})
            i = i + #comment
        -- String
        elseif code:sub(i,i) == '"' or code:sub(i,i) == "'" then
            local q = code:sub(i,i)
            local j = i + 1
            while j <= #code and code:sub(j,j) ~= q do
                if code:sub(j,j) == "\\" then j = j + 1 end
                j = j + 1
            end
            local str = code:sub(i, j)
            table.insert(tokens, {type = "STRING", value = str})
            i = j + 1
        -- Number
        elseif code:match("^%d", i) then
            local num = code:match("^%d+%.?%d*", i)
            table.insert(tokens, {type = "NUMBER", value = num})
            i = i + #num
        -- Identifier or keyword
        elseif code:match("^[%a_]", i) then
            local id = code:match("^[%a_][%w_]*", i)
            local token_type = keywords[id] and "KEYWORD" or "IDENT"
            table.insert(tokens, {type = token_type, value = id})
            i = i + #id
        -- Operators
        elseif code:match("^[%+%-%*/%%=<>~!&|%^]+", i) then
            local op = code:match("^[%+%-%*/%%=<>~!&|%^]+", i)
            table.insert(tokens, {type = "OP", value = op})
            i = i + #op
        -- Punctuation
        elseif code:match("^[%(%)%[%]%{%}%.,;:]", i) then
            local punct = code:sub(i,i)
            table.insert(tokens, {type = "PUNCT", value = punct})
            i = i + 1
        else
            -- Unknown character
            table.insert(tokens, {type = "UNKNOWN", value = code:sub(i,i)})
            i = i + 1
        end
    end
    
    return tokens
end

local sample_code = [[
local function hello(name)
    -- Say hello
    if name then
        return "Hello, " .. name
    end
    return "Hello, World"
end
]]

print("=== Tokenizer ===")
local tokens = tokenize_lua_like(sample_code)
for _, tok in ipairs(tokens) do
    if tok.type ~= "COMMENT" then
        print(string.format("  %-8s : %s", tok.type, tok.value))
    end
end
print("Total tokens:", #tokens)
```

```lua
-- ตัวอย่างที่ 51: Pattern-based word frequency counter
local function word_frequency(text)
    local freq = {}
    local total = 0
    
    for word in text:lower():gmatch("[%a']+") do
        freq[word] = (freq[word] or 0) + 1
        total = total + 1
    end
    
    -- Sort by frequency
    local sorted = {}
    for word, count in pairs(freq) do
        table.insert(sorted, {word = word, count = count})
    end
    table.sort(sorted, function(a, b)
        if a.count ~= b.count then return a.count > b.count end
        return a.word < b.word
    end)
    
    return sorted, total
end

local text = [[
Lua is a powerful, efficient, lightweight, embeddable scripting language.
Lua is designed to be embedded into other applications and extend them.
Lua combines simple procedural syntax with powerful data description constructs
based on associative arrays and extensible semantics.
]]

print("=== Word Frequency ===")
local freq, total = word_frequency(text)
print(string.format("Total words: %d, Unique: %d", total, #freq))
print("Top 10:")
for i = 1, math.min(10, #freq) do
    local f = freq[i]
    local bar = string.rep("█", f.count)
    print(string.format("  %-15s %3d %s", f.word, f.count, bar))
end
```

```lua
-- ตัวอย่างที่ 52: Pattern สำหรับ validate passwords
local function validate_password(password)
    local errors = {}
    
    if #password < 8 then
        table.insert(errors, "ต้องมีความยาวอย่างน้อย 8 ตัวอักษร")
    end
    
    if not password:match("%u") then
        table.insert(errors, "ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว")
    end
    
    if not password:match("%l") then
        table.insert(errors, "ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว")
    end
    
    if not password:match("%d") then
        table.insert(errors, "ต้องมีตัวเลขอย่างน้อย 1 ตัว")
    end
    
    if not password:match("[%p]") then
        table.insert(errors, "ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว (เช่น !@#$%)")
    end
    
    if password:match("%s") then
        table.insert(errors, "ไม่ควรมีช่องว่าง")
    end
    
    return #errors == 0, errors
end

local passwords = {"password", "Password1", "P@ssw0rd!", "Short1!", "no upper 1!"}
print("=== Password Validation ===")
for _, pwd in ipairs(passwords) do
    local ok, errors = validate_password(pwd)
    print(string.format("\n'%s': %s", pwd, ok and "VALID" or "INVALID"))
    for _, err in ipairs(errors) do
        print("  - " .. err)
    end
end
```

```lua
-- ตัวอย่างที่ 53: สรุป - Text processing pipeline
local function process_text(text, transformations)
    local result = text
    for _, transform in ipairs(transformations) do
        local name = transform[1]
        local func = transform[2]
        result = func(result)
    end
    return result
end

local pipeline = {
    {"trim", function(s) return s:match("^%s*(.-)%s*$") end},
    {"normalize_spaces", function(s) return s:gsub("%s+", " ") end},
    {"remove_special", function(s) return s:gsub("[^%w%s%.%,!%?]", "") end},
    {"capitalize", function(s) 
        return s:gsub("(%a)([%a]*)", function(f, r) 
            return f:upper() .. r:lower() 
        end)
    end},
}

local input = "  hello WORLD!   this IS a   TEST...   123  "
print("=== Text Processing Pipeline ===")
print("Input: '" .. input .. "'")

local result = process_text(input, pipeline)
print("Output: '" .. result .. "'")
```

---

## สรุปบทที่ 12

| ฟังก์ชัน | การใช้งาน | ตัวอย่าง |
|---------|-----------|---------|
| `string.find` | ค้นหา position | `s:find("pattern")` |
| `string.match` | จับ captures | `s:match("(%d+)")` |
| `string.gmatch` | iterator | `for w in s:gmatch("%a+")` |
| `string.gsub` | แทนที่ | `s:gsub("a", "b")` |

| Pattern | ความหมาย |
|---------|---------|
| `.` | ทุกอักขระ |
| `%a` | ตัวอักษร |
| `%d` | ตัวเลข |
| `%s` | whitespace |
| `%w` | alphanumeric |
| `%p` | punctuation |
| `*` | 0 หรือมากกว่า (greedy) |
| `+` | 1 หรือมากกว่า (greedy) |
| `-` | 0 หรือมากกว่า (lazy) |
| `?` | 0 หรือ 1 |
| `^` | ต้นบรรทัด |
| `$` | ท้ายบรรทัด |
| `()` | capture group |
| `%bxy` | balanced match |
| `%f[set]` | frontier pattern |

> **Tip:** ใน Lua pattern, `.` จะ match ทุกอักขระรวมถึง `\n` (ต่างจาก POSIX regex) และ `*`, `+`, `-` เป็น greedy ยกเว้น `-` ที่เป็น lazy
