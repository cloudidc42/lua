# บทที่ 7: Table - โครงสร้างข้อมูลหลักของ Lua

Table เป็นโครงสร้างข้อมูลที่ทรงพลังและยืดหยุ่นที่สุดใน Lua ในความเป็นจริงแล้ว Table คือโครงสร้างข้อมูลชนิดเดียวใน Lua ที่ใช้สร้างทุกอย่าง ตั้งแต่ Array, Dictionary, Object, Class, Namespace ไปจนถึงโมดูลต่างๆ

---

## 7.1 ทำความรู้จัก Table

Table ใน Lua คือ associative array — โครงสร้างข้อมูลที่ map key ไปยัง value โดย key และ value สามารถเป็น type ใดก็ได้ (ยกเว้น `nil` สำหรับ key)

```lua
-- สร้าง table ว่าง
local t = {}
print(type(t))  -- table

-- table literal
local person = {
    name = "สมชาย",
    age = 25,
    city = "กรุงเทพ"
}

print(person.name)   -- สมชาย
print(person["age"]) -- 25
print(person.city)   -- กรุงเทพ
```

---

## 7.2 Table ในฐานะ Array (1-indexed!)

ข้อสำคัญมาก: Lua ใช้ index เริ่มต้นที่ **1** ไม่ใช่ 0 อย่างภาษาอื่น

```lua
-- สร้าง array โดยใช้ table
local fruits = {"แอปเปิล", "กล้วย", "ส้ม", "มะม่วง", "องุ่น"}

-- เข้าถึงด้วย index (เริ่มที่ 1)
print(fruits[1])  -- แอปเปิล
print(fruits[2])  -- กล้วย
print(fruits[5])  -- องุ่น
print(fruits[0])  -- nil (ไม่มี index 0!)
print(fruits[6])  -- nil (เกิน range)

-- ความยาวของ array
print(#fruits)    -- 5
```

```lua
-- การวนลูปผ่าน array
local numbers = {10, 20, 30, 40, 50}

-- วิธีที่ 1: for loop แบบตัวเลข
for i = 1, #numbers do
    print("numbers[" .. i .. "] = " .. numbers[i])
end

-- วิธีที่ 2: ipairs
for index, value in ipairs(numbers) do
    print("index=" .. index .. ", value=" .. value)
end
```

```lua
-- การสร้าง array แบบต่างๆ
-- แบบที่ 1: กำหนดค่าตั้งแต่แรก
local colors = {"แดง", "เขียว", "น้ำเงิน"}

-- แบบที่ 2: เพิ่มทีละตัว
local seasons = {}
seasons[1] = "ฤดูร้อน"
seasons[2] = "ฤดูฝน"
seasons[3] = "ฤดูหนาว"

-- แบบที่ 3: ใช้ table.insert
local vegetables = {}
table.insert(vegetables, "แครอท")
table.insert(vegetables, "ผักกาด")
table.insert(vegetables, "มะเขือเทศ")

print(#colors)      -- 3
print(#seasons)     -- 3
print(#vegetables)  -- 3
```

---

## 7.3 Table ในฐานะ Dictionary / Hash Map

```lua
-- Dictionary: เก็บข้อมูลแบบ key-value
local student = {
    id = "S001",
    name = "นิดา",
    score = 95.5,
    passed = true
}

-- เข้าถึงด้วย dot notation
print(student.name)   -- นิดา
print(student.score)  -- 95.5

-- เข้าถึงด้วย bracket notation
print(student["id"])     -- S001
print(student["passed"]) -- true

-- เพิ่ม/แก้ไข key
student.grade = "A"
student["email"] = "nida@school.com"

print(student.grade)  -- A
print(student.email)  -- nida@school.com

-- ลบ key (ตั้งเป็น nil)
student.passed = nil
print(student.passed) -- nil
```

```lua
-- Dictionary กับ key ที่เป็นสตริงมีช่องว่าง
local config = {}
config["max size"] = 100  -- ต้องใช้ bracket notation
config["file path"] = "/home/user/data"

print(config["max size"])   -- 100
print(config["file path"])  -- /home/user/data
```

```lua
-- Dictionary ที่ซับซ้อน
local inventory = {
    apple = {count = 50, price = 5.0},
    banana = {count = 30, price = 3.0},
    orange = {count = 20, price = 8.0}
}

print(inventory.apple.count)   -- 50
print(inventory.banana.price)  -- 3.0

-- ลดจำนวน apple
inventory.apple.count = inventory.apple.count - 1
print(inventory.apple.count)   -- 49
```

---

## 7.4 Mixed Table (ผสมทั้ง Array และ Dictionary)

```lua
-- Table แบบผสม
local mixed = {
    "first",           -- index 1
    "second",          -- index 2
    name = "ผสม",      -- key = "name"
    "third",           -- index 3
    count = 100        -- key = "count"
}

print(mixed[1])      -- first
print(mixed[2])      -- second
print(mixed[3])      -- third
print(mixed.name)    -- ผสม
print(mixed.count)   -- 100

-- # จะนับเฉพาะ sequence part
print(#mixed)        -- 3 (นับเฉพาะ "first", "second", "third")
```

```lua
-- ตัวอย่างการใช้งานจริง: บันทึกข้อมูลการ์ด
local card = {
    "ราศีกรกฎ",         -- ชื่อตัวละคร (index 1)
    "นักนำ",            -- อาชีพ (index 2)
    hp = 250,
    mp = 100,
    attack = 45,
    defense = 30,
    skills = {"Slash", "Fire Ball", "Heal"}
}

print("ตัวละคร:", card[1])
print("อาชีพ:", card[2])
print("HP:", card.hp)
print("Skills:", card.skills[1], card.skills[2], card.skills[3])
```

---

## 7.5 Nested Tables (Table ซ้อน Table)

```lua
-- Table ซ้อนกัน
local school = {
    name = "โรงเรียนลูอา",
    address = {
        street = "ถนนโปรแกรมมิ่ง",
        district = "เขตเทคโนโลยี",
        province = "กรุงเทพ",
        zip = "10110"
    },
    departments = {
        "วิทยาศาสตร์",
        "คณิตศาสตร์",
        "ภาษาอังกฤษ"
    }
}

print(school.name)
print(school.address.street)
print(school.address.province)
print(school.departments[1])
print(school.departments[2])
```

```lua
-- ตาราง 2D: matrix ขนาด 3x3
local matrix = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
}

-- เข้าถึงองค์ประกอบ
print(matrix[1][1])  -- 1
print(matrix[2][2])  -- 5
print(matrix[3][3])  -- 9

-- พิมพ์ matrix ทั้งหมด
for i = 1, 3 do
    local row = ""
    for j = 1, 3 do
        row = row .. string.format("%3d", matrix[i][j])
    end
    print(row)
end
```

```lua
-- โครงสร้างข้อมูลสินค้า
local products = {
    {
        id = 1,
        name = "แล็ปท็อป",
        price = 25000,
        specs = {
            cpu = "Intel i7",
            ram = "16GB",
            storage = "512GB SSD"
        }
    },
    {
        id = 2,
        name = "สมาร์ทโฟน",
        price = 15000,
        specs = {
            cpu = "Snapdragon 888",
            ram = "8GB",
            storage = "256GB"
        }
    }
}

for _, product in ipairs(products) do
    print("สินค้า:", product.name)
    print("ราคา:", product.price)
    print("CPU:", product.specs.cpu)
    print("---")
end
```

---

## 7.6 table.insert() และ table.remove()

```lua
-- table.insert(t, value) - เพิ่มที่ท้าย
local stack = {}
table.insert(stack, "แรก")
table.insert(stack, "สอง")
table.insert(stack, "สาม")

print(stack[1], stack[2], stack[3])  -- แรก  สอง  สาม
print(#stack)  -- 3

-- table.insert(t, pos, value) - เพิ่มที่ตำแหน่งที่กำหนด
table.insert(stack, 2, "แทรก")
print(stack[1], stack[2], stack[3], stack[4])
-- แรก  แทรก  สอง  สาม
print(#stack)  -- 4
```

```lua
-- table.remove(t) - ลบที่ท้าย
local queue = {"A", "B", "C", "D", "E"}

local last = table.remove(queue)
print("ลบ:", last)  -- ลบ: E
print(#queue)       -- 4

-- table.remove(t, pos) - ลบที่ตำแหน่งที่กำหนด
local first = table.remove(queue, 1)
print("ลบตัวแรก:", first)  -- ลบตัวแรก: A
print(queue[1], queue[2], queue[3])  -- B  C  D
print(#queue)  -- 3
```

```lua
-- ตัวอย่าง: การจัดการรายการซื้อของ
local shopping_cart = {}

-- เพิ่มสินค้า
table.insert(shopping_cart, {name = "ข้าวสาร", qty = 2, price = 50})
table.insert(shopping_cart, {name = "ไข่ไก่", qty = 30, price = 120})
table.insert(shopping_cart, {name = "น้ำตาล", qty = 1, price = 25})

print("รายการในตะกร้า:")
for i, item in ipairs(shopping_cart) do
    print(i .. ". " .. item.name .. " x" .. item.qty .. " = " .. item.price .. " บาท")
end

-- ลบสินค้าตัวที่ 2
table.remove(shopping_cart, 2)
print("\nหลังลบไข่ไก่:")
for i, item in ipairs(shopping_cart) do
    print(i .. ". " .. item.name)
end
```

---

## 7.7 table.sort()

```lua
-- sort แบบ default (ascending)
local nums = {5, 3, 8, 1, 9, 2, 7, 4, 6}
table.sort(nums)
for _, v in ipairs(nums) do
    io.write(v .. " ")
end
print()  -- 1 2 3 4 5 6 7 8 9
```

```lua
-- sort แบบ descending
local scores = {85, 92, 78, 96, 61, 88}
table.sort(scores, function(a, b) return a > b end)

print("คะแนนเรียงมากไปน้อย:")
for _, s in ipairs(scores) do
    io.write(s .. " ")
end
print()  -- 96 92 88 85 78 61
```

```lua
-- sort table of tables ตามชื่อ
local students = {
    {name = "สมชาย", score = 85},
    {name = "อรุณ", score = 92},
    {name = "กมล", score = 78},
    {name = "นิดา", score = 96}
}

-- sort ตามคะแนน
table.sort(students, function(a, b) return a.score > b.score end)
print("เรียงตามคะแนน:")
for _, s in ipairs(students) do
    print(s.name .. ": " .. s.score)
end

-- sort ตามชื่อ (alphabetical)
table.sort(students, function(a, b) return a.name < b.name end)
print("\nเรียงตามชื่อ:")
for _, s in ipairs(students) do
    print(s.name .. ": " .. s.score)
end
```

```lua
-- stable sort (รักษาลำดับ relative order ของ element ที่ equal)
local items = {
    {cat = "A", val = 3},
    {cat = "B", val = 1},
    {cat = "A", val = 1},
    {cat = "B", val = 2},
    {cat = "A", val = 2},
}

-- sort ตาม category แล้วตาม value
table.sort(items, function(a, b)
    if a.cat ~= b.cat then
        return a.cat < b.cat
    end
    return a.val < b.val
end)

for _, item in ipairs(items) do
    print(item.cat, item.val)
end
-- A  1
-- A  2
-- A  3
-- B  1
-- B  2
```

---

## 7.8 table.concat()

```lua
-- table.concat(t, sep) - รวม elements เป็น string
local words = {"สวัสดี", "โลก", "Lua", "เจ๋ง"}
print(table.concat(words))           -- สวัสดีโลกLuaเจ๋ง
print(table.concat(words, " "))      -- สวัสดี โลก Lua เจ๋ง
print(table.concat(words, ", "))     -- สวัสดี, โลก, Lua, เจ๋ง
print(table.concat(words, " | "))    -- สวัสดี | โลก | Lua | เจ๋ง
```

```lua
-- table.concat กับ range
local items = {"A", "B", "C", "D", "E"}
print(table.concat(items, ",", 2, 4))  -- B,C,D (index 2 ถึง 4)
print(table.concat(items, "-", 1, 3))  -- A-B-C (index 1 ถึง 3)
```

```lua
-- ประสิทธิภาพ: สร้างสตริงยาวๆ
-- วิธีที่ดี: เก็บ parts แล้วค่อย concat
local parts = {}
for i = 1, 1000 do
    parts[i] = tostring(i)
end
local result = table.concat(parts, ", ")
print(string.sub(result, 1, 30) .. "...")
-- 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, ...

-- วิธีที่ไม่ดี (ช้า):
-- local s = ""
-- for i = 1, 1000 do
--     s = s .. tostring(i) .. ", "  -- สร้าง string ใหม่ทุกครั้ง
-- end
```

```lua
-- ใช้ table.concat สร้าง CSV
local headers = {"ชื่อ", "อายุ", "เมือง"}
local row1 = {"สมชาย", "25", "กรุงเทพ"}
local row2 = {"นิดา", "22", "เชียงใหม่"}

print(table.concat(headers, ","))
print(table.concat(row1, ","))
print(table.concat(row2, ","))
-- ชื่อ,อายุ,เมือง
-- สมชาย,25,กรุงเทพ
-- นิดา,22,เชียงใหม่
```

---

## 7.9 table.move()

```lua
-- table.move(a1, f, e, t [,a2])
-- คัดลอก elements จาก a1[f..e] ไปยัง a2[t..t+(e-f)]

local src = {1, 2, 3, 4, 5}
local dst = {10, 20, 30, 40, 50}

-- คัดลอก src[1..3] ไปยัง dst[2]
table.move(src, 1, 3, 2, dst)
print(dst[1], dst[2], dst[3], dst[4], dst[5])
-- 10  1  2  3  50
```

```lua
-- ใช้ table.move เพื่อ copy array
local original = {1, 2, 3, 4, 5}
local copy = {}
table.move(original, 1, #original, 1, copy)

-- แก้ไข copy ไม่กระทบ original
copy[1] = 100
print(original[1], copy[1])  -- 1  100
```

```lua
-- ใช้ table.move เพื่อ shift elements
local arr = {1, 2, 3, 4, 5}

-- เลื่อน elements ไปทางขวา 1 ตำแหน่ง
table.move(arr, 1, #arr, 2)
arr[1] = 0  -- ใส่ค่าใหม่ที่ตำแหน่งแรก

for i = 1, #arr do io.write(arr[i] .. " ") end
print()  -- 0 1 2 3 4 5
```

---

## 7.10 table.unpack()

```lua
-- table.unpack(t [, i [, j]])
-- แตก table เป็น multiple values

local t = {10, 20, 30, 40, 50}
print(table.unpack(t))          -- 10  20  30  40  50
print(table.unpack(t, 2, 4))    -- 20  30  40

-- ใช้กับฟังก์ชัน
local function sum(a, b, c)
    return a + b + c
end

local args = {5, 10, 15}
print(sum(table.unpack(args)))  -- 30
```

```lua
-- swap สองค่าด้วย table.unpack
local a, b = 10, 20
a, b = table.unpack({b, a})
print(a, b)  -- 20  10

-- หรือแบบง่ายกว่า:
a, b = b, a
print(a, b)  -- 10  20
```

```lua
-- ตัวอย่างการใช้ unpack กับ string.format
local data = {"สมชาย", 25, "กรุงเทพ"}
local fmt = "ชื่อ: %s, อายุ: %d, เมือง: %s"
print(string.format(fmt, table.unpack(data)))
-- ชื่อ: สมชาย, อายุ: 25, เมือง: กรุงเทพ
```

---

## 7.11 ความยาว Table กับ # Operator

```lua
-- # ใช้ได้กับ sequence เท่านั้น
local seq = {10, 20, 30, 40, 50}
print(#seq)  -- 5 (ถูกต้อง)

-- sparse array (มีช่องว่าง) - # ไม่น่าเชื่อถือ!
local sparse = {}
sparse[1] = "one"
sparse[3] = "three"
sparse[5] = "five"
-- ไม่มี index 2 และ 4

-- ผลลัพธ์อาจเป็น 1, 3, หรือ 5 ขึ้นอยู่กับ implementation
-- print(#sparse)  -- ไม่แน่นอน!
```

```lua
-- วิธีนับ element ทั้งหมดใน dictionary
local function count_table(t)
    local count = 0
    for _ in pairs(t) do
        count = count + 1
    end
    return count
end

local person = {name = "สมชาย", age = 25, city = "กรุงเทพ"}
print(count_table(person))  -- 3

local mixed = {10, 20, 30, name = "Test", value = 42}
print(#mixed)             -- 3 (นับเฉพาะ sequence)
print(count_table(mixed)) -- 5 (นับทั้งหมด)
```

---

## 7.12 pairs() และ ipairs()

```lua
-- ipairs: วนลูปเฉพาะ integer keys ตามลำดับ 1, 2, 3, ...
-- หยุดเมื่อพบ nil
local fruits = {"แอปเปิล", "กล้วย", "ส้ม", nil, "มะม่วง"}

print("ipairs:")
for i, v in ipairs(fruits) do
    print(i, v)
end
-- 1  แอปเปิล
-- 2  กล้วย
-- 3  ส้ม
-- (หยุดที่ nil ไม่ถึง "มะม่วง")
```

```lua
-- pairs: วนลูปทุก key ไม่ระบุลำดับ
local person = {
    name = "นิดา",
    age = 22,
    city = "เชียงใหม่"
}

print("pairs:")
for k, v in pairs(person) do
    print(k, v)
end
-- (ลำดับไม่แน่นอน)
-- name  นิดา
-- age   22
-- city  เชียงใหม่
```

```lua
-- pairs กับ mixed table
local data = {
    "ค่าแรก",           -- key = 1
    "ค่าสอง",           -- key = 2
    name = "ชื่อ",
    value = 100
}

print("pairs ผ่านทุก key:")
for k, v in pairs(data) do
    print(type(k), k, v)
end

print("\nipairs ผ่านเฉพาะ integer sequence:")
for i, v in ipairs(data) do
    print(i, v)
end
```

```lua
-- ตัวอย่าง: กรองข้อมูล
local inventory = {
    {name = "ดาบ", qty = 5},
    {name = "โล่", qty = 0},
    {name = "ธนู", qty = 12},
    {name = "หมวกเกราะ", qty = 0},
    {name = "ยา", qty = 99}
}

print("สินค้าที่มีในสต็อก:")
for _, item in ipairs(inventory) do
    if item.qty > 0 then
        print(item.name .. ": " .. item.qty .. " ชิ้น")
    end
end
```

---

## 7.13 next() Function

`next(table, key)` คืนคู่ key-value ถัดไปใน table — ใช้สำหรับ iteration manual

```lua
-- การใช้ next()
local t = {a = 1, b = 2, c = 3}

-- next(t, nil) ให้คู่แรก
local k, v = next(t)
print(k, v)  -- (คู่แรก - ลำดับไม่แน่นอน)

-- วนลูปด้วย next()
k, v = next(t, nil)
while k ~= nil do
    print(k, v)
    k, v = next(t, k)
end
```

```lua
-- ตรวจสอบว่า table ว่างหรือไม่
local function is_empty(t)
    return next(t) == nil
end

local empty = {}
local not_empty = {1, 2, 3}

print(is_empty(empty))      -- true
print(is_empty(not_empty))  -- false
```

```lua
-- ใช้ next() เพื่อลบ elements ขณะ iteration
-- (การลบขณะใช้ pairs() อาจมีปัญหา)
local data = {a = 1, b = 2, c = 3, d = 4, e = 5}

-- เก็บ keys ที่ต้องการลบก่อน
local to_delete = {}
for k, v in pairs(data) do
    if v % 2 == 0 then  -- ลบค่า even
        to_delete[#to_delete + 1] = k
    end
end

-- แล้วค่อยลบ
for _, k in ipairs(to_delete) do
    data[k] = nil
end

for k, v in pairs(data) do
    print(k, v)
end
-- a  1
-- c  3
-- e  5
```

---

## 7.14 Table ในฐานะ Namespace

```lua
-- ใช้ table เป็น namespace สำหรับจัดกลุ่มฟังก์ชัน
local MathUtils = {}

MathUtils.PI = 3.14159265358979

function MathUtils.circle_area(r)
    return MathUtils.PI * r * r
end

function MathUtils.circle_perimeter(r)
    return 2 * MathUtils.PI * r
end

function MathUtils.triangle_area(base, height)
    return 0.5 * base * height
end

-- ใช้งาน
print(MathUtils.PI)
print(MathUtils.circle_area(5))
print(MathUtils.circle_perimeter(5))
print(MathUtils.triangle_area(10, 6))
```

```lua
-- Namespace ซ้อนกัน
local App = {
    Config = {
        version = "1.0.0",
        debug = false,
        max_users = 100
    },
    Utils = {},
    DB = {}
}

function App.Utils.format_date(year, month, day)
    return string.format("%04d-%02d-%02d", year, month, day)
end

function App.DB.connect(host, port)
    return "Connected to " .. host .. ":" .. port
end

print(App.Config.version)
print(App.Utils.format_date(2024, 3, 15))
print(App.DB.connect("localhost", 5432))
```

---

## 7.15 Table References vs Values

Table ใน Lua เป็น reference type — การ assign จะ copy reference ไม่ใช่ copy ค่า

```lua
-- Reference behavior
local a = {1, 2, 3}
local b = a  -- b ชี้ไปที่ table เดียวกัน

b[1] = 100
print(a[1])  -- 100 (a เปลี่ยนตามด้วย!)
print(b[1])  -- 100

-- ตรวจสอบว่าเป็น reference เดียวกัน
print(a == b)  -- true (เป็น object เดียวกัน)
```

```lua
-- ต่างจาก primitive types
local x = 10
local y = x  -- y ได้รับ copy ของค่า

y = 20
print(x)  -- 10 (x ไม่เปลี่ยน)
print(y)  -- 20
```

```lua
-- ปัญหา reference ใน function parameter
local function modify(t)
    t[1] = 999  -- แก้ไข table ต้นฉบับ!
end

local original = {1, 2, 3}
modify(original)
print(original[1])  -- 999 (ถูกแก้ไขแล้ว!)
```

---

## 7.16 Shallow Copy และ Deep Copy

```lua
-- Shallow Copy: คัดลอก elements ชั้นบนสุดเท่านั้น
local function shallow_copy(t)
    local copy = {}
    for k, v in pairs(t) do
        copy[k] = v
    end
    return copy
end

local original = {1, 2, 3, name = "Test"}
local copy = shallow_copy(original)

copy[1] = 100
print(original[1])  -- 1 (ไม่เปลี่ยน)
print(copy[1])      -- 100 (เปลี่ยนเฉพาะ copy)
```

```lua
-- ปัญหา Shallow Copy กับ nested tables
local original = {
    data = {1, 2, 3}  -- nested table
}
local shallow = shallow_copy(original)

shallow.data[1] = 999  -- แก้ nested table
print(original.data[1])  -- 999 (เปลี่ยนตามด้วย!)

-- เพราะ shallow.data ยังเป็น reference เดิม
print(shallow.data == original.data)  -- true
```

```lua
-- Deep Copy: คัดลอกทุกชั้นลึก
local function deep_copy(t)
    if type(t) ~= "table" then
        return t
    end
    local copy = {}
    for k, v in pairs(t) do
        copy[deep_copy(k)] = deep_copy(v)
    end
    return setmetatable(copy, getmetatable(t))
end

local original = {
    name = "Parent",
    children = {"Child1", "Child2"},
    info = {age = 25, city = "กรุงเทพ"}
}

local deep = deep_copy(original)
deep.children[1] = "Modified"
deep.info.age = 99

print(original.children[1])  -- Child1 (ไม่เปลี่ยน!)
print(original.info.age)      -- 25 (ไม่เปลี่ยน!)
print(deep.children[1])       -- Modified
print(deep.info.age)          -- 99
```

---

## 7.17 Table Comparison

```lua
-- == เปรียบเทียบ reference ไม่ใช่ content
local a = {1, 2, 3}
local b = {1, 2, 3}
local c = a

print(a == b)  -- false (คนละ object แม้ content เหมือนกัน)
print(a == c)  -- true (เป็น object เดียวกัน)
```

```lua
-- เขียนฟังก์ชัน deep compare
local function tables_equal(t1, t2)
    if t1 == t2 then return true end
    if type(t1) ~= "table" or type(t2) ~= "table" then
        return t1 == t2
    end
    
    -- เช็คจำนวน keys
    local count1, count2 = 0, 0
    for _ in pairs(t1) do count1 = count1 + 1 end
    for _ in pairs(t2) do count2 = count2 + 1 end
    if count1 ~= count2 then return false end
    
    -- เช็ค content ทุก key
    for k, v in pairs(t1) do
        if not tables_equal(v, t2[k]) then
            return false
        end
    end
    
    return true
end

local a = {1, 2, {3, 4}}
local b = {1, 2, {3, 4}}
local c = {1, 2, {3, 5}}

print(tables_equal(a, b))  -- true
print(tables_equal(a, c))  -- false
```

---

## 7.18 2D Arrays

```lua
-- สร้าง 2D array ขนาด m x n
local function create_2d(rows, cols, default)
    local t = {}
    for i = 1, rows do
        t[i] = {}
        for j = 1, cols do
            t[i][j] = default or 0
        end
    end
    return t
end

-- สร้าง grid 4x4
local grid = create_2d(4, 4, 0)
grid[1][1] = 1
grid[2][3] = 5
grid[4][4] = 9

-- พิมพ์ grid
for i = 1, 4 do
    for j = 1, 4 do
        io.write(string.format("%3d", grid[i][j]))
    end
    print()
end
```

```lua
-- เกม Tic-Tac-Toe Board
local function create_board()
    return {
        {" ", " ", " "},
        {" ", " ", " "},
        {" ", " ", " "}
    }
end

local function print_board(board)
    print("+---+---+---+")
    for i = 1, 3 do
        io.write("| ")
        for j = 1, 3 do
            io.write(board[i][j] .. " | ")
        end
        print()
        print("+---+---+---+")
    end
end

local board = create_board()
board[1][1] = "X"
board[2][2] = "O"
board[3][3] = "X"
board[1][3] = "O"

print_board(board)
```

```lua
-- Matrix multiplication
local function matrix_multiply(A, B)
    local rows_A = #A
    local cols_A = #A[1]
    local cols_B = #B[1]
    
    local C = {}
    for i = 1, rows_A do
        C[i] = {}
        for j = 1, cols_B do
            C[i][j] = 0
            for k = 1, cols_A do
                C[i][j] = C[i][j] + A[i][k] * B[k][j]
            end
        end
    end
    return C
end

local A = {{1, 2}, {3, 4}}
local B = {{5, 6}, {7, 8}}
local C = matrix_multiply(A, B)

print("A * B =")
print(C[1][1], C[1][2])  -- 19  22
print(C[2][1], C[2][2])  -- 43  50
```

---

## 7.19 Sparse Arrays

```lua
-- Sparse Array: มีเฉพาะ index ที่ต้องการ
local sparse = {}
sparse[1] = "one"
sparse[100] = "hundred"
sparse[1000] = "thousand"

-- ค้นหาได้ทันที
print(sparse[100])    -- hundred
print(sparse[1000])   -- thousand
print(sparse[2])      -- nil

-- วนลูปด้วย pairs (ไม่ใช่ ipairs)
for k, v in pairs(sparse) do
    print(k, v)
end

-- # ไม่น่าเชื่อถือกับ sparse array
-- print(#sparse)  -- อาจให้ค่าใดก็ได้
```

```lua
-- ตัวอย่าง: bitmap สำหรับ prime number sieve
local function sieve_of_eratosthenes(n)
    local is_prime = {}
    for i = 2, n do
        is_prime[i] = true
    end
    
    for i = 2, math.floor(math.sqrt(n)) do
        if is_prime[i] then
            for j = i * i, n, i do
                is_prime[j] = false
            end
        end
    end
    
    local primes = {}
    for i = 2, n do
        if is_prime[i] then
            primes[#primes + 1] = i
        end
    end
    return primes
end

local primes = sieve_of_eratosthenes(50)
print("จำนวนเฉพาะไม่เกิน 50:")
print(table.concat(primes, ", "))
-- 2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47
```

---

## 7.20 Queue Implementation

```lua
-- Queue: FIFO (First In First Out)
local Queue = {}
Queue.__index = Queue

function Queue.new()
    return setmetatable({
        data = {},
        front = 1,
        back = 0
    }, Queue)
end

function Queue:push(value)
    self.back = self.back + 1
    self.data[self.back] = value
end

function Queue:pop()
    if self:is_empty() then
        return nil
    end
    local value = self.data[self.front]
    self.data[self.front] = nil
    self.front = self.front + 1
    return value
end

function Queue:peek()
    return self.data[self.front]
end

function Queue:is_empty()
    return self.front > self.back
end

function Queue:size()
    return self.back - self.front + 1
end

-- ทดสอบ Queue
local q = Queue.new()
q:push("ลูกค้าที่ 1")
q:push("ลูกค้าที่ 2")
q:push("ลูกค้าที่ 3")

print("ขนาด queue:", q:size())    -- 3
print("หน้าสุด:", q:peek())        -- ลูกค้าที่ 1

print("รับบริการ:", q:pop())       -- ลูกค้าที่ 1
print("รับบริการ:", q:pop())       -- ลูกค้าที่ 2
print("คงเหลือ:", q:size())        -- 1
```

---

## 7.21 Stack Implementation

```lua
-- Stack: LIFO (Last In First Out)
local Stack = {}
Stack.__index = Stack

function Stack.new()
    return setmetatable({data = {}, size = 0}, Stack)
end

function Stack:push(value)
    self.size = self.size + 1
    self.data[self.size] = value
end

function Stack:pop()
    if self:is_empty() then
        return nil
    end
    local value = self.data[self.size]
    self.data[self.size] = nil
    self.size = self.size - 1
    return value
end

function Stack:peek()
    return self.data[self.size]
end

function Stack:is_empty()
    return self.size == 0
end

-- ทดสอบ Stack - ตรวจสอบ parentheses matching
local function check_brackets(expr)
    local stack = Stack.new()
    local pairs_map = {[")"] = "(", ["]"] = "[", ["}"] = "{"}
    
    for i = 1, #expr do
        local ch = expr:sub(i, i)
        if ch == "(" or ch == "[" or ch == "{" then
            stack:push(ch)
        elseif ch == ")" or ch == "]" or ch == "}" then
            if stack:is_empty() or stack:peek() ~= pairs_map[ch] then
                return false, "ตำแหน่ง " .. i
            end
            stack:pop()
        end
    end
    
    return stack:is_empty(), "วงเล็บไม่ครบ"
end

local exprs = {
    "(a + b) * (c - d)",
    "{x: [1, 2, 3]}",
    "((a + b)",
    "(a + [b)]"
}

for _, expr in ipairs(exprs) do
    local ok, msg = check_brackets(expr)
    print(string.format("%-25s -> %s", expr, ok and "ถูกต้อง" or "ผิด: " .. msg))
end
```

---

## 7.22 Set Implementation

```lua
-- Set: เก็บค่าไม่ซ้ำ
local Set = {}
Set.__index = Set

function Set.new(items)
    local s = setmetatable({data = {}}, Set)
    if items then
        for _, v in ipairs(items) do
            s:add(v)
        end
    end
    return s
end

function Set:add(value)
    self.data[value] = true
end

function Set:remove(value)
    self.data[value] = nil
end

function Set:contains(value)
    return self.data[value] == true
end

function Set:size()
    local count = 0
    for _ in pairs(self.data) do count = count + 1 end
    return count
end

function Set:to_list()
    local list = {}
    for v in pairs(self.data) do
        list[#list + 1] = v
    end
    return list
end

function Set:union(other)
    local result = Set.new()
    for v in pairs(self.data) do result:add(v) end
    for v in pairs(other.data) do result:add(v) end
    return result
end

function Set:intersection(other)
    local result = Set.new()
    for v in pairs(self.data) do
        if other:contains(v) then result:add(v) end
    end
    return result
end

function Set:difference(other)
    local result = Set.new()
    for v in pairs(self.data) do
        if not other:contains(v) then result:add(v) end
    end
    return result
end

-- ทดสอบ Set
local A = Set.new({1, 2, 3, 4, 5})
local B = Set.new({3, 4, 5, 6, 7})

print("A:", table.concat(A:to_list(), ", "))
print("B:", table.concat(B:to_list(), ", "))
print("A∪B:", table.concat(A:union(B):to_list(), ", "))
print("A∩B:", table.concat(A:intersection(B):to_list(), ", "))
print("A-B:", table.concat(A:difference(B):to_list(), ", "))

-- ลบ duplicates จาก array
local list = {1, 2, 3, 2, 4, 1, 5, 3}
local unique_set = Set.new(list)
local unique = unique_set:to_list()
table.sort(unique)
print("ไม่ซ้ำ:", table.concat(unique, ", "))  -- 1, 2, 3, 4, 5
```

---

## 7.23 Record / Struct Pattern

```lua
-- Struct-like records ใน Lua
local function new_person(name, age, email)
    return {
        name = name,
        age = age,
        email = email
    }
end

local p1 = new_person("สมชาย", 25, "somchai@email.com")
local p2 = new_person("นิดา", 22, "nida@email.com")

print(p1.name, p1.age)  -- สมชาย  25
print(p2.name, p2.age)  -- นิดา    22
```

```lua
-- Record พร้อม default values
local function new_config(opts)
    opts = opts or {}
    return {
        host = opts.host or "localhost",
        port = opts.port or 8080,
        debug = opts.debug ~= nil and opts.debug or false,
        max_connections = opts.max_connections or 100,
        timeout = opts.timeout or 30
    }
end

local config1 = new_config()
local config2 = new_config({host = "server.com", port = 443, debug = true})

print(config1.host, config1.port)  -- localhost  8080
print(config2.host, config2.port)  -- server.com  443
print(config2.debug)               -- true
```

```lua
-- Record validation
local function new_product(name, price, qty)
    assert(type(name) == "string" and #name > 0, "name ต้องไม่ว่าง")
    assert(type(price) == "number" and price >= 0, "price ต้องไม่ติดลบ")
    assert(type(qty) == "number" and qty >= 0, "qty ต้องไม่ติดลบ")
    
    return {
        name = name,
        price = price,
        qty = qty,
        total = function(self) return self.price * self.qty end
    }
end

local ok, p = pcall(new_product, "แอปเปิล", 10, 50)
if ok then
    print(p.name, p:total())  -- แอปเปิล  500
end

ok, p = pcall(new_product, "", -5, 10)
if not ok then
    print("Error:", p)  -- Error: name ต้องไม่ว่าง
end
```

---

## 7.24 Matrix Operations

```lua
-- Matrix utilities
local Matrix = {}

function Matrix.new(rows, cols, val)
    local m = {}
    for i = 1, rows do
        m[i] = {}
        for j = 1, cols do
            m[i][j] = val or 0
        end
    end
    m.rows = rows
    m.cols = cols
    return m
end

function Matrix.print(m)
    for i = 1, m.rows do
        local row = {}
        for j = 1, m.cols do
            row[j] = string.format("%6.2f", m[i][j])
        end
        print(table.concat(row, " "))
    end
end

function Matrix.add(A, B)
    assert(A.rows == B.rows and A.cols == B.cols, "ขนาดต้องเท่ากัน")
    local C = Matrix.new(A.rows, A.cols)
    for i = 1, A.rows do
        for j = 1, A.cols do
            C[i][j] = A[i][j] + B[i][j]
        end
    end
    return C
end

function Matrix.transpose(A)
    local T = Matrix.new(A.cols, A.rows)
    for i = 1, A.rows do
        for j = 1, A.cols do
            T[j][i] = A[i][j]
        end
    end
    return T
end

-- ทดสอบ
local A = Matrix.new(2, 3)
A[1] = {1, 2, 3}; A[2] = {4, 5, 6}

local B = Matrix.new(2, 3)
B[1] = {7, 8, 9}; B[2] = {10, 11, 12}

print("A + B:")
Matrix.print(Matrix.add(A, B))

print("Transpose ของ A:")
Matrix.print(Matrix.transpose(A))
```

---

## 7.25 Table Serialization

```lua
-- แปลง table เป็น string (เพื่อ debug/save)
local function serialize(val, indent)
    indent = indent or 0
    local spaces = string.rep("  ", indent)
    
    if type(val) == "number" then
        return tostring(val)
    elseif type(val) == "string" then
        return string.format("%q", val)
    elseif type(val) == "boolean" then
        return tostring(val)
    elseif type(val) == "nil" then
        return "nil"
    elseif type(val) == "table" then
        local parts = {}
        local is_array = true
        local max_idx = 0
        
        -- ตรวจว่าเป็น array หรือไม่
        for k in pairs(val) do
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                is_array = false
                break
            end
            if k > max_idx then max_idx = k end
        end
        if max_idx ~= #val then is_array = false end
        
        if is_array then
            for i, v in ipairs(val) do
                parts[i] = spaces .. "  " .. serialize(v, indent + 1)
            end
        else
            for k, v in pairs(val) do
                local key
                if type(k) == "string" and k:match("^[%a_][%w_]*$") then
                    key = k
                else
                    key = "[" .. serialize(k) .. "]"
                end
                parts[#parts + 1] = spaces .. "  " .. key .. " = " .. serialize(v, indent + 1)
            end
        end
        
        if #parts == 0 then return "{}" end
        return "{\n" .. table.concat(parts, ",\n") .. "\n" .. spaces .. "}"
    else
        return tostring(val)
    end
end

local data = {
    name = "ทดสอบ",
    scores = {95, 87, 92, 78},
    info = {age = 25, city = "กรุงเทพ"},
    active = true
}

print(serialize(data))
```

---

## 7.26 ตัวอย่างขั้นสูง: Priority Queue

```lua
-- Priority Queue ด้วย min-heap
local PriorityQueue = {}
PriorityQueue.__index = PriorityQueue

function PriorityQueue.new()
    return setmetatable({heap = {}}, PriorityQueue)
end

function PriorityQueue:push(priority, value)
    local item = {priority = priority, value = value}
    table.insert(self.heap, item)
    self:_bubble_up(#self.heap)
end

function PriorityQueue:pop()
    local n = #self.heap
    if n == 0 then return nil end
    
    local top = self.heap[1]
    self.heap[1] = self.heap[n]
    self.heap[n] = nil
    
    if #self.heap > 0 then
        self:_sink_down(1)
    end
    
    return top.priority, top.value
end

function PriorityQueue:_bubble_up(i)
    while i > 1 do
        local parent = math.floor(i / 2)
        if self.heap[parent].priority > self.heap[i].priority then
            self.heap[parent], self.heap[i] = self.heap[i], self.heap[parent]
            i = parent
        else
            break
        end
    end
end

function PriorityQueue:_sink_down(i)
    local n = #self.heap
    while true do
        local smallest = i
        local left = 2 * i
        local right = 2 * i + 1
        
        if left <= n and self.heap[left].priority < self.heap[smallest].priority then
            smallest = left
        end
        if right <= n and self.heap[right].priority < self.heap[smallest].priority then
            smallest = right
        end
        
        if smallest == i then break end
        self.heap[i], self.heap[smallest] = self.heap[smallest], self.heap[i]
        i = smallest
    end
end

function PriorityQueue:size()
    return #self.heap
end

-- ทดสอบ
local pq = PriorityQueue.new()
pq:push(3, "งานปกติ")
pq:push(1, "งานด่วนมาก")
pq:push(2, "งานด่วน")
pq:push(5, "งานไม่เร่งด่วน")
pq:push(1, "งานฉุกเฉิน")

print("ลำดับงานตามความสำคัญ:")
while pq:size() > 0 do
    local priority, task = pq:pop()
    print(string.format("Priority %d: %s", priority, task))
end
```

---

## 7.27 ตัวอย่างขั้นสูง: Graph ด้วย Adjacency List

```lua
-- Graph สำหรับแผนที่เส้นทาง
local Graph = {}
Graph.__index = Graph

function Graph.new()
    return setmetatable({
        nodes = {},
        edges = {}
    }, Graph)
end

function Graph:add_node(name, data)
    self.nodes[name] = data or {}
    if not self.edges[name] then
        self.edges[name] = {}
    end
end

function Graph:add_edge(from, to, weight)
    weight = weight or 1
    table.insert(self.edges[from], {to = to, weight = weight})
end

function Graph:neighbors(node)
    return self.edges[node] or {}
end

function Graph:bfs(start)
    local visited = {}
    local queue = {start}
    local result = {}
    visited[start] = true
    
    while #queue > 0 do
        local node = table.remove(queue, 1)
        result[#result + 1] = node
        
        for _, edge in ipairs(self:neighbors(node)) do
            if not visited[edge.to] then
                visited[edge.to] = true
                queue[#queue + 1] = edge.to
            end
        end
    end
    
    return result
end

-- แผนที่เมือง
local map = Graph.new()
map:add_node("กรุงเทพ")
map:add_node("เชียงใหม่")
map:add_node("พิษณุโลก")
map:add_node("ขอนแก่น")
map:add_node("โคราช")

map:add_edge("กรุงเทพ", "โคราช", 250)
map:add_edge("กรุงเทพ", "พิษณุโลก", 380)
map:add_edge("โคราช", "ขอนแก่น", 190)
map:add_edge("พิษณุโลก", "เชียงใหม่", 290)
map:add_edge("ขอนแก่น", "เชียงใหม่", 520)

print("BFS จากกรุงเทพ:")
local path = map:bfs("กรุงเทพ")
print(table.concat(path, " -> "))
```

---

## 7.28 Memoization ด้วย Table

```lua
-- Fibonacci แบบปกติ (ช้าสำหรับ n ใหญ่)
local function fib_slow(n)
    if n <= 1 then return n end
    return fib_slow(n-1) + fib_slow(n-2)
end

-- Fibonacci แบบ memoization (เร็ว)
local memo = {}
local function fib(n)
    if n <= 1 then return n end
    if memo[n] then return memo[n] end
    memo[n] = fib(n-1) + fib(n-2)
    return memo[n]
end

-- เปรียบเทียบความเร็ว
print("Fibonacci ด้วย memoization:")
for i = 0, 20 do
    io.write(fib(i) .. " ")
end
print()

-- Memoize decorator
local function memoize(fn)
    local cache = {}
    return function(...)
        local key = table.concat({...}, ",")
        if cache[key] == nil then
            cache[key] = fn(...)
        end
        return cache[key]
    end
end

local function expensive(n)
    local sum = 0
    for i = 1, n do sum = sum + i end
    return sum
end

local fast_expensive = memoize(expensive)
print(fast_expensive(100))  -- 5050
print(fast_expensive(100))  -- 5050 (จาก cache)
print(fast_expensive(50))   -- 1275
```

---

## 7.29 Table เป็น Event System

```lua
-- Simple Event Emitter
local EventEmitter = {}
EventEmitter.__index = EventEmitter

function EventEmitter.new()
    return setmetatable({listeners = {}}, EventEmitter)
end

function EventEmitter:on(event, callback)
    if not self.listeners[event] then
        self.listeners[event] = {}
    end
    table.insert(self.listeners[event], callback)
    return self  -- method chaining
end

function EventEmitter:emit(event, ...)
    if self.listeners[event] then
        for _, cb in ipairs(self.listeners[event]) do
            cb(...)
        end
    end
end

function EventEmitter:off(event)
    self.listeners[event] = nil
end

-- ทดสอบ
local emitter = EventEmitter.new()

emitter:on("login", function(user)
    print("User logged in:", user)
end)

emitter:on("login", function(user)
    print("Logging activity for:", user)
end)

emitter:on("logout", function(user)
    print("User logged out:", user)
end)

emitter:emit("login", "สมชาย")
emitter:emit("login", "นิดา")
emitter:emit("logout", "สมชาย")
```

---

## 7.30 Table เป็น State Machine

```lua
-- Finite State Machine
local StateMachine = {}
StateMachine.__index = StateMachine

function StateMachine.new(initial_state)
    return setmetatable({
        state = initial_state,
        transitions = {},
        actions = {}
    }, StateMachine)
end

function StateMachine:add_transition(from, event, to, action)
    if not self.transitions[from] then
        self.transitions[from] = {}
    end
    self.transitions[from][event] = {next_state = to, action = action}
end

function StateMachine:trigger(event)
    local trans = self.transitions[self.state]
    if trans and trans[event] then
        local t = trans[event]
        if t.action then t.action(self.state, t.next_state) end
        self.state = t.next_state
        return true
    end
    return false
end

-- เครื่องขายน้ำอัตโนมัติ
local vending = StateMachine.new("idle")

vending:add_transition("idle", "insert_coin", "has_coin", function(from, to)
    print("รับเหรียญ -> สถานะ: " .. to)
end)

vending:add_transition("has_coin", "select_item", "dispensing", function(from, to)
    print("เลือกสินค้า -> สถานะ: " .. to)
end)

vending:add_transition("dispensing", "item_out", "idle", function(from, to)
    print("จ่ายสินค้าแล้ว -> สถานะ: " .. to)
end)

vending:add_transition("has_coin", "cancel", "idle", function(from, to)
    print("ยกเลิก คืนเหรียญ -> สถานะ: " .. to)
end)

-- ทดสอบ
print("สถานะเริ่มต้น:", vending.state)
vending:trigger("insert_coin")
vending:trigger("select_item")
vending:trigger("item_out")
print("สถานะสุดท้าย:", vending.state)
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: จัดการรายชื่อนักเรียน
เขียนโปรแกรมที่:
1. เก็บรายชื่อนักเรียนพร้อมคะแนน 3 วิชา
2. คำนวณคะแนนเฉลี่ยของแต่ละคน
3. เรียงลำดับตามคะแนนเฉลี่ย (มากไปน้อย)
4. แสดงผลในรูปแบบตาราง

```lua
-- เฉลย
local students = {
    {name = "สมชาย", scores = {85, 90, 78}},
    {name = "นิดา", scores = {92, 88, 95}},
    {name = "อรุณ", scores = {70, 75, 80}},
    {name = "กมล", scores = {95, 92, 98}},
    {name = "พิม", scores = {60, 65, 70}},
}

-- คำนวณเฉลี่ย
for _, student in ipairs(students) do
    local sum = 0
    for _, score in ipairs(student.scores) do
        sum = sum + score
    end
    student.average = sum / #student.scores
end

-- เรียงลำดับ
table.sort(students, function(a, b) return a.average > b.average end)

-- แสดงผล
print(string.format("%-12s %6s %6s %6s %8s", "ชื่อ", "วิชา1", "วิชา2", "วิชา3", "เฉลี่ย"))
print(string.rep("-", 42))
for rank, s in ipairs(students) do
    print(string.format("%-2d %-10s %6d %6d %6d %8.2f",
        rank, s.name, s.scores[1], s.scores[2], s.scores[3], s.average))
end
```

### แบบฝึกหัดที่ 2: Linked List
เขียน Doubly Linked List ที่มี operations:
- `push_front(value)` - เพิ่มหน้า
- `push_back(value)` - เพิ่มหลัง
- `pop_front()` - ลบหน้า
- `pop_back()` - ลบหลัง
- `to_array()` - แปลงเป็น array

```lua
-- เฉลย
local LinkedList = {}
LinkedList.__index = LinkedList

function LinkedList.new()
    return setmetatable({
        head = nil,
        tail = nil,
        length = 0
    }, LinkedList)
end

function LinkedList:push_back(value)
    local node = {value = value, next = nil, prev = self.tail}
    if self.tail then
        self.tail.next = node
    else
        self.head = node
    end
    self.tail = node
    self.length = self.length + 1
end

function LinkedList:push_front(value)
    local node = {value = value, next = self.head, prev = nil}
    if self.head then
        self.head.prev = node
    else
        self.tail = node
    end
    self.head = node
    self.length = self.length + 1
end

function LinkedList:pop_back()
    if not self.tail then return nil end
    local value = self.tail.value
    self.tail = self.tail.prev
    if self.tail then
        self.tail.next = nil
    else
        self.head = nil
    end
    self.length = self.length - 1
    return value
end

function LinkedList:pop_front()
    if not self.head then return nil end
    local value = self.head.value
    self.head = self.head.next
    if self.head then
        self.head.prev = nil
    else
        self.tail = nil
    end
    self.length = self.length - 1
    return value
end

function LinkedList:to_array()
    local arr = {}
    local node = self.head
    while node do
        arr[#arr + 1] = node.value
        node = node.next
    end
    return arr
end

-- ทดสอบ
local list = LinkedList.new()
list:push_back(1)
list:push_back(2)
list:push_back(3)
list:push_front(0)

print("List:", table.concat(list:to_array(), " -> "))  -- 0 -> 1 -> 2 -> 3
print("Pop front:", list:pop_front())  -- 0
print("Pop back:", list:pop_back())    -- 3
print("List:", table.concat(list:to_array(), " -> "))  -- 1 -> 2
```

### แบบฝึกหัดที่ 3: Table Serializer/Deserializer
เขียนฟังก์ชันที่แปลง table เป็น JSON-like string แล้วแปลงกลับ

### แบบฝึกหัดที่ 4: Cache ด้วย LRU (Least Recently Used)
เขียน LRU Cache ที่มีความจุจำกัด เมื่อเต็มจะลบ item ที่ใช้นานที่สุด

### แบบฝึกหัดที่ 5: Table Diff
เขียนฟังก์ชันที่เปรียบเทียบ 2 table และบอกความแตกต่าง (เพิ่มอะไร ลบอะไร เปลี่ยนอะไร)

---

## สรุปบทที่ 7

| เรื่อง | สิ่งสำคัญ |
|--------|-----------|
| Array | Index เริ่มที่ 1 |
| Dictionary | key-value pairs |
| `#` operator | ใช้ได้กับ sequence เท่านั้น |
| `pairs()` | ทุก key ไม่มีลำดับ |
| `ipairs()` | integer sequence หยุดที่ nil |
| Reference | Table ส่งผ่าน reference |
| Deep copy | ต้องเขียน recursive copy |
| `table.concat()` | ดีกว่า string concatenation |

Table คือ "Swiss Army Knife" ของ Lua — เรียนรู้ให้ลึกจะทำให้เขียน Lua ได้อย่างทรงพลัง!
