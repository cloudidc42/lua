# บทที่ 100: Capstone - สรุปและเส้นทางอาชีพ

## บทนำ

ยินดีด้วย! คุณเรียนจบหลักสูตร **Lua Programming: Basic to World-Class** แล้ว บทนี้สรุปทุกสิ่งที่เรียนมาและแนะนำเส้นทางอาชีพต่อไป

---

## 100.1 สิ่งที่คุณเรียนไปแล้ว

```
Level 1: Basic (Part 1-25)
─────────────────────────────────────────────────────────────────
Part 01  Introduction & Installation       ✓
Part 02  Variables & Data Types            ✓
Part 03  Operators & Expressions           ✓
Part 04  Strings                           ✓
Part 05  Control Flow                      ✓
Part 06  Functions                         ✓
Part 07  Tables                            ✓
Part 08  Math Library                      ✓
Part 09  I/O Operations                    ✓
Part 10  File System & OS Library          ✓
Part 11  Table Library                     ✓
Part 12  Pattern Matching                  ✓
Part 13  Error Handling                    ✓
Part 14  Multiple Returns & Varargs        ✓
Part 15  Scope & Variables                 ✓
Part 16  Closures                          ✓
Part 17  Metatables                        ✓
Part 18  Metamethods                       ✓
Part 19  Modules & Packages                ✓
Part 20  Debugging                         ✓
Part 21  OOP Basics                        ✓
Part 22  Inheritance                       ✓
Part 23  Multiple Inheritance & Mixins     ✓
Part 24  Coroutines                        ✓
Part 25  Iterators & Generators            ✓

Level 2: Intermediate (Part 26-50)
─────────────────────────────────────────────────────────────────
Part 26  Functional Programming            ✓
Part 27  Data Structures                   ✓
Part 28  Algorithms                        ✓
Part 29  Advanced String Processing        ✓
Part 30  JSON Processing                   ✓
Part 31  LuaRocks & Package Management     ✓
Part 32  Testing with Busted               ✓
Part 33  Testing with LuaUnit              ✓
Part 34  Performance Optimization          ✓
Part 35  Memory Management & GC            ✓
Part 36  Bit Operations                    ✓
Part 37  Design Patterns - Creational      ✓
Part 38  Design Patterns - Structural/Behavioral ✓
Part 39  Event System & Pub/Sub            ✓
Part 40  Configuration Management         ✓
Part 41-50 (Web, Database, Networking...) ✓

Level 3: Professional (Part 51-75)
─────────────────────────────────────────────────────────────────
Part 51  OpenResty - Nginx + Lua           ✓
Part 52  Lapis Web Framework               ✓
Part 53  REST API Development              ✓
Part 54  WebSocket Real-time               ✓
Part 55  Middleware Architecture           ✓
Part 56-75 (Advanced topics...)            ✓

Level 4: World-Class (Part 76-100)
─────────────────────────────────────────────────────────────────
Part 76-90 (LuaJIT, FFI, C API, LÖVE...)  ✓
Part 91  Custom Memory Allocator           ✓
Part 92  Type System & Static Analysis     ✓
Part 93  Language Server Protocol          ✓
Part 94  Advanced Profiling                ✓
Part 95  Code Generation                   ✓
Part 96  Contributing to Open Source       ✓
Part 97  Real Project: Chat Application    ✓
Part 98  Real Project: Game Server         ✓
Part 99  Real Project: API Gateway         ✓
Part 100 Capstone & Career Path            ✓ (You are here!)
```

---

## 100.2 Capstone Project - Build Your Own System

เพื่อพิสูจน์ว่าคุณเชี่ยวชาญ Lua แล้ว ให้สร้างระบบ production-grade ใดก็ได้จากตัวเลือกต่อไปนี้:

### Option A: Microservices Platform

```lua
-- ระบบ microservices ด้วย OpenResty ที่รองรับ:
-- 1. Service registry & discovery
-- 2. Load balancing
-- 3. Health monitoring
-- 4. Distributed tracing
-- 5. Config management

-- Service Registry
local ServiceRegistry = {}
ServiceRegistry.__index = ServiceRegistry

function ServiceRegistry.new()
    return setmetatable({
        services = {},
        health_checks = {},
    }, ServiceRegistry)
end

function ServiceRegistry:register(name, host, port, metadata)
    if not self.services[name] then
        self.services[name] = {}
    end
    
    local instance = {
        id = name .. "_" .. host .. "_" .. port,
        host = host,
        port = port,
        metadata = metadata or {},
        registered_at = os.time(),
        last_heartbeat = os.time(),
        healthy = true,
        weight = metadata and metadata.weight or 100,
    }
    
    self.services[name][instance.id] = instance
    return instance.id
end

function ServiceRegistry:deregister(name, instance_id)
    if self.services[name] then
        self.services[name][instance_id] = nil
    end
end

function ServiceRegistry:heartbeat(name, instance_id)
    local service = self.services[name]
    if service and service[instance_id] then
        service[instance_id].last_heartbeat = os.time()
        service[instance_id].healthy = true
    end
end

function ServiceRegistry:get_healthy(name)
    local service = self.services[name]
    if not service then return nil end
    
    local healthy = {}
    local now = os.time()
    
    for _, instance in pairs(service) do
        -- Deregister if no heartbeat for 30 seconds
        if now - instance.last_heartbeat > 30 then
            instance.healthy = false
        end
        
        if instance.healthy then
            table.insert(healthy, instance)
        end
    end
    
    return healthy
end

-- Weighted round-robin load balancer
function ServiceRegistry:get_instance(name)
    local healthy = self:get_healthy(name)
    if not healthy or #healthy == 0 then
        return nil, "No healthy instances for: " .. name
    end
    
    -- Build weight table
    local total_weight = 0
    for _, inst in ipairs(healthy) do
        total_weight = total_weight + inst.weight
    end
    
    local rand = math.random(1, total_weight)
    local cumulative = 0
    for _, inst in ipairs(healthy) do
        cumulative = cumulative + inst.weight
        if rand <= cumulative then
            return inst
        end
    end
    
    return healthy[1]
end

-- Test
local registry = ServiceRegistry.new()
registry:register("user-service", "10.0.0.1", 8080, { weight = 100 })
registry:register("user-service", "10.0.0.2", 8080, { weight = 50 })

local inst = registry:get_instance("user-service")
print("Selected:", inst.host .. ":" .. inst.port)
```

### Option B: Game Engine

```lua
-- 2D game engine ด้วย LÖVE2D ที่รองรับ:
-- ECS, Physics, Animation, Collision, Input, Audio

local GameEngine = {}

-- Component definitions
local COMPONENTS = {
    Transform = { x = 0, y = 0, rotation = 0, scale_x = 1, scale_y = 1 },
    Velocity = { vx = 0, vy = 0, friction = 1.0 },
    Sprite = { image = nil, quad = nil, origin_x = 0, origin_y = 0 },
    Collider = { shape = "rect", w = 32, h = 32, layer = 1, mask = 0xFF },
    Health = { current = 100, max = 100 },
    Player = { speed = 200, jump_force = -400 },
    Enemy = { ai_state = "patrol", patrol_speed = 60, detection_range = 200 },
    Animator = { current = nil, animations = {}, timer = 0 },
}

-- ECS World
local World = {}
World.__index = World

function World.new()
    return setmetatable({
        entities = {},
        systems = {},
        next_id = 1,
        -- Caches for system queries
        _query_cache = {},
    }, World)
end

function World:create_entity(...)
    local id = self.next_id
    self.next_id = self.next_id + 1
    
    local entity = { _id = id, _components = {} }
    self.entities[id] = entity
    
    -- Add components from arguments
    for _, comp_type in ipairs({...}) do
        self:add_component(id, comp_type)
    end
    
    return id
end

function World:add_component(entity_id, comp_type, values)
    local entity = self.entities[entity_id]
    if not entity then return end
    
    local template = COMPONENTS[comp_type]
    if not template then return end
    
    -- Deep copy template
    local comp = {}
    for k, v in pairs(template) do comp[k] = v end
    
    -- Apply custom values
    if values then
        for k, v in pairs(values) do comp[k] = v end
    end
    
    entity._components[comp_type] = comp
    self._query_cache = {}  -- Invalidate cache
    
    return comp
end

function World:get_component(entity_id, comp_type)
    local entity = self.entities[entity_id]
    return entity and entity._components[comp_type]
end

function World:query(...)
    local required = {...}
    local key = table.concat(required, ",")
    
    if self._query_cache[key] then
        return self._query_cache[key]
    end
    
    local result = {}
    for id, entity in pairs(self.entities) do
        local has_all = true
        for _, comp_type in ipairs(required) do
            if not entity._components[comp_type] then
                has_all = false
                break
            end
        end
        if has_all then
            table.insert(result, id)
        end
    end
    
    self._query_cache[key] = result
    return result
end

-- Physics system
function World:physics_system(dt)
    for _, id in ipairs(self:query("Transform", "Velocity")) do
        local tf = self:get_component(id, "Transform")
        local vel = self:get_component(id, "Velocity")
        
        tf.x = tf.x + vel.vx * dt
        tf.y = tf.y + vel.vy * dt
        
        -- Apply friction
        vel.vx = vel.vx * vel.friction
        vel.vy = vel.vy * vel.friction
    end
end

-- Gravity system
function World:gravity_system(dt)
    local GRAVITY = 980
    for _, id in ipairs(self:query("Velocity")) do
        local vel = self:get_component(id, "Velocity")
        vel.vy = vel.vy + GRAVITY * dt
    end
end

-- Demo
local world = World.new()

local player = world:create_entity("Transform", "Velocity", "Health", "Player")
local tf = world:get_component(player, "Transform")
tf.x, tf.y = 100, 100

local vel = world:get_component(player, "Velocity")
vel.friction = 0.85

print("Player entity:", player)
print("Player position:", tf.x, tf.y)
```

### Option C: Distributed Task Queue

```lua
-- Task queue system ด้วย Redis backend
-- รองรับ: priority, retry, delayed tasks, workers

local TaskQueue = {}
TaskQueue.__index = TaskQueue

local cjson = require("cjson")

function TaskQueue.new(redis_client, queue_name)
    return setmetatable({
        redis = redis_client,
        queue = queue_name,
        workers = {},
        handlers = {},
        running = false,
    }, TaskQueue)
end

-- Register a task handler
function TaskQueue:register(task_type, handler, opts)
    self.handlers[task_type] = {
        fn = handler,
        max_retries = opts and opts.max_retries or 3,
        timeout = opts and opts.timeout or 30,
        concurrency = opts and opts.concurrency or 1,
    }
end

-- Enqueue a task
function TaskQueue:enqueue(task_type, payload, opts)
    opts = opts or {}
    
    local task = {
        id = generate_id(),
        type = task_type,
        payload = payload,
        status = "pending",
        priority = opts.priority or 5,  -- 1=highest, 10=lowest
        retry_count = 0,
        max_retries = opts.max_retries or 3,
        created_at = os.time(),
        scheduled_at = opts.delay and (os.time() + opts.delay) or os.time(),
        metadata = opts.metadata or {},
    }
    
    local serialized = cjson.encode(task)
    
    -- Use sorted set for priority queue (score = priority * time)
    local score = task.priority * 1e10 + task.scheduled_at
    self.redis:zadd(self.queue .. ":pending", score, serialized)
    
    return task.id
end

-- Process next task
function TaskQueue:process_next()
    -- Get highest priority task that's due
    local now = os.time()
    local max_score = 10 * 1e10 + now  -- All priorities, due now
    
    local results = self.redis:zrangebyscore(
        self.queue .. ":pending", "-inf", max_score, 
        "LIMIT", 0, 1
    )
    
    if not results or #results == 0 then
        return false  -- No tasks
    end
    
    local serialized = results[1]
    local task = cjson.decode(serialized)
    
    -- Atomically move to processing
    local pipe = self.redis:pipeline()
    pipe:zrem(self.queue .. ":pending", serialized)
    pipe:hset(self.queue .. ":processing", task.id, serialized)
    pipe:exec()
    
    -- Execute handler
    local handler = self.handlers[task.type]
    if not handler then
        self:fail_task(task, "No handler for type: " .. task.type)
        return true
    end
    
    local ok, err = pcall(handler.fn, task.payload, task)
    
    if ok then
        self:complete_task(task)
    else
        task.retry_count = task.retry_count + 1
        task.last_error = tostring(err)
        
        if task.retry_count <= task.max_retries then
            -- Exponential backoff retry
            local delay = math.pow(2, task.retry_count) * 10
            task.scheduled_at = os.time() + delay
            
            local new_score = task.priority * 1e10 + task.scheduled_at
            self.redis:zadd(self.queue .. ":pending", new_score, cjson.encode(task))
            self.redis:hdel(self.queue .. ":processing", task.id)
        else
            self:fail_task(task, err)
        end
    end
    
    return true
end

function TaskQueue:complete_task(task)
    task.status = "completed"
    task.completed_at = os.time()
    
    self.redis:hdel(self.queue .. ":processing", task.id)
    self.redis:hset(self.queue .. ":completed", task.id, cjson.encode(task))
    -- Auto-expire completed tasks after 24 hours
    -- In production: use Redis EXPIRE on individual keys
end

function TaskQueue:fail_task(task, err)
    task.status = "failed"
    task.failed_at = os.time()
    task.error = tostring(err)
    
    self.redis:hdel(self.queue .. ":processing", task.id)
    self.redis:hset(self.queue .. ":failed", task.id, cjson.encode(task))
end

function generate_id()
    return string.format("%x%x", os.time(), math.random(0xFFFF))
end
```

---

## 100.3 เส้นทางอาชีพ (Career Paths)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Lua Career Paths                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Web Backend Developer                                       │
│     ├── OpenResty/Nginx developer                               │
│     ├── API Gateway specialist                                  │
│     └── High-performance web services                          │
│                                                                 │
│  2. Game Developer                                              │
│     ├── LÖVE2D indie game developer                            │
│     ├── Roblox Studio developer                                │
│     ├── WoW/GTA addon developer                                │
│     └── Game scripting engineer                                │
│                                                                 │
│  3. DevOps/Infrastructure                                       │
│     ├── Redis scripting expert                                  │
│     ├── Nginx/OpenResty infrastructure                         │
│     └── Kubernetes operator (with Lua webhooks)                │
│                                                                 │
│  4. Embedded Systems                                            │
│     ├── IoT firmware developer (NodeMCU)                       │
│     ├── Embedded scripting engine developer                    │
│     └── Automotive systems (ECU scripting)                     │
│                                                                 │
│  5. Editor/IDE Developer                                        │
│     ├── Neovim plugin developer                                │
│     ├── VS Code extension (via Lua LSP)                        │
│     └── IDE tooling developer                                  │
│                                                                 │
│  6. Language/Compiler Developer                                 │
│     ├── Teal language contributor                              │
│     ├── LuaJIT core developer                                  │
│     └── Programming language researcher                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 100.4 ทักษะที่ควรพัฒนาต่อ

```lua
-- Roadmap สำหรับแต่ละสาขา

local roadmap = {
    
    ["Web Backend"] = {
        {skill = "OpenResty advanced", priority = 1},
        {skill = "Redis data structures", priority = 1},
        {skill = "PostgreSQL/MySQL via lua-resty-mysql", priority = 2},
        {skill = "Kubernetes/Docker", priority = 2},
        {skill = "Grafana/Prometheus monitoring", priority = 3},
        {skill = "gRPC via lua-resty-grpc", priority = 3},
    },
    
    ["Game Dev"] = {
        {skill = "LÖVE2D framework mastery", priority = 1},
        {skill = "2D physics (box2d bindings)", priority = 1},
        {skill = "Asset pipeline (texture packing)", priority = 2},
        {skill = "Shader programming (GLSL)", priority = 2},
        {skill = "Networking (ENet/socket)", priority = 3},
        {skill = "Audio (lua-audio)", priority = 3},
    },
    
    ["Embedded/IoT"] = {
        {skill = "NodeMCU/ESP8266 Lua", priority = 1},
        {skill = "MQTT protocol", priority = 1},
        {skill = "Hardware I/O (GPIO, I2C, SPI)", priority = 2},
        {skill = "Power management", priority = 2},
        {skill = "OTA firmware updates", priority = 3},
    },
    
    ["Neovim Plugin Dev"] = {
        {skill = "Neovim API (vim.api, vim.fn)", priority = 1},
        {skill = "Treesitter parsing", priority = 1},
        {skill = "LSP client integration", priority = 2},
        {skill = "Telescope extensions", priority = 2},
        {skill = "Floating windows/UI", priority = 3},
    },
}

-- Print roadmap for chosen path
local path = "Web Backend"
print("=== Roadmap: " .. path .. " ===")
for _, item in ipairs(roadmap[path]) do
    local priority_str = ({"★★★", "★★☆", "★☆☆"})[item.priority]
    print(string.format("  %s %s", priority_str, item.skill))
end
```

---

## 100.5 แหล่งเรียนรู้เพิ่มเติม

```
Official Resources:
─────────────────────────────────────────────────────────────────
• lua.org              - Official Lua website
• luarocks.org         - Package repository
• lua-users.org        - Community wiki
• lua-l@lua.org        - Official mailing list

Books:
─────────────────────────────────────────────────────────────────
• "Programming in Lua" by Roberto Ierusalimschy (PIL)
  → lua.org/pil/  (free online, 4th edition for Lua 5.3+)
• "Lua 5.3 Reference Manual"
  → lua.org/manual/5.3/
• "Lua Game Development Cookbook"

Online Courses & Tutorials:
─────────────────────────────────────────────────────────────────
• learn.love2d.org      - LÖVE2D game dev
• leafo.net/lapis       - Lapis web framework
• openresty.org/docs/   - OpenResty documentation
• neovim.io/doc/user    - Neovim Lua API

Communities:
─────────────────────────────────────────────────────────────────
• r/lua                 - Reddit community
• Lua Discord           - Discord server
• lua-l mailing list    - Official mailing list
• GitHub: lunarmodules  - Quality Lua libraries

Projects to Study:
─────────────────────────────────────────────────────────────────
• github.com/lunarmodules/Penlight  - Utility library
• github.com/kikito/inspect.lua     - Inspection library
• github.com/rxi/json.lua           - JSON library
• github.com/keplerproject/luafilesystem
• github.com/LuaLS/lua-language-server
```

---

## 100.6 Final Project: Build Something Real

```lua
-- โปรเจกต์จบหลักสูตร - เลือก 1 อย่าง:

local final_projects = {
    {
        name = "Personal Blog Engine",
        description = "Static site generator ด้วย Lua",
        features = {
            "Markdown to HTML converter",
            "Template system",
            "RSS feed generation",
            "Syntax highlighting",
            "Search index",
        },
        skills_used = {
            "File I/O", "String patterns", "Templates",
            "JSON", "Coroutines"
        }
    },
    {
        name = "Real-time Dashboard",
        description = "Monitoring dashboard ด้วย OpenResty + WebSocket",
        features = {
            "Live metrics via WebSocket",
            "Server-Sent Events",
            "Redis time series",
            "Chart rendering",
            "Alert system",
        },
        skills_used = {
            "OpenResty", "WebSocket", "Redis",
            "JWT Auth", "Rate limiting"
        }
    },
    {
        name = "Lua Interpreter",
        description = "Interpreter สำหรับ subset ของ Lua ใน Lua",
        features = {
            "Lexer/Tokenizer",
            "Recursive descent parser",
            "AST evaluation",
            "Variable scoping",
            "Basic functions",
        },
        skills_used = {
            "String patterns", "Tables", "Closures",
            "Coroutines", "Metatables"
        }
    },
    {
        name = "Multiplayer Puzzle Game",
        description = "Turn-based puzzle game ด้วย LÖVE2D + network",
        features = {
            "Game state machine",
            "Networking with LuaSocket",
            "Puzzle solver AI",
            "Score system",
            "Leaderboard",
        },
        skills_used = {
            "LÖVE2D", "ECS", "Coroutines",
            "Networking", "JSON"
        }
    },
}

-- Display project options
for i, project in ipairs(final_projects) do
    print(string.format("\n%d. %s", i, project.name))
    print("   " .. project.description)
    print("   Features:")
    for _, f in ipairs(project.features) do
        print("   • " .. f)
    end
end
```

---

## 100.7 Code Review Checklist

```lua
-- ใช้ checklist นี้ก่อน submit code

local code_review_checklist = {
    
    correctness = {
        "ทดสอบ edge cases ทั้งหมดแล้ว",
        "ไม่มี off-by-one errors",
        "จัดการ nil values อย่างถูกต้อง",
        "ไม่มี infinite loops",
        "Resource cleanup ครบ (file handles, sockets)",
    },
    
    performance = {
        "ใช้ local variables ในส่วนที่ critical",
        "ไม่สร้าง table ใน hot path โดยไม่จำเป็น",
        "ใช้ table.concat แทน string concatenation ใน loop",
        "Memoize ฟังก์ชันที่ expensive",
        "ไม่ block coroutine หรือ main thread",
    },
    
    readability = {
        "ตั้งชื่อตัวแปรและฟังก์ชันสื่อความหมาย",
        "ฟังก์ชันไม่ยาวเกิน 50 บรรทัด",
        "Comment อธิบาย WHY ไม่ใช่ WHAT",
        "ใช้ early return แทน nested if",
    },
    
    security = {
        "Validate user input ทุกครั้ง",
        "ไม่ใช้ load() กับ input จาก user โดยตรง",
        "Sanitize SQL queries (ใช้ prepared statements)",
        "ไม่ log sensitive data (passwords, tokens)",
        "Rate limit API endpoints",
    },
    
    testing = {
        "Unit tests ครอบคลุม happy path",
        "Unit tests ครอบคลุม error cases",
        "Integration tests สำหรับ external services",
        "Performance benchmark สำหรับ critical paths",
    },
}

local function print_checklist(category, items)
    print("\n[" .. category:upper() .. "]")
    for i, item in ipairs(items) do
        print(string.format("  %d. [ ] %s", i, item))
    end
end

for category, items in pairs(code_review_checklist) do
    print_checklist(category, items)
end
```

---

## 100.8 The Lua Programmer's Manifesto

```
สิ่งที่นักพัฒนา Lua ระดับโลกยึดถือ:

1. SIMPLICITY OVER CLEVERNESS
   โค้ดที่อ่านง่ายดีกว่าโค้ดที่ "ฉลาด"
   Lua มีแค่ metatables และ closures - ใช้ให้ถูกที่

2. MEASURE BEFORE OPTIMIZE
   วัดก่อนเสมอ อย่า assume ว่าอะไรช้า
   debug.sethook และ os.clock() เป็นเพื่อนของคุณ

3. TABLES ARE EVERYTHING
   Table คือ array, dict, object, module, namespace
   เรียนรู้ table อย่างลึกซึ้งคือกุญแจสู่ Lua mastery

4. COROUTINES CHANGE EVERYTHING
   Async ไม่ต้องการ callback hell
   Coroutine คือ cooperative multitasking ที่สวยงาม

5. SMALL IS BEAUTIFUL
   Lua เล็กด้วยเหตุผล - นำทัศนคตินี้ไปกับโค้ดของคุณ
   Standard library น้อย แต่ composable อย่างยิ่ง

6. KNOW THE HOST
   Lua ถูกออกแบบมาเพื่อ embed
   รู้จัก environment ที่คุณทำงาน (OpenResty, LÖVE, Neovim)

7. CONTRIBUTE BACK
   Open source community ต้องการคุณ
   Bug reports, documentation, libraries - ทุกอย่างมีคุณค่า

8. NEVER STOP LEARNING
   ภาษาโปรแกรมมิ่งเป็นเครื่องมือ ไม่ใช่เป้าหมาย
   Lua เปิดประตูสู่ systems programming, game dev, web dev
```

---

## 100.9 Quick Reference Card

```lua
-- Lua 5.4 Quick Reference

-- TYPES
-- nil, boolean, number (integer/float), string, table, function, userdata, thread

-- VARIABLES
local x = 10              -- local (preferred)
y = 20                    -- global (avoid)
local a, b = 1, 2         -- multiple assignment
local a <const> = 42      -- constant (Lua 5.4)

-- STRINGS
local s = "hello"
local len = #s             -- length
local upper = s:upper()
local sub = s:sub(2, 4)   -- "ell"
local found = s:find("ll") -- position
local rep = s:format("%s world") -- formatting
local cat = s .. " world" -- concatenation

-- TABLES
local t = {1, 2, 3}       -- array (1-indexed!)
local d = {a=1, b=2}      -- dictionary
table.insert(t, 4)
table.remove(t, 1)
table.sort(t)
local concat = table.concat(t, ", ")

-- CONTROL FLOW  
if cond then ... elseif c2 then ... else ... end
while cond do ... end
repeat ... until cond
for i = 1, 10 do ... end          -- numeric for
for i, v in ipairs(t) do ... end  -- array iteration
for k, v in pairs(d) do ... end   -- table iteration

-- FUNCTIONS
local function f(a, b) return a + b end
local g = function(x) return x * 2 end
local h = function(...) return select('#', ...) end  -- varargs

-- OOP
local MyClass = {}
MyClass.__index = MyClass
function MyClass.new(x) return setmetatable({x=x}, MyClass) end
function MyClass:method() return self.x end
local obj = MyClass.new(42)
obj:method()

-- ERROR HANDLING
local ok, err = pcall(function()
    error("something went wrong")
end)
if not ok then print("Error:", err) end

-- COROUTINES
local co = coroutine.create(function(x)
    coroutine.yield(x + 1)
    return x + 2
end)
local ok, val = coroutine.resume(co, 10)  -- val = 11

-- METATABLES
local mt = {
    __index = function(t, k) return 0 end,
    __tostring = function(t) return "MyTable" end,
    __add = function(a, b) return ... end,
}
setmetatable(obj, mt)

-- I/O
local f = io.open("file.txt", "r")
local content = f:read("*a")
f:close()

-- PATTERNS
local date = "2024-01-15"
local y, m, d = date:match("(%d+)-(%d+)-(%d+)")
```

---

## 100.10 ข้อความสุดท้าย

```
ขอบคุณที่เรียนหลักสูตรนี้จนจบ!

คุณได้เรียนรู้ Lua จากพื้นฐานไปจนถึงระดับ world-class:
• Syntax พื้นฐาน และ data types ทั้ง 8 ชนิด
• Tables ซึ่งเป็นหัวใจของ Lua
• Metatables และ metamethods สำหรับ OOP
• Coroutines สำหรับ async programming
• Performance optimization ด้วย LuaJIT
• C API สำหรับ embedding และ extending
• Real-world projects: chat app, game server, API gateway
• Open source contribution practices

Lua เป็นภาษาที่เล็ก แต่ทรงพลัง
ด้วยความรู้ที่คุณมีตอนนี้ คุณสามารถ:
• สร้าง web services ที่รองรับ millions of requests/second
• พัฒนา games ที่ publish ได้จริง
• สร้าง tools ที่ช่วยเหลือ developer community
• Contribute กลับสู่ open source ecosystem

จงสร้างสิ่งที่ยิ่งใหญ่!
The Lua community is waiting for your contributions.

  "Simplicity is a great virtue but it requires hard work to 
   achieve it and education to appreciate it." 
   — Edsger W. Dijkstra

Good luck! สู้ๆ!
```

---

## แบบฝึกหัดสุดท้าย

1. **Complete one capstone project** จากตัวเลือกใน 100.2
2. **Publish a library** บน LuaRocks
3. **Contribute to open source** ส่ง PR ไปยัง Lua project ที่คุณใช้
4. **Teach someone** แบ่งปันความรู้ Lua ให้คนอื่น
5. **Build something nobody has built before** - นั่นคือเป้าหมายสูงสุด

---

*จบหลักสูตร Lua Programming: Basic to World-Class*

*ขอบคุณที่ร่วมเดินทางนี้ด้วยกัน!*
