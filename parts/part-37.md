# บทที่ 37: Design Patterns - Creational (รูปแบบการออกแบบเชิงสร้างสรรค์)

## บทนำ

Design Patterns คือแนวทางแก้ปัญหาที่ได้รับการพิสูจน์แล้วว่าใช้ได้ผลในการออกแบบ software ซึ่ง Gang of Four (GoF) ได้จัดแบ่งออกเป็น 3 กลุ่มหลัก ได้แก่ Creational, Structural และ Behavioral

**Creational Patterns** เน้นการสร้าง objects อย่างยืดหยุ่นและเหมาะสม ช่วยให้ code ไม่ต้องพึ่งพา concrete classes โดยตรง

## 37.1 Singleton Pattern

Singleton คือ pattern ที่รับประกันว่ามี instance ของ class ได้เพียงหนึ่งเดียว และให้ global access point

```lua
-- ตัวอย่างที่ 1: Singleton พื้นฐาน
print("=== Singleton Pattern พื้นฐาน ===")

local Singleton = {}
Singleton.__index = Singleton

local instance = nil

function Singleton.getInstance()
    if instance == nil then
        instance = setmetatable({
            data = {},
            createdAt = os.time()
        }, Singleton)
        print("Creating new Singleton instance")
    else
        print("Returning existing Singleton instance")
    end
    return instance
end

function Singleton:set(key, value)
    self.data[key] = value
end

function Singleton:get(key)
    return self.data[key]
end

-- ทดสอบ
local s1 = Singleton.getInstance()
local s2 = Singleton.getInstance()
local s3 = Singleton.getInstance()

s1:set("name", "World")
print(string.format("s1 == s2: %s", tostring(s1 == s2)))
print(string.format("s2 == s3: %s", tostring(s2 == s3)))
print(string.format("s3:get('name') = %s", s3:get("name")))
```

```lua
-- ตัวอย่างที่ 2: Thread-safe Singleton (using closure)
print("=== Singleton ด้วย Closure ===")

local function createSingletonFactory()
    local _instance = nil
    
    return {
        getInstance = function()
            if _instance == nil then
                _instance = {
                    id = math.random(1000, 9999),
                    config = {},
                    
                    setConfig = function(self, key, val)
                        self.config[key] = val
                    end,
                    
                    getConfig = function(self, key)
                        return self.config[key]
                    end
                }
            end
            return _instance
        end,
        
        resetInstance = function()  -- สำหรับ testing
            _instance = nil
        end
    }
end

local AppConfig = createSingletonFactory()

local config1 = AppConfig.getInstance()
config1:setConfig("debug", true)
config1:setConfig("version", "1.0.0")

local config2 = AppConfig.getInstance()
print(string.format("Same instance: %s", tostring(config1 == config2)))
print(string.format("debug: %s", tostring(config2:getConfig("debug"))))
print(string.format("version: %s", config2:getConfig("version")))
```

```lua
-- ตัวอย่างที่ 3: Logger Singleton (real-world)
print("=== Logger Singleton ===")

local Logger = (function()
    local _instance = nil
    local levels = {DEBUG=1, INFO=2, WARN=3, ERROR=4}
    
    local LoggerClass = {}
    LoggerClass.__index = LoggerClass
    
    function LoggerClass:log(level, message)
        if levels[level] >= levels[self.minLevel] then
            local timestamp = os.date("%H:%M:%S")
            print(string.format("[%s] [%s] %s", timestamp, level, message))
        end
    end
    
    function LoggerClass:debug(msg) self:log("DEBUG", msg) end
    function LoggerClass:info(msg)  self:log("INFO",  msg) end
    function LoggerClass:warn(msg)  self:log("WARN",  msg) end
    function LoggerClass:error(msg) self:log("ERROR", msg) end
    
    function LoggerClass:setLevel(level)
        assert(levels[level], "Invalid log level: " .. tostring(level))
        self.minLevel = level
    end
    
    return {
        getInstance = function()
            if _instance == nil then
                _instance = setmetatable({
                    minLevel = "INFO",
                    logCount = 0
                }, LoggerClass)
            end
            return _instance
        end
    }
end)()

local log = Logger.getInstance()
log:info("Application started")
log:debug("Debug message (hidden)")
log:setLevel("DEBUG")
log:debug("Debug message (now visible)")
log:warn("Low memory warning")
log:error("Critical error!")
```

```lua
-- ตัวอย่างที่ 4: Database Connection Singleton
print("=== Database Connection Singleton ===")

local Database = (function()
    local _connection = nil
    
    local DB = {}
    DB.__index = DB
    
    function DB:query(sql)
        print(string.format("[DB Query] %s", sql))
        return {rows = {}, count = 0}
    end
    
    function DB:execute(sql)
        print(string.format("[DB Execute] %s", sql))
        return true
    end
    
    function DB:disconnect()
        print("[DB] Disconnected")
        _connection = nil
    end
    
    return {
        connect = function(host, port, dbName)
            if _connection == nil then
                print(string.format("[DB] Connecting to %s:%d/%s", host, port, dbName))
                _connection = setmetatable({
                    host = host,
                    port = port,
                    dbName = dbName,
                    connected = true
                }, DB)
            else
                print("[DB] Reusing existing connection")
            end
            return _connection
        end,
        
        getConnection = function()
            return _connection
        end
    }
end)()

local db1 = Database.connect("localhost", 5432, "myapp")
local db2 = Database.connect("other-host", 5432, "other")  -- reuses existing

print(string.format("Same connection: %s", tostring(db1 == db2)))
db1:query("SELECT * FROM users")
db2:execute("INSERT INTO logs VALUES ('test')")
```

## 37.2 Factory Method Pattern

Factory Method กำหนด interface สำหรับสร้าง object แต่ให้ subclass เป็นผู้ตัดสินใจว่าจะสร้าง class ใด

```lua
-- ตัวอย่างที่ 5: Factory Method พื้นฐาน
print("=== Factory Method พื้นฐาน ===")

-- Product interface
local Animal = {}
Animal.__index = Animal

function Animal:speak()
    return "..."
end

function Animal:describe()
    return string.format("I am a %s and I say: %s", self.name, self:speak())
end

-- Concrete Products
local Dog = setmetatable({}, {__index = Animal})
Dog.__index = Dog

function Dog.new(name)
    return setmetatable({name = name or "Dog"}, Dog)
end

function Dog:speak() return "Woof!" end
function Dog:fetch() return self.name .. " fetches the ball!" end

local Cat = setmetatable({}, {__index = Animal})
Cat.__index = Cat

function Cat.new(name)
    return setmetatable({name = name or "Cat"}, Cat)
end

function Cat:speak() return "Meow!" end
function Cat:purr() return self.name .. " is purring..." end

local Bird = setmetatable({}, {__index = Animal})
Bird.__index = Bird

function Bird.new(name)
    return setmetatable({name = name or "Bird"}, Bird)
end

function Bird:speak() return "Tweet!" end
function Bird:fly() return self.name .. " is flying!" end

-- Factory
local AnimalFactory = {}

AnimalFactory.creators = {
    dog  = Dog.new,
    cat  = Cat.new,
    bird = Bird.new,
}

function AnimalFactory.create(animalType, name)
    local creator = AnimalFactory.creators[animalType:lower()]
    if not creator then
        error("Unknown animal type: " .. animalType)
    end
    return creator(name)
end

function AnimalFactory.register(type, creator)
    AnimalFactory.creators[type:lower()] = creator
end

-- ทดสอบ
local animals = {
    AnimalFactory.create("dog", "Rex"),
    AnimalFactory.create("cat", "Whiskers"),
    AnimalFactory.create("bird", "Tweety"),
}

for _, animal in ipairs(animals) do
    print(animal:describe())
end
```

```lua
-- ตัวอย่างที่ 6: UI Widget Factory
print("=== UI Widget Factory ===")

-- Widget base class
local Widget = {}
Widget.__index = Widget

function Widget:render()
    return string.format("<%s>%s</%s>", self.tag, self.content, self.tag)
end

function Widget:setContent(content)
    self.content = content
    return self  -- for chaining
end

-- Concrete widgets
local Button = setmetatable({}, {__index = Widget})
Button.__index = Button

function Button.new(text, style)
    return setmetatable({
        tag = "button",
        content = text or "Click me",
        style = style or "default"
    }, Button)
end

function Button:render()
    return string.format('<button class="%s">%s</button>', self.style, self.content)
end

local Input = setmetatable({}, {__index = Widget})
Input.__index = Input

function Input.new(type, placeholder)
    return setmetatable({
        tag = "input",
        inputType = type or "text",
        placeholder = placeholder or "",
        content = ""
    }, Input)
end

function Input:render()
    return string.format('<input type="%s" placeholder="%s">', 
        self.inputType, self.placeholder)
end

local Label = setmetatable({}, {__index = Widget})
Label.__index = Label

function Label.new(text)
    return setmetatable({tag = "label", content = text or ""}, Label)
end

-- Widget Factory
local WidgetFactory = {}

function WidgetFactory.create(widgetType, ...)
    local factories = {
        button = Button.new,
        input  = Input.new,
        label  = Label.new,
    }
    local factory = factories[widgetType:lower()]
    if not factory then
        error("Unknown widget type: " .. widgetType)
    end
    return factory(...)
end

-- ทดสอบ
local widgets = {
    WidgetFactory.create("button", "Submit", "primary"),
    WidgetFactory.create("input", "email", "Enter your email"),
    WidgetFactory.create("label", "Username:"),
    WidgetFactory.create("button", "Cancel", "secondary"),
}

print("Generated HTML:")
for _, widget in ipairs(widgets) do
    print("  " .. widget:render())
end
```

```lua
-- ตัวอย่างที่ 7: Logger Factory (multiple output targets)
print("=== Logger Factory ===")

local ConsoleLogger = {}
ConsoleLogger.__index = ConsoleLogger

function ConsoleLogger.new(prefix)
    return setmetatable({prefix = prefix or "[LOG]"}, ConsoleLogger)
end

function ConsoleLogger:write(level, message)
    print(string.format("%s [%s] %s", self.prefix, level, message))
end

local FileLogger = {}
FileLogger.__index = FileLogger

function FileLogger.new(filename)
    return setmetatable({filename = filename, buffer = {}}, FileLogger)
end

function FileLogger:write(level, message)
    local entry = string.format("[%s] [%s] %s\n", os.date("%Y-%m-%d %H:%M:%S"), level, message)
    table.insert(self.buffer, entry)
    -- In real code: write to file
    print(string.format("[FILE:%s] %s", self.filename, message))
end

function FileLogger:flush()
    -- Write buffer to file
    self.buffer = {}
end

local NullLogger = {}
NullLogger.__index = NullLogger

function NullLogger.new()
    return setmetatable({}, NullLogger)
end

function NullLogger:write(level, message) end  -- do nothing

local LoggerFactory = {}

function LoggerFactory.create(loggerType, ...)
    local factories = {
        console = ConsoleLogger.new,
        file    = FileLogger.new,
        null    = NullLogger.new,
    }
    local factory = factories[loggerType:lower()]
    if not factory then
        error("Unknown logger type: " .. loggerType)
    end
    return factory(...)
end

-- ทดสอบ
local consoleLog = LoggerFactory.create("console", "[APP]")
local fileLog = LoggerFactory.create("file", "app.log")
local nullLog = LoggerFactory.create("null")

consoleLog:write("INFO", "Server started")
fileLog:write("WARN", "High memory usage")
nullLog:write("DEBUG", "This goes nowhere")
```

## 37.3 Abstract Factory Pattern

Abstract Factory สร้าง family ของ objects ที่เกี่ยวข้องกันโดยไม่ระบุ concrete classes

```lua
-- ตัวอย่างที่ 8: Abstract Factory - UI Themes
print("=== Abstract Factory - UI Themes ===")

-- Abstract products
local Button_AF = {}
Button_AF.__index = Button_AF
function Button_AF:click() return "Button clicked" end
function Button_AF:render() return "Generic Button" end

local TextBox_AF = {}
TextBox_AF.__index = TextBox_AF
function TextBox_AF:input(text) self.value = text end
function TextBox_AF:getValue() return self.value or "" end
function TextBox_AF:render() return "Generic TextBox" end

-- Light Theme products
local LightButton = setmetatable({}, {__index = Button_AF})
LightButton.__index = LightButton

function LightButton.new(text)
    return setmetatable({text = text, theme = "light"}, LightButton)
end

function LightButton:render()
    return string.format("[LIGHT BUTTON: %s | bg=white, text=black]", self.text)
end

local LightTextBox = setmetatable({}, {__index = TextBox_AF})
LightTextBox.__index = LightTextBox

function LightTextBox.new(placeholder)
    return setmetatable({placeholder = placeholder, theme = "light", value = ""}, LightTextBox)
end

function LightTextBox:render()
    return string.format("[LIGHT TEXTBOX: '%s' | border=gray]", self.placeholder)
end

-- Dark Theme products
local DarkButton = setmetatable({}, {__index = Button_AF})
DarkButton.__index = DarkButton

function DarkButton.new(text)
    return setmetatable({text = text, theme = "dark"}, DarkButton)
end

function DarkButton:render()
    return string.format("[DARK BUTTON: %s | bg=black, text=white]", self.text)
end

local DarkTextBox = setmetatable({}, {__index = TextBox_AF})
DarkTextBox.__index = DarkTextBox

function DarkTextBox.new(placeholder)
    return setmetatable({placeholder = placeholder, theme = "dark", value = ""}, DarkTextBox)
end

function DarkTextBox:render()
    return string.format("[DARK TEXTBOX: '%s' | border=white]", self.placeholder)
end

-- Abstract Factories
local LightThemeFactory = {
    createButton  = function(text) return LightButton.new(text) end,
    createTextBox = function(ph)   return LightTextBox.new(ph) end,
    name = "Light Theme"
}

local DarkThemeFactory = {
    createButton  = function(text) return DarkButton.new(text) end,
    createTextBox = function(ph)   return DarkTextBox.new(ph) end,
    name = "Dark Theme"
}

-- Application สร้าง UI โดยใช้ factory
local function buildLoginForm(factory)
    print(string.format("\n--- Building Login Form with %s ---", factory.name))
    local components = {
        factory.createTextBox("Enter username"),
        factory.createTextBox("Enter password"),
        factory.createButton("Login"),
        factory.createButton("Cancel"),
    }
    for _, c in ipairs(components) do
        print("  " .. c:render())
    end
end

buildLoginForm(LightThemeFactory)
buildLoginForm(DarkThemeFactory)
```

```lua
-- ตัวอย่างที่ 9: Abstract Factory - Database backends
print("=== Abstract Factory - Database Backends ===")

-- MySQL implementations
local MySQLConnection = {}
MySQLConnection.__index = MySQLConnection

function MySQLConnection.new(config)
    print(string.format("[MySQL] Connecting to %s:%d", config.host, config.port))
    return setmetatable({config = config, dbType = "MySQL"}, MySQLConnection)
end

function MySQLConnection:query(sql)
    print(string.format("[MySQL] QUERY: %s", sql))
    return {rows = {}, total = 0}
end

local MySQLMigration = {}
MySQLMigration.__index = MySQLMigration

function MySQLMigration.new()
    return setmetatable({dbType = "MySQL"}, MySQLMigration)
end

function MySQLMigration:createTable(name, cols)
    print(string.format("[MySQL] CREATE TABLE %s (%s)", name, table.concat(cols, ", ")))
end

-- SQLite implementations
local SQLiteConnection = {}
SQLiteConnection.__index = SQLiteConnection

function SQLiteConnection.new(config)
    print(string.format("[SQLite] Opening file: %s", config.filename))
    return setmetatable({config = config, dbType = "SQLite"}, SQLiteConnection)
end

function SQLiteConnection:query(sql)
    print(string.format("[SQLite] QUERY: %s", sql))
    return {rows = {}, total = 0}
end

local SQLiteMigration = {}
SQLiteMigration.__index = SQLiteMigration

function SQLiteMigration.new()
    return setmetatable({dbType = "SQLite"}, SQLiteMigration)
end

function SQLiteMigration:createTable(name, cols)
    print(string.format("[SQLite] CREATE TABLE IF NOT EXISTS %s (%s)", 
        name, table.concat(cols, ", ")))
end

-- Database Factories
local MySQLFactory = {
    createConnection = function(config) return MySQLConnection.new(config) end,
    createMigration  = function()       return MySQLMigration.new() end,
    name = "MySQL"
}

local SQLiteFactory = {
    createConnection = function(config) return SQLiteConnection.new(config) end,
    createMigration  = function()       return SQLiteMigration.new() end,
    name = "SQLite"
}

-- Setup application
local function setupApplication(dbFactory, config)
    print(string.format("\n--- Setup with %s ---", dbFactory.name))
    local conn = dbFactory.createConnection(config)
    local migration = dbFactory.createMigration()
    
    migration:createTable("users", {"id INT PRIMARY KEY", "name VARCHAR(100)", "email VARCHAR(200)"})
    migration:createTable("posts", {"id INT PRIMARY KEY", "title VARCHAR(200)", "user_id INT"})
    
    conn:query("SELECT * FROM users LIMIT 10")
    return conn
end

setupApplication(MySQLFactory, {host = "localhost", port = 3306, dbName = "myapp"})
setupApplication(SQLiteFactory, {filename = "myapp.db"})
```

## 37.4 Builder Pattern

Builder แยกการสร้าง complex object ออกจาก representation ของมัน

```lua
-- ตัวอย่างที่ 10: Builder Pattern พื้นฐาน
print("=== Builder Pattern ===")

local Pizza = {}
Pizza.__index = Pizza

function Pizza.new(builder)
    return setmetatable({
        size     = builder.size,
        crust    = builder.crust,
        sauce    = builder.sauce,
        cheese   = builder.cheese,
        toppings = builder.toppings or {},
    }, Pizza)
end

function Pizza:describe()
    return string.format(
        "Pizza [%s, %s crust, %s sauce, %s cheese, toppings: %s]",
        self.size,
        self.crust,
        self.sauce,
        self.cheese,
        #self.toppings > 0 and table.concat(self.toppings, ", ") or "none"
    )
end

-- Pizza Builder
local PizzaBuilder = {}
PizzaBuilder.__index = PizzaBuilder

function PizzaBuilder.new()
    return setmetatable({
        size = "medium",
        crust = "thin",
        sauce = "tomato",
        cheese = "mozzarella",
        toppings = {}
    }, PizzaBuilder)
end

function PizzaBuilder:setSize(size)
    self.size = size
    return self
end

function PizzaBuilder:setCrust(crust)
    self.crust = crust
    return self
end

function PizzaBuilder:setSauce(sauce)
    self.sauce = sauce
    return self
end

function PizzaBuilder:setCheese(cheese)
    self.cheese = cheese
    return self
end

function PizzaBuilder:addTopping(topping)
    table.insert(self.toppings, topping)
    return self
end

function PizzaBuilder:build()
    return Pizza.new(self)
end

-- ทดสอบ
local veggiePizza = PizzaBuilder.new()
    :setSize("large")
    :setCrust("thick")
    :setSauce("pesto")
    :setCheese("gouda")
    :addTopping("mushrooms")
    :addTopping("bell peppers")
    :addTopping("olives")
    :build()

local meatPizza = PizzaBuilder.new()
    :setSize("medium")
    :setSauce("bbq")
    :addTopping("pepperoni")
    :addTopping("bacon")
    :addTopping("sausage")
    :build()

print(veggiePizza:describe())
print(meatPizza:describe())
```

```lua
-- ตัวอย่างที่ 11: Configuration Builder
print("=== Configuration Builder ===")

local Config = {}
Config.__index = Config

function Config.new(data)
    return setmetatable({_data = data}, Config)
end

function Config:get(key, default)
    local parts = {}
    for part in key:gmatch("[^.]+") do
        table.insert(parts, part)
    end
    
    local current = self._data
    for _, part in ipairs(parts) do
        if type(current) ~= "table" then return default end
        current = current[part]
    end
    
    return current ~= nil and current or default
end

-- Config Builder
local ConfigBuilder = {}
ConfigBuilder.__index = ConfigBuilder

function ConfigBuilder.new()
    return setmetatable({
        _config = {
            server   = {},
            database = {},
            cache    = {},
            logging  = {},
        }
    }, ConfigBuilder)
end

function ConfigBuilder:server(host, port)
    self._config.server.host = host
    self._config.server.port = port
    return self
end

function ConfigBuilder:database(host, port, name, user, password)
    self._config.database = {
        host = host, port = port, name = name,
        user = user, password = password
    }
    return self
end

function ConfigBuilder:cache(driver, host, port, ttl)
    self._config.cache = {
        driver = driver, host = host, port = port, ttl = ttl
    }
    return self
end

function ConfigBuilder:logging(level, file)
    self._config.logging = {level = level, file = file}
    return self
end

function ConfigBuilder:build()
    return Config.new(self._config)
end

local config = ConfigBuilder.new()
    :server("0.0.0.0", 8080)
    :database("localhost", 5432, "myapp", "user", "pass")
    :cache("redis", "localhost", 6379, 3600)
    :logging("INFO", "app.log")
    :build()

print(string.format("Server: %s:%s", config:get("server.host"), config:get("server.port")))
print(string.format("DB: %s/%s", config:get("database.host"), config:get("database.name")))
print(string.format("Cache TTL: %s", config:get("cache.ttl")))
print(string.format("Log Level: %s", config:get("logging.level")))
```

```lua
-- ตัวอย่างที่ 12: HTTP Request Builder
print("=== HTTP Request Builder ===")

local HttpRequest = {}
HttpRequest.__index = HttpRequest

function HttpRequest.new(data)
    return setmetatable(data, HttpRequest)
end

function HttpRequest:toString()
    local lines = {
        string.format("%s %s HTTP/1.1", self.method, self.path),
        string.format("Host: %s", self.host),
    }
    for k, v in pairs(self.headers) do
        table.insert(lines, string.format("%s: %s", k, v))
    end
    if self.body and #self.body > 0 then
        table.insert(lines, string.format("Content-Length: %d", #self.body))
        table.insert(lines, "")
        table.insert(lines, self.body)
    end
    return table.concat(lines, "\n")
end

local RequestBuilder = {}
RequestBuilder.__index = RequestBuilder

function RequestBuilder.new()
    return setmetatable({
        _method = "GET",
        _path = "/",
        _host = "localhost",
        _headers = {},
        _body = "",
        _queryParams = {},
    }, RequestBuilder)
end

function RequestBuilder:method(m)
    self._method = m:upper()
    return self
end

function RequestBuilder:url(host, path)
    self._host = host
    self._path = path
    return self
end

function RequestBuilder:header(key, value)
    self._headers[key] = value
    return self
end

function RequestBuilder:query(key, value)
    table.insert(self._queryParams, key .. "=" .. tostring(value))
    return self
end

function RequestBuilder:json(data)
    -- Simplified JSON serialization
    local function serialize(t, indent)
        indent = indent or ""
        if type(t) == "table" then
            local parts = {}
            for k, v in pairs(t) do
                table.insert(parts, string.format('"%s": %s', k, serialize(v, indent .. "  ")))
            end
            return "{" .. table.concat(parts, ", ") .. "}"
        elseif type(t) == "string" then
            return '"' .. t .. '"'
        else
            return tostring(t)
        end
    end
    self._body = serialize(data)
    self._headers["Content-Type"] = "application/json"
    return self
end

function RequestBuilder:build()
    local path = self._path
    if #self._queryParams > 0 then
        path = path .. "?" .. table.concat(self._queryParams, "&")
    end
    return HttpRequest.new({
        method = self._method,
        path = path,
        host = self._host,
        headers = self._headers,
        body = self._body,
    })
end

local request = RequestBuilder.new()
    :method("POST")
    :url("api.example.com", "/users")
    :header("Authorization", "Bearer token123")
    :header("Accept", "application/json")
    :json({name = "John", email = "john@example.com"})
    :build()

print(request:toString())
```

## 37.5 Prototype Pattern

Prototype ทำงานโดยการ clone object ที่มีอยู่แล้วแทนที่จะสร้างใหม่

```lua
-- ตัวอย่างที่ 13: Prototype Pattern พื้นฐาน
print("=== Prototype Pattern ===")

local function deepCopy(orig)
    local copy
    if type(orig) == "table" then
        copy = {}
        for k, v in pairs(orig) do
            copy[deepCopy(k)] = deepCopy(v)
        end
        setmetatable(copy, deepCopy(getmetatable(orig)))
    else
        copy = orig
    end
    return copy
end

local Character = {}
Character.__index = Character

function Character.new(name, class, level, stats)
    return setmetatable({
        name  = name,
        class = class,
        level = level or 1,
        stats = stats or {hp=100, mp=50, atk=10, def=10},
        skills = {},
        equipment = {},
    }, Character)
end

function Character:clone(newName)
    local copy = deepCopy(self)
    copy.name = newName or self.name .. "_copy"
    return copy
end

function Character:addSkill(skill)
    table.insert(self.skills, skill)
    return self
end

function Character:equip(item)
    table.insert(self.equipment, item)
    return self
end

function Character:describe()
    return string.format(
        "%s [%s Lv.%d] HP:%d MP:%d ATK:%d DEF:%d | Skills: %s | Equip: %s",
        self.name, self.class, self.level,
        self.stats.hp, self.stats.mp, self.stats.atk, self.stats.def,
        #self.skills > 0 and table.concat(self.skills, ",") or "none",
        #self.equipment > 0 and table.concat(self.equipment, ",") or "none"
    )
end

-- สร้าง prototype
local warriorTemplate = Character.new("Template", "Warrior", 1, 
    {hp=150, mp=30, atk=20, def=15})
warriorTemplate:addSkill("Slash"):addSkill("Block"):equip("Iron Sword"):equip("Shield")

-- Clone จาก prototype
local warrior1 = warriorTemplate:clone("Arthur")
local warrior2 = warriorTemplate:clone("Lancelot")

warrior1.level = 5
warrior1.stats.hp = 250
warrior1:addSkill("Charge")

warrior2.level = 3
warrior2:equip("Steel Helmet")

print("Template: " .. warriorTemplate:describe())
print("Clone 1:  " .. warrior1:describe())
print("Clone 2:  " .. warrior2:describe())
```

```lua
-- ตัวอย่างที่ 14: Prototype Registry
print("=== Prototype Registry ===")

local PrototypeRegistry = {}
PrototypeRegistry.__index = PrototypeRegistry

function PrototypeRegistry.new()
    return setmetatable({_prototypes = {}}, PrototypeRegistry)
end

function PrototypeRegistry:register(name, prototype)
    self._prototypes[name] = prototype
    print(string.format("[Registry] Registered prototype: '%s'", name))
end

function PrototypeRegistry:clone(name, overrides)
    local proto = self._prototypes[name]
    if not proto then
        error("Prototype not found: " .. name)
    end
    local copy = deepCopy(proto)
    if overrides then
        for k, v in pairs(overrides) do
            copy[k] = v
        end
    end
    return copy
end

function PrototypeRegistry:list()
    local names = {}
    for name, _ in pairs(self._prototypes) do
        table.insert(names, name)
    end
    table.sort(names)
    return names
end

local registry = PrototypeRegistry.new()

-- Register game item prototypes
registry:register("health_potion", {
    name = "Health Potion",
    type = "consumable",
    effect = "restore_hp",
    amount = 50,
    value = 25,
    stackable = true,
    maxStack = 99
})

registry:register("mana_potion", {
    name = "Mana Potion",
    type = "consumable",
    effect = "restore_mp",
    amount = 30,
    value = 30,
    stackable = true,
    maxStack = 99
})

registry:register("iron_sword", {
    name = "Iron Sword",
    type = "weapon",
    damage = 15,
    speed = 1.2,
    value = 100,
    stackable = false
})

-- Clone items
local bigPotion = registry:clone("health_potion", {
    name = "Super Health Potion",
    amount = 200,
    value = 100
})

local enchantedSword = registry:clone("iron_sword", {
    name = "Enchanted Iron Sword",
    damage = 25,
    magic = 10,
    value = 350
})

print("\nCloned Items:")
for k, v in pairs(bigPotion) do
    print(string.format("  %s = %s", k, tostring(v)))
end
print()
for k, v in pairs(enchantedSword) do
    print(string.format("  %s = %s", k, tostring(v)))
end
```

## 37.6 Object Pool Pattern

Object Pool จัดการ pool ของ objects ที่สามารถ reuse ได้ เหมาะกับ objects ที่สร้างได้ยาก/แพง

```lua
-- ตัวอย่างที่ 15: Object Pool พื้นฐาน
print("=== Object Pool Pattern ===")

local ObjectPool = {}
ObjectPool.__index = ObjectPool

function ObjectPool.new(factory, initialSize, maxSize)
    local pool = setmetatable({
        _factory = factory,
        _available = {},
        _inUse = {},
        _maxSize = maxSize or 10,
        _totalCreated = 0,
    }, ObjectPool)
    
    -- Pre-allocate objects
    for i = 1, (initialSize or 3) do
        pool:_create()
    end
    
    return pool
end

function ObjectPool:_create()
    if self._totalCreated >= self._maxSize then
        return nil
    end
    local obj = self._factory(self._totalCreated + 1)
    self._totalCreated = self._totalCreated + 1
    table.insert(self._available, obj)
    return obj
end

function ObjectPool:acquire()
    local obj = table.remove(self._available)
    if not obj then
        obj = self:_create()
        if not obj then
            error("Pool exhausted! Max size: " .. self._maxSize)
        end
        -- Remove from available since it was just created and removed
        table.remove(self._available)
    end
    self._inUse[obj] = true
    if obj.onAcquire then obj:onAcquire() end
    return obj
end

function ObjectPool:release(obj)
    if self._inUse[obj] then
        self._inUse[obj] = nil
        if obj.onRelease then obj:onRelease() end
        table.insert(self._available, obj)
    end
end

function ObjectPool:stats()
    local inUseCount = 0
    for _ in pairs(self._inUse) do inUseCount = inUseCount + 1 end
    return {
        total    = self._totalCreated,
        available = #self._available,
        inUse    = inUseCount,
    }
end

-- Database Connection Pool
local function createDBConnection(id)
    local conn = {
        id = id,
        isConnected = false,
        queryCount = 0,
    }
    
    function conn:onAcquire()
        self.isConnected = true
        print(string.format("[Pool] Connection %d acquired", self.id))
    end
    
    function conn:onRelease()
        self.isConnected = false
        print(string.format("[Pool] Connection %d released (queries: %d)", 
            self.id, self.queryCount))
        self.queryCount = 0
    end
    
    function conn:query(sql)
        if not self.isConnected then error("Not connected!") end
        self.queryCount = self.queryCount + 1
        print(string.format("[DB:%d] Query #%d: %s", self.id, self.queryCount, sql))
        return {}
    end
    
    return conn
end

local dbPool = ObjectPool.new(createDBConnection, 2, 5)

print("Initial stats:", dbPool:stats().available, "available")

local conn1 = dbPool:acquire()
local conn2 = dbPool:acquire()
conn1:query("SELECT * FROM users")
conn1:query("UPDATE users SET active=1")
conn2:query("SELECT * FROM posts")

dbPool:release(conn1)
print("After release stats:", dbPool:stats().available, "available")

local conn3 = dbPool:acquire()  -- gets conn1 back
conn3:query("INSERT INTO logs VALUES (1)")
dbPool:release(conn2)
dbPool:release(conn3)
```

```lua
-- ตัวอย่างที่ 16: Bullet Pool (Game development)
print("=== Bullet Pool (Game) ===")

local Bullet = {}
Bullet.__index = Bullet

function Bullet.new(id)
    return setmetatable({
        id = id,
        active = false,
        x = 0, y = 0,
        vx = 0, vy = 0,
        damage = 0,
        lifetime = 0,
    }, Bullet)
end

function Bullet:onAcquire()
    self.active = true
end

function Bullet:onRelease()
    self.active = false
    self.x, self.y = 0, 0
    self.vx, self.vy = 0, 0
    self.damage = 0
    self.lifetime = 0
end

function Bullet:fire(x, y, vx, vy, damage)
    self.x = x
    self.y = y
    self.vx = vx
    self.vy = vy
    self.damage = damage
    self.lifetime = 3.0  -- 3 seconds
end

function Bullet:update(dt)
    if not self.active then return end
    self.x = self.x + self.vx * dt
    self.y = self.y + self.vy * dt
    self.lifetime = self.lifetime - dt
end

function Bullet:isDead()
    return self.lifetime <= 0
end

local bulletPool = ObjectPool.new(Bullet.new, 10, 50)

-- Simulate firing bullets
local activeBullets = {}
for i = 1, 5 do
    local bullet = bulletPool:acquire()
    bullet:fire(0, 0, i * 10, i * 5, 25)
    table.insert(activeBullets, bullet)
    print(string.format("Fired bullet %d at velocity (%d, %d)", 
        bullet.id, bullet.vx, bullet.vy))
end

-- Simulate update
local dt = 0.5
for _, bullet in ipairs(activeBullets) do
    bullet:update(dt)
end

-- Return dead bullets to pool
local remaining = {}
for _, bullet in ipairs(activeBullets) do
    if bullet.lifetime <= 0 then
        bulletPool:release(bullet)
    else
        table.insert(remaining, bullet)
    end
end

local stats = bulletPool:stats()
print(string.format("\nPool stats: total=%d, available=%d, inUse=%d",
    stats.total, stats.available, stats.inUse))
```

## 37.7 Lazy Initialization

```lua
-- ตัวอย่างที่ 17: Lazy Initialization
print("=== Lazy Initialization ===")

local LazyLoader = {}
LazyLoader.__index = LazyLoader

function LazyLoader.new()
    return setmetatable({
        _cache = {},
        _initializers = {},
    }, LazyLoader)
end

function LazyLoader:register(key, initializer)
    self._initializers[key] = initializer
end

function LazyLoader:get(key)
    if self._cache[key] == nil then
        local init = self._initializers[key]
        if not init then
            error("No initializer registered for: " .. key)
        end
        print(string.format("[Lazy] Initializing '%s'...", key))
        self._cache[key] = init()
    end
    return self._cache[key]
end

function LazyLoader:isLoaded(key)
    return self._cache[key] ~= nil
end

-- ทดสอบ
local services = LazyLoader.new()

services:register("database", function()
    -- Expensive operation: connect to database
    return {
        host = "localhost",
        connected = true,
        query = function(self, sql) 
            return string.format("Results for: %s", sql) 
        end
    }
end)

services:register("emailService", function()
    -- Expensive operation: initialize email service
    return {
        send = function(self, to, subject, body)
            print(string.format("Email to %s: [%s] %s", to, subject, body))
        end
    }
end)

services:register("reportGenerator", function()
    -- Very expensive: load templates, fonts, etc.
    return {
        generate = function(self, data)
            return string.format("Report: %d records", #data)
        end
    }
end)

print("Services created (but not initialized)")
print("Is database loaded: " .. tostring(services:isLoaded("database")))

-- Access triggers initialization
local db = services:get("database")
print("Is database loaded: " .. tostring(services:isLoaded("database")))
print(db:query("SELECT * FROM users"))

local email = services:get("emailService")
email:send("user@example.com", "Welcome", "Hello!")

-- Second access - no re-initialization
local db2 = services:get("database")
print("Same database instance: " .. tostring(db == db2))
```

## 37.8 Registry / Service Locator Pattern

```lua
-- ตัวอย่างที่ 18: Service Locator
print("=== Service Locator Pattern ===")

local ServiceLocator = (function()
    local _services = {}
    local _factories = {}
    
    return {
        register = function(name, service)
            _services[name] = service
            print(string.format("[ServiceLocator] Registered: %s", name))
        end,
        
        registerFactory = function(name, factory)
            _factories[name] = factory
            print(string.format("[ServiceLocator] Registered factory: %s", name))
        end,
        
        get = function(name)
            if _services[name] then
                return _services[name]
            end
            if _factories[name] then
                local service = _factories[name]()
                _services[name] = service  -- cache it
                return service
            end
            error("Service not found: " .. name)
        end,
        
        has = function(name)
            return _services[name] ~= nil or _factories[name] ~= nil
        end,
        
        unregister = function(name)
            _services[name] = nil
            _factories[name] = nil
        end,
        
        reset = function()
            _services = {}
            _factories = {}
        end
    }
end)()

-- Register services
ServiceLocator.register("config", {
    get = function(self, key)
        local defaults = {
            debug = false,
            version = "1.0.0",
            maxConnections = 100
        }
        return defaults[key]
    end
})

ServiceLocator.registerFactory("logger", function()
    return {
        log = function(self, msg) print("[LOG] " .. msg) end
    }
end)

ServiceLocator.registerFactory("cache", function()
    local data = {}
    return {
        set = function(self, key, val) data[key] = val end,
        get = function(self, key) return data[key] end,
        del = function(self, key) data[key] = nil end
    }
end)

-- Use services
local config = ServiceLocator.get("config")
local logger = ServiceLocator.get("logger")
local cache = ServiceLocator.get("cache")

logger:log("Application starting")
logger:log("Version: " .. config:get("version"))

cache:set("user:1", {name = "Alice", age = 30})
local user = cache:get("user:1")
logger:log(string.format("Cached user: %s", user.name))
```

## 37.9 Real-world Example: Database Connection Pool

```lua
-- ตัวอย่างที่ 19: สมบูรณ์ Database Connection Pool
print("=== Database Connection Pool ===")

local ConnectionPool = {}
ConnectionPool.__index = ConnectionPool

function ConnectionPool.new(options)
    options = options or {}
    local pool = setmetatable({
        host     = options.host or "localhost",
        port     = options.port or 5432,
        database = options.database or "default",
        minSize  = options.minSize or 2,
        maxSize  = options.maxSize or 10,
        timeout  = options.timeout or 5000,
        
        _available = {},
        _inUse     = {},
        _waiting   = {},
        _idCounter = 0,
    }, ConnectionPool)
    
    -- Initialize min connections
    for i = 1, pool.minSize do
        pool:_createConnection()
    end
    
    return pool
end

function ConnectionPool:_createConnection()
    self._idCounter = self._idCounter + 1
    local id = self._idCounter
    local pool = self  -- capture for closure
    
    local conn = {
        id = id,
        pool = pool,
        queryCount = 0,
        created = os.time(),
    }
    
    function conn:execute(sql, params)
        self.queryCount = self.queryCount + 1
        print(string.format("[Conn#%d] Execute(%d): %s", self.id, self.queryCount, sql))
        return {affected = 1}
    end
    
    function conn:query(sql, params)
        self.queryCount = self.queryCount + 1
        print(string.format("[Conn#%d] Query(%d): %s", self.id, self.queryCount, sql))
        return {rows = {}, count = 0}
    end
    
    function conn:release()
        self.pool:release(self)
    end
    
    function conn:age()
        return os.time() - self.created
    end
    
    print(string.format("[Pool] Created connection #%d", id))
    table.insert(self._available, conn)
    return conn
end

function ConnectionPool:acquire()
    if #self._available > 0 then
        local conn = table.remove(self._available, 1)
        self._inUse[conn.id] = conn
        print(string.format("[Pool] Acquired connection #%d (%d available)", 
            conn.id, #self._available))
        return conn
    end
    
    local totalConns = self:_countTotal()
    if totalConns < self.maxSize then
        local conn = self:_createConnection()
        table.remove(self._available, 1)  -- remove from available
        self._inUse[conn.id] = conn
        return conn
    end
    
    error("Connection pool exhausted")
end

function ConnectionPool:release(conn)
    if self._inUse[conn.id] then
        self._inUse[conn.id] = nil
        table.insert(self._available, conn)
        print(string.format("[Pool] Released connection #%d (%d available)", 
            conn.id, #self._available))
    end
end

function ConnectionPool:_countTotal()
    local inUseCount = 0
    for _ in pairs(self._inUse) do inUseCount = inUseCount + 1 end
    return #self._available + inUseCount
end

function ConnectionPool:stats()
    local inUseCount = 0
    for _ in pairs(self._inUse) do inUseCount = inUseCount + 1 end
    return {
        total     = self:_countTotal(),
        available = #self._available,
        inUse     = inUseCount,
        maxSize   = self.maxSize,
    }
end

function ConnectionPool:withConnection(callback)
    local conn = self:acquire()
    local ok, err = pcall(callback, conn)
    conn:release()
    if not ok then error(err) end
end

-- ทดสอบ
local pool = ConnectionPool.new({
    host = "db.example.com",
    minSize = 2,
    maxSize = 5
})

print("\n--- Pool Stats ---")
local stats = pool:stats()
print(string.format("Total: %d, Available: %d, In Use: %d",
    stats.total, stats.available, stats.inUse))

-- ใช้งาน connections
pool:withConnection(function(conn)
    conn:query("SELECT * FROM users WHERE id = 1")
    conn:execute("UPDATE users SET last_login = NOW() WHERE id = 1")
end)

local conn1 = pool:acquire()
local conn2 = pool:acquire()
conn1:query("SELECT * FROM products")
conn2:execute("INSERT INTO orders VALUES (...)")

pool:release(conn1)
pool:release(conn2)

stats = pool:stats()
print(string.format("\nFinal Stats: Total: %d, Available: %d",
    stats.total, stats.available))
```

## 37.10 ตัวอย่าง Multiton (Keyed Singleton)

```lua
-- ตัวอย่างที่ 20: Multiton Pattern
print("=== Multiton Pattern ===")

local Multiton = {}
Multiton.__index = Multiton

local _instances = {}

function Multiton.getInstance(key)
    if not _instances[key] then
        _instances[key] = setmetatable({
            key = key,
            data = {},
            createdAt = os.time()
        }, Multiton)
        print(string.format("[Multiton] Created instance for key: '%s'", key))
    end
    return _instances[key]
end

function Multiton:set(k, v)
    self.data[k] = v
end

function Multiton:get(k)
    return self.data[k]
end

function Multiton.getAll()
    return _instances
end

-- ทดสอบ - เหมาะสำหรับ per-tenant configurations
local tenant1 = Multiton.getInstance("tenant_A")
local tenant2 = Multiton.getInstance("tenant_B")
local tenant1Again = Multiton.getInstance("tenant_A")

tenant1:set("theme", "blue")
tenant2:set("theme", "red")

print(string.format("tenant_A theme: %s", tenant1Again:get("theme")))
print(string.format("tenant_B theme: %s", tenant2:get("theme")))
print(string.format("tenant1 == tenant1Again: %s", tostring(tenant1 == tenant1Again)))
```

## 37.11 Fluent Builder with Validation

```lua
-- ตัวอย่างที่ 21: Builder with Validation
print("=== Builder with Validation ===")

local UserBuilder = {}
UserBuilder.__index = UserBuilder

function UserBuilder.new()
    return setmetatable({
        _data = {},
        _errors = {},
    }, UserBuilder)
end

function UserBuilder:name(name)
    if type(name) ~= "string" or #name < 2 then
        table.insert(self._errors, "Name must be at least 2 characters")
    else
        self._data.name = name
    end
    return self
end

function UserBuilder:email(email)
    if not email:match("^[%w.]+@[%w.]+%.[%a]+$") then
        table.insert(self._errors, "Invalid email format: " .. email)
    else
        self._data.email = email
    end
    return self
end

function UserBuilder:age(age)
    if type(age) ~= "number" or age < 0 or age > 150 then
        table.insert(self._errors, "Age must be between 0 and 150")
    else
        self._data.age = age
    end
    return self
end

function UserBuilder:role(role)
    local validRoles = {admin=true, user=true, moderator=true, guest=true}
    if not validRoles[role] then
        table.insert(self._errors, "Invalid role: " .. role)
    else
        self._data.role = role
    end
    return self
end

function UserBuilder:build()
    if #self._errors > 0 then
        return nil, "Validation errors:\n  - " .. table.concat(self._errors, "\n  - ")
    end
    
    -- Set defaults
    if not self._data.role then self._data.role = "user" end
    if not self._data.age then self._data.age = 0 end
    
    return {
        id = math.random(10000, 99999),
        name = self._data.name,
        email = self._data.email,
        age = self._data.age,
        role = self._data.role,
        createdAt = os.date("%Y-%m-%d")
    }, nil
end

-- Valid user
local user, err = UserBuilder.new()
    :name("Alice Smith")
    :email("alice@example.com")
    :age(28)
    :role("admin")
    :build()

if user then
    print(string.format("Created user: %s (%s) - %s", user.name, user.email, user.role))
else
    print("Error: " .. err)
end

-- Invalid user
local badUser, badErr = UserBuilder.new()
    :name("X")
    :email("not-an-email")
    :age(200)
    :role("superuser")
    :build()

if badUser then
    print("User created")
else
    print("Errors: " .. badErr)
end
```

## 37.12 สรุป Creational Patterns

```lua
-- ตัวอย่างที่ 22: เปรียบเทียบ patterns
print("=== สรุป Creational Patterns ===")

local patternSummary = {
    {
        name = "Singleton",
        intent = "รับประกัน instance เดียว, global access",
        when = "Config, Logger, Connection Pool, Registry",
        lua = "local _instance; function getInstance() ... end"
    },
    {
        name = "Factory Method",
        intent = "สร้าง object โดยไม่ระบุ concrete class",
        when = "Logger types, UI widgets, file parsers",
        lua = "function create(type) return types[type].new() end"
    },
    {
        name = "Abstract Factory",
        intent = "สร้าง family ของ objects ที่เกี่ยวข้องกัน",
        when = "UI themes, Database backends, OS components",
        lua = "local Factory = {createA=..., createB=...}"
    },
    {
        name = "Builder",
        intent = "สร้าง complex objects แบบ step-by-step",
        when = "Query builders, Config, HTTP requests",
        lua = "Builder.new():setA():setB():build()"
    },
    {
        name = "Prototype",
        intent = "Clone objects จาก prototype",
        when = "Game entities, Document templates",
        lua = "function obj:clone() return deepCopy(self) end"
    },
    {
        name = "Object Pool",
        intent = "Reuse expensive objects",
        when = "DB connections, game bullets, threads",
        lua = "pool:acquire() ... pool:release(obj)"
    },
}

for _, p in ipairs(patternSummary) do
    print(string.format("\n[%s]", p.name))
    print(string.format("  Intent: %s", p.intent))
    print(string.format("  When:   %s", p.when))
    print(string.format("  Lua:    %s", p.lua))
end
```

---

## แบบฝึกหัดบทที่ 37

1. สร้าง `ConfigurationSingleton` ที่รองรับการ load จาก environment variables
2. เขียน `ShapeFactory` ที่สร้าง Circle, Rectangle, Triangle พร้อม area() method
3. Implement `QueryBuilder` สำหรับ SQL ที่รองรับ SELECT, WHERE, JOIN, ORDER BY
4. สร้าง `DocumentCloner` ที่ clone documents พร้อม deep copy ของ nested structures
5. เขียน `ThreadPool` (simulated) ที่จัดการ worker objects
