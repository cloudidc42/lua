# บทที่ 6: ฟังก์ชัน (Functions)

## บทนำ

ฟังก์ชันใน Lua เป็น first-class value หมายความว่าสามารถเก็บในตัวแปร, ส่งเป็น argument, คืนค่าจากฟังก์ชัน และเก็บในตาราง ได้เหมือนค่าทั่วไป ความยืดหยุ่นนี้ทำให้ Lua รองรับ functional programming patterns ได้อย่างดีเยี่ยม

---

## 6.1 การประกาศฟังก์ชัน

### ตัวอย่างที่ 1: รูปแบบการประกาศฟังก์ชัน

```lua
-- รูปแบบที่ 1: function statement
function greet(name)
    print("Hello, " .. name .. "!")
end
greet("World")  -- Hello, World!

-- รูปแบบที่ 2: function expression (assign to variable)
local sayBye = function(name)
    print("Goodbye, " .. name .. "!")
end
sayBye("Alice")  -- Goodbye, Alice!

-- รูปแบบที่ 3: local function (แนะนำใช้เมื่อไม่ต้องการ global)
local function add(a, b)
    return a + b
end
print(add(3, 4))  -- 7

-- รูปแบบที่ 4: ใน table (method)
local math_utils = {}
function math_utils.square(x)
    return x * x
end
print(math_utils.square(5))  -- 25
```

### ตัวอย่างที่ 2: ความต่างระหว่าง local function และ function

```lua
-- 'local function f()' เทียบเท่า 'local f; f = function()'
-- ทำให้ f สามารถอ้างอิงตัวเองใน recursive call ได้

-- ถูกต้อง: local function สามารถ recursive ได้
local function countdown(n)
    if n <= 0 then
        print("Go!")
        return
    end
    print(n)
    countdown(n - 1)  -- อ้างอิง countdown ได้ เพราะ 'local function'
end
countdown(3)
-- 3
-- 2
-- 1
-- Go!

-- 'local f = function()' ไม่สามารถ recursive ตรงๆ ได้ใน declaration
-- local bad = function() bad() end  -- Error! bad ยังไม่ถูก assign

-- แก้ได้โดย:
local fib
fib = function(n)
    if n <= 1 then return n end
    return fib(n - 1) + fib(n - 2)
end
print(fib(10))  -- 55
```

---

## 6.2 การเรียกใช้ฟังก์ชัน

### ตัวอย่างที่ 3: การเรียกฟังก์ชัน

```lua
local function greet(name)
    return "Hello, " .. name
end

-- เรียกแบบปกติ
print(greet("Alice"))   -- Hello, Alice

-- เรียกด้วย string literal โดยตรง (ไม่ต้องใส่วงเล็บ)
print(greet "Bob")      -- Hello, Bob (ได้ผลเหมือนกัน)

-- เรียกด้วย table literal (ไม่ต้องใส่วงเล็บ)
local function info(t)
    return t.name .. " age " .. t.age
end
print(info {name = "Charlie", age = 25})  -- Charlie age 25

-- เรียกฟังก์ชันใน expression
local result = add(2, 3) * add(4, 5)  -- (2+3) * (4+5) = 45
-- (ใช้ add จากตัวอย่างก่อนหน้า)
```

### ตัวอย่างที่ 4: Parameter และ Argument

```lua
-- parameter มากกว่า argument: parameter ส่วนเกินเป็น nil
local function demo(a, b, c)
    print(a, b, c)
end
demo(1, 2)       -- 1  2  nil
demo(1, 2, 3, 4) -- 1  2  3  (argument ส่วนเกินถูกทิ้ง)

-- ตรวจสอบ nil parameter
local function greetSafely(name, title)
    name = name or "Guest"
    title = title or "Mr/Ms"
    return string.format("สวัสดี %s %s", title, name)
end

print(greetSafely("Alice", "Ms"))   -- สวัสดี Ms Alice
print(greetSafely("Bob"))           -- สวัสดี Mr/Ms Bob
print(greetSafely())                -- สวัสดี Mr/Ms Guest
```

---

## 6.3 Return Values

### ตัวอย่างที่ 5: Single return value

```lua
local function square(x)
    return x * x
end

local result = square(5)
print(result)  -- 25

-- ฟังก์ชันที่ไม่ return ค่า → คืน nil โดยอัตโนมัติ
local function printLine()
    print(string.rep("-", 20))
    -- ไม่มี return statement
end

local x = printLine()  -- พิมพ์ --------------------
print(x)               -- nil
```

### ตัวอย่างที่ 6: Multiple return values

```lua
-- Lua สนับสนุน multiple return values โดยตรง!
local function minMax(arr)
    local min, max = arr[1], arr[1]
    for _, v in ipairs(arr) do
        if v < min then min = v end
        if v > max then max = v end
    end
    return min, max  -- คืนค่า 2 ค่า
end

local nums = {5, 3, 8, 1, 9, 2, 7}
local lo, hi = minMax(nums)
print("Min:", lo, "Max:", hi)  -- Min: 1  Max: 9

-- รับค่าบางส่วน
local justMin = minMax(nums)  -- รับแค่ค่าแรก
print("Min only:", justMin)   -- Min only: 1
```

### ตัวอย่างที่ 7: Multiple return values ใน context ต่างๆ

```lua
local function divmod(a, b)
    return a // b, a % b
end

-- รับทั้งหมด
local q, r = divmod(17, 5)
print(q, r)  -- 3  2

-- ใช้ใน expression: ใช้แค่ค่าแรก
local only_q = divmod(17, 5)
print(only_q)  -- 3

-- ใน table constructor
local t = {divmod(17, 5)}
print(t[1], t[2])  -- 3  2

-- ใน function call: ส่งทุกค่าต่อ
print(divmod(17, 5))      -- 3  2
print(divmod(17, 5), "x") -- 3  x (ค่าสุดท้ายในรายการถูก truncate ถ้าไม่ใช่ท้ายสุด)

-- วงเล็บ truncate เหลือค่าเดียว
print((divmod(17, 5)))    -- 3 (แค่ค่าแรก)
```

### ตัวอย่างที่ 8: Return multiple values แบบซับซ้อน

```lua
-- คืนค่า success + result pattern (error handling)
local function safeDivide(a, b)
    if b == 0 then
        return nil, "หารด้วยศูนย์ไม่ได้"
    end
    return a / b, nil  -- result, nil error
end

local result, err = safeDivide(10, 2)
if err then
    print("Error:", err)
else
    print("Result:", result)
end
-- Result: 5.0

result, err = safeDivide(10, 0)
if err then
    print("Error:", err)  -- Error: หารด้วยศูนย์ไม่ได้
end

-- Pattern: คืนหลายค่าสำหรับ stats
local function stats(t)
    local n = #t
    local sum = 0
    local min, max = t[1], t[1]
    
    for _, v in ipairs(t) do
        sum = sum + v
        if v < min then min = v end
        if v > max then max = v end
    end
    
    local mean = sum / n
    
    -- คำนวณ standard deviation
    local variance = 0
    for _, v in ipairs(t) do
        variance = variance + (v - mean) ^ 2
    end
    variance = variance / n
    local stddev = math.sqrt(variance)
    
    return n, sum, min, max, mean, stddev
end

local data = {2, 4, 4, 4, 5, 5, 7, 9}
local n, sum, min, max, mean, stddev = stats(data)
print(string.format("n=%d sum=%d min=%d max=%d mean=%.2f std=%.2f",
    n, sum, min, max, mean, stddev))
-- n=8 sum=40 min=2 max=9 mean=5.00 std=2.00
```

---

## 6.4 Functions เป็น First-Class Values

### ตัวอย่างที่ 9: เก็บฟังก์ชันในตัวแปร

```lua
-- ฟังก์ชันเป็นค่าเหมือนตัวเลขหรือ string
local function add(a, b) return a + b end
local function sub(a, b) return a - b end
local function mul(a, b) return a * b end
local function div(a, b) return a / b end

-- เก็บใน table
local ops = {
    ["+"] = add,
    ["-"] = sub,
    ["*"] = mul,
    ["/"] = div,
}

-- Calculator
local function calc(a, op, b)
    local fn = ops[op]
    if not fn then
        return nil, "Unknown operator: " .. op
    end
    return fn(a, b)
end

print(calc(10, "+", 5))   -- 15.0
print(calc(10, "-", 3))   -- 7.0
print(calc(4, "*", 7))    -- 28.0
print(calc(15, "/", 3))   -- 5.0
print(calc(10, "^", 2))   -- nil  Unknown operator: ^
```

### ตัวอย่างที่ 10: ส่งฟังก์ชันเป็น argument

```lua
-- Higher-order function: รับ function เป็น parameter
local function applyTwice(fn, x)
    return fn(fn(x))
end

local function double(x) return x * 2 end
local function addTen(x) return x + 10 end

print(applyTwice(double, 3))    -- 12 (3 → 6 → 12)
print(applyTwice(addTen, 5))    -- 25 (5 → 15 → 25)

-- sort กับ comparison function
local people = {
    {name = "Charlie", age = 30},
    {name = "Alice", age = 25},
    {name = "Bob", age = 35},
}

-- เรียงตาม age
table.sort(people, function(a, b) return a.age < b.age end)
for _, p in ipairs(people) do
    print(p.name, p.age)
end
-- Alice  25
-- Charlie 30
-- Bob    35

-- เรียงตาม name
table.sort(people, function(a, b) return a.name < b.name end)
for _, p in ipairs(people) do
    print(p.name)
end
-- Alice
-- Bob
-- Charlie
```

### ตัวอย่างที่ 11: คืนฟังก์ชันจากฟังก์ชัน

```lua
-- Function factory
local function multiplier(n)
    return function(x)
        return x * n
    end
end

local double = multiplier(2)
local triple = multiplier(3)
local tenTimes = multiplier(10)

print(double(5))     -- 10
print(triple(4))     -- 12
print(tenTimes(7))   -- 70

-- Adder factory
local function adder(n)
    return function(x) return x + n end
end

local add5 = adder(5)
local add100 = adder(100)

print(add5(10))    -- 15
print(add100(42))  -- 142
```

---

## 6.5 Anonymous Functions (Lambda)

### ตัวอย่างที่ 12: Anonymous functions

```lua
-- Anonymous function = ฟังก์ชันไม่มีชื่อ
-- ใช้เมื่อต้องการใช้ครั้งเดียว

-- ใน function call
local nums = {5, 3, 8, 1, 9, 2}
table.sort(nums, function(a, b) return a < b end)
print(table.concat(nums, ", "))  -- 1, 2, 3, 5, 8, 9

-- assign ทันที
local square = (function(x) return x * x end)(5)
print(square)  -- 25

-- ใน table
local operations = {
    add = function(a, b) return a + b end,
    sub = function(a, b) return a - b end,
    mul = function(a, b) return a * b end,
}

for name, fn in pairs(operations) do
    print(name, fn(10, 3))
end
-- add  13
-- sub  7
-- mul  30
```

### ตัวอย่างที่ 13: Pipeline ด้วย anonymous functions

```lua
-- Compose functions
local function compose(...)
    local fns = {...}
    return function(x)
        local result = x
        for i = #fns, 1, -1 do
            result = fns[i](result)
        end
        return result
    end
end

local process = compose(
    function(x) return x * 2 end,
    function(x) return x + 1 end,
    function(x) return x ^ 2 end
)
-- process(3) = (3^2 + 1) * 2 = (9+1)*2 = 20
print(process(3))  -- 20
print(process(5))  -- 52  ((5^2+1)*2)
```

---

## 6.6 Closures

### ตัวอย่างที่ 14: Closure พื้นฐาน

```lua
-- Closure คือ function ที่ "จำ" ตัวแปรจาก enclosing scope

local function makeCounter(start)
    start = start or 0
    local count = start  -- upvalue
    
    return {
        increment = function() count = count + 1 end,
        decrement = function() count = count - 1 end,
        reset = function() count = start end,
        get = function() return count end,
    }
end

local c1 = makeCounter()
local c2 = makeCounter(100)

c1.increment()
c1.increment()
c1.increment()
print(c1.get())  -- 3

c2.increment()
c2.decrement()
print(c2.get())  -- 100

-- c1 และ c2 มี state แยกกัน
c1.reset()
print(c1.get())  -- 0
print(c2.get())  -- 100 (ไม่เปลี่ยน)
```

### ตัวอย่างที่ 15: Closure กับ loop

```lua
-- ข้อควรระวัง: closure ใน loop share upvalue เดียวกัน!

-- แบบที่ทำให้สับสน
local funcs = {}
for i = 1, 3 do
    funcs[i] = function() return i end
end

-- ทุก function อ้างอิง i ตัวเดียวกัน
for _, f in ipairs(funcs) do
    io.write(f() .. " ")
end
print()  -- 4 4 4  (ไม่ใช่ 1 2 3!)
-- เพราะ i = 4 หลัง loop จบ (loop ใน Lua สร้าง local ใหม่ทุก iteration!)

-- หมายเหตุ: จริงๆ แล้ว Lua numeric for สร้าง local ใหม่ทุก iteration
-- ดังนั้น output จะเป็น 1 2 3 ไม่ใช่ 4 4 4!
-- ลองทดสอบ:
local funcs2 = {}
for i = 1, 3 do
    funcs2[i] = function() return i end
end
for _, f in ipairs(funcs2) do
    io.write(f() .. " ")
end
print()  -- 1 2 3 (Lua for loop สร้าง local ใหม่ทุก iteration)

-- แต่ถ้าใช้ while loop:
local funcs3 = {}
local j = 1
while j <= 3 do
    local captured = j  -- ต้องสร้าง local copy
    funcs3[j] = function() return captured end
    j = j + 1
end
for _, f in ipairs(funcs3) do
    io.write(f() .. " ")
end
print()  -- 1 2 3 (ถูกต้อง เพราะ captured เป็น local ใหม่)
```

### ตัวอย่างที่ 16: Closure สำหรับ private state

```lua
-- Closure เป็น alternative ของ OOP
local function createAccount(owner, initialBalance)
    local balance = initialBalance or 0
    local history = {}
    
    local function logTransaction(type, amount, newBalance)
        table.insert(history, {
            type = type,
            amount = amount,
            balance = newBalance,
        })
    end
    
    return {
        deposit = function(amount)
            if amount <= 0 then return false, "จำนวนต้องมากกว่า 0" end
            balance = balance + amount
            logTransaction("deposit", amount, balance)
            return true
        end,
        
        withdraw = function(amount)
            if amount <= 0 then return false, "จำนวนต้องมากกว่า 0" end
            if amount > balance then return false, "เงินไม่พอ" end
            balance = balance - amount
            logTransaction("withdraw", amount, balance)
            return true
        end,
        
        getBalance = function() return balance end,
        getOwner = function() return owner end,
        
        getHistory = function()
            local copy = {}
            for i, h in ipairs(history) do copy[i] = h end
            return copy
        end,
    }
end

local acc = createAccount("Alice", 1000)
acc.deposit(500)
acc.withdraw(200)
acc.deposit(100)
print(string.format("เจ้าของ: %s, ยอดเงิน: %d", acc.getOwner(), acc.getBalance()))
-- เจ้าของ: Alice, ยอดเงิน: 1400

for _, h in ipairs(acc.getHistory()) do
    print(string.format("  %s: %d (คงเหลือ: %d)", h.type, h.amount, h.balance))
end
-- deposit: 500 (คงเหลือ: 1500)
-- withdraw: 200 (คงเหลือ: 1300)
-- deposit: 100 (คงเหลือ: 1400)
```

---

## 6.7 Variadic Functions (...)

### ตัวอย่างที่ 17: Variadic functions พื้นฐาน

```lua
-- ... (vararg) รับ arguments จำนวนไม่จำกัด

local function sum(...)
    local total = 0
    for _, v in ipairs({...}) do
        total = total + v
    end
    return total
end

print(sum(1, 2, 3))          -- 6
print(sum(10, 20, 30, 40))   -- 100
print(sum())                  -- 0

-- ดึง ... เป็น list
local function first(...)
    return (...)  -- วงเล็บ truncate เหลือค่าแรก
end
print(first(10, 20, 30))  -- 10

-- ความยาวของ vararg
local function countArgs(...)
    return select("#", ...)
end
print(countArgs(1, 2, 3))       -- 3
print(countArgs("a", nil, "c")) -- 3 (นับ nil ด้วย)
```

### ตัวอย่างที่ 18: select() กับ variadic

```lua
-- select(n, ...) — คืน arguments ตั้งแต่ตำแหน่ง n
-- select("#", ...) — คืนจำนวน arguments ทั้งหมด

local function demo(...)
    local n = select("#", ...)
    print("จำนวน arguments:", n)
    
    for i = 1, n do
        local v = select(i, ...)
        -- select(i,...) คืน arguments ตั้งแต่ i ถึงจบ
        -- ใช้ () ครอบเพื่อเอาแค่ค่าแรก
        print(string.format("  arg[%d] = %s", i, tostring((select(i, ...)))))
    end
end

demo("apple", nil, "cherry")
-- จำนวน arguments: 3
--   arg[1] = apple
--   arg[2] = nil
--   arg[3] = cherry

-- การใช้ select เพื่อ forward arguments
local function printf(fmt, ...)
    io.write(string.format(fmt, ...))
end

printf("Hello %s, you are %d years old!\n", "Bob", 25)
-- Hello Bob, you are 25 years old!
```

### ตัวอย่างที่ 19: table.pack() และ table.unpack()

```lua
-- table.pack(...) — แปลง varargs เป็น table พร้อม .n field
local function showArgs(...)
    local args = table.pack(...)
    print("จำนวน:", args.n)
    for i = 1, args.n do
        print(string.format("  [%d] = %s", i, tostring(args[i])))
    end
end

showArgs(10, nil, 30)
-- จำนวน: 3
--   [1] = 10
--   [2] = nil
--   [3] = 30

-- table.unpack(t, i, j) — แปลง table เป็น values
local t = {10, 20, 30, 40, 50}
print(table.unpack(t))           -- 10  20  30  40  50
print(table.unpack(t, 2, 4))     -- 20  30  40 (index 2-4 เท่านั้น)

-- ใช้ unpack กับ function call
local function add3(a, b, c) return a + b + c end
local args = {10, 20, 30}
print(add3(table.unpack(args)))  -- 60
```

### ตัวอย่างที่ 20: printf-style function

```lua
-- สร้าง printf ของตัวเอง
local function printf(fmt, ...)
    io.write(string.format(fmt, ...))
end

local function println(fmt, ...)
    print(string.format(fmt, ...))
end

printf("Pi = %.4f\n", math.pi)  -- Pi = 3.1416
println("%-10s: %d", "count", 42)  -- count     : 42

-- Log function พร้อม timestamp
local function log(level, fmt, ...)
    local msg = string.format(fmt, ...)
    print(string.format("[%s] %s: %s", os.date("%H:%M:%S"), level, msg))
end

log("INFO", "Server started on port %d", 8080)
log("ERROR", "Failed to connect to %s:%d", "db.host", 5432)
```

---

## 6.8 Default Parameters Pattern

### ตัวอย่างที่ 21: Default parameters

```lua
-- Pattern 1: ใช้ 'or'
local function greet(name, greeting)
    name = name or "Guest"
    greeting = greeting or "Hello"
    return greeting .. ", " .. name .. "!"
end

print(greet())                   -- Hello, Guest!
print(greet("Alice"))            -- Hello, Alice!
print(greet("Bob", "Hi"))        -- Hi, Bob!

-- ข้อควรระวัง: 'or' ไม่ทำงานกับ false หรือ 0
local function increment(x, amount)
    amount = amount or 1  -- ถ้า amount = 0 จะได้ 1 (ผิด!)
    return x + amount
end
print(increment(10))     -- 11
print(increment(10, 5))  -- 15
print(increment(10, 0))  -- 11 (ผิด! ควรได้ 10)

-- Pattern 2: ใช้ nil check ที่ถูกต้อง
local function increment2(x, amount)
    if amount == nil then amount = 1 end
    return x + amount
end
print(increment2(10, 0))  -- 10 (ถูกต้อง)
```

---

## 6.9 Named Parameters กับ Table

### ตัวอย่างที่ 22: Named parameters

```lua
-- ใช้ table เป็น parameter เดียวเพื่อรับค่าแบบ named

local function createWindow(options)
    options = options or {}
    local title = options.title or "Untitled"
    local width = options.width or 800
    local height = options.height or 600
    local x = options.x or 0
    local y = options.y or 0
    local resizable = options.resizable
    if resizable == nil then resizable = true end
    
    return string.format(
        "Window('%s', %dx%d at (%d,%d), resizable=%s)",
        title, width, height, x, y, tostring(resizable)
    )
end

-- เรียกใช้แบบ named (ลำดับไม่สำคัญ)
print(createWindow())
-- Window('Untitled', 800x600 at (0,0), resizable=true)

print(createWindow {title = "My App", width = 1024, height = 768})
-- Window('My App', 1024x768 at (0,0), resizable=true)

print(createWindow {x = 100, y = 50, resizable = false, title = "Game"})
-- Window('Game', 800x600 at (100,50), resizable=false)
```

### ตัวอย่างที่ 23: Configuration pattern

```lua
-- Pattern สำหรับ library configuration
local defaultConfig = {
    timeout = 30,
    retries = 3,
    verbose = false,
    encoding = "utf-8",
    maxSize = 1024 * 1024,  -- 1MB
}

local function merge(default, override)
    local result = {}
    for k, v in pairs(default) do
        result[k] = v
    end
    if override then
        for k, v in pairs(override) do
            result[k] = v
        end
    end
    return result
end

local function connect(host, port, options)
    local config = merge(defaultConfig, options)
    
    return string.format(
        "Connecting to %s:%d (timeout=%ds, retries=%d, verbose=%s)",
        host, port, config.timeout, config.retries, tostring(config.verbose)
    )
end

print(connect("localhost", 5432))
-- Connecting to localhost:5432 (timeout=30s, retries=3, verbose=false)

print(connect("db.example.com", 5432, {timeout = 60, verbose = true}))
-- Connecting to db.example.com:5432 (timeout=60s, retries=3, verbose=true)
```

---

## 6.10 Recursive Functions

### ตัวอย่างที่ 24: Factorial

```lua
-- Recursive factorial
local function factorial(n)
    if n <= 1 then return 1 end
    return n * factorial(n - 1)
end

for i = 0, 10 do
    print(string.format("%2d! = %d", i, factorial(i)))
end
-- 0! = 1
-- 1! = 1
-- 2! = 2
-- 3! = 6
-- 4! = 24
-- 5! = 120
-- 6! = 720
-- 7! = 5040
-- 8! = 40320
-- 9! = 362880
-- 10! = 3628800

-- Iterative factorial (สำหรับ n ใหญ่)
local function factIter(n)
    local result = 1
    for i = 2, n do
        result = result * i
    end
    return result
end

print(factIter(20))  -- 2432902008176640000
```

### ตัวอย่างที่ 25: Fibonacci

```lua
-- Naive recursive (ช้า - O(2^n))
local function fib(n)
    if n <= 1 then return n end
    return fib(n-1) + fib(n-2)
end

-- พิมพ์ Fibonacci 15 ตัวแรก
for i = 0, 14 do
    io.write(fib(i) .. " ")
end
print()
-- 0 1 1 2 3 5 8 13 21 34 55 89 144 233 377

-- Memoized recursive (เร็วกว่ามาก)
local memo = {}
local function fibMemo(n)
    if n <= 1 then return n end
    if memo[n] then return memo[n] end
    memo[n] = fibMemo(n-1) + fibMemo(n-2)
    return memo[n]
end

print(fibMemo(50))  -- 12586269025

-- Iterative (เร็วที่สุด)
local function fibIter(n)
    if n <= 1 then return n end
    local a, b = 0, 1
    for _ = 2, n do
        a, b = b, a + b
    end
    return b
end

print(fibIter(100))  -- 354224848179261915075 (อาจ overflow ใน integer)
```

### ตัวอย่างที่ 26: Tower of Hanoi

```lua
-- Tower of Hanoi
local moves = 0

local function hanoi(n, from, to, via)
    if n == 0 then return end
    hanoi(n - 1, from, via, to)
    moves = moves + 1
    print(string.format("Move disk %d: %s → %s", n, from, to))
    hanoi(n - 1, via, to, from)
end

hanoi(3, "A", "C", "B")
print(string.format("\nจำนวน moves: %d (ควรเป็น %d)", moves, 2^3 - 1))
-- Move disk 1: A → C
-- Move disk 2: A → B
-- Move disk 1: C → B
-- Move disk 3: A → C
-- Move disk 1: B → A
-- Move disk 2: B → C
-- Move disk 1: A → C
-- จำนวน moves: 7 (ควรเป็น 7)
```

### ตัวอย่างที่ 27: Quicksort recursive

```lua
local function quicksort(arr, lo, hi)
    lo = lo or 1
    hi = hi or #arr
    
    if lo >= hi then return end
    
    -- Partition
    local pivot = arr[hi]
    local i = lo - 1
    
    for j = lo, hi - 1 do
        if arr[j] <= pivot then
            i = i + 1
            arr[i], arr[j] = arr[j], arr[i]
        end
    end
    
    arr[i+1], arr[hi] = arr[hi], arr[i+1]
    local pi = i + 1
    
    quicksort(arr, lo, pi - 1)
    quicksort(arr, pi + 1, hi)
end

local data = {64, 34, 25, 12, 22, 11, 90, 1, 55}
quicksort(data)
print(table.concat(data, ", "))  -- 1, 11, 12, 22, 25, 34, 55, 64, 90
```

---

## 6.11 Tail Calls และ Tail Recursion

### ตัวอย่างที่ 28: Tail call optimization (TCO)

```lua
-- Lua 5.1+ รองรับ proper tail calls
-- Tail call: การ call ที่เป็น action สุดท้ายของฟังก์ชัน

-- ไม่ใช่ tail call (ต้องทำ * n หลัง recursive call)
local function factNonTail(n)
    if n <= 1 then return 1 end
    return n * factNonTail(n - 1)  -- ไม่ใช่ tail call
end

-- Tail recursive (ใช้ accumulator)
local function factTail(n, acc)
    acc = acc or 1
    if n <= 1 then return acc end
    return factTail(n - 1, n * acc)  -- tail call!
end

print(factNonTail(10))  -- 3628800
print(factTail(10))     -- 3628800

-- Tail recursive fibonacci
local function fibTail(n, a, b)
    a = a or 0
    b = b or 1
    if n == 0 then return a end
    if n == 1 then return b end
    return fibTail(n - 1, b, a + b)  -- tail call
end

print(fibTail(20))   -- 6765
print(fibTail(50))   -- 12586269025

-- Deep tail recursion ไม่ stack overflow
local function countDown(n)
    if n <= 0 then return "done" end
    return countDown(n - 1)  -- tail call
end
print(countDown(1000000))  -- done (ไม่ stack overflow!)
```

---

## 6.12 Method Syntax (Colon Notation)

### ตัวอย่างที่ 29: Colon syntax

```lua
-- obj:method(args) เทียบเท่า obj.method(obj, args)
-- Colon syntax ส่ง self โดยอัตโนมัติ

local Dog = {}
Dog.__index = Dog

function Dog.new(name, breed)
    return setmetatable({name = name, breed = breed, tricks = {}}, Dog)
end

-- ประกาศด้วย colon (รับ self อัตโนมัติ)
function Dog:bark()
    print(self.name .. " says: Woof!")
end

function Dog:learn(trick)
    table.insert(self.tricks, trick)
    print(self.name .. " learned: " .. trick)
end

function Dog:showTricks()
    if #self.tricks == 0 then
        print(self.name .. " doesn't know any tricks")
        return
    end
    print(self.name .. "'s tricks:")
    for i, trick in ipairs(self.tricks) do
        print(string.format("  %d. %s", i, trick))
    end
end

local rex = Dog.new("Rex", "German Shepherd")
rex:bark()
rex:learn("sit")
rex:learn("shake")
rex:learn("roll over")
rex:showTricks()
-- Rex says: Woof!
-- Rex learned: sit
-- Rex learned: shake
-- Rex learned: roll over
-- Rex's tricks:
--   1. sit
--   2. shake
--   3. roll over
```

### ตัวอย่างที่ 30: เปรียบเทียบ dot vs colon

```lua
local obj = {value = 42}

-- dot syntax: ต้องส่ง self เอง
function obj.getValueDot(self)
    return self.value
end

-- colon syntax: self ถูกส่งอัตโนมัติ
function obj:getValueColon()
    return self.value
end

-- เรียกใช้:
print(obj.getValueDot(obj))  -- 42 (ส่ง obj เป็น self)
print(obj:getValueColon())   -- 42 (colon ส่ง obj อัตโนมัติ)

-- ข้อควรระวัง: เรียกผิดวิธี
-- print(obj.getValueColon(obj))   -- OK (ส่ง obj เป็น arg แรก)
-- print(obj:getValueDot())         -- Error? (self จะได้รับ obj แต่ขาด parameter)
```

---

## 6.13 Higher-Order Functions

### ตัวอย่างที่ 31: map, filter, reduce

```lua
-- map: แปลงทุก element
local function map(t, fn)
    local result = {}
    for i, v in ipairs(t) do
        result[i] = fn(v)
    end
    return result
end

-- filter: กรอง elements
local function filter(t, pred)
    local result = {}
    for _, v in ipairs(t) do
        if pred(v) then
            table.insert(result, v)
        end
    end
    return result
end

-- reduce: รวม elements เป็นค่าเดียว
local function reduce(t, fn, init)
    local acc = init
    for _, v in ipairs(t) do
        acc = fn(acc, v)
    end
    return acc
end

local numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

-- หาผลรวมของกำลังสองของเลขคู่
local result = reduce(
    map(
        filter(numbers, function(x) return x % 2 == 0 end),
        function(x) return x * x end
    ),
    function(a, b) return a + b end,
    0
)
print(result)  -- 220 (4+16+36+64+100)
```

### ตัวอย่างที่ 32: Partial application

```lua
-- Partial application: ตรึง arguments บางส่วน

local function partial(fn, ...)
    local bound = {...}
    return function(...)
        local args = {}
        for _, v in ipairs(bound) do
            table.insert(args, v)
        end
        for i = 1, select("#", ...) do
            table.insert(args, select(i, ...))
        end
        return fn(table.unpack(args))
    end
end

local function add(a, b) return a + b end
local add10 = partial(add, 10)

print(add10(5))    -- 15
print(add10(20))   -- 30

-- ตัวอย่างที่ใช้ได้จริง
local function multiply(a, b) return a * b end
local double = partial(multiply, 2)
local triple = partial(multiply, 3)

local nums = {1, 2, 3, 4, 5}
local doubled = map(nums, double)
local tripled = map(nums, triple)

print(table.concat(doubled, ", "))  -- 2, 4, 6, 8, 10
print(table.concat(tripled, ", "))  -- 3, 6, 9, 12, 15
```

### ตัวอย่างที่ 33: Currying

```lua
-- Currying: แปลง f(a,b,c) → f(a)(b)(c)

local function curry(fn, arity)
    arity = arity or 2  -- default arity = 2
    
    local function curried(args)
        if #args >= arity then
            return fn(table.unpack(args))
        end
        return function(...)
            local newArgs = {}
            for _, v in ipairs(args) do
                table.insert(newArgs, v)
            end
            for i = 1, select("#", ...) do
                table.insert(newArgs, select(i, ...))
            end
            return curried(newArgs)
        end
    end
    
    return curried({})
end

local curriedAdd = curry(function(a, b) return a + b end)
print(curriedAdd(3)(4))   -- 7

local add5 = curriedAdd(5)
print(add5(10))   -- 15
print(add5(20))   -- 25

-- Curried map
local curriedMap = curry(function(fn, t) return map(t, fn) end)
local squareAll = curriedMap(function(x) return x * x end)
print(table.concat(squareAll({1,2,3,4,5}), ", "))  -- 1, 4, 9, 16, 25
```

---

## 6.14 Memoization

### ตัวอย่างที่ 34: Memoization pattern

```lua
-- Memoization: cache ผลลัพธ์เพื่อไม่ต้องคำนวณซ้ำ

local function memoize(fn)
    local cache = {}
    return function(...)
        -- สร้าง cache key จาก arguments
        local key = table.concat({...}, ",")
        if cache[key] == nil then
            cache[key] = fn(...)
        end
        return cache[key]
    end
end

-- Slow fibonacci (exponential)
local callCount = 0
local slowFib = memoize(function(n)
    callCount = callCount + 1
    if n <= 1 then return n end
    -- ต้องใช้ reference ที่ถูก memoize
    -- (จะ define ข้างล่าง)
    return n  -- placeholder
end)

-- Memoized fibonacci ที่ถูกต้อง
local fibCache = {}
local function memoFib(n)
    if fibCache[n] then return fibCache[n] end
    if n <= 1 then
        fibCache[n] = n
    else
        fibCache[n] = memoFib(n-1) + memoFib(n-2)
    end
    return fibCache[n]
end

-- วัดเวลา
local t1 = os.clock()
for i = 1, 1000 do memoFib(40) end
local t2 = os.clock()
print(string.format("Memoized fib(40) = %d (%.4f sec)", memoFib(40), t2-t1))
-- Memoized fib(40) = 102334155 (very fast)
```

### ตัวอย่างที่ 35: Generic memoize พร้อม TTL

```lua
-- Memoize พร้อม Time-To-Live (TTL) expiry
local function memoizeTTL(fn, ttl)
    local cache = {}
    
    return function(...)
        local key = ""
        for i = 1, select("#", ...) do
            key = key .. tostring(select(i, ...)) .. ","
        end
        
        local entry = cache[key]
        local now = os.time()
        
        if entry and (not ttl or now - entry.time < ttl) then
            return entry.value
        end
        
        local result = fn(...)
        cache[key] = {value = result, time = now}
        return result
    end
end

-- Simulate expensive calculation
local expensiveCalc = memoizeTTL(function(x)
    -- simulate work
    local sum = 0
    for i = 1, 1000000 do sum = sum + i end
    return sum + x
end, 60)  -- TTL = 60 seconds

local t1 = os.clock()
local r1 = expensiveCalc(100)
local t2 = os.clock()

local r2 = expensiveCalc(100)  -- จาก cache
local t3 = os.clock()

print(string.format("ครั้งแรก: %.4f sec, result = %d", t2-t1, r1))
print(string.format("จาก cache: %.6f sec, result = %d", t3-t2, r2))
```

---

## 6.15 ตัวอย่างรวม: Functional Programming Utilities

### ตัวอย่างที่ 36: Pipe operator pattern

```lua
-- สร้าง pipe function
local function pipe(...)
    local fns = {...}
    return function(x)
        local result = x
        for _, fn in ipairs(fns) do
            result = fn(result)
        end
        return result
    end
end

local process = pipe(
    function(x) return x * 2 end,     -- double
    function(x) return x + 10 end,    -- add 10
    function(x) return x * x end,     -- square
    tostring                           -- to string
)

print(process(5))   -- "400" ((5*2+10)^2 = 20^2 = 400)
print(process(3))   -- "256" ((3*2+10)^2 = 16^2 = 256)
```

### ตัวอย่างที่ 37: Iterator factory

```lua
-- สร้าง iterator สำหรับ range, filter, map

local function rangeIter(from, to, step)
    step = step or 1
    local i = from - step
    return function()
        i = i + step
        if (step > 0 and i <= to) or (step < 0 and i >= to) then
            return i
        end
    end
end

local function filterIter(iter, pred)
    return function()
        while true do
            local v = iter()
            if v == nil then return nil end
            if pred(v) then return v end
        end
    end
end

local function mapIter(iter, fn)
    return function()
        local v = iter()
        if v == nil then return nil end
        return fn(v)
    end
end

-- ใช้งาน: หาผลรวมของ squares ของเลขคู่ 1-10
local iter = filterIter(
    rangeIter(1, 10),
    function(x) return x % 2 == 0 end
)

local sum = 0
for v in mapIter(iter, function(x) return x * x end) do
    sum = sum + v
end
print("Sum of squares of evens 1-10:", sum)  -- 220
```

### ตัวอย่างที่ 38: Observer pattern ด้วย functions

```lua
local function createEventEmitter()
    local listeners = {}
    
    return {
        on = function(event, fn)
            listeners[event] = listeners[event] or {}
            table.insert(listeners[event], fn)
        end,
        
        emit = function(event, ...)
            if listeners[event] then
                for _, fn in ipairs(listeners[event]) do
                    fn(...)
                end
            end
        end,
        
        off = function(event, fn)
            if not listeners[event] then return end
            for i, f in ipairs(listeners[event]) do
                if f == fn then
                    table.remove(listeners[event], i)
                    return
                end
            end
        end,
    }
end

local emitter = createEventEmitter()

-- Register listeners
emitter.on("data", function(x) print("Listener 1:", x) end)
emitter.on("data", function(x) print("Listener 2:", x * 2) end)
emitter.on("error", function(msg) print("Error:", msg) end)

-- Emit events
emitter.emit("data", 42)
-- Listener 1: 42
-- Listener 2: 84

emitter.emit("error", "Connection failed")
-- Error: Connection failed
```

### ตัวอย่างที่ 39: Coroutine-like generator

```lua
-- Simple generator ด้วย closures
local function fibonacci()
    local a, b = 0, 1
    return function()
        local val = a
        a, b = b, a + b
        return val
    end
end

local fib = fibonacci()
for i = 1, 10 do
    io.write(fib() .. " ")
end
print()
-- 0 1 1 2 3 5 8 13 21 34

-- Infinite counter
local function counter(start, step)
    start = start or 0
    step = step or 1
    local current = start - step
    return function()
        current = current + step
        return current
    end
end

local c = counter(1, 2)  -- เลขคี่
for i = 1, 5 do
    io.write(c() .. " ")
end
print()  -- 1 3 5 7 9
```

### ตัวอย่างที่ 40: Function composition สร้าง validator

```lua
-- Validator functions
local function required(value)
    if value == nil or value == "" then
        return nil, "ต้องระบุค่า"
    end
    return value
end

local function minLength(min)
    return function(value)
        if type(value) ~= "string" then
            return nil, "ต้องเป็น string"
        end
        if #value < min then
            return nil, string.format("ต้องมีความยาวอย่างน้อย %d ตัว", min)
        end
        return value
    end
end

local function maxLength(max)
    return function(value)
        if #value > max then
            return nil, string.format("ต้องมีความยาวไม่เกิน %d ตัว", max)
        end
        return value
    end
end

local function matches(pattern, msg)
    return function(value)
        if not value:match(pattern) then
            return nil, msg or "รูปแบบไม่ถูกต้อง"
        end
        return value
    end
end

-- Compose validators
local function validate(value, ...)
    local validators = {...}
    local current = value
    for _, v in ipairs(validators) do
        local result, err = v(current)
        if err then return nil, err end
        current = result
    end
    return current
end

-- ทดสอบ
local function testValidate(value, ...)
    local result, err = validate(value, ...)
    if err then
        print(string.format("  ✗ %q: %s", tostring(value), err))
    else
        print(string.format("  ✓ %q: ผ่าน", result))
    end
end

print("ทดสอบ username:")
testValidate("alice", required, minLength(3), maxLength(20))
testValidate("ab", required, minLength(3), maxLength(20))
testValidate("", required, minLength(3), maxLength(20))
testValidate("a_very_long_username_that_exceeds_limit", required, minLength(3), maxLength(20))

print("\nทดสอบ email:")
local emailValidator = matches("[%w%.]+@[%w%.]+%.[%a]+", "รูปแบบ email ไม่ถูกต้อง")
testValidate("user@example.com", required, emailValidator)
testValidate("not-an-email", required, emailValidator)
```

### ตัวอย่างที่ 41: Strategy pattern

```lua
-- Strategy pattern: เปลี่ยน algorithm ได้ตอน runtime

local function createSorter(strategy)
    strategy = strategy or "bubble"
    
    local strategies = {
        bubble = function(arr)
            local n = #arr
            for i = 1, n-1 do
                for j = 1, n-i do
                    if arr[j] > arr[j+1] then
                        arr[j], arr[j+1] = arr[j+1], arr[j]
                    end
                end
            end
        end,
        
        insertion = function(arr)
            for i = 2, #arr do
                local key = arr[i]
                local j = i - 1
                while j > 0 and arr[j] > key do
                    arr[j+1] = arr[j]
                    j = j - 1
                end
                arr[j+1] = key
            end
        end,
        
        builtin = function(arr)
            table.sort(arr)
        end,
    }
    
    local fn = strategies[strategy]
    if not fn then error("Unknown strategy: " .. strategy) end
    
    return function(arr)
        local copy = {}
        for i, v in ipairs(arr) do copy[i] = v end
        fn(copy)
        return copy
    end
end

local data = {5, 3, 8, 1, 9, 2, 7, 4, 6}

for _, strategy in ipairs({"bubble", "insertion", "builtin"}) do
    local sorter = createSorter(strategy)
    local sorted = sorter(data)
    print(strategy .. ": " .. table.concat(sorted, ", "))
end
-- bubble:    1, 2, 3, 4, 5, 6, 7, 8, 9
-- insertion: 1, 2, 3, 4, 5, 6, 7, 8, 9
-- builtin:   1, 2, 3, 4, 5, 6, 7, 8, 9
```

### ตัวอย่างที่ 42: Decorator pattern

```lua
-- Decorator: เพิ่มพฤติกรรมโดยไม่แก้ไข function เดิม

-- Timing decorator
local function timed(fn, name)
    name = name or "function"
    return function(...)
        local t1 = os.clock()
        local results = {fn(...)}
        local t2 = os.clock()
        print(string.format("[TIMER] %s took %.6f sec", name, t2-t1))
        return table.unpack(results)
    end
end

-- Logging decorator
local function logged(fn, name)
    name = name or "function"
    return function(...)
        local args = {...}
        local argStr = table.concat(
            (function()
                local parts = {}
                for _, v in ipairs(args) do
                    table.insert(parts, tostring(v))
                end
                return parts
            end)(), ", "
        )
        print(string.format("[LOG] %s(%s)", name, argStr))
        local results = {fn(...)}
        print(string.format("[LOG] %s -> %s", name, tostring(results[1])))
        return table.unpack(results)
    end
end

-- Apply decorators
local function slowAdd(a, b)
    -- simulate slow operation
    return a + b
end

local decoratedAdd = logged(timed(slowAdd, "slowAdd"), "slowAdd")
local result = decoratedAdd(10, 20)
print("Result:", result)
-- [LOG] slowAdd(10, 20)
-- [TIMER] slowAdd took 0.000001 sec
-- [LOG] slowAdd -> 30
-- Result: 30
```

### ตัวอย่างที่ 43: เขียน JSON serializer ด้วย recursive functions

```lua
local function toJSON(value, indent, currentIndent)
    indent = indent or ""
    currentIndent = currentIndent or ""
    local nextIndent = currentIndent .. indent
    
    local t = type(value)
    
    if t == "nil" then
        return "null"
    elseif t == "boolean" then
        return tostring(value)
    elseif t == "number" then
        if math.type(value) == "integer" then
            return tostring(value)
        else
            return string.format("%.10g", value)
        end
    elseif t == "string" then
        -- Escape special characters
        local escaped = value
            :gsub("\\", "\\\\")
            :gsub('"', '\\"')
            :gsub("\n", "\\n")
            :gsub("\r", "\\r")
            :gsub("\t", "\\t")
        return '"' .. escaped .. '"'
    elseif t == "table" then
        -- Check if array-like
        local isArray = true
        local maxN = 0
        for k, _ in pairs(value) do
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                isArray = false
                break
            end
            if k > maxN then maxN = k end
        end
        isArray = isArray and maxN == #value
        
        if isArray then
            if #value == 0 then return "[]" end
            local items = {}
            for _, v in ipairs(value) do
                table.insert(items, nextIndent .. toJSON(v, indent, nextIndent))
            end
            if indent == "" then
                return "[" .. table.concat(items, ",") .. "]"
            else
                return "[\n" .. table.concat(items, ",\n") .. "\n" .. currentIndent .. "]"
            end
        else
            local keys = {}
            for k in pairs(value) do
                if type(k) == "string" then
                    table.insert(keys, k)
                end
            end
            table.sort(keys)
            
            if #keys == 0 then return "{}" end
            
            local items = {}
            for _, k in ipairs(keys) do
                local entry = nextIndent .. '"' .. k .. '": ' .. toJSON(value[k], indent, nextIndent)
                table.insert(items, entry)
            end
            
            if indent == "" then
                return "{" .. table.concat(items, ",") .. "}"
            else
                return "{\n" .. table.concat(items, ",\n") .. "\n" .. currentIndent .. "}"
            end
        end
    else
        return '"[' .. t .. ']"'
    end
end

-- ทดสอบ
local data = {
    name = "Alice",
    age = 30,
    scores = {95, 87, 92},
    address = {city = "Bangkok", zip = "10110"},
    active = true,
    notes = nil,
}

print(toJSON(data, "  "))
-- {
--   "active": true,
--   "address": {
--     "city": "Bangkok",
--     "zip": "10110"
--   },
--   "age": 30,
--   "name": "Alice",
--   "scores": [
--     95,
--     87,
--     92
--   ]
-- }
```

### ตัวอย่างที่ 44: State machine ด้วย functions

```lua
-- Finite State Machine
local function createFSM(initialState, transitions)
    local state = initialState
    local history = {state}
    
    return {
        getState = function() return state end,
        
        transition = function(event)
            local stateTransitions = transitions[state]
            if not stateTransitions then
                return false, "No transitions from state: " .. state
            end
            
            local newState = stateTransitions[event]
            if not newState then
                return false, string.format(
                    "No transition '%s' from state '%s'", event, state
                )
            end
            
            state = newState
            table.insert(history, state)
            return true
        end,
        
        getHistory = function()
            local copy = {}
            for i, s in ipairs(history) do copy[i] = s end
            return copy
        end,
    }
end

-- Traffic light FSM
local trafficLight = createFSM("red", {
    red = {next = "green"},
    green = {next = "yellow"},
    yellow = {next = "red"},
})

print("Current:", trafficLight.getState())  -- red
trafficLight.transition("next")
print("After next:", trafficLight.getState())  -- green
trafficLight.transition("next")
print("After next:", trafficLight.getState())  -- yellow
trafficLight.transition("next")
print("After next:", trafficLight.getState())  -- red

print("History:", table.concat(trafficLight.getHistory(), " → "))
-- History: red → green → yellow → red

-- Door FSM
local door = createFSM("closed", {
    closed = {open = "open", lock = "locked"},
    open = {close = "closed"},
    locked = {unlock = "closed"},
})

local ok, err = door.transition("open")
print("opened:", door.getState())    -- open
ok, err = door.transition("close")
print("closed:", door.getState())    -- closed
ok, err = door.transition("lock")
print("locked:", door.getState())    -- locked
ok, err = door.transition("open")   -- ไม่สามารถเปิดเมื่อล็อค
print("try open locked door:", ok, err)
-- opened: open
-- closed: closed
-- locked: locked
-- try open locked door: false  No transition 'open' from state 'locked'
```

### ตัวอย่างที่ 45: Function pipeline สำหรับ data processing

```lua
-- Data processing pipeline
local function createPipeline(...)
    local stages = {...}
    
    return function(input)
        local data = input
        for i, stage in ipairs(stages) do
            local success, result = pcall(stage, data)
            if not success then
                return nil, string.format("Stage %d failed: %s", i, result)
            end
            data = result
        end
        return data
    end
end

-- Stages
local function parseCSV(line)
    local fields = {}
    for f in (line .. ","):gmatch("([^,]*),") do
        table.insert(fields, f)
    end
    return fields
end

local function validateFields(fields)
    if #fields < 3 then
        error("ต้องมีอย่างน้อย 3 fields")
    end
    return fields
end

local function convertTypes(fields)
    return {
        name = fields[1],
        age = tonumber(fields[2]),
        score = tonumber(fields[3]),
    }
end

local function enrichData(record)
    record.grade = record.score >= 90 and "A"
                or record.score >= 80 and "B"
                or record.score >= 70 and "C"
                or "D"
    record.isAdult = record.age >= 18
    return record
end

-- สร้าง pipeline
local processRecord = createPipeline(
    parseCSV,
    validateFields,
    convertTypes,
    enrichData
)

-- ประมวลผล
local testData = {
    "Alice,25,92",
    "Bob,17,85",
    "Charlie,30,68",
    "Dave,22",       -- invalid: too few fields
}

for _, line in ipairs(testData) do
    local result, err = processRecord(line)
    if err then
        print(string.format("Error processing %q: %s", line, err))
    else
        print(string.format("%-8s age=%d score=%d grade=%s adult=%s",
            result.name, result.age, result.score,
            result.grade, tostring(result.isAdult)))
    end
end
-- Alice    age=25 score=92 grade=A adult=true
-- Bob      age=17 score=85 grade=B adult=false
-- Charlie  age=30 score=68 grade=D adult=true
-- Error processing "Dave,22": Stage 2 failed: ...ต้องมีอย่างน้อย 3 fields
```

---

## แบบฝึกหัดบทที่ 6

### แบบฝึกหัดที่ 1: Higher-order functions
เขียนฟังก์ชัน `zip(t1, t2)` ที่รวม 2 arrays เป็น array ของ pairs:
```
zip({1,2,3}, {"a","b","c"}) → {{1,"a"},{2,"b"},{3,"c"}}
```

### แบบฝึกหัดที่ 2: Memoization
เขียน `memoize(fn)` generic ที่รองรับ arguments หลายประเภท แล้ว memoize ฟังก์ชัน `expensiveCalc(n)` ที่คำนวณ `sum(1..n)`

### แบบฝึกหัดที่ 3: Recursion
เขียนฟังก์ชัน `deepCopy(t)` ที่ทำ deep copy ของ nested table

### แบบฝึกหัดที่ 4: Closures
เขียน `makeTimer()` ที่คืน object มี method `start()`, `stop()`, `elapsed()`, `reset()`

### แบบฝึกหัดที่ 5: Variadic
เขียนฟังก์ชัน `printf_table(...)` ที่รับ format string และ table แล้ว format ด้วย named placeholders:
```
printf_table("Hello {name}, you are {age}!", {name="Alice", age=30})
→ Hello Alice, you are 30!
```

### เฉลยแบบฝึกหัด

```lua
-- เฉลย 1: zip
local function zip(t1, t2)
    local result = {}
    local n = math.min(#t1, #t2)
    for i = 1, n do
        result[i] = {t1[i], t2[i]}
    end
    return result
end

local zipped = zip({1,2,3}, {"a","b","c"})
for _, pair in ipairs(zipped) do
    print(pair[1], pair[2])
end
-- 1  a
-- 2  b
-- 3  c

-- เฉลย 2: Memoization
local function memoize(fn)
    local cache = {}
    local SENTINEL = {}  -- unique key สำหรับ single arg
    
    return function(...)
        local n = select("#", ...)
        local key
        if n == 1 then
            local v = select(1, ...)
            key = tostring(v)
        else
            local parts = {}
            for i = 1, n do
                parts[i] = tostring(select(i, ...))
            end
            key = table.concat(parts, "\0")
        end
        
        if cache[key] == nil then
            cache[key] = fn(...)
        end
        return cache[key]
    end
end

local expensiveCalc = memoize(function(n)
    local sum = 0
    for i = 1, n do sum = sum + i end
    return sum
end)

print(expensiveCalc(1000))   -- 500500
print(expensiveCalc(1000))   -- 500500 (จาก cache)
print(expensiveCalc(100))    -- 5050

-- เฉลย 3: deepCopy
local function deepCopy(t)
    if type(t) ~= "table" then return t end
    local copy = {}
    for k, v in pairs(t) do
        copy[deepCopy(k)] = deepCopy(v)
    end
    return setmetatable(copy, getmetatable(t))
end

local original = {a = 1, b = {c = 2, d = {e = 3}}}
local copy = deepCopy(original)
copy.b.c = 99  -- แก้ copy ไม่กระทบ original
print(original.b.c)  -- 2 (ไม่เปลี่ยน)
print(copy.b.c)      -- 99

-- เฉลย 4: makeTimer
local function makeTimer()
    local startTime = nil
    local elapsed = 0
    local running = false
    
    return {
        start = function()
            if not running then
                startTime = os.clock()
                running = true
            end
        end,
        
        stop = function()
            if running then
                elapsed = elapsed + (os.clock() - startTime)
                running = false
            end
        end,
        
        elapsed = function()
            if running then
                return elapsed + (os.clock() - startTime)
            end
            return elapsed
        end,
        
        reset = function()
            elapsed = 0
            startTime = running and os.clock() or nil
        end,
    }
end

local timer = makeTimer()
timer.start()
-- simulate work
local x = 0
for i = 1, 1000000 do x = x + i end
timer.stop()
print(string.format("Elapsed: %.4f sec", timer.elapsed()))

-- เฉลย 5: printf_table
local function printf_table(fmt, data)
    return (fmt:gsub("{(%w+)}", function(key)
        local v = data[key]
        if v == nil then return "{" .. key .. "}" end
        return tostring(v)
    end))
end

print(printf_table("Hello {name}, you are {age}!", {name="Alice", age=30}))
-- Hello Alice, you are 30!

print(printf_table("Server: {host}:{port}", {host="localhost", port=8080}))
-- Server: localhost:8080
```

---

## สรุปบทที่ 6

| แนวคิด | รายละเอียด |
|--------|------------|
| Function declaration | `function name()` หรือ `local function name()` |
| Function expression | `local f = function() end` |
| Multiple return | `return a, b, c` — คืนค่าหลายค่าพร้อมกัน |
| First-class | เก็บใน variable, ส่งเป็น arg, คืนจาก function |
| Anonymous | `function(x) return x end` |
| Closure | function + upvalues จาก enclosing scope |
| Variadic | `function f(...) local args = {...} end` |
| select | `select(n, ...)` และ `select("#", ...)` |
| table.pack/unpack | แปลงระหว่าง varargs และ table |
| Tail call | `return f(...)` — ไม่สร้าง stack frame ใหม่ |
| Method syntax | `obj:method()` ส่ง obj เป็น self อัตโนมัติ |
| Higher-order | map, filter, reduce, compose, pipe |
| Memoization | cache ผลลัพธ์ไว้ใน table |
| Recursion | factorial, fibonacci, quicksort, TOH |

**หลักการสำคัญ:**
- ฟังก์ชันใน Lua เป็น first-class values เหมือน number หรือ string
- Closures จำ upvalues จาก scope ที่สร้างขึ้น
- Multiple return values เป็น feature ที่ทรงพลังใน Lua
- Tail calls ไม่สร้าง stack frame ใหม่ (Proper Tail Calls)
- ใช้ `...` และ `select` สำหรับ variadic functions
- `table.pack` / `table.unpack` ช่วยจัดการ varargs ได้สะดวก
