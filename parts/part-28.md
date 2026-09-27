# บทที่ 28: Algorithms ใน Lua

## บทนำ

Algorithm (อัลกอริทึม) คือขั้นตอนการแก้ปัญหาอย่างมีระบบ บทนี้ครอบคลุมอัลกอริทึมที่สำคัญในการเขียนโปรแกรม พร้อม Time Complexity และตัวอย่างการใช้งานใน Lua 5.4

---

## 28.1 Sorting Algorithms

### Bubble Sort

Time Complexity: O(n²) average/worst, O(n) best (sorted)
Space Complexity: O(1)

```lua
-- Bubble Sort
local function bubbleSort(arr)
    local n = #arr
    local swapped
    for i = 1, n - 1 do
        swapped = false
        for j = 1, n - i do
            if arr[j] > arr[j + 1] then
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
                swapped = true
            end
        end
        -- ถ้าไม่มีการ swap แปลว่า sorted แล้ว
        if not swapped then break end
    end
    return arr
end

-- ทดสอบ
local data = {64, 34, 25, 12, 22, 11, 90}
print("Before:", table.concat(data, ", "))
bubbleSort(data)
print("After:", table.concat(data, ", "))
-- After: 11, 12, 22, 25, 34, 64, 90

-- Bubble Sort แบบ generic comparator
local function bubbleSortGeneric(arr, cmp)
    cmp = cmp or function(a, b) return a > b end
    local n = #arr
    for i = 1, n - 1 do
        for j = 1, n - i do
            if cmp(arr[j], arr[j + 1]) then
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
            end
        end
    end
    return arr
end

-- Sort descending
local data2 = {5, 2, 8, 1, 9, 3}
bubbleSortGeneric(data2)
print("Descending:", table.concat(data2, ", "))  -- 9, 8, 5, 3, 2, 1

-- Sort by string length
local words = {"banana", "apple", "cherry", "fig", "date"}
bubbleSortGeneric(words, function(a, b) return #a > #b end)
print("By length:", table.concat(words, ", "))  -- fig, date, apple, banana, cherry
```

### Selection Sort

Time Complexity: O(n²) all cases
Space Complexity: O(1)

```lua
-- Selection Sort
local function selectionSort(arr)
    local n = #arr
    for i = 1, n - 1 do
        local minIdx = i
        for j = i + 1, n do
            if arr[j] < arr[minIdx] then
                minIdx = j
            end
        end
        if minIdx ~= i then
            arr[i], arr[minIdx] = arr[minIdx], arr[i]
        end
    end
    return arr
end

local data = {29, 10, 14, 37, 13}
print("Before:", table.concat(data, ", "))
selectionSort(data)
print("After:", table.concat(data, ", "))  -- 10, 13, 14, 29, 37

-- Selection Sort ด้วย struct
local students = {
    { name = "สมชาย", gpa = 3.2 },
    { name = "สมหญิง", gpa = 3.8 },
    { name = "สมพงษ์", gpa = 2.9 },
    { name = "สมศักดิ์", gpa = 3.5 },
}

local function selectionSortBy(arr, key)
    local n = #arr
    for i = 1, n - 1 do
        local maxIdx = i
        for j = i + 1, n do
            if arr[j][key] > arr[maxIdx][key] then
                maxIdx = j
            end
        end
        arr[i], arr[maxIdx] = arr[maxIdx], arr[i]
    end
    return arr
end

selectionSortBy(students, "gpa")
print("\nStudents sorted by GPA (desc):")
for _, s in ipairs(students) do
    print(string.format("  %s: %.1f", s.name, s.gpa))
end
```

### Insertion Sort

Time Complexity: O(n²) average/worst, O(n) best (nearly sorted)
Space Complexity: O(1)

```lua
-- Insertion Sort
local function insertionSort(arr)
    for i = 2, #arr do
        local key = arr[i]
        local j = i - 1
        while j > 0 and arr[j] > key do
            arr[j + 1] = arr[j]
            j = j - 1
        end
        arr[j + 1] = key
    end
    return arr
end

local data = {12, 11, 13, 5, 6}
insertionSort(data)
print("Insertion sort:", table.concat(data, ", "))  -- 5, 6, 11, 12, 13

-- Insertion Sort เหมาะกับ nearly-sorted data
local nearlySorted = {1, 2, 3, 5, 4, 6, 7, 8, 9, 10}
insertionSort(nearlySorted)
print("Nearly sorted:", table.concat(nearlySorted, ", "))

-- Binary Insertion Sort (ปรับปรุงการค้นหาตำแหน่ง)
local function binaryInsertionSort(arr)
    local function binarySearch(arr, target, low, high)
        while low <= high do
            local mid = math.floor((low + high) / 2)
            if arr[mid] == target then return mid + 1
            elseif arr[mid] < target then low = mid + 1
            else high = mid - 1
            end
        end
        return low
    end

    for i = 2, #arr do
        local key = arr[i]
        local pos = binarySearch(arr, key, 1, i - 1)
        -- shift elements
        for j = i, pos + 1, -1 do
            arr[j] = arr[j - 1]
        end
        arr[pos] = key
    end
    return arr
end

local data2 = {37, 23, 0, 17, 12, 72, 31, 46, 100, 88, 54}
binaryInsertionSort(data2)
print("Binary insertion:", table.concat(data2, ", "))
```

### Merge Sort

Time Complexity: O(n log n) all cases
Space Complexity: O(n)

```lua
-- Merge Sort
local function mergeSort(arr)
    local function merge(left, right)
        local result = {}
        local i, j = 1, 1
        while i <= #left and j <= #right do
            if left[i] <= right[j] then
                result[#result + 1] = left[i]
                i = i + 1
            else
                result[#result + 1] = right[j]
                j = j + 1
            end
        end
        while i <= #left do
            result[#result + 1] = left[i]
            i = i + 1
        end
        while j <= #right do
            result[#result + 1] = right[j]
            j = j + 1
        end
        return result
    end

    local function sort(a)
        if #a <= 1 then return a end
        local mid = math.floor(#a / 2)
        local left = {}
        local right = {}
        for i = 1, mid do left[i] = a[i] end
        for i = mid + 1, #a do right[i - mid] = a[i] end
        return merge(sort(left), sort(right))
    end

    return sort(arr)
end

local data = {38, 27, 43, 3, 9, 82, 10}
local sorted = mergeSort(data)
print("Merge sort:", table.concat(sorted, ", "))  -- 3, 9, 10, 27, 38, 43, 82

-- นับ inversions ด้วย merge sort
local function countInversions(arr)
    local count = 0
    local function mergeCount(left, right)
        local result = {}
        local i, j = 1, 1
        while i <= #left and j <= #right do
            if left[i] <= right[j] then
                result[#result + 1] = left[i]
                i = i + 1
            else
                result[#result + 1] = right[j]
                count = count + (#left - i + 1)  -- inversions
                j = j + 1
            end
        end
        while i <= #left do result[#result+1] = left[i]; i = i + 1 end
        while j <= #right do result[#result+1] = right[j]; j = j + 1 end
        return result
    end

    local function sortCount(a)
        if #a <= 1 then return a end
        local mid = math.floor(#a / 2)
        local left, right = {}, {}
        for i = 1, mid do left[i] = a[i] end
        for i = mid + 1, #a do right[i - mid] = a[i] end
        return mergeCount(sortCount(left), sortCount(right))
    end

    sortCount(arr)
    return count
end

local inv = {2, 3, 8, 6, 1}
print("Inversions:", countInversions(inv))  -- 5
```

### Quick Sort

Time Complexity: O(n log n) average, O(n²) worst
Space Complexity: O(log n) average

```lua
-- Quick Sort
local function quickSort(arr, low, high)
    low = low or 1
    high = high or #arr

    local function partition(a, l, h)
        local pivot = a[h]
        local i = l - 1
        for j = l, h - 1 do
            if a[j] <= pivot then
                i = i + 1
                a[i], a[j] = a[j], a[i]
            end
        end
        a[i + 1], a[h] = a[h], a[i + 1]
        return i + 1
    end

    if low < high then
        local pi = partition(arr, low, high)
        quickSort(arr, low, pi - 1)
        quickSort(arr, pi + 1, high)
    end
    return arr
end

local data = {10, 7, 8, 9, 1, 5}
quickSort(data)
print("Quick sort:", table.concat(data, ", "))  -- 1, 5, 7, 8, 9, 10

-- Quick Sort ด้วย median-of-three pivot (better performance)
local function quickSortM3(arr, low, high)
    low = low or 1
    high = high or #arr

    local function medianOfThree(a, l, h)
        local mid = math.floor((l + h) / 2)
        if a[l] > a[mid] then a[l], a[mid] = a[mid], a[l] end
        if a[l] > a[h] then a[l], a[h] = a[h], a[l] end
        if a[mid] > a[h] then a[mid], a[h] = a[h], a[mid] end
        a[mid], a[h - 1] = a[h - 1], a[mid]
        return a[h - 1]
    end

    local function partition(a, l, h)
        if h - l < 2 then
            if a[l] > a[h] then a[l], a[h] = a[h], a[l] end
            return l
        end
        local pivot = medianOfThree(a, l, h)
        local i, j = l, h - 1
        while true do
            i = i + 1
            while a[i] < pivot do i = i + 1 end
            j = j - 1
            while a[j] > pivot do j = j - 1 end
            if i >= j then break end
            a[i], a[j] = a[j], a[i]
        end
        a[i], a[h - 1] = a[h - 1], a[i]
        return i
    end

    if low < high then
        local pi = partition(arr, low, high)
        quickSortM3(arr, low, pi - 1)
        quickSortM3(arr, pi + 1, high)
    end
    return arr
end

local data2 = {3, 6, 8, 10, 1, 2, 1}
quickSortM3(data2)
print("QuickSort M3:", table.concat(data2, ", "))  -- 1, 1, 2, 3, 6, 8, 10
```

### Heap Sort

Time Complexity: O(n log n) all cases
Space Complexity: O(1)

```lua
-- Heap Sort
local function heapSort(arr)
    local n = #arr

    local function heapify(a, n, i)
        local largest = i
        local l = 2 * i
        local r = 2 * i + 1

        if l <= n and a[l] > a[largest] then largest = l end
        if r <= n and a[r] > a[largest] then largest = r end

        if largest ~= i then
            a[i], a[largest] = a[largest], a[i]
            heapify(a, n, largest)
        end
    end

    -- Build max-heap
    for i = math.floor(n / 2), 1, -1 do
        heapify(arr, n, i)
    end

    -- Extract elements
    for i = n, 2, -1 do
        arr[1], arr[i] = arr[i], arr[1]
        heapify(arr, i - 1, 1)
    end

    return arr
end

local data = {12, 11, 13, 5, 6, 7}
heapSort(data)
print("Heap sort:", table.concat(data, ", "))  -- 5, 6, 7, 11, 12, 13
```

### Radix Sort

Time Complexity: O(nk) where k = number of digits
Space Complexity: O(n + k)

```lua
-- Radix Sort (สำหรับ integers ไม่ติดลบ)
local function radixSort(arr)
    local function countingSort(a, exp)
        local n = #a
        local output = {}
        local count = {}
        for i = 0, 9 do count[i] = 0 end

        for _, v in ipairs(a) do
            local digit = math.floor(v / exp) % 10
            count[digit] = count[digit] + 1
        end

        for i = 1, 9 do
            count[i] = count[i] + count[i - 1]
        end

        for i = n, 1, -1 do
            local digit = math.floor(a[i] / exp) % 10
            output[count[digit]] = a[i]
            count[digit] = count[digit] - 1
        end

        for i = 1, n do a[i] = output[i] end
    end

    local maxVal = arr[1]
    for _, v in ipairs(arr) do
        if v > maxVal then maxVal = v end
    end

    local exp = 1
    while math.floor(maxVal / exp) > 0 do
        countingSort(arr, exp)
        exp = exp * 10
    end

    return arr
end

local data = {170, 45, 75, 90, 802, 24, 2, 66}
radixSort(data)
print("Radix sort:", table.concat(data, ", "))  -- 2, 24, 45, 66, 75, 90, 170, 802
```

---

## 28.2 Searching Algorithms

### Linear Search

Time Complexity: O(n)

```lua
-- Linear Search
local function linearSearch(arr, target)
    for i, v in ipairs(arr) do
        if v == target then return i end
    end
    return nil
end

-- Linear Search กับ predicate
local function linearSearchIf(arr, predicate)
    for i, v in ipairs(arr) do
        if predicate(v) then return i, v end
    end
    return nil
end

local data = {3, 1, 4, 1, 5, 9, 2, 6}
print("Find 5:", linearSearch(data, 5))  -- 5
print("Find 7:", linearSearch(data, 7))  -- nil

-- หา first element > 4
local idx, val = linearSearchIf(data, function(x) return x > 4 end)
print(string.format("First > 4: index=%d, value=%d", idx, val))  -- 5, 5
```

### Binary Search

Time Complexity: O(log n) - ต้องใช้กับ sorted array
Space Complexity: O(1) iterative, O(log n) recursive

```lua
-- Binary Search (iterative)
local function binarySearch(arr, target)
    local low, high = 1, #arr
    while low <= high do
        local mid = math.floor((low + high) / 2)
        if arr[mid] == target then
            return mid
        elseif arr[mid] < target then
            low = mid + 1
        else
            high = mid - 1
        end
    end
    return nil
end

-- Binary Search (recursive)
local function binarySearchRec(arr, target, low, high)
    low = low or 1
    high = high or #arr
    if low > high then return nil end
    local mid = math.floor((low + high) / 2)
    if arr[mid] == target then return mid
    elseif arr[mid] < target then return binarySearchRec(arr, target, mid + 1, high)
    else return binarySearchRec(arr, target, low, mid - 1)
    end
end

-- หา leftmost occurrence
local function lowerBound(arr, target)
    local low, high = 1, #arr + 1
    while low < high do
        local mid = math.floor((low + high) / 2)
        if arr[mid] < target then low = mid + 1
        else high = mid
        end
    end
    return low
end

-- หา rightmost occurrence
local function upperBound(arr, target)
    local low, high = 1, #arr + 1
    while low < high do
        local mid = math.floor((low + high) / 2)
        if arr[mid] <= target then low = mid + 1
        else high = mid
        end
    end
    return low - 1
end

local sorted = {1, 2, 3, 3, 3, 4, 5, 6}
print("Search 3:", binarySearch(sorted, 3))     -- 3 หรือ 4 หรือ 5 (any)
print("LowerBound 3:", lowerBound(sorted, 3))   -- 3 (first occurrence)
print("UpperBound 3:", upperBound(sorted, 3))   -- 5 (last occurrence)
print("Count of 3:", upperBound(sorted, 3) - lowerBound(sorted, 3) + 1)  -- 3
```

### Jump Search

Time Complexity: O(√n)
Space Complexity: O(1)

```lua
-- Jump Search
local function jumpSearch(arr, target)
    local n = #arr
    local step = math.floor(math.sqrt(n))
    local prev = 0

    -- Jump ไปข้างหน้า step elements ทีละ block
    while arr[math.min(step, n)] < target do
        prev = step
        step = step + math.floor(math.sqrt(n))
        if prev >= n then return nil end
    end

    -- Linear search ใน block
    while arr[prev + 1] < target do
        prev = prev + 1
        if prev == math.min(step, n) then return nil end
    end

    if arr[prev + 1] == target then
        return prev + 1
    end
    return nil
end

local sorted = {0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233}
print("Jump search 55:", jumpSearch(sorted, 55))   -- 11
print("Jump search 100:", jumpSearch(sorted, 100)) -- nil
```

---

## 28.3 Graph Algorithms

### BFS และ DFS (ครบถ้วน)

```lua
-- Graph representation
local function newGraph(directed)
    return {
        adj = {},
        directed = directed or false,
        addEdge = function(self, u, v, w)
            if not self.adj[u] then self.adj[u] = {} end
            if not self.adj[v] then self.adj[v] = {} end
            table.insert(self.adj[u], { to = v, weight = w or 1 })
            if not self.directed then
                table.insert(self.adj[v], { to = u, weight = w or 1 })
            end
        end,
        neighbors = function(self, v)
            return self.adj[v] or {}
        end
    }
end

-- BFS: Breadth-First Search
local function bfs(graph, start)
    local visited = {}
    local order = {}
    local parent = {}
    local queue = {start}
    visited[start] = true
    parent[start] = nil

    while #queue > 0 do
        local v = table.remove(queue, 1)
        order[#order + 1] = v
        for _, edge in ipairs(graph:neighbors(v)) do
            if not visited[edge.to] then
                visited[edge.to] = true
                parent[edge.to] = v
                queue[#queue + 1] = edge.to
            end
        end
    end
    return order, parent
end

-- DFS: Depth-First Search
local function dfs(graph, start)
    local visited = {}
    local order = {}
    local parent = {}

    local function dfsHelper(v)
        visited[v] = true
        order[#order + 1] = v
        for _, edge in ipairs(graph:neighbors(v)) do
            if not visited[edge.to] then
                parent[edge.to] = v
                dfsHelper(edge.to)
            end
        end
    end

    parent[start] = nil
    dfsHelper(start)
    return order, parent
end

-- ทดสอบ
local g = newGraph(false)
g:addEdge("A", "B")
g:addEdge("A", "C")
g:addEdge("B", "D")
g:addEdge("C", "E")
g:addEdge("D", "E")
g:addEdge("E", "F")

local bfsOrder = bfs(g, "A")
local dfsOrder = dfs(g, "A")
print("BFS:", table.concat(bfsOrder, " -> "))
print("DFS:", table.concat(dfsOrder, " -> "))
```

### Dijkstra's Algorithm

Time Complexity: O((V + E) log V) with priority queue

```lua
-- Dijkstra's Shortest Path Algorithm
local function dijkstra(graph, start)
    local INF = math.huge
    local dist = {}
    local visited = {}
    local prev = {}

    -- Initialize
    for v in pairs(graph.adj) do
        dist[v] = INF
        prev[v] = nil
    end
    dist[start] = 0

    -- Simple priority queue (min-heap would be better)
    local function getMinUnvisited()
        local minDist = INF
        local minV = nil
        for v, d in pairs(dist) do
            if not visited[v] and d < minDist then
                minDist = d
                minV = v
            end
        end
        return minV
    end

    while true do
        local u = getMinUnvisited()
        if u == nil then break end
        visited[u] = true

        for _, edge in ipairs(graph:neighbors(u)) do
            local v = edge.to
            local w = edge.weight
            if not visited[v] and dist[u] + w < dist[v] then
                dist[v] = dist[u] + w
                prev[v] = u
            end
        end
    end

    return dist, prev
end

-- สร้าง shortest path จาก prev table
local function reconstructPath(prev, target)
    local path = {}
    local current = target
    while current do
        table.insert(path, 1, current)
        current = prev[current]
    end
    return path
end

-- ทดสอบ Dijkstra
local wg = newGraph(true)
wg:addEdge("A", "B", 4)
wg:addEdge("A", "C", 2)
wg:addEdge("B", "D", 3)
wg:addEdge("C", "B", 1)
wg:addEdge("C", "D", 5)
wg:addEdge("D", "E", 1)
wg:addEdge("B", "E", 6)

local dist, prev = dijkstra(wg, "A")
print("\nDijkstra from A:")
for v, d in pairs(dist) do
    if d < math.huge then
        local path = reconstructPath(prev, v)
        print(string.format("  To %s: dist=%d, path=%s", v, d, table.concat(path, "->")))
    end
end
```

### A* Algorithm

```lua
-- A* Algorithm
local function aStar(graph, start, goal, heuristic)
    local INF = math.huge
    local openSet = { [start] = true }
    local cameFrom = {}
    local gScore = {}
    local fScore = {}

    for v in pairs(graph.adj) do
        gScore[v] = INF
        fScore[v] = INF
    end
    gScore[start] = 0
    fScore[start] = heuristic(start, goal)

    local function getLowestF()
        local minF = INF
        local minV = nil
        for v in pairs(openSet) do
            if fScore[v] < minF then
                minF = fScore[v]
                minV = v
            end
        end
        return minV
    end

    while next(openSet) do
        local current = getLowestF()
        if current == goal then
            -- Reconstruct path
            local path = {}
            local c = goal
            while c do
                table.insert(path, 1, c)
                c = cameFrom[c]
            end
            return path, gScore[goal]
        end

        openSet[current] = nil
        for _, edge in ipairs(graph:neighbors(current)) do
            local neighbor = edge.to
            local tentativeG = gScore[current] + edge.weight
            if tentativeG < gScore[neighbor] then
                cameFrom[neighbor] = current
                gScore[neighbor] = tentativeG
                fScore[neighbor] = gScore[neighbor] + heuristic(neighbor, goal)
                openSet[neighbor] = true
            end
        end
    end

    return nil, INF  -- ไม่พบ path
end

-- ทดสอบ A* บน grid (2D)
-- สร้าง grid graph
local function createGridGraph(rows, cols, obstacles)
    local g = newGraph(false)
    local obstSet = {}
    for _, o in ipairs(obstacles or {}) do
        obstSet[o[1] .. "," .. o[2]] = true
    end

    local function node(r, c) return r .. "," .. c end
    local function valid(r, c)
        return r >= 1 and r <= rows and c >= 1 and c <= cols
            and not obstSet[node(r, c)]
    end

    local dirs = {{0,1},{0,-1},{1,0},{-1,0}}
    for r = 1, rows do
        for c = 1, cols do
            if not obstSet[node(r,c)] then
                for _, d in ipairs(dirs) do
                    local nr, nc = r + d[1], c + d[2]
                    if valid(nr, nc) then
                        g:addEdge(node(r,c), node(nr,nc), 1)
                    end
                end
            end
        end
    end
    return g
end

local function manhattanDist(a, b)
    local ar, ac = a:match("(%d+),(%d+)")
    local br, bc = b:match("(%d+),(%d+)")
    return math.abs(tonumber(ar) - tonumber(br)) + math.abs(tonumber(ac) - tonumber(bc))
end

local obstacles = {{2,2},{2,3},{3,2}}
local grid = createGridGraph(4, 4, obstacles)
local path, cost = aStar(grid, "1,1", "4,4", manhattanDist)
if path then
    print("\nA* path from (1,1) to (4,4):", table.concat(path, " -> "))
    print("Cost:", cost)
end
```

---

## 28.4 String Algorithms

### KMP Pattern Matching

Time Complexity: O(n + m) where n = text length, m = pattern length

```lua
-- KMP (Knuth-Morris-Pratt) Algorithm
local function kmpSearch(text, pattern)
    local n = #text
    local m = #pattern
    local matches = {}

    if m == 0 then return matches end

    -- Build failure function (partial match table)
    local function buildLPS(p)
        local lps = {}
        lps[1] = 0
        local len = 0
        local i = 2
        while i <= #p do
            if p:sub(i, i) == p:sub(len + 1, len + 1) then
                len = len + 1
                lps[i] = len
                i = i + 1
            else
                if len ~= 0 then
                    len = lps[len]
                else
                    lps[i] = 0
                    i = i + 1
                end
            end
        end
        return lps
    end

    local lps = buildLPS(pattern)
    local i = 1  -- text index
    local j = 1  -- pattern index

    while i <= n do
        if text:sub(i, i) == pattern:sub(j, j) then
            i = i + 1
            j = j + 1
        end

        if j > m then
            matches[#matches + 1] = i - j  -- found at position
            j = lps[j - 1] + 1
        elseif i <= n and text:sub(i, i) ~= pattern:sub(j, j) then
            if j ~= 1 then
                j = lps[j - 1] + 1
            else
                i = i + 1
            end
        end
    end

    return matches
end

local text = "AABAACAADAABAABA"
local pattern = "AABA"
local positions = kmpSearch(text, pattern)
print("KMP matches of '" .. pattern .. "' in '" .. text .. "':")
print("Positions:", table.concat(positions, ", "))  -- 1, 10, 13

-- เปรียบเทียบกับ naive search
local function naiveSearch(text, pattern)
    local n, m = #text, #pattern
    local matches = {}
    for i = 1, n - m + 1 do
        if text:sub(i, i + m - 1) == pattern then
            matches[#matches + 1] = i
        end
    end
    return matches
end

local positions2 = naiveSearch(text, pattern)
print("Naive matches:", table.concat(positions2, ", "))  -- same result
```

### Levenshtein Distance

Time Complexity: O(mn)
Space Complexity: O(mn)

```lua
-- Levenshtein Distance (Edit Distance)
local function levenshtein(s1, s2)
    local m, n = #s1, #s2
    local dp = {}

    for i = 0, m do
        dp[i] = {}
        dp[i][0] = i
    end
    for j = 0, n do
        dp[0][j] = j
    end

    for i = 1, m do
        for j = 1, n do
            if s1:sub(i, i) == s2:sub(j, j) then
                dp[i][j] = dp[i-1][j-1]
            else
                dp[i][j] = 1 + math.min(
                    dp[i-1][j],    -- delete
                    dp[i][j-1],    -- insert
                    dp[i-1][j-1]   -- replace
                )
            end
        end
    end

    return dp[m][n]
end

print("\nLevenshtein Distance:")
print("kitten -> sitting:", levenshtein("kitten", "sitting"))  -- 3
print("sunday -> saturday:", levenshtein("sunday", "saturday")) -- 3
print("hello -> hello:", levenshtein("hello", "hello"))        -- 0
print("cat -> dog:", levenshtein("cat", "dog"))                -- 3

-- หา closest word จาก dictionary
local function findClosestWord(word, dictionary)
    local minDist = math.huge
    local closest = nil
    for _, w in ipairs(dictionary) do
        local d = levenshtein(word, w)
        if d < minDist then
            minDist = d
            closest = w
        end
    end
    return closest, minDist
end

local dictionary = {"apple", "application", "apply", "apt", "apt",
                    "banana", "band", "bandana"}
local word = "appel"
local closest, dist = findClosestWord(word, dictionary)
print(string.format("Closest to '%s': '%s' (distance: %d)", word, closest, dist))
-- Closest to 'appel': 'apple' (distance: 1)
```

---

## 28.5 Dynamic Programming

### Fibonacci

```lua
-- Fibonacci แบบต่างๆ

-- Naive recursive: O(2^n)
local function fibNaive(n)
    if n <= 1 then return n end
    return fibNaive(n-1) + fibNaive(n-2)
end

-- Memoization (Top-down): O(n)
local function fibMemo(n)
    local memo = {}
    local function fib(k)
        if k <= 1 then return k end
        if memo[k] then return memo[k] end
        memo[k] = fib(k-1) + fib(k-2)
        return memo[k]
    end
    return fib(n)
end

-- Tabulation (Bottom-up): O(n), O(n) space
local function fibDP(n)
    if n <= 1 then return n end
    local dp = {[0] = 0, [1] = 1}
    for i = 2, n do
        dp[i] = dp[i-1] + dp[i-2]
    end
    return dp[n]
end

-- Space optimized: O(n), O(1) space
local function fibOpt(n)
    if n <= 1 then return n end
    local a, b = 0, 1
    for _ = 2, n do
        a, b = b, a + b
    end
    return b
end

print("Fibonacci(10):", fibOpt(10))  -- 55
print("Fibonacci(20):", fibOpt(20))  -- 6765
print("Fibonacci(30):", fibOpt(30))  -- 832040

-- เปรียบเทียบ performance
local t1 = os.clock()
fibNaive(30)
print(string.format("Naive fib(30): %.4f s", os.clock() - t1))

local t2 = os.clock()
fibMemo(30)
print(string.format("Memo fib(30): %.6f s", os.clock() - t2))
```

### 0/1 Knapsack Problem

Time Complexity: O(nW)
Space Complexity: O(nW)

```lua
-- 0/1 Knapsack Problem
local function knapsack(weights, values, capacity)
    local n = #weights
    local dp = {}

    -- Initialize DP table
    for i = 0, n do
        dp[i] = {}
        for w = 0, capacity do
            dp[i][w] = 0
        end
    end

    -- Fill DP table
    for i = 1, n do
        for w = 0, capacity do
            -- ไม่เอา item i
            dp[i][w] = dp[i-1][w]
            -- เอา item i (ถ้าน้ำหนักพอ)
            if weights[i] <= w then
                local withItem = dp[i-1][w - weights[i]] + values[i]
                if withItem > dp[i][w] then
                    dp[i][w] = withItem
                end
            end
        end
    end

    -- Trace back items ที่เลือก
    local selected = {}
    local w = capacity
    for i = n, 1, -1 do
        if dp[i][w] ~= dp[i-1][w] then
            selected[#selected + 1] = i
            w = w - weights[i]
        end
    end

    return dp[n][capacity], selected
end

local weights = {2, 3, 4, 5}
local values  = {3, 4, 5, 6}
local capacity = 8

local maxValue, items = knapsack(weights, values, capacity)
print("\nKnapsack (capacity=8):")
print("Max value:", maxValue)  -- 10
print("Selected items:", table.concat(items, ", "))

-- พิมพ์รายละเอียด
for _, i in ipairs(items) do
    print(string.format("  Item %d: weight=%d, value=%d", i, weights[i], values[i]))
end
```

### Longest Common Subsequence (LCS)

Time Complexity: O(mn)

```lua
-- Longest Common Subsequence
local function lcs(s1, s2)
    local m, n = #s1, #s2
    local dp = {}

    for i = 0, m do
        dp[i] = {}
        for j = 0, n do
            dp[i][j] = 0
        end
    end

    for i = 1, m do
        for j = 1, n do
            if s1:sub(i,i) == s2:sub(j,j) then
                dp[i][j] = dp[i-1][j-1] + 1
            else
                dp[i][j] = math.max(dp[i-1][j], dp[i][j-1])
            end
        end
    end

    -- Trace back LCS string
    local result = {}
    local i, j = m, n
    while i > 0 and j > 0 do
        if s1:sub(i,i) == s2:sub(j,j) then
            table.insert(result, 1, s1:sub(i,i))
            i, j = i-1, j-1
        elseif dp[i-1][j] > dp[i][j-1] then
            i = i - 1
        else
            j = j - 1
        end
    end

    return dp[m][n], table.concat(result)
end

local s1 = "ABCBDAB"
local s2 = "BDCAB"
local length, seq = lcs(s1, s2)
print("\nLCS of", s1, "and", s2)
print("Length:", length)  -- 4
print("LCS:", seq)        -- BDAB หรือ BCAB

-- LCS สำหรับหา diff
local function diff(old, new)
    local function words(s)
        local w = {}
        for word in s:gmatch("%S+") do w[#w+1] = word end
        return w
    end

    local a, b = words(old), words(new)
    local _, lcsSeq = lcs(table.concat(a, " "), table.concat(b, " "))
    return lcsSeq
end
```

---

## 28.6 Divide and Conquer

```lua
-- Maximum Subarray (Kadane's Algorithm) - O(n)
local function maxSubarray(arr)
    local maxSum = arr[1]
    local currentSum = arr[1]
    local start, stop, tempStart = 1, 1, 1

    for i = 2, #arr do
        if currentSum + arr[i] < arr[i] then
            currentSum = arr[i]
            tempStart = i
        else
            currentSum = currentSum + arr[i]
        end

        if currentSum > maxSum then
            maxSum = currentSum
            start = tempStart
            stop = i
        end
    end

    return maxSum, start, stop
end

local arr = {-2, 1, -3, 4, -1, 2, 1, -5, 4}
local sum, s, e = maxSubarray(arr)
print("\nMax subarray:")
print("Sum:", sum)  -- 6
print("From index", s, "to", e)  -- 4 to 7
print("Subarray:", table.concat({table.unpack(arr, s, e)}, ", "))  -- 4,-1,2,1

-- Power function (fast exponentiation) - O(log n)
local function fastPower(base, exp, mod)
    mod = mod or math.maxinteger
    local result = 1
    base = base % mod
    while exp > 0 do
        if exp % 2 == 1 then
            result = (result * base) % mod
        end
        exp = math.floor(exp / 2)
        base = (base * base) % mod
    end
    return result
end

print("\nFast Power:")
print("2^10:", fastPower(2, 10))        -- 1024
print("3^20 mod 1000007:", fastPower(3, 20, 1000007))
print("2^100 mod 10^9+7:", fastPower(2, 100, 10^9 + 7))
```

---

## 28.7 Greedy Algorithms

### Activity Selection

```lua
-- Activity Selection Problem
local function activitySelection(activities)
    -- Sort by finish time
    table.sort(activities, function(a, b) return a.finish < b.finish end)

    local selected = { activities[1] }
    local lastFinish = activities[1].finish

    for i = 2, #activities do
        if activities[i].start >= lastFinish then
            selected[#selected + 1] = activities[i]
            lastFinish = activities[i].finish
        end
    end

    return selected
end

local activities = {
    { name = "A1", start = 1, finish = 4 },
    { name = "A2", start = 3, finish = 5 },
    { name = "A3", start = 0, finish = 6 },
    { name = "A4", start = 5, finish = 7 },
    { name = "A5", start = 3, finish = 9 },
    { name = "A6", start = 5, finish = 9 },
    { name = "A7", start = 6, finish = 10 },
    { name = "A8", start = 8, finish = 11 },
    { name = "A9", start = 8, finish = 12 },
    { name = "A10", start = 2, finish = 14 },
    { name = "A11", start = 12, finish = 16 },
}

local selected = activitySelection(activities)
print("\nActivity Selection:")
print("Selected activities:")
for _, a in ipairs(selected) do
    print(string.format("  %s [%d - %d]", a.name, a.start, a.finish))
end
```

### Fractional Knapsack (Greedy)

```lua
-- Fractional Knapsack (Greedy)
local function fractionalKnapsack(items, capacity)
    -- Calculate value/weight ratio and sort descending
    local items_copy = {}
    for i, item in ipairs(items) do
        items_copy[i] = {
            name = item.name,
            weight = item.weight,
            value = item.value,
            ratio = item.value / item.weight
        }
    end
    table.sort(items_copy, function(a, b) return a.ratio > b.ratio end)

    local totalValue = 0
    local selected = {}
    local remaining = capacity

    for _, item in ipairs(items_copy) do
        if remaining <= 0 then break end
        if item.weight <= remaining then
            selected[#selected + 1] = { name = item.name, fraction = 1.0, value = item.value }
            totalValue = totalValue + item.value
            remaining = remaining - item.weight
        else
            local fraction = remaining / item.weight
            selected[#selected + 1] = {
                name = item.name,
                fraction = fraction,
                value = item.value * fraction
            }
            totalValue = totalValue + item.value * fraction
            remaining = 0
        end
    end

    return totalValue, selected
end

local items = {
    { name = "ทอง",    weight = 10, value = 60 },
    { name = "เงิน",   weight = 20, value = 100 },
    { name = "ทองแดง", weight = 30, value = 120 },
}

local maxVal, picked = fractionalKnapsack(items, 50)
print("\nFractional Knapsack (capacity=50):")
print(string.format("Max value: %.2f", maxVal))
for _, p in ipairs(picked) do
    print(string.format("  %s: %.0f%%", p.name, p.fraction * 100))
end
```

---

## 28.8 สรุป Time Complexity

```lua
print("\n=== Algorithm Complexity Summary ===")
print(string.format("%-25s %-15s %-15s %-10s", "Algorithm", "Best", "Average", "Worst"))
print(string.rep("-", 70))

local complexities = {
    {"Bubble Sort",     "O(n)",      "O(n²)",     "O(n²)"},
    {"Selection Sort",  "O(n²)",     "O(n²)",     "O(n²)"},
    {"Insertion Sort",  "O(n)",      "O(n²)",     "O(n²)"},
    {"Merge Sort",      "O(n log n)","O(n log n)","O(n log n)"},
    {"Quick Sort",      "O(n log n)","O(n log n)","O(n²)"},
    {"Heap Sort",       "O(n log n)","O(n log n)","O(n log n)"},
    {"Radix Sort",      "O(nk)",     "O(nk)",     "O(nk)"},
    {"Linear Search",   "O(1)",      "O(n)",      "O(n)"},
    {"Binary Search",   "O(1)",      "O(log n)",  "O(log n)"},
    {"BFS",             "O(V+E)",    "O(V+E)",    "O(V+E)"},
    {"DFS",             "O(V+E)",    "O(V+E)",    "O(V+E)"},
    {"Dijkstra",        "O((V+E)logV)","O((V+E)logV)","O(V²)"},
    {"KMP",             "O(n+m)",    "O(n+m)",    "O(n+m)"},
    {"Levenshtein",     "O(mn)",     "O(mn)",     "O(mn)"},
}

for _, row in ipairs(complexities) do
    print(string.format("%-25s %-15s %-15s %-10s",
        row[1], row[2], row[3], row[4]))
end
```

---

## บทสรุป

การเลือก Algorithm ที่เหมาะสมขึ้นอยู่กับ:
1. **ขนาด Input** - n เล็ก: algorithm ใดก็ได้, n ใหญ่: ต้องใช้ที่มี time complexity ต่ำ
2. **ลักษณะ Data** - sorted แล้วหรือยัง, มี duplicates, range ของค่า
3. **Operation ที่ใช้บ่อย** - search, insert, delete, sort
4. **Memory Constraint** - O(1) vs O(n) space
5. **Stability** - ต้องการ stable sort หรือไม่ (เช่น Merge Sort เป็น stable)
