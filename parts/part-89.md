# บทที่ 89: Advanced Security Engineering

## บทนำ: Security Engineering ใน Lua

Security ไม่ใช่แค่ feature ที่เพิ่มทีหลัง แต่ต้องฝังอยู่ใน development process ตั้งแต่แรก บทนี้จะสำรวจ threat modeling, vulnerabilities, และ best practices สำหรับ Lua

### Threat Modeling และ STRIDE

STRIDE เป็น framework ที่ Microsoft พัฒนาขึ้นสำหรับการระบุ threats:

- **S**poofing - แกล้งทำเป็นคนอื่น
- **T**ampering - แก้ไขข้อมูล
- **R**epudiation - ปฏิเสธว่าไม่ได้ทำ
- **I**nformation Disclosure - เปิดเผยข้อมูลลับ
- **D**enial of Service - รบกวนการให้บริการ
- **E**levation of Privilege - เพิ่มสิทธิ์

```lua
-- ตัวอย่างที่ 1: Threat Model Builder
local ThreatModel = {}
ThreatModel.__index = ThreatModel

ThreatModel.STRIDE = {
    SPOOFING = "S",
    TAMPERING = "T",
    REPUDIATION = "R",
    INFORMATION_DISCLOSURE = "I",
    DENIAL_OF_SERVICE = "D",
    ELEVATION_OF_PRIVILEGE = "E"
}

ThreatModel.RISK_LEVEL = {
    CRITICAL = 4,
    HIGH = 3,
    MEDIUM = 2,
    LOW = 1
}

function ThreatModel.new(system_name)
    return setmetatable({
        system = system_name,
        components = {},
        data_flows = {},
        threats = {},
        mitigations = {}
    }, ThreatModel)
end

function ThreatModel:add_component(name, type, trust_level)
    self.components[name] = {
        name = name,
        type = type,  -- "process", "datastore", "external"
        trust_level = trust_level or "medium",
        threats = {}
    }
end

function ThreatModel:add_data_flow(from, to, data_type, encrypted)
    table.insert(self.data_flows, {
        from = from,
        to = to,
        data_type = data_type,
        encrypted = encrypted or false
    })
end

function ThreatModel:add_threat(component, stride_type, description, risk, mitigation)
    local threat = {
        id = string.format("T%03d", #self.threats + 1),
        component = component,
        stride = stride_type,
        description = description,
        risk = risk or ThreatModel.RISK_LEVEL.MEDIUM,
        mitigation = mitigation,
        status = "open"
    }
    
    table.insert(self.threats, threat)
    
    if self.components[component] then
        table.insert(self.components[component].threats, threat.id)
    end
    
    return threat.id
end

function ThreatModel:generate_report()
    print(string.format("\n=== Threat Model Report: %s ===", self.system))
    print(string.format("Components: %d | Data Flows: %d | Threats: %d",
                       self:count_table(self.components),
                       #self.data_flows,
                       #self.threats))
    
    -- Group by risk level
    local by_risk = {
        [ThreatModel.RISK_LEVEL.CRITICAL] = {},
        [ThreatModel.RISK_LEVEL.HIGH] = {},
        [ThreatModel.RISK_LEVEL.MEDIUM] = {},
        [ThreatModel.RISK_LEVEL.LOW] = {}
    }
    
    for _, threat in ipairs(self.threats) do
        table.insert(by_risk[threat.risk], threat)
    end
    
    local risk_names = {[4]="CRITICAL", [3]="HIGH", [2]="MEDIUM", [1]="LOW"}
    
    for risk = 4, 1, -1 do
        local threats = by_risk[risk]
        if #threats > 0 then
            print(string.format("\n[%s] %d threat(s):", risk_names[risk], #threats))
            for _, t in ipairs(threats) do
                print(string.format("  %s [%s] %s: %s",
                                   t.id, t.stride, t.component, t.description))
                if t.mitigation then
                    print(string.format("     Mitigation: %s", t.mitigation))
                end
            end
        end
    end
end

function ThreatModel:count_table(t)
    local count = 0
    for _ in pairs(t) do count = count + 1 end
    return count
end

-- ตัวอย่าง: Threat Model สำหรับ Web Application
local model = ThreatModel.new("Lua Web Application")

model:add_component("WebServer", "process", "medium")
model:add_component("Database", "datastore", "high")
model:add_component("User", "external", "low")
model:add_component("AdminPanel", "process", "high")

model:add_data_flow("User", "WebServer", "HTTP Request", false)
model:add_data_flow("WebServer", "Database", "SQL Query", true)

-- Add threats
model:add_threat("WebServer", "S", 
    "Request spoofing via forged headers",
    ThreatModel.RISK_LEVEL.HIGH,
    "Validate X-Forwarded-For against whitelist, use mTLS")

model:add_threat("Database", "T",
    "SQL injection via user input",
    ThreatModel.RISK_LEVEL.CRITICAL,
    "Use parameterized queries, input validation")

model:add_threat("WebServer", "I",
    "Sensitive data in error messages",
    ThreatModel.RISK_LEVEL.MEDIUM,
    "Generic error messages in production, detailed logs server-side only")

model:add_threat("WebServer", "D",
    "Resource exhaustion via slow HTTP requests",
    ThreatModel.RISK_LEVEL.HIGH,
    "Request timeout limits, rate limiting, connection limits")

model:add_threat("AdminPanel", "E",
    "Privilege escalation via path traversal",
    ThreatModel.RISK_LEVEL.CRITICAL,
    "Validate all paths against whitelist, sandboxed execution")

model:generate_report()
```

## Code Injection via load()

```lua
-- ตัวอย่างที่ 2: Dangers of load() and loadstring()
-- VULNERABLE: Never do this!
local function vulnerable_eval(user_input)
    -- DANGER: user_input can contain malicious code
    local fn, err = load(user_input)
    if fn then
        return fn()
    end
end

-- ตัวอย่างการ exploit:
-- vulnerable_eval("os.execute('rm -rf /')")  -- NEVER RUN THIS
-- vulnerable_eval("io.open('/etc/passwd'):read('*a')")

-- SAFE: Whitelist approach
local function safe_eval_expression(expr)
    -- Only allow arithmetic and specific functions
    local safe_env = {
        math = {
            abs = math.abs, ceil = math.ceil, floor = math.floor,
            max = math.max, min = math.min, sqrt = math.sqrt,
            sin = math.sin, cos = math.cos, pi = math.pi
        },
        -- Explicitly NO os, io, require, load, etc.
        tostring = tostring,
        tonumber = tonumber
    }
    
    -- Validate expression doesn't contain dangerous patterns
    local dangerous_patterns = {
        "load", "loadstring", "loadfile", "dofile",
        "require", "pcall", "xpcall", "rawget", "rawset",
        "getmetatable", "setmetatable", "debug", "io", "os",
        "package", "coroutine", "collectgarbage"
    }
    
    local lower_expr = expr:lower()
    for _, pattern in ipairs(dangerous_patterns) do
        if lower_expr:find(pattern, 1, true) then
            return nil, "Dangerous keyword: " .. pattern
        end
    end
    
    -- Only allow safe characters
    if expr:match("[^%d%s%+%-%*%/%^%(%)%.,]") then
        return nil, "Invalid characters in expression"
    end
    
    local fn, err = load("return " .. expr, "safe_eval", "t", safe_env)
    if not fn then
        return nil, "Syntax error: " .. (err or "unknown")
    end
    
    local ok, result = pcall(fn)
    if not ok then
        return nil, "Execution error: " .. tostring(result)
    end
    
    return result
end

-- ตัวอย่างการใช้งาน safe_eval
local test_expressions = {
    "2 + 2",               -- OK: 4
    "math.sqrt(16)",       -- OK: 4
    "math.pi * 5^2",       -- OK: 78.54
    "os.execute('ls')",    -- BLOCKED
    "load('return 1')",    -- BLOCKED
    "require('socket')",   -- BLOCKED
}

print("\n=== Safe Expression Evaluator ===")
for _, expr in ipairs(test_expressions) do
    local result, err = safe_eval_expression(expr)
    if result then
        print(string.format("  '%s' => %s", expr, tostring(result)))
    else
        print(string.format("  '%s' => BLOCKED: %s", expr, err))
    end
end
```

## Sandbox Implementation

```lua
-- ตัวอย่างที่ 3: Comprehensive Lua Sandbox
local Sandbox = {}
Sandbox.__index = Sandbox

function Sandbox.new(options)
    options = options or {}
    return setmetatable({
        allowed_modules = options.modules or {},
        cpu_limit = options.cpu_limit or 1000000,  -- instruction count
        memory_limit = options.memory_limit or 1024 * 1024,  -- 1MB
        time_limit = options.time_limit or 5,  -- seconds
        io_access = options.io_access or false,
        network_access = options.network_access or false,
        execution_count = 0
    }, Sandbox)
end

function Sandbox:create_env()
    -- Minimal safe environment
    local safe_env = {
        -- Basic types and conversions
        type = type,
        tostring = tostring,
        tonumber = tonumber,
        ipairs = ipairs,
        pairs = pairs,
        next = next,
        select = select,
        unpack = table.unpack or unpack,
        
        -- Safe string operations
        string = {
            byte = string.byte,
            char = string.char,
            find = string.find,
            format = string.format,
            gmatch = string.gmatch,
            gsub = string.gsub,
            len = string.len,
            lower = string.lower,
            match = string.match,
            rep = string.rep,
            reverse = string.reverse,
            sub = string.sub,
            upper = string.upper
        },
        
        -- Safe table operations
        table = {
            concat = table.concat,
            insert = table.insert,
            remove = table.remove,
            sort = table.sort,
            unpack = table.unpack or unpack
        },
        
        -- Safe math operations
        math = {
            abs = math.abs, ceil = math.ceil, floor = math.floor,
            max = math.max, min = math.min, sqrt = math.sqrt,
            sin = math.sin, cos = math.cos, tan = math.tan,
            pi = math.pi, huge = math.huge, random = math.random,
            exp = math.exp, log = math.log, pow = math.pow
        },
        
        -- Error handling (safe)
        error = error,
        assert = assert,
        pcall = pcall,
        
        -- Print (captured, not to stdout by default)
        print = function(...)
            -- Override to capture output
        end
    }
    
    -- Explicitly blocked
    -- No: os, io, require, load, loadfile, dofile
    -- No: debug, package, coroutine (unless specifically allowed)
    -- No: getmetatable, setmetatable (prevent sandbox escape)
    
    return safe_env
end

function Sandbox:execute(code, input_data)
    local env = self:create_env()
    
    -- Add input data to environment
    if input_data then
        for k, v in pairs(input_data) do
            env[k] = v
        end
    end
    
    -- Capture output
    local output = {}
    env.print = function(...)
        local args = {}
        for i = 1, select('#', ...) do
            table.insert(args, tostring(select(i, ...)))
        end
        table.insert(output, table.concat(args, "\t"))
    end
    
    -- Instruction count limiter
    local instructions = 0
    local cpu_limit = self.cpu_limit
    
    debug.sethook(function()
        instructions = instructions + 1
        if instructions > cpu_limit then
            error("CPU limit exceeded")
        end
    end, "", 100)
    
    -- Compile
    local fn, err = load(code, "sandbox", "t", env)
    if not fn then
        debug.sethook()
        return nil, "Compile error: " .. (err or "unknown")
    end
    
    -- Execute with timeout simulation
    local start = os.clock()
    local ok, result = pcall(fn)
    local elapsed = os.clock() - start
    
    debug.sethook()
    
    if not ok then
        if result:find("CPU limit exceeded") then
            return nil, "TIMEOUT: CPU limit exceeded"
        end
        return nil, "Runtime error: " .. tostring(result)
    end
    
    return {
        result = result,
        output = output,
        elapsed_ms = elapsed * 1000,
        instructions = instructions
    }
end

-- ตัวอย่างการใช้งาน Sandbox
local sb = Sandbox.new({cpu_limit = 100000})

local test_codes = {
    -- Safe code
    [[
        local sum = 0
        for i = 1, 100 do
            sum = sum + i
        end
        return sum
    ]],
    
    -- Safe string manipulation
    [[
        local s = "Hello, World!"
        return string.upper(string.reverse(s))
    ]],
    
    -- Attempt to escape sandbox
    [[
        return os.execute("ls")
    ]],
    
    -- Infinite loop (should hit CPU limit)
    [[
        local i = 0
        while true do i = i + 1 end
        return i
    ]]
}

print("\n=== Sandbox Execution ===")
for i, code in ipairs(test_codes) do
    local result, err = sb:execute(code)
    if result then
        print(string.format("Code %d: OK (result=%s, %.2fms, %d instructions)",
                           i, tostring(result.result), 
                           result.elapsed_ms, result.instructions))
    else
        print(string.format("Code %d: ERROR - %s", i, err))
    end
end
```

## Sandbox Escape Techniques and Defenses

```lua
-- ตัวอย่างที่ 4: Common Sandbox Escape Attempts
-- (Educational purposes - understanding attack vectors)

-- ATTACK 1: getmetatable escape
local function demo_metatable_escape_attempt()
    -- Attacker tries to use string metatable to access original env
    local attempt = [[
        local s = ""
        local mt = getmetatable(s)
        -- Try to get __index which points to string library
        local string_lib = mt.__index
        -- Try to access global string library and from there...
    ]]
    print("Metatable escape blocked by removing getmetatable from sandbox env")
end

-- ATTACK 2: coroutine escape
local function demo_coroutine_escape_attempt()
    local attempt = [[
        -- Coroutines share the same environment in some versions
        -- Attackers might try to use coroutine.wrap to bypass hooks
        local co = coroutine.wrap(function()
            -- might escape debug hook here
        end)
    ]]
    print("Coroutine escape blocked by removing coroutine from sandbox env")
end

-- DEFENSE: Strict sandbox with all escape vectors closed
local function create_strict_sandbox()
    local env = {}
    
    -- Only add explicitly safe functions
    env.math = {abs = math.abs, floor = math.floor, ceil = math.ceil}
    env.string = {format = string.format, sub = string.sub}
    env.table = {insert = table.insert, concat = table.concat}
    env.type = type
    env.tostring = tostring
    env.tonumber = tonumber
    env.ipairs = ipairs
    env.pairs = pairs
    env.error = error
    env.assert = assert
    
    -- Explicitly block everything else
    env._G = nil  -- No global access
    env.getmetatable = nil
    env.setmetatable = nil
    env.rawget = nil
    env.rawset = nil
    env.rawequal = nil
    env.load = nil
    env.loadstring = nil
    env.loadfile = nil
    env.dofile = nil
    env.require = nil
    env.pcall = nil  -- Even pcall can be dangerous in some contexts
    env.xpcall = nil
    env.debug = nil
    env.package = nil
    env.io = nil
    env.os = nil
    env.coroutine = nil
    
    return env
end

-- ตัวอย่างที่ 5: Input Validation and Sanitization
local Validator = {}
Validator.__index = Validator

function Validator.new()
    return setmetatable({
        rules = {}
    }, Validator)
end

function Validator:add_rule(field, checks)
    self.rules[field] = checks
end

function Validator:validate(data)
    local errors = {}
    
    for field, checks in pairs(self.rules) do
        local value = data[field]
        
        for _, check in ipairs(checks) do
            local ok, err = check(value, field, data)
            if not ok then
                if not errors[field] then
                    errors[field] = {}
                end
                table.insert(errors[field], err)
            end
        end
    end
    
    local has_errors = next(errors) ~= nil
    return not has_errors, errors
end

-- Validation helpers
local V = {}

function V.required(value, field)
    if value == nil or value == "" then
        return false, field .. " is required"
    end
    return true
end

function V.min_length(min)
    return function(value, field)
        if type(value) ~= "string" or #value < min then
            return false, string.format("%s must be at least %d characters", field, min)
        end
        return true
    end
end

function V.max_length(max)
    return function(value, field)
        if type(value) == "string" and #value > max then
            return false, string.format("%s must be at most %d characters", field, max)
        end
        return true
    end
end

function V.pattern(pat, message)
    return function(value, field)
        if type(value) ~= "string" or not value:match(pat) then
            return false, message or (field .. " has invalid format")
        end
        return true
    end
end

function V.integer(value, field)
    if type(value) ~= "number" or math.floor(value) ~= value then
        return false, field .. " must be an integer"
    end
    return true
end

function V.range(min, max)
    return function(value, field)
        if type(value) ~= "number" or value < min or value > max then
            return false, string.format("%s must be between %d and %d", field, min, max)
        end
        return true
    end
end

function V.email(value, field)
    if type(value) ~= "string" or 
       not value:match("^[%w.+%-]+@[%w%-]+%.[%a]+$") then
        return false, field .. " must be a valid email"
    end
    return true
end

function V.no_sql_injection(value, field)
    if type(value) ~= "string" then return true end
    
    local sql_patterns = {
        "'", "--", ";%s*drop", ";%s*delete", ";%s*insert",
        ";%s*update", "union%s+select", "exec%s*%(", "xp_cmdshell"
    }
    
    local lower_val = value:lower()
    for _, pattern in ipairs(sql_patterns) do
        if lower_val:find(pattern) then
            return false, field .. " contains invalid characters"
        end
    end
    return true
end

function V.no_xss(value, field)
    if type(value) ~= "string" then return true end
    
    local xss_patterns = {
        "<script", "javascript:", "on%a+%s*=", "<iframe",
        "document%.cookie", "window%.location"
    }
    
    local lower_val = value:lower()
    for _, pattern in ipairs(xss_patterns) do
        if lower_val:find(pattern) then
            return false, field .. " contains potentially unsafe content"
        end
    end
    return true
end

-- ตัวอย่างการใช้งาน Validator
local validator = Validator.new()

validator:add_rule("username", {
    V.required,
    V.min_length(3),
    V.max_length(50),
    V.pattern("^[%w_%-]+$", "Username can only contain letters, numbers, _ and -"),
    V.no_sql_injection,
    V.no_xss
})

validator:add_rule("email", {
    V.required,
    V.email,
    V.max_length(255)
})

validator:add_rule("age", {
    V.required,
    V.integer,
    V.range(13, 120)
})

local test_inputs = {
    {username = "alice_123", email = "alice@example.com", age = 25},  -- valid
    {username = "a", email = "not-an-email", age = 200},               -- invalid
    {username = "'; DROP TABLE users; --", email = "x@y.com", age = 20},  -- SQL injection
    {username = "<script>alert(1)</script>", email = "x@y.com", age = 20}  -- XSS
}

print("\n=== Input Validation ===")
for i, input in ipairs(test_inputs) do
    local ok, errors = validator:validate(input)
    if ok then
        print(string.format("Input %d: VALID", i))
    else
        print(string.format("Input %d: INVALID", i))
        for field, errs in pairs(errors) do
            for _, err in ipairs(errs) do
                print(string.format("  - %s", err))
            end
        end
    end
end
```

## Memory Safety

```lua
-- ตัวอย่างที่ 6: Memory Safety Patterns
-- Safe string operations that prevent buffer overruns
local SafeString = {}

function SafeString.safe_concat(strings, max_length)
    max_length = max_length or 65536  -- 64KB default limit
    local parts = {}
    local total = 0
    
    for _, s in ipairs(strings) do
        local str = tostring(s)
        if total + #str > max_length then
            -- Truncate to fit
            local remaining = max_length - total
            if remaining > 0 then
                table.insert(parts, str:sub(1, remaining))
            end
            break
        end
        table.insert(parts, str)
        total = total + #str
    end
    
    return table.concat(parts), total
end

function SafeString.safe_format(fmt, ...)
    -- Validate format string doesn't have unbounded specifiers
    local count = 0
    for _ in fmt:gmatch("%%[^%%]") do
        count = count + 1
    end
    
    local args = {...}
    if count > #args then
        error(string.format("Format string expects %d args, got %d", count, #args))
    end
    
    -- Limit each string argument
    local safe_args = {}
    for _, arg in ipairs(args) do
        if type(arg) == "string" and #arg > 1000 then
            table.insert(safe_args, arg:sub(1, 1000) .. "...[truncated]")
        else
            table.insert(safe_args, arg)
        end
    end
    
    return string.format(fmt, table.unpack(safe_args))
end

-- ตัวอย่างที่ 7: Secure Secret Handling
local SecretManager = {}
SecretManager.__index = SecretManager

function SecretManager.new()
    local self = setmetatable({
        secrets = {},
        _internal = {}
    }, SecretManager)
    
    -- Prevent secrets from being accessed via standard table operations
    local mt = getmetatable(self)
    mt.__pairs = function() error("Cannot iterate secrets") end
    mt.__tostring = function() return "[SecretManager]" end
    
    return self
end

function SecretManager:store(name, value)
    if type(value) ~= "string" then
        error("Secrets must be strings")
    end
    
    -- Validate name (alphanumeric and underscores only)
    if not name:match("^[%w_]+$") then
        error("Invalid secret name: " .. name)
    end
    
    self.secrets[name] = value
end

function SecretManager:get(name)
    local secret = self.secrets[name]
    if not secret then
        return nil  -- Don't error, just return nil
    end
    return secret
end

function SecretManager:rotate(name, new_value)
    if not self.secrets[name] then
        error("Secret not found: " .. name)
    end
    
    -- Securely overwrite old value
    local old = self.secrets[name]
    -- In real Lua, we can't truly zero memory, but we can make it harder
    self.secrets[name] = string.rep("\0", #old)  -- overwrite
    self.secrets[name] = new_value  -- set new
    
    return true
end

function SecretManager:mask(name, show_chars)
    local secret = self.secrets[name]
    if not secret then return nil end
    
    show_chars = show_chars or 4
    if #secret <= show_chars * 2 then
        return string.rep("*", #secret)
    end
    
    return secret:sub(1, show_chars) .. 
           string.rep("*", #secret - show_chars * 2) .. 
           secret:sub(-show_chars)
end

-- ตัวอย่างการใช้งาน
local secrets = SecretManager.new()
secrets:store("DB_PASSWORD", "super_secret_password_123")
secrets:store("API_KEY", "sk_live_abcdefghijklmnop")

print("\n=== Secure Secret Handling ===")
print("DB Password (masked):", secrets:mask("DB_PASSWORD"))
print("API Key (masked):", secrets:mask("API_KEY", 6))
print("Non-existent:", tostring(secrets:get("FAKE_KEY")))
```

## Side-Channel Attacks

```lua
-- ตัวอย่างที่ 8: Timing Attack Demonstration and Prevention
-- VULNERABLE: String comparison susceptible to timing attacks
local function vulnerable_compare(a, b)
    -- This returns early on first mismatch, leaking timing information
    return a == b
end

-- ตัวอย่างการ exploit timing attack
-- An attacker can measure how long the comparison takes
-- If it takes longer, more characters matched -> can guess character by character

-- SAFE: Constant-time comparison
local function constant_time_compare(a, b)
    if type(a) ~= "string" or type(b) ~= "string" then
        return false
    end
    
    -- Must be same length (don't leak length via timing)
    if #a ~= #b then
        return false
    end
    
    local result = 0
    for i = 1, #a do
        -- XOR bytes - result is 0 only if all bytes match
        result = result | (a:byte(i) ~ b:byte(i))
    end
    
    -- Extra dummy work to prevent compiler optimization
    for i = 1, 10 do
        result = result | 0
    end
    
    return result == 0
end

-- ตัวอย่างที่ 9: Timing Attack on HMAC Verification
local function unsafe_verify_hmac(message, provided_hmac, secret)
    -- Simulate HMAC computation
    local computed = string.format("%x", 
        (#message + #secret) * 31337 % 2^32)
    
    -- VULNERABLE: direct comparison
    return computed == provided_hmac
end

local function safe_verify_hmac(message, provided_hmac, secret)
    -- Compute expected HMAC
    local computed = string.format("%x",
        (#message + #secret) * 31337 % 2^32)
    
    -- SAFE: constant-time comparison
    return constant_time_compare(computed, provided_hmac)
end

-- ตัวอย่างที่ 10: Timing Attack Measurement Simulation
local function measure_timing(fn, ...)
    local iterations = 1000
    local total = 0
    
    for _ = 1, iterations do
        local start = os.clock()
        fn(...)
        local elapsed = os.clock() - start
        total = total + elapsed
    end
    
    return total / iterations
end

print("\n=== Timing Attack Demonstration ===")

-- Simulate timing difference (in real attack, differences are in nanoseconds)
local correct_token = "secret_token_abc123"
local wrong_token_1 = "wrong_1_____________"  -- different first char
local wrong_token_2 = "secret_token_abc___"   -- differs near end

print("Vulnerable comparison timings (normalized):")
local t1 = measure_timing(vulnerable_compare, correct_token, wrong_token_1)
local t2 = measure_timing(vulnerable_compare, correct_token, wrong_token_2)
print(string.format("  Early mismatch: %.6f", t1))
print(string.format("  Late mismatch:  %.6f (might be slightly different)", t2))

print("\nConstant-time comparison timings:")
local ct1 = measure_timing(constant_time_compare, correct_token, wrong_token_1)
local ct2 = measure_timing(constant_time_compare, correct_token, wrong_token_2)
print(string.format("  Early mismatch: %.6f", ct1))
print(string.format("  Late mismatch:  %.6f (should be similar)", ct2))
```

## Secure Coding Guidelines

```lua
-- ตัวอย่างที่ 11: Secure Password Hashing
-- Note: In production, use proper crypto library like luacrypto or openssl

local function pbkdf2_simulate(password, salt, iterations, key_length)
    -- This is a SIMULATION for educational purposes
    -- In production, use a proper PBKDF2 implementation
    local result = password .. salt
    for i = 1, iterations do
        -- Simulate hashing (NOT cryptographically secure)
        local hash = 0
        for j = 1, #result do
            hash = (hash * 31 + result:byte(j)) % (2^32)
        end
        result = string.format("%08x", hash) .. result:sub(1, -9)
    end
    return result:sub(1, key_length * 2)
end

local function hash_password(password, options)
    options = options or {}
    
    -- Generate random salt
    math.randomseed(os.time())
    local salt_bytes = {}
    for i = 1, 32 do
        table.insert(salt_bytes, math.random(0, 255))
    end
    local salt = string.char(table.unpack(salt_bytes))
    
    local iterations = options.iterations or 100000
    local key_length = options.key_length or 32
    local algorithm = options.algorithm or "pbkdf2-sha256"
    
    local hash = pbkdf2_simulate(password, salt, math.floor(iterations/1000), key_length)
    
    -- Format: algorithm$iterations$salt_hex$hash_hex
    local salt_hex = ""
    for i = 1, #salt do
        salt_hex = salt_hex .. string.format("%02x", salt:byte(i))
    end
    
    return string.format("%s$%d$%s$%s", algorithm, iterations, salt_hex, hash)
end

local function verify_password(password, stored_hash)
    -- Parse stored hash
    local algorithm, iterations, salt_hex, expected_hash = 
        stored_hash:match("^([^$]+)%$(%d+)%$([^$]+)%$(.+)$")
    
    if not algorithm then
        return false, "Invalid hash format"
    end
    
    -- Reconstruct salt from hex
    local salt = ""
    for hex in salt_hex:gmatch("(%x%x)") do
        salt = salt .. string.char(tonumber(hex, 16))
    end
    
    -- Recompute hash
    local key_length = #expected_hash / 2
    local computed = pbkdf2_simulate(password, salt, 
                                     math.floor(tonumber(iterations)/1000), 
                                     key_length)
    
    -- Constant-time comparison
    return constant_time_compare(computed, expected_hash)
end

print("\n=== Secure Password Hashing ===")
local password = "MySecurePassword123!"
local stored = hash_password(password)
print("Stored hash:", stored:sub(1, 50) .. "...")

local ok = verify_password(password, stored)
print("Verify correct password:", ok)

local wrong_ok = verify_password("WrongPassword", stored)
print("Verify wrong password:", wrong_ok)
```

## Dependency Security

```lua
-- ตัวอย่างที่ 12: Dependency Security Scanner
local DependencyScanner = {}
DependencyScanner.__index = DependencyScanner

-- Simulated CVE database
local CVE_DATABASE = {
    ["luasocket"] = {
        {version = "2.0", cve = "CVE-2023-1234", severity = "HIGH",
         description = "Buffer overflow in socket handling",
         fixed_in = "3.0.0"},
    },
    ["lua-cjson"] = {
        {version = "2.1.0", cve = "CVE-2022-5678", severity = "MEDIUM",
         description = "Integer overflow in JSON parsing",
         fixed_in = "2.1.1"},
    }
}

function DependencyScanner.new()
    return setmetatable({
        dependencies = {},
        scan_results = {}
    }, DependencyScanner)
end

function DependencyScanner:add_dependency(name, version, source)
    table.insert(self.dependencies, {
        name = name,
        version = version,
        source = source or "luarocks"
    })
end

function DependencyScanner:scan()
    self.scan_results = {
        critical = {},
        high = {},
        medium = {},
        low = {},
        clean = {}
    }
    
    for _, dep in ipairs(self.dependencies) do
        local cves = CVE_DATABASE[dep.name]
        local found_vulnerabilities = {}
        
        if cves then
            for _, cve in ipairs(cves) do
                -- Check if current version is vulnerable
                if dep.version == cve.version or 
                   self:version_less_than(dep.version, cve.fixed_in) then
                    table.insert(found_vulnerabilities, cve)
                end
            end
        end
        
        if #found_vulnerabilities > 0 then
            local severity = "low"
            for _, v in ipairs(found_vulnerabilities) do
                if v.severity == "CRITICAL" then severity = "critical"
                elseif v.severity == "HIGH" and severity ~= "critical" then severity = "high"
                elseif v.severity == "MEDIUM" and 
                       severity ~= "critical" and severity ~= "high" then
                    severity = "medium"
                end
            end
            
            table.insert(self.scan_results[severity], {
                dep = dep,
                vulnerabilities = found_vulnerabilities
            })
        else
            table.insert(self.scan_results.clean, dep)
        end
    end
    
    return self.scan_results
end

function DependencyScanner:version_less_than(v1, v2)
    -- Simple version comparison
    local function parse(v)
        local parts = {}
        for p in v:gmatch("(%d+)") do
            table.insert(parts, tonumber(p))
        end
        return parts
    end
    
    local p1 = parse(v1)
    local p2 = parse(v2)
    
    for i = 1, math.max(#p1, #p2) do
        local a = p1[i] or 0
        local b = p2[i] or 0
        if a < b then return true
        elseif a > b then return false
        end
    end
    return false
end

function DependencyScanner:report()
    print("\n=== Dependency Security Scan ===")
    
    local total_vulns = 0
    for _, severity in ipairs({"critical", "high", "medium", "low"}) do
        local items = self.scan_results[severity]
        if #items > 0 then
            total_vulns = total_vulns + #items
            print(string.format("\n[%s] %d vulnerable package(s):", 
                               severity:upper(), #items))
            for _, item in ipairs(items) do
                print(string.format("  %s@%s", item.dep.name, item.dep.version))
                for _, v in ipairs(item.vulnerabilities) do
                    print(string.format("    %s (%s): %s", v.cve, v.severity, v.description))
                    print(string.format("    Fix: upgrade to %s", v.fixed_in))
                end
            end
        end
    end
    
    print(string.format("\n%d clean packages, %d vulnerable",
                       #self.scan_results.clean, total_vulns))
end

-- ตัวอย่างการใช้งาน
local scanner = DependencyScanner.new()
scanner:add_dependency("luasocket", "2.0")
scanner:add_dependency("lua-cjson", "2.1.0")
scanner:add_dependency("luafilesystem", "1.8.0")
scanner:add_dependency("copas", "3.0.0")

scanner:scan()
scanner:report()
```

## OWASP Top 10 in Lua Context

```lua
-- ตัวอย่างที่ 13: OWASP A01 - Broken Access Control
local AccessControl = {}
AccessControl.__index = AccessControl

function AccessControl.new()
    return setmetatable({
        policies = {},
        roles = {},
        user_roles = {}
    }, AccessControl)
end

function AccessControl:define_role(name, permissions)
    self.roles[name] = permissions
end

function AccessControl:assign_role(user_id, role)
    if not self.roles[role] then
        error("Unknown role: " .. role)
    end
    if not self.user_roles[user_id] then
        self.user_roles[user_id] = {}
    end
    table.insert(self.user_roles[user_id], role)
end

function AccessControl:can(user_id, action, resource)
    local user_role_list = self.user_roles[user_id]
    if not user_role_list then return false end
    
    for _, role_name in ipairs(user_role_list) do
        local perms = self.roles[role_name]
        if perms then
            -- Check for wildcard permission
            if perms["*:*"] then return true end
            
            -- Check for exact match
            local key = action .. ":" .. resource
            if perms[key] then return true end
            
            -- Check for action wildcard
            if perms[action .. ":*"] then return true end
            
            -- Check for resource wildcard  
            if perms["*:" .. resource] then return true end
        end
    end
    
    return false
end

function AccessControl:enforce(user_id, action, resource)
    if not self:can(user_id, action, resource) then
        error(string.format("FORBIDDEN: User %s cannot %s on %s",
                           user_id, action, resource))
    end
end

-- ตัวอย่างที่ 14: OWASP A02 - Cryptographic Failures
local function check_sensitive_data_exposure(config)
    local issues = {}
    
    -- Check for weak encryption
    if config.encryption and config.encryption == "md5" then
        table.insert(issues, "CRITICAL: MD5 is not suitable for password hashing")
    end
    
    if config.encryption and config.encryption == "sha1" then
        table.insert(issues, "HIGH: SHA-1 is deprecated for security use")
    end
    
    -- Check for HTTPS
    if config.protocol and config.protocol == "http" then
        table.insert(issues, "HIGH: HTTP transmits data in cleartext")
    end
    
    -- Check for TLS version
    if config.tls_version and config.tls_version < 1.2 then
        table.insert(issues, "HIGH: TLS " .. config.tls_version .. " is deprecated")
    end
    
    -- Check cookie security
    if config.cookies then
        if not config.cookies.secure then
            table.insert(issues, "MEDIUM: Cookies not marked as Secure")
        end
        if not config.cookies.httponly then
            table.insert(issues, "MEDIUM: Cookies not marked as HttpOnly")
        end
        if not config.cookies.samesite then
            table.insert(issues, "MEDIUM: Cookies missing SameSite attribute")
        end
    end
    
    return issues
end

-- ตัวอย่างที่ 15: OWASP A03 - Injection Prevention
local function parameterized_query(template, params)
    -- Safe: parameters are escaped and cannot break out of string context
    local query = template
    local param_values = {}
    
    for i, param in ipairs(params) do
        local escaped
        if type(param) == "string" then
            -- Escape single quotes
            escaped = "'" .. param:gsub("'", "''") .. "'"
        elseif type(param) == "number" then
            escaped = tostring(param)
        elseif type(param) == "boolean" then
            escaped = param and "TRUE" or "FALSE"
        elseif param == nil then
            escaped = "NULL"
        else
            error("Unsupported parameter type: " .. type(param))
        end
        
        table.insert(param_values, escaped)
    end
    
    -- Replace placeholders
    local i = 0
    query = query:gsub("%$%d+", function()
        i = i + 1
        return param_values[i] or "NULL"
    end)
    
    return query
end

-- Test injection prevention
print("\n=== SQL Injection Prevention ===")
local template = "SELECT * FROM users WHERE username = $1 AND age > $2"

local safe_query = parameterized_query(template, {"alice", 18})
print("Safe query:", safe_query)

local injection_attempt = parameterized_query(template, 
    {"'; DROP TABLE users; --", 18})
print("Injection attempt (escaped):", injection_attempt)

-- ตัวอย่างที่ 16: OWASP A07 - Identification and Authentication Failures
local AuthSystem = {}
AuthSystem.__index = AuthSystem

function AuthSystem.new()
    return setmetatable({
        users = {},
        sessions = {},
        failed_attempts = {},
        lockout_threshold = 5,
        lockout_duration = 900  -- 15 minutes
    }, AuthSystem)
end

function AuthSystem:register(username, password)
    if self.users[username] then
        error("Username already exists")
    end
    
    -- Validate password strength
    local strength_issues = self:check_password_strength(password)
    if #strength_issues > 0 then
        error("Weak password: " .. table.concat(strength_issues, ", "))
    end
    
    self.users[username] = {
        password_hash = hash_password(password),
        created_at = os.time(),
        mfa_enabled = false,
        last_login = nil
    }
end

function AuthSystem:check_password_strength(password)
    local issues = {}
    
    if #password < 12 then
        table.insert(issues, "Too short (min 12 chars)")
    end
    if not password:match("[A-Z]") then
        table.insert(issues, "Missing uppercase letter")
    end
    if not password:match("[a-z]") then
        table.insert(issues, "Missing lowercase letter")
    end
    if not password:match("[0-9]") then
        table.insert(issues, "Missing digit")
    end
    if not password:match("[^%w]") then
        table.insert(issues, "Missing special character")
    end
    
    -- Check common passwords
    local common_passwords = {"password123!", "Password123!", "Admin@1234"}
    for _, common in ipairs(common_passwords) do
        if password == common then
            table.insert(issues, "Password is too common")
        end
    end
    
    return issues
end

function AuthSystem:login(username, password, ip_address)
    -- Check rate limiting
    local attempts = self.failed_attempts[ip_address] or {count = 0, last = 0}
    
    if attempts.count >= self.lockout_threshold then
        local elapsed = os.time() - attempts.last
        if elapsed < self.lockout_duration then
            return nil, string.format("Account locked. Try again in %d minutes", 
                                     math.ceil((self.lockout_duration - elapsed) / 60))
        else
            -- Reset after lockout duration
            self.failed_attempts[ip_address] = {count = 0, last = 0}
        end
    end
    
    local user = self.users[username]
    
    -- Always do password check (prevent username enumeration)
    local hash_to_check = user and user.password_hash or hash_password("dummy")
    local ok = verify_password(password, hash_to_check)
    
    if not ok or not user then
        -- Record failed attempt
        local fa = self.failed_attempts[ip_address] or {count = 0}
        fa.count = fa.count + 1
        fa.last = os.time()
        self.failed_attempts[ip_address] = fa
        
        -- Generic error (don't reveal if username exists)
        return nil, "Invalid credentials"
    end
    
    -- Successful login
    self.failed_attempts[ip_address] = nil
    user.last_login = os.time()
    
    -- Generate secure session token
    local token = self:generate_session_token()
    self.sessions[token] = {
        username = username,
        ip = ip_address,
        created_at = os.time(),
        expires_at = os.time() + 86400  -- 24 hours
    }
    
    return token
end

function AuthSystem:generate_session_token()
    -- In production, use a cryptographically secure random generator
    local chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    local token = ""
    math.randomseed(os.time() + math.random())
    for _ = 1, 64 do
        local idx = math.random(1, #chars)
        token = token .. chars:sub(idx, idx)
    end
    return token
end

function AuthSystem:validate_session(token, ip_address)
    local session = self.sessions[token]
    if not session then return false, "Invalid session" end
    
    if os.time() > session.expires_at then
        self.sessions[token] = nil
        return false, "Session expired"
    end
    
    -- Optional: IP binding
    -- if session.ip ~= ip_address then
    --     return false, "Session IP mismatch"
    -- end
    
    return true, session.username
end
```

## Fuzzing Basics

```lua
-- ตัวอย่างที่ 17: Simple Fuzzer
local Fuzzer = {}
Fuzzer.__index = Fuzzer

function Fuzzer.new(options)
    options = options or {}
    return setmetatable({
        max_iterations = options.max_iterations or 1000,
        seed = options.seed or os.time(),
        mutation_rate = options.mutation_rate or 0.1,
        crashes = {},
        interesting = {}
    }, Fuzzer)
end

function Fuzzer:generate_string(base, max_len)
    max_len = max_len or 100
    local len = math.random(0, max_len)
    local result = {}
    
    for i = 1, len do
        local roll = math.random()
        if roll < 0.7 and base and i <= #base then
            -- Keep original character with mutation chance
            if math.random() < self.mutation_rate then
                table.insert(result, string.char(math.random(0, 255)))
            else
                table.insert(result, base:sub(i, i))
            end
        else
            -- Random character
            local char_type = math.random(1, 5)
            if char_type == 1 then
                -- Special chars
                local specials = "!@#$%^&*(){}[]|\\;'\"<>?/.,`~"
                table.insert(result, specials:sub(math.random(1, #specials), 
                                                   math.random(1, #specials)))
            elseif char_type == 2 then
                -- Null bytes and control chars
                table.insert(result, string.char(math.random(0, 31)))
            elseif char_type == 3 then
                -- High bytes
                table.insert(result, string.char(math.random(128, 255)))
            else
                -- Normal printable
                table.insert(result, string.char(math.random(32, 126)))
            end
        end
    end
    
    return table.concat(result)
end

function Fuzzer:fuzz(target_fn, base_inputs, validator)
    math.randomseed(self.seed)
    
    print(string.format("Fuzzing with seed %d, %d iterations", 
                       self.seed, self.max_iterations))
    
    local crashes = 0
    local errors = 0
    local successes = 0
    
    for i = 1, self.max_iterations do
        -- Generate fuzzed input
        local fuzzed_inputs = {}
        for _, base in ipairs(base_inputs) do
            if type(base) == "string" then
                table.insert(fuzzed_inputs, self:generate_string(base))
            elseif type(base) == "number" then
                -- Fuzz numbers: edge cases
                local interesting_nums = {
                    0, -1, 1, 2^31 - 1, -2^31, 2^32, math.huge, -math.huge,
                    0/0, math.random(-1000, 1000)
                }
                table.insert(fuzzed_inputs, 
                             interesting_nums[math.random(1, #interesting_nums)])
            end
        end
        
        -- Try to crash the target
        local ok, result = pcall(target_fn, table.unpack(fuzzed_inputs))
        
        if not ok then
            crashes = crashes + 1
            table.insert(self.crashes, {
                iteration = i,
                input = fuzzed_inputs,
                error = result
            })
        else
            if validator and not validator(result) then
                -- Interesting: didn't crash but produced unexpected output
                errors = errors + 1
            else
                successes = successes + 1
            end
        end
    end
    
    print(string.format("Results: %d successes, %d crashes, %d errors",
                       successes, crashes, errors))
    
    if #self.crashes > 0 then
        print("\nCrash samples:")
        for _, crash in ipairs(self.crashes) do
            if _ <= 3 then  -- Show first 3
                print(string.format("  Iteration %d: %s", 
                                   crash.iteration,
                                   tostring(crash.error):sub(1, 80)))
            end
        end
    end
    
    return self.crashes
end

-- ตัวอย่างการ fuzz a simple parser
local function simple_json_key_parser(input)
    if type(input) ~= "string" then error("expected string") end
    if #input > 10000 then error("input too long") end
    
    local key = input:match('"([^"]*)"')
    if not key then
        return nil
    end
    return key
end

local fuzzer = Fuzzer.new({max_iterations = 100, seed = 42})
print("\n=== Fuzzing JSON Key Parser ===")
fuzzer:fuzz(simple_json_key_parser, {'"test_key"'})
```

## Penetration Testing Tools

```lua
-- ตัวอย่างที่ 18: Security Testing Framework
local PentestFramework = {}
PentestFramework.__index = PentestFramework

function PentestFramework.new(target)
    return setmetatable({
        target = target,
        findings = {},
        tests_run = 0
    }, PentestFramework)
end

function PentestFramework:run_test(name, fn)
    self.tests_run = self.tests_run + 1
    
    local ok, result = pcall(fn, self.target)
    
    if ok and result and result.vulnerable then
        table.insert(self.findings, {
            name = name,
            severity = result.severity or "MEDIUM",
            description = result.description or "Vulnerability found",
            evidence = result.evidence,
            remediation = result.remediation
        })
        print(string.format("[VULN] %s (%s)", name, result.severity or "MEDIUM"))
    elseif not ok then
        print(string.format("[ERROR] %s: %s", name, result))
    else
        print(string.format("[SAFE] %s", name))
    end
end

function PentestFramework:test_auth_bypass(endpoint, admin_endpoint)
    self:run_test("Authentication Bypass Check", function(target)
        -- Test if admin endpoints are accessible without auth
        local test_paths = {
            "/admin", "/admin/", "/Admin",
            "/.admin", "/admin.php",
            "/api/admin", "/api/v1/admin"
        }
        
        for _, path in ipairs(test_paths) do
            -- Simulate HTTP request
            local response = target.request({
                method = "GET",
                path = path,
                headers = {}  -- No auth headers
            })
            
            if response and response.status == 200 then
                return {
                    vulnerable = true,
                    severity = "CRITICAL",
                    description = "Admin endpoint accessible without authentication",
                    evidence = path .. " returned 200",
                    remediation = "Add authentication middleware to admin routes"
                }
            end
        end
        
        return {vulnerable = false}
    end)
end

function PentestFramework:test_xss_reflected(endpoint)
    self:run_test("Reflected XSS", function(target)
        local payloads = {
            "<script>alert(1)</script>",
            '"><script>alert(1)</script>',
            "javascript:alert(1)",
            "<img src=x onerror=alert(1)>",
            "';alert(1);//"
        }
        
        for _, payload in ipairs(payloads) do
            local response = target.request({
                method = "GET",
                path = "/search?q=" .. payload,
                headers = {}
            })
            
            if response and response.body then
                -- Check if payload appears in response unescaped
                if response.body:find(payload, 1, true) then
                    return {
                        vulnerable = true,
                        severity = "HIGH",
                        description = "Reflected XSS vulnerability found",
                        evidence = string.format("Payload '%s' reflected in response", payload),
                        remediation = "HTML-encode all user input before rendering"
                    }
                end
            end
        end
        
        return {vulnerable = false}
    end)
end

function PentestFramework:test_insecure_direct_object_reference()
    self:run_test("Insecure Direct Object Reference (IDOR)", function(target)
        -- Try to access another user's resource
        local response1 = target.request({
            method = "GET",
            path = "/api/user/1/profile",  -- Target user 1
            headers = {["Authorization"] = "Bearer user_2_token"}  -- Auth as user 2
        })
        
        if response1 and response1.status == 200 then
            return {
                vulnerable = true,
                severity = "HIGH",
                description = "User can access other users' data (IDOR)",
                evidence = "User 2 can access User 1's profile",
                remediation = "Validate that authenticated user owns the requested resource"
            }
        end
        
        return {vulnerable = false}
    end)
end

function PentestFramework:report()
    print(string.format("\n=== Penetration Test Report ==="))
    print(string.format("Target: %s", tostring(self.target.name or "unknown")))
    print(string.format("Tests run: %d", self.tests_run))
    print(string.format("Findings: %d", #self.findings))
    
    if #self.findings > 0 then
        -- Group by severity
        local by_severity = {}
        for _, finding in ipairs(self.findings) do
            if not by_severity[finding.severity] then
                by_severity[finding.severity] = {}
            end
            table.insert(by_severity[finding.severity], finding)
        end
        
        for _, severity in ipairs({"CRITICAL", "HIGH", "MEDIUM", "LOW"}) do
            if by_severity[severity] then
                print(string.format("\n[%s]", severity))
                for _, f in ipairs(by_severity[severity]) do
                    print(string.format("  %s", f.name))
                    print(string.format("  Description: %s", f.description))
                    if f.evidence then
                        print(string.format("  Evidence: %s", f.evidence))
                    end
                    if f.remediation then
                        print(string.format("  Fix: %s", f.remediation))
                    end
                end
            end
        end
    else
        print("No vulnerabilities found!")
    end
end

-- ตัวอย่างการใช้งาน (Simulated target)
local mock_target = {
    name = "Example Application",
    request = function(req)
        -- Simulated responses
        if req.path == "/admin" and not req.headers["Authorization"] then
            return {status = 200, body = "Admin Panel"}  -- Vulnerable!
        end
        if req.path and req.path:find("/search") then
            local query = req.path:match("q=(.+)") or ""
            -- Simulate reflected XSS (vulnerable)
            return {status = 200, body = "<p>Results for: " .. query .. "</p>"}
        end
        if req.path and req.path:find("/api/user/1/profile") then
            return {status = 200, body = '{"id":1,"email":"user1@test.com"}'}
        end
        return {status = 404}
    end
}

local pentest = PentestFramework.new(mock_target)
print("\n=== Running Security Tests ===")
pentest:test_auth_bypass("/admin")
pentest:test_xss_reflected("/search")
pentest:test_insecure_direct_object_reference()
pentest:report()
```

## CVE Analysis Pattern

```lua
-- ตัวอย่างที่ 19: CVE Impact Analyzer
local CVEAnalyzer = {}
CVEAnalyzer.__index = CVEAnalyzer

function CVEAnalyzer.new()
    return setmetatable({
        cves = {},
        affected_systems = {}
    }, CVEAnalyzer)
end

function CVEAnalyzer:add_cve(id, data)
    self.cves[id] = {
        id = id,
        description = data.description,
        cvss_score = data.cvss_score,
        cvss_vector = data.cvss_vector,
        affected_versions = data.affected_versions or {},
        fixed_versions = data.fixed_versions or {},
        attack_vector = data.attack_vector,
        exploit_available = data.exploit_available or false,
        poc_public = data.poc_public or false
    }
end

function CVEAnalyzer:calculate_priority(cve_id)
    local cve = self.cves[cve_id]
    if not cve then return nil end
    
    local score = cve.cvss_score
    local priority = "P4"  -- Low
    
    -- Base priority on CVSS score
    if score >= 9.0 then
        priority = "P1"  -- Critical
    elseif score >= 7.0 then
        priority = "P2"  -- High
    elseif score >= 4.0 then
        priority = "P3"  -- Medium
    end
    
    -- Increase priority if exploit is available
    if cve.exploit_available and priority ~= "P1" then
        local upgrade = {P2 = "P1", P3 = "P2", P4 = "P3"}
        priority = upgrade[priority] or priority
    end
    
    return priority
end

function CVEAnalyzer:generate_remediation_plan()
    local plan = {}
    
    for id, cve in pairs(self.cves) do
        local priority = self:calculate_priority(id)
        table.insert(plan, {
            cve = id,
            priority = priority,
            score = cve.cvss_score,
            description = cve.description:sub(1, 60) .. "...",
            exploit_available = cve.exploit_available
        })
    end
    
    -- Sort by priority
    local priority_order = {P1 = 1, P2 = 2, P3 = 3, P4 = 4}
    table.sort(plan, function(a, b)
        if priority_order[a.priority] ~= priority_order[b.priority] then
            return priority_order[a.priority] < priority_order[b.priority]
        end
        return a.score > b.score
    end)
    
    return plan
end

-- ตัวอย่างการใช้งาน
local analyzer = CVEAnalyzer.new()

analyzer:add_cve("CVE-2024-1234", {
    description = "Remote code execution in web server via malformed HTTP headers",
    cvss_score = 9.8,
    cvss_vector = "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H",
    attack_vector = "Network",
    exploit_available = true,
    poc_public = true
})

analyzer:add_cve("CVE-2024-5678", {
    description = "Privilege escalation via path traversal vulnerability",
    cvss_score = 7.5,
    cvss_vector = "CVSS:3.1/AV:N/AC:H/PR:L/UI:N/S:U/C:H/I:H/A:H",
    attack_vector = "Network",
    exploit_available = false
})

analyzer:add_cve("CVE-2024-9012", {
    description = "Information disclosure in debug endpoint",
    cvss_score = 5.3,
    attack_vector = "Network",
    exploit_available = false
})

print("\n=== CVE Remediation Plan ===")
local plan = analyzer:generate_remediation_plan()
for _, item in ipairs(plan) do
    print(string.format("[%s] %s (CVSS: %.1f) %s - %s",
                       item.priority, item.cve, item.score,
                       item.exploit_available and "[EXPLOIT AVAILABLE]" or "",
                       item.description))
end
```

## สรุปบทที่ 89

ในบทนี้เราได้เรียนรู้ Advanced Security Engineering ครอบคลุม:

1. **Threat Modeling** - STRIDE methodology การระบุ threats
2. **Code Injection** - ความเสี่ยงของ load() และการป้องกัน
3. **Sandbox Implementation** - การสร้าง secure sandbox
4. **Sandbox Escaping** - เทคนิค escape และ defenses
5. **Input Validation** - Comprehensive validator
6. **Memory Safety** - Safe string operations
7. **Secret Management** - Secure secret handling
8. **Side-Channel Attacks** - Timing attacks และ constant-time comparison
9. **Secure Password Hashing** - PBKDF2 pattern
10. **Dependency Security** - Vulnerability scanning
11. **OWASP Top 10** - Access control, injection prevention
12. **Authentication Security** - Brute force protection, session management
13. **Fuzzing** - Basic fuzzing techniques
14. **Penetration Testing** - Security test framework
15. **CVE Analysis** - Vulnerability prioritization

Key security principles:
- Never trust user input - validate everything
- Defense in depth - multiple security layers
- Principle of least privilege - minimal access rights
- Fail securely - errors should not leak information
- Constant-time operations for security-sensitive comparisons
