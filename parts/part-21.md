# บทที่ 21: Object-Oriented Programming (OOP) ใน Lua

## บทนำ

Object-Oriented Programming (OOP) คือรูปแบบการเขียนโปรแกรมที่จัดระเบียบโค้ดเป็น objects ที่มีทั้ง data (fields/properties) และ behavior (methods) Lua ไม่มี class built-in เหมือนภาษาอื่น แต่เราสามารถสร้าง OOP system ที่ทรงพลังได้ด้วย tables และ metatables

แนวคิดหลักของ OOP:
- **Class**: แบบพิมพ์ (template) สำหรับสร้าง objects
- **Object/Instance**: สิ่งที่สร้างจาก class
- **Encapsulation**: การซ่อน implementation details
- **Inheritance**: การสืบทอด properties/methods จาก class อื่น
- **Polymorphism**: การที่ objects ต่าง types สามารถตอบสนองต่อ interface เดียวกันได้

---

## 21.1 Class System จาก Scratch

### Class พื้นฐาน

```lua
-- ตัวอย่างที่ 1: Class อย่างง่าย
local Animal = {}
Animal.__index = Animal

-- Constructor
function Animal.new(name, sound)
    local self = setmetatable({}, Animal)
    self.name = name
    self.sound = sound
    return self
end

-- Instance methods
function Animal:speak()
    return self.name .. " says " .. self.sound
end

function Animal:getName()
    return self.name
end

function Animal:setName(name)
    self.name = name
end

function Animal:__tostring()
    return string.format("Animal(%s)", self.name)
end

-- ใช้งาน
local cat = Animal.new("Whiskers", "Meow")
local dog = Animal.new("Rex", "Woof")

print(cat:speak())     -- Whiskers says Meow
print(dog:speak())     -- Rex says Woof
print(tostring(cat))   -- Animal(Whiskers)
```

```lua
-- ตัวอย่างที่ 2: Class factory function
local function createClass(base)
    local cls = {}
    cls.__index = cls
    
    if base then
        setmetatable(cls, {__index = base})
    end
    
    function cls.new(...)
        local instance = setmetatable({}, cls)
        if instance.init then
            instance:init(...)
        end
        return instance
    end
    
    function cls:isA(klass)
        local mt = getmetatable(self)
        while mt do
            if mt == klass then return true end
            local parent = getmetatable(mt)
            mt = parent and parent.__index
        end
        return false
    end
    
    return cls
end

-- สร้าง classes
local Shape = createClass()

function Shape:init(color)
    self.color = color or "black"
end

function Shape:getColor()
    return self.color
end

function Shape:area()
    return 0  -- override ใน subclasses
end

function Shape:__tostring()
    return string.format("Shape(color=%s, area=%.2f)", self.color, self:area())
end

-- Test
local s = Shape.new("red")
print(s:getColor())  -- red
print(s:area())      -- 0
print(tostring(s))   -- Shape(color=red, area=0.00)
```

---

## 21.2 Constructor Pattern

```lua
-- ตัวอย่างที่ 3: Constructor patterns ต่างๆ
local Person = {}
Person.__index = Person

-- Pattern 1: Simple constructor
function Person.new(name, age)
    return setmetatable({
        name = name,
        age = age,
        _id = math.random(10000, 99999)
    }, Person)
end

-- Pattern 2: Constructor with validation
function Person.create(config)
    assert(type(config) == "table", "config must be a table")
    assert(type(config.name) == "string" and #config.name > 0,
        "name must be non-empty string")
    assert(type(config.age) == "number" and config.age >= 0 and config.age <= 150,
        "age must be between 0 and 150")
    
    return Person.new(config.name, config.age)
end

-- Pattern 3: Copy constructor
function Person.copy(other)
    return Person.new(other.name, other.age)
end

function Person:__tostring()
    return string.format("Person{name=%q, age=%d, id=%d}",
        self.name, self.age, self._id)
end

function Person:greet()
    return string.format("Hi, I'm %s and I'm %d years old.", self.name, self.age)
end

function Person:birthday()
    self.age = self.age + 1
    return self
end

-- Test
local p1 = Person.new("Alice", 30)
print(tostring(p1))
print(p1:greet())
p1:birthday():birthday()
print("After 2 birthdays:", p1.age)

local p2 = Person.create({name = "Bob", age = 25})
print(tostring(p2))

local p3 = Person.copy(p1)
p3.name = "Charlie"
print(tostring(p1))  -- ไม่เปลี่ยน
print(tostring(p3))  -- Charlie
```

---

## 21.3 Instance Methods กับ Colon Syntax

```lua
-- ตัวอย่างที่ 4: Colon syntax และ self
local Counter = {}
Counter.__index = Counter

function Counter.new(start, step)
    return setmetatable({
        _value = start or 0,
        _step = step or 1,
        _history = {}
    }, Counter)
end

-- Colon syntax: function Counter:method() เหมือนกับ
-- function Counter.method(self)
function Counter:increment()
    self._history[#self._history + 1] = self._value
    self._value = self._value + self._step
    return self  -- method chaining
end

function Counter:decrement()
    self._history[#self._history + 1] = self._value
    self._value = self._value - self._step
    return self
end

function Counter:reset()
    self._history[#self._history + 1] = self._value
    self._value = 0
    return self
end

function Counter:getValue()
    return self._value
end

function Counter:getHistory()
    return {table.unpack(self._history)}
end

function Counter:setStep(step)
    self._step = step
    return self
end

function Counter:__tostring()
    return string.format("Counter(value=%d, step=%d)", self._value, self._step)
end

-- Method chaining
local c = Counter.new(0, 1)
c:increment():increment():increment():setStep(5):increment():increment()

print(tostring(c))  -- Counter(value=13, step=5)
print("History:", table.concat(c:getHistory(), ", "))
```

---

## 21.4 Inheritance

```lua
-- ตัวอย่างที่ 5: Inheritance พื้นฐาน
-- Base class
local Vehicle = {}
Vehicle.__index = Vehicle

function Vehicle.new(make, model, year)
    return setmetatable({
        make = make,
        model = model,
        year = year,
        speed = 0,
        fuel = 100
    }, Vehicle)
end

function Vehicle:accelerate(amount)
    self.speed = self.speed + amount
    self.fuel = self.fuel - (amount * 0.1)
    return self
end

function Vehicle:brake(amount)
    self.speed = math.max(0, self.speed - amount)
    return self
end

function Vehicle:refuel(amount)
    self.fuel = math.min(100, self.fuel + amount)
    return self
end

function Vehicle:describe()
    return string.format("%d %s %s", self.year, self.make, self.model)
end

function Vehicle:status()
    return string.format("Speed: %d km/h, Fuel: %.1f%%", self.speed, self.fuel)
end

function Vehicle:__tostring()
    return self:describe() .. " [" .. self:status() .. "]"
end

-- Subclass: Car
local Car = setmetatable({}, {__index = Vehicle})
Car.__index = Car

function Car.new(make, model, year, doors)
    local self = Vehicle.new(make, model, year)
    self.doors = doors or 4
    self.gear = 1
    return setmetatable(self, Car)
end

function Car:shiftGear(gear)
    self.gear = math.max(1, math.min(6, gear))
    return self
end

function Car:honk()
    return self:describe() .. ": Beep beep!"
end

function Car:describe()
    return Vehicle.describe(self) ..
        string.format(" (%d-door)", self.doors)
end

-- Subclass: ElectricCar
local ElectricCar = setmetatable({}, {__index = Car})
ElectricCar.__index = ElectricCar

function ElectricCar.new(make, model, year, range)
    local self = Car.new(make, model, year)
    self.batteryRange = range or 300  -- km
    self.charge = 100
    return setmetatable(self, ElectricCar)
end

function ElectricCar:accelerate(amount)
    -- Override: use battery instead of fuel
    self.speed = self.speed + amount
    self.charge = self.charge - (amount * 0.05)
    return self
end

function ElectricCar:recharge(amount)
    self.charge = math.min(100, self.charge + amount)
    return self
end

function ElectricCar:status()
    return string.format("Speed: %d km/h, Battery: %.1f%%",
        self.speed, self.charge)
end

function ElectricCar:describe()
    return Car.describe(self) .. " [Electric]"
end

-- Test
local sedan = Car.new("Toyota", "Camry", 2023, 4)
sedan:accelerate(60):shiftGear(4)
print(tostring(sedan))
print(sedan:honk())

local tesla = ElectricCar.new("Tesla", "Model 3", 2024, 500)
tesla:accelerate(100)
print(tostring(tesla))
print(tesla:describe())
```

---

## 21.5 isinstance Check

```lua
-- ตัวอย่างที่ 6: Type checking / isinstance
local function isinstance(obj, class)
    local mt = getmetatable(obj)
    while mt do
        if mt == class then return true end
        -- Traverse inheritance chain
        local parent_mt = getmetatable(mt)
        if parent_mt then
            mt = parent_mt.__index
        else
            break
        end
    end
    return false
end

-- ดีกว่า: เพิ่ม classname
local function createClass(name, base)
    local cls = {}
    cls.__index = cls
    cls.__name = name
    cls._isClass = true
    
    if base then
        setmetatable(cls, {__index = base, __name = "class:" .. name})
    end
    
    function cls.new(...)
        local instance = setmetatable({_class = cls}, cls)
        if instance.init then
            instance:init(...)
        end
        return instance
    end
    
    function cls:instanceof(klass)
        local mt = getmetatable(self)
        while mt do
            if mt == klass then return true end
            local parent = getmetatable(mt)
            if parent then
                mt = parent.__index
            else
                break
            end
        end
        return false
    end
    
    function cls:className()
        return cls.__name
    end
    
    cls.__tostring = function(self)
        return cls.__name .. "{}"
    end
    
    return cls
end

-- สร้าง class hierarchy
local Base = createClass("Base")
function Base:init(x)
    self.x = x
end
function Base:getX() return self.x end

local Middle = createClass("Middle", Base)
function Middle:init(x, y)
    Base.init(self, x)  -- call super
    self.y = y
end
function Middle:getY() return self.y end

local Derived = createClass("Derived", Middle)
function Derived:init(x, y, z)
    Middle.init(self, x, y)  -- call super
    self.z = z
end
function Derived:getZ() return self.z end

-- Test
local d = Derived.new(1, 2, 3)
print("Is Derived:", d:instanceof(Derived))    -- true
print("Is Middle:", d:instanceof(Middle))       -- true
print("Is Base:", d:instanceof(Base))           -- true
print("Class name:", d:className())             -- Derived
print("x:", d:getX(), "y:", d:getY(), "z:", d:getZ())
```

---

## 21.6 Class Methods vs Instance Methods

```lua
-- ตัวอย่างที่ 7: Class methods และ Instance methods
local Database = {}
Database.__index = Database

-- Class-level state
Database._connections = {}
Database._connectionCount = 0
Database._maxConnections = 5

-- Class methods (เรียกด้วย Database.method())
function Database.getConnectionCount()
    return Database._connectionCount
end

function Database.setMaxConnections(n)
    Database._maxConnections = n
end

function Database.getAllConnections()
    return {table.unpack(Database._connections)}
end

-- Constructor
function Database.new(host, port, dbname)
    if Database._connectionCount >= Database._maxConnections then
        error("Max connections reached: " .. Database._maxConnections)
    end
    
    local self = setmetatable({
        host = host or "localhost",
        port = port or 5432,
        dbname = dbname or "default",
        _id = Database._connectionCount + 1,
        _queryHistory = {},
        _connected = false
    }, Database)
    
    Database._connectionCount = Database._connectionCount + 1
    Database._connections[#Database._connections + 1] = self
    
    return self
end

-- Instance methods
function Database:connect()
    print(string.format("[DB#%d] Connecting to %s:%d/%s...",
        self._id, self.host, self.port, self.dbname))
    self._connected = true
    return self
end

function Database:disconnect()
    if self._connected then
        print(string.format("[DB#%d] Disconnecting...", self._id))
        self._connected = false
        
        -- Remove from class-level connections list
        for i, conn in ipairs(Database._connections) do
            if conn == self then
                table.remove(Database._connections, i)
                Database._connectionCount = Database._connectionCount - 1
                break
            end
        end
    end
    return self
end

function Database:query(sql)
    assert(self._connected, "Not connected!")
    self._queryHistory[#self._queryHistory + 1] = {
        sql = sql,
        time = os.time()
    }
    print(string.format("[DB#%d] Query: %s", self._id, sql))
    return {rows = {}, affected = 0}
end

function Database:getQueryCount()
    return #self._queryHistory
end

function Database:__tostring()
    return string.format("DB#%d(%s:%d/%s %s)",
        self._id, self.host, self.port, self.dbname,
        self._connected and "connected" or "disconnected")
end

-- Test
print("Max connections:", Database._maxConnections)

local db1 = Database.new("prod.db.com", 5432, "myapp")
local db2 = Database.new("localhost", 5432, "test")

db1:connect()
db2:connect()

db1:query("SELECT * FROM users")
db1:query("SELECT * FROM products")
db2:query("SELECT * FROM test_data")

print("Active connections:", Database.getConnectionCount())
print(tostring(db1))
print("DB1 query count:", db1:getQueryCount())

db1:disconnect()
print("After disconnect:", Database.getConnectionCount())
```

---

## 21.7 toString/print Override

```lua
-- ตัวอย่างที่ 8: __tostring สำหรับ debugging
local Point3D = {}
Point3D.__index = Point3D

function Point3D.new(x, y, z)
    return setmetatable({x=x or 0, y=y or 0, z=z or 0}, Point3D)
end

function Point3D:__tostring()
    return string.format("Point3D(%.3f, %.3f, %.3f)", self.x, self.y, self.z)
end

-- Custom print method
function Point3D:print(label)
    if label then
        io.write(label .. ": ")
    end
    print(tostring(self))
    return self
end

function Point3D:distance(other)
    local dx = self.x - other.x
    local dy = self.y - other.y
    local dz = self.z - other.z
    return math.sqrt(dx*dx + dy*dy + dz*dz)
end

function Point3D.__add(a, b)
    return Point3D.new(a.x+b.x, a.y+b.y, a.z+b.z)
end

function Point3D.__sub(a, b)
    return Point3D.new(a.x-b.x, a.y-b.y, a.z-b.z)
end

function Point3D.__eq(a, b)
    local eps = 1e-9
    return math.abs(a.x-b.x) < eps and
           math.abs(a.y-b.y) < eps and
           math.abs(a.z-b.z) < eps
end

local p1 = Point3D.new(1, 2, 3)
local p2 = Point3D.new(4, 5, 6)

p1:print("p1")
p2:print("p2")
(p1 + p2):print("p1 + p2")
print("Distance:", p1:distance(p2))
```

---

## 21.8 ตัวอย่างสมบูรณ์: Animal Class Hierarchy

```lua
-- ตัวอย่างที่ 9: Animal → Dog, Cat hierarchy
local Animal = {}
Animal.__index = Animal
Animal._count = 0

function Animal.new(name, species, sound)
    Animal._count = Animal._count + 1
    return setmetatable({
        name = name,
        species = species,
        sound = sound,
        energy = 100,
        age = 0,
        _id = Animal._count
    }, Animal)
end

function Animal:speak()
    return string.format("%s (%s) says: %s!", self.name, self.species, self.sound)
end

function Animal:eat(food)
    self.energy = math.min(100, self.energy + 20)
    return string.format("%s eats %s (energy: %d)", self.name, food, self.energy)
end

function Animal:sleep(hours)
    self.energy = math.min(100, self.energy + hours * 5)
    return string.format("%s sleeps for %d hours (energy: %d)",
        self.name, hours, self.energy)
end

function Animal:birthday()
    self.age = self.age + 1
    return string.format("Happy birthday %s! Now %d years old.", self.name, self.age)
end

function Animal:isAlive()
    return self.energy > 0
end

function Animal:__tostring()
    return string.format("[%s] %s the %s (age: %d, energy: %d)",
        self.species, self.name, self.species, self.age, self.energy)
end

-- Dog extends Animal
local Dog = setmetatable({}, {__index = Animal})
Dog.__index = Dog

function Dog.new(name, breed)
    local self = Animal.new(name, "Dog", "Woof")
    self.breed = breed or "Mixed"
    self.tricks = {}
    self.happiness = 100
    return setmetatable(self, Dog)
end

function Dog:fetch(item)
    self.energy = self.energy - 10
    self.happiness = math.min(100, self.happiness + 20)
    return string.format("%s fetches the %s! (happiness: %d)",
        self.name, item, self.happiness)
end

function Dog:learnTrick(trick)
    self.tricks[#self.tricks + 1] = trick
    return string.format("%s learned: %s! (knows %d tricks)",
        self.name, trick, #self.tricks)
end

function Dog:performTrick()
    if #self.tricks == 0 then
        return self.name .. " doesn't know any tricks."
    end
    local trick = self.tricks[math.random(#self.tricks)]
    self.energy = self.energy - 5
    return string.format("%s performs: %s!", self.name, trick)
end

function Dog:wag()
    return self.name .. " wags their tail happily!"
end

function Dog:__tostring()
    return string.format("[Dog] %s the %s (age: %d, energy: %d, happiness: %d, tricks: %d)",
        self.name, self.breed, self.age, self.energy, self.happiness, #self.tricks)
end

-- Cat extends Animal
local Cat = setmetatable({}, {__index = Animal})
Cat.__index = Cat

function Cat.new(name, indoor)
    local self = Animal.new(name, "Cat", "Meow")
    self.indoor = indoor ~= false
    self.affection = 50  -- Cats are selective
    self.hunting_skill = 5
    return setmetatable(self, Cat)
end

function Cat:purr()
    self.affection = math.min(100, self.affection + 10)
    return string.format("%s purrs... (affection: %d)", self.name, self.affection)
end

function Cat:hiss(target)
    self.affection = math.max(0, self.affection - 20)
    return string.format("%s hisses at %s!", self.name, target)
end

function Cat:hunt(prey)
    if math.random() < self.hunting_skill / 10 then
        self.hunting_skill = math.min(10, self.hunting_skill + 0.1)
        self.energy = self.energy + 10
        return string.format("%s caught the %s! (skill: %.1f)",
            self.name, prey, self.hunting_skill)
    else
        self.energy = self.energy - 5
        return string.format("%s missed the %s... (skill: %.1f)",
            self.name, prey, self.hunting_skill)
    end
end

function Cat:cuddle()
    local response = math.random()
    if self.affection > 70 then
        return self.name .. " cuddles back lovingly!"
    elseif self.affection > 40 then
        return self.name .. " tolerates the cuddle."
    else
        return self.name .. " scratches you and runs away!"
    end
end

function Cat:__tostring()
    local indoor_str = self.indoor and "indoor" or "outdoor"
    return string.format("[Cat] %s (%s, age: %d, energy: %d, affection: %d)",
        self.name, indoor_str, self.age, self.energy, self.affection)
end

-- Kitten extends Cat (multiple levels)
local Kitten = setmetatable({}, {__index = Cat})
Kitten.__index = Kitten

function Kitten.new(name)
    local self = Cat.new(name, true)
    self.playfulness = 100
    self.species = "Kitten"
    return setmetatable(self, Kitten)
end

function Kitten:play(toy)
    self.energy = self.energy - 15
    self.playfulness = math.min(100, self.playfulness + 10)
    return string.format("%s plays with %s! (playfulness: %d)",
        self.name, toy, self.playfulness)
end

function Kitten:grow()
    -- Kitten grows into Cat
    print(self.name .. " is growing up!")
    return Cat.new(self.name, self.indoor)
end

function Kitten:__tostring()
    return string.format("[Kitten] %s (age: %d, energy: %d, playfulness: %d)",
        self.name, self.age, self.energy, self.playfulness)
end

-- Test
math.randomseed(42)

print("=== Animal Hierarchy Demo ===\n")

local rex = Dog.new("Rex", "German Shepherd")
print(tostring(rex))
print(rex:speak())
print(rex:eat("bone"))
print(rex:fetch("ball"))
print(rex:learnTrick("sit"))
print(rex:learnTrick("shake"))
print(rex:learnTrick("roll over"))
print(rex:performTrick())
print(rex:wag())
rex:birthday()
print(tostring(rex))

print()

local luna = Cat.new("Luna", true)
print(tostring(luna))
print(luna:speak())
print(luna:purr())
print(luna:hunt("mouse"))
print(luna:hunt("mouse"))
print(luna:cuddle())
print(tostring(luna))

print()

local mochi = Kitten.new("Mochi")
print(tostring(mochi))
print(mochi:speak())
print(mochi:play("yarn ball"))
print(mochi:play("laser pointer"))
print(tostring(mochi))

print()
print(string.format("Total animals created: %d", Animal._count))
```

---

## 21.9 Bank Account System

```lua
-- ตัวอย่างที่ 10: Bank account system
local Account = {}
Account.__index = Account
Account._nextId = 1000

function Account.new(owner, initialBalance)
    Account._nextId = Account._nextId + 1
    return setmetatable({
        _id = Account._nextId,
        _owner = owner,
        _balance = initialBalance or 0,
        _transactions = {},
        _frozen = false,
        _createdAt = os.time()
    }, Account)
end

function Account:_addTransaction(type, amount, description)
    self._transactions[#self._transactions + 1] = {
        type = type,
        amount = amount,
        balance = self._balance,
        description = description or "",
        timestamp = os.time()
    }
end

function Account:deposit(amount, description)
    assert(not self._frozen, "Account is frozen")
    assert(type(amount) == "number" and amount > 0,
        "Deposit amount must be positive")
    
    self._balance = self._balance + amount
    self:_addTransaction("deposit", amount, description or "Deposit")
    return self
end

function Account:withdraw(amount, description)
    assert(not self._frozen, "Account is frozen")
    assert(type(amount) == "number" and amount > 0,
        "Withdrawal amount must be positive")
    assert(amount <= self._balance,
        string.format("Insufficient funds: need %.2f, have %.2f",
            amount, self._balance))
    
    self._balance = self._balance - amount
    self:_addTransaction("withdrawal", -amount, description or "Withdrawal")
    return self
end

function Account:transfer(target, amount, description)
    assert(target ~= self, "Cannot transfer to same account")
    self:withdraw(amount, description or "Transfer out")
    target:deposit(amount, description or "Transfer in")
    return self
end

function Account:freeze()
    self._frozen = true
    print(string.format("Account #%d frozen", self._id))
    return self
end

function Account:unfreeze()
    self._frozen = false
    print(string.format("Account #%d unfrozen", self._id))
    return self
end

function Account:getBalance()
    return self._balance
end

function Account:getId()
    return self._id
end

function Account:getOwner()
    return self._owner
end

function Account:printStatement(n)
    n = n or #self._transactions
    print(string.format("\n=== Statement for Account #%d (%s) ===",
        self._id, self._owner))
    print(string.format("%-20s %-12s %-12s %s",
        "Description", "Amount", "Balance", "Type"))
    print(string.rep("-", 60))
    
    local start = math.max(1, #self._transactions - n + 1)
    for i = start, #self._transactions do
        local t = self._transactions[i]
        print(string.format("%-20s %+11.2f %11.2f %s",
            t.description:sub(1, 20),
            t.amount,
            t.balance,
            t.type))
    end
    print(string.rep("-", 60))
    print(string.format("%-20s %23.2f", "Current Balance:", self._balance))
end

function Account:__tostring()
    return string.format("Account{#%d, owner=%s, balance=%.2f, %s}",
        self._id, self._owner, self._balance,
        self._frozen and "frozen" or "active")
end

-- SavingsAccount extends Account
local SavingsAccount = setmetatable({}, {__index = Account})
SavingsAccount.__index = SavingsAccount

function SavingsAccount.new(owner, initialBalance, interestRate)
    local self = Account.new(owner, initialBalance)
    self._interestRate = interestRate or 0.03  -- 3% per year
    self._interestCycle = 0
    return setmetatable(self, SavingsAccount)
end

function SavingsAccount:applyInterest()
    self._interestCycle = self._interestCycle + 1
    local interest = self._balance * self._interestRate
    self:deposit(interest, string.format("Interest (cycle %d)", self._interestCycle))
    return interest
end

function SavingsAccount:withdraw(amount, description)
    -- Savings accounts have withdrawal limit
    local maxWithdrawal = self._balance * 0.9  -- can only withdraw 90%
    if amount > maxWithdrawal then
        error(string.format("Savings withdrawal limit: %.2f (90%% of balance)",
            maxWithdrawal))
    end
    return Account.withdraw(self, amount, description)
end

function SavingsAccount:__tostring()
    return string.format("SavingsAccount{#%d, owner=%s, balance=%.2f, rate=%.1f%%}",
        self._id, self._owner, self._balance, self._interestRate * 100)
end

-- Test
print("=== Banking System ===")

local alice_checking = Account.new("Alice", 1000)
local alice_savings = SavingsAccount.new("Alice", 5000, 0.05)
local bob_checking = Account.new("Bob", 500)

alice_checking:deposit(500, "Salary")
alice_checking:withdraw(200, "Rent")
alice_checking:transfer(bob_checking, 100, "Lunch repayment")

alice_savings:applyInterest()
alice_savings:applyInterest()

print(tostring(alice_checking))
print(tostring(alice_savings))
print(tostring(bob_checking))

alice_checking:printStatement()
alice_savings:printStatement()

-- Test frozen account
local ok, err = pcall(function()
    alice_checking:freeze()
    alice_checking:deposit(100)
end)
alice_checking:unfreeze()
print("Frozen account error:", ok, err and err:match("frozen$"))
```

---

## 21.10 Shape Hierarchy

```lua
-- ตัวอย่างที่ 11: Shape hierarchy
local Shape = {}
Shape.__index = Shape

function Shape.new(color)
    return setmetatable({
        color = color or "black",
        _type = "Shape"
    }, Shape)
end

function Shape:getColor() return self.color end
function Shape:setColor(c) self.color = c; return self end
function Shape:getType() return self._type end

function Shape:area()
    error(self._type .. ".area() not implemented", 2)
end

function Shape:perimeter()
    error(self._type .. ".perimeter() not implemented", 2)
end

function Shape:scale(factor)
    error(self._type .. ".scale() not implemented", 2)
end

function Shape:contains(x, y)
    error(self._type .. ".contains() not implemented", 2)
end

function Shape:describe()
    return string.format("%s[color=%s, area=%.4f, perimeter=%.4f]",
        self._type, self.color, self:area(), self:perimeter())
end

function Shape:__tostring()
    return self:describe()
end

-- Circle
local Circle = setmetatable({}, {__index = Shape})
Circle.__index = Circle

function Circle.new(cx, cy, r, color)
    local self = Shape.new(color)
    self._type = "Circle"
    self.cx = cx or 0
    self.cy = cy or 0
    self.r = r or 1
    return setmetatable(self, Circle)
end

function Circle:area()
    return math.pi * self.r * self.r
end

function Circle:perimeter()
    return 2 * math.pi * self.r
end

function Circle:scale(factor)
    self.r = self.r * factor
    return self
end

function Circle:contains(x, y)
    local dx = x - self.cx
    local dy = y - self.cy
    return dx*dx + dy*dy <= self.r * self.r
end

function Circle:describe()
    return string.format("Circle[center=(%.1f,%.1f), r=%.2f, color=%s, area=%.4f]",
        self.cx, self.cy, self.r, self.color, self:area())
end

-- Rectangle
local Rectangle = setmetatable({}, {__index = Shape})
Rectangle.__index = Rectangle

function Rectangle.new(x, y, w, h, color)
    local self = Shape.new(color)
    self._type = "Rectangle"
    self.x = x or 0
    self.y = y or 0
    self.w = w or 1
    self.h = h or 1
    return setmetatable(self, Rectangle)
end

function Rectangle:area()
    return self.w * self.h
end

function Rectangle:perimeter()
    return 2 * (self.w + self.h)
end

function Rectangle:scale(factor)
    self.w = self.w * factor
    self.h = self.h * factor
    return self
end

function Rectangle:contains(x, y)
    return x >= self.x and x <= self.x + self.w and
           y >= self.y and y <= self.y + self.h
end

function Rectangle:isSquare()
    return math.abs(self.w - self.h) < 1e-9
end

function Rectangle:describe()
    return string.format("Rectangle[(%g,%g) %gx%g, color=%s, area=%.4f%s]",
        self.x, self.y, self.w, self.h, self.color, self:area(),
        self:isSquare() and " (square)" or "")
end

-- Triangle
local Triangle = setmetatable({}, {__index = Shape})
Triangle.__index = Triangle

function Triangle.new(x1, y1, x2, y2, x3, y3, color)
    local self = Shape.new(color)
    self._type = "Triangle"
    self.x1, self.y1 = x1 or 0, y1 or 0
    self.x2, self.y2 = x2 or 1, y2 or 0
    self.x3, self.y3 = x3 or 0, y3 or 1
    return setmetatable(self, Triangle)
end

function Triangle:_sides()
    local function dist(x1,y1,x2,y2)
        return math.sqrt((x2-x1)^2 + (y2-y1)^2)
    end
    local a = dist(self.x1,self.y1, self.x2,self.y2)
    local b = dist(self.x2,self.y2, self.x3,self.y3)
    local c = dist(self.x3,self.y3, self.x1,self.y1)
    return a, b, c
end

function Triangle:area()
    -- Using cross product formula
    return math.abs(
        (self.x2 - self.x1) * (self.y3 - self.y1) -
        (self.x3 - self.x1) * (self.y2 - self.y1)
    ) / 2
end

function Triangle:perimeter()
    local a, b, c = self:_sides()
    return a + b + c
end

function Triangle:scale(factor)
    local cx = (self.x1 + self.x2 + self.x3) / 3
    local cy = (self.y1 + self.y2 + self.y3) / 3
    self.x1 = cx + (self.x1 - cx) * factor
    self.y1 = cy + (self.y1 - cy) * factor
    self.x2 = cx + (self.x2 - cx) * factor
    self.y2 = cy + (self.y2 - cy) * factor
    self.x3 = cx + (self.x3 - cx) * factor
    self.y3 = cy + (self.y3 - cy) * factor
    return self
end

function Triangle:contains(x, y)
    -- Barycentric coordinates method
    local v0x = self.x3 - self.x1
    local v0y = self.y3 - self.y1
    local v1x = self.x2 - self.x1
    local v1y = self.y2 - self.y1
    local v2x = x - self.x1
    local v2y = y - self.y1
    
    local dot00 = v0x*v0x + v0y*v0y
    local dot01 = v0x*v1x + v0y*v1y
    local dot02 = v0x*v2x + v0y*v2y
    local dot11 = v1x*v1x + v1y*v1y
    local dot12 = v1x*v2x + v1y*v2y
    
    local inv = 1 / (dot00*dot11 - dot01*dot01)
    local u = (dot11*dot02 - dot01*dot12) * inv
    local v = (dot00*dot12 - dot01*dot02) * inv
    
    return u >= 0 and v >= 0 and u + v <= 1
end

function Triangle:getType()
    local a, b, c = self:_sides()
    local sides = {a, b, c}
    table.sort(sides)
    
    if math.abs(sides[1] - sides[2]) < 1e-9 and
       math.abs(sides[2] - sides[3]) < 1e-9 then
        return "equilateral"
    elseif math.abs(sides[1] - sides[2]) < 1e-9 or
           math.abs(sides[2] - sides[3]) < 1e-9 then
        return "isosceles"
    else
        return "scalene"
    end
end

function Triangle:describe()
    return string.format("Triangle[%s, color=%s, area=%.4f, perimeter=%.4f]",
        self:getType(), self.color, self:area(), self:perimeter())
end

-- Polymorphism test
local shapes = {
    Circle.new(0, 0, 5, "red"),
    Rectangle.new(0, 0, 8, 4, "blue"),
    Triangle.new(0, 0, 6, 0, 3, 5, "green"),
    Circle.new(2, 2, 3, "yellow"),
    Rectangle.new(-1, -1, 4, 4, "purple"),  -- square
}

print("=== Shape Collection ===")
local totalArea = 0
for _, shape in ipairs(shapes) do
    print(shape:describe())
    totalArea = totalArea + shape:area()
end
print(string.format("\nTotal area: %.4f", totalArea))

-- Sort by area
table.sort(shapes, function(a, b) return a:area() < b:area() end)
print("\nSorted by area:")
for _, shape in ipairs(shapes) do
    print(string.format("  %-12s area=%.4f", shape:getType(), shape:area()))
end

-- Scale all shapes
print("\nAfter scaling by 2:")
for _, shape in ipairs(shapes) do
    shape:scale(2)
    print(string.format("  %-12s area=%.4f", shape:getType(), shape:area()))
end
```

---

## 21.11 Comparing OOP Styles

```lua
-- ตัวอย่างที่ 12: OOP Style 1 - Closure-based (private by default)
local function createStack()
    local _data = {}  -- truly private
    
    return {
        push = function(val)
            _data[#_data + 1] = val
        end,
        pop = function()
            if #_data == 0 then return nil end
            local val = _data[#_data]
            _data[#_data] = nil
            return val
        end,
        peek = function()
            return _data[#_data]
        end,
        size = function()
            return #_data
        end,
        isEmpty = function()
            return #_data == 0
        end,
        toArray = function()
            return {table.unpack(_data)}
        end
    }
end

local s1 = createStack()
s1.push(1)
s1.push(2)
s1.push(3)
print("Size:", s1.size())    -- 3
print("Pop:", s1.pop())      -- 3
print("Peek:", s1.peek())    -- 2
```

```lua
-- ตัวอย่างที่ 13: OOP Style 2 - Prototype-based (metatable)
local StackProto = {}
StackProto.__index = StackProto

function StackProto.new()
    return setmetatable({_data = {}}, StackProto)
end

function StackProto:push(val)
    self._data[#self._data + 1] = val
    return self
end

function StackProto:pop()
    if #self._data == 0 then return nil end
    local val = self._data[#self._data]
    self._data[#self._data] = nil
    return val
end

function StackProto:peek()
    return self._data[#self._data]
end

function StackProto:size()
    return #self._data
end

function StackProto:isEmpty()
    return #self._data == 0
end

function StackProto:__len()
    return #self._data
end

function StackProto:__tostring()
    local items = {}
    for i = #self._data, 1, -1 do
        items[#items + 1] = tostring(self._data[i])
    end
    return "Stack[" .. table.concat(items, " | ") .. "]"
end

local s2 = StackProto.new()
s2:push(10):push(20):push(30)
print(tostring(s2))   -- Stack[30 | 20 | 10]
print("Size:", #s2)   -- 3
```

```lua
-- ตัวอย่างที่ 14: OOP Style 3 - Class framework
local Class = {}
Class.__index = Class

function Class:extend()
    local cls = {}
    cls.__index = cls
    cls.super = self
    
    setmetatable(cls, {
        __index = self,
        __call = function(c, ...)
            local instance = setmetatable({}, c)
            if instance.init then
                instance:init(...)
            end
            return instance
        end
    })
    
    return cls
end

-- สร้าง classes ด้วย Class framework
local Animal = Class:extend()

function Animal:init(name, sound)
    self.name = name
    self.sound = sound
end

function Animal:speak()
    return self.name .. " says " .. self.sound
end

local Dog = Animal:extend()

function Dog:init(name, breed)
    Dog.super.init(self, name, "Woof")  -- call parent
    self.breed = breed
end

function Dog:fetch(item)
    return self.name .. " fetches " .. item
end

-- Create instances using __call
local fido = Dog("Fido", "Labrador")
print(fido:speak())        -- Fido says Woof
print(fido:fetch("stick")) -- Fido fetches stick
print("Breed:", fido.breed)
```

---

## 21.12 Mixin และ Multiple Inheritance

```lua
-- ตัวอย่างที่ 15: Mixin pattern
local function mixin(cls, ...)
    for _, m in ipairs({...}) do
        for k, v in pairs(m) do
            if type(v) == "function" and not cls[k] then
                cls[k] = v
            end
        end
    end
    return cls
end

-- Define mixins
local Serializable = {
    toJSON = function(self)
        local parts = {}
        for k, v in pairs(self) do
            if type(v) ~= "function" then
                if type(v) == "string" then
                    parts[#parts+1] = string.format('"%s":"%s"', k, v)
                else
                    parts[#parts+1] = string.format('"%s":%s', k, tostring(v))
                end
            end
        end
        table.sort(parts)
        return "{" .. table.concat(parts, ",") .. "}"
    end,
    
    fromJSON = function(cls, json)
        -- Simplified JSON parser
        local obj = cls.new and cls.new() or setmetatable({}, cls)
        for k, v in json:gmatch('"(%w+)":"?([^",}]+)"?') do
            local n = tonumber(v)
            obj[k] = n or (v == "true" and true or (v == "false" and false or v))
        end
        return obj
    end
}

local Comparable = {
    equals = function(self, other)
        if getmetatable(self) ~= getmetatable(other) then return false end
        for k, v in pairs(self) do
            if type(v) ~= "function" and v ~= other[k] then return false end
        end
        return true
    end,
    
    notEquals = function(self, other)
        return not self:equals(other)
    end
}

local Observable = {
    _listeners = nil,
    
    addListener = function(self, event, fn)
        self._listeners = self._listeners or {}
        self._listeners[event] = self._listeners[event] or {}
        table.insert(self._listeners[event], fn)
    end,
    
    emit = function(self, event, ...)
        if not self._listeners then return end
        for _, fn in ipairs(self._listeners[event] or {}) do
            fn(...)
        end
    end
}

-- Class with mixins
local User = {}
User.__index = User

mixin(User, Serializable, Comparable, Observable)

function User.new(name, email, age)
    return setmetatable({
        name = name,
        email = email,
        age = age
    }, User)
end

function User:updateAge(newAge)
    local old = self.age
    self.age = newAge
    self:emit("ageChanged", old, newAge)
end

-- Test
local u1 = User.new("Alice", "alice@example.com", 30)
local u2 = User.new("Alice", "alice@example.com", 30)
local u3 = User.new("Bob", "bob@example.com", 25)

print(u1:toJSON())
print("u1 == u2:", u1:equals(u2))   -- true
print("u1 == u3:", u1:equals(u3))   -- false

u1:addListener("ageChanged", function(old, new)
    print(string.format("Age changed: %d -> %d", old, new))
end)

u1:updateAge(31)  -- triggers listener
```

---

## 21.13 ตัวอย่างสมบูรณ์: Game Entity System

```lua
-- ตัวอย่างที่ 16: Game entity system
local Entity = {}
Entity.__index = Entity
Entity._nextId = 0

function Entity.new(x, y)
    Entity._nextId = Entity._nextId + 1
    return setmetatable({
        id = Entity._nextId,
        x = x or 0,
        y = y or 0,
        active = true,
        tags = {}
    }, Entity)
end

function Entity:addTag(tag)
    self.tags[tag] = true
    return self
end

function Entity:hasTag(tag)
    return self.tags[tag] == true
end

function Entity:moveTo(x, y)
    self.x = x
    self.y = y
    return self
end

function Entity:distanceTo(other)
    local dx = self.x - other.x
    local dy = self.y - other.y
    return math.sqrt(dx*dx + dy*dy)
end

function Entity:destroy()
    self.active = false
end

function Entity:__tostring()
    return string.format("Entity#%d(%.1f, %.1f)", self.id, self.x, self.y)
end

-- Character extends Entity
local Character = setmetatable({}, {__index = Entity})
Character.__index = Character

function Character.new(name, x, y, hp)
    local self = Entity.new(x, y)
    self.name = name
    self.hp = hp or 100
    self.maxHp = hp or 100
    self.level = 1
    self.exp = 0
    self.inventory = {}
    return setmetatable(self, Character)
end

function Character:takeDamage(amount)
    self.hp = math.max(0, self.hp - amount)
    if self.hp <= 0 then
        print(self.name .. " was defeated!")
        self:destroy()
    end
    return self
end

function Character:heal(amount)
    if not self.active then return self end
    self.hp = math.min(self.maxHp, self.hp + amount)
    return self
end

function Character:gainExp(amount)
    self.exp = self.exp + amount
    local expNeeded = self.level * 100
    if self.exp >= expNeeded then
        self.exp = self.exp - expNeeded
        self.level = self.level + 1
        self.maxHp = self.maxHp + 10
        self.hp = self.maxHp
        print(string.format("%s leveled up to %d!", self.name, self.level))
    end
    return self
end

function Character:addItem(item)
    self.inventory[#self.inventory + 1] = item
    return self
end

function Character:isAlive()
    return self.active and self.hp > 0
end

function Character:__tostring()
    return string.format("Character[%s, HP:%d/%d, Lv:%d, (%.1f,%.1f)]",
        self.name, self.hp, self.maxHp, self.level, self.x, self.y)
end

-- Warrior extends Character
local Warrior = setmetatable({}, {__index = Character})
Warrior.__index = Warrior

function Warrior.new(name, x, y)
    local self = Character.new(name, x, y, 150)
    self.attack = 20
    self.defense = 15
    self.rage = 0
    self:addTag("melee")
    return setmetatable(self, Warrior)
end

function Warrior:attack_target(target)
    local damage = self.attack + math.random(-5, 5)
    local actualDamage = math.max(1, damage - (target.defense or 0))
    print(string.format("%s attacks %s for %d damage!",
        self.name, target.name, actualDamage))
    target:takeDamage(actualDamage)
    self.rage = math.min(100, self.rage + 10)
    return self
end

function Warrior:berserk()
    if self.rage < 50 then
        print(self.name .. ": Not enough rage!")
        return self
    end
    self.rage = 0
    local oldAttack = self.attack
    self.attack = self.attack * 2
    print(self.name .. " goes BERSERK! Attack doubled!")
    -- Reset after 3 turns (simplified: just show it)
    -- In a real game, you'd track turns
    self.attack = oldAttack  -- simplified: immediate reset
    return self
end

-- Mage extends Character  
local Mage = setmetatable({}, {__index = Character})
Mage.__index = Mage

function Mage.new(name, x, y)
    local self = Character.new(name, x, y, 80)
    self.mana = 100
    self.maxMana = 100
    self.spellPower = 30
    self:addTag("ranged")
    self:addTag("magic")
    return setmetatable(self, Mage)
end

function Mage:castFireball(target)
    if self.mana < 20 then
        print(self.name .. ": Not enough mana!")
        return self
    end
    self.mana = self.mana - 20
    local damage = self.spellPower + math.random(-10, 10)
    print(string.format("%s casts Fireball on %s for %d magic damage!",
        self.name, target.name, damage))
    target:takeDamage(damage)
    return self
end

function Mage:castHeal(target)
    if self.mana < 15 then
        print(self.name .. ": Not enough mana!")
        return self
    end
    self.mana = self.mana - 15
    local healing = math.floor(self.spellPower * 0.7)
    print(string.format("%s heals %s for %d HP!",
        self.name, target.name, healing))
    target:heal(healing)
    return self
end

function Mage:__tostring()
    return string.format("Mage[%s, HP:%d/%d, MP:%d/%d, Lv:%d]",
        self.name, self.hp, self.maxHp, self.mana, self.maxMana, self.level)
end

-- Monster extends Entity
local Monster = setmetatable({}, {__index = Entity})
Monster.__index = Monster

function Monster.new(name, x, y, hp, attack)
    local self = Entity.new(x, y)
    self.name = name
    self.hp = hp or 50
    self.maxHp = hp or 50
    self.attack = attack or 10
    self.defense = 5
    self.expReward = hp or 50
    self.active = true
    self:addTag("enemy")
    return setmetatable(self, Monster)
end

function Monster:takeDamage(amount)
    self.hp = math.max(0, self.hp - amount)
    if self.hp <= 0 then
        print(self.name .. " was defeated! +" .. self.expReward .. " EXP")
        self.active = false
    end
    return self
end

function Monster:attack_target(target)
    if not self.active then return end
    local damage = self.attack + math.random(-3, 3)
    print(string.format("%s attacks %s for %d damage!",
        self.name, target.name, damage))
    target:takeDamage(damage)
end

function Monster:__tostring()
    return string.format("Monster[%s, HP:%d/%d, ATK:%d]",
        self.name, self.hp, self.maxHp, self.attack)
end

-- Simulate a battle
math.randomseed(42)
print("=== RPG Battle Simulation ===\n")

local hero = Warrior.new("Thor", 0, 0)
local healer = Mage.new("Merlin", 1, 0)
local boss = Monster.new("Dragon", 5, 5, 200, 25)
local minion1 = Monster.new("Goblin", 3, 3, 30, 8)
local minion2 = Monster.new("Orc", 4, 2, 50, 12)

local enemies = {boss, minion1, minion2}
local players = {hero, healer}

print("--- Initial State ---")
print(tostring(hero))
print(tostring(healer))
for _, e in ipairs(enemies) do print(tostring(e)) end

print("\n--- Battle Begins ---\n")

for round = 1, 5 do
    print(string.format("=== Round %d ===", round))
    
    -- Players attack
    if hero:isAlive() then
        for _, enemy in ipairs(enemies) do
            if enemy.active then
                hero:attack_target(enemy)
                if not enemy.active then
                    hero:gainExp(enemy.expReward)
                end
                break
            end
        end
    end
    
    if healer:isAlive() then
        -- Heal if hero is low HP
        if hero.hp < hero.maxHp * 0.5 then
            healer:castHeal(hero)
        else
            -- Attack enemy
            for _, enemy in ipairs(enemies) do
                if enemy.active then
                    healer:castFireball(enemy)
                    if not enemy.active then
                        healer:gainExp(enemy.expReward)
                    end
                    break
                end
            end
        end
    end
    
    -- Enemies attack
    for _, enemy in ipairs(enemies) do
        if enemy.active then
            -- Attack random player
            local target = math.random() < 0.7 and hero or healer
            if target:isAlive() then
                enemy:attack_target(target)
            end
        end
    end
    
    -- Check if all enemies defeated
    local allDefeated = true
    for _, enemy in ipairs(enemies) do
        if enemy.active then allDefeated = false; break end
    end
    
    if allDefeated then
        print("\n*** All enemies defeated! Victory! ***")
        break
    end
    
    -- Check if all players defeated
    local allDefeated2 = true
    for _, player in ipairs(players) do
        if player:isAlive() then allDefeated2 = false; break end
    end
    
    if allDefeated2 then
        print("\n*** All heroes defeated! Game Over! ***")
        break
    end
    
    print()
end

print("\n--- Final State ---")
print(tostring(hero))
print(tostring(healer))
for _, e in ipairs(enemies) do
    if e.active then
        print(tostring(e))
    else
        print(e.name .. " [DEFEATED]")
    end
end
```

---

## 21.14 OOP Patterns

```lua
-- ตัวอย่างที่ 17: Observer Pattern
local EventEmitter = {}
EventEmitter.__index = EventEmitter

function EventEmitter.new()
    return setmetatable({_events = {}}, EventEmitter)
end

function EventEmitter:on(event, handler)
    self._events[event] = self._events[event] or {}
    table.insert(self._events[event], handler)
    return function()  -- unsubscribe
        for i, h in ipairs(self._events[event] or {}) do
            if h == handler then
                table.remove(self._events[event], i)
                break
            end
        end
    end
end

function EventEmitter:emit(event, ...)
    for _, h in ipairs(self._events[event] or {}) do
        h(...)
    end
end

-- Observable class
local Observable = setmetatable({}, {__index = EventEmitter})
Observable.__index = Observable

function Observable.new(data)
    local self = EventEmitter.new()
    self._data = data or {}
    return setmetatable(self, Observable)
end

function Observable:get(key)
    return self._data[key]
end

function Observable:set(key, value)
    local old = self._data[key]
    self._data[key] = value
    self:emit("change", key, value, old)
    self:emit("change:" .. key, value, old)
end

function Observable:__index(key)
    -- First check Observable methods
    local method = Observable[key]
    if method then return method end
    -- Then check data
    return rawget(self, "_data") and rawget(self, "_data")[key]
end

-- Test
local state = Observable.new({name = "Alice", score = 0})

local unsub1 = state:on("change", function(key, new, old)
    print(string.format("State changed: %s = %s (was %s)", key, tostring(new), tostring(old)))
end)

local unsub2 = state:on("change:score", function(new, old)
    if new > (old or 0) then
        print("Score increased by:", new - old)
    end
end)

state:set("name", "Bob")
state:set("score", 100)
state:set("score", 150)

unsub2()  -- unsubscribe score listener
state:set("score", 200)  -- ไม่มี score listener แล้ว
```

```lua
-- ตัวอย่างที่ 18: Strategy Pattern
local Sorter = {}
Sorter.__index = Sorter

function Sorter.new(strategy)
    return setmetatable({_strategy = strategy}, Sorter)
end

function Sorter:setStrategy(strategy)
    self._strategy = strategy
    return self
end

function Sorter:sort(data)
    -- Make a copy
    local copy = {table.unpack(data)}
    return self._strategy(copy)
end

-- Strategies
local strategies = {
    bubble = function(arr)
        local n = #arr
        for i = 1, n-1 do
            for j = 1, n-i do
                if arr[j] > arr[j+1] then
                    arr[j], arr[j+1] = arr[j+1], arr[j]
                end
            end
        end
        return arr
    end,
    
    insertion = function(arr)
        for i = 2, #arr do
            local key = arr[i]
            local j = i - 1
            while j > 0 and arr[j] > key do
                arr[j+1] = arr[j]
                j = j - 1
            end
            arr[j+1] = key
        end
        return arr
    end,
    
    builtin = function(arr)
        table.sort(arr)
        return arr
    end
}

local data = {64, 34, 25, 12, 22, 11, 90}
local sorter = Sorter.new(strategies.bubble)

print("Original:", table.concat(data, ", "))
print("Bubble:", table.concat(sorter:sort(data), ", "))
print("Insertion:", table.concat(sorter:setStrategy(strategies.insertion):sort(data), ", "))
print("Built-in:", table.concat(sorter:setStrategy(strategies.builtin):sort(data), ", "))
```

```lua
-- ตัวอย่างที่ 19: Decorator Pattern
local function logged(fn, name)
    return function(self, ...)
        print(string.format("[LOG] %s.%s called", self.name or "?", name))
        local results = {fn(self, ...)}
        print(string.format("[LOG] %s.%s returned: %s",
            self.name or "?", name, table.concat(
                (function()
                    local s = {}
                    for _, v in ipairs(results) do s[#s+1] = tostring(v) end
                    return s
                end)(), ", "
            )))
        return table.unpack(results)
    end
end

local function validated(fn, validator)
    return function(self, ...)
        local ok, err = validator(...)
        if not ok then
            error("Validation failed: " .. (err or "unknown"), 2)
        end
        return fn(self, ...)
    end
end

-- Service class
local UserService = {}
UserService.__index = UserService

function UserService.new(name)
    return setmetatable({name = name, users = {}}, UserService)
end

function UserService:createUser(username, email)
    self.users[username] = {username = username, email = email}
    return self.users[username]
end

function UserService:getUser(username)
    return self.users[username]
end

-- Apply decorators
UserService.createUser = logged(
    validated(UserService.createUser, function(username, email)
        if type(username) ~= "string" or #username < 3 then
            return false, "username must be >= 3 chars"
        end
        if not email:match("@") then
            return false, "invalid email"
        end
        return true
    end),
    "createUser"
)

UserService.getUser = logged(UserService.getUser, "getUser")

local svc = UserService.new("UserService")
svc:createUser("alice", "alice@example.com")
local user = svc:getUser("alice")
print("Found user:", user and user.email)
```

---

## 21.15 ตัวอย่างรวม: Complete OOP Application

```lua
-- ตัวอย่างที่ 20: Library Management System
local Book = {}
Book.__index = Book
Book._nextId = 0

function Book.new(title, author, isbn, year)
    Book._nextId = Book._nextId + 1
    return setmetatable({
        id = Book._nextId,
        title = title,
        author = author,
        isbn = isbn,
        year = year,
        available = true,
        borrowedBy = nil,
        borrowDate = nil
    }, Book)
end

function Book:__tostring()
    local status = self.available and "Available" or
        string.format("Borrowed by %s", self.borrowedBy)
    return string.format("[%d] '%s' by %s (%d) - %s",
        self.id, self.title, self.author, self.year, status)
end

local Member = {}
Member.__index = Member
Member._nextId = 0

function Member.new(name, email)
    Member._nextId = Member._nextId + 1
    return setmetatable({
        id = Member._nextId,
        name = name,
        email = email,
        borrowedBooks = {},
        borrowHistory = {}
    }, Member)
end

function Member:getBorrowCount()
    return #self.borrowedBooks
end

function Member:__tostring()
    return string.format("Member[%d: %s (%s), borrowed: %d]",
        self.id, self.name, self.email, self:getBorrowCount())
end

local Library = {}
Library.__index = Library

function Library.new(name)
    return setmetatable({
        name = name,
        books = {},
        members = {},
        _bookIndex = {},   -- isbn -> book
        _memberIndex = {}  -- email -> member
    }, Library)
end

function Library:addBook(title, author, isbn, year)
    if self._bookIndex[isbn] then
        error("Book already exists with ISBN: " .. isbn)
    end
    local book = Book.new(title, author, isbn, year)
    self.books[#self.books + 1] = book
    self._bookIndex[isbn] = book
    print(string.format("[Library] Added: %s", tostring(book)))
    return book
end

function Library:registerMember(name, email)
    if self._memberIndex[email] then
        error("Member already registered: " .. email)
    end
    local member = Member.new(name, email)
    self.members[#self.members + 1] = member
    self._memberIndex[email] = member
    print(string.format("[Library] Registered: %s", tostring(member)))
    return member
end

function Library:borrow(memberEmail, isbn)
    local member = self._memberIndex[memberEmail]
    assert(member, "Member not found: " .. memberEmail)
    
    local book = self._bookIndex[isbn]
    assert(book, "Book not found: " .. isbn)
    assert(book.available, "Book not available: " .. book.title)
    
    if member:getBorrowCount() >= 3 then
        error(member.name .. " has reached the borrowing limit (3 books)")
    end
    
    book.available = false
    book.borrowedBy = member.name
    book.borrowDate = os.time()
    
    member.borrowedBooks[#member.borrowedBooks + 1] = book
    member.borrowHistory[#member.borrowHistory + 1] = {
        book = book,
        borrowDate = os.time(),
        returnDate = nil
    }
    
    print(string.format("[Library] %s borrowed '%s'",
        member.name, book.title))
    return book
end

function Library:returnBook(memberEmail, isbn)
    local member = self._memberIndex[memberEmail]
    assert(member, "Member not found: " .. memberEmail)
    
    local book = self._bookIndex[isbn]
    assert(book, "Book not found: " .. isbn)
    assert(not book.available, "Book is not borrowed")
    assert(book.borrowedBy == member.name, "Book not borrowed by this member")
    
    book.available = true
    book.borrowedBy = nil
    book.borrowDate = nil
    
    for i, b in ipairs(member.borrowedBooks) do
        if b.isbn == isbn then
            table.remove(member.borrowedBooks, i)
            break
        end
    end
    
    for _, h in ipairs(member.borrowHistory) do
        if h.book.isbn == isbn and not h.returnDate then
            h.returnDate = os.time()
            break
        end
    end
    
    print(string.format("[Library] %s returned '%s'",
        member.name, book.title))
    return book
end

function Library:search(query)
    local results = {}
    query = query:lower()
    for _, book in ipairs(self.books) do
        if book.title:lower():find(query) or
           book.author:lower():find(query) then
            results[#results + 1] = book
        end
    end
    return results
end

function Library:printCatalog()
    print(string.format("\n=== %s Catalog ===", self.name))
    print(string.format("%-5s %-30s %-20s %-6s %s",
        "ID", "Title", "Author", "Year", "Status"))
    print(string.rep("-", 70))
    for _, book in ipairs(self.books) do
        print(string.format("%-5d %-30s %-20s %-6d %s",
            book.id,
            book.title:sub(1,30),
            book.author:sub(1,20),
            book.year,
            book.available and "Available" or "Borrowed"))
    end
end

function Library:printMemberInfo(email)
    local member = self._memberIndex[email]
    if not member then
        print("Member not found")
        return
    end
    
    print(string.format("\n=== Member: %s ===", member.name))
    print("Email:", member.email)
    print("Currently borrowed:")
    if #member.borrowedBooks == 0 then
        print("  (none)")
    else
        for _, book in ipairs(member.borrowedBooks) do
            print(string.format("  - %s (%s)", book.title, book.author))
        end
    end
end

-- Test the library system
print("=== Library Management System ===\n")

local lib = Library.new("City Public Library")

-- Add books
lib:addBook("The Lua Programming Language", "Roberto Ierusalimschy", "978-8590379850", 2016)
lib:addBook("Programming in Lua", "Roberto Ierusalimschy", "978-8590379867", 2019)
lib:addBook("Clean Code", "Robert C. Martin", "978-0132350884", 2008)
lib:addBook("Design Patterns", "Gang of Four", "978-0201633610", 1994)
lib:addBook("The Pragmatic Programmer", "Andrew Hunt", "978-0135957059", 2019)

-- Register members
lib:registerMember("Alice Smith", "alice@example.com")
lib:registerMember("Bob Jones", "bob@example.com")

print()

-- Borrow books
lib:borrow("alice@example.com", "978-8590379850")
lib:borrow("alice@example.com", "978-0132350884")
lib:borrow("bob@example.com", "978-0201633610")

-- Print catalog
lib:printCatalog()

-- Member info
lib:printMemberInfo("alice@example.com")

-- Return a book
print()
lib:returnBook("alice@example.com", "978-8590379850")

-- Search
print("\nSearch results for 'lua':")
local results = lib:search("lua")
for _, book in ipairs(results) do
    print("  " .. tostring(book))
end

lib:printCatalog()
```

---

## 21.16 ตัวอย่างเพิ่มเติม: Advanced OOP Techniques

```lua
-- ตัวอย่างที่ 21: Abstract class simulation
local function abstract(methodName)
    return function(self, ...)
        error(string.format(
            "Abstract method '%s' must be implemented by class '%s'",
            methodName,
            (getmetatable(self) or {}).__name or type(self)
        ), 2)
    end
end

local AbstractAnimal = {}
AbstractAnimal.__index = AbstractAnimal
AbstractAnimal.__name = "AbstractAnimal"

-- Abstract methods
AbstractAnimal.sound = abstract("sound")
AbstractAnimal.diet = abstract("diet")

-- Concrete methods
function AbstractAnimal:breathe()
    return self.name .. " breathes air"
end

function AbstractAnimal:eat(food)
    return string.format("%s eats %s (%s)",
        self.name, food, self:diet())
end

function AbstractAnimal:describe()
    return string.format("%s: sound=%s, diet=%s",
        self.name, self:sound(), self:diet())
end

-- Concrete implementations
local Lion = setmetatable({}, {__index = AbstractAnimal})
Lion.__index = Lion
Lion.__name = "Lion"

function Lion.new(name)
    return setmetatable({name = name}, Lion)
end

function Lion:sound() return "Roar" end
function Lion:diet() return "carnivore" end
function Lion:hunt(prey) return self.name .. " hunts " .. prey end

local Elephant = setmetatable({}, {__index = AbstractAnimal})
Elephant.__index = Elephant
Elephant.__name = "Elephant"

function Elephant.new(name)
    return setmetatable({name = name}, Elephant)
end

function Elephant:sound() return "Trumpet" end
function Elephant:diet() return "herbivore" end
function Elephant:remember(thing) return self.name .. " remembers " .. thing end

-- Test
local simba = Lion.new("Simba")
local dumbo = Elephant.new("Dumbo")

print(simba:describe())
print(dumbo:describe())
print(simba:breathe())
print(dumbo:eat("leaves"))
print(simba:hunt("gazelle"))
print(dumbo:remember("watering hole"))

-- Test abstract method error
local bad = setmetatable({name = "Bad"}, AbstractAnimal)
local ok, err = pcall(function() bad:sound() end)
print("Abstract error:", err and err:match("Abstract method") and "OK")
```

```lua
-- ตัวอย่างที่ 22: Interface simulation
local function implements(obj, interface)
    for methodName, _ in pairs(interface) do
        if type(obj[methodName]) ~= "function" then
            return false, "Missing method: " .. methodName
        end
    end
    return true
end

-- Define interfaces
local Drawable = {
    draw = true,
    getPosition = true,
    getBounds = true
}

local Clickable = {
    onClick = true,
    isPointInside = true
}

-- UI Components
local Button = {}
Button.__index = Button

function Button.new(x, y, w, h, label)
    return setmetatable({
        x = x, y = y, w = w, h = h,
        label = label or "Button",
        _onClick = nil
    }, Button)
end

function Button:draw()
    print(string.format("[Button] Drawing '%s' at (%d,%d) size %dx%d",
        self.label, self.x, self.y, self.w, self.h))
end

function Button:getPosition()
    return self.x, self.y
end

function Button:getBounds()
    return self.x, self.y, self.w, self.h
end

function Button:isPointInside(px, py)
    return px >= self.x and px <= self.x + self.w and
           py >= self.y and py <= self.y + self.h
end

function Button:onClick(handler)
    if type(handler) == "function" then
        self._onClick = handler
    elseif self._onClick then
        self._onClick(self)
    end
    return self
end

-- Check interfaces
local btn = Button.new(10, 20, 100, 30, "Submit")

local drawOk, drawErr = implements(btn, Drawable)
local clickOk, clickErr = implements(btn, Clickable)

print("Implements Drawable:", drawOk)
print("Implements Clickable:", clickOk)

btn:draw()
print("Point (50,30) inside:", btn:isPointInside(50, 30))
print("Point (200,30) inside:", btn:isPointInside(200, 30))

btn:onClick(function(b)
    print("Button '" .. b.label .. "' clicked!")
end)
btn:onClick()  -- trigger click
```

```lua
-- ตัวอย่างที่ 23: Fluent interface / Builder
local QueryBuilder = {}
QueryBuilder.__index = QueryBuilder

function QueryBuilder.new()
    return setmetatable({
        _select = {},
        _from = nil,
        _joins = {},
        _where = {},
        _groupBy = {},
        _having = {},
        _orderBy = {},
        _limit = nil,
        _offset = nil
    }, QueryBuilder)
end

function QueryBuilder:select(...)
    self._select = {...}
    return self
end

function QueryBuilder:from(table_name, alias)
    self._from = alias and (table_name .. " AS " .. alias) or table_name
    return self
end

function QueryBuilder:join(table_name, condition, type)
    type = type or "INNER"
    self._joins[#self._joins + 1] = string.format(
        "%s JOIN %s ON %s", type, table_name, condition)
    return self
end

function QueryBuilder:leftJoin(table_name, condition)
    return self:join(table_name, condition, "LEFT")
end

function QueryBuilder:where(condition)
    self._where[#self._where + 1] = condition
    return self
end

function QueryBuilder:groupBy(...)
    self._groupBy = {...}
    return self
end

function QueryBuilder:having(condition)
    self._having[#self._having + 1] = condition
    return self
end

function QueryBuilder:orderBy(field, direction)
    self._orderBy[#self._orderBy + 1] = field .. " " .. (direction or "ASC")
    return self
end

function QueryBuilder:limit(n)
    self._limit = n
    return self
end

function QueryBuilder:offset(n)
    self._offset = n
    return self
end

function QueryBuilder:build()
    local parts = {}
    
    -- SELECT
    local selectStr = #self._select > 0 and
        table.concat(self._select, ", ") or "*"
    parts[#parts+1] = "SELECT " .. selectStr
    
    -- FROM
    if self._from then
        parts[#parts+1] = "FROM " .. self._from
    end
    
    -- JOINs
    for _, j in ipairs(self._joins) do
        parts[#parts+1] = j
    end
    
    -- WHERE
    if #self._where > 0 then
        parts[#parts+1] = "WHERE " .. table.concat(self._where, " AND ")
    end
    
    -- GROUP BY
    if #self._groupBy > 0 then
        parts[#parts+1] = "GROUP BY " .. table.concat(self._groupBy, ", ")
    end
    
    -- HAVING
    if #self._having > 0 then
        parts[#parts+1] = "HAVING " .. table.concat(self._having, " AND ")
    end
    
    -- ORDER BY
    if #self._orderBy > 0 then
        parts[#parts+1] = "ORDER BY " .. table.concat(self._orderBy, ", ")
    end
    
    -- LIMIT / OFFSET
    if self._limit then
        parts[#parts+1] = "LIMIT " .. self._limit
    end
    if self._offset then
        parts[#parts+1] = "OFFSET " .. self._offset
    end
    
    return table.concat(parts, "\n")
end

function QueryBuilder:__tostring()
    return self:build()
end

-- Test
local query = QueryBuilder.new()
    :select("u.id", "u.name", "u.email", "COUNT(o.id) as order_count")
    :from("users", "u")
    :leftJoin("orders o", "o.user_id = u.id")
    :where("u.active = 1")
    :where("u.age >= 18")
    :groupBy("u.id", "u.name", "u.email")
    :having("COUNT(o.id) > 0")
    :orderBy("order_count", "DESC")
    :orderBy("u.name")
    :limit(10)
    :offset(20)

print(tostring(query))
```

---

## แบบฝึกหัด

### ระดับพื้นฐาน

1. **Student Grade System**: สร้าง class hierarchy:
   - `Person` (name, age, email)
   - `Student` extends `Person` (student_id, grades)
   - Methods: `addGrade(subject, score)`, `getGPA()`, `getReport()`
   - `Teacher` extends `Person` (subject, salary)
   - Methods: `assignGrade(student, subject, score)`, `getStudents()`

2. **Vehicle System**: สร้าง:
   - `Vehicle` (make, model, year, speed, fuel)
   - `Car` extends `Vehicle` (doors, gear)
   - `Truck` extends `Vehicle` (payload, axles)
   - `Motorcycle` extends `Vehicle` (type: sport/cruiser)
   - Abstract method `describe()` และ `fuelEfficiency()`

3. **Shape Calculator**: สร้าง shape hierarchy ที่:
   - `Shape` abstract class
   - `Circle`, `Rectangle`, `Triangle`, `Polygon`
   - Methods: `area()`, `perimeter()`, `scale(f)`, `translate(dx, dy)`
   - `ShapeCollection` ที่จัดการ list ของ shapes

### ระดับกลาง

4. **Inventory System**: สร้าง:
   - `Item` (name, price, quantity, category)
   - `Perishable` extends `Item` (expiry_date, batch)
   - `Electronics` extends `Item` (warranty, serial_number)
   - `Inventory` ที่จัดการ items ทั้งหมด
   - Methods: add, remove, search, restock, report

5. **Social Network Mini**: สร้าง:
   - `User` (profile, posts, friends)
   - `Post` (content, likes, comments, timestamp)
   - `Comment` extends `Post` (parent_post)
   - Methods: befriend, post, like, comment, feed generation

6. **Task Manager**: สร้าง:
   - `Task` (title, description, priority, status, due_date)
   - `Project` (tasks, team, deadline)
   - `Epic` extends `Project` (sub_projects)
   - Methods: assign, complete, prioritize, filter, report

### ระดับสูง

7. **Plugin-based Application**: สร้าง application framework ที่:
   - Core `Application` class
   - `Plugin` abstract class
   - Plugins ลงทะเบียน hooks และ commands
   - Application สามารถ load/unload plugins
   - Plugins สามารถ communicate กันได้

8. **Entity Component System (ECS)**: สร้าง:
   - `Entity` (id, components)
   - `Component` abstract class
   - `System` abstract class (process entities ด้วย certain components)
   - `World` จัดการ entities, components, systems
   - ตัวอย่าง: physics, rendering, health systems

9. **ORM Mini Framework**: สร้าง:
   - `Model` base class
   - Field definitions (IntField, StringField, BoolField)
   - `save()`, `find()`, `delete()`, `findAll()` methods
   - Relationships: `hasOne`, `hasMany`, `belongsTo`
   - Query builder integration

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ OOP ใน Lua:

| แนวคิด | การ Implement ใน Lua |
|--------|---------------------|
| Class | Table + `__index` |
| Object | `setmetatable({}, Class)` |
| Constructor | `Class.new()` ที่ return setmetatable |
| Instance Method | `function Class:method()` |
| Class Method | `function Class.method()` |
| Inheritance | `setmetatable(Child, {__index = Parent})` |
| Polymorphism | Override methods ใน subclass |
| Encapsulation | Locals ใน closure หรือ prefix `_` |
| Abstract | Error ใน base method |
| Interface | `implements()` function |

หลักการ OOP ที่ดีใน Lua:
1. ใช้ `__index` สำหรับ inheritance chain
2. เรียก constructor ของ parent ใน child constructor
3. ใช้ colon syntax `:` สำหรับ instance methods
4. ใช้ dot syntax `.` สำหรับ class methods
5. Prefix `_` หรือ local สำหรับ private members
6. เขียน `__tostring` เพื่อ debugging
7. ใช้ `pcall` สำหรับ robust error handling
