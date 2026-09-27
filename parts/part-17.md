# บทที่ 17: Metatables พื้นฐาน

## บทนำ

**Metatable** คือ table พิเศษที่กำหนดพฤติกรรมของ table อื่น เมื่อ Lua ต้องการดำเนินการบางอย่างกับ table (เช่น บวก, เปรียบเทียบ, เรียกใช้เป็นฟังก์ชัน) Lua จะตรวจสอบ metatable ก่อนว่ามีการกำหนดพฤติกรรมพิเศษไว้หรือไม่

Metamethods (ฟังก์ชันใน metatable ที่ขึ้นต้นด้วย `__`) ทำให้ table ทำงานเหมือน objects ที่มีพฤติกรรม custom ได้อย่างเต็มที่

---

## 17.1 Metatable คืออะไร

```lua
-- ตัวอย่างที่ 1: ทำความเข้าใจ metatable
-- โดยปกติ table ไม่สามารถบวกกันได้
local a = {1, 2, 3}
local b = {4, 5, 6}
-- print(a + b)  -- ERROR: attempt to perform arithmetic on a table value

-- แต่ถ้าเราใส่ metatable ที่นิยาม __add:
local mt = {
    __add = function(t1, t2)
        local result = {}
        for i = 1, math.max(#t1, #t2) do
            result[i] = (t1[i] or 0) + (t2[i] or 0)
        end
        return result
    end
}

setmetatable(a, mt)
setmetatable(b, mt)

local c = a + b  -- ตอนนี้ใช้ได้แล้ว!
for _, v in ipairs(c) do io.write(v .. " ") end
print()  -- 5 7 9
```

```lua
-- ตัวอย่างที่ 2: setmetatable และ getmetatable
local t = {}
local mt = {__type = "custom_table"}

setmetatable(t, mt)

local retrieved_mt = getmetatable(t)
print(retrieved_mt == mt)      -- true
print(retrieved_mt.__type)     -- custom_table
```

```lua
-- ตัวอย่างที่ 3: ป้องกันการเปลี่ยน metatable
local protected = {}
local secret_mt = {
    __metatable = "protected",  -- ค่านี้จะถูกคืนจาก getmetatable แทน mt จริง
    __index = protected
}

setmetatable(protected, secret_mt)

-- getmetatable คืน "protected" ไม่ใช่ secret_mt
print(getmetatable(protected))  -- protected

-- setmetatable จะ error เพราะมี __metatable
local ok, err = pcall(setmetatable, protected, {})
print(ok, err)  -- false  cannot change a protected metatable
```

---

## 17.2 __index สำหรับ Default Values

`__index` ถูกเรียกเมื่อ Lua ไม่พบ key ใน table

```lua
-- ตัวอย่างที่ 4: __index เป็น function
local defaults = {color="black", size=12, bold=false}

local text_style = setmetatable({}, {
    __index = function(t, k)
        return defaults[k]
    end
})

-- ไม่มีค่าใน text_style เลย แต่ fallback ไปที่ defaults
print(text_style.color)  -- black  (จาก defaults)
print(text_style.size)   -- 12     (จาก defaults)

-- ตั้งค่าใน text_style จะ override defaults
text_style.color = "red"
print(text_style.color)  -- red    (จาก text_style โดยตรง)
```

```lua
-- ตัวอย่างที่ 5: __index ที่ log การเข้าถึง
local function make_logged_table(name, data)
    return setmetatable({}, {
        __index = function(t, k)
            local v = data[k]
            print(string.format("[%s] accessed key '%s' = %s", name, k, tostring(v)))
            return v
        end
    })
end

local config = make_logged_table("config", {
    host = "localhost",
    port = 8080,
    debug = true
})

print(config.host)   -- [config] accessed key 'host' = localhost / localhost
print(config.port)   -- [config] accessed key 'port' = 8080 / 8080
print(config.other)  -- [config] accessed key 'other' = nil / nil
```

```lua
-- ตัวอย่างที่ 6: default value table
function default_table(default_value)
    return setmetatable({}, {
        __index = function(t, k)
            return default_value
        end
    })
end

local scores = default_table(0)
scores.alice = 95
scores.bob = 87

print(scores.alice)    -- 95   (ค่าที่ตั้งไว้)
print(scores.bob)      -- 87   (ค่าที่ตั้งไว้)
print(scores.charlie)  -- 0    (default)
print(scores.dave)     -- 0    (default)
```

---

## 17.3 __index เป็น Table (Prototype Chain)

```lua
-- ตัวอย่างที่ 7: __index เป็น table สำหรับ prototype-based inheritance
local Animal = {}
Animal.__index = Animal

function Animal.new(name, sound)
    local self = setmetatable({}, Animal)
    self.name = name
    self.sound = sound
    self.energy = 100
    return self
end

function Animal:speak()
    return self.name .. " says: " .. self.sound
end

function Animal:eat(food)
    self.energy = math.min(100, self.energy + 10)
    return self.name .. " eats " .. food
end

function Animal:status()
    return string.format("%s (energy: %d)", self.name, self.energy)
end

-- ใช้งาน
local cat = Animal.new("Whiskers", "Meow")
print(cat:speak())    -- Whiskers says: Meow
print(cat:eat("fish"))  -- Whiskers eats fish
print(cat:status())   -- Whiskers (energy: 100)
```

```lua
-- ตัวอย่างที่ 8: inheritance chain
local Dog = setmetatable({}, {__index = Animal})
Dog.__index = Dog

function Dog.new(name)
    local self = Animal.new(name, "Woof")
    return setmetatable(self, Dog)
end

-- Override method
function Dog:speak()
    return Animal.speak(self) .. "!"  -- เรียก parent method
end

-- Add new method
function Dog:fetch(item)
    self.energy = math.max(0, self.energy - 20)
    return self.name .. " fetches " .. item .. "! (energy: " .. self.energy .. ")"
end

local dog = Dog.new("Rex")
print(dog:speak())          -- Rex says: Woof!
print(dog:fetch("ball"))    -- Rex fetches ball! (energy: 80)
print(dog:eat("bone"))      -- Rex eats bone  (inherited from Animal)
print(dog:status())         -- Rex (energy: 90)  (inherited from Animal)

-- type checking
print(getmetatable(dog) == Dog)     -- true
print(getmetatable(dog) == Animal)  -- false
```

```lua
-- ตัวอย่างที่ 9: multi-level inheritance
local Puppy = setmetatable({}, {__index = Dog})
Puppy.__index = Puppy

function Puppy.new(name)
    local self = Dog.new(name)
    self.age = "puppy"
    return setmetatable(self, Puppy)
end

function Puppy:speak()
    return Dog.speak(self) .. " (tiny voice)"
end

local puppy = Puppy.new("Spot")
print(puppy:speak())        -- Spot says: Woof!! (tiny voice)
print(puppy:fetch("toy"))   -- Spot fetches toy! (energy: 80)  (from Dog)
print(puppy:eat("milk"))    -- Spot eats milk  (from Animal)
```

---

## 17.4 __newindex เพื่อ Intercept Assignment

```lua
-- ตัวอย่างที่ 10: __newindex พื้นฐาน
local logged_writes = {}
local mt = {
    __newindex = function(t, k, v)
        logged_writes[#logged_writes+1] = {key=k, value=v}
        rawset(t, k, v)  -- ใช้ rawset เพื่อเซ็ตค่าจริงๆ
    end
}

local t = setmetatable({}, mt)
t.x = 10
t.y = 20
t.name = "test"

print("logged writes:")
for _, w in ipairs(logged_writes) do
    print(string.format("  %s = %s", w.key, tostring(w.value)))
end
-- x = 10
-- y = 20
-- name = test
```

```lua
-- ตัวอย่างที่ 11: __newindex สำหรับ validation
local function make_validated_table(validators)
    return setmetatable({}, {
        __newindex = function(t, k, v)
            local validator = validators[k]
            if validator then
                local ok, err = validator(v)
                if not ok then
                    error(string.format("invalid value for '%s': %s", k, err), 2)
                end
            end
            rawset(t, k, v)
        end
    })
end

local config = make_validated_table({
    port = function(v)
        if type(v) ~= "number" then return false, "must be number" end
        if v < 1 or v > 65535 then return false, "must be 1-65535" end
        return true
    end,
    host = function(v)
        if type(v) ~= "string" then return false, "must be string" end
        if #v == 0 then return false, "cannot be empty" end
        return true
    end
})

config.host = "localhost"  -- OK
config.port = 8080         -- OK
print(config.host, config.port)  -- localhost  8080

local ok, err = pcall(function() config.port = 99999 end)
print(ok, err)  -- false  invalid value for 'port': must be 1-65535

local ok2, err2 = pcall(function() config.host = "" end)
print(ok2, err2)  -- false  invalid value for 'host': cannot be empty
```

---

## 17.5 __newindex สำหรับ Read-Only Tables

```lua
-- ตัวอย่างที่ 12: read-only table อย่างง่าย
function readonly(t)
    return setmetatable({}, {
        __index = t,
        __newindex = function(_, k, v)
            error("attempt to update a read-only table", 2)
        end,
        __metatable = false  -- ป้องกันการดู metatable
    })
end

local CONSTANTS = readonly({
    PI = math.pi,
    E = math.exp(1),
    MAX_INT = math.maxinteger,
    APP_NAME = "MyApp"
})

print(CONSTANTS.PI)       -- 3.1415926535898
print(CONSTANTS.APP_NAME) -- MyApp

local ok, err = pcall(function()
    CONSTANTS.PI = 3  -- พยายามแก้ไข!
end)
print(ok, err)  -- false  attempt to update a read-only table
```

```lua
-- ตัวอย่างที่ 13: deep read-only (recursive)
function deep_readonly(t, visited)
    visited = visited or {}
    if visited[t] then return visited[t] end

    local proxy = setmetatable({}, {
        __index = function(_, k)
            local v = t[k]
            if type(v) == "table" then
                return deep_readonly(v, visited)
            end
            return v
        end,
        __newindex = function()
            error("attempt to update a read-only table", 2)
        end,
        __len = function() return #t end,
        __pairs = function()
            return next, t, nil
        end
    })

    visited[t] = proxy
    return proxy
end

local data = deep_readonly({
    name = "config",
    server = {
        host = "localhost",
        port = 8080
    }
})

print(data.name)          -- config
print(data.server.host)   -- localhost
print(data.server.port)   -- 8080

local ok, err = pcall(function() data.server.port = 9999 end)
print(ok, err)  -- false  attempt to update a read-only table
```

---

## 17.6 __tostring

```lua
-- ตัวอย่างที่ 14: __tostring พื้นฐาน
local Vector = {}
Vector.__index = Vector

function Vector.new(x, y)
    return setmetatable({x=x, y=y}, Vector)
end

Vector.__tostring = function(v)
    return string.format("Vector(%g, %g)", v.x, v.y)
end

local v = Vector.new(3, 4)
print(v)           -- Vector(3, 4)
print(tostring(v)) -- Vector(3, 4)

-- ใน string concatenation ต้องใช้ tostring() ก่อน
print("My vector: " .. tostring(v))  -- My vector: Vector(3, 4)
```

```lua
-- ตัวอย่างที่ 15: __tostring สำหรับ object ซับซ้อน
local Person = {}
Person.__index = Person

function Person.new(name, age, job)
    return setmetatable({
        name = name,
        age = age,
        job = job or "unknown"
    }, Person)
end

Person.__tostring = function(p)
    return string.format("Person{name=%q, age=%d, job=%q}",
        p.name, p.age, p.job)
end

local p = Person.new("Alice", 30, "Engineer")
print(p)  -- Person{name="Alice", age=30, job="Engineer"}

-- Useful สำหรับ debugging
local people = {
    Person.new("Bob", 25, "Designer"),
    Person.new("Carol", 35)
}

for i, person in ipairs(people) do
    print(i, tostring(person))
end
```

```lua
-- ตัวอย่างที่ 16: __tostring สำหรับ collection
local Set = {}
Set.__index = Set

function Set.new(items)
    local self = setmetatable({_data = {}}, Set)
    if items then
        for _, v in ipairs(items) do
            self._data[v] = true
        end
    end
    return self
end

function Set:add(v) self._data[v] = true end
function Set:remove(v) self._data[v] = nil end
function Set:contains(v) return self._data[v] == true end
function Set:size()
    local n = 0
    for _ in pairs(self._data) do n = n + 1 end
    return n
end

Set.__tostring = function(s)
    local items = {}
    for k in pairs(s._data) do
        items[#items+1] = tostring(k)
    end
    table.sort(items)
    return "Set{" .. table.concat(items, ", ") .. "}"
end

local s = Set.new({3, 1, 4, 1, 5, 9, 2, 6})
print(s)  -- Set{1, 2, 3, 4, 5, 6, 9}  (no duplicates, sorted)
s:add(10)
print(s)  -- Set{1, 2, 3, 4, 5, 6, 9, 10}
```

---

## 17.7 rawget() และ rawset()

```lua
-- ตัวอย่างที่ 17: rawget bypasses __index
local mt = {
    __index = function(t, k)
        return "default"
    end
}

local t = setmetatable({real_key = "real_value"}, mt)

print(t.real_key)           -- real_value  (จาก table โดยตรง)
print(t.missing_key)        -- default     (จาก __index)

print(rawget(t, "real_key"))    -- real_value  (bypass __index)
print(rawget(t, "missing_key")) -- nil          (nil, ไม่เรียก __index)
```

```lua
-- ตัวอย่างที่ 18: rawset bypasses __newindex
local readonly_mt = {
    __newindex = function(t, k, v)
        error("read-only!", 2)
    end
}

local t = setmetatable({}, readonly_mt)

-- t.x = 10  -- ERROR!

rawset(t, "x", 10)  -- OK, bypass __newindex
print(t.x)  -- 10

-- แต่ครั้งถัดไปถ้าพยายามแก้ค่าที่มีอยู่แล้ว จะไม่ error
-- เพราะ __newindex ถูกเรียกแค่ตอน key ไม่มีอยู่!
-- t.x = 20  -- ไม่ error! เพราะ x มีอยู่แล้วใน table
-- rawset ใช้สำหรับตั้งค่าในขณะอยู่ใน __newindex
```

```lua
-- ตัวอย่างที่ 19: ใช้ rawget/rawset ใน metamethods อย่างถูกต้อง
local tracked = {}
local track_mt = {
    __index = function(t, k)
        local val = rawget(t, "_data_" .. k)  -- ป้องกัน infinite loop
        if val ~= nil then return val end
        return nil
    end,
    __newindex = function(t, k, v)
        -- บันทึก history
        local hist_key = "_hist_" .. k
        local hist = rawget(t, hist_key) or {}
        hist[#hist+1] = {value=v, time=os.time()}
        rawset(t, hist_key, hist)
        rawset(t, "_data_" .. k, v)  -- เซ็ตค่าจริง
    end
}

local obj = setmetatable({}, track_mt)
obj.name = "Alice"
obj.name = "Bob"
obj.name = "Carol"

print(obj.name)  -- Carol

-- ดู history
local hist = rawget(obj, "_hist_name")
print("name history:")
for _, h in ipairs(hist) do
    print("  " .. h.value)
end
-- Alice
-- Bob
-- Carol
```

---

## 17.8 __len สำหรับ Custom # Operator

```lua
-- ตัวอย่างที่ 20: __len สำหรับ custom length
local Map = {}
Map.__index = Map

function Map.new()
    return setmetatable({_store = {}, _count = 0}, Map)
end

function Map:set(k, v)
    if self._store[k] == nil then
        self._count = self._count + 1
    end
    self._store[k] = v
end

function Map:get(k)
    return self._store[k]
end

function Map:delete(k)
    if self._store[k] ~= nil then
        self._store[k] = nil
        self._count = self._count - 1
    end
end

Map.__len = function(self)
    return self._count
end

local m = Map.new()
m:set("a", 1)
m:set("b", 2)
m:set("c", 3)
print(#m)  -- 3

m:delete("b")
print(#m)  -- 2
```

```lua
-- ตัวอย่างที่ 21: __len สำหรับ string-keyed table
local SparseArray = {}
SparseArray.__index = SparseArray

function SparseArray.new()
    return setmetatable({_data={}, _max_idx=0}, SparseArray)
end

function SparseArray:set(i, v)
    self._data[i] = v
    if i > self._max_idx then
        self._max_idx = i
    end
end

function SparseArray:get(i)
    return self._data[i]
end

SparseArray.__len = function(self)
    return self._max_idx  -- คืน highest index
end

local sa = SparseArray.new()
sa:set(1, "first")
sa:set(5, "fifth")
sa:set(100, "hundredth")

print(#sa)          -- 100  (highest index)
print(sa:get(1))    -- first
print(sa:get(5))    -- fifth
print(sa:get(100))  -- hundredth
print(sa:get(50))   -- nil  (sparse!)
```

---

## 17.9 __call เพื่อให้ Table เรียกได้เหมือนฟังก์ชัน

```lua
-- ตัวอย่างที่ 22: table callable
local Multiplier = setmetatable({}, {
    __call = function(self, x)
        return x * (self.factor or 1)
    end
})

Multiplier.factor = 5
print(Multiplier(10))  -- 50
print(Multiplier(3))   -- 15
```

```lua
-- ตัวอย่างที่ 23: callable object factory
local function make_callable(fn, state)
    local obj = {state = state or {}}
    setmetatable(obj, {
        __call = function(self, ...)
            return fn(self.state, ...)
        end
    })
    return obj
end

-- Counter ที่เรียกได้โดยตรง
local counter = make_callable(function(state)
    state.n = (state.n or 0) + 1
    return state.n
end)

print(counter())  -- 1
print(counter())  -- 2
print(counter())  -- 3
print("current:", counter.state.n)  -- current: 3
```

```lua
-- ตัวอย่างที่ 24: function-like class
local Regex = {}
Regex.__index = Regex

function Regex.new(pattern, flags)
    return setmetatable({
        pattern = pattern,
        flags = flags or ""
    }, Regex)
end

function Regex:match(str)
    return str:match(self.pattern)
end

-- __call ทำให้ใช้เหมือนเรียกฟังก์ชัน
Regex.__call = function(self, str)
    return self:match(str)
end

Regex.__tostring = function(self)
    return "Regex(" .. self.pattern .. ")"
end

local email_pattern = Regex.new("[%a%d%.]+@[%a%d%.]+%.[%a]+")
local phone_pattern = Regex.new("%d%d%d%-%d%d%d%-%d%d%d%d")

-- ใช้แบบ method
print(email_pattern:match("user@example.com"))  -- user@example.com

-- ใช้แบบ callable
print(email_pattern("invalid"))  -- nil
print(phone_pattern("123-456-7890"))  -- 123-456-7890
```

```lua
-- ตัวอย่างที่ 25: functor (object ที่ทำงานเหมือน function)
local Formatter = {}
Formatter.__index = Formatter

function Formatter.new(template)
    return setmetatable({template = template}, Formatter)
end

Formatter.__call = function(self, data)
    return (self.template:gsub("{(%w+)}", function(key)
        return tostring(data[key] or "")
    end))
end

local greet = Formatter.new("Hello, {name}! You have {count} messages.")
print(greet({name="Alice", count=5}))
-- Hello, Alice! You have 5 messages.

local report = Formatter.new("[{level}] {timestamp}: {message}")
print(report({level="ERROR", timestamp="10:30:00", message="Connection failed"}))
-- [ERROR] 10:30:00: Connection failed
```

---

## 17.10 __concat สำหรับ Custom .. Operator

```lua
-- ตัวอย่างที่ 26: __concat สำหรับ string builder
local StringBuilder = {}
StringBuilder.__index = StringBuilder

function StringBuilder.new(s)
    return setmetatable({_parts = {s or ""}}, StringBuilder)
end

function StringBuilder:append(s)
    self._parts[#self._parts+1] = tostring(s)
    return self
end

function StringBuilder:toString()
    return table.concat(self._parts)
end

function StringBuilder:len()
    local total = 0
    for _, p in ipairs(self._parts) do total = total + #p end
    return total
end

StringBuilder.__tostring = function(self)
    return self:toString()
end

StringBuilder.__concat = function(a, b)
    local result = StringBuilder.new()
    -- a หรือ b อาจเป็น string ปกติ
    if type(a) == "table" then
        for _, p in ipairs(a._parts) do result:append(p) end
    else
        result:append(tostring(a))
    end
    if type(b) == "table" then
        for _, p in ipairs(b._parts) do result:append(p) end
    else
        result:append(tostring(b))
    end
    return result
end

local sb1 = StringBuilder.new("Hello")
local sb2 = StringBuilder.new(", World!")
local sb3 = sb1 .. sb2
print(tostring(sb3))   -- Hello, World!
print(tostring(sb3 .. " How are you?"))  -- Hello, World! How are you?
```

```lua
-- ตัวอย่างที่ 27: __concat สำหรับ list
local List = {}
List.__index = List

function List.new(items)
    local self = setmetatable({}, List)
    if items then
        for _, v in ipairs(items) do self[#self+1] = v end
    end
    return self
end

List.__concat = function(a, b)
    local result = List.new()
    if type(a) == "table" and getmetatable(a) == List then
        for _, v in ipairs(a) do result[#result+1] = v end
    end
    if type(b) == "table" and getmetatable(b) == List then
        for _, v in ipairs(b) do result[#result+1] = v end
    end
    return result
end

List.__tostring = function(self)
    local items = {}
    for _, v in ipairs(self) do items[#items+1] = tostring(v) end
    return "List[" .. table.concat(items, ", ") .. "]"
end

local l1 = List.new({1, 2, 3})
local l2 = List.new({4, 5, 6})
local l3 = l1 .. l2
print(tostring(l3))  -- List[1, 2, 3, 4, 5, 6]
```

---

## 17.11 Proxy Tables

```lua
-- ตัวอย่างที่ 28: proxy ที่ intercept ทุก operation
function make_proxy(target, hooks)
    hooks = hooks or {}
    return setmetatable({}, {
        __index = function(p, k)
            local v = target[k]
            if hooks.on_read then
                hooks.on_read(k, v)
            end
            return v
        end,

        __newindex = function(p, k, v)
            local old = target[k]
            if hooks.on_write then
                hooks.on_write(k, old, v)
            end
            target[k] = v
        end,

        __len = function(p)
            return #target
        end,

        __pairs = function(p)
            if hooks.on_iterate then hooks.on_iterate() end
            return next, target, nil
        end
    })
end

local data = {x=0, y=0, z=0}
local proxy = make_proxy(data, {
    on_read = function(k, v)
        print(string.format("  read: %s = %s", k, tostring(v)))
    end,
    on_write = function(k, old, new)
        print(string.format("  write: %s: %s -> %s",
            k, tostring(old), tostring(new)))
    end
})

print("Setting x:")
proxy.x = 10     --   write: x: 0 -> 10

print("Getting x:")
print(proxy.x)   --   read: x = 10 / 10

print("Setting y:")
proxy.y = 20     --   write: y: 0 -> 20
```

```lua
-- ตัวอย่างที่ 29: property system ด้วย proxy
function make_property_table()
    local values = {}
    local getters = {}
    local setters = {}

    local obj = setmetatable({}, {
        __index = function(t, k)
            if getters[k] then
                return getters[k]()
            end
            return values[k]
        end,

        __newindex = function(t, k, v)
            if setters[k] then
                setters[k](v)
            else
                values[k] = v
            end
        end
    })

    -- Helper สำหรับ define computed property
    function obj:define_property(name, getter, setter)
        if getter then getters[name] = getter end
        if setter then setters[name] = setter end
    end

    return obj
end

local person = make_property_table()
person.first_name = "Alice"
person.last_name = "Smith"

-- computed property
person:define_property("full_name",
    function() return person.first_name .. " " .. person.last_name end,
    function(v)
        local parts = v:match("(%S+) (%S+)")
        if parts then
            local f, l = v:match("(%S+) (%S+)")
            person.first_name = f
            person.last_name = l
        end
    end
)

print(person.full_name)  -- Alice Smith
person.full_name = "Bob Jones"
print(person.first_name)  -- Bob
print(person.last_name)   -- Jones
```

---

## 17.12 Default Value Tables

```lua
-- ตัวอย่างที่ 30: autovivification (auto-create nested tables)
function autovivify()
    return setmetatable({}, {
        __index = function(t, k)
            local subtable = autovivify()
            rawset(t, k, subtable)
            return subtable
        end
    })
end

-- ใช้ได้โดยไม่ต้อง create tables ด้วยมือ
local tree = autovivify()
tree.a.b.c = "deep value"
tree.x.y = 42

print(tree.a.b.c)       -- deep value
print(tree.x.y)          -- 42
print(type(tree.a))      -- table
print(type(tree.a.b))    -- table
print(tree.missing.key)  -- nil (auto-created but empty)
```

```lua
-- ตัวอย่างที่ 31: table ที่นับจำนวน access
function counting_table(data)
    local access_count = {}

    return setmetatable({}, {
        __index = function(t, k)
            access_count[k] = (access_count[k] or 0) + 1
            return data[k]
        end,
        __newindex = function(t, k, v)
            data[k] = v
        end,
        get_access_counts = function()
            return access_count
        end
    })
end

-- แต่ get_access_counts จะถูกเรียกจาก __index ด้วย...
-- ต้องใช้ rawget เพื่อ avoid

local data = {a=1, b=2, c=3}
local ct = counting_table(data)

ct.a  -- access a
ct.b  -- access b
ct.a  -- access a again
ct.a  -- access a again

-- ดู access counts ผ่าน data โดยตรง (workaround)
local counts = access_count  -- ถ้า access_count เป็น local ใน scope นี้
-- ในที่นี้ access_count เป็น upvalue ของ counting_table
```

---

## 17.13 Validation Tables

```lua
-- ตัวอย่างที่ 32: schema validation table
function schema_table(schema)
    local data = {}

    return setmetatable({}, {
        __newindex = function(t, k, v)
            local rule = schema[k]
            if rule then
                -- Type check
                if rule.type and type(v) ~= rule.type then
                    error(string.format(
                        "field '%s': expected %s, got %s",
                        k, rule.type, type(v)), 2)
                end
                -- Range check
                if rule.min and v < rule.min then
                    error(string.format(
                        "field '%s': value %s < min %s",
                        k, v, rule.min), 2)
                end
                if rule.max and v > rule.max then
                    error(string.format(
                        "field '%s': value %s > max %s",
                        k, v, rule.max), 2)
                end
                -- Custom validator
                if rule.validate then
                    local ok, err = rule.validate(v)
                    if not ok then
                        error(string.format("field '%s': %s", k, err), 2)
                    end
                end
            elseif not schema["*"] then
                -- Strict mode: ไม่อนุญาต field ที่ไม่ได้นิยาม
                -- (comment out ถ้าต้องการ lenient mode)
            end
            data[k] = v
        end,

        __index = function(t, k)
            return data[k]
        end
    })
end

local user = schema_table({
    name = {
        type = "string",
        validate = function(v)
            if #v < 2 then return false, "too short (min 2 chars)" end
            if #v > 50 then return false, "too long (max 50 chars)" end
            return true
        end
    },
    age = {
        type = "number",
        min = 0,
        max = 150
    },
    email = {
        type = "string",
        validate = function(v)
            if not v:match("[%a%d%.]+@[%a%d%.]+%.[%a]+") then
                return false, "invalid email format"
            end
            return true
        end
    }
})

user.name = "Alice"
user.age = 30
user.email = "alice@example.com"
print(user.name, user.age, user.email)  -- Alice  30  alice@example.com

local ok, err = pcall(function() user.age = -5 end)
print(ok, err)  -- false  field 'age': value -5 < min 0

local ok2, err2 = pcall(function() user.email = "invalid" end)
print(ok2, err2)  -- false  field 'email': invalid email format
```

---

## 17.14 Arithmetic Metamethods

```lua
-- ตัวอย่างที่ 33: Vector arithmetic ครบชุด
local Vector2D = {}
Vector2D.__index = Vector2D

function Vector2D.new(x, y)
    return setmetatable({x=x or 0, y=y or 0}, Vector2D)
end

Vector2D.__add = function(a, b)
    return Vector2D.new(a.x + b.x, a.y + b.y)
end

Vector2D.__sub = function(a, b)
    return Vector2D.new(a.x - b.x, a.y - b.y)
end

Vector2D.__mul = function(a, b)
    if type(a) == "number" then
        return Vector2D.new(a * b.x, a * b.y)
    elseif type(b) == "number" then
        return Vector2D.new(a.x * b, a.y * b)
    else
        -- dot product
        return a.x * b.x + a.y * b.y
    end
end

Vector2D.__div = function(a, b)
    if type(b) == "number" then
        return Vector2D.new(a.x / b, a.y / b)
    end
    error("can only divide vector by number")
end

Vector2D.__unm = function(a)  -- unary minus
    return Vector2D.new(-a.x, -a.y)
end

Vector2D.__eq = function(a, b)
    return a.x == b.x and a.y == b.y
end

Vector2D.__lt = function(a, b)
    return a:length() < b:length()
end

Vector2D.__le = function(a, b)
    return a:length() <= b:length()
end

Vector2D.__tostring = function(v)
    return string.format("(%g, %g)", v.x, v.y)
end

function Vector2D:length()
    return math.sqrt(self.x*self.x + self.y*self.y)
end

function Vector2D:normalize()
    local len = self:length()
    if len == 0 then return Vector2D.new(0, 0) end
    return self / len
end

function Vector2D:dot(other)
    return self * other
end

local v1 = Vector2D.new(3, 4)
local v2 = Vector2D.new(1, 2)

print(tostring(v1 + v2))    -- (4, 6)
print(tostring(v1 - v2))    -- (2, 2)
print(tostring(v1 * 2))     -- (6, 8)
print(tostring(v1 / 2))     -- (1.5, 2)
print(tostring(-v1))         -- (-3, -4)
print(v1 == Vector2D.new(3,4))  -- true
print("length:", v1:length())   -- length: 5.0
print("normalized:", tostring(v1:normalize()))  -- (0.6, 0.8)
print("dot product:", v1:dot(v2))  -- 11  (3*1 + 4*2)
```

---

## 17.15 Comparison Metamethods

```lua
-- ตัวอย่างที่ 34: custom comparison
local Version = {}
Version.__index = Version

function Version.new(major, minor, patch)
    return setmetatable({
        major = major or 0,
        minor = minor or 0,
        patch = patch or 0
    }, Version)
end

function Version.parse(s)
    local maj, min, pat = s:match("^(%d+)%.(%d+)%.(%d+)$")
    if not maj then
        maj, min = s:match("^(%d+)%.(%d+)$")
        pat = 0
    end
    if not maj then
        maj = s:match("^(%d+)$")
        min, pat = 0, 0
    end
    return Version.new(tonumber(maj), tonumber(min), tonumber(pat))
end

local function compare(a, b)
    if a.major ~= b.major then return a.major - b.major end
    if a.minor ~= b.minor then return a.minor - b.minor end
    return a.patch - b.patch
end

Version.__eq = function(a, b) return compare(a, b) == 0 end
Version.__lt = function(a, b) return compare(a, b) < 0 end
Version.__le = function(a, b) return compare(a, b) <= 0 end

Version.__tostring = function(v)
    return string.format("%d.%d.%d", v.major, v.minor, v.patch)
end

local v1 = Version.parse("1.2.3")
local v2 = Version.parse("1.10.0")
local v3 = Version.parse("2.0.0")

print(tostring(v1), tostring(v2), tostring(v3))
-- 1.2.3  1.10.0  2.0.0

print(v1 < v2)   -- true
print(v2 < v3)   -- true
print(v1 == v1)  -- true
print(v3 > v1)   -- true

-- Sort versions
local versions = {v3, v1, v2, Version.parse("1.2.3"), Version.parse("0.9.0")}
table.sort(versions)
for _, v in ipairs(versions) do io.write(tostring(v) .. " ") end
print()  -- 0.9.0 1.2.3 1.2.3 1.10.0 2.0.0
```

---

## 17.16 Complete Object System

```lua
-- ตัวอย่างที่ 35: class system ด้วย metatables
local Class = {}
Class.__index = Class

function Class:new(...)
    local instance = setmetatable({}, self)
    if instance.init then instance:init(...) end
    return instance
end

function Class:extend()
    local cls = {}
    cls.__index = cls
    setmetatable(cls, self)
    cls.super = self
    return cls
end

function Class:is_a(klass)
    local mt = getmetatable(self)
    while mt do
        if mt == klass then return true end
        mt = getmetatable(mt)
    end
    return false
end

-- Shape base class
local Shape = Class:extend()

function Shape:init(color)
    self.color = color or "black"
end

function Shape:area()
    return 0
end

function Shape:describe()
    return string.format("%s (color=%s, area=%.2f)",
        self:type_name(), self.color, self:area())
end

function Shape:type_name()
    return "Shape"
end

-- Circle
local Circle = Shape:extend()

function Circle:init(radius, color)
    Shape.init(self, color)
    self.radius = radius
end

function Circle:area()
    return math.pi * self.radius * self.radius
end

function Circle:type_name()
    return "Circle"
end

function Circle:circumference()
    return 2 * math.pi * self.radius
end

-- Rectangle
local Rectangle = Shape:extend()

function Rectangle:init(width, height, color)
    Shape.init(self, color)
    self.width = width
    self.height = height
end

function Rectangle:area()
    return self.width * self.height
end

function Rectangle:type_name()
    return "Rectangle"
end

function Rectangle:perimeter()
    return 2 * (self.width + self.height)
end

-- ใช้งาน
local shapes = {
    Circle:new(5, "red"),
    Rectangle:new(4, 6, "blue"),
    Circle:new(3),
    Rectangle:new(10, 2, "green")
}

for _, shape in ipairs(shapes) do
    print(shape:describe())
end
-- Circle (color=red, area=78.54)
-- Rectangle (color=blue, area=24.00)
-- Circle (color=black, area=28.27)
-- Rectangle (color=green, area=20.00)

-- Polymorphism
local total_area = 0
for _, shape in ipairs(shapes) do
    total_area = total_area + shape:area()
end
print(string.format("Total area: %.2f", total_area))
```

---

## 17.17 __pairs และ __ipairs (Lua 5.2+)

```lua
-- ตัวอย่างที่ 36: custom iteration
local FilteredTable = {}
FilteredTable.__index = FilteredTable

function FilteredTable.new(data, filter_fn)
    return setmetatable({
        _data = data,
        _filter = filter_fn
    }, FilteredTable)
end

-- Note: __pairs support varies by Lua version
-- ใน Lua 5.4, pairs() ตรวจ __pairs ก่อน
FilteredTable.__pairs = function(self)
    local data = self._data
    local filter = self._filter

    return function(t, k)
        local next_k, next_v = next(data, k)
        while next_k ~= nil and not filter(next_k, next_v) do
            next_k, next_v = next(data, next_k)
        end
        return next_k, next_v
    end, self, nil
end

local numbers = FilteredTable.new(
    {a=1, b=2, c=3, d=4, e=5, f=6},
    function(k, v) return v % 2 == 0 end  -- แค่เลขคู่
)

-- Note: ใน Lua 5.4 อาจต้องใช้วิธีอื่น
-- ใช้ method แทน
function FilteredTable:iter()
    local data = self._data
    local filter = self._filter
    local k = nil

    return function()
        local v
        repeat
            k, v = next(data, k)
        until k == nil or filter(k, v)
        return k, v
    end
end

for k, v in numbers:iter() do
    print(k, v)
end
-- b  2
-- d  4
-- f  6
```

---

## 17.18 Practical Examples

```lua
-- ตัวอย่างที่ 37: Observable table (reactive)
function make_reactive(data)
    local watchers = {}

    return setmetatable({}, {
        __index = data,
        __newindex = function(t, k, v)
            local old = data[k]
            data[k] = v
            if watchers[k] and old ~= v then
                for _, fn in ipairs(watchers[k]) do
                    fn(v, old)
                end
            end
        end,
        watch = function(key, fn)
            if not watchers[key] then watchers[key] = {} end
            watchers[key][#watchers[key]+1] = fn
        end
    })
end

local state = make_reactive({count=0, name="init"})
local mt = getmetatable(state)

-- Watch แบบ manual (เพราะ watch อยู่ใน metatable ไม่ได้ผ่าน __index)
local watchers_public = {}
mt.watch("count", function(new, old)
    print(string.format("count changed: %d -> %d", old, new))
end)

state.count = 1   -- count changed: 0 -> 1
state.count = 5   -- count changed: 1 -> 5
state.count = 5   -- ไม่มี notification (ค่าเดิม)
state.name = "updated"  -- ไม่มี notification (ไม่มี watcher)
```

```lua
-- ตัวอย่างที่ 38: mixin system
function mixin(target_class, ...)
    for _, source in ipairs({...}) do
        for k, v in pairs(source) do
            if not target_class[k] then  -- ไม่ override ที่มีอยู่แล้ว
                target_class[k] = v
            end
        end
    end
    return target_class
end

local Serializable = {
    serialize = function(self)
        local parts = {}
        for k, v in pairs(self) do
            if type(v) ~= "function" then
                parts[#parts+1] = k .. "=" .. tostring(v)
            end
        end
        table.sort(parts)
        return "{" .. table.concat(parts, ",") .. "}"
    end
}

local Comparable = {
    equals = function(self, other)
        for k, v in pairs(self) do
            if type(v) ~= "function" and other[k] ~= v then
                return false
            end
        end
        return true
    end
}

local Point = {}
Point.__index = Point

function Point.new(x, y)
    return setmetatable({x=x, y=y}, Point)
end

-- Add mixins
mixin(Point, Serializable, Comparable)

local p1 = Point.new(1, 2)
local p2 = Point.new(1, 2)
local p3 = Point.new(3, 4)

print(p1:serialize())    -- {x=1,y=2}
print(p1:equals(p2))     -- true
print(p1:equals(p3))     -- false
```

```lua
-- ตัวอย่างที่ 39: type-safe wrapper
function typed(value, type_name)
    return setmetatable({_value = value, _type = type_name}, {
        __index = function(t, k)
            if k == "value" then return t._value end
            if k == "type" then return t._type end
        end,
        __newindex = function(t, k, v)
            if k == "value" then
                if type(v) ~= t._type then
                    error(string.format("type error: expected %s, got %s",
                        t._type, type(v)), 2)
                end
                rawset(t, "_value", v)
            else
                rawset(t, k, v)
            end
        end,
        __tostring = function(t)
            return string.format("%s(%s)", t._type, tostring(t._value))
        end
    })
end

local n = typed(42, "number")
print(tostring(n))  -- number(42)
print(n.value)      -- 42

n.value = 100       -- OK
print(n.value)      -- 100

local ok, err = pcall(function()
    n.value = "not a number"
end)
print(ok, err)  -- false  type error: expected number, got string
```

```lua
-- ตัวอย่างที่ 40: Finite State Machine ด้วย metatables
local FSM = {}
FSM.__index = FSM

function FSM.new(initial_state, transitions)
    return setmetatable({
        state = initial_state,
        transitions = transitions,
        history = {initial_state}
    }, FSM)
end

function FSM:trigger(event)
    local from = self.state
    local trans = self.transitions[from]
    if not trans then
        return false, "no transitions from state: " .. tostring(from)
    end

    local to = trans[event]
    if not to then
        return false, string.format("no transition '%s' from '%s'", event, from)
    end

    -- Handle function transitions
    if type(to) == "function" then
        to = to(self, event)
    end

    self.state = to
    self.history[#self.history+1] = to
    return true, to
end

function FSM:is_in(state)
    return self.state == state
end

FSM.__tostring = function(self)
    return "FSM{state=" .. tostring(self.state) .. "}"
end

-- Traffic light FSM
local traffic_light = FSM.new("red", {
    red    = {timer = "green"},
    green  = {timer = "yellow"},
    yellow = {timer = "red"}
})

print(tostring(traffic_light))  -- FSM{state=red}

for i = 1, 6 do
    local ok, new_state = traffic_light:trigger("timer")
    print(string.format("trigger -> %s", new_state))
end
-- trigger -> green
-- trigger -> yellow
-- trigger -> red
-- trigger -> green
-- trigger -> yellow
-- trigger -> red
```

---

## แบบฝึกหัด (Exercises)

### ระดับพื้นฐาน

**แบบฝึกหัดที่ 1:** สร้าง Vector3D class ที่มี arithmetic operations (+, -, scalar multiplication) และ __tostring

```lua
-- Template:
local Vector3D = {}
Vector3D.__index = Vector3D

function Vector3D.new(x, y, z)
    -- TODO
end

-- implement: __add, __sub, __mul, __tostring
-- และ methods: length(), dot(other), cross(other)

local v1 = Vector3D.new(1, 0, 0)
local v2 = Vector3D.new(0, 1, 0)
print(tostring(v1 + v2))     -- (1, 1, 0)
print(v1:dot(v2))            -- 0 (perpendicular)
print(tostring(v1:cross(v2))) -- (0, 0, 1)
```

**แบบฝึกหัดที่ 2:** สร้าง read-only table จาก table ปกติ โดยใช้ __index และ __newindex

**แบบฝึกหัดที่ 3:** สร้าง "default dictionary" ที่คืน 0 สำหรับ key ที่ไม่มี (เหมือน Python's defaultdict)

### ระดับกลาง

**แบบฝึกหัดที่ 4:** สร้าง Stack class ที่ใช้ __len, __tostring และ methods push/pop/peek

```lua
-- Template:
local Stack = {}
Stack.__index = Stack

-- Stack:push(v), Stack:pop(), Stack:peek()
-- #stack -> จำนวน elements
-- tostring(stack) -> "Stack[1, 2, 3]"
```

**แบบฝึกหัดที่ 5:** สร้าง Matrix 2x2 class พร้อม +, -, * (matrix multiplication) operations

**แบบฝึกหัดที่ 6:** สร้าง `make_enum(values)` ที่คืน read-only table และ error เมื่อใช้ค่าที่ไม่ valid

```lua
local Direction = make_enum({"NORTH", "SOUTH", "EAST", "WEST"})
print(Direction.NORTH)   -- "NORTH"
print(Direction.EAST)    -- "EAST"
-- Direction.UP  -- ERROR: invalid enum value
```

### ระดับยาก

**แบบฝึกหัดที่ 7:** implement Fraction class (เศษส่วน) ที่ทำ arithmetic ได้และลดรูปอัตโนมัติ

```lua
-- Fraction(3, 4) + Fraction(1, 4) == Fraction(1, 1) (= 1)
-- Fraction(1, 2) * Fraction(2, 3) == Fraction(1, 3)
-- tostring(Fraction(3, 6)) == "1/2"
```

**แบบฝึกหัดที่ 8:** สร้าง Observable class ที่ใช้ __newindex เพื่อ notify watchers เมื่อ property เปลี่ยน

**แบบฝึกหัดที่ 9:** implement Deep Proxy ที่ track ทุก read/write operation รวมถึง nested tables

**แบบฝึกหัดที่ 10:** สร้าง class system ที่รองรับ multiple inheritance โดยใช้ metatables

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Metatable คืออะไร** - table ที่กำหนดพฤติกรรม table อื่น
2. **setmetatable/getmetatable** - กำหนดและดึง metatable
3. **__index** - กำหนดพฤติกรรมเมื่อ key ไม่พบ (function หรือ table)
4. **Prototype Chain** - inheritance ผ่าน __index chain
5. **__newindex** - intercept การ assignment
6. **Read-Only Tables** - ป้องกันการแก้ไขด้วย __newindex
7. **__tostring** - custom string representation
8. **rawget/rawset** - bypass metamethods
9. **__len** - custom # operator
10. **__call** - ทำให้ table เรียกได้เหมือน function
11. **__concat** - custom .. operator
12. **Proxy Tables** - intercept ทุก operations
13. **Arithmetic Metamethods** - +, -, *, /, unary minus
14. **Comparison Metamethods** - ==, <, <=
15. **__metatable** - ป้องกันการเปลี่ยน metatable

Metatables เป็น mechanism หลักที่ทำให้ Lua รองรับ OOP, operator overloading, และ meta-programming ได้อย่างทรงพลัง แม้จะเป็นภาษาขนาดเล็กที่ออกแบบมาให้ embed ได้ง่าย
