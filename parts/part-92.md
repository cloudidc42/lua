# บทที่ 92: Type System และ Static Analysis ใน Lua

## บทนำ

Lua เป็นภาษา dynamically-typed แต่ในโปรเจกต์ขนาดใหญ่ การมี type information ช่วยให้ code ปลอดภัยและบำรุงรักษาง่ายขึ้น บทนี้จะสอนเครื่องมือต่างๆ สำหรับ static analysis

---

## 92.1 Lua's Dynamic Type System

```lua
-- Lua ไม่มี static types แต่เรายังตรวจสอบ type ได้
local x = 42
print(type(x))         -- "number"

x = "hello"
print(type(x))         -- "string"

x = { 1, 2, 3 }
print(type(x))         -- "table"

x = function() end
print(type(x))         -- "function"

x = nil
print(type(x))         -- "nil"

-- type checking at runtime
local function add(a, b)
    assert(type(a) == "number", "a must be number, got " .. type(a))
    assert(type(b) == "number", "b must be number, got " .. type(b))
    return a + b
end

print(add(1, 2))       -- 3
-- add("x", 2)          -- error: a must be number, got string
```

---

## 92.2 EmmyLua Type Annotations

EmmyLua annotations ช่วยให้ IDE (VS Code + lua-language-server) เข้าใจ types:

```lua
-- EmmyLua annotation syntax

-- Basic types
---@type number
local count = 0

---@type string
local name = "Lua"

---@type boolean
local active = true

---@type table
local data = {}

-- Union types
---@type number | string
local id = 42  -- หรือ "abc"

-- Nullable types
---@type string | nil
local optional_name = nil

-- Function type
---@type fun(x: number, y: number): number
local add_fn

-- Class definition
---@class Vector2
---@field x number X coordinate
---@field y number Y coordinate
local Vector2 = {}
Vector2.__index = Vector2

---@param x number
---@param y number
---@return Vector2
function Vector2.new(x, y)
    return setmetatable({ x = x, y = y }, Vector2)
end

---@param other Vector2
---@return Vector2
function Vector2:add(other)
    return Vector2.new(self.x + other.x, self.y + other.y)
end

---@return number
function Vector2:length()
    return math.sqrt(self.x^2 + self.y^2)
end

---@return string
function Vector2:__tostring()
    return string.format("Vector2(%g, %g)", self.x, self.y)
end

-- ตอนนี้ IDE จะ auto-complete .x .y ได้ถูกต้อง
local v1 = Vector2.new(3, 4)
local v2 = Vector2.new(1, 2)
local v3 = v1:add(v2)
print(tostring(v1))    -- Vector2(3, 4)
print(v1:length())     -- 5.0
```

---

## 92.3 Advanced EmmyLua Annotations

```lua
-- Generic types
---@class Stack<T>
---@field private items T[]
---@field private size number
local Stack = {}
Stack.__index = Stack

---@generic T
---@return Stack<T>
function Stack.new()
    return setmetatable({ items = {}, size = 0 }, Stack)
end

---@generic T
---@param item T
function Stack:push(item)
    self.size = self.size + 1
    self.items[self.size] = item
end

---@generic T
---@return T | nil
function Stack:pop()
    if self.size == 0 then return nil end
    local item = self.items[self.size]
    self.items[self.size] = nil
    self.size = self.size - 1
    return item
end

---@generic T
---@return T | nil
function Stack:peek()
    return self.items[self.size]
end

-- Enum simulation with annotations
---@enum Color
local Color = {
    RED   = "red",
    GREEN = "green",
    BLUE  = "blue"
}

---@param color Color
---@return string
local function color_to_hex(color)
    local map = {
        [Color.RED]   = "#FF0000",
        [Color.GREEN] = "#00FF00",
        [Color.BLUE]  = "#0000FF"
    }
    return map[color] or "#000000"
end

print(color_to_hex(Color.RED))    -- #FF0000

-- Tuple types
---@return number, string, boolean  Returns id, name, active
local function get_user()
    return 1, "Alice", true
end

local id, username, is_active = get_user()

-- Overloaded functions
---@overload fun(x: number): number
---@overload fun(x: string): string
---@param x number | string
---@return number | string
local function identity(x)
    return x
end

-- Type aliases
---@alias Callback fun(err: string | nil, result: any)
---@alias Handler fun(request: table): table

---@param cb Callback
local function async_op(cb)
    -- simulate async
    cb(nil, { status = "ok" })
end

-- Interface-like patterns
---@class Serializable
---@field serialize fun(self: Serializable): string
---@field deserialize fun(self: Serializable, data: string): boolean

---@class Animal : Serializable
---@field name string
---@field sound string
local Animal = {}
Animal.__index = Animal

function Animal.new(name, sound)
    return setmetatable({ name=name, sound=sound }, Animal)
end

function Animal:speak()
    print(self.name .. " says " .. self.sound)
end

function Animal:serialize()
    return string.format('{"name":"%s","sound":"%s"}', self.name, self.sound)
end

local dog = Animal.new("Rex", "Woof")
dog:speak()
print(dog:serialize())
```

---

## 92.4 Teal - Typed Lua Language

Teal เป็น typed superset ของ Lua คล้าย TypeScript สำหรับ JavaScript:

```
-- Teal syntax (.tl files)
-- ไม่ใช่ Lua ปกติ แต่แสดงให้เห็น concept

-- Variable types
local x: integer = 42
local name: string = "Lua"
local active: boolean = true

-- Function types  
local function add(a: integer, b: integer): integer
    return a + b
end

-- Record types (like class)
local record Vector2
    x: number
    y: number
end

local function new_vec(x: number, y: number): Vector2
    return { x = x, y = y }
end

-- Generics
local function first<T>(items: {T}): T
    return items[1]
end

-- Union types
local function parse(input: string): integer | boolean
    local n = tonumber(input)
    if n then return math.floor(n) end
    if input == "true" then return true end
    if input == "false" then return false end
    error("Cannot parse: " .. input)
end

-- Enum
local enum Direction
    "north"
    "south" 
    "east"
    "west"
end

-- Type errors caught at compile time!
-- local bad: integer = "hello"  -- ERROR: type mismatch
```

---

## 92.5 luacheck - Static Analyzer

```bash
# ติดตั้ง luacheck
luarocks install luacheck

# ใช้งาน
luacheck myfile.lua
luacheck src/        # ตรวจทั้ง directory
luacheck --globals love --std love src/  # custom globals

# ตัวอย่าง .luacheckrc config file
```

```lua
-- .luacheckrc (Lua format)
return {
    -- Global variables ที่อนุญาต
    globals = {
        "love",         -- LÖVE2D
        "ngx",          -- OpenResty
        "vim",          -- Neovim plugin
        "redis",        -- Redis Lua scripting
    },
    
    -- Read-only globals
    read_globals = {
        "jit",          -- LuaJIT
        "bit",          -- Bit library
    },
    
    -- Ignore specific warnings
    ignore = {
        "212",          -- Unused argument
        "213",          -- Unused loop variable
    },
    
    -- Max line length
    max_line_length = 120,
    
    -- Max cyclomatic complexity
    max_cyclomatic_complexity = 10,
    
    -- Per-file settings
    files = {
        ["tests/"] = {
            globals = { "describe", "it", "before_each", "after_each" }
        }
    },
    
    -- Exclude files
    exclude_files = {
        "vendor/",
        "build/"
    }
}
```

---

## 92.6 Gradual Typing System

สร้าง runtime type checking system ที่ optional:

```lua
-- Gradual Type System
local types = {}

-- Type constructors
types.number = { name = "number", check = function(v) return type(v) == "number" end }
types.string = { name = "string", check = function(v) return type(v) == "string" end }
types.boolean = { name = "boolean", check = function(v) return type(v) == "boolean" end }
types.table = { name = "table", check = function(v) return type(v) == "table" end }
types.func = { name = "function", check = function(v) return type(v) == "function" end }
types.any = { name = "any", check = function() return true end }
types.nil_t = { name = "nil", check = function(v) return v == nil end }

-- Optional type
function types.optional(t)
    return {
        name = t.name .. "?",
        check = function(v) return v == nil or t.check(v) end
    }
end

-- Union type
function types.union(...)
    local ts = { ... }
    local names = {}
    for _, t in ipairs(ts) do table.insert(names, t.name) end
    return {
        name = table.concat(names, " | "),
        check = function(v)
            for _, t in ipairs(ts) do
                if t.check(v) then return true end
            end
            return false
        end
    }
end

-- List type
function types.list(item_type)
    return {
        name = item_type.name .. "[]",
        check = function(v)
            if type(v) ~= "table" then return false end
            for _, item in ipairs(v) do
                if not item_type.check(item) then return false end
            end
            return true
        end
    }
end

-- Record type
function types.record(fields)
    local field_names = {}
    for k in pairs(fields) do table.insert(field_names, k) end
    return {
        name = "{ " .. table.concat(field_names, ", ") .. " }",
        check = function(v)
            if type(v) ~= "table" then return false end
            for field, field_type in pairs(fields) do
                if not field_type.check(v[field]) then
                    return false
                end
            end
            return true
        end
    }
end

-- Typed function wrapper
local ENABLE_TYPE_CHECKING = true

function types.typed(fn, param_types, return_types)
    if not ENABLE_TYPE_CHECKING then return fn end
    
    return function(...)
        local args = { ... }
        
        -- Check parameters
        if param_types then
            for i, param_type in ipairs(param_types) do
                local val = args[i]
                if not param_type.check(val) then
                    error(string.format(
                        "Type error: argument %d expected %s, got %s (%s)",
                        i, param_type.name, type(val), tostring(val)
                    ), 2)
                end
            end
        end
        
        local results = { fn(...) }
        
        -- Check return values
        if return_types then
            for i, ret_type in ipairs(return_types) do
                local val = results[i]
                if not ret_type.check(val) then
                    error(string.format(
                        "Type error: return value %d expected %s, got %s",
                        i, ret_type.name, type(val)
                    ), 2)
                end
            end
        end
        
        return table.unpack(results)
    end
end

-- การใช้งาน
local add = types.typed(
    function(a, b) return a + b end,
    { types.number, types.number },
    { types.number }
)

print(add(1, 2))        -- 3
-- add("x", 2)          -- Type error: argument 1 expected number, got string

-- Complex types
local UserType = types.record({
    name = types.string,
    age = types.number,
    email = types.optional(types.string)
})

local create_user = types.typed(
    function(name, age, email)
        return { name=name, age=age, email=email }
    end,
    { types.string, types.number, types.optional(types.string) },
    { UserType }
)

local user = create_user("Alice", 30)
print(user.name, user.age)   -- Alice  30

local NumberList = types.list(types.number)
local sum = types.typed(
    function(nums)
        local total = 0
        for _, n in ipairs(nums) do total = total + n end
        return total
    end,
    { NumberList },
    { types.number }
)

print(sum({1, 2, 3, 4, 5}))  -- 15
```

---

## 92.7 lua-language-server Configuration

```json
// .vscode/settings.json
{
    "Lua.workspace.library": [
        "${3rd}/love2d/library",
        "./lib"
    ],
    "Lua.diagnostics.globals": [
        "love",
        "ngx",
        "redis"
    ],
    "Lua.diagnostics.disable": [
        "lowercase-global"
    ],
    "Lua.runtime.version": "Lua 5.4",
    "Lua.completion.enable": true,
    "Lua.hover.enable": true,
    "Lua.signatureHelp.enable": true,
    "Lua.diagnostics.enable": true,
    "Lua.format.enable": true,
    "Lua.format.defaultConfig": {
        "indent_style": "space",
        "indent_size": "4"
    }
}
```

```lua
-- .luarc.json equivalent in Lua
-- สร้างไฟล์ .luarc.json ใน root directory

--[[
{
    "runtime": {
        "version": "Lua 5.4",
        "pathStrict": true
    },
    "diagnostics": {
        "globals": ["love", "ngx"],
        "disable": ["lowercase-global"],
        "severity": {
            "undefined-global": "Warning",
            "unused-local": "Information"
        }
    },
    "workspace": {
        "library": ["./lib", "./vendor"],
        "ignoreDir": ["build", ".git"]
    },
    "completion": {
        "workspaceWord": true,
        "callSnippet": "Replace"
    }
}
]]
```

---

## 92.8 Custom Type Checker

```lua
-- Mini type inference engine
local TypeInfer = {}
TypeInfer.__index = TypeInfer

function TypeInfer.new()
    return setmetatable({
        env = {},      -- variable -> type
        errors = {}
    }, TypeInfer)
end

function TypeInfer:infer(ast_node)
    -- Simplified AST node types
    if ast_node.kind == "number" then
        return "number"
    elseif ast_node.kind == "string" then
        return "string"
    elseif ast_node.kind == "boolean" then
        return "boolean"
    elseif ast_node.kind == "nil" then
        return "nil"
    elseif ast_node.kind == "var" then
        return self.env[ast_node.name] or "unknown"
    elseif ast_node.kind == "assign" then
        local val_type = self:infer(ast_node.value)
        self.env[ast_node.name] = val_type
        return val_type
    elseif ast_node.kind == "binop" then
        return self:infer_binop(ast_node)
    elseif ast_node.kind == "call" then
        return self:infer_call(ast_node)
    end
    return "unknown"
end

function TypeInfer:infer_binop(node)
    local left = self:infer(node.left)
    local right = self:infer(node.right)
    
    if node.op == "+" or node.op == "-" or 
       node.op == "*" or node.op == "/" then
        if left ~= "number" or right ~= "number" then
            table.insert(self.errors, string.format(
                "Type error: '%s' requires numbers, got %s and %s",
                node.op, left, right
            ))
            return "error"
        end
        return "number"
    elseif node.op == ".." then
        if (left ~= "string" and left ~= "number") or
           (right ~= "string" and right ~= "number") then
            table.insert(self.errors, "Type error: '..' requires strings/numbers")
            return "error"
        end
        return "string"
    elseif node.op == "==" or node.op == "~=" then
        return "boolean"
    elseif node.op == "<" or node.op == ">" or 
           node.op == "<=" or node.op == ">=" then
        if left ~= right then
            table.insert(self.errors, string.format(
                "Type error: comparison between %s and %s", left, right
            ))
        end
        return "boolean"
    end
    return "unknown"
end

function TypeInfer:infer_call(node)
    -- Look up function return type
    local func_types = {
        tostring = "string",
        tonumber = { "number", "nil" },
        type = "string",
        print = "nil",
        pairs = "function",
        ipairs = "function"
    }
    return func_types[node.name] or "unknown"
end

function TypeInfer:report()
    if #self.errors == 0 then
        print("Type check passed!")
    else
        print("Type errors found:")
        for _, err in ipairs(self.errors) do
            print("  ERROR:", err)
        end
    end
    print("Variable types:")
    for name, t in pairs(self.env) do
        print(string.format("  %s: %s", name, t))
    end
end

-- Simulate type checking a simple program
local checker = TypeInfer.new()

-- x = 42
checker:infer({ kind="assign", name="x", value={ kind="number" } })

-- y = "hello"
checker:infer({ kind="assign", name="y", value={ kind="string" } })

-- z = x + x (valid)
checker:infer({ kind="assign", name="z", value={
    kind="binop", op="+",
    left={ kind="var", name="x" },
    right={ kind="var", name="x" }
}})

-- bad = x + y (type error!)
checker:infer({ kind="assign", name="bad", value={
    kind="binop", op="+",
    left={ kind="var", name="x" },
    right={ kind="var", name="y" }
}})

checker:report()
```

---

## 92.9 Runtime Type Contracts

```lua
-- Design by Contract สำหรับ Lua

local Contract = {}
Contract.__index = Contract

-- Enable/disable contracts globally
Contract.enabled = true

-- Pre-condition check
function Contract.requires(condition, message)
    if not Contract.enabled then return end
    if not condition then
        error("Precondition failed: " .. (message or "unknown"), 2)
    end
end

-- Post-condition check
function Contract.ensures(condition, message)
    if not Contract.enabled then return end
    if not condition then
        error("Postcondition failed: " .. (message or "unknown"), 2)
    end
end

-- Invariant check
function Contract.invariant(obj, check_fn, message)
    if not Contract.enabled then return end
    if not check_fn(obj) then
        error("Invariant violated: " .. (message or "unknown"), 2)
    end
end

-- Contract-decorated function
function Contract.decorate(spec)
    return function(fn)
        return function(...)
            if Contract.enabled then
                -- Check preconditions
                if spec.requires then
                    spec.requires(...)
                end
            end
            
            local results = { fn(...) }
            
            if Contract.enabled then
                -- Check postconditions
                if spec.ensures then
                    spec.ensures(table.unpack(results))
                end
            end
            
            return table.unpack(results)
        end
    end
end

-- ตัวอย่าง: Bank account with contracts
local BankAccount = {}
BankAccount.__index = BankAccount

function BankAccount.new(initial_balance)
    Contract.requires(type(initial_balance) == "number", 
        "initial_balance must be a number")
    Contract.requires(initial_balance >= 0, 
        "initial_balance must be non-negative")
    
    local account = setmetatable({ 
        balance = initial_balance,
        owner = "unknown"
    }, BankAccount)
    
    -- Check invariant
    Contract.invariant(account, 
        function(a) return a.balance >= 0 end,
        "Balance must be non-negative")
    
    return account
end

function BankAccount:deposit(amount)
    Contract.requires(type(amount) == "number", "amount must be number")
    Contract.requires(amount > 0, "deposit amount must be positive")
    
    local old_balance = self.balance
    self.balance = self.balance + amount
    
    Contract.ensures(self.balance == old_balance + amount, 
        "Balance should increase by amount")
    Contract.ensures(self.balance >= 0, 
        "Balance must remain non-negative")
    
    return self.balance
end

function BankAccount:withdraw(amount)
    Contract.requires(type(amount) == "number", "amount must be number")
    Contract.requires(amount > 0, "withdrawal amount must be positive")
    Contract.requires(amount <= self.balance, 
        string.format("Insufficient funds: have %.2f, need %.2f", 
            self.balance, amount))
    
    local old_balance = self.balance
    self.balance = self.balance - amount
    
    Contract.ensures(self.balance == old_balance - amount,
        "Balance should decrease by amount")
    Contract.ensures(self.balance >= 0,
        "Balance must remain non-negative")
    
    return self.balance
end

-- Test contracts
local acc = BankAccount.new(1000)
print("Balance:", acc:deposit(500))    -- 1500
print("Balance:", acc:withdraw(200))   -- 1300

-- Test violation
local ok, err = pcall(function()
    acc:withdraw(5000)  -- Should fail contract
end)
print("Expected error:", ok, err)
```

---

## 92.10 Type-Safe Table Access

```lua
-- Typed Table wrapper
local TypedTable = {}
TypedTable.__index = TypedTable

function TypedTable.new(schema)
    local tt = setmetatable({
        _schema = schema,
        _data = {}
    }, TypedTable)
    
    -- Set default values
    for field, def in pairs(schema) do
        if def.default ~= nil then
            tt._data[field] = def.default
        end
    end
    
    return tt
end

function TypedTable:__newindex(key, value)
    local def = self._schema[key]
    
    if not def then
        error(string.format("Unknown field: '%s'", key), 2)
    end
    
    -- Type check
    if def.type and not def.type.check(value) then
        error(string.format(
            "Type error for field '%s': expected %s, got %s",
            key, def.type.name, type(value)
        ), 2)
    end
    
    -- Validation
    if def.validate then
        local ok, msg = def.validate(value)
        if not ok then
            error(string.format("Validation failed for '%s': %s", key, msg), 2)
        end
    end
    
    self._data[key] = value
end

function TypedTable:__index(key)
    if TypedTable[key] then return TypedTable[key] end
    
    local val = self._data[key]
    if val == nil then
        local def = self._schema[key]
        if def and def.required then
            error(string.format("Required field '%s' not set", key), 2)
        end
    end
    return val
end

function TypedTable:validate_all()
    local errors = {}
    for field, def in pairs(self._schema) do
        if def.required and self._data[field] == nil then
            table.insert(errors, "Required field missing: " .. field)
        end
    end
    return #errors == 0, errors
end

-- เครื่องมือสร้าง type descriptors
local T = {
    string = { name = "string", check = function(v) return type(v) == "string" end },
    number = { name = "number", check = function(v) return type(v) == "number" end },
    boolean = { name = "boolean", check = function(v) return type(v) == "boolean" end },
    positive = { 
        name = "positive number", 
        check = function(v) return type(v) == "number" and v > 0 end 
    }
}

-- ใช้งาน: User model with schema
local UserSchema = {
    name = { 
        type = T.string, 
        required = true,
        validate = function(v)
            if #v < 2 then return false, "Name too short (min 2 chars)" end
            if #v > 100 then return false, "Name too long (max 100 chars)" end
            return true
        end
    },
    age = { 
        type = T.positive, 
        required = true,
        validate = function(v)
            if v < 0 or v > 150 then 
                return false, "Age must be 0-150" 
            end
            return true
        end
    },
    email = { type = T.string, required = false },
    active = { type = T.boolean, default = true }
}

local user = TypedTable.new(UserSchema)
user.name = "Alice"
user.age = 30
user.email = "alice@example.com"

print(user.name, user.age, user.active)  -- Alice  30  true

-- Test validation
local ok, err = pcall(function()
    user.age = -5  -- Invalid!
end)
print("Error caught:", err)

local ok2, err2 = pcall(function()
    user.unknown_field = "test"  -- Unknown field!
end)
print("Error caught:", err2)
```

---

## แบบฝึกหัด

1. **เพิ่ม Annotations**: เพิ่ม EmmyLua annotations ให้กับ codebase ที่มีอยู่
2. **Teal Project**: แปลง Lua module เป็น Teal และ compile กลับมา
3. **luacheck Setup**: ตั้งค่า luacheck สำหรับ project ของคุณ
4. **Custom Validator**: สร้าง validation library ที่ใช้ใน API
5. **Type Coverage**: วัด percentage ของ functions ที่มี type annotations

---

*ต่อไป: [Part 93 - Language Server Protocol](part-93.md)*
