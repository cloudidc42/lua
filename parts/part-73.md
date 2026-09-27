# บทที่ 73: IoT กับ NodeMCU/ESP8266

## บทนำ

NodeMCU เป็น open-source platform สำหรับ IoT (Internet of Things) ที่ใช้ ESP8266 WiFi chip พร้อมด้วย Lua interpreter ทำให้สามารถเขียนโปรแกรมด้วยภาษา Lua บน microcontroller ได้อย่างสะดวก

## NodeMCU Platform

```lua
-- ข้อมูลพื้นฐานของ NodeMCU/ESP8266

local platform_info = {
    chip = "ESP8266",
    cpu_freq = "80/160 MHz",
    flash_memory = "4MB (ทั่วไป)",
    ram = "80KB (ใช้งานได้จริง ~50KB)",
    gpio_pins = 17,
    adc_pins = 1,  -- 10-bit ADC
    wifi = "802.11 b/g/n 2.4GHz",
    interfaces = {"UART", "SPI", "I2C", "I2S", "PWM"},
    operating_voltage = "3.3V",
    current_draw = {
        active = "~80mA",
        modem_sleep = "~15mA",
        light_sleep = "~0.9mA",
        deep_sleep = "~20μA",
    }
}

print("=== NodeMCU/ESP8266 Specifications ===")
print("Chip: " .. platform_info.chip)
print("CPU: " .. platform_info.cpu_freq)
print("Flash: " .. platform_info.flash_memory)
print("RAM: " .. platform_info.ram)
print("WiFi: " .. platform_info.wifi)
print("GPIO Pins: " .. platform_info.gpio_pins)
print("\nPower consumption:")
for mode, current in pairs(platform_info.current_draw) do
    print(string.format("  %-15s: %s", mode, current))
end
```

## Memory Constraints

```lua
-- การจัดการ memory บน microcontroller

-- ตรวจสอบ memory ที่เหลือ
print("=== Memory Management ===")

-- ใน NodeMCU จะใช้:
-- node.heap() ตรวจสอบ free heap
-- collectgarbage("count") ดู Lua memory usage

-- จำลอง memory monitoring
local function get_memory_info()
    local info = {
        free_heap = 42000,  -- bytes (จำลอง)
        lua_memory = 18000,  -- bytes
        total_heap = 80000,
    }
    info.used_heap = info.total_heap - info.free_heap
    info.usage_percent = math.floor(info.used_heap / info.total_heap * 100)
    return info
end

local mem = get_memory_info()
print(string.format("Free heap: %d bytes (%.1f KB)", mem.free_heap, mem.free_heap/1024))
print(string.format("Used heap: %d bytes (%d%%)", mem.used_heap, mem.usage_percent))
print(string.format("Lua memory: %d bytes", mem.lua_memory))

-- เทคนิคประหยัด memory ใน Lua สำหรับ MCU

-- 1. ใช้ local variables แทน global
local function memory_efficient_example()
    -- ดีกว่า
    local data = {}
    for i = 1, 10 do
        data[i] = i * 2
    end
    -- local ถูก GC เมื่อออกจาก scope
    return data
end

-- 2. ล้าง table หลังใช้งาน
local function process_and_free()
    local temp_data = {}
    for i = 1, 100 do
        temp_data[i] = string.format("item_%d", i)
    end
    
    local result = #temp_data
    
    -- ล้าง table
    for k in pairs(temp_data) do
        temp_data[k] = nil
    end
    temp_data = nil
    collectgarbage("collect")
    
    return result
end

print("Processed items: " .. process_and_free())

-- 3. หลีกเลี่ยง string concatenation ใน loop
local function build_string_efficient(items)
    local parts = {}
    for i, item in ipairs(items) do
        parts[i] = tostring(item)
    end
    return table.concat(parts, ",")
end

-- ไม่ดี (สร้าง string ใหม่ทุกรอบ):
-- local s = ""
-- for i = 1, 100 do s = s .. i .. "," end

local items = {1, 2, 3, 4, 5}
print("Efficient string build: " .. build_string_efficient(items))

-- 4. ใช้ chunk loading สำหรับ code ขนาดใหญ่
-- ใน NodeMCU: dofile("module.lua") แทน require
```

## Flash Storage (SPIFFS)

```lua
-- SPIFFS (SPI Flash File System) สำหรับจัดเก็บไฟล์บน flash

-- ใน NodeMCU จะใช้ file module
-- file.open(), file.write(), file.read(), file.close()

-- จำลอง SPIFFS operations
local SPIFFS = {}
SPIFFS._files = {}  -- จำลอง file system

function SPIFFS.open(filename, mode)
    mode = mode or "r"
    local handle = {
        filename = filename,
        mode = mode,
        position = 1,
        content = mode == "w" and "" or (SPIFFS._files[filename] or ""),
    }
    
    function handle:write(data)
        if self.mode ~= "w" and self.mode ~= "a" and self.mode ~= "w+" then
            error("File not opened for writing")
        end
        if self.mode == "a" then
            self.content = self.content .. data
        else
            self.content = self.content .. data
        end
        return true
    end
    
    function handle:read(length)
        if not length then
            -- อ่านทั้งหมด
            return self.content
        end
        local chunk = self.content:sub(self.position, self.position + length - 1)
        self.position = self.position + #chunk
        return chunk
    end
    
    function handle:readline()
        local line_end = self.content:find("\n", self.position) or (#self.content + 1)
        local line = self.content:sub(self.position, line_end - 1)
        self.position = line_end + 1
        if #line == 0 and self.position > #self.content then
            return nil
        end
        return line
    end
    
    function handle:close()
        if self.mode == "w" or self.mode == "a" or self.mode == "w+" then
            SPIFFS._files[self.filename] = self.content
        end
    end
    
    function handle:seek(where, offset)
        offset = offset or 0
        if where == "set" then
            self.position = offset + 1
        elseif where == "cur" then
            self.position = self.position + offset
        elseif where == "end" then
            self.position = #self.content + offset
        end
    end
    
    return handle
end

function SPIFFS.remove(filename)
    SPIFFS._files[filename] = nil
    return true
end

function SPIFFS.rename(old, new)
    SPIFFS._files[new] = SPIFFS._files[old]
    SPIFFS._files[old] = nil
end

function SPIFFS.list()
    local files = {}
    for name, content in pairs(SPIFFS._files) do
        table.insert(files, {name = name, size = #content})
    end
    return files
end

-- ทดสอบ SPIFFS
print("=== SPIFFS File System ===")

-- เขียนไฟล์
local f = SPIFFS.open("config.json", "w")
f:write('{"ssid":"MyWiFi","password":"secret123","server":"api.example.com"}')
f:close()
print("Written: config.json")

-- เขียนไฟล์ข้อมูล sensor
f = SPIFFS.open("sensor_log.csv", "w")
f:write("timestamp,temperature,humidity\n")
f:write("2024-03-01T10:00:00,25.5,60.2\n")
f:write("2024-03-01T10:01:00,25.6,60.1\n")
f:write("2024-03-01T10:02:00,25.4,60.3\n")
f:close()
print("Written: sensor_log.csv")

-- อ่านไฟล์
f = SPIFFS.open("config.json", "r")
local config_data = f:read()
f:close()
print("Config: " .. config_data)

-- อ่านทีละบรรทัด
f = SPIFFS.open("sensor_log.csv", "r")
print("\nSensor log:")
local line = f:readline()
while line do
    if #line > 0 then
        print("  " .. line)
    end
    line = f:readline()
end
f:close()

-- List files
print("\nFiles in SPIFFS:")
for _, file in ipairs(SPIFFS.list()) do
    print(string.format("  %s (%d bytes)", file.name, file.size))
end
```

## WiFi Connection

```lua
-- การเชื่อมต่อ WiFi บน NodeMCU

-- ใน NodeMCU จริงจะใช้ wifi module
-- wifi.setmode(wifi.STATION)
-- wifi.sta.config({ssid="...", pwd="..."})
-- wifi.sta.connect()

-- จำลอง WiFi module
local wifi_mock = {
    STATION = 1,
    SOFTAP = 2,
    STATIONAP = 3,
    
    status_codes = {
        [0] = "STATION_IDLE",
        [1] = "STATION_CONNECTING",
        [2] = "STATION_WRONG_PASSWORD",
        [3] = "STATION_NO_AP_FOUND",
        [4] = "STATION_CONNECT_FAIL",
        [5] = "STATION_GOT_IP",
    },
    
    current_mode = nil,
    connected = false,
    ip_address = nil,
    ssid = nil,
}

wifi_mock.sta = {}

function wifi_mock.setmode(mode)
    wifi_mock.current_mode = mode
    print("WiFi mode set to: " .. (mode == 1 and "STATION" or mode == 2 and "SOFTAP" or "STATIONAP"))
end

function wifi_mock.sta.config(config)
    print(string.format("Configuring WiFi: SSID=%s", config.ssid))
    wifi_mock.ssid = config.ssid
    wifi_mock._password = config.pwd
    wifi_mock._save = config.save
end

function wifi_mock.sta.connect()
    print("Connecting to WiFi...")
    -- จำลองการเชื่อมต่อ
    wifi_mock.connected = true
    wifi_mock.ip_address = "192.168.1.105"
    print("Connected! IP: " .. wifi_mock.ip_address)
end

function wifi_mock.sta.getip()
    if wifi_mock.connected then
        return wifi_mock.ip_address, "255.255.255.0", "192.168.1.1"
    end
    return nil, nil, nil
end

function wifi_mock.sta.status()
    return wifi_mock.connected and 5 or 1
end

-- ตัวอย่างการเชื่อมต่อ WiFi
print("=== WiFi Connection ===")

wifi_mock.setmode(wifi_mock.STATION)
wifi_mock.sta.config({
    ssid = "HomeNetwork",
    pwd = "mypassword",
    save = true,  -- บันทึก credentials ใน flash
})
wifi_mock.sta.connect()

local ip, netmask, gateway = wifi_mock.sta.getip()
if ip then
    print(string.format("IP: %s, Netmask: %s, Gateway: %s", ip, netmask, gateway))
end

local status = wifi_mock.sta.status()
print("Status: " .. (wifi_mock.status_codes[status] or "UNKNOWN"))

-- WiFi event handlers (ใน NodeMCU จริง)
local function setup_wifi_events()
    -- wifi.eventmon.register(wifi.eventmon.STA_CONNECTED, function(T)
    --     print("WiFi connected to " .. T.SSID)
    -- end)
    
    -- wifi.eventmon.register(wifi.eventmon.STA_GOT_IP, function(T)
    --     print("Got IP: " .. T.IP)
    -- end)
    
    -- wifi.eventmon.register(wifi.eventmon.STA_DISCONNECTED, function(T)
    --     print("WiFi disconnected: " .. T.REASON)
    -- end)
    
    print("WiFi event handlers registered")
end

setup_wifi_events()

-- Smart connect (retry logic)
local function smart_wifi_connect(ssid, password, max_retries)
    max_retries = max_retries or 3
    local retries = 0
    
    local function try_connect()
        retries = retries + 1
        print(string.format("Connection attempt %d/%d...", retries, max_retries))
        
        wifi_mock.sta.config({ssid = ssid, pwd = password})
        wifi_mock.sta.connect()
        
        -- ตรวจสอบสถานะ
        local status = wifi_mock.sta.status()
        if status == 5 then  -- GOT_IP
            local ip = wifi_mock.sta.getip()
            print("Connected! IP: " .. (ip or "?"))
            return true
        end
        
        if retries < max_retries then
            print("Retrying...")
            return try_connect()
        end
        
        print("Failed to connect after " .. max_retries .. " attempts")
        return false
    end
    
    return try_connect()
end

print("\nSmart connect:")
smart_wifi_connect("MySSID", "MyPassword", 3)
```

## HTTP Client บน ESP8266

```lua
-- HTTP Client สำหรับส่งข้อมูลไปยัง server

-- ใน NodeMCU จะใช้ http module หรือ net module
-- http.get(url, headers, callback)
-- http.post(url, headers, body, callback)

-- จำลอง HTTP client
local http_mock = {}

function http_mock.get(url, headers, callback)
    print(string.format("HTTP GET: %s", url))
    
    -- จำลอง response
    local status_code = 200
    local body = '{"status":"ok","data":{"temperature":25.5,"humidity":60}}'
    
    -- เรียก callback
    if callback then
        callback(status_code, body, headers)
    end
end

function http_mock.post(url, headers, body, callback)
    print(string.format("HTTP POST: %s", url))
    print(string.format("Body: %s", body))
    
    -- จำลอง response
    local status_code = 201
    local response_body = '{"id":"sensor-001","status":"created"}'
    
    if callback then
        callback(status_code, response_body, {})
    end
end

-- ตัวอย่างการส่งข้อมูล sensor ไปยัง server
print("=== HTTP Client ===")

local sensor_data = {
    device_id = "esp8266-001",
    temperature = 25.7,
    humidity = 62.3,
    timestamp = os.time(),
}

-- แปลงเป็น JSON (จำลอง)
local function simple_json_encode(t)
    local parts = {}
    for k, v in pairs(t) do
        local value_str
        if type(v) == "string" then
            value_str = '"' .. v .. '"'
        elseif type(v) == "number" then
            value_str = tostring(v)
        elseif type(v) == "boolean" then
            value_str = tostring(v)
        end
        table.insert(parts, '"' .. k .. '":' .. value_str)
    end
    return "{" .. table.concat(parts, ",") .. "}"
end

local json_body = simple_json_encode(sensor_data)

http_mock.post(
    "http://api.example.com/sensors/data",
    {
        ["Content-Type"] = "application/json",
        ["Authorization"] = "Bearer device-token-123",
        ["X-Device-ID"] = "esp8266-001",
    },
    json_body,
    function(code, body, response_headers)
        print(string.format("Response: %d - %s", code, body))
    end
)

-- GET request เพื่อดึง config
http_mock.get(
    "http://api.example.com/devices/esp8266-001/config",
    {["Authorization"] = "Bearer device-token-123"},
    function(code, body, headers)
        print(string.format("Config response: %d - %s", code, body))
    end
)
```

## MQTT Protocol

```lua
-- MQTT เป็น lightweight messaging protocol เหมาะสำหรับ IoT

-- ใน NodeMCU จะใช้ mqtt module
-- mqtt.Client(clientid, keepalive, username, password)

-- จำลอง MQTT client
local MQTTClient = {}
MQTTClient.__index = MQTTClient

function MQTTClient.new(client_id, options)
    options = options or {}
    local self = setmetatable({}, MQTTClient)
    self.client_id = client_id
    self.keepalive = options.keepalive or 60
    self.username = options.username
    self.password = options.password
    self.connected = false
    self.subscriptions = {}
    self.message_queue = {}
    return self
end

function MQTTClient:connect(host, port, secure, callback)
    port = port or 1883
    print(string.format("MQTT connecting to %s:%d (client: %s)", 
        host, port, self.client_id))
    
    -- จำลองการเชื่อมต่อสำเร็จ
    self.connected = true
    self.broker = host
    
    print("MQTT connected!")
    if callback then callback(0) end
end

function MQTTClient:publish(topic, payload, qos, retain)
    qos = qos or 0
    retain = retain or false
    
    if not self.connected then
        print("Error: Not connected to MQTT broker")
        return false
    end
    
    print(string.format("MQTT Publish: topic='%s', payload='%s', qos=%d, retain=%s",
        topic, tostring(payload):sub(1, 50), qos, tostring(retain)))
    
    -- จำลองการเก็บ message
    table.insert(self.message_queue, {
        topic = topic,
        payload = payload,
        qos = qos,
        retain = retain,
        timestamp = os.time(),
    })
    
    return true
end

function MQTTClient:subscribe(topic, qos, callback)
    qos = qos or 0
    print(string.format("MQTT Subscribe: topic='%s', qos=%d", topic, qos))
    
    self.subscriptions[topic] = {
        topic = topic,
        qos = qos,
        callback = callback,
    }
end

function MQTTClient:unsubscribe(topic)
    self.subscriptions[topic] = nil
    print("MQTT Unsubscribed: " .. topic)
end

function MQTTClient:on(event, callback)
    -- event handlers: "connect", "offline", "message"
    self["on_" .. event] = callback
end

function MQTTClient:close()
    self.connected = false
    print("MQTT disconnected")
end

-- Simulate receiving message
function MQTTClient:_simulate_receive(topic, payload)
    for pattern, sub in pairs(self.subscriptions) do
        -- MQTT wildcard matching (simplified)
        local mqtt_pattern = pattern:gsub("+", "[^/]+"):gsub("#", ".+")
        if topic:match("^" .. mqtt_pattern .. "$") then
            if sub.callback then
                sub.callback(self, topic, payload)
            end
        end
    end
end

-- ทดสอบ MQTT
print("=== MQTT Client ===")

local mqtt_client = MQTTClient.new("esp8266-sensor-001", {
    keepalive = 60,
    username = "device",
    password = "devicepassword",
})

mqtt_client:connect("mqtt.example.com", 1883, false, function(status)
    if status == 0 then
        print("MQTT ready!")
    end
end)

-- Subscribe to topics
mqtt_client:subscribe("devices/esp8266-001/commands", 1, function(client, topic, payload)
    print(string.format("Received command: topic=%s, payload=%s", topic, payload))
end)

mqtt_client:subscribe("devices/esp8266-001/config/#", 0, function(client, topic, payload)
    print(string.format("Config update: %s = %s", topic, payload))
end)

-- Publish sensor data
local sensor_readings = {
    {temp = 25.5, humid = 60, light = 450},
    {temp = 25.7, humid = 61, light = 480},
    {temp = 25.3, humid = 59, light = 420},
}

for i, reading in ipairs(sensor_readings) do
    mqtt_client:publish(
        "sensors/esp8266-001/temperature",
        tostring(reading.temp),
        0,  -- QoS 0
        false
    )
    
    mqtt_client:publish(
        "sensors/esp8266-001/data",
        simple_json_encode(reading),
        1,  -- QoS 1 (at least once)
        false
    )
end

-- จำลองการรับ command
print("\nSimulating incoming command:")
mqtt_client:_simulate_receive(
    "devices/esp8266-001/commands",
    '{"action":"restart","delay":5}'
)

mqtt_client:_simulate_receive(
    "devices/esp8266-001/config/interval",
    "30"
)

print(string.format("Messages published: %d", #mqtt_client.message_queue))
```

## DHT11/DHT22 Sensor Reading

```lua
-- DHT11/DHT22 Temperature & Humidity sensors

-- ใน NodeMCU จริงจะใช้:
-- dht.read(pin)  -- สำหรับ DHT11/22
-- dht.read11(pin)  -- สำหรับ DHT11 เท่านั้น

-- จำลอง DHT sensor
local DHT = {}

function DHT.read(pin, sensor_type)
    sensor_type = sensor_type or "DHT22"
    
    -- จำลองการอ่านค่า (ในอุปกรณ์จริงจะอ่านจาก GPIO)
    local base_temp = 25.0
    local base_humid = 60.0
    
    -- เพิ่ม noise เล็กน้อย
    local temp_noise = (math.random() - 0.5) * 2
    local humid_noise = (math.random() - 0.5) * 4
    
    local temperature = base_temp + temp_noise
    local humidity = base_humid + humid_noise
    
    -- DHT11 มี resolution ต่ำกว่า (integer เท่านั้น)
    if sensor_type == "DHT11" then
        temperature = math.floor(temperature)
        humidity = math.floor(humidity)
    else
        -- DHT22: 0.1°C resolution
        temperature = math.floor(temperature * 10) / 10
        humidity = math.floor(humidity * 10) / 10
    end
    
    -- คำนวณ Heat Index (ความร้อนที่รู้สึกได้จริง)
    local heat_index = calculate_heat_index(temperature, humidity)
    
    return {
        status = 0,  -- 0 = OK
        temperature = temperature,
        humidity = humidity,
        heat_index = heat_index,
    }
end

function calculate_heat_index(temp_c, humidity)
    -- Steadman's formula (simplified)
    local t = temp_c * 9/5 + 32  -- แปลงเป็น Fahrenheit
    local rh = humidity
    
    local hi = -42.379 + 2.04901523*t + 10.14333127*rh 
        - 0.22475541*t*rh - 6.83783e-3*t^2 
        - 5.481717e-2*rh^2 + 1.22874e-3*t^2*rh 
        + 8.5282e-4*t*rh^2 - 1.99e-6*t^2*rh^2
    
    return (hi - 32) * 5/9  -- แปลงกลับเป็น Celsius
end

-- Sensor monitor
local function monitor_sensors(pin, interval_ms)
    interval_ms = interval_ms or 2000
    
    print(string.format("=== DHT22 Sensor on Pin %d ===", pin))
    print(string.format("Sampling every %d ms", interval_ms))
    print(string.rep("-", 60))
    print(string.format("%-20s %-15s %-15s %-15s", 
        "Timestamp", "Temp (°C)", "Humidity (%)", "Heat Index"))
    print(string.rep("-", 60))
    
    -- อ่านค่า 5 ครั้ง
    math.randomseed(42)
    for i = 1, 5 do
        local reading = DHT.read(pin, "DHT22")
        
        if reading.status == 0 then
            print(string.format("%-20s %-15.1f %-15.1f %-15.1f",
                os.date("%H:%M:%S"),
                reading.temperature,
                reading.humidity,
                reading.heat_index))
        else
            print("Read error!")
        end
    end
end

monitor_sensors(4)  -- GPIO 4 (D2 on NodeMCU)

-- DHT class สำหรับ continuous monitoring
local SensorManager = {}
SensorManager.__index = SensorManager

function SensorManager.new(pin, sensor_type, options)
    local self = setmetatable({}, SensorManager)
    self.pin = pin
    self.sensor_type = sensor_type or "DHT22"
    self.options = options or {}
    self.readings = {}
    self.max_readings = options.max_readings or 100
    self.alert_temp_high = options.alert_temp_high or 35
    self.alert_humid_high = options.alert_humid_high or 80
    self.callbacks = {}
    return self
end

function SensorManager:read()
    local reading = DHT.read(self.pin, self.sensor_type)
    
    if reading.status == 0 then
        reading.timestamp = os.time()
        
        -- เก็บ reading history
        table.insert(self.readings, reading)
        if #self.readings > self.max_readings then
            table.remove(self.readings, 1)
        end
        
        -- ตรวจสอบ alerts
        self:check_alerts(reading)
        
        -- เรียก callbacks
        for _, cb in ipairs(self.callbacks) do
            cb(reading)
        end
    end
    
    return reading
end

function SensorManager:check_alerts(reading)
    if reading.temperature > self.alert_temp_high then
        print(string.format("⚠️  ALERT: Temperature too high! %.1f°C > %.1f°C",
            reading.temperature, self.alert_temp_high))
    end
    
    if reading.humidity > self.alert_humid_high then
        print(string.format("⚠️  ALERT: Humidity too high! %.1f%% > %.1f%%",
            reading.humidity, self.alert_humid_high))
    end
end

function SensorManager:on_reading(callback)
    table.insert(self.callbacks, callback)
end

function SensorManager:get_average()
    if #self.readings == 0 then return nil end
    
    local sum_temp, sum_humid = 0, 0
    for _, r in ipairs(self.readings) do
        sum_temp = sum_temp + r.temperature
        sum_humid = sum_humid + r.humidity
    end
    
    return {
        temperature = sum_temp / #self.readings,
        humidity = sum_humid / #self.readings,
        count = #self.readings,
    }
end

print("\n=== Sensor Manager ===")
local sensor = SensorManager.new(4, "DHT22", {
    alert_temp_high = 26,  -- ตั้งต่ำเพื่อทดสอบ alert
    max_readings = 50,
})

sensor:on_reading(function(data)
    -- ส่งข้อมูลไปยัง MQTT
    if mqtt_client.connected then
        mqtt_client:publish(
            "sensors/temperature",
            string.format("%.1f", data.temperature),
            0, false
        )
    end
end)

math.randomseed(100)
for i = 1, 5 do
    sensor:read()
end

local avg = sensor:get_average()
if avg then
    print(string.format("Average: Temp=%.1f°C, Humid=%.1f%% (%d readings)",
        avg.temperature, avg.humidity, avg.count))
end
```

## GPIO Control

```lua
-- GPIO (General Purpose Input/Output) control

-- NodeMCU GPIO numbering:
-- D0 = GPIO16, D1 = GPIO5, D2 = GPIO4, D3 = GPIO0
-- D4 = GPIO2, D5 = GPIO14, D6 = GPIO12, D7 = GPIO13
-- D8 = GPIO15

-- จำลอง GPIO module
local gpio = {}
gpio.HIGH = 1
gpio.LOW = 0
gpio.INPUT = 1
gpio.OUTPUT = 0
gpio.PULLUP = 1

local gpio_state = {}

function gpio.mode(pin, mode, pullup)
    gpio_state[pin] = {
        mode = mode,
        value = mode == gpio.INPUT and gpio.LOW or gpio.LOW,
        pullup = pullup,
    }
    print(string.format("GPIO %d set to %s%s", 
        pin, 
        mode == gpio.INPUT and "INPUT" or "OUTPUT",
        pullup and " (PULLUP)" or ""))
end

function gpio.write(pin, value)
    if not gpio_state[pin] then
        error(string.format("GPIO %d not initialized", pin))
    end
    if gpio_state[pin].mode ~= gpio.OUTPUT then
        error(string.format("GPIO %d is not in OUTPUT mode", pin))
    end
    gpio_state[pin].value = value
    -- print(string.format("GPIO %d = %s", pin, value == gpio.HIGH and "HIGH" or "LOW"))
end

function gpio.read(pin)
    if not gpio_state[pin] then
        -- จำลองการอ่านค่า
        return math.random(0, 1)
    end
    return gpio_state[pin].value
end

-- LED control
local function setup_led(pin)
    gpio.mode(pin, gpio.OUTPUT)
    gpio.write(pin, gpio.LOW)
end

local function led_on(pin)
    gpio.write(pin, gpio.HIGH)
end

local function led_off(pin)
    gpio.write(pin, gpio.LOW)
end

local function led_toggle(pin)
    local current = gpio.read(pin)
    gpio.write(pin, current == gpio.HIGH and gpio.LOW or gpio.HIGH)
end

-- Button input
local function setup_button(pin, callback)
    gpio.mode(pin, gpio.INPUT, gpio.PULLUP)
    
    -- ใน NodeMCU จริงจะใช้ gpio.trig
    -- gpio.trig(pin, "down", callback)
    print(string.format("Button on GPIO %d configured", pin))
end

-- Blink pattern
local function blink_pattern(pin, pattern, repeat_count)
    -- pattern = {on_time, off_time, on_time, off_time, ...} ใน ms
    print(string.format("=== GPIO %d Blink Pattern ===", pin))
    
    for rep = 1, (repeat_count or 1) do
        for i, duration in ipairs(pattern) do
            if i % 2 == 1 then
                gpio.write(pin, gpio.HIGH)
                print(string.format("  ON  for %dms", duration))
            else
                gpio.write(pin, gpio.LOW)
                print(string.format("  OFF for %dms", duration))
            end
            -- os.execute(string.format("sleep %g", duration/1000))
        end
    end
end

-- Status LED patterns
local BLINK_PATTERNS = {
    connecting = {500, 500},         -- ช้า: กำลังเชื่อมต่อ
    connected = {100, 900},          -- เร็ว: เชื่อมต่อแล้ว
    error = {100, 100, 100, 700},    -- สั้นๆ สองครั้ง: error
    ota_update = {50, 50},           -- เร็วมาก: กำลัง update
}

print("=== GPIO Control Examples ===")

setup_led(2)  -- D4 = GPIO2 (built-in LED)
print("LED setup on GPIO 2")

print("Connecting pattern:")
blink_pattern(2, BLINK_PATTERNS.connecting, 2)

print("Error pattern:")
blink_pattern(2, BLINK_PATTERNS.error, 2)

-- Traffic light simulation
local TRAFFIC_PINS = {red = 5, yellow = 4, green = 14}

local function setup_traffic_light()
    for color, pin in pairs(TRAFFIC_PINS) do
        gpio.mode(pin, gpio.OUTPUT)
        gpio.write(pin, gpio.LOW)
    end
    print("Traffic light initialized")
end

local function set_traffic_light(color)
    -- ปิดทั้งหมดก่อน
    for c, pin in pairs(TRAFFIC_PINS) do
        gpio.write(pin, gpio.LOW)
    end
    -- เปิดสีที่ต้องการ
    if TRAFFIC_PINS[color] then
        gpio.write(TRAFFIC_PINS[color], gpio.HIGH)
        print("Traffic light: " .. color:upper())
    end
end

setup_traffic_light()
set_traffic_light("red")
set_traffic_light("green")
set_traffic_light("yellow")
```

## PWM (Pulse Width Modulation)

```lua
-- PWM สำหรับควบคุม LED brightness, motor speed, servo

-- ใน NodeMCU จะใช้ pwm module
-- pwm.setup(pin, frequency, duty_cycle)
-- pwm.start(pin)
-- pwm.setduty(pin, duty)
-- pwm.stop(pin)

-- จำลอง PWM module
local pwm_mock = {}
local pwm_state = {}

function pwm_mock.setup(pin, freq, duty)
    -- duty: 0-1023 (10-bit resolution)
    pwm_state[pin] = {
        frequency = freq,
        duty = duty,
        running = false,
    }
    print(string.format("PWM setup: pin=%d, freq=%dHz, duty=%d/1023 (%.1f%%)",
        pin, freq, duty, duty/1023*100))
end

function pwm_mock.start(pin)
    if pwm_state[pin] then
        pwm_state[pin].running = true
        print(string.format("PWM started on pin %d", pin))
    end
end

function pwm_mock.setduty(pin, duty)
    if pwm_state[pin] then
        pwm_state[pin].duty = duty
        print(string.format("PWM duty set: pin=%d, duty=%d/1023 (%.1f%%)",
            pin, duty, duty/1023*100))
    end
end

function pwm_mock.stop(pin)
    if pwm_state[pin] then
        pwm_state[pin].running = false
        print(string.format("PWM stopped on pin %d", pin))
    end
end

-- LED dimmer
local function create_led_dimmer(pin, frequency)
    frequency = frequency or 1000  -- 1kHz
    pwm_mock.setup(pin, frequency, 0)
    pwm_mock.start(pin)
    
    return {
        pin = pin,
        
        set_brightness = function(self, percent)
            -- percent: 0-100
            local duty = math.floor(percent / 100 * 1023)
            pwm_mock.setduty(self.pin, duty)
        end,
        
        fade = function(self, from_pct, to_pct, steps)
            steps = steps or 10
            local step_size = (to_pct - from_pct) / steps
            print(string.format("Fading from %d%% to %d%%...", from_pct, to_pct))
            
            for i = 0, steps do
                local brightness = from_pct + i * step_size
                self:set_brightness(brightness)
                -- ใน production: os.execute("sleep " .. (duration/steps/1000))
            end
        end,
        
        off = function(self)
            pwm_mock.setduty(self.pin, 0)
        end,
        
        full = function(self)
            pwm_mock.setduty(self.pin, 1023)
        end,
    }
end

-- Servo control
local function create_servo(pin)
    -- Servo ใช้ 50Hz PWM
    -- pulse width: 1ms = 0°, 2ms = 180°
    -- ที่ 50Hz, period = 20ms
    -- duty = pulse_width / period * 1023
    
    local FREQ = 50
    local MIN_DUTY = math.floor(1.0/20.0 * 1023)  -- 1ms pulse
    local MAX_DUTY = math.floor(2.0/20.0 * 1023)  -- 2ms pulse
    
    pwm_mock.setup(pin, FREQ, MIN_DUTY)
    pwm_mock.start(pin)
    
    return {
        pin = pin,
        
        set_angle = function(self, angle)
            -- angle: 0-180 degrees
            angle = math.max(0, math.min(180, angle))
            local duty = MIN_DUTY + (MAX_DUTY - MIN_DUTY) * angle / 180
            pwm_mock.setduty(self.pin, math.floor(duty))
            print(string.format("Servo angle: %d°", angle))
        end,
        
        sweep = function(self, from_angle, to_angle, steps)
            local step_size = (to_angle - from_angle) / steps
            for i = 0, steps do
                self:set_angle(from_angle + i * step_size)
            end
        end,
    }
end

print("=== PWM Examples ===")

-- LED dimmer
local led = create_led_dimmer(5)  -- D1 = GPIO5
print("\nLED Dimming:")
led:set_brightness(0)
led:set_brightness(25)
led:set_brightness(50)
led:set_brightness(75)
led:set_brightness(100)

-- Servo
print("\nServo Control:")
local servo = create_servo(14)  -- D5 = GPIO14
servo:set_angle(0)
servo:set_angle(90)
servo:set_angle(180)
servo:set_angle(45)
```

## I2C Interface

```lua
-- I2C (Inter-Integrated Circuit) สำหรับเชื่อมต่อ sensors

-- ใน NodeMCU จะใช้ i2c module
-- i2c.setup(id, sda_pin, scl_pin, speed)
-- i2c.start(id), i2c.stop(id)
-- i2c.address(id, device_addr, direction)
-- i2c.write(id, data), i2c.read(id, length)

-- จำลอง I2C module
local i2c_mock = {}
local i2c_bus = {}

function i2c_mock.setup(id, sda, scl, speed)
    i2c_bus[id] = {
        sda = sda,
        scl = scl,
        speed = speed,
        buffer = {},
    }
    print(string.format("I2C setup: id=%d, SDA=GPIO%d, SCL=GPIO%d, speed=%d",
        id, sda, scl, speed))
end

function i2c_mock.start(id)
    i2c_bus[id].in_transaction = true
end

function i2c_mock.stop(id)
    i2c_bus[id].in_transaction = false
    i2c_bus[id].buffer = {}
end

function i2c_mock.address(id, addr, direction)
    i2c_bus[id].target_addr = addr
    i2c_bus[id].direction = direction  -- 0 = write, 1 = read
    -- ส่ง address byte
    return true  -- ACK
end

function i2c_mock.write(id, data)
    if type(data) == "number" then
        table.insert(i2c_bus[id].buffer, data)
    else
        for i = 1, #data do
            table.insert(i2c_bus[id].buffer, string.byte(data, i))
        end
    end
    return true
end

function i2c_mock.read(id, length)
    -- จำลองการอ่านข้อมูล
    local bytes = {}
    for i = 1, length do
        table.insert(bytes, math.random(0, 255))
    end
    return string.char(table.unpack(bytes))
end

-- BMP280 Barometric Pressure Sensor
local BMP280 = {}

BMP280.I2C_ADDR = 0x76  -- หรือ 0x77
BMP280.REG_TEMP_MSB = 0xFA
BMP280.REG_PRESS_MSB = 0xF7
BMP280.REG_CONFIG = 0xF5
BMP280.REG_CTRL_MEAS = 0xF4
BMP280.REG_STATUS = 0xF3
BMP280.REG_RESET = 0xE0
BMP280.REG_ID = 0xD0
BMP280.REG_CALIB = 0x88

function BMP280.new(i2c_id, address)
    local sensor = {
        i2c_id = i2c_id,
        address = address or BMP280.I2C_ADDR,
    }
    
    function sensor:init()
        -- Reset
        self:write_reg(BMP280.REG_RESET, 0xB6)
        
        -- ตั้งค่า oversampling และ mode
        self:write_reg(BMP280.REG_CTRL_MEAS, 0x27)  -- temp x1, press x1, normal mode
        self:write_reg(BMP280.REG_CONFIG, 0xA0)  -- standby 1000ms, filter x16
        
        print(string.format("BMP280 initialized at 0x%02X", self.address))
        return true
    end
    
    function sensor:write_reg(reg, value)
        i2c_mock.start(self.i2c_id)
        i2c_mock.address(self.i2c_id, self.address, 0)
        i2c_mock.write(self.i2c_id, reg)
        i2c_mock.write(self.i2c_id, value)
        i2c_mock.stop(self.i2c_id)
    end
    
    function sensor:read_reg(reg, length)
        -- Write register address
        i2c_mock.start(self.i2c_id)
        i2c_mock.address(self.i2c_id, self.address, 0)
        i2c_mock.write(self.i2c_id, reg)
        
        -- Read data
        i2c_mock.start(self.i2c_id)
        i2c_mock.address(self.i2c_id, self.address, 1)
        local data = i2c_mock.read(self.i2c_id, length)
        i2c_mock.stop(self.i2c_id)
        
        return data
    end
    
    function sensor:read()
        -- จำลองการอ่านค่า temperature และ pressure
        local temp = 23.5 + (math.random() - 0.5) * 2
        local pressure = 1013.25 + (math.random() - 0.5) * 10
        local altitude = 44330 * (1 - (pressure/1013.25)^0.1903)
        
        return {
            temperature = math.floor(temp * 100) / 100,
            pressure = math.floor(pressure * 100) / 100,
            altitude = math.floor(altitude * 10) / 10,
        }
    end
    
    return sensor
end

print("=== I2C BMP280 Sensor ===")
i2c_mock.setup(0, 4, 5, 400)  -- id=0, SDA=D2, SCL=D1, 400kHz

local bmp280 = BMP280.new(0)
bmp280:init()

math.randomseed(123)
for i = 1, 3 do
    local data = bmp280:read()
    print(string.format("Reading %d: Temp=%.2f°C, Pressure=%.2fhPa, Alt=%.1fm",
        i, data.temperature, data.pressure, data.altitude))
end
```

## Timer Callbacks

```lua
-- Timer สำหรับ scheduled tasks บน NodeMCU

-- ใน NodeMCU จะใช้ tmr module
-- tmr.create()
-- timer:alarm(interval, mode, callback)
-- timer:stop()

-- จำลอง Timer module
local Timer = {}
Timer.__index = Timer

Timer.ALARM_SINGLE = 0
Timer.ALARM_SEMI = 1
Timer.ALARM_AUTO = 2

local active_timers = {}

function Timer.create()
    local self = setmetatable({}, Timer)
    self.id = tostring(#active_timers + 1)
    self.running = false
    self.callback = nil
    self.interval = nil
    self.mode = nil
    self.fire_count = 0
    table.insert(active_timers, self)
    return self
end

function Timer:alarm(interval_ms, mode, callback)
    self.interval = interval_ms
    self.mode = mode
    self.callback = callback
    self.running = true
    print(string.format("Timer %s: %dms, mode=%s",
        self.id, interval_ms,
        mode == Timer.ALARM_SINGLE and "SINGLE" or
        mode == Timer.ALARM_AUTO and "AUTO" or "SEMI"))
end

function Timer:stop()
    self.running = false
    print("Timer " .. self.id .. " stopped")
end

function Timer:fire()
    if self.running and self.callback then
        self.fire_count = self.fire_count + 1
        self.callback()
        
        if self.mode == Timer.ALARM_SINGLE then
            self.running = false
        end
    end
end

-- Sensor reading timer
local function setup_sensor_timers(sensor_manager)
    -- Timer 1: อ่าน sensor ทุก 5 วินาที
    local sensor_timer = Timer.create()
    sensor_timer:alarm(5000, Timer.ALARM_AUTO, function()
        local reading = sensor_manager:read()
        print(string.format("[Timer] Sensor: Temp=%.1f°C, Humid=%.1f%%",
            reading.temperature, reading.humidity))
    end)
    
    -- Timer 2: ส่งข้อมูลทุก 30 วินาที
    local upload_timer = Timer.create()
    upload_timer:alarm(30000, Timer.ALARM_AUTO, function()
        local avg = sensor_manager:get_average()
        if avg then
            print(string.format("[Timer] Upload: Avg Temp=%.1f°C, Avg Humid=%.1f%%",
                avg.temperature, avg.humidity))
        end
    end)
    
    -- Timer 3: ตรวจสอบ connectivity ทุก 60 วินาที
    local watchdog_timer = Timer.create()
    watchdog_timer:alarm(60000, Timer.ALARM_AUTO, function()
        print("[Timer] Watchdog: Checking connectivity...")
        if not wifi_mock.connected then
            print("[Timer] WiFi disconnected! Reconnecting...")
        end
    end)
    
    return {sensor_timer, upload_timer, watchdog_timer}
end

print("=== Timer System ===")
local timers = setup_sensor_timers(sensor)

-- จำลองการทำงาน timers
print("\nSimulating timer fires:")
for i, timer in ipairs(timers) do
    timer:fire()  -- จำลอง fire
end
```

## ADC (Analog to Digital Converter)

```lua
-- ADC สำหรับอ่านค่า analog เช่น LDR, potentiometer

-- ESP8266 มี ADC 1 ช่อง (A0/ADC0)
-- ค่า 0-1023 (10-bit) ที่ 0-3.3V

local adc_mock = {}

function adc_mock.read(channel)
    -- channel = 0 สำหรับ ESP8266
    -- จำลองการอ่านค่า
    return math.random(0, 1023)
end

-- LDR (Light Dependent Resistor) - วัดแสง
local function create_ldr_sensor(adc_channel, vcc_voltage)
    vcc_voltage = vcc_voltage or 3.3
    
    return {
        channel = adc_channel,
        
        read_raw = function(self)
            return adc_mock.read(self.channel)
        end,
        
        read_voltage = function(self)
            local raw = self:read_raw()
            return raw / 1023 * vcc_voltage
        end,
        
        read_lux = function(self)
            -- ค่าประมาณ (ขึ้นกับ resistor ที่ใช้)
            local voltage = self:read_voltage()
            local resistance = (vcc_voltage - voltage) / voltage * 10000  -- 10kΩ resistor
            
            -- LDR characteristics approximation
            local lux = 500 / (resistance / 1000)
            return math.max(0, math.floor(lux * 10) / 10)
        end,
        
        read_category = function(self)
            local lux = self:read_lux()
            if lux < 10 then return "DARK"
            elseif lux < 100 then return "DIM"
            elseif lux < 1000 then return "NORMAL"
            elseif lux < 10000 then return "BRIGHT"
            else return "VERY_BRIGHT"
            end
        end,
    }
end

print("=== ADC / LDR Sensor ===")
local ldr = create_ldr_sensor(0)

math.randomseed(456)
print("Light readings:")
for i = 1, 5 do
    local raw = ldr:read_raw()
    local voltage = ldr:read_voltage()
    local lux = ldr:read_lux()
    local category = ldr:read_category()
    print(string.format("  Reading %d: raw=%d, %.2fV, %.1f lux (%s)",
        i, raw, voltage, lux, category))
end

-- Soil moisture sensor
local function create_moisture_sensor(adc_channel)
    return {
        channel = adc_channel,
        
        -- Calibration values (ต้องทำการ calibrate จริง)
        DRY_VALUE = 1023,  -- ADC value in air
        WET_VALUE = 300,   -- ADC value in water
        
        read_percent = function(self)
            local raw = adc_mock.read(self.channel)
            local pct = (self.DRY_VALUE - raw) / (self.DRY_VALUE - self.WET_VALUE) * 100
            return math.max(0, math.min(100, math.floor(pct)))
        end,
        
        needs_watering = function(self)
            return self:read_percent() < 30
        end,
    }
end

local moisture = create_moisture_sensor(0)
math.randomseed(789)
print("\nSoil Moisture readings:")
for i = 1, 3 do
    local pct = moisture:read_percent()
    local needs_water = moisture:needs_watering()
    print(string.format("  Reading %d: %d%% %s",
        i, pct, needs_water and "(ต้องรดน้ำ!)" or ""))
end
```

## OTA Updates

```lua
-- OTA (Over The Air) Update - อัพเดต firmware ผ่าน WiFi

-- NodeMCU มี OTA update ผ่าน
-- 1. MQTT OTA
-- 2. HTTP OTA 
-- 3. nodemcu-uploader

local OTA = {}

function OTA.check_for_update(current_version, update_server)
    print(string.format("Checking for updates (current: %s)...", current_version))
    
    -- จำลอง check request
    local latest_version = "2.1.0"  -- จาก server
    local update_available = false
    
    -- เปรียบเทียบ version (simple string comparison)
    if latest_version > current_version then
        update_available = true
    end
    
    return {
        available = update_available,
        current = current_version,
        latest = latest_version,
        download_url = update_available and 
            string.format("%s/firmware/v%s.bin", update_server, latest_version) or nil,
    }
end

function OTA.download_and_flash(url, callback)
    print("Downloading firmware from: " .. url)
    
    -- จำลองการดาวน์โหลด
    local total_size = 512 * 1024  -- 512KB
    local downloaded = 0
    
    -- Progress callback
    while downloaded < total_size do
        downloaded = math.min(downloaded + 4096, total_size)
        local progress = math.floor(downloaded / total_size * 100)
        io.write(string.format("\r  Progress: %d%% (%d/%d bytes)", 
            progress, downloaded, total_size))
        io.flush()
    end
    print()
    
    print("Download complete! Verifying...")
    
    -- Verify checksum (จำลอง)
    local checksum_ok = true
    if checksum_ok then
        print("Checksum OK. Flashing...")
        -- ใน production จะ flash จริง และ restart
        -- node.flashreload("/firmware.bin")
        print("Flash complete! Restarting...")
        
        if callback then callback(true, "Update successful") end
    else
        if callback then callback(false, "Checksum verification failed") end
    end
end

print("=== OTA Update System ===")

local CURRENT_VERSION = "2.0.5"
local UPDATE_SERVER = "http://ota.example.com"

local update_info = OTA.check_for_update(CURRENT_VERSION, UPDATE_SERVER)
print(string.format("Current: %s, Latest: %s", update_info.current, update_info.latest))

if update_info.available then
    print("Update available! Downloading...")
    OTA.download_and_flash(update_info.download_url, function(success, message)
        print("OTA Result: " .. (success and "✓ " or "✗ ") .. message)
    end)
else
    print("Already up to date!")
end
```

## Low-power Modes

```lua
-- Power management สำหรับ battery-powered devices

local PowerManager = {}

-- Power modes
PowerManager.MODES = {
    ACTIVE = {name = "Active", current_ma = 80, wake_time_ms = 0},
    MODEM_SLEEP = {name = "Modem Sleep", current_ma = 15, wake_time_ms = 3},
    LIGHT_SLEEP = {name = "Light Sleep", current_ma = 0.9, wake_time_ms = 1},
    DEEP_SLEEP = {name = "Deep Sleep", current_ma = 0.02, wake_time_ms = 8000},
}

function PowerManager.calculate_battery_life(capacity_mah, duty_cycle)
    -- capacity_mah: battery capacity in mAh
    -- duty_cycle: {active_time_ms, sleep_time_ms, mode}
    
    local active_time = duty_cycle.active_time_ms / 1000  -- convert to hours * 3600
    local sleep_time = duty_cycle.sleep_time_ms / 1000
    local total_time = active_time + sleep_time
    
    local sleep_mode = PowerManager.MODES[duty_cycle.mode] or PowerManager.MODES.DEEP_SLEEP
    
    -- คำนวณ average current
    local avg_current = (
        PowerManager.MODES.ACTIVE.current_ma * active_time +
        sleep_mode.current_ma * sleep_time
    ) / total_time
    
    -- Battery life
    local battery_life_hours = capacity_mah / avg_current
    
    return {
        avg_current_ma = avg_current,
        battery_life_hours = battery_life_hours,
        battery_life_days = battery_life_hours / 24,
    }
end

-- Deep sleep ตั้งเวลาตื่น
function PowerManager.deep_sleep(duration_us, callback_before_sleep)
    print(string.format("Entering deep sleep for %.1f seconds...", duration_us/1000000))
    
    if callback_before_sleep then
        callback_before_sleep()
    end
    
    -- บันทึก state ก่อน sleep (RTC memory)
    local rtc_data = {
        sleep_count = 1,
        last_reading = os.time(),
        accumulated_temp = 25.5,
    }
    
    print("Saving state to RTC memory...")
    print(string.format("  sleep_count: %d", rtc_data.sleep_count))
    
    -- ใน NodeMCU จริง:
    -- node.dsleep(duration_us)
    -- node.dsleep(duration_us, 4)  -- with WiFi off during wake
    
    print("💤 Device sleeping... (would wake after " .. duration_us/1e6 .. "s)")
end

-- Sensor node power optimization
local function optimized_sensor_node()
    print("=== Optimized Sensor Node ===")
    
    -- วิเคราะห์ battery life
    local scenarios = {
        {name = "High frequency (every 30s)", active = 500, sleep = 29500, mode = "LIGHT_SLEEP"},
        {name = "Medium frequency (every 5m)", active = 500, sleep = 299500, mode = "DEEP_SLEEP"},
        {name = "Low frequency (every 30m)", active = 500, sleep = 1799500, mode = "DEEP_SLEEP"},
    }
    
    local BATTERY_CAPACITY = 2000  -- 2000mAh (AA batteries)
    
    print(string.format("\nBattery capacity: %dmAh", BATTERY_CAPACITY))
    print(string.rep("-", 70))
    print(string.format("%-35s %-15s %-15s", "Scenario", "Avg Current", "Battery Life"))
    print(string.rep("-", 70))
    
    for _, scenario in ipairs(scenarios) do
        local life = PowerManager.calculate_battery_life(BATTERY_CAPACITY, {
            active_time_ms = scenario.active,
            sleep_time_ms = scenario.sleep,
            mode = scenario.mode,
        })
        
        print(string.format("%-35s %-15s %-15s",
            scenario.name,
            string.format("%.2f mA", life.avg_current_ma),
            string.format("%.0f days", life.battery_life_days)))
    end
end

optimized_sensor_node()

-- Deep sleep example
PowerManager.deep_sleep(300 * 1000000, function()  -- 5 นาที
    print("Saving data before sleep...")
    -- บันทึกข้อมูลสำคัญ
end)
```

## Real-Time Clock

```lua
-- Real-time clock สำหรับ timestamp ที่แม่นยำ

-- NodeMCU มี rtctime module สำหรับ NTP time sync
-- rtctime.set(timestamp, usec)
-- rtctime.get()

-- จำลอง RTC module
local rtctime = {}
local rtc_offset = 0

function rtctime.set(timestamp, usec)
    rtc_offset = timestamp - os.time()
    print(string.format("RTC set to: %s", os.date("%Y-%m-%d %H:%M:%S", timestamp)))
end

function rtctime.get()
    local t = os.time() + rtc_offset
    return t, 0  -- timestamp, microseconds
end

-- NTP time sync
local sntp_mock = {}

function sntp_mock.sync(servers, success_cb, error_cb)
    servers = servers or {"pool.ntp.org", "time.nist.gov"}
    print("Syncing time with NTP...")
    print("NTP servers: " .. table.concat(servers, ", "))
    
    -- จำลอง NTP sync
    local ntp_time = os.time()  -- ใช้ system time เป็น mock
    rtctime.set(ntp_time, 0)
    
    if success_cb then
        success_cb(0, {
            offset = 0,
            delay = 50,  -- ms
            server = servers[1],
        })
    end
end

-- Scheduled task system
local ScheduledTask = {}

function ScheduledTask.new(name, cron_like, callback)
    return {
        name = name,
        schedule = cron_like,  -- {hour, minute}
        callback = callback,
        last_run = nil,
    }
end

function ScheduledTask.check_and_run(task)
    local t = os.time() + rtc_offset
    local hour = tonumber(os.date("%H", t))
    local minute = tonumber(os.date("%M", t))
    
    if hour == task.schedule.hour and minute == task.schedule.minute then
        if task.last_run ~= t // 60 then  -- ไม่ run ซ้ำในนาทีเดียวกัน
            task.last_run = t // 60
            print(string.format("Running task '%s' at %02d:%02d", 
                task.name, hour, minute))
            task.callback()
        end
    end
end

print("=== Real-Time Clock ===")

sntp_mock.sync({"pool.ntp.org"}, function(status, info)
    print(string.format("NTP sync OK: offset=%dms, server=%s",
        info.offset, info.server))
end)

local ts, us = rtctime.get()
print("Current time: " .. os.date("%Y-%m-%d %H:%M:%S", ts))

-- ตั้งเวลางาน
local daily_tasks = {
    ScheduledTask.new("midnight_report", {hour = 0, minute = 0}, function()
        print("Sending daily report...")
    end),
    ScheduledTask.new("morning_check", {hour = 8, minute = 0}, function()
        print("Morning system check...")
    end),
}
print(string.format("Scheduled %d daily tasks", #daily_tasks))
```

## Building IoT Sensor Node

```lua
-- ประกอบ IoT Sensor Node สมบูรณ์

local IoTSensorNode = {}
IoTSensorNode.__index = IoTSensorNode

function IoTSensorNode.new(config)
    local self = setmetatable({}, IoTSensorNode)
    self.config = config
    self.state = "INIT"
    self.readings_sent = 0
    self.uptime = 0
    self.start_time = os.time()
    return self
end

function IoTSensorNode:initialize()
    print("=== IoT Sensor Node Initializing ===")
    
    -- 1. ตั้งค่า GPIO
    print("1. Setting up GPIO...")
    gpio.mode(self.config.led_pin, gpio.OUTPUT)
    gpio.write(self.config.led_pin, gpio.LOW)
    
    -- 2. เชื่อมต่อ WiFi
    print("2. Connecting to WiFi...")
    wifi_mock.setmode(wifi_mock.STATION)
    wifi_mock.sta.config({
        ssid = self.config.wifi_ssid,
        pwd = self.config.wifi_password,
    })
    wifi_mock.sta.connect()
    
    -- 3. Sync time
    print("3. Syncing time...")
    sntp_mock.sync({"pool.ntp.org"}, function(status, info)
        print("   Time synced!")
    end)
    
    -- 4. เชื่อมต่อ MQTT
    print("4. Connecting to MQTT broker...")
    self.mqtt = MQTTClient.new(self.config.device_id, {
        keepalive = 60,
        username = self.config.mqtt_user,
        password = self.config.mqtt_password,
    })
    self.mqtt:connect(self.config.mqtt_broker, 1883)
    
    -- Subscribe to command topic
    self.mqtt:subscribe(
        string.format("devices/%s/commands", self.config.device_id),
        1,
        function(client, topic, payload)
            self:handle_command(payload)
        end
    )
    
    -- 5. ตั้งค่า sensors
    print("5. Initializing sensors...")
    self.sensor_manager = SensorManager.new(
        self.config.dht_pin, "DHT22", {
            alert_temp_high = self.config.temp_alert or 35,
            alert_humid_high = self.config.humid_alert or 80,
        }
    )
    
    self.state = "RUNNING"
    print("=== Initialization Complete ===\n")
end

function IoTSensorNode:handle_command(payload)
    print("Received command: " .. payload)
    
    -- Parse command (simplified)
    if payload:match('"action":"restart"') then
        print("Restarting device...")
        -- node.restart()
    elseif payload:match('"action":"update"') then
        print("Starting OTA update...")
    elseif payload:match('"action":"config"') then
        print("Updating configuration...")
    end
end

function IoTSensorNode:read_and_publish()
    if self.state ~= "RUNNING" then return end
    
    local reading = self.sensor_manager:read()
    
    if reading.status == 0 then
        local ts, _ = rtctime.get()
        
        -- สร้าง payload
        local payload = string.format(
            '{"device_id":"%s","temperature":%.1f,"humidity":%.1f,"heat_index":%.1f,"timestamp":%d}',
            self.config.device_id,
            reading.temperature,
            reading.humidity,
            reading.heat_index,
            ts
        )
        
        -- Publish to multiple topics
        self.mqtt:publish(
            string.format("sensors/%s/data", self.config.device_id),
            payload, 0, false
        )
        
        self.mqtt:publish(
            string.format("sensors/%s/temperature", self.config.device_id),
            string.format("%.1f", reading.temperature), 0, true
        )
        
        self.readings_sent = self.readings_sent + 1
        
        -- Status LED blink
        gpio.write(self.config.led_pin, gpio.HIGH)
        -- ใน production: tmr.create():alarm(100, tmr.ALARM_SINGLE, ...)
        gpio.write(self.config.led_pin, gpio.LOW)
    end
end

function IoTSensorNode:get_status()
    return {
        state = self.state,
        uptime = os.time() - self.start_time,
        readings_sent = self.readings_sent,
        wifi_connected = wifi_mock.connected,
        mqtt_connected = self.mqtt and self.mqtt.connected,
        free_heap = get_memory_info().free_heap,
    }
end

-- รัน sensor node
local node = IoTSensorNode.new({
    device_id = "esp8266-home-sensor",
    wifi_ssid = "HomeNetwork",
    wifi_password = "password123",
    mqtt_broker = "mqtt.example.com",
    mqtt_user = "device_user",
    mqtt_password = "device_pass",
    led_pin = 2,  -- D4
    dht_pin = 4,  -- D2
    temp_alert = 35,
    humid_alert = 80,
    read_interval = 5000,  -- 5 seconds
})

node:initialize()

-- จำลองการทำงาน
math.randomseed(999)
print("Running sensor node for 5 readings:")
for i = 1, 5 do
    node:read_and_publish()
    print(string.format("  Reading %d sent", i))
end

local status = node:get_status()
print("\nNode Status:")
for k, v in pairs(status) do
    print(string.format("  %-20s: %s", k, tostring(v)))
end
```

## สรุปบทที่ 73

```lua
-- สรุป IoT กับ NodeMCU/ESP8266

local topics = {
    "NodeMCU Platform - ESP8266 specs และ capabilities",
    "Memory Constraints - การจัดการ memory บน MCU",
    "Flash Storage (SPIFFS) - file system บน flash",
    "WiFi Connection - การเชื่อมต่อ wireless",
    "HTTP Client - ส่งข้อมูลไปยัง server",
    "MQTT Protocol - lightweight messaging",
    "DHT11/22 Sensor - temperature & humidity",
    "GPIO Control - digital input/output",
    "PWM - LED dimming, servo control",
    "I2C Interface - sensor communication",
    "ADC - analog sensor reading",
    "Timer Callbacks - scheduled tasks",
    "OTA Updates - firmware update over WiFi",
    "Low-power Modes - battery optimization",
    "Real-Time Clock - NTP time sync",
    "IoT Sensor Node - complete implementation",
}

print("=" .. string.rep("=", 55))
print("บทที่ 73: IoT กับ NodeMCU/ESP8266")
print("=" .. string.rep("=", 55))
for i, topic in ipairs(topics) do
    print(string.format("  %2d. %s", i, topic))
end

print("\nKey takeaways:")
print("  * ESP8266 มี RAM เพียง ~50KB ต้องจัดการ memory อย่างระมัดระวัง")
print("  * MQTT เหมาะสำหรับ IoT เพราะ lightweight และรองรับ pub/sub")
print("  * Deep sleep ช่วยยืด battery life ได้อย่างมาก")
print("  * SPIFFS สำหรับเก็บ config และ log data")
print("  * OTA updates สำคัญสำหรับ deployed devices")
```
