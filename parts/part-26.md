# บทที่ 26: Functional Programming ใน Lua

## บทนำ

Functional Programming (FP) เป็นกระบวนทัศน์การเขียนโปรแกรมที่เน้นการใช้ฟังก์ชันเป็นหน่วยหลักของการคำนวณ Lua รองรับ FP ได้ดีเยี่ยมเพราะฟังก์ชันเป็น first-class values และมีความยืดหยุ่นสูง

---

## 26.1 First-Class Functions

ใน Lua ฟังก์ชันเป็น first-class citizens หมายความว่าสามารถเก็บในตัวแปร ส่งเป็น argument และ return จากฟังก์ชันได้

```lua
-- ฟังก์ชันเก็บในตัวแปร
local greet = function(name)
    return "สวัสดี, " .. name
end
print(greet("โลก"))  -- สวัสดี, โลก

-- ฟังก์ชันเป็น argument
local function apply(f, x)
    return f(x)
end

local double = function(n) return n * 2 end
print(apply(double, 5))  -- 10

-- ฟังก์ชัน return ฟังก์ชัน
local function multiplier(factor)
    return function(n)
        return n * factor
    end
end

local triple = multiplier(3)
print(triple(7))  -- 21
print(triple(10)) -- 30
```

```lua
-- เก็บฟังก์ชันในตาราง
local ops = {
    add = function(a, b) return a + b end,
    sub = function(a, b) return a - b end,
    mul = function(a, b) return a * b end,
    div = function(a, b) return a / b end,
}

print(ops.add(10, 5))  -- 15
print(ops.sub(10, 5))  -- 5
print(ops.mul(10, 5))  -- 50
print(ops.div(10, 5))  -- 2.0

-- วนซ้ำใช้งานทุก operation
for name, op in pairs(ops) do
    print(name .. "(10, 5) = " .. op(10, 5))
end
```

```lua
-- Higher-order function
local function compose(f, g)
    return function(x)
        return f(g(x))
    end
end

local addOne = function(x) return x + 1 end
local square = function(x) return x * x end

local squareThenAdd = compose(addOne, square)
local addThenSquare = compose(square, addOne)

print(squareThenAdd(4))  -- 17 (4^2 + 1)
print(addThenSquare(4))  -- 25 ((4+1)^2)
```

---

## 26.2 Pure Functions

Pure function คือฟังก์ชันที่:
1. ให้ผลลัพธ์เดียวกันเสมอเมื่อรับ input เดียวกัน
2. ไม่มี side effects

```lua
-- Pure function ตัวอย่าง
local function pureAdd(a, b)
    return a + b
end

-- เรียกกี่ครั้งก็ได้ผลเหมือนกัน
print(pureAdd(3, 4))  -- 7
print(pureAdd(3, 4))  -- 7
print(pureAdd(3, 4))  -- 7

-- Impure function (ไม่ควรใช้ใน FP)
local count = 0
local function impureIncrement()
    count = count + 1  -- side effect!
    return count
end
print(impureIncrement())  -- 1
print(impureIncrement())  -- 2 (ผลต่างกัน!)
```

```lua
-- Pure function กับ table
local function pureMap(t, f)
    local result = {}
    for i, v in ipairs(t) do
        result[i] = f(v)
    end
    return result  -- ไม่แก้ไข t ต้นฉบับ
end

local numbers = {1, 2, 3, 4, 5}
local doubled = pureMap(numbers, function(x) return x * 2 end)

print(table.concat(numbers, ", "))  -- 1, 2, 3, 4, 5 (ไม่เปลี่ยน)
print(table.concat(doubled, ", "))  -- 2, 4, 6, 8, 10
```

```lua
-- Pure function สำหรับ string
local function pureUpperCase(s)
    return s:upper()
end

local function pureConcat(a, b)
    return a .. b
end

local function pureReverse(s)
    return s:reverse()
end

print(pureUpperCase("hello"))          -- HELLO
print(pureConcat("foo", "bar"))        -- foobar
print(pureReverse("lua"))              -- aul
```

---

## 26.3 Immutability Patterns

Lua ไม่มี built-in immutability แต่เราสามารถจำลองได้ด้วย metatable

```lua
-- Immutable table ด้วย metatable
local function freeze(t)
    local frozen = {}
    -- copy ค่าทั้งหมด
    for k, v in pairs(t) do
        if type(v) == "table" then
            frozen[k] = freeze(v)  -- recursive freeze
        else
            frozen[k] = v
        end
    end
    setmetatable(frozen, {
        __newindex = function(_, k, _)
            error("Cannot modify frozen table (key: " .. tostring(k) .. ")", 2)
        end,
        __index = frozen,
    })
    return frozen
end

local config = freeze({
    host = "localhost",
    port = 8080,
    debug = false,
})

print(config.host)  -- localhost
print(config.port)  -- 8080

-- พยายามแก้ไขจะ error
local ok, err = pcall(function()
    config.host = "example.com"
end)
print(ok)   -- false
print(err)  -- Cannot modify frozen table
```

```lua
-- Immutable update - สร้าง copy ใหม่แทนการแก้ไข
local function update(t, changes)
    local new_t = {}
    for k, v in pairs(t) do
        new_t[k] = v
    end
    for k, v in pairs(changes) do
        new_t[k] = v
    end
    return new_t
end

local user = { name = "สมชาย", age = 30, city = "กรุงเทพ" }
local updatedUser = update(user, { age = 31, city = "เชียงใหม่" })

print(user.name, user.age, user.city)        -- สมชาย 30 กรุงเทพ (ไม่เปลี่ยน)
print(updatedUser.name, updatedUser.age, updatedUser.city)  -- สมชาย 31 เชียงใหม่
```

```lua
-- Persistent data structure แบบง่าย
local function cons(head, tail)
    return { head = head, tail = tail }
end

local function car(list) return list.head end
local function cdr(list) return list.tail end

-- สร้าง immutable list
local myList = cons(1, cons(2, cons(3, nil)))

-- อ่านค่า
print(car(myList))           -- 1
print(car(cdr(myList)))      -- 2
print(car(cdr(cdr(myList)))) -- 3

-- เพิ่มต้น list โดยไม่แก้ไขของเดิม
local extended = cons(0, myList)
print(car(extended))         -- 0
print(car(cdr(extended)))    -- 1 (ยังเหมือนเดิม)
```

---

## 26.4 map(), filter(), reduce()

```lua
-- map: แปลงแต่ละ element ด้วยฟังก์ชัน
local function map(t, f)
    local result = {}
    for i, v in ipairs(t) do
        result[i] = f(v)
    end
    return result
end

local nums = {1, 2, 3, 4, 5}
local squares = map(nums, function(x) return x * x end)
local strings = map(nums, function(x) return "item_" .. x end)

print(table.concat(squares, ", "))  -- 1, 4, 9, 16, 25
print(table.concat(strings, ", "))  -- item_1, item_2, item_3, item_4, item_5
```

```lua
-- filter: กรองเฉพาะ element ที่ผ่านเงื่อนไข
local function filter(t, predicate)
    local result = {}
    for _, v in ipairs(t) do
        if predicate(v) then
            result[#result + 1] = v
        end
    end
    return result
end

local numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
local evens = filter(numbers, function(x) return x % 2 == 0 end)
local odds = filter(numbers, function(x) return x % 2 ~= 0 end)
local bigNums = filter(numbers, function(x) return x > 5 end)

print(table.concat(evens, ", "))    -- 2, 4, 6, 8, 10
print(table.concat(odds, ", "))     -- 1, 3, 5, 7, 9
print(table.concat(bigNums, ", "))  -- 6, 7, 8, 9, 10
```

```lua
-- reduce: รวมค่าทั้งหมดเป็นค่าเดียว
local function reduce(t, f, initial)
    local acc = initial
    for _, v in ipairs(t) do
        acc = f(acc, v)
    end
    return acc
end

local nums = {1, 2, 3, 4, 5}

-- หาผลรวม
local sum = reduce(nums, function(acc, x) return acc + x end, 0)
print(sum)  -- 15

-- หาผลคูณ
local product = reduce(nums, function(acc, x) return acc * x end, 1)
print(product)  -- 120

-- หาค่าสูงสุด
local maxVal = reduce(nums, function(acc, x) return math.max(acc, x) end, -math.huge)
print(maxVal)  -- 5

-- สร้าง string
local concat = reduce(nums, function(acc, x) return acc .. tostring(x) end, "")
print(concat)  -- 12345
```

```lua
-- รวม map, filter, reduce เข้าด้วยกัน
local data = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5}

local result = reduce(
    map(
        filter(data, function(x) return x > 3 end),
        function(x) return x * x end
    ),
    function(acc, x) return acc + x end,
    0
)

print(result)  -- ผลรวมของกำลังสองของตัวเลขที่มากกว่า 3
-- filter: {4,5,9,6,5,5} -> map: {16,25,81,36,25,25} -> reduce: 208
```

---

## 26.5 compose() และ pipe()

```lua
-- compose: รวมฟังก์ชันจากขวาไปซ้าย
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

local addOne = function(x) return x + 1 end
local double = function(x) return x * 2 end
local square = function(x) return x * x end

-- f(g(h(x))) = square(double(addOne(x)))
local transform = compose(square, double, addOne)
print(transform(3))  -- square(double(addOne(3))) = square(double(4)) = square(8) = 64

-- ทดสอบ compose หลายฟังก์ชัน
local toString = tostring
local upper = string.upper
local addExclaim = function(s) return s .. "!" end

local shout = compose(addExclaim, upper, toString)
print(shout(42))    -- 42!
print(shout("hi"))  -- HI!
```

```lua
-- pipe: รวมฟังก์ชันจากซ้ายไปขวา (ตรงข้าม compose)
local function pipe(...)
    local fns = {...}
    return function(x)
        local result = x
        for i = 1, #fns do
            result = fns[i](result)
        end
        return result
    end
end

local process = pipe(
    function(x) return x + 1 end,
    function(x) return x * 2 end,
    function(x) return x - 3 end
)

print(process(5))  -- ((5+1)*2)-3 = 9

-- pipe กับ string processing
local processName = pipe(
    string.lower,
    string.gsub,  -- ไม่สามารถใช้ตรงๆ ต้องห่อ
    function(s) return s:gsub("%s+", "_") end,
    function(s) return s:gsub("[^%w_]", "") end
)

print(processName("Hello World! 123"))  -- hello_world_123
```

```lua
-- compose กับ multiple arity
local function composeTwo(f, g)
    return function(...)
        return f(g(...))
    end
end

local function identity(x) return x end

local function reduceRight(fns)
    if #fns == 0 then return identity end
    if #fns == 1 then return fns[1] end
    local result = fns[#fns]
    for i = #fns - 1, 1, -1 do
        result = composeTwo(fns[i], result)
    end
    return result
end

local pipeline = reduceRight({
    function(x) return x .. "!" end,
    string.upper,
    function(x) return x:sub(1, 5) end,
})

print(pipeline("hello world"))  -- HELLO!
```

---

## 26.6 Currying

Currying คือการแปลงฟังก์ชันที่รับหลาย argument เป็นฟังก์ชันที่รับทีละ argument

```lua
-- curry พื้นฐาน
local function curry(f)
    return function(a)
        return function(b)
            return f(a, b)
        end
    end
end

local add = function(a, b) return a + b end
local curriedAdd = curry(add)

local add5 = curriedAdd(5)
print(add5(3))   -- 8
print(add5(10))  -- 15
print(add5(20))  -- 25
```

```lua
-- curry อัตโนมัติ (auto-curry)
local function autocurry(f, arity)
    arity = arity or debug.getinfo(f, "u").nparams
    local function helper(args)
        if #args >= arity then
            return f(table.unpack(args))
        end
        return function(...)
            local new_args = {}
            for _, v in ipairs(args) do
                new_args[#new_args + 1] = v
            end
            for _, v in ipairs({...}) do
                new_args[#new_args + 1] = v
            end
            return helper(new_args)
        end
    end
    return helper({})
end

local function multiply(a, b, c)
    return a * b * c
end

local curriedMul = autocurry(multiply, 3)
print(curriedMul(2)(3)(4))    -- 24
print(curriedMul(2, 3)(4))    -- 24
print(curriedMul(2)(3, 4))    -- 24
print(curriedMul(2, 3, 4))    -- 24
```

```lua
-- ตัวอย่างการใช้ curry จริง
local function prop(key)
    return function(obj)
        return obj[key]
    end
end

local getName = prop("name")
local getAge = prop("age")

local people = {
    { name = "อลิส", age = 30 },
    { name = "บ็อบ", age = 25 },
    { name = "ชาร์ลี", age = 35 },
}

-- ใช้กับ map
local names = map(people, getName)
local ages = map(people, getAge)

for _, n in ipairs(names) do io.write(n .. " ") end
print()  -- อลิส บ็อบ ชาร์ลี

for _, a in ipairs(ages) do io.write(a .. " ") end
print()  -- 30 25 35
```

---

## 26.7 Partial Application

```lua
-- partial application
local function partial(f, ...)
    local partial_args = {...}
    return function(...)
        local args = {}
        for _, v in ipairs(partial_args) do
            args[#args + 1] = v
        end
        for _, v in ipairs({...}) do
            args[#args + 1] = v
        end
        return f(table.unpack(args))
    end
end

local function add(a, b, c)
    return a + b + c
end

local add10 = partial(add, 10)
local add10and20 = partial(add, 10, 20)

print(add10(5, 3))    -- 18
print(add10and20(7))  -- 37
```

```lua
-- partial กับ format string
local function formatString(template, ...)
    local args = {...}
    return (template:gsub("{(%d+)}", function(n)
        return tostring(args[tonumber(n)] or "")
    end))
end

local greetTemplate = partial(formatString, "สวัสดี {1}, คุณอายุ {2} ปี")
print(greetTemplate("สมชาย", 30))  -- สวัสดี สมชาย, คุณอายุ 30 ปี
print(greetTemplate("สมหญิง", 25)) -- สวัสดี สมหญิง, คุณอายุ 25 ปี
```

```lua
-- flip: สลับลำดับ argument สองตัวแรก
local function flip(f)
    return function(a, b, ...)
        return f(b, a, ...)
    end
end

local divide = function(a, b) return a / b end
local flipDiv = flip(divide)

print(divide(10, 2))   -- 5.0
print(flipDiv(10, 2))  -- 0.2 (2/10)

-- ใช้ flip กับ string.find
local findIn = flip(string.find)
local findInHello = partial(findIn, "hello")
print(findInHello("l"))  -- 3
```

---

## 26.8 Functors

Functor คือ container ที่สามารถ map ฟังก์ชันผ่าน value ภายในได้

```lua
-- Box functor
local Box = {}
Box.__index = Box

function Box.of(value)
    return setmetatable({ value = value }, Box)
end

function Box:map(f)
    return Box.of(f(self.value))
end

function Box:fold(f)
    return f(self.value)
end

function Box:__tostring()
    return "Box(" .. tostring(self.value) .. ")"
end

-- การใช้งาน
local result = Box.of(5)
    :map(function(x) return x * 2 end)
    :map(function(x) return x + 1 end)
    :map(function(x) return x * x end)
    :fold(function(x) return x end)

print(result)  -- 121 (((5*2)+1)^2 = 11^2 = 121)

-- functor laws
local boxA = Box.of(10)
local id = function(x) return x end
-- Identity: fmap(id) == id
print(tostring(boxA:map(id)))  -- Box(10)
```

```lua
-- Maybe functor
local Maybe = {}
Maybe.__index = Maybe

function Maybe.Just(value)
    return setmetatable({ _value = value, _isJust = true }, Maybe)
end

function Maybe.Nothing()
    return setmetatable({ _isJust = false }, Maybe)
end

function Maybe:isNothing()
    return not self._isJust
end

function Maybe:map(f)
    if self._isJust then
        return Maybe.Just(f(self._value))
    end
    return self
end

function Maybe:getOrElse(default)
    if self._isJust then
        return self._value
    end
    return default
end

function Maybe:__tostring()
    if self._isJust then
        return "Just(" .. tostring(self._value) .. ")"
    end
    return "Nothing"
end

-- การใช้งาน
local val1 = Maybe.Just(10)
    :map(function(x) return x * 2 end)
    :map(function(x) return x + 5 end)
    :getOrElse(0)

print(val1)  -- 25

local val2 = Maybe.Nothing()
    :map(function(x) return x * 2 end)
    :getOrElse(0)

print(val2)  -- 0

-- ป้องกัน nil safely
local function safeDiv(a, b)
    if b == 0 then return Maybe.Nothing() end
    return Maybe.Just(a / b)
end

print(tostring(safeDiv(10, 2)))  -- Just(5.0)
print(tostring(safeDiv(10, 0)))  -- Nothing
```

---

## 26.9 Monads

Monad คือ Functor ที่มีเพิ่ม flatMap (หรือ chain/bind) เพื่อจัดการ nested context

```lua
-- Maybe Monad (เพิ่มจาก Functor)
local Maybe = {}
Maybe.__index = Maybe

function Maybe.Just(value)
    return setmetatable({ _value = value, _isJust = true }, Maybe)
end

function Maybe.Nothing()
    return setmetatable({ _isJust = false }, Maybe)
end

function Maybe:map(f)
    if self._isJust then
        return Maybe.Just(f(self._value))
    end
    return self
end

function Maybe:chain(f)  -- flatMap / bind
    if self._isJust then
        return f(self._value)
    end
    return self
end

function Maybe:getOrElse(default)
    return self._isJust and self._value or default
end

function Maybe:__tostring()
    if self._isJust then
        return "Just(" .. tostring(self._value) .. ")"
    end
    return "Nothing"
end

-- ตัวอย่าง: ค้นหาที่อาจ fail
local users = {
    {id = 1, name = "อลิส", addressId = 10},
    {id = 2, name = "บ็อบ", addressId = nil},
}

local addresses = {
    [10] = {street = "ถนนสุขุมวิท", city = "กรุงเทพ"},
}

local function findUser(id)
    for _, u in ipairs(users) do
        if u.id == id then return Maybe.Just(u) end
    end
    return Maybe.Nothing()
end

local function findAddress(addressId)
    if addressId == nil then return Maybe.Nothing() end
    local addr = addresses[addressId]
    if addr then return Maybe.Just(addr) end
    return Maybe.Nothing()
end

-- ค้นหา address ของ user 1
local city1 = findUser(1)
    :chain(function(u) return findAddress(u.addressId) end)
    :map(function(addr) return addr.city end)
    :getOrElse("ไม่ทราบ")

print(city1)  -- กรุงเทพ

-- ค้นหา address ของ user 2 (ไม่มี addressId)
local city2 = findUser(2)
    :chain(function(u) return findAddress(u.addressId) end)
    :map(function(addr) return addr.city end)
    :getOrElse("ไม่ทราบ")

print(city2)  -- ไม่ทราบ

-- ค้นหา user ที่ไม่มีในระบบ
local city3 = findUser(99)
    :chain(function(u) return findAddress(u.addressId) end)
    :map(function(addr) return addr.city end)
    :getOrElse("ไม่ทราบ")

print(city3)  -- ไม่ทราบ
```

---

## 26.10 Either Monad

Either แทน computation ที่อาจสำเร็จ (Right) หรือล้มเหลว (Left)

```lua
-- Either Monad
local Either = {}
Either.__index = Either

function Either.Right(value)
    return setmetatable({ _value = value, _isRight = true }, Either)
end

function Either.Left(error)
    return setmetatable({ _error = error, _isRight = false }, Either)
end

function Either:isRight() return self._isRight end
function Either:isLeft() return not self._isRight end

function Either:map(f)
    if self._isRight then
        return Either.Right(f(self._value))
    end
    return self
end

function Either:chain(f)
    if self._isRight then
        return f(self._value)
    end
    return self
end

function Either:mapLeft(f)
    if not self._isRight then
        return Either.Left(f(self._error))
    end
    return self
end

function Either:fold(onLeft, onRight)
    if self._isRight then
        return onRight(self._value)
    end
    return onLeft(self._error)
end

function Either:getOrElse(default)
    return self._isRight and self._value or default
end

function Either:__tostring()
    if self._isRight then
        return "Right(" .. tostring(self._value) .. ")"
    end
    return "Left(" .. tostring(self._error) .. ")"
end

-- ตัวอย่างการใช้งาน
local function divide(a, b)
    if b == 0 then
        return Either.Left("หารด้วยศูนย์ไม่ได้!")
    end
    return Either.Right(a / b)
end

local function sqrt(n)
    if n < 0 then
        return Either.Left("ไม่สามารถหารากที่สองของจำนวนลบ!")
    end
    return Either.Right(math.sqrt(n))
end

-- สำเร็จ
local result1 = divide(16, 4)
    :chain(sqrt)
    :map(function(x) return math.floor(x) end)

print(tostring(result1))  -- Right(2)

-- ล้มเหลวตอน divide
local result2 = divide(16, 0)
    :chain(sqrt)

print(tostring(result2))  -- Left(หารด้วยศูนย์ไม่ได้!)

-- ล้มเหลวตอน sqrt
local result3 = divide(-16, 4)
    :chain(sqrt)

print(tostring(result3))  -- Left(ไม่สามารถหารากที่สองของจำนวนลบ!)

-- fold เพื่อ handle ทั้งสองกรณี
result1:fold(
    function(err) print("Error: " .. err) end,
    function(val) print("Success: " .. val) end
)  -- Success: 2
```

---

## 26.11 Promise-like Patterns

```lua
-- Promise-like pattern สำหรับ async simulation
local Promise = {}
Promise.__index = Promise

function Promise.resolve(value)
    return setmetatable({
        _state = "fulfilled",
        _value = value,
        _callbacks = {}
    }, Promise)
end

function Promise.reject(reason)
    return setmetatable({
        _state = "rejected",
        _reason = reason,
        _callbacks = {}
    }, Promise)
end

function Promise.new(executor)
    local promise = setmetatable({
        _state = "pending",
        _value = nil,
        _callbacks = {}
    }, Promise)

    local function resolve(value)
        if promise._state == "pending" then
            promise._state = "fulfilled"
            promise._value = value
        end
    end

    local function reject(reason)
        if promise._state == "pending" then
            promise._state = "rejected"
            promise._reason = reason
        end
    end

    local ok, err = pcall(executor, resolve, reject)
    if not ok then reject(err) end

    return promise
end

function Promise:andThen(onFulfilled, onRejected)
    if self._state == "fulfilled" and onFulfilled then
        local ok, result = pcall(onFulfilled, self._value)
        if ok then
            return Promise.resolve(result)
        else
            return Promise.reject(result)
        end
    elseif self._state == "rejected" and onRejected then
        local ok, result = pcall(onRejected, self._reason)
        if ok then
            return Promise.resolve(result)
        else
            return Promise.reject(result)
        end
    end
    return self
end

function Promise:catch(onRejected)
    return self:andThen(nil, onRejected)
end

-- การใช้งาน
Promise.resolve(10)
    :andThen(function(x) return x * 2 end)
    :andThen(function(x) return x + 5 end)
    :andThen(function(x) print("Result: " .. x) end)
-- Result: 25

Promise.reject("เกิดข้อผิดพลาด!")
    :andThen(function(x) return x * 2 end)
    :catch(function(err) print("Error: " .. err) end)
-- Error: เกิดข้อผิดพลาด!

-- ตัวอย่าง validate
local function validateAge(age)
    if type(age) ~= "number" then
        return Promise.reject("age ต้องเป็นตัวเลข")
    end
    if age < 0 or age > 150 then
        return Promise.reject("age ไม่ถูกต้อง: " .. age)
    end
    return Promise.resolve(age)
end

validateAge(25)
    :andThen(function(age) print("อายุที่ถูกต้อง: " .. age) end)
    :catch(function(err) print("Error: " .. err) end)
-- อายุที่ถูกต้อง: 25

validateAge(-5)
    :andThen(function(age) print("อายุที่ถูกต้อง: " .. age) end)
    :catch(function(err) print("Error: " .. err) end)
-- Error: age ไม่ถูกต้อง: -5
```

---

## 26.12 Lazy Sequences

Lazy sequence คือ sequence ที่คำนวณค่าเมื่อต้องการเท่านั้น

```lua
-- Lazy sequence generator
local function lazyRange(start, stop, step)
    step = step or 1
    local current = start
    return {
        next = function()
            if stop and current > stop then return nil end
            local val = current
            current = current + step
            return val
        end,
        take = function(self, n)
            local result = {}
            for i = 1, n do
                local val = self.next()
                if val == nil then break end
                result[i] = val
            end
            return result
        end
    }
end

local range = lazyRange(1, 100)
print(table.concat(range:take(5), ", "))   -- 1, 2, 3, 4, 5
print(table.concat(range:take(5), ", "))   -- 6, 7, 8, 9, 10

-- Infinite sequence
local naturals = lazyRange(1)
print(table.concat(naturals:take(10), ", "))  -- 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
```

```lua
-- Lazy map และ filter
local function lazyMap(seq, f)
    return {
        next = function()
            local val = seq.next()
            if val == nil then return nil end
            return f(val)
        end,
        take = function(self, n)
            local result = {}
            for i = 1, n do
                local val = self.next()
                if val == nil then break end
                result[i] = val
            end
            return result
        end
    }
end

local function lazyFilter(seq, pred)
    return {
        next = function()
            while true do
                local val = seq.next()
                if val == nil then return nil end
                if pred(val) then return val end
            end
        end,
        take = function(self, n)
            local result = {}
            for i = 1, n do
                local val = self.next()
                if val == nil then break end
                result[i] = val
            end
            return result
        end
    }
end

-- Lazy pipeline: หา 5 จำนวนคู่แรกที่เป็นกำลังสอง
local nums = lazyRange(1)
local squares = lazyMap(nums, function(x) return x * x end)
local evenSquares = lazyFilter(squares, function(x) return x % 2 == 0 end)

print(table.concat(evenSquares:take(5), ", "))  -- 4, 16, 36, 64, 100
```

---

## 26.13 Memoize with Cache

```lua
-- Memoize พื้นฐาน
local function memoize(f)
    local cache = {}
    return function(...)
        local key = table.concat({...}, ",")
        if cache[key] == nil then
            cache[key] = f(...)
        end
        return cache[key]
    end
end

-- Fibonacci ที่ไม่ได้ memoize (ช้า)
local function fib(n)
    if n <= 1 then return n end
    return fib(n - 1) + fib(n - 2)
end

-- Fibonacci ที่ memoize (เร็ว)
local function makeMemoFib()
    local cache = {}
    local function mfib(n)
        if n <= 1 then return n end
        if cache[n] then return cache[n] end
        cache[n] = mfib(n - 1) + mfib(n - 2)
        return cache[n]
    end
    return mfib
end

local memoFib = makeMemoFib()

-- วัดเวลา
local t1 = os.clock()
for i = 1, 30 do fib(25) end
local t2 = os.clock()
print(string.format("Non-memoized: %.4f s", t2 - t1))

local t3 = os.clock()
for i = 1, 30 do memoFib(25) end
local t4 = os.clock()
print(string.format("Memoized: %.4f s", t4 - t3))

print(memoFib(30))  -- 832040
```

```lua
-- Memoize แบบ LRU (Least Recently Used)
local function memoizeLRU(f, maxSize)
    maxSize = maxSize or 100
    local cache = {}
    local order = {}  -- LRU order

    local function evict()
        if #order > maxSize then
            local oldest = table.remove(order, 1)
            cache[oldest] = nil
        end
    end

    return function(...)
        local key = table.concat({...}, "|")
        if cache[key] ~= nil then
            -- ย้ายไปท้าย order (most recently used)
            for i, k in ipairs(order) do
                if k == key then
                    table.remove(order, i)
                    break
                end
            end
            order[#order + 1] = key
            return cache[key]
        end

        local result = f(...)
        cache[key] = result
        order[#order + 1] = key
        evict()
        return result
    end
end

local callCount = 0
local expensiveCalc = memoizeLRU(function(n)
    callCount = callCount + 1
    return n * n * n
end, 3)

print(expensiveCalc(2))  -- 8 (computed)
print(expensiveCalc(3))  -- 27 (computed)
print(expensiveCalc(2))  -- 8 (cached)
print(expensiveCalc(4))  -- 64 (computed)
print(expensiveCalc(5))  -- 125 (computed, evicts 3)
print("Total computations: " .. callCount)  -- 4
```

---

## 26.14 Point-Free Style

Point-free style คือการเขียนฟังก์ชันโดยไม่ระบุ argument อย่างชัดเจน

```lua
-- ฟังก์ชัน utility สำหรับ point-free
local function prop(key)
    return function(obj) return obj[key] end
end

local function gt(n)
    return function(x) return x > n end
end

local function lt(n)
    return function(x) return x < n end
end

local function eq(n)
    return function(x) return x == n end
end

local function not_(f)
    return function(...)
        return not f(...)
    end
end

local function and_(f, g)
    return function(...)
        return f(...) and g(...)
    end
end

local function or_(f, g)
    return function(...)
        return f(...) or g(...)
    end
end

-- Point-free filter
local isAdult = gt(17)  -- age > 17
local isChild = lt(18)  -- age < 18
local isTeenager = and_(gt(12), lt(20))  -- 13-19

local ages = {5, 13, 17, 18, 21, 25, 12}
print(table.concat(filter(ages, isAdult), ", "))    -- 18, 21, 25
print(table.concat(filter(ages, isTeenager), ", ")) -- 13, 17, 18
```

```lua
-- Point-free pipeline
local people = {
    { name = "อลิส", age = 30, score = 85 },
    { name = "บ็อบ", age = 17, score = 92 },
    { name = "ชาร์ลี", age = 25, score = 78 },
    { name = "ดาน่า", age = 15, score = 95 },
    { name = "อีฟ", age = 28, score = 88 },
}

-- Point-free: ดึงชื่อคนที่อายุ >= 18 และคะแนน >= 80
local getName = prop("name")
local getAge = prop("age")
local getScore = prop("score")
local isAdultAge = function(p) return getAge(p) >= 18 end
local hasHighScore = function(p) return getScore(p) >= 80 end
local isQualified = and_(isAdultAge, hasHighScore)

local qualifiedNames = map(filter(people, isQualified), getName)
print(table.concat(qualifiedNames, ", "))  -- อลิส, อีฟ
```

---

## 26.15 Transducers

Transducer คือ composable transformation ที่ไม่สร้าง intermediate collections

```lua
-- Transducer implementation
local function mapping(f)
    return function(reducer)
        return function(acc, item)
            return reducer(acc, f(item))
        end
    end
end

local function filtering(pred)
    return function(reducer)
        return function(acc, item)
            if pred(item) then
                return reducer(acc, item)
            end
            return acc
        end
    end
end

local function transduce(xf, reducer, init, coll)
    local transformedReducer = xf(reducer)
    local acc = init
    for _, v in ipairs(coll) do
        acc = transformedReducer(acc, v)
    end
    return acc
end

local function composeXf(...)
    local fns = {...}
    return function(reducer)
        local result = reducer
        for i = #fns, 1, -1 do
            result = fns[i](result)
        end
        return result
    end
end

-- append reducer
local append = function(acc, item)
    acc[#acc + 1] = item
    return acc
end

-- ตัวอย่าง: filter even numbers and double them
local xf = composeXf(
    filtering(function(x) return x % 2 == 0 end),
    mapping(function(x) return x * 2 end)
)

local data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
local result = transduce(xf, append, {}, data)
print(table.concat(result, ", "))  -- 4, 8, 12, 16, 20

-- ใช้ sum reducer แทน
local sum_reducer = function(acc, item) return acc + item end
local total = transduce(xf, sum_reducer, 0, data)
print(total)  -- 60
```

---

## 26.16 สรุปและตัวอย่างรวม

```lua
-- ตัวอย่างรวม: Data processing pipeline แบบ functional
local function pipe(...)
    local fns = {...}
    return function(data)
        local result = data
        for _, f in ipairs(fns) do
            result = f(result)
        end
        return result
    end
end

-- Dataset
local employees = {
    { name = "สมชาย",   dept = "IT",  salary = 45000, years = 5 },
    { name = "สมหญิง",  dept = "HR",  salary = 38000, years = 3 },
    { name = "สมพงษ์",  dept = "IT",  salary = 52000, years = 8 },
    { name = "สมศักดิ์", dept = "IT",  salary = 41000, years = 2 },
    { name = "สมรัก",   dept = "HR",  salary = 42000, years = 6 },
    { name = "สมใจ",    dept = "FIN", salary = 55000, years = 10 },
}

-- Functional pipeline
local processIT = pipe(
    function(data) return filter(data, function(e) return e.dept == "IT" end) end,
    function(data) return filter(data, function(e) return e.years >= 3 end) end,
    function(data) return map(data, function(e)
        return { name = e.name, salary = e.salary * 1.1 }  -- เพิ่มเงินเดือน 10%
    end) end,
    function(data)
        local total = reduce(data, function(acc, e) return acc + e.salary end, 0)
        return { employees = data, totalSalary = total }
    end
)

local result = processIT(employees)
print("IT employees with >= 3 years experience (after 10% raise):")
for _, e in ipairs(result.employees) do
    print(string.format("  %s: %.0f บาท", e.name, e.salary))
end
print(string.format("Total salary budget: %.0f บาท", result.totalSalary))
```

```lua
-- Monad chaining สำหรับ validation
local Validation = {}
Validation.__index = Validation

function Validation.success(value)
    return setmetatable({ ok = true, value = value, errors = {} }, Validation)
end

function Validation.failure(errors)
    if type(errors) == "string" then errors = {errors} end
    return setmetatable({ ok = false, value = nil, errors = errors }, Validation)
end

function Validation:andThen(f)
    if self.ok then
        return f(self.value)
    end
    return self
end

function Validation:combine(other)
    if self.ok and other.ok then
        return Validation.success({self.value, other.value})
    end
    local errors = {}
    for _, e in ipairs(self.errors) do errors[#errors+1] = e end
    for _, e in ipairs(other.errors) do errors[#errors+1] = e end
    return Validation.failure(errors)
end

-- Validators
local function validateName(name)
    if type(name) ~= "string" or #name < 2 then
        return Validation.failure("ชื่อต้องมีอย่างน้อย 2 ตัวอักษร")
    end
    return Validation.success(name)
end

local function validateEmail(email)
    if not email:match("[^@]+@[^@]+%.[^@]+") then
        return Validation.failure("email ไม่ถูกต้อง")
    end
    return Validation.success(email)
end

local function validateAge(age)
    if type(age) ~= "number" or age < 18 or age > 120 then
        return Validation.failure("อายุต้องอยู่ระหว่าง 18-120")
    end
    return Validation.success(age)
end

-- ทดสอบ
local v1 = validateName("สมชาย")
    :andThen(function(name)
        return validateEmail("somchai@example.com")
    end)
print(v1.ok, v1.value)  -- true somchai@example.com

local v2 = validateAge(15)
print(v2.ok, v2.errors[1])  -- false อายุต้องอยู่ระหว่าง 18-120

print("\n--- Functional Programming สรุป ---")
print("1. First-class functions - ฟังก์ชันเป็น value")
print("2. Pure functions - ไม่มี side effects")
print("3. Immutability - ไม่แก้ไข state")
print("4. map/filter/reduce - transformation patterns")
print("5. compose/pipe - function composition")
print("6. curry/partial - function specialization")
print("7. Functors/Monads - wrapping context")
print("8. Lazy evaluation - คำนวณเมื่อต้องการ")
print("9. Memoization - cache results")
print("10. Transducers - efficient transformations")
```

---

## บทสรุป

Functional Programming ใน Lua ช่วยให้:
- โค้ดอ่านง่ายและทดสอบได้ง่าย
- ลด bugs จาก side effects
- นำ code กลับมาใช้ซ้ำได้ง่ายผ่าน composition
- จัดการ error อย่างปลอดภัยด้วย Maybe/Either monad

แม้ Lua จะไม่ใช่ภาษา functional โดยธรรมชาติ แต่รองรับ FP patterns ได้เป็นอย่างดี และนักพัฒนาสามารถใช้เทคนิคเหล่านี้ร่วมกับ OOP หรือ Procedural style ได้ตามความเหมาะสม
