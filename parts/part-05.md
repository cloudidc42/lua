# บทที่ 5: การควบคุมการทำงาน (Control Flow)

## บทนำ

Control Flow คือกลไกที่ควบคุมลำดับการทำงานของโปรแกรม Lua มีโครงสร้าง control flow ครบครัน ได้แก่ if/elseif/else, while, repeat...until, numeric for, generic for, break, และ goto ซึ่งทำให้เขียนโปรแกรมได้อย่างยืดหยุ่น

---

## 5.1 Truthy และ Falsy ใน Lua

### ตัวอย่างที่ 1: ค่า Falsy ใน Lua

```lua
-- ใน Lua มีเพียง nil และ false เท่านั้นที่เป็น falsy
-- ทุกค่าอื่นล้วนเป็น truthy!

-- Falsy values
if nil then print("nil is truthy") else print("nil is falsy") end
-- nil is falsy

if false then print("false is truthy") else print("false is falsy") end
-- false is falsy

-- Truthy values (ที่อาจสร้างความสับสน)
if 0 then print("0 is truthy") end          -- 0 is truthy
if "" then print("empty string is truthy") end  -- empty string is truthy
if {} then print("empty table is truthy") end   -- empty table is truthy
if 0.0 then print("0.0 is truthy") end      -- 0.0 is truthy

-- เปรียบเทียบกับ Python/JavaScript ที่ 0 และ "" เป็น falsy
print("---")
local x = 0
if x then
    print("x =", x, "is truthy in Lua!")  -- x = 0 is truthy in Lua!
end
```

### ตัวอย่างที่ 2: ใช้ Truthy/Falsy ในการเขียนโค้ด

```lua
-- Pattern: default value ด้วย 'or'
local name = nil
local displayName = name or "Anonymous"
print(displayName)  -- Anonymous

local count = 0
local displayCount = count or 10  -- 0 เป็น truthy! ดังนั้น displayCount = 0
print(displayCount)  -- 0 (ไม่ใช่ 10!)

-- ถ้าต้องการ default สำหรับ nil เท่านั้น:
local function default(val, def)
    if val == nil then return def end
    return val
end

print(default(nil, "N/A"))  -- N/A
print(default(0, "N/A"))    -- 0
print(default("", "N/A"))   -- (empty string)

-- Pattern: short circuit evaluation
local function riskyOp(x)
    return x ~= 0 and 100 / x or nil
end
print(riskyOp(5))   -- 20.0
print(riskyOp(0))   -- nil (ไม่ได้ทำ 100/0)
```

---

## 5.2 if / elseif / else / end

### ตัวอย่างที่ 3: โครงสร้าง if พื้นฐาน

```lua
-- if เดี่ยว
local x = 10
if x > 0 then
    print("x เป็นบวก")
end
-- x เป็นบวก

-- if-else
local score = 75
if score >= 60 then
    print("ผ่าน")
else
    print("ไม่ผ่าน")
end
-- ผ่าน

-- if-elseif-else
local temp = 28
if temp >= 35 then
    print("ร้อนมาก")
elseif temp >= 25 then
    print("ร้อน")
elseif temp >= 15 then
    print("อากาศดี")
else
    print("หนาว")
end
-- ร้อน
```

### ตัวอย่างที่ 4: เกรด (หลาย elseif)

```lua
local function getGrade(score)
    if score >= 90 then
        return "A"
    elseif score >= 80 then
        return "B"
    elseif score >= 70 then
        return "C"
    elseif score >= 60 then
        return "D"
    else
        return "F"
    end
end

local scores = {95, 83, 72, 65, 40, 100, 59}
for _, s in ipairs(scores) do
    print(string.format("คะแนน %3d → เกรด %s", s, getGrade(s)))
end
-- คะแนน  95 → เกรด A
-- คะแนน  83 → เกรด B
-- คะแนน  72 → เกรด C
-- คะแนน  65 → เกรด D
-- คะแนน  40 → เกรด F
-- คะแนน 100 → เกรด A
-- คะแนน  59 → เกรด F
```

### ตัวอย่างที่ 5: Nested if

```lua
local function classify(x)
    if type(x) == "number" then
        if x > 0 then
            if x % 2 == 0 then
                return "จำนวนเต็มบวกคู่"
            else
                return "จำนวนเต็มบวกคี่"
            end
        elseif x < 0 then
            return "จำนวนลบ"
        else
            return "ศูนย์"
        end
    elseif type(x) == "string" then
        if #x == 0 then
            return "สตริงว่าง"
        else
            return "สตริงยาว " .. #x .. " ตัว"
        end
    else
        return "ประเภทอื่น: " .. type(x)
    end
end

print(classify(4))      -- จำนวนเต็มบวกคู่
print(classify(7))      -- จำนวนเต็มบวกคี่
print(classify(-3))     -- จำนวนลบ
print(classify(0))      -- ศูนย์
print(classify("Hi"))   -- สตริงยาว 2 ตัว
print(classify(""))     -- สตริงว่าง
print(classify(true))   -- ประเภทอื่น: boolean
```

### ตัวอย่างที่ 6: Guard clauses pattern

```lua
-- แทนที่จะ nest ลึกๆ ใช้ early return (guard clause)

-- แบบ nested (อ่านยาก)
local function processUserBad(user)
    if user ~= nil then
        if user.active then
            if user.age >= 18 then
                return "ประมวลผลสำเร็จ: " .. user.name
            else
                return "อายุไม่ถึง"
            end
        else
            return "user ไม่ active"
        end
    else
        return "user เป็น nil"
    end
end

-- แบบ guard clause (อ่านง่ายกว่า)
local function processUser(user)
    if user == nil then return "user เป็น nil" end
    if not user.active then return "user ไม่ active" end
    if user.age < 18 then return "อายุไม่ถึง" end
    return "ประมวลผลสำเร็จ: " .. user.name
end

print(processUser(nil))
-- user เป็น nil

print(processUser({active = false, age = 20, name = "Tom"}))
-- user ไม่ active

print(processUser({active = true, age = 16, name = "Ann"}))
-- อายุไม่ถึง

print(processUser({active = true, age = 25, name = "Bob"}))
-- ประมวลผลสำเร็จ: Bob
```

---

## 5.3 while loop

### ตัวอย่างที่ 7: while loop พื้นฐาน

```lua
-- while condition do ... end
local i = 1
while i <= 5 do
    print(i)
    i = i + 1
end
-- 1
-- 2
-- 3
-- 4
-- 5

-- ใช้ while กับ string processing
local s = "Hello, World!"
local pos = 1
local vowels = 0
while pos <= #s do
    local c = s:sub(pos, pos):lower()
    if c == "a" or c == "e" or c == "i" or c == "o" or c == "u" then
        vowels = vowels + 1
    end
    pos = pos + 1
end
print("จำนวน vowel:", vowels)  -- จำนวน vowel: 3
```

### ตัวอย่างที่ 8: while กับ input simulation

```lua
-- Simulated input processing
local inputs = {5, 3, 8, -1, 2}  -- -1 คือสัญญาณหยุด
local idx = 1
local sum = 0
local count = 0

while idx <= #inputs do
    local val = inputs[idx]
    if val == -1 then
        break  -- หยุดเมื่อพบ sentinel value
    end
    sum = sum + val
    count = count + 1
    idx = idx + 1
end

print(string.format("รับค่า %d ตัว รวม = %d เฉลี่ย = %.2f",
    count, sum, count > 0 and sum/count or 0))
-- รับค่า 3 ตัว รวม = 16 เฉลี่ย = 5.33
```

### ตัวอย่างที่ 9: Collatz conjecture

```lua
-- Collatz sequence: ถ้า n คู่ → n/2, ถ้าคี่ → 3n+1
local function collatz(n)
    local steps = 0
    print(n, "")
    while n ~= 1 do
        if n % 2 == 0 then
            n = n // 2
        else
            n = 3 * n + 1
        end
        io.write(n .. " ")
        steps = steps + 1
    end
    print()
    return steps
end

local steps = collatz(27)
print("จำนวน steps:", steps)
-- 27 82 41 124 62 31 94 47 142 71 214 107 322 161 484 ...
-- จำนวน steps: 111
```

---

## 5.4 repeat...until loop

### ตัวอย่างที่ 10: repeat...until พื้นฐาน

```lua
-- repeat...until: ทำก่อน ตรวจเงื่อนไขทีหลัง (do-while)
-- ทำงานอย่างน้อย 1 ครั้ง

local i = 1
repeat
    print(i)
    i = i + 1
until i > 5
-- 1
-- 2
-- 3
-- 4
-- 5

-- เปรียบเทียบกับ while
local j = 10  -- เริ่มที่ 10 (เกิน condition แล้ว)
while j <= 5 do
    print("while:", j)  -- ไม่ทำงานเลย
    j = j + 1
end

local k = 10
repeat
    print("repeat:", k)  -- ทำงาน 1 ครั้ง แม้เงื่อนไขไม่ตรง
    k = k + 1
until k <= 5
-- repeat: 10
```

### ตัวอย่างที่ 11: repeat ใช้กับ validation

```lua
-- Simulate user input validation
local attempts = 0
local maxAttempts = 3
local passwords = {"wrong", "wrong2", "secret123"}  -- simulate inputs

local i = 1
local success = false
repeat
    local input = passwords[i] or ""
    i = i + 1
    attempts = attempts + 1
    
    if input == "secret123" then
        success = true
        print("เข้าสู่ระบบสำเร็จ!")
    else
        print(string.format("รหัสผ่านผิด (ครั้งที่ %d/%d)", attempts, maxAttempts))
    end
until success or attempts >= maxAttempts

if not success then
    print("เข้าสู่ระบบล้มเหลว บัญชีถูกล็อค")
end
-- รหัสผ่านผิด (ครั้งที่ 1/3)
-- รหัสผ่านผิด (ครั้งที่ 2/3)
-- เข้าสู่ระบบสำเร็จ!
```

### ตัวอย่างที่ 12: Newton's method ด้วย repeat

```lua
-- Newton's method สำหรับหา square root
local function sqrt_newton(n, tolerance)
    tolerance = tolerance or 1e-10
    local x = n / 2  -- initial guess
    local iterations = 0
    
    repeat
        local prev = x
        x = (x + n / x) / 2
        iterations = iterations + 1
    until math.abs(x - prev) < tolerance
    
    return x, iterations
end

local result, iters = sqrt_newton(2)
print(string.format("√2 = %.15f", result))
print(string.format("iterations = %d", iters))
print(string.format("error = %.2e", math.abs(result - math.sqrt(2))))
-- √2 = 1.414213562373095
-- iterations = 5
-- error = 0.00e+00
```

---

## 5.5 Numeric for loop

### ตัวอย่างที่ 13: Numeric for พื้นฐาน

```lua
-- for var = start, stop, step do
-- step เป็น optional (default = 1)

-- นับขึ้น
for i = 1, 5 do
    io.write(i .. " ")
end
print()  -- 1 2 3 4 5

-- นับลง (step เป็นลบ)
for i = 5, 1, -1 do
    io.write(i .. " ")
end
print()  -- 5 4 3 2 1

-- step ที่ไม่ใช่ 1
for i = 0, 10, 2 do
    io.write(i .. " ")
end
print()  -- 0 2 4 6 8 10

-- step เป็น float
for x = 0.0, 1.0, 0.2 do
    io.write(string.format("%.1f ", x))
end
print()  -- 0.0 0.2 0.4 0.6 0.8 1.0
```

### ตัวอย่างที่ 14: loop variable เป็น local

```lua
-- ตัวแปร loop เป็น local ภายใน loop เท่านั้น
for i = 1, 3 do
    -- i เป็น local ที่นี่
    print(i)
end
-- print(i)  -- Error! i ไม่มีนอก loop

-- ข้อควรระวัง: อย่าแก้ไขตัวแปร loop
for i = 1, 5 do
    if i == 3 then
        -- i = 10  -- ทำได้ แต่ไม่ควร! อาจสร้างความสับสน
        -- Lua อนุญาตให้แก้ไข แต่ loop ยังวนต่อตามที่กำหนด
    end
    print(i)  -- ยังพิมพ์ 1,2,3,4,5 ตามปกติ (Lua copy ค่า start,stop,step)
end
```

### ตัวอย่างที่ 15: สร้าง multiplication table

```lua
-- ตาราง สูตรคูณ
print("ตารางสูตรคูณ 1-5")
io.write(string.rep(" ", 4))
for j = 1, 5 do
    io.write(string.format("%5d", j))
end
print()
print(string.rep("-", 29))

for i = 1, 5 do
    io.write(string.format("%3d|", i))
    for j = 1, 5 do
        io.write(string.format("%5d", i * j))
    end
    print()
end
--      1    2    3    4    5
-- -------------------------
--  1|    1    2    3    4    5
--  2|    2    4    6    8   10
--  3|    3    6    9   12   15
--  4|    4    8   12   16   20
--  5|    5   10   15   20   25
```

### ตัวอย่างที่ 16: สรุปผล loop

```lua
-- Sum of squares
local sum = 0
for i = 1, 100 do
    sum = sum + i * i
end
print("Σ(i²) จาก 1 ถึง 100 =", sum)  -- 338350

-- Factorial
local n = 10
local fact = 1
for i = 2, n do
    fact = fact * i
end
print(n .. "! =", fact)  -- 10! = 3628800

-- Fibonacci
local a, b = 0, 1
for i = 1, 10 do
    io.write(a .. " ")
    a, b = b, a + b
end
print()  -- 0 1 1 2 3 5 8 13 21 34
```

---

## 5.6 Generic for loop กับ pairs() และ ipairs()

### ตัวอย่างที่ 17: ipairs() — iterate array

```lua
-- ipairs(t) — วน loop ตาม index 1, 2, 3, ...
-- หยุดเมื่อพบ nil

local fruits = {"apple", "banana", "cherry", "date"}

for i, v in ipairs(fruits) do
    print(i, v)
end
-- 1  apple
-- 2  banana
-- 3  cherry
-- 4  date

-- ipairs หยุดที่ nil
local sparse = {"a", "b", nil, "d", "e"}
for i, v in ipairs(sparse) do
    print(i, v)
end
-- 1  a
-- 2  b
-- (หยุดที่ nil ไม่แสดง d, e)
```

### ตัวอย่างที่ 18: pairs() — iterate ทุก key

```lua
-- pairs(t) — วน loop ทุก key-value pair
-- ลำดับไม่แน่นอนสำหรับ hash keys

local person = {
    name = "Alice",
    age = 30,
    city = "Bangkok",
    active = true,
}

for key, value in pairs(person) do
    print(key, "=", value)
end
-- (ลำดับอาจต่างกัน)
-- name  =  Alice
-- age   =  30
-- city  =  Bangkok
-- active = true

-- pairs รวม array part ด้วย
local mixed = {10, 20, 30, x = 100, y = 200}
for k, v in pairs(mixed) do
    print(k, v)
end
-- 1  10
-- 2  20
-- 3  30
-- x  100
-- y  200
```

### ตัวอย่างที่ 19: ความต่างระหว่าง pairs และ ipairs

```lua
local t = {10, 20, nil, 40, 50, x = 99}

print("--- ipairs ---")
for i, v in ipairs(t) do
    print(i, v)
end
-- 1  10
-- 2  20
-- (หยุดที่ index 3 ซึ่งเป็น nil)

print("--- pairs ---")
for k, v in pairs(t) do
    print(k, v)
end
-- 1  10
-- 2  20
-- 4  40    (ข้าม nil แต่ยังแสดง 4, 5)
-- 5  50
-- x  99
```

### ตัวอย่างที่ 20: Generic for กับ custom iterator

```lua
-- สร้าง custom iterator
local function range(from, to, step)
    step = step or 1
    return function(_, current)
        current = current + step
        if (step > 0 and current <= to) or
           (step < 0 and current >= to) then
            return current
        end
    end, nil, from - step
end

for i in range(1, 5) do
    io.write(i .. " ")
end
print()  -- 1 2 3 4 5

for i in range(10, 1, -2) do
    io.write(i .. " ")
end
print()  -- 10 8 6 4 2
```

---

## 5.7 break statement

### ตัวอย่างที่ 21: break ออกจาก loop

```lua
-- break ออกจาก loop ใกล้สุด

-- หา element ใน array
local data = {3, 7, 2, 9, 4, 6, 1, 8}
local target = 9
local found_at = nil

for i, v in ipairs(data) do
    if v == target then
        found_at = i
        break  -- ออกจาก loop ทันที
    end
end

if found_at then
    print(string.format("พบ %d ที่ index %d", target, found_at))
else
    print("ไม่พบ")
end
-- พบ 9 ที่ index 4
```

### ตัวอย่างที่ 22: break กับ while loop

```lua
-- Infinite loop with break
local function getFirstPrime(start)
    local n = start
    while true do
        local isPrime = true
        if n < 2 then
            n = n + 1
        else
            for i = 2, math.floor(math.sqrt(n)) do
                if n % i == 0 then
                    isPrime = false
                    break  -- break ออกจาก for loop
                end
            end
            if isPrime then
                return n  -- ออกจาก while ด้วย return
            end
            n = n + 1
        end
    end
end

print(getFirstPrime(10))   -- 11
print(getFirstPrime(50))   -- 53
print(getFirstPrime(100))  -- 101
```

### ตัวอย่างที่ 23: break กับ nested loops

```lua
-- break ออกแค่ loop ใกล้สุด ไม่ใช่ทุก loop!
local found = false

for i = 1, 5 do
    for j = 1, 5 do
        if i * j == 12 then
            print(string.format("พบ: %d × %d = 12", i, j))
            found = true
            break  -- ออกแค่ inner loop
        end
    end
    if found then break end  -- ต้อง break outer loop ด้วยตัวเอง
end

-- ใช้ flag เพื่อ break หลาย loop
-- หรือใช้ goto (ดูตัวอย่างข้างล่าง)
```

---

## 5.8 goto statement (Lua 5.2+)

### ตัวอย่างที่ 24: goto พื้นฐาน

```lua
-- goto label
-- ::label:: — กำหนด label

-- Simple goto
goto skip_this
print("บรรทัดนี้จะไม่ถูก print")
::skip_this::
print("มาต่อที่นี่")  -- มาต่อที่นี่

-- วน loop ด้วย goto (เพื่อเข้าใจ concept)
local i = 0
::loop_start::
i = i + 1
if i <= 3 then
    print("i =", i)
    goto loop_start
end
-- i = 1
-- i = 2
-- i = 3
```

### ตัวอย่างที่ 25: goto สำหรับ continue pattern

```lua
-- Lua ไม่มี continue statement!
-- ใช้ goto แทน

-- พิมพ์เลขคู่ 1-10
for i = 1, 10 do
    if i % 2 ~= 0 then goto continue end  -- ข้ามเลขคี่
    print(i)
    ::continue::
end
-- 2
-- 4
-- 6
-- 8
-- 10

-- กรอง items
local items = {1, -2, 3, -4, 5, -6, 7, -8, 9, 10}
local positives = {}

for _, v in ipairs(items) do
    if v <= 0 then goto skip end  -- ข้ามค่าที่ไม่ต้องการ
    table.insert(positives, v)
    ::skip::
end
print(table.concat(positives, ", "))  -- 1, 3, 5, 7, 9, 10
```

### ตัวอย่างที่ 26: goto สำหรับ break nested loops

```lua
-- break ออกหลาย loop พร้อมกัน
local matrix = {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {3, 10, 11, 12},
    {13, 14, 15, 16},
}

local target = 7
local found_i, found_j = nil, nil

for i = 1, #matrix do
    for j = 1, #matrix[i] do
        if matrix[i][j] == target then
            found_i, found_j = i, j
            goto found  -- ข้ามออกทุก loop
        end
    end
end
::found::

if found_i then
    print(string.format("พบ %d ที่ [%d][%d]", target, found_i, found_j))
else
    print("ไม่พบ")
end
-- พบ 7 ที่ [2][3]
```

---

## 5.9 Nested Loops

### ตัวอย่างที่ 27: Nested loops พื้นฐาน

```lua
-- สร้าง pattern ด้วย nested loops

-- สี่เหลี่ยม
for i = 1, 4 do
    for j = 1, 4 do
        io.write("* ")
    end
    print()
end
-- * * * *
-- * * * *
-- * * * *
-- * * * *

-- สามเหลี่ยม
for i = 1, 5 do
    for j = 1, i do
        io.write("* ")
    end
    print()
end
-- *
-- * *
-- * * *
-- * * * *
-- * * * * *

-- สามเหลี่ยมกลับ
for i = 5, 1, -1 do
    for j = 1, i do
        io.write("* ")
    end
    print()
end
```

### ตัวอย่างที่ 28: Prime sieve (Sieve of Eratosthenes)

```lua
local function sieve(limit)
    -- สร้าง boolean array
    local is_prime = {}
    for i = 2, limit do
        is_prime[i] = true
    end
    
    -- วน loop กรอง
    for i = 2, math.floor(math.sqrt(limit)) do
        if is_prime[i] then
            for j = i * i, limit, i do
                is_prime[j] = false
            end
        end
    end
    
    -- รวบรวมผล
    local primes = {}
    for i = 2, limit do
        if is_prime[i] then
            table.insert(primes, i)
        end
    end
    return primes
end

local primes = sieve(50)
print("จำนวนเฉพาะถึง 50:")
print(table.concat(primes, ", "))
-- 2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47
print("จำนวน:", #primes)  -- 15
```

---

## 5.10 Loop Patterns

### ตัวอย่างที่ 29: Counting pattern

```lua
-- นับสิ่งต่างๆ ใน loop

local text = "Hello, World! How are you?"
local counts = {
    letters = 0,
    digits = 0,
    spaces = 0,
    other = 0,
}

for i = 1, #text do
    local c = text:sub(i, i)
    if c:match("%a") then
        counts.letters = counts.letters + 1
    elseif c:match("%d") then
        counts.digits = counts.digits + 1
    elseif c == " " then
        counts.spaces = counts.spaces + 1
    else
        counts.other = counts.other + 1
    end
end

for k, v in pairs(counts) do
    print(string.format("%-10s: %d", k, v))
end
-- letters   : 20
-- spaces    : 4
-- other     : 3
-- digits    : 0
```

### ตัวอย่างที่ 30: Summing pattern

```lua
-- หา sum, max, min ใน loop

local nums = {4, 7, 2, 9, 1, 5, 8, 3, 6}

local sum = 0
local max = nums[1]
local min = nums[1]

for _, v in ipairs(nums) do
    sum = sum + v
    if v > max then max = v end
    if v < min then min = v end
end

local avg = sum / #nums
print(string.format("Sum: %d, Min: %d, Max: %d, Avg: %.2f",
    sum, min, max, avg))
-- Sum: 45, Min: 1, Max: 9, Avg: 5.00
```

### ตัวอย่างที่ 31: Searching pattern

```lua
-- Binary search
local function binarySearch(arr, target)
    local lo, hi = 1, #arr
    while lo <= hi do
        local mid = (lo + hi) // 2
        if arr[mid] == target then
            return mid
        elseif arr[mid] < target then
            lo = mid + 1
        else
            hi = mid - 1
        end
    end
    return nil
end

local sorted = {1, 3, 5, 7, 9, 11, 13, 15, 17, 19}
print(binarySearch(sorted, 7))    -- 4
print(binarySearch(sorted, 13))   -- 7
print(binarySearch(sorted, 6))    -- nil
```

---

## 5.11 FizzBuzz

### ตัวอย่างที่ 32: FizzBuzz หลากหลายวิธี

```lua
-- วิธีที่ 1: แบบ classic
print("--- Classic FizzBuzz ---")
for i = 1, 20 do
    if i % 15 == 0 then
        print("FizzBuzz")
    elseif i % 3 == 0 then
        print("Fizz")
    elseif i % 5 == 0 then
        print("Buzz")
    else
        print(i)
    end
end

-- วิธีที่ 2: สร้าง result ก่อน
print("--- String building FizzBuzz ---")
for i = 1, 20 do
    local result = ""
    if i % 3 == 0 then result = result .. "Fizz" end
    if i % 5 == 0 then result = result .. "Buzz" end
    if result == "" then result = tostring(i) end
    io.write(result .. " ")
end
print()

-- วิธีที่ 3: table-driven (extensible)
print("--- Table-driven FizzBuzz ---")
local rules = {
    {divisor = 3, word = "Fizz"},
    {divisor = 5, word = "Buzz"},
    {divisor = 7, word = "Bang"},  -- เพิ่ม rule ได้ง่าย
}

for i = 1, 30 do
    local output = ""
    for _, rule in ipairs(rules) do
        if i % rule.divisor == 0 then
            output = output .. rule.word
        end
    end
    if output == "" then output = tostring(i) end
    io.write(output .. " ")
end
print()
```

---

## 5.12 Infinite Loops กับ break

### ตัวอย่างที่ 33: Infinite loop patterns

```lua
-- Pattern: event loop simulation
local events = {
    {type = "click", x = 10, y = 20},
    {type = "move", x = 15, y = 25},
    {type = "quit"},
    {type = "click", x = 5, y = 5},  -- ไม่ถูก process
}

local idx = 1
while true do
    if idx > #events then break end
    local event = events[idx]
    idx = idx + 1
    
    if event.type == "quit" then
        print("รับ quit event, หยุดทำงาน")
        break
    elseif event.type == "click" then
        print(string.format("Click ที่ (%d, %d)", event.x, event.y))
    elseif event.type == "move" then
        print(string.format("Move ไปยัง (%d, %d)", event.x, event.y))
    end
end
-- Click ที่ (10, 20)
-- Move ไปยัง (15, 25)
-- รับ quit event, หยุดทำงาน
```

### ตัวอย่างที่ 34: Retry pattern

```lua
-- ลองซ้ำจนกว่าจะสำเร็จหรือหมด attempt
local function simulateNetwork(attempt)
    -- Simulate: สำเร็จที่ attempt ที่ 3
    return attempt == 3
end

local MAX_RETRIES = 5
local attempt = 0
local success = false

while true do
    attempt = attempt + 1
    if attempt > MAX_RETRIES then
        print("หมด retry แล้ว")
        break
    end
    
    print(string.format("พยายามครั้งที่ %d...", attempt))
    if simulateNetwork(attempt) then
        success = true
        print("สำเร็จ!")
        break
    else
        print("  ล้มเหลว, ลองใหม่...")
    end
end

if not success then
    print("การเชื่อมต่อล้มเหลว")
end
-- พยายามครั้งที่ 1...
--   ล้มเหลว, ลองใหม่...
-- พยายามครั้งที่ 2...
--   ล้มเหลว, ลองใหม่...
-- พยายามครั้งที่ 3...
-- สำเร็จ!
```

---

## 5.13 Table Iteration Patterns

### ตัวอย่างที่ 35: iterate และ transform

```lua
-- Map: สร้าง array ใหม่จากการ transform
local function map(t, fn)
    local result = {}
    for i, v in ipairs(t) do
        result[i] = fn(v)
    end
    return result
end

local nums = {1, 2, 3, 4, 5}
local squares = map(nums, function(x) return x * x end)
print(table.concat(squares, ", "))  -- 1, 4, 9, 16, 25

-- Filter: กรอง elements
local function filter(t, pred)
    local result = {}
    for _, v in ipairs(t) do
        if pred(v) then
            table.insert(result, v)
        end
    end
    return result
end

local evens = filter(nums, function(x) return x % 2 == 0 end)
print(table.concat(evens, ", "))  -- 2, 4

-- Reduce
local function reduce(t, fn, init)
    local acc = init
    for _, v in ipairs(t) do
        acc = fn(acc, v)
    end
    return acc
end

local sum = reduce(nums, function(a, b) return a + b end, 0)
print("Sum:", sum)  -- Sum: 15
```

### ตัวอย่างที่ 36: Deep iteration ด้วย recursion

```lua
-- Iterate nested table
local function flatten(t, result)
    result = result or {}
    for _, v in ipairs(t) do
        if type(v) == "table" then
            flatten(v, result)
        else
            table.insert(result, v)
        end
    end
    return result
end

local nested = {1, {2, 3}, {4, {5, 6}}, 7}
local flat = flatten(nested)
print(table.concat(flat, ", "))  -- 1, 2, 3, 4, 5, 6, 7

-- Count items recursively
local function deepCount(t)
    local count = 0
    for _, v in pairs(t) do
        if type(v) == "table" then
            count = count + deepCount(v)
        else
            count = count + 1
        end
    end
    return count
end

local data = {a = 1, b = {c = 2, d = {e = 3, f = 4}}, g = 5}
print("จำนวน items:", deepCount(data))  -- 5
```

---

## 5.14 Pattern Matching กับ Loops

### ตัวอย่างที่ 37: Parse log file

```lua
-- Simulate log entries
local logs = {
    "[2024-01-15 10:30:25] INFO: Server started",
    "[2024-01-15 10:30:26] DEBUG: Config loaded",
    "[2024-01-15 10:31:00] ERROR: Connection failed",
    "[2024-01-15 10:31:01] WARN: Retry attempt 1",
    "[2024-01-15 10:31:05] INFO: Connection restored",
    "[2024-01-15 10:32:00] ERROR: Disk full",
}

local error_count = 0
local info_count = 0

for _, line in ipairs(logs) do
    local date, time, level, msg = line:match(
        "%[(%d+%-%d+%-%d+) (%d+:%d+:%d+)%] (%w+): (.*)"
    )
    if level == "ERROR" then
        error_count = error_count + 1
        print(string.format("[%s %s] ERROR: %s", date, time, msg))
    elseif level == "INFO" then
        info_count = info_count + 1
    end
end

print(string.format("\nสรุป: ERROR=%d, INFO=%d", error_count, info_count))
-- [2024-01-15 10:31:00] ERROR: Connection failed
-- [2024-01-15 10:32:00] ERROR: Disk full
-- สรุป: ERROR=2, INFO=2
```

---

## 5.15 ตัวอย่างรวม: Algorithms

### ตัวอย่างที่ 38: Bubble Sort

```lua
local function bubbleSort(arr)
    local n = #arr
    for i = 1, n - 1 do
        local swapped = false
        for j = 1, n - i do
            if arr[j] > arr[j + 1] then
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
                swapped = true
            end
        end
        if not swapped then break end  -- optimization
    end
    return arr
end

local data = {64, 34, 25, 12, 22, 11, 90}
local sorted = bubbleSort(data)
print(table.concat(sorted, ", "))  -- 11, 12, 22, 25, 34, 64, 90
```

### ตัวอย่างที่ 39: Selection Sort

```lua
local function selectionSort(arr)
    local n = #arr
    for i = 1, n - 1 do
        local min_idx = i
        for j = i + 1, n do
            if arr[j] < arr[min_idx] then
                min_idx = j
            end
        end
        if min_idx ~= i then
            arr[i], arr[min_idx] = arr[min_idx], arr[i]
        end
    end
    return arr
end

local data = {29, 10, 14, 37, 13}
selectionSort(data)
print(table.concat(data, ", "))  -- 10, 13, 14, 29, 37
```

### ตัวอย่างที่ 40: Number guessing game simulation

```lua
math.randomseed(42)

local function guessingGame(secret, maxGuesses)
    print(string.format("เกมทายตัวเลข 1-100 (ไม่เกิน %d ครั้ง)", maxGuesses))
    
    -- Simulate optimal binary search strategy
    local lo, hi = 1, 100
    local guesses = 0
    
    repeat
        local guess = (lo + hi) // 2
        guesses = guesses + 1
        
        io.write(string.format("ครั้งที่ %d: ทาย %d → ", guesses, guess))
        
        if guess == secret then
            print("ถูกต้อง!")
            return guesses
        elseif guess < secret then
            print("น้อยไป")
            lo = guess + 1
        else
            print("มากไป")
            hi = guess - 1
        end
    until guesses >= maxGuesses
    
    print("หมดโอกาสแล้ว! เลขคือ", secret)
    return nil
end

local result = guessingGame(73, 10)
if result then
    print("ทายถูกใน", result, "ครั้ง")
end
```

---

## แบบฝึกหัดบทที่ 5

### แบบฝึกหัดที่ 1: Pattern ใน loop
เขียนโปรแกรมพิมพ์ diamond pattern:
```
    *
   ***
  *****
 *******
*********
 *******
  *****
   ***
    *
```

### แบบฝึกหัดที่ 2: Prime checker
เขียนฟังก์ชัน `isPrime(n)` แล้วพิมพ์จำนวนเฉพาะ 10 ตัวแรกที่มากกว่า 1000

### แบบฝึกหัดที่ 3: Caesar cipher
เขียนฟังก์ชัน `caesarEncode(text, shift)` และ `caesarDecode(text, shift)` 

### แบบฝึกหัดที่ 4: Matrix multiplication
เขียนฟังก์ชันคูณ matrix 2x2 สองตัว

### แบบฝึกหัดที่ 5: Histogram
รับ array ของตัวเลข แสดง histogram แนวนอน:
```
1: *****
2: ***
3: ********
```

### เฉลยแบบฝึกหัด

```lua
-- เฉลย 1: Diamond pattern
local function diamond(n)
    -- upper half
    for i = 1, n do
        print(string.rep(" ", n - i) .. string.rep("*", 2 * i - 1))
    end
    -- lower half
    for i = n - 1, 1, -1 do
        print(string.rep(" ", n - i) .. string.rep("*", 2 * i - 1))
    end
end
diamond(5)

-- เฉลย 2: Prime checker
local function isPrime(n)
    if n < 2 then return false end
    if n == 2 then return true end
    if n % 2 == 0 then return false end
    for i = 3, math.floor(math.sqrt(n)), 2 do
        if n % i == 0 then return false end
    end
    return true
end

local count = 0
local n = 1001
while count < 10 do
    if isPrime(n) then
        io.write(n .. " ")
        count = count + 1
    end
    n = n + 1
end
print()
-- 1009 1013 1019 1021 1031 1033 1039 1049 1051 1061

-- เฉลย 3: Caesar cipher
local function caesarEncode(text, shift)
    return text:gsub("[%a]", function(c)
        local base = c:match("%u") and string.byte("A") or string.byte("a")
        return string.char((string.byte(c) - base + shift) % 26 + base)
    end)
end

local function caesarDecode(text, shift)
    return caesarEncode(text, 26 - shift)
end

local msg = "Hello, World!"
local encoded = caesarEncode(msg, 3)
local decoded = caesarDecode(encoded, 3)
print(encoded)  -- Khoor, Zruog!
print(decoded)  -- Hello, World!

-- เฉลย 4: Matrix multiplication (2x2)
local function matMul(a, b)
    return {
        {a[1][1]*b[1][1] + a[1][2]*b[2][1], a[1][1]*b[1][2] + a[1][2]*b[2][2]},
        {a[2][1]*b[1][1] + a[2][2]*b[2][1], a[2][1]*b[1][2] + a[2][2]*b[2][2]},
    }
end

local A = {{1,2},{3,4}}
local B = {{5,6},{7,8}}
local C = matMul(A, B)
print(C[1][1], C[1][2])  -- 19  22
print(C[2][1], C[2][2])  -- 43  50

-- เฉลย 5: Histogram
local function histogram(data)
    for i, count in ipairs(data) do
        print(string.format("%2d: %s", i, string.rep("*", count)))
    end
end

histogram({5, 3, 8, 2, 6})
--  1: *****
--  2: ***
--  3: ********
--  4: **
--  5: ******
```

---

## สรุปบทที่ 5

| โครงสร้าง | รูปแบบ | หมายเหตุ |
|-----------|--------|----------|
| if | `if cond then ... end` | ต้องมี `end` |
| if-else | `if cond then ... else ... end` | |
| if-elseif | `if c1 then ... elseif c2 then ... end` | ไม่ต้องมี else |
| while | `while cond do ... end` | ตรวจก่อน ทำทีหลัง |
| repeat | `repeat ... until cond` | ทำก่อน ตรวจทีหลัง |
| numeric for | `for i=s,e,step do ... end` | step default=1 |
| generic for | `for k,v in iter do ... end` | |
| break | `break` | ออก loop ใกล้สุด |
| goto | `goto label` / `::label::` | Lua 5.2+ |

**สิ่งสำคัญที่ต้องจำ:**
- `nil` และ `false` เท่านั้นที่เป็น falsy (0 และ "" เป็น truthy!)
- `ipairs` หยุดที่ nil, `pairs` iterate ทุก key
- Lua ไม่มี `continue` ใช้ `goto` แทน
- `break` ออกแค่ loop ใกล้สุด
- ใช้ `goto` เพื่อ break หลาย loop พร้อมกัน
