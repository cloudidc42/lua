# บทที่ 23: OOP - Multiple Inheritance และ Mixins

## บทนำ

Multiple Inheritance คือความสามารถที่ class หนึ่งสามารถสืบทอดคุณสมบัติจาก class หลายๆ class พร้อมกัน Lua ไม่มี built-in support สำหรับ multiple inheritance แต่เราสามารถสร้างได้หลายวิธีผ่าน metatable

Mixins เป็นอีกแนวทางหนึ่งที่ composing behavior โดยไม่ต้องใช้ inheritance hierarchy ทำให้โค้ดยืดหยุ่นและนำกลับมาใช้ซ้ำได้ง่ายกว่า

ในบทนี้เราจะเรียนรู้:
- Multiple Inheritance ด้วย __index หลายชั้น
- Mixin Pattern ที่หลากหลาย
- Trait Composition
- การแก้ปัญหา method conflicts
- Real-world examples: สัตว์ที่บินได้ ว่ายน้ำได้ เดินได้
- GUI Widget Mixins
- Serializable, Observable, EventEmitter Mixins

---

## 23.1 Multiple Inheritance พื้นฐาน

### ตัวอย่างที่ 1: Multiple Inheritance ด้วย search()

```lua
-- วิธีมาตรฐาน: ใช้ function search() ใน __index
local function search(bases, key)
    for _, base in ipairs(bases) do
        local value = base[key]
        if value then return value end
    end
end

-- สร้าง class ที่รองรับ multiple inheritance
local function createMultiClass(...)
    local bases = {...}
    local cls = {}
    cls.__index = cls
    cls._bases = bases
    
    -- ตั้ง metatable สำหรับ class เพื่อ search ใน bases
    setmetatable(cls, {
        __index = function(t, k)
            return search(bases, k)
        end
    })
    
    return cls
end

-- Base classes
local Flyable = {}
function Flyable:fly()
    print(self.name .. " กำลังบิน!")
end
function Flyable:land()
    print(self.name .. " ลงจอด")
end
function Flyable:getAltitude()
    return self.altitude or 0
end

local Swimmable = {}
function Swimmable:swim()
    print(self.name .. " กำลังว่ายน้ำ!")
end
function Swimmable:dive(depth)
    print(self.name .. " ดำน้ำลึก " .. depth .. " เมตร")
end
function Swimmable:surface()
    print(self.name .. " ขึ้นมาบนผิวน้ำ")
end

-- Duck สืบทอดจากทั้ง Flyable และ Swimmable
local Duck = createMultiClass(Flyable, Swimmable)
Duck.__index = Duck

function Duck.new(name)
    return setmetatable({
        name = name,
        altitude = 0,
        underwater = false
    }, Duck)
end

function Duck:quack()
    print(self.name .. " ร้อง: แกร๊กๆ!")
end

-- ทดสอบ
local donald = Duck.new("โดนัลด์")
donald:quack()
donald:fly()    -- จาก Flyable
donald:swim()   -- จาก Swimmable
donald:dive(5)  -- จาก Swimmable
donald:land()   -- จาก Flyable
```

### ตัวอย่างที่ 2: Multiple Inheritance แบบ Priority (ลำดับสำคัญ)

```lua
-- กำหนดลำดับ priority สำหรับ method resolution
local function createClassWithPriority(...)
    local bases = {...}
    local cls = {}
    cls.__index = cls
    cls._bases = bases
    
    setmetatable(cls, {
        __index = function(t, k)
            -- ค้นหาตาม priority (ลำดับที่ส่งเข้ามา)
            for _, base in ipairs(bases) do
                local v = rawget(base, k)
                if v ~= nil then return v end
            end
            -- ค้นหาใน grandparents
            for _, base in ipairs(bases) do
                local meta = getmetatable(base)
                if meta and meta.__index then
                    local v = meta.__index(base, k)
                    if v ~= nil then return v end
                end
            end
        end
    })
    
    return cls
end

local A = {}
function A:method() print("A:method") end
function A:shared() print("A:shared") end

local B = {}
function B:method() print("B:method") end
function B:bOnly() print("B:bOnly") end

local C = {}
function C:method() print("C:method") end
function C:cOnly() print("C:cOnly") end
function C:shared() print("C:shared") end

-- D สืบทอดจาก B และ C (B มี priority สูงกว่า)
local D = createClassWithPriority(B, C)
D.__index = D

function D.new()
    return setmetatable({}, D)
end

local d = D.new()
-- B มี priority สูงกว่า ดังนั้น B:method จะถูกเรียก
d:method()   -- B:method
d:bOnly()    -- B:bOnly
d:cOnly()    -- C:cOnly
d:shared()   -- A:shared จาก B (ถ้า B extends A)
```

### ตัวอย่างที่ 3: Copy-based Multiple Inheritance

```lua
-- Copy-based: คัดลอก methods จาก bases ทั้งหมดไปยัง class
local function mixin(target, source, override)
    for k, v in pairs(source) do
        if type(v) == "function" then
            if override or target[k] == nil then
                target[k] = v
            end
        end
    end
    return target
end

-- ใช้งาน
local Logger = {}
function Logger:log(msg)
    print(string.format("[%s] %s", os.date("%H:%M:%S"), msg))
end
function Logger:warn(msg)
    print(string.format("[WARN] %s", msg))
end
function Logger:error(msg)
    print(string.format("[ERROR] %s", msg))
end

local Validator = {}
function Validator:validate(value, rules)
    for _, rule in ipairs(rules) do
        if not rule.check(value) then
            self:log("Validation failed: " .. rule.message)
            return false, rule.message
        end
    end
    return true
end
function Validator:isRequired(val)
    return val ~= nil and val ~= ""
end
function Validator:isNumber(val)
    return type(val) == "number"
end
function Validator:isInRange(val, min, max)
    return type(val) == "number" and val >= min and val <= max
end

local Cacheable = {}
Cacheable._cache = {}
function Cacheable:cache(key, value)
    Cacheable._cache[key] = {value = value, time = os.time()}
end
function Cacheable:getCached(key, maxAge)
    local entry = Cacheable._cache[key]
    if entry and (not maxAge or os.time() - entry.time < maxAge) then
        return entry.value
    end
    return nil
end
function Cacheable:invalidate(key)
    Cacheable._cache[key] = nil
end

-- UserService ใช้ทั้ง 3 mixins
local UserService = {}
UserService.__index = UserService

mixin(UserService, Logger)
mixin(UserService, Validator)
mixin(UserService, Cacheable)

function UserService.new()
    return setmetatable({users = {}}, UserService)
end

function UserService:createUser(name, email, age)
    self:log("สร้าง user: " .. name)
    
    -- Validate
    if not self:isRequired(name) then
        self:error("ชื่อไม่สามารถว่างได้")
        return nil
    end
    if not self:isInRange(age, 0, 150) then
        self:error("อายุไม่ถูกต้อง: " .. tostring(age))
        return nil
    end
    
    local user = {name=name, email=email, age=age}
    table.insert(self.users, user)
    self:cache("user_" .. name, user)
    self:log("สร้าง user สำเร็จ")
    return user
end

function UserService:getUser(name)
    local cached = self:getCached("user_" .. name, 60)
    if cached then
        self:log("ได้ user จาก cache: " .. name)
        return cached
    end
    
    for _, user in ipairs(self.users) do
        if user.name == name then
            self:cache("user_" .. name, user)
            return user
        end
    end
    return nil
end

-- ทดสอบ
local service = UserService.new()
service:createUser("สมชาย", "somchai@test.com", 30)
service:createUser("สมหญิง", "somying@test.com", 25)
service:createUser("", "empty@test.com", 20)  -- error: ชื่อว่าง
service:createUser("ผิด", "wrong@test.com", 200)  -- error: อายุเกิน

local u = service:getUser("สมชาย")
if u then print("พบ user:", u.name, u.email) end

-- เรียกอีกครั้งจาก cache
local u2 = service:getUser("สมชาย")
```

---

## 23.2 Mixin Pattern ขั้นสูง

### ตัวอย่างที่ 4: Mixin Factory

```lua
-- สร้าง mixins ด้วย factory functions
local function createTimestampMixin()
    return {
        initTimestamp = function(self)
            self._createdAt = os.time()
            self._updatedAt = os.time()
        end,
        touch = function(self)
            self._updatedAt = os.time()
        end,
        getAge = function(self)
            return os.time() - (self._createdAt or os.time())
        end,
        getCreatedAt = function(self)
            return os.date("%Y-%m-%d %H:%M:%S", self._createdAt)
        end,
        getUpdatedAt = function(self)
            return os.date("%Y-%m-%d %H:%M:%S", self._updatedAt)
        end
    }
end

local function createVersionMixin()
    return {
        initVersion = function(self)
            self._version = 1
            self._history = {}
        end,
        bumpVersion = function(self, note)
            table.insert(self._history, {
                version = self._version,
                note = note or "no note",
                time = os.time()
            })
            self._version = self._version + 1
        end,
        getVersion = function(self)
            return self._version
        end,
        getHistory = function(self)
            return self._history
        end
    }
end

local function createSoftDeleteMixin()
    return {
        softDelete = function(self)
            self._deleted = true
            self._deletedAt = os.time()
        end,
        restore = function(self)
            self._deleted = false
            self._deletedAt = nil
        end,
        isDeleted = function(self)
            return self._deleted == true
        end
    }
end

-- Document class ที่ใช้ mixins
local Document = {}
Document.__index = Document

-- Apply mixins
local tsMixin = createTimestampMixin()
local verMixin = createVersionMixin()
local sdMixin = createSoftDeleteMixin()

for k, v in pairs(tsMixin) do Document[k] = v end
for k, v in pairs(verMixin) do Document[k] = v end
for k, v in pairs(sdMixin) do Document[k] = v end

function Document.new(title, content)
    local self = setmetatable({
        title = title,
        content = content
    }, Document)
    self:initTimestamp()
    self:initVersion()
    return self
end

function Document:update(newContent, note)
    self.content = newContent
    self:touch()
    self:bumpVersion(note)
end

function Document:describe()
    print(string.format(
        "Document: '%s'\n  Version: %d\n  Created: %s\n  Updated: %s\n  Deleted: %s",
        self.title,
        self:getVersion(),
        self:getCreatedAt(),
        self:getUpdatedAt(),
        tostring(self:isDeleted())
    ))
end

-- ทดสอบ
local doc = Document.new("Lua Guide", "เนื้อหาเดิม...")
doc:describe()

doc:update("เนื้อหาใหม่ version 2", "แก้ไขเนื้อหาหลัก")
doc:update("เนื้อหา version 3 ที่ดีกว่า", "ปรับปรุงตัวอย่าง")

print("\nหลังอัพเดท:")
doc:describe()

print("\nHistory:")
for _, h in ipairs(doc:getHistory()) do
    print(string.format("  v%d: %s", h.version, h.note))
end

doc:softDelete()
print("\nหลัง soft delete - isDeleted:", doc:isDeleted())
doc:restore()
print("หลัง restore - isDeleted:", doc:isDeleted())
```

### ตัวอย่างที่ 5: Trait Composition

```lua
-- Trait: คล้าย Mixin แต่มีการตรวจสอบ requirements
local function createTrait(name, required, methods)
    return {
        _traitName = name,
        _required = required or {},  -- methods ที่ต้องมีใน class
        _methods = methods
    }
end

local function applyTrait(cls, trait)
    -- ตรวจสอบว่า class มี required methods ครบ
    for _, req in ipairs(trait._required) do
        assert(cls[req] ~= nil, 
            string.format("Class ต้องมี method '%s' เพื่อใช้ trait '%s'",
                req, trait._traitName))
    end
    
    -- Copy methods
    for k, v in pairs(trait._methods) do
        if cls[k] == nil then  -- ไม่ override methods ที่มีอยู่แล้ว
            cls[k] = v
        end
    end
    
    -- Track applied traits
    cls._traits = cls._traits or {}
    table.insert(cls._traits, trait._traitName)
    
    return cls
end

function hasTrait(obj, traitName)
    local cls = getmetatable(obj)
    if cls and cls._traits then
        for _, t in ipairs(cls._traits) do
            if t == traitName then return true end
        end
    end
    return false
end

-- สร้าง Traits
local Comparable = createTrait("Comparable", {"getValue"}, {
    lessThan = function(self, other)
        return self:getValue() < other:getValue()
    end,
    greaterThan = function(self, other)
        return self:getValue() > other:getValue()
    end,
    equalTo = function(self, other)
        return self:getValue() == other:getValue()
    end,
    between = function(self, min, max)
        local v = self:getValue()
        return v >= min:getValue() and v <= max:getValue()
    end
})

local Printable = createTrait("Printable", {"toString"}, {
    print = function(self)
        print(self:toString())
    end,
    println = function(self)
        print(self:toString() .. "\n")
    end
})

local Clonable = createTrait("Clonable", {}, {
    clone = function(self)
        local copy = {}
        for k, v in pairs(self) do
            copy[k] = v
        end
        return setmetatable(copy, getmetatable(self))
    end,
    deepClone = function(self)
        local function deepCopy(obj)
            if type(obj) ~= "table" then return obj end
            local copy = {}
            for k, v in pairs(obj) do
                copy[deepCopy(k)] = deepCopy(v)
            end
            return setmetatable(copy, getmetatable(obj))
        end
        return deepCopy(self)
    end
})

-- Temperature class
local Temperature = {}
Temperature.__index = Temperature

-- ต้องมี getValue ก่อน apply Comparable trait
function Temperature.new(celsius)
    return setmetatable({celsius = celsius}, Temperature)
end

function Temperature:getValue()
    return self.celsius
end

function Temperature:toFahrenheit()
    return self.celsius * 9/5 + 32
end

function Temperature:toKelvin()
    return self.celsius + 273.15
end

function Temperature:toString()
    return string.format("%.1f°C (%.1f°F, %.2fK)", 
        self.celsius, self:toFahrenheit(), self:toKelvin())
end

-- Apply traits
applyTrait(Temperature, Comparable)
applyTrait(Temperature, Printable)
applyTrait(Temperature, Clonable)

-- ทดสอบ
local t1 = Temperature.new(20)
local t2 = Temperature.new(35)
local t3 = Temperature.new(0)

t1:print()
t2:print()
t3:print()

print("\nการเปรียบเทียบ:")
print(string.format("%.1f < %.1f: %s", t1:getValue(), t2:getValue(), tostring(t1:lessThan(t2))))
print(string.format("%.1f > %.1f: %s", t2:getValue(), t1:getValue(), tostring(t2:greaterThan(t1))))
print(string.format("%.1f อยู่ระหว่าง %.1f และ %.1f: %s",
    t1:getValue(), t3:getValue(), t2:getValue(), 
    tostring(t1:between(t3, t2))))

local t4 = t1:clone()
t4.celsius = 25
print("\nClone:", t4:toString())
print("Original:", t1:toString())

print("\nHas trait 'Comparable':", hasTrait(t1, "Comparable"))
print("Has trait 'Printable':", hasTrait(t1, "Printable"))
print("Has trait 'Clonable':", hasTrait(t1, "Clonable"))
```

---

## 23.3 Observable Mixin

### ตัวอย่างที่ 6: Observable Pattern

```lua
-- Observable Mixin: เพิ่ม event system ให้กับ class ใดก็ได้
local Observable = {}

function Observable:initObservable()
    self._observers = {}
    self._onceObservers = {}
end

function Observable:on(event, handler)
    if not self._observers then self:initObservable() end
    self._observers[event] = self._observers[event] or {}
    table.insert(self._observers[event], handler)
    return self  -- chainable
end

function Observable:once(event, handler)
    if not self._onceObservers then self:initObservable() end
    self._onceObservers[event] = self._onceObservers[event] or {}
    table.insert(self._onceObservers[event], handler)
    return self
end

function Observable:off(event, handler)
    if not self._observers then return end
    if handler then
        local handlers = self._observers[event] or {}
        for i, h in ipairs(handlers) do
            if h == handler then
                table.remove(handlers, i)
                return
            end
        end
    else
        self._observers[event] = nil
    end
end

function Observable:emit(event, ...)
    if not self._observers then return end
    
    -- Regular handlers
    local handlers = self._observers[event] or {}
    for _, handler in ipairs(handlers) do
        handler(self, ...)
    end
    
    -- One-time handlers
    local onceHandlers = self._onceObservers and 
                         self._onceObservers[event] or {}
    for _, handler in ipairs(onceHandlers) do
        handler(self, ...)
    end
    if self._onceObservers then
        self._onceObservers[event] = nil
    end
end

function Observable:listenerCount(event)
    local count = 0
    if self._observers and self._observers[event] then
        count = count + #self._observers[event]
    end
    if self._onceObservers and self._onceObservers[event] then
        count = count + #self._onceObservers[event]
    end
    return count
end

-- Apply Observable to a class
local function makeObservable(cls)
    for k, v in pairs(Observable) do
        if type(v) == "function" then
            cls[k] = v
        end
    end
    return cls
end

-- Counter ที่ใช้ Observable
local Counter = {}
Counter.__index = Counter
makeObservable(Counter)

function Counter.new(initial)
    local self = setmetatable({value = initial or 0}, Counter)
    self:initObservable()
    return self
end

function Counter:increment(by)
    local old = self.value
    self.value = self.value + (by or 1)
    self:emit("change", self.value, old)
    if self.value > old then
        self:emit("increment", self.value - old)
    end
end

function Counter:decrement(by)
    local old = self.value
    self.value = self.value - (by or 1)
    self:emit("change", self.value, old)
    if self.value < old then
        self:emit("decrement", old - self.value)
    end
end

function Counter:reset()
    local old = self.value
    self.value = 0
    self:emit("reset", old)
    self:emit("change", 0, old)
end

function Counter:getValue()
    return self.value
end

-- ทดสอบ Observable Counter
local counter = Counter.new(0)

-- Subscribe to events
counter:on("change", function(self, newVal, oldVal)
    print(string.format("  [change] %d -> %d", oldVal, newVal))
end)

counter:on("increment", function(self, amount)
    print(string.format("  [increment] +%d (ปัจจุบัน: %d)", amount, self:getValue()))
end)

counter:once("reset", function(self, oldVal)
    print(string.format("  [reset] ล้างค่าจาก %d (handler นี้เรียกแค่ครั้งเดียว)", oldVal))
end)

print("=== Observable Counter Demo ===")
counter:increment()
counter:increment(5)
counter:decrement(2)
counter:reset()
counter:reset()  -- handler "once" จะไม่ถูกเรียกอีก

print("\nจำนวน 'change' listeners:", counter:listenerCount("change"))
```

### ตัวอย่างที่ 7: EventEmitter Mixin ขั้นสูง

```lua
-- EventEmitter ที่สมบูรณ์ยิ่งขึ้น
local EventEmitter = {}
EventEmitter.__index = EventEmitter

function EventEmitter.new()
    return setmetatable({
        _events = {},
        _maxListeners = 10
    }, EventEmitter)
end

function EventEmitter:setMaxListeners(n)
    self._maxListeners = n
    return self
end

function EventEmitter:addListener(event, listener)
    return self:on(event, listener)
end

function EventEmitter:on(event, listener)
    self._events[event] = self._events[event] or {}
    local listeners = self._events[event]
    
    if #listeners >= self._maxListeners then
        print(string.format("[WARNING] Event '%s' มี listeners เกิน %d", 
            event, self._maxListeners))
    end
    
    table.insert(listeners, {fn = listener, once = false})
    self:emit("newListener", event, listener)
    return self
end

function EventEmitter:once(event, listener)
    self._events[event] = self._events[event] or {}
    table.insert(self._events[event], {fn = listener, once = true})
    return self
end

function EventEmitter:removeListener(event, listener)
    if not self._events[event] then return self end
    local listeners = self._events[event]
    for i, l in ipairs(listeners) do
        if l.fn == listener then
            table.remove(listeners, i)
            self:emit("removeListener", event, listener)
            return self
        end
    end
    return self
end

function EventEmitter:removeAllListeners(event)
    if event then
        self._events[event] = nil
    else
        self._events = {}
    end
    return self
end

function EventEmitter:emit(event, ...)
    local listeners = self._events[event]
    if not listeners then return false end
    
    local toRemove = {}
    for i, l in ipairs(listeners) do
        l.fn(...)
        if l.once then
            table.insert(toRemove, i)
        end
    end
    
    -- ลบ once listeners (ลบจากท้ายไปหน้า)
    for i = #toRemove, 1, -1 do
        table.remove(listeners, toRemove[i])
    end
    
    return true
end

function EventEmitter:listeners(event)
    local result = {}
    if self._events[event] then
        for _, l in ipairs(self._events[event]) do
            table.insert(result, l.fn)
        end
    end
    return result
end

function EventEmitter:listenerCount(event)
    return self._events[event] and #self._events[event] or 0
end

function EventEmitter:eventNames()
    local names = {}
    for event in pairs(self._events) do
        table.insert(names, event)
    end
    return names
end

-- ทดสอบ EventEmitter
local emitter = EventEmitter.new()

-- Listener สำหรับ "newListener" event
emitter:on("newListener", function(event, listener)
    if event ~= "newListener" then  -- ป้องกัน recursive
        print("  เพิ่ม listener สำหรับ event: " .. event)
    end
end)

-- สร้าง handlers
local function onData(data)
    print("  [data] ได้รับ: " .. tostring(data))
end

local function onError(err)
    print("  [error] ข้อผิดพลาด: " .. tostring(err))
end

local function onClose()
    print("  [close] การเชื่อมต่อปิดแล้ว (once)")
end

-- Register
emitter:on("data", onData)
emitter:on("error", onError)
emitter:once("close", onClose)

print("=== EventEmitter Demo ===")
print("Events:", table.concat(emitter:eventNames(), ", "))
print("data listeners:", emitter:listenerCount("data"))

emitter:emit("data", "Hello World")
emitter:emit("data", "Lua is awesome")
emitter:emit("error", "Connection timeout")
emitter:emit("close")
emitter:emit("close")  -- ไม่มี listener แล้ว

print("\nหลังลบ data listener:")
emitter:removeListener("data", onData)
print("data listeners:", emitter:listenerCount("data"))
emitter:emit("data", "ข้อความนี้จะไม่ถูกรับ")
```

---

## 23.4 Serializable Mixin

### ตัวอย่างที่ 8: Serializable Mixin

```lua
-- Serializable Mixin: แปลง object เป็น string และกลับมา
local Serializable = {}

function Serializable:serialize()
    local result = {}
    for k, v in pairs(self) do
        if type(v) ~= "function" and not k:match("^_") then
            local serializedVal
            if type(v) == "table" then
                serializedVal = "{table}"  -- simplified
            elseif type(v) == "string" then
                serializedVal = '"' .. v:gsub('"', '\\"') .. '"'
            else
                serializedVal = tostring(v)
            end
            table.insert(result, k .. "=" .. serializedVal)
        end
    end
    table.sort(result)
    return "{" .. table.concat(result, ",") .. "}"
end

function Serializable:toJSON()
    local function escapeString(s)
        return s:gsub('\\', '\\\\'):gsub('"', '\\"'):gsub('\n', '\\n')
    end
    
    local function valueToJSON(v, indent)
        indent = indent or 0
        local spaces = string.rep("  ", indent)
        
        if type(v) == "nil" then
            return "null"
        elseif type(v) == "boolean" then
            return tostring(v)
        elseif type(v) == "number" then
            if v ~= v then return "null"  -- NaN
            elseif v == math.huge then return "null"  -- Infinity
            else return string.format("%g", v) end
        elseif type(v) == "string" then
            return '"' .. escapeString(v) .. '"'
        elseif type(v) == "table" then
            -- ตรวจสอบว่าเป็น array หรือ object
            local isArray = true
            local maxIdx = 0
            for k, _ in pairs(v) do
                if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                    isArray = false
                    break
                end
                maxIdx = math.max(maxIdx, k)
            end
            
            if isArray and maxIdx == #v then
                -- Array
                local items = {}
                for _, item in ipairs(v) do
                    table.insert(items, spaces .. "  " .. valueToJSON(item, indent+1))
                end
                if #items == 0 then return "[]" end
                return "[\n" .. table.concat(items, ",\n") .. "\n" .. spaces .. "]"
            else
                -- Object
                local items = {}
                local keys = {}
                for k in pairs(v) do
                    if type(k) == "string" and not k:match("^_") then
                        table.insert(keys, k)
                    end
                end
                table.sort(keys)
                for _, k in ipairs(keys) do
                    local val = v[k]
                    if type(val) ~= "function" then
                        table.insert(items, spaces .. '  "' .. k .. '": ' .. 
                            valueToJSON(val, indent+1))
                    end
                end
                if #items == 0 then return "{}" end
                return "{\n" .. table.concat(items, ",\n") .. "\n" .. spaces .. "}"
            end
        else
            return '"[' .. type(v) .. ']"'
        end
    end
    
    return valueToJSON(self)
end

function Serializable:toCSV(fields)
    local values = {}
    for _, field in ipairs(fields) do
        local v = self[field]
        if type(v) == "string" and v:find(",") then
            table.insert(values, '"' .. v .. '"')
        else
            table.insert(values, tostring(v or ""))
        end
    end
    return table.concat(values, ",")
end

-- Product class ที่ใช้ Serializable
local Product = {}
Product.__index = Product

for k, v in pairs(Serializable) do
    if type(v) == "function" then
        Product[k] = v
    end
end

function Product.new(id, name, price, category, tags)
    return setmetatable({
        id = id,
        name = name,
        price = price,
        category = category,
        tags = tags or {},
        inStock = true
    }, Product)
end

-- ทดสอบ
local p = Product.new("P001", "MacBook Pro", 89000, "Electronics", 
    {"laptop", "apple", "pro"})

print("=== Serialization Demo ===")
print("\ntoString:")
print(p:serialize())

print("\nJSON:")
print(p:toJSON())

print("\nCSV:")
print("id,name,price,category")
print(p:toCSV({"id", "name", "price", "category"}))
```

---

## 23.5 Real Example: สัตว์ที่บิน ว่ายน้ำ เดินได้

### ตัวอย่างที่ 9-15: Animal Abilities System

```lua
-- ===== Ability Mixins =====
local Walkable = {}
function Walkable:walk(destination)
    print(string.format("%s เดินไปที่ %s", self.name, destination))
    self:emit("move", "walk", destination)
end
function Walkable:run(destination)
    print(string.format("%s วิ่งไปที่ %s ด้วยความเร็ว %d km/h", 
        self.name, destination, self.runSpeed or 10))
    self:emit("move", "run", destination)
end
function Walkable:canWalk() return true end

local Flyable = {}
function Flyable:fly(destination)
    print(string.format("%s บินไปที่ %s", self.name, destination))
    self:emit("move", "fly", destination)
end
function Flyable:soar()
    print(string.format("%s เหินฟ้าสูง", self.name))
end
function Flyable:canFly() return true end
function Flyable:getMaxAltitude()
    return self.maxAltitude or 1000
end

local Swimmable = {}
function Swimmable:swim(destination)
    print(string.format("%s ว่ายน้ำไปที่ %s", self.name, destination))
    self:emit("move", "swim", destination)
end
function Swimmable:dive(depth)
    print(string.format("%s ดำลึก %d เมตร", self.name, depth))
end
function Swimmable:canSwim() return true end
function Swimmable:getMaxDepth()
    return self.maxDepth or 10
end

local Climbable = {}
function Climbable:climb(target)
    print(string.format("%s ปีนขึ้น %s", self.name, target))
    self:emit("move", "climb", target)
end
function Climbable:canClimb() return true end

-- ===== Base Animal =====
local function createAnimalMixin()
    return {
        initAnimal = function(self, name, species)
            self.name = name
            self.species = species
            self._moveLog = {}
            self._events = {}
        end,
        
        on = function(self, event, handler)
            self._events = self._events or {}
            self._events[event] = self._events[event] or {}
            table.insert(self._events[event], handler)
        end,
        
        emit = function(self, event, ...)
            if self._events and self._events[event] then
                for _, h in ipairs(self._events[event]) do
                    h(self, ...)
                end
            end
        end,
        
        describe = function(self)
            local abilities = {}
            if self.canWalk and self:canWalk() then table.insert(abilities, "เดินได้") end
            if self.canFly and self:canFly() then table.insert(abilities, "บินได้") end
            if self.canSwim and self:canSwim() then table.insert(abilities, "ว่ายน้ำได้") end
            if self.canClimb and self:canClimb() then table.insert(abilities, "ปีนได้") end
            print(string.format("%s (%s): %s", 
                self.name, self.species, table.concat(abilities, ", ")))
        end
    }
end

local animalBase = createAnimalMixin()

local function makeAnimal(cls)
    for k, v in pairs(animalBase) do cls[k] = v end
    return cls
end

-- ===== Concrete Animals =====

-- Eagle: บินได้ + เดินได้
local Eagle = {}; Eagle.__index = Eagle
makeAnimal(Eagle)
for k, v in pairs(Flyable) do Eagle[k] = v end
for k, v in pairs(Walkable) do Eagle[k] = v end

function Eagle.new(name)
    local self = setmetatable({}, Eagle)
    self:initAnimal(name, "นกอินทรี")
    self.maxAltitude = 3000
    self.runSpeed = 5
    return self
end

function Eagle:hunt(prey)
    print(string.format("%s โฉบจับ %s", self.name, prey))
end

-- Duck: บินได้ + ว่ายน้ำได้ + เดินได้
local DuckAnimal = {}; DuckAnimal.__index = DuckAnimal
makeAnimal(DuckAnimal)
for k, v in pairs(Flyable) do DuckAnimal[k] = v end
for k, v in pairs(Swimmable) do DuckAnimal[k] = v end
for k, v in pairs(Walkable) do DuckAnimal[k] = v end

function DuckAnimal.new(name)
    local self = setmetatable({}, DuckAnimal)
    self:initAnimal(name, "เป็ด")
    self.maxAltitude = 200
    self.maxDepth = 3
    self.runSpeed = 3
    return self
end

function DuckAnimal:quack()
    print(self.name .. " ร้อง: แกร๊ก!")
end

-- Penguin: ว่ายน้ำได้ + เดินได้ (บินไม่ได้!)
local Penguin = {}; Penguin.__index = Penguin
makeAnimal(Penguin)
for k, v in pairs(Swimmable) do Penguin[k] = v end
for k, v in pairs(Walkable) do Penguin[k] = v end

function Penguin.new(name)
    local self = setmetatable({}, Penguin)
    self:initAnimal(name, "เพนกวิน")
    self.maxDepth = 535  -- Emperor penguin!
    self.runSpeed = 3
    return self
end

function Penguin:slide()
    print(self.name .. " ไถลบนน้ำแข็ง!")
end

-- Monkey: เดินได้ + ปีนได้
local Monkey = {}; Monkey.__index = Monkey
makeAnimal(Monkey)
for k, v in pairs(Walkable) do Monkey[k] = v end
for k, v in pairs(Climbable) do Monkey[k] = v end

function Monkey.new(name)
    local self = setmetatable({}, Monkey)
    self:initAnimal(name, "ลิง")
    self.runSpeed = 15
    return self
end

function Monkey:swing(branch)
    print(self.name .. " ห้อยโหนไปที่ " .. branch)
end

-- Flying Fish: บินได้ + ว่ายน้ำได้ (ปลาบิน)
local FlyingFish = {}; FlyingFish.__index = FlyingFish
makeAnimal(FlyingFish)
for k, v in pairs(Flyable) do FlyingFish[k] = v end
for k, v in pairs(Swimmable) do FlyingFish[k] = v end

function FlyingFish.new(name)
    local self = setmetatable({}, FlyingFish)
    self:initAnimal(name, "ปลาบิน")
    self.maxAltitude = 6  -- เมตร
    self.maxDepth = 20
    return self
end

-- Duck-billed Platypus: ว่ายน้ำได้ + เดินได้ (สัตว์แปลก!)
local Platypus = {}; Platypus.__index = Platypus
makeAnimal(Platypus)
for k, v in pairs(Swimmable) do Platypus[k] = v end
for k, v in pairs(Walkable) do Platypus[k] = v end

function Platypus.new(name)
    local self = setmetatable({}, Platypus)
    self:initAnimal(name, "ตุ่นปากเป็ด")
    self.maxDepth = 5
    self.runSpeed = 2
    return self
end

function Platypus:echolocate()
    print(self.name .. " ใช้ electroreception ตรวจจับเหยื่อ")
end

-- ===== ทดสอบ Animal System =====
print("=== Animal Abilities System ===\n")

local animals = {
    Eagle.new("อินทรี"),
    DuckAnimal.new("โดนัลด์"),
    Penguin.new("เพนกวิน"),
    Monkey.new("ลิงชิมแปนซี"),
    FlyingFish.new("ปลาบิน"),
    Platypus.new("ตุ่น")
}

print("--- รายการสัตว์และความสามารถ ---")
for _, animal in ipairs(animals) do
    animal:describe()
end

print("\n--- ทดสอบการเคลื่อนไหว ---")
for _, animal in ipairs(animals) do
    print("\n[" .. animal.species .. "]")
    if animal.canWalk and animal:canWalk() then
        animal:walk("ทุ่งหญ้า")
    end
    if animal.canFly and animal:canFly() then
        animal:fly("ท้องฟ้า")
    end
    if animal.canSwim and animal:canSwim() then
        animal:swim("แม่น้ำ")
    end
end

-- Polymorphic function
print("\n--- สัตว์ที่บินได้ทั้งหมด ---")
for _, animal in ipairs(animals) do
    if animal.canFly and animal:canFly() then
        print(string.format("  %s (%s) - บินสูงสุด %d เมตร", 
            animal.name, animal.species, animal:getMaxAltitude()))
    end
end

print("\n--- สัตว์ที่ว่ายน้ำได้ทั้งหมด ---")
for _, animal in ipairs(animals) do
    if animal.canSwim and animal:canSwim() then
        print(string.format("  %s (%s) - ดำได้ลึกสุด %d เมตร", 
            animal.name, animal.species, animal:getMaxDepth()))
    end
end
```

---

## 23.6 Decorator Pattern ด้วย Mixin

### ตัวอย่างที่ 16-20: Decorator Pattern

```lua
-- Decorator Pattern: เพิ่ม behavior แบบ dynamic
local function decorator(obj, decoratorFn)
    return decoratorFn(obj)
end

-- Base function/object
local function createCoffee(type, cost)
    return {
        type = type,
        cost = cost,
        description = type,
        getDescription = function(self)
            return self.description
        end,
        getCost = function(self)
            return self.cost
        end
    }
end

-- Decorators
local function withMilk(coffee)
    local original = coffee
    return {
        type = original.type,
        cost = original.cost + 15,
        description = original.description .. " + นม",
        getDescription = function(self) return self.description end,
        getCost = function(self) return self.cost end
    }
end

local function withSugar(coffee)
    local original = coffee
    return {
        type = original.type,
        cost = original.cost + 5,
        description = original.description .. " + น้ำตาล",
        getDescription = function(self) return self.description end,
        getCost = function(self) return self.cost end
    }
end

local function withWhip(coffee)
    local original = coffee
    return {
        type = original.type,
        cost = original.cost + 25,
        description = original.description .. " + วิปครีม",
        getDescription = function(self) return self.description end,
        getCost = function(self) return self.cost end
    }
end

local function withShot(coffee)
    local original = coffee
    return {
        type = original.type,
        cost = original.cost + 20,
        description = original.description .. " + espresso shot",
        getDescription = function(self) return self.description end,
        getCost = function(self) return self.cost end
    }
end

-- ทดสอบ Decorator
print("=== Coffee Decorator Demo ===\n")

local espresso = createCoffee("Espresso", 55)
print(string.format("%s: ฿%d", espresso:getDescription(), espresso:getCost()))

local lattea = withMilk(withShot(createCoffee("Latte", 65)))
print(string.format("%s: ฿%d", lattea:getDescription(), lattea:getCost()))

local fancyCoffee = withWhip(withSugar(withMilk(withShot(createCoffee("Mocha", 75)))))
print(string.format("%s: ฿%d", fancyCoffee:getDescription(), fancyCoffee:getCost()))
```

### ตัวอย่างที่ 21: OOP Decorator Pattern

```lua
-- OOP Decorator Pattern ที่สมบูรณ์กว่า
local Component = {}
Component.__index = Component

function Component.new(name)
    return setmetatable({name = name}, Component)
end

function Component:operation()
    return self.name
end

function Component:cost()
    return 0
end

-- Abstract Decorator
local Decorator = setmetatable({}, {__index = Component})
Decorator.__index = Decorator

function Decorator.new(component)
    local self = setmetatable({}, Decorator)
    self._component = component
    self.name = component.name
    return self
end

function Decorator:operation()
    return self._component:operation()
end

function Decorator:cost()
    return self._component:cost()
end

-- Concrete Decorators
local function makeDecorator(name, addedDesc, addedCost)
    local D = setmetatable({}, {__index = Decorator})
    D.__index = D
    
    function D.new(component)
        local self = Decorator.new(component)
        setmetatable(self, D)
        return self
    end
    
    function D:operation()
        return Decorator.operation(self) .. " + " .. addedDesc
    end
    
    function D:cost()
        return Decorator.cost(self) + addedCost
    end
    
    return D
end

local LoggingDecorator = makeDecorator("LoggingDecorator", "logged", 0)
local CachingDecorator = makeDecorator("CachingDecorator", "cached", 10)
local RetryDecorator = makeDecorator("RetryDecorator", "with-retry", 5)
local MetricsDecorator = makeDecorator("MetricsDecorator", "with-metrics", 3)

-- ทดสอบ
local base = Component.new("DataService")
print("Base:", base:operation(), "Cost:", base:cost())

local withLogging = LoggingDecorator.new(base)
print("With Logging:", withLogging:operation(), "Cost:", withLogging:cost())

local fullService = MetricsDecorator.new(
    RetryDecorator.new(
        CachingDecorator.new(
            LoggingDecorator.new(base)
        )
    )
)
print("Full Service:", fullService:operation(), "Cost:", fullService:cost())
```

---

## 23.7 GUI Widget Mixins ขั้นสูง

### ตัวอย่างที่ 22-27: GUI Mixin System

```lua
-- GUI Mixin System ที่ครบถ้วน

-- ===== Mixins =====

-- Resizable Mixin
local Resizable = {}
function Resizable:resize(width, height)
    local oldW, oldH = self.width, self.height
    self.width = math.max(self.minWidth or 0, width)
    self.height = math.max(self.minHeight or 0, height)
    if self.onResize then
        self:onResize(self.width, self.height, oldW, oldH)
    end
    return self
end
function Resizable:setMinSize(w, h)
    self.minWidth = w
    self.minHeight = h
    return self
end
function Resizable:getSize()
    return self.width, self.height
end

-- Draggable Mixin
local Draggable = {}
function Draggable:startDrag(x, y)
    self._dragStartX = x - self.x
    self._dragStartY = y - self.y
    self._dragging = true
    print(self.id .. " เริ่ม drag")
end
function Draggable:drag(x, y)
    if self._dragging then
        self.x = x - self._dragStartX
        self.y = y - self._dragStartY
    end
end
function Draggable:endDrag()
    self._dragging = false
    print(string.format("%s หยุด drag ที่ (%d, %d)", self.id, self.x, self.y))
end
function Draggable:isDragging()
    return self._dragging == true
end

-- Focusable Mixin
local Focusable = {}
Focusable._focused = nil  -- shared state

function Focusable:focus()
    if Focusable._focused and Focusable._focused ~= self then
        Focusable._focused:blur()
    end
    self._focused_self = true
    Focusable._focused = self
    if self.onFocus then self:onFocus() end
    print(self.id .. " ได้รับ focus")
end
function Focusable:blur()
    self._focused_self = false
    if Focusable._focused == self then
        Focusable._focused = nil
    end
    if self.onBlur then self:onBlur() end
    print(self.id .. " เสีย focus")
end
function Focusable:isFocused()
    return self._focused_self == true
end

-- Tooltip Mixin
local Tooltip = {}
function Tooltip:setTooltip(text)
    self._tooltip = text
    return self
end
function Tooltip:showTooltip()
    if self._tooltip then
        print(string.format("[Tooltip] %s: %s", self.id, self._tooltip))
    end
end
function Tooltip:hideTooltip()
    -- ซ่อน tooltip
end

-- Animation Mixin
local Animatable = {}
function Animatable:animate(property, from, to, duration)
    self._animations = self._animations or {}
    table.insert(self._animations, {
        property = property,
        from = from,
        to = to,
        duration = duration,
        elapsed = 0,
        active = true
    })
    print(string.format("%s: animate %s จาก %s ไป %s ใน %.1fs",
        self.id, property, tostring(from), tostring(to), duration))
end
function Animatable:updateAnimations(dt)
    if not self._animations then return end
    for _, anim in ipairs(self._animations) do
        if anim.active then
            anim.elapsed = anim.elapsed + dt
            local t = math.min(1, anim.elapsed / anim.duration)
            self[anim.property] = anim.from + (anim.to - anim.from) * t
            if t >= 1 then anim.active = false end
        end
    end
end

-- ===== Widget Class ใช้ทุก Mixins =====
local UIWidget = {}
UIWidget.__index = UIWidget

-- Apply all mixins
for k, v in pairs(Resizable) do UIWidget[k] = v end
for k, v in pairs(Draggable) do UIWidget[k] = v end
for k, v in pairs(Focusable) do UIWidget[k] = v end
for k, v in pairs(Tooltip) do UIWidget[k] = v end
for k, v in pairs(Animatable) do UIWidget[k] = v end

function UIWidget.new(id, x, y, w, h)
    local self = setmetatable({
        id = id,
        x = x, y = y,
        width = w, height = h,
        visible = true,
        enabled = true
    }, UIWidget)
    return self
end

function UIWidget:render()
    print(string.format("[%s] @(%d,%d) %dx%d focused=%s",
        self.id, self.x, self.y, self.width, self.height,
        tostring(self:isFocused())))
end

function UIWidget:onFocus()
    print("  -> " .. self.id .. " กำลังถูก highlight")
end

function UIWidget:onBlur()
    print("  -> " .. self.id .. " ยกเลิก highlight")
end

function UIWidget:onResize(w, h, oldW, oldH)
    print(string.format("  -> %s ขนาดเปลี่ยน: %dx%d -> %dx%d",
        self.id, oldW, oldH, w, h))
end

-- ทดสอบ GUI Widget
print("=== GUI Widget Mixin Demo ===\n")

local panel = UIWidget.new("panel1", 10, 10, 200, 150)
local button = UIWidget.new("btn1", 20, 20, 100, 30)
local input = UIWidget.new("input1", 20, 60, 160, 30)

panel:setTooltip("แผง UI หลัก")
button:setTooltip("คลิกเพื่อส่ง")
input:setTooltip("กรอกข้อความ")

-- Focus
button:focus()
input:focus()  -- button จะเสีย focus อัตโนมัติ

-- Drag
panel:startDrag(50, 50)
panel:drag(100, 100)
panel:endDrag()

-- Resize
panel:setMinSize(100, 80)
panel:resize(300, 200)
panel:resize(50, 50)  -- จะถูก clamp ด้วย minSize

-- Animation
button:animate("x", 20, 200, 0.5)
button:animate("width", 100, 150, 0.3)

-- Tooltips
panel:showTooltip()
button:showTooltip()

-- Render
print("\n--- Widget States ---")
panel:render()
button:render()
input:render()
```

---

## 23.8 Role/Interface Simulation

### ตัวอย่างที่ 28-32: Role Pattern

```lua
-- Role Pattern: ใช้ mixins เป็น "roles" ที่ class สามารถรับได้
local RoleSystem = {}

-- สร้าง Role
function RoleSystem.createRole(name, requirements, methods)
    return {
        _roleName = name,
        _requirements = requirements or {},
        _methods = methods or {}
    }
end

-- Apply role to class
function RoleSystem.applyRole(cls, role)
    -- ตรวจสอบ requirements
    for _, req in ipairs(role._requirements) do
        assert(rawget(cls, req) ~= nil or cls[req] ~= nil,
            string.format("Class ต้องมี '%s' เพื่อรับ role '%s'",
                req, role._roleName))
    end
    
    -- Apply methods
    for k, v in pairs(role._methods) do
        if cls[k] == nil then
            cls[k] = v
        else
            -- Method conflict: เพิ่ม _original_ version
            cls["_" .. role._roleName .. "_" .. k] = v
        end
    end
    
    cls._roles = cls._roles or {}
    table.insert(cls._roles, role._roleName)
    return cls
end

function RoleSystem.hasRole(obj, roleName)
    local cls = getmetatable(obj)
    if cls and cls._roles then
        for _, r in ipairs(cls._roles) do
            if r == roleName then return true end
        end
    end
    return false
end

-- สร้าง Roles
local Greetable = RoleSystem.createRole("Greetable", {"getName"}, {
    greet = function(self)
        print("สวัสดี! ฉันชื่อ " .. self:getName())
    end,
    farewell = function(self)
        print("ลาก่อน! จาก " .. self:getName())
    end
})

local Rankable = RoleSystem.createRole("Rankable", {}, {
    initRank = function(self, initial)
        self._rank = initial or 1
        self._rankHistory = {}
    end,
    promote = function(self)
        table.insert(self._rankHistory, {rank = self._rank, time = os.time()})
        self._rank = self._rank + 1
        print(string.format("%s ได้รับการเลื่อนตำแหน่งเป็น rank %d!", 
            self:getName(), self._rank))
    end,
    getRank = function(self)
        return self._rank or 1
    end
})

local Rewarding = RoleSystem.createRole("Rewarding", {"getName"}, {
    initRewards = function(self)
        self._rewards = {}
        self._points = 0
    end,
    addReward = function(self, reward, points)
        table.insert(self._rewards, reward)
        self._points = self._points + (points or 10)
        print(string.format("%s ได้รับรางวัล '%s' (+%d คะแนน, รวม %d)",
            self:getName(), reward, points or 10, self._points))
    end,
    getPoints = function(self)
        return self._points
    end,
    getRewards = function(self)
        return self._rewards
    end
})

-- Player class ที่รับ Roles
local GamePlayer = {}
GamePlayer.__index = GamePlayer

function GamePlayer:getName() return self.playerName end

RoleSystem.applyRole(GamePlayer, Greetable)
RoleSystem.applyRole(GamePlayer, Rankable)
RoleSystem.applyRole(GamePlayer, Rewarding)

function GamePlayer.new(name)
    local self = setmetatable({playerName = name}, GamePlayer)
    self:initRank(1)
    self:initRewards()
    return self
end

-- ทดสอบ Role System
print("=== Role System Demo ===\n")

local player = GamePlayer.new("วีรชาติ")
player:greet()

player:promote()
player:promote()
player:promote()

player:addReward("ผู้ชนะรอบแรก", 50)
player:addReward("การเล่นต่อเนื่อง 7 วัน", 100)
player:addReward("สังหาร Boss", 200)

player:farewell()

print(string.format("\nสรุป: rank %d, %d คะแนน, %d รางวัล",
    player:getRank(), player:getPoints(), #player:getRewards()))

print("Has role 'Greetable':", RoleSystem.hasRole(player, "Greetable"))
print("Has role 'Rankable':", RoleSystem.hasRole(player, "Rankable"))
print("Has role 'Rewarding':", RoleSystem.hasRole(player, "Rewarding"))
print("Has role 'Admin':", RoleSystem.hasRole(player, "Admin"))
```

---

## 23.9 Aspect-Oriented Patterns

### ตัวอย่างที่ 33-37: AOP ใน Lua

```lua
-- Aspect-Oriented Programming (AOP) ด้วย Lua
-- เพิ่ม cross-cutting concerns โดยไม่แก้ไขโค้ดหลัก

-- Aspect Weaver
local AOP = {}

-- Before advice
function AOP.before(obj, methodName, advice)
    local original = obj[methodName]
    assert(type(original) == "function", 
        "ไม่พบ method: " .. methodName)
    
    obj[methodName] = function(self, ...)
        advice(self, methodName, ...)
        return original(self, ...)
    end
end

-- After advice
function AOP.after(obj, methodName, advice)
    local original = obj[methodName]
    obj[methodName] = function(self, ...)
        local results = {original(self, ...)}
        advice(self, methodName, results, ...)
        return table.unpack(results)
    end
end

-- Around advice
function AOP.around(obj, methodName, advice)
    local original = obj[methodName]
    obj[methodName] = function(self, ...)
        return advice(self, original, methodName, ...)
    end
end

-- Apply aspects to all methods
function AOP.applyToAll(obj, aspect)
    for k, v in pairs(obj) do
        if type(v) == "function" and not k:match("^_") and k ~= "new" then
            aspect(obj, k)
        end
    end
end

-- ===== Aspects =====

-- Logging Aspect
local function loggingAspect(obj, methodName)
    AOP.around(obj, methodName, function(self, original, name, ...)
        print(string.format("[LOG] เรียก %s.%s(%s)", 
            tostring(self.name or "obj"), name,
            table.concat({...}, ", ")))
        local start = os.clock()
        local results = {original(self, ...)}
        local elapsed = os.clock() - start
        print(string.format("[LOG] %s.%s ใช้เวลา %.4fs", 
            tostring(self.name or "obj"), name, elapsed))
        return table.unpack(results)
    end)
end

-- Validation Aspect
local function validationAspect(validations)
    return function(obj, methodName)
        if validations[methodName] then
            AOP.before(obj, methodName, function(self, name, ...)
                local args = {...}
                for i, validate in ipairs(validations[methodName]) do
                    local ok, err = validate(args[i])
                    if not ok then
                        error(string.format("Validation error for %s arg[%d]: %s",
                            name, i, err))
                    end
                end
            end)
        end
    end
end

-- Caching Aspect
local function cachingAspect(obj, methodName, ttl)
    local cache = {}
    AOP.around(obj, methodName, function(self, original, name, ...)
        local key = name .. "_" .. table.concat({...}, "_")
        local now = os.time()
        
        if cache[key] and (not ttl or now - cache[key].time < ttl) then
            print(string.format("[CACHE] hit: %s", key))
            return table.unpack(cache[key].results)
        end
        
        local results = {original(self, ...)}
        cache[key] = {results = results, time = now}
        print(string.format("[CACHE] miss: %s (cached)", key))
        return table.unpack(results)
    end)
end

-- ===== Target Class =====
local Calculator = {}
Calculator.__index = Calculator

function Calculator.new(name)
    return setmetatable({name = name}, Calculator)
end

function Calculator:add(a, b)
    return a + b
end

function Calculator:multiply(a, b)
    return a * b
end

function Calculator:factorial(n)
    if n <= 1 then return 1 end
    return n * self:factorial(n - 1)
end

function Calculator:fibonacci(n)
    if n <= 1 then return n end
    return self:fibonacci(n-1) + self:fibonacci(n-2)
end

-- Apply Aspects
print("=== AOP Demo ===\n")

local calc = Calculator.new("MyCalc")

-- Caching สำหรับ fibonacci (recursive ช้ามาก)
cachingAspect(calc, "fibonacci", 60)

-- Logging สำหรับ add และ multiply
AOP.around(calc, "add", function(self, original, name, a, b)
    print(string.format("[LOG] %d + %d = ?", a, b))
    local result = original(self, a, b)
    print(string.format("[LOG] result = %d", result))
    return result
end)

print("--- Add with logging ---")
print(calc:add(3, 4))

print("\n--- Fibonacci with caching ---")
print("fib(10) =", calc:fibonacci(10))
print("fib(10) =", calc:fibonacci(10))  -- จาก cache
print("fib(8) =", calc:fibonacci(8))
```

---

## 23.10 Conflicting Methods Resolution

### ตัวอย่างที่ 38-40: Method Conflict Resolution

```lua
-- การแก้ปัญหา method conflicts ใน multiple inheritance

-- สร้าง class ที่จัดการ conflicts
local function createClassWithConflictResolution(name, bases, resolutions)
    local cls = {}
    cls.__index = cls
    cls._name = name
    cls._bases = bases
    resolutions = resolutions or {}
    
    -- Copy methods จาก bases
    for i = #bases, 1, -1 do  -- คัดลอกจากท้ายไปหน้า (base แรกมี priority สูงสุด)
        local base = bases[i]
        for k, v in pairs(base) do
            if type(v) == "function" then
                cls[k] = v
            end
        end
    end
    
    -- Apply manual resolutions
    for methodName, resolver in pairs(resolutions) do
        cls[methodName] = function(self, ...)
            return resolver(self, bases, ...)
        end
    end
    
    return cls
end

-- ===== ตัวอย่างจริง =====
local Saveable = {}
function Saveable:save()
    print("Saveable:save() - บันทึกเป็น binary")
end
function Saveable:load()
    print("Saveable:load() - โหลดจาก binary")
end
function Saveable:format()
    return "binary"
end

local JSONable = {}
function JSONable:save()
    print("JSONable:save() - บันทึกเป็น JSON")
end
function JSONable:load()
    print("JSONable:load() - โหลดจาก JSON")
end
function JSONable:format()
    return "json"
end

local XMLable = {}
function XMLable:save()
    print("XMLable:save() - บันทึกเป็น XML")
end
function XMLable:format()
    return "xml"
end

-- Document ที่รองรับทั้ง 3 formats
local MultiFormatDocument = createClassWithConflictResolution(
    "MultiFormatDocument",
    {JSONable, Saveable, XMLable},  -- JSON มี priority สูงสุด
    {
        -- แก้ conflict ด้วย custom resolver
        save = function(self, bases, format)
            format = format or "json"
            for _, base in ipairs(bases) do
                if base.format and base:format() == format then
                    base.save(self)
                    return
                end
            end
            -- Default: ใช้ bases[1]
            bases[1].save(self)
        end,
        format = function(self, bases)
            local formats = {}
            for _, base in ipairs(bases) do
                if base.format then
                    table.insert(formats, base.format(base))
                end
            end
            return table.concat(formats, ", ")
        end
    }
)
MultiFormatDocument.__index = MultiFormatDocument

function MultiFormatDocument.new(content)
    return setmetatable({content = content}, MultiFormatDocument)
end

-- ทดสอบ
print("=== Method Conflict Resolution ===\n")

local doc = MultiFormatDocument.new("เนื้อหาสำคัญ")

print("Formats รองรับ:", doc:format())
doc:save()         -- default (JSON)
doc:save("xml")    -- ระบุ format
doc:save("binary") -- ระบุ format
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Animal Kingdom
สร้าง system สำหรับ animal kingdom โดยใช้ mixins:
- สร้าง abilities: `Camouflage`, `Venomous`, `Nocturnal`, `Migratory`, `HibernatesInWinter`
- สร้างสัตว์อย่างน้อย 8 ชนิด โดยแต่ละชนิดมี abilities ต่างกัน
- เขียน function `findAnimalsWithAbility(animals, ability)` 
- เขียน function `getAnimalProfile(animal)` ที่แสดง abilities ทั้งหมด

### แบบฝึกหัดที่ 2: Plugin System
สร้าง Plugin System โดยใช้ Mixin pattern:
- `PluginManager` class ที่จัดการ plugins
- Plugins: `AuthPlugin`, `LogPlugin`, `RateLimitPlugin`, `CachePlugin`
- แต่ละ plugin เป็น mixin ที่เพิ่ม methods ให้ request handler
- ลำดับการ execute plugins ควรปรับได้

### แบบฝึกหัดที่ 3: State Machine Mixin
สร้าง `StateMachine` mixin ที่:
- จัดการ states และ transitions
- Emit events เมื่อ state เปลี่ยน
- ป้องกัน invalid transitions
- ทดสอบกับ `TrafficLight` (red/yellow/green) และ `Order` (pending/paid/shipped/delivered/cancelled)

### แบบฝึกหัดที่ 4: Reactive System
สร้าง Reactive programming mini-library ด้วย mixins:
- `Observable<T>`: เก็บ value และ notify เมื่อเปลี่ยน
- `Computed`: computed value ที่ depends on observables
- `Watch`: callback เมื่อ observable เปลี่ยน
- ทดสอบกับ form validation

### แบบฝึกหัดที่ 5: Repository Pattern
สร้าง Repository pattern ด้วย multiple inheritance:
- `Repository` base: `find()`, `findAll()`, `save()`, `delete()`
- `Pageable` mixin: `findPage(page, size)`
- `Searchable` mixin: `search(query)`
- `Auditable` mixin: บันทึก who/when ทุก operation
- สร้าง `UserRepository` และ `ProductRepository`

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Multiple Inheritance ด้วย search()**: ค้นหา method ใน bases หลายตัวตามลำดับ
2. **Copy-based Inheritance**: คัดลอก methods จาก bases มายัง class
3. **Mixin Pattern**: เพิ่ม functionality แบบ compose ได้อย่างยืดหยุ่น
4. **Trait Composition**: Mixin ที่มีการตรวจสอบ requirements
5. **Observable Mixin**: เพิ่ม event system ให้กับ class ใดก็ได้
6. **EventEmitter**: ระบบ event ที่สมบูรณ์พร้อม once/off
7. **Serializable**: แปลง object เป็นรูปแบบต่างๆ
8. **Real Animal Example**: สัตว์ที่มี abilities หลากหลายผ่าน mixins
9. **Decorator Pattern**: เพิ่ม behavior แบบ dynamic ทับซ้อน
10. **Role Pattern**: จัดกลุ่ม methods เป็น roles ที่ assign ให้ class
11. **AOP Patterns**: เพิ่ม cross-cutting concerns โดยไม่แก้โค้ดหลัก
12. **Conflict Resolution**: จัดการเมื่อ methods ชนกัน

> **เคล็ดลับ**: Mixins และ Multiple Inheritance ทำให้โค้ด Lua ยืดหยุ่นมาก แต่ควรใช้อย่างระมัดระวัง - composition มักดีกว่า inheritance ในกรณีส่วนใหญ่
