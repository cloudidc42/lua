# บทที่ 76: Advanced OOP - Mixins และ Traits

## บทนำ

ใน OOP แบบดั้งเดิม เราใช้ inheritance เพื่อแชร์ code ระหว่าง class แต่เมื่อ hierarchy ลึกมากขึ้น ปัญหาก็เริ่มปรากฏ บทนี้จะสำรวจ Mixins และ Traits ซึ่งเป็นแนวทางที่ยืดหยุ่นกว่าในการ compose behavior

---

## 1. ปัญหาของ Deep Inheritance

```lua
-- ตัวอย่างปัญหา: Diamond Problem ใน multiple inheritance
-- Animal -> FlyingAnimal -> Duck
-- Animal -> SwimmingAnimal -> Duck

local Animal = {}
Animal.__index = Animal

function Animal.new(name)
    return setmetatable({ name = name }, Animal)
end

function Animal:breathe()
    return self.name .. " กำลังหายใจ"
end

function Animal:eat()
    return self.name .. " กำลังกินอาหาร"
end

-- FlyingAnimal สืบทอดจาก Animal
local FlyingAnimal = setmetatable({}, { __index = Animal })
FlyingAnimal.__index = FlyingAnimal

function FlyingAnimal.new(name)
    local obj = Animal.new(name)
    return setmetatable(obj, FlyingAnimal)
end

function FlyingAnimal:fly()
    return self.name .. " กำลังบิน"
end

-- SwimmingAnimal สืบทอดจาก Animal
local SwimmingAnimal = setmetatable({}, { __index = Animal })
SwimmingAnimal.__index = SwimmingAnimal

function SwimmingAnimal.new(name)
    local obj = Animal.new(name)
    return setmetatable(obj, SwimmingAnimal)
end

function SwimmingAnimal:swim()
    return self.name .. " กำลังว่ายน้ำ"
end

-- Duck ต้องการทั้ง fly และ swim - แต่ Lua ไม่มี multiple inheritance โดยตรง
-- ต้องเลือก parent เดียว
local Duck = setmetatable({}, { __index = FlyingAnimal })
Duck.__index = Duck

function Duck.new(name)
    local obj = FlyingAnimal.new(name)
    return setmetatable(obj, Duck)
end

-- ต้อง copy method จาก SwimmingAnimal มาเอง - ไม่ elegant
Duck.swim = SwimmingAnimal.swim

local donald = Duck.new("Donald")
print(donald:breathe())  -- Donald กำลังหายใจ
print(donald:fly())      -- Donald กำลังบิน
print(donald:swim())     -- Donald กำลังว่ายน้ำ
```

---

## 2. ปัญหาของ God Object

```lua
-- ปัญหา: class ที่ทำหน้าที่มากเกินไป (God Object)
local GodUser = {}
GodUser.__index = GodUser

function GodUser.new(data)
    return setmetatable(data, GodUser)
end

-- ทำหน้าที่มากเกินไปใน class เดียว
function GodUser:save() 
    print("บันทึก user ลง database")
end

function GodUser:toJSON()
    local parts = {}
    for k, v in pairs(self) do
        table.insert(parts, '"' .. k .. '":"' .. tostring(v) .. '"')
    end
    return "{" .. table.concat(parts, ",") .. "}"
end

function GodUser:validate()
    return self.email ~= nil and self.name ~= nil
end

function GodUser:sendEmail(subject, body)
    print("ส่ง email ถึง " .. (self.email or "unknown"))
end

function GodUser:logActivity(action)
    print("[LOG] User " .. (self.name or "unknown") .. " ทำ: " .. action)
end

function GodUser:cache()
    print("Cache user data")
end

-- ปัญหา: เมื่อต้องการให้ Product มี serialize และ validate ด้วย
-- ต้อง copy methods ทั้งหมดมาอีกครั้ง - code duplication!
local user = GodUser.new({ name = "Alice", email = "alice@example.com" })
print(user:toJSON())
print(user:validate())
```

---

## 3. Mixin พื้นฐาน

```lua
-- Solution: แยก behavior ออกเป็น Mixin modules
local function applyMixin(target, mixin)
    for name, method in pairs(mixin) do
        if type(method) == "function" then
            target[name] = method
        end
    end
    return target
end

-- Mixin สำหรับ Serialization
local SerializableMixin = {}

function SerializableMixin:toJSON()
    local parts = {}
    for k, v in pairs(self) do
        if type(v) ~= "function" then
            if type(v) == "string" then
                table.insert(parts, string.format('"%s":"%s"', k, v))
            else
                table.insert(parts, string.format('"%s":%s', k, tostring(v)))
            end
        end
    end
    return "{" .. table.concat(parts, ",") .. "}"
end

function SerializableMixin:fromJSON(json)
    -- Simplified JSON parser
    print("Parsing JSON: " .. json)
    return self
end

function SerializableMixin:toTable()
    local result = {}
    for k, v in pairs(self) do
        if type(v) ~= "function" then
            result[k] = v
        end
    end
    return result
end

-- Mixin สำหรับ Validation
local ValidatableMixin = {}

function ValidatableMixin:validate()
    if not self._validations then return true end
    
    local errors = {}
    for field, rules in pairs(self._validations) do
        local value = self[field]
        
        if rules.required and (value == nil or value == "") then
            table.insert(errors, field .. " is required")
        end
        
        if rules.minLength and type(value) == "string" and #value < rules.minLength then
            table.insert(errors, field .. " must be at least " .. rules.minLength .. " characters")
        end
        
        if rules.maxLength and type(value) == "string" and #value > rules.maxLength then
            table.insert(errors, field .. " must be at most " .. rules.maxLength .. " characters")
        end
    end
    
    return #errors == 0, errors
end

function ValidatableMixin:addValidation(field, rules)
    if not self._validations then self._validations = {} end
    self._validations[field] = rules
    return self
end

-- สร้าง User class แบบ clean โดยใช้ Mixins
local User = {}
User.__index = User

applyMixin(User, SerializableMixin)
applyMixin(User, ValidatableMixin)

function User.new(data)
    local obj = setmetatable(data or {}, User)
    obj._validations = {
        name = { required = true, minLength = 2 },
        email = { required = true }
    }
    return obj
end

local alice = User.new({ name = "Alice", email = "alice@example.com" })
print(alice:toJSON())

local isValid, errors = alice:validate()
print("Valid:", isValid)

local bob = User.new({ name = "B", email = "" })
local isValid2, errors2 = bob:validate()
print("Valid:", isValid2)
for _, err in ipairs(errors2) do
    print("  Error:", err)
end
```

---

## 4. Observable Mixin

```lua
-- Observable/Event Mixin - เพิ่ม event system ให้กับ object ใดก็ได้
local ObservableMixin = {}

function ObservableMixin:on(event, handler)
    if not self._listeners then self._listeners = {} end
    if not self._listeners[event] then self._listeners[event] = {} end
    table.insert(self._listeners[event], handler)
    return self
end

function ObservableMixin:off(event, handler)
    if not self._listeners or not self._listeners[event] then return self end
    
    if handler then
        -- Remove specific handler
        local listeners = self._listeners[event]
        for i = #listeners, 1, -1 do
            if listeners[i] == handler then
                table.remove(listeners, i)
            end
        end
    else
        -- Remove all handlers for event
        self._listeners[event] = {}
    end
    return self
end

function ObservableMixin:emit(event, ...)
    if not self._listeners or not self._listeners[event] then return self end
    
    for _, handler in ipairs(self._listeners[event]) do
        handler(self, ...)
    end
    return self
end

function ObservableMixin:once(event, handler)
    local wrapper
    wrapper = function(...)
        handler(...)
        self:off(event, wrapper)
    end
    return self:on(event, wrapper)
end

-- ใช้ Observable Mixin กับ Model
local function applyMixin(target, mixin)
    for name, method in pairs(mixin) do
        if type(method) == "function" then
            target[name] = method
        end
    end
end

local Product = {}
Product.__index = Product
applyMixin(Product, ObservableMixin)

function Product.new(data)
    return setmetatable(data or {}, Product)
end

function Product:setPrice(newPrice)
    local oldPrice = self.price
    self.price = newPrice
    self:emit("priceChanged", oldPrice, newPrice)
end

function Product:setStock(qty)
    local oldQty = self.stock
    self.stock = qty
    self:emit("stockChanged", oldQty, qty)
    
    if qty == 0 then
        self:emit("outOfStock")
    end
end

-- ทดสอบ
local laptop = Product.new({ name = "Laptop", price = 30000, stock = 10 })

laptop:on("priceChanged", function(self, old, new)
    print(string.format("ราคา %s เปลี่ยนจาก %d เป็น %d", self.name, old, new))
end)

laptop:on("outOfStock", function(self)
    print(string.format("สินค้า %s หมดแล้ว!", self.name))
end)

laptop:setPrice(28000)
laptop:setStock(1)
laptop:setStock(0)
```

---

## 5. Cacheable Mixin

```lua
-- Cacheable Mixin - เพิ่ม caching ให้กับ expensive methods
local CacheableMixin = {}

function CacheableMixin:initCache(ttl)
    self._cache = {}
    self._cacheTTL = ttl or 300  -- default 5 minutes
    return self
end

function CacheableMixin:cacheGet(key)
    if not self._cache then return nil end
    
    local entry = self._cache[key]
    if not entry then return nil end
    
    -- Check TTL (ในตัวอย่างนี้ใช้ os.time)
    if os.time() - entry.timestamp > self._cacheTTL then
        self._cache[key] = nil
        return nil
    end
    
    return entry.value
end

function CacheableMixin:cacheSet(key, value)
    if not self._cache then self._cache = {} end
    self._cache[key] = {
        value = value,
        timestamp = os.time()
    }
    return value
end

function CacheableMixin:cacheDelete(key)
    if self._cache then
        self._cache[key] = nil
    end
end

function CacheableMixin:cacheClear()
    self._cache = {}
end

function CacheableMixin:cached(key, fn)
    local value = self:cacheGet(key)
    if value ~= nil then
        return value
    end
    value = fn()
    return self:cacheSet(key, value)
end

-- ใช้งาน
local function applyMixin(target, mixin)
    for name, method in pairs(mixin) do
        if type(method) == "function" then
            target[name] = method
        end
    end
end

local DataRepository = {}
DataRepository.__index = DataRepository
applyMixin(DataRepository, CacheableMixin)

function DataRepository.new()
    local obj = setmetatable({}, DataRepository)
    obj:initCache(60)  -- TTL 60 seconds
    return obj
end

function DataRepository:fetchUser(id)
    return self:cached("user:" .. id, function()
        print("  [DB Query] Fetching user " .. id .. " from database...")
        -- Simulate expensive database query
        return { id = id, name = "User " .. id, email = "user" .. id .. "@example.com" }
    end)
end

function DataRepository:fetchProducts()
    return self:cached("products:all", function()
        print("  [DB Query] Fetching all products from database...")
        return {
            { id = 1, name = "Laptop", price = 30000 },
            { id = 2, name = "Mouse", price = 500 },
        }
    end)
end

local repo = DataRepository.new()

print("First call:")
local user1 = repo:fetchUser(42)
print("Got:", user1.name)

print("\nSecond call (should use cache):")
local user2 = repo:fetchUser(42)
print("Got:", user2.name)

print("\nDifferent user:")
local user3 = repo:fetchUser(99)
print("Got:", user3.name)
```

---

## 6. Trait System

```lua
-- Trait System - เหมือน Mixin แต่มีการ check required methods
local Trait = {}
Trait.__index = Trait

function Trait.define(spec)
    local trait = setmetatable({}, Trait)
    trait._name = spec.name or "UnnamedTrait"
    trait._requires = spec.requires or {}      -- methods ที่ class ต้องมี
    trait._provides = {}                        -- methods ที่ trait ให้
    trait._defaults = spec.defaults or {}       -- default implementations
    
    -- Copy provided methods
    if spec.methods then
        for name, method in pairs(spec.methods) do
            trait._provides[name] = method
        end
    end
    
    return trait
end

function Trait:applyTo(class)
    -- Check required methods
    local missing = {}
    for _, required in ipairs(self._requires) do
        if not class[required] then
            table.insert(missing, required)
        end
    end
    
    if #missing > 0 then
        -- Apply defaults for missing required methods
        for _, name in ipairs(missing) do
            if self._defaults[name] then
                class[name] = self._defaults[name]
            else
                error(string.format(
                    "Trait '%s' requires method '%s' but class doesn't provide it",
                    self._name, name
                ))
            end
        end
    end
    
    -- Apply provided methods
    for name, method in pairs(self._provides) do
        if not class[name] then  -- Don't override existing methods
            class[name] = method
        end
    end
    
    -- Track applied traits
    if not class._traits then class._traits = {} end
    class._traits[self._name] = true
    
    return class
end

function Trait:hasTrait(class)
    return class._traits and class._traits[self._name] == true
end

-- สร้าง Traits ต่างๆ
local Printable = Trait.define({
    name = "Printable",
    requires = { "toString" },
    methods = {
        print = function(self)
            print(self:toString())
        end,
        println = function(self)
            print(self:toString() .. "\n")
        end
    }
})

local Comparable = Trait.define({
    name = "Comparable",
    requires = { "compareTo" },
    methods = {
        lessThan = function(self, other)
            return self:compareTo(other) < 0
        end,
        greaterThan = function(self, other)
            return self:compareTo(other) > 0
        end,
        equals = function(self, other)
            return self:compareTo(other) == 0
        end,
        between = function(self, min, max)
            return self:compareTo(min) >= 0 and self:compareTo(max) <= 0
        end
    }
})

-- สร้าง class ที่ใช้ Traits
local Temperature = {}
Temperature.__index = Temperature

function Temperature.new(celsius)
    return setmetatable({ celsius = celsius }, Temperature)
end

function Temperature:toString()
    return string.format("%.1f°C", self.celsius)
end

function Temperature:compareTo(other)
    return self.celsius - other.celsius
end

-- Apply Traits
Printable:applyTo(Temperature)
Comparable:applyTo(Temperature)

local t1 = Temperature.new(25)
local t2 = Temperature.new(30)
local t3 = Temperature.new(20)

t1:print()
print("t1 < t2:", t1:lessThan(t2))
print("t1 > t3:", t1:greaterThan(t3))
print("t1 between t3 and t2:", t1:between(t3, t2))
```

---

## 7. Conflict Resolution ใน Mixins

```lua
-- การจัดการ conflict เมื่อ Mixins มี methods ชื่อเดียวกัน
local function createMixinManager()
    local manager = {}
    
    -- Apply multiple mixins with conflict resolution
    function manager.apply(target, mixins, options)
        options = options or {}
        local conflicts = {}
        local methodSources = {}  -- track where each method came from
        
        for _, mixin in ipairs(mixins) do
            local mixinName = mixin._name or "Unknown"
            
            for name, method in pairs(mixin) do
                if type(method) == "function" and not name:match("^_") then
                    if methodSources[name] then
                        -- Conflict detected
                        table.insert(conflicts, {
                            method = name,
                            from = methodSources[name],
                            and_ = mixinName
                        })
                    else
                        methodSources[name] = mixinName
                        target[name] = method
                    end
                end
            end
        end
        
        -- Handle conflicts based on options
        if #conflicts > 0 then
            for _, conflict in ipairs(conflicts) do
                local resolution = options.resolve and options.resolve[conflict.method]
                
                if resolution == "first" then
                    -- Keep first (already applied)
                    print(string.format("[Mixin] Conflict: '%s' - keeping from '%s'",
                        conflict.method, conflict.from))
                        
                elseif resolution == "last" then
                    -- Override with last
                    local lastMixin = nil
                    for i = #mixins, 1, -1 do
                        if mixins[i][conflict.method] then
                            lastMixin = mixins[i]
                            break
                        end
                    end
                    if lastMixin then
                        target[conflict.method] = lastMixin[conflict.method]
                        print(string.format("[Mixin] Conflict: '%s' - using from '%s'",
                            conflict.method, lastMixin._name or "Unknown"))
                    end
                    
                elseif type(resolution) == "function" then
                    -- Custom resolver
                    target[conflict.method] = resolution
                    print(string.format("[Mixin] Conflict: '%s' - using custom resolver",
                        conflict.method))
                    
                else
                    -- Default: warn and keep first
                    print(string.format("[WARN] Mixin conflict: method '%s' exists in both '%s' and '%s'. Keeping '%s'",
                        conflict.method, conflict.from, conflict.and_, conflict.from))
                end
            end
        end
        
        return target
    end
    
    return manager
end

-- Test conflict resolution
local MixinA = { _name = "MixinA" }
function MixinA:greet() return "Hello from MixinA" end
function MixinA:getName() return "MixinA" end

local MixinB = { _name = "MixinB" }
function MixinB:greet() return "Hello from MixinB" end
function MixinB:getAge() return 25 end

local MyClass = {}
MyClass.__index = MyClass

function MyClass.new()
    return setmetatable({}, MyClass)
end

local mixinMgr = createMixinManager()

-- Apply with conflict resolution
mixinMgr.apply(MyClass, { MixinA, MixinB }, {
    resolve = {
        greet = "last"  -- ใช้ version จาก Mixin ที่ apply ทีหลัง
    }
})

local obj = MyClass.new()
print(obj:greet())     -- Hello from MixinB
print(obj:getName())   -- MixinA
print(obj:getAge())    -- 25
```

---

## 8. Required Methods Enforcement

```lua
-- Enforcing required methods เหมือน Interface ใน OOP languages อื่น
local Interface = {}
Interface.__index = Interface

function Interface.define(name, methods)
    return {
        _name = name,
        _methods = methods
    }
end

function Interface.implement(class, interface)
    local missing = {}
    
    for _, methodSpec in ipairs(interface._methods) do
        local methodName, methodType
        
        if type(methodSpec) == "string" then
            methodName = methodSpec
            methodType = "function"
        else
            methodName = methodSpec.name
            methodType = methodSpec.type or "function"
        end
        
        if class[methodName] == nil then
            table.insert(missing, methodName)
        elseif methodType == "function" and type(class[methodName]) ~= "function" then
            table.insert(missing, methodName .. " (must be function)")
        end
    end
    
    if #missing > 0 then
        error(string.format(
            "Class must implement interface '%s'. Missing: %s",
            interface._name,
            table.concat(missing, ", ")
        ))
    end
    
    -- Mark as implementing
    if not class._interfaces then class._interfaces = {} end
    class._interfaces[interface._name] = true
    
    return true
end

function Interface.check(obj, interface)
    local class = getmetatable(obj)
    return class and class._interfaces and class._interfaces[interface._name] == true
end

-- กำหนด Interfaces
local ISerializable = Interface.define("ISerializable", {
    "serialize",
    "deserialize"
})

local IComparable = Interface.define("IComparable", {
    "compareTo",
    "equals"
})

local IDisposable = Interface.define("IDisposable", {
    "dispose",
    "isDisposed"
})

-- Implement interfaces
local Document = {}
Document.__index = Document

function Document.new(content)
    return setmetatable({ content = content, _disposed = false }, Document)
end

function Document:serialize()
    return "DOC:" .. self.content
end

function Document:deserialize(data)
    self.content = data:gsub("^DOC:", "")
    return self
end

function Document:compareTo(other)
    return self.content < other.content and -1 or (self.content > other.content and 1 or 0)
end

function Document:equals(other)
    return self.content == other.content
end

function Document:dispose()
    self.content = nil
    self._disposed = true
end

function Document:isDisposed()
    return self._disposed
end

-- Verify implementations
local ok1 = Interface.implement(Document, ISerializable)
local ok2 = Interface.implement(Document, IComparable)
local ok3 = Interface.implement(Document, IDisposable)

print("ISerializable implemented:", ok1)
print("IComparable implemented:", ok2)
print("IDisposable implemented:", ok3)

local doc = Document.new("Hello World")
print(doc:serialize())
print("Is ISerializable:", Interface.check(doc, ISerializable))
```

---

## 9. Default Implementations ใน Traits

```lua
-- Trait พร้อม default implementations
local function defineTrait(config)
    local trait = {
        _name = config.name or "Trait",
        _requires = config.requires or {},
        _methods = config.methods or {},
        _defaults = config.defaults or {}
    }
    
    function trait:mixin(target)
        -- Check and fill required methods with defaults
        for _, req in ipairs(self._requires) do
            if not target[req] then
                if self._defaults[req] then
                    target[req] = self._defaults[req]
                    print(string.format("[Trait:%s] Using default for required method '%s'",
                        self._name, req))
                else
                    error(string.format("[Trait:%s] Missing required method: %s", self._name, req))
                end
            end
        end
        
        -- Apply trait methods
        for name, method in pairs(self._methods) do
            if not target[name] then
                target[name] = method
            end
        end
        
        return target
    end
    
    return trait
end

-- Enumerable Trait - เหมือน Ruby's Enumerable
local EnumerableTrait = defineTrait({
    name = "Enumerable",
    requires = { "each" },
    defaults = {
        -- Default each ที่ iterate over ipairs
        each = function(self, fn)
            for i, v in ipairs(self) do
                fn(v, i)
            end
        end
    },
    methods = {
        map = function(self, fn)
            local result = {}
            self:each(function(item, i)
                result[i] = fn(item)
            end)
            return result
        end,
        
        filter = function(self, predicate)
            local result = {}
            self:each(function(item)
                if predicate(item) then
                    table.insert(result, item)
                end
            end)
            return result
        end,
        
        reduce = function(self, fn, initial)
            local acc = initial
            self:each(function(item)
                if acc == nil then
                    acc = item
                else
                    acc = fn(acc, item)
                end
            end)
            return acc
        end,
        
        find = function(self, predicate)
            local found = nil
            self:each(function(item)
                if found == nil and predicate(item) then
                    found = item
                end
            end)
            return found
        end,
        
        any = function(self, predicate)
            return self:find(predicate) ~= nil
        end,
        
        all = function(self, predicate)
            local result = true
            self:each(function(item)
                if not predicate(item) then
                    result = false
                end
            end)
            return result
        end,
        
        count = function(self, predicate)
            if not predicate then
                local n = 0
                self:each(function() n = n + 1 end)
                return n
            end
            return #self:filter(predicate)
        end,
        
        toArray = function(self)
            local result = {}
            self:each(function(item)
                table.insert(result, item)
            end)
            return result
        end
    }
})

-- ใช้งาน EnumerableTrait
local NumberList = {}
NumberList.__index = NumberList
EnumerableTrait:mixin(NumberList)

function NumberList.new(...)
    return setmetatable({ ... }, NumberList)
end

local nums = NumberList.new(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

local doubled = nums:map(function(n) return n * 2 end)
print("Doubled:", table.concat(doubled, ", "))

local evens = nums:filter(function(n) return n % 2 == 0 end)
print("Evens:", table.concat(evens, ", "))

local sum = nums:reduce(function(acc, n) return acc + n end, 0)
print("Sum:", sum)

print("Any > 5:", nums:any(function(n) return n > 5 end))
print("All > 0:", nums:all(function(n) return n > 0 end))
print("Count > 5:", nums:count(function(n) return n > 5 end))
```

---

## 10. Role Composition Pattern

```lua
-- Role Composition - composing roles/behaviors ที่ซับซ้อน
local Role = {}
Role.__index = Role

function Role.new(name, config)
    local role = setmetatable({}, Role)
    role._name = name
    role._before = {}     -- before advice
    role._after = {}      -- after advice  
    role._around = {}     -- around advice
    role._methods = config.methods or {}
    role._state = config.state or {}
    return role
end

function Role:before(methodName, advice)
    if not self._before[methodName] then
        self._before[methodName] = {}
    end
    table.insert(self._before[methodName], advice)
    return self
end

function Role:after(methodName, advice)
    if not self._after[methodName] then
        self._after[methodName] = {}
    end
    table.insert(self._after[methodName], advice)
    return self
end

function Role:around(methodName, advice)
    if not self._around[methodName] then
        self._around[methodName] = {}
    end
    table.insert(self._around[methodName], advice)
    return self
end

function Role:applyTo(class)
    -- Initialize state
    if not class._roleInit then class._roleInit = {} end
    
    local initState = function(obj)
        for k, v in pairs(self._state) do
            if obj[k] == nil then
                if type(v) == "function" then
                    obj[k] = v()
                else
                    obj[k] = v
                end
            end
        end
    end
    table.insert(class._roleInit, initState)
    
    -- Apply methods
    for name, method in pairs(self._methods) do
        local existing = class[name]
        local beforeAdvices = self._before[name] or {}
        local afterAdvices = self._after[name] or {}
        
        class[name] = function(self, ...)
            -- Run before advices
            for _, advice in ipairs(beforeAdvices) do
                advice(self, ...)
            end
            
            -- Run original or new method
            local result
            if existing then
                result = existing(self, ...)
            else
                result = method(self, ...)
            end
            
            -- Run after advices
            for _, advice in ipairs(afterAdvices) do
                advice(self, result, ...)
            end
            
            return result
        end
    end
    
    return class
end

-- สร้าง Roles
local LoggingRole = Role.new("Logging", {
    state = { _log = function() return {} end },
    methods = {
        log = function(self, message)
            table.insert(self._log, {
                time = os.time(),
                message = message
            })
            print("[LOG " .. self._name .. "] " .. message)
        end,
        getLogs = function(self)
            return self._log
        end
    }
})

local TimingRole = Role.new("Timing", {
    state = { _timings = function() return {} end },
    methods = {
        startTimer = function(self, name)
            self._timings[name] = os.clock()
        end,
        stopTimer = function(self, name)
            local start = self._timings[name]
            if start then
                local elapsed = os.clock() - start
                print(string.format("[TIMER] %s took %.4f seconds", name, elapsed))
                return elapsed
            end
        end
    }
})

-- Class ที่ใช้ Roles
local Service = {}
Service.__index = Service

LoggingRole:applyTo(Service)
TimingRole:applyTo(Service)

function Service.new(name)
    local obj = setmetatable({ _name = name }, Service)
    -- Initialize role state
    if Service._roleInit then
        for _, init in ipairs(Service._roleInit) do
            init(obj)
        end
    end
    return obj
end

function Service:process(data)
    self:log("Starting process with data: " .. tostring(data))
    self:startTimer("process")
    
    -- Simulate work
    local result = data .. "_processed"
    
    self:stopTimer("process")
    self:log("Process complete: " .. result)
    return result
end

local svc = Service.new("DataService")
svc:process("test_data")

print("\nLog count:", #svc:getLogs())
```

---

## 11. Aspect-Oriented Programming (AOP)

```lua
-- AOP ใน Lua - weaving behavior ข้ามหลาย classes
local AOP = {}

-- Pointcut - กำหนดว่า advice จะ apply ที่ไหน
function AOP.pointcut(pattern)
    return {
        matches = function(className, methodName)
            local fullName = className .. "." .. methodName
            return fullName:match(pattern) ~= nil
        end
    }
end

-- Advice types
function AOP.before(pointcut, advice)
    return { type = "before", pointcut = pointcut, advice = advice }
end

function AOP.after(pointcut, advice)
    return { type = "after", pointcut = pointcut, advice = advice }
end

function AOP.around(pointcut, advice)
    return { type = "around", pointcut = pointcut, advice = advice }
end

-- Weave aspects into a class
function AOP.weave(class, className, aspects)
    for methodName, method in pairs(class) do
        if type(method) == "function" then
            local beforeAdvices = {}
            local afterAdvices = {}
            local aroundAdvices = {}
            
            for _, aspect in ipairs(aspects) do
                if aspect.pointcut.matches(className, methodName) then
                    if aspect.type == "before" then
                        table.insert(beforeAdvices, aspect.advice)
                    elseif aspect.type == "after" then
                        table.insert(afterAdvices, aspect.advice)
                    elseif aspect.type == "around" then
                        table.insert(aroundAdvices, aspect.advice)
                    end
                end
            end
            
            if #beforeAdvices > 0 or #afterAdvices > 0 or #aroundAdvices > 0 then
                local original = method
                
                class[methodName] = function(self, ...)
                    -- Before advices
                    for _, advice in ipairs(beforeAdvices) do
                        advice(className, methodName, self, ...)
                    end
                    
                    -- Around advices (wrap original)
                    local proceed = original
                    for i = #aroundAdvices, 1, -1 do
                        local outer = aroundAdvices[i]
                        local inner = proceed
                        proceed = function(...)
                            return outer(inner, className, methodName, ...)
                        end
                    end
                    
                    local result = proceed(self, ...)
                    
                    -- After advices
                    for _, advice in ipairs(afterAdvices) do
                        advice(className, methodName, self, result, ...)
                    end
                    
                    return result
                end
            end
        end
    end
    
    return class
end

-- ตัวอย่างการใช้ AOP
-- Logging Aspect - log ทุก method call
local loggingAspect = AOP.before(
    AOP.pointcut(".*%..*"),  -- matches all methods in all classes
    function(className, methodName, self, ...)
        print(string.format("[AOP:Before] %s.%s called", className, methodName))
    end
)

-- Timing Aspect - วัด performance
local timingAspect = AOP.around(
    AOP.pointcut("Service%..*"),  -- matches all methods in Service class
    function(proceed, className, methodName, self, ...)
        local start = os.clock()
        local result = proceed(self, ...)
        local elapsed = os.clock() - start
        print(string.format("[AOP:Timing] %s.%s took %.6f sec", className, methodName, elapsed))
        return result
    end
)

-- Service class
local Service = {}
Service.__index = Service

function Service.new()
    return setmetatable({}, Service)
end

function Service:calculate(n)
    local sum = 0
    for i = 1, n do sum = sum + i end
    return sum
end

function Service:greet(name)
    return "Hello, " .. name
end

-- Weave aspects
AOP.weave(Service, "Service", { loggingAspect, timingAspect })

local svc = Service.new()
print(svc:calculate(1000))
print(svc:greet("World"))
```

---

## 12. Method Weaving - Before/After/Around Advice

```lua
-- Method Weaving แบบ explicit
local function weaveMethod(obj, methodName, advices)
    local original = obj[methodName]
    if not original then
        error("Method '" .. methodName .. "' not found")
    end
    
    obj[methodName] = function(self, ...)
        -- Before advice
        if advices.before then
            advices.before(self, ...)
        end
        
        -- Around advice wraps the call
        local result
        if advices.around then
            result = advices.around(function(...)
                return original(self, ...)
            end, self, ...)
        else
            result = original(self, ...)
        end
        
        -- After advice
        if advices.after then
            advices.after(self, result, ...)
        end
        
        return result
    end
    
    return obj
end

-- ตัวอย่าง: เพิ่ม transaction behavior ให้กับ database methods
local DB = {}
DB.__index = DB

function DB.new()
    return setmetatable({ 
        data = {},
        transactionLog = {}
    }, DB)
end

function DB:insert(key, value)
    self.data[key] = value
    return true
end

function DB:get(key)
    return self.data[key]
end

function DB:delete(key)
    self.data[key] = nil
    return true
end

-- Weave transaction logging
weaveMethod(DB, "insert", {
    before = function(self, key, value)
        table.insert(self.transactionLog, {
            op = "INSERT",
            key = key,
            oldValue = self.data[key],
            timestamp = os.time()
        })
    end,
    after = function(self, result, key, value)
        if result then
            print(string.format("[TX] INSERT key='%s' value='%s' OK", key, tostring(value)))
        end
    end
})

weaveMethod(DB, "delete", {
    around = function(proceed, self, key)
        print("[TX] Starting DELETE for key=" .. key)
        local backup = self.data[key]
        local result = proceed(key)
        print("[TX] DELETE complete, backed up:", backup)
        return result
    end
})

local db = DB.new()
db:insert("user:1", "Alice")
db:insert("user:2", "Bob")
print(db:get("user:1"))
db:delete("user:1")
print("After delete:", db:get("user:1"))
```

---

## 13. Decorator Pattern vs Mixin

```lua
-- Decorator Pattern: wrap object เพื่อเพิ่ม behavior
-- Mixin: copy methods เข้า class โดยตรง

-- === Decorator Approach ===
local function LoggingDecorator(target)
    local proxy = {}
    local mt = {
        __index = function(t, key)
            local value = target[key]
            if type(value) == "function" then
                return function(_, ...)
                    print(string.format("[LOG] Calling %s", key))
                    local result = value(target, ...)
                    print(string.format("[LOG] %s returned: %s", key, tostring(result)))
                    return result
                end
            end
            return value
        end,
        __newindex = function(t, key, value)
            target[key] = value
        end
    }
    return setmetatable(proxy, mt)
end

local function CachingDecorator(target, ttl)
    local cache = {}
    ttl = ttl or 60
    
    local proxy = {}
    local mt = {
        __index = function(t, key)
            local value = target[key]
            if type(value) == "function" then
                return function(_, ...)
                    local cacheKey = key .. ":" .. table.concat({...}, ":")
                    local entry = cache[cacheKey]
                    
                    if entry and os.time() - entry.time < ttl then
                        print(string.format("[CACHE HIT] %s", cacheKey))
                        return entry.value
                    end
                    
                    print(string.format("[CACHE MISS] %s", cacheKey))
                    local result = value(target, ...)
                    cache[cacheKey] = { value = result, time = os.time() }
                    return result
                end
            end
            return value
        end
    }
    return setmetatable(proxy, mt)
end

-- Base service (ไม่มี logging หรือ caching)
local UserService = {}
UserService.__index = UserService

function UserService.new()
    return setmetatable({ users = {} }, UserService)
end

function UserService:getUser(id)
    print("  [UserService] Fetching user " .. id)
    return { id = id, name = "User " .. id }
end

function UserService:createUser(name, email)
    local id = #self.users + 1
    self.users[id] = { id = id, name = name, email = email }
    return id
end

-- Compose decorators
local service = UserService.new()
local cachedService = CachingDecorator(service, 30)
local loggedCachedService = LoggingDecorator(cachedService)

print("=== First call ===")
local user = loggedCachedService:getUser(1)
print("Got:", user.name)

print("\n=== Second call (should be cached) ===")
local user2 = loggedCachedService:getUser(1)
print("Got:", user2.name)

-- === Mixin Approach (compare) ===
print("\n=== Mixin Approach ===")
local MixinService = {}
MixinService.__index = MixinService

function MixinService.new()
    return setmetatable({ _cache = {} }, MixinService)
end

function MixinService:getUser(id)
    -- ต้อง handle caching ใน method เอง
    local cached = self._cache["user:" .. id]
    if cached then
        print("[CACHE HIT] user:" .. id)
        return cached
    end
    
    print("  [MixinService] Fetching user " .. id)
    local user = { id = id, name = "User " .. id }
    self._cache["user:" .. id] = user
    return user
end

local ms = MixinService.new()
ms:getUser(1)
ms:getUser(1)  -- cached
```

---

## 14. Real Example: Serializable Mixin

```lua
-- Serializable Mixin แบบ Complete
local SerializableMixin = {}

-- JSON Serialization
function SerializableMixin:toJSON(options)
    options = options or {}
    local indent = options.indent or 0
    local space = options.space or ""
    
    local function serialize(value, depth)
        depth = depth or 0
        local t = type(value)
        
        if t == "nil" then
            return "null"
        elseif t == "boolean" then
            return tostring(value)
        elseif t == "number" then
            if value ~= value then return "null" end  -- NaN
            return tostring(value)
        elseif t == "string" then
            return '"' .. value:gsub('"', '\\"'):gsub('\n', '\\n') .. '"'
        elseif t == "table" then
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
                    table.insert(items, serialize(v, depth + 1))
                end
                return "[" .. table.concat(items, ",") .. "]"
            else
                local pairs_list = {}
                for k, v in pairs(value) do
                    if type(k) == "string" and type(v) ~= "function" then
                        table.insert(pairs_list, '"' .. k .. '":' .. serialize(v, depth + 1))
                    end
                end
                table.sort(pairs_list)
                return "{" .. table.concat(pairs_list, ",") .. "}"
            end
        else
            return '"[' .. t .. ']"'
        end
    end
    
    local data = {}
    for k, v in pairs(self) do
        if type(v) ~= "function" and not k:match("^_") then
            data[k] = v
        end
    end
    
    return serialize(data)
end

function SerializableMixin:toCSV(fields)
    local headers = fields or {}
    
    if #headers == 0 then
        for k, v in pairs(self) do
            if type(v) ~= "function" and not k:match("^_") then
                table.insert(headers, k)
            end
        end
        table.sort(headers)
    end
    
    local values = {}
    for _, field in ipairs(headers) do
        local v = self[field]
        if type(v) == "string" then
            table.insert(values, '"' .. v:gsub('"', '""') .. '"')
        else
            table.insert(values, tostring(v or ""))
        end
    end
    
    return table.concat(headers, ",") .. "\n" .. table.concat(values, ",")
end

function SerializableMixin:clone()
    local copy = {}
    for k, v in pairs(self) do
        if type(v) ~= "function" then
            copy[k] = v
        end
    end
    return setmetatable(copy, getmetatable(self))
end

function SerializableMixin:merge(other)
    for k, v in pairs(other) do
        if type(v) ~= "function" then
            self[k] = v
        end
    end
    return self
end

-- Apply to class
local function applyMixin(target, mixin)
    for name, method in pairs(mixin) do
        if type(method) == "function" then
            target[name] = method
        end
    end
end

local Order = {}
Order.__index = Order
applyMixin(Order, SerializableMixin)

function Order.new(data)
    return setmetatable(data, Order)
end

local order = Order.new({
    id = "ORD-001",
    customer = "Alice",
    items = { "Laptop", "Mouse", "Keyboard" },
    total = 35000,
    status = "pending"
})

print("JSON:")
print(order:toJSON())

print("\nCSV:")
print(order:toCSV({ "id", "customer", "total", "status" }))

local order2 = order:clone()
order2:merge({ status = "completed", total = 34500 })
print("\nCloned & Merged:")
print(order2:toJSON())
```

---

## 15. Validatable Mixin แบบ Complete

```lua
-- Validatable Mixin พร้อม rule system ที่สมบูรณ์
local ValidatableMixin = {}

-- Built-in validators
local validators = {
    required = function(value, _)
        return value ~= nil and value ~= "", "is required"
    end,
    
    minLength = function(value, min)
        if type(value) ~= "string" then return true end
        return #value >= min, "must be at least " .. min .. " characters"
    end,
    
    maxLength = function(value, max)
        if type(value) ~= "string" then return true end
        return #value <= max, "must be at most " .. max .. " characters"
    end,
    
    min = function(value, min)
        if type(value) ~= "number" then return true end
        return value >= min, "must be at least " .. min
    end,
    
    max = function(value, max)
        if type(value) ~= "number" then return true end
        return value <= max, "must be at most " .. max
    end,
    
    pattern = function(value, pat)
        if type(value) ~= "string" then return true end
        return value:match(pat) ~= nil, "must match pattern " .. pat
    end,
    
    email = function(value, _)
        if type(value) ~= "string" then return true end
        local valid = value:match("^[^@]+@[^@]+%.[^@]+$") ~= nil
        return valid, "must be a valid email address"
    end,
    
    oneOf = function(value, options)
        for _, opt in ipairs(options) do
            if value == opt then return true end
        end
        return false, "must be one of: " .. table.concat(options, ", ")
    end,
    
    custom = function(value, fn)
        return fn(value)
    end
}

function ValidatableMixin:validates(field, rules)
    if not self._schema then self._schema = {} end
    self._schema[field] = rules
    return self
end

function ValidatableMixin:validate()
    if not self._schema then return true, {} end
    
    local errors = {}
    
    for field, rules in pairs(self._schema) do
        local value = self[field]
        
        for ruleName, ruleParam in pairs(rules) do
            local validator = validators[ruleName]
            if validator then
                local ok, message = validator(value, ruleParam)
                if not ok then
                    if not errors[field] then errors[field] = {} end
                    table.insert(errors[field], field .. " " .. message)
                end
            end
        end
    end
    
    -- Check if any errors
    local hasErrors = next(errors) ~= nil
    return not hasErrors, errors
end

function ValidatableMixin:isValid()
    local ok, _ = self:validate()
    return ok
end

function ValidatableMixin:errors()
    local _, errs = self:validate()
    return errs
end

function ValidatableMixin:assertValid()
    local ok, errs = self:validate()
    if not ok then
        local messages = {}
        for field, fieldErrors in pairs(errs) do
            for _, msg in ipairs(fieldErrors) do
                table.insert(messages, msg)
            end
        end
        error("Validation failed: " .. table.concat(messages, "; "))
    end
    return self
end

-- Apply and use
local function applyMixin(target, mixin)
    for name, method in pairs(mixin) do
        if type(method) == "function" then
            target[name] = method
        end
    end
end

local User = {}
User.__index = User
applyMixin(User, ValidatableMixin)

function User.new(data)
    local obj = setmetatable(data or {}, User)
    obj:validates("name", { required = true, minLength = 2, maxLength = 50 })
    obj:validates("email", { required = true, email = true })
    obj:validates("age", { min = 0, max = 150 })
    obj:validates("role", { oneOf = { "admin", "user", "moderator" } })
    return obj
end

-- Valid user
local alice = User.new({
    name = "Alice",
    email = "alice@example.com",
    age = 25,
    role = "admin"
})

print("Alice is valid:", alice:isValid())

-- Invalid user
local bad = User.new({
    name = "X",  -- too short
    email = "not-an-email",
    age = 200,   -- too high
    role = "superuser"  -- not in list
})

local ok, errs = bad:validate()
print("\nBad user is valid:", ok)
print("Errors:")
for field, fieldErrors in pairs(errs) do
    for _, err in ipairs(fieldErrors) do
        print("  -", err)
    end
end
```

---

## 16. Building a Flexible OOP Framework

```lua
-- Framework OOP สมบูรณ์แบบที่รวม Mixin + Trait + Interface
local OOPFramework = {}

-- Class factory
function OOPFramework.class(config)
    config = config or {}
    
    local class = {}
    class.__index = class
    class._name = config.name or "AnonymousClass"
    class._mixins = {}
    class._traits = {}
    class._interfaces = {}
    
    -- Inherit from parent
    if config.extends then
        setmetatable(class, { __index = config.extends })
    end
    
    -- Apply mixins
    if config.mixins then
        for _, mixin in ipairs(config.mixins) do
            for name, method in pairs(mixin) do
                if type(method) == "function" and not name:match("^_") then
                    if not class[name] then
                        class[name] = method
                    end
                end
            end
            table.insert(class._mixins, mixin._name or "unknown")
        end
    end
    
    -- Apply traits
    if config.traits then
        for _, trait in ipairs(config.traits) do
            if trait.applyTo then
                trait:applyTo(class)
            end
            table.insert(class._traits, trait._name or "unknown")
        end
    end
    
    -- new() constructor
    function class.new(...)
        local obj = setmetatable({}, class)
        if obj.initialize then
            obj:initialize(...)
        end
        return obj
    end
    
    -- instanceof check
    function class.isInstance(obj)
        return getmetatable(obj) == class
    end
    
    -- Class info
    function class:getClass()
        return class._name
    end
    
    function class:hasMixin(name)
        for _, m in ipairs(class._mixins) do
            if m == name then return true end
        end
        return false
    end
    
    return class
end

-- === Define Mixins ===
local Timestampable = {
    _name = "Timestampable",
    
    setCreatedAt = function(self)
        self.createdAt = os.time()
    end,
    
    setUpdatedAt = function(self)
        self.updatedAt = os.time()
    end,
    
    touch = function(self)
        self:setUpdatedAt()
    end
}

local SoftDeletable = {
    _name = "SoftDeletable",
    
    softDelete = function(self)
        self.deletedAt = os.time()
        self.isDeleted = true
    end,
    
    restore = function(self)
        self.deletedAt = nil
        self.isDeleted = false
    end,
    
    isActive = function(self)
        return not self.isDeleted
    end
}

-- สร้าง Entity class ด้วย framework
local Entity = OOPFramework.class({
    name = "Entity",
    mixins = { Timestampable, SoftDeletable }
})

function Entity:initialize(data)
    if data then
        for k, v in pairs(data) do
            self[k] = v
        end
    end
    self:setCreatedAt()
end

function Entity:toString()
    return string.format("<%s id=%s>", self:getClass(), tostring(self.id))
end

-- สร้าง subclass
local Article = OOPFramework.class({
    name = "Article",
    extends = Entity
})

function Article:initialize(data)
    Entity.initialize(self, data)
end

function Article:publish()
    self.publishedAt = os.time()
    self.status = "published"
    self:touch()
    return self
end

-- Test
local article = Article.new({
    id = 1,
    title = "Hello World",
    content = "This is my first article",
    status = "draft"
})

print(article:toString())
print("Has Timestampable:", article:hasMixin("Timestampable"))
print("Has SoftDeletable:", article:hasMixin("SoftDeletable"))
print("Is active:", article:isActive())

article:publish()
print("Status:", article.status)

article:softDelete()
print("Is active after delete:", article:isActive())

article:restore()
print("Is active after restore:", article:isActive())
```

---

## 17. Observer Pattern ด้วย Mixin

```lua
-- Observer Pattern เป็น Mixin ที่สมบูรณ์
local ObserverMixin = {
    _name = "ObserverMixin"
}

function ObserverMixin:initObservable()
    self._observers = {}
    self._eventBuffer = {}
    self._bufferEnabled = false
    return self
end

function ObserverMixin:subscribe(event, observer, options)
    options = options or {}
    
    if not self._observers then self:initObservable() end
    if not self._observers[event] then self._observers[event] = {} end
    
    local subscription = {
        observer = observer,
        once = options.once or false,
        priority = options.priority or 0
    }
    
    table.insert(self._observers[event], subscription)
    
    -- Sort by priority (higher priority first)
    table.sort(self._observers[event], function(a, b)
        return a.priority > b.priority
    end)
    
    -- Return unsubscribe function
    return function()
        if self._observers[event] then
            for i, sub in ipairs(self._observers[event]) do
                if sub == subscription then
                    table.remove(self._observers[event], i)
                    return
                end
            end
        end
    end
end

function ObserverMixin:notify(event, data)
    if not self._observers then return self end
    
    -- Buffer events if enabled
    if self._bufferEnabled then
        table.insert(self._eventBuffer, { event = event, data = data })
        return self
    end
    
    local observers = self._observers[event]
    if not observers then return self end
    
    local toRemove = {}
    
    for i, sub in ipairs(observers) do
        sub.observer(data, event, self)
        if sub.once then
            table.insert(toRemove, i)
        end
    end
    
    -- Remove 'once' subscribers
    for i = #toRemove, 1, -1 do
        table.remove(observers, toRemove[i])
    end
    
    return self
end

function ObserverMixin:bufferEvents()
    self._bufferEnabled = true
    self._eventBuffer = {}
    return self
end

function ObserverMixin:flushEvents()
    self._bufferEnabled = false
    for _, buffered in ipairs(self._eventBuffer) do
        self:notify(buffered.event, buffered.data)
    end
    self._eventBuffer = {}
    return self
end

-- Apply to a shopping cart
local function applyMixin(target, mixin)
    for name, method in pairs(mixin) do
        if type(method) == "function" then
            target[name] = method
        end
    end
end

local Cart = {}
Cart.__index = Cart
applyMixin(Cart, ObserverMixin)

function Cart.new()
    local obj = setmetatable({ items = {}, total = 0 }, Cart)
    obj:initObservable()
    return obj
end

function Cart:addItem(item)
    table.insert(self.items, item)
    self.total = self.total + item.price
    self:notify("itemAdded", item)
    self:notify("totalChanged", self.total)
end

function Cart:removeItem(index)
    local item = self.items[index]
    if item then
        table.remove(self.items, index)
        self.total = self.total - item.price
        self:notify("itemRemoved", item)
        self:notify("totalChanged", self.total)
    end
end

function Cart:clear()
    self.items = {}
    self.total = 0
    self:notify("cleared")
    self:notify("totalChanged", 0)
end

-- Subscribe to events
local cart = Cart.new()

local unsubTotal = cart:subscribe("totalChanged", function(total)
    print(string.format("  Cart total: ฿%d", total))
end)

cart:subscribe("itemAdded", function(item)
    print(string.format("  Added: %s (฿%d)", item.name, item.price))
end)

-- Once subscriber
cart:subscribe("cleared", function()
    print("  Cart was cleared!")
end, { once = true })

print("Adding items:")
cart:addItem({ name = "Laptop", price = 30000 })
cart:addItem({ name = "Mouse", price = 500 })
cart:addItem({ name = "Keyboard", price = 1500 })

print("\nRemoving item:")
cart:removeItem(2)

print("\nClearing:")
cart:clear()
cart:clear()  -- once subscriber won't fire again
```

---

## 18. Proxy Mixin สำหรับ Lazy Loading

```lua
-- Lazy Loading ด้วย Proxy Mixin
local LazyLoadMixin = {
    _name = "LazyLoadMixin"
}

function LazyLoadMixin:lazyLoad(propName, loader)
    if not self._lazyLoaders then self._lazyLoaders = {} end
    self._lazyLoaders[propName] = loader
    
    -- Install __index to handle lazy loading
    local mt = getmetatable(self)
    if not mt then
        mt = {}
        setmetatable(self, mt)
    end
    
    local originalIndex = mt.__index
    mt.__index = function(t, key)
        -- Check lazy loaders
        if t._lazyLoaders and t._lazyLoaders[key] then
            local value = t._lazyLoaders[key](t)
            rawset(t, key, value)  -- cache the loaded value
            return value
        end
        
        -- Fall back to original index
        if type(originalIndex) == "function" then
            return originalIndex(t, key)
        elseif originalIndex then
            return originalIndex[key]
        end
    end
    
    return self
end

-- ใช้งาน
local BlogPost = {}
BlogPost.__index = BlogPost

function BlogPost.new(id, title)
    local obj = setmetatable({ id = id, title = title }, BlogPost)
    
    -- Lazy load comments (expensive operation)
    obj:lazyLoad("comments", function(self)
        print(string.format("  [Lazy] Loading comments for post %d...", self.id))
        return {
            { author = "Alice", text = "Great post!" },
            { author = "Bob", text = "Thanks for sharing!" }
        }
    end)
    
    -- Lazy load author
    obj:lazyLoad("author", function(self)
        print(string.format("  [Lazy] Loading author for post %d...", self.id))
        return { name = "John Doe", email = "john@example.com" }
    end)
    
    -- Lazy load related posts
    obj:lazyLoad("related", function(self)
        print(string.format("  [Lazy] Loading related posts for %d...", self.id))
        return {
            { id = 2, title = "Related Post 1" },
            { id = 3, title = "Related Post 2" }
        }
    end)
    
    return obj
end

-- Apply mixin
for name, method in pairs(LazyLoadMixin) do
    if type(method) == "function" and not name:match("^_") then
        BlogPost[name] = method
    end
end

print("Creating post (no loading yet)")
local post = BlogPost.new(1, "Hello Lua")
print("Title:", post.title)  -- No loading

print("\nAccessing author (triggers lazy load):")
print("Author:", post.author.name)

print("\nAccessing author again (cached, no reload):")
print("Author:", post.author.name)

print("\nAccessing comments:")
for _, comment in ipairs(post.comments) do
    print(string.format("  %s: %s", comment.author, comment.text))
end
```

---

## 19. Composing Multiple Mixins

```lua
-- การ compose หลาย Mixins เข้าด้วยกัน
local function compose(...)
    local mixins = { ... }
    
    return function(class)
        for _, mixin in ipairs(mixins) do
            if type(mixin) == "function" then
                mixin(class)
            elseif type(mixin) == "table" then
                for name, method in pairs(mixin) do
                    if type(method) == "function" and not name:match("^_") then
                        if not class[name] then
                            class[name] = method
                        end
                    end
                end
            end
        end
        return class
    end
end

-- Mixin functions (alternative style using functions)
local function withLogging(class)
    function class:log(level, message)
        print(string.format("[%s] [%s] %s", os.date("%H:%M:%S"), level:upper(), message))
    end
    
    function class:debug(msg) self:log("debug", msg) end
    function class:info(msg) self:log("info", msg) end
    function class:warn(msg) self:log("warn", msg) end
    function class:error(msg) self:log("error", msg) end
    
    return class
end

local function withRetry(class)
    function class:retry(fn, maxAttempts, delay)
        maxAttempts = maxAttempts or 3
        delay = delay or 0
        
        for attempt = 1, maxAttempts do
            local ok, result = pcall(fn, self)
            if ok then
                return result
            end
            
            if attempt < maxAttempts then
                self:warn(string.format("Attempt %d failed: %s. Retrying...", attempt, result))
            else
                error(string.format("All %d attempts failed. Last error: %s", maxAttempts, result))
            end
        end
    end
    
    return class
end

local function withMetrics(class)
    function class:initMetrics()
        self._metrics = {
            calls = {},
            errors = {},
            totalTime = 0
        }
    end
    
    function class:recordCall(name, duration, success)
        if not self._metrics then self:initMetrics() end
        
        if not self._metrics.calls[name] then
            self._metrics.calls[name] = { count = 0, totalTime = 0, errors = 0 }
        end
        
        local m = self._metrics.calls[name]
        m.count = m.count + 1
        m.totalTime = m.totalTime + duration
        if not success then m.errors = m.errors + 1 end
    end
    
    function class:getMetrics()
        return self._metrics
    end
    
    return class
end

-- Compose all mixins
local HttpClient = {}
HttpClient.__index = HttpClient

compose(withLogging, withRetry, withMetrics)(HttpClient)

function HttpClient.new(baseURL)
    local obj = setmetatable({ baseURL = baseURL }, HttpClient)
    obj:initMetrics()
    return obj
end

function HttpClient:get(path)
    local url = self.baseURL .. path
    self:info("GET " .. url)
    
    local start = os.clock()
    local success = true
    
    -- Simulate HTTP call
    local result
    self:retry(function()
        result = { status = 200, body = "Response from " .. url }
    end, 3)
    
    local duration = os.clock() - start
    self:recordCall("GET", duration, success)
    
    return result
end

local client = HttpClient.new("https://api.example.com")
local response = client:get("/users")
print("Response:", response.body)

local metrics = client:getMetrics()
print("GET calls:", metrics.calls["GET"].count)
```

---

## 20. Mixin Registry และ Auto-Discovery

```lua
-- Mixin Registry สำหรับ manage mixins ในระบบขนาดใหญ่
local MixinRegistry = {}

local _registry = {}
local _dependencies = {}

function MixinRegistry.register(name, mixin, deps)
    _registry[name] = mixin
    _dependencies[name] = deps or {}
    mixin._name = name
end

function MixinRegistry.get(name)
    return _registry[name]
end

function MixinRegistry.apply(class, names)
    -- Resolve dependency order using topological sort
    local resolved = {}
    local visited = {}
    
    local function resolve(name)
        if visited[name] then return end
        visited[name] = true
        
        -- First resolve dependencies
        for _, dep in ipairs(_dependencies[name] or {}) do
            if not _registry[dep] then
                error("Mixin '" .. name .. "' depends on '" .. dep .. "' which is not registered")
            end
            resolve(dep)
        end
        
        table.insert(resolved, name)
    end
    
    for _, name in ipairs(names) do
        resolve(name)
    end
    
    -- Apply in resolved order
    for _, name in ipairs(resolved) do
        local mixin = _registry[name]
        if mixin then
            for k, v in pairs(mixin) do
                if type(v) == "function" and not k:match("^_") then
                    if not class[k] then
                        class[k] = v
                    end
                end
            end
        end
    end
    
    -- Track applied mixins
    if not class._appliedMixins then class._appliedMixins = {} end
    for _, name in ipairs(resolved) do
        class._appliedMixins[name] = true
    end
    
    return class
end

function MixinRegistry.hasMixin(class, name)
    return class._appliedMixins and class._appliedMixins[name] == true
end

-- Register mixins
MixinRegistry.register("Events", {
    on = function(self, event, fn)
        if not self._events then self._events = {} end
        if not self._events[event] then self._events[event] = {} end
        table.insert(self._events[event], fn)
    end,
    emit = function(self, event, ...)
        if self._events and self._events[event] then
            for _, fn in ipairs(self._events[event]) do
                fn(...)
            end
        end
    end
})

MixinRegistry.register("Lifecycle", {
    initialize = function(self, data)
        if data then
            for k, v in pairs(data) do self[k] = v end
        end
        self:emit("initialized", self)
    end,
    destroy = function(self)
        self:emit("beforeDestroy", self)
        -- cleanup
        self._events = {}
        self:emit("destroyed", self)
    end
}, { "Events" })  -- depends on Events

MixinRegistry.register("Persistence", {
    save = function(self)
        self:emit("beforeSave", self)
        print("[Persistence] Saving " .. (self._name or "entity"))
        self._saved = true
        self._savedAt = os.time()
        self:emit("afterSave", self)
    end,
    load = function(self, id)
        print("[Persistence] Loading id=" .. id)
        return self
    end
}, { "Events", "Lifecycle" })  -- depends on Events and Lifecycle

-- สร้าง class ที่ใช้ registry
local Widget = {}
Widget.__index = Widget

MixinRegistry.apply(Widget, { "Persistence", "Lifecycle" })

function Widget.new(data)
    local obj = setmetatable({}, Widget)
    obj:initialize(data)
    return obj
end

local w = Widget.new({ _name = "Button", label = "Click Me" })
w:on("afterSave", function(self)
    print("Widget saved successfully:", self._name)
end)

w:save()
print("Has Events mixin:", MixinRegistry.hasMixin(Widget, "Events"))
print("Has Persistence mixin:", MixinRegistry.hasMixin(Widget, "Persistence"))
```

---

## สรุป

บทนี้ครอบคลุม:

1. **ปัญหาของ Deep Inheritance** - Diamond problem, God Object
2. **Mixin พื้นฐาน** - การ copy methods ระหว่าง classes  
3. **Observable Mixin** - Event system
4. **Cacheable Mixin** - Caching layer
5. **Trait System** - Mixin พร้อม required methods check
6. **Conflict Resolution** - จัดการเมื่อ Mixins ชนกัน
7. **Required Methods Enforcement** - Interface-like system
8. **Default Implementations** - Fallback methods
9. **Role Composition** - Composing roles/behaviors
10. **Aspect-Oriented Programming** - Cross-cutting concerns
11. **Method Weaving** - Before/After/Around advice
12. **Decorator vs Mixin** - เปรียบเทียบ 2 แนวทาง
13. **Serializable Mixin** - Complete serialization
14. **Validatable Mixin** - Validation rule system
15. **OOP Framework** - Framework สมบูรณ์แบบ
16. **Observer Pattern** - Event-driven programming
17. **Lazy Loading** - Proxy-based lazy loading
18. **Composing Mixins** - Function-based composition
19. **Mixin Registry** - Auto-discovery system

แนวคิดหลัก: **Favor composition over inheritance** - ใช้ Mixins และ Traits แทน deep class hierarchies เพื่อ code ที่ยืดหยุ่นและ reusable มากขึ้น
