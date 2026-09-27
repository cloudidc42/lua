# บทที่ 38: Design Patterns - Structural & Behavioral (รูปแบบโครงสร้างและพฤติกรรม)

## บทนำ

**Structural Patterns** เกี่ยวกับการจัดโครงสร้างของ classes และ objects เพื่อสร้าง structure ขนาดใหญ่จากส่วนเล็กๆ

**Behavioral Patterns** เกี่ยวกับวิธีที่ objects สื่อสารและแบ่งความรับผิดชอบ

## ส่วนที่ 1: Structural Patterns

## 38.1 Adapter Pattern

Adapter แปลง interface ของ class หนึ่งไปเป็นอีก interface หนึ่ง ทำให้ classes ที่ไม่เข้ากันสามารถทำงานร่วมกันได้

```lua
-- ตัวอย่างที่ 1: Adapter Pattern พื้นฐาน
print("=== Adapter Pattern ===")

-- Legacy interface (เก่า)
local LegacyPrinter = {}
LegacyPrinter.__index = LegacyPrinter

function LegacyPrinter.new()
    return setmetatable({}, LegacyPrinter)
end

function LegacyPrinter:printDocument(text, font, size)
    print(string.format("[Legacy Printer] Font:%s Size:%d\n  %s", font, size, text))
end

-- New interface ที่เราต้องการใช้
local ModernPrinter = {}
ModernPrinter.__index = ModernPrinter

function ModernPrinter:print(document)
    -- document = {content, style = {font, size}}
end

-- Adapter: แปลง LegacyPrinter ให้ทำงานกับ ModernPrinter interface
local PrinterAdapter = setmetatable({}, {__index = ModernPrinter})
PrinterAdapter.__index = PrinterAdapter

function PrinterAdapter.new(legacyPrinter)
    return setmetatable({_printer = legacyPrinter}, PrinterAdapter)
end

function PrinterAdapter:print(document)
    -- แปลง modern document format เป็น legacy parameters
    local font = document.style and document.style.font or "Arial"
    local size = document.style and document.style.size or 12
    self._printer:printDocument(document.content, font, size)
end

-- ใช้งาน
local legacy = LegacyPrinter.new()
local adapted = PrinterAdapter.new(legacy)

adapted:print({
    content = "Hello, World!",
    style = {font = "Helvetica", size = 14}
})

adapted:print({
    content = "This is a test document",
    style = {font = "Times New Roman", size = 11}
})
```

```lua
-- ตัวอย่างที่ 2: Multiple Payment Gateway Adapter
print("=== Payment Gateway Adapter ===")

-- Stripe API (ของจริง)
local StripeGateway = {}
StripeGateway.__index = StripeGateway

function StripeGateway.new(apiKey)
    return setmetatable({apiKey = apiKey}, StripeGateway)
end

function StripeGateway:chargeCard(cardToken, amountCents, currency, description)
    print(string.format("[Stripe] Charging %d %s for '%s'", amountCents, currency, description))
    return {
        id = "ch_" .. tostring(math.random(100000, 999999)),
        status = "succeeded",
        amount = amountCents
    }
end

-- PayPal API (ของจริง)
local PayPalGateway = {}
PayPalGateway.__index = PayPalGateway

function PayPalGateway.new(clientId, secret)
    return setmetatable({clientId = clientId, secret = secret}, PayPalGateway)
end

function PayPalGateway:createPayment(amount, currency, returnUrl, cancelUrl)
    print(string.format("[PayPal] Creating payment: %.2f %s", amount, currency))
    return {
        paymentId = "PAY-" .. tostring(math.random(100000, 999999)),
        approvalUrl = "https://paypal.com/approve/" .. math.random(1000),
        state = "created"
    }
end

-- Unified Payment Interface
local PaymentAdapter = {}
PaymentAdapter.__index = PaymentAdapter

-- Stripe Adapter
local StripeAdapter = setmetatable({}, {__index = PaymentAdapter})
StripeAdapter.__index = StripeAdapter

function StripeAdapter.new(apiKey)
    return setmetatable({
        _gateway = StripeGateway.new(apiKey),
        gatewayName = "Stripe"
    }, StripeAdapter)
end

function StripeAdapter:pay(amount, currency, source, description)
    local amountCents = math.floor(amount * 100)
    local result = self._gateway:chargeCard(source, amountCents, currency, description)
    return {
        success = result.status == "succeeded",
        transactionId = result.id,
        amount = amount,
        currency = currency
    }
end

-- PayPal Adapter
local PayPalAdapter = setmetatable({}, {__index = PaymentAdapter})
PayPalAdapter.__index = PayPalAdapter

function PayPalAdapter.new(clientId, secret)
    return setmetatable({
        _gateway = PayPalGateway.new(clientId, secret),
        gatewayName = "PayPal"
    }, PayPalAdapter)
end

function PayPalAdapter:pay(amount, currency, source, description)
    local result = self._gateway:createPayment(amount, currency, 
        "https://return.url", "https://cancel.url")
    return {
        success = result.state == "created",
        transactionId = result.paymentId,
        amount = amount,
        currency = currency,
        approvalUrl = result.approvalUrl
    }
end

-- ใช้ payment processor ผ่าน unified interface
local function processPayment(gateway, amount, currency, description)
    print(string.format("\nProcessing payment via %s...", gateway.gatewayName))
    local result = gateway:pay(amount, currency, "tok_visa_test", description)
    if result.success then
        print(string.format("  SUCCESS: TxID=%s Amount=%.2f %s",
            result.transactionId, result.amount, result.currency))
    else
        print("  FAILED")
    end
    return result
end

local stripe = StripeAdapter.new("sk_test_123")
local paypal = PayPalAdapter.new("client_id", "secret")

processPayment(stripe, 99.99, "USD", "Premium subscription")
processPayment(paypal, 49.99, "USD", "Monthly plan")
```

## 38.2 Bridge Pattern

Bridge แยก abstraction ออกจาก implementation ทำให้ทั้งสองสามารถเปลี่ยนแปลงอิสระจากกัน

```lua
-- ตัวอย่างที่ 3: Bridge Pattern
print("=== Bridge Pattern ===")

-- Implementation interface
local Renderer = {}
Renderer.__index = Renderer
function Renderer:renderCircle(x, y, radius) end
function Renderer:renderRect(x, y, w, h) end

-- Concrete Implementations
local VectorRenderer = setmetatable({}, {__index = Renderer})
VectorRenderer.__index = VectorRenderer

function VectorRenderer.new()
    return setmetatable({type = "Vector"}, VectorRenderer)
end

function VectorRenderer:renderCircle(x, y, radius)
    print(string.format("[Vector] Circle at (%d,%d) r=%d (SVG path)", x, y, radius))
end

function VectorRenderer:renderRect(x, y, w, h)
    print(string.format("[Vector] Rect at (%d,%d) %dx%d (SVG rect)", x, y, w, h))
end

local RasterRenderer = setmetatable({}, {__index = Renderer})
RasterRenderer.__index = RasterRenderer

function RasterRenderer.new()
    return setmetatable({type = "Raster"}, RasterRenderer)
end

function RasterRenderer:renderCircle(x, y, radius)
    print(string.format("[Raster] Drawing %d pixels for circle at (%d,%d)", 
        math.floor(math.pi * radius * radius), x, y))
end

function RasterRenderer:renderRect(x, y, w, h)
    print(string.format("[Raster] Filling %d pixels for rect at (%d,%d)", w*h, x, y))
end

-- Abstraction
local Shape = {}
Shape.__index = Shape

function Shape.new(renderer)
    return setmetatable({renderer = renderer}, Shape)
end

function Shape:draw() end
function Shape:resize(factor) end

-- Refined Abstractions
local Circle = setmetatable({}, {__index = Shape})
Circle.__index = Circle

function Circle.new(renderer, x, y, radius)
    local c = Shape.new(renderer)
    c.x, c.y, c.radius = x, y, radius
    return setmetatable(c, Circle)
end

function Circle:draw()
    self.renderer:renderCircle(self.x, self.y, self.radius)
end

function Circle:resize(factor)
    self.radius = math.floor(self.radius * factor)
end

local Rectangle = setmetatable({}, {__index = Shape})
Rectangle.__index = Rectangle

function Rectangle.new(renderer, x, y, w, h)
    local r = Shape.new(renderer)
    r.x, r.y, r.w, r.h = x, y, w, h
    return setmetatable(r, Rectangle)
end

function Rectangle:draw()
    self.renderer:renderRect(self.x, self.y, self.w, self.h)
end

-- ทดสอบ - เปลี่ยน renderer ได้โดยไม่ต้องเปลี่ยน shapes
local vector = VectorRenderer.new()
local raster = RasterRenderer.new()

local shapes_v = {
    Circle.new(vector, 10, 10, 50),
    Rectangle.new(vector, 0, 0, 100, 80),
}

local shapes_r = {
    Circle.new(raster, 10, 10, 50),
    Rectangle.new(raster, 0, 0, 100, 80),
}

print("Vector rendering:")
for _, s in ipairs(shapes_v) do s:draw() end

print("\nRaster rendering:")
for _, s in ipairs(shapes_r) do s:draw() end
```

## 38.3 Composite Pattern

Composite ช่วยให้เราปฏิบัติต่อ objects แบบเดียว (Leaf) และกลุ่มของ objects (Composite) ด้วยวิธีเดียวกัน

```lua
-- ตัวอย่างที่ 4: Composite Pattern - File System
print("=== Composite Pattern - File System ===")

local FileSystemItem = {}
FileSystemItem.__index = FileSystemItem

function FileSystemItem:getSize() return 0 end
function FileSystemItem:print(indent) end

-- Leaf: File
local File = setmetatable({}, {__index = FileSystemItem})
File.__index = File

function File.new(name, size)
    return setmetatable({
        name = name,
        size = size,
        type = "file"
    }, File)
end

function File:getSize() return self.size end

function File:print(indent)
    indent = indent or ""
    print(string.format("%s📄 %s (%d KB)", indent, self.name, self.size))
end

-- Composite: Directory
local Directory = setmetatable({}, {__index = FileSystemItem})
Directory.__index = Directory

function Directory.new(name)
    return setmetatable({
        name = name,
        children = {},
        type = "directory"
    }, Directory)
end

function Directory:add(item)
    table.insert(self.children, item)
    return self
end

function Directory:remove(name)
    for i, child in ipairs(self.children) do
        if child.name == name then
            table.remove(self.children, i)
            return true
        end
    end
    return false
end

function Directory:getSize()
    local total = 0
    for _, child in ipairs(self.children) do
        total = total + child:getSize()
    end
    return total
end

function Directory:print(indent)
    indent = indent or ""
    print(string.format("%s📁 %s/ (%d KB total)", indent, self.name, self:getSize()))
    for _, child in ipairs(self.children) do
        child:print(indent .. "  ")
    end
end

function Directory:find(name)
    for _, child in ipairs(self.children) do
        if child.name == name then return child end
        if child.type == "directory" then
            local found = child:find(name)
            if found then return found end
        end
    end
    return nil
end

-- สร้าง file system tree
local root = Directory.new("root")
    :add(File.new("README.md", 5))
    :add(File.new(".gitignore", 1))

local src = Directory.new("src")
    :add(File.new("main.lua", 45))
    :add(File.new("config.lua", 12))

local utils = Directory.new("utils")
    :add(File.new("string.lua", 8))
    :add(File.new("table.lua", 15))
    :add(File.new("math.lua", 6))

src:add(utils)
root:add(src)

local docs = Directory.new("docs")
    :add(File.new("api.md", 30))
    :add(File.new("guide.md", 25))
root:add(docs)

root:print()
print(string.format("\nTotal size: %d KB", root:getSize()))

local found = root:find("string.lua")
if found then
    print(string.format("\nFound: %s (%d KB)", found.name, found.size))
end
```

## 38.4 Decorator Pattern

Decorator เพิ่ม behavior ให้ objects โดย dynamic โดยไม่แก้ไข class เดิม

```lua
-- ตัวอย่างที่ 5: Decorator Pattern - Text processing
print("=== Decorator Pattern - Text Processing ===")

-- Component interface
local TextProcessor = {}
TextProcessor.__index = TextProcessor
function TextProcessor:process(text) return text end

-- Concrete Component
local PlainText = setmetatable({}, {__index = TextProcessor})
PlainText.__index = PlainText

function PlainText.new()
    return setmetatable({}, PlainText)
end

function PlainText:process(text)
    return text
end

-- Base Decorator
local TextDecorator = setmetatable({}, {__index = TextProcessor})
TextDecorator.__index = TextDecorator

function TextDecorator.new(component)
    return setmetatable({_component = component}, TextDecorator)
end

function TextDecorator:process(text)
    return self._component:process(text)
end

-- Concrete Decorators
local UpperCaseDecorator = setmetatable({}, {__index = TextDecorator})
UpperCaseDecorator.__index = UpperCaseDecorator

function UpperCaseDecorator.new(component)
    return setmetatable(TextDecorator.new(component), UpperCaseDecorator)
end

function UpperCaseDecorator:process(text)
    return self._component:process(text):upper()
end

local TrimDecorator = setmetatable({}, {__index = TextDecorator})
TrimDecorator.__index = TrimDecorator

function TrimDecorator.new(component)
    return setmetatable(TextDecorator.new(component), TrimDecorator)
end

function TrimDecorator:process(text)
    local result = self._component:process(text)
    return result:match("^%s*(.-)%s*$")
end

local HTMLEscapeDecorator = setmetatable({}, {__index = TextDecorator})
HTMLEscapeDecorator.__index = HTMLEscapeDecorator

function HTMLEscapeDecorator.new(component)
    return setmetatable(TextDecorator.new(component), HTMLEscapeDecorator)
end

function HTMLEscapeDecorator:process(text)
    local result = self._component:process(text)
    result = result:gsub("&", "&amp;")
    result = result:gsub("<", "&lt;")
    result = result:gsub(">", "&gt;")
    result = result:gsub('"', "&quot;")
    return result
end

local WrapDecorator = setmetatable({}, {__index = TextDecorator})
WrapDecorator.__index = WrapDecorator

function WrapDecorator.new(component, width)
    local d = TextDecorator.new(component)
    d._width = width or 40
    return setmetatable(d, WrapDecorator)
end

function WrapDecorator:process(text)
    local result = self._component:process(text)
    local wrapped = {}
    local lineLen = 0
    local line = {}
    
    for word in result:gmatch("%S+") do
        if lineLen + #word + (lineLen > 0 and 1 or 0) > self._width then
            table.insert(wrapped, table.concat(line, " "))
            line = {word}
            lineLen = #word
        else
            table.insert(line, word)
            lineLen = lineLen + #word + (lineLen > 0 and 1 or 0)
        end
    end
    if #line > 0 then
        table.insert(wrapped, table.concat(line, " "))
    end
    return table.concat(wrapped, "\n")
end

-- ทดสอบ: chain decorators
local text = "  Hello, <World>! This is a test of the decorator pattern in Lua.  "

local plain = PlainText.new()
local trimmed = TrimDecorator.new(plain)
local htmlSafe = HTMLEscapeDecorator.new(trimmed)
local upper = UpperCaseDecorator.new(trimmed)
local wrapped = WrapDecorator.new(TrimDecorator.new(plain), 30)

print("Original: '" .. text .. "'")
print("\nTrimmed:  '" .. trimmed:process(text) .. "'")
print("\nHTML safe: '" .. htmlSafe:process(text) .. "'")
print("\nUpper: '" .. upper:process(text) .. "'")
print("\nWrapped (30):")
print(wrapped:process(text))
```

```lua
-- ตัวอย่างที่ 6: Decorator Pattern - Logger Enhancement
print("=== Logger Decorator ===")

local BaseLogger = {}
BaseLogger.__index = BaseLogger

function BaseLogger.new()
    return setmetatable({}, BaseLogger)
end

function BaseLogger:log(level, message)
    print(string.format("[%s] %s", level, message))
end

-- Timestamp Decorator
local TimestampLogger = {}
TimestampLogger.__index = TimestampLogger

function TimestampLogger.new(logger)
    return setmetatable({_logger = logger}, TimestampLogger)
end

function TimestampLogger:log(level, message)
    local ts = os.date("%Y-%m-%d %H:%M:%S")
    self._logger:log(level, string.format("[%s] %s", ts, message))
end

-- Color Decorator (for terminal)
local ColorLogger = {}
ColorLogger.__index = ColorLogger

ColorLogger.colors = {
    DEBUG = "\27[36m",  -- Cyan
    INFO  = "\27[32m",  -- Green
    WARN  = "\27[33m",  -- Yellow
    ERROR = "\27[31m",  -- Red
    RESET = "\27[0m"
}

function ColorLogger.new(logger)
    return setmetatable({_logger = logger}, ColorLogger)
end

function ColorLogger:log(level, message)
    local color = ColorLogger.colors[level] or ""
    local reset = ColorLogger.colors.RESET
    self._logger:log(level, color .. message .. reset)
end

-- Prefix Decorator
local PrefixLogger = {}
PrefixLogger.__index = PrefixLogger

function PrefixLogger.new(logger, prefix)
    return setmetatable({_logger = logger, _prefix = prefix}, PrefixLogger)
end

function PrefixLogger:log(level, message)
    self._logger:log(level, "[" .. self._prefix .. "] " .. message)
end

-- chain: base -> timestamp -> prefix -> color
local logger = ColorLogger.new(
    PrefixLogger.new(
        TimestampLogger.new(BaseLogger.new()),
        "APP"
    )
)

logger:log("INFO", "Server started on port 8080")
logger:log("WARN", "High memory usage detected")
logger:log("ERROR", "Database connection failed")
logger:log("DEBUG", "Processing request #42")
```

## 38.5 Facade Pattern

Facade ให้ interface ที่เรียบง่ายสำหรับ subsystem ที่ซับซ้อน

```lua
-- ตัวอย่างที่ 7: Facade Pattern - Home Theater
print("=== Facade Pattern - Home Theater ===")

-- Complex subsystems
local DVD = {}
DVD.__index = DVD
function DVD.new()
    return setmetatable({movie = nil}, DVD)
end
function DVD:on() print("[DVD] Player on") end
function DVD:off() print("[DVD] Player off") end
function DVD:play(movie) self.movie = movie; print("[DVD] Playing: " .. movie) end
function DVD:stop() print("[DVD] Stopped") end

local Amplifier = {}
Amplifier.__index = Amplifier
function Amplifier.new()
    return setmetatable({volume = 0}, Amplifier)
end
function Amplifier:on() print("[Amp] On") end
function Amplifier:off() print("[Amp] Off") end
function Amplifier:setVolume(v) self.volume = v; print("[Amp] Volume: " .. v) end
function Amplifier:setDVD(dvd) print("[Amp] Connected to DVD") end

local Projector = {}
Projector.__index = Projector
function Projector.new()
    return setmetatable({}, Projector)
end
function Projector:on() print("[Projector] On") end
function Projector:off() print("[Projector] Off") end
function Projector:widescreen() print("[Projector] Widescreen mode") end

local Screen = {}
Screen.__index = Screen
function Screen.new()
    return setmetatable({}, Screen)
end
function Screen:down() print("[Screen] Down") end
function Screen:up() print("[Screen] Up") end

local Lights = {}
Lights.__index = Lights
function Lights.new()
    return setmetatable({level = 100}, Lights)
end
function Lights:dim(level) self.level = level; print("[Lights] Dim to " .. level .. "%") end
function Lights:on() self.level = 100; print("[Lights] Full brightness") end

-- Facade
local HomeTheaterFacade = {}
HomeTheaterFacade.__index = HomeTheaterFacade

function HomeTheaterFacade.new()
    local facade = setmetatable({}, HomeTheaterFacade)
    facade.dvd = DVD.new()
    facade.amp = Amplifier.new()
    facade.projector = Projector.new()
    facade.screen = Screen.new()
    facade.lights = Lights.new()
    return facade
end

function HomeTheaterFacade:watchMovie(movie)
    print("\n=== Get ready to watch " .. movie .. "! ===")
    self.lights:dim(10)
    self.screen:down()
    self.projector:on()
    self.projector:widescreen()
    self.amp:on()
    self.amp:setDVD(self.dvd)
    self.amp:setVolume(5)
    self.dvd:on()
    self.dvd:play(movie)
end

function HomeTheaterFacade:endMovie()
    print("\n=== Shutting down movie theater... ===")
    self.lights:on()
    self.screen:up()
    self.projector:off()
    self.amp:off()
    self.dvd:stop()
    self.dvd:off()
end

local theater = HomeTheaterFacade.new()
theater:watchMovie("Inception")
theater:endMovie()
```

## 38.6 Flyweight Pattern

Flyweight แชร์ state ร่วมกันระหว่าง objects จำนวนมากเพื่อประหยัดหน่วยความจำ

```lua
-- ตัวอย่างที่ 8: Flyweight Pattern - Game Characters
print("=== Flyweight Pattern - Game Characters ===")

-- Flyweight: intrinsic (shared) state
local CharacterType = {}
CharacterType.__index = CharacterType

function CharacterType.new(name, sprite, speed, damage)
    return setmetatable({
        name = name,
        sprite = sprite,
        speed = speed,
        damage = damage,
    }, CharacterType)
end

function CharacterType:render(x, y, health)
    print(string.format("[Render] %s (sprite=%s) at (%d,%d) HP:%d", 
        self.name, self.sprite, x, y, health))
end

-- Flyweight Factory
local CharacterTypeFactory = {}
CharacterTypeFactory._types = {}

function CharacterTypeFactory.getType(name, sprite, speed, damage)
    local key = name .. sprite
    if not CharacterTypeFactory._types[key] then
        CharacterTypeFactory._types[key] = CharacterType.new(name, sprite, speed, damage)
        print(string.format("[Factory] Created new CharacterType: %s", name))
    end
    return CharacterTypeFactory._types[key]
end

function CharacterTypeFactory.count()
    local n = 0
    for _ in pairs(CharacterTypeFactory._types) do n = n + 1 end
    return n
end

-- Extrinsic (unique) state per instance
local GameCharacter = {}
GameCharacter.__index = GameCharacter

function GameCharacter.new(typeName, sprite, speed, damage, x, y)
    return setmetatable({
        -- Get shared flyweight
        _type = CharacterTypeFactory.getType(typeName, sprite, speed, damage),
        -- Unique extrinsic state
        x = x,
        y = y,
        health = 100,
        id = math.random(10000),
    }, GameCharacter)
end

function GameCharacter:render()
    self._type:render(self.x, self.y, self.health)
end

function GameCharacter:move(dx, dy)
    self.x = self.x + dx * self._type.speed
    self.y = self.y + dy * self._type.speed
end

-- Spawn many characters sharing type data
local characters = {}
for i = 1, 5 do
    table.insert(characters, GameCharacter.new("Goblin", "goblin.png", 2, 5, 
        math.random(0, 100), math.random(0, 100)))
end
for i = 1, 3 do
    table.insert(characters, GameCharacter.new("Orc", "orc.png", 1, 15,
        math.random(0, 100), math.random(0, 100)))
end
for i = 1, 2 do
    table.insert(characters, GameCharacter.new("Dragon", "dragon.png", 3, 50,
        math.random(0, 100), math.random(0, 100)))
end

print(string.format("\nCreated %d characters using only %d unique type objects",
    #characters, CharacterTypeFactory.count()))

for _, c in ipairs(characters) do
    c:render()
end
```

## 38.7 Proxy Pattern

Proxy ให้ surrogate หรือ placeholder สำหรับ object อื่น ควบคุมการเข้าถึง

```lua
-- ตัวอย่างที่ 9: Proxy Pattern - Virtual Proxy (Lazy Loading)
print("=== Proxy Pattern - Lazy Loading ===")

-- Real Subject (expensive to create)
local RealImage = {}
RealImage.__index = RealImage

function RealImage.new(filename)
    print(string.format("[RealImage] Loading heavy image: %s", filename))
    -- Simulate expensive loading
    return setmetatable({
        filename = filename,
        data = string.rep("X", 1024),  -- 1KB of image data
        width = 1920,
        height = 1080,
    }, RealImage)
end

function RealImage:display()
    print(string.format("[RealImage] Displaying %s (%dx%d)", 
        self.filename, self.width, self.height))
end

function RealImage:getSize()
    return string.format("%dx%d", self.width, self.height)
end

-- Proxy: loads image on demand
local ImageProxy = {}
ImageProxy.__index = ImageProxy

function ImageProxy.new(filename)
    return setmetatable({
        filename = filename,
        _realImage = nil,
    }, ImageProxy)
end

function ImageProxy:_load()
    if not self._realImage then
        self._realImage = RealImage.new(self.filename)
    end
    return self._realImage
end

function ImageProxy:display()
    self:_load():display()
end

function ImageProxy:getSize()
    return self:_load():getSize()
end

function ImageProxy:getFilename()
    return self.filename  -- no loading needed
end

-- ทดสอบ
print("Creating image proxies (no loading yet)...")
local images = {
    ImageProxy.new("photo1.jpg"),
    ImageProxy.new("photo2.png"),
    ImageProxy.new("photo3.gif"),
}

print("\nAccessing filename (no loading):")
for _, img in ipairs(images) do
    print("  " .. img:getFilename())
end

print("\nDisplaying first image (triggers loading):")
images[1]:display()

print("\nDisplaying first image again (no reload):")
images[1]:display()
```

```lua
-- ตัวอย่างที่ 10: Protection Proxy
print("=== Protection Proxy ===")

local SensitiveData = {}
SensitiveData.__index = SensitiveData

function SensitiveData.new()
    return setmetatable({
        _data = {
            salary = {alice = 5000, bob = 6000},
            ssn = {alice = "123-45-6789", bob = "987-65-4321"},
        }
    }, SensitiveData)
end

function SensitiveData:getSalary(name)
    return self._data.salary[name]
end

function SensitiveData:getSSN(name)
    return self._data.ssn[name]
end

function SensitiveData:setSalary(name, amount)
    self._data.salary[name] = amount
    return true
end

-- Protection Proxy
local SecureDataProxy = {}
SecureDataProxy.__index = SecureDataProxy

local roles = {
    admin   = {canRead = true,  canWrite = true,  canReadSSN = true},
    manager = {canRead = true,  canWrite = true,  canReadSSN = false},
    user    = {canRead = true,  canWrite = false, canReadSSN = false},
}

function SecureDataProxy.new(data, userRole)
    return setmetatable({
        _data = data,
        _role = userRole,
        _perms = roles[userRole] or {canRead = false, canWrite = false, canReadSSN = false}
    }, SecureDataProxy)
end

function SecureDataProxy:getSalary(name)
    if not self._perms.canRead then
        error("Access denied: " .. self._role .. " cannot read salary data")
    end
    return self._data:getSalary(name)
end

function SecureDataProxy:getSSN(name)
    if not self._perms.canReadSSN then
        error("Access denied: " .. self._role .. " cannot read SSN data")
    end
    return self._data:getSSN(name)
end

function SecureDataProxy:setSalary(name, amount)
    if not self._perms.canWrite then
        error("Access denied: " .. self._role .. " cannot modify salary data")
    end
    return self._data:setSalary(name, amount)
end

local data = SensitiveData.new()

local adminProxy = SecureDataProxy.new(data, "admin")
local managerProxy = SecureDataProxy.new(data, "manager")
local userProxy = SecureDataProxy.new(data, "user")

print("Admin reading Alice's salary: " .. tostring(adminProxy:getSalary("alice")))
print("Admin reading Alice's SSN: " .. adminProxy:getSSN("alice"))

print("Manager reading Alice's salary: " .. tostring(managerProxy:getSalary("alice")))
local ok, err = pcall(function() managerProxy:getSSN("alice") end)
if not ok then print("Manager SSN access: " .. err) end

local ok2, err2 = pcall(function() userProxy:setSalary("alice", 9999) end)
if not ok2 then print("User write access: " .. err2) end
```

---

## ส่วนที่ 2: Behavioral Patterns

## 38.8 Observer Pattern

Observer กำหนด dependency แบบ one-to-many ระหว่าง objects เมื่อ object หนึ่งเปลี่ยน ทุก objects ที่ depend จะได้รับการแจ้ง

```lua
-- ตัวอย่างที่ 11: Observer Pattern พื้นฐาน
print("=== Observer Pattern ===")

local Subject = {}
Subject.__index = Subject

function Subject.new()
    return setmetatable({
        _observers = {},
        _state = nil,
    }, Subject)
end

function Subject:attach(observer, event)
    event = event or "default"
    if not self._observers[event] then
        self._observers[event] = {}
    end
    table.insert(self._observers[event], observer)
end

function Subject:detach(observer, event)
    event = event or "default"
    if not self._observers[event] then return end
    for i, obs in ipairs(self._observers[event]) do
        if obs == observer then
            table.remove(self._observers[event], i)
            return
        end
    end
end

function Subject:notify(event, data)
    event = event or "default"
    if self._observers[event] then
        for _, obs in ipairs(self._observers[event]) do
            obs:update(event, data, self)
        end
    end
    -- Also notify "all" observers
    if self._observers["all"] then
        for _, obs in ipairs(self._observers["all"]) do
            obs:update(event, data, self)
        end
    end
end

function Subject:setState(state)
    self._state = state
    self:notify("stateChange", state)
end

function Subject:getState()
    return self._state
end

-- Concrete Observers
local LogObserver = {}
LogObserver.__index = LogObserver

function LogObserver.new(name)
    return setmetatable({name = name, log = {}}, LogObserver)
end

function LogObserver:update(event, data, subject)
    local entry = string.format("[%s] Event: %s, Data: %s", 
        self.name, event, tostring(data))
    table.insert(self.log, entry)
    print(entry)
end

local CounterObserver = {}
CounterObserver.__index = CounterObserver

function CounterObserver.new()
    return setmetatable({count = 0, eventCounts = {}}, CounterObserver)
end

function CounterObserver:update(event, data, subject)
    self.count = self.count + 1
    self.eventCounts[event] = (self.eventCounts[event] or 0) + 1
end

function CounterObserver:report()
    print(string.format("Total events: %d", self.count))
    for event, count in pairs(self.eventCounts) do
        print(string.format("  %s: %d", event, count))
    end
end

-- ทดสอบ
local sensor = Subject.new()
local logger = LogObserver.new("Logger")
local counter = CounterObserver.new()

sensor:attach(logger, "temperature")
sensor:attach(logger, "pressure")
sensor:attach(counter, "all")

sensor:notify("temperature", 25.5)
sensor:notify("pressure", 1013.2)
sensor:notify("temperature", 26.1)
sensor:notify("error", "Sensor malfunction")

print("\nEvent counts:")
counter:report()
```

## 38.9 Strategy Pattern

Strategy กำหนด family ของ algorithms, encapsulates แต่ละอัน และทำให้สามารถแทนกันได้

```lua
-- ตัวอย่างที่ 12: Strategy Pattern - Sorting
print("=== Strategy Pattern - Sorting ===")

-- Sorting strategies
local BubbleSort = {name = "BubbleSort"}
function BubbleSort:sort(arr)
    local a = {table.unpack(arr)}
    local n = #a
    for i = 1, n do
        for j = 1, n - i do
            if a[j] > a[j + 1] then
                a[j], a[j + 1] = a[j + 1], a[j]
            end
        end
    end
    return a
end

local QuickSort = {name = "QuickSort"}
function QuickSort:sort(arr)
    local a = {table.unpack(arr)}
    local function qsort(lo, hi)
        if lo >= hi then return end
        local pivot = a[hi]
        local i = lo - 1
        for j = lo, hi - 1 do
            if a[j] <= pivot then
                i = i + 1
                a[i], a[j] = a[j], a[i]
            end
        end
        a[i + 1], a[hi] = a[hi], a[i + 1]
        local pi = i + 1
        qsort(lo, pi - 1)
        qsort(pi + 1, hi)
    end
    qsort(1, #a)
    return a
end

local MergeSort = {name = "MergeSort"}
function MergeSort:sort(arr)
    if #arr <= 1 then return arr end
    local mid = math.floor(#arr / 2)
    local left = MergeSort:sort({table.unpack(arr, 1, mid)})
    local right = MergeSort:sort({table.unpack(arr, mid + 1)})
    
    local result = {}
    local i, j = 1, 1
    while i <= #left and j <= #right do
        if left[i] <= right[j] then
            table.insert(result, left[i]); i = i + 1
        else
            table.insert(result, right[j]); j = j + 1
        end
    end
    while i <= #left do table.insert(result, left[i]); i = i + 1 end
    while j <= #right do table.insert(result, right[j]); j = j + 1 end
    return result
end

-- Context
local Sorter = {}
Sorter.__index = Sorter

function Sorter.new(strategy)
    return setmetatable({_strategy = strategy}, Sorter)
end

function Sorter:setStrategy(strategy)
    self._strategy = strategy
end

function Sorter:sort(arr)
    print(string.format("Sorting with %s...", self._strategy.name))
    local result = self._strategy:sort(arr)
    print("  Result: " .. table.concat(result, ", "))
    return result
end

local data = {64, 34, 25, 12, 22, 11, 90}
print("Original: " .. table.concat(data, ", "))

local sorter = Sorter.new(BubbleSort)
sorter:sort(data)

sorter:setStrategy(QuickSort)
sorter:sort(data)

sorter:setStrategy(MergeSort)
sorter:sort(data)
```

## 38.10 Command Pattern

Command encapsulates request เป็น object ทำให้สามารถ queue, log, undo/redo ได้

```lua
-- ตัวอย่างที่ 13: Command Pattern - Text Editor with Undo/Redo
print("=== Command Pattern - Text Editor ===")

-- Receiver
local TextEditor = {}
TextEditor.__index = TextEditor

function TextEditor.new()
    return setmetatable({
        content = "",
        cursor = 0
    }, TextEditor)
end

function TextEditor:getContent() return self.content end

function TextEditor:insertAt(pos, text)
    self.content = self.content:sub(1, pos) .. text .. self.content:sub(pos + 1)
    self.cursor = pos + #text
end

function TextEditor:deleteAt(pos, length)
    local deleted = self.content:sub(pos + 1, pos + length)
    self.content = self.content:sub(1, pos) .. self.content:sub(pos + length + 1)
    self.cursor = pos
    return deleted
end

-- Commands
local InsertCommand = {}
InsertCommand.__index = InsertCommand

function InsertCommand.new(editor, pos, text)
    return setmetatable({
        editor = editor,
        pos = pos,
        text = text,
        _executed = false
    }, InsertCommand)
end

function InsertCommand:execute()
    self.editor:insertAt(self.pos, self.text)
    self._executed = true
end

function InsertCommand:undo()
    self.editor:deleteAt(self.pos, #self.text)
    self._executed = false
end

local DeleteCommand = {}
DeleteCommand.__index = DeleteCommand

function DeleteCommand.new(editor, pos, length)
    return setmetatable({
        editor = editor,
        pos = pos,
        length = length,
        _deletedText = nil
    }, DeleteCommand)
end

function DeleteCommand:execute()
    self._deletedText = self.editor:deleteAt(self.pos, self.length)
end

function DeleteCommand:undo()
    if self._deletedText then
        self.editor:insertAt(self.pos, self._deletedText)
    end
end

-- Command History (Invoker)
local CommandHistory = {}
CommandHistory.__index = CommandHistory

function CommandHistory.new()
    return setmetatable({
        _history = {},
        _redoStack = {},
    }, CommandHistory)
end

function CommandHistory:execute(command)
    command:execute()
    table.insert(self._history, command)
    self._redoStack = {}  -- clear redo on new command
end

function CommandHistory:undo()
    if #self._history == 0 then
        print("Nothing to undo")
        return false
    end
    local command = table.remove(self._history)
    command:undo()
    table.insert(self._redoStack, command)
    return true
end

function CommandHistory:redo()
    if #self._redoStack == 0 then
        print("Nothing to redo")
        return false
    end
    local command = table.remove(self._redoStack)
    command:execute()
    table.insert(self._history, command)
    return true
end

-- ทดสอบ
local editor = TextEditor.new()
local history = CommandHistory.new()

print("Starting with empty document")

history:execute(InsertCommand.new(editor, 0, "Hello"))
print("After insert 'Hello': '" .. editor:getContent() .. "'")

history:execute(InsertCommand.new(editor, 5, ", World"))
print("After insert ', World': '" .. editor:getContent() .. "'")

history:execute(InsertCommand.new(editor, 12, "!"))
print("After insert '!': '" .. editor:getContent() .. "'")

history:execute(DeleteCommand.new(editor, 5, 7))
print("After delete 7 chars at pos 5: '" .. editor:getContent() .. "'")

print("\n-- Undo operations --")
history:undo()
print("After undo: '" .. editor:getContent() .. "'")
history:undo()
print("After undo: '" .. editor:getContent() .. "'")

print("\n-- Redo operation --")
history:redo()
print("After redo: '" .. editor:getContent() .. "'")
```

## 38.11 Iterator Pattern

```lua
-- ตัวอย่างที่ 14: Iterator Pattern
print("=== Iterator Pattern ===")

-- Tree Node
local TreeNode = {}
TreeNode.__index = TreeNode

function TreeNode.new(value)
    return setmetatable({
        value = value,
        children = {}
    }, TreeNode)
end

function TreeNode:addChild(node)
    table.insert(self.children, node)
    return node
end

-- Depth-First Iterator
local DFSIterator = {}
DFSIterator.__index = DFSIterator

function DFSIterator.new(root)
    return setmetatable({
        _stack = {root},
        _visited = {}
    }, DFSIterator)
end

function DFSIterator:hasNext()
    return #self._stack > 0
end

function DFSIterator:next()
    if #self._stack == 0 then return nil end
    local node = table.remove(self._stack)
    -- Push children in reverse order (so first child is processed first)
    for i = #node.children, 1, -1 do
        table.insert(self._stack, node.children[i])
    end
    return node.value
end

-- Breadth-First Iterator
local BFSIterator = {}
BFSIterator.__index = BFSIterator

function BFSIterator.new(root)
    return setmetatable({_queue = {root}}, BFSIterator)
end

function BFSIterator:hasNext()
    return #self._queue > 0
end

function BFSIterator:next()
    if #self._queue == 0 then return nil end
    local node = table.remove(self._queue, 1)
    for _, child in ipairs(node.children) do
        table.insert(self._queue, child)
    end
    return node.value
end

-- Range Iterator
local function range(from, to, step)
    step = step or 1
    return {
        hasNext = function(self) 
            return step > 0 and self._current <= to or 
                   step < 0 and self._current >= to 
        end,
        next = function(self)
            local val = self._current
            self._current = self._current + step
            return val
        end,
        _current = from
    }
end

-- Build tree
local root = TreeNode.new(1)
local n2 = root:addChild(TreeNode.new(2))
local n3 = root:addChild(TreeNode.new(3))
local n4 = root:addChild(TreeNode.new(4))
n2:addChild(TreeNode.new(5))
n2:addChild(TreeNode.new(6))
n3:addChild(TreeNode.new(7))

print("DFS traversal:")
local dfs = DFSIterator.new(root)
local vals = {}
while dfs:hasNext() do
    table.insert(vals, dfs:next())
end
print("  " .. table.concat(vals, " -> "))

print("BFS traversal:")
local bfs = BFSIterator.new(root)
vals = {}
while bfs:hasNext() do
    table.insert(vals, bfs:next())
end
print("  " .. table.concat(vals, " -> "))

print("Range(1, 10, 2):")
local r = range(1, 10, 2)
vals = {}
while r:hasNext() do
    table.insert(vals, r:next())
end
print("  " .. table.concat(vals, ", "))
```

## 38.12 Template Method Pattern

```lua
-- ตัวอย่างที่ 15: Template Method Pattern
print("=== Template Method Pattern ===")

local DataProcessor = {}
DataProcessor.__index = DataProcessor

function DataProcessor.new()
    return setmetatable({}, DataProcessor)
end

-- Template method (defines the algorithm structure)
function DataProcessor:process(data)
    print("\n[Template] Starting data processing pipeline")
    local validated = self:validate(data)
    if not validated then
        print("[Template] Validation failed, aborting")
        return nil
    end
    local parsed = self:parse(data)
    local transformed = self:transform(parsed)
    local result = self:output(transformed)
    self:cleanup()
    print("[Template] Pipeline complete")
    return result
end

-- Abstract methods (to be overridden)
function DataProcessor:validate(data) return true end
function DataProcessor:parse(data) return data end
function DataProcessor:transform(data) return data end
function DataProcessor:output(data) return data end
function DataProcessor:cleanup() end

-- Concrete Processor: CSV
local CSVProcessor = setmetatable({}, {__index = DataProcessor})
CSVProcessor.__index = CSVProcessor

function CSVProcessor.new()
    return setmetatable(DataProcessor.new(), CSVProcessor)
end

function CSVProcessor:validate(data)
    print("[CSV] Validating CSV data...")
    if type(data) ~= "string" then return false end
    return #data > 0
end

function CSVProcessor:parse(data)
    print("[CSV] Parsing CSV...")
    local rows = {}
    local headers = nil
    for line in (data .. "\n"):gmatch("([^\n]*)\n") do
        if line ~= "" then
            local fields = {}
            for field in (line .. ","):gmatch("([^,]*),") do
                table.insert(fields, field)
            end
            if not headers then
                headers = fields
            else
                local row = {}
                for i, h in ipairs(headers) do
                    row[h] = fields[i]
                end
                table.insert(rows, row)
            end
        end
    end
    return {headers = headers, rows = rows}
end

function CSVProcessor:transform(parsed)
    print("[CSV] Transforming data...")
    local result = {}
    for _, row in ipairs(parsed.rows) do
        local transformed = {}
        for k, v in pairs(row) do
            transformed[k:lower()] = v:match("^%s*(.-)%s*$")  -- trim
        end
        table.insert(result, transformed)
    end
    return result
end

function CSVProcessor:output(data)
    print("[CSV] Outputting results...")
    for i, row in ipairs(data) do
        local parts = {}
        for k, v in pairs(row) do
            table.insert(parts, k .. "=" .. v)
        end
        print(string.format("  Row %d: {%s}", i, table.concat(parts, ", ")))
    end
    return data
end

function CSVProcessor:cleanup()
    print("[CSV] Cleaning up...")
end

-- ทดสอบ
local csvData = "Name,Age,City\nAlice,30,Bangkok\nBob,25,Chiang Mai\nCharlie,35,Phuket"
local processor = CSVProcessor.new()
processor:process(csvData)
```

## 38.13 State Machine Pattern (Game AI)

```lua
-- ตัวอย่างที่ 16: State Machine - Game AI
print("=== State Machine - Game AI ===")

-- State interface
local State = {}
State.__index = State
function State:enter(entity) end
function State:update(entity, dt) end
function State:exit(entity) end
function State:getName() return "Unknown" end

-- Concrete States
local IdleState = setmetatable({}, {__index = State})
IdleState.__index = IdleState

function IdleState.new()
    return setmetatable({name = "Idle", timer = 0}, IdleState)
end

function IdleState:getName() return "Idle" end

function IdleState:enter(entity)
    print(string.format("[%s] Entering IDLE state", entity.name))
    entity.velocity = {x = 0, y = 0}
end

function IdleState:update(entity, dt)
    self.timer = self.timer + dt
    if entity:canSeePlayer() then
        entity:changeState("chase")
    elseif self.timer > 3.0 then
        self.timer = 0
        entity:changeState("patrol")
    end
end

local PatrolState = setmetatable({}, {__index = State})
PatrolState.__index = PatrolState

function PatrolState.new()
    return setmetatable({name = "Patrol", waypointIndex = 1}, PatrolState)
end

function PatrolState:getName() return "Patrol" end

function PatrolState:enter(entity)
    print(string.format("[%s] Entering PATROL state", entity.name))
    entity.velocity = {x = 1, y = 0}
end

function PatrolState:update(entity, dt)
    if entity:canSeePlayer() then
        entity:changeState("chase")
        return
    end
    entity.x = entity.x + entity.velocity.x * dt
    if entity.x > 10 or entity.x < 0 then
        entity.velocity.x = -entity.velocity.x
    end
end

local ChaseState = setmetatable({}, {__index = State})
ChaseState.__index = ChaseState

function ChaseState.new()
    return setmetatable({name = "Chase"}, ChaseState)
end

function ChaseState:getName() return "Chase" end

function ChaseState:enter(entity)
    print(string.format("[%s] Entering CHASE state - FOUND PLAYER!", entity.name))
    entity.velocity = {x = 3, y = 0}
end

function ChaseState:update(entity, dt)
    if entity:isInAttackRange() then
        entity:changeState("attack")
    elseif not entity:canSeePlayer() then
        entity:changeState("idle")
    end
end

local AttackState = setmetatable({}, {__index = State})
AttackState.__index = AttackState

function AttackState.new()
    return setmetatable({name = "Attack", attackCooldown = 0}, AttackState)
end

function AttackState:getName() return "Attack" end

function AttackState:enter(entity)
    print(string.format("[%s] Entering ATTACK state!", entity.name))
end

function AttackState:update(entity, dt)
    self.attackCooldown = self.attackCooldown - dt
    if self.attackCooldown <= 0 then
        self.attackCooldown = 1.0
        print(string.format("[%s] ATTACK! Dealing %d damage", entity.name, entity.damage))
    end
    if not entity:isInAttackRange() then
        entity:changeState("chase")
    end
end

-- Enemy entity with state machine
local Enemy = {}
Enemy.__index = Enemy

function Enemy.new(name, x, y)
    local states = {
        idle   = IdleState.new(),
        patrol = PatrolState.new(),
        chase  = ChaseState.new(),
        attack = AttackState.new(),
    }
    
    local enemy = setmetatable({
        name = name,
        x = x, y = y,
        velocity = {x = 0, y = 0},
        damage = 10,
        _states = states,
        _currentState = states.idle,
        
        -- Simulated environment
        playerX = 15,
        playerY = 0,
        sightRange = 8,
        attackRange = 2,
    }, Enemy)
    
    enemy._currentState:enter(enemy)
    return enemy
end

function Enemy:changeState(stateName)
    if self._currentState then
        self._currentState:exit(self)
    end
    self._currentState = self._states[stateName]
    if self._currentState then
        self._currentState:enter(self)
    end
end

function Enemy:getCurrentState()
    return self._currentState:getName()
end

function Enemy:canSeePlayer()
    local dist = math.abs(self.x - self.playerX)
    return dist <= self.sightRange
end

function Enemy:isInAttackRange()
    local dist = math.abs(self.x - self.playerX)
    return dist <= self.attackRange
end

function Enemy:update(dt)
    self._currentState:update(self, dt)
end

-- Simulate game loop
local enemy = Enemy.new("Guard", 0, 0)

print("\n-- Game Loop Simulation --")
local scenarios = {
    {dt = 2.0, playerX = 15, desc = "Player far away"},
    {dt = 0.5, playerX = 5, desc = "Player enters sight range"},
    {dt = 0.5, playerX = 1, desc = "Player in attack range"},
    {dt = 1.5, playerX = 1, desc = "Attacking"},
    {dt = 0.5, playerX = 20, desc = "Player escapes"},
}

for _, s in ipairs(scenarios) do
    print(string.format("\n[Scenario: %s]", s.desc))
    enemy.playerX = s.playerX
    enemy:update(s.dt)
    print(string.format("  State: %s, EnemyX: %.1f", enemy:getCurrentState(), enemy.x))
end
```

## 38.14 Chain of Responsibility Pattern

```lua
-- ตัวอย่างที่ 17: Chain of Responsibility
print("=== Chain of Responsibility ===")

local Handler = {}
Handler.__index = Handler

function Handler.new(name, level)
    return setmetatable({
        name = name,
        level = level,
        _next = nil
    }, Handler)
end

function Handler:setNext(handler)
    self._next = handler
    return handler  -- for chaining
end

function Handler:handle(request)
    if request.level <= self.level then
        print(string.format("[%s] Handling request: %s (level %d)",
            self.name, request.message, request.level))
        return true
    elseif self._next then
        print(string.format("[%s] Passing up to next handler...", self.name))
        return self._next:handle(request)
    else
        print(string.format("[UNHANDLED] Request '%s' (level %d) was not handled",
            request.message, request.level))
        return false
    end
end

-- Support ticket escalation system
local level1 = Handler.new("L1 Support", 1)  -- handles level 1
local level2 = Handler.new("L2 Support", 2)  -- handles level 1-2
local level3 = Handler.new("L3 Support", 3)  -- handles level 1-3
local manager = Handler.new("Manager",   5)  -- handles anything

level1:setNext(level2):setNext(level3):setNext(manager)

local tickets = {
    {level = 1, message = "Password reset"},
    {level = 2, message = "Account locked"},
    {level = 3, message = "Data corruption"},
    {level = 5, message = "Security breach"},
    {level = 6, message = "Alien invasion"},
}

for _, ticket in ipairs(tickets) do
    print(string.format("\n-- Ticket: %s (L%d) --", ticket.message, ticket.level))
    level1:handle(ticket)
end
```

## 38.15 Mediator Pattern

```lua
-- ตัวอย่างที่ 18: Mediator Pattern - Chat Room
print("=== Mediator Pattern - Chat Room ===")

-- Mediator
local ChatRoom = {}
ChatRoom.__index = ChatRoom

function ChatRoom.new(name)
    return setmetatable({
        name = name,
        _participants = {},
        _messageLog = {},
    }, ChatRoom)
end

function ChatRoom:join(user)
    self._participants[user.name] = user
    user._room = self
    print(string.format("[%s] %s joined the room", self.name, user.name))
    self:broadcast(string.format("%s has joined!", user.name), "System")
end

function ChatRoom:leave(user)
    self._participants[user.name] = nil
    user._room = nil
    print(string.format("[%s] %s left the room", self.name, user.name))
    self:broadcast(string.format("%s has left.", user.name), "System")
end

function ChatRoom:send(message, fromUser, toUserName)
    local entry = {
        from = fromUser.name,
        to = toUserName,
        message = message,
        time = os.time()
    }
    table.insert(self._messageLog, entry)
    
    if toUserName then
        -- Private message
        local target = self._participants[toUserName]
        if target then
            target:receive(message, fromUser.name, true)
        else
            print(string.format("[%s] User %s not found", self.name, toUserName))
        end
    else
        -- Broadcast
        self:broadcast(message, fromUser.name)
    end
end

function ChatRoom:broadcast(message, fromName)
    for name, user in pairs(self._participants) do
        if name ~= fromName then
            user:receive(message, fromName, false)
        end
    end
end

-- Colleague
local ChatUser = {}
ChatUser.__index = ChatUser

function ChatUser.new(name)
    return setmetatable({name = name, _room = nil}, ChatUser)
end

function ChatUser:send(message, to)
    if not self._room then
        error("Not in a room!")
    end
    if to then
        print(string.format("[%s -> %s (private)]: %s", self.name, to, message))
    else
        print(string.format("[%s -> All]: %s", self.name, message))
    end
    self._room:send(message, self, to)
end

function ChatUser:receive(message, from, isPrivate)
    local prefix = isPrivate and "[PM]" or ""
    print(string.format("  [%s receives%s from %s]: %s", 
        self.name, prefix, from, message))
end

-- ทดสอบ
local room = ChatRoom.new("LuaChat")
local alice = ChatUser.new("Alice")
local bob = ChatUser.new("Bob")
local charlie = ChatUser.new("Charlie")

room:join(alice)
room:join(bob)
room:join(charlie)

print()
alice:send("Hello everyone!")
bob:send("Hi Alice!")
alice:send("How's it going, Bob?", "Bob")  -- private
charlie:send("I can see public messages but not PMs to others!")
```

## 38.16 Memento Pattern

```lua
-- ตัวอย่างที่ 19: Memento Pattern - Game Save System
print("=== Memento Pattern - Game Save ===")

-- Memento
local GameState = {}
GameState.__index = GameState

function GameState.new(level, hp, mp, x, y, inventory)
    return setmetatable({
        level = level,
        hp = hp,
        mp = mp,
        x = x,
        y = y,
        inventory = {table.unpack(inventory or {})},
        timestamp = os.time()
    }, GameState)
end

function GameState:describe()
    return string.format(
        "L%d HP:%d MP:%d Pos:(%d,%d) Items:[%s]",
        self.level, self.hp, self.mp, self.x, self.y,
        table.concat(self.inventory, ",")
    )
end

-- Originator
local GameCharacter_M = {}
GameCharacter_M.__index = GameCharacter_M

function GameCharacter_M.new(name)
    return setmetatable({
        name = name,
        level = 1,
        hp = 100,
        mp = 50,
        x = 0, y = 0,
        inventory = {},
    }, GameCharacter_M)
end

function GameCharacter_M:save()
    return GameState.new(self.level, self.hp, self.mp, 
        self.x, self.y, self.inventory)
end

function GameCharacter_M:restore(state)
    self.level = state.level
    self.hp = state.hp
    self.mp = state.mp
    self.x = state.x
    self.y = state.y
    self.inventory = {table.unpack(state.inventory)}
    print(string.format("[%s] Restored state: %s", self.name, state:describe()))
end

function GameCharacter_M:describe()
    return string.format("[%s] Level:%d HP:%d/%d MP:%d Pos:(%d,%d)",
        self.name, self.level, self.hp, 100 + self.level * 10,
        self.mp, self.x, self.y)
end

-- Caretaker
local SaveSystem = {}
SaveSystem.__index = SaveSystem

function SaveSystem.new(maxSlots)
    return setmetatable({
        _saves = {},
        _maxSlots = maxSlots or 5,
        _autoSaveSlot = "auto"
    }, SaveSystem)
end

function SaveSystem:save(character, slotName)
    slotName = slotName or "slot_" .. (os.time() % 1000)
    if self:count() >= self._maxSlots and not self._saves[slotName] then
        print("[SaveSystem] Warning: Max save slots reached!")
    end
    self._saves[slotName] = character:save()
    print(string.format("[SaveSystem] Saved to slot '%s': %s",
        slotName, self._saves[slotName]:describe()))
    return slotName
end

function SaveSystem:load(character, slotName)
    local state = self._saves[slotName]
    if not state then
        error("Save slot not found: " .. slotName)
    end
    character:restore(state)
end

function SaveSystem:count()
    local n = 0
    for _ in pairs(self._saves) do n = n + 1 end
    return n
end

function SaveSystem:listSlots()
    local slots = {}
    for name, state in pairs(self._saves) do
        table.insert(slots, string.format("  [%s] %s", name, state:describe()))
    end
    return slots
end

-- Game simulation
local hero = GameCharacter_M.new("Hero")
local saves = SaveSystem.new(5)

print(hero:describe())
saves:save(hero, "start")

-- Play game
hero.level = 5
hero.hp = 200
hero.x = 50
hero.y = 30
table.insert(hero.inventory, "Sword")
table.insert(hero.inventory, "Shield")
print(hero:describe())
saves:save(hero, "dungeon_entrance")

hero.level = 10
hero.hp = 50   -- nearly dead!
hero.x = 100
hero.y = 80
table.insert(hero.inventory, "Dragon Egg")
print(hero:describe())
saves:save(hero, "boss_room")

print("\n-- Load from dungeon entrance --")
saves:load(hero, "dungeon_entrance")
print(hero:describe())

print("\n-- All Save Slots --")
for _, slot in ipairs(saves:listSlots()) do
    print(slot)
end
```

## 38.17 Visitor Pattern

```lua
-- ตัวอย่างที่ 20: Visitor Pattern
print("=== Visitor Pattern ===")

-- Element interface
local Shape_V = {}
Shape_V.__index = Shape_V
function Shape_V:accept(visitor) end

-- Concrete Elements
local Circle_V = setmetatable({}, {__index = Shape_V})
Circle_V.__index = Circle_V

function Circle_V.new(radius)
    return setmetatable({radius = radius, type = "circle"}, Circle_V)
end

function Circle_V:accept(visitor)
    return visitor:visitCircle(self)
end

local Rectangle_V = setmetatable({}, {__index = Shape_V})
Rectangle_V.__index = Rectangle_V

function Rectangle_V.new(width, height)
    return setmetatable({width = width, height = height, type = "rectangle"}, Rectangle_V)
end

function Rectangle_V:accept(visitor)
    return visitor:visitRectangle(self)
end

local Triangle_V = setmetatable({}, {__index = Shape_V})
Triangle_V.__index = Triangle_V

function Triangle_V.new(base, height)
    return setmetatable({base = base, height = height, type = "triangle"}, Triangle_V)
end

function Triangle_V:accept(visitor)
    return visitor:visitTriangle(self)
end

-- Visitors
local AreaCalculator = {}
AreaCalculator.__index = AreaCalculator

function AreaCalculator.new()
    return setmetatable({totalArea = 0}, AreaCalculator)
end

function AreaCalculator:visitCircle(circle)
    local area = math.pi * circle.radius ^ 2
    self.totalArea = self.totalArea + area
    print(string.format("  Circle(r=%d): area = %.2f", circle.radius, area))
    return area
end

function AreaCalculator:visitRectangle(rect)
    local area = rect.width * rect.height
    self.totalArea = self.totalArea + area
    print(string.format("  Rectangle(%dx%d): area = %d", rect.width, rect.height, area))
    return area
end

function AreaCalculator:visitTriangle(tri)
    local area = 0.5 * tri.base * tri.height
    self.totalArea = self.totalArea + area
    print(string.format("  Triangle(b=%d, h=%d): area = %.1f", tri.base, tri.height, area))
    return area
end

local ExportVisitor = {}
ExportVisitor.__index = ExportVisitor

function ExportVisitor.new(format)
    return setmetatable({format = format, output = {}}, ExportVisitor)
end

function ExportVisitor:visitCircle(circle)
    if self.format == "SVG" then
        local svg = string.format('<circle r="%d" cx="0" cy="0"/>', circle.radius)
        table.insert(self.output, svg)
        return svg
    end
    return string.format("circle(%d)", circle.radius)
end

function ExportVisitor:visitRectangle(rect)
    if self.format == "SVG" then
        local svg = string.format('<rect width="%d" height="%d"/>', rect.width, rect.height)
        table.insert(self.output, svg)
        return svg
    end
    return string.format("rect(%dx%d)", rect.width, rect.height)
end

function ExportVisitor:visitTriangle(tri)
    if self.format == "SVG" then
        local svg = string.format('<polygon points="0,0 %d,0 %d,-%d"/>',
            tri.base, tri.base / 2, tri.height)
        table.insert(self.output, svg)
        return svg
    end
    return string.format("triangle(%d, %d)", tri.base, tri.height)
end

function ExportVisitor:getOutput()
    return table.concat(self.output, "\n")
end

-- ทดสอบ
local shapes = {
    Circle_V.new(5),
    Rectangle_V.new(4, 6),
    Triangle_V.new(3, 8),
    Circle_V.new(10),
}

print("Calculating areas:")
local areaCalc = AreaCalculator.new()
for _, shape in ipairs(shapes) do
    shape:accept(areaCalc)
end
print(string.format("Total area: %.2f", areaCalc.totalArea))

print("\nExporting to SVG:")
local exporter = ExportVisitor.new("SVG")
for _, shape in ipairs(shapes) do
    shape:accept(exporter)
end
print(exporter:getOutput())
```

## 38.18 สรุป Patterns ทั้งหมด

```lua
-- ตัวอย่างที่ 21: Event System using Observer + Command
print("=== Combined: Observer + Command ===")

local EventBus = {}
EventBus.__index = EventBus

function EventBus.new()
    return setmetatable({
        _handlers = {},
        _history = {},
    }, EventBus)
end

function EventBus:on(event, handler)
    if not self._handlers[event] then
        self._handlers[event] = {}
    end
    table.insert(self._handlers[event], handler)
end

function EventBus:emit(event, data)
    local entry = {event = event, data = data, time = os.time()}
    table.insert(self._history, entry)
    
    if self._handlers[event] then
        for _, handler in ipairs(self._handlers[event]) do
            handler(data)
        end
    end
end

function EventBus:getHistory()
    return self._history
end

local bus = EventBus.new()

bus:on("user.login", function(data)
    print(string.format("[Auth] User %s logged in from %s", data.user, data.ip))
end)

bus:on("user.login", function(data)
    print(string.format("[Audit] Login recorded: %s", data.user))
end)

bus:on("order.created", function(data)
    print(string.format("[Email] Sending order confirmation to %s", data.email))
end)

bus:on("order.created", function(data)
    print(string.format("[Inventory] Reserving items for order #%s", data.orderId))
end)

bus:emit("user.login", {user = "alice", ip = "192.168.1.1"})
bus:emit("order.created", {orderId = "ORD-001", email = "alice@test.com", total = 99.99})
bus:emit("user.login", {user = "bob", ip = "10.0.0.1"})

print(string.format("\nEvent history: %d events recorded", #bus:getHistory()))
```

---

## แบบฝึกหัดบทที่ 38

1. สร้าง `LoggingDecorator` สำหรับ Database connection ที่ log ทุก query
2. เขียน State Machine สำหรับ Traffic Light (Red -> Green -> Yellow -> Red)
3. Implement `CommandQueue` ที่รองรับ batch execution และ undo ทั้ง batch
4. สร้าง Composite สำหรับ expression tree (Add, Multiply, Literal) ที่ evaluate ได้
5. เขียน Visitor สำหรับ serialization ของ AST (Abstract Syntax Tree)
