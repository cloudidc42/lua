# บทที่ 3: ตัวดำเนินการและนิพจน์

## สารบัญ

1. [Arithmetic Operators](#arithmetic-operators)
2. [Integer vs Float Division](#integer-vs-float-division)
3. [Modulo กับตัวเลขติดลบ](#modulo-กับตัวเลขติดลบ)
4. [Power Operator](#power-operator)
5. [Relational Operators](#relational-operators)
6. [Logical Operators](#logical-operators)
7. [Short-circuit Evaluation](#short-circuit-evaluation)
8. [String Concatenation Operator](#string-concatenation-operator)
9. [Length Operator](#length-operator)
10. [Bitwise Operators (Lua 5.3+)](#bitwise-operators-lua-53)
11. [Operator Precedence](#operator-precedence)
12. [Ternary-like Idiom](#ternary-like-idiom)
13. [Math Expressions](#math-expressions)
14. [String to Number Operations](#string-to-number-operations)
15. [Complex Expressions](#complex-expressions)
16. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Arithmetic Operators

### ตัวดำเนินการคณิตศาสตร์พื้นฐาน

Lua มีตัวดำเนินการคณิตศาสตร์ดังนี้:

| ตัวดำเนินการ | ชื่อ                  | ตัวอย่าง         | ผลลัพธ์     |
|-------------|----------------------|-----------------|------------|
| `+`         | บวก (Addition)        | `10 + 3`        | `13`       |
| `-`         | ลบ (Subtraction)      | `10 - 3`        | `7`        |
| `*`         | คูณ (Multiplication)  | `10 * 3`        | `30`       |
| `/`         | หาร Float (Division)  | `10 / 3`        | `3.3333...`|
| `//`        | หาร Floor (Floor div) | `10 // 3`       | `3`        |
| `%`         | Modulo (Remainder)    | `10 % 3`        | `1`        |
| `^`         | ยกกำลัง (Power)       | `2 ^ 10`        | `1024.0`   |
| `-`         | Negate (Unary minus)  | `-x`            | ค่าติดลบ   |

```lua
-- ตัวอย่างที่ 1: ตัวดำเนินการคณิตศาสตร์พื้นฐาน
local a = 10
local b = 3

print("=== Arithmetic Operators ===")
print("a = " .. a .. ", b = " .. b)
print()

print("a + b  = " .. (a + b))   -- 13
print("a - b  = " .. (a - b))   -- 7
print("a * b  = " .. (a * b))   -- 30
print("a / b  = " .. (a / b))   -- 3.3333333333333
print("a // b = " .. (a // b))  -- 3
print("a % b  = " .. (a % b))   -- 1
print("a ^ b  = " .. (a ^ b))   -- 1000.0
print("-a     = " .. (-a))       -- -10
```

### ตัวอย่างการใช้งานจริง

```lua
-- ตัวอย่างที่ 2: การใช้งานในชีวิตจริง
-- คำนวณราคาสินค้า

local price = 850.0      -- ราคาก่อนภาษี (บาท)
local taxRate = 0.07     -- ภาษีมูลค่าเพิ่ม 7%
local discount = 50.0    -- ส่วนลด (บาท)
local quantity = 3       -- จำนวน

local subtotal = price * quantity
local discountAmount = discount * quantity
local afterDiscount = subtotal - discountAmount
local tax = afterDiscount * taxRate
local total = afterDiscount + tax

print("=== ใบเสร็จรับเงิน ===")
print(string.format("ราคาต่อหน่วย:     %10.2f บาท", price))
print(string.format("จำนวน:            %10d ชิ้น", quantity))
print(string.format("รวมก่อนลด:        %10.2f บาท", subtotal))
print(string.format("ส่วนลด:           %10.2f บาท", -discountAmount))
print(string.format("หลังหักส่วนลด:    %10.2f บาท", afterDiscount))
print(string.format("ภาษี %.0f%%:        %10.2f บาท", taxRate*100, tax))
print(string.rep("-", 35))
print(string.format("รวมทั้งหมด:       %10.2f บาท", total))
```

### ตัวดำเนินการกับ Integer vs Float

```lua
-- ตัวอย่างที่ 3: Integer และ Float arithmetic
-- Lua 5.3+ แยก integer และ float

local i1 = 10       -- integer
local i2 = 3        -- integer
local f1 = 10.0     -- float
local f2 = 3.0      -- float

-- Integer + Integer = Integer
print(math.type(i1 + i2))   -- integer
print(i1 + i2)               -- 13

-- Float + Float = Float
print(math.type(f1 + f2))   -- float
print(f1 + f2)               -- 13.0

-- Integer + Float = Float (promotion)
print(math.type(i1 + f2))   -- float
print(i1 + f2)               -- 13.0

-- / ผลลัพธ์เป็น Float เสมอ
print(math.type(i1 / i2))   -- float
print(i1 / i2)               -- 3.3333333333333

-- // กับ integer ผลลัพธ์เป็น integer
print(math.type(i1 // i2))  -- integer
print(i1 // i2)              -- 3

-- // กับ float ผลลัพธ์เป็น float
print(math.type(f1 // f2))  -- float
print(f1 // f2)              -- 3.0
```

### Unary Minus

```lua
-- ตัวอย่างที่ 4: Unary minus operator
local x = 42
local y = -x
local z = -3.14

print(x)     -- 42
print(y)     -- -42
print(z)     -- -3.14
print(-y)    -- 42 (double negation)
print(-(x + y))  -- 0

-- Unary minus กับ integer/float
print(math.type(-42))    -- integer
print(math.type(-42.0))  -- float

-- ใช้กับนิพจน์
local a, b = 5, 3
print(-(a * b))  -- -15
print(-a * -b)   -- 15 (คูณสองค่าลบ = บวก)
```

---

## Integer vs Float Division

### Float Division (/)

```lua
-- ตัวอย่างที่ 5: Float division (/)
-- / เสมอคืนค่า float

print(10 / 2)    -- 5.0 (ไม่ใช่ 5)
print(10 / 3)    -- 3.3333333333333
print(7 / 2)     -- 3.5
print(-7 / 2)    -- -3.5
print(1 / 3)     -- 0.33333333333333

-- ตรวจสอบ type
print(math.type(10 / 2))   -- float (ไม่ใช่ integer!)
print(math.type(10 / 2.0)) -- float

-- Division by zero
print(1 / 0)    -- inf
print(-1 / 0)   -- -inf
print(0 / 0)    -- -nan
```

### Floor Division (//)

```lua
-- ตัวอย่างที่ 6: Floor division (//)
-- // คืนค่า floor (ปัดลง) ของผลหาร

print(10 // 3)     -- 3   (floor(3.333) = 3)
print(10 // -3)    -- -4  (floor(-3.333) = -4 ไม่ใช่ -3!)
print(-10 // 3)    -- -4  (floor(-3.333) = -4)
print(-10 // -3)   -- 3   (floor(3.333) = 3)

-- เปรียบเทียบกับ truncation
print(math.floor(10/3))    -- 3  (เหมือน //)
print(math.floor(-10/3))   -- -4 (เหมือน //)

-- // กับ float
print(10.0 // 3.0)   -- 3.0 (float result)
print(10.5 // 3.0)   -- 3.0

-- ใช้ทำอะไรได้บ้าง
-- 1. แบ่งหน้าเว็บ
local items = 47
local perPage = 10
local pages = (items + perPage - 1) // perPage
print("จำนวนหน้า: " .. pages)  -- 5

-- 2. แปลงวินาทีเป็นชั่วโมง:นาที:วินาที
local totalSeconds = 3723
local hours = totalSeconds // 3600
local minutes = (totalSeconds % 3600) // 60
local seconds = totalSeconds % 60
print(string.format("เวลา: %02d:%02d:%02d", hours, minutes, seconds))
-- 01:02:03
```

### เปรียบเทียบ / กับ //

```lua
-- ตัวอย่างที่ 7: เปรียบเทียบ / และ //
local pairs_list = {
    {7, 2},
    {-7, 2},
    {7, -2},
    {-7, -2},
    {10, 3},
    {-10, 3},
}

print(string.format("%-8s %-8s %-12s %-12s", "a", "b", "a/b (float)", "a//b (floor)"))
print(string.rep("-", 45))
for _, p in ipairs(pairs_list) do
    local a, b = p[1], p[2]
    print(string.format("%-8d %-8d %-12.4f %-12d", a, b, a/b, a//b))
end
```

---

## Modulo กับตัวเลขติดลบ

### Modulo พื้นฐาน

```lua
-- ตัวอย่างที่ 8: Modulo operator (%)
-- Lua ใช้นิยามแบบ floor: a % b = a - floor(a/b)*b

print("=== Modulo ===")
print(10 % 3)    -- 1
print(10 % 4)    -- 2
print(10 % 5)    -- 0
print(10 % 1)    -- 0

-- ใช้ตรวจสอบเลขคี่/คู่
for i = 1, 10 do
    if i % 2 == 0 then
        io.write(i .. "(คู่) ")
    else
        io.write(i .. "(คี่) ")
    end
end
print()
```

### Modulo กับตัวเลขติดลบ

```lua
-- ตัวอย่างที่ 9: Modulo กับตัวเลขติดลบ
-- ผลลัพธ์มีเครื่องหมายเหมือน divisor (b) ไม่ใช่ dividend (a)

print("=== Modulo กับค่าติดลบ ===")
print(" 10 %  3 = " .. (10 % 3))    -- 1  (positive)
print("-10 %  3 = " .. (-10 % 3))   -- 2  (positive - เหมือน b)
print(" 10 % -3 = " .. (10 % -3))   -- -2 (negative - เหมือน b)
print("-10 % -3 = " .. (-10 % -3))  -- -1 (negative - เหมือน b)

-- เปรียบเทียบกับภาษาอื่น (C/Java ใช้ truncation modulo)
-- ใน C: -10 % 3 = -1  (ต่างกับ Lua ที่ได้ 2!)
-- ใน Lua: -10 % 3 = 2

-- สูตร: a % b = a - math.floor(a/b) * b
local function myMod(a, b)
    return a - math.floor(a/b) * b
end

print("\nตรวจสอบสูตร:")
print(myMod(10, 3))    -- 1
print(myMod(-10, 3))   -- 2
print(myMod(10, -3))   -- -2
print(myMod(-10, -3))  -- -1
```

### การใช้งาน Modulo จริง

```lua
-- ตัวอย่างที่ 10: การใช้งาน modulo ในชีวิตจริง

-- 1. วนรอบ (circular index)
local days = {"อาทิตย์", "จันทร์", "อังคาร", "พุธ", "พฤหัส", "ศุกร์", "เสาร์"}
local today = 3  -- วันพุธ (0-indexed)

for i = 0, 13 do
    local dayIndex = (today + i) % 7 + 1
    print("อีก " .. i .. " วัน: " .. days[dayIndex])
end
```

```lua
-- ตัวอย่างที่ 11: Modulo สำหรับการแบ่งกลุ่ม
-- แบ่งนักเรียน 25 คนเป็น 4 กลุ่ม

print("=== การแบ่งกลุ่ม ===")
local students = 25
local groups = 4

for i = 1, students do
    local group = (i - 1) % groups + 1
    io.write(string.format("นักเรียน %2d → กลุ่ม %d\n", i, group))
end

-- สรุป
print("\nสรุป:")
for g = 1, groups do
    local count = math.floor((students - g) / groups) + 1
    print("กลุ่ม " .. g .. ": " .. count .. " คน")
end
```

```lua
-- ตัวอย่างที่ 12: ตรวจสอบ divisibility
local function isDivisibleBy(n, d)
    return n % d == 0
end

local function fizzBuzz(n)
    if isDivisibleBy(n, 15) then
        return "FizzBuzz"
    elseif isDivisibleBy(n, 3) then
        return "Fizz"
    elseif isDivisibleBy(n, 5) then
        return "Buzz"
    else
        return tostring(n)
    end
end

print("=== FizzBuzz 1-30 ===")
for i = 1, 30 do
    io.write(fizzBuzz(i) .. " ")
    if i % 10 == 0 then print() end
end
```

---

## Power Operator

### ยกกำลังด้วย ^

```lua
-- ตัวอย่างที่ 13: Power operator (^)
-- ^ เสมอคืนค่า float

print("=== Power Operator ===")
print(2 ^ 10)     -- 1024.0
print(2 ^ 0)      -- 1.0
print(2 ^ -1)     -- 0.5 (negative power)
print(4 ^ 0.5)    -- 2.0 (square root)
print(8 ^ (1/3))  -- 2.0 (cube root)
print((-2) ^ 3)   -- -8.0

-- type ของผลลัพธ์
print(math.type(2 ^ 10))   -- float (เสมอ)
print(math.type(2 ^ 10 + 0))  -- float

-- เปรียบเทียบกับ math.pow (ใช้ ^ แทน)
-- math.pow(2, 10) -- deprecated ใน Lua 5.3
print(2 ^ 10)     -- ใช้ ^ แทน

-- ยกกำลังหลายชั้น (right-associative)
print(2 ^ 2 ^ 3)     -- 2^(2^3) = 2^8 = 256.0 (ไม่ใช่ (2^2)^3 = 64)
print((2 ^ 2) ^ 3)   -- 64.0
```

### การใช้งาน Power จริง

```lua
-- ตัวอย่างที่ 14: การใช้งาน power
-- 1. ดอกเบี้ยทบต้น (Compound Interest)
local principal = 10000  -- เงินต้น (บาท)
local rate = 0.05        -- ดอกเบี้ย 5% ต่อปี
local years = 10         -- 10 ปี

-- A = P(1 + r)^n
local amount = principal * (1 + rate) ^ years
print(string.format("เงินต้น: %.2f บาท", principal))
print(string.format("ดอกเบี้ย: %.0f%% ต่อปี", rate * 100))
print(string.format("ระยะเวลา: %d ปี", years))
print(string.format("เงินสุดท้าย: %.2f บาท", amount))
print(string.format("ดอกเบี้ยที่ได้: %.2f บาท", amount - principal))

-- 2. ตารางการเติบโต
print("\n=== ตารางการเติบโต ===")
for y = 1, years do
    local a = principal * (1 + rate) ^ y
    print(string.format("ปีที่ %2d: %.2f บาท (เพิ่มขึ้น %.2f%%)",
        y, a, (a/principal - 1) * 100))
end
```

```lua
-- ตัวอย่างที่ 15: Power ในการคำนวณทางวิทยาศาสตร์
-- ระยะทางแบบ Euclidean ใน n มิติ

local function distance(p1, p2)
    local sum = 0
    for i = 1, #p1 do
        sum = sum + (p1[i] - p2[i]) ^ 2
    end
    return math.sqrt(sum)
end

-- 2D distance
local A = {0, 0}
local B = {3, 4}
print("ระยะ A-B (2D): " .. distance(A, B))  -- 5.0

-- 3D distance
local C = {0, 0, 0}
local D = {1, 2, 2}
print("ระยะ C-D (3D): " .. distance(C, D))  -- 3.0

-- Quadratic formula: x = (-b ± sqrt(b² - 4ac)) / 2a
local function quadratic(a, b, c)
    local discriminant = b^2 - 4*a*c
    if discriminant < 0 then
        return nil, "ไม่มีรากจริง"
    elseif discriminant == 0 then
        return -b / (2*a)
    else
        local x1 = (-b + math.sqrt(discriminant)) / (2*a)
        local x2 = (-b - math.sqrt(discriminant)) / (2*a)
        return x1, x2
    end
end

-- x² - 5x + 6 = 0  (roots: 3, 2)
local x1, x2 = quadratic(1, -5, 6)
print(string.format("x² - 5x + 6 = 0: x = %.1f หรือ %.1f", x1, x2))

-- x² + 1 = 0 (no real roots)
local root, err = quadratic(1, 0, 1)
if err then print(err) end
```

---

## Relational Operators

### ตัวดำเนินการเปรียบเทียบ

```lua
-- ตัวอย่างที่ 16: Relational operators
-- ==  เท่ากับ
-- ~=  ไม่เท่ากับ (ต่างจาก != ในภาษาอื่น)
-- <   น้อยกว่า
-- >   มากกว่า
-- <=  น้อยกว่าหรือเท่ากับ
-- >=  มากกว่าหรือเท่ากับ

local a = 10
local b = 20
local c = 10

print("=== Relational Operators ===")
print("a=" .. a .. ", b=" .. b .. ", c=" .. c)
print()

print("a == c:  " .. tostring(a == c))   -- true
print("a == b:  " .. tostring(a == b))   -- false
print("a ~= b:  " .. tostring(a ~= b))   -- true
print("a ~= c:  " .. tostring(a ~= c))   -- false
print("a < b:   " .. tostring(a < b))    -- true
print("a > b:   " .. tostring(a > b))    -- false
print("a <= c:  " .. tostring(a <= c))   -- true
print("a >= b:  " .. tostring(a >= b))   -- false
```

### Equality Rules

```lua
-- ตัวอย่างที่ 17: กฎการเปรียบเทียบความเท่ากัน
print("=== Equality Rules ===")

-- Numbers
print(1 == 1.0)      -- true (integer == float ถ้าค่าเท่ากัน)
print(1 == 1)        -- true
print(1.0 == 1.0)    -- true

-- Strings
print("hello" == "hello")  -- true
print("Hello" == "hello")  -- false (case sensitive)
print("1" == 1)            -- false (ต่างชนิด!)

-- Booleans
print(true == true)   -- true
print(false == false) -- true
print(true == false)  -- false
print(true == 1)      -- false (true ≠ 1 ใน Lua!)

-- nil
print(nil == nil)     -- true
print(nil == false)   -- false (nil ≠ false!)

-- Tables (reference equality)
local t1 = {1, 2, 3}
local t2 = {1, 2, 3}
local t3 = t1

print(t1 == t2)  -- false (คนละ object)
print(t1 == t3)  -- true (reference เดียวกัน)
```

### String Comparison

```lua
-- ตัวอย่างที่ 18: String comparison (lexicographic)
print("=== String Comparison ===")

-- ตามลำดับ lexicographic (ASCII/Unicode)
print("apple" < "banana")   -- true  (a < b)
print("banana" < "apple")   -- false
print("apple" < "apple")    -- false
print("apple" <= "apple")   -- true
print("Apple" < "apple")    -- true (A=65 < a=97 ใน ASCII)

-- การเรียง string
local words = {"cherry", "apple", "banana", "date", "elderberry"}
table.sort(words)
print("เรียงตามตัวอักษร:")
for _, w in ipairs(words) do
    io.write(w .. " ")
end
print()

-- เรียงแบบไม่สนใจ case
table.sort(words, function(a, b)
    return a:lower() < b:lower()
end)
print("เรียงไม่สนใจ case:")
for _, w in ipairs(words) do
    io.write(w .. " ")
end
print()
```

### Comparison กับ nil

```lua
-- ตัวอย่างที่ 19: การเปรียบเทียบกับ nil
local x = nil
local y = 0
local z = false

-- == กับ nil
print(x == nil)    -- true
print(y == nil)    -- false (0 ≠ nil!)
print(z == nil)    -- false (false ≠ nil!)

-- ตรวจสอบ nil แบบถูกต้อง
if x == nil then
    print("x เป็น nil")
end

-- ~= nil ใช้ตรวจว่ามีค่า
if y ~= nil then
    print("y มีค่า: " .. y)
end

-- Idiomatic way
if x then
    print("x มีค่าและเป็น truthy")
else
    print("x เป็น nil หรือ false")
end
```

---

## Logical Operators

### and, or, not

```lua
-- ตัวอย่างที่ 20: Logical operators
-- and: คืนค่า falsy ตัวแรก หรือค่าสุดท้ายถ้าทั้งหมด truthy
-- or:  คืนค่า truthy ตัวแรก หรือค่าสุดท้ายถ้าทั้งหมด falsy
-- not: คืน true/false (boolean จริงๆ)

print("=== Logical Operators ===")

-- and
print(true and true)    -- true
print(true and false)   -- false
print(false and true)   -- false
print(false and false)  -- false

-- or
print(true or true)     -- true
print(true or false)    -- true
print(false or true)    -- true
print(false or false)   -- false

-- not
print(not true)         -- false
print(not false)        -- true
print(not nil)          -- true
print(not 0)            -- false (0 เป็น truthy!)
print(not "")           -- false ("" เป็น truthy!)
```

### Logical Operators คืนค่าจริง (ไม่ใช่ Boolean)

```lua
-- ตัวอย่างที่ 21: and/or คืนค่าจริง ไม่ใช่แค่ true/false
-- นี่คือความแตกต่างสำคัญของ Lua!

-- and: คืน operand แรกถ้า falsy, ไม่เช่นนั้นคืน operand ที่สอง
print(1 and 2)       -- 2 (1 เป็น truthy, คืน 2)
print(false and 2)   -- false (false เป็น falsy, คืน false)
print(nil and 2)     -- nil (nil เป็น falsy, คืน nil)
print("a" and "b")   -- b
print(0 and "zero")  -- zero (0 เป็น truthy ใน Lua!)

-- or: คืน operand แรกถ้า truthy, ไม่เช่นนั้นคืน operand ที่สอง
print(1 or 2)        -- 1 (1 เป็น truthy, คืน 1)
print(false or 2)    -- 2 (false เป็น falsy, ลองต่อ)
print(nil or "def")  -- def
print("a" or "b")    -- a
print(false or nil)  -- nil (ทั้งคู่ falsy, คืนตัวสุดท้าย)

-- not เสมอคืน boolean จริงๆ
print(not 0)         -- false (boolean)
print(not nil)       -- true (boolean)
print(type(not 0))   -- boolean
```

---

## Short-circuit Evaluation

### and Short-circuits เมื่อ Falsy

```lua
-- ตัวอย่างที่ 22: Short-circuit evaluation
-- and หยุดประเมินเมื่อพบ falsy
-- or หยุดประเมินเมื่อพบ truthy

local function check(name, value)
    print("ประเมิน " .. name)
    return value
end

print("--- and short-circuit ---")
-- ถ้า check("A", false) เป็น false → ไม่ประเมิน B
local result = check("A", false) and check("B", true)
print("ผลลัพธ์: " .. tostring(result))
-- Output: ประเมิน A   (ไม่เห็น "ประเมิน B")
-- ผลลัพธ์: false

print()
print("--- or short-circuit ---")
-- ถ้า check("A", true) เป็น true → ไม่ประเมิน B
result = check("A", true) or check("B", false)
print("ผลลัพธ์: " .. tostring(result))
-- Output: ประเมิน A   (ไม่เห็น "ประเมิน B")
-- ผลลัพธ์: true
```

### ประโยชน์ของ Short-circuit

```lua
-- ตัวอย่างที่ 23: ประโยชน์ของ short-circuit evaluation

-- 1. Safe navigation (nil check)
local user = {profile = {name = "สมชาย"}}
-- ไม่ต้องกลัว error ถ้า user หรือ profile เป็น nil
local name = user and user.profile and user.profile.name
print("ชื่อ: " .. tostring(name))  -- ชื่อ: สมชาย

local noUser = nil
local safeName = noUser and noUser.profile and noUser.profile.name
print("ชื่อ (no user): " .. tostring(safeName))  -- nil

-- 2. Default values
local config = {}
local timeout = config.timeout or 30
local host = config.host or "localhost"
print("timeout: " .. timeout)  -- 30
print("host: " .. host)        -- localhost

-- 3. Guard conditions
local function divide(a, b)
    return b ~= 0 and a / b or error("หารด้วยศูนย์ไม่ได้")
end

local ok, result = pcall(divide, 10, 2)
if ok then print("10/2 = " .. result) end

ok, result = pcall(divide, 10, 0)
if not ok then print("Error: " .. result) end
```

### or สำหรับ Default Values

```lua
-- ตัวอย่างที่ 24: or สำหรับ default values (pattern ที่ใช้บ่อยมาก)
local function greet(name, greeting)
    name = name or "ผู้มาเยือน"        -- default
    greeting = greeting or "สวัสดี"     -- default
    return greeting .. " " .. name .. "!"
end

print(greet())                    -- สวัสดี ผู้มาเยือน!
print(greet("สมชาย"))             -- สวัสดี สมชาย!
print(greet("สมหญิง", "ยินดีต้อนรับ"))  -- ยินดีต้อนรับ สมหญิง!

-- ระวัง! ถ้า parameter สามารถเป็น false หรือ 0 ได้
local function setFlag(value)
    -- ผิด! ถ้า value = false จะได้ true แทน
    -- value = value or true
    
    -- ถูกต้อง
    if value == nil then value = true end
    return value
end

print(setFlag(nil))    -- true (correct)
print(setFlag(false))  -- false (correct)
print(setFlag(true))   -- true (correct)
```

---

## String Concatenation Operator

### .. operator

```lua
-- ตัวอย่างที่ 25: String concatenation (..)
local s1 = "Hello"
local s2 = "World"

print(s1 .. ", " .. s2 .. "!")  -- Hello, World!

-- .. กับตัวเลข (auto-convert)
local n = 42
print("ตัวเลข: " .. n)      -- ตัวเลข: 42
print(n .. " บาท")           -- 42 บาท

-- ต้องระวัง: เว้นช่องว่างระหว่าง .. กับตัวเลข
-- เพราะ Lua parser อาจ confused กับ decimal point
local x = 10
print(x .. "!")      -- 10!
print(10 .. "!")     -- 10! (ต้องมีช่องว่างหลัง 10)
-- print(10.."!")    -- อาจ error ใน Lua รุ่นเก่า

-- .. เป็น right-associative
local result = "a" .. "b" .. "c" .. "d"
-- ประมวลผลเป็น: "a" .. ("b" .. ("c" .. "d"))
print(result)  -- abcd
```

### Concatenation Performance

```lua
-- ตัวอย่างที่ 26: Performance ของ string concatenation
-- ใน Lua, string เป็น immutable
-- การ concatenate ใน loop ขนาดใหญ่ช้า

-- วิธีที่ไม่ดี (สร้าง string ใหม่ทุกครั้ง)
local function buildStringBad(n)
    local result = ""
    for i = 1, n do
        result = result .. tostring(i) .. ","
    end
    return result
end

-- วิธีที่ดี (สร้าง string ครั้งเดียวตอนจบ)
local function buildStringGood(n)
    local parts = {}
    for i = 1, n do
        parts[i] = tostring(i)
    end
    return table.concat(parts, ",")
end

-- เปรียบเทียบเวลา (สำหรับ n ใหญ่ เห็นความต่างชัด)
local n = 1000

local t1 = os.clock()
local bad = buildStringBad(n)
local t2 = os.clock()
local good = buildStringGood(n)
local t3 = os.clock()

print(string.format("วิธีไม่ดี:  %.4f วินาที", t2-t1))
print(string.format("วิธีดี:     %.4f วินาที", t3-t2))
print("ผลลัพธ์เหมือนกัน: " .. tostring(bad == good))
```

---

## Length Operator

### # operator

```lua
-- ตัวอย่างที่ 27: Length operator (#)
-- สำหรับ string: คืนจำนวน bytes
-- สำหรับ table sequence: คืนจำนวน elements

-- String
print(#"hello")        -- 5
print(#"")             -- 0
print(#"สวัสดี")       -- 18 (6 ตัวอักษร x 3 bytes/char)

local s = "Hello, World!"
print(#s)              -- 13

-- Table (sequence)
local t = {10, 20, 30, 40, 50}
print(#t)  -- 5

-- เพิ่ม element
t[6] = 60
print(#t)  -- 6

-- Table ที่ไม่ใช่ sequence (undefined behavior)
local sparse = {[1] = "a", [3] = "c", [5] = "e"}  -- gaps!
print(#sparse)  -- 1 หรือ 3 หรือ 5 (undefined!)
-- ไม่ควรใช้ # กับ sparse tables

-- ลบ element ด้วย nil
t[6] = nil
print(#t)  -- 5 (กลับมา 5)
```

### ความระวังใน # operator

```lua
-- ตัวอย่างที่ 28: ข้อควรระวังของ # operator

-- 1. Sparse tables
local sparse = {}
sparse[1] = "a"
sparse[3] = "c"  -- ข้ามดัชนี 2
sparse[5] = "e"

-- ผลลัพธ์ undefined สำหรับ sparse tables!
print("#sparse = " .. #sparse)  -- อาจได้ 1, 3, หรือ 5

-- 2. Mixed tables
local mixed = {10, 20, 30, key="value", 40}
print("#mixed = " .. #mixed)  -- 4 (นับแค่ sequence part)

-- 3. ใช้ pairs/count แทนสำหรับ non-sequence
local function tableCount(t)
    local count = 0
    for _ in pairs(t) do
        count = count + 1
    end
    return count
end

print("จำนวน entries ใน sparse: " .. tableCount(sparse))  -- 3
print("จำนวน entries ใน mixed: " .. tableCount(mixed))    -- 5

-- 4. String เป็น UTF-8 → # นับ bytes ไม่ใช่ characters
local thai = "สวัสดี"
print("bytes: " .. #thai)                -- 18
-- characters จริงๆ ต้องใช้ utf8 library
if utf8 then
    print("characters: " .. utf8.len(thai))  -- 6
end
```

---

## Bitwise Operators (Lua 5.3+)

### ตัวดำเนินการ Bitwise

```lua
-- ตัวอย่างที่ 29: Bitwise operators (Lua 5.3+)
-- ทำงานกับ integer เท่านั้น

-- &  AND
-- |  OR
-- ~  XOR (binary) หรือ NOT (unary)
-- << Left shift
-- >> Right shift

local a = 0b1010  -- 10 ในฐาน 2 (Lua 5.3+ ไม่รองรับ 0b)
-- ใช้ hex แทน
a = 0xA   -- 1010 = 10
local b = 0x6   -- 0110 = 6

print("a = " .. a .. " (0b1010)")
print("b = " .. b .. " (0b0110)")
print()

print("a & b  = " .. (a & b))   -- AND:  0b0010 = 2
print("a | b  = " .. (a | b))   -- OR:   0b1110 = 14
print("a ~ b  = " .. (a ~ b))   -- XOR:  0b1100 = 12
print("~a     = " .. (~a))       -- NOT:  -(a+1) = -11
print("a << 1 = " .. (a << 1))  -- LEFT SHIFT:  0b10100 = 20
print("a >> 1 = " .. (a >> 1))  -- RIGHT SHIFT: 0b0101 = 5
```

### แสดงใน Binary

```lua
-- ตัวอย่างที่ 30: แสดงตัวเลขในรูปแบบ binary
local function toBinary(n, bits)
    bits = bits or 8
    if n < 0 then
        -- two's complement
        n = n + (1 << bits)
    end
    local result = ""
    for i = bits - 1, 0, -1 do
        if n & (1 << i) ~= 0 then
            result = result .. "1"
        else
            result = result .. "0"
        end
    end
    return result
end

local a = 0xA   -- 10
local b = 0x6   -- 6

print(string.format("a     = %3d = %s", a, toBinary(a)))
print(string.format("b     = %3d = %s", b, toBinary(b)))
print(string.rep("-", 30))
print(string.format("a & b = %3d = %s  (AND)", a&b, toBinary(a&b)))
print(string.format("a | b = %3d = %s  (OR)", a|b, toBinary(a|b)))
print(string.format("a ~ b = %3d = %s  (XOR)", a~b, toBinary(a~b)))
```

### Bitwise ในการเขียน Code จริง

```lua
-- ตัวอย่างที่ 31: Bitwise operators ในงานจริง

-- 1. Flags (Permission system)
local PERM_READ    = 0x1   -- 0001
local PERM_WRITE   = 0x2   -- 0010
local PERM_EXECUTE = 0x4   -- 0100
local PERM_ADMIN   = 0x8   -- 1000

-- กำหนด permissions
local userPerms = PERM_READ | PERM_WRITE  -- 0011 = 3

-- ตรวจสอบ permission
local function hasPermission(perms, flag)
    return (perms & flag) ~= 0
end

print("=== ตรวจสอบ Permissions ===")
print("Read:    " .. tostring(hasPermission(userPerms, PERM_READ)))     -- true
print("Write:   " .. tostring(hasPermission(userPerms, PERM_WRITE)))    -- true
print("Execute: " .. tostring(hasPermission(userPerms, PERM_EXECUTE)))  -- false
print("Admin:   " .. tostring(hasPermission(userPerms, PERM_ADMIN)))    -- false

-- เพิ่ม permission
userPerms = userPerms | PERM_EXECUTE
print("หลังเพิ่ม Execute: " .. tostring(hasPermission(userPerms, PERM_EXECUTE)))

-- ลบ permission
userPerms = userPerms & ~PERM_WRITE
print("หลังลบ Write: " .. tostring(hasPermission(userPerms, PERM_WRITE)))
```

```lua
-- ตัวอย่างที่ 32: Bitwise สำหรับ Color manipulation
-- Color RGBA ใน hex: 0xRRGGBBAA

local function makeColor(r, g, b, a)
    a = a or 255
    return (r << 24) | (g << 16) | (b << 8) | a
end

local function getR(color) return (color >> 24) & 0xFF end
local function getG(color) return (color >> 16) & 0xFF end
local function getB(color) return (color >> 8) & 0xFF end
local function getA(color) return color & 0xFF end

local red   = makeColor(255, 0, 0)
local green = makeColor(0, 255, 0)
local blue  = makeColor(0, 0, 255)
local semiTransparent = makeColor(100, 150, 200, 128)

print(string.format("Red:   #%08X  R=%d G=%d B=%d A=%d",
    red, getR(red), getG(red), getB(red), getA(red)))
print(string.format("Green: #%08X  R=%d G=%d B=%d A=%d",
    green, getR(green), getG(green), getB(green), getA(green)))
print(string.format("Semi:  #%08X  R=%d G=%d B=%d A=%d",
    semiTransparent, 
    getR(semiTransparent), getG(semiTransparent),
    getB(semiTransparent), getA(semiTransparent)))
```

```lua
-- ตัวอย่างที่ 33: Shift operators
-- Left shift (<<): คูณด้วย 2^n
-- Right shift (>>): หารด้วย 2^n (สำหรับ positive)

local n = 1

print("=== Left Shift (×2 แต่ละครั้ง) ===")
for i = 0, 10 do
    print(string.format("1 << %2d = %d", i, n << i))
end

print("\n=== Right Shift (÷2 แต่ละครั้ง) ===")
local x = 1024
for i = 0, 10 do
    print(string.format("%d >> %2d = %d", 1024, i, x >> i))
end

-- เร็วกว่าการ * หรือ / สำหรับ powers of 2
print("\n=== Performance: Shift vs Multiply ===")
local N = 10000000

local t1 = os.clock()
local sum1 = 0
for i = 1, N do sum1 = sum1 + i * 8 end

local t2 = os.clock()
local sum2 = 0
for i = 1, N do sum2 = sum2 + (i << 3) end

local t3 = os.clock()
print(string.format("Multiply: %.4f sec", t2-t1))
print(string.format("Shift:    %.4f sec", t3-t2))
print("Results equal: " .. tostring(sum1 == sum2))
```

---

## Operator Precedence

### ตารางความสำคัญ (สูงสุดไปต่ำสุด)

```
ระดับ 12 (สูงสุด): unary operators: not, # , - (unary), ~ (unary)
ระดับ 11:           ^ (right-associative)
ระดับ 10:           * , / , // , %
ระดับ 9:            + , - (binary)
ระดับ 8:            .. (right-associative)
ระดับ 7:            << , >>
ระดับ 6:            &
ระดับ 5:            ~
ระดับ 4:            |
ระดับ 3:            < , > , <= , >= , ~= , ==
ระดับ 2:            and
ระดับ 1 (ต่ำสุด):   or
```

```lua
-- ตัวอย่างที่ 34: Operator precedence
print("=== Operator Precedence ===")

-- ^ มีความสำคัญสูงกว่า * 
print(2 ^ 3 * 4)     -- (2^3) * 4 = 8 * 4 = 32
print(2 * 3 ^ 2)     -- 2 * (3^2) = 2 * 9 = 18

-- * / ก่อน + -
print(2 + 3 * 4)     -- 2 + (3*4) = 2 + 12 = 14
print(10 - 2 * 3)    -- 10 - (2*3) = 10 - 6 = 4

-- unary - มีความสำคัญสูงกว่า ^
print(-2 ^ 2)        -- -(2^2) = -4 ไม่ใช่ (-2)^2 = 4!
print((-2) ^ 2)      -- 4

-- not มีความสำคัญสูงกว่า and/or
print(not true or true)     -- (not true) or true = false or true = true
print(not (true or true))   -- not (true) = false

-- and ก่อน or
print(true or false and false)   -- true or (false and false) = true or false = true
print((true or false) and false) -- true and false = false
```

### วงเล็บเพื่อความชัดเจน

```lua
-- ตัวอย่างที่ 35: ใช้วงเล็บเพื่อความชัดเจน
-- แม้จะไม่จำเป็น แต่ช่วยให้อ่านง่ายขึ้น

-- นิพจน์ที่อาจสับสน
local a, b, c = 2, 3, 4

-- ชัดเจนขึ้น
local result1 = (a + b) * c       -- 20
local result2 = a + (b * c)       -- 14 (เหมือนไม่มี bracket)
local result3 = (a ^ b) + c       -- 12
local result4 = a ^ (b + c)       -- 2^7 = 128

print("(2+3)*4 = " .. result1)   -- 20
print("2+(3*4) = " .. result2)   -- 14
print("(2^3)+4 = " .. result3)   -- 12
print("2^(3+4) = " .. result4)   -- 128

-- Complex boolean expressions
local x, y, z = true, false, true
local expr1 = (x and y) or z     -- false or true = true
local expr2 = x and (y or z)     -- true and true = true
local expr3 = not (x and y)      -- not false = true
local expr4 = (not x) and y      -- false and false = false

print()
print("(true and false) or true = " .. tostring(expr1))
print("true and (false or true) = " .. tostring(expr2))
print("not (true and false) = " .. tostring(expr3))
print("(not true) and false = " .. tostring(expr4))
```

---

## Ternary-like Idiom

### Ternary ด้วย and/or

```lua
-- ตัวอย่างที่ 36: Ternary idiom ใน Lua
-- Lua ไม่มี ? : operator แต่ใช้ and/or แทนได้

-- Pattern: condition and valueIfTrue or valueIfFalse
local x = 10
local result = x > 5 and "มาก" or "น้อย"
print(result)  -- มาก

local y = 3
result = y > 5 and "มาก" or "น้อย"
print(result)  -- น้อย

-- ใช้ใน expressions
local abs_val = x >= 0 and x or -x
print("abs(" .. x .. ") = " .. abs_val)  -- 10

local abs_neg = (-8) >= 0 and (-8) or -(-8)
print("abs(-8) = " .. abs_neg)  -- 8

-- ระวัง! ใช้ไม่ได้ถ้า valueIfTrue เป็น false หรือ nil!
local flag = true
local bad = flag and false or "default"
print("BAD result: " .. tostring(bad))  -- default (ผิด! ควรได้ false)

-- วิธีที่ถูกต้องเมื่อค่าอาจเป็น false
local function ternary(cond, t, f)
    if cond then return t else return f end
end

print("GOOD result: " .. tostring(ternary(flag, false, "default")))  -- false
```

### Ternary ในสถานการณ์ต่างๆ

```lua
-- ตัวอย่างที่ 37: Ternary ในสถานการณ์จริง
local function grade(score)
    return score >= 80 and "A" or
           score >= 70 and "B" or
           score >= 60 and "C" or
           score >= 50 and "D" or "F"
end

local scores = {95, 82, 73, 65, 55, 42}
for _, s in ipairs(scores) do
    print(string.format("คะแนน %d → เกรด %s", s, grade(s)))
end
```

```lua
-- ตัวอย่างที่ 38: Ternary สำหรับ string formatting
local items = {"apple", "banana", "cherry"}

for i, item in ipairs(items) do
    local suffix = i < #items and ", " or "."
    io.write(item .. suffix)
end
print()
-- Output: apple, banana, cherry.

-- อีกตัวอย่าง
local count = 5
print("มี " .. count .. " " .. (count == 1 and "รายการ" or "รายการ"))
-- มี 5 รายการ
```

---

## Math Expressions

### นิพจน์คณิตศาสตร์ซับซ้อน

```lua
-- ตัวอย่างที่ 39: นิพจน์คณิตศาสตร์
-- สูตรต่างๆ ที่ใช้บ่อย

-- พื้นที่วงกลม
local function circleArea(r)
    return math.pi * r ^ 2
end

-- เส้นรอบวง
local function circumference(r)
    return 2 * math.pi * r
end

-- ปริมาตรทรงกลม
local function sphereVolume(r)
    return (4/3) * math.pi * r ^ 3
end

-- พื้นที่สามเหลี่ยม (Heron's formula)
local function triangleArea(a, b, c)
    local s = (a + b + c) / 2
    return math.sqrt(s * (s-a) * (s-b) * (s-c))
end

for r = 1, 5 do
    print(string.format("r=%-3d  พื้นที่=%-10.4f  เส้นรอบวง=%-10.4f  ปริมาตร=%.4f",
        r, circleArea(r), circumference(r), sphereVolume(r)))
end

print()
local sides = {{3, 4, 5}, {5, 12, 13}, {8, 15, 17}}
for _, s in ipairs(sides) do
    print(string.format("สามเหลี่ยม %d,%d,%d  พื้นที่ = %.4f",
        s[1], s[2], s[3], triangleArea(table.unpack(s))))
end
```

### Statistical Formulas

```lua
-- ตัวอย่างที่ 40: สูตรสถิติ
local function mean(data)
    local sum = 0
    for _, v in ipairs(data) do sum = sum + v end
    return sum / #data
end

local function variance(data)
    local m = mean(data)
    local sum = 0
    for _, v in ipairs(data) do
        sum = sum + (v - m) ^ 2
    end
    return sum / #data
end

local function stddev(data)
    return math.sqrt(variance(data))
end

-- Pearson correlation coefficient
local function correlation(x, y)
    assert(#x == #y, "ต้องมีจำนวน element เท่ากัน")
    local n = #x
    local mx, my = mean(x), mean(y)
    
    local numerator = 0
    local sumX2, sumY2 = 0, 0
    
    for i = 1, n do
        local dx = x[i] - mx
        local dy = y[i] - my
        numerator = numerator + dx * dy
        sumX2 = sumX2 + dx ^ 2
        sumY2 = sumY2 + dy ^ 2
    end
    
    return numerator / math.sqrt(sumX2 * sumY2)
end

local data1 = {2, 4, 4, 4, 5, 5, 7, 9}
local data2 = {1, 3, 3, 5, 4, 6, 8, 8}

print("=== สถิติ ===")
print("ค่าเฉลี่ย: " .. string.format("%.4f", mean(data1)))
print("ความแปรปรวน: " .. string.format("%.4f", variance(data1)))
print("ส่วนเบี่ยงเบนมาตรฐาน: " .. string.format("%.4f", stddev(data1)))
print("ค่าสัมประสิทธิ์สหสัมพันธ์: " .. 
    string.format("%.4f", correlation(data1, data2)))
```

---

## String to Number Operations

### ตัวเลขใน String

```lua
-- ตัวอย่างที่ 41: String-to-number operations
-- Lua แปลง string เป็น number อัตโนมัติในนิพจน์คณิตศาสตร์

-- Auto-coercion
print("10" + 5)     -- 15
print("3.14" * 2)   -- 6.28
print("10" ^ 2)     -- 100.0
print("10" % 3)     -- 1

-- แต่ไม่ทำงานกับทุก operation
-- print("hello" + 1)  -- Error!
-- print("10" and "20")  -- "20" (logical, ไม่แปลง)

-- ตรวจสอบก่อนใช้
local inputs = {"42", "3.14", "0xFF", "1e3", "hello", "12abc"}

for _, s in ipairs(inputs) do
    local n = tonumber(s)
    if n then
        print(string.format("%-10s → %g", s, n))
    else
        print(string.format("%-10s → แปลงไม่ได้", s))
    end
end
```

### String Arithmetic ระวัง!

```lua
-- ตัวอย่างที่ 42: ระวัง string arithmetic
local a = "10"
local b = "20"

-- +=, -=, *=, /= ไม่มีใน Lua!
-- ต้องเขียนเต็ม
print(a + b)     -- 30 (auto-coerce)
print(a .. b)    -- "1020" (concatenation)
print(a == b)    -- false (string comparison)
print(a < b)     -- true (lexicographic, "10" < "20" = "1" < "2")
print(a * 2)     -- 20 (auto-coerce)

-- ปัญหา: เมื่อ user input มาเป็น string
local userInput = "5"
local times = 3

-- อาจทำให้สับสน
print(userInput + times)   -- 8 (numeric add)
print(userInput .. times)  -- "53" (string concat)
print(userInput * times)   -- 15 (numeric multiply)

-- แนะนำ: แปลงชัดเจนเสมอ
local num = tonumber(userInput)
if num then
    print("ผลคูณ: " .. (num * times))
end
```

---

## Complex Expressions

### นิพจน์ซับซ้อนและการแยกส่วน

```lua
-- ตัวอย่างที่ 43: Complex expressions
-- แยกนิพจน์ซับซ้อนออกเป็นส่วนย่อย

-- สูตรการเคลื่อนที่แบบ projectile
local function projectile(v0, angle_deg, t)
    local g = 9.8  -- gravity m/s²
    local theta = math.rad(angle_deg)  -- แปลงเป็น radians
    
    local vx = v0 * math.cos(theta)
    local vy = v0 * math.sin(theta)
    
    local x = vx * t
    local y = vy * t - 0.5 * g * t ^ 2
    
    return x, y
end

local v0 = 50      -- ความเร็วต้น m/s
local angle = 45   -- มุม degrees

print("=== การเคลื่อนที่แบบ Projectile ===")
print(string.format("ความเร็วต้น: %d m/s, มุม: %d°", v0, angle))
print()
print(string.format("%-8s %-12s %-12s", "t(s)", "x(m)", "y(m)"))
print(string.rep("-", 35))

for t = 0, 10 do
    local x, y = projectile(v0, angle, t)
    if y >= -0.1 then  -- หยุดเมื่อถึงพื้น
        print(string.format("%-8.1f %-12.2f %-12.2f", t, x, y))
    end
end
```

### Chain Comparisons (Lua ทำไม่ได้โดยตรง)

```lua
-- ตัวอย่างที่ 44: Chain comparisons
-- Lua ไม่รองรับ 1 < x < 10 โดยตรง (ต่างจาก Python)

local x = 5

-- ผิด! 1 < x < 10 จะประมวลผลเป็น (1 < x) < 10
-- = true < 10 → error ใน Lua!

-- ถูกต้อง
if 1 < x and x < 10 then
    print("x อยู่ระหว่าง 1 ถึง 10")
end

-- สร้าง function ช่วย
local function between(value, low, high)
    return value > low and value < high
end

local function inRange(value, low, high)
    return value >= low and value <= high
end

print(between(5, 1, 10))    -- true
print(between(0, 1, 10))    -- false
print(inRange(5, 1, 10))    -- true
print(inRange(1, 1, 10))    -- true (inclusive)
print(inRange(10, 1, 10))   -- true (inclusive)
print(inRange(11, 1, 10))   -- false
```

### Expression Evaluation Order

```lua
-- ตัวอย่างที่ 45: การประเมิน expression และ side effects
-- Lua ประเมิน expression ซ้ายไปขวา

local count = 0
local function next_val()
    count = count + 1
    return count
end

-- การประเมินใน arithmetic
local result = next_val() + next_val() * next_val()
-- next_val() = 1, next_val() = 2, next_val() = 3
-- = 1 + 2 * 3 = 1 + 6 = 7
print("result: " .. result)  -- 7
print("count: " .. count)    -- 3

count = 0
-- Multiple assignment ประเมิน right side ทั้งหมดก่อน
local a, b = next_val(), next_val()
print("a=" .. a .. ", b=" .. b)  -- a=1, b=2
```

### Complex Boolean Expressions

```lua
-- ตัวอย่างที่ 46: Complex boolean expressions
local function isValidEmail(email)
    if type(email) ~= "string" then return false end
    if #email == 0 then return false end
    
    -- ต้องมี @ หนึ่งตัว
    local atPos = email:find("@")
    if not atPos or atPos == 1 or atPos == #email then
        return false
    end
    
    -- ส่วนหลัง @ ต้องมี .
    local domain = email:sub(atPos + 1)
    if not domain:find("%.") then return false end
    
    -- ไม่มี special chars ผิดๆ
    if email:find("[%s,;:]") then return false end
    
    return true
end

local emails = {
    "user@example.com",
    "invalid",
    "@nodomain.com",
    "noat.com",
    "user@",
    "user@domain",
    "valid.user@sub.domain.com",
    "has space@domain.com",
}

print("=== ตรวจสอบ Email ===")
for _, email in ipairs(emails) do
    local valid = isValidEmail(email)
    print(string.format("%-30s → %s", email, valid and "ถูกต้อง" or "ไม่ถูกต้อง"))
end
```

### นิพจน์ที่ใช้ใน Games

```lua
-- ตัวอย่างที่ 47: นิพจน์ที่ใช้ใน game development
-- Linear interpolation (lerp)
local function lerp(a, b, t)
    return a + (b - a) * t
end

-- Clamp
local function clamp(value, min, max)
    return math.max(min, math.min(max, value))
end

-- Smooth step
local function smoothstep(edge0, edge1, x)
    local t = clamp((x - edge0) / (edge1 - edge0), 0, 1)
    return t * t * (3 - 2 * t)
end

-- Distance squared (เร็วกว่า distance เพราะไม่ต้อง sqrt)
local function distSq(x1, y1, x2, y2)
    return (x2 - x1) ^ 2 + (y2 - y1) ^ 2
end

-- Map range
local function mapRange(value, inMin, inMax, outMin, outMax)
    return outMin + (value - inMin) / (inMax - inMin) * (outMax - outMin)
end

print("=== Game Math Functions ===")
print(string.format("lerp(0, 100, 0.25) = %.1f", lerp(0, 100, 0.25)))     -- 25
print(string.format("lerp(0, 100, 0.75) = %.1f", lerp(0, 100, 0.75)))     -- 75
print(string.format("clamp(150, 0, 100) = %.1f", clamp(150, 0, 100)))      -- 100
print(string.format("clamp(-10, 0, 100) = %.1f", clamp(-10, 0, 100)))      -- 0
print(string.format("smoothstep(0, 1, 0.5) = %.4f", smoothstep(0, 1, 0.5)))  -- 0.5

-- ตรวจสอบ collision (circle)
local px, py = 0, 0      -- player
local ex, ey = 3, 4      -- enemy
local radius = 6         -- hitbox radius

if distSq(px, py, ex, ey) <= radius ^ 2 then
    print("ชนกัน!")
else
    print("ไม่ชน (ระยะ = " .. math.sqrt(distSq(px, py, ex, ey)) .. ")")
end

-- Map temperature 0-100°C to RGB (blue=cold, red=hot)
local function tempToColor(temp)
    local r = math.floor(mapRange(clamp(temp, 0, 100), 0, 100, 0, 255))
    local b = math.floor(mapRange(clamp(temp, 0, 100), 0, 100, 255, 0))
    return r, 0, b
end

print("\n=== อุณหภูมิ → สี ===")
for t = 0, 100, 10 do
    local r, g, b = tempToColor(t)
    print(string.format("%3d°C → RGB(%3d, %3d, %3d)", t, r, g, b))
end
```

### Operator Overloading ด้วย Metamethods

```lua
-- ตัวอย่างที่ 48: Custom operators ด้วย metamethods
-- สร้าง Vector2 ที่รองรับ +, -, *, == operators

local Vector2 = {}
Vector2.__index = Vector2

function Vector2.new(x, y)
    return setmetatable({x = x or 0, y = y or 0}, Vector2)
end

-- + operator
function Vector2.__add(a, b)
    return Vector2.new(a.x + b.x, a.y + b.y)
end

-- - operator
function Vector2.__sub(a, b)
    return Vector2.new(a.x - b.x, a.y - b.y)
end

-- * operator (scalar multiplication)
function Vector2.__mul(a, b)
    if type(a) == "number" then
        return Vector2.new(a * b.x, a * b.y)
    elseif type(b) == "number" then
        return Vector2.new(a.x * b, a.y * b)
    else
        return a.x * b.x + a.y * b.y  -- dot product
    end
end

-- unary - operator
function Vector2.__unm(a)
    return Vector2.new(-a.x, -a.y)
end

-- == operator
function Vector2.__eq(a, b)
    return a.x == b.x and a.y == b.y
end

-- tostring
function Vector2.__tostring(v)
    return string.format("Vector2(%.2f, %.2f)", v.x, v.y)
end

-- Length
function Vector2:length()
    return math.sqrt(self.x ^ 2 + self.y ^ 2)
end

-- Normalize
function Vector2:normalize()
    local len = self:length()
    if len == 0 then return Vector2.new(0, 0) end
    return Vector2.new(self.x / len, self.y / len)
end

-- ทดสอบ
local v1 = Vector2.new(3, 4)
local v2 = Vector2.new(1, 2)

print(tostring(v1))              -- Vector2(3.00, 4.00)
print(tostring(v2))              -- Vector2(1.00, 2.00)
print(tostring(v1 + v2))         -- Vector2(4.00, 6.00)
print(tostring(v1 - v2))         -- Vector2(2.00, 2.00)
print(tostring(v1 * 2))          -- Vector2(6.00, 8.00)
print(tostring(3 * v2))          -- Vector2(3.00, 6.00)
print("dot product: " .. (v1 * v2))  -- 3*1 + 4*2 = 11
print("length: " .. v1:length())     -- 5.0
print(tostring(v1:normalize()))      -- Vector2(0.60, 0.80)
print("v1 == v1: " .. tostring(v1 == v1))  -- true
print("v1 == v2: " .. tostring(v1 == v2))  -- false
```

---

## โปรแกรมรวมตัวอย่าง

### โปรแกรม Expression Evaluator

```lua
-- ตัวอย่างที่ 49: โปรแกรมแสดง operator ทั้งหมด
local function showAllOps(a, b)
    print(string.format("a = %g, b = %g", a, b))
    print(string.rep("-", 40))
    
    -- Arithmetic
    print(string.format("a + b   = %g", a + b))
    print(string.format("a - b   = %g", a - b))
    print(string.format("a * b   = %g", a * b))
    print(string.format("a / b   = %g", a / b))
    print(string.format("a // b  = %g", a // b))
    print(string.format("a %% b  = %g", a % b))
    print(string.format("a ^ b   = %g", a ^ b))
    print(string.format("-a      = %g", -a))
    print()
    
    -- Relational
    print(string.format("a == b  = %s", tostring(a == b)))
    print(string.format("a ~= b  = %s", tostring(a ~= b)))
    print(string.format("a < b   = %s", tostring(a < b)))
    print(string.format("a > b   = %s", tostring(a > b)))
    print(string.format("a <= b  = %s", tostring(a <= b)))
    print(string.format("a >= b  = %s", tostring(a >= b)))
    print()
    
    -- Bitwise (integers only)
    if math.type(a) == "integer" and math.type(b) == "integer" then
        print(string.format("a & b   = %d", a & b))
        print(string.format("a | b   = %d", a | b))
        print(string.format("a ~ b   = %d (XOR)", a ~ b))
        print(string.format("~a      = %d (NOT)", ~a))
        print(string.format("a << 1  = %d", a << 1))
        print(string.format("a >> 1  = %d", a >> 1))
    end
end

showAllOps(12, 5)
```

### โปรแกรมคำนวณ Compound Interest พร้อมกราฟ

```lua
-- ตัวอย่างที่ 50: Compound Interest กับ ASCII graph
local function compoundInterest(principal, rate, periods)
    local results = {}
    for t = 0, periods do
        results[t+1] = {
            period = t,
            amount = principal * (1 + rate) ^ t
        }
    end
    return results
end

local P = 10000  -- เงินต้น
local r = 0.08   -- ดอกเบี้ย 8%
local n = 10     -- 10 ปี

local data = compoundInterest(P, r, n)

print("=== Compound Interest ===")
print(string.format("เงินต้น: ฿%.2f, อัตรา: %.0f%%, ระยะ: %d ปี", P, r*100, n))
print()

-- หาค่าสูงสุดสำหรับ scale
local maxVal = data[#data].amount

-- แสดงกราฟ
local barWidth = 40
print(string.format("%-4s %-12s %s", "ปี", "จำนวนเงิน", "กราฟ"))
print(string.rep("-", 60))

for _, item in ipairs(data) do
    local barLen = math.floor(item.amount / maxVal * barWidth)
    local bar = string.rep("█", barLen) .. string.rep("░", barWidth - barLen)
    print(string.format("%-4d ฿%-11.2f %s", item.period, item.amount, bar))
end

print()
print(string.format("เงินต้น:    ฿%.2f", P))
print(string.format("เงินสุดท้าย: ฿%.2f", data[n+1].amount))
print(string.format("ดอกเบี้ยรวม: ฿%.2f", data[n+1].amount - P))
print(string.format("เพิ่มขึ้น %.1f%%", (data[n+1].amount/P - 1) * 100))
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: เครื่องคิดเลข Scientific

สร้างเครื่องคิดเลข scientific ที่รองรับ operations ต่างๆ:

```lua
-- แบบฝึกหัดที่ 1 - เฉลย
local ScientificCalc = {}

function ScientificCalc.eval(expr)
    -- ใช้ load() เพื่อประเมิน expression
    -- (ในงานจริงควรมี parser ที่ปลอดภัยกว่า)
    local fn, err = load("return " .. expr)
    if not fn then
        return nil, "Expression ผิด: " .. err
    end
    
    -- inject math functions
    local env = setmetatable({}, {__index = math})
    env.abs = math.abs
    env.sqrt = math.sqrt
    env.sin = math.sin
    env.cos = math.cos
    env.tan = math.tan
    env.log = math.log
    env.exp = math.exp
    env.pi = math.pi
    env.e = math.exp(1)
    
    -- ไม่สามารถ inject ง่ายๆ กับ load ใน Lua 5.4
    -- ดังนั้นใช้ pcall แทน
    local ok, result = pcall(fn)
    if not ok then
        return nil, "Error: " .. tostring(result)
    end
    return result
end

-- ทดสอบ
local expressions = {
    "2 + 3 * 4",
    "math.sqrt(144)",
    "math.sin(math.pi / 6)",
    "2 ^ 10",
    "(3 + 4) ^ 2 / 2",
    "math.floor(3.7)",
    "math.ceil(3.2)",
}

print("=== Scientific Calculator ===")
for _, expr in ipairs(expressions) do
    local result, err = ScientificCalc.eval(expr)
    if err then
        print(string.format("%-30s = ERROR: %s", expr, err))
    else
        print(string.format("%-30s = %g", expr, result))
    end
end
```

### แบบฝึกหัดที่ 2: Bit Manipulation

เขียน function สำหรับ bit manipulation:

```lua
-- แบบฝึกหัดที่ 2 - เฉลย
local BitUtils = {}

-- ตั้งค่า bit ที่ตำแหน่ง n
function BitUtils.setBit(num, n)
    return num | (1 << n)
end

-- ล้าง bit ที่ตำแหน่ง n
function BitUtils.clearBit(num, n)
    return num & ~(1 << n)
end

-- สลับ bit ที่ตำแหน่ง n
function BitUtils.toggleBit(num, n)
    return num ~ (1 << n)
end

-- ตรวจสอบ bit ที่ตำแหน่ง n
function BitUtils.testBit(num, n)
    return (num >> n) & 1 == 1
end

-- นับจำนวน set bits (popcount)
function BitUtils.popcount(num)
    local count = 0
    while num ~= 0 do
        count = count + (num & 1)
        num = num >> 1
    end
    return count
end

-- แสดงเป็น binary string
function BitUtils.toBinary(num, bits)
    bits = bits or 8
    local result = ""
    for i = bits - 1, 0, -1 do
        result = result .. (BitUtils.testBit(num, i) and "1" or "0")
        if i % 4 == 0 and i > 0 then result = result .. " " end
    end
    return result
end

-- ทดสอบ
local n = 0b00001010  -- เขียนด้วย hex: 0xA = 10

print("=== Bit Manipulation ===")
print(string.format("n = %d = %s", n, BitUtils.toBinary(n)))
print()

-- setBit
local after_set = BitUtils.setBit(n, 0)  -- ตั้ง bit 0
print(string.format("setBit(n, 0)    = %d = %s", after_set, BitUtils.toBinary(after_set)))

-- clearBit
local after_clear = BitUtils.clearBit(n, 1)  -- ล้าง bit 1
print(string.format("clearBit(n, 1)  = %d = %s", after_clear, BitUtils.toBinary(after_clear)))

-- toggleBit
local after_toggle = BitUtils.toggleBit(n, 3)  -- สลับ bit 3
print(string.format("toggleBit(n, 3) = %d = %s", after_toggle, BitUtils.toBinary(after_toggle)))

-- testBit
for i = 0, 7 do
    print(string.format("bit[%d] = %s", i, tostring(BitUtils.testBit(n, i))))
end

-- popcount
print("\npopcount(" .. n .. ") = " .. BitUtils.popcount(n))

-- แสดง popcount 0-15
print("\n=== popcount 0-15 ===")
for i = 0, 15 do
    print(string.format("%2d = %s → popcount = %d",
        i, BitUtils.toBinary(i, 4), BitUtils.popcount(i)))
end
```

### แบบฝึกหัดที่ 3: Logic Gate Simulator

จำลอง logic gates ด้วย Lua operators:

```lua
-- แบบฝึกหัดที่ 3 - เฉลย
local Gates = {}

-- Basic gates
function Gates.AND(a, b)   return a and b end
function Gates.OR(a, b)    return a or b end
function Gates.NOT(a)      return not a end
function Gates.NAND(a, b)  return not (a and b) end
function Gates.NOR(a, b)   return not (a or b) end
function Gates.XOR(a, b)   return (a or b) and not (a and b) end
function Gates.XNOR(a, b)  return not Gates.XOR(a, b) end

-- แสดง Truth Table
local function showTruthTable(name, fn, inputs)
    print("\n=== " .. name .. " Truth Table ===")
    
    if #inputs == 1 then
        print("A     | OUT")
        print("------+-----")
        for _, a in ipairs({true, false}) do
            print(string.format("%-5s | %s",
                tostring(a), tostring(fn(a))))
        end
    else
        print("A     | B     | OUT")
        print("------+-------+-----")
        for _, a in ipairs({true, false}) do
            for _, b in ipairs({true, false}) do
                print(string.format("%-5s | %-5s | %s",
                    tostring(a), tostring(b), tostring(fn(a, b))))
            end
        end
    end
end

-- สร้าง Half Adder จาก gates
local function halfAdder(a, b)
    local sum   = Gates.XOR(a, b)   -- bit sum
    local carry = Gates.AND(a, b)   -- carry bit
    return sum, carry
end

-- สร้าง Full Adder
local function fullAdder(a, b, cin)
    local s1, c1 = halfAdder(a, b)
    local sum, c2 = halfAdder(s1, cin)
    local carry = Gates.OR(c1, c2)
    return sum, carry
end

showTruthTable("AND",  Gates.AND)
showTruthTable("OR",   Gates.OR)
showTruthTable("NOT",  Gates.NOT, {1})
showTruthTable("XOR",  Gates.XOR)
showTruthTable("NAND", Gates.NAND)

-- ทดสอบ Half Adder
print("\n=== Half Adder ===")
print("A     | B     | Sum   | Carry")
print("------+-------+-------+------")
for _, a in ipairs({true, false}) do
    for _, b in ipairs({true, false}) do
        local sum, carry = halfAdder(a, b)
        print(string.format("%-5s | %-5s | %-5s | %s",
            tostring(a), tostring(b), tostring(sum), tostring(carry)))
    end
end

-- ทดสอบ Full Adder
print("\n=== Full Adder ===")
print("A     | B     | Cin   | Sum   | Cout")
print("------+-------+-------+-------+------")
for _, a in ipairs({true, false}) do
    for _, b in ipairs({true, false}) do
        for _, cin in ipairs({true, false}) do
            local sum, cout = fullAdder(a, b, cin)
            print(string.format("%-5s | %-5s | %-5s | %-5s | %s",
                tostring(a), tostring(b), tostring(cin),
                tostring(sum), tostring(cout)))
        end
    end
end
```

### แบบฝึกหัดที่ 4: Operator Precedence Quiz

สร้างโปรแกรมที่แสดงว่า expression ต่างๆ ประมวลผลอย่างไร:

```lua
-- แบบฝึกหัดที่ 4 - เฉลย
local function explainExpression(expr, expected)
    local fn = load("return " .. expr)
    local ok, result = pcall(fn)
    
    local correct = ok and result == expected
    print(string.format("%-30s = %-10s %s",
        expr,
        ok and tostring(result) or "ERROR",
        correct and "✓" or (ok and "✗ (คาดว่า " .. tostring(expected) .. ")" or "")))
end

print("=== Operator Precedence Quiz ===")
print()
print("ทดสอบความเข้าใจ precedence:")
print()

-- Arithmetic
explainExpression("2 + 3 * 4",     14)      -- * ก่อน +
explainExpression("2 * 3 + 4 * 5", 26)      -- * ก่อน +
explainExpression("10 - 2 - 3",    5)       -- left-to-right
explainExpression("2 ^ 3 ^ 2",     512)     -- right-to-left: 2^(3^2)=2^9=512
explainExpression("-2 ^ 2",        -4)      -- -(2^2) ไม่ใช่ (-2)^2

-- Mixed
explainExpression("not false and true", true)  -- (not false) and true
explainExpression("1 + 2 > 2 + 0",     true)  -- (1+2) > (2+0)
explainExpression("2 * 3 == 6",        true)

-- String
explainExpression([["a" .. "b" .. "c"]], "abc")
explainExpression([[#"hello" + 1]],     6)

-- Bitwise
explainExpression("1 | 2 & 3",    3)  -- 1 | (2&3) = 1|2 = 3
explainExpression("4 | 2 ~ 1",    6)  -- 4 | (2~1) = 4|3 = 7... ตรวจสอบ
```

### แบบฝึกหัดที่ 5: Expression Parser

สร้าง simple math expression evaluator:

```lua
-- แบบฝึกหัดที่ 5 - เฉลย
-- Simple recursive descent parser

local function parseNumber(s, pos)
    local num, newPos = s:match("^(-?%d+%.?%d*)", pos)
    if num then
        return tonumber(num), pos + #num
    end
    return nil, pos
end

local function skipSpaces(s, pos)
    local _, e = s:find("^%s*", pos)
    return (e or pos - 1) + 1
end

-- Forward declarations
local parseExpr

local function parsePrimary(s, pos)
    pos = skipSpaces(s, pos)
    
    -- วงเล็บ
    if s:sub(pos, pos) == "(" then
        local val, newPos = parseExpr(s, pos + 1)
        newPos = skipSpaces(s, newPos)
        if s:sub(newPos, newPos) == ")" then
            return val, newPos + 1
        end
        return nil, pos, "คาดหวัง ')'"
    end
    
    -- ตัวเลข
    return parseNumber(s, pos)
end

local function parseMulDiv(s, pos)
    local left, newPos = parsePrimary(s, pos)
    if not left then return nil, pos end
    
    while true do
        newPos = skipSpaces(s, newPos)
        local op = s:sub(newPos, newPos)
        if op == "*" or op == "/" or op == "%" then
            local right, np = parsePrimary(s, newPos + 1)
            if not right then break end
            if op == "*" then left = left * right
            elseif op == "/" then
                if right == 0 then return nil, pos, "หารด้วยศูนย์" end
                left = left / right
            else left = left % right end
            newPos = np
        else
            break
        end
    end
    
    return left, newPos
end

parseExpr = function(s, pos)
    local left, newPos = parseMulDiv(s, pos)
    if not left then return nil, pos end
    
    while true do
        newPos = skipSpaces(s, newPos)
        local op = s:sub(newPos, newPos)
        if op == "+" or op == "-" then
            local right, np = parseMulDiv(s, newPos + 1)
            if not right then break end
            if op == "+" then left = left + right
            else left = left - right end
            newPos = np
        else
            break
        end
    end
    
    return left, newPos
end

local function evaluate(expr)
    local result, pos = parseExpr(expr, 1)
    return result
end

-- ทดสอบ
local tests = {
    "2 + 3",
    "10 - 4",
    "3 * 4",
    "15 / 3",
    "2 + 3 * 4",
    "(2 + 3) * 4",
    "10 % 3",
    "100 / 5 / 4",
    "1 + 2 + 3 + 4 + 5",
}

print("=== Expression Parser ===")
for _, expr in ipairs(tests) do
    local result = evaluate(expr)
    -- เปรียบเทียบกับ Lua built-in
    local expected = load("return " .. expr)()
    local ok = math.abs(result - expected) < 0.0001
    print(string.format("%-25s = %-8g %s", expr, result, ok and "✓" or "✗"))
end
```

---

## สรุปบทที่ 3

ในบทนี้เราได้เรียนรู้:

### ตัวดำเนินการทั้งหมดใน Lua

| หมวดหมู่      | ตัวดำเนินการ                     | หมายเหตุ                         |
|--------------|----------------------------------|----------------------------------|
| Arithmetic   | `+`, `-`, `*`, `/`, `//`, `%`, `^` | `/` เสมอได้ float, `^` เสมอได้ float |
| Unary        | `-` (negate)                     | `-(value)`                       |
| Relational   | `==`, `~=`, `<`, `>`, `<=`, `>=` | `~=` แทน `!=`                   |
| Logical      | `and`, `or`, `not`               | คืนค่าจริง ไม่ใช่แค่ boolean     |
| Concat       | `..`                             | เชื่อม strings                   |
| Length       | `#`                              | bytes สำหรับ string, length สำหรับ table |
| Bitwise      | `&`, `\|`, `~`, `<<`, `>>`      | Lua 5.3+, integers เท่านั้น      |

### ลำดับความสำคัญ (สูงสุด → ต่ำสุด)

```
1.  not  #  -  ~          (unary)
2.  ^                      (right-associative)
3.  *  /  //  %
4.  +  -
5.  ..                     (right-associative)
6.  <<  >>
7.  &
8.  ~                      (binary XOR)
9.  |
10. <  >  <=  >=  ~=  ==
11. and
12. or                     (ต่ำสุด)
```

### Patterns ที่ควรจำ

```lua
-- Default value
local x = value or default

-- Ternary
local result = condition and ifTrue or ifFalse

-- Safe navigation
local name = user and user.profile and user.profile.name

-- Guard clause
local ok = b ~= 0 and a / b

-- Integer floor division
local pages = (items + perPage - 1) // perPage

-- Check flag
local hasFlag = (flags & FLAG) ~= 0

-- Set flag
flags = flags | FLAG

-- Clear flag
flags = flags & ~FLAG
```

**บทถัดไป:** บทที่ 4 - การควบคุมกระแสโปรแกรม (if/else, while, repeat/until, for loops, break, goto)
