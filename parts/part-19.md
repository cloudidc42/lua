# บทที่ 19: Modules และ Packages

## บทนำ

Module ใน Lua คือวิธีการจัดระเบียบโค้ดให้เป็นหน่วยที่ใช้ซ้ำได้ (reusable unit) Lua มีระบบ module ที่เรียบง่ายแต่ทรงพลัง โดยอาศัยฟังก์ชัน `require()` และ table เป็นหลัก

การใช้ modules ช่วยให้:
- แยก code ออกเป็น files ที่มีความรับผิดชอบชัดเจน
- ป้องกัน name collision ระหว่างส่วนต่างๆ ของโปรแกรม
- ใช้ code ซ้ำได้ใน projects อื่น
- ทดสอบแต่ละส่วนแยกกันได้

---

## 19.1 Module Pattern พื้นฐาน

### รูปแบบ Module ที่ถูกต้อง

```lua
-- ตัวอย่างที่ 1: Basic module pattern
-- ไฟล์: mymodule.lua

local M = {}  -- M คือ module table

-- Private variable (ไม่ export)
local secret = "this is private"
local count = 0

-- Private function
local function increment()
    count = count + 1
end

-- Public function
function M.greet(name)
    increment()
    return "Hello, " .. name .. "!"
end

function M.getCount()
    return count
end

function M.reset()
    count = 0
end

-- Constants
M.VERSION = "1.0.0"
M.MAX_COUNT = 100

return M  -- ต้อง return module table เสมอ
```

```lua
-- ตัวอย่างที่ 2: การใช้ module
-- ไฟล์: main.lua

-- สมมติว่า mymodule.lua อยู่ใน path
-- local mymod = require("mymodule")

-- สาธิตด้วย inline module definition:
local function loadModule()
    local M = {}
    local count = 0
    
    function M.greet(name)
        count = count + 1
        return "Hello, " .. name .. "! (call #" .. count .. ")"
    end
    
    function M.getCount() return count end
    M.VERSION = "1.0.0"
    
    return M
end

local mymod = loadModule()

print(mymod.greet("Alice"))   -- Hello, Alice! (call #1)
print(mymod.greet("Bob"))     -- Hello, Bob! (call #2)
print(mymod.getCount())       -- 2
print(mymod.VERSION)          -- 1.0.0
```

### Module ด้วย Closure

```lua
-- ตัวอย่างที่ 3: Module ที่ใช้ closure สำหรับ private state
local function createCounter(start, step)
    local value = start or 0
    local stepSize = step or 1
    
    return {
        increment = function()
            value = value + stepSize
        end,
        decrement = function()
            value = value - stepSize
        end,
        get = function()
            return value
        end,
        reset = function()
            value = start or 0
        end,
        setStep = function(newStep)
            stepSize = newStep
        end
    }
end

local c1 = createCounter(0, 1)
local c2 = createCounter(100, 5)

c1.increment()
c1.increment()
c1.increment()
print(c1.get())  -- 3

c2.increment()
print(c2.get())  -- 105

-- c1 และ c2 มี state แยกกัน
print(c1.get(), c2.get())  -- 3  105
```

---

## 19.2 require() Function

### วิธีการทำงานของ require()

```lua
-- ตัวอย่างที่ 4: สาธิตการทำงานของ require()
-- require() ทำงานดังนี้:
-- 1. ตรวจสอบ package.loaded[modname]
-- 2. ถ้าเจอ return cached version
-- 3. ถ้าไม่เจอ ค้นหาไฟล์ใน package.path
-- 4. โหลดและ execute ไฟล์
-- 5. เก็บผลลัพธ์ใน package.loaded[modname]
-- 6. Return ผลลัพธ์

-- สาธิต module caching
local function simulateRequire()
    local loaded = {}  -- คล้าย package.loaded
    
    local function myRequire(name, loader)
        if loaded[name] then
            print(string.format("require('%s'): returning cached", name))
            return loaded[name]
        end
        
        print(string.format("require('%s'): loading for first time", name))
        local module = loader()
        loaded[name] = module
        return module
    end
    
    -- Module definition
    local function mathUtils()
        print("Initializing mathUtils...")
        return {
            square = function(x) return x * x end,
            cube = function(x) return x * x * x end
        }
    end
    
    -- First require: loads module
    local m1 = myRequire("mathUtils", mathUtils)
    -- Second require: returns cached
    local m2 = myRequire("mathUtils", mathUtils)
    
    print(m1 == m2)  -- true (same table)
    print(m1.square(4))  -- 16
end

simulateRequire()
```

```lua
-- ตัวอย่างที่ 5: package.loaded - Module caching
-- ดู modules ที่โหลดอยู่
print("Loaded modules:")
for name, mod in pairs(package.loaded) do
    if type(mod) ~= "boolean" then
        print(string.format("  %s = %s", name, type(mod)))
    end
end

-- ตรวจสอบ module ที่โหลด
print("\nstring module loaded:", package.loaded["string"] ~= nil)
print("math module loaded:", package.loaded["math"] ~= nil)

-- Force reload โดย clear cache
-- package.loaded["mymodule"] = nil
-- require("mymodule")  -- จะ load ใหม่
```

---

## 19.3 package.path และ package.cpath

```lua
-- ตัวอย่างที่ 6: ดู package.path
print("package.path:")
for path in package.path:gmatch("[^;]+") do
    print("  " .. path)
end

print("\npackage.cpath:")
for path in package.cpath:gmatch("[^;]+") do
    print("  " .. path)
end
```

```lua
-- ตัวอย่างที่ 7: เพิ่ม path ใหม่
-- เพิ่ม custom path สำหรับ module search
local function addPath(newPath)
    package.path = newPath .. "/?.lua;" ..
                   newPath .. "/?/init.lua;" ..
                   package.path
end

-- สาธิต (ไม่ได้เพิ่มจริง แค่แสดงวิธี)
local function showPathAddition()
    local original = package.path
    local customPath = "/home/user/mylibrary"
    local newPath = customPath .. "/?.lua;" .. original
    print("Original path count:", select(2, original:gsub(";", ";")) + 1)
    print("New path would add:", customPath)
end

showPathAddition()
```

```lua
-- ตัวอย่างที่ 8: package.searchpath()
-- ค้นหาไฟล์ใน path

local function findModule(name)
    -- แปลง "." เป็น "/" ใน module name
    local filename = package.searchpath(name, package.path)
    if filename then
        print("Found: " .. filename)
    else
        print("Not found: " .. name)
    end
    return filename
end

-- ค้นหา built-in modules (จะหาไม่เจอ เพราะเป็น C modules)
findModule("string")
findModule("math")
-- ค้นหา Lua modules
findModule("nonexistent_module")
```

---

## 19.4 การเขียน Module ที่ดี

### Module พร้อม Error Handling

```lua
-- ตัวอย่างที่ 9: Module พร้อม error handling
local M = {}

-- Private
local _initialized = false
local _data = {}

local function checkInit()
    if not _initialized then
        error("Module not initialized. Call M.init() first.", 3)
    end
end

-- Public API
function M.init(config)
    if _initialized then
        return false, "Already initialized"
    end
    
    -- Validate config
    if type(config) ~= "table" then
        return false, "Config must be a table"
    end
    
    _data.name = config.name or "default"
    _data.version = config.version or "1.0"
    _initialized = true
    
    return true
end

function M.getName()
    checkInit()
    return _data.name
end

function M.getVersion()
    checkInit()
    return _data.version
end

function M.isInitialized()
    return _initialized
end

-- Test
local ok, err = M.init({name = "MyApp", version = "2.0"})
print(ok, err)            -- true  nil
print(M.getName())        -- MyApp
print(M.getVersion())     -- 2.0

local ok2, err2 = M.init({})  -- try to init again
print(ok2, err2)          -- false  Already initialized
```

### Module พร้อม Metatable

```lua
-- ตัวอย่างที่ 10: Module ที่ใช้ metatable สำหรับ callable
local Logger = {}
Logger.__index = Logger

-- Module-level default logger
local defaultLogger

-- Constructor
function Logger.new(name, level)
    local levels = {DEBUG=1, INFO=2, WARN=3, ERROR=4}
    return setmetatable({
        name = name,
        level = levels[level] or levels.INFO,
        levels = levels,
        output = io.stdout
    }, Logger)
end

function Logger:log(levelName, msg)
    local levelNum = self.levels[levelName] or 0
    if levelNum >= self.level then
        local timestamp = os.date("%H:%M:%S")
        self.output:write(string.format(
            "[%s] [%s] [%s] %s\n",
            timestamp, levelName, self.name, msg
        ))
    end
end

function Logger:debug(msg) self:log("DEBUG", msg) end
function Logger:info(msg)  self:log("INFO",  msg) end
function Logger:warn(msg)  self:log("WARN",  msg) end
function Logger:error(msg) self:log("ERROR", msg) end

-- Module-level functions using default logger
function Logger.setDefault(logger)
    defaultLogger = logger
end

function Logger.getDefault()
    if not defaultLogger then
        defaultLogger = Logger.new("root", "INFO")
    end
    return defaultLogger
end

-- Make module callable: Logger("name") instead of Logger.new("name")
setmetatable(Logger, {
    __call = function(_, name, level)
        return Logger.new(name, level)
    end
})

-- Test
local log = Logger("MyApp", "DEBUG")
log:debug("Application starting")
log:info("Loading configuration")
log:warn("Config file not found, using defaults")
log:error("Failed to connect to database")
```

---

## 19.5 Circular Requires

```lua
-- ตัวอย่างที่ 11: Circular dependency problem
-- ไฟล์ a.lua:
--   local b = require("b")
--   local M = {}
--   function M.hello() return "Hello from A, " .. b.world() end
--   return M
--
-- ไฟล์ b.lua:
--   local a = require("a")  -- CIRCULAR!
--   local M = {}
--   function M.world() return "World from B, " .. a.hello() end
--   return M

-- วิธีแก้: ใช้ lazy loading หรือ dependency injection

-- วิธีที่ 1: Lazy loading
local function createModuleA()
    local M = {}
    local _b = nil  -- lazy loaded
    
    local function getB()
        if not _b then
            -- _b = require("b")  -- load เมื่อใช้จริง
            _b = {world = function() return "World" end}  -- mock
        end
        return _b
    end
    
    function M.hello()
        return "Hello from A, " .. getB().world()
    end
    
    return M
end

-- วิธีที่ 2: เพิ่ม module ใน package.loaded ก่อน
-- ใน a.lua:
local function createModuleB_safe()
    local M = {}
    -- package.loaded["b"] = M  -- register before requiring a
    -- local a = require("a")    -- now a can use b (as empty table initially)
    
    function M.world()
        return "World"
    end
    
    return M
end

local a = createModuleA()
print(a.hello())  -- Hello from A, World
```

---

## 19.6 Submodules

```lua
-- ตัวอย่างที่ 12: Submodule structure
-- โครงสร้าง:
-- mylib/
--   init.lua        (main module)
--   utils.lua       (utilities submodule)
--   math.lua        (math submodule)
--   string.lua      (string submodule)

-- สาธิต submodule pattern
local mylib = {}

-- utils submodule
mylib.utils = {
    isEmpty = function(t)
        return next(t) == nil
    end,
    deepCopy = function(orig)
        local copy = {}
        for k, v in pairs(orig) do
            if type(v) == "table" then
                copy[k] = mylib.utils.deepCopy(v)
            else
                copy[k] = v
            end
        end
        return copy
    end,
    merge = function(t1, t2)
        local result = mylib.utils.deepCopy(t1)
        for k, v in pairs(t2) do
            result[k] = v
        end
        return result
    end
}

-- math submodule
mylib.math = {
    clamp = function(val, min, max)
        return math.max(min, math.min(max, val))
    end,
    lerp = function(a, b, t)
        return a + (b - a) * t
    end,
    sign = function(x)
        if x > 0 then return 1
        elseif x < 0 then return -1
        else return 0 end
    end,
    round = function(x, decimals)
        local mult = 10^(decimals or 0)
        return math.floor(x * mult + 0.5) / mult
    end
}

-- string submodule
mylib.string = {
    trim = function(s)
        return s:match("^%s*(.-)%s*$")
    end,
    split = function(s, sep)
        sep = sep or "%s"
        local parts = {}
        for part in s:gmatch("[^" .. sep .. "]+") do
            parts[#parts + 1] = part
        end
        return parts
    end,
    startsWith = function(s, prefix)
        return s:sub(1, #prefix) == prefix
    end,
    endsWith = function(s, suffix)
        return suffix == "" or s:sub(-#suffix) == suffix
    end,
    capitalize = function(s)
        return s:sub(1,1):upper() .. s:sub(2):lower()
    end
}

-- Test
print(mylib.math.clamp(15, 0, 10))           -- 10
print(mylib.math.lerp(0, 100, 0.3))          -- 30.0
print(mylib.math.round(3.14159, 2))          -- 3.14
print(mylib.string.trim("  hello  "))        -- "hello"
print(mylib.string.capitalize("wORLD"))      -- World

local parts = mylib.string.split("a,b,c,d", ",")
print(table.concat(parts, "|"))              -- a|b|c|d

local t1 = {a=1, b=2}
local t2 = {b=3, c=4}
local merged = mylib.utils.merge(t1, t2)
print(merged.a, merged.b, merged.c)          -- 1  3  4
```

### init.lua Convention

```lua
-- ตัวอย่างที่ 13: init.lua pattern
-- เมื่อ require("mypackage") Lua จะค้นหา:
-- 1. mypackage.lua
-- 2. mypackage/init.lua

-- สร้าง package structure ใน memory:
local function createPackage()
    -- นี่คือ init.lua ของ package
    local pkg = {}
    pkg._VERSION = "1.0.0"
    pkg._DESCRIPTION = "My Lua Package"
    
    -- Auto-load submodules
    local submodules = {"math", "string", "utils"}
    
    -- Lazy load submodules
    local loaded = {}
    setmetatable(pkg, {
        __index = function(t, k)
            -- ถ้าเป็น submodule ที่รู้จัก
            for _, name in ipairs(submodules) do
                if k == name then
                    if not loaded[k] then
                        -- loaded[k] = require("mypackage." .. k)
                        loaded[k] = {name = k, loaded = true}
                        print("Lazy loading submodule: " .. k)
                    end
                    return loaded[k]
                end
            end
            return nil
        end
    })
    
    function pkg.version()
        return pkg._VERSION
    end
    
    return pkg
end

local pkg = createPackage()
print(pkg.version())     -- 1.0.0
print(pkg.math.name)     -- Lazy loading submodule: math \n math
print(pkg.string.name)   -- Lazy loading submodule: string \n string
print(pkg.math.name)     -- math (no lazy loading, already cached)
```

---

## 19.7 Lazy Loading Pattern

```lua
-- ตัวอย่างที่ 14: Lazy loading module
local function lazyRequire(name)
    -- Return proxy ที่จะ load module เมื่อถูกใช้จริง
    local _module = nil
    return setmetatable({}, {
        __index = function(_, k)
            if not _module then
                print("Lazy loading: " .. name)
                -- _module = require(name)
                _module = {name = name, loaded = true, value = 42}
            end
            return _module[k]
        end,
        __call = function(_, ...)
            if not _module then
                print("Lazy loading (call): " .. name)
                _module = {name = name}
            end
            if type(_module) == "function" then
                return _module(...)
            end
        end
    })
end

local heavyModule = lazyRequire("heavy_computation")
print("Module defined, not yet loaded")
-- Module ยังไม่โหลด

local val = heavyModule.value  -- ตอนนี้จึงโหลด
print("Got value:", val)       -- Lazy loading: heavy_computation \n Got value: 42

local val2 = heavyModule.name  -- ใช้ cached module
print("Got name:", val2)       -- Got name: heavy_computation
```

```lua
-- ตัวอย่างที่ 15: Lazy module registry
local Registry = {}
Registry._loaders = {}
Registry._loaded = {}

function Registry.register(name, loader)
    Registry._loaders[name] = loader
end

function Registry.get(name)
    if not Registry._loaded[name] then
        local loader = Registry._loaders[name]
        if not loader then
            error("Unknown module: " .. name)
        end
        print("Loading module: " .. name)
        Registry._loaded[name] = loader()
    end
    return Registry._loaded[name]
end

-- Register modules
Registry.register("database", function()
    return {
        connect = function(url) print("Connecting to:", url) end,
        query = function(sql) return {} end
    }
end)

Registry.register("cache", function()
    local data = {}
    return {
        get = function(k) return data[k] end,
        set = function(k, v) data[k] = v end,
        clear = function() data = {} end
    }
end)

Registry.register("config", function()
    return {
        host = "localhost",
        port = 3000,
        debug = false
    }
end)

-- ใช้ modules
local cfg = Registry.get("config")
print("Host:", cfg.host)

local cache = Registry.get("cache")
cache.set("user:1", {name = "Alice"})
print("Cached user:", cache.get("user:1").name)

-- Module ที่ไม่ได้ใช้จะไม่ถูก load
print("\nLoaded modules:")
for name in pairs(Registry._loaded) do
    print("  " .. name)
end
```

---

## 19.8 Plugin System

```lua
-- ตัวอย่างที่ 16: Plugin system พื้นฐาน
local PluginSystem = {}
PluginSystem._plugins = {}
PluginSystem._hooks = {}

function PluginSystem.register(name, plugin)
    if PluginSystem._plugins[name] then
        error("Plugin already registered: " .. name)
    end
    
    -- Validate plugin interface
    assert(type(plugin) == "table", "Plugin must be a table")
    assert(type(plugin.name) == "string", "Plugin must have a name")
    
    PluginSystem._plugins[name] = plugin
    
    -- Install hooks
    if plugin.hooks then
        for hookName, fn in pairs(plugin.hooks) do
            PluginSystem._hooks[hookName] = PluginSystem._hooks[hookName] or {}
            table.insert(PluginSystem._hooks[hookName], {
                plugin = name,
                fn = fn,
                priority = plugin.priority or 10
            })
            -- Sort by priority
            table.sort(PluginSystem._hooks[hookName], function(a, b)
                return a.priority < b.priority
            end)
        end
    end
    
    -- Call install hook
    if plugin.install then
        plugin.install()
    end
    
    print("Plugin installed: " .. name)
end

function PluginSystem.trigger(hookName, ...)
    local handlers = PluginSystem._hooks[hookName]
    if not handlers then return end
    
    for _, handler in ipairs(handlers) do
        local ok, err = pcall(handler.fn, ...)
        if not ok then
            print(string.format("Plugin '%s' hook '%s' error: %s",
                handler.plugin, hookName, err))
        end
    end
end

function PluginSystem.list()
    for name, plugin in pairs(PluginSystem._plugins) do
        print(string.format("  %s v%s - %s",
            name,
            plugin.version or "?",
            plugin.description or ""))
    end
end

-- สร้าง plugins
local LogPlugin = {
    name = "Logger",
    version = "1.0",
    description = "Logs all events",
    priority = 1,  -- รันก่อน
    hooks = {
        onRequest = function(req)
            print("[LOG] Request:", req.method, req.path)
        end,
        onResponse = function(res)
            print("[LOG] Response:", res.status)
        end
    },
    install = function()
        print("Logger plugin installed")
    end
}

local AuthPlugin = {
    name = "Auth",
    version = "2.0",
    description = "Authentication middleware",
    priority = 5,
    hooks = {
        onRequest = function(req)
            if not req.headers or not req.headers.token then
                print("[AUTH] Warning: No token provided")
            else
                print("[AUTH] Token verified:", req.headers.token)
            end
        end
    }
}

local MetricsPlugin = {
    name = "Metrics",
    version = "1.0",
    description = "Collects performance metrics",
    priority = 10,
    hooks = {
        onRequest = function(req)
            req._startTime = os.clock()
        end,
        onResponse = function(res)
            print(string.format("[METRICS] Response time: %.4fs", os.clock()))
        end
    }
}

PluginSystem.register("logger", LogPlugin)
PluginSystem.register("auth", AuthPlugin)
PluginSystem.register("metrics", MetricsPlugin)

print("\nInstalled plugins:")
PluginSystem.list()

print("\nSimulating request:")
local request = {
    method = "GET",
    path = "/api/users",
    headers = {token = "abc123"}
}
PluginSystem.trigger("onRequest", request)

local response = {status = 200}
PluginSystem.trigger("onResponse", response)
```

---

## 19.9 Module Versioning

```lua
-- ตัวอย่างที่ 17: Module versioning
local function parseVersion(vstr)
    local major, minor, patch = vstr:match("(%d+)%.(%d+)%.?(%d*)")
    return {
        major = tonumber(major) or 0,
        minor = tonumber(minor) or 0,
        patch = tonumber(patch) or 0,
        string = vstr
    }
end

local function compareVersions(v1, v2)
    local a = parseVersion(v1)
    local b = parseVersion(v2)
    
    if a.major ~= b.major then return a.major - b.major end
    if a.minor ~= b.minor then return a.minor - b.minor end
    return a.patch - b.patch
end

local function checkCompatibility(required, provided)
    local req = parseVersion(required)
    local prov = parseVersion(provided)
    
    -- Major version must match
    if req.major ~= prov.major then
        return false, string.format(
            "Major version mismatch: requires %d, got %d",
            req.major, prov.major
        )
    end
    
    -- Provided minor must be >= required minor
    if prov.minor < req.minor then
        return false, string.format(
            "Minor version too old: requires %d.%d+, got %d.%d",
            req.major, req.minor, prov.major, prov.minor
        )
    end
    
    return true
end

-- Module with version checking
local function createVersionedModule(name, version)
    local M = {}
    M._NAME = name
    M._VERSION = version
    
    function M.checkVersion(required)
        local ok, err = checkCompatibility(required, version)
        if not ok then
            error(string.format("Module '%s' version error: %s", name, err), 2)
        end
        return true
    end
    
    return M
end

local myLib = createVersionedModule("myLib", "2.3.1")

print("Module:", myLib._NAME, "v" .. myLib._VERSION)

local ok, err = pcall(function()
    myLib.checkVersion("2.0")   -- OK: 2.x >= 2.0
end)
print("Check 2.0:", ok)  -- true

ok, err = pcall(function()
    myLib.checkVersion("2.4")   -- FAIL: 2.3 < 2.4
end)
print("Check 2.4:", ok, err and err:match("version error.*$"))
```

---

## 19.10 Private Module State

```lua
-- ตัวอย่างที่ 18: Private state ด้วย closure
local function createDatabase()
    -- Private state - ไม่มีใครเข้าถึงได้โดยตรง
    local _connections = {}
    local _queryCount = 0
    local _maxConnections = 10
    
    local function _validateConnection(conn)
        return type(conn) == "table" and conn.id ~= nil
    end
    
    -- Public API
    local db = {}
    
    function db.connect(config)
        if #_connections >= _maxConnections then
            error("Max connections reached")
        end
        
        local conn = {
            id = #_connections + 1,
            host = config.host or "localhost",
            port = config.port or 5432,
            active = true
        }
        _connections[#_connections + 1] = conn
        print(string.format("Connected: conn#%d to %s:%d",
            conn.id, conn.host, conn.port))
        return conn
    end
    
    function db.disconnect(conn)
        assert(_validateConnection(conn), "Invalid connection")
        for i, c in ipairs(_connections) do
            if c.id == conn.id then
                table.remove(_connections, i)
                conn.active = false
                print("Disconnected: conn#" .. conn.id)
                return true
            end
        end
        return false
    end
    
    function db.query(conn, sql)
        assert(_validateConnection(conn) and conn.active, "Invalid or closed connection")
        _queryCount = _queryCount + 1
        print(string.format("[conn#%d] Query #%d: %s", conn.id, _queryCount, sql))
        return {rows = {}, count = 0}
    end
    
    function db.stats()
        return {
            activeConnections = #_connections,
            totalQueries = _queryCount,
            maxConnections = _maxConnections
        }
    end
    
    return db
end

local db = createDatabase()

local conn1 = db.connect({host = "db.example.com", port = 5432})
local conn2 = db.connect({host = "localhost"})

db.query(conn1, "SELECT * FROM users")
db.query(conn1, "SELECT * FROM products")
db.query(conn2, "SELECT * FROM orders")

local stats = db.stats()
print(string.format("Active: %d, Total queries: %d",
    stats.activeConnections, stats.totalQueries))

db.disconnect(conn1)
print("Active after disconnect:", db.stats().activeConnections)
```

---

## 19.11 Namespace Collision Avoidance

```lua
-- ตัวอย่างที่ 19: Namespace management
local namespace = {}

function namespace.create(name)
    local ns = {}
    ns._name = name
    ns._modules = {}
    
    function ns.define(modName, factory)
        local fullName = name .. "." .. modName
        if ns._modules[modName] then
            error("Module already defined: " .. fullName)
        end
        ns._modules[modName] = factory()
        return ns._modules[modName]
    end
    
    function ns.require(modName)
        if not ns._modules[modName] then
            error("Module not found in namespace '" .. name .. "': " .. modName)
        end
        return ns._modules[modName]
    end
    
    setmetatable(ns, {
        __index = function(t, k)
            return ns._modules[k]
        end,
        __tostring = function()
            local mods = {}
            for k in pairs(ns._modules) do
                mods[#mods + 1] = k
            end
            table.sort(mods)
            return "Namespace(" .. name .. ") {" .. table.concat(mods, ", ") .. "}"
        end
    })
    
    return ns
end

-- สร้าง namespaces
local app = namespace.create("app")
local lib = namespace.create("lib")

-- app namespace
app.define("config", function()
    return {debug = true, version = "1.0"}
end)

app.define("router", function()
    local routes = {}
    return {
        add = function(path, handler)
            routes[path] = handler
        end,
        handle = function(path)
            local h = routes[path]
            if h then return h()
            else return "404 Not Found" end
        end
    }
end)

-- lib namespace
lib.define("utils", function()
    return {
        uuid = function()
            return string.format("%04x%04x-%04x-%04x-%04x-%04x%04x%04x",
                math.random(0, 0xffff), math.random(0, 0xffff),
                math.random(0, 0xffff),
                math.random(0, 0x0fff) | 0x4000,
                math.random(0, 0x3fff) | 0x8000,
                math.random(0, 0xffff), math.random(0, 0xffff),
                math.random(0, 0xffff))
        end
    }
end)

-- ใช้ namespaces
local config = app.require("config")
print("Debug:", config.debug)

local router = app.require("router")
router.add("/", function() return "Home Page" end)
router.add("/about", function() return "About Page" end)
print(router.handle("/"))       -- Home Page
print(router.handle("/about"))  -- About Page
print(router.handle("/other"))  -- 404 Not Found

local utils = lib.require("utils")
print("UUID:", utils.uuid())

print(tostring(app))
print(tostring(lib))
```

---

## 19.12 Standard Library Modules

```lua
-- ตัวอย่างที่ 20: os module
print("=== os module ===")
print("Date:", os.date("%Y-%m-%d"))
print("Time:", os.date("%H:%M:%S"))
print("Epoch:", os.time())

-- Measure execution time
local function measureTime(fn, ...)
    local start = os.clock()
    local results = {fn(...)}
    local elapsed = os.clock() - start
    return elapsed, table.unpack(results)
end

local time, result = measureTime(function()
    local sum = 0
    for i = 1, 1000000 do sum = sum + i end
    return sum
end)

print(string.format("Sum: %d (%.4f seconds)", result, time))
```

```lua
-- ตัวอย่างที่ 21: io module
print("=== io module ===")

-- Write to temp file
local tmpFile = "/tmp/lua_test_" .. os.time() .. ".txt"
local f = io.open(tmpFile, "w")
if f then
    f:write("Line 1\nLine 2\nLine 3\n")
    f:close()
    
    -- Read back
    local rf = io.open(tmpFile, "r")
    if rf then
        for line in rf:lines() do
            print("Read:", line)
        end
        rf:close()
    end
    
    os.remove(tmpFile)
    print("Temp file cleaned up")
end
```

```lua
-- ตัวอย่างที่ 22: math module
print("=== math module ===")
print("pi:", math.pi)
print("e (approx):", math.exp(1))
print("huge:", math.huge)
print("maxinteger:", math.maxinteger)
print("mininteger:", math.mininteger)
print("type(1.0):", math.type(1.0))    -- float
print("type(1):", math.type(1))        -- integer
print("tointeger(5.0):", math.tointeger(5.0))  -- 5

-- Trigonometry
local angle = math.pi / 4  -- 45 degrees
print(string.format("sin(45°) = %.4f", math.sin(angle)))
print(string.format("cos(45°) = %.4f", math.cos(angle)))
print(string.format("tan(45°) = %.4f", math.tan(angle)))

-- Log and exp
print(string.format("log(100, 10) = %.4f", math.log(100, 10)))
print(string.format("log2(8) = %.4f", math.log(8, 2)))
```

```lua
-- ตัวอย่างที่ 23: string module
print("=== string module ===")

local s = "Hello, World! 123"

-- Pattern matching
print(s:match("(%a+)%s+(%a+)"))     -- Hello  World
print(s:match("%d+"))                -- 123
print(s:gsub("%a+", function(w) return w:upper() end))  -- HELLO, WORLD! 123

-- Format
print(string.format("%-10s|%10s", "left", "right"))
print(string.format("%05.2f", 3.14))

-- Byte operations
print(string.byte("A"))       -- 65
print(string.char(65, 66, 67))  -- ABC
print(string.len("Hello"))    -- 5

-- Find and sub
local start, finish = string.find(s, "World")
print(string.format("'World' at: %d-%d", start, finish))
print(string.sub(s, start, finish))  -- World
```

```lua
-- ตัวอย่างที่ 24: table module
print("=== table module ===")

-- sort
local nums = {5, 3, 8, 1, 9, 2, 7, 4, 6}
table.sort(nums)
print("Sorted:", table.concat(nums, ", "))

-- sort with custom comparator
local people = {
    {name = "Charlie", age = 35},
    {name = "Alice", age = 25},
    {name = "Bob", age = 30},
}
table.sort(people, function(a, b) return a.age < b.age end)
for _, p in ipairs(people) do
    print(string.format("  %s: %d", p.name, p.age))
end

-- move (Lua 5.3+)
local src = {1, 2, 3, 4, 5}
local dst = {10, 20, 30}
table.move(src, 2, 4, 2, dst)  -- copy src[2..4] to dst starting at 2
print("After move:", table.concat(dst, ", "))  -- 10, 2, 3, 4

-- pack and unpack
local packed = table.pack(10, 20, 30, 40)
print("Packed n:", packed.n)  -- 4
print("Unpacked:", table.unpack(packed, 1, packed.n))
```

```lua
-- ตัวอย่างที่ 25: coroutine module
print("=== coroutine module ===")

-- Generator pattern
local function range(from, to, step)
    step = step or 1
    return coroutine.wrap(function()
        local i = from
        while i <= to do
            coroutine.yield(i)
            i = i + step
        end
    end)
end

local sum = 0
for n in range(1, 10) do
    sum = sum + n
end
print("Sum 1-10:", sum)  -- 55

-- Coroutine states
local co = coroutine.create(function(a, b)
    print("Start:", a, b)
    local c = coroutine.yield(a + b)
    print("Resume with:", c)
    return "done"
end)

print("Status:", coroutine.status(co))  -- suspended
local ok, val = coroutine.resume(co, 10, 20)
print("Yield value:", val)              -- 30
print("Status:", coroutine.status(co))  -- suspended
ok, val = coroutine.resume(co, 99)
print("Return value:", val)             -- done
print("Status:", coroutine.status(co))  -- dead
```

---

## 19.13 การเขียน Module สำหรับ LuaRocks

```lua
-- ตัวอย่างที่ 26: Structure สำหรับ LuaRocks package

-- mypackage-1.0.0-1.rockspec (ตัวอย่าง)
--[[
package = "mypackage"
version = "1.0.0-1"
source = {
    url = "https://github.com/user/mypackage/archive/v1.0.0.tar.gz"
}
description = {
    summary = "A useful Lua package",
    detailed = "Detailed description...",
    license = "MIT",
    homepage = "https://github.com/user/mypackage"
}
dependencies = {
    "lua >= 5.1",
    "luasocket >= 3.0"
}
build = {
    type = "builtin",
    modules = {
        ["mypackage"] = "src/mypackage.lua",
        ["mypackage.utils"] = "src/mypackage/utils.lua",
    }
}
]]

-- โครงสร้างไฟล์ที่แนะนำสำหรับ LuaRocks:
-- mypackage/
--   mypackage-1.0.0-1.rockspec
--   src/
--     mypackage.lua           -- main module
--     mypackage/
--       utils.lua             -- submodule
--       init.lua              -- alternative entry point
--   spec/
--     mypackage_spec.lua      -- tests (busted format)
--   README.md
--   LICENSE

-- ตัวอย่าง src/mypackage.lua
local M = {}

M._NAME = "mypackage"
M._VERSION = "1.0.0"
M._DESCRIPTION = "A useful Lua package"

-- Dependencies check
local function checkDependency(name, version)
    local ok, mod = pcall(require, name)
    if not ok then
        error(string.format("Missing dependency: %s >= %s", name, version), 2)
    end
    return mod
end

-- Main functionality
function M.process(data, options)
    options = options or {}
    
    if type(data) ~= "table" then
        error("data must be a table", 2)
    end
    
    local result = {}
    local transform = options.transform or function(x) return x end
    local filter = options.filter or function() return true end
    
    for k, v in pairs(data) do
        if filter(v, k) then
            result[k] = transform(v)
        end
    end
    
    return result
end

function M.version()
    return M._VERSION
end

return M
```

---

## 19.14 Module Testing Pattern

```lua
-- ตัวอย่างที่ 27: Module ที่ testable
-- แยก business logic ออกจาก side effects

-- Pure functions - ง่ายต่อการ test
local StringUtils = {}

function StringUtils.toCamelCase(str)
    return str:gsub("[-_](%a)", function(c)
        return c:upper()
    end)
end

function StringUtils.toSnakeCase(str)
    return str:gsub("%u", function(c)
        return "_" .. c:lower()
    end):gsub("^_", "")
end

function StringUtils.truncate(str, maxLen, ellipsis)
    ellipsis = ellipsis or "..."
    if #str <= maxLen then return str end
    return str:sub(1, maxLen - #ellipsis) .. ellipsis
end

function StringUtils.template(tmpl, vars)
    return (tmpl:gsub("{(%w+)}", function(key)
        return tostring(vars[key] or "{" .. key .. "}")
    end))
end

-- Simple test framework
local function test(name, fn)
    local ok, err = pcall(fn)
    if ok then
        print("[PASS] " .. name)
    else
        print("[FAIL] " .. name .. ": " .. err)
    end
end

local function assertEqual(a, b, msg)
    if a ~= b then
        error(string.format("%s: expected '%s', got '%s'",
            msg or "assertEqual failed", tostring(b), tostring(a)), 2)
    end
end

-- Tests
test("toCamelCase with dash", function()
    assertEqual(StringUtils.toCamelCase("hello-world"), "helloWorld")
end)

test("toCamelCase with underscore", function()
    assertEqual(StringUtils.toCamelCase("foo_bar_baz"), "fooBarBaz")
end)

test("toSnakeCase", function()
    assertEqual(StringUtils.toSnakeCase("helloWorld"), "hello_world")
end)

test("truncate short string", function()
    assertEqual(StringUtils.truncate("Hello", 10), "Hello")
end)

test("truncate long string", function()
    assertEqual(StringUtils.truncate("Hello, World!", 8), "Hello...")
end)

test("template substitution", function()
    local result = StringUtils.template("Hello, {name}! You are {age}.", {
        name = "Alice", age = 30
    })
    assertEqual(result, "Hello, Alice! You are 30.")
end)
```

---

## 19.15 Advanced Module Patterns

```lua
-- ตัวอย่างที่ 28: Mixin pattern
local function mixin(target, ...)
    for _, source in ipairs({...}) do
        for k, v in pairs(source) do
            if type(v) == "function" and not target[k] then
                target[k] = v
            end
        end
    end
    return target
end

-- Mixins
local Serializable = {
    serialize = function(self)
        local parts = {}
        for k, v in pairs(self) do
            if type(v) ~= "function" then
                parts[#parts + 1] = string.format("%s=%s", k, tostring(v))
            end
        end
        table.sort(parts)
        return "{" .. table.concat(parts, ",") .. "}"
    end
}

local Comparable = {
    equals = function(self, other)
        return self:serialize() == other:serialize()
    end
}

local Printable = {
    print = function(self)
        print(self:serialize())
    end
}

-- Class ที่ใช้ mixins
local Point = {}
Point.__index = Point

function Point.new(x, y)
    return setmetatable({x=x, y=y}, Point)
end

mixin(Point, Serializable, Comparable, Printable)

local p1 = Point.new(1, 2)
local p2 = Point.new(1, 2)
local p3 = Point.new(3, 4)

p1:print()                  -- {x=1,y=2}
print(p1:equals(p2))       -- true
print(p1:equals(p3))       -- false
print(p1:serialize())      -- {x=1,y=2}
```

```lua
-- ตัวอย่างที่ 29: Singleton pattern
local function createSingleton(factory)
    local instance = nil
    return {
        getInstance = function()
            if not instance then
                instance = factory()
                print("Creating singleton instance")
            end
            return instance
        end,
        resetInstance = function()  -- สำหรับ testing เท่านั้น
            instance = nil
        end
    }
end

local AppConfig = createSingleton(function()
    return {
        debug = false,
        logLevel = "INFO",
        maxRetries = 3,
        timeout = 30
    }
end)

local c1 = AppConfig.getInstance()
local c2 = AppConfig.getInstance()

print(c1 == c2)   -- true (same instance)
c1.debug = true
print(c2.debug)   -- true (same object)
```

```lua
-- ตัวอย่างที่ 30: Event-driven module
local EventBus = {}
EventBus.__index = EventBus

local function createEventBus()
    return setmetatable({
        _handlers = {},
        _wildcardHandlers = {}
    }, EventBus)
end

function EventBus:on(event, handler, options)
    options = options or {}
    local entry = {
        fn = handler,
        once = options.once or false,
        priority = options.priority or 0
    }
    
    self._handlers[event] = self._handlers[event] or {}
    table.insert(self._handlers[event], entry)
    table.sort(self._handlers[event], function(a, b)
        return a.priority > b.priority
    end)
    
    return self  -- for chaining
end

function EventBus:once(event, handler)
    return self:on(event, handler, {once = true})
end

function EventBus:emit(event, data)
    local handlers = self._handlers[event]
    if not handlers then return self end
    
    local toRemove = {}
    for i, entry in ipairs(handlers) do
        entry.fn(data)
        if entry.once then
            table.insert(toRemove, 1, i)  -- insert at front for reverse removal
        end
    end
    
    for _, i in ipairs(toRemove) do
        table.remove(handlers, i)
    end
    
    return self
end

function EventBus:off(event, handler)
    if not self._handlers[event] then return self end
    for i, entry in ipairs(self._handlers[event]) do
        if entry.fn == handler then
            table.remove(self._handlers[event], i)
            break
        end
    end
    return self
end

-- Test
local bus = createEventBus()

local log = {}

bus:on("user:login", function(data)
    log[#log+1] = string.format("Login: %s", data.username)
end)

bus:once("user:login", function(data)
    log[#log+1] = string.format("First login ever: %s", data.username)
end)

bus:on("user:logout", function(data)
    log[#log+1] = string.format("Logout: %s", data.username)
end)

bus:emit("user:login", {username = "Alice"})
bus:emit("user:login", {username = "Bob"})
bus:emit("user:logout", {username = "Alice"})

for _, entry in ipairs(log) do
    print(entry)
end
-- Login: Alice
-- First login ever: Alice  (once only)
-- Login: Bob
-- Logout: Alice
```

```lua
-- ตัวอย่างที่ 31: Module ที่มี configuration
local function createModule(config)
    -- Default configuration
    local defaults = {
        timeout = 30,
        retries = 3,
        verbose = false,
        prefix = "[Module]"
    }
    
    -- Merge with defaults
    local cfg = {}
    for k, v in pairs(defaults) do cfg[k] = v end
    if config then
        for k, v in pairs(config) do cfg[k] = v end
    end
    
    -- Private helpers
    local function log(msg)
        if cfg.verbose then
            print(cfg.prefix .. " " .. msg)
        end
    end
    
    local function retry(fn, maxRetries)
        local attempts = 0
        while attempts < maxRetries do
            local ok, result = pcall(fn)
            if ok then return result end
            attempts = attempts + 1
            log(string.format("Retry %d/%d after error: %s",
                attempts, maxRetries, tostring(result)))
        end
        error("Max retries exceeded")
    end
    
    -- Public API
    local M = {}
    
    function M.configure(newConfig)
        for k, v in pairs(newConfig) do
            cfg[k] = v
        end
    end
    
    function M.fetch(url)
        return retry(function()
            log("Fetching: " .. url)
            -- Simulate fetch
            if math.random() < 0.3 then
                error("Network error")
            end
            return {status = 200, body = "response from " .. url}
        end, cfg.retries)
    end
    
    function M.getConfig()
        local copy = {}
        for k, v in pairs(cfg) do copy[k] = v end
        return copy
    end
    
    return M
end

math.randomseed(42)
local http = createModule({verbose = true, retries = 5})
print("Config:", http.getConfig().verbose, http.getConfig().retries)
```

---

## 19.16 ตัวอย่างรวม: Package Manager Mini

```lua
-- ตัวอย่างที่ 32: Mini package manager
local PackageManager = {}
PackageManager.__index = PackageManager

function PackageManager.new()
    return setmetatable({
        _registry = {},     -- available packages
        _installed = {},    -- installed packages
        _loading = {}       -- currently loading (for circular dep detection)
    }, PackageManager)
end

function PackageManager:register(name, version, deps, factory)
    if not self._registry[name] then
        self._registry[name] = {}
    end
    self._registry[name][version] = {
        deps = deps or {},
        factory = factory
    }
    print(string.format("Registered: %s@%s", name, version))
end

function PackageManager:install(name, version)
    version = version or self:_latestVersion(name)
    local key = name .. "@" .. version
    
    if self._installed[key] then
        return self._installed[key]
    end
    
    if self._loading[key] then
        error("Circular dependency detected: " .. key)
    end
    
    local pkg = self._registry[name] and self._registry[name][version]
    if not pkg then
        error(string.format("Package not found: %s@%s", name, version))
    end
    
    self._loading[key] = true
    
    -- Install dependencies first
    local deps = {}
    for depName, depVersion in pairs(pkg.deps) do
        deps[depName] = self:install(depName, depVersion)
    end
    
    -- Create package instance
    local instance = pkg.factory(deps)
    self._installed[key] = instance
    self._loading[key] = nil
    
    print(string.format("Installed: %s@%s", name, version))
    return instance
end

function PackageManager:_latestVersion(name)
    local versions = {}
    if self._registry[name] then
        for v in pairs(self._registry[name]) do
            versions[#versions + 1] = v
        end
        table.sort(versions)
        return versions[#versions]
    end
    error("Package not found: " .. name)
end

-- Test
local pm = PackageManager.new()

-- Register packages
pm:register("lodash", "4.17", {}, function(deps)
    return {
        map = function(t, fn)
            local result = {}
            for k, v in pairs(t) do result[k] = fn(v) end
            return result
        end,
        filter = function(t, fn)
            local result = {}
            for _, v in ipairs(t) do
                if fn(v) then result[#result+1] = v end
            end
            return result
        end
    }
end)

pm:register("utils", "1.0", {lodash = "4.17"}, function(deps)
    return {
        doubleAll = function(t)
            return deps.lodash.map(t, function(x) return x * 2 end)
        end,
        evens = function(t)
            return deps.lodash.filter(t, function(x) return x % 2 == 0 end)
        end
    }
end)

pm:register("app", "1.0", {utils = "1.0"}, function(deps)
    return {
        run = function()
            local nums = {1, 2, 3, 4, 5}
            local doubled = deps.utils.doubleAll(nums)
            local evens = deps.utils.evens(doubled)
            print("Doubled:", table.concat(doubled, ", "))
            print("Evens of doubled:", table.concat(evens, ", "))
        end
    }
end)

print("\n--- Installing packages ---")
local app = pm:install("app")

print("\n--- Running app ---")
app.run()
```

---

## 19.17 ตัวอย่างรวม: Module System สมบูรณ์

```lua
-- ตัวอย่างที่ 33: Complete module system
local ModuleSystem = (function()
    local _loaded = {}
    local _preloads = {}
    local _aliases = {}
    
    local M = {}
    
    -- Register a preloaded module
    function M.preload(name, factory)
        _preloads[name] = factory
    end
    
    -- Create an alias
    function M.alias(alias, original)
        _aliases[alias] = original
    end
    
    -- Require a module
    function M.require(name)
        -- Resolve alias
        name = _aliases[name] or name
        
        -- Check cache
        if _loaded[name] then
            return _loaded[name]
        end
        
        -- Check preloads
        if _preloads[name] then
            local mod = _preloads[name]()
            _loaded[name] = mod
            return mod
        end
        
        -- Fall back to Lua's require
        return require(name)
    end
    
    -- List all loaded modules
    function M.list()
        local names = {}
        for k in pairs(_loaded) do names[#names+1] = k end
        table.sort(names)
        return names
    end
    
    -- Unload a module (for testing)
    function M.unload(name)
        _loaded[name] = nil
    end
    
    return M
end)()

-- Register modules
ModuleSystem.preload("math.extra", function()
    return {
        fibonacci = function(n)
            if n <= 1 then return n end
            local a, b = 0, 1
            for _ = 2, n do a, b = b, a+b end
            return b
        end,
        isPrime = function(n)
            if n < 2 then return false end
            if n == 2 then return true end
            if n % 2 == 0 then return false end
            for i = 3, math.sqrt(n), 2 do
                if n % i == 0 then return false end
            end
            return true
        end
    }
end)

ModuleSystem.preload("string.extra", function()
    return {
        words = function(s)
            local w = {}
            for word in s:gmatch("%S+") do w[#w+1] = word end
            return w
        end,
        wordCount = function(s)
            local count = 0
            for _ in s:gmatch("%S+") do count = count + 1 end
            return count
        end
    }
end)

-- Set alias
ModuleSystem.alias("mathex", "math.extra")
ModuleSystem.alias("strex", "string.extra")

-- Use modules
local mathex = ModuleSystem.require("mathex")
print("Fibonacci(10):", mathex.fibonacci(10))
print("isPrime(17):", mathex.isPrime(17))
print("isPrime(18):", mathex.isPrime(18))

local strex = ModuleSystem.require("strex")
local text = "The quick brown fox jumps"
print("Words:", table.concat(strex.words(text), ", "))
print("Word count:", strex.wordCount(text))

print("Loaded modules:", table.concat(ModuleSystem.list(), ", "))
```

```lua
-- ตัวอย่างที่ 34: Hot reload module
local HotReload = {}

function HotReload.watch(modName, factory)
    local lastMod = nil
    local instance = nil
    
    return setmetatable({}, {
        __index = function(_, k)
            -- Check if needs reload
            local currentMod = os.time()  -- ใน production จะเช็ค file mtime
            if lastMod ~= currentMod then
                print(string.format("[HotReload] Reloading %s", modName))
                instance = factory()
                lastMod = currentMod
            end
            return instance[k]
        end,
        __call = function(_, ...)
            if instance then return instance(...) end
        end
    })
end

-- สาธิต (simplified)
local reloadCount = 0
local hotModule = HotReload.watch("my.module", function()
    reloadCount = reloadCount + 1
    return {
        version = reloadCount,
        greet = function(name)
            return string.format("v%d Hello, %s!", reloadCount, name)
        end
    }
end)

print(hotModule.greet("Alice"))   -- v1 Hello, Alice!
-- Force reload
print(hotModule.greet("Bob"))     -- same or new version
```

```lua
-- ตัวอย่างที่ 35: Module documentation pattern
local function documented(M, docs)
    local meta = getmetatable(M) or {}
    meta.__docs = docs
    
    -- Add help function
    M.help = function(fnName)
        if fnName then
            local doc = docs[fnName]
            if doc then
                print("=== " .. fnName .. " ===")
                print(doc.description)
                if doc.params then
                    print("Parameters:")
                    for pname, pdesc in pairs(doc.params) do
                        print(string.format("  %s: %s", pname, pdesc))
                    end
                end
                if doc.returns then
                    print("Returns:", doc.returns)
                end
                if doc.example then
                    print("Example:")
                    print(doc.example)
                end
            else
                print("No documentation for: " .. fnName)
            end
        else
            -- List all documented functions
            print("Available functions:")
            for fname in pairs(docs) do
                print("  " .. fname .. " - " .. (docs[fname].description or ""))
            end
        end
    end
    
    return setmetatable(M, meta)
end

-- สร้าง documented module
local mathLib = documented({
    add = function(a, b) return a + b end,
    multiply = function(a, b) return a * b end,
    factorial = function(n)
        if n <= 1 then return 1 end
        return n * mathLib.factorial(n - 1)
    end
}, {
    add = {
        description = "Add two numbers",
        params = {a = "First number", b = "Second number"},
        returns = "Sum of a and b",
        example = "mathLib.add(2, 3) --> 5"
    },
    multiply = {
        description = "Multiply two numbers",
        params = {a = "First number", b = "Second number"},
        returns = "Product of a and b",
        example = "mathLib.multiply(4, 5) --> 20"
    },
    factorial = {
        description = "Calculate factorial",
        params = {n = "Non-negative integer"},
        returns = "n!",
        example = "mathLib.factorial(5) --> 120"
    }
})

mathLib.help()          -- List all functions
print()
mathLib.help("add")     -- Show add documentation
print()
print(mathLib.add(3, 4))
print(mathLib.factorial(6))
```

---

## แบบฝึกหัด

### ระดับพื้นฐาน

1. **สร้าง MathUtils Module**: เขียน module ที่มีฟังก์ชัน:
   - `isPrime(n)` - ตรวจสอบจำนวนเฉพาะ
   - `primeFactors(n)` - หาตัวประกอบเฉพาะ
   - `gcd(a, b)` - หาร.ม.ก.
   - `lcm(a, b)` - หาค.ร.น.
   - `fibonacci(n)` - หาเลข Fibonacci ที่ n
   - Module ต้อง return table และมี `_VERSION`

2. **สร้าง StringUtils Module**: เขียน module ที่มี:
   - `trim(s)` - ตัด whitespace หัวท้าย
   - `split(s, sep)` - แบ่ง string ด้วย separator
   - `join(parts, sep)` - รวม table of strings
   - `repeat_str(s, n)` - ทำซ้ำ string
   - `reverse(s)` - กลับ string
   - `isPalindrome(s)` - ตรวจ palindrome

3. **สร้าง Config Module**: Module สำหรับจัดการ configuration:
   - `set(key, value)` - ตั้งค่า
   - `get(key, default)` - อ่านค่า (พร้อม default)
   - `has(key)` - ตรวจสอบว่ามี key
   - `delete(key)` - ลบค่า
   - `all()` - ดูค่าทั้งหมด
   - Support nested keys: `config.get("db.host")`

### ระดับกลาง

4. **Plugin System**: สร้าง plugin system ที่:
   - มี `register(name, plugin)` และ `unregister(name)`
   - Support hooks: `before`, `after`, `transform`
   - Plugin สามารถ depend on Pluginsอื่นได้
   - ตรวจสอบ circular dependencies
   - มี priority ordering

5. **Module Loader**: สร้าง custom module loader ที่:
   - Support multiple paths
   - Cache loaded modules
   - Support `require.path` configuration
   - มี `reload(name)` สำหรับ force reload
   - แสดง dependency graph

6. **Event Module**: เขียน event system สมบูรณ์:
   - `on(event, handler)` - subscribe
   - `off(event, handler)` - unsubscribe
   - `emit(event, ...data)` - trigger
   - `once(event, handler)` - one-time handler
   - `onAny(handler)` - handle all events
   - Support async handlers ด้วย coroutines

### ระดับสูง

7. **Dependency Injection Container**: สร้าง DI container:
   - `bind(name, factory)` - register service
   - `singleton(name, factory)` - register singleton
   - `resolve(name)` - get service instance
   - Auto-inject dependencies
   - Detect circular dependencies

8. **Middleware Pipeline**: สร้าง middleware system:
   - `use(middleware)` - เพิ่ม middleware
   - Middleware สามารถ transform request/response
   - Support async middleware ด้วย coroutines
   - มี error handling middleware
   - สามารถ branch pipeline ตาม condition

9. **Hot-Reloadable Config**: สร้าง config system ที่:
   - อ่านจากไฟล์ (TOML-like format)
   - Watch ไฟล์สำหรับ changes
   - Notify subscribers เมื่อ config เปลี่ยน
   - Validate ค่าตาม schema
   - Support default values และ required fields

---

## สรุป

ใน บทที่ 19 เราได้เรียนรู้เกี่ยวกับ Modules และ Packages ใน Lua:

| หัวข้อ | รายละเอียด |
|--------|-----------|
| Module Pattern | return table จาก file |
| `require()` | โหลดและ cache modules |
| `package.path` | paths สำหรับค้นหา modules |
| `package.loaded` | cache ของ loaded modules |
| Submodules | การจัดระเบียบ modules ย่อย |
| Lazy Loading | โหลดเมื่อต้องการจริงๆ |
| Plugin System | extensible architecture |
| Private State | closure สำหรับ encapsulation |

หลักการสำคัญในการเขียน Lua modules:
1. ใช้ local variable สำหรับ private state
2. Return table เป็น public API เสมอ
3. ใช้ `require()` และ `package.loaded` อย่างถูกต้อง
4. ระวัง circular dependencies
5. ตั้งชื่อ module ให้สอดคล้องกับ file path
