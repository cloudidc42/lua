# บทที่ 95: Code Generation และ Metaprogramming ขั้นสูง

## บทนำ

Code generation คือการเขียน code ที่สร้าง code อื่น เป็นเทคนิคอันทรงพลังที่ช่วยลด boilerplate, เพิ่มความสม่ำเสมอ และทำให้ API ดีขึ้น

---

## 95.1 Lua's Code Generation Capabilities

```lua
-- load() - compile and run code at runtime
local code = [[
    local x = 10
    local y = 20
    return x + y
]]

local fn, err = load(code)
if fn then
    print(fn())  -- 30
else
    print("Error:", err)
end

-- loadstring() - alias for load() in some versions
-- load() with environment parameter
local env = { x = 100, print = print }
local fn2 = load("print(x * 2)", "chunk", "t", env)
fn2()  -- 200

-- Dynamic function creation
local function make_adder(n)
    return load(string.format("return function(x) return x + %d end", n))()
end

local add5 = make_adder(5)
local add10 = make_adder(10)
print(add5(3))   -- 8
print(add10(3))  -- 13
```

---

## 95.2 Template-Based Code Generation

```lua
-- Code Template Engine
local CodeTemplate = {}
CodeTemplate.__index = CodeTemplate

function CodeTemplate.new(template)
    return setmetatable({ template = template }, CodeTemplate)
end

function CodeTemplate:render(vars)
    local code = self.template
    
    -- Replace {{VARNAME}} with values
    code = code:gsub("{{(%w+)}}", function(name)
        local val = vars[name]
        if val == nil then
            error("Template variable not found: " .. name)
        end
        return tostring(val)
    end)
    
    -- Replace {{{CODE}}} with raw code (no escaping)
    code = code:gsub("{{{(%w+)}}}", function(name)
        return tostring(vars[name] or "")
    end)
    
    return code
end

-- Template สำหรับ generate getter/setter
local property_template = CodeTemplate.new([[
function {{CLASS}}:get_{{NAME}}()
    return self._{{NAME}}
end

function {{CLASS}}:set_{{NAME}}(value)
    {{VALIDATION}}
    self._{{NAME}} = value
end
]])

-- Generate code
local getters_setters = {}

local properties = {
    { name = "name", validation = "assert(type(value) == 'string', 'name must be string')" },
    { name = "age",  validation = "assert(type(value) == 'number' and value >= 0, 'age must be positive number')" },
    { name = "email", validation = "-- no validation" }
}

for _, prop in ipairs(properties) do
    local code = property_template:render({
        CLASS = "Person",
        NAME = prop.name,
        VALIDATION = prop.validation
    })
    table.insert(getters_setters, code)
end

local generated_code = table.concat(getters_setters, "\n")
print("Generated code:")
print(generated_code)

-- Compile and load
load(generated_code)()
```

---

## 95.3 AST-Based Code Generation

```lua
-- AST Node Types
local AST = {}

function AST.number(n) return { kind="number", value=n } end
function AST.string(s) return { kind="string", value=s } end
function AST.ident(name) return { kind="ident", name=name } end
function AST.binop(op, left, right) return { kind="binop", op=op, left=left, right=right } end
function AST.assign(name, value) return { kind="assign", name=name, value=value } end
function AST.local_assign(name, value) return { kind="local", name=name, value=value } end
function AST.block(stmts) return { kind="block", stmts=stmts } end
function AST.func(params, body) return { kind="func", params=params, body=body } end
function AST.call(func, args) return { kind="call", func=func, args=args } end
function AST.return_stmt(values) return { kind="return", values=values } end
function AST.if_stmt(cond, then_b, else_b) 
    return { kind="if", cond=cond, then_=then_b, else_=else_b }
end
function AST.for_num(var, start, stop, step, body)
    return { kind="for_num", var=var, start=start, stop=stop, step=step, body=body }
end

-- Code Generator from AST
local CodeGen = {}
CodeGen.__index = CodeGen

function CodeGen.new()
    return setmetatable({ indent = 0 }, CodeGen)
end

function CodeGen:indent_str()
    return string.rep("    ", self.indent)
end

function CodeGen:gen(node)
    if node.kind == "number" then
        return tostring(node.value)
    elseif node.kind == "string" then
        return string.format("%q", node.value)
    elseif node.kind == "ident" then
        return node.name
    elseif node.kind == "binop" then
        return string.format("(%s %s %s)",
            self:gen(node.left), node.op, self:gen(node.right))
    elseif node.kind == "assign" then
        return self:indent_str() .. node.name .. " = " .. self:gen(node.value)
    elseif node.kind == "local" then
        return self:indent_str() .. "local " .. node.name .. " = " .. self:gen(node.value)
    elseif node.kind == "block" then
        local lines = {}
        for _, stmt in ipairs(node.stmts) do
            table.insert(lines, self:gen(stmt))
        end
        return table.concat(lines, "\n")
    elseif node.kind == "func" then
        local params = table.concat(node.params, ", ")
        self.indent = self.indent + 1
        local body_code = self:gen(node.body)
        self.indent = self.indent - 1
        return string.format("function(%s)\n%s\nend", params, body_code)
    elseif node.kind == "call" then
        local args = {}
        for _, arg in ipairs(node.args) do
            table.insert(args, self:gen(arg))
        end
        return self:indent_str() .. self:gen(node.func) .. 
               "(" .. table.concat(args, ", ") .. ")"
    elseif node.kind == "return" then
        local vals = {}
        for _, v in ipairs(node.values) do
            table.insert(vals, self:gen(v))
        end
        return self:indent_str() .. "return " .. table.concat(vals, ", ")
    elseif node.kind == "if" then
        local code = self:indent_str() .. "if " .. self:gen(node.cond) .. " then\n"
        self.indent = self.indent + 1
        code = code .. self:gen(node.then_) .. "\n"
        self.indent = self.indent - 1
        if node.else_ then
            code = code .. self:indent_str() .. "else\n"
            self.indent = self.indent + 1
            code = code .. self:gen(node.else_) .. "\n"
            self.indent = self.indent - 1
        end
        code = code .. self:indent_str() .. "end"
        return code
    elseif node.kind == "for_num" then
        local step_code = node.step and (", " .. self:gen(node.step)) or ""
        local code = string.format("%sfor %s = %s, %s%s do\n",
            self:indent_str(), node.var, 
            self:gen(node.start), self:gen(node.stop), step_code)
        self.indent = self.indent + 1
        code = code .. self:gen(node.body) .. "\n"
        self.indent = self.indent - 1
        code = code .. self:indent_str() .. "end"
        return code
    end
    return "-- unknown node: " .. (node.kind or "nil")
end

-- Build a simple function using AST
local gen = CodeGen.new()

-- function factorial(n)
--     if n <= 1 then return 1 end
--     return n * factorial(n - 1)
-- end
local factorial_ast = AST.block({
    AST.if_stmt(
        AST.binop("<=", AST.ident("n"), AST.number(1)),
        AST.block({ AST.return_stmt({ AST.number(1) }) })
    ),
    AST.return_stmt({
        AST.binop("*", AST.ident("n"),
            AST.call(AST.ident("factorial"),
                { AST.binop("-", AST.ident("n"), AST.number(1)) }
            )
        )
    })
})

local code = gen:gen(factorial_ast)
print("Generated AST code:")
print(code)
```

---

## 95.4 ORM Code Generator

```lua
-- Schema definition
local Schema = {
    tables = {}
}

function Schema:table(name, definition)
    local tbl = {
        name = name,
        fields = definition.fields or {},
        relations = definition.relations or {}
    }
    table.insert(self.tables, tbl)
    return tbl
end

-- ORM Code Generator
local ORMGenerator = {}
ORMGenerator.__index = ORMGenerator

function ORMGenerator.new()
    return setmetatable({ output = {} }, ORMGenerator)
end

function ORMGenerator:emit(line)
    table.insert(self.output, line)
end

function ORMGenerator:generate_model(table_def)
    local name = table_def.name
    -- PascalCase from snake_case
    local class_name = name:gsub("_(%a)", function(c) return c:upper() end)
    class_name = class_name:sub(1,1):upper() .. class_name:sub(2)
    
    self:emit("-- Generated Model: " .. class_name)
    self:emit("local " .. class_name .. " = {}")
    self:emit(class_name .. ".__index = " .. class_name)
    self:emit(class_name .. "._table_name = '" .. name .. "'")
    self:emit("")
    
    -- Constructor
    self:emit("function " .. class_name .. ".new(data)")
    self:emit("    return setmetatable(data or {}, " .. class_name .. ")")
    self:emit("end")
    self:emit("")
    
    -- Getters/setters for each field
    for _, field in ipairs(table_def.fields) do
        local fname = field.name
        local ftype = field.type
        
        -- Getter
        self:emit("function " .. class_name .. ":get_" .. fname .. "()")
        self:emit("    return self." .. fname)
        self:emit("end")
        self:emit("")
        
        -- Setter with type validation
        self:emit("function " .. class_name .. ":set_" .. fname .. "(value)")
        if ftype == "string" then
            self:emit("    assert(value == nil or type(value) == 'string', '" .. fname .. " must be string')")
        elseif ftype == "number" or ftype == "integer" then
            self:emit("    assert(value == nil or type(value) == 'number', '" .. fname .. " must be number')")
        elseif ftype == "boolean" then
            self:emit("    assert(value == nil or type(value) == 'boolean', '" .. fname .. " must be boolean')")
        end
        self:emit("    self." .. fname .. " = value")
        self:emit("    return self")
        self:emit("end")
        self:emit("")
    end
    
    -- CRUD methods
    self:emit("function " .. class_name .. ":save(db)")
    self:emit("    if self.id then")
    self:emit("        return self:update(db)")
    self:emit("    else")
    self:emit("        return self:insert(db)")
    self:emit("    end")
    self:emit("end")
    self:emit("")
    
    -- to_table for JSON
    self:emit("function " .. class_name .. ":to_table()")
    self:emit("    return {")
    for _, field in ipairs(table_def.fields) do
        self:emit("        " .. field.name .. " = self." .. field.name .. ",")
    end
    self:emit("    }")
    self:emit("end")
    self:emit("")
    
    -- find_by methods
    for _, field in ipairs(table_def.fields) do
        if field.indexed then
            local mname = field.name:sub(1,1):upper() .. field.name:sub(2)
            self:emit("function " .. class_name .. ".find_by_" .. field.name .. "(db, value)")
            self:emit("    return db:query('SELECT * FROM " .. name .. 
                      " WHERE " .. field.name .. " = ?', value)")
            self:emit("end")
            self:emit("")
        end
    end
    
    self:emit("return " .. class_name)
    self:emit("")
    
    return table.concat(self.output, "\n")
end

-- Define a schema
local user_schema = {
    name = "users",
    fields = {
        { name = "id",         type = "integer", primary = true },
        { name = "username",   type = "string",  indexed = true, unique = true },
        { name = "email",      type = "string",  indexed = true, unique = true },
        { name = "password",   type = "string" },
        { name = "created_at", type = "string" },
        { name = "active",     type = "boolean" }
    }
}

local generator = ORMGenerator.new()
local generated_code = generator:generate_model(user_schema)

print("Generated ORM Model:")
print(generated_code)

-- Load and test
local env = { assert = assert, type = type, setmetatable = setmetatable }
local model_fn, err = load(generated_code, "User", "t", env)
if model_fn then
    local User = model_fn()
    print("\nTesting generated model:")
    local u = User.new({ username="alice", email="alice@example.com" })
    print("  Username:", u:get_username())
    u:set_username("alice_updated")
    print("  Updated:", u:get_username())
    local t = u:to_table()
    print("  to_table:", t.username, t.email)
else
    print("Error loading generated code:", err)
end
```

---

## 95.5 Serialization Code Generator

```lua
-- Efficient serializer generator
-- Generates optimized serialize/deserialize code for a given schema

local SerializerGen = {}
SerializerGen.__index = SerializerGen

function SerializerGen.new()
    return setmetatable({}, SerializerGen)
end

function SerializerGen:generate(schema_name, fields)
    local serialize_parts = {}
    local deserialize_parts = {}
    
    -- Build serialize function
    table.insert(serialize_parts, "function(obj)")
    table.insert(serialize_parts, "    local parts = {}")
    
    for i, field in ipairs(fields) do
        if field.type == "string" then
            table.insert(serialize_parts, string.format(
                '    parts[%d] = "%s=" .. tostring(obj.%s or "")',
                i, field.name, field.name))
        elseif field.type == "number" then
            table.insert(serialize_parts, string.format(
                '    parts[%d] = "%s=" .. tostring(obj.%s or 0)',
                i, field.name, field.name))
        elseif field.type == "boolean" then
            table.insert(serialize_parts, string.format(
                '    parts[%d] = "%s=" .. (obj.%s and "1" or "0")',
                i, field.name, field.name))
        end
    end
    
    table.insert(serialize_parts, '    return table.concat(parts, ",")')
    table.insert(serialize_parts, "end")
    
    -- Build deserialize function
    table.insert(deserialize_parts, "function(data)")
    table.insert(deserialize_parts, "    local obj = {}")
    table.insert(deserialize_parts, "    for pair in data:gmatch('[^,]+') do")
    table.insert(deserialize_parts, "        local k, v = pair:match('([^=]+)=(.*)')")
    table.insert(deserialize_parts, "        if k then")
    
    for _, field in ipairs(fields) do
        table.insert(deserialize_parts, string.format(
            '            if k == "%s" then', field.name))
        if field.type == "number" then
            table.insert(deserialize_parts, string.format(
                '                obj.%s = tonumber(v)', field.name))
        elseif field.type == "boolean" then
            table.insert(deserialize_parts, string.format(
                '                obj.%s = v == "1"', field.name))
        else
            table.insert(deserialize_parts, string.format(
                '                obj.%s = v', field.name))
        end
        table.insert(deserialize_parts, '            end')
    end
    
    table.insert(deserialize_parts, "        end")
    table.insert(deserialize_parts, "    end")
    table.insert(deserialize_parts, "    return obj")
    table.insert(deserialize_parts, "end")
    
    local serialize_code = table.concat(serialize_parts, "\n")
    local deserialize_code = table.concat(deserialize_parts, "\n")
    
    return load("return " .. serialize_code)(),
           load("return " .. deserialize_code)()
end

-- Test
local gen = SerializerGen.new()
local serialize, deserialize = gen:generate("Point3D", {
    { name = "x", type = "number" },
    { name = "y", type = "number" },
    { name = "z", type = "number" },
    { name = "label", type = "string" },
    { name = "visible", type = "boolean" }
})

local point = { x=1.5, y=2.7, z=-0.3, label="Origin", visible=true }
local serialized = serialize(point)
print("Serialized:", serialized)

local restored = deserialize(serialized)
print("Restored:", restored.x, restored.y, restored.z, restored.label, restored.visible)
```

---

## 95.6 Macro System

```lua
-- Macro system สำหรับ Lua
-- Transforms code at load time

local Macro = {}
Macro.__index = Macro

-- Registry of macros
local macro_registry = {}

function Macro.define(name, transformer)
    macro_registry[name] = transformer
end

-- Preprocessor that expands macros
function Macro.preprocess(source)
    -- Simple macro expansion: @macro_name(args)
    return source:gsub("@(%w+)%((.-)%)", function(name, args)
        local transformer = macro_registry[name]
        if transformer then
            return transformer(args)
        end
        return "@" .. name .. "(" .. args .. ")"  -- leave unchanged
    end)
end

-- Define useful macros
Macro.define("log", function(args)
    return string.format('print("[LOG]", %s)', args)
end)

Macro.define("assert_type", function(args)
    local var, typ = args:match("(%w+),%s*(%w+)")
    if var and typ then
        return string.format(
            'assert(type(%s) == "%s", "%s must be %s, got " .. type(%s))',
            var, typ, var, typ, var)
    end
    return args
end)

Macro.define("time", function(args)
    return string.format([[
do
    local _t_start = os.clock()
    local _t_result = {%s}
    print(string.format("Time: %%.3f ms", (os.clock() - _t_start) * 1000))
    return table.unpack(_t_result)
end]], args)
end)

Macro.define("swap", function(args)
    local a, b = args:match("(%w+),%s*(%w+)")
    if a and b then
        return string.format("%s, %s = %s, %s", a, b, b, a)
    end
    return args
end)

-- Test macros
local source = [[
local function process(x, items)
    @assert_type(x, number)
    @assert_type(items, table)
    
    @log("Processing " .. x .. " items")
    
    local a = 5
    local b = 10
    @swap(a, b)
    @log("After swap: a=" .. a .. " b=" .. b)
    
    local sum = 0
    for _, v in ipairs(items) do
        sum = sum + v
    end
    
    return sum
end

print(process(5, {1, 2, 3, 4, 5}))
]]

local expanded = Macro.preprocess(source)
print("Expanded code:")
print(expanded)
print("\nRunning:")
load(expanded)()
```

---

## 95.7 Domain-Specific Code Generator

```lua
-- State machine code generator
local StateMachineGen = {}
StateMachineGen.__index = StateMachineGen

function StateMachineGen.new()
    return setmetatable({
        name = "StateMachine",
        states = {},
        transitions = {},
        initial = nil
    }, StateMachineGen)
end

function StateMachineGen:state(name, config)
    self.states[name] = config or {}
    return self
end

function StateMachineGen:transition(from, event, to, action)
    table.insert(self.transitions, {
        from = from, event = event, to = to, action = action
    })
    return self
end

function StateMachineGen:initial_state(name)
    self.initial = name
    return self
end

function StateMachineGen:class_name(name)
    self.name = name
    return self
end

function StateMachineGen:generate()
    local lines = {}
    local function emit(line) table.insert(lines, line or "") end
    
    emit("-- Generated State Machine: " .. self.name)
    emit("local " .. self.name .. " = {}")
    emit(self.name .. ".__index = " .. self.name)
    emit("")
    emit("-- States")
    for state_name in pairs(self.states) do
        emit(self.name .. "." .. state_name:upper() .. ' = "' .. state_name .. '"')
    end
    emit("")
    emit("function " .. self.name .. ".new()")
    emit("    return setmetatable({")
    emit('        state = "' .. (self.initial or "initial") .. '",')
    emit("        history = {},")
    emit("        listeners = {}")
    emit("    }, " .. self.name .. ")")
    emit("end")
    emit("")
    emit("function " .. self.name .. ":on(event, handler)")
    emit("    self.listeners[event] = self.listeners[event] or {}")
    emit("    table.insert(self.listeners[event], handler)")
    emit("end")
    emit("")
    emit("function " .. self.name .. ":fire(event, ...)")
    emit("    local handlers = self.listeners[event]")
    emit("    if handlers then")
    emit("        for _, h in ipairs(handlers) do h(self, ...) end")
    emit("    end")
    emit("end")
    emit("")
    emit("function " .. self.name .. ":send(event, data)")
    
    -- Generate transition table
    emit("    local transitions = {")
    for _, t in ipairs(self.transitions) do
        local action = t.action or "nil"
        emit(string.format("        ['%s:%s'] = { to='%s', action=%s },",
            t.from, t.event, t.to, action))
    end
    emit("    }")
    emit("")
    emit("    local key = self.state .. ':' .. event")
    emit("    local transition = transitions[key]")
    emit("    if not transition then")
    emit("        return false, 'No transition from ' .. self.state .. ' on ' .. event")
    emit("    end")
    emit("")
    emit("    local old_state = self.state")
    emit("    table.insert(self.history, { state=old_state, event=event })")
    emit("")
    emit("    if transition.action then")
    emit("        transition.action(self, data)")
    emit("    end")
    emit("")
    emit("    self.state = transition.to")
    emit("    self:fire('state_changed', old_state, self.state, event)")
    emit("    return true")
    emit("end")
    emit("")
    emit("function " .. self.name .. ":is(state)")
    emit("    return self.state == state")
    emit("end")
    emit("")
    emit("return " .. self.name)
    
    return table.concat(lines, "\n")
end

-- Define a traffic light state machine
local TrafficLight = StateMachineGen.new()
TrafficLight:class_name("TrafficLight")
    :state("red")
    :state("yellow")
    :state("green")
    :initial_state("red")
    :transition("red",    "go",    "green",  function(sm) print("  🟢 Green light!") end)
    :transition("green",  "slow",  "yellow", function(sm) print("  🟡 Yellow light!") end)
    :transition("yellow", "stop",  "red",    function(sm) print("  🔴 Red light!") end)

local code = TrafficLight:generate()
print("Generated Traffic Light State Machine:")
print(code)

-- Load and run
local SM = load(code)()

local light = SM.new()
light:on("state_changed", function(sm, from, to, event)
    print(string.format("  State: %s -> %s (event: %s)", from, to, event))
end)

print("\nRunning traffic light simulation:")
light:send("go")
light:send("slow")
light:send("stop")
print("Current state:", light.state)
```

---

## แบบฝึกหัด

1. **API Client Generator**: สร้าง generator ที่อ่าน OpenAPI spec และ generate Lua client
2. **Test Generator**: Generate test cases จาก function signature
3. **Migration Generator**: Generate database migration files
4. **Validator Generator**: สร้าง validator functions จาก JSON Schema
5. **Documentation Generator**: Extract comments และ annotations เป็น HTML docs

---

*ต่อไป: [Part 96 - Contributing to Lua Open Source](part-96.md)*
