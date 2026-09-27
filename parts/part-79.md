# บทที่ 79: Plugin System ใน Lua

## บทนำ

Plugin System ช่วยให้ application สามารถ extend functionality ได้โดยไม่ต้องแก้ code หลัก เหมาะสำหรับ application ที่ต้องการ customization สูง เช่น web framework, editor, game engine

---

## 1. ทำไมต้องมี Plugin Architecture

```lua
-- ปัญหา: Monolithic application ที่ hard-coded ทุกอย่าง
local MonolithApp = {}

function MonolithApp.handleRequest(request)
    -- Hard-coded: authentication
    if not request.token then
        return { status = 401, body = "Unauthorized" }
    end
    
    -- Hard-coded: logging
    print("[LOG] " .. request.method .. " " .. request.path)
    
    -- Hard-coded: rate limiting
    -- (ต้องแก้ code ทุกครั้งที่ต้องการเพิ่ม/ลบ feature)
    
    return { status = 200, body = "OK" }
end

-- Solution: Plugin Architecture
-- - เพิ่ม feature ได้โดยไม่แก้ core code
-- - แต่ละ plugin มีชีวิตของตัวเอง
-- - สามารถ enable/disable ได้
-- - สามารถ configure ได้แยกกัน

print("Plugin Architecture Benefits:")
print("1. Open/Closed Principle - open for extension, closed for modification")
print("2. Single Responsibility - แต่ละ plugin ทำหน้าที่เดียว")
print("3. Easy testing - test plugin แยกจาก core")
print("4. Hot reload - เปลี่ยน plugin โดยไม่ restart application")
```

---

## 2. Plugin Discovery (Directory Scanning)

```lua
-- Plugin Discovery ด้วยการ scan directory
local PluginDiscovery = {}

function PluginDiscovery.scan(directory)
    local plugins = {}
    
    -- ใช้ os.execute เพื่อ list files (platform-specific)
    -- ในตัวอย่างนี้จำลองการ scan
    local pluginFiles = {
        directory .. "/auth.plugin.lua",
        directory .. "/logger.plugin.lua",
        directory .. "/cache.plugin.lua",
        directory .. "/metrics.plugin.lua"
    }
    
    for _, filePath in ipairs(pluginFiles) do
        -- ตรวจสอบว่า file มีอยู่จริง
        local f = io.open(filePath, "r")
        if f then
            f:close()
            local plugin = PluginDiscovery.loadFromFile(filePath)
            if plugin then
                table.insert(plugins, plugin)
            end
        else
            -- จำลอง plugin ที่หาไม่เจอ
            print(string.format("[Discovery] File not found: %s (simulating)", filePath))
        end
    end
    
    return plugins
end

function PluginDiscovery.loadFromFile(filePath)
    local ok, plugin = pcall(function()
        -- Load plugin file
        local fn, err = loadfile(filePath)
        if not fn then
            error("Failed to load: " .. err)
        end
        return fn()
    end)
    
    if not ok then
        print(string.format("[Discovery] Error loading %s: %s", filePath, plugin))
        return nil
    end
    
    -- Validate plugin structure
    if not plugin or not plugin.name then
        print(string.format("[Discovery] Invalid plugin structure in %s", filePath))
        return nil
    end
    
    return plugin
end

function PluginDiscovery.loadFromString(code, name)
    local fn, err = load(code, "@" .. (name or "plugin"))
    if not fn then
        error("Syntax error in plugin: " .. err)
    end
    return fn()
end

-- ตัวอย่าง: Load plugin จาก string (simulating file)
local samplePlugin = PluginDiscovery.loadFromString([[
return {
    name = "sample",
    version = "1.0.0",
    description = "A sample plugin",
    author = "Dev Team",
    
    init = function(self, app)
        print("Sample plugin initialized")
    end,
    
    hooks = {
        beforeRequest = function(ctx)
            ctx.pluginData = "added by sample"
        end
    }
}
]])

print("Loaded plugin:", samplePlugin.name, "v" .. samplePlugin.version)
```

---

## 3. Plugin Manifest/Metadata

```lua
-- Plugin Manifest - metadata สำหรับ plugin
local PluginManifest = {}
PluginManifest.__index = PluginManifest

function PluginManifest.new(data)
    local manifest = setmetatable({}, PluginManifest)
    
    -- Required fields
    manifest.name = data.name or error("Plugin must have a name")
    manifest.version = data.version or "0.1.0"
    
    -- Optional metadata
    manifest.description = data.description or ""
    manifest.author = data.author or "Unknown"
    manifest.license = data.license or "MIT"
    manifest.homepage = data.homepage
    manifest.repository = data.repository
    
    -- Dependencies
    manifest.dependencies = data.dependencies or {}
    manifest.optionalDeps = data.optionalDependencies or {}
    
    -- Compatibility
    manifest.minHostVersion = data.minHostVersion or "0.0.0"
    manifest.maxHostVersion = data.maxHostVersion
    manifest.luaVersion = data.luaVersion or "5.4"
    
    -- Permissions
    manifest.permissions = data.permissions or {}
    
    -- Configuration schema
    manifest.config = data.config or {}
    
    return manifest
end

function PluginManifest:validate()
    local errors = {}
    
    -- Validate version format
    if not self.version:match("^%d+%.%d+%.%d+") then
        table.insert(errors, "Invalid version format: " .. self.version)
    end
    
    -- Validate name format
    if not self.name:match("^[a-z][a-z0-9%-_]*$") then
        table.insert(errors, "Invalid name format (must be lowercase alphanumeric)")
    end
    
    return #errors == 0, errors
end

function PluginManifest:isCompatible(hostVersion)
    -- Simple version comparison
    local function parseVersion(v)
        local major, minor, patch = v:match("(%d+)%.(%d+)%.(%d+)")
        return tonumber(major), tonumber(minor), tonumber(patch)
    end
    
    local hMaj, hMin, hPatch = parseVersion(hostVersion)
    local minMaj, minMin, minPatch = parseVersion(self.minHostVersion)
    
    if hMaj < minMaj then return false end
    if hMaj == minMaj and hMin < minMin then return false end
    if hMaj == minMaj and hMin == minMin and hPatch < minPatch then return false end
    
    return true
end

function PluginManifest:requires()
    return self.dependencies
end

-- ตัวอย่าง
local authManifest = PluginManifest.new({
    name = "auth-plugin",
    version = "2.1.0",
    description = "JWT Authentication plugin",
    author = "Security Team",
    dependencies = {
        "jwt-lib >= 1.0.0",
        "crypto >= 2.0.0"
    },
    minHostVersion = "1.0.0",
    permissions = { "read:request", "write:response", "access:headers" },
    config = {
        secretKey = { type = "string", required = true },
        expiresIn = { type = "number", default = 3600 },
        algorithm = { type = "string", default = "HS256" }
    }
})

local valid, errors = authManifest:validate()
print("Manifest valid:", valid)
print("Compatible with v1.5.0:", authManifest:isCompatible("1.5.0"))
print("Compatible with v0.9.0:", authManifest:isCompatible("0.9.0"))
```

---

## 4. Plugin Lifecycle

```lua
-- Plugin Lifecycle: init -> start -> (running) -> stop -> destroy
local PluginLifecycle = {}

local STATES = {
    UNREGISTERED = "unregistered",
    REGISTERED = "registered",
    INITIALIZING = "initializing",
    INITIALIZED = "initialized",
    STARTING = "starting",
    RUNNING = "running",
    STOPPING = "stopping",
    STOPPED = "stopped",
    ERROR = "error"
}

local PluginInstance = {}
PluginInstance.__index = PluginInstance

function PluginInstance.new(plugin, config)
    return setmetatable({
        plugin = plugin,
        config = config or {},
        state = STATES.REGISTERED,
        error = nil,
        startTime = nil,
        _resources = {}
    }, PluginInstance)
end

function PluginInstance:init(app)
    if self.state ~= STATES.REGISTERED then
        error("Cannot init: plugin is in state " .. self.state)
    end
    
    self.state = STATES.INITIALIZING
    
    local ok, err = pcall(function()
        if self.plugin.init then
            self.plugin:init(app, self.config)
        end
    end)
    
    if not ok then
        self.state = STATES.ERROR
        self.error = err
        error(string.format("Plugin '%s' init failed: %s", self.plugin.name, err))
    end
    
    self.state = STATES.INITIALIZED
    print(string.format("[Lifecycle] Plugin '%s' initialized", self.plugin.name))
end

function PluginInstance:start()
    if self.state ~= STATES.INITIALIZED and self.state ~= STATES.STOPPED then
        error("Cannot start: plugin is in state " .. self.state)
    end
    
    self.state = STATES.STARTING
    
    local ok, err = pcall(function()
        if self.plugin.start then
            self.plugin:start(self.config)
        end
    end)
    
    if not ok then
        self.state = STATES.ERROR
        self.error = err
        error(string.format("Plugin '%s' start failed: %s", self.plugin.name, err))
    end
    
    self.state = STATES.RUNNING
    self.startTime = os.time()
    print(string.format("[Lifecycle] Plugin '%s' started", self.plugin.name))
end

function PluginInstance:stop()
    if self.state ~= STATES.RUNNING then
        print(string.format("[Lifecycle] Plugin '%s' not running, skipping stop", self.plugin.name))
        return
    end
    
    self.state = STATES.STOPPING
    
    local ok, err = pcall(function()
        if self.plugin.stop then
            self.plugin:stop()
        end
    end)
    
    if not ok then
        print(string.format("[Lifecycle] Warning: Plugin '%s' stop error: %s", 
            self.plugin.name, err))
    end
    
    -- Cleanup resources
    for _, resource in ipairs(self._resources) do
        if resource.cleanup then
            pcall(resource.cleanup)
        end
    end
    
    self.state = STATES.STOPPED
    print(string.format("[Lifecycle] Plugin '%s' stopped (ran for %d sec)",
        self.plugin.name, os.time() - (self.startTime or os.time())))
end

function PluginInstance:registerResource(resource)
    table.insert(self._resources, resource)
end

function PluginInstance:isRunning()
    return self.state == STATES.RUNNING
end

-- ตัวอย่าง plugin lifecycle
local loggerPlugin = {
    name = "logger",
    version = "1.0.0",
    
    init = function(self, app, config)
        self._logFile = config.logFile
        self._level = config.level or "info"
        print(string.format("    Logger init: level=%s", self._level))
    end,
    
    start = function(self, config)
        self._started = true
        print("    Logger started, writing to stdout")
    end,
    
    stop = function(self)
        self._started = false
        print("    Logger stopped, flushing buffers")
    end,
    
    log = function(self, level, message)
        if self._started then
            print(string.format("[%s] %s", level:upper(), message))
        end
    end
}

local instance = PluginInstance.new(loggerPlugin, {
    level = "debug",
    logFile = "/var/log/app.log"
})

local fakeApp = { version = "2.0.0", name = "TestApp" }

instance:init(fakeApp)
instance:start()
loggerPlugin:log("info", "Application started")
instance:stop()

print("Plugin state:", instance.state)
```

---

## 5. Plugin Dependencies

```lua
-- Plugin Dependency Resolution
local DependencyResolver = {}

function DependencyResolver.resolve(plugins)
    -- Build dependency graph
    local graph = {}
    local pluginMap = {}
    
    for _, plugin in ipairs(plugins) do
        pluginMap[plugin.name] = plugin
        graph[plugin.name] = plugin.dependencies or {}
    end
    
    -- Topological sort (Kahn's algorithm)
    local inDegree = {}
    local adj = {}  -- reverse adjacency list
    
    for name, _ in pairs(pluginMap) do
        inDegree[name] = 0
        adj[name] = {}
    end
    
    for name, deps in pairs(graph) do
        for _, dep in ipairs(deps) do
            if not pluginMap[dep] then
                error(string.format("Plugin '%s' requires '%s' which is not registered", 
                    name, dep))
            end
            inDegree[name] = (inDegree[name] or 0) + 1
            table.insert(adj[dep], name)
        end
    end
    
    -- Find nodes with no dependencies
    local queue = {}
    for name, degree in pairs(inDegree) do
        if degree == 0 then
            table.insert(queue, name)
        end
    end
    
    -- Sort queue for deterministic ordering
    table.sort(queue)
    
    local sorted = {}
    while #queue > 0 do
        local node = table.remove(queue, 1)
        table.insert(sorted, pluginMap[node])
        
        -- Reduce in-degree for dependents
        for _, dependent in ipairs(adj[node] or {}) do
            inDegree[dependent] = inDegree[dependent] - 1
            if inDegree[dependent] == 0 then
                table.insert(queue, dependent)
                table.sort(queue)
            end
        end
    end
    
    -- Check for cycles
    if #sorted ~= #plugins then
        error("Circular dependency detected in plugins")
    end
    
    return sorted
end

function DependencyResolver.checkMissing(plugins)
    local available = {}
    for _, plugin in ipairs(plugins) do
        available[plugin.name] = true
    end
    
    local missing = {}
    for _, plugin in ipairs(plugins) do
        for _, dep in ipairs(plugin.dependencies or {}) do
            if not available[dep] then
                table.insert(missing, {
                    plugin = plugin.name,
                    requires = dep
                })
            end
        end
    end
    
    return missing
end

-- Test dependency resolution
local plugins = {
    { name = "metrics", version = "1.0.0", dependencies = { "logger" } },
    { name = "auth", version = "1.0.0", dependencies = { "logger", "cache" } },
    { name = "cache", version = "1.0.0", dependencies = { "logger" } },
    { name = "logger", version = "1.0.0", dependencies = {} },
    { name = "api", version = "1.0.0", dependencies = { "auth", "metrics" } }
}

print("Plugins to load:", #plugins)

local missing = DependencyResolver.checkMissing(plugins)
if #missing > 0 then
    print("Missing dependencies!")
else
    print("All dependencies satisfied")
end

local sorted = DependencyResolver.resolve(plugins)
print("\nLoad order:")
for i, plugin in ipairs(sorted) do
    print(string.format("  %d. %s", i, plugin.name))
end
```

---

## 6. Hot Reload Plugins

```lua
-- Hot Reload: เปลี่ยน plugin code โดยไม่ restart application
local HotReloadManager = {}
HotReloadManager.__index = HotReloadManager

function HotReloadManager.new(registry)
    return setmetatable({
        registry = registry,
        _fileTimestamps = {},
        _watchInterval = 1  -- seconds
    }, HotReloadManager)
end

function HotReloadManager:getFileTimestamp(filePath)
    -- Get file modification time
    -- (ในระบบจริงใช้ lfs.attributes หรือ os.stat)
    return os.time()  -- placeholder
end

function HotReloadManager:watch(pluginName, filePath)
    self._fileTimestamps[pluginName] = {
        path = filePath,
        lastModified = self:getFileTimestamp(filePath)
    }
    print(string.format("[HotReload] Watching %s -> %s", pluginName, filePath))
end

function HotReloadManager:checkForChanges()
    local changed = {}
    
    for name, info in pairs(self._fileTimestamps) do
        local currentTime = self:getFileTimestamp(info.path)
        if currentTime > info.lastModified then
            table.insert(changed, name)
            info.lastModified = currentTime
        end
    end
    
    return changed
end

function HotReloadManager:reload(pluginName)
    print(string.format("[HotReload] Reloading plugin: %s", pluginName))
    
    local info = self._fileTimestamps[pluginName]
    if not info then
        print("[HotReload] Plugin not being watched: " .. pluginName)
        return false
    end
    
    -- Stop old version
    local oldInstance = self.registry:getInstance(pluginName)
    if oldInstance and oldInstance:isRunning() then
        oldInstance:stop()
    end
    
    -- Load new version
    local ok, err = pcall(function()
        local fn = loadfile(info.path)
        if fn then
            local newPlugin = fn()
            self.registry:update(pluginName, newPlugin)
            
            -- Start new version
            local newInstance = self.registry:getInstance(pluginName)
            if newInstance then
                newInstance:start()
            end
        end
    end)
    
    if not ok then
        print(string.format("[HotReload] Failed to reload %s: %s", pluginName, err))
        -- Restart old version on failure
        if oldInstance then
            oldInstance:start()
        end
        return false
    end
    
    print(string.format("[HotReload] Successfully reloaded: %s", pluginName))
    return true
end

-- Simulate hot reload scenario
local function simulateHotReload()
    print("=== Hot Reload Simulation ===")
    
    -- Initial plugin version
    local pluginV1 = {
        name = "greeting",
        version = "1.0.0",
        greet = function(self, name)
            return "Hello, " .. name
        end
    }
    
    -- Updated plugin version (would be in file)
    local pluginV2 = {
        name = "greeting",
        version = "2.0.0",
        greet = function(self, name)
            return "Sawadee krap, " .. name .. "!"
        end
    }
    
    -- Simulate the loaded plugin
    local currentPlugin = pluginV1
    print("Current version:", currentPlugin.version)
    print("Greeting:", currentPlugin:greet("Alice"))
    
    -- Simulate file change detected
    print("\n[HotReload] File change detected for 'greeting'")
    currentPlugin = pluginV2
    
    print("New version:", currentPlugin.version)
    print("Greeting:", currentPlugin:greet("Alice"))
end

simulateHotReload()
```

---

## 7. Plugin Sandboxing

```lua
-- Plugin Sandbox: จำกัดสิทธิ์ของ plugin
local Sandbox = {}

function Sandbox.create(permissions)
    permissions = permissions or {}
    
    -- Create restricted environment
    local env = {
        -- Safe standard functions
        print = print,
        tostring = tostring,
        tonumber = tonumber,
        type = type,
        pairs = pairs,
        ipairs = ipairs,
        next = next,
        select = select,
        unpack = table.unpack,
        pcall = pcall,
        xpcall = xpcall,
        error = error,
        assert = assert,
        
        -- Safe libraries
        string = {
            format = string.format,
            len = string.len,
            sub = string.sub,
            find = string.find,
            match = string.match,
            gmatch = string.gmatch,
            gsub = string.gsub,
            upper = string.upper,
            lower = string.lower,
            rep = string.rep,
            byte = string.byte,
            char = string.char
        },
        
        table = {
            insert = table.insert,
            remove = table.remove,
            concat = table.concat,
            sort = table.sort,
            unpack = table.unpack,
            move = table.move
        },
        
        math = math,
        
        -- Conditional: io (only if permitted)
        io = permissions.allowIO and io or nil,
        
        -- Conditional: os (restricted)
        os = permissions.allowOS and {
            time = os.time,
            date = os.date,
            clock = os.clock
        } or nil,
        
        -- No: load, loadfile, dofile, require (dangerous)
        -- No: debug (can escape sandbox)
        -- No: package (can require anything)
    }
    
    -- Add allowed requires
    if permissions.allow then
        env.require = function(moduleName)
            -- Check if module is in allowed list
            local allowed = false
            for _, allowed_module in ipairs(permissions.allow) do
                if moduleName == allowed_module or moduleName:match("^" .. allowed_module) then
                    allowed = true
                    break
                end
            end
            
            if not allowed then
                error(string.format("Plugin not allowed to require '%s'", moduleName))
            end
            
            return require(moduleName)
        end
    end
    
    return env
end

function Sandbox.run(code, env, name)
    name = name or "sandboxed_plugin"
    
    local fn, err = load(code, "@" .. name, "t", env)
    if not fn then
        error("Sandbox compile error: " .. err)
    end
    
    local ok, result = pcall(fn)
    if not ok then
        error("Sandbox runtime error: " .. result)
    end
    
    return result
end

-- Test sandboxing
local trustedCode = [[
local result = {}
for i = 1, 5 do
    table.insert(result, i * i)
end
return result
]]

local untrustedCode = [[
-- นี่คือ malicious code ที่ plugin อาจพยายาม
return function()
    -- loadfile("dangerous.lua")  -- blocked!
    -- require("socket")  -- blocked!
    os.execute("rm -rf /")  -- blocked! (os not available)
end
]]

local safeEnv = Sandbox.create({
    allowIO = false,
    allowOS = true,  -- only safe os functions
    allow = { "json" }
})

local ok, result = pcall(function()
    return Sandbox.run(trustedCode, safeEnv, "trusted_plugin")
end)

if ok then
    print("Safe plugin result:", table.concat(result, ", "))
end

-- Test blocked operation
local ok2, err2 = pcall(function()
    Sandbox.run("os.execute('ls')", safeEnv, "malicious_plugin")
end)
print("Blocked dangerous code:", not ok2)
if not ok2 then
    print("Error:", err2)
end
```

---

## 8. Hooks และ Extension Points

```lua
-- Hook System - extension points สำหรับ plugins
local HookSystem = {}
HookSystem.__index = HookSystem

function HookSystem.new()
    return setmetatable({
        _hooks = {},
        _filters = {},
        _priority = {}
    }, HookSystem)
end

-- Action hooks (ทำ side effects)
function HookSystem:addAction(hookName, fn, priority)
    priority = priority or 10
    
    if not self._hooks[hookName] then
        self._hooks[hookName] = {}
    end
    
    table.insert(self._hooks[hookName], {
        fn = fn,
        priority = priority
    })
    
    -- Sort by priority
    table.sort(self._hooks[hookName], function(a, b)
        return a.priority < b.priority
    end)
    
    return function()
        -- Remove this hook
        for i, hook in ipairs(self._hooks[hookName]) do
            if hook.fn == fn then
                table.remove(self._hooks[hookName], i)
                return
            end
        end
    end
end

function HookSystem:doAction(hookName, ...)
    if not self._hooks[hookName] then return end
    
    for _, hook in ipairs(self._hooks[hookName]) do
        local ok, err = pcall(hook.fn, ...)
        if not ok then
            print(string.format("[Hook] Error in action '%s': %s", hookName, err))
        end
    end
end

-- Filter hooks (transform data)
function HookSystem:addFilter(filterName, fn, priority)
    priority = priority or 10
    
    if not self._filters[filterName] then
        self._filters[filterName] = {}
    end
    
    table.insert(self._filters[filterName], {
        fn = fn,
        priority = priority
    })
    
    table.sort(self._filters[filterName], function(a, b)
        return a.priority < b.priority
    end)
end

function HookSystem:applyFilter(filterName, value, ...)
    if not self._filters[filterName] then
        return value
    end
    
    for _, filter in ipairs(self._filters[filterName]) do
        local ok, result = pcall(filter.fn, value, ...)
        if ok then
            value = result
        else
            print(string.format("[Hook] Error in filter '%s': %s", filterName, result))
        end
    end
    
    return value
end

-- WordPress-style hooks
local hooks = HookSystem.new()

-- Register hooks from "plugins"
hooks:addAction("app.start", function(app)
    print("[Hook] App starting, version:", app.version)
end, 5)

hooks:addAction("app.start", function(app)
    print("[Hook] Checking dependencies...")
end, 10)

hooks:addAction("request.received", function(req)
    print("[Hook] Request received:", req.method, req.path)
end)

-- Register filters
hooks:addFilter("response.body", function(body)
    return body .. "\n<!-- generated by Lua -->"
end)

hooks:addFilter("user.name", function(name)
    return name:upper()
end)

-- Use hooks in application
local app = { version = "2.0.0" }
hooks:doAction("app.start", app)

local request = { method = "GET", path = "/api/users" }
hooks:doAction("request.received", request)

local responseBody = "<html><body>Hello</body></html>"
responseBody = hooks:applyFilter("response.body", responseBody)
print("\nFiltered response body:")
print(responseBody)

local userName = "alice"
print("\nFiltered user name:", hooks:applyFilter("user.name", userName))
```

---

## 9. Plugin Registry

```lua
-- Plugin Registry: centralized plugin management
local Registry = {}
Registry.__index = Registry

function Registry.new()
    return setmetatable({
        _plugins = {},
        _instances = {},
        _hooks = {},
        _config = {}
    }, Registry)
end

function Registry:register(plugin, config)
    -- Validate plugin
    if not plugin.name then
        error("Plugin must have a name")
    end
    
    if self._plugins[plugin.name] then
        print(string.format("[Registry] Warning: Plugin '%s' already registered, updating",
            plugin.name))
    end
    
    self._plugins[plugin.name] = plugin
    self._config[plugin.name] = config or {}
    
    print(string.format("[Registry] Registered: %s v%s",
        plugin.name, plugin.version or "?"))
    
    return self
end

function Registry:unregister(pluginName)
    if self._instances[pluginName] then
        self._instances[pluginName]:stop()
        self._instances[pluginName] = nil
    end
    self._plugins[pluginName] = nil
    self._config[pluginName] = nil
    print(string.format("[Registry] Unregistered: %s", pluginName))
end

function Registry:get(pluginName)
    return self._plugins[pluginName]
end

function Registry:getInstance(pluginName)
    return self._instances[pluginName]
end

function Registry:update(pluginName, newPlugin)
    self._plugins[pluginName] = newPlugin
    -- Instance will be recreated on next start
    self._instances[pluginName] = nil
end

function Registry:loadAll(app)
    -- Resolve load order
    local loadOrder = {}
    local loaded = {}
    
    local function load(name)
        if loaded[name] then return end
        local plugin = self._plugins[name]
        if not plugin then
            error("Plugin not found: " .. name)
        end
        
        -- Load dependencies first
        for _, dep in ipairs(plugin.dependencies or {}) do
            load(dep)
        end
        
        table.insert(loadOrder, name)
        loaded[name] = true
    end
    
    for name, _ in pairs(self._plugins) do
        load(name)
    end
    
    -- Load in order
    for _, name in ipairs(loadOrder) do
        self:_loadPlugin(name, app)
    end
end

function Registry:_loadPlugin(pluginName, app)
    local plugin = self._plugins[pluginName]
    local config = self._config[pluginName]
    
    if not plugin then return end
    
    -- Create instance wrapper
    local instance = {
        plugin = plugin,
        config = config,
        _running = false,
        
        isRunning = function(self) return self._running end,
        
        stop = function(self)
            if plugin.stop then
                pcall(plugin.stop, plugin)
            end
            self._running = false
        end,
        
        start = function(self)
            if plugin.start then
                pcall(plugin.start, plugin, config)
            end
            self._running = true
        end
    }
    
    -- Initialize
    if plugin.init then
        local ok, err = pcall(plugin.init, plugin, app, config)
        if not ok then
            print(string.format("[Registry] Error initializing '%s': %s", pluginName, err))
            return
        end
    end
    
    -- Register hooks
    if plugin.hooks then
        for hookName, handler in pairs(plugin.hooks) do
            if not self._hooks[hookName] then
                self._hooks[hookName] = {}
            end
            table.insert(self._hooks[hookName], {
                plugin = pluginName,
                handler = handler
            })
        end
    end
    
    instance:start()
    self._instances[pluginName] = instance
end

function Registry:trigger(hookName, ...)
    local handlers = self._hooks[hookName]
    if not handlers then return end
    
    local results = {}
    for _, h in ipairs(handlers) do
        local ok, result = pcall(h.handler, ...)
        if ok and result ~= nil then
            table.insert(results, result)
        end
    end
    return results
end

function Registry:list()
    local list = {}
    for name, plugin in pairs(self._plugins) do
        local instance = self._instances[name]
        table.insert(list, {
            name = name,
            version = plugin.version or "?",
            running = instance and instance:isRunning() or false
        })
    end
    table.sort(list, function(a, b) return a.name < b.name end)
    return list
end

-- Test Registry
local registry = Registry.new()

-- Define some plugins
local authPlugin = {
    name = "auth",
    version = "1.0.0",
    dependencies = { "logger" },
    
    init = function(self, app, config)
        print("  [auth] Initializing with secret:", config.secret and "***" or "none")
    end,
    
    start = function(self, config)
        print("  [auth] Started")
    end,
    
    hooks = {
        beforeRequest = function(ctx)
            print("  [auth] Checking authentication")
            ctx.authenticated = true
        end
    }
}

local loggerPlugin = {
    name = "logger",
    version = "1.0.0",
    dependencies = {},
    
    init = function(self, app, config)
        print("  [logger] Initializing")
    end,
    
    start = function(self, config)
        print("  [logger] Started")
    end,
    
    hooks = {
        afterRequest = function(ctx)
            print("  [logger] Logging request:", ctx.path or "unknown")
        end
    }
}

registry:register(loggerPlugin)
registry:register(authPlugin, { secret = "my-secret-key" })

local app = { version = "1.0.0" }
registry:loadAll(app)

print("\nRegistered plugins:")
for _, p in ipairs(registry:list()) do
    print(string.format("  %s v%s - %s", p.name, p.version, p.running and "running" or "stopped"))
end

-- Trigger hooks
print("\nTriggering hooks:")
registry:trigger("beforeRequest", { path = "/api/users" })
registry:trigger("afterRequest", { path = "/api/users" })
```

---

## 10. Version Compatibility

```lua
-- Version Compatibility Checking
local VersionCheck = {}

function VersionCheck.parse(versionStr)
    local major, minor, patch, pre = versionStr:match("^(%d+)%.(%d+)%.(%d+)%-?(.*)$")
    if not major then
        major, minor = versionStr:match("^(%d+)%.(%d+)$")
        patch = "0"
    end
    
    return {
        major = tonumber(major) or 0,
        minor = tonumber(minor) or 0,
        patch = tonumber(patch) or 0,
        pre = pre or ""
    }
end

function VersionCheck.compare(v1str, v2str)
    local v1 = VersionCheck.parse(v1str)
    local v2 = VersionCheck.parse(v2str)
    
    if v1.major ~= v2.major then return v1.major - v2.major end
    if v1.minor ~= v2.minor then return v1.minor - v2.minor end
    if v1.patch ~= v2.patch then return v1.patch - v2.patch end
    
    -- Pre-release versions are lower than release
    if v1.pre == "" and v2.pre ~= "" then return 1 end
    if v1.pre ~= "" and v2.pre == "" then return -1 end
    
    return 0
end

function VersionCheck.satisfies(version, constraint)
    -- Parse constraint like ">=1.0.0", "^2.0.0", "~1.2.0", "1.x"
    local op, constraintVersion = constraint:match("^([><=~^]+)(.+)$")
    
    if not op then
        -- Exact match
        return VersionCheck.compare(version, constraint) == 0
    end
    
    local cmp = VersionCheck.compare(version, constraintVersion)
    
    if op == "=" or op == "==" then return cmp == 0
    elseif op == ">" then return cmp > 0
    elseif op == ">=" then return cmp >= 0
    elseif op == "<" then return cmp < 0
    elseif op == "<=" then return cmp <= 0
    elseif op == "~" then
        -- Compatible: same major.minor, patch can vary
        local v = VersionCheck.parse(version)
        local cv = VersionCheck.parse(constraintVersion)
        return v.major == cv.major and v.minor == cv.minor and v.patch >= cv.patch
    elseif op == "^" then
        -- Compatible: same major, minor.patch can vary upward
        local v = VersionCheck.parse(version)
        local cv = VersionCheck.parse(constraintVersion)
        return v.major == cv.major and 
               (v.minor > cv.minor or (v.minor == cv.minor and v.patch >= cv.patch))
    end
    
    return false
end

-- Test version checking
local testCases = {
    { version = "2.1.0", constraint = ">=2.0.0", expected = true },
    { version = "1.9.0", constraint = ">=2.0.0", expected = false },
    { version = "2.1.0", constraint = "^2.0.0", expected = true },
    { version = "3.0.0", constraint = "^2.0.0", expected = false },
    { version = "1.2.3", constraint = "~1.2.0", expected = true },
    { version = "1.3.0", constraint = "~1.2.0", expected = false },
    { version = "2.0.0", constraint = "=2.0.0", expected = true },
    { version = "2.0.1", constraint = "=2.0.0", expected = false },
}

print("Version compatibility tests:")
local allPassed = true
for _, tc in ipairs(testCases) do
    local result = VersionCheck.satisfies(tc.version, tc.constraint)
    local status = result == tc.expected and "PASS" or "FAIL"
    if status == "FAIL" then allPassed = false end
    print(string.format("  %s: %s satisfies %s -> %s (expected %s)",
        status, tc.version, tc.constraint, tostring(result), tostring(tc.expected)))
end

print("All tests passed:", allPassed)
```

---

## 11. Plugin Configuration

```lua
-- Plugin Configuration System
local PluginConfig = {}
PluginConfig.__index = PluginConfig

function PluginConfig.new(schema, defaults)
    return setmetatable({
        _schema = schema or {},
        _defaults = defaults or {},
        _values = {},
        _errors = {}
    }, PluginConfig)
end

function PluginConfig:load(data)
    self._values = {}
    
    -- Apply defaults first
    for key, default in pairs(self._defaults) do
        self._values[key] = default
    end
    
    -- Apply provided values
    if data then
        for key, value in pairs(data) do
            self._values[key] = value
        end
    end
    
    -- Validate against schema
    self._errors = {}
    for key, schema in pairs(self._schema) do
        local value = self._values[key]
        
        if schema.required and value == nil then
            table.insert(self._errors, key .. " is required")
        end
        
        if value ~= nil then
            if schema.type and type(value) ~= schema.type then
                table.insert(self._errors, string.format(
                    "%s must be %s, got %s", key, schema.type, type(value)))
            end
            
            if schema.min and type(value) == "number" and value < schema.min then
                table.insert(self._errors, key .. " must be >= " .. schema.min)
            end
            
            if schema.max and type(value) == "number" and value > schema.max then
                table.insert(self._errors, key .. " must be <= " .. schema.max)
            end
            
            if schema.enum then
                local valid = false
                for _, allowed in ipairs(schema.enum) do
                    if value == allowed then valid = true; break end
                end
                if not valid then
                    table.insert(self._errors, string.format(
                        "%s must be one of: %s",
                        key, table.concat(schema.enum, ", ")))
                end
            end
        end
    end
    
    return #self._errors == 0, self._errors
end

function PluginConfig:get(key, default)
    local value = self._values[key]
    return value ~= nil and value or default
end

function PluginConfig:set(key, value)
    self._values[key] = value
    return self
end

function PluginConfig:isValid()
    return #self._errors == 0
end

function PluginConfig:errors()
    return self._errors
end

-- Example: Rate Limiter Plugin Config
local rateLimiterConfig = PluginConfig.new(
    -- Schema
    {
        maxRequests = { type = "number", required = true, min = 1, max = 10000 },
        windowMs = { type = "number", required = true, min = 100 },
        strategy = { type = "string", enum = { "fixed", "sliding", "token-bucket" } },
        keyFn = { type = "function" },
        skipFn = { type = "function" }
    },
    -- Defaults
    {
        maxRequests = 100,
        windowMs = 60000,  -- 1 minute
        strategy = "fixed"
    }
)

-- Load valid config
local ok, errors = rateLimiterConfig:load({
    maxRequests = 50,
    windowMs = 30000,
    strategy = "sliding"
})

print("Config valid:", ok)
print("maxRequests:", rateLimiterConfig:get("maxRequests"))
print("strategy:", rateLimiterConfig:get("strategy"))

-- Load invalid config
local ok2, errors2 = rateLimiterConfig:load({
    maxRequests = "fifty",  -- wrong type
    strategy = "unknown"    -- not in enum
})

print("\nInvalid config:", not ok2)
for _, err in ipairs(errors2) do
    print("  Error:", err)
end
```

---

## 12. Plugin Testing Framework

```lua
-- Plugin Testing Utilities
local PluginTester = {}
PluginTester.__index = PluginTester

function PluginTester.new()
    return setmetatable({
        _results = { passed = 0, failed = 0, errors = {} }
    }, PluginTester)
end

function PluginTester:createMockApp(overrides)
    local app = {
        version = "1.0.0",
        name = "TestApp",
        hooks = {},
        
        on = function(self, event, handler)
            if not self.hooks[event] then self.hooks[event] = {} end
            table.insert(self.hooks[event], handler)
        end,
        
        emit = function(self, event, ...)
            if self.hooks[event] then
                for _, h in ipairs(self.hooks[event]) do
                    h(...)
                end
            end
        end
    }
    
    -- Apply overrides
    if overrides then
        for k, v in pairs(overrides) do
            app[k] = v
        end
    end
    
    return app
end

function PluginTester:createMockRequest(overrides)
    local req = {
        method = "GET",
        path = "/test",
        headers = {},
        query = {},
        body = nil,
        params = {}
    }
    
    if overrides then
        for k, v in pairs(overrides) do
            req[k] = v
        end
    end
    
    return req
end

function PluginTester:test(name, fn)
    local ok, err = pcall(fn, self)
    
    if ok then
        self._results.passed = self._results.passed + 1
        print(string.format("  PASS: %s", name))
    else
        self._results.failed = self._results.failed + 1
        table.insert(self._results.errors, { test = name, error = err })
        print(string.format("  FAIL: %s", name))
        print(string.format("    Error: %s", err))
    end
end

function PluginTester:assert(condition, message)
    if not condition then
        error(message or "Assertion failed", 2)
    end
end

function PluginTester:assertEqual(a, b, message)
    if a ~= b then
        error(string.format("%s: expected %s, got %s",
            message or "assertEqual failed", tostring(b), tostring(a)), 2)
    end
end

function PluginTester:report()
    print(string.format("\nPlugin Test Results: %d passed, %d failed",
        self._results.passed, self._results.failed))
    
    if #self._results.errors > 0 then
        print("Failed tests:")
        for _, err in ipairs(self._results.errors) do
            print(string.format("  - %s: %s", err.test, err.error))
        end
    end
    
    return self._results.failed == 0
end

-- Test a plugin
local function testAuthPlugin()
    local tester = PluginTester.new()
    
    -- The plugin to test
    local authPlugin = {
        name = "auth",
        
        init = function(self, app, config)
            self._secret = config.secret
            self._tokens = {}
        end,
        
        generateToken = function(self, userId)
            local token = "token_" .. userId .. "_" .. os.time()
            self._tokens[token] = { userId = userId, createdAt = os.time() }
            return token
        end,
        
        verify = function(self, token)
            return self._tokens[token] ~= nil
        end,
        
        getUserId = function(self, token)
            local data = self._tokens[token]
            return data and data.userId or nil
        end
    }
    
    print("Testing auth plugin:")
    
    tester:test("plugin has required name", function(t)
        t:assert(authPlugin.name ~= nil, "Missing name")
        t:assertEqual(authPlugin.name, "auth")
    end)
    
    tester:test("init sets secret", function(t)
        local app = tester:createMockApp()
        authPlugin:init(app, { secret = "test-secret" })
        t:assertEqual(authPlugin._secret, "test-secret")
    end)
    
    tester:test("generates valid token", function(t)
        local token = authPlugin:generateToken(42)
        t:assert(token ~= nil, "Token should not be nil")
        t:assert(type(token) == "string", "Token should be string")
        t:assert(token:match("^token_"), "Token should start with 'token_'")
    end)
    
    tester:test("verifies valid token", function(t)
        local token = authPlugin:generateToken(99)
        t:assert(authPlugin:verify(token), "Should verify valid token")
    end)
    
    tester:test("rejects invalid token", function(t)
        t:assert(not authPlugin:verify("invalid-token"), "Should reject invalid token")
    end)
    
    tester:test("returns correct user id", function(t)
        local token = authPlugin:generateToken(123)
        t:assertEqual(authPlugin:getUserId(token), 123)
    end)
    
    tester:test("returns nil for invalid token user id", function(t)
        local userId = authPlugin:getUserId("fake-token")
        t:assert(userId == nil, "Should return nil for invalid token")
    end)
    
    return tester:report()
end

testAuthPlugin()
```

---

## 13. Web Framework Plugin System

```lua
-- Mini Web Framework Plugin System
local WebFramework = {}
WebFramework.__index = WebFramework

function WebFramework.new(config)
    local fw = setmetatable({
        _config = config or {},
        _plugins = {},
        _middleware = {},
        _routes = {},
        _hooks = {}
    }, WebFramework)
    
    return fw
end

function WebFramework:use(plugin, config)
    -- Install plugin
    table.insert(self._plugins, {
        plugin = plugin,
        config = config or {}
    })
    
    -- Initialize plugin
    if plugin.install then
        plugin:install(self, config or {})
    end
    
    -- Register middleware
    if plugin.middleware then
        for _, mw in ipairs(plugin.middleware) do
            table.insert(self._middleware, mw)
        end
    end
    
    -- Register routes
    if plugin.routes then
        for path, handler in pairs(plugin.routes) do
            self._routes[path] = handler
        end
    end
    
    -- Register hooks
    if plugin.hooks then
        for hookName, fn in pairs(plugin.hooks) do
            if not self._hooks[hookName] then
                self._hooks[hookName] = {}
            end
            table.insert(self._hooks[hookName], fn)
        end
    end
    
    print(string.format("[Framework] Plugin '%s' installed", plugin.name))
    return self
end

function WebFramework:hook(name, fn)
    if not self._hooks[name] then self._hooks[name] = {} end
    table.insert(self._hooks[name], fn)
end

function WebFramework:_triggerHooks(name, ctx)
    if not self._hooks[name] then return ctx end
    
    for _, fn in ipairs(self._hooks[name]) do
        local ok, result = pcall(fn, ctx)
        if not ok then
            print(string.format("[Framework] Hook error '%s': %s", name, result))
        elseif result ~= nil then
            ctx = result
        end
    end
    
    return ctx
end

function WebFramework:handle(request)
    local ctx = {
        request = request,
        response = {
            status = 200,
            headers = { ["Content-Type"] = "application/json" },
            body = ""
        },
        state = {}
    }
    
    -- Run middleware chain
    local middlewareIndex = 0
    
    local function next()
        middlewareIndex = middlewareIndex + 1
        local mw = self._middleware[middlewareIndex]
        if mw then
            local ok, err = pcall(mw, ctx, next)
            if not ok then
                ctx.response.status = 500
                ctx.response.body = '{"error":"Middleware error"}'
                print("[Framework] Middleware error:", err)
            end
        else
            -- Run route handler
            local handler = self._routes[request.path]
            if handler then
                handler(ctx)
            else
                ctx.response.status = 404
                ctx.response.body = '{"error":"Not Found"}'
            end
        end
    end
    
    -- Before hooks
    ctx = self:_triggerHooks("beforeRequest", ctx)
    
    next()
    
    -- After hooks
    ctx = self:_triggerHooks("afterRequest", ctx)
    
    return ctx.response
end

-- Define plugins
local corsPlugin = {
    name = "cors",
    
    install = function(self, app, config)
        self._origins = config.origins or { "*" }
    end,
    
    middleware = {
        function(ctx, next)
            ctx.response.headers["Access-Control-Allow-Origin"] = "*"
            ctx.response.headers["Access-Control-Allow-Methods"] = "GET,POST,PUT,DELETE"
            next()
        end
    },
    
    hooks = {
        beforeRequest = function(ctx)
            print("  [CORS] Adding CORS headers")
            return ctx
        end
    }
}

local authMiddlewarePlugin = {
    name = "auth-middleware",
    
    install = function(self, app, config)
        self._tokens = config.validTokens or {}
    end,
    
    middleware = {
        function(ctx, next)
            local token = ctx.request.headers["Authorization"]
            if token and token:match("^Bearer ") then
                token = token:sub(8)
                ctx.state.authenticated = true
                ctx.state.userId = 42  -- normally decoded from JWT
                print("  [Auth] Request authenticated, userId=42")
            else
                print("  [Auth] No authentication token")
                ctx.state.authenticated = false
            end
            next()
        end
    }
}

local loggerPlugin = {
    name = "request-logger",
    
    hooks = {
        beforeRequest = function(ctx)
            print(string.format("  [Logger] --> %s %s",
                ctx.request.method, ctx.request.path))
            ctx._startTime = os.clock()
            return ctx
        end,
        afterRequest = function(ctx)
            local duration = os.clock() - (ctx._startTime or os.clock())
            print(string.format("  [Logger] <-- %d (%.4f ms)",
                ctx.response.status, duration * 1000))
            return ctx
        end
    }
}

-- Create framework and install plugins
local app = WebFramework.new({ port = 3000 })

app:use(loggerPlugin)
app:use(corsPlugin, { origins = { "https://myapp.com" } })
app:use(authMiddlewarePlugin, { validTokens = { "secret-token" } })

-- Define routes
app._routes["/api/users"] = function(ctx)
    if not ctx.state.authenticated then
        ctx.response.status = 401
        ctx.response.body = '{"error":"Unauthorized"}'
        return
    end
    ctx.response.body = '{"users":[{"id":1,"name":"Alice"}]}'
end

app._routes["/api/public"] = function(ctx)
    ctx.response.body = '{"message":"Public endpoint"}'
end

-- Handle requests
print("\n=== Request 1: Authenticated ===")
local r1 = app:handle({
    method = "GET",
    path = "/api/users",
    headers = { Authorization = "Bearer secret-token" }
})
print("Status:", r1.status)
print("Body:", r1.body)

print("\n=== Request 2: Unauthenticated ===")
local r2 = app:handle({
    method = "GET",
    path = "/api/users",
    headers = {}
})
print("Status:", r2.status)

print("\n=== Request 3: Public endpoint ===")
local r3 = app:handle({
    method = "GET",
    path = "/api/public",
    headers = {}
})
print("Status:", r3.status)
print("Body:", r3.body)
```

---

## 14. Editor Plugin System

```lua
-- Editor Plugin System - เหมือน VS Code extensions
local EditorPluginSystem = {}
EditorPluginSystem.__index = EditorPluginSystem

function EditorPluginSystem.new()
    local editor = setmetatable({
        _plugins = {},
        _commands = {},
        _keybindings = {},
        _completionProviders = {},
        _diagnosticProviders = {},
        _themeProviders = {},
        _statusBarItems = {},
        _events = {}
    }, EditorPluginSystem)
    
    return editor
end

-- Extension API
function EditorPluginSystem:registerCommand(name, fn, description)
    self._commands[name] = { fn = fn, description = description or "" }
    return function() end  -- Returns disposable
end

function EditorPluginSystem:registerKeybinding(key, commandName)
    self._keybindings[key] = commandName
    return function() end
end

function EditorPluginSystem:registerCompletionProvider(pattern, provider)
    table.insert(self._completionProviders, {
        pattern = pattern,
        provider = provider
    })
end

function EditorPluginSystem:executeCommand(name, ...)
    local cmd = self._commands[name]
    if cmd then
        return cmd.fn(...)
    end
    error("Unknown command: " .. name)
end

function EditorPluginSystem:emit(event, data)
    if self._events[event] then
        for _, handler in ipairs(self._events[event]) do
            handler(data)
        end
    end
end

function EditorPluginSystem:on(event, handler)
    if not self._events[event] then self._events[event] = {} end
    table.insert(self._events[event], handler)
    
    -- Return disposable
    return function()
        for i, h in ipairs(self._events[event]) do
            if h == handler then
                table.remove(self._events[event], i)
                return
            end
        end
    end
end

function EditorPluginSystem:installPlugin(plugin)
    if not plugin.activate then
        error("Plugin must have an activate function")
    end
    
    -- Create plugin context (limited API)
    local context = {
        subscriptions = {},
        
        registerCommand = function(name, fn)
            local disposable = self:registerCommand(name, fn)
            table.insert(context.subscriptions, disposable)
        end,
        
        registerKeybinding = function(key, cmd)
            local disposable = self:registerKeybinding(key, cmd)
            table.insert(context.subscriptions, disposable)
        end,
        
        onEvent = function(event, handler)
            local disposable = self:on(event, handler)
            table.insert(context.subscriptions, disposable)
        end,
        
        showMessage = function(msg)
            print("[Editor] Message:", msg)
        end
    }
    
    -- Activate the plugin
    local ok, err = pcall(plugin.activate, plugin, context)
    if not ok then
        print(string.format("[Editor] Plugin '%s' activation failed: %s",
            plugin.name or "?", err))
        return false
    end
    
    self._plugins[plugin.name] = {
        plugin = plugin,
        context = context
    }
    
    print(string.format("[Editor] Plugin '%s' activated", plugin.name))
    return true
end

function EditorPluginSystem:deactivatePlugin(pluginName)
    local entry = self._plugins[pluginName]
    if not entry then return end
    
    -- Call deactivate
    if entry.plugin.deactivate then
        pcall(entry.plugin.deactivate, entry.plugin)
    end
    
    -- Cleanup subscriptions
    for _, disposable in ipairs(entry.context.subscriptions) do
        if type(disposable) == "function" then
            pcall(disposable)
        end
    end
    
    self._plugins[pluginName] = nil
    print(string.format("[Editor] Plugin '%s' deactivated", pluginName))
end

-- Example plugins
local luaFormatterPlugin = {
    name = "lua-formatter",
    displayName = "Lua Formatter",
    
    activate = function(self, context)
        context.registerCommand("lua.format", function(document)
            print("  [LuaFormatter] Formatting document:", document)
            return document  -- would return formatted code
        end)
        
        context.registerKeybinding("ctrl+shift+f", "lua.format")
        
        context.onEvent("documentSaved", function(doc)
            print("  [LuaFormatter] Auto-formatting on save:", doc)
        end)
        
        context.showMessage("Lua Formatter activated!")
    end,
    
    deactivate = function(self)
        print("  [LuaFormatter] Deactivating")
    end
}

local gitIntegrationPlugin = {
    name = "git-integration",
    
    activate = function(self, context)
        context.registerCommand("git.status", function()
            print("  [Git] git status")
            return { modified = 3, staged = 1, untracked = 2 }
        end)
        
        context.registerCommand("git.commit", function(message)
            print("  [Git] Committing:", message)
            return true
        end)
        
        context.registerKeybinding("ctrl+shift+g", "git.status")
        
        context.showMessage("Git Integration ready")
    end
}

-- Test editor plugin system
local editor = EditorPluginSystem.new()

editor:installPlugin(luaFormatterPlugin)
editor:installPlugin(gitIntegrationPlugin)

print("\nAvailable commands:")
for name, cmd in pairs(editor._commands) do
    print(string.format("  %s: %s", name, cmd.description or "no description"))
end

print("\nExecuting commands:")
editor:executeCommand("lua.format", "main.lua")

local status = editor:executeCommand("git.status")
print("Git status - modified:", status.modified)

-- Simulate events
print("\nSimulating events:")
editor:emit("documentSaved", "main.lua")

-- Deactivate plugin
editor:deactivatePlugin("lua-formatter")
```

---

## 15. Plugin Communication

```lua
-- Plugin-to-Plugin Communication via Events
local EventBus = {}
EventBus.__index = EventBus

local _globalBus = nil

function EventBus.getInstance()
    if not _globalBus then
        _globalBus = setmetatable({
            _channels = {},
            _history = {}
        }, EventBus)
    end
    return _globalBus
end

function EventBus:publish(channel, message, sender)
    if not self._channels[channel] then
        self._channels[channel] = {}
    end
    
    local envelope = {
        channel = channel,
        message = message,
        sender = sender or "unknown",
        timestamp = os.time(),
        id = tostring(os.time()) .. "_" .. math.random(1000, 9999)
    }
    
    -- Store in history
    table.insert(self._history, envelope)
    if #self._history > 1000 then
        table.remove(self._history, 1)
    end
    
    -- Deliver to subscribers
    local count = 0
    for _, subscriber in ipairs(self._channels[channel]) do
        local ok, err = pcall(subscriber.handler, envelope)
        if not ok then
            print(string.format("[EventBus] Error delivering to '%s': %s",
                subscriber.name, err))
        end
        count = count + 1
    end
    
    return count
end

function EventBus:subscribe(channel, handler, subscriberName)
    if not self._channels[channel] then
        self._channels[channel] = {}
    end
    
    local sub = {
        name = subscriberName or "anonymous",
        handler = handler,
        channel = channel
    }
    
    table.insert(self._channels[channel], sub)
    
    -- Return unsubscribe function
    return function()
        for i, s in ipairs(self._channels[channel]) do
            if s == sub then
                table.remove(self._channels[channel], i)
                return
            end
        end
    end
end

-- Plugins that communicate via EventBus
local orderPlugin = {
    name = "orders",
    
    init = function(self)
        local bus = EventBus.getInstance()
        
        -- Subscribe to payment events
        self._unsubPayment = bus:subscribe("payment.completed", function(envelope)
            print(string.format("  [Orders] Payment received for order %s - fulfilling",
                tostring(envelope.message.orderId)))
        end, "orders-plugin")
        
        print("  [Orders] Plugin initialized, listening for payments")
    end,
    
    createOrder = function(self, item, price)
        local orderId = math.random(10000, 99999)
        local bus = EventBus.getInstance()
        
        bus:publish("order.created", {
            orderId = orderId,
            item = item,
            price = price
        }, self.name)
        
        return orderId
    end
}

local inventoryPlugin = {
    name = "inventory",
    
    init = function(self)
        local bus = EventBus.getInstance()
        
        bus:subscribe("order.created", function(envelope)
            local data = envelope.message
            print(string.format("  [Inventory] Checking stock for order %d: %s",
                data.orderId, data.item))
        end, "inventory-plugin")
        
        print("  [Inventory] Plugin initialized, listening for orders")
    end
}

local notificationPlugin = {
    name = "notifications",
    
    init = function(self)
        local bus = EventBus.getInstance()
        
        bus:subscribe("order.created", function(envelope)
            print(string.format("  [Notifications] Sending order confirmation for #%d",
                envelope.message.orderId))
        end, "notifications-plugin")
        
        bus:subscribe("payment.completed", function(envelope)
            print(string.format("  [Notifications] Sending payment receipt for order #%s",
                tostring(envelope.message.orderId)))
        end, "notifications-plugin")
        
        print("  [Notifications] Plugin initialized")
    end
}

-- Initialize plugins
print("Initializing plugins:")
orderPlugin:init()
inventoryPlugin:init()
notificationPlugin:init()

-- Simulate order flow
print("\nCreating order:")
local orderId = orderPlugin:createOrder("Laptop", 35000)

print("\nProcessing payment:")
local bus = EventBus.getInstance()
bus:publish("payment.completed", {
    orderId = orderId,
    amount = 35000,
    method = "credit_card"
}, "payment-gateway")

print("\nEvent bus history entries:", #bus._history)
```

---

## สรุป

บทนี้ครอบคลุม:

1. **ทำไมต้องมี Plugin Architecture** - Open/Closed Principle
2. **Plugin Discovery** - Directory scanning และ dynamic loading
3. **Plugin Manifest** - Metadata, version, dependencies
4. **Plugin Lifecycle** - init → start → running → stop → destroy
5. **Plugin Dependencies** - Dependency resolution with topological sort
6. **Hot Reload** - Reload plugins without restart
7. **Plugin Sandboxing** - จำกัดสิทธิ์ plugin code
8. **Hooks System** - Action hooks และ Filter hooks
9. **Plugin Registry** - Centralized management
10. **Version Compatibility** - Semver checking
11. **Plugin Configuration** - Schema-based config validation
12. **Plugin Testing** - Test utilities สำหรับ plugins
13. **Web Framework Plugin** - Real-world web framework example
14. **Editor Plugin System** - VS Code-style extensions
15. **Plugin Communication** - EventBus สำหรับ inter-plugin messaging

Plugin System ช่วยให้ application มีความยืดหยุ่นสูง รองรับ extension โดยไม่ต้องแก้ core code และเหมาะกับ application ที่ต้องการ customization หลากหลาย
