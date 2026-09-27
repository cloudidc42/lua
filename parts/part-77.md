# บทที่ 77: Metaprogramming ใน Lua

## บทนำ

Metaprogramming คือการเขียน code ที่ generate หรือ manipulate code อื่น ใน Lua เราสามารถทำได้หลายวิธีเนื่องจาก metatables ที่ทรงพลัง, `load()` function, และ environment system ที่ยืดหยุ่น

---

## 1. Metaprogramming คืออะไร

```lua
-- Metaprogramming = Code ที่เขียน/จัดการ Code อื่น
-- ใน Lua มี 3 ระดับ:
-- 1. Runtime reflection (ตรวจสอบ type, structure ขณะ run)
-- 2. Code generation (สร้าง code แบบ dynamic)
-- 3. Metamethods (ปรับ behavior ของ operators และ operations)

-- ตัวอย่างง่ายๆ: สร้าง getter/setter automatically
local function makeProperties(class, properties)
    for _, prop in ipairs(properties) do
        local propName = prop
        
        -- Auto-generate getter
        class["get" .. propName:sub(1,1):upper() .. propName:sub(2)] = function(self)
            return self["_" .. propName]
        end
        
        -- Auto-generate setter
        class["set" .. propName:sub(1,1):upper() .. propName:sub(2)] = function(self, value)
            self["_" .. propName] = value
            return self
        end
    end
end

local Person = {}
Person.__index = Person

makeProperties(Person, { "name", "email", "age" })

function Person.new(data)
    local obj = setmetatable({}, Person)
    for k, v in pairs(data) do
        obj["_" .. k] = v
    end
    return obj
end

local p = Person.new({ name = "Alice", email = "alice@example.com", age = 30 })
print(p:getName())   -- Alice
print(p:getEmail())  -- alice@example.com

p:setName("Bob"):setAge(25)
print(p:getName(), p:getAge())
```

---

## 2. load() และ loadstring()

```lua
-- load() ใช้สำหรับ compile และ execute strings เป็น Lua code

-- 1. Execute string as code
local code = "return 1 + 2 + 3"
local fn = load(code)
print(fn())  -- 6

-- 2. Execute với variables
local template = "local x = %d; local y = %d; return x + y"
local fn2 = load(string.format(template, 10, 20))
print(fn2())  -- 30

-- 3. Load với environment
local env = { x = 100, y = 200, print = print }
local code3 = "return x + y"
-- ใน Lua 5.4 ใช้ load ด้วย environment
local fn3 = load("return x + y", "chunk", "t", env)
if fn3 then
    print(fn3())  -- 300
end

-- 4. สร้าง function จาก string แบบ dynamic
local function createAdder(n)
    local code = string.format("return function(x) return x + %d end", n)
    local factory = load(code)
    return factory()
end

local add10 = createAdder(10)
local add20 = createAdder(20)

print(add10(5))   -- 15
print(add20(5))   -- 25

-- 5. Load file contents
local function loadFromString(code, name)
    name = name or "dynamic_code"
    local fn, err = load(code, "@" .. name)
    if not fn then
        error("Compile error in " .. name .. ": " .. err)
    end
    return fn
end

local fn4 = loadFromString("local t = {}; for i=1,5 do t[i]=i*i end; return t", "squares")
local result = fn4()
for i, v in ipairs(result) do
    io.write(v .. " ")
end
print()  -- 1 4 9 16 25
```

---

## 3. Dynamic Method Generation ด้วย __index

```lua
-- __index เป็นหัวใจของ metaprogramming ใน Lua
-- สามารถ intercept property access และสร้าง method dynamically

local DynamicClass = {}
DynamicClass.__index = function(self, key)
    -- ถ้า key ขึ้นต้นด้วย "get" - สร้าง getter
    if key:match("^get(.+)") then
        local field = key:match("^get(.+)")
        field = field:sub(1,1):lower() .. field:sub(2)
        return function(self)
            return self[field]
        end
    end
    
    -- ถ้า key ขึ้นต้นด้วย "set" - สร้าง setter
    if key:match("^set(.+)") then
        local field = key:match("^set(.+)")
        field = field:sub(1,1):lower() .. field:sub(2)
        return function(self, value)
            self[field] = value
            return self
        end
    end
    
    -- ถ้า key ขึ้นต้นด้วย "has" - สร้าง existence check
    if key:match("^has(.+)") then
        local field = key:match("^has(.+)")
        field = field:sub(1,1):lower() .. field:sub(2)
        return function(self)
            return self[field] ~= nil
        end
    end
    
    -- ถ้าไม่ match อะไร ก็คืน method จาก class
    return DynamicClass[key]
end

function DynamicClass.new(data)
    return setmetatable(data or {}, DynamicClass)
end

local obj = DynamicClass.new({ 
    name = "Alice", 
    email = "alice@example.com"
})

-- Dynamic getters/setters ถูกสร้างขณะ call
print(obj:getName())   -- Alice
print(obj:getEmail())  -- alice@example.com
print(obj:hasPhone())  -- false

obj:setPhone("080-123-4567")
print(obj:getPhone())  -- 080-123-4567
print(obj:hasPhone())  -- true
```

---

## 4. Proxy Objects

```lua
-- Proxy Object - intercept ทุก operation บน object
local function createProxy(target, handlers)
    handlers = handlers or {}
    
    local proxy = {}
    local mt = {
        __index = function(t, key)
            if handlers.get then
                return handlers.get(target, key)
            end
            return target[key]
        end,
        
        __newindex = function(t, key, value)
            if handlers.set then
                handlers.set(target, key, value)
            else
                target[key] = value
            end
        end,
        
        __len = function(t)
            return #target
        end,
        
        __pairs = function(t)
            return pairs(target)
        end,
        
        __call = function(t, ...)
            if handlers.apply then
                return handlers.apply(target, ...)
            elseif type(target) == "function" then
                return target(...)
            end
        end
    }
    
    return setmetatable(proxy, mt)
end

-- ตัวอย่าง: Read-only proxy
local function readOnly(target)
    return createProxy(target, {
        set = function(t, key, value)
            error("Attempt to write to read-only object: " .. tostring(key))
        end
    })
end

-- ตัวอย่าง: Logging proxy
local function logProxy(target, name)
    return createProxy(target, {
        get = function(t, key)
            local value = t[key]
            if type(value) == "function" then
                return function(self, ...)
                    print(string.format("[Proxy:%s] Calling %s", name, key))
                    return value(t, ...)
                end
            end
            print(string.format("[Proxy:%s] Getting %s = %s", name, key, tostring(value)))
            return value
        end,
        set = function(t, key, value)
            print(string.format("[Proxy:%s] Setting %s = %s", name, key, tostring(value)))
            t[key] = value
        end
    })
end

-- ตัวอย่าง: Validation proxy
local function validatingProxy(target, schema)
    return createProxy(target, {
        set = function(t, key, value)
            if schema[key] then
                local validator = schema[key]
                local ok, err = validator(value)
                if not ok then
                    error(string.format("Validation failed for '%s': %s", key, err))
                end
            end
            t[key] = value
        end
    })
end

-- Test proxies
local data = { name = "Alice", age = 30 }

print("=== Read-only proxy ===")
local ro = readOnly(data)
print(ro.name)  -- OK

local ok, err = pcall(function()
    ro.name = "Bob"  -- Error!
end)
print("Write error:", err)

print("\n=== Logging proxy ===")
local logged = logProxy(data, "data")
logged.email = "alice@test.com"
print(logged.name)

print("\n=== Validation proxy ===")
local validated = validatingProxy(data, {
    age = function(v)
        if type(v) ~= "number" then return false, "must be a number" end
        if v < 0 or v > 150 then return false, "must be between 0 and 150" end
        return true
    end,
    email = function(v)
        if type(v) ~= "string" then return false, "must be a string" end
        if not v:match("@") then return false, "must contain @" end
        return true
    end
})

validated.age = 25  -- OK
print("Age set to 25")

local ok2, err2 = pcall(function()
    validated.age = -5  -- Error!
end)
print("Validation error:", err2)
```

---

## 5. Transparent Proxy

```lua
-- Transparent Proxy - proxy ที่ดูเหมือน original object ทุกอย่าง
local function transparentProxy(target)
    -- สร้าง metatable ที่ forward ทุก operation
    local mt = {}
    
    mt.__index = function(proxy, key)
        local value = target[key]
        if type(value) == "function" then
            -- Wrap function เพื่อ redirect self
            return function(self, ...)
                if self == proxy then
                    return value(target, ...)
                end
                return value(self, ...)
            end
        end
        return value
    end
    
    mt.__newindex = function(proxy, key, value)
        target[key] = value
    end
    
    mt.__eq = function(a, b)
        if a == proxy then a = target end
        if b == proxy then b = target end
        return a == b
    end
    
    mt.__tostring = function()
        return tostring(target)
    end
    
    mt.__len = function()
        return #target
    end
    
    -- Store reference to target
    local proxyObj = setmetatable({}, mt)
    
    -- Method to get underlying object
    function proxyObj:__getTarget()
        return target
    end
    
    return proxyObj
end

-- Observable proxy - ทำให้ object เป็น observable โดยไม่ modify original
local function observableProxy(target, onChange)
    local mt = {
        __index = target,
        __newindex = function(proxy, key, value)
            local oldValue = target[key]
            target[key] = value
            if onChange then
                onChange(key, value, oldValue)
            end
        end
    }
    return setmetatable({}, mt)
end

-- ทดสอบ
local original = { x = 10, y = 20 }

print("=== Observable Proxy ===")
local observed = observableProxy(original, function(key, newVal, oldVal)
    print(string.format("Changed: %s from %s to %s", 
        key, tostring(oldVal), tostring(newVal)))
end)

observed.x = 15    -- triggers onChange
observed.z = 30    -- triggers onChange
print("x =", observed.x)  -- reads from original
print("z =", observed.z)
```

---

## 6. Method Missing Equivalent

```lua
-- method_missing เหมือนใน Ruby - handle unknown method calls
-- ใน Lua ทำได้ผ่าน __index metamethod

local FlexibleObject = {}
FlexibleObject.__index = function(self, key)
    -- Check actual class first
    local classMethod = FlexibleObject[key]
    if classMethod then return classMethod end
    
    -- Dynamic method generation based on naming conventions
    -- findByXxx - query methods
    local findField = key:match("^findBy(.+)$")
    if findField then
        findField = findField:sub(1,1):lower() .. findField:sub(2)
        return function(self, value)
            return string.format("findBy%s(%s) - searching for %s=%s", 
                findField, value, findField, tostring(value))
        end
    end
    
    -- isXxx - boolean check methods
    local checkField = key:match("^is(.+)$")
    if checkField then
        checkField = checkField:sub(1,1):lower() .. checkField:sub(2)
        return function(self)
            return self[checkField] == true
        end
    end
    
    -- countXxx - counting methods
    local countField = key:match("^count(.+)$")
    if countField then
        countField = countField:sub(1,1):lower() .. countField:sub(2)
        return function(self)
            local collection = self[countField]
            if type(collection) == "table" then
                return #collection
            end
            return 0
        end
    end
    
    -- Default: return a "method not found" handler
    return function(self, ...)
        print(string.format("[Warning] Method '%s' not found on %s", 
            key, tostring(self)))
        return nil
    end
end

function FlexibleObject.new(data)
    return setmetatable(data or {}, FlexibleObject)
end

local user = FlexibleObject.new({ 
    name = "Alice",
    email = "alice@example.com",
    active = true,
    posts = { "Post 1", "Post 2", "Post 3" }
})

-- Dynamic methods
print(user:findByName("Alice"))
print(user:findByEmail("test@test.com"))
print(user:isActive())
print(user:countPosts())
print(user:someUnknownMethod())  -- handled gracefully
```

---

## 7. Dynamic Dispatch

```lua
-- Dynamic Dispatch - เลือก method ขณะ runtime
local Dispatcher = {}
Dispatcher.__index = Dispatcher

function Dispatcher.new()
    local obj = setmetatable({
        _handlers = {},
        _middleware = {}
    }, Dispatcher)
    return obj
end

function Dispatcher:register(event, handler)
    if not self._handlers[event] then
        self._handlers[event] = {}
    end
    table.insert(self._handlers[event], handler)
    return self
end

function Dispatcher:use(middleware)
    table.insert(self._middleware, middleware)
    return self
end

function Dispatcher:dispatch(event, data)
    local handlers = self._handlers[event]
    if not handlers then
        print(string.format("[Dispatcher] No handlers for event: %s", event))
        return
    end
    
    -- Apply middleware
    local context = { event = event, data = data, stopped = false }
    
    for _, mw in ipairs(self._middleware) do
        if not context.stopped then
            mw(context, function() end)
        end
    end
    
    if context.stopped then return end
    
    -- Call handlers
    for _, handler in ipairs(handlers) do
        handler(data, context)
    end
end

-- Multi-method dispatch based on type
local function multiMethod(...)
    local methods = {}
    local args = { ... }
    
    for i = 1, #args, 2 do
        local signature = args[i]
        local fn = args[i + 1]
        table.insert(methods, { signature = signature, fn = fn })
    end
    
    return function(...)
        local callArgs = { ... }
        
        -- Find matching method based on argument types
        for _, method in ipairs(methods) do
            local sig = method.signature
            local match = true
            
            for i, expectedType in ipairs(sig) do
                if type(callArgs[i]) ~= expectedType then
                    match = false
                    break
                end
            end
            
            if match then
                return method.fn(...)
            end
        end
        
        error("No matching method for argument types: " .. 
            table.concat(
                (function()
                    local types = {}
                    for _, v in ipairs(callArgs) do
                        table.insert(types, type(v))
                    end
                    return types
                end)(),
                ", "
            )
        )
    end
end

-- ตัวอย่าง multi-method dispatch
local process = multiMethod(
    { "string" }, function(s)
        return "Processing string: " .. s
    end,
    { "number" }, function(n)
        return "Processing number: " .. n
    end,
    { "table" }, function(t)
        return "Processing table with " .. #t .. " items"
    end,
    { "string", "number" }, function(s, n)
        return string.format("Processing string '%s' with number %d", s, n)
    end
)

print(process("hello"))
print(process(42))
print(process({ 1, 2, 3 }))
print(process("count", 5))

-- Dynamic dispatch based on object type
local dispatcher = Dispatcher.new()

dispatcher:use(function(ctx, next)
    print(string.format("[MW] Event: %s at %s", ctx.event, os.date("%H:%M:%S")))
    next()
end)

dispatcher:register("user.created", function(data)
    print("Handler 1: New user created:", data.name)
end)

dispatcher:register("user.created", function(data)
    print("Handler 2: Sending welcome email to:", data.email)
end)

dispatcher:dispatch("user.created", { 
    name = "Alice", 
    email = "alice@example.com" 
})
```

---

## 8. Code Generation via String Templates

```lua
-- Code Generation - สร้าง Lua code แบบ dynamic
local CodeGen = {}
CodeGen.__index = CodeGen

function CodeGen.new()
    return setmetatable({ lines = {}, indent = 0 }, CodeGen)
end

function CodeGen:write(line)
    if line then
        table.insert(self.lines, string.rep("  ", self.indent) .. line)
    end
    return self
end

function CodeGen:blank()
    table.insert(self.lines, "")
    return self
end

function CodeGen:beginBlock(line)
    self:write(line)
    self.indent = self.indent + 1
    return self
end

function CodeGen:endBlock(suffix)
    self.indent = self.indent - 1
    self:write("end" .. (suffix or ""))
    return self
end

function CodeGen:generate()
    return table.concat(self.lines, "\n")
end

function CodeGen:execute()
    local code = self:generate()
    local fn, err = load(code)
    if not fn then
        error("Code generation error: " .. err .. "\n\nGenerated code:\n" .. code)
    end
    return fn()
end

-- ตัวอย่าง: Generate enum-like class
local function generateEnum(name, values)
    local gen = CodeGen.new()
    
    gen:write("local " .. name .. " = {}")
    gen:blank()
    
    for i, value in ipairs(values) do
        gen:write(string.format('%s.%s = %d', name, value, i))
    end
    
    gen:blank()
    gen:write(string.format('%s._names = {', name))
    for i, value in ipairs(values) do
        gen:write(string.format('  [%d] = "%s",', i, value))
    end
    gen:write("}")
    gen:blank()
    
    gen:beginBlock(string.format("function %s:name(value)", name))
    gen:write("return self._names[value] or 'unknown'")
    gen:endBlock()
    gen:blank()
    
    gen:write("return " .. name)
    
    local code = gen:generate()
    print("Generated code:")
    print(code)
    print("---")
    
    local fn = load(code)
    return fn()
end

local Status = generateEnum("Status", { "PENDING", "ACTIVE", "INACTIVE", "DELETED" })

print("PENDING =", Status.PENDING)
print("ACTIVE =", Status.ACTIVE)
print("Name of 2:", Status:name(2))
```

---

## 9. Template-Based Code Generation

```lua
-- Template system สำหรับ code generation
local Template = {}
Template.__index = Template

function Template.new(templateStr)
    return setmetatable({ template = templateStr }, Template)
end

function Template:render(vars)
    local result = self.template
    
    -- Replace {{varName}} with values
    result = result:gsub("{{(%w+)}}", function(key)
        local value = vars[key]
        if value == nil then
            return "nil"
        end
        return tostring(value)
    end)
    
    -- Handle {{#each list}} ... {{/each}}
    result = result:gsub("{{#each (%w+)}}(.-){{/each}}", function(listName, body)
        local list = vars[listName]
        if type(list) ~= "table" then return "" end
        
        local parts = {}
        for i, item in ipairs(list) do
            local itemStr = body:gsub("{{this}}", tostring(item))
            itemStr = itemStr:gsub("{{@index}}", tostring(i))
            table.insert(parts, itemStr)
        end
        return table.concat(parts)
    end)
    
    -- Handle {{#if condition}} ... {{/if}}
    result = result:gsub("{{#if (%w+)}}(.-){{/if}}", function(condName, body)
        if vars[condName] then
            return body
        end
        return ""
    end)
    
    return result
end

-- Class Generator Template
local classTemplate = Template.new([[
local {{className}} = {}
{{className}}.__index = {{className}}

function {{className}}.new(data)
    local obj = setmetatable({}, {{className}})
    if data then
        for k, v in pairs(data) do
            obj[k] = v
        end
    end
    return obj
end

{{#each methods}}
function {{className}}:{{this}}()
    -- TODO: implement {{this}}
end

{{/each}}
{{#if hasToString}}
function {{className}}:__tostring()
    return "{{className}}(" .. tostring(self.id) .. ")"
end
{{/if}}

return {{className}}
]])

local generatedCode = classTemplate:render({
    className = "Customer",
    methods = { "save", "delete", "validate" },
    hasToString = true
})

print("Generated Class Code:")
print(generatedCode)

-- Execute generated code
local fn = load(generatedCode)
local Customer = fn()

local c = Customer.new({ id = 1, name = "Alice" })
print("\nCreated customer:", c.id, c.name)
print("Has save method:", type(c.save) == "function")
```

---

## 10. Auto-Memoization

```lua
-- Auto-memoization: cache ผลลัพธ์ของ function อัตโนมัติ
local function memoize(fn, keyFn)
    local cache = {}
    
    keyFn = keyFn or function(...)
        local parts = {}
        for _, v in ipairs({ ... }) do
            table.insert(parts, tostring(v))
        end
        return table.concat(parts, ":")
    end
    
    return function(...)
        local key = keyFn(...)
        
        if cache[key] ~= nil then
            return cache[key]
        end
        
        local result = fn(...)
        cache[key] = result
        return result
    end
end

-- Memoize ทุก method ใน object
local function memoizeAll(obj, methods)
    for _, methodName in ipairs(methods) do
        local original = obj[methodName]
        if type(original) == "function" then
            local cache = {}
            obj[methodName] = function(self, ...)
                local key = table.concat({ ... }, ":")
                if cache[key] == nil then
                    cache[key] = original(self, ...)
                end
                return cache[key]
            end
        end
    end
    return obj
end

-- Auto-memoize via __index proxy
local function createMemoizedProxy(target)
    local caches = {}
    
    return setmetatable({}, {
        __index = function(proxy, key)
            local value = target[key]
            
            if type(value) ~= "function" then
                return value
            end
            
            -- Return memoized version
            return function(self, ...)
                if not caches[key] then caches[key] = {} end
                
                local cacheKey = table.concat({ ... }, ":")
                
                if caches[key][cacheKey] == nil then
                    print(string.format("  [Cache MISS] %s(%s)", key, cacheKey))
                    caches[key][cacheKey] = value(target, ...)
                else
                    print(string.format("  [Cache HIT] %s(%s)", key, cacheKey))
                end
                
                return caches[key][cacheKey]
            end
        end
    })
end

-- Fibonacci without memoize (slow for large n)
local callCount = 0
local function fibonacci(n)
    callCount = callCount + 1
    if n <= 1 then return n end
    return fibonacci(n - 1) + fibonacci(n - 2)
end

-- With memoize
callCount = 0
local memoFib
memoFib = memoize(function(n)
    callCount = callCount + 1
    if n <= 1 then return n end
    return memoFib(n - 1) + memoFib(n - 2)
end)

print("Fibonacci(30) =", memoFib(30))
print("Function calls:", callCount)

-- Object with memoized proxy
local Calculator = {}
Calculator.__index = Calculator

function Calculator.new()
    return setmetatable({}, Calculator)
end

function Calculator:expensiveCalc(n)
    print("  Computing for n=" .. n)
    local result = 0
    for i = 1, n * 1000 do
        result = result + i
    end
    return result
end

local calc = Calculator.new()
local memoCalc = createMemoizedProxy(calc)

print("\nMemoized Calculator:")
memoCalc:expensiveCalc(10)
memoCalc:expensiveCalc(10)  -- should hit cache
memoCalc:expensiveCalc(20)
memoCalc:expensiveCalc(20)  -- should hit cache
```

---

## 11. Auto-Logging via Metaprogramming

```lua
-- Auto-logging: log ทุก method call โดยไม่ต้องแก้ code
local function autoLog(class, className, options)
    options = options or {}
    local logLevel = options.logLevel or "info"
    local exclude = options.exclude or {}
    
    -- Create exclusion set
    local excludeSet = {}
    for _, name in ipairs(exclude) do
        excludeSet[name] = true
    end
    
    -- Wrap each method
    for name, method in pairs(class) do
        if type(method) == "function" 
           and not name:match("^__")  -- skip metamethods
           and not excludeSet[name] then
            
            local original = method
            class[name] = function(self, ...)
                local args = { ... }
                local argStr = ""
                for i, v in ipairs(args) do
                    if i > 1 then argStr = argStr .. ", " end
                    argStr = argStr .. tostring(v)
                end
                
                print(string.format("[%s] [%s] %s.%s(%s)",
                    logLevel:upper(),
                    os.date("%H:%M:%S"),
                    className, name, argStr))
                
                local start = os.clock()
                local results = { original(self, ...) }
                local elapsed = os.clock() - start
                
                if elapsed > 0.001 then  -- log slow calls
                    print(string.format("[PERF] %s.%s took %.4f seconds", 
                        className, name, elapsed))
                end
                
                return table.unpack(results)
            end
        end
    end
    
    return class
end

-- ใช้งาน auto-logging
local BankAccount = {}
BankAccount.__index = BankAccount

function BankAccount.new(owner, balance)
    return setmetatable({ owner = owner, balance = balance or 0 }, BankAccount)
end

function BankAccount:deposit(amount)
    if amount <= 0 then error("Amount must be positive") end
    self.balance = self.balance + amount
    return self.balance
end

function BankAccount:withdraw(amount)
    if amount > self.balance then
        error("Insufficient funds")
    end
    self.balance = self.balance - amount
    return self.balance
end

function BankAccount:getBalance()
    return self.balance
end

-- Apply auto-logging
autoLog(BankAccount, "BankAccount", { 
    logLevel = "info",
    exclude = { "new" }
})

local account = BankAccount.new("Alice", 1000)
account:deposit(500)
account:withdraw(200)
print("Balance:", account:getBalance())

local ok, err = pcall(function()
    account:withdraw(2000)
end)
print("Error:", err)
```

---

## 12. DSL Construction via Metaprogramming

```lua
-- Mini DSL Builder
local DSLBuilder = {}

function DSLBuilder.create(grammar)
    local dsl = {}
    
    -- สร้าง grammar rules เป็น methods
    for ruleName, ruleConfig in pairs(grammar) do
        if ruleConfig.type == "method" then
            dsl[ruleName] = function(self, ...)
                if ruleConfig.handler then
                    ruleConfig.handler(self, ...)
                end
                if ruleConfig.returns == "self" then
                    return self
                end
                return ruleConfig.returns
            end
        end
    end
    
    function dsl:build()
        return self._result
    end
    
    return dsl
end

-- HTML Builder DSL
local HTMLBuilder = {}
HTMLBuilder.__index = HTMLBuilder

local function createTag(name, selfClosing)
    return function(self, attrs, content)
        local attrStr = ""
        if type(attrs) == "table" then
            for k, v in pairs(attrs) do
                attrStr = attrStr .. string.format(' %s="%s"', k, v)
            end
        elseif type(attrs) == "string" then
            content = attrs
            attrs = nil
        end
        
        if selfClosing then
            table.insert(self._parts, string.format("<%s%s/>", name, attrStr))
        elseif content then
            table.insert(self._parts, string.format("<%s%s>%s</%s>", name, attrStr, content, name))
        else
            table.insert(self._parts, string.format("<%s%s>", name, attrStr))
            self._openTags = self._openTags or {}
            table.insert(self._openTags, name)
        end
        
        return self
    end
end

-- Dynamically create methods for each HTML tag
local tags = { "div", "p", "h1", "h2", "h3", "span", "ul", "ol", "li", "a", "table", "tr", "td", "th" }
local selfClosingTags = { "br", "hr", "img", "input" }

for _, tag in ipairs(tags) do
    HTMLBuilder[tag] = createTag(tag, false)
end

for _, tag in ipairs(selfClosingTags) do
    HTMLBuilder[tag] = createTag(tag, true)
end

function HTMLBuilder.new()
    return setmetatable({ _parts = {}, _openTags = {} }, HTMLBuilder)
end

function HTMLBuilder:end_()
    local tag = table.remove(self._openTags)
    if tag then
        table.insert(self._parts, "</" .. tag .. ">")
    end
    return self
end

function HTMLBuilder:text(content)
    table.insert(self._parts, content)
    return self
end

function HTMLBuilder:render()
    return table.concat(self._parts, "\n")
end

local html = HTMLBuilder.new()
html:div({ class = "container" })
    :h1("Welcome to Lua")
    :p("This is a paragraph")
    :ul()
        :li("Item 1")
        :li("Item 2")
        :li("Item 3")
    :end_()
:end_()

print(html:render())
```

---

## 13. Macro-like Patterns

```lua
-- Macro-like patterns ใน Lua
-- Lua ไม่มี macro จริงๆ แต่เราสามารถ simulate ได้บางส่วน

-- 1. Function-based macros
local function ASSERT(condition, message)
    if not condition then
        local info = debug.getinfo(2, "Sl")
        error(string.format("Assertion failed at %s:%d: %s",
            info.source, info.currentline, message or "assertion failed"), 2)
    end
end

local function TODO(message)
    local info = debug.getinfo(2, "Sl")
    print(string.format("[TODO] %s:%d - %s", info.source, info.currentline, message or ""))
end

local function DEPRECATED(newName)
    local info = debug.getinfo(2, "Sl")
    print(string.format("[DEPRECATED] %s:%d - use %s instead", 
        info.source, info.currentline, newName or ""))
end

-- 2. compile-time-like expansion via code generation
local function defineConstants(...)
    local code = {}
    for i = 1, select('#', ...), 2 do
        local name = select(i, ...)
        local value = select(i + 1, ...)
        
        if type(value) == "string" then
            table.insert(code, string.format('local %s = "%s"', name, value))
        else
            table.insert(code, string.format('local %s = %s', name, tostring(value)))
        end
    end
    return load(table.concat(code, "\n"))
end

-- 3. Pattern-based code transformation
local function transform(code, transforms)
    for pattern, replacement in pairs(transforms) do
        code = code:gsub(pattern, replacement)
    end
    return code
end

-- Simple "unless" macro simulation
local function addUnlessMacro(code)
    return transform(code, {
        ["unless (.+) then"] = "if not (%1) then"
    })
end

-- Test macros
ASSERT(1 + 1 == 2, "Math is broken")
ASSERT(type("hello") == "string", "Type check failed")

local function oldFunction()
    DEPRECATED("newFunction")
    return "old result"
end

oldFunction()

-- transform test
local code = [[
local x = 10
unless x > 100 then
    print("x is small")
end
]]

local transformed = addUnlessMacro(code)
print("\nTransformed code:")
print(transformed)

local fn = load(transformed)
if fn then fn() end
```

---

## 14. Runtime Type Inspection

```lua
-- Runtime type inspection ด้วย debug library
local TypeInspector = {}

function TypeInspector.getInfo(fn)
    if type(fn) ~= "function" then
        return nil, "not a function"
    end
    
    local info = debug.getinfo(fn, "nSluf")
    return {
        name = info.name,
        source = info.source,
        currentLine = info.currentline,
        lineDefined = info.linedefined,
        lastLineDefined = info.lastlinedefined,
        nParams = info.nparams,
        isVarArg = info.isvararg == 1,
        nUpValues = info.nups,
        what = info.what  -- "Lua", "C", or "main"
    }
end

function TypeInspector.getUpvalues(fn)
    if type(fn) ~= "function" then return {} end
    
    local upvalues = {}
    local i = 1
    while true do
        local name, value = debug.getupvalue(fn, i)
        if not name then break end
        upvalues[name] = value
        i = i + 1
    end
    return upvalues
end

function TypeInspector.getLocals(level)
    level = level or 2  -- default: caller's frame
    local locals = {}
    local i = 1
    while true do
        local name, value = debug.getlocal(level, i)
        if not name then break end
        if not name:match("^%(") then  -- skip internal variables
            locals[name] = value
        end
        i = i + 1
    end
    return locals
end

function TypeInspector.inspect(value, depth)
    depth = depth or 0
    local indent = string.rep("  ", depth)
    local t = type(value)
    
    if t == "table" then
        local mt = getmetatable(value)
        local lines = { indent .. "{" }
        
        for k, v in pairs(value) do
            if type(k) == "string" then
                local valStr = TypeInspector.inspect(v, depth + 1)
                table.insert(lines, indent .. "  " .. k .. " = " .. valStr)
            end
        end
        
        if mt then
            table.insert(lines, indent .. "  [metatable] = " .. tostring(mt))
        end
        
        table.insert(lines, indent .. "}")
        return table.concat(lines, "\n")
    elseif t == "function" then
        local info = debug.getinfo(value, "nSl")
        return string.format("function(%s:%d)", 
            info.source or "?", info.linedefined or 0)
    elseif t == "string" then
        return string.format('"%s"', value)
    else
        return tostring(value)
    end
end

-- Test inspection
local x = 42
local name = "Alice"

local function myFunc(a, b)
    local localX = a + b
    local locals = TypeInspector.getLocals(1)
    return locals
end

local captured = { value = 100 }
local function withUpvalue()
    return captured.value
end

print("Function info:")
local info = TypeInspector.getInfo(myFunc)
if info then
    print("  nParams:", info.nParams)
    print("  what:", info.what)
    print("  lineDefined:", info.lineDefined)
end

print("\nUpvalues:")
local upvals = TypeInspector.getUpvalues(withUpvalue)
for k, v in pairs(upvals) do
    print(string.format("  %s = %s", k, TypeInspector.inspect(v)))
end

print("\nInspect complex object:")
local obj = {
    name = "Test",
    data = { 1, 2, 3 },
    nested = { x = 10, y = 20 }
}
print(TypeInspector.inspect(obj))
```

---

## 15. Schema Validation via Metaprogramming

```lua
-- Auto schema validation เมื่อ set fields
local function createSchemaClass(name, schema)
    local class = {}
    class.__index = class
    class._name = name
    class._schema = schema
    
    -- Validate value against field schema
    local function validateField(fieldName, value, fieldSchema)
        if fieldSchema.required and value == nil then
            return false, fieldName .. " is required"
        end
        
        if value ~= nil then
            if fieldSchema.type and type(value) ~= fieldSchema.type then
                return false, string.format("%s must be %s, got %s", 
                    fieldName, fieldSchema.type, type(value))
            end
            
            if fieldSchema.validate then
                local ok, err = fieldSchema.validate(value)
                if not ok then
                    return false, fieldName .. ": " .. (err or "invalid")
                end
            end
        end
        
        return true
    end
    
    -- __newindex to validate on set
    class.__newindex = function(self, key, value)
        local fieldSchema = schema[key]
        if fieldSchema then
            local ok, err = validateField(key, value, fieldSchema)
            if not ok then
                error("Schema validation: " .. err, 2)
            end
        end
        rawset(self, key, value)
    end
    
    function class.new(data)
        local obj = setmetatable({}, class)
        
        -- Set defaults
        for fieldName, fieldSchema in pairs(schema) do
            if fieldSchema.default ~= nil then
                rawset(obj, fieldName, fieldSchema.default)
            end
        end
        
        -- Apply initial data (with validation)
        if data then
            for k, v in pairs(data) do
                obj[k] = v  -- triggers __newindex
            end
        end
        
        return obj
    end
    
    function class:toTable()
        local result = {}
        for k, _ in pairs(schema) do
            result[k] = self[k]
        end
        return result
    end
    
    return class
end

-- สร้าง schema classes
local User = createSchemaClass("User", {
    name = {
        type = "string",
        required = true,
        validate = function(v)
            if #v < 2 then return false, "too short" end
            return true
        end
    },
    email = {
        type = "string",
        required = true,
        validate = function(v)
            return v:match("@") ~= nil, "invalid email"
        end
    },
    age = {
        type = "number",
        validate = function(v)
            return v >= 0 and v <= 150, "must be 0-150"
        end
    },
    role = {
        type = "string",
        default = "user"
    }
})

print("Creating valid user:")
local alice = User.new({ name = "Alice", email = "alice@example.com", age = 30 })
print("Name:", alice.name)
print("Role:", alice.role)  -- default value

print("\nTrying invalid user:")
local ok, err = pcall(function()
    local bad = User.new({ name = "X", email = "invalid" })
end)
print("Error:", err)

print("\nTrying to set invalid age:")
local ok2, err2 = pcall(function()
    alice.age = 200
end)
print("Error:", err2)
```

---

## 16. Reactive Properties

```lua
-- Reactive Properties - เมื่อ property เปลี่ยน, dependent properties update อัตโนมัติ
local Reactive = {}
Reactive.__index = Reactive

function Reactive.new(data)
    local obj = setmetatable({}, Reactive)
    obj._data = {}
    obj._computed = {}
    obj._watchers = {}
    obj._dirty = {}
    
    -- Set initial data
    for k, v in pairs(data or {}) do
        obj._data[k] = v
    end
    
    return obj
end

function Reactive:computed(name, deps, compute)
    self._computed[name] = {
        deps = deps,
        compute = compute,
        cached = nil,
        isDirty = true
    }
    
    -- Mark as dirty when deps change
    for _, dep in ipairs(deps) do
        if not self._watchers[dep] then
            self._watchers[dep] = {}
        end
        table.insert(self._watchers[dep], function()
            self._computed[name].isDirty = true
        end)
    end
end

function Reactive:set(key, value)
    local old = self._data[key]
    if old == value then return end
    
    self._data[key] = value
    
    -- Notify watchers
    if self._watchers[key] then
        for _, watcher in ipairs(self._watchers[key]) do
            watcher(value, old)
        end
    end
end

function Reactive:get(key)
    -- Check computed first
    if self._computed[key] then
        local computed = self._computed[key]
        if computed.isDirty then
            computed.cached = computed.compute(self)
            computed.isDirty = false
        end
        return computed.cached
    end
    
    return self._data[key]
end

function Reactive:watch(key, fn)
    if not self._watchers[key] then
        self._watchers[key] = {}
    end
    table.insert(self._watchers[key], fn)
end

-- Proxy interface
function Reactive:createProxy()
    return setmetatable({}, {
        __index = function(t, key)
            return self:get(key)
        end,
        __newindex = function(t, key, value)
            self:set(key, value)
        end
    })
end

-- ใช้งาน
local store = Reactive.new({
    firstName = "Alice",
    lastName = "Smith",
    price = 100,
    quantity = 5
})

-- Define computed properties
store:computed("fullName", { "firstName", "lastName" }, function(s)
    return s:get("firstName") .. " " .. s:get("lastName")
end)

store:computed("total", { "price", "quantity" }, function(s)
    return s:get("price") * s:get("quantity")
end)

-- Watch for changes
store:watch("firstName", function(newVal, oldVal)
    print(string.format("firstName changed: %s -> %s", oldVal, newVal))
end)

-- Use via proxy
local proxy = store:createProxy()

print("Full name:", proxy.fullName)
print("Total:", proxy.total)

proxy.firstName = "Bob"
print("New full name:", proxy.fullName)

proxy.price = 150
print("New total:", proxy.total)
```

---

## 17. Method Chaining ด้วย Metaprogramming

```lua
-- Auto method chaining ทุก method ที่ return nil
local function makeChainable(class)
    local wrapped = {}
    
    setmetatable(wrapped, {
        __index = class,
        __newindex = class
    })
    
    for name, method in pairs(class) do
        if type(method) == "function" then
            wrapped[name] = function(self, ...)
                local results = { method(self, ...) }
                
                -- If method returns nothing, return self for chaining
                if #results == 0 or results[1] == nil then
                    return self
                end
                
                return table.unpack(results)
            end
        end
    end
    
    return wrapped
end

-- Query Builder ที่ใช้ method chaining
local QueryBuilder = {}
QueryBuilder.__index = QueryBuilder

function QueryBuilder.new(table_name)
    return setmetatable({
        _table = table_name,
        _conditions = {},
        _fields = { "*" },
        _orderBy = nil,
        _limit = nil,
        _offset = nil
    }, QueryBuilder)
end

function QueryBuilder:select(...)
    self._fields = { ... }
end

function QueryBuilder:where(condition)
    table.insert(self._conditions, condition)
end

function QueryBuilder:orderBy(field, direction)
    self._orderBy = field .. " " .. (direction or "ASC")
end

function QueryBuilder:limit(n)
    self._limit = n
end

function QueryBuilder:offset(n)
    self._offset = n
end

function QueryBuilder:build()
    local sql = string.format("SELECT %s FROM %s",
        table.concat(self._fields, ", "),
        self._table)
    
    if #self._conditions > 0 then
        sql = sql .. " WHERE " .. table.concat(self._conditions, " AND ")
    end
    
    if self._orderBy then
        sql = sql .. " ORDER BY " .. self._orderBy
    end
    
    if self._limit then
        sql = sql .. " LIMIT " .. self._limit
    end
    
    if self._offset then
        sql = sql .. " OFFSET " .. self._offset
    end
    
    return sql
end

-- Make chainable
QueryBuilder = makeChainable(QueryBuilder)

local query = QueryBuilder.new("users")
    :select("id", "name", "email")
    :where("age > 18")
    :where("active = 1")
    :orderBy("name")
    :limit(10)
    :offset(0)
    :build()

print("Generated SQL:")
print(query)
```

---

## 18. Dependency Injection via Metaprogramming

```lua
-- Dependency Injection Container
local Container = {}
Container.__index = Container

function Container.new()
    return setmetatable({
        _bindings = {},
        _singletons = {},
        _instances = {}
    }, Container)
end

function Container:bind(name, factory)
    self._bindings[name] = { factory = factory, singleton = false }
    return self
end

function Container:singleton(name, factory)
    self._bindings[name] = { factory = factory, singleton = true }
    return self
end

function Container:instance(name, obj)
    self._instances[name] = obj
    return self
end

function Container:make(name)
    -- Check cached instances
    if self._instances[name] then
        return self._instances[name]
    end
    
    local binding = self._bindings[name]
    if not binding then
        error("No binding found for: " .. name)
    end
    
    -- For singletons, check cache
    if binding.singleton and self._singletons[name] then
        return self._singletons[name]
    end
    
    -- Create instance, passing container for further resolution
    local instance = binding.factory(self)
    
    if binding.singleton then
        self._singletons[name] = instance
    end
    
    return instance
end

-- Auto-inject via __index
function Container:proxy()
    return setmetatable({}, {
        __index = function(t, key)
            local ok, result = pcall(function()
                return self:make(key)
            end)
            if ok then return result end
            return nil
        end
    })
end

-- Test DI container
local container = Container.new()

-- Register services
container:singleton("logger", function(c)
    return {
        log = function(self, msg)
            print("[Logger] " .. msg)
        end
    }
end)

container:singleton("db", function(c)
    local logger = c:make("logger")
    return {
        query = function(self, sql)
            logger:log("Executing: " .. sql)
            return { rows = {}, count = 0 }
        end
    }
end)

container:bind("userRepo", function(c)
    local db = c:make("db")
    return {
        find = function(self, id)
            return db:query("SELECT * FROM users WHERE id=" .. id)
        end
    }
end)

-- Use services
local logger = container:make("logger")
logger:log("Application started")

local db = container:make("db")
db:query("SELECT * FROM products")

local userRepo = container:make("userRepo")
userRepo:find(1)

-- Verify singleton behavior
local logger2 = container:make("logger")
print("Same logger instance:", logger == logger2)
```

---

## 19. Event-Driven Metaprogramming

```lua
-- Event-driven class system
local EventDrivenClass = {}

function EventDrivenClass.create(config)
    local class = {}
    class.__index = class
    class._eventHooks = {}
    
    -- Register lifecycle hooks
    if config.hooks then
        for event, handlers in pairs(config.hooks) do
            class._eventHooks[event] = handlers
        end
    end
    
    -- Wrap methods with lifecycle events
    if config.methods then
        for name, method in pairs(config.methods) do
            class[name] = function(self, ...)
                -- before:methodName
                local beforeEvent = "before:" .. name
                if class._eventHooks[beforeEvent] then
                    for _, hook in ipairs(class._eventHooks[beforeEvent]) do
                        hook(self, ...)
                    end
                end
                
                -- Execute method
                local results = { method(self, ...) }
                
                -- after:methodName
                local afterEvent = "after:" .. name
                if class._eventHooks[afterEvent] then
                    for _, hook in ipairs(class._eventHooks[afterEvent]) do
                        hook(self, results[1], ...)
                    end
                end
                
                return table.unpack(results)
            end
        end
    end
    
    function class.new(data)
        local obj = setmetatable(data or {}, class)
        
        if class._eventHooks["initialize"] then
            for _, hook in ipairs(class._eventHooks["initialize"]) do
                hook(obj)
            end
        end
        
        return obj
    end
    
    return class
end

-- Test event-driven class
local Article = EventDrivenClass.create({
    methods = {
        publish = function(self)
            self.status = "published"
            self.publishedAt = os.time()
        end,
        archive = function(self)
            self.status = "archived"
        end,
        save = function(self)
            print("Saving article:", self.title)
            return true
        end
    },
    hooks = {
        initialize = {
            function(self)
                print("Article initialized:", self.title)
                self.status = self.status or "draft"
            end
        },
        ["before:publish"] = {
            function(self)
                print("Before publish - validating:", self.title)
                if not self.title or #self.title < 3 then
                    error("Title too short")
                end
            end
        },
        ["after:publish"] = {
            function(self)
                print("After publish - sending notifications for:", self.title)
            end
        },
        ["before:save"] = {
            function(self)
                self.updatedAt = os.time()
                print("Touching updatedAt")
            end
        }
    }
})

local article = Article.new({ title = "Hello Lua Metaprogramming" })
article:publish()
article:save()
print("Status:", article.status)
```

---

## 20. Complete Metaprogramming Framework

```lua
-- Complete Metaprogramming Framework
local Meta = {}

-- Define a class with full meta support
function Meta.defineClass(name, config)
    config = config or {}
    
    local class = {}
    class.__index = class
    class._meta = {
        name = name,
        abstract = config.abstract or false,
        sealed = config.sealed or false
    }
    
    -- Inherit from parent
    if config.extends then
        if config.extends._meta.sealed then
            error("Cannot extend sealed class: " .. config.extends._meta.name)
        end
        setmetatable(class, { __index = config.extends })
    end
    
    -- Apply mixins
    for _, mixin in ipairs(config.mixins or {}) do
        for k, v in pairs(mixin) do
            if not k:match("^_") and not class[k] then
                class[k] = v
            end
        end
    end
    
    -- Add methods
    for name_, method in pairs(config.methods or {}) do
        class[name_] = method
    end
    
    -- Virtual methods
    for _, virtualName in ipairs(config.virtual or {}) do
        class[virtualName] = function(self, ...)
            error(string.format(
                "Abstract method '%s' must be implemented in class '%s'",
                virtualName,
                getmetatable(self) and getmetatable(self)._meta and getmetatable(self)._meta.name or "?"
            ))
        end
    end
    
    -- Constructor
    function class.new(...)
        if class._meta.abstract then
            error("Cannot instantiate abstract class: " .. name)
        end
        
        local obj = setmetatable({}, class)
        
        if obj.constructor then
            obj:constructor(...)
        end
        
        return obj
    end
    
    -- instanceof
    function class.instanceof(obj)
        local mt = getmetatable(obj)
        while mt do
            if mt == class then return true end
            local parentMt = getmetatable(mt)
            mt = parentMt and parentMt.__index
        end
        return false
    end
    
    return class
end

-- Test the framework
local Shape = Meta.defineClass("Shape", {
    abstract = true,
    virtual = { "area", "perimeter" },
    methods = {
        constructor = function(self, color)
            self.color = color or "black"
        end,
        describe = function(self)
            return string.format("I am a %s %s with area=%.2f",
                self.color,
                getmetatable(self)._meta.name,
                self:area())
        end
    }
})

local Circle = Meta.defineClass("Circle", {
    extends = Shape,
    methods = {
        constructor = function(self, radius, color)
            Shape.constructor(self, color)
            self.radius = radius
        end,
        area = function(self)
            return math.pi * self.radius ^ 2
        end,
        perimeter = function(self)
            return 2 * math.pi * self.radius
        end
    }
})

local Rectangle = Meta.defineClass("Rectangle", {
    extends = Shape,
    methods = {
        constructor = function(self, w, h, color)
            Shape.constructor(self, color)
            self.width = w
            self.height = h
        end,
        area = function(self)
            return self.width * self.height
        end,
        perimeter = function(self)
            return 2 * (self.width + self.height)
        end
    }
})

-- Test
local ok, err = pcall(function() Shape.new() end)
print("Abstract error:", err and err:match("abstract") ~= nil)

local c = Circle.new(5, "red")
print(c:describe())
print("Circle instanceof Shape:", Shape.instanceof(c))
print("Circle instanceof Circle:", Circle.instanceof(c))
print("Circle instanceof Rectangle:", Rectangle.instanceof(c))

local r = Rectangle.new(4, 6, "blue")
print(r:describe())
```

---

## สรุป

บทนี้ครอบคลุม:

1. **Metaprogramming คืออะไร** - Code ที่จัดการ code อื่น
2. **load() และ loadstring()** - Execute code จาก string
3. **Dynamic Method Generation** - สร้าง methods ขณะ runtime ด้วย `__index`
4. **Proxy Objects** - Intercept operations บน objects
5. **Transparent Proxies** - Proxies ที่ดูเหมือน original
6. **Method Missing** - Handle unknown method calls
7. **Dynamic Dispatch** - เลือก method ตาม argument types
8. **Code Generation** - สร้าง Lua code แบบ programmatic
9. **Template-based Generation** - Template system สำหรับ code gen
10. **Auto-memoization** - Cache function results อัตโนมัติ
11. **Auto-logging** - Log method calls โดยไม่แก้ code
12. **DSL Construction** - Build mini languages
13. **Macro Patterns** - Compile-time-like code transformation
14. **Runtime Type Inspection** - Introspect objects ขณะ runtime
15. **Schema Validation** - Auto-validate via `__newindex`
16. **Reactive Properties** - Properties ที่ update อัตโนมัติ
17. **Method Chaining** - Auto chain methods
18. **Dependency Injection** - DI container
19. **Event-Driven Classes** - Lifecycle hooks
20. **Complete Framework** - Framework ที่รวมทุกอย่าง

Metaprogramming ใน Lua ทรงพลังมากเพราะ metatables ทำให้เราสามารถ intercept operations ได้เกือบทุกอย่าง
