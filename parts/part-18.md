# บทที่ 18: Metamethods ทั้งหมด

## บทนำ

Metamethods คือฟังก์ชันพิเศษที่ Lua เรียกใช้โดยอัตโนมัติเมื่อมีการดำเนินการบางอย่างกับ table หรือ userdata Metamethods ถูกเก็บไว้ใน metatable ซึ่งเป็น table พิเศษที่บอก Lua ว่าควรทำอะไรเมื่อพบ operator หรือการดำเนินการต่างๆ

Metamethods ช่วยให้เราสามารถ:
- กำหนดพฤติกรรมของ operator สำหรับ object ที่สร้างเอง
- สร้าง class และ object-oriented programming
- ควบคุมการเข้าถึงและแก้ไข table
- สร้าง proxy object
- กำหนดการแสดงผลของ object

---

## 18.1 Arithmetic Metamethods (การคำนวณ)

### __add - การบวก

```lua
-- ตัวอย่างที่ 1: Vector2D พื้นฐานด้วย __add
local Vector2D = {}
Vector2D.__index = Vector2D

function Vector2D.new(x, y)
    return setmetatable({x = x, y = y}, Vector2D)
end

function Vector2D:__add(other)
    return Vector2D.new(self.x + other.x, self.y + other.y)
end

function Vector2D:__tostring()
    return string.format("Vector2D(%g, %g)", self.x, self.y)
end

local v1 = Vector2D.new(1, 2)
local v2 = Vector2D.new(3, 4)
local v3 = v1 + v2

print(tostring(v3))  -- Vector2D(4, 6)
```

```lua
-- ตัวอย่างที่ 2: __add กับ mixed types
local Number = {}
Number.__index = Number

function Number.new(val)
    return setmetatable({value = val}, Number)
end

Number.__add = function(a, b)
    local aval = type(a) == "table" and a.value or a
    local bval = type(b) == "table" and b.value or b
    return Number.new(aval + bval)
end

local n1 = Number.new(10)
local n2 = Number.new(5)
local n3 = n1 + n2       -- Number + Number
local n4 = n1 + 3        -- Number + number
local n5 = 7 + n2        -- number + Number

print(n3.value)  -- 15
print(n4.value)  -- 13
print(n5.value)  -- 12
```

### __sub - การลบ

```lua
-- ตัวอย่างที่ 3: __sub สำหรับ Set
local Set = {}
Set.__index = Set

function Set.new(list)
    local s = setmetatable({}, Set)
    s._data = {}
    for _, v in ipairs(list or {}) do
        s._data[v] = true
    end
    return s
end

-- Set difference: A - B = elements in A but not in B
function Set:__sub(other)
    local result = Set.new()
    for k in pairs(self._data) do
        if not other._data[k] then
            result._data[k] = true
        end
    end
    return result
end

function Set:toList()
    local list = {}
    for k in pairs(self._data) do
        list[#list + 1] = k
    end
    table.sort(list)
    return list
end

local A = Set.new({1, 2, 3, 4, 5})
local B = Set.new({3, 4, 5, 6, 7})
local diff = A - B

print(table.concat(diff:toList(), ", "))  -- 1, 2
```

### __mul - การคูณ

```lua
-- ตัวอย่างที่ 4: Matrix multiplication
local Matrix = {}
Matrix.__index = Matrix

function Matrix.new(rows, cols, data)
    local m = setmetatable({}, Matrix)
    m.rows = rows
    m.cols = cols
    m.data = data or {}
    -- Initialize with zeros if no data
    if not data then
        for i = 1, rows do
            m.data[i] = {}
            for j = 1, cols do
                m.data[i][j] = 0
            end
        end
    end
    return m
end

function Matrix:__mul(other)
    assert(self.cols == other.rows, "Matrix dimensions incompatible for multiplication")
    local result = Matrix.new(self.rows, other.cols)
    for i = 1, self.rows do
        for j = 1, other.cols do
            local sum = 0
            for k = 1, self.cols do
                sum = sum + self.data[i][k] * other.data[k][j]
            end
            result.data[i][j] = sum
        end
    end
    return result
end

function Matrix:print()
    for i = 1, self.rows do
        local row = {}
        for j = 1, self.cols do
            row[j] = string.format("%6.2f", self.data[i][j])
        end
        print(table.concat(row, " "))
    end
end

local A = Matrix.new(2, 3, {{1,2,3},{4,5,6}})
local B = Matrix.new(3, 2, {{7,8},{9,10},{11,12}})
local C = A * B
C:print()
-- Output:
--  58.00  64.00
-- 139.00 154.00
```

```lua
-- ตัวอย่างที่ 5: Vector scaling ด้วย __mul
local Vec = {}
Vec.__index = Vec

function Vec.new(x, y, z)
    return setmetatable({x=x, y=y, z=z or 0}, Vec)
end

Vec.__mul = function(a, b)
    if type(a) == "number" then
        -- scalar * vector
        return Vec.new(a * b.x, a * b.y, a * b.z)
    elseif type(b) == "number" then
        -- vector * scalar
        return Vec.new(a.x * b, a.y * b, a.z * b)
    else
        -- dot product
        return a.x * b.x + a.y * b.y + a.z * b.z
    end
end

function Vec:__tostring()
    return string.format("Vec(%g, %g, %g)", self.x, self.y, self.z)
end

local v = Vec.new(1, 2, 3)
local scaled = v * 2
print(tostring(scaled))    -- Vec(2, 4, 6)
print(tostring(3 * v))     -- Vec(3, 6, 9)

local v2 = Vec.new(4, 5, 6)
local dot = v * v2
print(dot)  -- 32 (1*4 + 2*5 + 3*6)
```

### __div - การหาร

```lua
-- ตัวอย่างที่ 6: Fraction (เศษส่วน) ด้วย __div
local Fraction = {}
Fraction.__index = Fraction

local function gcd(a, b)
    a, b = math.abs(a), math.abs(b)
    while b ~= 0 do
        a, b = b, a % b
    end
    return a
end

function Fraction.new(num, den)
    assert(den ~= 0, "Denominator cannot be zero")
    local g = gcd(math.abs(num), math.abs(den))
    if den < 0 then num, den = -num, -den end
    return setmetatable({num = num/g, den = den/g}, Fraction)
end

function Fraction:__div(other)
    if type(other) == "number" then
        return Fraction.new(self.num, self.den * other)
    end
    return Fraction.new(self.num * other.den, self.den * other.num)
end

function Fraction:__tostring()
    if self.den == 1 then return tostring(self.num) end
    return self.num .. "/" .. self.den
end

local f1 = Fraction.new(3, 4)
local f2 = Fraction.new(2, 5)
local result = f1 / f2

print(tostring(result))   -- 15/8
print(tostring(f1 / 2))   -- 3/8
```

### __mod - การหารเอาเศษ

```lua
-- ตัวอย่างที่ 7: __mod สำหรับ time/clock
local Time = {}
Time.__index = Time

function Time.new(hours, minutes, seconds)
    local total = (hours * 3600) + (minutes * 60) + (seconds or 0)
    return setmetatable({total = total}, Time)
end

function Time:__mod(period)
    -- Return time within a period (e.g., mod 12 hours)
    local periodSecs = type(period) == "table" and period.total or period * 3600
    return Time.new(0, 0, self.total % periodSecs)
end

function Time:__tostring()
    local h = math.floor(self.total / 3600)
    local m = math.floor((self.total % 3600) / 60)
    local s = self.total % 60
    return string.format("%02d:%02d:%02d", h, m, s)
end

local t = Time.new(14, 30, 0)  -- 14:30:00
local t12 = t % 12             -- mod 12 hours
print(tostring(t))    -- 14:30:00
print(tostring(t12))  -- 02:30:00
```

### __pow - การยกกำลัง

```lua
-- ตัวอย่างที่ 8: Complex number ด้วย __pow
local Complex = {}
Complex.__index = Complex

function Complex.new(r, i)
    return setmetatable({r = r, i = i or 0}, Complex)
end

function Complex:__pow(n)
    -- De Moivre's theorem: (r + i*j)^n
    local magnitude = math.sqrt(self.r^2 + self.i^2)
    local angle = math.atan(self.i, self.r)
    local newMag = magnitude^n
    local newAngle = angle * n
    return Complex.new(
        newMag * math.cos(newAngle),
        newMag * math.sin(newAngle)
    )
end

function Complex:__tostring()
    if self.i >= 0 then
        return string.format("%.4f+%.4fi", self.r, self.i)
    else
        return string.format("%.4f%.4fi", self.r, self.i)
    end
end

local c = Complex.new(1, 1)  -- 1+1i
local c3 = c^3               -- (1+1i)^3 = -2+2i
print(tostring(c3))           -- -2.0000+2.0000i
```

### __unm - การนิเสธ (unary minus)

```lua
-- ตัวอย่างที่ 9: __unm สำหรับ Vector
local Vec3 = {}
Vec3.__index = Vec3

function Vec3.new(x, y, z)
    return setmetatable({x=x, y=y, z=z}, Vec3)
end

function Vec3:__unm()
    return Vec3.new(-self.x, -self.y, -self.z)
end

function Vec3:__tostring()
    return string.format("(%g, %g, %g)", self.x, self.y, self.z)
end

local v = Vec3.new(1, -2, 3)
local neg = -v
print(tostring(neg))  -- (-1, 2, -3)
```

### __idiv - การหารจำนวนเต็ม (Lua 5.3+)

```lua
-- ตัวอย่างที่ 10: __idiv สำหรับ Measurement
local Measurement = {}
Measurement.__index = Measurement

function Measurement.new(value, unit)
    return setmetatable({value = value, unit = unit}, Measurement)
end

function Measurement:__idiv(divisor)
    local divVal = type(divisor) == "table" and divisor.value or divisor
    return Measurement.new(math.floor(self.value / divVal), self.unit)
end

function Measurement:__tostring()
    return self.value .. " " .. self.unit
end

local dist = Measurement.new(100, "cm")
local parts = dist // 3
print(tostring(parts))  -- 33 cm
```

---

## 18.2 Bitwise Metamethods (Lua 5.3+)

### __band, __bor, __bxor - Bitwise AND, OR, XOR

```lua
-- ตัวอย่างที่ 11: Flags/Permission system ด้วย bitwise metamethods
local Flags = {}
Flags.__index = Flags

function Flags.new(value)
    return setmetatable({value = value or 0}, Flags)
end

-- Bitwise AND: ตรวจสอบว่ามี flag ครบหรือไม่
function Flags:__band(other)
    local oval = type(other) == "table" and other.value or other
    return Flags.new(self.value & oval)
end

-- Bitwise OR: รวม flags
function Flags:__bor(other)
    local oval = type(other) == "table" and other.value or other
    return Flags.new(self.value | oval)
end

-- Bitwise XOR: toggle flags
function Flags:__bxor(other)
    local oval = type(other) == "table" and other.value or other
    return Flags.new(self.value ~ oval)
end

function Flags:__tostring()
    return string.format("Flags(0x%X)", self.value)
end

function Flags:has(flag)
    local fval = type(flag) == "table" and flag.value or flag
    return (self.value & fval) == fval
end

-- กำหนด permission flags
local READ    = Flags.new(0x1)
local WRITE   = Flags.new(0x2)
local EXECUTE = Flags.new(0x4)

local perms = READ | WRITE
print(tostring(perms))         -- Flags(0x3)
print(perms:has(READ))         -- true
print(perms:has(EXECUTE))      -- false

local withExec = perms | EXECUTE
print(tostring(withExec))      -- Flags(0x7)

local toggled = withExec ~ WRITE  -- remove WRITE
print(toggled:has(WRITE))      -- false
```

### __bnot - Bitwise NOT

```lua
-- ตัวอย่างที่ 12: __bnot สำหรับ BitMask
local BitMask = {}
BitMask.__index = BitMask

function BitMask.new(value, bits)
    return setmetatable({
        value = value,
        bits = bits or 8  -- default 8-bit
    }, BitMask)
end

function BitMask:__bnot()
    local mask = (1 << self.bits) - 1
    return BitMask.new((~self.value) & mask, self.bits)
end

function BitMask:__tostring()
    local fmt = string.format("%%0%db", self.bits)
    return string.format(fmt, self.value)
end

local m = BitMask.new(0b10101010, 8)
local inverted = ~m
print(tostring(m))        -- 10101010
print(tostring(inverted)) -- 01010101
```

### __shl, __shr - Bit Shift

```lua
-- ตัวอย่างที่ 13: __shl และ __shr สำหรับ Register
local Register = {}
Register.__index = Register

function Register.new(value)
    return setmetatable({value = value & 0xFFFF}, Register)  -- 16-bit
end

function Register:__shl(n)
    local shift = type(n) == "table" and n.value or n
    return Register.new((self.value << shift) & 0xFFFF)
end

function Register:__shr(n)
    local shift = type(n) == "table" and n.value or n
    return Register.new(self.value >> shift)
end

function Register:__tostring()
    return string.format("Reg(0x%04X = %d)", self.value, self.value)
end

local reg = Register.new(0x00FF)
print(tostring(reg))          -- Reg(0x00FF = 255)
print(tostring(reg << 4))     -- Reg(0x0FF0 = 4080)
print(tostring(reg >> 2))     -- Reg(0x003F = 63)
```

---

## 18.3 Comparison Metamethods

### __eq - ความเท่ากัน

```lua
-- ตัวอย่างที่ 14: __eq สำหรับ Point
local Point = {}
Point.__index = Point

function Point.new(x, y)
    return setmetatable({x = x, y = y}, Point)
end

function Point:__eq(other)
    return self.x == other.x and self.y == other.y
end

function Point:__tostring()
    return string.format("Point(%g, %g)", self.x, self.y)
end

local p1 = Point.new(1, 2)
local p2 = Point.new(1, 2)
local p3 = Point.new(3, 4)

print(p1 == p2)  -- true
print(p1 == p3)  -- false
print(p1 ~= p3)  -- true
```

```lua
-- ตัวอย่างที่ 15: __eq สำหรับ Fraction
local Frac = {}
Frac.__index = Frac

function Frac.new(n, d)
    local g = math.abs(n)
    local temp = math.abs(d)
    while temp ~= 0 do g, temp = temp, g % temp end
    return setmetatable({n = n/g, d = d/g}, Frac)
end

function Frac:__eq(other)
    return self.n == other.n and self.d == other.d
end

function Frac:__tostring()
    return self.n .. "/" .. self.d
end

local f1 = Frac.new(1, 2)
local f2 = Frac.new(2, 4)   -- same as 1/2 after reduction
local f3 = Frac.new(3, 4)

print(f1 == f2)  -- true (both reduce to 1/2)
print(f1 == f3)  -- false
```

### __lt - น้อยกว่า

```lua
-- ตัวอย่างที่ 16: __lt สำหรับ Temperature
local Temp = {}
Temp.__index = Temp

function Temp.new(value, unit)
    return setmetatable({value = value, unit = unit or "C"}, Temp)
end

function Temp:toCelsius()
    if self.unit == "C" then return self.value
    elseif self.unit == "F" then return (self.value - 32) * 5/9
    elseif self.unit == "K" then return self.value - 273.15
    end
end

function Temp:__lt(other)
    return self:toCelsius() < other:toCelsius()
end

function Temp:__le(other)
    return self:toCelsius() <= other:toCelsius()
end

function Temp:__tostring()
    return string.format("%.1f°%s", self.value, self.unit)
end

local t1 = Temp.new(100, "C")     -- 100°C
local t2 = Temp.new(212, "F")     -- 212°F = 100°C
local t3 = Temp.new(373.15, "K")  -- 373.15K = 100°C
local t4 = Temp.new(50, "C")      -- 50°C

print(t4 < t1)   -- true
print(t1 < t4)   -- false
print(t1 <= t2)  -- true (equal)

-- การ sort
local temps = {t1, t4, t3, Temp.new(0, "C"), Temp.new(37, "C")}
table.sort(temps, function(a, b) return a < b end)
for _, t in ipairs(temps) do
    print(string.format("  %s = %.1f°C", tostring(t), t:toCelsius()))
end
```

### __le - น้อยกว่าหรือเท่ากับ

```lua
-- ตัวอย่างที่ 17: Ordered Set ด้วย __le
local OrderedSet = {}
OrderedSet.__index = OrderedSet

function OrderedSet.new(list)
    local s = setmetatable({_data = {}}, OrderedSet)
    for _, v in ipairs(list or {}) do
        s._data[v] = true
    end
    return s
end

-- A <= B means A is subset of B
function OrderedSet:__le(other)
    for k in pairs(self._data) do
        if not other._data[k] then return false end
    end
    return true
end

-- A < B means A is proper subset of B
function OrderedSet:__lt(other)
    return self <= other and not (other <= self)
end

local A = OrderedSet.new({1, 2})
local B = OrderedSet.new({1, 2, 3})
local C = OrderedSet.new({1, 2})

print(A <= B)  -- true (A is subset of B)
print(A < B)   -- true (A is proper subset)
print(A <= C)  -- true (equal sets)
print(A < C)   -- false (not proper subset)
```

---

## 18.4 String Metamethods

### __concat - การต่อสตริง

```lua
-- ตัวอย่างที่ 18: StringBuilder ด้วย __concat
local StringBuilder = {}
StringBuilder.__index = StringBuilder

function StringBuilder.new(str)
    return setmetatable({parts = {str or ""}}, StringBuilder)
end

function StringBuilder:__concat(other)
    local result = StringBuilder.new()
    -- Copy current parts
    for _, p in ipairs(self.parts) do
        result.parts[#result.parts + 1] = p
    end
    -- Add other
    if type(other) == "table" then
        for _, p in ipairs(other.parts) do
            result.parts[#result.parts + 1] = p
        end
    else
        result.parts[#result.parts + 1] = tostring(other)
    end
    return result
end

function StringBuilder:__tostring()
    return table.concat(self.parts)
end

local sb = StringBuilder.new("Hello")
local result = sb .. ", " .. "World" .. "!"
print(tostring(result))  -- Hello, World!
```

```lua
-- ตัวอย่างที่ 19: Path object ด้วย __concat
local Path = {}
Path.__index = Path

function Path.new(str)
    return setmetatable({path = str or ""}, Path)
end

function Path:__concat(other)
    local otherStr = type(other) == "table" and other.path or tostring(other)
    local separator = "/"
    -- ป้องกัน double slash
    if self.path:sub(-1) == separator then
        return Path.new(self.path .. otherStr)
    else
        return Path.new(self.path .. separator .. otherStr)
    end
end

function Path:__tostring()
    return self.path
end

local base = Path.new("/home/user")
local full = base .. "documents" .. "file.txt"
print(tostring(full))  -- /home/user/documents/file.txt
```

### __len - ความยาว

```lua
-- ตัวอย่างที่ 20: __len สำหรับ custom collection
local Queue = {}
Queue.__index = Queue

function Queue.new()
    return setmetatable({_data = {}, _head = 1, _tail = 0}, Queue)
end

function Queue:push(val)
    self._tail = self._tail + 1
    self._data[self._tail] = val
end

function Queue:pop()
    if self._head > self._tail then return nil end
    local val = self._data[self._head]
    self._data[self._head] = nil
    self._head = self._head + 1
    return val
end

function Queue:__len()
    return self._tail - self._head + 1
end

local q = Queue.new()
q:push("a")
q:push("b")
q:push("c")
print(#q)   -- 3
q:pop()
print(#q)   -- 2
```

```lua
-- ตัวอย่างที่ 21: __len สำหรับ sparse array
local SparseArray = {}
SparseArray.__index = SparseArray

function SparseArray.new()
    return setmetatable({_data = {}, _count = 0}, SparseArray)
end

function SparseArray:set(index, value)
    if value == nil then
        if self._data[index] ~= nil then
            self._count = self._count - 1
        end
    else
        if self._data[index] == nil then
            self._count = self._count + 1
        end
    end
    self._data[index] = value
end

function SparseArray:__len()
    return self._count  -- จำนวน element จริง ไม่ใช่ max index
end

local sa = SparseArray.new()
sa:set(1, "a")
sa:set(100, "b")
sa:set(1000, "c")
print(#sa)  -- 3 (มี 3 elements แม้ index จะห่างกัน)
```

---

## 18.5 __call - การเรียกใช้เหมือน function

```lua
-- ตัวอย่างที่ 22: __call สำหรับ function object
local Memoize = {}
Memoize.__index = Memoize

function Memoize.new(fn)
    return setmetatable({_fn = fn, _cache = {}}, Memoize)
end

function Memoize:__call(...)
    local key = table.concat({...}, ",")
    if self._cache[key] == nil then
        self._cache[key] = self._fn(...)
    end
    return self._cache[key]
end

-- Fibonacci ที่ช้ามาก
local function slowFib(n)
    if n <= 1 then return n end
    return slowFib(n-1) + slowFib(n-2)
end

local fastFib = Memoize.new(function(n)
    if n <= 1 then return n end
    -- ต้องอ้างอิง fastFib เอง (แต่ตัวนี้จะใช้ cache)
    return n  -- simplified version for demo
end)

print(fastFib(10))  -- 10
print(fastFib(10))  -- 10 (from cache)
```

```lua
-- ตัวอย่างที่ 23: __call สำหรับ Validator
local Validator = {}
Validator.__index = Validator

function Validator.new(rules)
    return setmetatable({rules = rules}, Validator)
end

function Validator:__call(value)
    for _, rule in ipairs(self.rules) do
        local ok, err = rule(value)
        if not ok then
            return false, err
        end
    end
    return true, nil
end

local isEmail = Validator.new({
    function(v)
        if type(v) ~= "string" then
            return false, "Must be a string"
        end
        return true
    end,
    function(v)
        if not v:match("^[%w%.]+@[%w%.]+%.[%a]+$") then
            return false, "Invalid email format"
        end
        return true
    end,
    function(v)
        if #v > 100 then
            return false, "Email too long"
        end
        return true
    end,
})

local ok, err = isEmail("user@example.com")
print(ok, err)  -- true  nil

ok, err = isEmail("not-an-email")
print(ok, err)  -- false  Invalid email format
```

```lua
-- ตัวอย่างที่ 24: __call สำหรับ Currying
local Curry = {}
Curry.__index = Curry

function Curry.new(fn, arity, args)
    args = args or {}
    local c = setmetatable({
        _fn = fn,
        _arity = arity,
        _args = args
    }, Curry)
    return c
end

function Curry:__call(...)
    local newArgs = {}
    for _, v in ipairs(self._args) do
        newArgs[#newArgs + 1] = v
    end
    for _, v in ipairs({...}) do
        newArgs[#newArgs + 1] = v
    end
    
    if #newArgs >= self._arity then
        return self._fn(table.unpack(newArgs))
    else
        return Curry.new(self._fn, self._arity, newArgs)
    end
end

local add = Curry.new(function(a, b, c) return a + b + c end, 3)

local add1 = add(1)        -- partially applied
local add1_2 = add1(2)     -- partially applied
local result = add1_2(3)   -- fully applied

print(result)          -- 6
print(add(1)(2)(3))   -- 6
print(add(1, 2)(3))   -- 6
```

---

## 18.6 __index และ __newindex

### __index - การเข้าถึง field ที่ไม่มี

```lua
-- ตัวอย่างที่ 25: __index เป็น function
local defaults = {
    color = "white",
    size = 1,
    visible = true
}

local obj = setmetatable({name = "MyObject"}, {
    __index = function(t, k)
        print(string.format("Accessing missing key: %s", k))
        return defaults[k]
    end
})

print(obj.name)     -- MyObject (ไม่ผ่าน __index เพราะมีอยู่แล้ว)
print(obj.color)    -- Accessing missing key: color \n white
print(obj.size)     -- Accessing missing key: size \n 1
```

```lua
-- ตัวอย่างที่ 26: __index chain สำหรับ prototype inheritance
local Animal = {}
Animal.__index = Animal

function Animal.new(name, sound)
    return setmetatable({name = name, sound = sound}, Animal)
end

function Animal:speak()
    return self.name .. " says " .. self.sound
end

function Animal:describe()
    return "I am " .. self.name
end

-- Dog inherits from Animal
local Dog = setmetatable({}, {__index = Animal})
Dog.__index = Dog

function Dog.new(name)
    return setmetatable(Animal.new(name, "Woof"), Dog)
end

function Dog:fetch(item)
    return self.name .. " fetches the " .. item
end

local rex = Dog.new("Rex")
print(rex:speak())          -- Rex says Woof
print(rex:describe())       -- I am Rex
print(rex:fetch("ball"))    -- Rex fetches the ball
```

```lua
-- ตัวอย่างที่ 27: __index สำหรับ lazy properties
local LazyObject = {}
LazyObject.__index = function(t, k)
    -- คำนวณค่าเมื่อถูกเข้าถึงครั้งแรก
    local computeFns = {
        expensive1 = function()
            print("Computing expensive1...")
            return math.pi * 2
        end,
        expensive2 = function()
            print("Computing expensive2...")
            local sum = 0
            for i = 1, 1000 do sum = sum + i end
            return sum
        end
    }
    
    if computeFns[k] then
        local val = computeFns[k]()
        rawset(t, k, val)  -- cache ค่าที่คำนวณแล้ว
        return val
    end
    return nil
end

local lazy = setmetatable({}, LazyObject)

print(lazy.expensive1)  -- Computing expensive1... \n 6.2831853...
print(lazy.expensive1)  -- 6.2831853... (ไม่คำนวณซ้ำ)
print(lazy.expensive2)  -- Computing expensive2... \n 500500
```

### __newindex - การกำหนดค่า field ใหม่

```lua
-- ตัวอย่างที่ 28: __newindex สำหรับ readonly
local function readOnly(t)
    return setmetatable({}, {
        __index = t,
        __newindex = function(_, k, v)
            error(string.format("Attempt to set readonly field '%s'", k), 2)
        end
    })
end

local config = readOnly({
    host = "localhost",
    port = 8080,
    debug = false
})

print(config.host)   -- localhost
print(config.port)   -- 8080

local ok, err = pcall(function()
    config.host = "newhost"  -- จะ error
end)
print(ok, err)  -- false  Attempt to set readonly field 'host'
```

```lua
-- ตัวอย่างที่ 29: __newindex สำหรับ validation
local ValidatedTable = {}

function ValidatedTable.new(schema)
    local data = {}
    return setmetatable(data, {
        __newindex = function(t, k, v)
            local rule = schema[k]
            if rule then
                local ok, err = rule(v)
                if not ok then
                    error(string.format("Validation failed for '%s': %s", k, err), 2)
                end
            end
            rawset(t, k, v)
        end,
        __index = data
    })
end

local person = ValidatedTable.new({
    age = function(v)
        if type(v) ~= "number" then return false, "must be number" end
        if v < 0 or v > 150 then return false, "must be 0-150" end
        return true
    end,
    name = function(v)
        if type(v) ~= "string" then return false, "must be string" end
        if #v < 1 then return false, "cannot be empty" end
        return true
    end
})

person.name = "Alice"
person.age = 30
print(person.name, person.age)  -- Alice  30

local ok, err = pcall(function()
    person.age = -5  -- invalid age
end)
print(ok, err)  -- false  Validation failed for 'age': must be 0-150
```

```lua
-- ตัวอย่างที่ 30: __newindex สำหรับ change tracking
local function trackChanges(obj)
    local changes = {}
    local proxy = setmetatable({}, {
        __index = obj,
        __newindex = function(t, k, v)
            local old = obj[k]
            if old ~= v then
                changes[#changes + 1] = {
                    field = k,
                    old = old,
                    new = v,
                    time = os.time()
                }
            end
            obj[k] = v
        end
    })
    return proxy, changes
end

local data = {name = "Alice", score = 100}
local proxy, log = trackChanges(data)

proxy.name = "Bob"
proxy.score = 150
proxy.name = "Charlie"

for i, change in ipairs(log) do
    print(string.format("[%d] %s: %s -> %s",
        i, change.field,
        tostring(change.old),
        tostring(change.new)))
end
-- [1] name: Alice -> Bob
-- [2] score: 100 -> 150
-- [3] name: Bob -> Charlie
```

---

## 18.7 __gc - Garbage Collection

```lua
-- ตัวอย่างที่ 31: __gc สำหรับ resource management
local Resource = {}
Resource.__index = Resource

-- หมายเหตุ: __gc ต้องตั้งค่าก่อน setmetatable ใน Lua 5.4
-- หรือใช้ tofinalize() ใน Lua 5.1-5.3

function Resource.new(name)
    print(string.format("[Resource] Opening: %s", name))
    local r = setmetatable({
        name = name,
        closed = false
    }, Resource)
    return r
end

function Resource:close()
    if not self.closed then
        print(string.format("[Resource] Closing: %s", self.name))
        self.closed = true
    end
end

Resource.__gc = function(self)
    if not self.closed then
        print(string.format("[Resource] GC collecting: %s", self.name))
        self:close()
    end
end

-- สร้างและใช้ resource
do
    local r = Resource.new("database_connection")
    -- ใช้ resource...
    print("Using resource:", r.name)
    -- r จะถูก GC เมื่อออกจาก scope
end

collectgarbage("collect")
print("After GC")
```

```lua
-- ตัวอย่างที่ 32: __gc สำหรับนับ instances
local Counter = {}
Counter.__index = Counter
Counter._count = 0

function Counter.new(id)
    Counter._count = Counter._count + 1
    print(string.format("Created instance #%d (total: %d)", id, Counter._count))
    return setmetatable({id = id}, Counter)
end

Counter.__gc = function(self)
    Counter._count = Counter._count - 1
    print(string.format("Destroyed instance #%d (total: %d)", self.id, Counter._count))
end

function Counter.getCount()
    return Counter._count
end

do
    local a = Counter.new(1)
    local b = Counter.new(2)
    local c = Counter.new(3)
    print("Active:", Counter.getCount())
end

collectgarbage("collect")
print("After GC, active:", Counter.getCount())
```

---

## 18.8 __tostring - การแปลงเป็น String

```lua
-- ตัวอย่างที่ 33: __tostring ที่ละเอียด
local Table2D = {}
Table2D.__index = Table2D

function Table2D.new(headers, rows)
    return setmetatable({headers = headers, rows = rows}, Table2D)
end

function Table2D:__tostring()
    local widths = {}
    -- คำนวณความกว้างแต่ละ column
    for i, h in ipairs(self.headers) do
        widths[i] = #tostring(h)
    end
    for _, row in ipairs(self.rows) do
        for i, cell in ipairs(row) do
            widths[i] = math.max(widths[i] or 0, #tostring(cell))
        end
    end
    
    local function formatRow(cells)
        local parts = {}
        for i, cell in ipairs(cells) do
            parts[i] = string.format("%-" .. widths[i] .. "s", tostring(cell))
        end
        return "| " .. table.concat(parts, " | ") .. " |"
    end
    
    local separator = "+" .. string.rep("-", 0)
    local sepParts = {}
    for _, w in ipairs(widths) do
        sepParts[#sepParts + 1] = string.rep("-", w + 2)
    end
    separator = "+" .. table.concat(sepParts, "+") .. "+"
    
    local lines = {separator, formatRow(self.headers), separator}
    for _, row in ipairs(self.rows) do
        lines[#lines + 1] = formatRow(row)
    end
    lines[#lines + 1] = separator
    
    return table.concat(lines, "\n")
end

local t = Table2D.new(
    {"Name", "Age", "Score"},
    {
        {"Alice", 25, 95},
        {"Bob", 30, 87},
        {"Charlie", 22, 92},
    }
)
print(tostring(t))
```

---

## 18.9 __name - ชื่อ Type

```lua
-- ตัวอย่างที่ 34: __name สำหรับ error messages ที่ชัดเจน
local Color = {}
Color.__index = Color
Color.__name = "Color"  -- ชื่อที่จะแสดงใน error messages

function Color.new(r, g, b)
    assert(type(r) == "number" and r >= 0 and r <= 255, "Invalid red value")
    assert(type(g) == "number" and g >= 0 and g <= 255, "Invalid green value")
    assert(type(b) == "number" and b >= 0 and b <= 255, "Invalid blue value")
    return setmetatable({r=r, g=g, b=b}, Color)
end

function Color:__tostring()
    return string.format("Color(#%02X%02X%02X)", self.r, self.g, self.b)
end

local red = Color.new(255, 0, 0)
print(tostring(red))  -- Color(#FF0000)

-- __name ช่วยให้ error message ชัดขึ้น
-- ใน Lua 5.4 tostring จะแสดง "Color: 0x..." แทน "table: 0x..."
```

---

## 18.10 __close - To-Be-Closed Variables (Lua 5.4)

```lua
-- ตัวอย่างที่ 35: __close สำหรับ automatic cleanup
local File = {}
File.__index = File

function File.new(path, mode)
    local f = io.open(path, mode or "r")
    if not f then
        error("Cannot open file: " .. path)
    end
    print("Opening file: " .. path)
    return setmetatable({_file = f, path = path}, File)
end

function File:read(fmt)
    return self._file:read(fmt)
end

function File:write(data)
    return self._file:write(data)
end

-- __close ถูกเรียกเมื่อ variable ออกจาก scope (Lua 5.4+)
function File:__close(err)
    if self._file then
        print("Auto-closing file: " .. self.path)
        self._file:close()
        self._file = nil
    end
    if err then
        print("Error occurred: " .. tostring(err))
    end
end

-- ใน Lua 5.4 ใช้ <close> attribute:
-- local f <close> = File.new("test.txt")
-- f จะถูก close อัตโนมัติเมื่อออกจาก block

-- สำหรับ demonstration:
local function withFile(path, mode, fn)
    local f = File.new(path, mode)
    local ok, err = pcall(fn, f)
    f:__close(not ok and err or nil)
    if not ok then error(err, 2) end
end

-- สร้างไฟล์ test
local tmpPath = "/tmp/test_close.txt"
local wf = io.open(tmpPath, "w")
if wf then
    wf:write("Hello World")
    wf:close()
    
    withFile(tmpPath, "r", function(f)
        local content = f:read("*a")
        print("Read:", content)
    end)
end
```

---

## 18.11 ตัวอย่างสมบูรณ์: Vector2D class

```lua
-- ตัวอย่างที่ 36: Vector2D ครบทุก metamethods
local Vector2D = {}
Vector2D.__index = Vector2D
Vector2D.__name = "Vector2D"

function Vector2D.new(x, y)
    return setmetatable({x = x or 0, y = y or 0}, Vector2D)
end

-- Arithmetic
function Vector2D.__add(a, b)
    if type(a) == "number" then return Vector2D.new(a + b.x, a + b.y) end
    if type(b) == "number" then return Vector2D.new(a.x + b, a.y + b) end
    return Vector2D.new(a.x + b.x, a.y + b.y)
end

function Vector2D.__sub(a, b)
    if type(b) == "number" then return Vector2D.new(a.x - b, a.y - b) end
    return Vector2D.new(a.x - b.x, a.y - b.y)
end

function Vector2D.__mul(a, b)
    if type(a) == "number" then return Vector2D.new(a * b.x, a * b.y) end
    if type(b) == "number" then return Vector2D.new(a.x * b, a.y * b) end
    return a.x * b.x + a.y * b.y  -- dot product
end

function Vector2D.__div(a, b)
    local divisor = type(b) == "table" and (b.x ~= 0 and b.x or 1) or b
    return Vector2D.new(a.x / (type(b)=="number" and b or b.x),
                        a.y / (type(b)=="number" and b or b.y))
end

function Vector2D.__unm(a)
    return Vector2D.new(-a.x, -a.y)
end

function Vector2D.__mod(a, b)
    local m = type(b) == "number" and b or math.sqrt(b.x^2 + b.y^2)
    return Vector2D.new(a.x % m, a.y % m)
end

-- Comparison
function Vector2D.__eq(a, b)
    return a.x == b.x and a.y == b.y
end

function Vector2D.__lt(a, b)
    return (a.x^2 + a.y^2) < (b.x^2 + b.y^2)  -- compare magnitudes
end

function Vector2D.__le(a, b)
    return (a.x^2 + a.y^2) <= (b.x^2 + b.y^2)
end

-- String
function Vector2D.__concat(a, b)
    return tostring(a) .. tostring(b)
end

function Vector2D.__len(a)
    return math.sqrt(a.x^2 + a.y^2)
end

function Vector2D:__tostring()
    return string.format("Vector2D(%g, %g)", self.x, self.y)
end

-- Methods
function Vector2D:magnitude()
    return math.sqrt(self.x^2 + self.y^2)
end

function Vector2D:normalize()
    local mag = self:magnitude()
    if mag == 0 then return Vector2D.new(0, 0) end
    return Vector2D.new(self.x/mag, self.y/mag)
end

function Vector2D:dot(other)
    return self.x * other.x + self.y * other.y
end

function Vector2D:angle()
    return math.atan(self.y, self.x)
end

-- Test
local v1 = Vector2D.new(3, 4)
local v2 = Vector2D.new(1, 2)

print(tostring(v1))              -- Vector2D(3, 4)
print(tostring(v1 + v2))         -- Vector2D(4, 6)
print(tostring(v1 - v2))         -- Vector2D(2, 2)
print(tostring(v1 * 2))          -- Vector2D(6, 8)
print(tostring(2 * v1))          -- Vector2D(6, 8)
print(v1 * v2)                   -- 11 (dot product)
print(tostring(-v1))             -- Vector2D(-3, -4)
print(v1 == Vector2D.new(3, 4))  -- true
print(v2 < v1)                   -- true (|v2| < |v1|)
print(v1:magnitude())            -- 5
print(tostring(v1:normalize()))  -- Vector2D(0.6, 0.8)
```

---

## 18.12 ตัวอย่างสมบูรณ์: Set class

```lua
-- ตัวอย่างที่ 37: Set class ครบทุก operations
local Set = {}
Set.__index = Set
Set.__name = "Set"

function Set.new(items)
    local s = setmetatable({_data = {}, _size = 0}, Set)
    if items then
        for _, v in ipairs(items) do
            s:add(v)
        end
    end
    return s
end

function Set:add(item)
    if not self._data[item] then
        self._data[item] = true
        self._size = self._size + 1
    end
    return self
end

function Set:remove(item)
    if self._data[item] then
        self._data[item] = nil
        self._size = self._size - 1
    end
    return self
end

function Set:contains(item)
    return self._data[item] == true
end

-- Union: A + B หรือ A | B
function Set.__add(a, b)
    local result = Set.new()
    for k in pairs(a._data) do result:add(k) end
    for k in pairs(b._data) do result:add(k) end
    return result
end

Set.__bor = Set.__add  -- สำหรับ A | B

-- Intersection: A * B หรือ A & B
function Set.__mul(a, b)
    local result = Set.new()
    for k in pairs(a._data) do
        if b._data[k] then result:add(k) end
    end
    return result
end

Set.__band = Set.__mul  -- สำหรับ A & B

-- Difference: A - B
function Set.__sub(a, b)
    local result = Set.new()
    for k in pairs(a._data) do
        if not b._data[k] then result:add(k) end
    end
    return result
end

-- Symmetric difference: A ~ B
function Set.__bxor(a, b)
    return (a + b) - (a * b)
end

-- Subset: A <= B
function Set.__le(a, b)
    for k in pairs(a._data) do
        if not b._data[k] then return false end
    end
    return true
end

-- Proper subset: A < B
function Set.__lt(a, b)
    return a <= b and not (b <= a)
end

-- Equality
function Set.__eq(a, b)
    return a <= b and b <= a
end

-- Length
function Set:__len()
    return self._size
end

-- String representation
function Set:__tostring()
    local items = {}
    for k in pairs(self._data) do
        items[#items + 1] = tostring(k)
    end
    table.sort(items)
    return "{" .. table.concat(items, ", ") .. "}"
end

-- Iterator
function Set:iter()
    return next, self._data, nil
end

-- Test
local A = Set.new({1, 2, 3, 4, 5})
local B = Set.new({3, 4, 5, 6, 7})
local C = Set.new({1, 2})

print("A =", tostring(A))            -- {1, 2, 3, 4, 5}
print("B =", tostring(B))            -- {3, 4, 5, 6, 7}
print("A ∪ B =", tostring(A + B))    -- {1, 2, 3, 4, 5, 6, 7}
print("A ∩ B =", tostring(A * B))    -- {3, 4, 5}
print("A - B =", tostring(A - B))    -- {1, 2}
print("A △ B =", tostring(A ~ B))    -- (A XOR B)
print("|A| =", #A)                   -- 5
print("C ⊆ A:", C <= A)              -- true
print("C ⊂ A:", C < A)               -- true
print("A = A:", A == Set.new({1,2,3,4,5}))  -- true
```

---

## 18.13 ตัวอย่างสมบูรณ์: Matrix class

```lua
-- ตัวอย่างที่ 38: Matrix class พร้อม operators
local Matrix = {}
Matrix.__index = Matrix
Matrix.__name = "Matrix"

function Matrix.new(rows, cols, init)
    local m = setmetatable({rows=rows, cols=cols, data={}}, Matrix)
    local initVal = init or 0
    for i = 1, rows do
        m.data[i] = {}
        for j = 1, cols do
            m.data[i][j] = initVal
        end
    end
    return m
end

function Matrix.identity(n)
    local m = Matrix.new(n, n)
    for i = 1, n do m.data[i][i] = 1 end
    return m
end

function Matrix.fromArray(arr)
    local rows = #arr
    local cols = #arr[1]
    local m = Matrix.new(rows, cols)
    for i = 1, rows do
        for j = 1, cols do
            m.data[i][j] = arr[i][j]
        end
    end
    return m
end

function Matrix:get(i, j) return self.data[i][j] end
function Matrix:set(i, j, v) self.data[i][j] = v end

function Matrix.__add(a, b)
    assert(a.rows==b.rows and a.cols==b.cols, "Matrix size mismatch")
    local result = Matrix.new(a.rows, a.cols)
    for i = 1, a.rows do
        for j = 1, a.cols do
            result.data[i][j] = a.data[i][j] + b.data[i][j]
        end
    end
    return result
end

function Matrix.__sub(a, b)
    assert(a.rows==b.rows and a.cols==b.cols, "Matrix size mismatch")
    local result = Matrix.new(a.rows, a.cols)
    for i = 1, a.rows do
        for j = 1, a.cols do
            result.data[i][j] = a.data[i][j] - b.data[i][j]
        end
    end
    return result
end

function Matrix.__mul(a, b)
    if type(b) == "number" then
        local result = Matrix.new(a.rows, a.cols)
        for i = 1, a.rows do
            for j = 1, a.cols do
                result.data[i][j] = a.data[i][j] * b
            end
        end
        return result
    elseif type(a) == "number" then
        return b * a  -- scalar * matrix
    end
    assert(a.cols == b.rows, "Matrix dimensions incompatible")
    local result = Matrix.new(a.rows, b.cols)
    for i = 1, a.rows do
        for j = 1, b.cols do
            local sum = 0
            for k = 1, a.cols do
                sum = sum + a.data[i][k] * b.data[k][j]
            end
            result.data[i][j] = sum
        end
    end
    return result
end

function Matrix:__unm()
    return self * (-1)
end

function Matrix.__eq(a, b)
    if a.rows ~= b.rows or a.cols ~= b.cols then return false end
    for i = 1, a.rows do
        for j = 1, a.cols do
            if a.data[i][j] ~= b.data[i][j] then return false end
        end
    end
    return true
end

function Matrix:__len()
    return self.rows * self.cols
end

function Matrix:transpose()
    local result = Matrix.new(self.cols, self.rows)
    for i = 1, self.rows do
        for j = 1, self.cols do
            result.data[j][i] = self.data[i][j]
        end
    end
    return result
end

function Matrix:trace()
    assert(self.rows == self.cols, "Trace requires square matrix")
    local sum = 0
    for i = 1, self.rows do sum = sum + self.data[i][i] end
    return sum
end

function Matrix:__tostring()
    local lines = {}
    for i = 1, self.rows do
        local row = {}
        for j = 1, self.cols do
            row[j] = string.format("%8.3f", self.data[i][j])
        end
        lines[i] = "[" .. table.concat(row, " ") .. "]"
    end
    return table.concat(lines, "\n")
end

-- Test
local A = Matrix.fromArray({{1,2,3},{4,5,6}})
local B = Matrix.fromArray({{7,8},{9,10},{11,12}})
local I = Matrix.identity(3)

print("A:")
print(tostring(A))
print("\nB:")
print(tostring(B))
print("\nA * B:")
print(tostring(A * B))
print("\nI:")
print(tostring(I))
print("\nA^T:")
print(tostring(A:transpose()))
print("\n|A| =", #A, "(elements)")
```

---

## 18.14 ตัวอย่างเพิ่มเติม

```lua
-- ตัวอย่างที่ 39: Proxy ด้วย __index และ __newindex
local function createProxy(target, onChange)
    return setmetatable({}, {
        __index = target,
        __newindex = function(t, k, v)
            local old = target[k]
            target[k] = v
            if onChange then
                onChange(k, old, v)
            end
        end,
        __len = function() return #target end,
        __pairs = function() return pairs(target) end,
    })
end

local state = {x = 0, y = 0, name = "origin"}
local proxy = createProxy(state, function(k, old, new)
    print(string.format("Changed %s: %s -> %s", k, tostring(old), tostring(new)))
end)

proxy.x = 10     -- Changed x: 0 -> 10
proxy.y = 20     -- Changed y: 0 -> 20
proxy.name = "point"  -- Changed name: origin -> point

print(proxy.x, proxy.y)  -- 10  20
```

```lua
-- ตัวอย่างที่ 40: Observable ด้วย metamethods
local Observable = {}
Observable.__index = Observable

function Observable.new(initial)
    local o = setmetatable({
        _value = initial,
        _listeners = {}
    }, Observable)
    return o
end

function Observable:subscribe(fn)
    self._listeners[#self._listeners + 1] = fn
    return function()  -- unsubscribe function
        for i, listener in ipairs(self._listeners) do
            if listener == fn then
                table.remove(self._listeners, i)
                break
            end
        end
    end
end

function Observable:__call(newVal)
    if newVal ~= nil then
        local old = self._value
        self._value = newVal
        for _, listener in ipairs(self._listeners) do
            listener(newVal, old)
        end
    end
    return self._value
end

function Observable:__tostring()
    return string.format("Observable(%s)", tostring(self._value))
end

local count = Observable.new(0)

-- Subscribe to changes
local unsub = count:subscribe(function(new, old)
    print(string.format("Count changed: %d -> %d", old, new))
end)

count(1)   -- Count changed: 0 -> 1
count(2)   -- Count changed: 1 -> 2
count(5)   -- Count changed: 2 -> 5

unsub()    -- ยกเลิก subscription
count(10)  -- ไม่มี output

print(count())  -- 10
```

```lua
-- ตัวอย่างที่ 41: JSON-like serialization ด้วย __tostring
local JsonValue = {}
JsonValue.__index = JsonValue

function JsonValue.new(val)
    return setmetatable({value = val}, JsonValue)
end

local function jsonEncode(val, indent, level)
    level = level or 0
    indent = indent or "  "
    local currentIndent = string.rep(indent, level)
    local nextIndent = string.rep(indent, level + 1)
    
    local t = type(val)
    if t == "nil" then
        return "null"
    elseif t == "boolean" then
        return val and "true" or "false"
    elseif t == "number" then
        if val ~= val then return "null" end  -- NaN
        return tostring(val)
    elseif t == "string" then
        -- Escape special characters
        val = val:gsub('\\', '\\\\')
        val = val:gsub('"', '\\"')
        val = val:gsub('\n', '\\n')
        val = val:gsub('\t', '\\t')
        return '"' .. val .. '"'
    elseif t == "table" then
        -- Check if array
        local isArray = #val > 0
        if isArray then
            local items = {}
            for i, v in ipairs(val) do
                items[i] = nextIndent .. jsonEncode(v, indent, level + 1)
            end
            return "[\n" .. table.concat(items, ",\n") .. "\n" .. currentIndent .. "]"
        else
            local items = {}
            for k, v in pairs(val) do
                items[#items + 1] = nextIndent .. '"' .. tostring(k) .. '": ' ..
                    jsonEncode(v, indent, level + 1)
            end
            table.sort(items)
            return "{\n" .. table.concat(items, ",\n") .. "\n" .. currentIndent .. "}"
        end
    end
    return '"[' .. t .. ']"'
end

function JsonValue:__tostring()
    return jsonEncode(self.value)
end

local data = JsonValue.new({
    name = "Alice",
    age = 30,
    scores = {95, 87, 92},
    address = {
        city = "Bangkok",
        country = "Thailand"
    }
})

print(tostring(data))
```

```lua
-- ตัวอย่างที่ 42: __index เป็น table (prototype chain)
-- สร้าง class hierarchy ด้วย __index chain

local Base = {}
Base.__index = Base

function Base:init(name)
    self.name = name
end

function Base:getName()
    return self.name
end

function Base:type()
    return "Base"
end

-- Middle class
local Middle = setmetatable({}, {__index = Base})
Middle.__index = Middle

function Middle:type()
    return "Middle"
end

function Middle:describe()
    return self:type() .. " named " .. self:getName()
end

-- Derived class
local Derived = setmetatable({}, {__index = Middle})
Derived.__index = Derived

function Derived:type()
    return "Derived"
end

function Derived.new(name, extra)
    local obj = setmetatable({}, Derived)
    obj:init(name)
    obj.extra = extra
    return obj
end

function Derived:fullInfo()
    return self:describe() .. " [" .. tostring(self.extra) .. "]"
end

local d = Derived.new("TestObj", 42)
print(d:type())       -- Derived
print(d:getName())    -- TestObj (from Base)
print(d:describe())   -- Derived named TestObj (from Middle)
print(d:fullInfo())   -- Derived named TestObj [42]
```

```lua
-- ตัวอย่างที่ 43: __newindex สำหรับ immutable value objects
local function immutable(data)
    local copy = {}
    for k, v in pairs(data) do copy[k] = v end
    
    return setmetatable({}, {
        __index = copy,
        __newindex = function(_, k, v)
            error("Cannot modify immutable object", 2)
        end,
        __tostring = function()
            local parts = {}
            for k, v in pairs(copy) do
                parts[#parts + 1] = string.format("%s=%s", k, tostring(v))
            end
            table.sort(parts)
            return "Immutable{" .. table.concat(parts, ", ") .. "}"
        end,
        __len = function()
            local count = 0
            for _ in pairs(copy) do count = count + 1 end
            return count
        end
    })
end

local point = immutable({x = 1, y = 2, z = 3})
print(point.x)          -- 1
print(tostring(point))  -- Immutable{x=1, y=2, z=3}
print(#point)           -- 3

local ok, err = pcall(function()
    point.x = 10  -- error
end)
print(ok, err)  -- false  Cannot modify immutable object
```

```lua
-- ตัวอย่างที่ 44: __call สำหรับ Builder Pattern
local QueryBuilder = {}
QueryBuilder.__index = QueryBuilder

function QueryBuilder.new(table_name)
    return setmetatable({
        _table = table_name,
        _select = {"*"},
        _where = {},
        _order = nil,
        _limit = nil
    }, QueryBuilder)
end

function QueryBuilder:select(...)
    self._select = {...}
    return self
end

function QueryBuilder:where(condition)
    self._where[#self._where + 1] = condition
    return self
end

function QueryBuilder:orderBy(field, dir)
    self._order = field .. " " .. (dir or "ASC")
    return self
end

function QueryBuilder:limit(n)
    self._limit = n
    return self
end

-- __call จะ build และ return query string
function QueryBuilder:__call()
    local sql = "SELECT " .. table.concat(self._select, ", ")
    sql = sql .. " FROM " .. self._table
    if #self._where > 0 then
        sql = sql .. " WHERE " .. table.concat(self._where, " AND ")
    end
    if self._order then
        sql = sql .. " ORDER BY " .. self._order
    end
    if self._limit then
        sql = sql .. " LIMIT " .. self._limit
    end
    return sql
end

function QueryBuilder:__tostring()
    return self()  -- เรียก __call
end

local query = QueryBuilder.new("users")
    :select("id", "name", "email")
    :where("age > 18")
    :where("active = 1")
    :orderBy("name")
    :limit(10)

print(query())
-- SELECT id, name, email FROM users WHERE age > 18 AND active = 1 ORDER BY name ASC LIMIT 10
```

```lua
-- ตัวอย่างที่ 45: รวม metamethods ใน Event System
local EventEmitter = {}
EventEmitter.__index = EventEmitter

function EventEmitter.new()
    return setmetatable({
        _events = {},
        _count = 0
    }, EventEmitter)
end

-- __call: emit event
function EventEmitter:__call(event, ...)
    if self._events[event] then
        for _, handler in ipairs(self._events[event]) do
            handler(...)
        end
    end
end

-- __add: combine two emitters
function EventEmitter.__add(a, b)
    local combined = EventEmitter.new()
    for event, handlers in pairs(a._events) do
        combined._events[event] = combined._events[event] or {}
        for _, h in ipairs(handlers) do
            combined._events[event][#combined._events[event] + 1] = h
        end
    end
    for event, handlers in pairs(b._events) do
        combined._events[event] = combined._events[event] or {}
        for _, h in ipairs(handlers) do
            combined._events[event][#combined._events[event] + 1] = h
        end
    end
    return combined
end

-- __len: จำนวน event types
function EventEmitter:__len()
    local count = 0
    for _ in pairs(self._events) do count = count + 1 end
    return count
end

function EventEmitter:on(event, handler)
    self._events[event] = self._events[event] or {}
    self._events[event][#self._events[event] + 1] = handler
    self._count = self._count + 1
    return self
end

function EventEmitter:emit(event, ...)
    self(event, ...)  -- ใช้ __call
end

local emitter = EventEmitter.new()

emitter:on("data", function(val)
    print("Handler 1:", val)
end)

emitter:on("data", function(val)
    print("Handler 2:", val * 2)
end)

emitter:on("error", function(err)
    print("Error:", err)
end)

emitter:emit("data", 42)
-- Handler 1: 42
-- Handler 2: 84

print("Event types:", #emitter)  -- 2
```

---

## แบบฝึกหัด

### ระดับพื้นฐาน

1. **Complex Number**: สร้าง class `Complex` ที่รองรับ metamethods ต่อไปนี้:
   - `__add`, `__sub`, `__mul` สำหรับการคำนวณ complex numbers
   - `__eq` สำหรับการเปรียบเทียบ
   - `__tostring` ที่แสดงในรูป `a+bi` หรือ `a-bi`
   - `__unm` สำหรับ negation
   - method `magnitude()` และ `conjugate()`

2. **Temperature**: สร้าง class `Temperature` ที่:
   - แปลงระหว่าง Celsius, Fahrenheit, Kelvin
   - รองรับ `__add`, `__sub` (บวก/ลบ degrees)
   - รองรับ `__lt`, `__le`, `__eq` สำหรับเปรียบเทียบ
   - `__tostring` แสดงทั้ง C, F, K

3. **Stack**: สร้าง class `Stack` ที่:
   - ใช้ `__len` แสดงจำนวน elements
   - ใช้ `__tostring` แสดง contents
   - ใช้ `__call` สำหรับ push operation
   - ใช้ `__concat` รวม 2 stacks เข้าด้วยกัน

### ระดับกลาง

4. **Polynomial**: สร้าง class สำหรับ polynomial (เช่น 3x² + 2x + 1):
   - `__add`, `__sub`, `__mul` สำหรับ polynomial arithmetic
   - `__call` สำหรับ evaluate polynomial ที่ค่า x ใดๆ
   - `__tostring` แสดงรูปแบบที่อ่านง่าย
   - `__eq` เปรียบเทียบ polynomial
   - `__len` return degree ของ polynomial

5. **Interval**: สร้าง class `Interval` ที่แทน [a, b]:
   - `__add`, `__mul` สำหรับ interval arithmetic
   - `__le` สำหรับ subset checking
   - `__eq` สำหรับ equality
   - `__len` return length ของ interval
   - method `contains(x)` ตรวจสอบว่า x อยู่ใน interval หรือไม่

6. **Money**: สร้าง class `Money` ที่จัดการสกุลเงิน:
   - `__add`, `__sub` ที่ตรวจสอบ currency ต้องตรงกัน
   - `__mul` สำหรับคูณด้วย scalar
   - `__lt`, `__le`, `__eq` สำหรับเปรียบเทียบ
   - `__tostring` แสดงในรูป "USD 10.50"

### ระดับสูง

7. **Lazy Sequence**: สร้าง class ที่ใช้ lazy evaluation:
   - `__index` สร้าง element เมื่อถูกเข้าถึงครั้งแรก
   - `__len` return จำนวน elements ที่สร้างแล้ว
   - `__call` สำหรับ map/filter operations
   - เก็บ cache ของ elements ที่คำนวณแล้ว

8. **Observable Table**: สร้าง proxy ที่:
   - Track การเปลี่ยนแปลงทั้งหมดด้วย `__newindex`
   - ให้ subscribe/unsubscribe ด้วย `__add`/`__sub`
   - `__len` return จำนวน subscribers
   - สามารถ undo การเปลี่ยนแปลงได้

9. **Graph**: สร้าง class สำหรับ directed graph:
   - `__add` เพิ่ม edge
   - `__sub` ลบ edge
   - `__mul` คำนวณ composition (path of length 2)
   - `__len` return จำนวน edges
   - `__call` DFS/BFS traversal

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Metamethods ทั้งหมดใน Lua:

| Category | Metamethods |
|----------|-------------|
| Arithmetic | `__add`, `__sub`, `__mul`, `__div`, `__mod`, `__pow`, `__unm`, `__idiv` |
| Bitwise | `__band`, `__bor`, `__bxor`, `__bnot`, `__shl`, `__shr` |
| Comparison | `__eq`, `__lt`, `__le` |
| String | `__concat`, `__len` |
| Call | `__call` |
| Indexing | `__index`, `__newindex` |
| GC | `__gc` |
| Special | `__tostring`, `__name`, `__close` |

Metamethods เป็นพื้นฐานสำคัญของการทำ OOP และ DSL (Domain-Specific Language) ใน Lua การเข้าใจ metamethods อย่างลึกซึ้งจะช่วยให้เขียนโค้ดที่สวยงามและมีประสิทธิภาพมากขึ้น
