# บทที่ 25: Iterators และ Generators

## บทนำ

Iterators เป็นกลไกสำคัญที่ทำให้ `for-in` loop ทำงานได้ใน Lua การเข้าใจ iterators อย่างถ่องแท้ช่วยให้เราสร้าง custom loops ที่หลากหลาย รวมถึง generators สำหรับ sequences ที่ซับซ้อน

ในบทนี้เราจะเรียนรู้:
- กลไกภายในของ `for-in` loop
- Stateful iterators (closure-based)
- Stateless iterators (pairs/ipairs style)
- Generic for protocol
- Custom pairs/ipairs
- Range, Filter, Map, Zip iterators
- Chain, Take/Drop, Flatten, Unique iterators
- Batch/Chunk, Reverse iterators
- Coroutine-based generators
- Infinite generators

---

## 25.1 กลไกภายในของ for-in loop

### ตัวอย่างที่ 1: ทำความเข้าใจ Generic for

```lua
-- for-in ใน Lua ทำงานอย่างไร?
-- for var_1, ..., var_n in explist do block end
-- แปลงเป็น:
--   local iterFunc, state, control = explist
--   while true do
--     local var_1, ..., var_n = iterFunc(state, control)
--     if var_1 == nil then break end
--     control = var_1
--     block
--   end

-- ดูตัวอย่างง่ายๆ
-- ipairs มีลักษณะเช่นนี้:
local function myIpairs(t)
    -- คืน: iterator function, state, initial control variable
    local function iterFunc(state, control)
        local nextIndex = control + 1
        local value = state[nextIndex]
        if value == nil then return nil end  -- หยุด loop
        return nextIndex, value  -- คืน key, value
    end
    
    return iterFunc, t, 0  -- iterator fn, table เป็น state, 0 เป็น control แรก
end

-- ทดสอบ
local fruits = {"แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง"}

print("ใช้ myIpairs:")
for i, v in myIpairs(fruits) do
    print(string.format("  [%d] %s", i, v))
end

-- เปรียบเทียบกับ ipairs จริง
print("\nใช้ ipairs จริง:")
for i, v in ipairs(fruits) do
    print(string.format("  [%d] %s", i, v))
end
```

### ตัวอย่างที่ 2: Stateless Iterator

```lua
-- Stateless Iterator: ไม่เก็บ state ภายใน
-- ทำงานได้ด้วย state และ control variable เท่านั้น

-- Iterator function สำหรับ integer range
local function rangeIter(state, control)
    -- state = {stop, step}
    -- control = current value
    local next = control + state.step
    if (state.step > 0 and next > state.stop) or
       (state.step < 0 and next < state.stop) then
        return nil
    end
    return next
end

-- Factory ที่คืน iterator function, state, initial control
local function range(start, stop, step)
    step = step or 1
    -- คืน iterator, state, initial (start - step เพื่อให้ครั้งแรกได้ start)
    return rangeIter, {stop = stop, step = step}, start - step
end

-- ทดสอบ
print("range(1, 10):")
for i in range(1, 10) do
    io.write(i .. " ")
end
print()

print("\nrange(0, 20, 5):")
for i in range(0, 20, 5) do
    io.write(i .. " ")
end
print()

print("\nrange(10, 1, -1):")
for i in range(10, 1, -1) do
    io.write(i .. " ")
end
print()
```

### ตัวอย่างที่ 3: Stateful Iterator ด้วย Closure

```lua
-- Stateful Iterator: เก็บ state ภายใน closure
-- ง่ายกว่า stateless แต่ใช้ memory มากกว่าเล็กน้อย

-- Simple counter iterator
local function counter(from, to, step)
    step = step or 1
    local current = from - step
    
    return function()
        current = current + step
        if (step > 0 and current <= to) or
           (step < 0 and current >= to) then
            return current
        end
        -- คืน nil เพื่อหยุด loop
    end
end

print("counter(1, 10, 2):")
for v in counter(1, 10, 2) do
    io.write(v .. " ")
end
print()

-- Iterator ที่ track index
local function indexed(t)
    local i = 0
    local n = #t
    return function()
        i = i + 1
        if i <= n then
            return i, t[i]
        end
    end
end

print("\nindexed:")
for i, v in indexed({"a", "b", "c", "d"}) do
    print(string.format("  [%d] = %s", i, v))
end

-- Reverse iterator
local function reversed(t)
    local i = #t + 1
    return function()
        i = i - 1
        if i >= 1 then
            return i, t[i]
        end
    end
end

print("\nreversed:")
for i, v in reversed({"ก", "ข", "ค", "ง", "จ"}) do
    print(string.format("  [%d] = %s", i, v))
end
```

---

## 25.2 Custom pairs และ ipairs

### ตัวอย่างที่ 4: Custom pairs

```lua
-- สร้าง pairs ที่เรียงลำดับ key
local function sortedPairs(t, compareFn)
    -- เก็บ keys ทั้งหมด
    local keys = {}
    for k in pairs(t) do
        table.insert(keys, k)
    end
    
    -- เรียงลำดับ
    if compareFn then
        table.sort(keys, compareFn)
    else
        table.sort(keys, function(a, b)
            return tostring(a) < tostring(b)
        end)
    end
    
    local i = 0
    return function()
        i = i + 1
        local key = keys[i]
        if key ~= nil then
            return key, t[key]
        end
    end
end

local data = {
    banana = 3,
    apple = 1,
    cherry = 5,
    date = 2,
    elderberry = 4
}

print("sortedPairs (by key):")
for k, v in sortedPairs(data) do
    print(string.format("  %s: %d", k, v))
end

print("\nsortedPairs (by value desc):")
for k, v in sortedPairs(data, function(a, b)
    return data[a] > data[b]
end) do
    print(string.format("  %s: %d", k, v))
end
```

### ตัวอย่างที่ 5: Custom ipairs ที่ยืดหยุ่น

```lua
-- ipairs ที่รองรับ start/stop/step
local function ipairsRange(t, start, stop, step)
    start = start or 1
    stop = stop or #t
    step = step or 1
    
    local i = start - step
    return function()
        i = i + step
        if (step > 0 and i <= stop) or
           (step < 0 and i >= stop) then
            if t[i] ~= nil then
                return i, t[i]
            end
        end
    end
end

local nums = {10, 20, 30, 40, 50, 60, 70, 80, 90, 100}

print("ipairsRange(t, 3, 7):")  -- index 3 ถึง 7
for i, v in ipairsRange(nums, 3, 7) do
    print(string.format("  [%d] = %d", i, v))
end

print("\nipairsRange(t, 2, 10, 2):")  -- ทุก 2 ตัว
for i, v in ipairsRange(nums, 2, 10, 2) do
    print(string.format("  [%d] = %d", i, v))
end

print("\nipairsRange(t, 8, 2, -2):")  -- ย้อนกลับ
for i, v in ipairsRange(nums, 8, 2, -2) do
    print(string.format("  [%d] = %d", i, v))
end

-- ipairs ที่ข้ามค่า nil
local sparse = {1, nil, 3, nil, 5, nil, 7}
local function safeIpairs(t)
    local i = 0
    local n = 0
    -- หา max index
    for k in pairs(t) do
        if type(k) == "number" and k > n then n = k end
    end
    
    return function()
        repeat
            i = i + 1
        until i > n or t[i] ~= nil
        if i <= n then
            return i, t[i]
        end
    end
end

print("\nsafeIpairs (ข้ามค่า nil):")
for i, v in safeIpairs(sparse) do
    print(string.format("  [%d] = %d", i, v))
end
```

---

## 25.3 Filter Iterator

### ตัวอย่างที่ 6: Filter Iterator

```lua
-- Filter Iterator: กรองค่าตาม predicate

-- Filter แบบ stateful
local function filter(gen, pred)
    return function()
        while true do
            local val = gen()
            if val == nil then return nil end
            if pred(val) then return val end
        end
    end
end

-- Filter ที่รองรับ multiple return values
local function filterKV(t, pred)
    local keys = {}
    for k in pairs(t) do table.insert(keys, k) end
    table.sort(keys, function(a, b) return tostring(a) < tostring(b) end)
    
    local i = 0
    return function()
        while true do
            i = i + 1
            local k = keys[i]
            if k == nil then return nil end
            local v = t[k]
            if pred(k, v) then
                return k, v
            end
        end
    end
end

-- สร้าง generator จาก array
local function fromArray(arr)
    local i = 0
    return function()
        i = i + 1
        return arr[i]
    end
end

-- ทดสอบ
print("=== Filter Iterator ===\n")

-- กรองเลขคู่
local numbers = fromArray({1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
local evens = filter(numbers, function(x) return x % 2 == 0 end)

print("เลขคู่:")
for v in evens do io.write(v .. " ") end
print()

-- กรองคำที่ยาวกว่า 4 ตัวอักษร
local words = fromArray({"Lua", "Python", "Go", "JavaScript", "C", "Ruby", "Rust"})
local longWords = filter(words, function(w) return #w > 4 end)

print("\nคำที่ยาวกว่า 4 ตัวอักษร:")
for w in longWords do io.write(w .. " ") end
print()

-- Filter table
local students = {
    Alice = 85, Bob = 72, Charlie = 90, 
    Diana = 65, Eve = 88, Frank = 55
}

print("\nnักเรียนที่ได้คะแนน >= 80:")
for name, score in filterKV(students, function(k, v) return v >= 80 end) do
    print(string.format("  %s: %d", name, score))
end
```

### ตัวอย่างที่ 7: Chained Filters

```lua
-- เชื่อมต่อ filters หลายชั้น

-- Iterator factory ที่ใช้ method chaining
local IterChain = {}
IterChain.__index = IterChain

function IterChain.from(gen)
    return setmetatable({_gen = gen}, IterChain)
end

function IterChain.fromArray(arr)
    local i = 0
    return IterChain.from(function()
        i = i + 1
        return arr[i]
    end)
end

function IterChain:filter(pred)
    local gen = self._gen
    return IterChain.from(function()
        while true do
            local v = gen()
            if v == nil then return nil end
            if pred(v) then return v end
        end
    end)
end

function IterChain:map(fn)
    local gen = self._gen
    return IterChain.from(function()
        local v = gen()
        if v == nil then return nil end
        return fn(v)
    end)
end

function IterChain:take(n)
    local gen = self._gen
    local count = 0
    return IterChain.from(function()
        if count >= n then return nil end
        local v = gen()
        if v == nil then return nil end
        count = count + 1
        return v
    end)
end

function IterChain:skip(n)
    local gen = self._gen
    local skipped = false
    return IterChain.from(function()
        if not skipped then
            for i = 1, n do gen() end
            skipped = true
        end
        return gen()
    end)
end

function IterChain:toArray()
    local result = {}
    for v in self._gen do
        table.insert(result, v)
    end
    return result
end

function IterChain:forEach(fn)
    for v in self._gen do fn(v) end
end

function IterChain:sum()
    local total = 0
    for v in self._gen do total = total + v end
    return total
end

function IterChain:count()
    local n = 0
    for _ in self._gen do n = n + 1 end
    return n
end

function IterChain:first()
    return self._gen()
end

-- ใช้ __call เพื่อให้ใช้ใน for-in ได้
function IterChain:__call()
    return self._gen()
end

function IterChain:iter()
    return self._gen
end

-- ทดสอบ
print("=== Chained Iterator ===\n")

local data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15}

-- หา sum ของ square ของเลขคี่ที่มากกว่า 5 และน้อยกว่า 13
local result = IterChain.fromArray(data)
    :filter(function(x) return x % 2 ~= 0 end)  -- เลขคี่
    :filter(function(x) return x > 5 end)         -- > 5
    :filter(function(x) return x < 13 end)        -- < 13
    :map(function(x) return x * x end)             -- ยกกำลัง 2
    :toArray()

print("Odd numbers > 5 and < 13, squared:")
print("{" .. table.concat(result, ", ") .. "}")

local sum = IterChain.fromArray(data)
    :filter(function(x) return x % 2 ~= 0 end)
    :filter(function(x) return x > 5 and x < 13 end)
    :map(function(x) return x * x end)
    :sum()
print("Sum:", sum)

-- First 5 even numbers > 10
local first5Evens = IterChain.fromArray(data)
    :filter(function(x) return x % 2 == 0 end)
    :filter(function(x) return x > 5 end)
    :take(5)
    :toArray()
print("First 5 even > 5:", "{" .. table.concat(first5Evens, ", ") .. "}")
```

---

## 25.4 Map Iterator

### ตัวอย่างที่ 8: Map Iterator

```lua
-- Map Iterator: แปลงแต่ละค่า

local function map(gen, fn)
    return function()
        local v = gen()
        if v == nil then return nil end
        return fn(v)
    end
end

-- Map ที่รองรับ multiple values
local function mapKV(gen, fn)
    return function()
        local k, v = gen()
        if k == nil then return nil end
        return fn(k, v)
    end
end

-- สร้าง from functions
local function fromArray(arr)
    local i = 0
    return function()
        i = i + 1
        return arr[i]
    end
end

local function fromPairs(t)
    return next, t, nil
end

-- ทดสอบ Map
print("=== Map Iterator ===\n")

-- แปลงตัวเลขเป็น string
local nums = fromArray({1, 2, 3, 4, 5})
local strs = map(nums, function(x) return string.format("num_%d", x) end)

for s in strs do io.write(s .. " ") end
print()

-- แปลง degrees เป็น radians
local degrees = fromArray({0, 30, 45, 60, 90, 180, 270, 360})
local radians = map(degrees, function(d) return d * math.pi / 180 end)

print("\nDegrees -> Radians:")
for r in radians do
    io.write(string.format("%.4f ", r))
end
print()

-- Map ที่แปลงหลายค่า (one-to-many: flatMap)
local function flatMap(gen, fn)
    local outerVal = nil
    local innerGen = nil
    
    return function()
        while true do
            if innerGen then
                local v = innerGen()
                if v ~= nil then return v end
                innerGen = nil
            end
            
            outerVal = gen()
            if outerVal == nil then return nil end
            
            innerGen = fn(outerVal)
        end
    end
end

-- ขยาย: แต่ละตัวเลข n ผลิต {n, n*2, n*3}
local source = fromArray({1, 2, 3})
local expanded = flatMap(source, function(n)
    local i = 0
    return function()
        i = i + 1
        if i <= 3 then return n * i end
    end
end)

print("\nflatMap (n -> {n, 2n, 3n}):")
for v in expanded do io.write(v .. " ") end
print()
```

---

## 25.5 Zip Iterator

### ตัวอย่างที่ 9: Zip Iterator

```lua
-- Zip: รวม iterators หลายตัวเป็นคู่

-- zip 2 generators
local function zip(gen1, gen2)
    return function()
        local v1 = gen1()
        local v2 = gen2()
        if v1 == nil or v2 == nil then return nil end
        return v1, v2
    end
end

-- zipN: zip หลาย generators
local function zipN(...)
    local gens = {...}
    return function()
        local values = {}
        for _, gen in ipairs(gens) do
            local v = gen()
            if v == nil then return nil end
            table.insert(values, v)
        end
        return table.unpack(values)
    end
end

-- zipLongest: zip และเติม nil สำหรับที่หมดก่อน
local function zipLongest(gen1, gen2, fill1, fill2)
    local done1, done2 = false, false
    return function()
        if done1 and done2 then return nil end
        
        local v1 = not done1 and gen1() or nil
        local v2 = not done2 and gen2() or nil
        
        if v1 == nil then done1 = true; v1 = fill1 end
        if v2 == nil then done2 = true; v2 = fill2 end
        
        if done1 and done2 then return nil end
        return v1, v2
    end
end

-- zipWithIndex
local function zipWithIndex(gen, start)
    start = start or 1
    local i = start - 1
    return function()
        local v = gen()
        if v == nil then return nil end
        i = i + 1
        return i, v
    end
end

-- สร้าง array generators
local function arr(t)
    local i = 0
    return function() i = i + 1; return t[i] end
end

-- ทดสอบ Zip
print("=== Zip Iterator ===\n")

local names = arr({"Alice", "Bob", "Charlie", "Diana"})
local scores = arr({95, 87, 92, 78})
print("zip(names, scores):")
for name, score in zip(names, scores) do
    print(string.format("  %s: %d", name, score))
end

print("\nzipN(letters, numbers, booleans):")
local letters = arr({"A", "B", "C"})
local numbers = arr({1, 2, 3})
local bools = arr({true, false, true})
for l, n, b in zipN(letters, numbers, bools) do
    print(string.format("  %s, %d, %s", l, n, tostring(b)))
end

print("\nzipLongest:")
local short = arr({1, 2, 3})
local long = arr({10, 20, 30, 40, 50})
for a, b in zipLongest(short, long, 0, 0) do
    print(string.format("  %d + %d = %d", a, b, a+b))
end

print("\nzipWithIndex:")
local colors = arr({"แดง", "เขียว", "น้ำเงิน", "เหลือง"})
for i, color in zipWithIndex(colors, 1) do
    print(string.format("  [%d] %s", i, color))
end

-- Unzip: แยก pairs กลับเป็น 2 arrays
local function unzip(gen)
    local arr1, arr2 = {}, {}
    for v1, v2 in gen do
        table.insert(arr1, v1)
        table.insert(arr2, v2)
    end
    return arr1, arr2
end

local pairs_gen = zip(arr({1, 2, 3}), arr({"a", "b", "c"}))
local nums, chars = unzip(pairs_gen)
print("\nunzip result:")
print("  nums:", table.concat(nums, ", "))
print("  chars:", table.concat(chars, ", "))
```

---

## 25.6 Chain Iterator

### ตัวอย่างที่ 10: Chain Iterator

```lua
-- Chain: ต่อ iterators เป็น sequence

-- chain 2 generators
local function chain(gen1, gen2)
    local useFirst = true
    return function()
        if useFirst then
            local v = gen1()
            if v ~= nil then return v end
            useFirst = false
        end
        return gen2()
    end
end

-- chainN: chain หลาย generators
local function chainN(...)
    local gens = {...}
    local i = 1
    return function()
        while i <= #gens do
            local v = gens[i]()
            if v ~= nil then return v end
            i = i + 1
        end
        return nil
    end
end

-- Cycle: วนซ้ำ generator
local function cycle(arr_data, times)
    local arr_copy = {}
    for _, v in ipairs(arr_data) do table.insert(arr_copy, v) end
    
    local i = 0
    local rep = 0
    times = times or math.huge
    
    return function()
        if rep >= times then return nil end
        i = i + 1
        if i > #arr_copy then
            i = 1
            rep = rep + 1
            if rep >= times then return nil end
        end
        return arr_copy[i]
    end
end

-- Repeat: ทำซ้ำค่าเดิม
local function repeatVal(value, times)
    local count = 0
    times = times or math.huge
    return function()
        if count >= times then return nil end
        count = count + 1
        return value
    end
end

-- สร้าง arr generator
local function arr(t)
    local i = 0
    return function() i = i + 1; return t[i] end
end

-- ทดสอบ Chain
print("=== Chain Iterator ===\n")

local gen1 = arr({1, 2, 3})
local gen2 = arr({4, 5, 6})
local gen3 = arr({7, 8, 9})

print("chain(1-3, 4-6):")
for v in chain(gen1, gen2) do io.write(v .. " ") end
print()

print("\nchainN(1-3, 4-6, 7-9):")
for v in chainN(arr({1,2,3}), arr({4,5,6}), arr({7,8,9})) do
    io.write(v .. " ")
end
print()

print("\ncycle({1,2,3}, 3 times):")
for v in cycle({1, 2, 3}, 3) do io.write(v .. " ") end
print()

print("\nrepeat('x', 5 times):")
for v in repeatVal("x", 5) do io.write(v .. " ") end
print()

-- Interleave: สลับกัน
local function interleave(...)
    local gens = {...}
    local i = 0
    local active = #gens
    local done = {}
    
    return function()
        while active > 0 do
            i = (i % #gens) + 1
            if not done[i] then
                local v = gens[i]()
                if v ~= nil then
                    return v
                else
                    done[i] = true
                    active = active - 1
                end
            end
        end
        return nil
    end
end

print("\ninterleave({A,B,C}, {1,2,3}, {x,y}):")
for v in interleave(arr({"A","B","C"}), arr({1,2,3}), arr({"x","y"})) do
    io.write(tostring(v) .. " ")
end
print()
```

---

## 25.7 Take / Drop Iterators

### ตัวอย่างที่ 11: Take, Drop, TakeWhile, DropWhile

```lua
-- Take/Drop family

-- Take: เอาแค่ n ตัวแรก
local function take(gen, n)
    local count = 0
    return function()
        if count >= n then return nil end
        local v = gen()
        if v == nil then return nil end
        count = count + 1
        return v
    end
end

-- Drop: ข้าม n ตัวแรก
local function drop(gen, n)
    local dropped = false
    return function()
        if not dropped then
            for i = 1, n do
                if gen() == nil then 
                    dropped = true
                    return nil 
                end
            end
            dropped = true
        end
        return gen()
    end
end

-- TakeWhile: เอาจนกว่า predicate เป็น false
local function takeWhile(gen, pred)
    local done = false
    return function()
        if done then return nil end
        local v = gen()
        if v == nil or not pred(v) then
            done = true
            return nil
        end
        return v
    end
end

-- DropWhile: ข้ามจนกว่า predicate เป็น false
local function dropWhile(gen, pred)
    local dropping = true
    return function()
        while true do
            local v = gen()
            if v == nil then return nil end
            if dropping then
                if not pred(v) then
                    dropping = false
                    return v
                end
            else
                return v
            end
        end
    end
end

-- SliceTo: เอา slice จาก start ถึง stop
local function sliceTo(gen, start, stop)
    return take(drop(gen, start - 1), stop - start + 1)
end

-- สร้าง arr generator
local function arr(t)
    local i = 0
    return function() i = i + 1; return t[i] end
end

-- Infinite counting
local function counting(from)
    local n = (from or 1) - 1
    return function()
        n = n + 1
        return n
    end
end

-- ทดสอบ
print("=== Take / Drop ===\n")

print("take(counting, 5):")
for v in take(counting(), 5) do io.write(v .. " ") end
print()

print("\ndrop(arr(1..10), 5):")
for v in drop(arr({1,2,3,4,5,6,7,8,9,10}), 5) do io.write(v .. " ") end
print()

print("\ntakeWhile(counting, x < 8):")
for v in takeWhile(counting(), function(x) return x < 8 end) do
    io.write(v .. " ")
end
print()

print("\ndropWhile(arr, x < 5):")
local data = arr({1, 2, 3, 4, 5, 6, 7, 8})
for v in dropWhile(data, function(x) return x < 5 end) do
    io.write(v .. " ")
end
print()

print("\nsliceTo(1..20, 5, 12):")
for v in sliceTo(counting(), 5, 12) do io.write(v .. " ") end
print()
```

---

## 25.8 Flatten Iterator

### ตัวอย่างที่ 12: Flatten Iterator

```lua
-- Flatten: ทำ nested structure ให้ราบ

-- Flatten 1 level
local function flatten1(gen)
    local current = nil
    local i = 0
    local currentArr = nil
    
    return function()
        while true do
            -- ถ้ากำลัง iterate ใน nested array
            if currentArr then
                i = i + 1
                if i <= #currentArr then
                    return currentArr[i]
                end
                currentArr = nil
                i = 0
            end
            
            -- ขอ item ถัดไปจาก outer gen
            local v = gen()
            if v == nil then return nil end
            
            if type(v) == "table" then
                currentArr = v
                i = 0
            else
                return v
            end
        end
    end
end

-- Deep flatten (flatten ทุก level)
local function deepFlatten(gen)
    local stack = {gen}
    
    return function()
        while #stack > 0 do
            local current = stack[#stack]
            local v = current()
            
            if v == nil then
                table.remove(stack)
            elseif type(v) == "table" then
                -- Push iterator สำหรับ nested table
                local inner = ipairs(v)
                local innerGen = function()
                    local _, val = inner({}, 0)  -- ไม่ work แบบนี้
                    return val
                end
                
                -- แก้ไข: ใช้ closure ที่ถูกต้อง
                local idx = 0
                local tbl = v
                table.insert(stack, function()
                    idx = idx + 1
                    return tbl[idx]
                end)
            else
                return v
            end
        end
        return nil
    end
end

-- Flatten ที่ดีกว่า: recursive
local function flattenDeep(arr)
    return coroutine.wrap(function()
        local function process(t)
            for _, v in ipairs(t) do
                if type(v) == "table" then
                    process(v)
                else
                    coroutine.yield(v)
                end
            end
        end
        process(arr)
    end)
end

-- ทดสอบ
print("=== Flatten Iterator ===\n")

local nested = {1, {2, 3}, {4, {5, 6}}, 7, {8, {9, {10}}}}

print("deepFlatten:")
for v in flattenDeep(nested) do io.write(v .. " ") end
print()

-- Flatten array of arrays
local function arrOfArr(t)
    local i = 0
    return function() i = i + 1; return t[i] end
end

local matrix = {{1,2,3}, {4,5,6}, {7,8,9}}
print("\nflatten matrix:")
for v in flatten1(arrOfArr(matrix)) do io.write(v .. " ") end
print()

-- flatten strings
local words = {{"hello", "world"}, {"foo", "bar"}, {"baz"}}
print("\nflatten strings:")
for v in flatten1(arrOfArr(words)) do io.write(v .. " ") end
print()
```

---

## 25.9 Unique Iterator

### ตัวอย่างที่ 13: Unique / Distinct Iterator

```lua
-- Unique: คืนค่าที่ไม่ซ้ำกัน

-- Unique ด้วย set
local function unique(gen)
    local seen = {}
    return function()
        while true do
            local v = gen()
            if v == nil then return nil end
            local key = tostring(v)
            if not seen[key] then
                seen[key] = true
                return v
            end
        end
    end
end

-- UniqueBy: unique ตาม key function
local function uniqueBy(gen, keyFn)
    local seen = {}
    return function()
        while true do
            local v = gen()
            if v == nil then return nil end
            local key = tostring(keyFn(v))
            if not seen[key] then
                seen[key] = true
                return v
            end
        end
    end
end

-- Consecutive unique (เหมือน Unix uniq)
local function consecutiveUnique(gen)
    local prev = {}  -- sentinel
    local NONE = {}
    prev = NONE
    
    return function()
        while true do
            local v = gen()
            if v == nil then return nil end
            if v ~= prev then
                prev = v
                return v
            end
        end
    end
end

-- สร้าง array generator
local function arr(t)
    local i = 0
    return function() i = i + 1; return t[i] end
end

-- ทดสอบ
print("=== Unique Iterator ===\n")

local withDups = arr({1, 2, 3, 2, 4, 1, 5, 3, 6})
print("unique:")
for v in unique(withDups) do io.write(v .. " ") end
print()

local consecutive = arr({1, 1, 2, 2, 2, 3, 1, 1, 4})
print("\nconsecutiveUnique:")
for v in consecutiveUnique(consecutive) do io.write(v .. " ") end
print()

-- uniqueBy สำหรับ objects
local people = arr({
    {name="Alice", dept="IT"},
    {name="Bob", dept="Finance"},
    {name="Charlie", dept="IT"},
    {name="Diana", dept="Management"},
    {name="Eve", dept="Finance"}
})

print("\nuniqueBy department:")
for p in uniqueBy(people, function(p) return p.dept end) do
    print(string.format("  %s (%s)", p.name, p.dept))
end
```

---

## 25.10 Batch/Chunk Iterator

### ตัวอย่างที่ 14: Batch/Chunk Iterator

```lua
-- Chunk/Batch: แบ่งเป็นกลุ่มๆ

-- Chunk: แบ่ง n ตัวต่อกลุ่ม
local function chunk(gen, size)
    return function()
        local batch = {}
        for i = 1, size do
            local v = gen()
            if v == nil then break end
            table.insert(batch, v)
        end
        if #batch == 0 then return nil end
        return batch
    end
end

-- ChunkWhile: แบ่งตาม condition
local function chunkWhile(gen, pred)
    local buf = {}
    local nextVal = nil
    local done = false
    
    return function()
        if done then return nil end
        
        -- ถ้ายังไม่มีค่าใน buffer
        if #buf == 0 then
            local v = gen()
            if v == nil then return nil end
            table.insert(buf, v)
        end
        
        -- เก็บค่าที่ต่อเนื่องกันตาม predicate
        while true do
            local v = gen()
            if v == nil then
                done = true
                local result = buf
                buf = {}
                return result
            end
            
            if pred(buf[#buf], v) then
                table.insert(buf, v)
            else
                local result = buf
                buf = {v}
                return result
            end
        end
    end
end

-- Sliding window
local function slidingWindow(gen, size, step)
    step = step or 1
    local window = {}
    local firstRun = true
    
    return function()
        if firstRun then
            -- เติม window แรก
            for i = 1, size do
                local v = gen()
                if v == nil then return nil end
                table.insert(window, v)
            end
            firstRun = false
            return {table.unpack(window)}
        else
            -- เลื่อน window
            for i = 1, step do
                table.remove(window, 1)
                local v = gen()
                if v == nil then
                    if #window > 0 then
                        return nil
                    end
                    return nil
                end
                table.insert(window, v)
            end
            return {table.unpack(window)}
        end
    end
end

-- สร้าง array generator
local function arr(t)
    local i = 0
    return function() i = i + 1; return t[i] end
end

-- ทดสอบ
print("=== Batch/Chunk Iterator ===\n")

local data = arr({1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
print("chunk(size=3):")
for batch in chunk(data, 3) do
    print("  {" .. table.concat(batch, ", ") .. "}")
end

print("\nchunkWhile (consecutive increasing):")
local data2 = arr({1, 2, 3, 1, 2, 5, 6, 7, 1})
for group in chunkWhile(data2, function(prev, curr) return curr == prev + 1 end) do
    print("  {" .. table.concat(group, ", ") .. "}")
end

print("\nslidingWindow(size=3, step=1):")
local data3 = arr({1, 2, 3, 4, 5, 6, 7})
for w in slidingWindow(data3, 3, 1) do
    print("  {" .. table.concat(w, ", ") .. "}")
end

print("\nslidingWindow(size=4, step=2):")
local data4 = arr({10, 20, 30, 40, 50, 60, 70, 80})
for w in slidingWindow(data4, 4, 2) do
    print("  {" .. table.concat(w, ", ") .. "}")
end
```

---

## 25.11 File Line Iterator

### ตัวอย่างที่ 15: File Iterator

```lua
-- File Line Iterator (จำลอง)

-- สร้าง "file" จาก string (จำลอง)
local function stringLines(content)
    local pos = 1
    return function()
        if pos > #content then return nil end
        local lineEnd = content:find("\n", pos, true)
        local line
        if lineEnd then
            line = content:sub(pos, lineEnd - 1)
            pos = lineEnd + 1
        else
            line = content:sub(pos)
            pos = #content + 1
        end
        return line
    end
end

-- Line iterator พร้อม line numbers
local function numberedLines(content)
    local lines = stringLines(content)
    local lineNum = 0
    return function()
        local line = lines()
        if line == nil then return nil end
        lineNum = lineNum + 1
        return lineNum, line
    end
end

-- Filter lines
local function linesMatching(content, pattern)
    local lines = numberedLines(content)
    return function()
        while true do
            local num, line = lines()
            if num == nil then return nil end
            if line:match(pattern) then
                return num, line
            end
        end
    end
end

-- CSV line parser iterator
local function csvLines(content, hasHeader)
    local lines = stringLines(content)
    local headers = nil
    
    if hasHeader then
        local headerLine = lines()
        if headerLine then
            headers = {}
            for field in (headerLine .. ","):gmatch("([^,]*),") do
                table.insert(headers, field:match("^%s*(.-)%s*$"))
            end
        end
    end
    
    return function()
        local line = lines()
        if line == nil or line == "" then return nil end
        
        local fields = {}
        for field in (line .. ","):gmatch("([^,]*),") do
            table.insert(fields, field:match("^%s*(.-)%s*$"))
        end
        
        if headers then
            local obj = {}
            for i, h in ipairs(headers) do
                obj[h] = fields[i]
            end
            return obj
        end
        
        return fields
    end
end

-- ทดสอบ
print("=== File Line Iterator ===\n")

local sampleText = [[บรรทัดที่ 1: สวัสดี Lua
บรรทัดที่ 2: Coroutines เยี่ยม
บรรทัดที่ 3: Iterators สนุก
บรรทัดที่ 4: OOP ใน Lua
บรรทัดที่ 5: ลาก่อน!]]

print("Numbered lines:")
for num, line in numberedLines(sampleText) do
    print(string.format("  %2d: %s", num, line))
end

print("\nLines matching 'Lua':")
for num, line in linesMatching(sampleText, "Lua") do
    print(string.format("  Line %d: %s", num, line))
end

-- CSV parsing
local csvContent = [[name,age,salary,department
สมชาย,30,50000,IT
สมหญิง,25,45000,Finance
ประสิทธิ์,35,70000,Management
มาลี,28,52000,IT]]

print("\nCSV with header:")
for row in csvLines(csvContent, true) do
    print(string.format("  %-10s age=%-3s salary=%-7s dept=%s",
        row.name, row.age, row.salary, row.department))
end
```

---

## 25.12 Coroutine-based Generators ขั้นสูง

### ตัวอย่างที่ 16-20: Advanced Generators

```lua
-- Generator ที่ใช้ coroutines: เขียนง่ายกว่า closure-based

-- สร้าง generator factory
local function gen(fn)
    return function(...)
        local args = {...}
        return coroutine.wrap(function()
            fn(table.unpack(args))
        end)
    end
end

-- Range generator
local Range = gen(function(start, stop, step)
    step = step or 1
    local i = start
    while (step > 0 and i <= stop) or (step < 0 and i >= stop) do
        coroutine.yield(i)
        i = i + step
    end
end)

-- Repeat generator
local Repeat = gen(function(val, times)
    local count = 0
    while not times or count < times do
        coroutine.yield(val)
        count = count + 1
    end
end)

-- Accumulate (running total)
local Accumulate = gen(function(iterable, fn, initial)
    fn = fn or function(a, b) return a + b end
    local acc = initial
    for v in iterable do
        if acc == nil then
            acc = v
        else
            acc = fn(acc, v)
        end
        coroutine.yield(acc)
    end
end)

-- Product: cartesian product
local Product = gen(function(...)
    local pools = {...}
    
    local function cartesian(pools, index, current)
        if index > #pools then
            coroutine.yield({table.unpack(current)})
            return
        end
        for _, v in ipairs(pools[index]) do
            table.insert(current, v)
            cartesian(pools, index + 1, current)
            table.remove(current)
        end
    end
    
    cartesian(pools, 1, {})
end)

-- Combinations
local Combinations = gen(function(arr, r)
    local n = #arr
    local indices = {}
    for i = 1, r do indices[i] = i end
    
    -- First combination
    local comb = {}
    for _, i in ipairs(indices) do table.insert(comb, arr[i]) end
    coroutine.yield(comb)
    
    while true do
        -- Find rightmost element that can be incremented
        local i = r
        while i >= 1 and indices[i] == i + n - r do
            i = i - 1
        end
        if i < 1 then break end
        
        indices[i] = indices[i] + 1
        for j = i + 1, r do
            indices[j] = indices[j-1] + 1
        end
        
        comb = {}
        for _, idx in ipairs(indices) do
            table.insert(comb, arr[idx])
        end
        coroutine.yield(comb)
    end
end)

-- Permutations
local Permutations = gen(function(arr)
    local n = #arr
    local c = {}
    for i = 1, n do c[i] = 0 end
    
    coroutine.yield({table.unpack(arr)})
    
    local i = 1
    while i <= n do
        if c[i] < i then
            if i % 2 == 0 then
                arr[1], arr[i] = arr[i], arr[1]
            else
                arr[c[i]+1], arr[i] = arr[i], arr[c[i]+1]
            end
            coroutine.yield({table.unpack(arr)})
            c[i] = c[i] + 1
            i = 1
        else
            c[i] = 0
            i = i + 1
        end
    end
end)

-- ทดสอบ
print("=== Advanced Generators ===\n")

print("Range(1, 10, 2):")
for v in Range(1, 10, 2) do io.write(v .. " ") end
print()

print("\nAccumulate (running sum):")
local data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
local iter = (function()
    local i = 0
    return function()
        i = i + 1
        return data[i]
    end
end)()
for v in Accumulate(iter) do io.write(v .. " ") end
print()

print("\nProduct({A,B}, {1,2,3}):")
for combo in Product({"A", "B"}, {1, 2, 3}) do
    print(string.format("  (%s, %d)", combo[1], combo[2]))
end

print("\nCombinations({1,2,3,4}, 2):")
for combo in Combinations({1, 2, 3, 4}, 2) do
    print(string.format("  {%s}", table.concat(combo, ", ")))
end

print("\nPermutations({1,2,3}):")
for perm in Permutations({1, 2, 3}) do
    print(string.format("  {%s}", table.concat(perm, ", ")))
end
```

---

## 25.13 Infinite Generators

### ตัวอย่างที่ 21-25: Infinite Sequences

```lua
-- Infinite generators ที่มีประโยชน์

-- สร้าง infinite gen ง่ายๆ
local function infiniteRange(start, step)
    return coroutine.wrap(function()
        local n = start or 0
        step = step or 1
        while true do
            coroutine.yield(n)
            n = n + step
        end
    end)
end

-- Collatz Sequence (3n+1)
local function collatz(n)
    return coroutine.wrap(function()
        local current = n
        coroutine.yield(current)
        while current ~= 1 do
            if current % 2 == 0 then
                current = current / 2
            else
                current = 3 * current + 1
            end
            coroutine.yield(current)
        end
    end)
end

-- Recaman Sequence
local function recaman()
    return coroutine.wrap(function()
        local seen = {}
        local a = 0
        coroutine.yield(a)
        seen[a] = true
        
        for n = 1, math.huge do
            local candidate = a - n
            if candidate > 0 and not seen[candidate] then
                a = candidate
            else
                a = a + n
            end
            coroutine.yield(a)
            seen[a] = true
        end
    end)
end

-- Triangular Numbers: 1, 3, 6, 10, 15, ...
local function triangular()
    return coroutine.wrap(function()
        local n = 0
        local i = 0
        while true do
            i = i + 1
            n = n + i
            coroutine.yield(n)
        end
    end)
end

-- Catalan Numbers
local function catalan()
    return coroutine.wrap(function()
        local function binomial(n, k)
            if k > n - k then k = n - k end
            local result = 1
            for i = 0, k - 1 do
                result = result * (n - i) / (i + 1)
            end
            return math.floor(result + 0.5)
        end
        
        for n = 0, math.huge do
            local c = binomial(2*n, n) / (n + 1)
            coroutine.yield(math.floor(c + 0.5))
        end
    end)
end

-- Thue-Morse Sequence
local function thueMorse()
    return coroutine.wrap(function()
        local n = 0
        while true do
            -- นับจำนวน 1 bits
            local x = n
            local count = 0
            while x > 0 do
                count = count + (x & 1)
                x = x >> 1
            end
            coroutine.yield(count % 2)
            n = n + 1
        end
    end)
end

-- Helper: take n items from generator
local function take(gen, n)
    local result = {}
    for i = 1, n do
        local v = gen()
        if v == nil then break end
        table.insert(result, v)
    end
    return result
end

-- ทดสอบ
print("=== Infinite Generators ===\n")

print("Collatz(27):")
local col = {}
for v in collatz(27) do table.insert(col, v) end
print("Length:", #col)
print("Sequence:", table.concat({col[1], "...", col[#col]}, " "))

print("\nRecaman (first 15):")
print(table.concat(take(recaman(), 15), ", "))

print("\nTriangular (first 12):")
print(table.concat(take(triangular(), 12), ", "))

print("\nCatalan (first 10):")
print(table.concat(take(catalan(), 10), ", "))

print("\nThue-Morse (first 20):")
print(table.concat(take(thueMorse(), 20), ""))

-- Sieve of Eratosthenes (infinite)
local function sieve()
    return coroutine.wrap(function()
        -- ใช้ incremental sieve
        local composites = {}
        local n = 2
        
        while true do
            if not composites[n] then
                coroutine.yield(n)
                composites[n * n] = composites[n * n] or {}
                table.insert(composites[n * n], n)
            else
                for _, p in ipairs(composites[n]) do
                    local next = n + p
                    composites[next] = composites[next] or {}
                    table.insert(composites[next], p)
                end
                composites[n] = nil
            end
            n = n + 1
        end
    end)
end

print("\nPrimes (first 25 using sieve):")
print(table.concat(take(sieve(), 25), ", "))
```

---

## 25.14 Iterator Utilities Library

### ตัวอย่างที่ 26-35: Complete Iterator Library

```lua
-- Iterator Utilities Library ที่สมบูรณ์

local Iter = {}

-- ===== Constructors =====

function Iter.from(t)
    if type(t) == "function" then
        return t
    end
    local i = 0
    return function()
        i = i + 1
        return t[i]
    end
end

function Iter.range(start, stop, step)
    step = step or 1
    local i = start - step
    return function()
        i = i + step
        if (step > 0 and i <= stop) or (step < 0 and i >= stop) then
            return i
        end
    end
end

function Iter.count(from, step)
    from = from or 0
    step = step or 1
    local n = from - step
    return function()
        n = n + step
        return n
    end
end

function Iter.repeat_(val, times)
    local count = 0
    return function()
        if times and count >= times then return nil end
        count = count + 1
        return val
    end
end

function Iter.cycle(t, times)
    local i = 0
    local rep = 0
    return function()
        if times and rep >= times then return nil end
        i = i + 1
        if i > #t then
            i = 1
            rep = rep + 1
            if times and rep >= times then return nil end
        end
        return t[i]
    end
end

-- ===== Transformers =====

function Iter.map(gen, fn)
    return function()
        local v = gen()
        if v == nil then return nil end
        return fn(v)
    end
end

function Iter.filter(gen, pred)
    return function()
        while true do
            local v = gen()
            if v == nil then return nil end
            if pred(v) then return v end
        end
    end
end

function Iter.flatMap(gen, fn)
    local inner = nil
    return function()
        while true do
            if inner then
                local v = inner()
                if v ~= nil then return v end
                inner = nil
            end
            local v = gen()
            if v == nil then return nil end
            inner = Iter.from(fn(v))
        end
    end
end

function Iter.take(gen, n)
    local count = 0
    return function()
        if count >= n then return nil end
        local v = gen()
        if v == nil then return nil end
        count = count + 1
        return v
    end
end

function Iter.drop(gen, n)
    for i = 1, n do gen() end
    return gen
end

function Iter.takeWhile(gen, pred)
    local done = false
    return function()
        if done then return nil end
        local v = gen()
        if v == nil or not pred(v) then
            done = true
            return nil
        end
        return v
    end
end

function Iter.dropWhile(gen, pred)
    local dropping = true
    return function()
        while true do
            local v = gen()
            if v == nil then return nil end
            if dropping then
                if not pred(v) then
                    dropping = false
                    return v
                end
            else
                return v
            end
        end
    end
end

function Iter.unique(gen, keyFn)
    local seen = {}
    return function()
        while true do
            local v = gen()
            if v == nil then return nil end
            local key = tostring(keyFn and keyFn(v) or v)
            if not seen[key] then
                seen[key] = true
                return v
            end
        end
    end
end

function Iter.zip(...)
    local gens = {...}
    return function()
        local values = {}
        for _, g in ipairs(gens) do
            local v = g()
            if v == nil then return nil end
            table.insert(values, v)
        end
        return table.unpack(values)
    end
end

function Iter.chain(...)
    local gens = {...}
    local i = 1
    return function()
        while i <= #gens do
            local v = gens[i]()
            if v ~= nil then return v end
            i = i + 1
        end
        return nil
    end
end

function Iter.enumerate(gen, start)
    start = start or 1
    local i = start - 1
    return function()
        local v = gen()
        if v == nil then return nil end
        i = i + 1
        return i, v
    end
end

function Iter.chunk(gen, size)
    return function()
        local batch = {}
        for _ = 1, size do
            local v = gen()
            if v == nil then break end
            table.insert(batch, v)
        end
        if #batch == 0 then return nil end
        return batch
    end
end

-- ===== Terminators =====

function Iter.toArray(gen)
    local result = {}
    for v in gen do table.insert(result, v) end
    return result
end

function Iter.toSet(gen, keyFn)
    local result = {}
    for v in gen do
        local key = keyFn and keyFn(v) or v
        result[key] = v
    end
    return result
end

function Iter.reduce(gen, fn, initial)
    local acc = initial
    for v in gen do
        if acc == nil then acc = v
        else acc = fn(acc, v) end
    end
    return acc
end

function Iter.sum(gen)
    return Iter.reduce(gen, function(a, b) return a + b end, 0)
end

function Iter.product_(gen)
    return Iter.reduce(gen, function(a, b) return a * b end, 1)
end

function Iter.min(gen, keyFn)
    local minVal, minKey
    for v in gen do
        local key = keyFn and keyFn(v) or v
        if minKey == nil or key < minKey then
            minVal, minKey = v, key
        end
    end
    return minVal
end

function Iter.max(gen, keyFn)
    local maxVal, maxKey
    for v in gen do
        local key = keyFn and keyFn(v) or v
        if maxKey == nil or key > maxKey then
            maxVal, maxKey = v, key
        end
    end
    return maxVal
end

function Iter.count_(gen)
    local n = 0
    for _ in gen do n = n + 1 end
    return n
end

function Iter.any(gen, pred)
    for v in gen do
        if pred(v) then return true end
    end
    return false
end

function Iter.all(gen, pred)
    for v in gen do
        if not pred(v) then return false end
    end
    return true
end

function Iter.first(gen)
    return gen()
end

function Iter.last(gen)
    local last = nil
    for v in gen do last = v end
    return last
end

function Iter.nth(gen, n)
    local count = 0
    for v in gen do
        count = count + 1
        if count == n then return v end
    end
    return nil
end

function Iter.groupBy(gen, keyFn)
    local groups = {}
    local order = {}
    for v in gen do
        local key = tostring(keyFn(v))
        if not groups[key] then
            groups[key] = {}
            table.insert(order, key)
        end
        table.insert(groups[key], v)
    end
    return groups, order
end

function Iter.forEach(gen, fn)
    for v in gen do fn(v) end
end

-- ===== ทดสอบ Library =====
print("=== Iter Library Demo ===\n")

local data = {5, 3, 8, 1, 9, 2, 7, 4, 6, 10}

-- Complex query: หา top-3 even numbers squared, sorted
local result = Iter.toArray(
    Iter.take(
        Iter.map(
            Iter.filter(
                Iter.from({table.unpack(data)}),
                function(x) return x % 2 == 0 end
            ),
            function(x) return x * x end
        ),
        3
    )
)

print("Top 3 even squares (from order):", table.concat(result, ", "))

-- Sum of odd numbers in range 1-100
local oddSum = Iter.sum(
    Iter.filter(
        Iter.range(1, 100),
        function(x) return x % 2 ~= 0 end
    )
)
print("Sum of odds 1-100:", oddSum)

-- GroupBy
local people = {
    {name="Alice", dept="IT", salary=50000},
    {name="Bob", dept="Finance", salary=60000},
    {name="Charlie", dept="IT", salary=55000},
    {name="Diana", dept="Management", salary=90000},
    {name="Eve", dept="Finance", salary=65000}
}

local groups, order = Iter.groupBy(
    Iter.from(people),
    function(p) return p.dept end
)

print("\nGroupBy department:")
for _, dept in ipairs(order) do
    local members = groups[dept]
    local names = {}
    for _, p in ipairs(members) do table.insert(names, p.name) end
    print(string.format("  %s: %s", dept, table.concat(names, ", ")))
end

-- Statistics
local salaries = Iter.map(Iter.from(people), function(p) return p.salary end)
-- เนื่องจาก gen ใช้ได้ครั้งเดียว ต้องสร้างใหม่ทุกครั้ง
local function getSalaries()
    return Iter.map(Iter.from(people), function(p) return p.salary end)
end

print(string.format("\nSalary stats:"))
print(string.format("  Min: %.0f", Iter.min(getSalaries())))
print(string.format("  Max: %.0f", Iter.max(getSalaries())))
print(string.format("  Sum: %.0f", Iter.sum(getSalaries())))
print(string.format("  Count: %d", Iter.count_(getSalaries())))
local total = Iter.sum(getSalaries())
local count = Iter.count_(getSalaries())
print(string.format("  Avg: %.0f", total / count))
```

---

## 25.15 Iterator ขั้นสูง: Lazy Collections

### ตัวอย่างที่ 36-40: Lazy Collection

```lua
-- Lazy Collection: คล้าย Stream ใน Java/Kotlin

local LazySeq = {}
LazySeq.__index = LazySeq

function LazySeq.of(...)
    local items = {...}
    return LazySeq.new(Iter.from(items))
end

function LazySeq.from(source)
    if type(source) == "table" then
        return LazySeq.new(Iter.from(source))
    end
    return LazySeq.new(source)
end

function LazySeq.range(start, stop, step)
    return LazySeq.new(Iter.range(start, stop, step))
end

function LazySeq.new(gen)
    return setmetatable({_gen = gen}, LazySeq)
end

-- Transformers (lazy)
function LazySeq:map(fn)
    return LazySeq.new(Iter.map(self._gen, fn))
end

function LazySeq:filter(pred)
    return LazySeq.new(Iter.filter(self._gen, pred))
end

function LazySeq:take(n)
    return LazySeq.new(Iter.take(self._gen, n))
end

function LazySeq:drop(n)
    return LazySeq.new(Iter.drop(self._gen, n))
end

function LazySeq:takeWhile(pred)
    return LazySeq.new(Iter.takeWhile(self._gen, pred))
end

function LazySeq:dropWhile(pred)
    return LazySeq.new(Iter.dropWhile(self._gen, pred))
end

function LazySeq:unique(keyFn)
    return LazySeq.new(Iter.unique(self._gen, keyFn))
end

function LazySeq:flatMap(fn)
    return LazySeq.new(Iter.flatMap(self._gen, fn))
end

function LazySeq:zip(other)
    return LazySeq.new(Iter.zip(self._gen, other._gen or other))
end

function LazySeq:enumerate(start)
    return LazySeq.new(Iter.enumerate(self._gen, start))
end

function LazySeq:chunk(size)
    return LazySeq.new(Iter.chunk(self._gen, size))
end

-- Terminators (eager)
function LazySeq:toArray()
    return Iter.toArray(self._gen)
end

function LazySeq:sum()
    return Iter.sum(self._gen)
end

function LazySeq:count()
    return Iter.count_(self._gen)
end

function LazySeq:reduce(fn, initial)
    return Iter.reduce(self._gen, fn, initial)
end

function LazySeq:min(keyFn)
    return Iter.min(self._gen, keyFn)
end

function LazySeq:max(keyFn)
    return Iter.max(self._gen, keyFn)
end

function LazySeq:any(pred)
    return Iter.any(self._gen, pred)
end

function LazySeq:all(pred)
    return Iter.all(self._gen, pred)
end

function LazySeq:first()
    return Iter.first(self._gen)
end

function LazySeq:last()
    return Iter.last(self._gen)
end

function LazySeq:nth(n)
    return Iter.nth(self._gen, n)
end

function LazySeq:forEach(fn)
    Iter.forEach(self._gen, fn)
end

function LazySeq:groupBy(keyFn)
    return Iter.groupBy(self._gen, keyFn)
end

-- ทำให้ใช้ใน for-in ได้
function LazySeq:iter()
    return self._gen
end

-- ทดสอบ LazySeq
print("=== Lazy Collection Demo ===\n")

-- หา 5 prime แรกที่มากกว่า 100
local function isPrime(n)
    if n < 2 then return false end
    if n == 2 then return true end
    if n % 2 == 0 then return false end
    for i = 3, math.sqrt(n), 2 do
        if n % i == 0 then return false end
    end
    return true
end

local first5BigPrimes = LazySeq.range(101, 10000)
    :filter(isPrime)
    :take(5)
    :toArray()

print("5 primes > 100:", table.concat(first5BigPrimes, ", "))

-- Word frequency analysis
local text = "lua is great lua is powerful lua programming lua coroutines are fun"
local words = {}
for word in text:gmatch("%S+") do table.insert(words, word) end

local wordFreq = LazySeq.from(words)
    :reduce(function(acc, word)
        acc[word] = (acc[word] or 0) + 1
        return acc
    end, {})

print("\nWord frequency:")
local wordList = {}
for word, count in pairs(wordFreq) do
    table.insert(wordList, {word=word, count=count})
end
table.sort(wordList, function(a, b) return a.count > b.count end)

for _, wc in ipairs(wordList) do
    print(string.format("  %-15s: %d", wc.word, wc.count))
end

-- Fibonacci primes (lazy)
local function fibGen()
    return coroutine.wrap(function()
        local a, b = 0, 1
        while true do
            coroutine.yield(a)
            a, b = b, a + b
        end
    end)
end

print("\nFibonacci primes < 1000:")
local fibPrimes = LazySeq.new(fibGen())
    :filter(function(x) return x >= 2 and isPrime(x) end)
    :takeWhile(function(x) return x < 1000 end)
    :toArray()
print(table.concat(fibPrimes, ", "))

-- Matrix operations
local matrix = {{1,2,3}, {4,5,6}, {7,8,9}}

print("\nMatrix flatten and sum:")
local matSum = LazySeq.from(matrix)
    :flatMap(function(row)
        return row  -- คืน array ที่จะถูก flatten
    end)
    :sum()
print("Sum:", matSum)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Iterator Library Extensions
ขยาย Iterator library ด้วย:
- `Iter.scan(gen, fn, initial)`: running aggregate
- `Iter.pairwise(gen)`: {v1,v2}, {v2,v3}, {v3,v4}, ...
- `Iter.rotate(gen, n)`: เลื่อน n ตำแหน่ง
- `Iter.tee(gen, n)`: แยก generator เป็น n copies
- `Iter.starMap(gen, fn)`: map ด้วย unpacked args

### แบบฝึกหัดที่ 2: Custom for-in Protocol
สร้าง custom iterator สำหรับ:
- `bitsOf(n)`: iterate bits ของตัวเลข (MSB ไป LSB)
- `digitsOf(n, base)`: iterate digits ในฐาน base
- `wordsOf(str)`: iterate คำในประโยค
- `sentencesOf(str)`: iterate ประโยคใน paragraph

### แบบฝึกหัดที่ 3: Reactive Iterator
สร้าง reactive system ที่:
- `observable(gen)`: monitor changes ใน iterator
- `debounce(gen, ms)`: รอหลังจาก last event
- `throttle(gen, ms)`: จำกัด rate
- `buffer(gen, time)`: รวม events ในช่วงเวลา

### แบบฝึกหัดที่ 4: SQL-like Query Builder
สร้าง query builder สำหรับ Lua tables:
```lua
local results = Query.from(people)
    :where(function(p) return p.age > 25 end)
    :select(function(p) return {name=p.name, salary=p.salary} end)
    :orderBy(function(p) return p.salary end, "desc")
    :limit(5)
    :toArray()
```

### แบบฝึกหัดที่ 5: Iterator Protocol สำหรับ Custom Data Structure
สร้าง data structures ที่รองรับ for-in:
- Linked List (forward และ backward)
- Binary Search Tree (in-order, pre-order, post-order)
- Graph (DFS, BFS)
- Trie (words ที่มี prefix)

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Generic for Protocol**: `iter, state, control = explist` และการทำงาน
2. **Stateless Iterator**: ทำงานด้วย state และ control variable ไม่เก็บ state เอง
3. **Stateful Iterator**: ใช้ closure เก็บ state ภายใน
4. **Custom pairs/ipairs**: สร้าง iterators ที่ยืดหยุ่นกว่า built-in
5. **Filter Iterator**: กรองค่าตาม predicate
6. **Map Iterator**: แปลงค่าแต่ละตัว รวมถึง flatMap
7. **Zip Iterator**: รวม iterators หลายตัวเป็นคู่ หรือ tuple
8. **Chain Iterator**: ต่อ iterators เรียงต่อกัน
9. **Take/Drop**: จำกัดหรือข้ามค่า รวมถึง while variants
10. **Flatten**: ทำ nested structure ให้ราบ
11. **Unique**: กรองค่าซ้ำออก
12. **Chunk/Batch**: แบ่งเป็นกลุ่ม รวมถึง sliding window
13. **File Iterators**: อ่านข้อมูลแบบ line-by-line หรือ CSV
14. **Coroutine Generators**: เขียน iterators แบบง่ายด้วย coroutines
15. **Infinite Sequences**: Fibonacci, Primes, Collatz, ฯลฯ
16. **Iterator Utilities Library**: library ครบถ้วนสำหรับ functional programming
17. **Lazy Collections**: method chaining ด้วย lazy evaluation

> **เคล็ดลับ**: Iterators เป็นหัวใจของ functional programming ใน Lua การเข้าใจ generic for protocol ทำให้เขียนโค้ดที่หรูหราและมีประสิทธิภาพได้ การใช้ coroutines สำหรับ generators ทำให้โค้ดอ่านง่ายกว่าการเขียน closure โดยตรง
