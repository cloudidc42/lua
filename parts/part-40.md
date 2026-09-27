# บทที่ 40: Configuration Management (การจัดการ Configuration)

## บทนำ

Configuration Management เป็นหนึ่งในแง่มุมสำคัญของ software development ที่ดี การแยก configuration ออกจาก code ทำให้ application สามารถทำงานในหลาย environments ได้โดยไม่ต้องแก้ไข code

**12-Factor App Principle #3:** "Store config in the environment" - config ควรอยู่ใน environment ไม่ใช่ใน code

## 40.1 Config File Formats

### INI Format Parser

```lua
-- ตัวอย่างที่ 1: INI Parser
print("=== INI Format Parser ===")

local INIParser = {}
INIParser.__index = INIParser

function INIParser.new()
    return setmetatable({}, INIParser)
end

function INIParser:parse(content)
    local config = {_default = {}}
    local currentSection = "_default"
    
    for line in (content .. "\n"):gmatch("([^\n]*)\n") do
        -- Remove comments
        line = line:match("^(.-)%s*[;#].*$") or line
        line = line:match("^%s*(.-)%s*$")  -- trim whitespace
        
        if line == "" then
            -- Skip empty lines
        elseif line:match("^%[(.-)%]$") then
            -- Section header
            currentSection = line:match("^%[(.-)%]$")
            if not config[currentSection] then
                config[currentSection] = {}
            end
        elseif line:match("^(.-)%s*=%s*(.*)$") then
            -- Key-value pair
            local key, value = line:match("^(.-)%s*=%s*(.*)$")
            key = key:match("^%s*(.-)%s*$")
            value = value:match("^%s*(.-)%s*$")
            
            -- Remove quotes
            if value:match('^"(.*)"$') then
                value = value:match('^"(.*)"$')
            elseif value:match("^'(.*)'$") then
                value = value:match("^'(.*)'$")
            end
            
            -- Type conversion
            if value == "true" then value = true
            elseif value == "false" then value = false
            elseif value == "null" or value == "nil" then value = nil
            elseif tonumber(value) then value = tonumber(value)
            end
            
            config[currentSection][key] = value
        end
    end
    
    return config
end

function INIParser:serialize(config)
    local lines = {}
    
    -- Default section first (without header)
    if config._default then
        for k, v in pairs(config._default) do
            table.insert(lines, string.format("%s = %s", k, tostring(v)))
        end
        if next(config._default) then table.insert(lines, "") end
    end
    
    -- Other sections
    for section, values in pairs(config) do
        if section ~= "_default" then
            table.insert(lines, string.format("[%s]", section))
            for k, v in pairs(values) do
                if type(v) == "string" then
                    table.insert(lines, string.format('%s = "%s"', k, v))
                else
                    table.insert(lines, string.format("%s = %s", k, tostring(v)))
                end
            end
            table.insert(lines, "")
        end
    end
    
    return table.concat(lines, "\n")
end

-- ทดสอบ
local iniContent = [[
; Application Configuration
app_name = MyApp
version = 1.0.0

[database]
host = localhost
port = 5432
name = myapp_db
user = admin
password = "secret123"
pool_size = 10

[server]
host = 0.0.0.0
port = 8080
debug = true
workers = 4

[cache]
driver = redis
host = localhost
port = 6379
ttl = 3600
enabled = true
]]

local parser = INIParser.new()
local config = parser:parse(iniContent)

print(string.format("App: %s v%s", config._default.app_name, config._default.version))
print(string.format("DB: %s@%s:%s/%s",
    config.database.user, config.database.host,
    config.database.port, config.database.name))
print(string.format("Server: %s:%s (debug=%s)",
    config.server.host, config.server.port, tostring(config.server.debug)))
print(string.format("Cache: %s enabled=%s",
    config.cache.driver, tostring(config.cache.enabled)))

-- Round-trip test
local serialized = parser:serialize(config)
print("\nSerialized:")
print(serialized)
```

### JSON Config Parser (Simplified)

```lua
-- ตัวอย่างที่ 2: Simple JSON Config Parser
print("=== JSON Config Parser ===")

local JSONConfig = {}
JSONConfig.__index = JSONConfig

-- Simplified JSON parser (handles common config patterns)
function JSONConfig.parse(json)
    local pos = 1
    
    local function skipWhitespace()
        while pos <= #json and json:sub(pos, pos):match("%s") do
            pos = pos + 1
        end
    end
    
    local function parseValue()
        skipWhitespace()
        if pos > #json then return nil end
        
        local char = json:sub(pos, pos)
        
        if char == '"' then
            -- String
            pos = pos + 1
            local start = pos
            local result = ""
            while pos <= #json do
                local c = json:sub(pos, pos)
                if c == '"' then
                    pos = pos + 1
                    return result
                elseif c == "\\" then
                    pos = pos + 1
                    local escape = json:sub(pos, pos)
                    if escape == "n" then result = result .. "\n"
                    elseif escape == "t" then result = result .. "\t"
                    elseif escape == "r" then result = result .. "\r"
                    else result = result .. escape end
                else
                    result = result .. c
                end
                pos = pos + 1
            end
            return result
            
        elseif char == "{" then
            -- Object
            pos = pos + 1
            local obj = {}
            skipWhitespace()
            if json:sub(pos, pos) == "}" then
                pos = pos + 1
                return obj
            end
            while pos <= #json do
                skipWhitespace()
                local key = parseValue()
                skipWhitespace()
                pos = pos + 1  -- skip ':'
                local value = parseValue()
                obj[key] = value
                skipWhitespace()
                if json:sub(pos, pos) == "}" then
                    pos = pos + 1
                    return obj
                elseif json:sub(pos, pos) == "," then
                    pos = pos + 1
                end
            end
            return obj
            
        elseif char == "[" then
            -- Array
            pos = pos + 1
            local arr = {}
            skipWhitespace()
            if json:sub(pos, pos) == "]" then
                pos = pos + 1
                return arr
            end
            while pos <= #json do
                table.insert(arr, parseValue())
                skipWhitespace()
                if json:sub(pos, pos) == "]" then
                    pos = pos + 1
                    return arr
                elseif json:sub(pos, pos) == "," then
                    pos = pos + 1
                end
            end
            return arr
            
        elseif json:sub(pos, pos + 3) == "true" then
            pos = pos + 4
            return true
        elseif json:sub(pos, pos + 4) == "false" then
            pos = pos + 5
            return false
        elseif json:sub(pos, pos + 3) == "null" then
            pos = pos + 4
            return nil
        else
            -- Number
            local numStr = json:match("^-?%d+%.?%d*[eE]?[+-]?%d*", pos)
            if numStr then
                pos = pos + #numStr
                return tonumber(numStr)
            end
        end
        
        return nil
    end
    
    return parseValue()
end

-- JSON serializer
function JSONConfig.serialize(value, indent, level)
    indent = indent or "  "
    level = level or 0
    local prefix = string.rep(indent, level)
    local childPrefix = string.rep(indent, level + 1)
    
    if type(value) == "nil" then
        return "null"
    elseif type(value) == "boolean" then
        return tostring(value)
    elseif type(value) == "number" then
        if value == math.floor(value) then
            return string.format("%d", value)
        else
            return string.format("%g", value)
        end
    elseif type(value) == "string" then
        return '"' .. value:gsub('"', '\\"'):gsub("\n", "\\n"):gsub("\t", "\\t") .. '"'
    elseif type(value) == "table" then
        -- Check if array
        local isArray = true
        local maxN = 0
        for k, _ in pairs(value) do
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                isArray = false
                break
            end
            maxN = math.max(maxN, k)
        end
        isArray = isArray and maxN == #value
        
        if isArray then
            local items = {}
            for _, v in ipairs(value) do
                table.insert(items, childPrefix .. JSONConfig.serialize(v, indent, level + 1))
            end
            if #items == 0 then return "[]" end
            return "[\n" .. table.concat(items, ",\n") .. "\n" .. prefix .. "]"
        else
            local items = {}
            local keys = {}
            for k, _ in pairs(value) do table.insert(keys, k) end
            table.sort(keys, function(a, b) return tostring(a) < tostring(b) end)
            for _, k in ipairs(keys) do
                local v = value[k]
                table.insert(items, childPrefix .. 
                    '"' .. tostring(k) .. '": ' .. JSONConfig.serialize(v, indent, level + 1))
            end
            if #items == 0 then return "{}" end
            return "{\n" .. table.concat(items, ",\n") .. "\n" .. prefix .. "}"
        end
    end
    return '"[unknown]"'
end

-- ทดสอบ
local jsonConfig = [[
{
  "app": {
    "name": "MyApp",
    "version": "2.0.0",
    "debug": false
  },
  "database": {
    "host": "localhost",
    "port": 5432,
    "name": "production_db",
    "pool": {
      "min": 2,
      "max": 10,
      "timeout": 30000
    }
  },
  "features": ["auth", "analytics", "notifications"],
  "maxRetries": 3
}
]]

local config = JSONConfig.parse(jsonConfig)
print(string.format("App: %s v%s", config.app.name, config.app.version))
print(string.format("DB Pool: min=%d max=%d", config.database.pool.min, config.database.pool.max))
print(string.format("Features: %s", table.concat(config.features, ", ")))

print("\nRe-serialized:")
print(JSONConfig.serialize(config))
```

### TOML-like Parser

```lua
-- ตัวอย่างที่ 3: TOML-like Parser
print("=== TOML-like Parser ===")

local TOMLParser = {}
TOMLParser.__index = TOMLParser

function TOMLParser.new()
    return setmetatable({}, TOMLParser)
end

function TOMLParser:parse(content)
    local result = {}
    local current = result
    local path = {}
    
    for line in (content .. "\n"):gmatch("([^\n]*)\n") do
        -- Remove inline comments
        local cleanLine = line:match("^(.-)%s*#.*$") or line
        cleanLine = cleanLine:match("^%s*(.-)%s*$")
        
        if cleanLine == "" then
            -- skip
        elseif cleanLine:match("^%[%[(.+)%]%]$") then
            -- Array of tables [[section]]
            local sectionPath = cleanLine:match("^%[%[(.+)%]%]$")
            local parts = {}
            for part in sectionPath:gmatch("[^.]+") do
                table.insert(parts, part)
            end
            -- Navigate/create path
            current = result
            for i = 1, #parts - 1 do
                if not current[parts[i]] then current[parts[i]] = {} end
                current = current[parts[i]]
            end
            local lastKey = parts[#parts]
            if not current[lastKey] then current[lastKey] = {} end
            local newTable = {}
            table.insert(current[lastKey], newTable)
            current = newTable
            
        elseif cleanLine:match("^%[(.+)%]$") then
            -- Section [section]
            local sectionPath = cleanLine:match("^%[(.+)%]$")
            current = result
            for part in sectionPath:gmatch("[^.]+") do
                if not current[part] then current[part] = {} end
                current = current[part]
            end
            
        elseif cleanLine:match("^(.-)%s*=%s*(.+)$") then
            -- Key = Value
            local key, rawValue = cleanLine:match("^(.-)%s*=%s*(.+)$")
            key = key:match("^%s*(.-)%s*$")
            rawValue = rawValue:match("^%s*(.-)%s*$")
            
            local value
            if rawValue:match('^"""(.*)"""$') then
                value = rawValue:match('^"""(.*)"""$')
            elseif rawValue:match('^"(.*)"$') then
                value = rawValue:match('^"(.*)"$')
            elseif rawValue:match("^'(.*)'$") then
                value = rawValue:match("^'(.*)'$")
            elseif rawValue == "true" then value = true
            elseif rawValue == "false" then value = false
            elseif rawValue:match("^%d%d%d%d%-%d%d%-%d%d$") then
                value = rawValue  -- date string
            elseif rawValue:match("^%[(.*)%]$") then
                -- inline array
                local arrContent = rawValue:match("^%[(.*)%]$")
                value = {}
                for item in arrContent:gmatch("[^,]+") do
                    item = item:match("^%s*(.-)%s*$")
                    if item:match('^"(.*)"$') then
                        table.insert(value, item:match('^"(.*)"$'))
                    elseif tonumber(item) then
                        table.insert(value, tonumber(item))
                    elseif item == "true" then
                        table.insert(value, true)
                    elseif item == "false" then
                        table.insert(value, false)
                    else
                        table.insert(value, item)
                    end
                end
            elseif tonumber(rawValue) then
                value = tonumber(rawValue)
            else
                value = rawValue
            end
            
            current[key] = value
        end
    end
    
    return result
end

local tomlContent = [[
# Application config
title = "TOML Example Config"
version = "1.0.0"

[server]
host = "localhost"
port = 8080
workers = 4
debug = false
allowed_hosts = ["localhost", "127.0.0.1", "::1"]

[database]
host = "db.example.com"
port = 5432
name = "myapp"
max_connections = 100

[database.credentials]
user = "admin"
password = "secret"

[logging]
level = "info"
format = "json"
outputs = ["stdout", "file"]

[[servers]]
name = "primary"
ip = "192.168.1.1"

[[servers]]
name = "backup"
ip = "192.168.1.2"
]]

local tomlParser = TOMLParser.new()
local config = tomlParser:parse(tomlContent)

print(string.format("Title: %s", config.title))
print(string.format("Server: %s:%d", config.server.host, config.server.port))
print(string.format("DB Credentials: %s@%s", 
    config.database.credentials.user, config.database.host))
print(string.format("Allowed hosts: %s", table.concat(config.server.allowed_hosts, ", ")))
if config.servers then
    for _, s in ipairs(config.servers) do
        print(string.format("  Server %s: %s", s.name, s.ip))
    end
end
```

### Lua Table Config (Native)

```lua
-- ตัวอย่างที่ 4: Lua Table as Config
print("=== Lua Table Config ===")

-- config.lua ตัวอย่าง (เป็น Lua code)
local configContent = [[
return {
    app = {
        name = "MyApp",
        version = "3.0.0",
        debug = false,
    },
    
    database = {
        host = "localhost",
        port = 5432,
        name = "production",
        pool = {
            min = 2,
            max = 20,
        }
    },
    
    cache = {
        driver = "redis",
        ttl = 3600,
    },
    
    features = {
        payments = true,
        analytics = false,
        betaUI = false,
    }
}
]]

-- Load config from string (simulating file load)
local function loadLuaConfig(content)
    local fn, err = load(content)
    if not fn then
        error("Failed to parse config: " .. err)
    end
    local ok, result = pcall(fn)
    if not ok then
        error("Failed to evaluate config: " .. result)
    end
    return result
end

local luaConfig = loadLuaConfig(configContent)

print(string.format("App: %s", luaConfig.app.name))
print(string.format("DB Pool: %d-%d", luaConfig.database.pool.min, luaConfig.database.pool.max))
print(string.format("Payments: %s", tostring(luaConfig.features.payments)))

-- ข้อดีของ Lua config: สามารถมี expressions และ functions ได้
local dynamicConfigContent = [[
local env = os.getenv("APP_ENV") or "development"
local isProd = env == "production"

return {
    env = env,
    debug = not isProd,
    database = {
        host = isProd and "prod-db.example.com" or "localhost",
        port = 5432,
    },
    logLevel = isProd and "warn" or "debug",
}
]]

local dynamicConfig = loadLuaConfig(dynamicConfigContent)
print(string.format("\nDynamic config env: %s, debug: %s",
    dynamicConfig.env, tostring(dynamicConfig.debug)))
```

## 40.2 Environment Variable Config

```lua
-- ตัวอย่างที่ 5: Environment Variable Config
print("=== Environment Variable Config ===")

local EnvConfig = {}
EnvConfig.__index = EnvConfig

function EnvConfig.new(options)
    options = options or {}
    return setmetatable({
        _prefix = options.prefix or "",
        _separator = options.separator or "_",
        _defaults = options.defaults or {},
        _required = options.required or {},
        _schema = options.schema or {},
        _values = {},
    }, EnvConfig)
end

function EnvConfig:_getEnv(key)
    local fullKey = self._prefix ~= "" and (self._prefix .. self._separator .. key) or key
    return os.getenv(fullKey) or os.getenv(key)
end

function EnvConfig:_coerce(value, targetType)
    if value == nil then return nil end
    if targetType == "number" then
        return tonumber(value)
    elseif targetType == "boolean" then
        return value == "true" or value == "1" or value == "yes"
    elseif targetType == "array" then
        local arr = {}
        for item in value:gmatch("[^,]+") do
            table.insert(arr, item:match("^%s*(.-)%s*$"))
        end
        return arr
    end
    return value
end

function EnvConfig:load()
    local errors = {}
    
    -- Load from schema
    for key, schema in pairs(self._schema) do
        local envValue = self:_getEnv(key)
        
        if envValue == nil then
            if schema.required then
                table.insert(errors, string.format("Required env var missing: %s", key))
            elseif schema.default ~= nil then
                self._values[key] = schema.default
            end
        else
            local coerced = self:_coerce(envValue, schema.type)
            
            -- Validate
            if schema.validate and not schema.validate(coerced) then
                table.insert(errors, string.format("Invalid value for %s: %s", key, envValue))
            else
                self._values[key] = coerced
            end
        end
    end
    
    -- Check required vars
    for _, key in ipairs(self._required) do
        if not self._values[key] and not self:_getEnv(key) then
            table.insert(errors, string.format("Required env var missing: %s", key))
        end
    end
    
    if #errors > 0 then
        return nil, errors
    end
    
    return self._values, nil
end

function EnvConfig:get(key, default)
    local value = self._values[key]
    if value ~= nil then return value end
    
    local envValue = self:_getEnv(key)
    if envValue ~= nil then
        local schema = self._schema[key]
        if schema then
            return self:_coerce(envValue, schema.type)
        end
        return envValue
    end
    
    return default ~= nil and default or self._defaults[key]
end

-- ทดสอบ (simulate env vars)
local function withEnv(vars, fn)
    -- In real code, these would be actual env vars
    -- Here we simulate by temporarily setting defaults
    fn(vars)
end

local envConfig = EnvConfig.new({
    prefix = "APP",
    schema = {
        PORT = {type = "number", default = 3000, validate = function(v) return v > 0 and v < 65536 end},
        HOST = {type = "string", default = "localhost"},
        DEBUG = {type = "boolean", default = false},
        DB_HOST = {type = "string", required = false, default = "localhost"},
        DB_PORT = {type = "number", default = 5432},
        DB_NAME = {type = "string", default = "myapp"},
        MAX_CONNECTIONS = {type = "number", default = 10},
        ALLOWED_ORIGINS = {type = "array", default = {"localhost"}},
    }
})

local values, errors = envConfig:load()
if errors then
    print("Config errors: " .. table.concat(errors, ", "))
else
    print(string.format("Port: %s", tostring(envConfig:get("PORT"))))
    print(string.format("Debug: %s", tostring(envConfig:get("DEBUG"))))
    print(string.format("DB: %s:%s/%s",
        envConfig:get("DB_HOST"),
        envConfig:get("DB_PORT"),
        envConfig:get("DB_NAME")))
end
```

## 40.3 Command-line Argument Parsing

```lua
-- ตัวอย่างที่ 6: Command-line Argument Parser
print("=== CLI Argument Parser ===")

local ArgParser = {}
ArgParser.__index = ArgParser

function ArgParser.new(description)
    return setmetatable({
        description = description or "",
        _options = {},
        _positional = {},
        _parsed = {},
        _help = false,
    }, ArgParser)
end

function ArgParser:addOption(short, long, options)
    options = options or {}
    local opt = {
        short = short,
        long = long,
        description = options.description or "",
        default = options.default,
        required = options.required or false,
        type = options.type or "string",  -- string, number, boolean, array
        action = options.action or "store",
        choices = options.choices,
    }
    table.insert(self._options, opt)
    return self
end

function ArgParser:addPositional(name, options)
    options = options or {}
    table.insert(self._positional, {
        name = name,
        description = options.description or "",
        required = options.required ~= false,
    })
    return self
end

function ArgParser:parse(args)
    local result = {}
    local errors = {}
    
    -- Set defaults
    for _, opt in ipairs(self._options) do
        if opt.default ~= nil then
            local key = opt.long:gsub("^%-%-", ""):gsub("%-", "_")
            result[key] = opt.default
        end
    end
    
    local i = 1
    local positionalIdx = 1
    
    while i <= #args do
        local arg = args[i]
        
        if arg == "--help" or arg == "-h" then
            self:printHelp()
            return nil
        elseif arg:match("^%-%-(.+)=(.+)$") then
            -- --key=value
            local key, value = arg:match("^%-%-(.+)=(.+)$")
            local opt = self:_findOption(nil, "--" .. key)
            if opt then
                local resultKey = key:gsub("%-", "_")
                result[resultKey] = self:_coerceValue(value, opt.type)
            else
                table.insert(errors, "Unknown option: --" .. key)
            end
        elseif arg:match("^%-%-(%.-)$") then
            -- --key [value]
            local key = arg:match("^%-%-(.-)$")
            local opt = self:_findOption(nil, "--" .. key)
            if opt then
                local resultKey = key:gsub("%-", "_")
                if opt.type == "boolean" or opt.action == "store_true" then
                    result[resultKey] = true
                elseif opt.action == "store_false" then
                    result[resultKey] = false
                else
                    i = i + 1
                    if i <= #args then
                        result[resultKey] = self:_coerceValue(args[i], opt.type)
                    else
                        table.insert(errors, "Missing value for --" .. key)
                    end
                end
            else
                table.insert(errors, "Unknown option: --" .. key)
            end
        elseif arg:match("^%-(%a)$") then
            -- -k [value]
            local flag = "-" .. arg:match("^%-(%a)$")
            local opt = self:_findOption(flag, nil)
            if opt then
                local key = opt.long:gsub("^%-%-", ""):gsub("%-", "_")
                if opt.type == "boolean" or opt.action == "store_true" then
                    result[key] = true
                else
                    i = i + 1
                    if i <= #args then
                        result[key] = self:_coerceValue(args[i], opt.type)
                    else
                        table.insert(errors, "Missing value for " .. flag)
                    end
                end
            else
                table.insert(errors, "Unknown flag: " .. flag)
            end
        else
            -- Positional argument
            local pos = self._positional[positionalIdx]
            if pos then
                result[pos.name] = arg
                positionalIdx = positionalIdx + 1
            else
                table.insert(errors, "Unexpected argument: " .. arg)
            end
        end
        
        i = i + 1
    end
    
    -- Check required options
    for _, opt in ipairs(self._options) do
        if opt.required then
            local key = opt.long:gsub("^%-%-", ""):gsub("%-", "_")
            if result[key] == nil then
                table.insert(errors, string.format("Required option missing: %s", opt.long))
            end
        end
    end
    
    -- Check required positionals
    for _, pos in ipairs(self._positional) do
        if pos.required and result[pos.name] == nil then
            table.insert(errors, string.format("Required argument missing: %s", pos.name))
        end
    end
    
    if #errors > 0 then
        return nil, errors
    end
    
    return result, nil
end

function ArgParser:_findOption(short, long)
    for _, opt in ipairs(self._options) do
        if (short and opt.short == short) or (long and opt.long == long) then
            return opt
        end
    end
    return nil
end

function ArgParser:_coerceValue(value, targetType)
    if targetType == "number" then
        return tonumber(value) or 0
    elseif targetType == "boolean" then
        return value == "true" or value == "1"
    end
    return value
end

function ArgParser:printHelp()
    print(string.format("Usage: program [options]"))
    if self.description ~= "" then
        print(self.description)
    end
    print("\nOptions:")
    for _, opt in ipairs(self._options) do
        local flags = ""
        if opt.short then flags = flags .. opt.short .. ", " end
        flags = flags .. opt.long
        if opt.type ~= "boolean" and opt.action ~= "store_true" then
            flags = flags .. " <" .. opt.type .. ">"
        end
        local default = ""
        if opt.default ~= nil then
            default = string.format(" (default: %s)", tostring(opt.default))
        end
        print(string.format("  %-30s %s%s", flags, opt.description, default))
    end
end

-- Setup CLI parser
local parser = ArgParser.new("Application launcher")
parser:addOption("-p", "--port",    {type="number", default=8080, description="Server port"})
parser:addOption("-H", "--host",    {type="string", default="localhost", description="Server host"})
parser:addOption("-d", "--debug",   {type="boolean", action="store_true", description="Enable debug mode"})
parser:addOption("-e", "--env",     {type="string", default="development", 
                                      description="Environment", choices={"development","staging","production"}})
parser:addOption("-w", "--workers", {type="number", default=1, description="Worker count"})
parser:addOption(nil, "--config",   {type="string", description="Config file path"})
parser:addOption("-v", "--verbose", {type="boolean", action="store_true", description="Verbose output"})

-- Simulate command line args
local args = {"--port=9090", "-d", "--env", "production", "-w", "4", "--verbose"}
local result, errors = parser:parse(args)

if errors then
    for _, e in ipairs(errors) do print("Error: " .. e) end
else
    print(string.format("Port: %d", result.port))
    print(string.format("Debug: %s", tostring(result.debug)))
    print(string.format("Env: %s", result.env))
    print(string.format("Workers: %d", result.workers))
    print(string.format("Verbose: %s", tostring(result.verbose)))
end
```

## 40.4 Config Validation

```lua
-- ตัวอย่างที่ 7: Config Schema Validation
print("=== Config Validation ===")

local ConfigValidator = {}
ConfigValidator.__index = ConfigValidator

function ConfigValidator.new(schema)
    return setmetatable({_schema = schema}, ConfigValidator)
end

function ConfigValidator:validate(config, schema, path)
    schema = schema or self._schema
    path = path or ""
    local errors = {}
    
    for key, rule in pairs(schema) do
        local fullPath = path ~= "" and (path .. "." .. key) or key
        local value = config[key]
        
        -- Required check
        if rule.required ~= false and value == nil then
            if rule.default ~= nil then
                config[key] = rule.default
                value = rule.default
            else
                table.insert(errors, string.format("Missing required field: %s", fullPath))
                goto continue
            end
        end
        
        if value == nil then goto continue end
        
        -- Type check
        if rule.type then
            local expectedType = rule.type
            local actualType = type(value)
            
            if expectedType == "integer" then
                if actualType ~= "number" or value ~= math.floor(value) then
                    table.insert(errors, string.format("%s must be an integer", fullPath))
                end
            elseif expectedType == "array" then
                if actualType ~= "table" then
                    table.insert(errors, string.format("%s must be an array", fullPath))
                end
            elseif actualType ~= expectedType then
                table.insert(errors, string.format("%s must be %s, got %s",
                    fullPath, expectedType, actualType))
                goto continue
            end
        end
        
        -- Range checks
        if type(value) == "number" then
            if rule.min and value < rule.min then
                table.insert(errors, string.format("%s must be >= %s", fullPath, rule.min))
            end
            if rule.max and value > rule.max then
                table.insert(errors, string.format("%s must be <= %s", fullPath, rule.max))
            end
        end
        
        -- String checks
        if type(value) == "string" then
            if rule.minLength and #value < rule.minLength then
                table.insert(errors, string.format("%s must be at least %d chars", fullPath, rule.minLength))
            end
            if rule.maxLength and #value > rule.maxLength then
                table.insert(errors, string.format("%s must be at most %d chars", fullPath, rule.maxLength))
            end
            if rule.pattern and not value:match(rule.pattern) then
                table.insert(errors, string.format("%s must match pattern %s", fullPath, rule.pattern))
            end
            if rule.enum then
                local valid = false
                for _, choice in ipairs(rule.enum) do
                    if value == choice then valid = true; break end
                end
                if not valid then
                    table.insert(errors, string.format("%s must be one of: %s",
                        fullPath, table.concat(rule.enum, ", ")))
                end
            end
        end
        
        -- Nested object
        if rule.properties and type(value) == "table" then
            local nestedErrors = self:validate(value, rule.properties, fullPath)
            for _, e in ipairs(nestedErrors) do
                table.insert(errors, e)
            end
        end
        
        -- Custom validator
        if rule.validate then
            local ok, errMsg = rule.validate(value)
            if not ok then
                table.insert(errors, string.format("%s: %s", fullPath, errMsg))
            end
        end
        
        ::continue::
    end
    
    return errors
end

-- Define schema
local appSchema = {
    port = {
        type = "integer",
        min = 1,
        max = 65535,
        default = 8080,
        required = true,
    },
    host = {
        type = "string",
        default = "0.0.0.0",
        pattern = "^[%d%.]+$",
    },
    env = {
        type = "string",
        enum = {"development", "staging", "production"},
        default = "development",
    },
    database = {
        type = "table",
        required = true,
        properties = {
            host = {type = "string", required = true},
            port = {type = "integer", min = 1, max = 65535, default = 5432},
            name = {type = "string", required = true, minLength = 1},
            maxConnections = {type = "integer", min = 1, max = 1000, default = 10},
        }
    },
    apiKey = {
        type = "string",
        required = true,
        minLength = 32,
        validate = function(v)
            if v:match("^[A-Za-z0-9_-]+$") then
                return true
            end
            return false, "API key must be alphanumeric"
        end
    },
}

local validator = ConfigValidator.new(appSchema)

-- Valid config
local goodConfig = {
    port = 9090,
    env = "production",
    database = {
        host = "prod-db.example.com",
        name = "production",
        maxConnections = 50,
    },
    apiKey = "abc123def456ghi789jkl012mno345pqr678",
}

local errors = validator:validate(goodConfig)
if #errors == 0 then
    print("Valid config! Defaults applied:")
    print(string.format("  port: %d, env: %s", goodConfig.port, goodConfig.env))
    print(string.format("  db port: %d", goodConfig.database.port))
else
    print("Errors: " .. table.concat(errors, ", "))
end

-- Invalid config
local badConfig = {
    port = 99999,  -- too high
    env = "local",  -- not in enum
    database = {
        name = "db",
        -- missing host
    },
    apiKey = "short",  -- too short
}

errors = validator:validate(badConfig)
print("\nInvalid config errors:")
for _, e in ipairs(errors) do
    print("  - " .. e)
end
```

## 40.5 Default Values and Config Merging

```lua
-- ตัวอย่างที่ 8: Config Merging
print("=== Config Merging ===")

local function deepMerge(target, source, strategy)
    strategy = strategy or "replace"  -- replace, merge, append
    
    for k, v in pairs(source) do
        if type(v) == "table" and type(target[k]) == "table" then
            deepMerge(target[k], v, strategy)
        elseif strategy == "append" and type(v) == "table" and type(target[k]) == "table" then
            for _, item in ipairs(v) do
                table.insert(target[k], item)
            end
        else
            target[k] = v
        end
    end
    return target
end

local function deepCopy(t)
    if type(t) ~= "table" then return t end
    local copy = {}
    for k, v in pairs(t) do
        copy[deepCopy(k)] = deepCopy(v)
    end
    return setmetatable(copy, getmetatable(t))
end

-- Base config (defaults)
local defaults = {
    server = {
        host = "localhost",
        port = 8080,
        timeout = 30,
        keepAlive = true,
    },
    database = {
        host = "localhost",
        port = 5432,
        pool = {min = 1, max = 5},
    },
    logging = {
        level = "info",
        format = "text",
        outputs = {"stdout"},
    },
    cache = {
        enabled = false,
        ttl = 300,
    },
}

-- Environment-specific overrides
local productionOverrides = {
    server = {
        host = "0.0.0.0",
        port = 443,
    },
    database = {
        host = "prod-db.example.com",
        pool = {min = 5, max = 50},
    },
    logging = {
        level = "warn",
        format = "json",
    },
    cache = {
        enabled = true,
        ttl = 3600,
    },
}

-- Merge configs
local finalConfig = deepMerge(deepCopy(defaults), productionOverrides)

print("Final production config:")
print(string.format("  Server: %s:%d", finalConfig.server.host, finalConfig.server.port))
print(string.format("  DB: %s (pool: %d-%d)", 
    finalConfig.database.host, finalConfig.database.pool.min, finalConfig.database.pool.max))
print(string.format("  Logging: %s/%s", finalConfig.logging.level, finalConfig.logging.format))
print(string.format("  Cache: enabled=%s, ttl=%d", 
    tostring(finalConfig.cache.enabled), finalConfig.cache.ttl))
print(string.format("  Timeout: %d (from default)", finalConfig.server.timeout))
```

## 40.6 Hot Reload Config

```lua
-- ตัวอย่างที่ 9: Hot Reload Configuration
print("=== Hot Reload Config ===")

local HotReloadConfig = {}
HotReloadConfig.__index = HotReloadConfig

function HotReloadConfig.new(options)
    options = options or {}
    local hrc = setmetatable({
        _config = {},
        _filename = options.filename,
        _checkInterval = options.checkInterval or 5,
        _lastModified = 0,
        _callbacks = {},
        _version = 0,
        _autoReloadEnabled = true,
    }, HotReloadConfig)
    
    return hrc
end

function HotReloadConfig:load(configData)
    local oldConfig = self._config
    self._config = configData
    self._version = self._version + 1
    
    -- Find changed keys
    local changes = {}
    for k, v in pairs(configData) do
        if oldConfig[k] ~= v then
            table.insert(changes, {key = k, old = oldConfig[k], new = v})
        end
    end
    for k, v in pairs(oldConfig) do
        if configData[k] == nil then
            table.insert(changes, {key = k, old = v, new = nil})
        end
    end
    
    if #changes > 0 then
        self:_notifyChange(changes)
    end
    
    return changes
end

function HotReloadConfig:onChange(callback)
    table.insert(self._callbacks, callback)
    return self
end

function HotReloadConfig:_notifyChange(changes)
    print(string.format("[HotReload] Config changed (v%d), %d keys changed", 
        self._version, #changes))
    for _, cb in ipairs(self._callbacks) do
        pcall(cb, changes, self._version)
    end
end

function HotReloadConfig:get(key, default)
    local value = self._config[key]
    return value ~= nil and value or default
end

function HotReloadConfig:getAll()
    return self._config
end

function HotReloadConfig:version()
    return self._version
end

-- ทดสอบ
local hotConfig = HotReloadConfig.new({filename = "config.json"})

hotConfig:onChange(function(changes, version)
    print(string.format("  Config updated to version %d:", version))
    for _, change in ipairs(changes) do
        print(string.format("    %s: %s -> %s", 
            change.key, 
            tostring(change.old), 
            tostring(change.new)))
    end
end)

-- Initial load
hotConfig:load({
    logLevel = "info",
    maxConnections = 10,
    debug = false,
    version = "1.0.0"
})

print(string.format("Initial config v%d:", hotConfig:version()))
print(string.format("  logLevel: %s", hotConfig:get("logLevel")))

-- Simulate hot reload
print("\nSimulating config file change...")
hotConfig:load({
    logLevel = "debug",  -- changed
    maxConnections = 20, -- changed
    debug = true,        -- changed
    version = "1.0.0",   -- same
    newFeature = "enabled"  -- added
})

print(string.format("After reload v%d:", hotConfig:version()))
print(string.format("  logLevel: %s", hotConfig:get("logLevel")))
print(string.format("  maxConnections: %d", hotConfig:get("maxConnections")))
print(string.format("  newFeature: %s", hotConfig:get("newFeature", "not set")))
```

## 40.7 Secrets Management

```lua
-- ตัวอย่างที่ 10: Secrets Management
print("=== Secrets Management ===")

local SecretsManager = {}
SecretsManager.__index = SecretsManager

function SecretsManager.new(options)
    options = options or {}
    return setmetatable({
        _secrets = {},
        _encrypted = {},
        _redactedKeys = options.redactedKeys or {
            "password", "secret", "token", "key", "apikey",
            "api_key", "private_key", "credentials",
        },
        _rotationCallbacks = {},
    }, SecretsManager)
end

function SecretsManager:_isSecret(key)
    local lowerKey = key:lower()
    for _, pattern in ipairs(self._redactedKeys) do
        if lowerKey:find(pattern, 1, true) then
            return true
        end
    end
    return false
end

function SecretsManager:set(key, value)
    self._secrets[key] = value
end

function SecretsManager:get(key)
    return self._secrets[key]
end

function SecretsManager:getOrEnv(key, envKey)
    return self._secrets[key] or os.getenv(envKey or key)
end

function SecretsManager:require(key)
    local value = self:get(key) or os.getenv(key)
    if value == nil then
        error(string.format("Required secret not found: %s", key))
    end
    return value
end

function SecretsManager:toSafe(config)
    -- Return config with secrets redacted
    local function redact(t, path)
        local result = {}
        path = path or ""
        for k, v in pairs(t) do
            local fullKey = path ~= "" and (path .. "." .. k) or tostring(k)
            if type(v) == "table" then
                result[k] = redact(v, fullKey)
            elseif self:_isSecret(tostring(k)) then
                result[k] = "***REDACTED***"
            else
                result[k] = v
            end
        end
        return result
    end
    return redact(config)
end

function SecretsManager:onRotation(key, callback)
    if not self._rotationCallbacks[key] then
        self._rotationCallbacks[key] = {}
    end
    table.insert(self._rotationCallbacks[key], callback)
end

function SecretsManager:rotate(key, newValue)
    local oldValue = self._secrets[key]
    self._secrets[key] = newValue
    print(string.format("[Secrets] Rotating key: %s", key))
    
    if self._rotationCallbacks[key] then
        for _, cb in ipairs(self._rotationCallbacks[key]) do
            pcall(cb, key, newValue, oldValue)
        end
    end
end

-- ทดสอบ
local secrets = SecretsManager.new()

-- Load secrets (in real code, from vault/environment)
secrets:set("DB_PASSWORD", "super_secret_password_123")
secrets:set("JWT_SECRET", "jwt_signing_key_xyz789")
secrets:set("STRIPE_API_KEY", "sk_live_abc123def456")
secrets:set("REDIS_AUTH", "redis_auth_token")

-- Application config with secrets
local appConfig = {
    database = {
        host = "db.example.com",
        port = 5432,
        user = "admin",
        password = secrets:get("DB_PASSWORD"),
    },
    auth = {
        jwtSecret = secrets:get("JWT_SECRET"),
        tokenExpiry = 3600,
    },
    payments = {
        apiKey = secrets:get("STRIPE_API_KEY"),
        webhookSecret = "whsec_example",
    },
}

-- Safe config for logging (no secrets)
local safeConfig = secrets:toSafe(appConfig)
print("Safe config (for logging):")
print(string.format("  DB: %s@%s", appConfig.database.user, appConfig.database.host))
print(string.format("  DB password: %s", safeConfig.database.password))
print(string.format("  JWT secret: %s", safeConfig.auth.jwtSecret))
print(string.format("  Stripe key: %s", safeConfig.payments.apiKey))

-- Setup rotation handler
secrets:onRotation("DB_PASSWORD", function(key, newVal, oldVal)
    print(string.format("[Rotation] Key '%s' rotated, reconnecting database...", key))
end)

secrets:rotate("DB_PASSWORD", "new_super_secret_password_456")
```

## 40.8 Environment-Specific Configs

```lua
-- ตัวอย่างที่ 11: Multi-environment Config
print("=== Multi-environment Config ===")

local MultiEnvConfig = {}
MultiEnvConfig.__index = MultiEnvConfig

function MultiEnvConfig.new(baseConfig)
    return setmetatable({
        _base = baseConfig or {},
        _environments = {},
        _current = nil,
    }, MultiEnvConfig)
end

function MultiEnvConfig:addEnv(envName, overrides)
    self._environments[envName] = overrides
    return self
end

function MultiEnvConfig:use(envName)
    if not self._environments[envName] then
        error("Unknown environment: " .. envName)
    end
    self._current = envName
    return self
end

function MultiEnvConfig:resolve()
    local function merge(target, source)
        local result = {}
        for k, v in pairs(target) do result[k] = v end
        for k, v in pairs(source) do
            if type(v) == "table" and type(result[k]) == "table" then
                result[k] = merge(result[k], v)
            else
                result[k] = v
            end
        end
        return result
    end
    
    if not self._current then
        return self._base
    end
    
    return merge(self._base, self._environments[self._current])
end

function MultiEnvConfig:get(key)
    local config = self:resolve()
    local parts = {}
    for part in key:gmatch("[^.]+") do
        table.insert(parts, part)
    end
    local current = config
    for _, part in ipairs(parts) do
        if type(current) ~= "table" then return nil end
        current = current[part]
    end
    return current
end

-- Setup multi-env config
local config = MultiEnvConfig.new({
    app = {name = "MyService", version = "1.0.0"},
    server = {host = "localhost", port = 3000, workers = 1},
    database = {
        host = "localhost",
        port = 5432,
        name = "myapp",
        pool = {min = 1, max = 5},
    },
    logging = {level = "debug", format = "text"},
    features = {
        rateLimit = false,
        caching = false,
        metrics = false,
    },
})

config:addEnv("development", {
    logging = {level = "debug"},
    features = {rateLimit = false},
})

config:addEnv("staging", {
    server = {host = "0.0.0.0", workers = 2},
    database = {
        host = "staging-db.example.com",
        pool = {min = 2, max = 10},
    },
    logging = {level = "info", format = "json"},
    features = {
        rateLimit = true,
        caching = true,
        metrics = true,
    },
})

config:addEnv("production", {
    server = {host = "0.0.0.0", port = 8080, workers = 8},
    database = {
        host = "prod-db.example.com",
        pool = {min = 5, max = 50},
    },
    logging = {level = "warn", format = "json"},
    features = {
        rateLimit = true,
        caching = true,
        metrics = true,
    },
})

-- ทดสอบแต่ละ environment
for _, env in ipairs({"development", "staging", "production"}) do
    config:use(env)
    local resolved = config:resolve()
    print(string.format("\n[%s]", env:upper()))
    print(string.format("  Server: %s:%d (%d workers)",
        resolved.server.host, resolved.server.port, resolved.server.workers))
    print(string.format("  DB: %s (pool: %d-%d)",
        resolved.database.host, resolved.database.pool.min, resolved.database.pool.max))
    print(string.format("  Log: %s/%s",
        resolved.logging.level, resolved.logging.format))
    print(string.format("  Features: rateLimit=%s, caching=%s, metrics=%s",
        tostring(resolved.features.rateLimit),
        tostring(resolved.features.caching),
        tostring(resolved.features.metrics)))
end
```

## 40.9 12-Factor App Config

```lua
-- ตัวอย่างที่ 12: 12-Factor App Config implementation
print("=== 12-Factor App Config ===")

local TwelveFactorConfig = {}
TwelveFactorConfig.__index = TwelveFactorConfig

function TwelveFactorConfig.new()
    return setmetatable({
        _specs = {},
        _values = {},
        _loaded = false,
    }, TwelveFactorConfig)
end

function TwelveFactorConfig:define(name, options)
    options = options or {}
    self._specs[name] = {
        envVar = options.envVar or name,
        default = options.default,
        required = options.required or false,
        type = options.type or "string",
        description = options.description or "",
        sensitive = options.sensitive or false,
        choices = options.choices,
    }
    return self
end

function TwelveFactorConfig:load()
    local errors = {}
    
    for name, spec in pairs(self._specs) do
        local rawValue = os.getenv(spec.envVar)
        
        if rawValue == nil then
            if spec.required and spec.default == nil then
                table.insert(errors, string.format(
                    "Required environment variable not set: %s (%s)",
                    spec.envVar, spec.description))
            else
                self._values[name] = spec.default
            end
        else
            -- Type coercion
            local value = rawValue
            if spec.type == "number" or spec.type == "integer" then
                value = tonumber(rawValue)
                if value == nil then
                    table.insert(errors, string.format(
                        "%s must be a number, got: %s", spec.envVar, rawValue))
                end
            elseif spec.type == "boolean" then
                value = rawValue == "true" or rawValue == "1" or rawValue == "yes"
            elseif spec.type == "json" then
                local fn = load("return " .. rawValue)
                if fn then value = fn() end
            elseif spec.type == "array" then
                value = {}
                for item in rawValue:gmatch("[^,]+") do
                    table.insert(value, item:match("^%s*(.-)%s*$"))
                end
            end
            
            -- Validate choices
            if spec.choices and value then
                local valid = false
                for _, choice in ipairs(spec.choices) do
                    if value == choice then valid = true; break end
                end
                if not valid then
                    table.insert(errors, string.format(
                        "%s must be one of [%s], got: %s",
                        spec.envVar, table.concat(spec.choices, ", "), tostring(value)))
                end
            end
            
            self._values[name] = value
        end
    end
    
    self._loaded = true
    
    if #errors > 0 then
        return false, errors
    end
    
    return true, nil
end

function TwelveFactorConfig:get(name)
    if not self._loaded then
        error("Config not loaded. Call :load() first.")
    end
    if not self._specs[name] then
        error("Unknown config key: " .. name)
    end
    return self._values[name]
end

function TwelveFactorConfig:printSummary()
    print(string.format("%-25s %-15s %-10s %s",
        "Variable", "Current Value", "Sensitive", "Description"))
    print(string.rep("-", 80))
    
    local names = {}
    for name, _ in pairs(self._specs) do table.insert(names, name) end
    table.sort(names)
    
    for _, name in ipairs(names) do
        local spec = self._specs[name]
        local value = self._values[name]
        local displayValue
        if spec.sensitive then
            displayValue = "***"
        elseif value == nil then
            displayValue = "(not set)"
        else
            displayValue = tostring(value)
        end
        if #displayValue > 15 then
            displayValue = displayValue:sub(1, 12) .. "..."
        end
        print(string.format("%-25s %-15s %-10s %s",
            spec.envVar, displayValue,
            tostring(spec.sensitive), spec.description))
    end
end

-- Define config
local appConfig = TwelveFactorConfig.new()

appConfig
    :define("port", {
        envVar = "PORT",
        type = "integer",
        default = 3000,
        description = "HTTP server port",
    })
    :define("host", {
        envVar = "HOST",
        default = "0.0.0.0",
        description = "HTTP server host",
    })
    :define("nodeEnv", {
        envVar = "NODE_ENV",
        default = "development",
        choices = {"development", "staging", "production", "test"},
        description = "Application environment",
    })
    :define("databaseUrl", {
        envVar = "DATABASE_URL",
        required = false,
        default = "postgres://localhost:5432/myapp",
        sensitive = true,
        description = "Database connection URL",
    })
    :define("jwtSecret", {
        envVar = "JWT_SECRET",
        required = false,
        default = "dev-secret-key-change-in-production",
        sensitive = true,
        description = "JWT signing secret",
    })
    :define("redisUrl", {
        envVar = "REDIS_URL",
        default = "redis://localhost:6379",
        description = "Redis connection URL",
    })
    :define("debug", {
        envVar = "DEBUG",
        type = "boolean",
        default = false,
        description = "Enable debug logging",
    })
    :define("allowedOrigins", {
        envVar = "ALLOWED_ORIGINS",
        type = "array",
        default = {"localhost:3000"},
        description = "CORS allowed origins",
    })

local ok, errors = appConfig:load()
if not ok then
    print("Configuration errors:")
    for _, e in ipairs(errors) do
        print("  - " .. e)
    end
else
    print("Configuration loaded successfully!")
    appConfig:printSummary()
    
    print(string.format("\nPort: %d", appConfig:get("port")))
    print(string.format("Environment: %s", appConfig:get("nodeEnv")))
    print(string.format("Debug: %s", tostring(appConfig:get("debug"))))
end
```

## 40.10 Config with Schema and Auto-complete Support

```lua
-- ตัวอย่างที่ 13: Type-safe Config Builder
print("=== Type-safe Config Builder ===")

local function createConfigSchema(definition)
    local schema = {_defs = definition, _data = {}}
    
    -- Generate getters and setters
    local function makeAccessors(obj, defs, data)
        for key, def in pairs(defs) do
            if type(def) == "table" and def._type then
                -- Leaf value
                local function getter()
                    return data[key] ~= nil and data[key] or def._default
                end
                local function setter(value)
                    -- Validate type
                    if def._type == "string" and type(value) ~= "string" then
                        error(string.format("Config.%s must be a string", key))
                    elseif def._type == "number" and type(value) ~= "number" then
                        error(string.format("Config.%s must be a number", key))
                    elseif def._type == "boolean" and type(value) ~= "boolean" then
                        error(string.format("Config.%s must be a boolean", key))
                    end
                    if def._validate then
                        local ok, err = def._validate(value)
                        if not ok then
                            error(string.format("Config.%s: %s", key, err))
                        end
                    end
                    data[key] = value
                end
                obj["get" .. key:sub(1,1):upper() .. key:sub(2)] = getter
                obj["set" .. key:sub(1,1):upper() .. key:sub(2)] = setter
                obj[key] = getter  -- shorthand getter
            else
                -- Nested object
                data[key] = data[key] or {}
                obj[key] = {}
                makeAccessors(obj[key], def, data[key])
            end
        end
    end
    
    makeAccessors(schema, definition, schema._data)
    return schema
end

-- Define config schema
local Config = createConfigSchema({
    server = {
        host = {_type = "string", _default = "localhost"},
        port = {_type = "number", _default = 8080, 
            _validate = function(v)
                if v < 1 or v > 65535 then
                    return false, "port must be 1-65535"
                end
                return true
            end
        },
    },
    database = {
        host = {_type = "string", _default = "localhost"},
        port = {_type = "number", _default = 5432},
        name = {_type = "string", _default = "myapp"},
    },
    debug = {_type = "boolean", _default = false},
})

-- Use type-safe setters
Config.server.setHost("production-server.com")
Config.server.setPort(443)
Config.database.setHost("db.production.com")
Config.debug.setDebug = nil  -- this won't work, but accessor handles it

print(string.format("Server: %s:%d",
    Config.server.host(), Config.server.port()))
print(string.format("Database: %s/%s",
    Config.database.host(), Config.database.name()))

-- Test validation error
local ok, err = pcall(function()
    Config.server.setPort(99999)
end)
if not ok then print("Validation error: " .. err) end
```

## 40.11 Config File Discovery

```lua
-- ตัวอย่างที่ 14: Config File Discovery
print("=== Config File Discovery ===")

local ConfigLoader = {}
ConfigLoader.__index = ConfigLoader

function ConfigLoader.new()
    return setmetatable({
        _searchPaths = {},
        _loaders = {},
        _loaded = {},
    }, ConfigLoader)
end

function ConfigLoader:addSearchPath(path)
    table.insert(self._searchPaths, path)
    return self
end

function ConfigLoader:registerLoader(extension, loader)
    self._loaders[extension] = loader
    return self
end

function ConfigLoader:discover(baseName)
    -- Search for config files in order
    local files = {}
    
    for _, dir in ipairs(self._searchPaths) do
        for ext, _ in pairs(self._loaders) do
            local filename = dir .. "/" .. baseName .. "." .. ext
            -- In real code: check if file exists
            table.insert(files, {path = filename, ext = ext})
        end
    end
    
    return files
end

function ConfigLoader:load(filename)
    local ext = filename:match("%.([^%.]+)$")
    local loader = self._loaders[ext]
    if not loader then
        error("No loader for extension: " .. (ext or "unknown"))
    end
    
    -- In real code: read file content
    print(string.format("[ConfigLoader] Loading: %s", filename))
    return loader(filename)
end

function ConfigLoader:loadAll(baseName)
    local result = {}
    local candidates = self:discover(baseName)
    
    for _, candidate in ipairs(candidates) do
        -- In real code: check if file exists before loading
        print(string.format("[ConfigLoader] Checking: %s", candidate.path))
    end
    
    return result
end

-- Register loaders
local loader = ConfigLoader.new()
    :addSearchPath("/etc/myapp")
    :addSearchPath("/home/user/.config/myapp")
    :addSearchPath("./config")
    :addSearchPath(".")

loader:registerLoader("json", function(path)
    return {_source = path, _format = "json"}
end)

loader:registerLoader("ini", function(path)
    return {_source = path, _format = "ini"}
end)

loader:registerLoader("lua", function(path)
    return {_source = path, _format = "lua"}
end)

local discovered = loader:discover("app")
print("Config files to search:")
for _, f in ipairs(discovered) do
    print(string.format("  %s", f.path))
end
```

## 40.12 Complete Config System

```lua
-- ตัวอย่างที่ 15: Complete Configuration System
print("=== Complete Configuration System ===")

local ConfigSystem = {}
ConfigSystem.__index = ConfigSystem

function ConfigSystem.new()
    return setmetatable({
        _layers = {},  -- ordered list of config layers
        _cache = nil,
        _dirty = true,
        _listeners = {},
        _version = 0,
    }, ConfigSystem)
end

function ConfigSystem:addLayer(name, config, priority)
    priority = priority or (#self._layers + 1)
    table.insert(self._layers, {
        name = name,
        config = config,
        priority = priority,
    })
    table.sort(self._layers, function(a, b)
        return a.priority < b.priority
    end)
    self._dirty = true
    print(string.format("[Config] Added layer: %s (priority: %d)", name, priority))
end

function ConfigSystem:_buildCache()
    if not self._dirty then return end
    
    local function merge(target, source)
        for k, v in pairs(source) do
            if type(v) == "table" and type(target[k]) == "table" then
                merge(target[k], v)
            else
                target[k] = v
            end
        end
    end
    
    local result = {}
    for _, layer in ipairs(self._layers) do
        merge(result, layer.config)
    end
    
    self._cache = result
    self._dirty = false
    self._version = self._version + 1
end

function ConfigSystem:get(path, default)
    self:_buildCache()
    
    local parts = {}
    for part in path:gmatch("[^.]+") do
        table.insert(parts, part)
    end
    
    local current = self._cache
    for _, part in ipairs(parts) do
        if type(current) ~= "table" then
            return default
        end
        current = current[part]
    end
    
    return current ~= nil and current or default
end

function ConfigSystem:set(path, value, layerName)
    layerName = layerName or "runtime"
    
    -- Find or create the layer
    local targetLayer = nil
    for _, layer in ipairs(self._layers) do
        if layer.name == layerName then
            targetLayer = layer
            break
        end
    end
    
    if not targetLayer then
        self:addLayer(layerName, {}, 999)
        targetLayer = self._layers[#self._layers]
    end
    
    -- Set value in layer
    local parts = {}
    for part in path:gmatch("[^.]+") do
        table.insert(parts, part)
    end
    
    local current = targetLayer.config
    for i = 1, #parts - 1 do
        if not current[parts[i]] then current[parts[i]] = {} end
        current = current[parts[i]]
    end
    current[parts[#parts]] = value
    
    self._dirty = true
    self._version = self._version + 1
    
    -- Notify listeners
    for _, listener in ipairs(self._listeners) do
        pcall(listener, path, value)
    end
end

function ConfigSystem:watch(callback)
    table.insert(self._listeners, callback)
    return function()
        for i, l in ipairs(self._listeners) do
            if l == callback then
                table.remove(self._listeners, i)
                return
            end
        end
    end
end

function ConfigSystem:dump()
    self:_buildCache()
    local function printTable(t, indent)
        indent = indent or ""
        for k, v in pairs(t) do
            if type(v) == "table" then
                print(string.format("%s%s:", indent, k))
                printTable(v, indent .. "  ")
            else
                print(string.format("%s%s = %s", indent, k, tostring(v)))
            end
        end
    end
    printTable(self._cache)
end

function ConfigSystem:version()
    return self._version
end

-- Setup complete config system
local config = ConfigSystem.new()

-- Layer 1: Defaults (lowest priority)
config:addLayer("defaults", {
    server = {host = "localhost", port = 3000, timeout = 30},
    database = {host = "localhost", port = 5432, pool = {min = 1, max = 5}},
    logging = {level = "info", format = "text"},
    features = {rateLimit = false, metrics = false},
}, 1)

-- Layer 2: File config
config:addLayer("file", {
    database = {host = "db.example.com", pool = {max = 20}},
    logging = {format = "json"},
}, 2)

-- Layer 3: Environment variables (highest priority for static)
config:addLayer("environment", {
    server = {port = 8080},
    logging = {level = "warn"},
}, 3)

-- Watch for changes
local unwatch = config:watch(function(path, value)
    print(string.format("[Config Watch] %s changed to: %s", path, tostring(value)))
end)

print("\nCurrent config:")
print(string.format("  server.port: %d", config:get("server.port")))
print(string.format("  database.host: %s", config:get("database.host")))
print(string.format("  logging.level: %s", config:get("logging.level")))
print(string.format("  features.rateLimit: %s", tostring(config:get("features.rateLimit"))))

-- Runtime override
print("\nSetting runtime override:")
config:set("features.rateLimit", true, "runtime")
config:set("server.workers", 4, "runtime")

print(string.format("  features.rateLimit: %s", tostring(config:get("features.rateLimit"))))
print(string.format("  server.workers: %d", config:get("server.workers")))

-- Stop watching
unwatch()
config:set("logging.level", "debug")  -- no notification
print(string.format("  (silent) logging.level: %s", config:get("logging.level")))
print(string.format("\nConfig version: %d", config:version()))
```

## 40.13 สรุปบทที่ 40

```lua
-- ตัวอย่างที่ 16: สรุป Configuration Patterns
print("=== สรุป Configuration Management ===")

local summary = {
    {
        approach = "INI Files",
        pros = "Simple, human-readable, widely supported",
        cons = "No types, no nesting, limited data types",
        useWhen = "Simple apps, legacy systems",
    },
    {
        approach = "JSON Config",
        pros = "Universal format, nested structure, tooling",
        cons = "No comments, verbose, strict syntax",
        useWhen = "APIs, modern web apps",
    },
    {
        approach = "TOML",
        pros = "Human-readable, typed, sections",
        cons = "Less widely supported",
        useWhen = "CLI tools, Rust ecosystem",
    },
    {
        approach = "Lua Table",
        pros = "Full Lua power, expressions, dynamic",
        cons = "Security risk (code execution)",
        useWhen = "Lua apps, game configs",
    },
    {
        approach = "Environment Variables",
        pros = "12-factor compliant, secure, flexible",
        cons = "No structure, string only",
        useWhen = "Production deployments, containers",
    },
    {
        approach = "Layered Config",
        pros = "Flexible, override per environment",
        cons = "Complex, hard to debug",
        useWhen = "Multi-environment apps",
    },
}

print(string.format("%-25s %-40s %-30s", "Approach", "Pros/Cons (short)", "Use When"))
print(string.rep("-", 100))
for _, s in ipairs(summary) do
    print(string.format("\n[%s]", s.approach))
    print(string.format("  + %s", s.pros))
    print(string.format("  - %s", s.cons))
    print(string.format("  When: %s", s.useWhen))
end

print("\n=== Best Practices ===")
local practices = {
    "1. Never hardcode config values in code",
    "2. Use environment variables for secrets",
    "3. Provide sensible defaults for all non-secret values",
    "4. Validate config at startup, fail fast if invalid",
    "5. Document all config options with descriptions",
    "6. Use different configs per environment (dev/staging/prod)",
    "7. Never commit secrets to version control",
    "8. Use a schema to enforce types and constraints",
    "9. Support hot-reload for non-critical configs",
    "10. Log config (with secrets redacted) at startup",
}
for _, p in ipairs(practices) do
    print("  " .. p)
end
```

---

## แบบฝึกหัดบทที่ 40

1. เขียน full YAML parser (subset) ที่รองรับ lists, dicts, strings, numbers
2. สร้าง `ConfigWatcher` ที่ monitor file system และ reload เมื่อ file เปลี่ยน
3. Implement `SecureConfig` ที่ encrypt/decrypt sensitive values ด้วย XOR cipher
4. เขียน `ConfigMigrator` ที่แปลง config format เก่าเป็น format ใหม่พร้อม versioning
5. สร้าง CLI tool ที่รับ arguments แล้ว override config จาก file สำหรับ database migration
