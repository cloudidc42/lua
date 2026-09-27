# บทที่ 24: Coroutines

## บทนำ

Coroutines เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ Lua ช่วยให้เราเขียนโค้ดที่ทำงานแบบ "cooperative multitasking" ได้ นั่นคือ function หลายตัวสามารถทำงานสลับกันได้โดยไม่ต้องใช้ threads จริงๆ

Coroutine แตกต่างจาก function ทั่วไปตรงที่สามารถ **หยุดชั่วคราว** กลางการทำงานและ **กลับมาทำงานต่อ** ได้ในภายหลัง

ในบทนี้เราจะเรียนรู้:
- `coroutine.create()` - สร้าง coroutine
- `coroutine.resume()` - เริ่ม/ต่อ coroutine
- `coroutine.yield()` - หยุดชั่วคราว
- `coroutine.status()` - ตรวจสอบสถานะ
- `coroutine.wrap()` - สร้าง coroutine แบบ simplified
- การส่งค่าระหว่าง resume และ yield
- Producer-Consumer Pattern
- Generator Pattern
- Coroutine Scheduler

---

## 24.1 Coroutine พื้นฐาน

### ตัวอย่างที่ 1: coroutine.create และ coroutine.resume

```lua
-- สร้าง coroutine อย่างง่าย
local function greeting()
    print("สวัสดี!")
    print("ฉันคือ coroutine")
    print("ลาก่อน!")
end

-- สร้าง coroutine (ยังไม่เริ่มทำงาน)
local co = coroutine.create(greeting)

print("ประเภท:", type(co))        -- thread
print("สถานะ:", coroutine.status(co))  -- suspended

-- เริ่มทำงาน
coroutine.resume(co)

print("สถานะหลัง resume:", coroutine.status(co))  -- dead
```

### ตัวอย่างที่ 2: coroutine.yield พื้นฐาน

```lua
-- Coroutine ที่หยุดชั่วคราวได้
local function countTo5()
    for i = 1, 5 do
        print("นับ:", i)
        coroutine.yield()  -- หยุดชั่วคราว ส่งคืนการควบคุมให้ผู้เรียก
    end
    print("นับเสร็จแล้ว!")
end

local co = coroutine.create(countTo5)

print("=== เริ่ม coroutine ===")
print("Resume 1:")
coroutine.resume(co)  -- ทำงานจนถึง yield แรก

print("Resume 2:")
coroutine.resume(co)  -- ทำงานต่อจาก yield แรก ถึง yield ที่สอง

print("Resume 3-6:")
for i = 3, 6 do
    print("Resume " .. i .. ":")
    local ok, val = coroutine.resume(co)
    print("  ok=" .. tostring(ok) .. ", status=" .. coroutine.status(co))
end
```

### ตัวอย่างที่ 3: coroutine.status ทั้ง 4 สถานะ

```lua
-- ทำความเข้าใจ 4 สถานะของ coroutine
-- 1. suspended: หยุดรอ resume (เพิ่งสร้างหรือหลัง yield)
-- 2. running: กำลังทำงานอยู่
-- 3. normal: coroutine ที่ resume coroutine อื่นอยู่
-- 4. dead: ทำงานเสร็จแล้ว หรือเกิด error

local coA, coB

coB = coroutine.create(function()
    print("coB: สถานะ coA ขณะ coB ทำงาน:", coroutine.status(coA))
    coroutine.yield()
    print("coB: ทำงานต่อ")
end)

coA = coroutine.create(function()
    print("coA: สถานะก่อน resume coB:", coroutine.status(coB))
    coroutine.resume(coB)  -- coA จะอยู่ในสถานะ "normal" ขณะที่ coB ทำงาน
    print("coA: กลับมาหลัง resume coB")
    print("coA: สถานะ coB ตอนนี้:", coroutine.status(coB))
end)

print("ก่อน resume coA:", coroutine.status(coA))
coroutine.resume(coA)
print("หลัง resume coA (ครั้งแรก):")
print("  coA status:", coroutine.status(coA))
print("  coB status:", coroutine.status(coB))

coroutine.resume(coB)  -- ต่อ coB
coroutine.resume(coA)  -- ต่อ coA ถ้ายังไม่ dead

print("\nสถานะสุดท้าย:")
print("  coA:", coroutine.status(coA))
print("  coB:", coroutine.status(coB))
```

---

## 24.2 การส่งค่าระหว่าง Resume และ Yield

### ตัวอย่างที่ 4: ส่งค่าจาก resume ไปยัง coroutine

```lua
-- resume สามารถส่งค่าให้ coroutine ได้
local function calculator()
    print("Calculator พร้อมแล้ว")
    
    while true do
        -- yield คืนค่าให้ resume แล้วรับค่าใหม่จาก resume ถัดไป
        local operation, a, b = coroutine.yield("พร้อมรับคำสั่ง")
        
        if operation == "quit" then
            print("Calculator ปิดแล้ว")
            return "bye"
        end
        
        local result
        if operation == "add" then result = a + b
        elseif operation == "sub" then result = a - b
        elseif operation == "mul" then result = a * b
        elseif operation == "div" then
            if b == 0 then result = "error: หารด้วยศูนย์"
            else result = a / b end
        end
        
        print(string.format("  %s(%d, %d) = %s", operation, a, b, tostring(result)))
        coroutine.yield(result)
    end
end

local co = coroutine.create(calculator)

-- เริ่ม coroutine
local ok, msg = coroutine.resume(co)
print("Coroutine ตอบ:", msg)

-- ส่งคำสั่ง
ok, msg = coroutine.resume(co, "add", 10, 5)
print("หลัง yield:", msg)

ok, msg = coroutine.resume(co, "พร้อม")  -- รับ "พร้อมรับคำสั่ง"
print("Status:", msg)

ok, msg = coroutine.resume(co, "mul", 4, 7)
print("หลัง yield:", msg)

ok, msg = coroutine.resume(co, "พร้อม")

ok, msg = coroutine.resume(co, "div", 10, 0)
print("หลัง yield:", msg)

ok, msg = coroutine.resume(co, "ต่อไป")

ok, msg = coroutine.resume(co, "quit")
print("สุดท้าย:", msg)
```

### ตัวอย่างที่ 5: ส่งค่าจาก yield กลับไปยังผู้เรียก

```lua
-- yield ส่งค่ากลับให้ผู้เรียก (ค่าจาก resume)
-- resume รับค่าที่ yield ส่งมา

local function fibonacci()
    local a, b = 0, 1
    while true do
        coroutine.yield(a)  -- yield ส่ง a กลับให้ resume
        a, b = b, a + b
    end
end

local fib = coroutine.create(fibonacci)

print("Fibonacci sequence:")
for i = 1, 15 do
    local ok, value = coroutine.resume(fib)
    io.write(value .. " ")
end
print()
```

### ตัวอย่างที่ 6: Two-way Communication

```lua
-- การสื่อสารสองทาง: ส่งไป-รับกลับ
local function pingPong()
    local count = 0
    while true do
        count = count + 1
        local received = coroutine.yield("pong #" .. count)
        print("  pingPong ได้รับ:", received)
    end
end

local co = coroutine.create(pingPong)

-- เริ่ม coroutine
print("=== Ping-Pong ===")
local ok, response = coroutine.resume(co)  -- เริ่มทำงาน
print("ได้รับ:", response)

-- ส่ง ping รับ pong
for i = 1, 5 do
    ok, response = coroutine.resume(co, "ping #" .. i)
    print("ได้รับ:", response)
end
```

---

## 24.3 coroutine.wrap()

### ตัวอย่างที่ 7: coroutine.wrap พื้นฐาน

```lua
-- coroutine.wrap ทำให้ใช้งานง่ายขึ้น
-- แทนที่จะเรียก coroutine.resume(co, ...) ใช้แค่ wrap_fn(...)

local function range(start, stop, step)
    step = step or 1
    local i = start
    while (step > 0 and i <= stop) or (step < 0 and i >= stop) do
        coroutine.yield(i)
        i = i + step
    end
end

-- ใช้ coroutine.wrap
local rangeIter = coroutine.wrap(function()
    return range(1, 10, 2)
end)

print("Odd numbers 1-10:")
for i = 1, 5 do
    print(rangeIter())  -- เรียกได้เหมือน function ปกติ
end

-- สร้าง range generator factory
local function makeRange(from, to, step)
    return coroutine.wrap(function()
        local current = from
        step = step or 1
        while (step > 0 and current <= to) or (step < 0 and current >= to) do
            coroutine.yield(current)
            current = current + step
        end
    end)
end

print("\nRange 0 to 20 step 5:")
local r = makeRange(0, 20, 5)
while true do
    local val = r()
    if val == nil then break end
    io.write(val .. " ")
end
print()

print("\nCountdown 10 to 1:")
local countdown = makeRange(10, 1, -1)
while true do
    local val = countdown()
    if val == nil then break end
    io.write(val .. " ")
end
print()
```

### ตัวอย่างที่ 8: wrap ใน for-in loop

```lua
-- ใช้ coroutine.wrap กับ for-in
local function permutations(arr)
    local n = #arr
    local c = {}
    for i = 1, n do c[i] = 0 end
    
    coroutine.yield(arr)  -- yield ชุดแรก
    
    local i = 1
    while i <= n do
        if c[i] < i then
            if i % 2 == 0 then
                arr[1], arr[i] = arr[i], arr[1]
            else
                arr[c[i]+1], arr[i] = arr[i], arr[c[i]+1]
            end
            
            -- copy array
            local perm = {}
            for _, v in ipairs(arr) do table.insert(perm, v) end
            coroutine.yield(perm)
            
            c[i] = c[i] + 1
            i = 1
        else
            c[i] = 0
            i = i + 1
        end
    end
end

-- สร้าง iterator ด้วย wrap
local function perms(arr)
    return coroutine.wrap(function()
        permutations(arr)
    end)
end

print("Permutations of {1, 2, 3}:")
local count = 0
for perm in perms({1, 2, 3}) do
    count = count + 1
    print(string.format("  %d: {%s}", count, table.concat(perm, ", ")))
end
print("จำนวน:", count)
```

---

## 24.4 Producer-Consumer Pattern

### ตัวอย่างที่ 9: Producer-Consumer พื้นฐาน

```lua
-- Classic Producer-Consumer Pattern ด้วย Coroutines

-- Producer: สร้างข้อมูล
local function producer()
    local items = {"แอปเปิ้ล", "กล้วย", "ส้ม", "องุ่น", "มะม่วง"}
    
    for _, item in ipairs(items) do
        print("[Producer] ผลิต: " .. item)
        coroutine.yield(item)  -- ส่งให้ consumer
    end
    
    print("[Producer] ผลิตเสร็จแล้ว")
    coroutine.yield(nil)  -- สัญญาณหยุด
end

-- Consumer: ใช้ข้อมูล
local function consumer(producerCo)
    while true do
        -- ขอข้อมูลจาก producer
        local ok, item = coroutine.resume(producerCo)
        
        if not ok or item == nil then
            print("[Consumer] ไม่มีข้อมูลแล้ว หยุดทำงาน")
            break
        end
        
        print("[Consumer] ใช้: " .. item)
    end
end

print("=== Producer-Consumer Pattern ===\n")
local prod = coroutine.create(producer)
consumer(prod)
```

### ตัวอย่างที่ 10: Pipeline ด้วย Coroutines

```lua
-- Pipeline: ต่อ coroutines เป็น chain

-- สร้าง pipeline stage
local function makeStage(name, transform)
    return function(input)
        return coroutine.wrap(function()
            for item in input do
                local result = transform(item)
                if result ~= nil then
                    print(string.format("  [%s] %s -> %s", name, 
                        tostring(item), tostring(result)))
                    coroutine.yield(result)
                end
            end
        end)
    end
end

-- Source: สร้างข้อมูลต้นทาง
local function source(data)
    return coroutine.wrap(function()
        for _, v in ipairs(data) do
            coroutine.yield(v)
        end
    end)
end

-- Stages
local double = makeStage("double", function(x) return x * 2 end)
local addTen = makeStage("addTen", function(x) return x + 10 end)
local filterEven = makeStage("filterEven", function(x)
    return x % 2 == 0 and x or nil
end)
local toString = makeStage("toString", function(x)
    return "value=" .. x
end)

-- สร้าง pipeline
print("=== Pipeline Demo ===")
local data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

-- Chain: source -> double -> addTen -> filterEven -> toString
local pipeline = toString(filterEven(addTen(double(source(data)))))

print("\nผลลัพธ์:")
for result in pipeline do
    print("  " .. result)
end
```

### ตัวอย่างที่ 11: Data Stream Processing

```lua
-- การประมวลผล data stream ด้วย coroutines

-- Stream generator
local function csvStream(rows)
    return coroutine.wrap(function()
        for _, row in ipairs(rows) do
            coroutine.yield(row)
        end
    end)
end

-- Parse CSV row
local function parseCSV(stream)
    return coroutine.wrap(function()
        for line in stream do
            local fields = {}
            for field in (line .. ","):gmatch("([^,]*),") do
                table.insert(fields, field:match("^%s*(.-)%s*$"))
            end
            coroutine.yield(fields)
        end
    end)
end

-- Filter stage
local function filterStage(stream, predicate)
    return coroutine.wrap(function()
        for item in stream do
            if predicate(item) then
                coroutine.yield(item)
            end
        end
    end)
end

-- Map stage
local function mapStage(stream, transform)
    return coroutine.wrap(function()
        for item in stream do
            coroutine.yield(transform(item))
        end
    end)
end

-- Reduce (terminal)
local function reduce(stream, fn, initial)
    local acc = initial
    for item in stream do
        acc = fn(acc, item)
    end
    return acc
end

-- ทดสอบ Stream Processing
print("=== Stream Processing Demo ===\n")

local csvData = {
    "สมชาย,25,50000,IT",
    "สมหญิง,30,65000,Finance",
    "ประสิทธิ์,22,40000,IT",
    "วิชัย,35,80000,Management",
    "มาลี,28,55000,IT",
    "ประยุทธ์,45,120000,Management",
    "สุดา,26,48000,Finance"
}

-- Pipeline:
-- 1. อ่าน CSV
-- 2. Parse เป็น fields
-- 3. Filter เฉพาะ IT department
-- 4. Map เป็น object
-- 5. Reduce หาเงินเดือนเฉลี่ย

local stream = csvStream(csvData)
local parsed = parseCSV(stream)

local itOnly = filterStage(parsed, function(fields)
    return fields[4] == "IT"
end)

local asObject = mapStage(itOnly, function(fields)
    return {
        name = fields[1],
        age = tonumber(fields[2]),
        salary = tonumber(fields[3]),
        dept = fields[4]
    }
end)

print("พนักงาน IT:")
local itEmployees = {}

-- ต้องเก็บใน table ก่อน เพราะ stream ใช้ได้ครั้งเดียว
for emp in asObject do
    table.insert(itEmployees, emp)
    print(string.format("  %s, อายุ %d, เงินเดือน %.0f",
        emp.name, emp.age, emp.salary))
end

local total = 0
for _, e in ipairs(itEmployees) do total = total + e.salary end
local avg = #itEmployees > 0 and total / #itEmployees or 0

print(string.format("\nพนักงาน IT: %d คน", #itEmployees))
print(string.format("เงินเดือนเฉลี่ย: %.0f บาท", avg))
```

---

## 24.5 Generator Pattern

### ตัวอย่างที่ 12: Infinite Sequences

```lua
-- Infinite sequences ด้วย coroutines

-- Fibonacci infinite
local function fibGen()
    return coroutine.wrap(function()
        local a, b = 0, 1
        while true do
            coroutine.yield(a)
            a, b = b, a + b
        end
    end)
end

-- Prime infinite
local function primeGen()
    return coroutine.wrap(function()
        local function isPrime(n)
            if n < 2 then return false end
            if n == 2 then return true end
            if n % 2 == 0 then return false end
            for i = 3, math.sqrt(n), 2 do
                if n % i == 0 then return false end
            end
            return true
        end
        
        local n = 2
        while true do
            if isPrime(n) then
                coroutine.yield(n)
            end
            n = n + 1
        end
    end)
end

-- Powers of 2
local function powersOf2()
    return coroutine.wrap(function()
        local n = 1
        while true do
            coroutine.yield(n)
            n = n * 2
        end
    end)
end

-- Natural numbers
local function naturals(start)
    return coroutine.wrap(function()
        local n = start or 1
        while true do
            coroutine.yield(n)
            n = n + 1
        end
    end)
end

-- Take n elements from generator
local function take(gen, n)
    local results = {}
    for i = 1, n do
        local val = gen()
        if val == nil then break end
        table.insert(results, val)
    end
    return results
end

-- ทดสอบ
print("=== Infinite Sequences ===\n")

local fib = fibGen()
print("Fibonacci (first 15):")
print(table.concat(take(fib, 15), ", "))

local primes = primeGen()
print("\nPrimes (first 20):")
print(table.concat(take(primes, 20), ", "))

local powers = powersOf2()
print("\nPowers of 2 (first 12):")
print(table.concat(take(powers, 12), ", "))

-- Combine generators
print("\nFibonacci primes (Fibonacci numbers ที่เป็น prime, first 10):")
local function fibPrimes()
    return coroutine.wrap(function()
        local function isPrime(n)
            if n < 2 then return false end
            if n == 2 then return true end
            if n % 2 == 0 then return false end
            for i = 3, math.sqrt(n), 2 do
                if n % i == 0 then return false end
            end
            return true
        end
        
        local a, b = 0, 1
        while true do
            if isPrime(a) then
                coroutine.yield(a)
            end
            a, b = b, a + b
            if a > 1000000 then break end  -- หยุดที่ 1 ล้าน
        end
    end)
end

local fp = fibPrimes()
local results = {}
for v in fp do table.insert(results, v) end
print(table.concat(results, ", "))
```

### ตัวอย่างที่ 13: Lazy Evaluation ด้วย Generator

```lua
-- Lazy Evaluation: คำนวณเฉพาะเมื่อต้องการ

-- Range generator (lazy)
local function lazyRange(start, stop, step)
    step = step or 1
    return coroutine.wrap(function()
        local i = start
        while (step > 0 and i <= stop) or (step < 0 and i >= stop) do
            coroutine.yield(i)
            i = i + step
        end
    end)
end

-- Map (lazy)
local function lazyMap(gen, fn)
    return coroutine.wrap(function()
        for v in gen do
            coroutine.yield(fn(v))
        end
    end)
end

-- Filter (lazy)
local function lazyFilter(gen, pred)
    return coroutine.wrap(function()
        for v in gen do
            if pred(v) then
                coroutine.yield(v)
            end
        end
    end)
end

-- Take (lazy)
local function lazyTake(gen, n)
    return coroutine.wrap(function()
        local count = 0
        for v in gen do
            if count >= n then break end
            coroutine.yield(v)
            count = count + 1
        end
    end)
end

-- TakeWhile (lazy)
local function lazyTakeWhile(gen, pred)
    return coroutine.wrap(function()
        for v in gen do
            if not pred(v) then break end
            coroutine.yield(v)
        end
    end)
end

-- Zip (lazy)
local function lazyZip(gen1, gen2)
    return coroutine.wrap(function()
        while true do
            local v1 = gen1()
            local v2 = gen2()
            if v1 == nil or v2 == nil then break end
            coroutine.yield(v1, v2)
        end
    end)
end

-- ทดสอบ Lazy Evaluation
print("=== Lazy Evaluation Demo ===\n")

-- หาผลรวมของเลขคู่ที่เป็น perfect square ใน range 1-1000
local function isPerfectSquare(n)
    local sq = math.floor(math.sqrt(n))
    return sq * sq == n
end

print("เลขคู่ที่เป็น perfect square ใน 1-100:")
local result1 = lazyFilter(
    lazyFilter(
        lazyRange(1, 100),
        function(x) return x % 2 == 0 end
    ),
    isPerfectSquare
)

local vals = {}
for v in result1 do table.insert(vals, v) end
print(table.concat(vals, ", "))

-- Sum ของ 10 prime แรก ที่มากกว่า 100
print("\n10 primes แรกที่มากกว่า 100:")
local function isPrime(n)
    if n < 2 then return false end
    if n == 2 then return true end
    if n % 2 == 0 then return false end
    for i = 3, math.sqrt(n), 2 do
        if n % i == 0 then return false end
    end
    return true
end

local bigPrimes = lazyTake(
    lazyFilter(
        lazyRange(101, math.huge),
        isPrime
    ),
    10
)

local sum = 0
local primeList = {}
for v in bigPrimes do
    sum = sum + v
    table.insert(primeList, v)
end
print(table.concat(primeList, ", "))
print("ผลรวม:", sum)
```

---

## 24.6 Coroutine-based Iterators

### ตัวอย่างที่ 14: Tree Iterator

```lua
-- Iterator ที่ traverse tree structure

-- สร้าง tree node
local function node(value, ...)
    return {value = value, children = {...}}
end

-- In-order traversal (for BST)
local function inorder(tree)
    return coroutine.wrap(function()
        local function traverse(n)
            if not n then return end
            -- สำหรับ binary tree: left, root, right
            if n.children and n.children[1] then
                traverse(n.children[1])
            end
            coroutine.yield(n.value)
            if n.children and n.children[2] then
                traverse(n.children[2])
            end
        end
        traverse(tree)
    end)
end

-- Pre-order traversal (DFS)
local function preorder(tree)
    return coroutine.wrap(function()
        local function traverse(n)
            if not n then return end
            coroutine.yield(n.value)
            if n.children then
                for _, child in ipairs(n.children) do
                    traverse(child)
                end
            end
        end
        traverse(tree)
    end)
end

-- Post-order traversal
local function postorder(tree)
    return coroutine.wrap(function()
        local function traverse(n)
            if not n then return end
            if n.children then
                for _, child in ipairs(n.children) do
                    traverse(child)
                end
            end
            coroutine.yield(n.value)
        end
        traverse(tree)
    end)
end

-- BFS (Level-order)
local function bfs(tree)
    return coroutine.wrap(function()
        local queue = {tree}
        while #queue > 0 do
            local current = table.remove(queue, 1)
            coroutine.yield(current.value)
            if current.children then
                for _, child in ipairs(current.children) do
                    table.insert(queue, child)
                end
            end
        end
    end)
end

-- สร้าง tree
--        1
--      / | \
--     2  3  4
--    / \    |
--   5   6   7
--       |
--       8

local tree = node(1,
    node(2,
        node(5),
        node(6,
            node(8))),
    node(3),
    node(4,
        node(7))
)

print("=== Tree Traversal ===\n")

io.write("Pre-order:  ")
for v in preorder(tree) do io.write(v .. " ") end
print()

io.write("Post-order: ")
for v in postorder(tree) do io.write(v .. " ") end
print()

io.write("BFS:        ")
for v in bfs(tree) do io.write(v .. " ") end
print()
```

### ตัวอย่างที่ 15: Directory Iterator (Simulated)

```lua
-- Recursive structure iterator (เหมือน directory walking)

-- สำหรับ filesystem จำลอง
local filesystem = {
    name = "root",
    type = "dir",
    children = {
        {name = "home", type = "dir", children = {
            {name = "user1", type = "dir", children = {
                {name = "documents", type = "dir", children = {
                    {name = "report.pdf", type = "file", size = 1024},
                    {name = "notes.txt", type = "file", size = 256}
                }},
                {name = "pictures", type = "dir", children = {
                    {name = "photo1.jpg", type = "file", size = 2048},
                    {name = "photo2.jpg", type = "file", size = 3072}
                }}
            }},
            {name = "user2", type = "dir", children = {
                {name = "code", type = "dir", children = {
                    {name = "main.lua", type = "file", size = 512},
                    {name = "utils.lua", type = "file", size = 384}
                }}
            }}
        }},
        {name = "etc", type = "dir", children = {
            {name = "config.cfg", type = "file", size = 128},
            {name = "hosts", type = "file", size = 64}
        }},
        {name = "usr", type = "dir", children = {
            {name = "bin", type = "dir", children = {
                {name = "lua", type = "file", size = 8192},
                {name = "bash", type = "file", size = 4096}
            }}
        }}
    }
}

-- Walk filesystem
local function walkFiles(fs, path)
    return coroutine.wrap(function()
        path = path or ""
        
        local function walk(node, currentPath)
            local fullPath = currentPath .. "/" .. node.name
            
            if node.type == "file" then
                coroutine.yield({
                    path = fullPath,
                    name = node.name,
                    size = node.size,
                    type = "file"
                })
            elseif node.type == "dir" and node.children then
                coroutine.yield({
                    path = fullPath,
                    name = node.name,
                    type = "dir"
                })
                for _, child in ipairs(node.children) do
                    walk(child, fullPath)
                end
            end
        end
        
        walk(fs, path)
    end)
end

-- Find files by extension
local function findFiles(fs, ext)
    return coroutine.wrap(function()
        for entry in walkFiles(fs) do
            if entry.type == "file" and 
               entry.name:match("%." .. ext .. "$") then
                coroutine.yield(entry)
            end
        end
    end)
end

print("=== Filesystem Walk ===\n")

-- หาไฟล์ทั้งหมด
print("ไฟล์ .lua:")
for f in findFiles(filesystem, "lua") do
    print(string.format("  %s (%d bytes)", f.path, f.size))
end

print("\nไฟล์ .jpg:")
for f in findFiles(filesystem, "jpg") do
    print(string.format("  %s (%d bytes)", f.path, f.size))
end

-- คำนวณขนาดรวม
local totalSize = 0
local fileCount = 0
for entry in walkFiles(filesystem) do
    if entry.type == "file" then
        totalSize = totalSize + entry.size
        fileCount = fileCount + 1
    end
end
print(string.format("\nไฟล์ทั้งหมด: %d ไฟล์, %d bytes", fileCount, totalSize))
```

---

## 24.7 Coroutine Scheduler

### ตัวอย่างที่ 16-20: Simple Scheduler

```lua
-- Coroutine Scheduler: จัดการหลาย coroutines
local Scheduler = {}
Scheduler.__index = Scheduler

function Scheduler.new()
    return setmetatable({
        ready = {},      -- coroutines ที่พร้อมทำงาน
        waiting = {},    -- coroutines ที่รอ
        sleeping = {},   -- coroutines ที่นอนหลับ
        current = nil,   -- coroutine ที่กำลังทำงาน
        time = 0         -- virtual time
    }, Scheduler)
end

function Scheduler:spawn(fn, name)
    local co = coroutine.create(fn)
    table.insert(self.ready, {co = co, name = name or "task_" .. #self.ready})
    print(string.format("[Scheduler] spawn '%s'", name or "task"))
    return co
end

function Scheduler:yield()
    -- ให้ scheduler จัดการ coroutines อื่น
    coroutine.yield("yield")
end

function Scheduler:sleep(seconds)
    -- นอนหลับ n วินาที (virtual time)
    coroutine.yield("sleep", seconds)
end

function Scheduler:waitFor(condition)
    -- รอจนกว่า condition เป็น true
    coroutine.yield("wait", condition)
end

function Scheduler:run()
    print("[Scheduler] เริ่มทำงาน\n")
    
    while #self.ready > 0 or #self.sleeping > 0 do
        -- อัพเดท sleeping tasks
        local newSleeping = {}
        for _, task in ipairs(self.sleeping) do
            task.wakeTime = task.wakeTime - 0.1
            if task.wakeTime <= 0 then
                table.insert(self.ready, task)
            else
                table.insert(newSleeping, task)
            end
        end
        self.sleeping = newSleeping
        
        -- ทำงาน ready tasks
        if #self.ready > 0 then
            local task = table.remove(self.ready, 1)
            self.current = task
            
            local ok, signal, data = coroutine.resume(task.co)
            
            if not ok then
                print(string.format("[Scheduler] Error in '%s': %s", 
                    task.name, tostring(signal)))
            elseif coroutine.status(task.co) == "dead" then
                print(string.format("[Scheduler] '%s' เสร็จสิ้น", task.name))
            elseif signal == "sleep" then
                task.wakeTime = data or 1
                table.insert(self.sleeping, task)
            elseif signal == "yield" then
                table.insert(self.ready, task)
            elseif signal == "wait" then
                task.condition = data
                table.insert(self.waiting, task)
            end
        end
        
        -- Check waiting tasks
        local newWaiting = {}
        for _, task in ipairs(self.waiting) do
            if task.condition and task.condition() then
                table.insert(self.ready, task)
            else
                table.insert(newWaiting, task)
            end
        end
        self.waiting = newWaiting
        
        self.time = self.time + 0.1
    end
    
    print("\n[Scheduler] เสร็จสิ้นทั้งหมด")
end

-- สร้าง global scheduler instance สำหรับ tasks ใช้
local scheduler = Scheduler.new()

-- Tasks
local task1 = scheduler:spawn(function()
    print("[Task1] เริ่มทำงาน")
    for i = 1, 3 do
        print("[Task1] ขั้นตอน " .. i)
        scheduler:yield()
    end
    print("[Task1] เสร็จแล้ว")
end, "Task1")

local task2 = scheduler:spawn(function()
    print("[Task2] เริ่มทำงาน")
    print("[Task2] กำลังประมวลผล...")
    scheduler:yield()
    print("[Task2] ยังทำงานอยู่...")
    scheduler:yield()
    print("[Task2] เสร็จแล้ว")
end, "Task2")

local task3 = scheduler:spawn(function()
    print("[Task3] เริ่ม (นอนหลับ 0.5s)")
    scheduler:sleep(0.5)
    print("[Task3] ตื่นแล้ว!")
    scheduler:sleep(0.3)
    print("[Task3] เสร็จแล้ว")
end, "Task3")

scheduler:run()
```

### ตัวอย่างที่ 21: Async Simulation

```lua
-- จำลอง async/await ด้วย coroutines

-- Async framework
local Async = {}

-- "Promise" ง่ายๆ
local function createPromise(fn)
    local promise = {
        _status = "pending",
        _value = nil,
        _callbacks = {}
    }
    
    local function resolve(value)
        if promise._status ~= "pending" then return end
        promise._status = "fulfilled"
        promise._value = value
        for _, cb in ipairs(promise._callbacks) do
            cb(value)
        end
    end
    
    local function reject(reason)
        if promise._status ~= "pending" then return end
        promise._status = "rejected"
        promise._value = reason
    end
    
    -- Execute
    local ok, err = pcall(fn, resolve, reject)
    if not ok then reject(err) end
    
    function promise:andThen(callback)
        if self._status == "fulfilled" then
            callback(self._value)
        else
            table.insert(self._callbacks, callback)
        end
        return self
    end
    
    function promise:getStatus() return self._status end
    function promise:getValue() return self._value end
    
    return promise
end

-- Simulate async operation
local function asyncFetch(url, delay)
    return createPromise(function(resolve, reject)
        -- จำลองการ fetch (ใน real world จะใช้ async I/O)
        print(string.format("  [Async] กำลัง fetch: %s (delay: %dms)", url, delay))
        -- ใน real scenario: หลังจาก delay จะ resolve
        resolve({url = url, data = "data from " .. url, status = 200})
    end)
end

-- Await wrapper ด้วย coroutines
local function await(promise)
    local co = coroutine.running()
    
    if promise._status == "fulfilled" then
        return promise._value
    end
    
    promise:andThen(function(value)
        coroutine.resume(co, value)
    end)
    
    return coroutine.yield()
end

-- Async function
local function asyncTask(name, urls)
    return coroutine.create(function()
        print(string.format("\n[%s] เริ่มงาน async", name))
        
        local results = {}
        for _, url in ipairs(urls) do
            local promise = asyncFetch(url, 100)
            -- ในตัวอย่างนี้ promise resolve ทันที
            if promise._status == "fulfilled" then
                local data = promise._value
                table.insert(results, data.data)
                print(string.format("  [%s] ได้รับ: %s", name, data.data))
            end
        end
        
        print(string.format("[%s] เสร็จสิ้น ได้รับ %d responses", name, #results))
        return results
    end)
end

print("=== Async Simulation ===")

local urls = {
    "https://api.example.com/users",
    "https://api.example.com/products",
    "https://api.example.com/orders"
}

local task = asyncTask("MyTask", urls)
coroutine.resume(task)
```

---

## 24.8 Error Handling ใน Coroutines

### ตัวอย่างที่ 22: Error Handling

```lua
-- การจัดการ error ใน coroutines

-- 1. Error แบบปกติ
local co1 = coroutine.create(function()
    print("co1: เริ่มทำงาน")
    error("บางอย่างผิดพลาด!")
    print("co1: บรรทัดนี้จะไม่ถูกรัน")
end)

local ok, err = coroutine.resume(co1)
print("ok:", ok)         -- false
print("error:", err)     -- บางอย่างผิดพลาด! (พร้อม stack trace)
print("status:", coroutine.status(co1))  -- dead

-- 2. Error handling ด้วย pcall ภายใน coroutine
local co2 = coroutine.create(function()
    print("\nco2: เริ่มทำงาน")
    
    local ok, err = pcall(function()
        error("error ภายใน pcall")
    end)
    
    print("co2: pcall result:", ok, err)
    coroutine.yield("ยังทำงานได้หลัง error ภายใน pcall")
    print("co2: เสร็จสิ้น")
end)

ok, err = coroutine.resume(co2)
print("Resume 1:", ok, err)
ok, err = coroutine.resume(co2)
print("Resume 2:", ok, err)

-- 3. Protected coroutine wrapper
local function safeCoroutine(fn)
    local co = coroutine.create(fn)
    
    return {
        resume = function(...)
            if coroutine.status(co) == "dead" then
                return false, "coroutine ตายแล้ว"
            end
            
            local ok, val = coroutine.resume(co, ...)
            if not ok then
                return false, val, "error"
            end
            return true, val, coroutine.status(co)
        end,
        status = function()
            return coroutine.status(co)
        end
    }
end

print("\n=== Protected Coroutine ===")
local safe = safeCoroutine(function()
    for i = 1, 3 do
        if i == 2 then error("error ที่ i=2") end
        coroutine.yield(i)
    end
end)

for attempt = 1, 5 do
    local ok, val, status = safe.resume()
    print(string.format("Attempt %d: ok=%s, val=%s, status=%s",
        attempt, tostring(ok), tostring(val), tostring(status)))
end
```

### ตัวอย่างที่ 23: Error Propagation

```lua
-- Error propagation ใน coroutine chain

local function riskyTask(n)
    return coroutine.create(function()
        for i = 1, n do
            if i == 3 then
                error(string.format("Task error ที่ i=%d", i))
            end
            coroutine.yield(i * 10)
        end
    end)
end

-- Supervisor ที่จัดการ errors
local function supervisor(tasks)
    local results = {}
    local errors = {}
    
    for name, co in pairs(tasks) do
        results[name] = {}
        local running = true
        
        while running do
            local ok, val = coroutine.resume(co)
            
            if not ok then
                errors[name] = val
                running = false
                print(string.format("[Supervisor] Error ใน '%s': %s", name, val))
            elseif coroutine.status(co) == "dead" then
                running = false
                print(string.format("[Supervisor] '%s' เสร็จสิ้น", name))
            else
                table.insert(results[name], val)
            end
        end
    end
    
    return results, errors
end

print("=== Error Propagation Demo ===\n")

local tasks = {
    task1 = riskyTask(2),    -- เสร็จปกติ
    task2 = riskyTask(5),    -- จะเกิด error ที่ i=3
    task3 = riskyTask(1),    -- เสร็จปกติ
}

local results, errors = supervisor(tasks)

print("\nResults:")
for name, vals in pairs(results) do
    print(string.format("  %s: {%s}", name, table.concat(vals, ", ")))
end

print("\nErrors:")
for name, err in pairs(errors) do
    print(string.format("  %s: %s", name, err))
end
```

---

## 24.9 coroutine.running()

### ตัวอย่างที่ 24: coroutine.running()

```lua
-- coroutine.running() คืน coroutine ที่กำลังทำงานอยู่
-- และ boolean ว่าเป็น main thread หรือไม่

-- ใน main thread
local co, isMain = coroutine.running()
print("Main thread:")
print("  co =", co)           -- nil (หรือ main coroutine ใน Lua 5.4)
print("  isMain =", isMain)   -- true

-- ภายใน coroutine
local innerCo = coroutine.create(function()
    local self, isMain = coroutine.running()
    print("\nภายใน coroutine:")
    print("  self =", self)
    print("  isMain =", isMain)  -- false
    
    -- ใช้สำหรับ identify ตัวเอง
    print("  เหมือนกับ coroutine ภายนอก?", self == innerCo)
end)

-- ต้องรัน innerCo ก่อนจึงจะเปรียบเทียบได้
coroutine.resume(innerCo)

-- Use case: ตรวจสอบว่าอยู่ใน coroutine หรือไม่
local function requiresCoroutine()
    local co, isMain = coroutine.running()
    if isMain then
        error("ต้องเรียกจากภายใน coroutine เท่านั้น!")
    end
    print("ทำงานใน coroutine - OK!")
end

print("\n--- ทดสอบ requiresCoroutine ---")
-- เรียกจาก main
local ok, err = pcall(requiresCoroutine)
print("จาก main:", ok, err)

-- เรียกจาก coroutine
local testCo = coroutine.create(requiresCoroutine)
ok, err = coroutine.resume(testCo)
print("จาก coroutine:", ok, err)
```

---

## 24.10 ตัวอย่างขั้นสูง: Async Task System

### ตัวอย่างที่ 25-30: Complete Async System

```lua
-- Complete Async Task System

-- Task states
local PENDING = "pending"
local RUNNING = "running"
local COMPLETED = "completed"
local FAILED = "failed"
local CANCELLED = "cancelled"

-- Task class
local Task = {}
Task.__index = Task

function Task.new(name, fn)
    local task = setmetatable({
        name = name,
        fn = fn,
        status = PENDING,
        result = nil,
        error = nil,
        co = nil,
        dependencies = {},
        callbacks = {}
    }, Task)
    
    task.co = coroutine.create(function()
        task.status = RUNNING
        local ok, result = pcall(fn)
        if ok then
            task.status = COMPLETED
            task.result = result
        else
            task.status = FAILED
            task.error = result
        end
        return task.result
    end)
    
    return task
end

function Task:onComplete(callback)
    if self.status == COMPLETED then
        callback(self.result)
    else
        table.insert(self.callbacks, {type="complete", fn=callback})
    end
    return self
end

function Task:onError(callback)
    if self.status == FAILED then
        callback(self.error)
    else
        table.insert(self.callbacks, {type="error", fn=callback})
    end
    return self
end

function Task:dependsOn(...)
    for _, dep in ipairs({...}) do
        table.insert(self.dependencies, dep)
    end
    return self
end

function Task:isReady()
    for _, dep in ipairs(self.dependencies) do
        if dep.status ~= COMPLETED then
            return false
        end
    end
    return true
end

function Task:run()
    if self.status ~= PENDING then return end
    if not self:isReady() then return end
    
    local ok, val = coroutine.resume(self.co)
    
    -- Notify callbacks
    for _, cb in ipairs(self.callbacks) do
        if cb.type == "complete" and self.status == COMPLETED then
            cb.fn(self.result)
        elseif cb.type == "error" and self.status == FAILED then
            cb.fn(self.error)
        end
    end
    
    return ok, val
end

-- TaskRunner
local TaskRunner = {}
TaskRunner.__index = TaskRunner

function TaskRunner.new()
    return setmetatable({
        tasks = {},
        maxRetries = 3
    }, TaskRunner)
end

function TaskRunner:add(task)
    table.insert(self.tasks, task)
    return self
end

function TaskRunner:runAll()
    print("[TaskRunner] เริ่มรัน " .. #self.tasks .. " tasks\n")
    
    local completed = 0
    local failed = 0
    local maxIterations = 100
    local iteration = 0
    
    while completed + failed < #self.tasks and iteration < maxIterations do
        iteration = iteration + 1
        
        for _, task in ipairs(self.tasks) do
            if task.status == PENDING and task:isReady() then
                print(string.format("[Runner] รัน task: '%s'", task.name))
                task:run()
                
                if task.status == COMPLETED then
                    completed = completed + 1
                    print(string.format("[Runner] '%s' สำเร็จ: %s", 
                        task.name, tostring(task.result)))
                elseif task.status == FAILED then
                    failed = failed + 1
                    print(string.format("[Runner] '%s' ล้มเหลว: %s", 
                        task.name, tostring(task.error)))
                end
            end
        end
    end
    
    print(string.format("\n[Runner] สรุป: สำเร็จ %d, ล้มเหลว %d, รวม %d",
        completed, failed, #self.tasks))
    
    return completed, failed
end

-- ทดสอบ
print("=== Async Task System ===\n")

local runner = TaskRunner.new()

local taskA = Task.new("LoadConfig", function()
    print("  LoadConfig: กำลังโหลด config...")
    return {host="localhost", port=3000}
end)

local taskB = Task.new("ConnectDB", function()
    print("  ConnectDB: กำลังเชื่อมต่อ database...")
    return "connected"
end)

local taskC = Task.new("InitApp", function()
    print("  InitApp: กำลัง initialize app...")
    -- ต้องรอ taskA และ taskB ก่อน
    return "initialized with config and db"
end):dependsOn(taskA, taskB)

local taskD = Task.new("StartServer", function()
    print("  StartServer: กำลังเริ่ม server...")
    return "server started on port 3000"
end):dependsOn(taskC)

local taskE = Task.new("ErrorTask", function()
    print("  ErrorTask: จะเกิด error...")
    error("Simulated error!")
end)

-- Register callbacks
taskD:onComplete(function(result)
    print("  >> Callback: Server started! - " .. result)
end)

taskE:onError(function(err)
    print("  >> Error callback: " .. err)
end)

-- Add tasks
runner:add(taskA):add(taskB):add(taskC):add(taskD):add(taskE)

runner:runAll()
```

---

## 24.11 Generator Pattern ขั้นสูง

### ตัวอย่างที่ 31-35: Advanced Generators

```lua
-- Generator Combinators

-- สร้าง generator จาก table
local function fromTable(t)
    return coroutine.wrap(function()
        for _, v in ipairs(t) do
            coroutine.yield(v)
        end
    end)
end

-- Enumerate: เพิ่ม index
local function enumerate(gen, start)
    start = start or 1
    return coroutine.wrap(function()
        local i = start
        for v in gen do
            coroutine.yield(i, v)
            i = i + 1
        end
    end)
end

-- FlatMap
local function flatMap(gen, fn)
    return coroutine.wrap(function()
        for v in gen do
            local inner = fn(v)
            if type(inner) == "function" then
                for w in inner do
                    coroutine.yield(w)
                end
            else
                coroutine.yield(inner)
            end
        end
    end)
end

-- Scan (running aggregate)
local function scan(gen, fn, initial)
    return coroutine.wrap(function()
        local acc = initial
        coroutine.yield(acc)
        for v in gen do
            acc = fn(acc, v)
            coroutine.yield(acc)
        end
    end)
end

-- Pairwise (sliding window of 2)
local function pairwise(gen)
    return coroutine.wrap(function()
        local prev = nil
        local first = true
        for v in gen do
            if not first then
                coroutine.yield(prev, v)
            end
            prev = v
            first = false
        end
    end)
end

-- Sliding window
local function window(gen, size)
    return coroutine.wrap(function()
        local buf = {}
        for v in gen do
            table.insert(buf, v)
            if #buf == size then
                local w = {}
                for _, x in ipairs(buf) do table.insert(w, x) end
                coroutine.yield(w)
                table.remove(buf, 1)
            end
        end
    end)
end

-- Chunk
local function chunk(gen, size)
    return coroutine.wrap(function()
        local buf = {}
        for v in gen do
            table.insert(buf, v)
            if #buf == size then
                coroutine.yield(buf)
                buf = {}
            end
        end
        if #buf > 0 then
            coroutine.yield(buf)
        end
    end)
end

-- Interleave สอง generators
local function interleave(gen1, gen2)
    return coroutine.wrap(function()
        while true do
            local v1 = gen1()
            local v2 = gen2()
            if v1 == nil and v2 == nil then break end
            if v1 ~= nil then coroutine.yield(v1) end
            if v2 ~= nil then coroutine.yield(v2) end
        end
    end)
end

-- ทดสอบ Generator Combinators
print("=== Generator Combinators ===\n")

local data = {10, 20, 30, 40, 50, 60, 70, 80, 90, 100}

print("Enumerate:")
for i, v in enumerate(fromTable(data)) do
    io.write(string.format("(%d:%d) ", i, v))
end
print()

print("\nRunning sum (scan):")
local runningSum = scan(fromTable(data), function(acc, x) return acc + x end, 0)
for v in runningSum do
    io.write(v .. " ")
end
print()

print("\nPairwise differences:")
for a, b in pairwise(fromTable(data)) do
    io.write(string.format("(%d-%d=%d) ", b, a, b-a))
end
print()

print("\nSliding window (size=3):")
for w in window(fromTable(data), 3) do
    io.write(string.format("{%s} ", table.concat(w, ",")))
end
print()

print("\nChunk (size=3):")
for c in chunk(fromTable(data), 3) do
    print("  {" .. table.concat(c, ", ") .. "}")
end

print("\nInterleave {1,2,3} and {a,b,c,d}:")
local gen1 = fromTable({1, 2, 3})
local gen2 = fromTable({"a", "b", "c", "d"})
for v in interleave(gen1, gen2) do
    io.write(tostring(v) .. " ")
end
print()
```

---

## 24.12 Coroutine-based State Machine

### ตัวอย่างที่ 36-40: State Machine ด้วย Coroutines

```lua
-- State Machine ที่ใช้ coroutines สำหรับแต่ละ state

-- Traffic Light State Machine
local function trafficLightSM()
    return coroutine.wrap(function()
        while true do
            -- State: RED
            print("🔴 แสงแดง - หยุด!")
            for i = 1, 3 do
                coroutine.yield({state = "red", timeLeft = 3 - i + 1})
            end
            
            -- State: GREEN
            print("🟢 แสงเขียว - ไปได้!")
            for i = 1, 4 do
                coroutine.yield({state = "green", timeLeft = 4 - i + 1})
            end
            
            -- State: YELLOW
            print("🟡 แสงเหลือง - ระวัง!")
            for i = 1, 2 do
                coroutine.yield({state = "yellow", timeLeft = 2 - i + 1})
            end
        end
    end)
end

print("=== Traffic Light Simulation ===")
local light = trafficLightSM()
for tick = 1, 15 do
    local status = light()
    print(string.format("  Tick %2d: %-6s (เหลือ %d วินาที)", 
        tick, status.state, status.timeLeft))
end

-- Order Processing State Machine
local function orderStateMachine(orderId)
    return coroutine.wrap(function()
        print(string.format("\n[Order %s] สร้างคำสั่งซื้อ", orderId))
        coroutine.yield("pending")
        
        print(string.format("[Order %s] ตรวจสอบการชำระเงิน", orderId))
        coroutine.yield("payment_checking")
        
        local paid = true  -- จำลองว่าชำระแล้ว
        if paid then
            print(string.format("[Order %s] ชำระเงินสำเร็จ", orderId))
            coroutine.yield("paid")
        else
            print(string.format("[Order %s] ชำระเงินล้มเหลว", orderId))
            coroutine.yield("payment_failed")
            return
        end
        
        print(string.format("[Order %s] กำลังจัดเตรียมสินค้า", orderId))
        coroutine.yield("preparing")
        
        print(string.format("[Order %s] กำลังจัดส่ง", orderId))
        coroutine.yield("shipping")
        
        print(string.format("[Order %s] จัดส่งสำเร็จ!", orderId))
        coroutine.yield("delivered")
    end)
end

print("\n=== Order State Machine ===")
local order = orderStateMachine("ORD-001")
for state in order do
    print(string.format("  สถานะปัจจุบัน: %s", state))
end
```

---

## 24.13 ตัวอย่างเพิ่มเติม: Reader Monad Pattern

### ตัวอย่างที่ 41-45: Advanced Coroutine Patterns

```lua
-- Coroutine-based event loop (ง่ายๆ)

local EventLoop = {}
EventLoop.__index = EventLoop

function EventLoop.new()
    return setmetatable({
        queue = {},
        timers = {},
        running = false
    }, EventLoop)
end

function EventLoop:setTimeout(fn, delay)
    table.insert(self.timers, {
        fn = fn,
        fireAt = self._tick + (delay or 0)
    })
end

function EventLoop:emit(event, data)
    table.insert(self.queue, {event = event, data = data})
end

function EventLoop:on(event, handler)
    self._handlers = self._handlers or {}
    self._handlers[event] = self._handlers[event] or {}
    table.insert(self._handlers[event], handler)
end

function EventLoop:run(maxTicks)
    self.running = true
    self._tick = 0
    maxTicks = maxTicks or 20
    
    print("[EventLoop] เริ่มทำงาน (max " .. maxTicks .. " ticks)\n")
    
    while self.running and self._tick < maxTicks do
        self._tick = self._tick + 1
        
        -- Process timers
        local remaining = {}
        for _, timer in ipairs(self.timers) do
            if timer.fireAt <= self._tick then
                local co = coroutine.create(timer.fn)
                coroutine.resume(co)
            else
                table.insert(remaining, timer)
            end
        end
        self.timers = remaining
        
        -- Process queue
        local toProcess = self.queue
        self.queue = {}
        
        for _, msg in ipairs(toProcess) do
            if self._handlers and self._handlers[msg.event] then
                for _, handler in ipairs(self._handlers[msg.event]) do
                    local co = coroutine.create(function()
                        handler(msg.data)
                    end)
                    coroutine.resume(co)
                end
            end
        end
        
        -- Stop if nothing to do
        if #self.queue == 0 and #self.timers == 0 then
            self.running = false
        end
    end
    
    print("[EventLoop] หยุดทำงาน (tick " .. self._tick .. ")")
end

-- ทดสอบ EventLoop
print("=== Event Loop Demo ===\n")

local loop = EventLoop.new()

-- Register event handlers
loop:on("click", function(data)
    print(string.format("  [Handler] click: x=%d, y=%d", data.x, data.y))
end)

loop:on("keypress", function(data)
    print(string.format("  [Handler] keypress: '%s'", data.key))
end)

loop:on("resize", function(data)
    print(string.format("  [Handler] resize: %dx%d", data.width, data.height))
end)

-- Schedule events
loop:setTimeout(function()
    print("[Timer] เวลา 2: emit click event")
    loop:emit("click", {x=100, y=200})
end, 2)

loop:setTimeout(function()
    print("[Timer] เวลา 4: emit keypress")
    loop:emit("keypress", {key="Enter"})
end, 4)

loop:setTimeout(function()
    print("[Timer] เวลา 5: emit resize")
    loop:emit("resize", {width=1920, height=1080})
end, 5)

loop:setTimeout(function()
    print("[Timer] เวลา 7: emit multiple events")
    loop:emit("click", {x=50, y=75})
    loop:emit("keypress", {key="Escape"})
end, 7)

loop:run()
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Coroutine Pipeline
สร้าง data processing pipeline ที่:
- Source: อ่าน numbers จาก array
- Stage 1: กรองเฉพาะ prime numbers
- Stage 2: คำนวณ square root
- Stage 3: ปัดทศนิยมเป็น 2 ตำแหน่ง
- Stage 4: แปลงเป็น string
- Sink: เก็บใน array ผลลัพธ์

### แบบฝึกหัดที่ 2: Coroutine Scheduler
ปรับปรุง Scheduler ให้:
- รองรับ priority scheduling (task priority สูงทำงานก่อน)
- รองรับ task timeout (ยกเลิก task ที่ใช้เวลานานเกินไป)
- รองรับ task cancellation
- แสดง Gantt chart ของการทำงาน (text-based)

### แบบฝึกหัดที่ 3: Infinite Generator
สร้าง generators สำหรับ:
- Collatz sequence (3n+1 problem)
- Pascal's triangle rows
- Recaman sequence
- Van Eck's sequence

### แบบฝึกหัดที่ 4: Async I/O Simulation
สร้าง async system ที่จำลอง:
- File reading (yield หลังจาก "delay")
- HTTP requests (yield และรับ response)
- Database queries
- รวม results จากหลาย async operations

### แบบฝึกหัดที่ 5: Coroutine-based Parser
สร้าง parser สำหรับ simple expression language ด้วย coroutines:
- Tokenizer เป็น coroutine
- Parser consume tokens จาก tokenizer
- Evaluate expressions
- รองรับ: +, -, *, /, (), numbers

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **coroutine.create()**: สร้าง coroutine จาก function
2. **coroutine.resume()**: เริ่มหรือต่อการทำงานของ coroutine พร้อมส่งค่า
3. **coroutine.yield()**: หยุดชั่วคราวและส่งค่ากลับไปยังผู้เรียก
4. **coroutine.status()**: ตรวจสอบสถานะ (suspended/running/normal/dead)
5. **coroutine.wrap()**: สร้าง coroutine ที่เรียกใช้ได้เหมือน function
6. **Two-way Communication**: ส่งค่าไปมาระหว่าง resume และ yield
7. **Producer-Consumer**: pattern ที่ coroutines ทำงานร่วมกัน
8. **Pipeline**: ต่อ coroutines เป็น processing stages
9. **Generator Pattern**: สร้าง infinite sequences
10. **Lazy Evaluation**: คำนวณเฉพาะเมื่อต้องการ
11. **Coroutine Scheduler**: จัดการ cooperative multitasking
12. **Error Handling**: จัดการ errors ใน coroutines ด้วย pcall
13. **coroutine.running()**: ระบุ coroutine ที่กำลังทำงาน
14. **State Machine**: implement state machines ด้วย coroutines

> **เคล็ดลับ**: Coroutines เป็นเครื่องมือที่ทรงพลังมากใน Lua ใช้สำหรับ game AI, network programming, test automation, และ event-driven systems โดยไม่ต้องใช้ OS threads ที่ซับซ้อน
