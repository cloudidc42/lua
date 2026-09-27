# บทที่ 14: Multiple Returns และ Varargs

## บทนำ

Lua มีคุณสมบัติพิเศษที่ทำให้ต่างจากภาษาโปรแกรมทั่วไป นั่นคือ **ฟังก์ชันสามารถคืนค่าได้มากกว่าหนึ่งค่า** (Multiple Return Values) และ **ฟังก์ชันสามารถรับพารามิเตอร์จำนวนไม่จำกัด** (Variadic Functions / Varargs)

ความสามารถเหล่านี้ช่วยให้โค้ด Lua มีความยืดหยุ่นสูงและเขียนได้กระชับมากขึ้น บทนี้จะพาคุณสำรวจทุกแง่มุมของ Multiple Returns และ Varargs อย่างละเอียด

---

## 14.1 Functions ที่คืนค่าหลายค่า

### ตัวอย่างพื้นฐาน

ใน Lua ฟังก์ชันสามารถ return ค่าหลายค่าโดยคั่นด้วยลูกน้ำ

```lua
-- ตัวอย่างที่ 1: ฟังก์ชันคืนค่า 2 ค่า
function min_max(t)
    local min_val = t[1]
    local max_val = t[1]
    for _, v in ipairs(t) do
        if v < min_val then min_val = v end
        if v > max_val then max_val = v end
    end
    return min_val, max_val  -- คืนค่า 2 ค่า
end

local data = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3}
local lo, hi = min_max(data)
print("ต่ำสุด:", lo)   -- ต่ำสุด: 1
print("สูงสุด:", hi)   -- สูงสุด: 9
```

```lua
-- ตัวอย่างที่ 2: ฟังก์ชันแบ่งส่วนของสตริง
function split_first_rest(str, sep)
    local pos = str:find(sep, 1, true)
    if not pos then
        return str, ""
    end
    return str:sub(1, pos - 1), str:sub(pos + #sep)
end

local first, rest = split_first_rest("hello world foo", " ")
print("first:", first)  -- first: hello
print("rest:", rest)    -- rest: world foo
```

```lua
-- ตัวอย่างที่ 3: คืนค่าสำเร็จ/ล้มเหลว (pattern ทั่วไปใน Lua)
function safe_divide(a, b)
    if b == 0 then
        return nil, "division by zero"
    end
    return a / b, nil
end

local result, err = safe_divide(10, 2)
if err then
    print("Error:", err)
else
    print("Result:", result)  -- Result: 5.0
end

local result2, err2 = safe_divide(10, 0)
if err2 then
    print("Error:", err2)  -- Error: division by zero
end
```

```lua
-- ตัวอย่างที่ 4: ฟังก์ชันคืนค่า 3 ค่า - coordinates
function get_position()
    return 10.5, 20.3, 5.0  -- x, y, z
end

local x, y, z = get_position()
print(string.format("Position: (%.1f, %.1f, %.1f)", x, y, z))
-- Position: (10.5, 20.3, 5.0)
```

```lua
-- ตัวอย่างที่ 5: คืนค่าหลายชนิด
function describe_value(v)
    local t = type(v)
    local desc
    if t == "number" then
        if v == math.floor(v) then
            desc = "integer"
        else
            desc = "float"
        end
    elseif t == "string" then
        desc = "length " .. #v
    else
        desc = t
    end
    return t, desc, tostring(v)
end

local vtype, vdesc, vstr = describe_value(42)
print(vtype, vdesc, vstr)  -- number  integer  42

local vtype2, vdesc2, vstr2 = describe_value("hello")
print(vtype2, vdesc2, vstr2)  -- string  length 5  hello
```

---

## 14.2 การปรับจำนวนค่าที่คืน (Adjustment)

Lua จะปรับจำนวนค่าที่คืนมาให้เหมาะสมกับบริบท

### Truncation (ตัดทิ้ง)

```lua
-- ตัวอย่างที่ 6: ตัดค่าส่วนเกินออก
function three_values()
    return 1, 2, 3
end

local a = three_values()  -- รับแค่ค่าแรก
print(a)  -- 1

local b, c = three_values()  -- รับสองค่าแรก
print(b, c)  -- 1  2
```

### Padding with nil

```lua
-- ตัวอย่างที่ 7: เติม nil เมื่อตัวแปรมากกว่าค่าที่คืน
function two_values()
    return "hello", "world"
end

local p, q, r = two_values()
print(p)  -- hello
print(q)  -- world
print(r)  -- nil  (เติมด้วย nil อัตโนมัติ)
```

```lua
-- ตัวอย่างที่ 8: การปรับค่าในลิสต์ expression
function pair()
    return 10, 20
end

-- เมื่อฟังก์ชันไม่ได้อยู่ในตำแหน่งสุดท้าย จะได้แค่ค่าแรก
local t = {pair(), 30}
print(#t, t[1], t[2], t[3])  -- 3  10  30   (20 ถูกตัดทิ้ง!)

-- เมื่อฟังก์ชันอยู่ในตำแหน่งสุดท้าย จะได้ทุกค่า
local t2 = {30, pair()}
print(#t2, t2[1], t2[2], t2[3])  -- 3  30  10  20
```

```lua
-- ตัวอย่างที่ 9: วงเล็บบังคับให้ได้ค่าเดียว
function multi()
    return "a", "b", "c"
end

print(multi())      -- a  b  c  (ทุกค่า)
print((multi()))    -- a  (แค่ค่าแรก เพราะมีวงเล็บ)

local x = (multi())  -- x = "a" เท่านั้น
print(x)  -- a
```

```lua
-- ตัวอย่างที่ 10: ผลกระทบใน table constructor
function rgb()
    return 255, 128, 0
end

-- ที่ตำแหน่งสุดท้ายของ table
local color1 = {rgb()}
print(#color1)  -- 3

-- ไม่ใช่ตำแหน่งสุดท้าย
local color2 = {rgb(), 255}  -- rgb() ถูกปรับให้ได้แค่ค่าแรก
print(#color2)  -- 2: {255, 255}
```

---

## 14.3 Multiple Assignment กับ Function Calls

```lua
-- ตัวอย่างที่ 11: swap โดยไม่ต้องใช้ตัวแปรชั่วคราว
local a, b = 10, 20
print("before:", a, b)  -- before: 10  20

a, b = b, a  -- swap!
print("after:", a, b)   -- after: 20  10
```

```lua
-- ตัวอย่างที่ 12: multiple assignment กับฟังก์ชัน
function swap(x, y)
    return y, x
end

local p, q = 100, 200
p, q = swap(p, q)
print(p, q)  -- 200  100
```

```lua
-- ตัวอย่างที่ 13: destructuring pattern
function get_user()
    return "Alice", 30, "alice@example.com"
end

local name, age, email = get_user()
print(string.format("Name: %s, Age: %d, Email: %s", name, age, email))
-- Name: Alice, Age: 30, Email: alice@example.com
```

```lua
-- ตัวอย่างที่ 14: รับเฉพาะค่าที่ต้องการ
function many_values()
    return 1, 2, 3, 4, 5
end

-- ใช้ _ เพื่อข้ามค่าที่ไม่ต้องการ
local _, second, _, fourth = many_values()
print(second, fourth)  -- 2  4
```

```lua
-- ตัวอย่างที่ 15: ใช้กับ string.find
local s = "hello world 2024"
local start_pos, end_pos = string.find(s, "%d+")
print("found at:", start_pos, "to", end_pos)  -- found at: 13  to  16

local number_str = s:sub(start_pos, end_pos)
print("number:", number_str)  -- number: 2024
```

---

## 14.4 Chaining Functions กับ Multiple Returns

```lua
-- ตัวอย่างที่ 16: chain ผ่านการส่งผลลัพธ์
function parse_int(s)
    local n = tonumber(s)
    if not n then return nil, "not a number: " .. s end
    if n ~= math.floor(n) then return nil, "not an integer: " .. s end
    return math.floor(n), nil
end

function validate_range(n, lo, hi)
    if n < lo or n > hi then
        return nil, string.format("out of range [%d, %d]: %d", lo, hi, n)
    end
    return n, nil
end

-- chain: parse -> validate
local function parse_port(s)
    local n, err = parse_int(s)
    if err then return nil, err end
    return validate_range(n, 1, 65535)
end

local port, err = parse_port("8080")
print(port, err)  -- 8080  nil

local port2, err2 = parse_port("99999")
print(port2, err2)  -- nil  out of range [1, 65535]: 99999

local port3, err3 = parse_port("abc")
print(port3, err3)  -- nil  not a number: abc
```

```lua
-- ตัวอย่างที่ 17: pipeline transformation
function step1(x) return x * 2, x + 1 end
function step2(a, b) return a + b, a * b end
function step3(s, p) return s + p end

local result = step3(step2(step1(5)))
print(result)  -- step1(5) = 10,6 -> step2(10,6) = 16,60 -> step3(16,60) = 76
```

```lua
-- ตัวอย่างที่ 18: error propagation chain
function read_config(filename)
    -- จำลองการอ่านไฟล์
    if filename == "missing.cfg" then
        return nil, "file not found: " .. filename
    end
    return {host="localhost", port=3306}, nil
end

function connect_db(config)
    if not config then return nil, "no config" end
    if config.port < 1000 then
        return nil, "invalid port"
    end
    -- จำลองการเชื่อมต่อ
    return {connected=true, host=config.host}, nil
end

function run_query(conn, sql)
    if not conn or not conn.connected then
        return nil, "not connected"
    end
    return {rows={{1,"Alice"},{2,"Bob"}}}, nil
end

-- chain ทั้งหมด
local function do_query(filename, sql)
    local config, err = read_config(filename)
    if err then return nil, "config: " .. err end

    local conn, err2 = connect_db(config)
    if err2 then return nil, "connect: " .. err2 end

    return run_query(conn, sql)
end

local result, err = do_query("app.cfg", "SELECT * FROM users")
if result then
    print("rows:", #result.rows)  -- rows: 2
else
    print("error:", err)
end

local result2, err2 = do_query("missing.cfg", "SELECT 1")
print(result2, err2)  -- nil  config: file not found: missing.cfg
```

---

## 14.5 select() ฟังก์ชัน

`select()` เป็นฟังก์ชันพิเศษสำหรับจัดการ varargs

### select('#') - นับจำนวน

```lua
-- ตัวอย่างที่ 19: นับจำนวน arguments
function count_args(...)
    return select('#', ...)
end

print(count_args(1, 2, 3))        -- 3
print(count_args("a", "b"))       -- 2
print(count_args(nil, nil, nil))  -- 3 (นับ nil ด้วย!)
print(count_args())               -- 0
```

```lua
-- ตัวอย่างที่ 20: ความแตกต่างระหว่าง # และ select('#')
function demo(...)
    local t = {...}
    print("# operator:", #t)         -- อาจไม่ถูกต้องถ้ามี nil
    print("select('#'):", select('#', ...))  -- นับถูกเสมอ
end

demo(1, nil, 3)
-- # operator: 1 หรือ 3 (undefined behavior)
-- select('#'): 3
```

### select(n) - ดึงค่าตำแหน่งที่ n

```lua
-- ตัวอย่างที่ 21: ดึงค่าตั้งแต่ตำแหน่ง n
function show_from(n, ...)
    print("from position", n, ":")
    print(select(n, ...))
end

show_from(2, "a", "b", "c", "d")
-- from position 2 :
-- b  c  d

show_from(3, 10, 20, 30, 40, 50)
-- from position 3 :
-- 30  40  50
```

```lua
-- ตัวอย่างที่ 22: select กับ index ลบ (นับจากท้าย)
function last_arg(...)
    local n = select('#', ...)
    return (select(n, ...))  -- วงเล็บเพื่อให้ได้แค่ค่าเดียว
end

print(last_arg(1, 2, 3, 4, 5))  -- 5
print(last_arg("x", "y", "z"))  -- z
```

```lua
-- ตัวอย่างที่ 23: ใช้ select วน loop
function sum_all(...)
    local total = 0
    for i = 1, select('#', ...) do
        local v = select(i, ...)
        if type(v) == "number" then
            total = total + v
        end
    end
    return total
end

print(sum_all(1, 2, 3, 4, 5))           -- 15
print(sum_all(1, "skip", 3, nil, 5))    -- 9
```

```lua
-- ตัวอย่างที่ 24: สร้าง slice จาก varargs
function slice(from, to, ...)
    local result = {}
    local n = select('#', ...)
    to = to or n
    for i = from, math.min(to, n) do
        result[#result + 1] = (select(i, ...))
    end
    return table.unpack(result)
end

print(slice(2, 4, "a", "b", "c", "d", "e"))  -- b  c  d
```

---

## 14.6 table.pack() และ table.unpack()

### table.pack()

```lua
-- ตัวอย่างที่ 25: เก็บ varargs ลง table ด้วย table.pack
function inspect_args(...)
    local args = table.pack(...)
    print("count:", args.n)  -- field .n คือจำนวน arguments จริง
    for i = 1, args.n do
        print(i, args[i])
    end
end

inspect_args("hello", 42, true, nil, "world")
-- count: 5
-- 1  hello
-- 2  42
-- 3  true
-- 4  nil
-- 5  world
```

```lua
-- ตัวอย่างที่ 26: เปรียบเทียบ table.pack vs {...}
function compare(...)
    local packed = table.pack(...)   -- มี .n field
    local literal = {...}            -- ไม่มี .n field

    print("packed.n =", packed.n)
    print("#{...} =", #literal)     -- อาจผิดถ้ามี nil
end

compare(1, nil, 3)
-- packed.n = 3
-- #{...} = 1 หรือ 3 (undefined)
```

```lua
-- ตัวอย่างที่ 27: ใช้ table.pack เก็บ multiple returns
function get_stats(t)
    local sum = 0
    local min_v, max_v = t[1], t[1]
    for _, v in ipairs(t) do
        sum = sum + v
        if v < min_v then min_v = v end
        if v > max_v then max_v = v end
    end
    return sum, sum/#t, min_v, max_v
end

local stats = table.pack(get_stats({4, 7, 2, 9, 1, 5}))
print("n values returned:", stats.n)  -- 4
print("sum:", stats[1])    -- 28
print("avg:", stats[2])    -- 4.666...
print("min:", stats[3])    -- 1
print("max:", stats[4])    -- 9
```

### table.unpack()

```lua
-- ตัวอย่างที่ 28: กระจาย table เป็น arguments
local args = {10, 20, 30}
print(table.unpack(args))  -- 10  20  30

-- เหมือนกับ
print(args[1], args[2], args[3])  -- 10  20  30
```

```lua
-- ตัวอย่างที่ 29: unpack กับ range
local t = {1, 2, 3, 4, 5}

print(table.unpack(t))          -- 1  2  3  4  5
print(table.unpack(t, 2))       -- 2  3  4  5  (เริ่มจากตำแหน่ง 2)
print(table.unpack(t, 2, 4))    -- 2  3  4  (ตำแหน่ง 2 ถึง 4)
```

```lua
-- ตัวอย่างที่ 30: ส่ง table เป็น arguments ให้ฟังก์ชัน
function add3(a, b, c)
    return a + b + c
end

local nums = {10, 20, 30}
local result = add3(table.unpack(nums))
print(result)  -- 60
```

```lua
-- ตัวอย่างที่ 31: dynamic function call
function call_with_args(fn, args_table)
    return fn(table.unpack(args_table, 1, args_table.n or #args_table))
end

local function multiply(a, b, c)
    return a * b * c
end

print(call_with_args(multiply, {3, 4, 5}))  -- 60
print(call_with_args(math.max, {3, 7, 2, 9, 1}))  -- 9
```

```lua
-- ตัวอย่างที่ 32: pack/unpack round-trip
function passthrough(...)
    local saved = table.pack(...)
    -- ... ทำบางอย่าง ...
    return table.unpack(saved, 1, saved.n)
end

local a, b, c = passthrough(100, nil, 300)
print(a, b, c)  -- 100  nil  300  (nil ถูกเก็บรักษาไว้)
```

---

## 14.7 Variadic Functions (...)

### พื้นฐาน

```lua
-- ตัวอย่างที่ 33: ฟังก์ชัน variadic พื้นฐาน
function greet_all(...)
    for i, name in ipairs({...}) do
        print("Hello,", name .. "!")
    end
end

greet_all("Alice", "Bob", "Charlie")
-- Hello, Alice!
-- Hello, Bob!
-- Hello, Charlie!
```

```lua
-- ตัวอย่างที่ 34: sum varargs
function sum(...)
    local total = 0
    for _, v in ipairs({...}) do
        total = total + v
    end
    return total
end

print(sum(1, 2, 3, 4, 5))   -- 15
print(sum(10, 20))           -- 30
print(sum())                  -- 0
```

```lua
-- ตัวอย่างที่ 35: max varargs
function max(first, ...)
    local result = first
    for _, v in ipairs({...}) do
        if v > result then result = v end
    end
    return result
end

print(max(3, 1, 4, 1, 5, 9, 2, 6))  -- 9
print(max(42))                         -- 42
```

### การรวบรวม Varargs ลง Table

```lua
-- ตัวอย่างที่ 36: รวบรวมด้วย {...}
function collect_to_table(...)
    return {...}  -- ง่าย แต่อาจมีปัญหาถ้ามี nil
end

local t = collect_to_table(1, 2, 3)
for i, v in ipairs(t) do
    print(i, v)
end
```

```lua
-- ตัวอย่างที่ 37: รวบรวมด้วย table.pack (แนะนำ)
function safe_collect(...)
    local t = table.pack(...)
    -- ใช้ t.n แทน #t
    local result = {}
    for i = 1, t.n do
        result[i] = t[i]
    end
    return result, t.n
end

local vals, count = safe_collect(10, nil, 30, nil, 50)
print("count:", count)  -- count: 5
for i, v in ipairs(vals) do
    print(i, v)
end
```

---

## 14.8 Passing Varargs Through

```lua
-- ตัวอย่างที่ 38: ส่ง ... ต่อไปยังฟังก์ชันอื่น
function log_and_print(...)
    io.write("[LOG] ")
    print(...)  -- ส่ง ... ไปยัง print โดยตรง
end

log_and_print("hello", "world", 42)
-- [LOG] hello  world  42
```

```lua
-- ตัวอย่างที่ 39: wrapper function
function timed_call(fn, ...)
    local start = os.clock()
    local results = table.pack(fn(...))
    local elapsed = os.clock() - start
    print(string.format("elapsed: %.6f seconds", elapsed))
    return table.unpack(results, 1, results.n)
end

local function slow_sum(a, b, c)
    return a + b + c
end

local r = timed_call(slow_sum, 10, 20, 30)
print("result:", r)  -- result: 60
```

```lua
-- ตัวอย่างที่ 40: middleware pattern
function with_validation(fn, validator, ...)
    local ok, err = validator(...)
    if not ok then
        return nil, "validation failed: " .. tostring(err)
    end
    return fn(...)
end

local function add(a, b)
    return a + b
end

local function numbers_only(...)
    for i = 1, select('#', ...) do
        local v = select(i, ...)
        if type(v) ~= "number" then
            return false, "argument " .. i .. " is not a number"
        end
    end
    return true
end

print(with_validation(add, numbers_only, 10, 20))     -- 30
print(with_validation(add, numbers_only, 10, "abc"))  -- nil  validation failed: argument 2 is not a number
```

---

## 14.9 Mixed Fixed + Variadic Parameters

```lua
-- ตัวอย่างที่ 41: fixed params ก่อน varargs
function printf_simple(format, ...)
    io.write(string.format(format, ...))
end

printf_simple("Hello, %s! You are %d years old.\n", "Alice", 30)
-- Hello, Alice! You are 30 years old.
```

```lua
-- ตัวอย่างที่ 42: required + optional params
function create_point(x, y, ...)
    local extra = {...}
    local z = extra[1] or 0
    local label = extra[2] or "point"
    return {x=x, y=y, z=z, label=label}
end

local p1 = create_point(1, 2)
print(p1.x, p1.y, p1.z, p1.label)  -- 1  2  0  point

local p2 = create_point(3, 4, 5, "origin")
print(p2.x, p2.y, p2.z, p2.label)  -- 3  4  5  origin
```

```lua
-- ตัวอย่างที่ 43: tag + values pattern
function tag(tagname, ...)
    local attrs = {}
    local content = {}
    local args = table.pack(...)

    -- แยก attributes (tables) จาก content (strings)
    for i = 1, args.n do
        local v = args[i]
        if type(v) == "table" then
            for k, val in pairs(v) do
                attrs[#attrs+1] = k .. '="' .. val .. '"'
            end
        else
            content[#content+1] = tostring(v)
        end
    end

    local attr_str = #attrs > 0 and " " .. table.concat(attrs, " ") or ""
    return string.format("<%s%s>%s</%s>",
        tagname, attr_str, table.concat(content), tagname)
end

print(tag("p", "Hello World"))
-- <p>Hello World</p>

print(tag("a", {href="https://example.com"}, "Click here"))
-- <a href="https://example.com">Click here</a>
```

---

## 14.10 Printf-Style Formatting

```lua
-- ตัวอย่างที่ 44: printf implementation
function printf(fmt, ...)
    io.write(string.format(fmt, ...))
end

function fprintf(file, fmt, ...)
    file:write(string.format(fmt, ...))
end

function sprintf(fmt, ...)
    return string.format(fmt, ...)
end

printf("Name: %s, Score: %.2f\n", "Bob", 95.5)
-- Name: Bob, Score: 95.50

local msg = sprintf("Error %d: %s", 404, "Not Found")
print(msg)  -- Error 404: Not Found
```

```lua
-- ตัวอย่างที่ 45: log levels
local LOG_LEVELS = {DEBUG=1, INFO=2, WARN=3, ERROR=4}
local current_level = LOG_LEVELS.INFO

function log(level, fmt, ...)
    if LOG_LEVELS[level] >= current_level then
        local timestamp = os.date("%H:%M:%S")
        printf("[%s][%s] %s\n", timestamp, level, sprintf(fmt, ...))
    end
end

log("INFO", "Server started on port %d", 8080)
log("DEBUG", "This won't show")  -- ต่ำกว่า current_level
log("WARN", "Memory usage: %d%%", 85)
log("ERROR", "Connection failed: %s", "timeout")
```

---

## 14.11 Accumulator กับ Varargs

```lua
-- ตัวอย่างที่ 46: accumulate ค่าหลายชนิด
function stats(...)
    local n = select('#', ...)
    if n == 0 then
        return {count=0, sum=0, min=nil, max=nil, mean=nil}
    end

    local sum = 0
    local min_v = select(1, ...)
    local max_v = min_v

    for i = 1, n do
        local v = select(i, ...)
        sum = sum + v
        if v < min_v then min_v = v end
        if v > max_v then max_v = v end
    end

    return {
        count = n,
        sum = sum,
        min = min_v,
        max = max_v,
        mean = sum / n
    }
end

local s = stats(4, 7, 2, 9, 1, 5, 8, 3, 6)
print(string.format("count=%d sum=%d min=%d max=%d mean=%.2f",
    s.count, s.sum, s.min, s.max, s.mean))
-- count=9 sum=45 min=1 max=9 mean=5.00
```

```lua
-- ตัวอย่างที่ 47: string accumulator
function concat_all(sep, ...)
    local parts = {}
    for i = 1, select('#', ...) do
        local v = select(i, ...)
        parts[#parts+1] = tostring(v)
    end
    return table.concat(parts, sep)
end

print(concat_all(", ", "apple", "banana", "cherry"))
-- apple, banana, cherry

print(concat_all(" | ", 1, 2, 3, 4, 5))
-- 1 | 2 | 3 | 4 | 5
```

```lua
-- ตัวอย่างที่ 48: filter varargs
function filter(predicate, ...)
    local result = {}
    for i = 1, select('#', ...) do
        local v = select(i, ...)
        if predicate(v) then
            result[#result+1] = v
        end
    end
    return table.unpack(result)
end

local function is_even(n) return n % 2 == 0 end
print(filter(is_even, 1, 2, 3, 4, 5, 6, 7, 8))  -- 2  4  6  8

local function is_string(v) return type(v) == "string" end
print(filter(is_string, 1, "hello", true, "world", nil, "lua"))  -- hello  world  lua
```

---

## 14.12 Building Flexible APIs

```lua
-- ตัวอย่างที่ 49: flexible table constructor
function new_record(...)
    local record = {}
    local args = table.pack(...)

    -- รองรับทั้ง (key, value, key, value, ...) และ ({key=value, ...})
    if args.n == 1 and type(args[1]) == "table" then
        for k, v in pairs(args[1]) do
            record[k] = v
        end
    else
        -- ต้องเป็นคู่ key-value
        assert(args.n % 2 == 0, "need even number of args")
        for i = 1, args.n, 2 do
            record[args[i]] = args[i+1]
        end
    end
    return record
end

local r1 = new_record("name", "Alice", "age", 30)
print(r1.name, r1.age)  -- Alice  30

local r2 = new_record({name="Bob", age=25})
print(r2.name, r2.age)  -- Bob  25
```

```lua
-- ตัวอย่างที่ 50: event emitter
local EventEmitter = {}
EventEmitter.__index = EventEmitter

function EventEmitter.new()
    return setmetatable({handlers={}}, EventEmitter)
end

function EventEmitter:on(event, handler)
    if not self.handlers[event] then
        self.handlers[event] = {}
    end
    table.insert(self.handlers[event], handler)
    return self  -- method chaining
end

function EventEmitter:emit(event, ...)
    local handlers = self.handlers[event]
    if handlers then
        for _, handler in ipairs(handlers) do
            handler(...)  -- ส่ง varargs ต่อไปยัง handler
        end
    end
end

local emitter = EventEmitter.new()

emitter:on("data", function(val, source)
    print(string.format("received %s from %s", val, source))
end)

emitter:on("data", function(val, source)
    print(string.format("logged: %s", val))
end)

emitter:emit("data", "hello", "server")
-- received hello from server
-- logged: hello
```

```lua
-- ตัวอย่างที่ 51: method chaining API
local Query = {}
Query.__index = Query

function Query.new(table_name)
    return setmetatable({
        _table = table_name,
        _conditions = {},
        _columns = {"*"},
        _limit = nil
    }, Query)
end

function Query:select(...)
    self._columns = {...}
    return self
end

function Query:where(...)
    local conditions = {...}
    for _, c in ipairs(conditions) do
        self._conditions[#self._conditions+1] = c
    end
    return self
end

function Query:limit(n)
    self._limit = n
    return self
end

function Query:build()
    local sql = "SELECT " .. table.concat(self._columns, ", ")
    sql = sql .. " FROM " .. self._table
    if #self._conditions > 0 then
        sql = sql .. " WHERE " .. table.concat(self._conditions, " AND ")
    end
    if self._limit then
        sql = sql .. " LIMIT " .. self._limit
    end
    return sql
end

local q = Query.new("users")
    :select("id", "name", "email")
    :where("age > 18", "active = 1")
    :limit(10)
    :build()

print(q)
-- SELECT id, name, email FROM users WHERE age > 18 AND active = 1 LIMIT 10
```

---

## 14.13 Advanced Examples

```lua
-- ตัวอย่างที่ 52: memoize with varargs
function memoize(fn)
    local cache = {}
    return function(...)
        -- สร้าง cache key จาก arguments
        local key = table.concat(table.pack(...), "\0", 1, select('#', ...))
        if cache[key] == nil then
            cache[key] = table.pack(fn(...))
        end
        return table.unpack(cache[key], 1, cache[key].n)
    end
end

local fib
fib = memoize(function(n)
    if n <= 1 then return n end
    return fib(n-1) + fib(n-2)
end)

for i = 0, 10 do
    io.write(fib(i) .. " ")
end
print()  -- 0 1 1 2 3 5 8 13 21 34 55
```

```lua
-- ตัวอย่างที่ 53: curry แบบง่าย
function curry(fn, arity)
    arity = arity or debug.getinfo(fn, "u").nparams
    local function helper(args)
        if #args >= arity then
            return fn(table.unpack(args, 1, arity))
        end
        return function(...)
            local new_args = {table.unpack(args)}
            for i = 1, select('#', ...) do
                new_args[#new_args+1] = select(i, ...)
            end
            return helper(new_args)
        end
    end
    return helper({})
end

local add = curry(function(a, b, c) return a + b + c end, 3)
print(add(1)(2)(3))     -- 6
print(add(1, 2)(3))     -- 6
print(add(1)(2, 3))     -- 6
print(add(1, 2, 3))     -- 6
```

```lua
-- ตัวอย่างที่ 54: zip varargs
function zip(...)
    local tables = {...}
    local n = math.huge
    for _, t in ipairs(tables) do
        n = math.min(n, #t)
    end
    if n == math.huge then n = 0 end

    local result = {}
    for i = 1, n do
        local row = {}
        for _, t in ipairs(tables) do
            row[#row+1] = t[i]
        end
        result[#result+1] = row
    end
    return result
end

local names = {"Alice", "Bob", "Charlie"}
local ages  = {30, 25, 35}
local roles = {"admin", "user", "mod"}

for _, row in ipairs(zip(names, ages, roles)) do
    print(string.format("%s (%d) - %s", table.unpack(row)))
end
-- Alice (30) - admin
-- Bob (25) - user
-- Charlie (35) - mod
```

```lua
-- ตัวอย่างที่ 55: flatten varargs recursively
function flatten(...)
    local result = {}
    local function _flatten(val)
        if type(val) == "table" then
            for _, v in ipairs(val) do
                _flatten(v)
            end
        else
            result[#result+1] = val
        end
    end
    for i = 1, select('#', ...) do
        _flatten(select(i, ...))
    end
    return table.unpack(result)
end

print(flatten(1, {2, 3}, {4, {5, 6}}, 7))  -- 1  2  3  4  5  6  7
```

---

## 14.14 Real-World Patterns

```lua
-- ตัวอย่างที่ 56: assert with formatted message
function assertf(condition, fmt, ...)
    if not condition then
        error(string.format(fmt, ...), 2)
    end
    return condition
end

-- ใช้แทน assert ปกติ
local function divide(a, b)
    assertf(b ~= 0, "cannot divide %d by zero", a)
    return a / b
end

print(divide(10, 2))  -- 5.0
-- divide(10, 0)  -- error: cannot divide 10 by zero
```

```lua
-- ตัวอย่างที่ 57: type-safe call
function typed_call(fn, types, ...)
    local args = table.pack(...)
    assert(args.n == #types,
        string.format("expected %d args, got %d", #types, args.n))

    for i, expected in ipairs(types) do
        local actual = type(args[i])
        assert(actual == expected,
            string.format("arg %d: expected %s, got %s", i, expected, actual))
    end

    return fn(...)
end

local function add_strings(a, b)
    return a .. b
end

local result = typed_call(add_strings, {"string", "string"}, "Hello, ", "World!")
print(result)  -- Hello, World!

-- typed_call(add_strings, {"string", "string"}, "Hello", 42)
-- error: arg 2: expected string, got number
```

```lua
-- ตัวอย่างที่ 58: retry mechanism
function retry(fn, max_attempts, ...)
    local attempt = 0
    while attempt < max_attempts do
        attempt = attempt + 1
        local ok, result = pcall(fn, ...)
        if ok then
            return result
        end
        if attempt < max_attempts then
            print(string.format("attempt %d failed: %s, retrying...",
                attempt, tostring(result)))
        else
            error(string.format("all %d attempts failed: %s",
                max_attempts, tostring(result)))
        end
    end
end

local call_count = 0
local function flaky_function(x)
    call_count = call_count + 1
    if call_count < 3 then
        error("not ready yet")
    end
    return x * 2
end

local result = retry(flaky_function, 5, 21)
print("result:", result)  -- result: 42
```

---

## แบบฝึกหัด (Exercises)

### ระดับพื้นฐาน

**แบบฝึกหัดที่ 1:** เขียนฟังก์ชัน `divmod(a, b)` ที่คืนทั้ง quotient และ remainder

```lua
-- Template:
function divmod(a, b)
    -- TODO: คืนค่า a//b และ a%b
end

local q, r = divmod(17, 5)
print(q, r)  -- ควรได้: 3  2
```

**แบบฝึกหัดที่ 2:** เขียนฟังก์ชัน `clamp(value, min_val, max_val)` ที่คืนทั้งค่าที่ถูก clamp และ boolean ว่า clamp เกิดขึ้นหรือไม่

```lua
-- Template:
function clamp(value, min_val, max_val)
    -- TODO
end

local v, was_clamped = clamp(150, 0, 100)
print(v, was_clamped)  -- 100  true

local v2, was_clamped2 = clamp(50, 0, 100)
print(v2, was_clamped2)  -- 50  false
```

**แบบฝึกหัดที่ 3:** เขียน variadic function `average(...)` ที่คืนค่าเฉลี่ย

```lua
-- Template:
function average(...)
    -- TODO
end

print(average(1, 2, 3, 4, 5))  -- 3.0
print(average(10, 20))          -- 15.0
```

### ระดับกลาง

**แบบฝึกหัดที่ 4:** เขียนฟังก์ชัน `map(fn, ...)` ที่ apply fn กับแต่ละ argument และคืนค่าทั้งหมด

```lua
-- Template:
function map(fn, ...)
    -- TODO: คืนค่า fn applied กับทุก argument
end

print(map(function(x) return x*2 end, 1, 2, 3, 4))  -- 2  4  6  8
print(map(tostring, 1, 2.5, true))  -- "1"  "2.5"  "true"
```

**แบบฝึกหัดที่ 5:** เขียน `safe_call(fn, ...)` ที่ใช้ pcall และคืน `true, result...` หรือ `false, error_message`

**แบบฝึกหัดที่ 6:** เขียน `pipeline(...)` ที่รับ functions หลายตัวและคืน function ใหม่ที่ chain พวกมัน

```lua
-- Template:
function pipeline(...)
    -- TODO
end

local process = pipeline(
    function(x) return x * 2 end,
    function(x) return x + 10 end,
    function(x) return x / 2 end
)

print(process(5))  -- ((5*2)+10)/2 = 10.0
```

### ระดับยาก

**แบบฝึกหัดที่ 7:** เขียน `partial(fn, ...)` สำหรับ partial application

```lua
-- Template:
function partial(fn, ...)
    -- TODO: คืน function ที่ prepend ด้วย ... arguments
end

local add = function(a, b, c) return a + b + c end
local add5 = partial(add, 5)
print(add5(3, 2))  -- 10

local add5and3 = partial(add, 5, 3)
print(add5and3(2))  -- 10
```

**แบบฝึกหัดที่ 8:** เขียน `compose(...)` ที่ compose functions จากขวาไปซ้าย

```lua
-- Template:
function compose(...)
    -- TODO: f(g(h(x))) โดย compose(f, g, h)
end

local double = function(x) return x * 2 end
local inc = function(x) return x + 1 end
local square = function(x) return x * x end

local transform = compose(double, inc, square)  -- double(inc(square(x)))
print(transform(3))  -- double(inc(9)) = double(10) = 20
```

**แบบฝึกหัดที่ 9:** เขียน `interleave(...)` ที่รับ tables หลายตัวและสลับค่า

```lua
-- Template:
function interleave(...)
    -- TODO
end

print(interleave({1,2,3}, {"a","b","c"}, {true,false,true}))
-- 1  a  true  2  b  false  3  c  true
```

**แบบฝึกหัดที่ 10:** เขียน `batch_transform(data, ...)` ที่ apply functions หลายตัวกับข้อมูลและคืนผลลัพธ์ทุกตัว

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Multiple Return Values** - ฟังก์ชันสามารถคืนค่าหลายค่าได้
2. **Adjustment** - Lua ปรับจำนวนค่าโดยตัดทิ้งหรือเติม nil
3. **select()** - ใช้นับและดึง varargs
4. **table.pack()** - เก็บ varargs พร้อม .n field
5. **table.unpack()** - กระจาย table เป็น arguments
6. **Variadic Functions** - ฟังก์ชันรับ argument ไม่จำกัด
7. **Passing Through** - ส่ง ... ต่อไปยังฟังก์ชันอื่น
8. **Mixed Parameters** - ผสม fixed และ variadic params
9. **Flexible APIs** - สร้าง API ที่ยืดหยุ่น

ความสามารถเหล่านี้ทำให้ Lua เหมาะสำหรับการเขียน DSL (Domain-Specific Language) และ API ที่ expressive ได้อย่างมาก
