# บทที่ 91: Custom Memory Allocator ใน Lua

## บทนำ

Memory management เป็นหัวใจสำคัญของ performance-critical applications การเข้าใจและควบคุม memory allocation ใน Lua ช่วยให้เราสร้างระบบที่มีประสิทธิภาพสูงสุดได้

---

## 91.1 Lua's Default Memory Allocator

Lua ใช้ `realloc()` จาก C standard library เป็น default allocator:

```lua
-- ดู memory usage ปัจจุบัน
print("Memory used:", collectgarbage("count"), "KB")

-- Force full GC cycle
collectgarbage("collect")
print("After GC:", collectgarbage("count"), "KB")

-- ดู GC parameters
print("GC pause:", collectgarbage("param", "pause"))
print("GC stepmul:", collectgarbage("param", "stepmul"))
```

### lua_Alloc Function Signature (C)

```c
/* The allocator function type */
typedef void * (*lua_Alloc) (void *ud, void *ptr, size_t osize, size_t nsize);

/*
 * ud    = userdata pointer passed to lua_newstate
 * ptr   = pointer to block being allocated/reallocated/freed
 * osize = original size of block (0 if new allocation)
 * nsize = new size requested (0 if freeing)
 * 
 * Return: new pointer, or NULL if nsize == 0
 */
```

---

## 91.2 วัด Memory Usage อย่างละเอียด

```lua
-- Memory tracking module
local MemTracker = {}
MemTracker.__index = MemTracker

function MemTracker.new()
    return setmetatable({
        snapshots = {},
        baseline = collectgarbage("count")
    }, MemTracker)
end

function MemTracker:snapshot(label)
    collectgarbage("collect")  -- force GC before measuring
    local kb = collectgarbage("count")
    table.insert(self.snapshots, {
        label = label,
        kb = kb,
        delta = kb - self.baseline,
        time = os.clock()
    })
    return kb
end

function MemTracker:report()
    print(string.format("%-30s %10s %10s", "Label", "KB", "Delta KB"))
    print(string.rep("-", 55))
    for _, s in ipairs(self.snapshots) do
        print(string.format("%-30s %10.2f %10.2f", 
            s.label, s.kb, s.delta))
    end
end

-- การใช้งาน
local tracker = MemTracker.new()
tracker:snapshot("Start")

-- สร้าง objects จำนวนมาก
local t = {}
for i = 1, 10000 do
    t[i] = { x = i, y = i * 2, name = "obj_" .. i }
end
tracker:snapshot("After 10k objects")

-- ลบ objects
t = nil
collectgarbage("collect")
tracker:snapshot("After GC")

tracker:report()
```

**Output:**
```
Label                             KB     Delta KB
-------------------------------------------------------
Start                          25.45       0.00
After 10k objects            1843.21    1817.76
After GC                       26.12       0.67
```

---

## 91.3 Object Pool Allocator

Object Pool เป็น pattern ที่ดีที่สุดสำหรับ reduce GC pressure:

```lua
-- Generic Object Pool
local ObjectPool = {}
ObjectPool.__index = ObjectPool

function ObjectPool.new(factory, reset_fn, initial_size)
    local pool = setmetatable({
        factory = factory,
        reset_fn = reset_fn or function(obj) return obj end,
        free_list = {},
        active_count = 0,
        total_created = 0,
        reuse_count = 0
    }, ObjectPool)
    
    -- Pre-allocate
    for i = 1, (initial_size or 0) do
        table.insert(pool.free_list, factory())
        pool.total_created = pool.total_created + 1
    end
    
    return pool
end

function ObjectPool:acquire()
    local obj
    if #self.free_list > 0 then
        obj = table.remove(self.free_list)
        self.reuse_count = self.reuse_count + 1
    else
        obj = self.factory()
        self.total_created = self.total_created + 1
    end
    self.active_count = self.active_count + 1
    return obj
end

function ObjectPool:release(obj)
    self.reset_fn(obj)
    table.insert(self.free_list, obj)
    self.active_count = self.active_count - 1
end

function ObjectPool:stats()
    return {
        free = #self.free_list,
        active = self.active_count,
        total_created = self.total_created,
        reuse_count = self.reuse_count,
        reuse_ratio = self.reuse_count / math.max(1, self.total_created + self.reuse_count)
    }
end

-- ตัวอย่าง: Particle Pool สำหรับ game
local ParticlePool = ObjectPool.new(
    -- factory
    function()
        return { x=0, y=0, vx=0, vy=0, life=0, active=false, color={1,1,1,1} }
    end,
    -- reset function
    function(p)
        p.x, p.y = 0, 0
        p.vx, p.vy = 0, 0
        p.life = 0
        p.active = false
        p.color[1], p.color[2], p.color[3], p.color[4] = 1, 1, 1, 1
        return p
    end,
    100  -- pre-allocate 100 particles
)

-- Simulate particle system
local active_particles = {}

local function spawn_particle(x, y)
    local p = ParticlePool:acquire()
    p.x, p.y = x, y
    p.vx = math.random() * 2 - 1
    p.vy = math.random() * 2 - 1
    p.life = 1.0
    p.active = true
    table.insert(active_particles, p)
    return p
end

local function update_particles(dt)
    local i = 1
    while i <= #active_particles do
        local p = active_particles[i]
        p.x = p.x + p.vx * dt
        p.y = p.y + p.vy * dt
        p.life = p.life - dt
        
        if p.life <= 0 then
            ParticlePool:release(p)
            table.remove(active_particles, i)
        else
            i = i + 1
        end
    end
end

-- Simulate 1000 frames
math.randomseed(42)
for frame = 1, 1000 do
    -- Spawn some particles
    for i = 1, 10 do
        spawn_particle(math.random(0, 800), math.random(0, 600))
    end
    update_particles(1/60)
end

local stats = ParticlePool:stats()
print("Pool Stats:")
print("  Free:", stats.free)
print("  Active:", stats.active)
print("  Total created:", stats.total_created)
print("  Reuse count:", stats.reuse_count)
print(string.format("  Reuse ratio: %.1f%%", stats.reuse_ratio * 100))
```

---

## 91.4 Arena Allocator Pattern

Arena allocator จัดสรร memory จาก pre-allocated block ใหญ่:

```lua
-- Arena Allocator (Lua-level simulation)
local Arena = {}
Arena.__index = Arena

function Arena.new(capacity)
    return setmetatable({
        data = {},         -- ใช้ table แทน raw memory
        allocated = 0,
        capacity = capacity or 1024 * 1024,  -- 1MB default
        checkpoints = {}
    }, Arena)
end

function Arena:alloc(size, init_fn)
    if self.allocated + size > self.capacity then
        error("Arena out of memory! Used: " .. self.allocated .. 
              " / " .. self.capacity)
    end
    
    local obj = init_fn and init_fn() or {}
    local id = self.allocated + 1
    self.data[id] = obj
    self.allocated = self.allocated + size
    return id, obj
end

function Arena:checkpoint()
    table.insert(self.checkpoints, {
        allocated = self.allocated,
        data_size = #self.data
    })
    return #self.checkpoints
end

function Arena:rollback(checkpoint_id)
    local cp = self.checkpoints[checkpoint_id]
    if not cp then
        error("Invalid checkpoint: " .. tostring(checkpoint_id))
    end
    
    -- Free everything allocated after checkpoint
    for i = cp.data_size + 1, #self.data do
        self.data[i] = nil
    end
    self.allocated = cp.allocated
    
    -- Remove checkpoints after this one
    for i = #self.checkpoints, checkpoint_id, -1 do
        self.checkpoints[i] = nil
    end
end

function Arena:reset()
    self.data = {}
    self.allocated = 0
    self.checkpoints = {}
end

function Arena:usage()
    return {
        used = self.allocated,
        capacity = self.capacity,
        percent = (self.allocated / self.capacity) * 100,
        objects = #self.data
    }
end

-- การใช้งาน: Frame Arena สำหรับ game loop
local frame_arena = Arena.new(10 * 1024 * 1024)  -- 10MB per frame

-- Game loop simulation
for frame = 1, 3 do
    print("=== Frame " .. frame .. " ===")
    
    -- Checkpoint ต้น frame
    local frame_start = frame_arena:checkpoint()
    
    -- Allocate temporary data สำหรับ frame นี้
    for i = 1, 100 do
        local id, entity = frame_arena:alloc(64, function()
            return { x=0, y=0, name="entity_" .. i }
        end)
    end
    
    local usage = frame_arena:usage()
    print(string.format("  During frame: %.2f KB used", usage.used / 1024))
    
    -- Rollback ทุก frame (คืน memory)
    frame_arena:rollback(frame_start)
    
    usage = frame_arena:usage()
    print(string.format("  After rollback: %.2f KB used", usage.used / 1024))
end
```

---

## 91.5 Slab Allocator

Slab allocator เหมาะสำหรับ objects ขนาดเท่ากัน:

```lua
-- Slab Cache
local SlabCache = {}
SlabCache.__index = SlabCache

function SlabCache.new(object_size, slab_size)
    return setmetatable({
        object_size = object_size,
        slab_size = slab_size or 64,  -- objects per slab
        slabs = {},
        free_objects = {},
        stats = {
            slabs_created = 0,
            allocations = 0,
            frees = 0
        }
    }, SlabCache)
end

function SlabCache:_new_slab()
    local slab = {
        objects = {},
        free_count = self.slab_size
    }
    
    for i = 1, self.slab_size do
        slab.objects[i] = self:_create_object()
        table.insert(self.free_objects, slab.objects[i])
    end
    
    table.insert(self.slabs, slab)
    self.stats.slabs_created = self.stats.slabs_created + 1
    return slab
end

function SlabCache:_create_object()
    -- Generic object creation
    return { _slab_free = true, data = {} }
end

function SlabCache:alloc()
    if #self.free_objects == 0 then
        self:_new_slab()
    end
    
    local obj = table.remove(self.free_objects)
    obj._slab_free = false
    self.stats.allocations = self.stats.allocations + 1
    return obj
end

function SlabCache:free(obj)
    if obj._slab_free then
        error("Double free detected!")
    end
    obj._slab_free = true
    -- Clear data
    for k in pairs(obj.data) do
        obj.data[k] = nil
    end
    table.insert(self.free_objects, obj)
    self.stats.frees = self.stats.frees + 1
end

function SlabCache:report()
    local s = self.stats
    print(string.format("Slab Cache Stats:"))
    print(string.format("  Slabs: %d (%.2f KB)", 
        s.slabs_created, 
        s.slabs_created * self.slab_size * self.object_size / 1024))
    print(string.format("  Allocations: %d", s.allocations))
    print(string.format("  Frees: %d", s.frees))
    print(string.format("  Active: %d", s.allocations - s.frees))
    print(string.format("  Free pool: %d", #self.free_objects))
end

-- ตัวอย่าง: Network packet slab
local packet_cache = SlabCache.new(512, 32)

-- Simulate network traffic
local active_packets = {}
for i = 1, 1000 do
    local pkt = packet_cache:alloc()
    pkt.data.seq = i
    pkt.data.src = "192.168.1." .. math.random(1, 254)
    pkt.data.dst = "10.0.0.1"
    pkt.data.payload = string.rep("x", math.random(10, 100))
    table.insert(active_packets, pkt)
    
    -- Process and free some
    if #active_packets > 50 then
        local old = table.remove(active_packets, 1)
        packet_cache:free(old)
    end
end

-- Cleanup
for _, pkt in ipairs(active_packets) do
    packet_cache:free(pkt)
end

packet_cache:report()
```

---

## 91.6 Memory Leak Detection

```lua
-- Memory Leak Detector
local LeakDetector = {}
LeakDetector.__index = LeakDetector

function LeakDetector.new()
    return setmetatable({
        allocations = {},  -- id -> {stack, size, time}
        next_id = 1,
        enabled = true
    }, LeakDetector)
end

function LeakDetector:track(obj, label, size)
    if not self.enabled then return obj end
    
    local id = self.next_id
    self.next_id = self.next_id + 1
    
    self.allocations[id] = {
        label = label or "unknown",
        size = size or 0,
        time = os.clock(),
        stack = debug.traceback("", 2)
    }
    
    -- Attach ID to object
    if type(obj) == "table" then
        rawset(obj, "_leak_id", id)
    end
    
    return obj, id
end

function LeakDetector:release(id_or_obj)
    local id
    if type(id_or_obj) == "table" then
        id = rawget(id_or_obj, "_leak_id")
    else
        id = id_or_obj
    end
    
    if id and self.allocations[id] then
        self.allocations[id] = nil
        return true
    end
    return false
end

function LeakDetector:report()
    local count = 0
    local total_size = 0
    
    print("=== Memory Leak Report ===")
    for id, info in pairs(self.allocations) do
        count = count + 1
        total_size = total_size + info.size
        print(string.format("LEAK #%d: %s (size=%d, age=%.2fs)",
            id, info.label, info.size, os.clock() - info.time))
    end
    
    if count == 0 then
        print("No leaks detected!")
    else
        print(string.format("\nTotal: %d leaks, %d bytes", count, total_size))
    end
end

-- การใช้งาน
local detector = LeakDetector.new()

-- ติดตาม allocation
local obj1, id1 = detector:track({name="request_1"}, "HTTP Request", 1024)
local obj2, id2 = detector:track({name="request_2"}, "HTTP Request", 2048)
local conn, id3 = detector:track({host="db"}, "DB Connection", 4096)

-- Release บางส่วน (simulate proper cleanup)
detector:release(obj1)
detector:release(id2)
-- ลืม release conn! -> leak

detector:report()
-- Output: LEAK #3: DB Connection (size=4096, ...)
```

---

## 91.7 Weak Table สำหรับ Cache

```lua
-- Weak-keyed cache (GC-friendly)
local WeakCache = {}
WeakCache.__index = WeakCache

function WeakCache.new(mode)
    local cache = setmetatable({
        _data = setmetatable({}, { __mode = mode or "v" }),
        hits = 0,
        misses = 0
    }, WeakCache)
    return cache
end

function WeakCache:get(key)
    local val = self._data[key]
    if val ~= nil then
        self.hits = self.hits + 1
        return val
    end
    self.misses = self.misses + 1
    return nil
end

function WeakCache:set(key, value)
    self._data[key] = value
end

function WeakCache:stats()
    local count = 0
    for _ in pairs(self._data) do count = count + 1 end
    return {
        entries = count,
        hits = self.hits,
        misses = self.misses,
        hit_rate = self.hits / math.max(1, self.hits + self.misses)
    }
end

-- Image cache example - images can be GC'd when not referenced
local image_cache = WeakCache.new("v")  -- weak values

local function load_image(path)
    local cached = image_cache:get(path)
    if cached then
        return cached
    end
    
    -- Simulate loading image
    local img = { path = path, data = string.rep("pixel", 1000), width=100, height=100 }
    image_cache:set(path, img)
    return img
end

-- Use images
local img1 = load_image("/textures/player.png")
local img2 = load_image("/textures/enemy.png")
local img3 = load_image("/textures/player.png")  -- cache hit

print("Hit?", img1 == img3)  -- true, same table

local stats = image_cache:stats()
print(string.format("Cache: %d entries, %.0f%% hit rate",
    stats.entries, stats.hit_rate * 100))

-- Wenn img1, img2 out of scope, they can be GC'd
```

---

## 91.8 Generational GC Tuning (Lua 5.4)

```lua
-- Lua 5.4 generational GC
-- Two modes: incremental (default) and generational

-- Check current GC mode
-- In Lua 5.4: collectgarbage("incremental") or collectgarbage("generational")

-- Incremental mode parameters
local function set_incremental_gc(pause, stepmul, stepsize)
    collectgarbage("incremental", pause, stepmul, stepsize)
end

-- Generational mode parameters  
local function set_generational_gc(minormul, majormul)
    collectgarbage("generational", minormul, majormul)
end

-- GC tuning for different scenarios

-- Scenario 1: Game (minimize GC pauses)
local function tune_for_game()
    -- Generational GC with frequent minor collections
    -- and rare major collections
    collectgarbage("generational", 20, 100)
    print("Tuned for game: generational GC")
end

-- Scenario 2: Server (throughput optimized)
local function tune_for_server()
    -- Incremental GC with low pause
    collectgarbage("incremental", 100, 200, 13)
    print("Tuned for server: incremental GC")
end

-- Scenario 3: Batch processing (collect less frequently)
local function tune_for_batch()
    collectgarbage("incremental", 200, 400, 13)
    print("Tuned for batch: high pause incremental")
end

-- GC benchmark
local function gc_benchmark(name, setup_fn)
    setup_fn()
    collectgarbage("collect")
    
    local start_mem = collectgarbage("count")
    local start_time = os.clock()
    
    -- Allocate lots of short-lived objects
    for i = 1, 100000 do
        local _ = { x = i, y = i * 2, s = tostring(i) }
    end
    
    local alloc_time = os.clock() - start_time
    collectgarbage("collect")
    local gc_time = os.clock() - start_time - alloc_time
    
    print(string.format("%s: alloc=%.3fs, gc=%.3fs, mem=%.1fKB",
        name, alloc_time, gc_time, collectgarbage("count") - start_mem))
end

gc_benchmark("Default", function() 
    collectgarbage("incremental") 
end)
```

---

## 91.9 Custom Allocator via C (Concept)

```c
/* custom_alloc.c - Custom allocator สำหรับ Lua */
#include <lua.h>
#include <stdlib.h>
#include <string.h>

/* Statistics */
static struct {
    size_t total_allocated;
    size_t total_freed;
    size_t current_usage;
    size_t peak_usage;
    int allocation_count;
    int free_count;
} alloc_stats = {0};

/* Custom allocator function */
static void* tracked_alloc(void *ud, void *ptr, size_t osize, size_t nsize) {
    (void)ud;  /* unused */
    
    if (nsize == 0) {
        /* Free */
        if (ptr != NULL) {
            alloc_stats.total_freed += osize;
            alloc_stats.current_usage -= osize;
            alloc_stats.free_count++;
            free(ptr);
        }
        return NULL;
    }
    
    void* new_ptr;
    if (ptr == NULL) {
        /* New allocation */
        new_ptr = malloc(nsize);
        if (new_ptr) {
            alloc_stats.total_allocated += nsize;
            alloc_stats.current_usage += nsize;
            alloc_stats.allocation_count++;
        }
    } else {
        /* Reallocation */
        new_ptr = realloc(ptr, nsize);
        if (new_ptr) {
            if (nsize > osize) {
                alloc_stats.total_allocated += (nsize - osize);
                alloc_stats.current_usage += (nsize - osize);
            } else {
                alloc_stats.total_freed += (osize - nsize);
                alloc_stats.current_usage -= (osize - nsize);
            }
        }
    }
    
    /* Track peak */
    if (alloc_stats.current_usage > alloc_stats.peak_usage) {
        alloc_stats.peak_usage = alloc_stats.current_usage;
    }
    
    return new_ptr;
}

/* Create Lua state with custom allocator */
lua_State* create_tracked_lua_state(void) {
    return lua_newstate(tracked_alloc, NULL);
}

/* Get stats from Lua */
static int l_get_alloc_stats(lua_State *L) {
    lua_newtable(L);
    
    lua_pushinteger(L, alloc_stats.total_allocated);
    lua_setfield(L, -2, "total_allocated");
    
    lua_pushinteger(L, alloc_stats.total_freed);
    lua_setfield(L, -2, "total_freed");
    
    lua_pushinteger(L, alloc_stats.current_usage);
    lua_setfield(L, -2, "current_usage");
    
    lua_pushinteger(L, alloc_stats.peak_usage);
    lua_setfield(L, -2, "peak_usage");
    
    lua_pushinteger(L, alloc_stats.allocation_count);
    lua_setfield(L, -2, "allocation_count");
    
    lua_pushinteger(L, alloc_stats.free_count);
    lua_setfield(L, -2, "free_count");
    
    return 1;
}
```

---

## 91.10 Memory-Efficient Data Patterns

```lua
-- Pattern 1: String interning
local StringInterner = {}
StringInterner.__index = StringInterner

function StringInterner.new()
    return setmetatable({
        _pool = {},
        _count = 0
    }, StringInterner)
end

function StringInterner:intern(s)
    local existing = self._pool[s]
    if existing then
        return existing
    end
    self._pool[s] = s
    self._count = self._count + 1
    return s
end

function StringInterner:size()
    return self._count
end

-- ใช้งาน: ลด duplication ของ string
local interner = StringInterner.new()

-- Simulate reading 10000 records where many strings repeat
local records = {}
local cities = {"Bangkok", "Chiang Mai", "Phuket", "Pattaya", "Ayutthaya"}
for i = 1, 10000 do
    records[i] = {
        id = i,
        -- ใช้ intern เพื่อ share string objects
        city = interner:intern(cities[math.random(#cities)]),
        name = "User_" .. i  -- unique strings ไม่ต้อง intern
    }
end

print("Interned strings:", interner:size())  -- แค่ 5 unique cities

-- Pattern 2: Compact number storage
-- แทนที่ {x=1.5, y=2.3} ใช้ flat array
local function create_point_array(n)
    return {}  -- [x1, y1, x2, y2, ...]
end

local function set_point(arr, i, x, y)
    arr[i * 2 - 1] = x
    arr[i * 2] = y
end

local function get_point(arr, i)
    return arr[i * 2 - 1], arr[i * 2]
end

-- Benchmark: table of tables vs flat array
collectgarbage("collect")
local mem_before = collectgarbage("count")

-- Method 1: Table of tables
local point_tables = {}
for i = 1, 10000 do
    point_tables[i] = { x = i * 0.1, y = i * 0.2 }
end

collectgarbage("collect")
local mem_tables = collectgarbage("count") - mem_before

-- Method 2: Flat array
collectgarbage("collect")
mem_before = collectgarbage("count")

local point_flat = create_point_array(10000)
for i = 1, 10000 do
    set_point(point_flat, i, i * 0.1, i * 0.2)
end

collectgarbage("collect")
local mem_flat = collectgarbage("count") - mem_before

print(string.format("Table of tables: %.2f KB", mem_tables))
print(string.format("Flat array:       %.2f KB", mem_flat))
print(string.format("Savings: %.1f%%", (1 - mem_flat/mem_tables) * 100))
```

---

## 91.11 GC-Aware Data Structures

```lua
-- LRU Cache ที่ GC-friendly
local LRUCache = {}
LRUCache.__index = LRUCache

function LRUCache.new(capacity)
    local cache = setmetatable({
        capacity = capacity,
        size = 0,
        head = nil,  -- most recent
        tail = nil,  -- least recent
        map = {}     -- key -> node
    }, LRUCache)
    return cache
end

function LRUCache:_new_node(key, value)
    return { key=key, value=value, prev=nil, next=nil }
end

function LRUCache:_remove(node)
    if node.prev then node.prev.next = node.next
    else self.head = node.next end
    
    if node.next then node.next.prev = node.prev
    else self.tail = node.prev end
    
    node.prev, node.next = nil, nil
    self.size = self.size - 1
end

function LRUCache:_push_front(node)
    node.prev = nil
    node.next = self.head
    if self.head then self.head.prev = node end
    self.head = node
    if not self.tail then self.tail = node end
    self.size = self.size + 1
end

function LRUCache:get(key)
    local node = self.map[key]
    if not node then return nil end
    
    -- Move to front (most recently used)
    self:_remove(node)
    self:_push_front(node)
    return node.value
end

function LRUCache:put(key, value)
    local existing = self.map[key]
    if existing then
        existing.value = value
        self:_remove(existing)
        self:_push_front(existing)
        return
    end
    
    -- Evict LRU if at capacity
    if self.size >= self.capacity then
        local lru = self.tail
        self.map[lru.key] = nil
        self:_remove(lru)
    end
    
    local node = self:_new_node(key, value)
    self.map[key] = node
    self:_push_front(node)
end

function LRUCache:info()
    local keys = {}
    local node = self.head
    while node do
        table.insert(keys, node.key)
        node = node.next
    end
    return {size=self.size, order=keys}
end

-- Test LRU Cache
local cache = LRUCache.new(3)
cache:put("a", 1)
cache:put("b", 2)
cache:put("c", 3)

print("After a,b,c:", table.concat(cache:info().order, ","))
-- a,b,c -> head=c(most recent), tail=a(least recent)

cache:get("a")  -- access a, moves to front
print("After get(a):", table.concat(cache:info().order, ","))
-- head=a, then c, b

cache:put("d", 4)  -- evicts b (LRU)
print("After put(d):", table.concat(cache:info().order, ","))
print("b evicted:", cache:get("b") == nil)
```

---

## 91.12 Profiling Memory Allocations

```lua
-- Allocation profiler
local AllocProfiler = {}
AllocProfiler.__index = AllocProfiler

function AllocProfiler.new()
    return setmetatable({
        enabled = false,
        sites = {},  -- callsite -> {count, bytes}
        original_allocator = nil
    }, AllocProfiler)
end

function AllocProfiler:enable()
    self.enabled = true
    print("Allocation profiler enabled")
    print("Note: Use debug.sethook for accurate profiling")
end

-- Simplified allocation tracking via GC monitoring
function AllocProfiler:measure(label, fn)
    collectgarbage("collect")
    local mem_before = collectgarbage("count")
    local time_before = os.clock()
    
    local result = { fn() }
    
    local time_after = os.clock()
    local mem_after_alloc = collectgarbage("count")
    collectgarbage("collect")
    local mem_after_gc = collectgarbage("count")
    
    local site = self.sites[label] or { runs=0, total_alloc=0, total_time=0 }
    site.runs = site.runs + 1
    site.total_alloc = site.total_alloc + (mem_after_alloc - mem_before)
    site.total_time = site.total_time + (time_after - time_before)
    site.retained = mem_after_gc - mem_before
    self.sites[label] = site
    
    return table.unpack(result)
end

function AllocProfiler:report()
    print("\n=== Allocation Profile ===")
    print(string.format("%-30s %8s %12s %12s %10s", 
        "Label", "Runs", "Total KB", "Avg KB", "Time(ms)"))
    print(string.rep("-", 80))
    
    local sorted = {}
    for label, site in pairs(self.sites) do
        table.insert(sorted, { label=label, site=site })
    end
    table.sort(sorted, function(a, b)
        return a.site.total_alloc > b.site.total_alloc
    end)
    
    for _, entry in ipairs(sorted) do
        local s = entry.site
        print(string.format("%-30s %8d %12.2f %12.2f %10.1f",
            entry.label,
            s.runs,
            s.total_alloc,
            s.total_alloc / s.runs,
            s.total_time * 1000 / s.runs))
    end
end

-- ทดสอบ profiler
local profiler = AllocProfiler.new()
profiler:enable()

-- Profile different operations
for i = 1, 5 do
    profiler:measure("string concat (..)", function()
        local s = ""
        for j = 1, 1000 do
            s = s .. tostring(j)
        end
        return s
    end)
    
    profiler:measure("table.concat", function()
        local t = {}
        for j = 1, 1000 do
            t[j] = tostring(j)
        end
        return table.concat(t)
    end)
    
    profiler:measure("table creation", function()
        local result = {}
        for j = 1, 1000 do
            result[j] = { x=j, y=j*2 }
        end
        return result
    end)
end

profiler:report()
```

---

## แบบฝึกหัด

1. **Object Pool สำหรับ Request Handler**: สร้าง pool สำหรับ HTTP request objects
2. **Memory Budget System**: สร้างระบบที่จำกัด memory usage ต่อ component
3. **GC Tuning**: ทดสอบ GC settings ต่างๆ และวัดผลบน workload ของคุณ
4. **Leak Hunter**: สร้าง tool ที่ช่วยหา memory leak ในโปรแกรมของคุณ
5. **Compact Data**: แปลง data structure ที่มีอยู่ให้ใช้ memory น้อยลง 50%

---

*ต่อไป: [Part 92 - Type System และ Static Analysis](part-92.md)*
