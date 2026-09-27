# บทที่ 22: OOP - Inheritance (การสืบทอด)

## บทนำ

การสืบทอด (Inheritance) เป็นหนึ่งในหลักการสำคัญของ Object-Oriented Programming (OOP) ที่ช่วยให้เราสามารถสร้าง class ใหม่จาก class ที่มีอยู่แล้ว โดยไม่ต้องเขียนโค้ดซ้ำ Lua ไม่มี built-in class system แต่เราสามารถสร้างระบบ OOP ได้ผ่าน metatables และ metamethods

ในบทนี้เราจะเรียนรู้:
- การสร้าง Single Inheritance
- การเรียกใช้ Super class methods
- Method overriding
- Abstract class และ Interface patterns
- Mixin basics
- Polymorphism
- Duck typing ใน Lua

---

## 22.1 พื้นฐาน OOP ใน Lua - ทบทวน

ก่อนเรียนเรื่อง Inheritance มาทบทวนการสร้าง class พื้นฐานก่อน

### ตัวอย่างที่ 1: Class พื้นฐาน

```lua
-- class พื้นฐานที่ใช้ metatable
local Animal = {}
Animal.__index = Animal

-- Constructor
function Animal.new(name, sound)
    local self = setmetatable({}, Animal)
    self.name = name
    self.sound = sound
    return self
end

-- Method
function Animal:speak()
    print(self.name .. " พูดว่า: " .. self.sound)
end

function Animal:getName()
    return self.name
end

-- สร้าง instance
local dog = Animal.new("หมา", "โฮ่ง")
local cat = Animal.new("แมว", "เมี๊ยว")

dog:speak()   -- หมา พูดว่า: โฮ่ง
cat:speak()   -- แมว พูดว่า: เมี๊ยว
print(dog:getName())  -- หมา
```

### ตัวอย่างที่ 2: Class พร้อม toString และ __tostring

```lua
local Person = {}
Person.__index = Person

function Person.new(name, age)
    local self = setmetatable({}, Person)
    self.name = name
    self.age = age
    return self
end

function Person:greet()
    print("สวัสดี ฉันชื่อ " .. self.name .. " อายุ " .. self.age .. " ปี")
end

function Person:__tostring()
    return "Person(" .. self.name .. ", " .. self.age .. ")"
end

-- ตั้ง __tostring ใน metatable
local p1 = Person.new("สมชาย", 25)
p1:greet()

-- ต้องการให้ tostring ทำงาน ต้องตั้ง metatable บน instance
setmetatable(p1, {
    __index = Person,
    __tostring = Person.__tostring
})
print(tostring(p1))  -- Person(สมชาย, 25)
```

### ตัวอย่างที่ 3: Class Factory Function ที่ดีกว่า

```lua
-- Pattern ที่ดีกว่า: ใส่ __tostring ใน class table โดยตรง
local function createClass(parent)
    local cls = {}
    cls.__index = cls
    
    if parent then
        setmetatable(cls, {__index = parent})
    end
    
    function cls:new(...)
        local instance = setmetatable({}, cls)
        if instance.init then
            instance:init(...)
        end
        return instance
    end
    
    return cls
end

-- ใช้งาน
local Vehicle = createClass()

function Vehicle:init(brand, speed)
    self.brand = brand
    self.speed = speed
end

function Vehicle:describe()
    print("ยานพาหนะ: " .. self.brand .. ", ความเร็ว: " .. self.speed .. " km/h")
end

local car = Vehicle:new("Toyota", 180)
car:describe()  -- ยานพาหนะ: Toyota, ความเร็ว: 180 km/h
```

---

## 22.2 Single Inheritance (การสืบทอดแบบชั้นเดียว)

### ตัวอย่างที่ 4: Inheritance พื้นฐาน

```lua
-- Base class: Animal
local Animal = {}
Animal.__index = Animal

function Animal.new(name, age)
    local self = setmetatable({}, Animal)
    self.name = name
    self.age = age
    return self
end

function Animal:breathe()
    print(self.name .. " หายใจ...")
end

function Animal:eat(food)
    print(self.name .. " กินอาหาร: " .. food)
end

function Animal:sleep()
    print(self.name .. " นอนหลับ...")
end

function Animal:toString()
    return "Animal(" .. self.name .. ", age=" .. self.age .. ")"
end

-- Derived class: Dog สืบทอดจาก Animal
local Dog = setmetatable({}, {__index = Animal})
Dog.__index = Dog

function Dog.new(name, age, breed)
    -- เรียก parent constructor
    local self = Animal.new(name, age)
    setmetatable(self, Dog)
    self.breed = breed
    return self
end

-- Method เพิ่มเติม (ไม่มีใน Animal)
function Dog:bark()
    print(self.name .. " เห่า: โฮ่ง โฮ่ง!")
end

function Dog:fetch(item)
    print(self.name .. " วิ่งไปเอา " .. item .. " กลับมา!")
end

-- สร้าง instance
local myDog = Dog.new("บัดดี้", 3, "Golden Retriever")

-- Methods จาก Dog
myDog:bark()
myDog:fetch("ลูกบอล")

-- Methods สืบทอดจาก Animal
myDog:breathe()
myDog:eat("เนื้อสุนัข")
myDog:sleep()

print(myDog:toString())  -- Animal(บัดดี้, age=3)
print(myDog.breed)       -- Golden Retriever
```

### ตัวอย่างที่ 5: Method Overriding

```lua
-- Base class
local Shape = {}
Shape.__index = Shape

function Shape.new(color)
    local self = setmetatable({}, Shape)
    self.color = color or "ขาว"
    return self
end

function Shape:area()
    return 0  -- base implementation
end

function Shape:perimeter()
    return 0  -- base implementation
end

function Shape:describe()
    print(string.format("รูปทรง: สี=%s, พื้นที่=%.2f, เส้นรอบรูป=%.2f",
        self.color, self:area(), self:perimeter()))
end

-- Circle สืบทอดจาก Shape และ override area(), perimeter()
local Circle = setmetatable({}, {__index = Shape})
Circle.__index = Circle

function Circle.new(radius, color)
    local self = Shape.new(color)
    setmetatable(self, Circle)
    self.radius = radius
    return self
end

-- Override area()
function Circle:area()
    return math.pi * self.radius ^ 2
end

-- Override perimeter()
function Circle:perimeter()
    return 2 * math.pi * self.radius
end

-- Rectangle สืบทอดจาก Shape
local Rectangle = setmetatable({}, {__index = Shape})
Rectangle.__index = Rectangle

function Rectangle.new(width, height, color)
    local self = Shape.new(color)
    setmetatable(self, Rectangle)
    self.width = width
    self.height = height
    return self
end

function Rectangle:area()
    return self.width * self.height
end

function Rectangle:perimeter()
    return 2 * (self.width + self.height)
end

-- ทดสอบ
local c = Circle.new(5, "แดง")
local r = Rectangle.new(4, 6, "น้ำเงิน")

c:describe()  -- รูปทรง: สี=แดง, พื้นที่=78.54, เส้นรอบรูป=31.42
r:describe()  -- รูปทรง: สี=น้ำเงิน, พื้นที่=24.00, เส้นรอบรูป=20.00
```

### ตัวอย่างที่ 6: เรียกใช้ Parent Method จาก Child

```lua
local Animal = {}
Animal.__index = Animal

function Animal.new(name)
    return setmetatable({name = name}, Animal)
end

function Animal:toString()
    return "Animal[" .. self.name .. "]"
end

function Animal:describe()
    print("ฉันคือ: " .. self:toString())
end

-- Dog override toString และเรียก parent method
local Dog = setmetatable({}, {__index = Animal})
Dog.__index = Dog

function Dog.new(name, breed)
    local self = Animal.new(name)
    setmetatable(self, Dog)
    self.breed = breed
    return self
end

-- Override toString และเรียก parent toString ด้วย
function Dog:toString()
    local parentStr = Animal.toString(self)  -- เรียก parent method โดยตรง
    return parentStr .. " (สายพันธุ์: " .. self.breed .. ")"
end

local d = Dog.new("แม็กซ์", "Labrador")
d:describe()  -- ฉันคือ: Animal[แม็กซ์] (สายพันธุ์: Labrador)
```

---

## 22.3 การเรียก Super Constructor

### ตัวอย่างที่ 7: Super Constructor Pattern

```lua
-- Base class: Employee
local Employee = {}
Employee.__index = Employee

function Employee.new(name, id, salary)
    local self = setmetatable({}, Employee)
    self.name = name
    self.id = id
    self.salary = salary
    print("  [Employee constructor] สร้าง Employee: " .. name)
    return self
end

function Employee:getInfo()
    return string.format("พนักงาน: %s (ID: %s), เงินเดือน: %.0f บาท",
        self.name, self.id, self.salary)
end

function Employee:getSalary()
    return self.salary
end

-- Manager สืบทอดจาก Employee
local Manager = setmetatable({}, {__index = Employee})
Manager.__index = Manager

function Manager.new(name, id, salary, department, teamSize)
    print("  [Manager constructor] กำลังสร้าง Manager: " .. name)
    -- เรียก super constructor
    local self = Employee.new(name, id, salary)
    setmetatable(self, Manager)
    self.department = department
    self.teamSize = teamSize
    return self
end

-- Override getInfo
function Manager:getInfo()
    -- เรียก parent getInfo
    local baseInfo = Employee.getInfo(self)
    return baseInfo .. string.format(", แผนก: %s, ทีม: %d คน",
        self.department, self.teamSize)
end

-- เพิ่ม bonus สำหรับ Manager
function Manager:getBonus()
    return self.salary * 0.2 * self.teamSize
end

-- Director สืบทอดจาก Manager (deep chain)
local Director = setmetatable({}, {__index = Manager})
Director.__index = Director

function Director.new(name, id, salary, department, teamSize, region)
    print("  [Director constructor] กำลังสร้าง Director: " .. name)
    local self = Manager.new(name, id, salary, department, teamSize)
    setmetatable(self, Director)
    self.region = region
    return self
end

function Director:getInfo()
    local managerInfo = Manager.getInfo(self)
    return managerInfo .. ", ภูมิภาค: " .. self.region
end

-- ทดสอบ
print("=== สร้าง Employee ===")
local emp = Employee.new("สมชาย", "E001", 30000)
print(emp:getInfo())

print("\n=== สร้าง Manager ===")
local mgr = Manager.new("วิชัย", "M001", 60000, "IT", 5)
print(mgr:getInfo())
print("โบนัส: " .. mgr:getBonus())

print("\n=== สร้าง Director ===")
local dir = Director.new("ประยุทธ์", "D001", 120000, "Operations", 20, "ภาคกลาง")
print(dir:getInfo())
```

### ตัวอย่างที่ 8: super() Pattern แบบทั่วไป

```lua
-- สร้าง helper function สำหรับ super()
local function createClass(parent)
    local cls = {}
    cls.__index = cls
    cls._parent = parent
    
    if parent then
        setmetatable(cls, {__index = parent})
    end
    
    -- super() function สำหรับ class นี้
    function cls:super(method, ...)
        if self._parent and self._parent[method] then
            return self._parent[method](self, ...)
        end
    end
    
    return cls
end

-- ใช้งาน
local Base = createClass(nil)

function Base:init(x)
    self.x = x
    print("Base:init(" .. tostring(x) .. ")")
end

function Base:getValue()
    return self.x
end

local Derived = createClass(Base)

function Derived:init(x, y)
    self:super("init", x)  -- เรียก Base:init
    self.y = y
    print("Derived:init(" .. tostring(x) .. ", " .. tostring(y) .. ")")
end

function Derived:getValue()
    local baseVal = self:super("getValue")  -- เรียก Base:getValue
    return baseVal + self.y
end

-- สร้าง instance
local d = setmetatable({_parent = Base}, {__index = Derived})
Derived.init(d, 10, 20)
print("getValue =", d:getValue())  -- 30
```

---

## 22.4 Protected Methods Pattern

### ตัวอย่างที่ 9: Protected Methods ด้วย Convention

```lua
-- ใน Lua ไม่มี access modifier แต่ใช้ naming convention
-- _ หน้าชื่อ = protected (ไม่ควรเรียกจากภายนอก)
-- __ หน้าชื่อ = private

local BankAccount = {}
BankAccount.__index = BankAccount

function BankAccount.new(owner, initialBalance)
    local self = setmetatable({}, BankAccount)
    self.owner = owner
    self.__balance = initialBalance or 0  -- private
    self._transactionLog = {}             -- protected
    return self
end

-- Protected method (เรียกได้จาก subclass)
function BankAccount:_logTransaction(type, amount)
    table.insert(self._transactionLog, {
        type = type,
        amount = amount,
        time = os.time()
    })
end

-- Protected method
function BankAccount:_validateAmount(amount)
    if amount <= 0 then
        error("จำนวนเงินต้องมากกว่า 0")
    end
    return true
end

-- Public methods
function BankAccount:deposit(amount)
    self:_validateAmount(amount)
    self.__balance = self.__balance + amount
    self:_logTransaction("deposit", amount)
    print(string.format("%s ฝากเงิน: %.2f บาท (ยอดคงเหลือ: %.2f)", 
        self.owner, amount, self.__balance))
end

function BankAccount:withdraw(amount)
    self:_validateAmount(amount)
    if amount > self.__balance then
        error("ยอดเงินไม่เพียงพอ")
    end
    self.__balance = self.__balance - amount
    self:_logTransaction("withdraw", amount)
    print(string.format("%s ถอนเงิน: %.2f บาท (ยอดคงเหลือ: %.2f)",
        self.owner, amount, self.__balance))
end

function BankAccount:getBalance()
    return self.__balance
end

function BankAccount:getTransactionCount()
    return #self._transactionLog
end

-- SavingsAccount สืบทอดจาก BankAccount
local SavingsAccount = setmetatable({}, {__index = BankAccount})
SavingsAccount.__index = SavingsAccount

function SavingsAccount.new(owner, initialBalance, interestRate)
    local self = BankAccount.new(owner, initialBalance)
    setmetatable(self, SavingsAccount)
    self.interestRate = interestRate or 0.05
    return self
end

-- Override deposit เพิ่ม logging พิเศษ
function SavingsAccount:deposit(amount)
    -- เรียก parent method
    BankAccount.deposit(self, amount)
    -- เพิ่ม log พิเศษ (เข้าถึง protected method ได้)
    self:_logTransaction("savings_deposit", amount)
end

-- เพิ่ม method ใหม่
function SavingsAccount:addInterest()
    local interest = self:getBalance() * self.interestRate
    self:deposit(interest)
    print(string.format("  (ดอกเบี้ย %.1f%%: +%.2f บาท)", 
        self.interestRate * 100, interest))
end

-- ทดสอบ
local acc = SavingsAccount.new("สมหญิง", 10000, 0.03)
acc:deposit(5000)
acc:withdraw(2000)
acc:addInterest()
print("ยอดคงเหลือ:", acc:getBalance())
print("จำนวนธุรกรรม:", acc:getTransactionCount())
```

---

## 22.5 Abstract Class Pattern

### ตัวอย่างที่ 10: Abstract Class ใน Lua

```lua
-- Abstract class: ไม่สามารถสร้าง instance โดยตรงได้
local AbstractShape = {}
AbstractShape.__index = AbstractShape

-- Constructor ที่ป้องกันการสร้าง instance โดยตรง
function AbstractShape.new(...)
    if getmetatable(AbstractShape) == nil or 
       getmetatable({}).__index ~= AbstractShape then
        -- ตรวจสอบว่าถูกเรียกจาก subclass
    end
    error("ไม่สามารถสร้าง instance ของ AbstractShape โดยตรงได้")
end

-- Abstract methods (ต้อง override)
function AbstractShape:area()
    error("ต้อง override method area() ใน subclass")
end

function AbstractShape:perimeter()
    error("ต้อง override method perimeter() ใน subclass")
end

-- Concrete method (ไม่ต้อง override)
function AbstractShape:describe()
    print(string.format("พื้นที่: %.4f, เส้นรอบรูป: %.4f", 
        self:area(), self:perimeter()))
end

function AbstractShape:isLargerThan(other)
    return self:area() > other:area()
end

-- Helper สำหรับสร้าง concrete class จาก abstract class
local function concreteClass(abstract)
    local cls = setmetatable({}, {__index = abstract})
    cls.__index = cls
    return cls
end

-- Concrete: Triangle
local Triangle = concreteClass(AbstractShape)

function Triangle.new(a, b, c)
    -- ตรวจสอบว่าสร้างสามเหลี่ยมได้
    assert(a + b > c and b + c > a and a + c > b, 
           "ความยาวด้านไม่สามารถสร้างสามเหลี่ยมได้")
    local self = setmetatable({}, Triangle)
    self.a, self.b, self.c = a, b, c
    return self
end

function Triangle:area()
    local s = (self.a + self.b + self.c) / 2
    return math.sqrt(s * (s-self.a) * (s-self.b) * (s-self.c))
end

function Triangle:perimeter()
    return self.a + self.b + self.c
end

-- Concrete: Circle
local Circle = concreteClass(AbstractShape)

function Circle.new(r)
    local self = setmetatable({}, Circle)
    self.radius = r
    return self
end

function Circle:area()
    return math.pi * self.radius ^ 2
end

function Circle:perimeter()
    return 2 * math.pi * self.radius
end

-- ทดสอบ
local t = Triangle.new(3, 4, 5)
local c = Circle.new(4)

t:describe()
c:describe()

print("สามเหลี่ยมใหญ่กว่าวงกลม?", t:isLargerThan(c))
print("วงกลมใหญ่กว่าสามเหลี่ยม?", c:isLargerThan(t))

-- ทดสอบว่า abstract method ให้ error
local ok, err = pcall(function()
    local s = setmetatable({}, AbstractShape)
    s:area()
end)
print("Error จาก abstract method:", err)
```

### ตัวอย่างที่ 11: Abstract Class ที่ดีกว่า (Template Method Pattern)

```lua
-- Template Method Pattern: abstract class กำหนด algorithm
local DataProcessor = {}
DataProcessor.__index = DataProcessor

-- Constructor
function DataProcessor.new(name)
    assert(getmetatable(DataProcessor) ~= DataProcessor,
           "DataProcessor เป็น abstract class")
    local self = setmetatable({}, DataProcessor)
    self.name = name
    self.data = {}
    return self
end

-- Abstract methods
function DataProcessor:loadData()
    error(self.name .. ": ต้อง override loadData()")
end

function DataProcessor:processItem(item)
    error(self.name .. ": ต้อง override processItem()")
end

function DataProcessor:saveResult(result)
    error(self.name .. ": ต้อง override saveResult()")
end

-- Template method (ไม่ควร override)
function DataProcessor:run()
    print("=== " .. self.name .. " เริ่มทำงาน ===")
    
    -- 1. Load data (abstract)
    self:loadData()
    print("โหลดข้อมูล " .. #self.data .. " รายการ")
    
    -- 2. Process each item (abstract)
    local results = {}
    for i, item in ipairs(self.data) do
        local result = self:processItem(item)
        table.insert(results, result)
    end
    
    -- 3. Save results (abstract)
    self:saveResult(results)
    
    -- 4. Log (concrete - ทำงานเสมอ)
    print("=== " .. self.name .. " เสร็จสิ้น (ประมวล " .. #results .. " รายการ) ===")
    return results
end

-- Concrete: NumberDoubler
local NumberDoubler = setmetatable({}, {__index = DataProcessor})
NumberDoubler.__index = NumberDoubler

function NumberDoubler.new(numbers)
    -- ไม่เรียก abstract constructor โดยตรง
    local self = setmetatable({}, NumberDoubler)
    self.name = "NumberDoubler"
    self.inputData = numbers
    self.data = {}
    return self
end

function NumberDoubler:loadData()
    for _, v in ipairs(self.inputData) do
        table.insert(self.data, v)
    end
end

function NumberDoubler:processItem(item)
    return item * 2
end

function NumberDoubler:saveResult(result)
    self.output = result
    io.write("ผลลัพธ์: ")
    for _, v in ipairs(result) do
        io.write(v .. " ")
    end
    print()
end

-- ทดสอบ
local processor = NumberDoubler.new({1, 2, 3, 4, 5})
processor:run()
```

---

## 22.6 Interface Pattern

### ตัวอย่างที่ 12: Interface ใน Lua

```lua
-- Interface ใน Lua ทำผ่าน table ที่มีแต่ method signatures
-- และ function ตรวจสอบว่า class implement interface ครบหรือไม่

-- สร้าง interface
local function createInterface(name, methods)
    local interface = {
        _name = name,
        _methods = methods
    }
    
    -- Function ตรวจสอบว่า class implement interface
    function interface.check(cls)
        local missing = {}
        for _, method in ipairs(methods) do
            if type(cls[method]) ~= "function" then
                table.insert(missing, method)
            end
        end
        if #missing > 0 then
            error(string.format("Class ไม่ได้ implement interface '%s': ขาด %s",
                name, table.concat(missing, ", ")))
        end
        return true
    end
    
    return interface
end

-- กำหนด interfaces
local Printable = createInterface("Printable", {"print", "toString"})
local Comparable = createInterface("Comparable", {"compareTo", "equals"})
local Serializable = createInterface("Serializable", {"serialize", "deserialize"})

-- Class ที่ implement interfaces
local Product = {}
Product.__index = Product

function Product.new(id, name, price)
    local self = setmetatable({}, Product)
    self.id = id
    self.name = name
    self.price = price
    return self
end

-- Implement Printable
function Product:print()
    print(string.format("[%s] %s - %.2f บาท", self.id, self.name, self.price))
end

function Product:toString()
    return string.format("Product{id=%s, name=%s, price=%.2f}", 
        self.id, self.name, self.price)
end

-- Implement Comparable
function Product:compareTo(other)
    if self.price < other.price then return -1
    elseif self.price > other.price then return 1
    else return 0 end
end

function Product:equals(other)
    return self.id == other.id
end

-- Implement Serializable
function Product:serialize()
    return string.format("%s|%s|%.2f", self.id, self.name, self.price)
end

function Product.deserialize(data)
    local id, name, price = data:match("([^|]+)|([^|]+)|([^|]+)")
    return Product.new(id, name, tonumber(price))
end

-- ตรวจสอบ interfaces
local ok1 = Printable.check(Product)
local ok2 = Comparable.check(Product)
local ok3 = Serializable.check(Product)
print("Product implements Printable:", ok1)
print("Product implements Comparable:", ok2)
print("Product implements Serializable:", ok3)

-- ทดสอบ
local p1 = Product.new("P001", "แล็ปท็อป", 35000)
local p2 = Product.new("P002", "เมาส์", 500)

p1:print()
p2:print()

print("p1 เปรียบกับ p2:", p1:compareTo(p2))  -- 1 (p1 แพงกว่า)

local serialized = p1:serialize()
print("Serialized:", serialized)

local restored = Product.deserialize(serialized)
restored:print()
```

### ตัวอย่างที่ 13: Interface ที่ซับซ้อนขึ้น - Duck Typing

```lua
-- Duck Typing: "ถ้ามันเดินเหมือนเป็ด และร้องเหมือนเป็ด มันก็คือเป็ด"
-- ใน Lua เราตรวจสอบ interface ผ่าน duck typing

-- Function ที่ทำงานกับ "anything that can fly"
local function makeItFly(obj)
    -- ตรวจสอบ interface ก่อน
    assert(type(obj.fly) == "function", "Object ต้องมี method fly()")
    assert(type(obj.land) == "function", "Object ต้องมี method land()")
    
    print("กำลังทำให้บินขึ้น...")
    obj:fly()
    print("กำลังลงจอด...")
    obj:land()
end

-- Bird
local Bird = {}
Bird.__index = Bird
function Bird.new(name) return setmetatable({name=name}, Bird) end
function Bird:fly() print(self.name .. " บินโดยใช้ปีก") end
function Bird:land() print(self.name .. " ลงจอดด้วยขา") end

-- Airplane
local Airplane = {}
Airplane.__index = Airplane
function Airplane.new(model) return setmetatable({model=model}, Airplane) end
function Airplane:fly() print(self.model .. " บินขึ้นด้วยเครื่องยนต์") end
function Airplane:land() print(self.model .. " ลงจอดด้วย landing gear") end

-- Superhero (ไม่มี land!)
local Superhero = {}
Superhero.__index = Superhero
function Superhero.new(name) return setmetatable({name=name}, Superhero) end
function Superhero:fly() print(self.name .. " บินด้วยพลังพิเศษ") end

local bird = Bird.new("นกอินทรี")
local plane = Airplane.new("Boeing 737")
local hero = Superhero.new("Superman")

makeItFly(bird)
print()
makeItFly(plane)
print()

-- นี่จะ error เพราะ Superhero ไม่มี land()
local ok, err = pcall(makeItFly, hero)
print("Error:", err)
```

---

## 22.7 Mixin Pattern

### ตัวอย่างที่ 14: Mixin พื้นฐาน

```lua
-- Mixin: เพิ่ม functionality ให้กับ class โดยไม่ต้องใช้ inheritance
local function mixin(target, source)
    for key, value in pairs(source) do
        if type(value) == "function" and key ~= "new" then
            target[key] = value
        end
    end
    return target
end

-- Mixins
local Timestamped = {
    getCreatedAt = function(self)
        return self._createdAt or "ไม่ทราบ"
    end,
    setCreatedAt = function(self, time)
        self._createdAt = time
    end,
    touch = function(self)
        self._updatedAt = os.time()
    end,
    getUpdatedAt = function(self)
        return self._updatedAt
    end
}

local Taggable = {
    addTag = function(self, tag)
        self._tags = self._tags or {}
        table.insert(self._tags, tag)
    end,
    removeTag = function(self, tag)
        if not self._tags then return end
        for i, t in ipairs(self._tags) do
            if t == tag then
                table.remove(self._tags, i)
                return
            end
        end
    end,
    hasTag = function(self, tag)
        if not self._tags then return false end
        for _, t in ipairs(self._tags) do
            if t == tag then return true end
        end
        return false
    end,
    getTags = function(self)
        return self._tags or {}
    end
}

local Printable = {
    printInfo = function(self)
        print("Object info:")
        for k, v in pairs(self) do
            if type(v) ~= "function" then
                print("  " .. tostring(k) .. " = " .. tostring(v))
            end
        end
    end
}

-- Class ที่ใช้ Mixins
local Article = {}
Article.__index = Article

-- ใส่ mixins
mixin(Article, Timestamped)
mixin(Article, Taggable)
mixin(Article, Printable)

function Article.new(title, content)
    local self = setmetatable({}, Article)
    self.title = title
    self.content = content
    self:setCreatedAt(os.time())
    return self
end

function Article:preview()
    local preview = self.content:sub(1, 50)
    if #self.content > 50 then preview = preview .. "..." end
    print(string.format("[%s] %s", self.title, preview))
end

-- ทดสอบ
local article = Article.new("Lua Programming", 
    "Lua เป็นภาษาโปรแกรมที่ทรงพลังและเบา เหมาะสำหรับการฝังตัวในแอปพลิเคชัน")

article:preview()
article:addTag("programming")
article:addTag("lua")
article:addTag("tutorial")
article:touch()

print("Tags:", table.concat(article:getTags(), ", "))
print("มี tag 'lua'?", article:hasTag("lua"))
print("มี tag 'python'?", article:hasTag("python"))
article:removeTag("tutorial")
print("Tags หลังลบ:", table.concat(article:getTags(), ", "))
```

---

## 22.8 isinstance / is_a Checks

### ตัวอย่างที่ 15: Type Checking ใน Lua

```lua
-- สร้าง type checking system
local function createClass(name, parent)
    local cls = {}
    cls.__index = cls
    cls._name = name
    cls._parent = parent
    
    if parent then
        setmetatable(cls, {__index = parent})
    end
    
    function cls:isA(targetClass)
        local mt = getmetatable(self)
        while mt do
            if mt == targetClass then return true end
            mt = mt._parent
        end
        return false
    end
    
    function cls:getClass()
        return getmetatable(self)
    end
    
    function cls:getClassName()
        local mt = getmetatable(self)
        return mt and mt._name or "unknown"
    end
    
    return cls
end

-- Type hierarchy
local Animal = createClass("Animal")
local Mammal = createClass("Mammal", Animal)
local Dog = createClass("Dog", Mammal)
local Cat = createClass("Cat", Mammal)
local GoldenRetriever = createClass("GoldenRetriever", Dog)

-- Constructors
function Animal.new(cls, name)
    return setmetatable({name = name}, cls)
end

function Animal:speak()
    print(self.name .. " ส่งเสียง")
end

function Dog:speak()
    print(self.name .. " เห่า: โฮ่ง!")
end

function Cat:speak()
    print(self.name .. " ร้อง: เมี๊ยว!")
end

-- สร้าง instances
local a = Animal.new(Animal, "สัตว์ทั่วไป")
local d = Animal.new(Dog, "บัดดี้")
local c = Animal.new(Cat, "วิสกี้")
local gr = Animal.new(GoldenRetriever, "โกลเด้น")

-- ทดสอบ is_a
print("=== Type Checking ===")
print(d:getClassName(), "isA Dog?", d:isA(Dog))           -- true
print(d:getClassName(), "isA Mammal?", d:isA(Mammal))     -- true
print(d:getClassName(), "isA Animal?", d:isA(Animal))     -- true
print(d:getClassName(), "isA Cat?", d:isA(Cat))           -- false
print(c:getClassName(), "isA Cat?", c:isA(Cat))           -- true
print(gr:getClassName(), "isA Dog?", gr:isA(Dog))         -- true
print(gr:getClassName(), "isA GoldenRetriever?", gr:isA(GoldenRetriever))  -- true
print()

-- ทดสอบ polymorphism
local animals = {a, d, c, gr}
print("=== Polymorphism ===")
for _, animal in ipairs(animals) do
    io.write(animal:getClassName() .. ": ")
    animal:speak()
end
```

### ตัวอย่างที่ 16: instanceof Function

```lua
-- instanceof function แบบ generic
local function instanceof(obj, cls)
    local mt = getmetatable(obj)
    while mt do
        if mt == cls then return true end
        -- ตรวจสอบ parent class
        local parent = rawget(mt, "_parent")
        if parent then
            mt = parent
        else
            -- ลองผ่าน metatable chain
            local meta = getmetatable(mt)
            if meta and meta.__index then
                mt = meta.__index
            else
                break
            end
        end
    end
    return false
end

-- Test
local Vehicle = {}
Vehicle.__index = Vehicle
Vehicle._name = "Vehicle"

local Car = setmetatable({}, {__index = Vehicle})
Car.__index = Car
Car._name = "Car"
Car._parent = Vehicle

local Tesla = setmetatable({}, {__index = Car})
Tesla.__index = Tesla
Tesla._name = "Tesla"
Tesla._parent = Car

function Vehicle.new(cls, model)
    return setmetatable({model = model}, cls)
end

local v = Vehicle.new(Vehicle, "Generic Vehicle")
local c = Vehicle.new(Car, "Honda Civic")
local t = Vehicle.new(Tesla, "Model S")

print(c.model, "instanceof Car:", instanceof(c, Car))        -- true
print(c.model, "instanceof Vehicle:", instanceof(c, Vehicle)) -- true
print(c.model, "instanceof Tesla:", instanceof(c, Tesla))     -- false
print(t.model, "instanceof Tesla:", instanceof(t, Tesla))     -- true
print(t.model, "instanceof Car:", instanceof(t, Car))         -- true
print(t.model, "instanceof Vehicle:", instanceof(t, Vehicle)) -- true
```

---

## 22.9 Deep Inheritance Chains

### ตัวอย่างที่ 17: Deep Inheritance - ห่วงโซ่ยาว

```lua
-- สร้าง deep hierarchy
local function makeClass(name, parent)
    local cls = {}
    cls.__index = cls
    cls.__name = name
    
    if parent then
        setmetatable(cls, {__index = parent})
        cls.__parent = parent
    end
    
    function cls.new(...)
        local self = setmetatable({}, cls)
        if self.init then self:init(...) end
        return self
    end
    
    return cls
end

-- Hierarchy: LivingThing -> Eukaryote -> Animal -> Vertebrate -> Mammal -> Primate -> Human
local LivingThing = makeClass("LivingThing")
function LivingThing:init(name)
    self.name = name
    self.alive = true
end
function LivingThing:breathe()
    print(self.name .. " ดำรงชีวิตอยู่")
end

local Eukaryote = makeClass("Eukaryote", LivingThing)
function Eukaryote:hasCellNucleus()
    return true
end

local Animal = makeClass("Animal", Eukaryote)
function Animal:init(name, legs)
    LivingThing.init(self, name)
    self.legs = legs or 0
end
function Animal:move()
    print(self.name .. " เคลื่อนที่ด้วย " .. self.legs .. " ขา")
end

local Vertebrate = makeClass("Vertebrate", Animal)
function Vertebrate:hasSpine()
    return true
end

local Mammal = makeClass("Mammal", Vertebrate)
function Mammal:init(name, legs, furColor)
    Animal.init(self, name, legs)
    self.furColor = furColor or "น้ำตาล"
end
function Mammal:feedYoung()
    print(self.name .. " เลี้ยงลูกด้วยนม")
end

local Primate = makeClass("Primate", Mammal)
function Primate:useTool()
    print(self.name .. " ใช้เครื่องมือได้")
end

local Human = makeClass("Human", Primate)
function Human:init(name, job)
    Mammal.init(self, name, 2, "ไม่มีขน")
    self.job = job or "ว่างงาน"
end
function Human:speak(text)
    print(self.name .. " พูดว่า: " .. text)
end
function Human:work()
    print(self.name .. " ทำงาน: " .. self.job)
end

-- สร้าง instance
local bob = Human.new("บ็อบ", "โปรแกรมเมอร์")

-- เรียก methods จากทุก level
bob:breathe()      -- จาก LivingThing
bob:move()         -- จาก Animal
bob:feedYoung()    -- จาก Mammal
bob:useTool()      -- จาก Primate
bob:speak("สวัสดี ฉันเป็น Human!")  -- จาก Human
bob:work()         -- จาก Human

print("มีกระดูกสันหลัง:", bob:hasSpine())      -- true
print("มี cell nucleus:", bob:hasCellNucleus()) -- true
print("ยังมีชีวิต:", bob.alive)                  -- true
print("จำนวนขา:", bob.legs)                     -- 2
```

---

## 22.10 Polymorphism

### ตัวอย่างที่ 18: Polymorphism แบบสมบูรณ์

```lua
-- Polymorphism: object หลายประเภทที่ใช้ interface เดียวกัน

-- Base class
local Shape = {}
Shape.__index = Shape

function Shape.new(name, color)
    return setmetatable({
        shapeName = name,
        color = color or "ขาว"
    }, Shape)
end

function Shape:area() return 0 end
function Shape:perimeter() return 0 end
function Shape:draw()
    print(string.format("วาด %s สี%s (พื้นที่: %.2f)", 
        self.shapeName, self.color, self:area()))
end

-- Circle
local Circle = setmetatable({}, {__index = Shape})
Circle.__index = Circle
function Circle.new(r, color)
    local s = Shape.new("วงกลม", color)
    setmetatable(s, Circle)
    s.radius = r
    return s
end
function Circle:area() return math.pi * self.radius^2 end
function Circle:perimeter() return 2 * math.pi * self.radius end

-- Rectangle
local Rectangle = setmetatable({}, {__index = Shape})
Rectangle.__index = Rectangle
function Rectangle.new(w, h, color)
    local s = Shape.new("สี่เหลี่ยม", color)
    setmetatable(s, Rectangle)
    s.w, s.h = w, h
    return s
end
function Rectangle:area() return self.w * self.h end
function Rectangle:perimeter() return 2 * (self.w + self.h) end

-- Triangle
local Triangle = setmetatable({}, {__index = Shape})
Triangle.__index = Triangle
function Triangle.new(a, b, c, color)
    local s = Shape.new("สามเหลี่ยม", color)
    setmetatable(s, Triangle)
    s.a, s.b, s.c = a, b, c
    return s
end
function Triangle:area()
    local p = (self.a + self.b + self.c) / 2
    return math.sqrt(p * (p-self.a) * (p-self.b) * (p-self.c))
end
function Triangle:perimeter() return self.a + self.b + self.c end

-- Hexagon
local Hexagon = setmetatable({}, {__index = Shape})
Hexagon.__index = Hexagon
function Hexagon.new(side, color)
    local s = Shape.new("หกเหลี่ยม", color)
    setmetatable(s, Hexagon)
    s.side = side
    return s
end
function Hexagon:area() return (3 * math.sqrt(3) / 2) * self.side^2 end
function Hexagon:perimeter() return 6 * self.side end

-- Polymorphic function
local function totalArea(shapes)
    local total = 0
    for _, shape in ipairs(shapes) do
        total = total + shape:area()
    end
    return total
end

local function sortByArea(shapes)
    local sorted = {table.unpack(shapes)}
    table.sort(sorted, function(a, b) return a:area() < b:area() end)
    return sorted
end

local function printAllShapes(shapes)
    for _, shape in ipairs(shapes) do
        shape:draw()
    end
end

-- สร้าง shapes หลายแบบ
local shapes = {
    Circle.new(5, "แดง"),
    Rectangle.new(4, 6, "น้ำเงิน"),
    Triangle.new(3, 4, 5, "เขียว"),
    Hexagon.new(3, "เหลือง"),
    Circle.new(2, "ม่วง"),
    Rectangle.new(10, 2, "ส้ม"),
}

print("=== รูปทรงทั้งหมด ===")
printAllShapes(shapes)

print(string.format("\nพื้นที่รวม: %.2f", totalArea(shapes)))

print("\n=== เรียงตามพื้นที่ (น้อยไปมาก) ===")
local sorted = sortByArea(shapes)
printAllShapes(sorted)
```

---

## 22.11 Diamond Problem

### ตัวอย่างที่ 19: Diamond Problem และวิธีแก้

```lua
-- Diamond Problem:
--      A
--     / \
--    B   C
--     \ /
--      D
-- D สืบทอดจากทั้ง B และ C ซึ่งทั้งคู่สืบทอดจาก A

-- ปัญหา: D:method() ควรจะเรียก B หรือ C?

-- วิธีที่ 1: Last-write-wins (Lua default)
local A = {}; A.__index = A
function A:hello() print("A:hello") end
function A:value() return "A" end

local B = setmetatable({}, {__index = A}); B.__index = B
function B:hello() print("B:hello") end
function B:fromB() print("ฉันมาจาก B") end

local C = setmetatable({}, {__index = A}); C.__index = C
function C:hello() print("C:hello") end
function C:fromC() print("ฉันมาจาก C") end

-- D ใช้ multiple inheritance แบบ copy
local D = {}
D.__index = D

-- Copy methods จาก B ก่อน แล้วตามด้วย C (C จะ override B)
for k, v in pairs(B) do
    if type(v) == "function" then D[k] = v end
end
for k, v in pairs(C) do
    if type(v) == "function" then D[k] = v end
end

function D.new()
    return setmetatable({}, D)
end

-- D มี method ของตัวเอง
function D:hello()
    -- เรียกทั้ง B และ C
    B.hello(self)
    C.hello(self)
    print("D:hello")
end

local d = D.new()
d:hello()   -- B:hello, C:hello, D:hello
d:fromB()   -- ฉันมาจาก B
d:fromC()   -- ฉันมาจาก C

print("\n--- Diamond Problem Solution ---")

-- วิธีที่ 2: C3 Linearization (คล้าย Python MRO)
local function computeMRO(cls)
    local mro = {cls}
    local bases = cls._bases or {}
    
    for _, base in ipairs(bases) do
        local baseMRO = computeMRO(base)
        for _, c in ipairs(baseMRO) do
            -- เพิ่มเฉพาะที่ยังไม่มี
            local found = false
            for _, existing in ipairs(mro) do
                if existing == c then found = true; break end
            end
            if not found then
                table.insert(mro, c)
            end
        end
    end
    
    return mro
end

-- สร้าง class system ที่รองรับ MRO
local function mkClass(name, ...)
    local bases = {...}
    local cls = {_name = name, _bases = bases}
    cls.__index = cls
    
    -- สร้าง MRO
    cls._mro = computeMRO(cls)
    
    -- ตั้ง __index chain
    if #bases > 0 then
        -- ใช้ function สำหรับ __index เพื่อ search ตาม MRO
        setmetatable(cls, {
            __index = function(t, k)
                for i = 2, #cls._mro do
                    local v = rawget(cls._mro[i], k)
                    if v ~= nil then return v end
                end
            end
        })
    end
    
    return cls
end

local AA = mkClass("AA")
function AA:method() print("AA:method") end
function AA:shared() print("AA:shared") end

local BB = mkClass("BB", AA)
function BB:method() print("BB:method") end

local CC = mkClass("CC", AA)
function CC:method() print("CC:method") end
function CC:shared() print("CC:shared (override)") end

local DD = mkClass("DD", BB, CC)
function DD.new()
    return setmetatable({}, DD)
end

-- ทดสอบ MRO
print("MRO ของ DD:")
for i, cls in ipairs(DD._mro) do
    print(i, cls._name)
end

local dd = DD.new()
-- DD ไม่มี method ของตัวเอง จะค้นหาตาม MRO
-- ในกรณีนี้ B จะมาก่อน C
```

---

## 22.12 Type Hierarchy และ Class Registry

### ตัวอย่างที่ 20: Class Registry

```lua
-- Class Registry: เก็บรายการ class ทั้งหมด
local ClassRegistry = {}
ClassRegistry._classes = {}

function ClassRegistry.register(cls, name)
    assert(name, "ต้องระบุชื่อ class")
    ClassRegistry._classes[name] = cls
    cls._className = name
    return cls
end

function ClassRegistry.get(name)
    return ClassRegistry._classes[name]
end

function ClassRegistry.list()
    local names = {}
    for name in pairs(ClassRegistry._classes) do
        table.insert(names, name)
    end
    table.sort(names)
    return names
end

function ClassRegistry.isRegistered(cls)
    for _, c in pairs(ClassRegistry._classes) do
        if c == cls then return true end
    end
    return false
end

-- สร้าง classes และ register
local Animal = {}
Animal.__index = Animal
ClassRegistry.register(Animal, "Animal")

function Animal.new(name)
    return setmetatable({name = name}, Animal)
end
function Animal:speak() print(self.name .. " ส่งเสียง") end

local Dog = setmetatable({}, {__index = Animal})
Dog.__index = Dog
ClassRegistry.register(Dog, "Dog")

function Dog.new(name, breed)
    local self = Animal.new(name)
    setmetatable(self, Dog)
    self.breed = breed
    return self
end
function Dog:speak() print(self.name .. " เห่า") end
function Dog:fetch() print(self.name .. " คาบกลับมา") end

local Cat = setmetatable({}, {__index = Animal})
Cat.__index = Cat
ClassRegistry.register(Cat, "Cat")

function Cat.new(name)
    return setmetatable(Animal.new(name), Cat)
end
function Cat:speak() print(self.name .. " ร้องเมี๊ยว") end
function Cat:purr() print(self.name .. " ครั้น") end

-- ทดสอบ Registry
print("Classes ที่ลงทะเบียน:")
for _, name in ipairs(ClassRegistry.list()) do
    print("  -", name)
end

-- สร้าง instance จาก class name
local function createByName(className, ...)
    local cls = ClassRegistry.get(className)
    assert(cls, "ไม่พบ class: " .. className)
    return cls.new(...)
end

local d = createByName("Dog", "แม็กซ์", "Labrador")
local c = createByName("Cat", "วิสกี้")

d:speak()
d:fetch()
c:speak()
c:purr()
```

---

## 22.13 ตัวอย่างโครงการจริง: Game Entity System

### ตัวอย่างที่ 21-30: Game Entity Hierarchy

```lua
-- Game Entity System: ตัวอย่าง OOP จริงในเกม

-- ===== Base Entity =====
local Entity = {}
Entity.__index = Entity
Entity._name = "Entity"

function Entity.new(id, x, y)
    return setmetatable({
        id = id,
        x = x or 0,
        y = y or 0,
        active = true,
        tags = {}
    }, Entity)
end

function Entity:moveTo(x, y)
    self.x, self.y = x, y
end

function Entity:moveBy(dx, dy)
    self.x = self.x + dx
    self.y = self.y + dy
end

function Entity:getPosition()
    return self.x, self.y
end

function Entity:distanceTo(other)
    local dx = self.x - other.x
    local dy = self.y - other.y
    return math.sqrt(dx*dx + dy*dy)
end

function Entity:addTag(tag)
    self.tags[tag] = true
end

function Entity:hasTag(tag)
    return self.tags[tag] == true
end

function Entity:destroy()
    self.active = false
end

function Entity:toString()
    return string.format("%s[%s] @(%.1f, %.1f)", 
        self._name or "Entity", self.id, self.x, self.y)
end

-- ===== Drawable Entity =====
local DrawableEntity = setmetatable({}, {__index = Entity})
DrawableEntity.__index = DrawableEntity
DrawableEntity._name = "DrawableEntity"

function DrawableEntity.new(id, x, y, sprite)
    local self = Entity.new(id, x, y)
    setmetatable(self, DrawableEntity)
    self.sprite = sprite or "default"
    self.visible = true
    self.rotation = 0
    self.scale = 1.0
    return self
end

function DrawableEntity:draw()
    if self.visible then
        print(string.format("วาด %s: sprite=%s, pos=(%.1f,%.1f), rot=%.1f°",
            self:toString(), self.sprite, self.x, self.y, self.rotation))
    end
end

function DrawableEntity:hide() self.visible = false end
function DrawableEntity:show() self.visible = true end
function DrawableEntity:rotate(angle) self.rotation = (self.rotation + angle) % 360 end
function DrawableEntity:setScale(s) self.scale = s end

-- ===== Collidable Entity =====
local CollidableEntity = setmetatable({}, {__index = DrawableEntity})
CollidableEntity.__index = CollidableEntity
CollidableEntity._name = "CollidableEntity"

function CollidableEntity.new(id, x, y, sprite, width, height)
    local self = DrawableEntity.new(id, x, y, sprite)
    setmetatable(self, CollidableEntity)
    self.width = width or 32
    self.height = height or 32
    self.solid = true
    return self
end

function CollidableEntity:getBounds()
    return {
        left = self.x - self.width/2,
        right = self.x + self.width/2,
        top = self.y - self.height/2,
        bottom = self.y + self.height/2
    }
end

function CollidableEntity:intersects(other)
    if not other.getBounds then return false end
    local a = self:getBounds()
    local b = other:getBounds()
    return not (a.right < b.left or b.right < a.left or
                a.bottom < b.top or b.bottom < a.top)
end

-- ===== Living Entity =====
local LivingEntity = setmetatable({}, {__index = CollidableEntity})
LivingEntity.__index = LivingEntity
LivingEntity._name = "LivingEntity"

function LivingEntity.new(id, x, y, sprite, hp, maxHp)
    local self = CollidableEntity.new(id, x, y, sprite, 32, 48)
    setmetatable(self, LivingEntity)
    self.maxHp = maxHp or 100
    self.hp = hp or self.maxHp
    self.alive = true
    return self
end

function LivingEntity:takeDamage(amount)
    self.hp = math.max(0, self.hp - amount)
    print(string.format("%s รับความเสียหาย %d HP (เหลือ %d/%d)",
        self.id, amount, self.hp, self.maxHp))
    if self.hp <= 0 then
        self:die()
    end
end

function LivingEntity:heal(amount)
    self.hp = math.min(self.maxHp, self.hp + amount)
    print(string.format("%s ฟื้นฟู %d HP (ปัจจุบัน %d/%d)",
        self.id, amount, self.hp, self.maxHp))
end

function LivingEntity:die()
    self.alive = false
    self.active = false
    print(self.id .. " เสียชีวิต!")
end

function LivingEntity:isAlive()
    return self.alive and self.hp > 0
end

-- ===== Player =====
local Player = setmetatable({}, {__index = LivingEntity})
Player.__index = Player
Player._name = "Player"

function Player.new(id, x, y)
    local self = LivingEntity.new(id, x, y, "player_sprite", 100, 100)
    setmetatable(self, Player)
    self.speed = 5
    self.level = 1
    self.exp = 0
    self.expToNext = 100
    self.inventory = {}
    return self
end

function Player:gainExp(amount)
    self.exp = self.exp + amount
    print(string.format("%s ได้รับ %d EXP (รวม %d/%d)",
        self.id, amount, self.exp, self.expToNext))
    while self.exp >= self.expToNext do
        self:levelUp()
    end
end

function Player:levelUp()
    self.exp = self.exp - self.expToNext
    self.level = self.level + 1
    self.expToNext = self.expToNext * 1.5
    self.maxHp = self.maxHp + 20
    self.hp = self.maxHp
    print(string.format("*** %s เลเวลอัป! เลเวล %d ***", self.id, self.level))
end

function Player:pickupItem(item)
    table.insert(self.inventory, item)
    print(self.id .. " เก็บ: " .. item)
end

-- ===== Enemy =====
local Enemy = setmetatable({}, {__index = LivingEntity})
Enemy.__index = Enemy
Enemy._name = "Enemy"

function Enemy.new(id, x, y, enemyType, hp, damage)
    local self = LivingEntity.new(id, x, y, "enemy_" .. enemyType, hp, hp)
    setmetatable(self, Enemy)
    self.enemyType = enemyType
    self.damage = damage or 10
    self.expReward = hp / 2
    self.target = nil
    return self
end

function Enemy:setTarget(player)
    self.target = player
end

function Enemy:attack()
    if self.target and self.target:isAlive() then
        print(string.format("%s (%s) โจมตี %s สร้างความเสียหาย %d",
            self.id, self.enemyType, self.target.id, self.damage))
        self.target:takeDamage(self.damage)
    end
end

function Enemy:die()
    LivingEntity.die(self)  -- เรียก parent
    if self.target then
        self.target:gainExp(self.expReward)
    end
end

-- ===== Boss Enemy =====
local Boss = setmetatable({}, {__index = Enemy})
Boss.__index = Boss
Boss._name = "Boss"

function Boss.new(id, x, y, bossName, hp, damage)
    local self = Enemy.new(id, x, y, "boss", hp, damage)
    setmetatable(self, Boss)
    self.bossName = bossName
    self.phase = 1
    self.maxPhase = 3
    return self
end

function Boss:takeDamage(amount)
    -- Override: Boss อาจเปลี่ยน phase
    LivingEntity.takeDamage(self, amount)
    
    local hpPercent = self.hp / self.maxHp
    if hpPercent < 0.33 and self.phase < 3 then
        self.phase = 3
        print("*** " .. self.bossName .. " เข้าสู่ Phase 3! ***")
        self.damage = self.damage * 2
    elseif hpPercent < 0.66 and self.phase < 2 then
        self.phase = 2
        print("*** " .. self.bossName .. " เข้าสู่ Phase 2! ***")
        self.damage = math.floor(self.damage * 1.5)
    end
end

-- ===== ทดสอบ Game System =====
print("=== Game Simulation ===\n")

local hero = Player.new("วีรชาติ", 0, 0)
local goblin = Enemy.new("Goblin1", 5, 0, "goblin", 30, 8)
local dragon = Boss.new("Dragon1", 10, 0, "มังกรไฟ", 200, 25)

goblin:setTarget(hero)
dragon:setTarget(hero)

print("--- เริ่มการต่อสู้ ---")
hero:takeDamage(15)
goblin:attack()
hero:takeDamage(30)

print("\n--- โจมตี Goblin ---")
goblin:takeDamage(25)
goblin:takeDamage(10)  -- ตาย และ hero ได้ EXP

print("\n--- ต่อสู้กับ Dragon ---")
hero:heal(50)
dragon:takeDamage(80)   -- Phase 2
dragon:attack()
dragon:takeDamage(80)   -- Phase 3
dragon:attack()
dragon:takeDamage(50)   -- ตาย

print("\n--- สถานะสุดท้าย ---")
print(string.format("Hero: Level %d, HP %d/%d, EXP %d",
    hero.level, hero.hp, hero.maxHp, hero.exp))
```

---

## 22.14 ตัวอย่างเพิ่มเติม: GUI Widget Hierarchy

### ตัวอย่างที่ 31-35: GUI Widget System

```lua
-- GUI Widget OOP System (Simulated)

local Widget = {}
Widget.__index = Widget

function Widget.new(id, x, y, w, h)
    return setmetatable({
        id = id,
        x = x, y = y,
        width = w, height = h,
        visible = true,
        enabled = true,
        parent = nil,
        children = {},
        eventHandlers = {}
    }, Widget)
end

function Widget:addChild(child)
    table.insert(self.children, child)
    child.parent = self
end

function Widget:on(event, handler)
    self.eventHandlers[event] = handler
end

function Widget:emit(event, ...)
    if self.eventHandlers[event] then
        self.eventHandlers[event](self, ...)
    end
end

function Widget:render()
    if self.visible then
        print(string.format("[Widget:%s] @(%d,%d) %dx%d",
            self.id, self.x, self.y, self.width, self.height))
        for _, child in ipairs(self.children) do
            child:render()
        end
    end
end

-- Container Widget
local Container = setmetatable({}, {__index = Widget})
Container.__index = Container

function Container.new(id, x, y, w, h, layout)
    local self = Widget.new(id, x, y, w, h)
    setmetatable(self, Container)
    self.layout = layout or "vertical"
    self.padding = 5
    return self
end

function Container:render()
    if self.visible then
        print(string.format("[Container:%s] layout=%s @(%d,%d) %dx%d",
            self.id, self.layout, self.x, self.y, self.width, self.height))
        local offsetY = self.y + self.padding
        for _, child in ipairs(self.children) do
            child.x = self.x + self.padding
            child.y = offsetY
            child:render()
            if self.layout == "vertical" then
                offsetY = offsetY + child.height + self.padding
            end
        end
    end
end

-- Button Widget
local Button = setmetatable({}, {__index = Widget})
Button.__index = Button

function Button.new(id, x, y, w, h, text)
    local self = Widget.new(id, x, y, w, h)
    setmetatable(self, Button)
    self.text = text or "Button"
    self.pressed = false
    return self
end

function Button:click()
    if self.enabled then
        self.pressed = true
        print(string.format("[Button:%s] คลิก: '%s'", self.id, self.text))
        self:emit("click")
        self.pressed = false
    end
end

function Button:render()
    if self.visible then
        local state = self.enabled and (self.pressed and "[กด]" or "[ปกติ]") or "[ปิด]"
        print(string.format("[Button:%s] '%s' %s @(%d,%d)",
            self.id, self.text, state, self.x, self.y))
    end
end

-- TextInput Widget
local TextInput = setmetatable({}, {__index = Widget})
TextInput.__index = TextInput

function TextInput.new(id, x, y, w, placeholder)
    local self = Widget.new(id, x, y, w, 30)
    setmetatable(self, TextInput)
    self.value = ""
    self.placeholder = placeholder or "พิมพ์ที่นี่..."
    self.focused = false
    return self
end

function TextInput:type(text)
    self.value = self.value .. text
    self:emit("change", self.value)
end

function TextInput:clear()
    self.value = ""
    self:emit("change", self.value)
end

function TextInput:render()
    if self.visible then
        local display = self.value ~= "" and self.value or ("(" .. self.placeholder .. ")")
        local border = self.focused and ">>>" or "---"
        print(string.format("[Input:%s] %s %s %s @(%d,%d)",
            self.id, border, display, border, self.x, self.y))
    end
end

-- ทดสอบ Widget System
print("=== GUI Widget Demo ===\n")

local form = Container.new("form", 10, 10, 300, 400)

local nameInput = TextInput.new("nameInput", 0, 0, 280, "กรุณากรอกชื่อ")
local emailInput = TextInput.new("emailInput", 0, 0, 280, "กรุณากรอก email")
local submitBtn = Button.new("submit", 0, 0, 100, 35, "ส่งข้อมูล")
local cancelBtn = Button.new("cancel", 0, 0, 100, 35, "ยกเลิก")

form:addChild(nameInput)
form:addChild(emailInput)
form:addChild(submitBtn)
form:addChild(cancelBtn)

-- ลงทะเบียน event handlers
nameInput:on("change", function(self, val)
    print("  >> ชื่อเปลี่ยนเป็น: " .. val)
end)

submitBtn:on("click", function(self)
    print("  >> ส่งฟอร์ม: ชื่อ=" .. nameInput.value)
end)

-- จำลองการใช้งาน
nameInput.focused = true
nameInput:type("สมชาย")
nameInput:type(" ใจดี")
nameInput.focused = false

emailInput.focused = true
emailInput:type("somchai@example.com")
emailInput.focused = false

print("\n--- Render Form ---")
form:render()

print("\n--- ทดสอบคลิก ---")
submitBtn:click()
cancelBtn:click()
```

---

## 22.15 ตัวอย่างขั้นสูง: ORM-like System

### ตัวอย่างที่ 36-40: Simple ORM System

```lua
-- ORM-like System สำหรับ Lua

-- Base Model
local Model = {}
Model.__index = Model

-- Class-level storage (จำลอง database)
Model._storage = {}
Model._nextId = {}
Model._tableName = "model"

function Model:_getTable()
    local name = self._tableName
    if not Model._storage[name] then
        Model._storage[name] = {}
        Model._nextId[name] = 1
    end
    return Model._storage[name]
end

function Model:save()
    local tbl = self:_getTable()
    if not self.id then
        self.id = Model._nextId[self._tableName]
        Model._nextId[self._tableName] = self.id + 1
    end
    tbl[self.id] = self
    return self
end

function Model:delete()
    if self.id then
        local tbl = self:_getTable()
        tbl[self.id] = nil
    end
end

function Model.findById(cls, id)
    local name = cls._tableName
    local tbl = Model._storage[name]
    return tbl and tbl[id]
end

function Model.findAll(cls)
    local name = cls._tableName
    local tbl = Model._storage[name] or {}
    local results = {}
    for _, record in pairs(tbl) do
        table.insert(results, record)
    end
    return results
end

function Model.findWhere(cls, condition)
    local all = cls.findAll(cls)
    local results = {}
    for _, record in ipairs(all) do
        if condition(record) then
            table.insert(results, record)
        end
    end
    return results
end

function Model:toString()
    local parts = {self._tableName .. "{"}
    for k, v in pairs(self) do
        if type(v) ~= "function" and not k:match("^_") then
            table.insert(parts, string.format("  %s=%s", k, tostring(v)))
        end
    end
    table.insert(parts, "}")
    return table.concat(parts, "\n")
end

-- User Model
local User = setmetatable({}, {__index = Model})
User.__index = User
User._tableName = "users"

function User.new(name, email, role)
    return setmetatable({
        name = name,
        email = email,
        role = role or "user"
    }, User)
end

function User:setPassword(pwd)
    -- จำลอง hash
    self.passwordHash = "hash_" .. pwd
end

function User:checkPassword(pwd)
    return self.passwordHash == "hash_" .. pwd
end

function User:isAdmin()
    return self.role == "admin"
end

-- Post Model
local Post = setmetatable({}, {__index = Model})
Post.__index = Post
Post._tableName = "posts"

function Post.new(title, content, authorId)
    return setmetatable({
        title = title,
        content = content,
        authorId = authorId,
        createdAt = os.time(),
        published = false
    }, Post)
end

function Post:publish()
    self.published = true
    self.publishedAt = os.time()
    return self
end

function Post:getAuthor()
    return User.findById(User, self.authorId)
end

-- ทดสอบ ORM
print("=== ORM Demo ===\n")

-- สร้าง users
local u1 = User.new("สมชาย", "somchai@test.com", "admin")
u1:setPassword("password123")
u1:save()

local u2 = User.new("สมหญิง", "somying@test.com", "user")
u2:setPassword("abc123")
u2:save()

local u3 = User.new("ประสิทธิ์", "prasit@test.com", "user")
u3:save()

-- สร้าง posts
local p1 = Post.new("Lua Tutorial Part 1", "เนื้อหา Lua...", u1.id)
p1:save():publish()

local p2 = Post.new("Lua OOP Guide", "เนื้อหา OOP...", u1.id)
p2:save()

local p3 = Post.new("Hello World", "โพสต์แรกของฉัน...", u2.id)
p3:save():publish()

-- Queries
print("Users ทั้งหมด:")
for _, user in ipairs(User.findAll(User)) do
    print(string.format("  [%d] %s (%s) - %s", 
        user.id, user.name, user.email, user.role))
end

print("\nAdmin users:")
local admins = User.findWhere(User, function(u) return u:isAdmin() end)
for _, admin in ipairs(admins) do
    print("  " .. admin.name)
end

print("\nPosts ที่ published:")
local published = Post.findWhere(Post, function(p) return p.published end)
for _, post in ipairs(published) do
    local author = post:getAuthor()
    print(string.format("  '%s' โดย %s", post.title, author and author.name or "ไม่ทราบ"))
end

print("\nตรวจสอบ password:")
print("สมชาย + password123:", u1:checkPassword("password123"))
print("สมชาย + wrongpass:", u1:checkPassword("wrongpass"))
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Vehicle Hierarchy
สร้าง class hierarchy สำหรับยานพาหนะ:
- `Vehicle` (base): `brand`, `model`, `year`, methods: `start()`, `stop()`, `refuel(amount)`
- `LandVehicle` extends `Vehicle`: `wheels`, `topSpeed`
- `Car` extends `LandVehicle`: `doors`, `seats`, `openTrunk()`
- `Truck` extends `LandVehicle`: `payload`, `loadCargo(weight)`
- `ElectricCar` extends `Car`: `batteryCapacity`, `charge(amount)`, `getRange()`
- `WaterVehicle` extends `Vehicle`: `draft`, `float()`
- `Boat` extends `WaterVehicle`: `motorType`, `sail(destination)`

ต้องการ:
1. เรียก parent constructor ในทุก class
2. Override `start()` ใน ElectricCar เพื่อแสดงข้อความต่างออกไป
3. เพิ่ม `isElectric()` method ใน Vehicle ที่คืน false, override ใน ElectricCar ให้คืน true
4. สร้าง fleet = {} และทดสอบ polymorphism

### แบบฝึกหัดที่ 2: Abstract Payment System
สร้าง Abstract Payment Processor:
- `PaymentProcessor` (abstract): `processPayment(amount)`, `validateCard(cardNum)`, `refund(transactionId)`
- `CreditCardProcessor` extends: ใช้ลดแต้มเมื่อชำระ
- `DebitCardProcessor` extends: ตัดจากยอดคงเหลือ
- `CryptoProcessor` extends: แปลงอัตราแลกเปลี่ยน

### แบบฝึกหัดที่ 3: Observer Pattern ด้วย Inheritance
สร้าง `Observable` base class ที่มี:
- `subscribe(event, handler)`
- `unsubscribe(event, handler)`
- `emit(event, ...)`
จากนั้นสร้าง:
- `Store` extends Observable: state management
- `Counter` extends Store: count, increment(), decrement()
- `Timer` extends Observable: start(), stop(), tick()

### แบบฝึกหัดที่ 4: Type Checking System
สร้าง type checking system ที่สมบูรณ์:
1. `isInstanceOf(obj, cls)` - ตรวจสอบ class
2. `getTypeHierarchy(obj)` - คืน array ของ class ทั้งหมดใน hierarchy
3. `hasInterface(obj, interface)` - ตรวจสอบว่า implement interface ครบ
4. ทดสอบกับ class hierarchy อย่างน้อย 5 levels

### แบบฝึกหัดที่ 5: Design Pattern - Strategy
Implement Strategy Pattern ด้วย Inheritance:
- `Sorter` (abstract): `sort(array)`
- `BubbleSorter` extends Sorter
- `QuickSorter` extends Sorter  
- `MergeSorter` extends Sorter
- `SortContext`: เก็บ strategy และ `execute(array)`

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Single Inheritance**: การสืบทอด class ด้วย `setmetatable` และ `__index`
2. **Super Constructor**: การเรียก parent constructor ด้วย `ParentClass.new(self, ...)`
3. **Method Overriding**: การ override method และเรียก parent method
4. **Protected Methods**: การใช้ `_` convention สำหรับ protected methods
5. **Abstract Classes**: การสร้าง abstract class ที่ไม่สามารถ instantiate โดยตรง
6. **Interface Pattern**: การตรวจสอบ interface ด้วย duck typing
7. **Mixin Pattern**: การเพิ่ม functionality โดยไม่ใช้ inheritance
8. **instanceof/is_a**: การตรวจสอบ type hierarchy
9. **Deep Inheritance**: การสร้าง hierarchy หลายชั้น
10. **Polymorphism**: การใช้ objects หลายประเภทผ่าน interface เดียวกัน
11. **Diamond Problem**: การแก้ปัญหา multiple inheritance
12. **Duck Typing**: การตรวจสอบความสามารถแทนที่จะตรวจสอบประเภท

> **เคล็ดลับ**: ใน Lua เราต้องสร้าง OOP ด้วยมือ ทำให้เราเข้าใจกลไกภายในได้ดีกว่าภาษาที่มี class syntax ในตัว การเข้าใจ metatable และ __index chain คือกุญแจสำคัญของ OOP ใน Lua
