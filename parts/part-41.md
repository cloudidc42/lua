# บทที่ 41: Logging Systems

## บทนำ

Logging (การบันทึก log) คือหนึ่งในเครื่องมือที่สำคัญที่สุดสำหรับนักพัฒนาซอฟต์แวร์ ไม่ว่าจะเป็นการ debug โปรแกรม การติดตามพฤติกรรมของระบบใน production หรือการวิเคราะห์ปัญหาที่เกิดขึ้น Logging ที่ดีช่วยให้เราเข้าใจว่าโปรแกรมทำงานอย่างไร และเกิดข้อผิดพลาดที่ไหนเมื่อใด

ในบทนี้เราจะสร้างระบบ logging ที่สมบูรณ์ด้วย Lua ตั้งแต่พื้นฐานจนถึงระดับ production-ready

---

## 41.1 ทำไม Logging ถึงสำคัญ

### ตัวอย่างที่ 1: โปรแกรมที่ไม่มี logging

```lua
-- โปรแกรมที่ไม่มี logging - ยากต่อการ debug
local function processUser(userId)
    local user = database.find(userId)
    if not user then
        return nil, "user not found"
    end
    local result = calculateScore(user)
    saveResult(userId, result)
    return result
end

-- เมื่อเกิดข้อผิดพลาด เราไม่รู้ว่าเกิดที่ขั้นตอนไหน
local score, err = processUser(42)
if err then
    print("Error: " .. err)  -- ไม่รู้บริบทใดๆ
end
```

### ตัวอย่างที่ 2: โปรแกรมที่มี logging ที่ดี

```lua
-- โปรแกรมที่มี logging ที่ดี - ง่ายต่อการ debug
local log = require("logger")

local function processUser(userId)
    log.info("Processing user", {userId = userId})
    
    local user = database.find(userId)
    if not user then
        log.warn("User not found", {userId = userId})
        return nil, "user not found"
    end
    
    log.debug("User found", {userId = userId, username = user.name})
    
    local result = calculateScore(user)
    log.info("Score calculated", {userId = userId, score = result})
    
    local ok, saveErr = saveResult(userId, result)
    if not ok then
        log.error("Failed to save result", {
            userId = userId,
            score = result,
            error = saveErr
        })
        return nil, saveErr
    end
    
    log.info("User processed successfully", {userId = userId, score = result})
    return result
end
```

---

## 41.2 Log Levels

Log levels คือระดับความสำคัญของ log message แต่ละระดับช่วยให้เรากรองข้อมูลที่ต้องการได้

### ตัวอย่างที่ 3: นิยาม Log Levels

```lua
-- log_levels.lua
local LogLevel = {}

-- ลำดับความสำคัญจากน้อยไปมาก
LogLevel.TRACE = 1   -- ข้อมูลละเอียดมาก สำหรับ trace การทำงาน
LogLevel.DEBUG = 2   -- ข้อมูล debug สำหรับนักพัฒนา
LogLevel.INFO  = 3   -- ข้อมูลปกติที่ต้องการบันทึก
LogLevel.WARN  = 4   -- คำเตือน ไม่ใช่ error แต่ควรสังเกต
LogLevel.ERROR = 5   -- ข้อผิดพลาดที่เกิดขึ้น
LogLevel.FATAL = 6   -- ข้อผิดพลาดร้ายแรง โปรแกรมอาจหยุดทำงาน

-- แปลงตัวเลขเป็นชื่อ
LogLevel.names = {
    [1] = "TRACE",
    [2] = "DEBUG", 
    [3] = "INFO",
    [4] = "WARN",
    [5] = "ERROR",
    [6] = "FATAL"
}

-- แปลงชื่อเป็นตัวเลข
LogLevel.values = {
    TRACE = 1,
    DEBUG = 2,
    INFO  = 3,
    WARN  = 4,
    ERROR = 5,
    FATAL = 6
}

function LogLevel.getName(level)
    return LogLevel.names[level] or "UNKNOWN"
end

function LogLevel.getValue(name)
    return LogLevel.values[name:upper()] or LogLevel.INFO
end

return LogLevel
```

### ตัวอย่างที่ 4: การใช้ Log Levels

```lua
-- ทดสอบ log levels
local LogLevel = {
    TRACE = 1, DEBUG = 2, INFO = 3,
    WARN = 4, ERROR = 5, FATAL = 6,
    names = {"TRACE","DEBUG","INFO","WARN","ERROR","FATAL"}
}

local currentLevel = LogLevel.INFO

local function shouldLog(level)
    return level >= currentLevel
end

local function log(level, message)
    if shouldLog(level) then
        print(string.format("[%s] %s", LogLevel.names[level], message))
    end
end

-- ทดสอบ
currentLevel = LogLevel.DEBUG

log(LogLevel.TRACE, "This won't show at DEBUG level")
log(LogLevel.DEBUG, "Debug message - จะแสดง")
log(LogLevel.INFO,  "Info message - จะแสดง")
log(LogLevel.WARN,  "Warning message - จะแสดง")
log(LogLevel.ERROR, "Error message - จะแสดง")
log(LogLevel.FATAL, "Fatal message - จะแสดง")
```

---

## 41.3 สร้าง Logger พื้นฐาน

### ตัวอย่างที่ 5: Logger พื้นฐาน

```lua
-- basic_logger.lua
local Logger = {}
Logger.__index = Logger

local LEVELS = {
    TRACE = 1, DEBUG = 2, INFO = 3,
    WARN = 4, ERROR = 5, FATAL = 6
}

local LEVEL_NAMES = {"TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"}

function Logger.new(name, level)
    local self = setmetatable({}, Logger)
    self.name = name or "default"
    self.level = level or LEVELS.INFO
    self.handlers = {}
    return self
end

function Logger:setLevel(level)
    if type(level) == "string" then
        self.level = LEVELS[level:upper()] or LEVELS.INFO
    else
        self.level = level
    end
end

function Logger:addHandler(handler)
    table.insert(self.handlers, handler)
end

function Logger:_log(level, message, data)
    if level < self.level then return end
    
    local entry = {
        timestamp = os.time(),
        level = level,
        levelName = LEVEL_NAMES[level],
        name = self.name,
        message = message,
        data = data
    }
    
    for _, handler in ipairs(self.handlers) do
        handler(entry)
    end
end

function Logger:trace(msg, data) self:_log(LEVELS.TRACE, msg, data) end
function Logger:debug(msg, data) self:_log(LEVELS.DEBUG, msg, data) end
function Logger:info(msg, data)  self:_log(LEVELS.INFO,  msg, data) end
function Logger:warn(msg, data)  self:_log(LEVELS.WARN,  msg, data) end
function Logger:error(msg, data) self:_log(LEVELS.ERROR, msg, data) end
function Logger:fatal(msg, data) self:_log(LEVELS.FATAL, msg, data) end

-- Console handler พื้นฐาน
local function consoleHandler(entry)
    local timeStr = os.date("%Y-%m-%d %H:%M:%S", entry.timestamp)
    print(string.format("[%s] [%s] [%s] %s",
        timeStr, entry.levelName, entry.name, entry.message))
end

-- ทดสอบ
local log = Logger.new("MyApp", LEVELS.DEBUG)
log:addHandler(consoleHandler)

log:trace("นี่คือ trace")        -- ไม่แสดง (ต่ำกว่า DEBUG)
log:debug("Debug information")   -- แสดง
log:info("Application started")  -- แสดง
log:warn("Low memory warning")   -- แสดง
log:error("Connection failed")   -- แสดง
```

### ตัวอย่างที่ 6: Logger พร้อม Timestamp ที่ละเอียด

```lua
-- logger_with_precise_time.lua
local function getTimestamp()
    -- ใช้ os.clock() สำหรับ microsecond precision (approximate)
    local t = os.time()
    local date = os.date("*t", t)
    return string.format("%04d-%02d-%02d %02d:%02d:%02d",
        date.year, date.month, date.day,
        date.hour, date.min, date.sec)
end

local function formatEntry(entry)
    local parts = {
        string.format("[%s]", getTimestamp()),
        string.format("[%-5s]", entry.levelName),
        string.format("[%s]", entry.name),
        entry.message
    }
    return table.concat(parts, " ")
end

-- ทดสอบ timestamp format
local entry = {
    levelName = "INFO",
    name = "TestApp",
    message = "Server started on port 8080"
}

print(formatEntry(entry))
-- Output: [2024-01-15 10:30:45] [INFO ] [TestApp] Server started on port 8080
```

---

## 41.4 Structured Logging (JSON Logs)

Structured logging คือการบันทึก log ในรูปแบบ JSON เพื่อให้ง่ายต่อการค้นหาและวิเคราะห์

### ตัวอย่างที่ 7: JSON Serializer อย่างง่าย

```lua
-- simple_json.lua
local function jsonEscape(s)
    s = tostring(s)
    s = s:gsub('\\', '\\\\')
    s = s:gsub('"', '\\"')
    s = s:gsub('\n', '\\n')
    s = s:gsub('\r', '\\r')
    s = s:gsub('\t', '\\t')
    return s
end

local function toJSON(value, indent, currentIndent)
    indent = indent or 0
    currentIndent = currentIndent or 0
    local t = type(value)
    
    if t == "nil" then
        return "null"
    elseif t == "boolean" then
        return tostring(value)
    elseif t == "number" then
        if value ~= value then return "null" end  -- NaN
        if value == math.huge or value == -math.huge then return "null" end
        if math.floor(value) == value then
            return string.format("%d", value)
        end
        return string.format("%.10g", value)
    elseif t == "string" then
        return '"' .. jsonEscape(value) .. '"'
    elseif t == "table" then
        -- ตรวจสอบว่าเป็น array หรือ object
        local isArray = true
        local maxN = 0
        for k, _ in pairs(value) do
            if type(k) ~= "number" or k < 1 or math.floor(k) ~= k then
                isArray = false
                break
            end
            if k > maxN then maxN = k end
        end
        isArray = isArray and maxN == #value
        
        local parts = {}
        if isArray then
            for _, v in ipairs(value) do
                table.insert(parts, toJSON(v, indent, currentIndent))
            end
            return "[" .. table.concat(parts, ",") .. "]"
        else
            for k, v in pairs(value) do
                local key = '"' .. jsonEscape(tostring(k)) .. '"'
                table.insert(parts, key .. ":" .. toJSON(v, indent, currentIndent))
            end
            table.sort(parts)
            return "{" .. table.concat(parts, ",") .. "}"
        end
    else
        return '"[' .. t .. ']"'
    end
end

-- ทดสอบ
local data = {
    userId = 42,
    username = "john_doe",
    active = true,
    scores = {95, 87, 92},
    meta = {role = "admin", level = 5}
}

print(toJSON(data))
```

### ตัวอย่างที่ 8: Structured Logger

```lua
-- structured_logger.lua
local function toJSON(value)
    local t = type(value)
    if t == "nil" then return "null"
    elseif t == "boolean" then return tostring(value)
    elseif t == "number" then return tostring(value)
    elseif t == "string" then
        return '"' .. value:gsub('"', '\\"'):gsub('\n', '\\n') .. '"'
    elseif t == "table" then
        local parts = {}
        -- ตรวจสอบ array
        if #value > 0 then
            for _, v in ipairs(value) do
                table.insert(parts, toJSON(v))
            end
            return "[" .. table.concat(parts, ",") .. "]"
        else
            for k, v in pairs(value) do
                table.insert(parts, '"' .. tostring(k) .. '":' .. toJSON(v))
            end
            return "{" .. table.concat(parts, ",") .. "}"
        end
    else
        return '"' .. tostring(value) .. '"'
    end
end

local StructuredLogger = {}
StructuredLogger.__index = StructuredLogger

function StructuredLogger.new(service, version)
    local self = setmetatable({}, StructuredLogger)
    self.service = service
    self.version = version or "1.0.0"
    self.level = 3  -- INFO
    self.output = io.stdout
    return self
end

function StructuredLogger:_emit(level, levelName, message, fields)
    if level < self.level then return end
    
    local entry = {
        timestamp = os.date("%Y-%m-%dT%H:%M:%SZ"),
        level = levelName,
        service = self.service,
        version = self.version,
        message = message
    }
    
    -- merge fields
    if fields then
        for k, v in pairs(fields) do
            entry[k] = v
        end
    end
    
    self.output:write(toJSON(entry) .. "\n")
end

function StructuredLogger:info(msg, fields)  self:_emit(3, "INFO",  msg, fields) end
function StructuredLogger:warn(msg, fields)  self:_emit(4, "WARN",  msg, fields) end
function StructuredLogger:error(msg, fields) self:_emit(5, "ERROR", msg, fields) end
function StructuredLogger:debug(msg, fields) self:_emit(2, "DEBUG", msg, fields) end

-- ทดสอบ structured logger
local log = StructuredLogger.new("user-service", "2.1.0")

log:info("User logged in", {
    userId = 12345,
    ipAddress = "192.168.1.1",
    userAgent = "Mozilla/5.0"
})

log:warn("Rate limit approaching", {
    userId = 12345,
    requestCount = 95,
    limit = 100
})

log:error("Database connection failed", {
    host = "db.example.com",
    port = 5432,
    retryCount = 3
})
```

---

## 41.5 Log Rotation

Log rotation คือการหมุนเวียนไฟล์ log เพื่อไม่ให้ไฟล์ใหญ่เกินไป

### ตัวอย่างที่ 9: File Handler พื้นฐาน

```lua
-- file_handler.lua
local FileHandler = {}
FileHandler.__index = FileHandler

function FileHandler.new(filepath)
    local self = setmetatable({}, FileHandler)
    self.filepath = filepath
    self.file = io.open(filepath, "a")
    if not self.file then
        error("Cannot open log file: " .. filepath)
    end
    return self
end

function FileHandler:write(entry)
    local line = string.format("[%s] [%s] %s\n",
        os.date("%Y-%m-%d %H:%M:%S"),
        entry.levelName,
        entry.message)
    self.file:write(line)
    self.file:flush()
end

function FileHandler:close()
    if self.file then
        self.file:close()
        self.file = nil
    end
end

-- ทดสอบ
local handler = FileHandler.new("/tmp/test.log")
handler:write({levelName = "INFO", message = "Test log entry"})
handler:write({levelName = "ERROR", message = "Something went wrong"})
handler:close()

-- อ่านกลับมาตรวจสอบ
local f = io.open("/tmp/test.log", "r")
if f then
    print(f:read("*all"))
    f:close()
end
```

### ตัวอย่างที่ 10: Rotating File Handler

```lua
-- rotating_file_handler.lua
local RotatingFileHandler = {}
RotatingFileHandler.__index = RotatingFileHandler

function RotatingFileHandler.new(config)
    local self = setmetatable({}, RotatingFileHandler)
    self.basePath = config.path or "app.log"
    self.maxSize = config.maxSize or 1024 * 1024  -- 1MB default
    self.maxFiles = config.maxFiles or 5
    self.currentSize = 0
    self.file = nil
    self:_openFile()
    return self
end

function RotatingFileHandler:_openFile()
    if self.file then
        self.file:close()
    end
    self.file = io.open(self.basePath, "a")
    if not self.file then
        error("Cannot open log file: " .. self.basePath)
    end
    -- คำนวณขนาดปัจจุบัน
    local pos = self.file:seek("end")
    self.currentSize = pos or 0
end

function RotatingFileHandler:_rotate()
    if self.file then
        self.file:close()
        self.file = nil
    end
    
    -- ลบไฟล์เก่าสุด
    local oldestFile = self.basePath .. "." .. self.maxFiles
    os.remove(oldestFile)
    
    -- เลื่อนไฟล์ทั้งหมด
    for i = self.maxFiles - 1, 1, -1 do
        local src = self.basePath .. "." .. i
        local dst = self.basePath .. "." .. (i + 1)
        os.rename(src, dst)
    end
    
    -- เลื่อนไฟล์ปัจจุบัน
    os.rename(self.basePath, self.basePath .. ".1")
    
    -- เปิดไฟล์ใหม่
    self.file = io.open(self.basePath, "w")
    self.currentSize = 0
    
    print("[RotatingHandler] Log rotated: " .. self.basePath)
end

function RotatingFileHandler:write(message)
    local line = message .. "\n"
    local lineSize = #line
    
    -- ตรวจสอบว่าต้อง rotate หรือไม่
    if self.currentSize + lineSize > self.maxSize then
        self:_rotate()
    end
    
    self.file:write(line)
    self.file:flush()
    self.currentSize = self.currentSize + lineSize
end

function RotatingFileHandler:close()
    if self.file then
        self.file:close()
        self.file = nil
    end
end

-- ทดสอบ
local handler = RotatingFileHandler.new({
    path = "/tmp/rotating_test.log",
    maxSize = 500,  -- 500 bytes สำหรับ test
    maxFiles = 3
})

for i = 1, 20 do
    handler:write(string.format("[INFO] Log entry number %d - timestamp: %s",
        i, os.date("%Y-%m-%d %H:%M:%S")))
end

handler:close()
print("Rotation test completed")
```

### ตัวอย่างที่ 11: Time-Based Log Rotation

```lua
-- time_based_rotation.lua
local DailyRotatingHandler = {}
DailyRotatingHandler.__index = DailyRotatingHandler

function DailyRotatingHandler.new(config)
    local self = setmetatable({}, DailyRotatingHandler)
    self.basePath = config.path or "app.log"
    self.keepDays = config.keepDays or 7
    self.currentDate = nil
    self.file = nil
    self:_checkRotation()
    return self
end

function DailyRotatingHandler:_getDateString()
    return os.date("%Y-%m-%d")
end

function DailyRotatingHandler:_checkRotation()
    local today = self:_getDateString()
    if today ~= self.currentDate then
        if self.file then
            self.file:close()
            self.file = nil
        end
        
        -- สร้างชื่อไฟล์พร้อมวันที่
        local filename = self.basePath .. "." .. today
        self.file = io.open(filename, "a")
        self.currentDate = today
        
        -- ลบไฟล์เก่า (simulation)
        print("[DailyRotation] Opened new log file: " .. filename)
    end
end

function DailyRotatingHandler:write(message)
    self:_checkRotation()
    if self.file then
        self.file:write(message .. "\n")
        self.file:flush()
    end
end

-- ทดสอบ
local handler = DailyRotatingHandler.new({
    path = "/tmp/daily",
    keepDays = 7
})

handler:write("[INFO] Application started")
handler:write("[INFO] Processing request 1")
handler:write("[WARN] High memory usage detected")
handler:close = function(self)
    if self.file then self.file:close() end
end
handler:close()
```

---

## 41.6 Multiple Handlers

### ตัวอย่างที่ 12: Logger ที่รองรับ Multiple Handlers

```lua
-- multi_handler_logger.lua
local MultiLogger = {}
MultiLogger.__index = MultiLogger

local LEVELS = {TRACE=1, DEBUG=2, INFO=3, WARN=4, ERROR=5, FATAL=6}
local LEVEL_NAMES = {[1]="TRACE",[2]="DEBUG",[3]="INFO",[4]="WARN",[5]="ERROR",[6]="FATAL"}

function MultiLogger.new(name)
    local self = setmetatable({}, MultiLogger)
    self.name = name
    self.level = LEVELS.INFO
    self.handlers = {}
    return self
end

function MultiLogger:addHandler(name, handler)
    self.handlers[name] = handler
    return self
end

function MultiLogger:removeHandler(name)
    self.handlers[name] = nil
    return self
end

function MultiLogger:_log(level, msg, fields)
    if level < self.level then return end
    
    local entry = {
        time = os.time(),
        timeStr = os.date("%Y-%m-%d %H:%M:%S"),
        level = level,
        levelName = LEVEL_NAMES[level],
        logger = self.name,
        message = msg,
        fields = fields or {}
    }
    
    for handlerName, handler in pairs(self.handlers) do
        local ok, err = pcall(handler, entry)
        if not ok then
            io.stderr:write(string.format(
                "[LOGGER ERROR] Handler '%s' failed: %s\n", handlerName, err))
        end
    end
end

-- สร้าง handlers
local function makeConsoleHandler(colored)
    local colors = {
        TRACE = "\27[37m",   -- white
        DEBUG = "\27[36m",   -- cyan
        INFO  = "\27[32m",   -- green
        WARN  = "\27[33m",   -- yellow
        ERROR = "\27[31m",   -- red
        FATAL = "\27[35m",   -- magenta
        RESET = "\27[0m"
    }
    
    return function(entry)
        local color = colored and colors[entry.levelName] or ""
        local reset = colored and colors.RESET or ""
        print(string.format("%s[%s] [%s] [%s] %s%s",
            color, entry.timeStr, entry.levelName, entry.logger,
            entry.message, reset))
    end
end

local function makeFileHandler(filepath)
    local file = io.open(filepath, "a")
    return function(entry)
        if file then
            file:write(string.format("[%s] [%s] [%s] %s\n",
                entry.timeStr, entry.levelName, entry.logger, entry.message))
            file:flush()
        end
    end
end

-- ทดสอบ
local log = MultiLogger.new("Application")
log:addHandler("console", makeConsoleHandler(false))
log:addHandler("file", makeFileHandler("/tmp/multi_test.log"))

log:info("Application initialized")
log:warn("Configuration file not found, using defaults")
log:error("Failed to connect to cache server")

-- ลบ console handler (เฉพาะส่ง log ไปไฟล์)
log:removeHandler("console")
log:info("This goes only to file")

print("\nMulti-handler test completed")
```

### ตัวอย่างที่ 13: Remote/HTTP Handler

```lua
-- remote_handler.lua
-- Handler ที่ส่ง log ไปยัง remote endpoint (simulation)

local RemoteHandler = {}
RemoteHandler.__index = RemoteHandler

function RemoteHandler.new(config)
    local self = setmetatable({}, RemoteHandler)
    self.url = config.url
    self.apiKey = config.apiKey
    self.batchSize = config.batchSize or 10
    self.flushInterval = config.flushInterval or 5
    self.buffer = {}
    self.lastFlush = os.time()
    return self
end

function RemoteHandler:_flush()
    if #self.buffer == 0 then return end
    
    -- ใน production จริงจะใช้ HTTP client เช่น lua-http
    -- นี่คือ simulation
    print(string.format("[RemoteHandler] Sending %d log entries to %s",
        #self.buffer, self.url))
    
    for _, entry in ipairs(self.buffer) do
        print(string.format("  -> [%s] %s", entry.levelName, entry.message))
    end
    
    self.buffer = {}
    self.lastFlush = os.time()
end

function RemoteHandler:write(entry)
    table.insert(self.buffer, entry)
    
    -- flush เมื่อ buffer เต็มหรือถึงเวลา
    if #self.buffer >= self.batchSize or 
       (os.time() - self.lastFlush) >= self.flushInterval then
        self:_flush()
    end
end

function RemoteHandler:close()
    self:_flush()  -- flush ที่เหลือก่อนปิด
end

-- ทดสอบ
local remote = RemoteHandler.new({
    url = "https://logs.example.com/ingest",
    apiKey = "secret-key-123",
    batchSize = 3,
    flushInterval = 10
})

-- เพิ่ม log entries
for i = 1, 7 do
    remote:write({
        levelName = "INFO",
        message = string.format("Request processed: %d", i),
        timestamp = os.time()
    })
end

remote:close()
```

---

## 41.7 Log Formatting

### ตัวอย่างที่ 14: Custom Formatters

```lua
-- formatters.lua

-- Formatter 1: Simple text
local function textFormatter(entry)
    return string.format("[%s] [%s] %s",
        entry.timeStr or os.date("%H:%M:%S"),
        entry.levelName,
        entry.message)
end

-- Formatter 2: Detailed format
local function detailedFormatter(entry)
    local parts = {
        os.date("%Y-%m-%d %H:%M:%S"),
        string.format("%-5s", entry.levelName),
        entry.logger or "app",
        entry.message
    }
    
    -- เพิ่ม fields ถ้ามี
    if entry.fields and next(entry.fields) then
        local fieldParts = {}
        for k, v in pairs(entry.fields) do
            table.insert(fieldParts, k .. "=" .. tostring(v))
        end
        table.sort(fieldParts)
        table.insert(parts, "{" .. table.concat(fieldParts, " ") .. "}")
    end
    
    return table.concat(parts, " | ")
end

-- Formatter 3: JSON format
local function jsonFormatter(entry)
    local obj = {
        string.format('"ts":"%s"', os.date("%Y-%m-%dT%H:%M:%SZ")),
        string.format('"level":"%s"', entry.levelName),
        string.format('"msg":"%s"', (entry.message or ""):gsub('"', '\\"'))
    }
    
    if entry.logger then
        table.insert(obj, string.format('"logger":"%s"', entry.logger))
    end
    
    if entry.fields then
        for k, v in pairs(entry.fields) do
            if type(v) == "number" then
                table.insert(obj, string.format('"%s":%s', k, v))
            elseif type(v) == "boolean" then
                table.insert(obj, string.format('"%s":%s', k, tostring(v)))
            else
                table.insert(obj, string.format('"%s":"%s"', k, tostring(v)))
            end
        end
    end
    
    return "{" .. table.concat(obj, ",") .. "}"
end

-- ทดสอบ formatters
local entry = {
    timeStr = "2024-01-15 10:30:45",
    levelName = "WARN",
    logger = "db-pool",
    message = "Connection pool exhausted",
    fields = {
        poolSize = 10,
        activeConns = 10,
        waitingRequests = 5
    }
}

print("=== Text Format ===")
print(textFormatter(entry))
print()
print("=== Detailed Format ===")
print(detailedFormatter(entry))
print()
print("=== JSON Format ===")
print(jsonFormatter(entry))
```

### ตัวอย่างที่ 15: Colored Console Output

```lua
-- colored_logger.lua
local COLORS = {
    -- Foreground colors
    BLACK   = "\27[30m",
    RED     = "\27[31m",
    GREEN   = "\27[32m",
    YELLOW  = "\27[33m",
    BLUE    = "\27[34m",
    MAGENTA = "\27[35m",
    CYAN    = "\27[36m",
    WHITE   = "\27[37m",
    -- Bright
    BRIGHT_RED    = "\27[91m",
    BRIGHT_GREEN  = "\27[92m",
    BRIGHT_YELLOW = "\27[93m",
    -- Reset
    RESET = "\27[0m",
    BOLD  = "\27[1m"
}

local LEVEL_COLORS = {
    TRACE = COLORS.WHITE,
    DEBUG = COLORS.CYAN,
    INFO  = COLORS.BRIGHT_GREEN,
    WARN  = COLORS.BRIGHT_YELLOW,
    ERROR = COLORS.BRIGHT_RED,
    FATAL = COLORS.BOLD .. COLORS.MAGENTA
}

local function coloredFormat(entry)
    local levelColor = LEVEL_COLORS[entry.levelName] or COLORS.WHITE
    local timeColor = COLORS.BLUE
    local reset = COLORS.RESET
    
    return string.format("%s%s%s %s%-5s%s %s%s%s %s",
        timeColor, entry.timeStr or os.date("%H:%M:%S"), reset,
        levelColor, entry.levelName, reset,
        COLORS.CYAN, entry.logger or "app", reset,
        entry.message)
end

-- Detect if terminal supports colors
local function supportsColor()
    local term = os.getenv("TERM")
    local colorterm = os.getenv("COLORTERM")
    return term ~= nil or colorterm ~= nil
end

-- ทดสอบ
local levels = {"TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"}
local messages = {
    "Detailed trace information",
    "Variable x = 42",
    "Server listening on :8080",
    "High memory usage: 85%",
    "Failed to parse JSON",
    "Unhandled exception - shutting down"
}

for i, level in ipairs(levels) do
    local entry = {
        timeStr = os.date("%H:%M:%S"),
        levelName = level,
        logger = "demo",
        message = messages[i]
    }
    print(coloredFormat(entry))
end
```

---

## 41.8 Correlation IDs และ Context Propagation

Correlation ID ช่วยให้เราติดตาม request เดียวผ่าน services ต่างๆ

### ตัวอย่างที่ 16: Generating Correlation IDs

```lua
-- correlation_id.lua

-- สร้าง pseudo-random ID
local function generateId()
    local chars = "abcdefghijklmnopqrstuvwxyz0123456789"
    local id = {}
    math.randomseed(os.time() + math.floor(os.clock() * 1000000))
    for i = 1, 16 do
        local rand = math.random(1, #chars)
        table.insert(id, chars:sub(rand, rand))
    end
    return table.concat(id)
end

-- UUID v4 style (simplified)
local function generateUUID()
    local template = "xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx"
    return template:gsub("[xy]", function(c)
        local v = (c == "x") and math.random(0, 15) or math.random(8, 11)
        return string.format("%x", v)
    end)
end

math.randomseed(os.time())
print("Simple ID: " .. generateId())
print("UUID: " .. generateUUID())
print("UUID: " .. generateUUID())
print("UUID: " .. generateUUID())
```

### ตัวอย่างที่ 17: Context-Aware Logger

```lua
-- context_logger.lua
local ContextLogger = {}
ContextLogger.__index = ContextLogger

function ContextLogger.new(baseLogger)
    local self = setmetatable({}, ContextLogger)
    self.base = baseLogger
    self.context = {}
    return self
end

function ContextLogger:withContext(fields)
    local child = ContextLogger.new(self.base)
    -- Copy parent context
    for k, v in pairs(self.context) do
        child.context[k] = v
    end
    -- Add new fields
    for k, v in pairs(fields) do
        child.context[k] = v
    end
    return child
end

function ContextLogger:_mergeContext(fields)
    local merged = {}
    for k, v in pairs(self.context) do
        merged[k] = v
    end
    if fields then
        for k, v in pairs(fields) do
            merged[k] = v
        end
    end
    return merged
end

function ContextLogger:info(msg, fields)
    self.base:info(msg, self:_mergeContext(fields))
end

function ContextLogger:warn(msg, fields)
    self.base:warn(msg, self:_mergeContext(fields))
end

function ContextLogger:error(msg, fields)
    self.base:error(msg, self:_mergeContext(fields))
end

-- Simulation ของ base logger
local function createBaseLogger(name)
    return {
        name = name,
        info = function(self, msg, fields)
            local fieldStr = ""
            if fields and next(fields) then
                local parts = {}
                for k, v in pairs(fields) do
                    table.insert(parts, k .. "=" .. tostring(v))
                end
                fieldStr = " {" .. table.concat(parts, ", ") .. "}"
            end
            print(string.format("[INFO] [%s] %s%s", self.name, msg, fieldStr))
        end,
        warn = function(self, msg, fields)
            local fieldStr = ""
            if fields and next(fields) then
                local parts = {}
                for k, v in pairs(fields) do
                    table.insert(parts, k .. "=" .. tostring(v))
                end
                fieldStr = " {" .. table.concat(parts, ", ") .. "}"
            end
            print(string.format("[WARN] [%s] %s%s", self.name, msg, fieldStr))
        end,
        error = function(self, msg, fields)
            local fieldStr = ""
            if fields and next(fields) then
                local parts = {}
                for k, v in pairs(fields) do
                    table.insert(parts, k .. "=" .. tostring(v))
                end
                fieldStr = " {" .. table.concat(parts, ", ") .. "}"
            end
            print(string.format("[ERROR] [%s] %s%s", self.name, msg, fieldStr))
        end
    }
end

-- ทดสอบ
local base = createBaseLogger("api-server")
local log = ContextLogger.new(base)

-- Logger สำหรับ request นี้
local reqLog = log:withContext({
    requestId = "req-abc123",
    method = "POST",
    path = "/api/users"
})

reqLog:info("Request received")

-- Logger สำหรับ user-specific context
local userLog = reqLog:withContext({
    userId = 42,
    username = "john"
})

userLog:info("Processing user request")
userLog:warn("User has exceeded rate limit")

-- Error มี context ครบ
userLog:error("Failed to update user", {
    field = "email",
    reason = "already exists"
})
```

### ตัวอย่างที่ 18: Request-Scoped Logging

```lua
-- request_scoped_logger.lua

-- สร้าง correlation ID
local function newCorrelationId()
    math.randomseed(os.time())
    return string.format("%08x-%04x-%04x",
        math.random(0, 0xFFFFFFFF),
        math.random(0, 0xFFFF),
        math.random(0, 0xFFFF))
end

-- Request context
local function handleRequest(method, path, correlationId)
    correlationId = correlationId or newCorrelationId()
    
    local function log(level, msg, extra)
        local fields = {
            correlationId = correlationId,
            method = method,
            path = path
        }
        if extra then
            for k, v in pairs(extra) do fields[k] = v end
        end
        
        local fieldParts = {}
        for k, v in pairs(fields) do
            table.insert(fieldParts, string.format("%s=%q", k, tostring(v)))
        end
        
        print(string.format("[%s] %s | %s", level, msg, table.concat(fieldParts, " ")))
    end
    
    log("INFO", "Request started")
    
    -- Simulate processing
    os.execute("sleep 0")  -- no-op
    
    log("DEBUG", "Validating input", {bodySize = 256})
    log("INFO", "Calling database")
    log("INFO", "Request completed", {statusCode = 200, durationMs = 45})
    
    return correlationId
end

-- สร้าง requests ใหม่
print("=== Request 1 ===")
handleRequest("GET", "/api/products")

print("\n=== Request 2 (with existing correlation ID) ===")
handleRequest("POST", "/api/orders", "upstream-correlation-xyz")
```

---

## 41.9 Performance Impact of Logging

### ตัวอย่างที่ 19: Lazy Evaluation สำหรับ Log Messages

```lua
-- lazy_logging.lua

-- BAD: การสร้าง string ก่อนตรวจสอบ level (เสียเวลาแม้ไม่ได้ log)
local function badLogging(level, data)
    local currentLevel = 3  -- INFO
    -- string concatenation เกิดขึ้นก่อนเสมอ แม้ level จะต่ำเกินไป
    local msg = "Processing item " .. data.id .. " with value " .. 
                tostring(data.value) .. " and metadata " .. 
                table.concat(data.tags or {}, ", ")
    if level >= currentLevel then
        print(msg)
    end
end

-- GOOD: ใช้ function เพื่อ defer evaluation
local function goodLogging(level, msgFn)
    local currentLevel = 3  -- INFO
    if level >= currentLevel then
        print(msgFn())  -- สร้าง string เฉพาะเมื่อจำเป็น
    end
end

-- GOOD: ตรวจสอบ level ก่อน
local Logger = {}
function Logger:isDebugEnabled() return self.level <= 2 end
function Logger:isTraceEnabled() return self.level <= 1 end

-- Benchmark
local data = {id = 42, value = 3.14, tags = {"lua", "logging", "perf"}}

local start = os.clock()
for i = 1, 100000 do
    -- Bad: always builds string
    local _ = "Processing item " .. data.id
end
local badTime = os.clock() - start

start = os.clock()
local dummyLevel = 2  -- DEBUG
local currentLevel = 3  -- INFO
for i = 1, 100000 do
    -- Good: skips string building
    if dummyLevel >= currentLevel then
        local _ = "Processing item " .. data.id
    end
end
local goodTime = os.clock() - start

print(string.format("Without level check: %.4f sec", badTime))
print(string.format("With level check:    %.4f sec", goodTime))
print(string.format("Speedup: %.1fx", badTime / math.max(goodTime, 0.0001)))
```

### ตัวอย่างที่ 20: Buffered Logging สำหรับ Performance

```lua
-- buffered_logger.lua
local BufferedLogger = {}
BufferedLogger.__index = BufferedLogger

function BufferedLogger.new(config)
    local self = setmetatable({}, BufferedLogger)
    self.buffer = {}
    self.bufferSize = config.bufferSize or 100
    self.flushInterval = config.flushInterval or 1.0
    self.lastFlush = os.clock()
    self.output = config.output or io.stdout
    self.dropped = 0
    self.maxBuffer = config.maxBuffer or 1000
    return self
end

function BufferedLogger:write(entry)
    -- ทิ้ง entries ที่เกิน maxBuffer (backpressure)
    if #self.buffer >= self.maxBuffer then
        self.dropped = self.dropped + 1
        return
    end
    
    table.insert(self.buffer, entry)
    
    -- flush เมื่อ buffer เต็มหรือถึงเวลา
    if #self.buffer >= self.bufferSize or
       (os.clock() - self.lastFlush) >= self.flushInterval then
        self:flush()
    end
end

function BufferedLogger:flush()
    if #self.buffer == 0 then return end
    
    local lines = {}
    for _, entry in ipairs(self.buffer) do
        table.insert(lines, string.format("[%s] %s", entry.level, entry.message))
    end
    
    self.output:write(table.concat(lines, "\n") .. "\n")
    self.output:flush()
    
    if self.dropped > 0 then
        self.output:write(string.format("[WARN] Dropped %d log entries\n", self.dropped))
        self.dropped = 0
    end
    
    self.buffer = {}
    self.lastFlush = os.clock()
end

function BufferedLogger:close()
    self:flush()
end

-- ทดสอบ performance
local logger = BufferedLogger.new({
    bufferSize = 50,
    flushInterval = 0.1,
    output = io.open("/dev/null", "w") or io.stdout
})

local start = os.clock()
for i = 1, 10000 do
    logger:write({level = "INFO", message = "Test message " .. i})
end
logger:close()
local elapsed = os.clock() - start

print(string.format("Logged 10,000 entries in %.4f seconds", elapsed))
print(string.format("Rate: %.0f entries/sec", 10000 / math.max(elapsed, 0.001)))
```

---

## 41.10 Async Logging

### ตัวอย่างที่ 21: Async Logger ด้วย Coroutines

```lua
-- async_logger.lua
-- ใช้ coroutines เพื่อจำลอง async logging

local AsyncLogger = {}
AsyncLogger.__index = AsyncLogger

function AsyncLogger.new(handler)
    local self = setmetatable({}, AsyncLogger)
    self.queue = {}
    self.handler = handler
    self.running = false
    
    -- สร้าง worker coroutine
    self.worker = coroutine.create(function()
        while true do
            if #self.queue > 0 then
                local entry = table.remove(self.queue, 1)
                self.handler(entry)
            end
            coroutine.yield()
        end
    end)
    
    return self
end

function AsyncLogger:log(level, message, fields)
    table.insert(self.queue, {
        time = os.date("%H:%M:%S"),
        level = level,
        message = message,
        fields = fields
    })
end

-- ให้ worker ทำงาน
function AsyncLogger:tick()
    if coroutine.status(self.worker) ~= "dead" then
        coroutine.resume(self.worker)
    end
end

function AsyncLogger:flush()
    while #self.queue > 0 do
        self:tick()
    end
end

-- ทดสอบ
local entries = {}
local asyncLog = AsyncLogger.new(function(entry)
    table.insert(entries, string.format("[%s] [%s] %s",
        entry.time, entry.level, entry.message))
end)

-- Log หลาย entries
asyncLog:log("INFO", "Request 1 received")
asyncLog:log("DEBUG", "Processing request 1")
asyncLog:log("INFO", "Request 2 received")
asyncLog:log("ERROR", "Request 1 failed")

-- จำลอง event loop
print("Entries in queue: " .. #asyncLog.queue)

-- Process
asyncLog:flush()

print("After flush:")
for _, line in ipairs(entries) do
    print(line)
end
```

---

## 41.11 Log Aggregation

### ตัวอย่างที่ 22: Log Aggregator

```lua
-- log_aggregator.lua
-- รวม logs จากหลายแหล่งเข้าด้วยกัน

local LogAggregator = {}
LogAggregator.__index = LogAggregator

function LogAggregator.new()
    local self = setmetatable({}, LogAggregator)
    self.logs = {}
    self.sources = {}
    self.stats = {total = 0, byLevel = {}, bySource = {}}
    return self
end

function LogAggregator:addSource(name)
    self.sources[name] = true
    self.stats.bySource[name] = 0
end

function LogAggregator:ingest(source, level, message, timestamp)
    if not self.sources[source] then
        self:addSource(source)
    end
    
    local entry = {
        source = source,
        level = level,
        message = message,
        timestamp = timestamp or os.time()
    }
    
    table.insert(self.logs, entry)
    self.stats.total = self.stats.total + 1
    self.stats.byLevel[level] = (self.stats.byLevel[level] or 0) + 1
    self.stats.bySource[source] = (self.stats.bySource[source] or 0) + 1
    
    return entry
end

function LogAggregator:query(filter)
    local results = {}
    for _, entry in ipairs(self.logs) do
        local match = true
        if filter.level and entry.level ~= filter.level then match = false end
        if filter.source and entry.source ~= filter.source then match = false end
        if filter.search and not entry.message:find(filter.search) then match = false end
        if match then
            table.insert(results, entry)
        end
    end
    return results
end

function LogAggregator:getStats()
    return self.stats
end

-- ทดสอบ
local agg = LogAggregator.new()

-- จำลอง logs จากหลาย services
local services = {"api-server", "db-pool", "cache", "worker"}
local levels = {"INFO", "DEBUG", "WARN", "ERROR"}
local messages = {
    "Request processed successfully",
    "Cache hit for key: user:42",
    "Database query took 250ms",
    "Worker job completed",
    "Connection timeout",
    "High memory usage",
    "Request failed: 500"
}

math.randomseed(42)
for i = 1, 30 do
    local service = services[math.random(#services)]
    local level = levels[math.random(#levels)]
    local msg = messages[math.random(#messages)]
    agg:ingest(service, level, msg)
end

-- แสดงสถิติ
local stats = agg:getStats()
print("=== Log Statistics ===")
print("Total logs: " .. stats.total)
print("\nBy Level:")
for level, count in pairs(stats.byLevel) do
    print(string.format("  %-6s: %d", level, count))
end
print("\nBy Source:")
for source, count in pairs(stats.bySource) do
    print(string.format("  %-12s: %d", source, count))
end

-- Query
print("\n=== ERROR Logs ===")
local errors = agg:query({level = "ERROR"})
for _, e in ipairs(errors) do
    print(string.format("[%s] %s", e.source, e.message))
end

print("\n=== Logs from api-server ===")
local apiLogs = agg:query({source = "api-server"})
for _, e in ipairs(apiLogs) do
    print(string.format("[%s] %s", e.level, e.message))
end
```

---

## 41.12 Integration กับ Syslog

### ตัวอย่างที่ 23: Syslog Format

```lua
-- syslog_formatter.lua
-- RFC 5424 Syslog format

local SYSLOG_FACILITY = {
    KERN   = 0,
    USER   = 1,
    MAIL   = 2,
    DAEMON = 3,
    AUTH   = 4,
    LOCAL0 = 16,
    LOCAL1 = 17,
    LOCAL7 = 23
}

local SYSLOG_SEVERITY = {
    EMERGENCY = 0,  -- System is unusable
    ALERT     = 1,  -- Action must be taken immediately
    CRITICAL  = 2,  -- Critical conditions
    ERROR     = 3,  -- Error conditions
    WARNING   = 4,  -- Warning conditions
    NOTICE    = 5,  -- Normal but significant
    INFO      = 6,  -- Informational
    DEBUG     = 7   -- Debug-level messages
}

-- Map Lua log levels to syslog severity
local function luaLevelToSyslog(level)
    local mapping = {
        TRACE = SYSLOG_SEVERITY.DEBUG,
        DEBUG = SYSLOG_SEVERITY.DEBUG,
        INFO  = SYSLOG_SEVERITY.INFO,
        WARN  = SYSLOG_SEVERITY.WARNING,
        ERROR = SYSLOG_SEVERITY.ERROR,
        FATAL = SYSLOG_SEVERITY.CRITICAL
    }
    return mapping[level] or SYSLOG_SEVERITY.INFO
end

-- RFC 5424 format
local function formatSyslog(facility, level, hostname, appname, procid, msgid, message)
    local severity = luaLevelToSyslog(level)
    local priority = (facility * 8) + severity
    local timestamp = os.date("%Y-%m-%dT%H:%M:%SZ")
    
    return string.format("<%d>1 %s %s %s %s %s %s",
        priority,
        timestamp,
        hostname or "localhost",
        appname or "lua-app",
        procid or "-",
        msgid or "-",
        message)
end

-- ทดสอบ syslog format
local hostname = "web-server-01"
local appname = "myapp"

print(formatSyslog(SYSLOG_FACILITY.USER, "INFO", hostname, appname, "1234", nil, "Application started"))
print(formatSyslog(SYSLOG_FACILITY.USER, "ERROR", hostname, appname, "1234", nil, "Database connection failed"))
print(formatSyslog(SYSLOG_FACILITY.DAEMON, "WARN", hostname, appname, "1234", nil, "High memory usage"))
```

### ตัวอย่างที่ 24: UDP Syslog Sender (Simulation)

```lua
-- syslog_sender.lua
-- ใน production จริงใช้ socket library

local SyslogHandler = {}
SyslogHandler.__index = SyslogHandler

function SyslogHandler.new(config)
    local self = setmetatable({}, SyslogHandler)
    self.host = config.host or "localhost"
    self.port = config.port or 514
    self.facility = config.facility or 1  -- USER
    self.appname = config.appname or "lua-app"
    self.hostname = config.hostname or "localhost"
    return self
end

function SyslogHandler:format(level, message)
    local severityMap = {TRACE=7, DEBUG=7, INFO=6, WARN=4, ERROR=3, FATAL=2}
    local severity = severityMap[level] or 6
    local priority = (self.facility * 8) + severity
    local timestamp = os.date("%b %d %H:%M:%S")
    
    return string.format("<%d>%s %s %s: %s",
        priority, timestamp, self.hostname, self.appname, message)
end

function SyslogHandler:send(level, message)
    local formatted = self:format(level, message)
    -- ใน production จะส่งผ่าน UDP socket
    -- ตอนนี้แค่แสดงว่าจะส่งอะไร
    print(string.format("[Syslog -> %s:%d] %s", self.host, self.port, formatted))
end

-- ทดสอบ
local syslog = SyslogHandler.new({
    host = "syslog.example.com",
    port = 514,
    appname = "payment-service",
    hostname = "prod-server-01"
})

syslog:send("INFO", "Payment processed: order_id=12345 amount=99.99")
syslog:send("ERROR", "Payment failed: order_id=12346 reason=insufficient_funds")
syslog:send("WARN", "Payment gateway latency high: 2500ms")
```

---

## 41.13 Real Application Logging Patterns

### ตัวอย่างที่ 25: Logger Factory Pattern

```lua
-- logger_factory.lua
local LoggerFactory = {}
LoggerFactory.__index = LoggerFactory

local loggers = {}
local globalHandlers = {}
local globalLevel = 3  -- INFO

function LoggerFactory.setGlobalLevel(level)
    globalLevel = type(level) == "string" and 
                  ({TRACE=1,DEBUG=2,INFO=3,WARN=4,ERROR=5,FATAL=6})[level:upper()] or level
end

function LoggerFactory.addGlobalHandler(name, fn)
    globalHandlers[name] = fn
end

function LoggerFactory.getLogger(name)
    if loggers[name] then
        return loggers[name]
    end
    
    local logger = {
        name = name,
        level = nil,  -- nil = use global
        _handlers = {}
    }
    
    local function _doLog(lvl, lvlName, msg, fields)
        local effectiveLevel = logger.level or globalLevel
        if lvl < effectiveLevel then return end
        
        local entry = {
            time = os.date("%Y-%m-%d %H:%M:%S"),
            level = lvl,
            levelName = lvlName,
            logger = name,
            message = msg,
            fields = fields
        }
        
        for _, h in pairs(globalHandlers) do
            pcall(h, entry)
        end
        for _, h in pairs(logger._handlers) do
            pcall(h, entry)
        end
    end
    
    local LVLS = {TRACE=1,DEBUG=2,INFO=3,WARN=4,ERROR=5,FATAL=6}
    for lvlName, lvl in pairs(LVLS) do
        local lname = lvlName:lower()
        local l, n = lvl, lvlName
        logger[lname] = function(self, msg, fields)
            _doLog(l, n, msg, fields)
        end
    end
    
    loggers[name] = logger
    return logger
end

-- Setup global handlers
LoggerFactory.addGlobalHandler("console", function(entry)
    print(string.format("[%s] [%-5s] [%s] %s",
        entry.time, entry.levelName, entry.logger, entry.message))
end)

-- ทดสอบ factory pattern
local appLog = LoggerFactory.getLogger("app")
local dbLog = LoggerFactory.getLogger("database")
local apiLog = LoggerFactory.getLogger("api")

appLog:info("Application starting")
dbLog:info("Connecting to database")
dbLog:warn("Slow query detected", {query = "SELECT *", duration = 1200})
apiLog:error("External API timeout", {service = "payment", timeout = 30})
appLog:info("All services initialized")

-- ดึง logger เดิม (cached)
local appLog2 = LoggerFactory.getLogger("app")
print("\nSame logger instance: " .. tostring(appLog == appLog2))
```

### ตัวอย่างที่ 26: Middleware Logging Pattern

```lua
-- middleware_logger.lua
-- Pattern สำหรับ web framework middleware

local function createRequestLogger(logger)
    return function(request, next)
        local startTime = os.clock()
        local requestId = string.format("%08x", math.random(0, 0x7FFFFFFF))
        
        -- Log request เข้ามา
        logger:info("Request received", {
            requestId = requestId,
            method = request.method,
            path = request.path,
            userAgent = request.userAgent
        })
        
        -- เรียก handler ต่อไป
        local ok, response = pcall(next, request)
        
        local duration = math.floor((os.clock() - startTime) * 1000)
        
        if not ok then
            logger:error("Request failed", {
                requestId = requestId,
                error = tostring(response),
                durationMs = duration
            })
            return nil, response
        end
        
        -- Log response
        local level = "info"
        if response.status >= 500 then level = "error"
        elseif response.status >= 400 then level = "warn"
        end
        
        logger[level](logger, "Request completed", {
            requestId = requestId,
            status = response.status,
            durationMs = duration,
            bytes = response.bodySize or 0
        })
        
        return response
    end
end

-- Simulation
local function makeLogger(name)
    return {
        info = function(self, msg, f) 
            local parts = {}
            if f then for k,v in pairs(f) do table.insert(parts, k.."="..tostring(v)) end end
            print(string.format("[INFO] [%s] %s %s", name, msg, table.concat(parts, " ")))
        end,
        warn = function(self, msg, f)
            local parts = {}
            if f then for k,v in pairs(f) do table.insert(parts, k.."="..tostring(v)) end end
            print(string.format("[WARN] [%s] %s %s", name, msg, table.concat(parts, " ")))
        end,
        error = function(self, msg, f)
            local parts = {}
            if f then for k,v in pairs(f) do table.insert(parts, k.."="..tostring(v)) end end
            print(string.format("[ERROR] [%s] %s %s", name, msg, table.concat(parts, " ")))
        end
    }
end

local logger = makeLogger("http")
local requestLogger = createRequestLogger(logger)

-- จำลอง requests
math.randomseed(42)
local requests = {
    {method = "GET",  path = "/api/users",    userAgent = "Mozilla/5.0"},
    {method = "POST", path = "/api/login",    userAgent = "curl/7.68.0"},
    {method = "GET",  path = "/api/products", userAgent = "MyApp/1.0"},
}

local handlers = {
    function(req) return {status = 200, bodySize = 1024} end,
    function(req) return {status = 401, bodySize = 64} end,
    function(req) error("Database connection refused") end
}

for i, req in ipairs(requests) do
    print(string.format("\n--- Request %d ---", i))
    requestLogger(req, handlers[i])
end
```

### ตัวอย่างที่ 27: Error Logging with Stack Traces

```lua
-- error_logging.lua

local function getStackTrace()
    local trace = {}
    local level = 3  -- Skip this function and the caller
    while true do
        local info = debug.getinfo(level, "Sln")
        if not info then break end
        table.insert(trace, string.format("  at %s (%s:%d)",
            info.name or "?",
            info.short_src or "?",
            info.currentline or 0))
        level = level + 1
    end
    return table.concat(trace, "\n")
end

local function logError(logger, err, context)
    local message
    local stack
    
    if type(err) == "table" then
        message = err.message or tostring(err)
        stack = err.traceback or debug.traceback("", 2)
    else
        message = tostring(err)
        stack = debug.traceback("", 2)
    end
    
    print(string.format("[ERROR] %s", message))
    if context then
        print(string.format("  Context: %s", context))
    end
    if stack then
        print("  Stack trace:")
        for line in stack:gmatch("[^\n]+") do
            if not line:match("^stack traceback") then
                print("  " .. line)
            end
        end
    end
end

-- ทดสอบ
local function level3()
    error("Something went wrong in level3")
end

local function level2()
    level3()
end

local function level1()
    level2()
end

local ok, err = pcall(level1)
if not ok then
    logError(nil, err, "Processing user request")
end

-- Error object pattern
local function newError(code, message, details)
    return {
        code = code,
        message = message,
        details = details,
        traceback = debug.traceback("", 2)
    }
end

local function riskyOperation()
    return nil, newError("DB_ERROR", "Query failed", {
        query = "SELECT * FROM users",
        host = "db.example.com"
    })
end

local result, err2 = riskyOperation()
if not result then
    print("\n=== Structured Error ===")
    print(string.format("Code: %s", err2.code))
    print(string.format("Message: %s", err2.message))
    if err2.details then
        for k, v in pairs(err2.details) do
            print(string.format("  %s: %s", k, tostring(v)))
        end
    end
end
```

---

## 41.14 Production-Ready Logger

### ตัวอย่างที่ 28: Complete Production Logger

```lua
-- production_logger.lua
local Logger = {}
Logger.__index = Logger

local LEVELS = {
    TRACE = 1, DEBUG = 2, INFO = 3,
    WARN  = 4, ERROR = 5, FATAL = 6
}
local LEVEL_NAMES = {[1]="TRACE",[2]="DEBUG",[3]="INFO",[4]="WARN",[5]="ERROR",[6]="FATAL"}

-- Global registry
local _registry = {
    loggers = {},
    root = nil
}

local function jsonValue(v)
    local t = type(v)
    if t == "nil" then return "null"
    elseif t == "boolean" then return tostring(v)
    elseif t == "number" then return tostring(v)
    elseif t == "string" then return '"' .. v:gsub('"','\\"'):gsub('\n','\\n') .. '"'
    elseif t == "table" then
        local parts = {}
        for k, val in pairs(v) do
            table.insert(parts, '"'..tostring(k)..'":'..jsonValue(val))
        end
        return "{"..table.concat(parts, ",").."}"
    else return '"'..tostring(v)..'"' end
end

function Logger.new(config)
    local self = setmetatable({}, Logger)
    self.name = config.name or "root"
    self.level = config.level or LEVELS.INFO
    self.handlers = config.handlers or {}
    self.context = config.context or {}
    self.propagate = config.propagate ~= false
    self._entryCount = 0
    self._errorCount = 0
    return self
end

function Logger:child(name, extraContext)
    local childContext = {}
    for k, v in pairs(self.context) do childContext[k] = v end
    if extraContext then
        for k, v in pairs(extraContext) do childContext[k] = v end
    end
    
    local child = Logger.new({
        name = self.name .. "." .. name,
        level = self.level,
        handlers = self.handlers,
        context = childContext
    })
    return child
end

function Logger:_dispatch(level, message, fields)
    if level < self.level then return end
    
    self._entryCount = self._entryCount + 1
    if level >= LEVELS.ERROR then
        self._errorCount = self._errorCount + 1
    end
    
    -- Merge contexts
    local allFields = {}
    for k, v in pairs(self.context) do allFields[k] = v end
    if fields then
        for k, v in pairs(fields) do allFields[k] = v end
    end
    
    local entry = {
        timestamp   = os.date("%Y-%m-%dT%H:%M:%SZ"),
        level       = level,
        levelName   = LEVEL_NAMES[level],
        logger      = self.name,
        message     = message,
        fields      = allFields,
        pid         = 0  -- os.getenv doesn't give us PID easily
    }
    
    for _, handler in ipairs(self.handlers) do
        local ok, err = pcall(handler.write, handler, entry)
        if not ok then
            io.stderr:write("[LOGGER INTERNAL ERROR] " .. tostring(err) .. "\n")
        end
    end
end

-- Convenience methods
for lvlName, lvl in pairs(LEVELS) do
    local l, n = lvl, lvlName
    Logger[lvlName:lower()] = function(self, msg, fields)
        self:_dispatch(l, msg, fields)
    end
end

function Logger:getStats()
    return {
        total = self._entryCount,
        errors = self._errorCount
    }
end

-- Handlers
local ConsoleHandler = {}
ConsoleHandler.__index = ConsoleHandler

function ConsoleHandler.new(config)
    config = config or {}
    local self = setmetatable({}, ConsoleHandler)
    self.format = config.format or "text"  -- "text" or "json"
    self.minLevel = config.minLevel or LEVELS.TRACE
    self.stream = config.stream or io.stdout
    return self
end

function ConsoleHandler:write(entry)
    if entry.level < self.minLevel then return end
    
    local line
    if self.format == "json" then
        local obj = {
            '"ts":"' .. entry.timestamp .. '"',
            '"level":"' .. entry.levelName .. '"',
            '"logger":"' .. entry.logger .. '"',
            '"msg":"' .. entry.message:gsub('"', '\\"') .. '"'
        }
        for k, v in pairs(entry.fields or {}) do
            if type(v) == "number" then
                table.insert(obj, '"'..k..'":'..v)
            else
                table.insert(obj, '"'..k..'":"'..tostring(v)..'"')
            end
        end
        line = "{" .. table.concat(obj, ",") .. "}"
    else
        local fieldStr = ""
        if next(entry.fields or {}) then
            local parts = {}
            for k, v in pairs(entry.fields) do
                table.insert(parts, k.."="..tostring(v))
            end
            table.sort(parts)
            fieldStr = " [" .. table.concat(parts, " ") .. "]"
        end
        line = string.format("%s %-5s %s: %s%s",
            entry.timestamp, entry.levelName, entry.logger,
            entry.message, fieldStr)
    end
    
    self.stream:write(line .. "\n")
    self.stream:flush()
end

-- ทดสอบ production logger
local rootLogger = Logger.new({
    name = "app",
    level = LEVELS.DEBUG,
    handlers = {ConsoleHandler.new({format = "text"})}
})

local dbLogger = rootLogger:child("database", {component = "postgres"})
local apiLogger = rootLogger:child("api", {component = "rest"})

rootLogger:info("Application starting", {version = "2.1.0", env = "production"})
dbLogger:info("Connection pool initialized", {poolSize = 10, host = "db-01"})
apiLogger:info("HTTP server listening", {port = 8080, protocol = "http2"})
dbLogger:warn("Slow query detected", {query = "reports", durationMs = 1250})
apiLogger:error("Upstream timeout", {service = "payment-gw", timeout = 30})
rootLogger:info("Shutdown initiated")

local stats = rootLogger:getStats()
print(string.format("\nLogger stats: total=%d errors=%d", stats.total, stats.errors))
```

### ตัวอย่างที่ 29: Log Sampling

```lua
-- log_sampling.lua
-- Sampling ลด volume ของ logs โดยไม่สูญเสียข้อมูลสำคัญ

local SamplingLogger = {}
SamplingLogger.__index = SamplingLogger

function SamplingLogger.new(baseLogger, config)
    local self = setmetatable({}, SamplingLogger)
    self.base = baseLogger
    self.config = {
        DEBUG = config.debugRate or 0.1,   -- 10% of debug logs
        INFO  = config.infoRate  or 0.5,   -- 50% of info logs
        WARN  = config.warnRate  or 1.0,   -- 100% of warn logs
        ERROR = config.errorRate or 1.0,   -- 100% of error logs
        FATAL = 1.0                        -- 100% of fatal logs
    }
    self.counters = {}
    return self
end

function SamplingLogger:_shouldLog(level, key)
    local rate = self.config[level] or 1.0
    if rate >= 1.0 then return true end
    if rate <= 0.0 then return false end
    
    -- Deterministic sampling based on counter
    local counter = (self.counters[key] or 0) + 1
    self.counters[key] = counter
    
    -- Log every 1/rate messages
    local threshold = math.floor(1 / rate)
    return (counter % threshold) == 1
end

function SamplingLogger:log(level, key, msg, fields)
    if self:_shouldLog(level, key) then
        local logFn = self.base[level:lower()]
        if logFn then
            logFn(self.base, msg, fields)
        end
    end
end

-- ทดสอบ
local function makeSimpleLogger()
    local counts = {logged = 0, total = 0}
    local log = {counts = counts}
    for _, level in ipairs({"debug", "info", "warn", "error"}) do
        local l = level:upper()
        log[level] = function(self, msg, fields)
            self.counts.logged = self.counts.logged + 1
            -- ไม่ print ทั้งหมด เพื่อ demo
        end
    end
    return log
end

local base = makeSimpleLogger()
local sampledLog = SamplingLogger.new(base, {
    debugRate = 0.1,
    infoRate  = 0.25
})

-- จำลอง high-volume logging
for i = 1, 100 do
    sampledLog:log("DEBUG", "cache_hit", "Cache hit", {key = "user:" .. i})
    sampledLog:log("INFO",  "req", "Request handled", {reqId = i})
end

-- นับ counter
local debugLogged = 0
local infoLogged = 0
for key, count in pairs(sampledLog.counters) do
    if key == "cache_hit" then debugLogged = count
    elseif key == "req" then infoLogged = count end
end

print(string.format("Total debug calls: 100, counter reached: %d", debugLogged))
print(string.format("Total info calls:  100, counter reached: %d", infoLogged))
print(string.format("Debug - every ~%d logged", math.floor(1/0.1)))
print(string.format("Info  - every ~%d logged", math.floor(1/0.25)))
```

---

## 41.15 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 30: Rate-Limited Logging

```lua
-- rate_limited_logger.lua
-- จำกัดอัตรา log ต่อ key เพื่อป้องกัน log flooding

local RateLimitedLogger = {}
RateLimitedLogger.__index = RateLimitedLogger

function RateLimitedLogger.new(baseLog, ratePerSec)
    local self = setmetatable({}, RateLimitedLogger)
    self.base = baseLog
    self.ratePerSec = ratePerSec or 10
    self.buckets = {}  -- key -> {count, windowStart}
    return self
end

function RateLimitedLogger:_checkRate(key)
    local now = os.time()
    local bucket = self.buckets[key]
    
    if not bucket or (now - bucket.windowStart) >= 1 then
        -- รีเซ็ต window ใหม่
        self.buckets[key] = {count = 1, windowStart = now}
        return true
    end
    
    if bucket.count < self.ratePerSec then
        bucket.count = bucket.count + 1
        return true
    end
    
    -- เกิน rate limit
    return false
end

function RateLimitedLogger:warn(key, msg, fields)
    if self:_checkRate(key) then
        print(string.format("[WARN] %s | %s", msg, key))
    else
        -- นับจำนวนที่ถูก suppress (log สรุปทีหลัง)
        local bucket = self.buckets[key]
        bucket.suppressed = (bucket.suppressed or 0) + 1
    end
end

-- ทดสอบ
local log = RateLimitedLogger.new(nil, 3)  -- max 3 per second

print("Simulating 10 rapid warnings for same key:")
for i = 1, 10 do
    log:warn("conn_error", "Connection failed to " .. i, {host = "db-" .. i})
end

-- ดู suppressed count
local bucket = log.buckets["conn_error"]
if bucket then
    print(string.format("Suppressed: %d messages", bucket.suppressed or 0))
end
```

### ตัวอย่างที่ 31: Log Metrics

```lua
-- log_metrics.lua
-- ติดตาม metrics จาก logs

local LogMetrics = {}
LogMetrics.__index = LogMetrics

function LogMetrics.new()
    local self = setmetatable({}, LogMetrics)
    self.counters = {}
    self.timers = {}
    self.gauges = {}
    return self
end

function LogMetrics:increment(name, value, tags)
    local key = name
    if tags then
        local tagParts = {}
        for k, v in pairs(tags) do
            table.insert(tagParts, k .. ":" .. v)
        end
        table.sort(tagParts)
        key = name .. "{" .. table.concat(tagParts, ",") .. "}"
    end
    self.counters[key] = (self.counters[key] or 0) + (value or 1)
end

function LogMetrics:timing(name, durationMs, tags)
    local key = name
    if not self.timers[key] then
        self.timers[key] = {count = 0, sum = 0, min = math.huge, max = -math.huge}
    end
    local t = self.timers[key]
    t.count = t.count + 1
    t.sum = t.sum + durationMs
    if durationMs < t.min then t.min = durationMs end
    if durationMs > t.max then t.max = durationMs end
end

function LogMetrics:report()
    print("=== Log Metrics Report ===")
    print("\nCounters:")
    for name, value in pairs(self.counters) do
        print(string.format("  %-40s %d", name, value))
    end
    
    print("\nTimers:")
    for name, t in pairs(self.timers) do
        print(string.format("  %s:", name))
        print(string.format("    count=%-6d avg=%.1fms min=%.1fms max=%.1fms",
            t.count, t.sum/t.count, t.min, t.max))
    end
end

-- ทดสอบ
local metrics = LogMetrics.new()
math.randomseed(42)

-- จำลอง HTTP requests
local methods = {"GET", "POST", "PUT", "DELETE"}
local paths = {"/api/users", "/api/products", "/api/orders"}
local statuses = {200, 200, 200, 201, 400, 404, 500}

for i = 1, 100 do
    local method = methods[math.random(#methods)]
    local path = paths[math.random(#paths)]
    local status = statuses[math.random(#statuses)]
    local duration = math.random(10, 500)
    
    metrics:increment("http.requests.total", 1, {method=method, status=status})
    metrics:timing("http.request.duration", duration, {method=method})
    
    if status >= 500 then
        metrics:increment("http.errors.5xx")
    elseif status >= 400 then
        metrics:increment("http.errors.4xx")
    end
end

metrics:report()
```

### ตัวอย่างที่ 32: Log Redaction (ซ่อนข้อมูลลับ)

```lua
-- log_redaction.lua
-- ซ่อนข้อมูล sensitive ใน logs

local Redactor = {}
Redactor.__index = Redactor

function Redactor.new()
    local self = setmetatable({}, Redactor)
    self.patterns = {}
    self.fields = {}  -- field names to always redact
    return self
end

function Redactor:addPattern(pattern, replacement)
    table.insert(self.patterns, {
        pattern = pattern,
        replacement = replacement or "[REDACTED]"
    })
end

function Redactor:addSensitiveField(fieldName)
    self.fields[fieldName:lower()] = true
end

function Redactor:redactString(s)
    if type(s) ~= "string" then return s end
    for _, rule in ipairs(self.patterns) do
        s = s:gsub(rule.pattern, rule.replacement)
    end
    return s
end

function Redactor:redactFields(fields)
    if not fields then return fields end
    local redacted = {}
    for k, v in pairs(fields) do
        if self.fields[k:lower()] then
            redacted[k] = "[REDACTED]"
        elseif type(v) == "string" then
            redacted[k] = self:redactString(v)
        else
            redacted[k] = v
        end
    end
    return redacted
end

-- Setup redactor
local redactor = Redactor.new()

-- Patterns ที่ต้อง redact
redactor:addPattern("%d%d%d%d%-%d%d%d%d%-%d%d%d%d%-%d%d%d%d", "****-****-****-****")  -- Credit card
redactor:addPattern("[%w%.]+@[%w%.]+%.[%a]+", "[EMAIL]")  -- Email
redactor:addPattern("%d%d%d%-%d%d%-%d%d%d%d", "[SSN]")    -- SSN

-- Sensitive fields
redactor:addSensitiveField("password")
redactor:addSensitiveField("token")
redactor:addSensitiveField("apiKey")
redactor:addSensitiveField("secret")

-- ทดสอบ
local sensitiveMessage = "User john@example.com paid with card 4532-1234-5678-9012"
print("Original: " .. sensitiveMessage)
print("Redacted: " .. redactor:redactString(sensitiveMessage))

local sensitiveFields = {
    username = "john_doe",
    email = "john@example.com",
    password = "my_secret_password_123",
    token = "eyJhbGciOiJIUzI1NiJ9.abc",
    creditCard = "4532-1234-5678-9012",
    amount = 99.99
}

print("\nOriginal fields:")
for k, v in pairs(sensitiveFields) do
    print(string.format("  %s: %s", k, tostring(v)))
end

local redactedFields = redactor:redactFields(sensitiveFields)
print("\nRedacted fields:")
for k, v in pairs(redactedFields) do
    print(string.format("  %s: %s", k, tostring(v)))
end
```

### ตัวอย่างที่ 33: Log Search และ Filter

```lua
-- log_search.lua
-- ค้นหาและกรอง logs

local LogStore = {}
LogStore.__index = LogStore

function LogStore.new(maxEntries)
    local self = setmetatable({}, LogStore)
    self.entries = {}
    self.maxEntries = maxEntries or 10000
    return self
end

function LogStore:append(entry)
    table.insert(self.entries, entry)
    -- ลบ entries เก่าถ้าเกิน limit
    if #self.entries > self.maxEntries then
        table.remove(self.entries, 1)
    end
end

function LogStore:search(criteria)
    local results = {}
    for _, entry in ipairs(self.entries) do
        local match = true
        
        if criteria.level then
            local levelNums = {TRACE=1,DEBUG=2,INFO=3,WARN=4,ERROR=5,FATAL=6}
            if (levelNums[entry.level] or 0) < (levelNums[criteria.level] or 0) then
                match = false
            end
        end
        
        if match and criteria.message then
            if not entry.message:lower():find(criteria.message:lower(), 1, true) then
                match = false
            end
        end
        
        if match and criteria.logger then
            if not entry.logger:find(criteria.logger, 1, true) then
                match = false
            end
        end
        
        if match and criteria.after then
            if entry.timestamp < criteria.after then
                match = false
            end
        end
        
        if match then
            table.insert(results, entry)
            if criteria.limit and #results >= criteria.limit then
                break
            end
        end
    end
    return results
end

function LogStore:tail(n)
    n = n or 10
    local start = math.max(1, #self.entries - n + 1)
    local result = {}
    for i = start, #self.entries do
        table.insert(result, self.entries[i])
    end
    return result
end

-- ทดสอบ
local store = LogStore.new(1000)

-- เพิ่ม sample entries
local logData = {
    {level="INFO",  logger="api",      message="GET /api/users 200"},
    {level="DEBUG", logger="db",       message="Query: SELECT * FROM users"},
    {level="WARN",  logger="api",      message="Rate limit exceeded for IP 192.168.1.1"},
    {level="ERROR", logger="payment",  message="Payment gateway timeout"},
    {level="INFO",  logger="api",      message="POST /api/orders 201"},
    {level="ERROR", logger="db",       message="Connection pool exhausted"},
    {level="INFO",  logger="worker",   message="Job completed: send_email"},
    {level="FATAL", logger="app",      message="Unhandled exception: nil pointer"},
    {level="INFO",  logger="api",      message="GET /api/products 200"},
    {level="WARN",  logger="db",       message="Slow query: 1500ms"},
}

local baseTime = os.time()
for i, data in ipairs(logData) do
    store:append({
        timestamp = baseTime + i,
        level = data.level,
        logger = data.logger,
        message = data.message
    })
end

-- ค้นหา
print("=== ERROR and above ===")
for _, e in ipairs(store:search({level = "ERROR"})) do
    print(string.format("[%s] [%s] %s", e.level, e.logger, e.message))
end

print("\n=== Messages containing 'timeout' ===")
for _, e in ipairs(store:search({message = "timeout"})) do
    print(string.format("[%s] [%s] %s", e.level, e.logger, e.message))
end

print("\n=== Last 3 entries ===")
for _, e in ipairs(store:tail(3)) do
    print(string.format("[%s] [%s] %s", e.level, e.logger, e.message))
end
```

### ตัวอย่างที่ 34: Logger Hierarchy

```lua
-- logger_hierarchy.lua
-- ระบบ logger แบบ hierarchy เหมือน Python logging

local LogManager = {}
LogManager._loggers = {}

local Logger = {}
Logger.__index = Logger

local LEVELS = {TRACE=1,DEBUG=2,INFO=3,WARN=4,ERROR=5,FATAL=6}
local LEVEL_NAMES = {[1]="TRACE",[2]="DEBUG",[3]="INFO",[4]="WARN",[5]="ERROR",[6]="FATAL"}

function Logger.new(name)
    local self = setmetatable({}, Logger)
    self.name = name
    self._level = nil  -- nil = inherit from parent
    self._handlers = {}
    self.propagate = true
    return self
end

function Logger:getParent()
    -- หา parent โดยตัด prefix สุดท้ายออก
    local parentName = self.name:match("^(.+)%.[^%.]+$")
    if parentName then
        return LogManager.getLogger(parentName)
    end
    return LogManager.getLogger("root")
end

function Logger:getEffectiveLevel()
    if self._level then return self._level end
    if self.name == "root" then return LEVELS.INFO end
    local parent = self:getParent()
    if parent then return parent:getEffectiveLevel() end
    return LEVELS.INFO
end

function Logger:setLevel(level)
    self._level = LEVELS[level] or level
end

function Logger:addHandler(fn)
    table.insert(self._handlers, fn)
end

function Logger:_handle(entry)
    for _, h in ipairs(self._handlers) do
        h(entry)
    end
    if self.propagate and self.name ~= "root" then
        self:getParent():_handle(entry)
    end
end

function Logger:log(level, msg, fields)
    if level < self:getEffectiveLevel() then return end
    local entry = {
        logger = self.name,
        level = level,
        levelName = LEVEL_NAMES[level],
        message = msg,
        fields = fields,
        timestamp = os.date("%H:%M:%S")
    }
    self:_handle(entry)
end

for lvlName, lvl in pairs(LEVELS) do
    local l = lvl
    Logger[lvlName:lower()] = function(self, msg, f) self:log(l, msg, f) end
end

function LogManager.getLogger(name)
    if not LogManager._loggers[name] then
        LogManager._loggers[name] = Logger.new(name)
    end
    return LogManager._loggers[name]
end

-- Setup
local root = LogManager.getLogger("root")
root:addHandler(function(entry)
    print(string.format("[%s] [%s] [%s] %s",
        entry.timestamp, entry.levelName, entry.logger, entry.message))
end)
root:setLevel("DEBUG")

-- ทดสอบ hierarchy
local appLog = LogManager.getLogger("myapp")
local dbLog = LogManager.getLogger("myapp.database")
local queryLog = LogManager.getLogger("myapp.database.query")

-- query logger จะ inherit level จาก parent
appLog:setLevel("INFO")  -- INFO สำหรับ myapp และ descendants

appLog:info("Application ready")
dbLog:info("Database connected")
dbLog:debug("Debug DB message - ไม่แสดงถ้า level=INFO")
queryLog:warn("Slow query detected")
queryLog:error("Query failed")

-- เพิ่ม handler เฉพาะ error log
dbLog:addHandler(function(entry)
    if entry.level >= LEVELS.ERROR then
        io.stderr:write(string.format("[DB ALERT] %s\n", entry.message))
    end
end)
dbLog:error("Critical: Connection pool empty")
```

### ตัวอย่างที่ 35: Complete Logging Integration Example

```lua
-- integration_example.lua
-- ตัวอย่างการใช้ logging ใน application จริง

-- === ระบบ Order Processing ===

local function createLogger(name)
    return {
        name = name,
        _log = function(self, level, msg, fields)
            local f = fields or {}
            local parts = {}
            for k, v in pairs(f) do
                table.insert(parts, k .. "=" .. tostring(v))
            end
            local fieldStr = #parts > 0 and " " .. table.concat(parts, " ") or ""
            print(string.format("[%s] [%s] %s%s", level, self.name, msg, fieldStr))
        end
    }
end

local function addMethods(log)
    for _, level in ipairs({"INFO", "DEBUG", "WARN", "ERROR", "FATAL"}) do
        local l = level
        log[level:lower()] = function(self, msg, fields)
            self:_log(l, msg, fields)
        end
    end
    return log
end

local log = addMethods(createLogger("order-service"))
local dbLog = addMethods(createLogger("order-service.db"))
local payLog = addMethods(createLogger("order-service.payment"))

-- Order processing workflow
local function processOrder(order)
    local correlationId = string.format("ord-%06d", math.random(999999))
    
    log:info("Order received", {
        correlationId = correlationId,
        orderId = order.id,
        userId = order.userId,
        itemCount = #order.items,
        total = order.total
    })
    
    -- 1. Validate order
    log:debug("Validating order", {correlationId = correlationId})
    if order.total <= 0 then
        log:error("Invalid order total", {
            correlationId = correlationId,
            total = order.total
        })
        return false, "Invalid total"
    end
    
    -- 2. Check inventory
    log:debug("Checking inventory", {correlationId = correlationId, itemCount = #order.items})
    for _, item in ipairs(order.items) do
        if item.quantity > item.stock then
            log:warn("Insufficient stock", {
                correlationId = correlationId,
                productId = item.productId,
                requested = item.quantity,
                available = item.stock
            })
            return false, "Insufficient stock for " .. item.productId
        end
    end
    
    -- 3. Process payment
    payLog:info("Initiating payment", {
        correlationId = correlationId,
        amount = order.total,
        method = order.paymentMethod
    })
    
    -- Simulate payment processing
    local paymentOk = order.total < 10000  -- Mock: large orders fail
    if not paymentOk then
        payLog:error("Payment declined", {
            correlationId = correlationId,
            amount = order.total,
            reason = "Amount exceeds daily limit"
        })
        return false, "Payment declined"
    end
    
    payLog:info("Payment approved", {
        correlationId = correlationId,
        transactionId = "txn-" .. math.random(100000)
    })
    
    -- 4. Update database
    dbLog:debug("Updating order status", {correlationId = correlationId})
    dbLog:info("Order saved to database", {
        correlationId = correlationId,
        orderId = order.id
    })
    
    log:info("Order processed successfully", {
        correlationId = correlationId,
        orderId = order.id,
        status = "confirmed"
    })
    
    return true, correlationId
end

-- ทดสอบ
math.randomseed(42)

print("=== Order 1: Normal order ===")
local ok, result = processOrder({
    id = "ORD-001",
    userId = 42,
    total = 299.99,
    paymentMethod = "credit_card",
    items = {
        {productId = "PROD-A", quantity = 2, stock = 10},
        {productId = "PROD-B", quantity = 1, stock = 5}
    }
})
print("Result:", ok, result)

print("\n=== Order 2: Insufficient stock ===")
ok, result = processOrder({
    id = "ORD-002",
    userId = 43,
    total = 150.00,
    paymentMethod = "paypal",
    items = {
        {productId = "PROD-C", quantity = 10, stock = 3}
    }
})
print("Result:", ok, result)

print("\n=== Order 3: Large amount ===")
ok, result = processOrder({
    id = "ORD-003",
    userId = 44,
    total = 15000.00,
    paymentMethod = "credit_card",
    items = {
        {productId = "PROD-D", quantity = 1, stock = 2}
    }
})
print("Result:", ok, result)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Logging Systems ใน Lua ทั้งหมด:

1. **Log Levels** - TRACE, DEBUG, INFO, WARN, ERROR, FATAL และการใช้งาน
2. **Basic Logger** - การสร้าง logger พื้นฐาน
3. **Structured Logging** - JSON logging สำหรับ machine-readable logs
4. **Log Rotation** - การหมุนเวียนไฟล์ log อัตโนมัติ
5. **Multiple Handlers** - console, file, remote handlers
6. **Log Formatting** - text, JSON, colored formats
7. **Correlation IDs** - ติดตาม requests ข้าม services
8. **Context Propagation** - ส่ง context ผ่าน logger
9. **Performance** - lazy evaluation, buffering, sampling
10. **Async Logging** - ใช้ coroutines สำหรับ async logging
11. **Log Aggregation** - รวม logs จากหลายแหล่ง
12. **Syslog Integration** - integration กับ system logging
13. **Production Patterns** - factory pattern, middleware, error handling
14. **Log Redaction** - ซ่อนข้อมูล sensitive
15. **Log Search** - ค้นหาและกรอง logs

Logging ที่ดีเป็นส่วนสำคัญของ production-grade software ช่วยให้เราเข้าใจพฤติกรรมของระบบและแก้ไขปัญหาได้อย่างรวดเร็ว
