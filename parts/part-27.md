# บทที่ 27: Data Structures ใน Lua

## บทนำ

Data Structures (โครงสร้างข้อมูล) คือวิธีจัดระเบียบข้อมูลในหน่วยความจำเพื่อให้เข้าถึงและประมวลผลได้อย่างมีประสิทธิภาพ Lua table มีความยืดหยุ่นสูงพอที่จะสร้าง data structures ได้ทุกรูปแบบ

---

## 27.1 Stack (Array-Based)

Stack คือโครงสร้างแบบ LIFO (Last In, First Out)

```lua
-- Stack implementation
local Stack = {}
Stack.__index = Stack

function Stack.new()
    return setmetatable({ _data = {}, _size = 0 }, Stack)
end

function Stack:push(value)
    self._size = self._size + 1
    self._data[self._size] = value
end

function Stack:pop()
    if self._size == 0 then
        error("Stack underflow: stack is empty")
    end
    local value = self._data[self._size]
    self._data[self._size] = nil
    self._size = self._size - 1
    return value
end

function Stack:peek()
    if self._size == 0 then return nil end
    return self._data[self._size]
end

function Stack:isEmpty()
    return self._size == 0
end

function Stack:size()
    return self._size
end

function Stack:__tostring()
    local parts = {}
    for i = 1, self._size do
        parts[i] = tostring(self._data[i])
    end
    return "Stack[" .. table.concat(parts, " | ") .. "] <-- top"
end

-- ทดสอบ Stack
local s = Stack.new()
s:push(10)
s:push(20)
s:push(30)

print(tostring(s))       -- Stack[10 | 20 | 30] <-- top
print(s:peek())          -- 30
print(s:pop())           -- 30
print(s:pop())           -- 20
print(s:size())          -- 1
print(s:isEmpty())       -- false
print(s:pop())           -- 10
print(s:isEmpty())       -- true
```

```lua
-- ตัวอย่างการใช้งาน Stack: ตรวจสอบ bracket matching
local function checkBrackets(str)
    local stack = Stack.new()
    local pairs_map = { [")"] = "(", ["]"] = "[", ["}"] = "{" }
    local opens = { ["("] = true, ["["] = true, ["{"] = true }

    for i = 1, #str do
        local ch = str:sub(i, i)
        if opens[ch] then
            stack:push(ch)
        elseif pairs_map[ch] then
            if stack:isEmpty() then
                return false, "ปิดโดยไม่มีเปิดที่ position " .. i
            end
            if stack:peek() ~= pairs_map[ch] then
                return false, "ไม่ match ที่ position " .. i
            end
            stack:pop()
        end
    end

    if not stack:isEmpty() then
        return false, "มีวงเล็บที่เปิดแต่ไม่ปิด"
    end
    return true, "ถูกต้อง"
end

print(checkBrackets("(1 + 2) * [3 + {4 - 5}]"))  -- true ถูกต้อง
print(checkBrackets("(1 + 2"))                      -- false
print(checkBrackets(")1 + 2("))                     -- false
print(checkBrackets("{[()]}"))                      -- true ถูกต้อง
```

```lua
-- Stack สำหรับ undo/redo
local function createEditor()
    local content = ""
    local undoStack = Stack.new()
    local redoStack = Stack.new()

    return {
        write = function(self, text)
            undoStack:push(content)
            -- clear redo stack เมื่อมีการเขียนใหม่
            while not redoStack:isEmpty() do
                redoStack:pop()
            end
            content = content .. text
        end,
        undo = function(self)
            if undoStack:isEmpty() then
                print("ไม่มีอะไรให้ undo")
                return
            end
            redoStack:push(content)
            content = undoStack:pop()
        end,
        redo = function(self)
            if redoStack:isEmpty() then
                print("ไม่มีอะไรให้ redo")
                return
            end
            undoStack:push(content)
            content = redoStack:pop()
        end,
        getContent = function(self) return content end
    }
end

local editor = createEditor()
editor:write("Hello")
editor:write(", World")
editor:write("!")
print(editor:getContent())  -- Hello, World!
editor:undo()
print(editor:getContent())  -- Hello, World
editor:undo()
print(editor:getContent())  -- Hello
editor:redo()
print(editor:getContent())  -- Hello, World
```

---

## 27.2 Queue Implementation

Queue คือโครงสร้างแบบ FIFO (First In, First Out)

```lua
-- Queue implementation (efficient)
local Queue = {}
Queue.__index = Queue

function Queue.new()
    return setmetatable({ _data = {}, _head = 1, _tail = 0 }, Queue)
end

function Queue:enqueue(value)
    self._tail = self._tail + 1
    self._data[self._tail] = value
end

function Queue:dequeue()
    if self._head > self._tail then
        error("Queue is empty")
    end
    local value = self._data[self._head]
    self._data[self._head] = nil
    self._head = self._head + 1
    return value
end

function Queue:front()
    if self._head > self._tail then return nil end
    return self._data[self._head]
end

function Queue:isEmpty()
    return self._head > self._tail
end

function Queue:size()
    return self._tail - self._head + 1
end

function Queue:__tostring()
    local parts = {}
    for i = self._head, self._tail do
        parts[#parts + 1] = tostring(self._data[i])
    end
    return "front --> [" .. table.concat(parts, ", ") .. "] <-- back"
end

-- ทดสอบ Queue
local q = Queue.new()
q:enqueue("Task A")
q:enqueue("Task B")
q:enqueue("Task C")

print(tostring(q))          -- front --> [Task A, Task B, Task C] <-- back
print(q:front())            -- Task A
print(q:dequeue())          -- Task A
print(q:dequeue())          -- Task B
print(q:size())             -- 1
print(q:isEmpty())          -- false
```

```lua
-- ตัวอย่าง: BFS ใช้ Queue (preview)
local function bfsLevel(graph, start)
    local queue = Queue.new()
    local visited = {}
    local levels = {}

    queue:enqueue({node = start, level = 0})
    visited[start] = true

    while not queue:isEmpty() do
        local item = queue:dequeue()
        local node, level = item.node, item.level

        if not levels[level] then levels[level] = {} end
        levels[level][#levels[level] + 1] = node

        if graph[node] then
            for _, neighbor in ipairs(graph[node]) do
                if not visited[neighbor] then
                    visited[neighbor] = true
                    queue:enqueue({node = neighbor, level = level + 1})
                end
            end
        end
    end
    return levels
end

local graph = {
    A = {"B", "C"},
    B = {"D", "E"},
    C = {"F"},
    D = {},
    E = {"F"},
    F = {},
}

local levels = bfsLevel(graph, "A")
for level, nodes in ipairs(levels) do
    print("Level " .. (level-1) .. ": " .. table.concat(nodes, ", "))
end
```

---

## 27.3 Deque (Double-Ended Queue)

```lua
-- Deque: queue ที่เพิ่ม/ลบได้ทั้งสองด้าน
local Deque = {}
Deque.__index = Deque

function Deque.new()
    return setmetatable({ _data = {}, _head = 1, _tail = 0 }, Deque)
end

function Deque:pushFront(value)
    self._head = self._head - 1
    self._data[self._head] = value
end

function Deque:pushBack(value)
    self._tail = self._tail + 1
    self._data[self._tail] = value
end

function Deque:popFront()
    if self:isEmpty() then error("Deque is empty") end
    local val = self._data[self._head]
    self._data[self._head] = nil
    self._head = self._head + 1
    return val
end

function Deque:popBack()
    if self:isEmpty() then error("Deque is empty") end
    local val = self._data[self._tail]
    self._data[self._tail] = nil
    self._tail = self._tail - 1
    return val
end

function Deque:peekFront()
    return self._data[self._head]
end

function Deque:peekBack()
    return self._data[self._tail]
end

function Deque:isEmpty()
    return self._head > self._tail
end

function Deque:size()
    return self._tail - self._head + 1
end

-- ทดสอบ Deque
local dq = Deque.new()
dq:pushBack(1)
dq:pushBack(2)
dq:pushBack(3)
dq:pushFront(0)
dq:pushFront(-1)

print("Size:", dq:size())         -- 5
print("Front:", dq:peekFront())   -- -1
print("Back:", dq:peekBack())     -- 3
print(dq:popFront())              -- -1
print(dq:popBack())               -- 3
print("Size:", dq:size())         -- 3
```

---

## 27.4 Linked List

### Singly Linked List

```lua
-- Node สำหรับ Linked List
local function newNode(value)
    return { value = value, next = nil }
end

-- Singly Linked List
local LinkedList = {}
LinkedList.__index = LinkedList

function LinkedList.new()
    return setmetatable({ head = nil, _size = 0 }, LinkedList)
end

function LinkedList:pushFront(value)
    local node = newNode(value)
    node.next = self.head
    self.head = node
    self._size = self._size + 1
end

function LinkedList:pushBack(value)
    local node = newNode(value)
    if self.head == nil then
        self.head = node
    else
        local current = self.head
        while current.next do
            current = current.next
        end
        current.next = node
    end
    self._size = self._size + 1
end

function LinkedList:popFront()
    if self.head == nil then error("List is empty") end
    local val = self.head.value
    self.head = self.head.next
    self._size = self._size - 1
    return val
end

function LinkedList:contains(value)
    local current = self.head
    while current do
        if current.value == value then return true end
        current = current.next
    end
    return false
end

function LinkedList:remove(value)
    if self.head == nil then return false end
    if self.head.value == value then
        self.head = self.head.next
        self._size = self._size - 1
        return true
    end
    local current = self.head
    while current.next do
        if current.next.value == value then
            current.next = current.next.next
            self._size = self._size - 1
            return true
        end
        current = current.next
    end
    return false
end

function LinkedList:toArray()
    local result = {}
    local current = self.head
    while current do
        result[#result + 1] = current.value
        current = current.next
    end
    return result
end

function LinkedList:size()
    return self._size
end

function LinkedList:reverse()
    local prev = nil
    local current = self.head
    while current do
        local next = current.next
        current.next = prev
        prev = current
        current = next
    end
    self.head = prev
end

-- ทดสอบ
local list = LinkedList.new()
list:pushBack(1)
list:pushBack(2)
list:pushBack(3)
list:pushFront(0)

print(table.concat(list:toArray(), " -> "))  -- 0 -> 1 -> 2 -> 3
print("Contains 2:", list:contains(2))       -- true
list:remove(2)
print(table.concat(list:toArray(), " -> "))  -- 0 -> 1 -> 3
list:reverse()
print(table.concat(list:toArray(), " -> "))  -- 3 -> 1 -> 0
print("Size:", list:size())                  -- 3
```

### Doubly Linked List

```lua
-- Doubly Linked List
local DLL = {}
DLL.__index = DLL

local function newDLLNode(value)
    return { value = value, next = nil, prev = nil }
end

function DLL.new()
    return setmetatable({ head = nil, tail = nil, _size = 0 }, DLL)
end

function DLL:pushBack(value)
    local node = newDLLNode(value)
    if self.tail == nil then
        self.head = node
        self.tail = node
    else
        node.prev = self.tail
        self.tail.next = node
        self.tail = node
    end
    self._size = self._size + 1
end

function DLL:pushFront(value)
    local node = newDLLNode(value)
    if self.head == nil then
        self.head = node
        self.tail = node
    else
        node.next = self.head
        self.head.prev = node
        self.head = node
    end
    self._size = self._size + 1
end

function DLL:popBack()
    if self.tail == nil then error("DLL is empty") end
    local val = self.tail.value
    if self.head == self.tail then
        self.head = nil
        self.tail = nil
    else
        self.tail = self.tail.prev
        self.tail.next = nil
    end
    self._size = self._size - 1
    return val
end

function DLL:popFront()
    if self.head == nil then error("DLL is empty") end
    local val = self.head.value
    if self.head == self.tail then
        self.head = nil
        self.tail = nil
    else
        self.head = self.head.next
        self.head.prev = nil
    end
    self._size = self._size - 1
    return val
end

function DLL:toArray(reverse)
    local result = {}
    if reverse then
        local current = self.tail
        while current do
            result[#result + 1] = current.value
            current = current.prev
        end
    else
        local current = self.head
        while current do
            result[#result + 1] = current.value
            current = current.next
        end
    end
    return result
end

-- ทดสอบ
local dll = DLL.new()
dll:pushBack(1)
dll:pushBack(2)
dll:pushBack(3)
dll:pushFront(0)

print("Forward:", table.concat(dll:toArray(), " <-> "))      -- 0 <-> 1 <-> 2 <-> 3
print("Backward:", table.concat(dll:toArray(true), " <-> ")) -- 3 <-> 2 <-> 1 <-> 0
print(dll:popBack())   -- 3
print(dll:popFront())  -- 0
print("Forward:", table.concat(dll:toArray(), " <-> "))      -- 1 <-> 2
```

---

## 27.5 Binary Search Tree (BST)

```lua
-- Binary Search Tree
local BST = {}
BST.__index = BST

local function newBSTNode(value)
    return { value = value, left = nil, right = nil }
end

function BST.new()
    return setmetatable({ root = nil, _size = 0 }, BST)
end

function BST:insert(value)
    local function insertNode(node, v)
        if node == nil then
            self._size = self._size + 1
            return newBSTNode(v)
        end
        if v < node.value then
            node.left = insertNode(node.left, v)
        elseif v > node.value then
            node.right = insertNode(node.right, v)
        end
        -- duplicate: ไม่ insert
        return node
    end
    self.root = insertNode(self.root, value)
end

function BST:contains(value)
    local function search(node, v)
        if node == nil then return false end
        if v == node.value then return true end
        if v < node.value then return search(node.left, v) end
        return search(node.right, v)
    end
    return search(self.root, value)
end

function BST:inorder()
    local result = {}
    local function traverse(node)
        if node == nil then return end
        traverse(node.left)
        result[#result + 1] = node.value
        traverse(node.right)
    end
    traverse(self.root)
    return result
end

function BST:preorder()
    local result = {}
    local function traverse(node)
        if node == nil then return end
        result[#result + 1] = node.value
        traverse(node.left)
        traverse(node.right)
    end
    traverse(self.root)
    return result
end

function BST:postorder()
    local result = {}
    local function traverse(node)
        if node == nil then return end
        traverse(node.left)
        traverse(node.right)
        result[#result + 1] = node.value
    end
    traverse(self.root)
    return result
end

function BST:min()
    if self.root == nil then return nil end
    local current = self.root
    while current.left do current = current.left end
    return current.value
end

function BST:max()
    if self.root == nil then return nil end
    local current = self.root
    while current.right do current = current.right end
    return current.value
end

function BST:height()
    local function h(node)
        if node == nil then return 0 end
        return 1 + math.max(h(node.left), h(node.right))
    end
    return h(self.root)
end

function BST:delete(value)
    local function findMin(node)
        while node.left do node = node.left end
        return node
    end

    local function deleteNode(node, v)
        if node == nil then return nil end
        if v < node.value then
            node.left = deleteNode(node.left, v)
        elseif v > node.value then
            node.right = deleteNode(node.right, v)
        else
            -- พบ node ที่ต้องลบ
            if node.left == nil then
                self._size = self._size - 1
                return node.right
            elseif node.right == nil then
                self._size = self._size - 1
                return node.left
            end
            -- มีทั้งสอง children
            local minRight = findMin(node.right)
            node.value = minRight.value
            node.right = deleteNode(node.right, minRight.value)
            self._size = self._size + 1  -- compensate for the -1 in recursive call
        end
        return node
    end
    self.root = deleteNode(self.root, value)
end

-- ทดสอบ BST
local bst = BST.new()
local values = {5, 3, 7, 1, 4, 6, 8, 2}
for _, v in ipairs(values) do bst:insert(v) end

print("Inorder:", table.concat(bst:inorder(), ", "))    -- 1, 2, 3, 4, 5, 6, 7, 8
print("Preorder:", table.concat(bst:preorder(), ", "))  -- 5, 3, 1, 2, 4, 7, 6, 8
print("Min:", bst:min())      -- 1
print("Max:", bst:max())      -- 8
print("Height:", bst:height()) -- 4
print("Contains 4:", bst:contains(4))  -- true
print("Contains 9:", bst:contains(9))  -- false
bst:delete(3)
print("After delete 3:", table.concat(bst:inorder(), ", "))  -- 1, 2, 4, 5, 6, 7, 8
```

---

## 27.6 Heap / Priority Queue

```lua
-- Min-Heap / Priority Queue
local Heap = {}
Heap.__index = Heap

function Heap.new(comparator)
    -- comparator(a, b) returns true if a should be higher priority than b
    -- default: min-heap (smaller value = higher priority)
    return setmetatable({
        _data = {},
        _size = 0,
        _cmp = comparator or function(a, b) return a < b end
    }, Heap)
end

function Heap:_swap(i, j)
    self._data[i], self._data[j] = self._data[j], self._data[i]
end

function Heap:_parent(i) return math.floor(i / 2) end
function Heap:_left(i)   return i * 2 end
function Heap:_right(i)  return i * 2 + 1 end

function Heap:_siftUp(i)
    while i > 1 do
        local p = self:_parent(i)
        if self._cmp(self._data[i], self._data[p]) then
            self:_swap(i, p)
            i = p
        else
            break
        end
    end
end

function Heap:_siftDown(i)
    while true do
        local smallest = i
        local l = self:_left(i)
        local r = self:_right(i)

        if l <= self._size and self._cmp(self._data[l], self._data[smallest]) then
            smallest = l
        end
        if r <= self._size and self._cmp(self._data[r], self._data[smallest]) then
            smallest = r
        end

        if smallest ~= i then
            self:_swap(i, smallest)
            i = smallest
        else
            break
        end
    end
end

function Heap:push(value)
    self._size = self._size + 1
    self._data[self._size] = value
    self:_siftUp(self._size)
end

function Heap:pop()
    if self._size == 0 then error("Heap is empty") end
    local top = self._data[1]
    self._data[1] = self._data[self._size]
    self._data[self._size] = nil
    self._size = self._size - 1
    if self._size > 0 then self:_siftDown(1) end
    return top
end

function Heap:peek()
    return self._data[1]
end

function Heap:size() return self._size end
function Heap:isEmpty() return self._size == 0 end

-- ทดสอบ Min-Heap
local minHeap = Heap.new()
local nums = {5, 3, 8, 1, 9, 2, 7, 4, 6}
for _, v in ipairs(nums) do minHeap:push(v) end

print("Min-Heap sort:")
local sorted = {}
while not minHeap:isEmpty() do
    sorted[#sorted + 1] = minHeap:pop()
end
print(table.concat(sorted, ", "))  -- 1, 2, 3, 4, 5, 6, 7, 8, 9

-- Max-Heap
local maxHeap = Heap.new(function(a, b) return a > b end)
for _, v in ipairs(nums) do maxHeap:push(v) end

print("Max-Heap sort (descending):")
sorted = {}
while not maxHeap:isEmpty() do
    sorted[#sorted + 1] = maxHeap:pop()
end
print(table.concat(sorted, ", "))  -- 9, 8, 7, 6, 5, 4, 3, 2, 1

-- Priority Queue กับ struct
local pq = Heap.new(function(a, b) return a.priority < b.priority end)
pq:push({ task = "ทำงานที่ 1", priority = 3 })
pq:push({ task = "งานเร่งด่วน!", priority = 1 })
pq:push({ task = "ทำงานที่ 3", priority = 5 })
pq:push({ task = "งานสำคัญ", priority = 2 })

print("\nTask processing order:")
while not pq:isEmpty() do
    local item = pq:pop()
    print(string.format("  [P%d] %s", item.priority, item.task))
end
```

---

## 27.7 Hash Map (Beyond Tables)

```lua
-- Hash Map ด้วย separate chaining
local HashMap = {}
HashMap.__index = HashMap

function HashMap.new(capacity)
    capacity = capacity or 16
    local map = setmetatable({
        _buckets = {},
        _capacity = capacity,
        _size = 0,
        _loadFactor = 0.75,
    }, HashMap)

    for i = 1, capacity do
        map._buckets[i] = {}
    end
    return map
end

function HashMap:_hash(key)
    local h = 0
    local str = tostring(key)
    for i = 1, #str do
        h = (h * 31 + str:byte(i)) % self._capacity
    end
    return h + 1
end

function HashMap:set(key, value)
    local idx = self:_hash(key)
    local bucket = self._buckets[idx]

    -- ค้นหาว่ามี key อยู่แล้วหรือไม่
    for i, pair in ipairs(bucket) do
        if pair[1] == key then
            pair[2] = value
            return
        end
    end

    -- ไม่มี: เพิ่มใหม่
    bucket[#bucket + 1] = {key, value}
    self._size = self._size + 1

    -- resize ถ้า load factor สูงเกินไป
    if self._size > self._capacity * self._loadFactor then
        self:_resize()
    end
end

function HashMap:get(key)
    local idx = self:_hash(key)
    local bucket = self._buckets[idx]
    for _, pair in ipairs(bucket) do
        if pair[1] == key then return pair[2] end
    end
    return nil
end

function HashMap:delete(key)
    local idx = self:_hash(key)
    local bucket = self._buckets[idx]
    for i, pair in ipairs(bucket) do
        if pair[1] == key then
            table.remove(bucket, i)
            self._size = self._size - 1
            return true
        end
    end
    return false
end

function HashMap:contains(key)
    return self:get(key) ~= nil
end

function HashMap:_resize()
    local old_buckets = self._buckets
    self._capacity = self._capacity * 2
    self._buckets = {}
    self._size = 0
    for i = 1, self._capacity do
        self._buckets[i] = {}
    end
    for _, bucket in ipairs(old_buckets) do
        for _, pair in ipairs(bucket) do
            self:set(pair[1], pair[2])
        end
    end
end

function HashMap:size() return self._size end

function HashMap:keys()
    local keys = {}
    for _, bucket in ipairs(self._buckets) do
        for _, pair in ipairs(bucket) do
            keys[#keys + 1] = pair[1]
        end
    end
    return keys
end

-- ทดสอบ HashMap
local hm = HashMap.new(4)
hm:set("name", "สมชาย")
hm:set("age", 30)
hm:set("city", "กรุงเทพ")
hm:set("email", "somchai@example.com")
hm:set("phone", "081-234-5678")

print("name:", hm:get("name"))   -- สมชาย
print("age:", hm:get("age"))     -- 30
print("size:", hm:size())        -- 5
print("has email:", hm:contains("email"))  -- true

hm:delete("age")
print("After delete age:", hm:get("age"))  -- nil
print("size:", hm:size())  -- 4
```

---

## 27.8 Trie

Trie คือ tree structure สำหรับเก็บ string โดยเฉพาะ

```lua
-- Trie implementation
local Trie = {}
Trie.__index = Trie

function Trie.new()
    return setmetatable({
        _root = { children = {}, isEnd = false }
    }, Trie)
end

function Trie:insert(word)
    local node = self._root
    for i = 1, #word do
        local ch = word:sub(i, i)
        if not node.children[ch] then
            node.children[ch] = { children = {}, isEnd = false }
        end
        node = node.children[ch]
    end
    node.isEnd = true
end

function Trie:search(word)
    local node = self._root
    for i = 1, #word do
        local ch = word:sub(i, i)
        if not node.children[ch] then return false end
        node = node.children[ch]
    end
    return node.isEnd
end

function Trie:startsWith(prefix)
    local node = self._root
    for i = 1, #prefix do
        local ch = prefix:sub(i, i)
        if not node.children[ch] then return false end
        node = node.children[ch]
    end
    return true
end

function Trie:autocomplete(prefix)
    local node = self._root
    for i = 1, #prefix do
        local ch = prefix:sub(i, i)
        if not node.children[ch] then return {} end
        node = node.children[ch]
    end

    -- DFS หาคำทั้งหมดที่ขึ้นต้นด้วย prefix
    local results = {}
    local function dfs(n, current)
        if n.isEnd then
            results[#results + 1] = prefix .. current
        end
        -- sort keys for deterministic order
        local keys = {}
        for k in pairs(n.children) do keys[#keys+1] = k end
        table.sort(keys)
        for _, ch in ipairs(keys) do
            dfs(n.children[ch], current .. ch)
        end
    end
    dfs(node, "")
    return results
end

function Trie:delete(word)
    local function del(node, w, i)
        if i > #w then
            if not node.isEnd then return false end
            node.isEnd = false
            return next(node.children) == nil  -- ลบ node ถ้าไม่มีลูก
        end
        local ch = w:sub(i, i)
        if not node.children[ch] then return false end
        local shouldDelete = del(node.children[ch], w, i + 1)
        if shouldDelete then
            node.children[ch] = nil
            return not node.isEnd and next(node.children) == nil
        end
        return false
    end
    del(self._root, word, 1)
end

-- ทดสอบ Trie
local trie = Trie.new()
local words = {"apple", "app", "application", "apply", "apt",
               "banana", "band", "bandana", "bat"}
for _, w in ipairs(words) do trie:insert(w) end

print("search 'app':", trie:search("app"))           -- true
print("search 'ap':", trie:search("ap"))             -- false
print("startsWith 'app':", trie:startsWith("app"))   -- true
print("startsWith 'ban':", trie:startsWith("ban"))   -- true
print("startsWith 'xyz':", trie:startsWith("xyz"))   -- false

print("\nAutocomplete 'app':")
for _, w in ipairs(trie:autocomplete("app")) do
    print("  " .. w)
end
-- app, apple, application, apply

print("\nAutocomplete 'ban':")
for _, w in ipairs(trie:autocomplete("ban")) do
    print("  " .. w)
end
-- band, bandana, banana
```

---

## 27.9 Graph

```lua
-- Graph (adjacency list)
local Graph = {}
Graph.__index = Graph

function Graph.new(directed)
    return setmetatable({
        _adj = {},
        _directed = directed or false,
        _vertices = {},
    }, Graph)
end

function Graph:addVertex(v)
    if not self._adj[v] then
        self._adj[v] = {}
        self._vertices[#self._vertices + 1] = v
    end
end

function Graph:addEdge(u, v, weight)
    weight = weight or 1
    self:addVertex(u)
    self:addVertex(v)

    self._adj[u][#self._adj[u] + 1] = { to = v, weight = weight }
    if not self._directed then
        self._adj[v][#self._adj[v] + 1] = { to = u, weight = weight }
    end
end

function Graph:neighbors(v)
    return self._adj[v] or {}
end

function Graph:vertices()
    return self._vertices
end

function Graph:bfs(start)
    local visited = {}
    local order = {}
    local queue = {start}
    visited[start] = true

    while #queue > 0 do
        local v = table.remove(queue, 1)
        order[#order + 1] = v
        for _, edge in ipairs(self:neighbors(v)) do
            if not visited[edge.to] then
                visited[edge.to] = true
                queue[#queue + 1] = edge.to
            end
        end
    end
    return order
end

function Graph:dfs(start)
    local visited = {}
    local order = {}

    local function dfsHelper(v)
        visited[v] = true
        order[#order + 1] = v
        for _, edge in ipairs(self:neighbors(v)) do
            if not visited[edge.to] then
                dfsHelper(edge.to)
            end
        end
    end

    dfsHelper(start)
    return order
end

function Graph:hasPath(start, finish)
    local visited = {}
    local function dfs(v)
        if v == finish then return true end
        visited[v] = true
        for _, edge in ipairs(self:neighbors(v)) do
            if not visited[edge.to] and dfs(edge.to) then
                return true
            end
        end
        return false
    end
    return dfs(start)
end

-- ทดสอบ Graph
local g = Graph.new(false)  -- undirected
g:addEdge("A", "B")
g:addEdge("A", "C")
g:addEdge("B", "D")
g:addEdge("C", "D")
g:addEdge("D", "E")
g:addEdge("B", "E")

print("BFS from A:", table.concat(g:bfs("A"), " -> "))
print("DFS from A:", table.concat(g:dfs("A"), " -> "))
print("Path A->E:", g:hasPath("A", "E"))  -- true
print("Path E->C:", g:hasPath("E", "C"))  -- true
```

---

## 27.10 Circular Buffer

```lua
-- Circular Buffer (Ring Buffer)
local CircularBuffer = {}
CircularBuffer.__index = CircularBuffer

function CircularBuffer.new(capacity)
    return setmetatable({
        _data = {},
        _capacity = capacity,
        _head = 1,
        _tail = 1,
        _size = 0,
    }, CircularBuffer)
end

function CircularBuffer:write(value)
    if self._size == self._capacity then
        -- Overwrite oldest
        self._head = (self._head % self._capacity) + 1
        self._size = self._size - 1
    end
    self._data[self._tail] = value
    self._tail = (self._tail % self._capacity) + 1
    self._size = self._size + 1
end

function CircularBuffer:read()
    if self._size == 0 then return nil end
    local val = self._data[self._head]
    self._head = (self._head % self._capacity) + 1
    self._size = self._size - 1
    return val
end

function CircularBuffer:peek()
    if self._size == 0 then return nil end
    return self._data[self._head]
end

function CircularBuffer:isFull()
    return self._size == self._capacity
end

function CircularBuffer:isEmpty()
    return self._size == 0
end

function CircularBuffer:size() return self._size end

function CircularBuffer:toArray()
    local result = {}
    local idx = self._head
    for i = 1, self._size do
        result[i] = self._data[idx]
        idx = (idx % self._capacity) + 1
    end
    return result
end

-- ทดสอบ Circular Buffer
local cb = CircularBuffer.new(5)
for i = 1, 7 do
    cb:write(i)
    print(string.format("Write %d, buffer: [%s], size: %d",
        i, table.concat(cb:toArray(), ","), cb:size()))
end
-- ตอน write 6 และ 7 จะ overwrite 1 และ 2

print("Read:", cb:read())  -- 3
print("Current:", table.concat(cb:toArray(), ","))  -- 4,5,6,7
```

---

## 27.11 LRU Cache

```lua
-- LRU Cache ด้วย HashMap + Doubly Linked List
local LRUCache = {}
LRUCache.__index = LRUCache

function LRUCache.new(capacity)
    local cache = setmetatable({
        _capacity = capacity,
        _size = 0,
        _map = {},       -- key -> node
        _head = nil,     -- most recently used
        _tail = nil,     -- least recently used
    }, LRUCache)
    return cache
end

local function newLRUNode(key, value)
    return { key = key, value = value, prev = nil, next = nil }
end

function LRUCache:_addToFront(node)
    node.next = self._head
    node.prev = nil
    if self._head then self._head.prev = node end
    self._head = node
    if self._tail == nil then self._tail = node end
end

function LRUCache:_removeNode(node)
    if node.prev then node.prev.next = node.next end
    if node.next then node.next.prev = node.prev end
    if self._head == node then self._head = node.next end
    if self._tail == node then self._tail = node.prev end
    node.prev = nil
    node.next = nil
end

function LRUCache:get(key)
    local node = self._map[key]
    if node == nil then return nil end
    -- ย้ายไปต้น (most recently used)
    self:_removeNode(node)
    self:_addToFront(node)
    return node.value
end

function LRUCache:put(key, value)
    local existing = self._map[key]
    if existing then
        existing.value = value
        self:_removeNode(existing)
        self:_addToFront(existing)
        return
    end

    local node = newLRUNode(key, value)
    self._map[key] = node
    self:_addToFront(node)
    self._size = self._size + 1

    if self._size > self._capacity then
        -- ลบ LRU (tail)
        local lru = self._tail
        self._map[lru.key] = nil
        self:_removeNode(lru)
        self._size = self._size - 1
    end
end

function LRUCache:size() return self._size end

-- ทดสอบ LRU Cache
local cache = LRUCache.new(3)
cache:put("a", 1)
cache:put("b", 2)
cache:put("c", 3)

print("get a:", cache:get("a"))  -- 1 (a ถูก access, เป็น MRU)
cache:put("d", 4)               -- b ถูกลบ (LRU)
print("get b:", cache:get("b"))  -- nil (ถูกลบแล้ว)
print("get c:", cache:get("c"))  -- 3
print("get d:", cache:get("d"))  -- 4
print("size:", cache:size())     -- 3
```

---

## 27.12 Bloom Filter

```lua
-- Bloom Filter: probabilistic data structure สำหรับ membership test
local BloomFilter = {}
BloomFilter.__index = BloomFilter

function BloomFilter.new(size, numHashes)
    return setmetatable({
        _bits = {},
        _size = size or 1000,
        _numHashes = numHashes or 3,
        _count = 0,
    }, BloomFilter)
end

-- hash functions (ง่ายๆ สำหรับ demo)
function BloomFilter:_hash(item, seed)
    local h = seed
    local str = tostring(item)
    for i = 1, #str do
        h = (h * 31 + str:byte(i)) % self._size
    end
    return (h % self._size) + 1
end

function BloomFilter:add(item)
    for i = 1, self._numHashes do
        local idx = self:_hash(item, i * 2654435761)
        self._bits[idx] = true
    end
    self._count = self._count + 1
end

function BloomFilter:mightContain(item)
    for i = 1, self._numHashes do
        local idx = self:_hash(item, i * 2654435761)
        if not self._bits[idx] then return false end
    end
    return true  -- might contain (could be false positive)
end

function BloomFilter:count() return self._count end

-- ทดสอบ Bloom Filter
local bf = BloomFilter.new(1000, 3)

-- เพิ่ม items
local added = {"apple", "banana", "cherry", "date", "elderberry"}
for _, item in ipairs(added) do
    bf:add(item)
end

-- ทดสอบ items ที่เพิ่มแล้ว
for _, item in ipairs(added) do
    print(item .. ": " .. tostring(bf:mightContain(item)))  -- true ทั้งหมด
end

-- ทดสอบ items ที่ไม่ได้เพิ่ม
local notAdded = {"fig", "grape", "honeydew", "kiwi"}
print("\nNot added items:")
local falsePositives = 0
for _, item in ipairs(notAdded) do
    local result = bf:mightContain(item)
    if result then falsePositives = falsePositives + 1 end
    print(item .. ": " .. tostring(result))
end
print("False positives: " .. falsePositives .. "/" .. #notAdded)
```

---

## 27.13 ตัวอย่างรวม: การเลือกใช้ Data Structure

```lua
-- เปรียบเทียบ performance
print("=== Data Structure Performance Comparison ===\n")

local function timeIt(name, fn)
    local t1 = os.clock()
    fn()
    local t2 = os.clock()
    print(string.format("%-30s %.6f s", name, t2 - t1))
end

-- Stack operations
timeIt("Stack (1000 push+pop)", function()
    local s = Stack.new()
    for i = 1, 1000 do s:push(i) end
    while not s:isEmpty() do s:pop() end
end)

-- Queue operations
timeIt("Queue (1000 enqueue+dequeue)", function()
    local q = Queue.new()
    for i = 1, 1000 do q:enqueue(i) end
    while not q:isEmpty() do q:dequeue() end
end)

-- BST operations
timeIt("BST (1000 insert+search)", function()
    local bst2 = BST.new()
    for i = 1, 500 do bst2:insert(math.random(10000)) end
    for i = 1, 500 do bst2:contains(math.random(10000)) end
end)

-- Heap operations
timeIt("Heap (1000 push+pop)", function()
    local h = Heap.new()
    for i = 1, 1000 do h:push(math.random(10000)) end
    while not h:isEmpty() do h:pop() end
end)

print("\n=== Summary ===")
print("Stack: LIFO - ใช้สำหรับ backtracking, undo/redo")
print("Queue: FIFO - ใช้สำหรับ task scheduling, BFS")
print("Deque: Two-ended - ใช้สำหรับ sliding window")
print("LinkedList: Dynamic - ใช้เมื่อต้องการ insert/delete ถี่")
print("BST: Ordered - ใช้สำหรับ sorted data, range queries")
print("Heap: Priority - ใช้สำหรับ Dijkstra, priority tasks")
print("HashMap: O(1) avg - ใช้สำหรับ fast lookup")
print("Trie: String prefix - ใช้สำหรับ autocomplete")
print("Graph: Relations - ใช้สำหรับ network/path problems")
print("CircularBuffer: Fixed size - ใช้สำหรับ streaming data")
print("LRU Cache: Memory limit - ใช้สำหรับ caching")
print("BloomFilter: Probabilistic - ใช้สำหรับ fast rejection")
```

---

## บทสรุป

Data Structures ที่เหมาะสมช่วยให้:
- การเข้าถึงข้อมูลรวดเร็วขึ้น
- ใช้หน่วยความจำอย่างมีประสิทธิภาพ
- อัลกอริทึมทำงานได้เร็วขึ้น

การเลือก Data Structure ต้องพิจารณา:
1. **Time Complexity** ของ operations ที่ใช้บ่อย
2. **Space Complexity** ข้อจำกัดหน่วยความจำ
3. **Access Pattern** อ่านอย่างเดียว หรือ อ่าน/เขียนบ่อย
4. **Data ordering** ต้องการ sorted หรือไม่
