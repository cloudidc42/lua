# บทที่ 11: Table Library เชิงลึก

## สารบัญ
1. [บทนำ Table Library](#บทนำ)
2. [table.insert และ table.remove](#tableinsert-และ-tableremove)
3. [table.sort](#tablesort)
4. [table.concat](#tableconcat)
5. [table.move](#tablemove)
6. [table.pack และ table.unpack](#tablepack-และ-tableunpack)
7. [Sorting Algorithms](#sorting-algorithms)
8. [Binary Search](#binary-search)
9. [Functional Operations](#functional-operations)
10. [Table Utilities](#table-utilities)
11. [Deep Copy และ Shallow Copy](#deep-copy-และ-shallow-copy)
12. [Merge และ Diff Tables](#merge-และ-diff-tables)
13. [Group By และ Partition](#group-by-และ-partition)
14. [Advanced Patterns](#advanced-patterns)

---

## บทนำ

`table` library ใน Lua เป็น standard library ที่ให้ฟังก์ชันสำหรับจัดการ table (array/list) ทำให้สามารถทำงานกับข้อมูลได้อย่างมีประสิทธิภาพ

```lua
-- ตัวอย่างที่ 1: ภาพรวม table library
print("=== Table Library Functions ===")
for name, func in pairs(table) do
    print(string.format("  table.%-15s : %s", name, type(func)))
end
```

```lua
-- ตัวอย่างที่ 2: ความต่างระหว่าง array part และ hash part
local mixed = {10, 20, 30}    -- array part: indices 1,2,3
mixed.name = "test"            -- hash part
mixed[10] = "sparse"           -- sparse array

print("=== Table Structure ===")
print("#mixed (length):", #mixed)  -- 3 (เฉพาะ contiguous part)
print("mixed.name:", mixed.name)
print("mixed[10]:", mixed[10])

-- ipairs ทำงานกับ array part เท่านั้น
print("\nipairs:")
for i, v in ipairs(mixed) do
    print(i, v)
end

-- pairs ทำงานกับทุก key
print("\npairs:")
for k, v in pairs(mixed) do
    print(k, v)
end
```

```lua
-- ตัวอย่างที่ 3: Performance considerations
local N = 100000

-- Pre-allocate (efficient)
local t1 = {}
local start = os.clock()
for i = 1, N do
    t1[i] = i * 2
end
print(string.format("Pre-allocated: %.4f sec", os.clock() - start))

-- Dynamic growth (less efficient)
local t2 = {}
start = os.clock()
for i = 1, N do
    t2[#t2 + 1] = i * 2
end
print(string.format("Dynamic append: %.4f sec", os.clock() - start))

-- table.insert (slowest for large arrays)
local t3 = {}
start = os.clock()
for i = 1, N do
    table.insert(t3, i * 2)
end
print(string.format("table.insert: %.4f sec", os.clock() - start))
```

---

## table.insert และ table.remove

```lua
-- ตัวอย่างที่ 4: table.insert พื้นฐาน
local fruits = {"apple", "banana", "cherry"}

-- เพิ่มต่อท้าย (ไม่ระบุ position)
table.insert(fruits, "date")
print("หลัง insert ต่อท้าย:", table.concat(fruits, ", "))

-- เพิ่มที่ตำแหน่งที่กำหนด
table.insert(fruits, 2, "blueberry")
print("หลัง insert ที่ pos 2:", table.concat(fruits, ", "))

-- เพิ่มที่ตำแหน่งแรก
table.insert(fruits, 1, "avocado")
print("หลัง insert ที่ pos 1:", table.concat(fruits, ", "))

print("ขนาด:", #fruits)
```

```lua
-- ตัวอย่างที่ 5: table.remove
local numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

-- ลบตัวสุดท้าย (ค่า default)
local last = table.remove(numbers)
print("ลบตัวสุดท้าย:", last)
print("เหลือ:", table.concat(numbers, ", "))

-- ลบที่ตำแหน่งกำหนด - คืนค่าที่ถูกลบ
local third = table.remove(numbers, 3)
print("ลบตำแหน่ง 3:", third)
print("เหลือ:", table.concat(numbers, ", "))

-- ลบตัวแรก
local first = table.remove(numbers, 1)
print("ลบตำแหน่ง 1:", first)
print("เหลือ:", table.concat(numbers, ", "))
```

```lua
-- ตัวอย่างที่ 6: ใช้ table เป็น Stack
local Stack = {}
Stack.__index = Stack

function Stack.new()
    return setmetatable({_data = {}, _size = 0}, Stack)
end

function Stack:push(value)
    self._size = self._size + 1
    self._data[self._size] = value
end

function Stack:pop()
    if self._size == 0 then return nil end
    local value = self._data[self._size]
    self._data[self._size] = nil
    self._size = self._size - 1
    return value
end

function Stack:peek()
    return self._data[self._size]
end

function Stack:is_empty()
    return self._size == 0
end

function Stack:size()
    return self._size
end

-- ทดสอบ Stack
local s = Stack.new()
s:push(1); s:push(2); s:push(3); s:push(4); s:push(5)
print("=== Stack ===")
print("Size:", s:size())
print("Peek:", s:peek())
print("Pop:", s:pop(), s:pop(), s:pop())
print("Size หลัง pop:", s:size())
```

```lua
-- ตัวอย่างที่ 7: ใช้ table เป็น Queue
local Queue = {}
Queue.__index = Queue

function Queue.new()
    return setmetatable({_data = {}, _head = 1, _tail = 0}, Queue)
end

function Queue:enqueue(value)
    self._tail = self._tail + 1
    self._data[self._tail] = value
end

function Queue:dequeue()
    if self._head > self._tail then return nil end
    local value = self._data[self._head]
    self._data[self._head] = nil
    self._head = self._head + 1
    return value
end

function Queue:front()
    return self._data[self._head]
end

function Queue:size()
    return self._tail - self._head + 1
end

function Queue:is_empty()
    return self._head > self._tail
end

-- ทดสอบ Queue
local q = Queue.new()
q:enqueue("first"); q:enqueue("second"); q:enqueue("third")
print("=== Queue ===")
print("Size:", q:size())
print("Front:", q:front())
print("Dequeue:", q:dequeue(), q:dequeue())
print("Size หลัง dequeue:", q:size())
```

```lua
-- ตัวอย่างที่ 8: Deque (Double-ended Queue)
local Deque = {}
Deque.__index = Deque

function Deque.new()
    return setmetatable({_data = {}, _front = 0, _back = -1}, Deque)
end

function Deque:push_front(v)
    self._front = self._front - 1
    self._data[self._front] = v
end

function Deque:push_back(v)
    self._back = self._back + 1
    self._data[self._back] = v
end

function Deque:pop_front()
    if self:is_empty() then return nil end
    local v = self._data[self._front]
    self._data[self._front] = nil
    self._front = self._front + 1
    return v
end

function Deque:pop_back()
    if self:is_empty() then return nil end
    local v = self._data[self._back]
    self._data[self._back] = nil
    self._back = self._back - 1
    return v
end

function Deque:size()
    return self._back - self._front + 1
end

function Deque:is_empty()
    return self._front > self._back
end

local d = Deque.new()
d:push_back(1); d:push_back(2); d:push_back(3)
d:push_front(0); d:push_front(-1)
print("=== Deque ===")
print("Size:", d:size())
print("Pop front:", d:pop_front(), d:pop_front())
print("Pop back:", d:pop_back(), d:pop_back())
```

---

## table.sort

```lua
-- ตัวอย่างที่ 9: table.sort พื้นฐาน
local nums = {5, 3, 8, 1, 9, 2, 7, 4, 6}
print("ก่อน sort:", table.concat(nums, ", "))

-- sort ascending (ค่าเริ่มต้น)
table.sort(nums)
print("Ascending:", table.concat(nums, ", "))

-- sort descending
table.sort(nums, function(a, b) return a > b end)
print("Descending:", table.concat(nums, ", "))

-- sort strings
local words = {"banana", "Apple", "cherry", "Date", "elderberry"}
table.sort(words)
print("String sort (case-sensitive):", table.concat(words, ", "))

table.sort(words, function(a, b) return a:lower() < b:lower() end)
print("String sort (case-insensitive):", table.concat(words, ", "))
```

```lua
-- ตัวอย่างที่ 10: sort complex objects
local students = {
    {name = "Charlie", grade = 85, age = 20},
    {name = "Alice",   grade = 92, age = 19},
    {name = "Bob",     grade = 85, age = 21},
    {name = "Diana",   grade = 78, age = 18},
    {name = "Eve",     grade = 92, age = 20},
}

-- Sort by grade (descending), then by name (ascending)
table.sort(students, function(a, b)
    if a.grade ~= b.grade then
        return a.grade > b.grade  -- grade สูงกว่าขึ้นก่อน
    end
    return a.name < b.name  -- ถ้า grade เท่ากัน sort ตามชื่อ
end)

print("=== Sorted Students ===")
print(string.format("%-10s %-8s %-5s", "Name", "Grade", "Age"))
print(string.rep("-", 25))
for _, s in ipairs(students) do
    print(string.format("%-10s %-8d %-5d", s.name, s.grade, s.age))
end
```

```lua
-- ตัวอย่างที่ 11: Stable sort (รักษาลำดับเมื่อ key เท่ากัน)
local function stable_sort(t, comp)
    -- เพิ่ม original index
    local indexed = {}
    for i, v in ipairs(t) do
        indexed[i] = {value = v, index = i}
    end
    
    table.sort(indexed, function(a, b)
        local result = comp(a.value, b.value)
        if result == nil or (not comp(a.value, b.value) and not comp(b.value, a.value)) then
            -- Elements are equal, preserve original order
            return a.index < b.index
        end
        return result
    end)
    
    -- คืนค่ากลับ
    for i, item in ipairs(indexed) do
        t[i] = item.value
    end
    return t
end

local items = {
    {name = "A", priority = 1},
    {name = "B", priority = 2},
    {name = "C", priority = 1},
    {name = "D", priority = 2},
    {name = "E", priority = 1},
}

stable_sort(items, function(a, b) return a.priority < b.priority end)
print("=== Stable Sort ===")
for _, item in ipairs(items) do
    print(string.format("  %s (priority=%d)", item.name, item.priority))
end
```

```lua
-- ตัวอย่างที่ 12: Sort by multiple criteria
local function multi_sort(t, ...)
    local criteria = {...}
    
    table.sort(t, function(a, b)
        for _, criterion in ipairs(criteria) do
            local key, reverse = criterion[1], criterion[2]
            local va, vb
            
            if type(key) == "function" then
                va, vb = key(a), key(b)
            else
                va, vb = a[key], b[key]
            end
            
            if va ~= vb then
                if reverse then
                    return va > vb
                else
                    return va < vb
                end
            end
        end
        return false
    end)
    
    return t
end

local data = {
    {dept = "HR",  name = "Charlie", salary = 50000},
    {dept = "IT",  name = "Alice",   salary = 75000},
    {dept = "HR",  name = "Alice",   salary = 55000},
    {dept = "IT",  name = "Bob",     salary = 75000},
    {dept = "HR",  name = "Bob",     salary = 60000},
}

multi_sort(data, {"dept", false}, {"salary", true}, {"name", false})

print("=== Multi-criteria Sort ===")
print(string.format("%-6s %-10s %-10s", "Dept", "Name", "Salary"))
print(string.rep("-", 30))
for _, d in ipairs(data) do
    print(string.format("%-6s %-10s %-10d", d.dept, d.name, d.salary))
end
```

---

## table.concat

```lua
-- ตัวอย่างที่ 13: table.concat พื้นฐาน
local words = {"Hello", "World", "from", "Lua"}

-- concat ด้วย space
print(table.concat(words, " "))

-- concat ด้วย comma
print(table.concat(words, ", "))

-- concat บางส่วน (i, j)
print(table.concat(words, "-", 2, 3))  -- "World-from"

-- concat ตัวเลข
local numbers = {1, 2, 3, 4, 5}
print(table.concat(numbers, " + ") .. " = " .. (1+2+3+4+5))
```

```lua
-- ตัวอย่างที่ 14: สร้าง string builder ด้วย table.concat
local StringBuilder = {}
StringBuilder.__index = StringBuilder

function StringBuilder.new()
    return setmetatable({_parts = {}, _size = 0}, StringBuilder)
end

function StringBuilder:append(s)
    self._parts[#self._parts + 1] = tostring(s)
    self._size = self._size + #tostring(s)
    return self  -- method chaining
end

function StringBuilder:append_line(s)
    return self:append(s or ""):append("\n")
end

function StringBuilder:prepend(s)
    table.insert(self._parts, 1, tostring(s))
    self._size = self._size + #tostring(s)
    return self
end

function StringBuilder:to_string(sep)
    return table.concat(self._parts, sep or "")
end

function StringBuilder:clear()
    self._parts = {}
    self._size = 0
    return self
end

function StringBuilder:length()
    return self._size
end

-- ทดสอบ
local sb = StringBuilder.new()
sb:append("Hello"):append(", "):append("World"):append_line("!")
sb:append_line("Lua is awesome")
sb:append("Version: "):append(5.4)

print("=== StringBuilder ===")
print(sb:to_string())
print("Length:", sb:length())
```

```lua
-- ตัวอย่างที่ 15: สร้าง HTML builder
local HtmlBuilder = {}

function HtmlBuilder.tag(name, content, attrs)
    local attr_str = ""
    if attrs then
        for k, v in pairs(attrs) do
            attr_str = attr_str .. string.format(' %s="%s"', k, v)
        end
    end
    
    if content then
        return string.format("<%s%s>%s</%s>", name, attr_str, content, name)
    else
        return string.format("<%s%s/>", name, attr_str)
    end
end

function HtmlBuilder.table(data, headers)
    local rows = {}
    
    -- Header row
    if headers then
        local cells = {}
        for _, h in ipairs(headers) do
            table.insert(cells, HtmlBuilder.tag("th", h))
        end
        table.insert(rows, HtmlBuilder.tag("tr", table.concat(cells)))
    end
    
    -- Data rows
    for _, row in ipairs(data) do
        local cells = {}
        for _, cell in ipairs(row) do
            table.insert(cells, HtmlBuilder.tag("td", tostring(cell)))
        end
        table.insert(rows, HtmlBuilder.tag("tr", table.concat(cells)))
    end
    
    return HtmlBuilder.tag("table", "\n" .. table.concat(rows, "\n") .. "\n")
end

-- ทดสอบ
local headers = {"Name", "Score", "Grade"}
local data = {
    {"Alice", 95, "A"},
    {"Bob", 78, "C+"},
    {"Charlie", 88, "B+"},
}

print("=== HTML Table ===")
print(HtmlBuilder.table(data, headers))
```

---

## table.move

```lua
-- ตัวอย่างที่ 16: table.move พื้นฐาน
-- table.move(a1, f, e, t [,a2])
-- คัดลอก a1[f..e] ไปยัง a2[t..]

local source = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
local dest = {}

-- คัดลอก elements 3-7 ไปยัง dest เริ่มจาก index 1
table.move(source, 3, 7, 1, dest)
print("Copy to new table:", table.concat(dest, ", "))

-- เลื่อน elements ภายใน table เดียวกัน (shift right)
local t = {1, 2, 3, 4, 5}
table.move(t, 1, #t, 2)  -- เลื่อนทุก element ไปขวา 1
t[1] = 0  -- ใส่ค่าใหม่ที่ตำแหน่ง 1
print("After shift right:", table.concat(t, ", "))

-- เลื่อนไปซ้าย (shift left) - ลบตัวแรก
local t2 = {1, 2, 3, 4, 5}
table.move(t2, 2, #t2, 1)  -- เลื่อนไปซ้าย 1
t2[#t2] = nil  -- ลบตัวสุดท้าย (ที่ซ้ำกัน)
print("After shift left:", table.concat(t2, ", "))
```

```lua
-- ตัวอย่างที่ 17: Array slice ด้วย table.move
local function slice(t, start, stop, step)
    start = start or 1
    stop = stop or #t
    step = step or 1
    
    if start < 0 then start = #t + start + 1 end
    if stop < 0 then stop = #t + stop + 1 end
    
    local result = {}
    if step == 1 then
        -- เร็วกว่าด้วย table.move
        table.move(t, start, stop, 1, result)
    else
        local j = 1
        for i = start, stop, step do
            result[j] = t[i]
            j = j + 1
        end
    end
    return result
end

local arr = {10, 20, 30, 40, 50, 60, 70, 80, 90, 100}
print("=== Array Slice ===")
print("Original:", table.concat(arr, ", "))
print("slice(3,7):", table.concat(slice(arr, 3, 7), ", "))
print("slice(-3):", table.concat(slice(arr, -3), ", "))
print("slice(1,10,2):", table.concat(slice(arr, 1, 10, 2), ", "))
```

```lua
-- ตัวอย่างที่ 18: table.move สำหรับ rotating array
local function rotate_left(t, n)
    n = n % #t
    if n == 0 then return t end
    
    local result = {}
    table.move(t, n + 1, #t, 1, result)
    table.move(t, 1, n, #t - n + 1, result)
    return result
end

local function rotate_right(t, n)
    return rotate_left(t, #t - (n % #t))
end

local arr = {1, 2, 3, 4, 5, 6, 7}
print("=== Array Rotation ===")
print("Original:", table.concat(arr, ", "))
print("Rotate left 2:", table.concat(rotate_left(arr, 2), ", "))
print("Rotate right 3:", table.concat(rotate_right(arr, 3), ", "))
```

---

## table.pack และ table.unpack

```lua
-- ตัวอย่างที่ 19: table.pack
local packed = table.pack(10, 20, 30, "hello", true)
print("=== table.pack ===")
print("n (count):", packed.n)  -- 5
for i = 1, packed.n do
    print(i, packed[i])
end
```

```lua
-- ตัวอย่างที่ 20: table.unpack
local t = {100, 200, 300, 400, 500}

-- unpack ทั้ง table
print("=== table.unpack ===")
print(table.unpack(t))  -- 100 200 300 400 500

-- unpack บางส่วน
print(table.unpack(t, 2, 4))  -- 200 300 400

-- ใช้กับ function call
local function sum(a, b, c)
    return a + b + c
end

local args = {10, 20, 30}
print("Sum:", sum(table.unpack(args)))
```

```lua
-- ตัวอย่างที่ 21: variadic function patterns
local function variadic_demo(...)
    local args = table.pack(...)
    print("จำนวน arguments:", args.n)
    
    -- ประมวลผลรวมถึง nil
    for i = 1, args.n do
        print(i, args[i])  -- แสดงแม้จะเป็น nil
    end
end

variadic_demo(1, nil, 3, nil, 5)

-- สำคัญ: select('#', ...) นับรวม nil
local function count_with_nil(...)
    local n = select('#', ...)
    print("Count (with nil):", n)
    local packed = table.pack(...)
    print("Count (table.pack):", packed.n)
end

count_with_nil(1, nil, 3, nil, 5)
```

```lua
-- ตัวอย่างที่ 22: ส่ง table เป็น arguments
local function apply(func, args)
    return func(table.unpack(args, 1, args.n or #args))
end

local function add(a, b) return a + b end
local function multiply(a, b, c) return a * b * c end

print("=== Apply Function ===")
print(apply(add, {10, 20}))
print(apply(multiply, {2, 3, 4}))

-- Curry-like pattern
local function curry(func, ...)
    local saved_args = table.pack(...)
    return function(...)
        local new_args = table.pack(...)
        local combined = {}
        for i = 1, saved_args.n do combined[i] = saved_args[i] end
        for i = 1, new_args.n do combined[saved_args.n + i] = new_args[i] end
        combined.n = saved_args.n + new_args.n
        return func(table.unpack(combined, 1, combined.n))
    end
end

local add10 = curry(add, 10)
print("add10(5):", add10(5))
print("add10(20):", add10(20))
```

---

## Sorting Algorithms

```lua
-- ตัวอย่างที่ 23: Bubble Sort
local function bubble_sort(t, comp)
    comp = comp or function(a, b) return a < b end
    local n = #t
    local swapped
    
    for i = 1, n - 1 do
        swapped = false
        for j = 1, n - i do
            if not comp(t[j], t[j+1]) then
                t[j], t[j+1] = t[j+1], t[j]
                swapped = true
            end
        end
        if not swapped then break end  -- ออกเร็วถ้า sorted แล้ว
    end
    
    return t
end

local arr = {64, 34, 25, 12, 22, 11, 90}
print("=== Bubble Sort ===")
print("Before:", table.concat(arr, ", "))
bubble_sort(arr)
print("After:", table.concat(arr, ", "))
```

```lua
-- ตัวอย่างที่ 24: Quick Sort
local function quick_sort(t, lo, hi, comp)
    comp = comp or function(a, b) return a < b end
    lo = lo or 1
    hi = hi or #t
    
    if lo >= hi then return end
    
    -- Partition (Lomuto scheme)
    local pivot = t[hi]
    local i = lo - 1
    
    for j = lo, hi - 1 do
        if comp(t[j], pivot) then
            i = i + 1
            t[i], t[j] = t[j], t[i]
        end
    end
    
    i = i + 1
    t[i], t[hi] = t[hi], t[i]
    
    quick_sort(t, lo, i - 1, comp)
    quick_sort(t, i + 1, hi, comp)
    
    return t
end

local arr = {3, 6, 8, 10, 1, 2, 1}
print("=== Quick Sort ===")
print("Before:", table.concat(arr, ", "))
quick_sort(arr)
print("After:", table.concat(arr, ", "))

-- Quick sort descending
local arr2 = {5, 2, 9, 1, 7, 3, 8, 4, 6}
quick_sort(arr2, 1, #arr2, function(a, b) return a > b end)
print("Descending:", table.concat(arr2, ", "))
```

```lua
-- ตัวอย่างที่ 25: Merge Sort
local function merge_sort(t, comp)
    comp = comp or function(a, b) return a <= b end
    local n = #t
    
    if n <= 1 then return t end
    
    local function merge(left, right)
        local result = {}
        local i, j = 1, 1
        
        while i <= #left and j <= #right do
            if comp(left[i], right[j]) then
                table.insert(result, left[i])
                i = i + 1
            else
                table.insert(result, right[j])
                j = j + 1
            end
        end
        
        -- เพิ่มส่วนที่เหลือ
        while i <= #left do
            table.insert(result, left[i])
            i = i + 1
        end
        while j <= #right do
            table.insert(result, right[j])
            j = j + 1
        end
        
        return result
    end
    
    local function do_sort(arr)
        if #arr <= 1 then return arr end
        
        local mid = math.floor(#arr / 2)
        local left = {}
        local right = {}
        
        for i = 1, mid do left[i] = arr[i] end
        for i = mid + 1, #arr do right[i - mid] = arr[i] end
        
        return merge(do_sort(left), do_sort(right))
    end
    
    local sorted = do_sort(t)
    for i, v in ipairs(sorted) do t[i] = v end
    return t
end

local arr = {38, 27, 43, 3, 9, 82, 10}
print("=== Merge Sort ===")
print("Before:", table.concat(arr, ", "))
merge_sort(arr)
print("After:", table.concat(arr, ", "))
```

```lua
-- ตัวอย่างที่ 26: Insertion Sort (ดีสำหรับข้อมูลที่เกือบ sorted แล้ว)
local function insertion_sort(t, comp)
    comp = comp or function(a, b) return a < b end
    
    for i = 2, #t do
        local key = t[i]
        local j = i - 1
        
        while j >= 1 and not comp(t[j], key) do
            t[j + 1] = t[j]
            j = j - 1
        end
        
        t[j + 1] = key
    end
    
    return t
end

local arr = {5, 3, 4, 1, 2}
print("=== Insertion Sort ===")
print("Before:", table.concat(arr, ", "))
insertion_sort(arr)
print("After:", table.concat(arr, ", "))
```

```lua
-- ตัวอย่างที่ 27: Counting Sort (สำหรับ integer range ที่รู้ล่วงหน้า)
local function counting_sort(t, min_val, max_val)
    min_val = min_val or math.min(table.unpack(t))
    max_val = max_val or math.max(table.unpack(t))
    
    local count = {}
    for i = min_val, max_val do count[i] = 0 end
    
    for _, v in ipairs(t) do
        count[v] = count[v] + 1
    end
    
    local sorted = {}
    for i = min_val, max_val do
        for _ = 1, count[i] do
            table.insert(sorted, i)
        end
    end
    
    return sorted
end

local arr = {4, 2, 2, 8, 3, 3, 1, 7, 5}
print("=== Counting Sort ===")
print("Before:", table.concat(arr, ", "))
local sorted = counting_sort(arr, 1, 8)
print("After:", table.concat(sorted, ", "))
```

---

## Binary Search

```lua
-- ตัวอย่างที่ 28: Binary Search พื้นฐาน
local function binary_search(t, target, comp)
    comp = comp or function(a, b) return a < b end
    
    local lo, hi = 1, #t
    
    while lo <= hi do
        local mid = math.floor((lo + hi) / 2)
        
        if t[mid] == target then
            return mid  -- พบ! คืน index
        elseif comp(t[mid], target) then
            lo = mid + 1  -- ค้นครึ่งขวา
        else
            hi = mid - 1  -- ค้นครึ่งซ้าย
        end
    end
    
    return nil  -- ไม่พบ
end

local sorted_arr = {1, 3, 5, 7, 9, 11, 13, 15, 17, 19}
print("=== Binary Search ===")
print("Array:", table.concat(sorted_arr, ", "))
print("Search 7:", binary_search(sorted_arr, 7))    -- 4
print("Search 1:", binary_search(sorted_arr, 1))    -- 1
print("Search 19:", binary_search(sorted_arr, 19))  -- 10
print("Search 4:", binary_search(sorted_arr, 4))    -- nil
```

```lua
-- ตัวอย่างที่ 29: Binary Search - หา lower bound และ upper bound
local function lower_bound(t, target)
    -- หา index แรกที่ t[i] >= target
    local lo, hi = 1, #t + 1
    
    while lo < hi do
        local mid = math.floor((lo + hi) / 2)
        if t[mid] < target then
            lo = mid + 1
        else
            hi = mid
        end
    end
    
    return lo
end

local function upper_bound(t, target)
    -- หา index แรกที่ t[i] > target
    local lo, hi = 1, #t + 1
    
    while lo < hi do
        local mid = math.floor((lo + hi) / 2)
        if t[mid] <= target then
            lo = mid + 1
        else
            hi = mid
        end
    end
    
    return lo
end

local arr = {1, 2, 2, 2, 3, 4, 5, 5, 6}
print("=== Bounds ===")
print("Array:", table.concat(arr, ", "))
print("Lower bound of 2:", lower_bound(arr, 2))  -- 2
print("Upper bound of 2:", upper_bound(arr, 2))  -- 5
print("Count of 2:", upper_bound(arr, 2) - lower_bound(arr, 2))  -- 3
print("Count of 5:", upper_bound(arr, 5) - lower_bound(arr, 5))  -- 2
```

```lua
-- ตัวอย่างที่ 30: Binary Search บน sorted objects
local function binary_search_by(t, key_func, target)
    local lo, hi = 1, #t
    
    while lo <= hi do
        local mid = math.floor((lo + hi) / 2)
        local key = key_func(t[mid])
        
        if key == target then
            return mid
        elseif key < target then
            lo = mid + 1
        else
            hi = mid - 1
        end
    end
    
    return nil
end

local employees = {
    {id = 101, name = "Alice"},
    {id = 205, name = "Bob"},
    {id = 309, name = "Charlie"},
    {id = 412, name = "Diana"},
    {id = 507, name = "Eve"},
}

local idx = binary_search_by(employees, function(e) return e.id end, 309)
if idx then
    print("Found:", employees[idx].name, "at index", idx)
else
    print("Not found")
end
```

---

## Functional Operations

```lua
-- ตัวอย่างที่ 31: map - แปลงทุก element
local function map(t, func)
    local result = {}
    for i, v in ipairs(t) do
        result[i] = func(v, i)
    end
    return result
end

local numbers = {1, 2, 3, 4, 5}
print("=== map ===")
print("Original:", table.concat(numbers, ", "))
print("*2:", table.concat(map(numbers, function(x) return x * 2 end), ", "))
print("^2:", table.concat(map(numbers, function(x) return x^2 end), ", "))
print("str:", table.concat(map(numbers, function(x) return "n" .. x end), ", "))
```

```lua
-- ตัวอย่างที่ 32: filter - กรอง elements
local function filter(t, predicate)
    local result = {}
    for _, v in ipairs(t) do
        if predicate(v) then
            table.insert(result, v)
        end
    end
    return result
end

local numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
print("=== filter ===")
print("Evens:", table.concat(
    filter(numbers, function(x) return x % 2 == 0 end), ", "))
print("Odds:", table.concat(
    filter(numbers, function(x) return x % 2 == 1 end), ", "))
print(">5:", table.concat(
    filter(numbers, function(x) return x > 5 end), ", "))
```

```lua
-- ตัวอย่างที่ 33: reduce - รวม elements เป็นค่าเดียว
local function reduce(t, func, initial)
    local acc = initial
    local start = 1
    
    if acc == nil then
        if #t == 0 then return nil end
        acc = t[1]
        start = 2
    end
    
    for i = start, #t do
        acc = func(acc, t[i], i)
    end
    
    return acc
end

local numbers = {1, 2, 3, 4, 5}
print("=== reduce ===")
print("Sum:", reduce(numbers, function(a, b) return a + b end))
print("Product:", reduce(numbers, function(a, b) return a * b end))
print("Max:", reduce(numbers, function(a, b) return math.max(a, b) end))
print("Sum with initial 100:", reduce(numbers, function(a, b) return a + b end, 100))

-- สร้าง sentence จาก words
local words = {"Lua", "is", "a", "great", "language"}
local sentence = reduce(words, function(a, b) return a .. " " .. b end)
print("Sentence:", sentence)
```

```lua
-- ตัวอย่างที่ 34: flatten - ทำให้ nested array เป็น flat
local function flatten(t, depth)
    depth = depth or math.huge
    local result = {}
    
    local function do_flatten(arr, current_depth)
        for _, v in ipairs(arr) do
            if type(v) == "table" and current_depth > 0 then
                do_flatten(v, current_depth - 1)
            else
                table.insert(result, v)
            end
        end
    end
    
    do_flatten(t, depth)
    return result
end

local nested = {1, {2, 3}, {4, {5, 6}}, {7, {8, {9, 10}}}}
print("=== flatten ===")
print("Original:", require and "nested" or "nested")

local flat = flatten(nested)
print("Flat:", table.concat(flat, ", "))

local flat1 = flatten(nested, 1)
-- manually show since nested
local strs = {}
for _, v in ipairs(flat1) do
    strs[#strs+1] = type(v) == "table" and "{...}" or tostring(v)
end
print("Depth 1:", table.concat(strs, ", "))
```

```lua
-- ตัวอย่างที่ 35: zip - รวมหลาย array
local function zip(...)
    local arrays = {...}
    local result = {}
    local min_len = math.huge
    
    for _, arr in ipairs(arrays) do
        if #arr < min_len then min_len = #arr end
    end
    
    for i = 1, min_len do
        local row = {}
        for _, arr in ipairs(arrays) do
            table.insert(row, arr[i])
        end
        table.insert(result, row)
    end
    
    return result
end

local names = {"Alice", "Bob", "Charlie"}
local scores = {95, 78, 88}
local grades = {"A", "C+", "B+"}

print("=== zip ===")
local zipped = zip(names, scores, grades)
for _, row in ipairs(zipped) do
    print(string.format("  %-10s %3d  %s", row[1], row[2], row[3]))
end
```

```lua
-- ตัวอย่างที่ 36: chain operations (pipeline)
local function pipeline(data, ...)
    local result = data
    for _, func in ipairs({...}) do
        result = func(result)
    end
    return result
end

local numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

local result = pipeline(numbers,
    function(t) return filter(t, function(x) return x % 2 == 0 end) end,
    function(t) return map(t, function(x) return x * x end) end,
    function(t) return reduce(t, function(a, b) return a + b end, 0) end
)

print("Sum of squares of even numbers:", result)
-- 4 + 16 + 36 + 64 + 100 = 220
```

```lua
-- ตัวอย่างที่ 37: forEach, every, some, find
local function forEach(t, func)
    for i, v in ipairs(t) do
        func(v, i)
    end
end

local function every(t, predicate)
    for _, v in ipairs(t) do
        if not predicate(v) then return false end
    end
    return true
end

local function some(t, predicate)
    for _, v in ipairs(t) do
        if predicate(v) then return true end
    end
    return false
end

local function find(t, predicate)
    for i, v in ipairs(t) do
        if predicate(v) then return v, i end
    end
    return nil
end

local nums = {2, 4, 6, 8, 10}
print("=== Array predicates ===")
print("every even:", every(nums, function(x) return x % 2 == 0 end))
print("some > 5:", some(nums, function(x) return x > 5 end))
print("some odd:", some(nums, function(x) return x % 2 == 1 end))

local first_gt_5, idx = find(nums, function(x) return x > 5 end)
print("First > 5:", first_gt_5, "at index", idx)
```

---

## Table Utilities

```lua
-- ตัวอย่างที่ 38: keys(), values()
local function keys(t)
    local result = {}
    for k in pairs(t) do
        table.insert(result, k)
    end
    table.sort(result, function(a, b)
        return tostring(a) < tostring(b)
    end)
    return result
end

local function values(t)
    local result = {}
    for _, v in pairs(t) do
        table.insert(result, v)
    end
    return result
end

local person = {name = "Alice", age = 30, city = "Bangkok"}
print("=== keys/values ===")
print("Keys:", table.concat(keys(person), ", "))

local vals = values(person)
local str_vals = {}
for _, v in ipairs(vals) do str_vals[#str_vals+1] = tostring(v) end
print("Values:", table.concat(str_vals, ", "))
```

```lua
-- ตัวอย่างที่ 39: contains(), indexOf()
local function contains(t, value)
    for _, v in ipairs(t) do
        if v == value then return true end
    end
    return false
end

local function indexOf(t, value, from)
    from = from or 1
    for i = from, #t do
        if t[i] == value then return i end
    end
    return -1
end

local function lastIndexOf(t, value)
    for i = #t, 1, -1 do
        if t[i] == value then return i end
    end
    return -1
end

local fruits = {"apple", "banana", "cherry", "banana", "date"}
print("=== contains/indexOf ===")
print("contains 'banana':", contains(fruits, "banana"))
print("contains 'grape':", contains(fruits, "grape"))
print("indexOf 'banana':", indexOf(fruits, "banana"))
print("indexOf 'banana' from 3:", indexOf(fruits, "banana", 3))
print("lastIndexOf 'banana':", lastIndexOf(fruits, "banana"))
```

```lua
-- ตัวอย่างที่ 40: unique() - ลบ duplicates
local function unique(t, key_func)
    local seen = {}
    local result = {}
    
    for _, v in ipairs(t) do
        local key = key_func and key_func(v) or v
        if not seen[key] then
            seen[key] = true
            table.insert(result, v)
        end
    end
    
    return result
end

local with_dups = {1, 2, 3, 2, 4, 3, 5, 1, 6}
print("=== unique ===")
print("With duplicates:", table.concat(with_dups, ", "))
print("Unique:", table.concat(unique(with_dups), ", "))

-- Unique ด้วย custom key
local items = {
    {id = 1, name = "A"},
    {id = 2, name = "B"},
    {id = 1, name = "A duplicate"},
    {id = 3, name = "C"},
}
local unique_items = unique(items, function(x) return x.id end)
print("Unique items count:", #unique_items)
for _, item in ipairs(unique_items) do
    print("  id=" .. item.id .. " name=" .. item.name)
end
```

```lua
-- ตัวอย่างที่ 41: count(), sum(), avg()
local function count(t, predicate)
    if not predicate then return #t end
    local c = 0
    for _, v in ipairs(t) do
        if predicate(v) then c = c + 1 end
    end
    return c
end

local function sum(t, key_func)
    local total = 0
    for _, v in ipairs(t) do
        total = total + (key_func and key_func(v) or v)
    end
    return total
end

local function avg(t, key_func)
    if #t == 0 then return 0 end
    return sum(t, key_func) / #t
end

local function min_by(t, key_func)
    if #t == 0 then return nil end
    local min_item = t[1]
    local min_val = key_func(t[1])
    for i = 2, #t do
        local val = key_func(t[i])
        if val < min_val then
            min_val = val
            min_item = t[i]
        end
    end
    return min_item, min_val
end

local function max_by(t, key_func)
    if #t == 0 then return nil end
    local max_item = t[1]
    local max_val = key_func(t[1])
    for i = 2, #t do
        local val = key_func(t[i])
        if val > max_val then
            max_val = val
            max_item = t[i]
        end
    end
    return max_item, max_val
end

local numbers = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3}
print("=== Statistics ===")
print("Count:", count(numbers))
print("Count > 4:", count(numbers, function(x) return x > 4 end))
print("Sum:", sum(numbers))
print("Avg:", avg(numbers))

local students = {
    {name="Alice", score=85}, {name="Bob", score=92},
    {name="Charlie", score=78}, {name="Diana", score=96}
}

local min_student, min_score = min_by(students, function(s) return s.score end)
local max_student, max_score = max_by(students, function(s) return s.score end)
print("\nLowest:", min_student.name, min_score)
print("Highest:", max_student.name, max_score)
print("Average score:", avg(students, function(s) return s.score end))
```

---

## Deep Copy และ Shallow Copy

```lua
-- ตัวอย่างที่ 42: Shallow copy
local function shallow_copy(t)
    local copy = {}
    for k, v in pairs(t) do
        copy[k] = v
    end
    return copy
end

-- ทดสอบ
local original = {1, 2, 3, nested = {4, 5, 6}}
local shallow = shallow_copy(original)

-- แก้ไข shallow copy
shallow[1] = 100
shallow.nested[1] = 400  -- กระทบ original ด้วย!

print("=== Shallow Copy ===")
print("Original[1]:", original[1])           -- 1 (ไม่เปลี่ยน)
print("Shallow[1]:", shallow[1])             -- 100
print("Original nested[1]:", original.nested[1])  -- 400 (เปลี่ยน!)
print("Shallow nested[1]:", shallow.nested[1])    -- 400
```

```lua
-- ตัวอย่างที่ 43: Deep copy
local function deep_copy(t, seen)
    seen = seen or {}
    
    if type(t) ~= "table" then return t end
    if seen[t] then return seen[t] end  -- ป้องกัน circular reference
    
    local copy = {}
    seen[t] = copy
    
    for k, v in pairs(t) do
        copy[deep_copy(k, seen)] = deep_copy(v, seen)
    end
    
    -- คัดลอก metatable
    local mt = getmetatable(t)
    if mt then
        setmetatable(copy, mt)
    end
    
    return copy
end

-- ทดสอบ
local original = {
    1, 2, 3,
    nested = {4, 5, {6, 7}},
    info = {name = "test", data = {1, 2, 3}}
}

local deep = deep_copy(original)
deep[1] = 100
deep.nested[1] = 400
deep.nested[3][1] = 600
deep.info.name = "modified"

print("=== Deep Copy ===")
print("Original[1]:", original[1])                    -- 1
print("Original nested[1]:", original.nested[1])      -- 4
print("Original nested[3][1]:", original.nested[3][1]) -- 6
print("Original info.name:", original.info.name)       -- test

print("Deep[1]:", deep[1])                             -- 100
print("Deep nested[1]:", deep.nested[1])               -- 400
print("Deep info.name:", deep.info.name)               -- modified
```

```lua
-- ตัวอย่างที่ 44: Deep copy กับ circular reference
local a = {name = "a"}
local b = {name = "b"}
a.ref = b
b.ref = a  -- circular!

local copy_a = deep_copy(a)
print("=== Circular Reference Deep Copy ===")
print("copy_a.name:", copy_a.name)
print("copy_a.ref.name:", copy_a.ref.name)
print("copy_a.ref.ref.name:", copy_a.ref.ref.name)  -- ไม่ infinite loop
```

---

## Merge และ Diff Tables

```lua
-- ตัวอย่างที่ 45: Merge tables
local function merge(...)
    local result = {}
    for _, t in ipairs({...}) do
        for k, v in pairs(t) do
            result[k] = v
        end
    end
    return result
end

-- Merge แบบ deep
local function deep_merge(base, override)
    local result = deep_copy(base)
    
    for k, v in pairs(override) do
        if type(v) == "table" and type(result[k]) == "table" then
            result[k] = deep_merge(result[k], v)
        else
            result[k] = v
        end
    end
    
    return result
end

local default_config = {
    server = {host = "localhost", port = 8080, timeout = 30},
    db = {host = "localhost", port = 5432},
    debug = false,
}

local user_config = {
    server = {port = 9000, timeout = 60},  -- override port, timeout
    db = {host = "db.example.com"},         -- override host
    debug = true,
}

local final = deep_merge(default_config, user_config)
print("=== Deep Merge ===")
print("server.host:", final.server.host)     -- localhost (from default)
print("server.port:", final.server.port)     -- 9000 (overridden)
print("server.timeout:", final.server.timeout) -- 60 (overridden)
print("db.host:", final.db.host)             -- db.example.com
print("db.port:", final.db.port)             -- 5432 (from default)
print("debug:", final.debug)                 -- true
```

```lua
-- ตัวอย่างที่ 46: Diff tables
local function diff(t1, t2)
    local added = {}
    local removed = {}
    local changed = {}
    
    -- หา added และ changed
    for k, v in pairs(t2) do
        if t1[k] == nil then
            added[k] = v
        elseif t1[k] ~= v then
            changed[k] = {old = t1[k], new = v}
        end
    end
    
    -- หา removed
    for k, v in pairs(t1) do
        if t2[k] == nil then
            removed[k] = v
        end
    end
    
    return {added = added, removed = removed, changed = changed}
end

local v1 = {a = 1, b = 2, c = 3, d = 4}
local v2 = {a = 1, b = 20, c = 3, e = 5}  -- b changed, d removed, e added

local d = diff(v1, v2)
print("=== Diff ===")
print("Added:")
for k, v in pairs(d.added) do
    print(string.format("  + %s = %s", k, v))
end
print("Removed:")
for k, v in pairs(d.removed) do
    print(string.format("  - %s = %s", k, v))
end
print("Changed:")
for k, v in pairs(d.changed) do
    print(string.format("  ~ %s: %s -> %s", k, v.old, v.new))
end
```

---

## Group By และ Partition

```lua
-- ตัวอย่างที่ 47: groupBy
local function group_by(t, key_func)
    local result = {}
    
    for _, v in ipairs(t) do
        local key = key_func(v)
        if not result[key] then
            result[key] = {}
        end
        table.insert(result[key], v)
    end
    
    return result
end

local people = {
    {name = "Alice",   dept = "IT",  age = 25},
    {name = "Bob",     dept = "HR",  age = 30},
    {name = "Charlie", dept = "IT",  age = 28},
    {name = "Diana",   dept = "HR",  age = 22},
    {name = "Eve",     dept = "IT",  age = 35},
    {name = "Frank",   dept = "Finance", age = 40},
}

print("=== Group By Department ===")
local by_dept = group_by(people, function(p) return p.dept end)
for dept, members in pairs(by_dept) do
    local names = {}
    for _, m in ipairs(members) do table.insert(names, m.name) end
    print(string.format("  %-10s: %s", dept, table.concat(names, ", ")))
end

print("\n=== Group By Age Range ===")
local by_age = group_by(people, function(p)
    if p.age < 25 then return "young"
    elseif p.age < 35 then return "mid"
    else return "senior" end
end)

for group, members in pairs(by_age) do
    local names = {}
    for _, m in ipairs(members) do table.insert(names, m.name) end
    print(string.format("  %-10s: %s", group, table.concat(names, ", ")))
end
```

```lua
-- ตัวอย่างที่ 48: partition - แบ่ง array เป็น 2 กลุ่ม
local function partition(t, predicate)
    local pass = {}
    local fail = {}
    
    for _, v in ipairs(t) do
        if predicate(v) then
            table.insert(pass, v)
        else
            table.insert(fail, v)
        end
    end
    
    return pass, fail
end

local numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
local evens, odds = partition(numbers, function(x) return x % 2 == 0 end)
print("=== Partition ===")
print("Evens:", table.concat(evens, ", "))
print("Odds:", table.concat(odds, ", "))

-- Partition objects
local students = {
    {name = "A", score = 85}, {name = "B", score = 45},
    {name = "C", score = 72}, {name = "D", score = 55},
    {name = "E", score = 91}, {name = "F", score = 38},
}

local passed, failed = partition(students, function(s) return s.score >= 60 end)
print("\nPassed:")
for _, s in ipairs(passed) do print("  " .. s.name .. ": " .. s.score) end
print("Failed:")
for _, s in ipairs(failed) do print("  " .. s.name .. ": " .. s.score) end
```

```lua
-- ตัวอย่างที่ 49: chunk - แบ่ง array เป็น chunks เล็กๆ
local function chunk(t, size)
    local result = {}
    local current = {}
    
    for i, v in ipairs(t) do
        table.insert(current, v)
        if #current == size then
            table.insert(result, current)
            current = {}
        end
    end
    
    if #current > 0 then
        table.insert(result, current)
    end
    
    return result
end

local arr = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
print("=== Chunk ===")
local chunks = chunk(arr, 3)
for i, c in ipairs(chunks) do
    print(string.format("Chunk %d: [%s]", i, table.concat(c, ", ")))
end
```

---

## Advanced Patterns

```lua
-- ตัวอย่างที่ 50: Memoization ด้วย table
local function memoize(func)
    local cache = {}
    return function(...)
        local args = table.pack(...)
        local key = table.concat(map(
            {table.unpack(args, 1, args.n)},
            function(v) return tostring(v) end
        ), ",")
        
        if cache[key] == nil then
            cache[key] = func(...)
        end
        
        return cache[key]
    end
end

-- ทดสอบ Fibonacci ด้วย memoization
local fib
fib = memoize(function(n)
    if n <= 1 then return n end
    return fib(n - 1) + fib(n - 2)
end)

print("=== Memoized Fibonacci ===")
local start = os.clock()
for i = 0, 35 do
    io.write(fib(i) .. " ")
end
print()
print(string.format("Time: %.4f sec", os.clock() - start))
```

```lua
-- ตัวอย่างที่ 51: Set operations
local Set = {}
Set.__index = Set

function Set.new(t)
    local s = setmetatable({_data = {}}, Set)
    if t then
        for _, v in ipairs(t) do s:add(v) end
    end
    return s
end

function Set:add(v) self._data[v] = true end
function Set:remove(v) self._data[v] = nil end
function Set:has(v) return self._data[v] == true end

function Set:union(other)
    local result = Set.new()
    for k in pairs(self._data) do result:add(k) end
    for k in pairs(other._data) do result:add(k) end
    return result
end

function Set:intersection(other)
    local result = Set.new()
    for k in pairs(self._data) do
        if other:has(k) then result:add(k) end
    end
    return result
end

function Set:difference(other)
    local result = Set.new()
    for k in pairs(self._data) do
        if not other:has(k) then result:add(k) end
    end
    return result
end

function Set:to_array()
    local arr = {}
    for k in pairs(self._data) do table.insert(arr, k) end
    table.sort(arr)
    return arr
end

function Set:size()
    local n = 0
    for _ in pairs(self._data) do n = n + 1 end
    return n
end

-- ทดสอบ
local A = Set.new({1, 2, 3, 4, 5})
local B = Set.new({3, 4, 5, 6, 7})

print("=== Set Operations ===")
print("A:", table.concat(A:to_array(), ", "))
print("B:", table.concat(B:to_array(), ", "))
print("A ∪ B:", table.concat(A:union(B):to_array(), ", "))
print("A ∩ B:", table.concat(A:intersection(B):to_array(), ", "))
print("A - B:", table.concat(A:difference(B):to_array(), ", "))
print("B - A:", table.concat(B:difference(A):to_array(), ", "))
```

```lua
-- ตัวอย่างที่ 52: Priority Queue
local PriorityQueue = {}
PriorityQueue.__index = PriorityQueue

function PriorityQueue.new(comp)
    return setmetatable({
        _heap = {},
        _comp = comp or function(a, b) return a.priority < b.priority end
    }, PriorityQueue)
end

function PriorityQueue:_sift_up(i)
    while i > 1 do
        local parent = math.floor(i / 2)
        if self._comp(self._heap[i], self._heap[parent]) then
            self._heap[i], self._heap[parent] = self._heap[parent], self._heap[i]
            i = parent
        else
            break
        end
    end
end

function PriorityQueue:_sift_down(i)
    local n = #self._heap
    while true do
        local min_idx = i
        local left = 2 * i
        local right = 2 * i + 1
        
        if left <= n and self._comp(self._heap[left], self._heap[min_idx]) then
            min_idx = left
        end
        if right <= n and self._comp(self._heap[right], self._heap[min_idx]) then
            min_idx = right
        end
        
        if min_idx == i then break end
        
        self._heap[i], self._heap[min_idx] = self._heap[min_idx], self._heap[i]
        i = min_idx
    end
end

function PriorityQueue:push(item)
    table.insert(self._heap, item)
    self:_sift_up(#self._heap)
end

function PriorityQueue:pop()
    if #self._heap == 0 then return nil end
    local top = self._heap[1]
    self._heap[1] = self._heap[#self._heap]
    self._heap[#self._heap] = nil
    if #self._heap > 0 then
        self:_sift_down(1)
    end
    return top
end

function PriorityQueue:peek()
    return self._heap[1]
end

function PriorityQueue:size()
    return #self._heap
end

-- ทดสอบ Task Scheduler
local scheduler = PriorityQueue.new(function(a, b)
    return a.priority < b.priority  -- priority ต่ำ = สำคัญกว่า
end)

local tasks = {
    {name = "Low priority task", priority = 10},
    {name = "Critical task", priority = 1},
    {name = "High priority", priority = 2},
    {name = "Normal task", priority = 5},
    {name = "Urgent task", priority = 1},
}

for _, task in ipairs(tasks) do
    scheduler:push(task)
end

print("=== Priority Queue (Task Scheduler) ===")
while scheduler:size() > 0 do
    local task = scheduler:pop()
    print(string.format("  [P%d] %s", task.priority, task.name))
end
```

```lua
-- ตัวอย่างที่ 53: LRU Cache
local LRUCache = {}
LRUCache.__index = LRUCache

function LRUCache.new(capacity)
    return setmetatable({
        _capacity = capacity,
        _cache = {},
        _order = {},  -- linked list simulation
        _size = 0,
    }, LRUCache)
end

function LRUCache:get(key)
    if not self._cache[key] then return nil end
    -- Move to front (most recently used)
    self:_move_to_front(key)
    return self._cache[key]
end

function LRUCache:put(key, value)
    if self._cache[key] then
        self._cache[key] = value
        self:_move_to_front(key)
    else
        if self._size >= self._capacity then
            -- Evict least recently used (last element)
            local lru_key = self._order[#self._order]
            table.remove(self._order)
            self._cache[lru_key] = nil
            self._size = self._size - 1
        end
        self._cache[key] = value
        table.insert(self._order, 1, key)
        self._size = self._size + 1
    end
end

function LRUCache:_move_to_front(key)
    for i, k in ipairs(self._order) do
        if k == key then
            table.remove(self._order, i)
            break
        end
    end
    table.insert(self._order, 1, key)
end

function LRUCache:info()
    print(string.format("LRU Cache [size=%d/%d]", self._size, self._capacity))
    for i, k in ipairs(self._order) do
        print(string.format("  %d. %s = %s", i, k, tostring(self._cache[k])))
    end
end

-- ทดสอบ
local cache = LRUCache.new(3)
cache:put("a", 1)
cache:put("b", 2)
cache:put("c", 3)
print("=== LRU Cache ===")
cache:info()

cache:get("a")  -- a is now most recent
cache:put("d", 4)  -- evicts b (LRU)
print("\nหลัง get(a) และ put(d):")
cache:info()
print("b is evicted:", cache:get("b") == nil)
```

---

## สรุปบทที่ 11

| ฟังก์ชัน | การใช้งาน |
|---------|-----------|
| `table.insert(t, val)` | เพิ่มต่อท้าย |
| `table.insert(t, pos, val)` | แทรกที่ตำแหน่ง |
| `table.remove(t)` | ลบตัวสุดท้าย |
| `table.remove(t, pos)` | ลบที่ตำแหน่ง |
| `table.sort(t, comp)` | เรียงลำดับ |
| `table.concat(t, sep, i, j)` | รวมเป็น string |
| `table.move(a1, f, e, t, a2)` | คัดลอก/เลื่อน elements |
| `table.pack(...)` | บันทึก varargs |
| `table.unpack(t, i, j)` | กระจาย table |

> **Tip:** สำหรับ array ขนาดใหญ่ การใช้ `t[#t+1] = v` เร็วกว่า `table.insert(t, v)` เพราะไม่ต้องคำนวณ length ทุกครั้ง
