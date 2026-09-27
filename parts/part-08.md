# บทที่ 8: Math Library - คณิตศาสตร์ใน Lua

Lua มาพร้อมกับ `math` library ที่ครอบคลุมฟังก์ชันทางคณิตศาสตร์หลักทั้งหมด ตั้งแต่ trigonometry, logarithm ไปจนถึงการสุ่มตัวเลข บทนี้จะสำรวจทุกฟังก์ชันพร้อมตัวอย่างการใช้งานจริง

---

## 8.1 ค่าคงที่ทางคณิตศาสตร์

```lua
-- math.pi: ค่า pi (π ≈ 3.14159...)
print(math.pi)          -- 3.1415926535898

-- math.huge: infinity (∞)
print(math.huge)        -- inf
print(-math.huge)       -- -inf
print(1 / 0)            -- inf (Lua 5.4)
print(math.huge + 1)    -- inf
print(math.huge * 2)    -- inf

-- math.maxinteger: integer ที่ใหญ่ที่สุด (Lua 5.3+)
print(math.maxinteger)  -- 9223372036854775807 (2^63 - 1)
print(math.maxinteger + 1)  -- -9223372036854775808 (overflow!)

-- math.mininteger: integer ที่เล็กที่สุด (Lua 5.3+)
print(math.mininteger)  -- -9223372036854775808 (-2^63)
```

```lua
-- ทดสอบค่าพิเศษ
print(math.huge == math.huge)     -- true
print(math.huge > 1000000000)     -- true
print(0 / 0)                      -- -nan หรือ nan (Not a Number)
local nan = 0 / 0
print(nan == nan)                  -- false (NaN ≠ NaN เสมอ)
print(nan ~= nan)                  -- true (วิธีตรวจสอบ NaN)

-- ฟังก์ชันตรวจสอบ
local function is_nan(x)
    return x ~= x
end

local function is_finite(x)
    return x == x and math.abs(x) ~= math.huge
end

print(is_nan(nan))        -- true
print(is_nan(42))         -- false
print(is_finite(42))      -- true
print(is_finite(math.huge)) -- false
```

---

## 8.2 math.abs() - ค่าสัมบูรณ์

```lua
-- ค่าสัมบูรณ์
print(math.abs(5))      -- 5
print(math.abs(-5))     -- 5
print(math.abs(0))      -- 0
print(math.abs(-3.14))  -- 3.14
print(math.abs(math.mininteger))  -- ระวัง: -9223372036854775808 (overflow!)
```

```lua
-- ตัวอย่างการใช้งาน: หาระยะทางระหว่างจุด
local function distance_1d(a, b)
    return math.abs(a - b)
end

print(distance_1d(5, 10))    -- 5
print(distance_1d(10, 5))    -- 5
print(distance_1d(-3, 7))    -- 10

-- ตรวจสอบว่าค่าใกล้เคียงกันหรือไม่ (floating point comparison)
local function nearly_equal(a, b, epsilon)
    epsilon = epsilon or 1e-9
    return math.abs(a - b) < epsilon
end

print(nearly_equal(0.1 + 0.2, 0.3))       -- true
print(nearly_equal(0.1 + 0.2, 0.3, 0))    -- false (ต่างกันนิดหน่อย)
```

---

## 8.3 math.ceil() และ math.floor()

```lua
-- math.ceil: ปัดขึ้น (ceiling)
print(math.ceil(3.1))    -- 4
print(math.ceil(3.9))    -- 4
print(math.ceil(3.0))    -- 3
print(math.ceil(-3.1))   -- -3
print(math.ceil(-3.9))   -- -3

-- math.floor: ปัดลง (floor)
print(math.floor(3.1))   -- 3
print(math.floor(3.9))   -- 3
print(math.floor(3.0))   -- 3
print(math.floor(-3.1))  -- -4
print(math.floor(-3.9))  -- -4
```

```lua
-- ตัวอย่าง: การคำนวณจำนวนหน้า
local function page_count(items, per_page)
    return math.ceil(items / per_page)
end

print(page_count(100, 10))  -- 10
print(page_count(101, 10))  -- 11
print(page_count(10, 10))   -- 1
print(page_count(1, 10))    -- 1
```

```lua
-- ตัวอย่าง: truncate (ตัดทศนิยม)
local function trunc(x)
    if x >= 0 then
        return math.floor(x)
    else
        return math.ceil(x)
    end
end

print(trunc(3.7))   -- 3
print(trunc(-3.7))  -- -3

-- หรือใช้ math.tointeger (Lua 5.3+)
-- math.tointeger จะ return nil ถ้า convert ไม่ได้
print(math.tointeger(5.0))   -- 5 (เป็น .0 พอดี)
print(math.tointeger(5.1))   -- nil (มีทศนิยม)
print(math.tointeger(5))     -- 5
```

```lua
-- ปัดทศนิยม n ตำแหน่ง
local function round(x, decimals)
    decimals = decimals or 0
    local factor = 10 ^ decimals
    return math.floor(x * factor + 0.5) / factor
end

print(round(3.456))      -- 3
print(round(3.456, 1))   -- 3.5
print(round(3.456, 2))   -- 3.46
print(round(-3.456, 1))  -- -3.5
print(round(3.5))        -- 4
print(round(2.5))        -- 3
```

---

## 8.4 math.sqrt() - รากที่สอง

```lua
-- รากที่สอง
print(math.sqrt(4))     -- 2.0
print(math.sqrt(9))     -- 3.0
print(math.sqrt(2))     -- 1.4142135623731
print(math.sqrt(0))     -- 0.0
-- print(math.sqrt(-1)) -- -nan (NaN สำหรับ negative)

-- ระยะทางระหว่างจุด 2 มิติ
local function distance_2d(x1, y1, x2, y2)
    local dx = x2 - x1
    local dy = y2 - y1
    return math.sqrt(dx * dx + dy * dy)
end

print(distance_2d(0, 0, 3, 4))   -- 5.0
print(distance_2d(1, 1, 4, 5))   -- 5.0
print(distance_2d(0, 0, 1, 1))   -- 1.4142135623731
```

```lua
-- ระยะทาง 3 มิติ
local function distance_3d(x1, y1, z1, x2, y2, z2)
    return math.sqrt((x2-x1)^2 + (y2-y1)^2 + (z2-z1)^2)
end

print(distance_3d(0, 0, 0, 1, 2, 2))  -- 3.0

-- รากที่ n
local function nth_root(x, n)
    return x ^ (1 / n)
end

print(nth_root(8, 3))    -- 2.0 (รากที่ 3 ของ 8)
print(nth_root(16, 4))   -- 2.0 (รากที่ 4 ของ 16)
```

---

## 8.5 math.exp() และ math.log()

```lua
-- math.exp(x): e^x
print(math.exp(0))    -- 1.0
print(math.exp(1))    -- 2.718281828... (e)
print(math.exp(2))    -- 7.3890560989307
print(math.exp(-1))   -- 0.36787944117144

-- math.log(x): natural logarithm (ln)
print(math.log(1))           -- 0.0
print(math.log(math.exp(1))) -- 1.0
print(math.log(100))         -- 4.605170185988

-- math.log(x, base): log ฐาน base (Lua 5.2+)
print(math.log(100, 10))   -- 2.0 (log base 10)
print(math.log(8, 2))      -- 3.0 (log base 2)
print(math.log(1000, 10))  -- 3.0
```

```lua
-- ตัวอย่าง: ดอกเบี้ยทบต้น
-- FV = PV * e^(r*t) สำหรับ continuous compounding
local function continuous_compound(pv, rate, years)
    return pv * math.exp(rate * years)
end

local principal = 10000
local rate = 0.05  -- 5% ต่อปี
local years = 10

local fv = continuous_compound(principal, rate, years)
print(string.format("เงินต้น: %.2f บาท", principal))
print(string.format("อัตราดอกเบี้ย: %.0f%%/ปี", rate * 100))
print(string.format("ระยะเวลา: %d ปี", years))
print(string.format("มูลค่าอนาคต: %.2f บาท", fv))
```

```lua
-- ตัวอย่าง: Entropy ของข้อมูล
local function entropy(probabilities)
    local h = 0
    for _, p in ipairs(probabilities) do
        if p > 0 then
            h = h - p * math.log(p, 2)
        end
    end
    return h
end

-- เหรียญยุติธรรม (fair coin)
print("Entropy เหรียญยุติธรรม:", entropy({0.5, 0.5}))   -- 1.0 bit

-- เหรียญที่ bias
print("Entropy เหรียญ bias (90/10):", entropy({0.9, 0.1}))  -- ~0.469 bit

-- ลูกเต๋า 6 ด้าน
local dice = {}
for i = 1, 6 do dice[i] = 1/6 end
print("Entropy ลูกเต๋า 6 ด้าน:", entropy(dice))  -- ~2.585 bits
```

---

## 8.6 Trigonometric Functions

```lua
-- คำนวณใน Radians
print(math.sin(0))              -- 0.0
print(math.sin(math.pi / 2))   -- 1.0
print(math.sin(math.pi))       -- ~1.2246e-16 (≈ 0)

print(math.cos(0))             -- 1.0
print(math.cos(math.pi))       -- -1.0
print(math.cos(math.pi / 2))   -- ~6.1e-17 (≈ 0)

print(math.tan(0))             -- 0.0
print(math.tan(math.pi / 4))   -- 1.0
```

```lua
-- แปลง degree เป็น radian และกลับ
local function deg2rad(deg)
    return deg * math.pi / 180
end

local function rad2deg(rad)
    return rad * 180 / math.pi
end

print(deg2rad(90))    -- 1.5707963... (π/2)
print(deg2rad(180))   -- 3.1415926... (π)
print(rad2deg(math.pi))       -- 180.0
print(rad2deg(math.pi / 2))   -- 90.0
```

```lua
-- ตัวอย่าง: พื้นที่สามเหลี่ยมด้วยมุม
-- Area = (1/2) * a * b * sin(C)
local function triangle_area_sas(a, b, angle_C_deg)
    local C = deg2rad(angle_C_deg)
    return 0.5 * a * b * math.sin(C)
end

local function deg2rad(deg)
    return deg * math.pi / 180
end

print(string.format("พื้นที่ = %.2f", triangle_area_sas(5, 6, 60)))  -- 12.99

-- ตรวจสอบ Pythagorean theorem: sin²(x) + cos²(x) = 1
for _, angle in ipairs({0, 30, 45, 60, 90, 120, 180}) do
    local r = deg2rad(angle)
    local sum = math.sin(r)^2 + math.cos(r)^2
    print(string.format("sin²(%3d°) + cos²(%3d°) = %.10f", angle, angle, sum))
end
```

---

## 8.7 Inverse Trigonometric Functions

```lua
-- math.asin, math.acos, math.atan คืน radians

local function deg2rad(deg) return deg * math.pi / 180 end
local function rad2deg(rad) return rad * 180 / math.pi end

-- math.asin: arcsine [-1, 1] -> [-π/2, π/2]
print(rad2deg(math.asin(0)))     -- 0.0
print(rad2deg(math.asin(0.5)))   -- 30.0
print(rad2deg(math.asin(1)))     -- 90.0

-- math.acos: arccosine [-1, 1] -> [0, π]
print(rad2deg(math.acos(1)))     -- 0.0
print(rad2deg(math.acos(0)))     -- 90.0
print(rad2deg(math.acos(-1)))    -- 180.0

-- math.atan: arctangent ด้วย 2 arguments (y, x) สำหรับ atan2
print(rad2deg(math.atan(1)))         -- 45.0
print(rad2deg(math.atan(0, 1)))      -- 0.0   (ทิศ +x)
print(rad2deg(math.atan(0, -1)))     -- 180.0 (ทิศ -x)
print(rad2deg(math.atan(1, 0)))      -- 90.0  (ทิศ +y)
print(rad2deg(math.atan(-1, 0)))     -- -90.0 (ทิศ -y)
```

```lua
-- ตัวอย่าง: หามุมของ vector
local function vector_angle(x, y)
    local rad = math.atan(y, x)
    local deg = rad * 180 / math.pi
    -- แปลงให้อยู่ใน [0, 360)
    if deg < 0 then deg = deg + 360 end
    return deg
end

print(string.format("(1, 0) -> %.1f°", vector_angle(1, 0)))    -- 0.0°
print(string.format("(0, 1) -> %.1f°", vector_angle(0, 1)))    -- 90.0°
print(string.format("(-1, 0) -> %.1f°", vector_angle(-1, 0)))  -- 180.0°
print(string.format("(0, -1) -> %.1f°", vector_angle(0, -1)))  -- 270.0°
print(string.format("(1, 1) -> %.1f°", vector_angle(1, 1)))    -- 45.0°
```

```lua
-- ตัวอย่าง: กฎ cosine หามุมในสามเหลี่ยม
-- cos(C) = (a² + b² - c²) / (2ab)
local function angle_from_sides(a, b, c)
    local cos_C = (a*a + b*b - c*c) / (2*a*b)
    return math.acos(cos_C) * 180 / math.pi
end

-- สามเหลี่ยมด้านเท่า (3-4-5 ไม่ใช่ right triangle ที่มุม C)
-- 3-4-5 right triangle
local angle_C = angle_from_sides(3, 4, 5)
print(string.format("มุม C ใน 3-4-5 triangle = %.1f°", angle_C))  -- 90.0°
```

---

## 8.8 math.max() และ math.min()

```lua
-- math.max / math.min กับ multiple arguments
print(math.max(1, 3, 2, 5, 4))      -- 5
print(math.min(1, 3, 2, 5, 4))      -- 1
print(math.max(1))                   -- 1
print(math.max(-10, -5, -1, -20))    -- -1

-- ใช้ table.unpack
local values = {3, 1, 4, 1, 5, 9, 2, 6, 5}
print(math.max(table.unpack(values)))  -- 9
print(math.min(table.unpack(values)))  -- 1
```

```lua
-- clamp: จำกัดค่าในช่วง [min, max]
local function clamp(value, min_val, max_val)
    return math.max(min_val, math.min(max_val, value))
end

print(clamp(5, 0, 10))    -- 5
print(clamp(-5, 0, 10))   -- 0
print(clamp(15, 0, 10))   -- 10
print(clamp(0.5, 0, 1))   -- 0.5

-- ตัวอย่าง: จำกัด HP ของตัวละคร
local function damage(hp, dmg, max_hp)
    hp = hp - dmg
    return clamp(hp, 0, max_hp)  -- HP ไม่ต่ำกว่า 0 ไม่เกิน max
end

local function heal(hp, amount, max_hp)
    hp = hp + amount
    return clamp(hp, 0, max_hp)
end

local max_hp = 100
local current_hp = 80

current_hp = damage(current_hp, 30, max_hp)
print("HP หลังถูกโจมตี:", current_hp)  -- 50

current_hp = heal(current_hp, 200, max_hp)
print("HP หลังรักษา:", current_hp)     -- 100 (ไม่เกิน max)

current_hp = damage(current_hp, 500, max_hp)
print("HP หลังโจมตีหนัก:", current_hp) -- 0 (ไม่ต่ำกว่า 0)
```

---

## 8.9 math.random() และ math.randomseed()

```lua
-- ตั้ง seed ก่อนใช้ random (ไม่งั้นได้ค่าเดิมทุกครั้ง)
math.randomseed(os.time())

-- math.random(): ทศนิยมระหว่าง [0, 1)
print(math.random())         -- เช่น 0.37291029...

-- math.random(n): integer ระหว่าง [1, n]
print(math.random(6))        -- 1-6 (ลูกเต๋า)
print(math.random(100))      -- 1-100

-- math.random(m, n): integer ระหว่าง [m, n]
print(math.random(1, 10))    -- 1-10
print(math.random(-5, 5))    -- -5 ถึง 5
print(math.random(50, 100))  -- 50-100
```

```lua
-- สุ่มตัวเลขไม่ซ้ำ
local function shuffle(t)
    local n = #t
    for i = n, 2, -1 do
        local j = math.random(i)
        t[i], t[j] = t[j], t[i]
    end
    return t
end

local deck = {}
for i = 1, 10 do deck[i] = i end
shuffle(deck)
print("สุ่มลำดับ:", table.concat(deck, " "))
```

```lua
-- สุ่มเลือก element จาก array
local function random_choice(t)
    return t[math.random(#t)]
end

local names = {"สมชาย", "นิดา", "อรุณ", "กมล", "พิม"}
for i = 1, 5 do
    print("สุ่ม:", random_choice(names))
end
```

```lua
-- สุ่มเลือกหลาย element (ไม่ซ้ำ)
local function sample(t, k)
    local copy = {}
    for i, v in ipairs(t) do copy[i] = v end
    shuffle(copy)
    local result = {}
    for i = 1, k do result[i] = copy[i] end
    return result
end

local function shuffle(t)
    for i = #t, 2, -1 do
        local j = math.random(i)
        t[i], t[j] = t[j], t[i]
    end
    return t
end

local items = {"A", "B", "C", "D", "E", "F", "G", "H"}
local chosen = sample(items, 3)
print("เลือก 3 จาก 8:", table.concat(chosen, ", "))
```

```lua
-- Weighted Random (สุ่มตามน้ำหนัก)
local function weighted_random(items)
    -- items = {{value=.., weight=..}, ...}
    local total = 0
    for _, item in ipairs(items) do
        total = total + item.weight
    end
    
    local r = math.random() * total
    local cum = 0
    for _, item in ipairs(items) do
        cum = cum + item.weight
        if r <= cum then
            return item.value
        end
    end
    return items[#items].value
end

local loot_table = {
    {value = "ทองคำ", weight = 5},
    {value = "เงิน", weight = 20},
    {value = "ทองแดง", weight = 75}
}

-- ทดสอบ 1000 ครั้ง
local counts = {}
for i = 1, 1000 do
    local item = weighted_random(loot_table)
    counts[item] = (counts[item] or 0) + 1
end

for _, loot in ipairs(loot_table) do
    print(string.format("%-10s: %d ครั้ง (%.1f%%)",
        loot.value, counts[loot.value] or 0,
        (counts[loot.value] or 0) / 10))
end
```

---

## 8.10 math.fmod() และ math.modf()

```lua
-- math.fmod(x, y): modulus สำหรับ float
print(math.fmod(7, 3))      -- 1.0
print(math.fmod(7.5, 2.5))  -- 0.0 (7.5 / 2.5 = 3 พอดี)
print(math.fmod(7.5, 2))    -- 1.5
print(math.fmod(-7, 3))     -- -1.0 (เครื่องหมายตาม x)

-- % operator ก็คือ modulus
print(7 % 3)       -- 1
print(7.5 % 2)     -- 1.5
print(-7 % 3)      -- 2 (ใน Lua 5.3+ ผลบวกเสมอสำหรับ %)
```

```lua
-- math.modf(x): แบ่ง float เป็น integer และ fractional parts
local i, f = math.modf(3.75)
print(i, f)   -- 3.0  0.75

local i2, f2 = math.modf(-3.75)
print(i2, f2)  -- -3.0  -0.75

-- ตัวอย่าง: แยกชั่วโมงและนาทีจากทศนิยม
local hours_decimal = 2.75  -- 2 ชั่วโมง 45 นาที
local h, frac = math.modf(hours_decimal)
local m = frac * 60
print(string.format("%.2f ชม. = %d ชม. %d นาที", hours_decimal, h, m))
```

```lua
-- ตัวอย่าง: wrap angle ให้อยู่ใน [0, 360)
local function wrap_angle(angle)
    return angle % 360
end

print(wrap_angle(370))   -- 10.0
print(wrap_angle(-10))   -- 350.0
print(wrap_angle(720))   -- 0.0
print(wrap_angle(180))   -- 180.0

-- wrap ให้อยู่ใน [-180, 180)
local function wrap_angle_signed(angle)
    angle = angle % 360
    if angle > 180 then angle = angle - 360 end
    return angle
end

print(wrap_angle_signed(270))   -- -90.0
print(wrap_angle_signed(-90))   -- -90.0
print(wrap_angle_signed(180))   -- 180.0
```

---

## 8.11 math.type() - ตรวจสอบ Integer vs Float

```lua
-- math.type (Lua 5.3+)
print(math.type(1))      -- integer
print(math.type(1.0))    -- float
print(math.type(1e10))   -- float
print(math.type("1"))    -- false (ไม่ใช่ number เลย)
print(math.type(true))   -- false

-- ตรวจสอบ type
local function is_integer(x)
    return math.type(x) == "integer"
end

local function is_float(x)
    return math.type(x) == "float"
end

print(is_integer(5))    -- true
print(is_integer(5.0))  -- false (แม้จะเท่ากับ 5!)
print(is_float(5.0))    -- true
```

```lua
-- ความแตกต่างระหว่าง integer และ float
print(5 == 5.0)         -- true (ค่าเท่ากัน)
print(math.type(5) == math.type(5.0))  -- false (type ต่างกัน)

-- integer division
print(10 // 3)     -- 3  (integer result)
print(10.0 // 3)   -- 3.0 (float result)
print(10 // 3.0)   -- 3.0 (float result)

-- ตัวอย่าง: ตรวจสอบว่าเป็นจำนวนเต็มหรือไม่
local function is_whole_number(x)
    return x == math.floor(x)
end

print(is_whole_number(5))    -- true
print(is_whole_number(5.0))  -- true
print(is_whole_number(5.5))  -- false
```

---

## 8.12 การแปลง Degree / Radian

```lua
-- Utility functions
local DEG_TO_RAD = math.pi / 180
local RAD_TO_DEG = 180 / math.pi

local function deg_to_rad(deg)
    return deg * DEG_TO_RAD
end

local function rad_to_deg(rad)
    return rad * RAD_TO_DEG
end

-- ตารางค่า sin, cos ที่มุมหลัก
local angles = {0, 30, 45, 60, 90, 120, 135, 150, 180}
print(string.format("%-8s %-12s %-12s %-12s", "มุม", "sin", "cos", "tan"))
print(string.rep("-", 46))

for _, angle in ipairs(angles) do
    local r = deg_to_rad(angle)
    local s = math.sin(r)
    local c = math.cos(r)
    local t = (angle == 90 or angle == 270) and "∞" or string.format("%.4f", math.tan(r))
    print(string.format("%-8s %-12s %-12s %-12s",
        angle .. "°",
        string.format("%.4f", s),
        string.format("%.4f", c),
        t))
end
```

---

## 8.13 ฟังก์ชันทางสถิติ

```lua
-- ค่าเฉลี่ย (Mean)
local function mean(data)
    local sum = 0
    for _, v in ipairs(data) do sum = sum + v end
    return sum / #data
end

-- ค่ามัธยฐาน (Median)
local function median(data)
    local sorted = {}
    for i, v in ipairs(data) do sorted[i] = v end
    table.sort(sorted)
    local n = #sorted
    if n % 2 == 0 then
        return (sorted[n/2] + sorted[n/2 + 1]) / 2
    else
        return sorted[math.ceil(n/2)]
    end
end

-- ฐานนิยม (Mode)
local function mode(data)
    local counts = {}
    for _, v in ipairs(data) do
        counts[v] = (counts[v] or 0) + 1
    end
    local max_count = 0
    local modes = {}
    for v, count in pairs(counts) do
        if count > max_count then
            max_count = count
            modes = {v}
        elseif count == max_count then
            modes[#modes + 1] = v
        end
    end
    table.sort(modes)
    return modes, max_count
end

-- ส่วนเบี่ยงเบนมาตรฐาน (Standard Deviation)
local function std_dev(data)
    local m = mean(data)
    local sum_sq = 0
    for _, v in ipairs(data) do
        sum_sq = sum_sq + (v - m)^2
    end
    return math.sqrt(sum_sq / #data)
end

-- Variance
local function variance(data)
    local m = mean(data)
    local sum_sq = 0
    for _, v in ipairs(data) do
        sum_sq = sum_sq + (v - m)^2
    end
    return sum_sq / #data
end

-- ทดสอบ
local scores = {85, 90, 78, 95, 72, 88, 91, 84, 76, 93}

print("ข้อมูล:", table.concat(scores, ", "))
print(string.format("ค่าเฉลี่ย: %.2f", mean(scores)))
print(string.format("ค่ามัธยฐาน: %.2f", median(scores)))
local m, count = mode(scores)
print("ฐานนิยม:", table.concat(m, ", "), "(ปรากฏ " .. count .. " ครั้ง)")
print(string.format("ส่วนเบี่ยงเบนมาตรฐาน: %.2f", std_dev(scores)))
print(string.format("Variance: %.2f", variance(scores)))
```

```lua
-- Range, Min, Max
local function stats_summary(data)
    local n = #data
    local min_val = data[1]
    local max_val = data[1]
    local sum = 0
    
    for _, v in ipairs(data) do
        if v < min_val then min_val = v end
        if v > max_val then max_val = v end
        sum = sum + v
    end
    
    local avg = sum / n
    
    -- Variance
    local var_sum = 0
    for _, v in ipairs(data) do
        var_sum = var_sum + (v - avg)^2
    end
    
    return {
        count = n,
        min = min_val,
        max = max_val,
        range = max_val - min_val,
        sum = sum,
        mean = avg,
        variance = var_sum / n,
        std = math.sqrt(var_sum / n)
    }
end

local data = {4, 7, 13, 2, 1, 9, 5, 8, 3, 6}
local s = stats_summary(data)

print(string.format("จำนวน: %d", s.count))
print(string.format("Min: %g, Max: %g, Range: %g", s.min, s.max, s.range))
print(string.format("Sum: %g, Mean: %.4f", s.sum, s.mean))
print(string.format("Std Dev: %.4f", s.std))
```

---

## 8.14 การคำนวณเรขาคณิต

```lua
-- ฟังก์ชันเรขาคณิต 2D
local Geometry = {}

-- วงกลม
function Geometry.circle_area(r)
    return math.pi * r * r
end

function Geometry.circle_perimeter(r)
    return 2 * math.pi * r
end

-- สามเหลี่ยม (Heron's formula)
function Geometry.triangle_area_heron(a, b, c)
    local s = (a + b + c) / 2
    return math.sqrt(s * (s-a) * (s-b) * (s-c))
end

-- สี่เหลี่ยมผืนผ้า
function Geometry.rectangle_area(w, h)
    return w * h
end

function Geometry.rectangle_diagonal(w, h)
    return math.sqrt(w*w + h*h)
end

-- ห้าเหลี่ยมปกติ (regular polygon)
function Geometry.regular_polygon_area(n, side)
    return (n * side * side) / (4 * math.tan(math.pi / n))
end

-- ทดสอบ
print("วงกลม r=5:")
print(string.format("  พื้นที่ = %.4f", Geometry.circle_area(5)))
print(string.format("  เส้นรอบวง = %.4f", Geometry.circle_perimeter(5)))

print("\nสามเหลี่ยม 3-4-5:")
print(string.format("  พื้นที่ = %.4f", Geometry.triangle_area_heron(3, 4, 5)))

print("\nสี่เหลี่ยม 3x4:")
print(string.format("  พื้นที่ = %.4f", Geometry.rectangle_area(3, 4)))
print(string.format("  แนวทะแยง = %.4f", Geometry.rectangle_diagonal(3, 4)))

print("\nห้าเหลี่ยมปกติ ด้านละ 5:")
print(string.format("  พื้นที่ = %.4f", Geometry.regular_polygon_area(5, 5)))
```

```lua
-- ตัวอย่าง: การหมุน vector
local function rotate_vector(x, y, angle_deg)
    local rad = angle_deg * math.pi / 180
    local cos_a = math.cos(rad)
    local sin_a = math.sin(rad)
    return x * cos_a - y * sin_a, x * sin_a + y * cos_a
end

-- หมุน (1, 0) ทีละ 45 องศา
print("หมุน vector (1, 0):")
for deg = 0, 360, 45 do
    local nx, ny = rotate_vector(1, 0, deg)
    print(string.format("  %3d° -> (%.4f, %.4f)", deg, nx, ny))
end
```

---

## 8.15 การคำนวณการเงิน

```lua
-- ดอกเบี้ยทบต้น (Compound Interest)
-- FV = PV * (1 + r/n)^(n*t)
local function compound_interest(pv, annual_rate, n_per_year, years)
    local rate_per_period = annual_rate / n_per_year
    local total_periods = n_per_year * years
    return pv * (1 + rate_per_period) ^ total_periods
end

local principal = 100000  -- 100,000 บาท
local rate = 0.06         -- 6% ต่อปี

print("เงินต้น 100,000 บาท อัตรา 6%/ปี:")
print(string.format("  รายปี     10 ปี: %.2f บาท", compound_interest(principal, rate, 1, 10)))
print(string.format("  รายครึ่งปี 10 ปี: %.2f บาท", compound_interest(principal, rate, 2, 10)))
print(string.format("  รายเดือน   10 ปี: %.2f บาท", compound_interest(principal, rate, 12, 10)))
print(string.format("  รายวัน    10 ปี: %.2f บาท", compound_interest(principal, rate, 365, 10)))
```

```lua
-- Present Value (NPV calculation helper)
local function present_value(fv, rate, years)
    return fv / (1 + rate) ^ years
end

-- Net Present Value
local function npv(initial_investment, cash_flows, rate)
    local pv = -initial_investment
    for t, cf in ipairs(cash_flows) do
        pv = pv + present_value(cf, rate, t)
    end
    return pv
end

-- ตัวอย่าง: ลงทุน 100,000 บาท รับเงิน 30,000 ต่อปีเป็นเวลา 5 ปี
local cash_flows = {30000, 30000, 30000, 30000, 30000}
local npv_result = npv(100000, cash_flows, 0.10)
print(string.format("\nNPV (discount rate 10%%): %.2f บาท", npv_result))
print(npv_result > 0 and "คุ้มค่าลงทุน" or "ไม่คุ้มค่าลงทุน")
```

```lua
-- คำนวณ EMI (Equated Monthly Installment) - ผ่อนรายเดือน
local function emi(principal, annual_rate, months)
    local r = annual_rate / 12  -- อัตราต่อเดือน
    if r == 0 then
        return principal / months
    end
    return principal * r * (1 + r)^months / ((1 + r)^months - 1)
end

local loan = 500000      -- กู้ 500,000 บาท
local rate = 0.065       -- 6.5% ต่อปี
local term = 120         -- 10 ปี = 120 เดือน

local monthly = emi(loan, rate, term)
local total = monthly * term
local interest = total - loan

print(string.format("\nสินเชื่อ %.0f บาท อัตรา %.1f%% %d เดือน:", loan, rate*100, term))
print(string.format("  ผ่อนรายเดือน: %.2f บาท", monthly))
print(string.format("  ยอดรวม: %.2f บาท", total))
print(string.format("  ดอกเบี้ยรวม: %.2f บาท", interest))
```

---

## 8.16 เกมตัวเลขด้วย math.random

```lua
-- เกมทาย hi-lo
math.randomseed(os.time())

local function play_hilo_demo()
    local secret = math.random(1, 100)
    local attempts = 0
    local max_attempts = 7
    
    print("=== เกม Hi-Lo ===")
    print("คิดเลขระหว่าง 1-100 แล้วเดาดู (จำลอง)")
    print("เลขที่ถูก:", secret, "(แสดงสำหรับ demo)")
    
    -- จำลอง AI guessing (binary search)
    local low, high = 1, 100
    
    while attempts < max_attempts do
        attempts = attempts + 1
        local guess = math.floor((low + high) / 2)
        
        io.write(string.format("ครั้งที่ %d: เดา %d -> ", attempts, guess))
        
        if guess == secret then
            print("ถูกต้อง!")
            break
        elseif guess < secret then
            print("น้อยกว่า")
            low = guess + 1
        else
            print("มากกว่า")
            high = guess - 1
        end
    end
    
    print("ใช้ไป", attempts, "ครั้ง")
end

play_hilo_demo()
```

```lua
-- สุ่มสร้างแผนที่ dungeon อย่างง่าย
math.randomseed(42)  -- ใช้ seed คงที่เพื่อ reproducibility

local function generate_dungeon(width, height, fill_chance)
    local dungeon = {}
    
    for y = 1, height do
        dungeon[y] = {}
        for x = 1, width do
            if x == 1 or x == width or y == 1 or y == height then
                dungeon[y][x] = "#"  -- ผนัง
            elseif math.random() < fill_chance then
                dungeon[y][x] = "#"  -- ผนังสุ่ม
            else
                dungeon[y][x] = "."  -- พื้น
            end
        end
    end
    
    return dungeon
end

local dungeon = generate_dungeon(20, 10, 0.35)
print("แผนที่ dungeon สุ่ม:")
for _, row in ipairs(dungeon) do
    print(table.concat(row))
end
```

---

## 8.17 Interpolation และ Easing Functions

```lua
-- Linear Interpolation (lerp)
local function lerp(a, b, t)
    return a + (b - a) * t
end

-- ตัวอย่าง: animation
print("Lerp จาก 0 ถึง 100:")
for t = 0, 1, 0.1 do
    io.write(string.format("%.1f ", lerp(0, 100, t)))
end
print()
```

```lua
-- Easing functions สำหรับ animation
local Ease = {}

function Ease.in_quad(t)
    return t * t
end

function Ease.out_quad(t)
    return 1 - (1 - t)^2
end

function Ease.in_out_quad(t)
    if t < 0.5 then
        return 2 * t * t
    else
        return 1 - (-2 * t + 2)^2 / 2
    end
end

function Ease.in_sine(t)
    return 1 - math.cos(t * math.pi / 2)
end

function Ease.out_bounce(t)
    local n1 = 7.5625
    local d1 = 2.75
    if t < 1/d1 then
        return n1 * t * t
    elseif t < 2/d1 then
        t = t - 1.5/d1
        return n1 * t * t + 0.75
    elseif t < 2.5/d1 then
        t = t - 2.25/d1
        return n1 * t * t + 0.9375
    else
        t = t - 2.625/d1
        return n1 * t * t + 0.984375
    end
end

-- แสดงกราฟ ASCII ของ easing
local function show_easing(fn, name)
    local width = 40
    io.write(string.format("%-15s [", name))
    for i = 0, width do
        local t = i / width
        local val = fn(t)
        -- แสดงเป็น bar chart แนวนอน
        if math.abs(val - 0.5) < 0.1 then
            io.write("*")
        else
            io.write(" ")
        end
    end
    print("]")
end

print("Easing functions (ค่าใกล้ 0.5 แสดง *):")
show_easing(Ease.in_quad, "in_quad")
show_easing(Ease.out_quad, "out_quad")
show_easing(Ease.in_out_quad, "in_out_quad")
show_easing(Ease.in_sine, "in_sine")
```

---

## 8.18 ตัวอย่างขั้นสูง: Fast Inverse Square Root

```lua
-- Algorithm ที่มีชื่อเสียงจาก Quake III
-- 1 / sqrt(x) แบบเร็ว (เพื่อการศึกษา)
local function fast_inv_sqrt(x)
    -- วิธี Newton's method approximation
    local half = x * 0.5
    local approx = 1 / math.sqrt(x)  -- เริ่มจาก approximation ดีๆ
    
    -- Newton's iteration: y = y * (1.5 - half * y * y)
    approx = approx * (1.5 - half * approx * approx)
    
    return approx
end

-- ทดสอบ
for _, x in ipairs({1, 4, 9, 16, 25, 100}) do
    local exact = 1 / math.sqrt(x)
    local approx = fast_inv_sqrt(x)
    local error = math.abs(exact - approx) / exact * 100
    print(string.format("1/sqrt(%3d) = %.8f, approx = %.8f, error = %.6f%%",
        x, exact, approx, error))
end
```

---

## 8.19 การคำนวณตัวเลขขนาดใหญ่

```lua
-- Lua integer 64-bit สามารถเก็บได้ถึง ~9.2 * 10^18
print("Max integer:", math.maxinteger)

-- Factorial ด้วย float เพื่อรองรับเลขใหญ่
local function factorial_float(n)
    if n <= 1 then return 1.0 end
    local result = 1.0
    for i = 2, n do
        result = result * i
    end
    return result
end

for i = 1, 20 do
    print(string.format("%2d! = %.6e", i, factorial_float(i)))
end
```

```lua
-- Combinations C(n, k) = n! / (k! * (n-k)!)
local function combination(n, k)
    if k > n or k < 0 then return 0 end
    if k == 0 or k == n then return 1 end
    
    -- ใช้ Pascal's triangle เพื่อหลีกเลี่ยง overflow
    k = math.min(k, n - k)
    local result = 1
    for i = 1, k do
        result = result * (n - k + i) / i
    end
    return math.floor(result + 0.5)
end

-- ตัวอย่าง: โอกาสในการออกล็อตเตอรี่
print("C(6, 6) =", combination(6, 6))    -- 1
print("C(49, 6) =", combination(49, 6))  -- 13983816

-- โอกาส 1 ใน ?
print(string.format("โอกาสถูกล็อตเตอรี่ 6/49 = 1 ใน %d", combination(49, 6)))
```

---

## 8.20 Numeric Methods

```lua
-- หารากของสมการด้วย Newton-Raphson Method
-- หา x ที่ทำให้ f(x) = 0
local function newton_raphson(f, df, x0, tolerance, max_iter)
    tolerance = tolerance or 1e-10
    max_iter = max_iter or 100
    
    local x = x0
    for i = 1, max_iter do
        local fx = f(x)
        if math.abs(fx) < tolerance then
            return x, i
        end
        local dfx = df(x)
        if dfx == 0 then
            return nil, "Derivative is zero"
        end
        x = x - fx / dfx
    end
    return x, max_iter
end

-- หาราก sqrt(2) โดยหารากของ x^2 - 2 = 0
local sqrt2, iters = newton_raphson(
    function(x) return x*x - 2 end,   -- f(x) = x^2 - 2
    function(x) return 2*x end,         -- f'(x) = 2x
    1.5,                                 -- เริ่มที่ x = 1.5
    1e-12
)

print(string.format("sqrt(2) = %.15f (ใช้ %d iterations)", sqrt2, iters))
print(string.format("math.sqrt(2) = %.15f", math.sqrt(2)))
print(string.format("ความต่าง = %.2e", math.abs(sqrt2 - math.sqrt(2))))
```

```lua
-- Integration ด้วย Simpson's Rule
local function integrate_simpsons(f, a, b, n)
    if n % 2 ~= 0 then n = n + 1 end  -- n ต้องเป็นเลขคู่
    
    local h = (b - a) / n
    local sum = f(a) + f(b)
    
    for i = 1, n - 1 do
        local x = a + i * h
        sum = sum + (i % 2 == 0 and 2 or 4) * f(x)
    end
    
    return sum * h / 3
end

-- ∫₀^π sin(x) dx = 2
local result = integrate_simpsons(math.sin, 0, math.pi, 1000)
print(string.format("∫₀^π sin(x) dx = %.10f (ควรเป็น 2)", result))

-- ∫₀^1 x² dx = 1/3
result = integrate_simpsons(function(x) return x*x end, 0, 1, 1000)
print(string.format("∫₀^1 x² dx = %.10f (ควรเป็น 0.3333...)", result))
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: ตัวแปลงหน่วย

เขียนโปรแกรมแปลงหน่วยต่างๆ:
- องศา Celsius <-> Fahrenheit <-> Kelvin
- ระยะทาง: เมตร <-> ไมล์ <-> กิโลเมตร
- น้ำหนัก: กิโลกรัม <-> ปอนด์

```lua
-- เฉลย
local Convert = {}

function Convert.celsius_to_fahrenheit(c)
    return c * 9/5 + 32
end

function Convert.fahrenheit_to_celsius(f)
    return (f - 32) * 5/9
end

function Convert.celsius_to_kelvin(c)
    return c + 273.15
end

function Convert.km_to_miles(km)
    return km * 0.621371
end

function Convert.miles_to_km(miles)
    return miles / 0.621371
end

function Convert.kg_to_pounds(kg)
    return kg * 2.20462
end

-- ทดสอบ
print(string.format("100°C = %.1f°F", Convert.celsius_to_fahrenheit(100)))
print(string.format("212°F = %.1f°C", Convert.fahrenheit_to_celsius(212)))
print(string.format("0°C = %.2f K", Convert.celsius_to_kelvin(0)))
print(string.format("100 km = %.3f miles", Convert.km_to_miles(100)))
print(string.format("70 kg = %.2f lbs", Convert.kg_to_pounds(70)))
```

### แบบฝึกหัดที่ 2: เกมทอยลูกเต๋า

เขียนจำลองการทอย Yahtzee:
- ทอย 5 ลูก
- ตรวจสอบผลลัพธ์: three-of-a-kind, four-of-a-kind, full house, straight, yahtzee

```lua
-- เฉลย
math.randomseed(os.time())

local function roll_dice(n)
    local dice = {}
    for i = 1, n do
        dice[i] = math.random(1, 6)
    end
    return dice
end

local function check_yahtzee(dice)
    local counts = {}
    for _, d in ipairs(dice) do
        counts[d] = (counts[d] or 0) + 1
    end
    
    local max_count = 0
    local count_list = {}
    for _, c in pairs(counts) do
        count_list[#count_list + 1] = c
        if c > max_count then max_count = c end
    end
    table.sort(count_list, function(a, b) return a > b end)
    
    if max_count == 5 then return "YAHTZEE!"
    elseif max_count == 4 then return "Four of a Kind"
    elseif count_list[1] == 3 and count_list[2] == 2 then return "Full House"
    elseif max_count == 3 then return "Three of a Kind"
    else
        -- Check straight
        local sorted = {}
        for d in pairs(counts) do sorted[#sorted + 1] = d end
        table.sort(sorted)
        if #sorted == 5 then
            if sorted[5] - sorted[1] == 4 then return "Large Straight"
            end
        end
        if count_list[1] == 2 and count_list[2] == 2 then return "Two Pair"
        elseif max_count == 2 then return "One Pair"
        else return "Nothing"
        end
    end
end

-- ทอย 10 ครั้ง
for i = 1, 10 do
    local dice = roll_dice(5)
    local result = check_yahtzee(dice)
    print(string.format("ครั้งที่ %2d: [%s] -> %s",
        i, table.concat(dice, " "), result))
end
```

### แบบฝึกหัดที่ 3: Mandelbrot Set

เขียนโปรแกรมแสดง Mandelbrot Set แบบ ASCII

```lua
-- เฉลย
local function mandelbrot(cx, cy, max_iter)
    local x, y = 0, 0
    for i = 1, max_iter do
        local x2, y2 = x*x, y*y
        if x2 + y2 > 4 then return i end
        x, y = x2 - y2 + cx, 2*x*y + cy
    end
    return max_iter
end

local chars = " .:-=+*#%@"
local w, h = 60, 25
local x_min, x_max = -2.5, 1.0
local y_min, y_max = -1.25, 1.25

for row = 1, h do
    local line = {}
    for col = 1, w do
        local cx = x_min + (col - 1) * (x_max - x_min) / (w - 1)
        local cy = y_min + (row - 1) * (y_max - y_min) / (h - 1)
        local iter = mandelbrot(cx, cy, #chars)
        local idx = math.floor(iter * #chars / #chars) + 1
        line[col] = chars:sub(idx, idx)
    end
    print(table.concat(line))
end
```

---

## สรุปบทที่ 8

| ฟังก์ชัน | หน้าที่ |
|---------|---------|
| `math.pi`, `math.huge` | ค่าคงที่ |
| `math.abs(x)` | ค่าสัมบูรณ์ |
| `math.ceil(x)`, `math.floor(x)` | ปัดขึ้น/ลง |
| `math.sqrt(x)` | รากที่สอง |
| `math.exp(x)`, `math.log(x)` | e^x, ln(x) |
| `math.sin/cos/tan` | ตรีโกณมิติ (radians) |
| `math.random()` | สุ่มตัวเลข |
| `math.max/min` | ค่าสูงสุด/ต่ำสุด |
| `math.type(x)` | integer หรือ float |
| `math.tointeger(x)` | แปลงเป็น integer |

Math library ของ Lua ครอบคลุมทุกอย่างที่ต้องการสำหรับงานทั่วไป สำหรับงานหนักเช่น big number หรือ symbolic math อาจต้องใช้ library เพิ่มเติม
