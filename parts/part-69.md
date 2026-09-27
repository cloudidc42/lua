# บทที่ 69: Distributed Tracing

## บทนำ

ในระบบ Microservices ที่ซับซ้อน การติดตามว่า Request หนึ่งๆ ไหลผ่าน Service ต่างๆ อย่างไรนั้นเป็นสิ่งสำคัญมาก **Distributed Tracing** คือเทคนิคที่ช่วยให้เราสามารถมองเห็นเส้นทางของ Request ตั้งแต่ต้นจนจบ ระบุ Bottleneck และ Debug ปัญหาที่เกิดขึ้นใน Production ได้อย่างมีประสิทธิภาพ

ในบทนี้เราจะเรียนรู้การนำ Distributed Tracing มาใช้ใน OpenResty/Lua โดยใช้มาตรฐาน OpenTelemetry และการส่งข้อมูลไปยัง Zipkin และ Jaeger

---

## 69.1 OpenTelemetry Concepts

### 69.1.1 Trace, Span และ Context

**OpenTelemetry** กำหนดแนวคิดหลักสามประการ:

- **Trace**: การเดินทางทั้งหมดของ Request หนึ่งๆ ผ่านระบบ ประกอบด้วย Span หลายอัน
- **Span**: หน่วยงานย่อยของ Trace แต่ละ Span แทนการทำงานหนึ่งอย่าง เช่น HTTP Request, Database Query
- **Context**: ข้อมูลที่ถ่ายทอดระหว่าง Span เพื่อให้รู้ว่า Span ใดเป็น Parent/Child

```lua
-- โครงสร้างพื้นฐานของ Span
local span = {
    trace_id   = "4bf92f3577b34da6a3ce929d0e0e4736",  -- 128-bit hex
    span_id    = "00f067aa0ba902b7",                   -- 64-bit hex
    parent_id  = nil,                                   -- nil = root span
    name       = "HTTP GET /api/users",
    start_time = ngx.now() * 1000,                      -- microseconds
    end_time   = nil,
    status     = "OK",    -- OK, ERROR, UNSET
    attributes = {},
    events     = {},
}
```

### 69.1.2 Trace ID Generation

Trace ID ต้องมีความ Unique สูงมาก โดยทั่วไปใช้ 128-bit Random Number

```lua
-- ฟังก์ชันสร้าง Trace ID (128-bit = 32 hex chars)
local function generate_trace_id()
    local bytes = {}
    for i = 1, 16 do
        bytes[i] = math.random(0, 255)
    end
    return string.format(
        "%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x",
        table.unpack(bytes)
    )
end

-- ฟังก์ชันสร้าง Span ID (64-bit = 16 hex chars)
local function generate_span_id()
    local bytes = {}
    for i = 1, 8 do
        bytes[i] = math.random(0, 255)
    end
    return string.format(
        "%02x%02x%02x%02x%02x%02x%02x%02x",
        table.unpack(bytes)
    )
end

-- ใช้ OpenResty random bytes เพื่อความปลอดภัยกว่า
local function generate_id_secure(bytes_count)
    local random_bytes = require("resty.random").bytes(bytes_count)
    local hex = require("resty.string").to_hex(random_bytes)
    return hex
end
```

### 69.1.3 Context Propagation

Context Propagation คือกลไกที่ทำให้ Trace สามารถข้ามไปยัง Service อื่นได้

```lua
-- Context object สำหรับ propagation
local TraceContext = {}
TraceContext.__index = TraceContext

function TraceContext.new(trace_id, span_id, flags)
    return setmetatable({
        trace_id  = trace_id or generate_trace_id(),
        span_id   = span_id or generate_span_id(),
        flags     = flags or 1,  -- 1 = sampled
        baggage   = {},
    }, TraceContext)
end

function TraceContext:set_baggage(key, value)
    self.baggage[key] = value
end

function TraceContext:get_baggage(key)
    return self.baggage[key]
end

-- Convert to W3C traceparent format
function TraceContext:to_traceparent()
    return string.format("00-%s-%s-%02x",
        self.trace_id,
        self.span_id,
        self.flags
    )
end
```

---

## 69.2 W3C Traceparent Header

### 69.2.1 รูปแบบ Traceparent

W3C Trace Context specification กำหนดรูปแบบ Header ดังนี้:

```
traceparent: 00-{trace-id}-{parent-id}-{trace-flags}
```

- **version**: `00` (ปัจจุบัน)
- **trace-id**: 32 hex chars (128-bit)
- **parent-id**: 16 hex chars (64-bit)
- **trace-flags**: 2 hex chars (bit flags, `01` = sampled)

```lua
-- Parse W3C traceparent header
local function parse_traceparent(header)
    if not header then return nil end
    
    local version, trace_id, parent_id, flags =
        header:match("^(%x%x)-(%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x)-(%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x)-(%x%x)$")
    
    if not trace_id then
        return nil, "invalid traceparent format"
    end
    
    return {
        version   = version,
        trace_id  = trace_id,
        parent_id = parent_id,
        flags     = tonumber(flags, 16),
        sampled   = (tonumber(flags, 16) & 1) == 1,
    }
end

-- สร้าง traceparent header ใหม่
local function create_traceparent(trace_id, span_id, sampled)
    local flags = sampled and "01" or "00"
    return string.format("00-%s-%s-%s", trace_id, span_id, flags)
end
```

### 69.2.2 Tracestate Header

```lua
-- Parse tracestate header (vendor-specific key=value pairs)
local function parse_tracestate(header)
    if not header then return {} end
    
    local state = {}
    for entry in header:gmatch("[^,]+") do
        local key, value = entry:match("^%s*([^=]+)=(.+)%s*$")
        if key and value then
            state[key:match("^%s*(.-)%s*$")] = value:match("^%s*(.-)%s*$")
        end
    end
    return state
end

-- สร้าง tracestate header
local function build_tracestate(vendor, span_id, existing_state)
    local new_entry = string.format("%s=%s", vendor, span_id)
    
    if not existing_state or #existing_state == 0 then
        return new_entry
    end
    
    -- เพิ่ม vendor ใหม่ที่ต้นรายการ
    return new_entry .. "," .. existing_state
end
```

---

## 69.3 X-Trace-ID Header (Simple Correlation)

บางระบบใช้ Header ที่เรียบง่ายกว่า W3C เช่น `X-Trace-ID` หรือ `X-Request-ID`

```lua
-- Middleware สำหรับ X-Trace-ID
local function trace_id_middleware()
    local headers = ngx.req.get_headers()
    
    -- รับ Trace ID จาก Header หรือสร้างใหม่
    local trace_id = headers["x-trace-id"] 
                  or headers["x-request-id"]
                  or headers["x-correlation-id"]
                  or generate_id_secure(16)
    
    -- เก็บใน ngx.ctx เพื่อใช้ใน Request นี้
    ngx.ctx.trace_id = trace_id
    
    -- ส่ง Trace ID ใน Response Header ด้วย
    ngx.header["X-Trace-ID"] = trace_id
    
    -- Log เพื่อ correlation
    ngx.log(ngx.INFO, "trace_id=", trace_id, " method=", ngx.req.get_method(),
            " uri=", ngx.var.uri)
end
```

---

## 69.4 Implementing Tracing Middleware ใน OpenResty

### 69.4.1 โครงสร้าง Tracing Library

```lua
-- lib/tracing.lua
local _M = {
    _VERSION = "1.0.0"
}

local mt = { __index = _M }

-- สร้าง Tracer instance ใหม่
function _M.new(config)
    local self = setmetatable({}, mt)
    
    self.service_name = config.service_name or "unknown-service"
    self.sample_rate  = config.sample_rate or 1.0   -- 100% by default
    self.exporter     = config.exporter            -- Zipkin/Jaeger exporter
    self.propagator   = config.propagator or "w3c"  -- w3c or b3
    
    return self
end

-- เริ่ม Root Span จาก incoming request
function _M:start_request_span(name)
    local headers = ngx.req.get_headers()
    local ctx
    
    if self.propagator == "w3c" then
        ctx = self:extract_w3c(headers)
    else
        ctx = self:extract_b3(headers)
    end
    
    local span = self:new_span(name, ctx)
    ngx.ctx.current_span = span
    ngx.ctx.tracer = self
    
    return span
end

return _M
```

### 69.4.2 Span Implementation

```lua
-- lib/span.lua
local Span = {}
Span.__index = Span

function Span.new(tracer, name, parent_ctx)
    local now_ms = ngx.now() * 1000  -- milliseconds
    
    local trace_id, parent_span_id
    
    if parent_ctx then
        trace_id      = parent_ctx.trace_id
        parent_span_id = parent_ctx.span_id
    else
        trace_id = generate_id_secure(16)
    end
    
    return setmetatable({
        tracer       = tracer,
        name         = name,
        trace_id     = trace_id,
        span_id      = generate_id_secure(8),
        parent_id    = parent_span_id,
        start_time   = now_ms,
        end_time     = nil,
        status_code  = "UNSET",
        status_msg   = nil,
        attributes   = {},
        events       = {},
        finished     = false,
    }, Span)
end

-- เพิ่ม Attribute
function Span:set_attribute(key, value)
    self.attributes[key] = value
    return self  -- method chaining
end

-- เพิ่ม Event (timestamp + message)
function Span:add_event(name, attrs)
    table.insert(self.events, {
        name       = name,
        timestamp  = ngx.now() * 1000,
        attributes = attrs or {},
    })
    return self
end

-- ตั้ง Error status
function Span:record_error(err)
    self.status_code = "ERROR"
    self.status_msg  = tostring(err)
    self:set_attribute("error", true)
    self:set_attribute("error.message", tostring(err))
    self:add_event("exception", {
        ["exception.message"] = tostring(err),
        ["exception.type"]    = type(err),
    })
    return self
end

-- จบ Span
function Span:finish(status)
    if self.finished then return end
    
    self.end_time = ngx.now() * 1000
    self.finished = true
    
    if status then
        self.status_code = status
    elseif self.status_code == "UNSET" then
        self.status_code = "OK"
    end
    
    -- Export span
    if self.tracer and self.tracer.exporter then
        self.tracer.exporter:export(self)
    end
end

-- Duration ใน milliseconds
function Span:duration_ms()
    if self.end_time then
        return self.end_time - self.start_time
    end
    return ngx.now() * 1000 - self.start_time
end
```

### 69.4.3 Integration กับ OpenResty Phases

```lua
-- nginx.conf / access_by_lua_block
local tracing = require("lib.tracing")
local tracer = tracing.new({
    service_name = "api-gateway",
    sample_rate  = 0.1,  -- 10% sampling
    exporter     = require("lib.zipkin_exporter").new({
        endpoint = "http://zipkin:9411/api/v2/spans"
    }),
})

-- access phase: เริ่ม span
local span = tracer:start_request_span(
    ngx.req.get_method() .. " " .. ngx.var.uri
)

span:set_attribute("http.method",     ngx.req.get_method())
span:set_attribute("http.url",        ngx.var.scheme .. "://" .. ngx.var.host .. ngx.var.request_uri)
span:set_attribute("http.target",     ngx.var.uri)
span:set_attribute("net.peer.ip",     ngx.var.remote_addr)
span:set_attribute("service.name",    "api-gateway")
```

```lua
-- log phase: จบ span
-- log_by_lua_block
local span = ngx.ctx.current_span
if span then
    local status = ngx.status
    span:set_attribute("http.status_code", status)
    
    if status >= 500 then
        span:record_error("HTTP " .. status)
    elseif status >= 400 then
        span:set_attribute("http.error", true)
    end
    
    span:finish()
end
```

---

## 69.5 Span Timing และ Attributes

### 69.5.1 Semantic Conventions

OpenTelemetry กำหนด Attribute names มาตรฐานที่ทุกคนควรใช้

```lua
-- Semantic convention attributes
local SemanticAttributes = {
    -- HTTP
    HTTP_METHOD       = "http.method",
    HTTP_URL          = "http.url",
    HTTP_STATUS_CODE  = "http.status_code",
    HTTP_USER_AGENT   = "http.user_agent",
    HTTP_REQUEST_BODY_SIZE  = "http.request_content_length",
    HTTP_RESPONSE_BODY_SIZE = "http.response_content_length",
    
    -- Network
    NET_PEER_IP   = "net.peer.ip",
    NET_PEER_PORT = "net.peer.port",
    NET_HOST_NAME = "net.host.name",
    
    -- Database
    DB_SYSTEM    = "db.system",
    DB_NAME      = "db.name",
    DB_STATEMENT = "db.statement",
    DB_OPERATION = "db.operation",
    
    -- Messaging
    MESSAGING_SYSTEM     = "messaging.system",
    MESSAGING_DESTINATION = "messaging.destination",
    MESSAGING_OPERATION  = "messaging.operation",
    
    -- RPC
    RPC_SYSTEM  = "rpc.system",
    RPC_SERVICE = "rpc.service",
    RPC_METHOD  = "rpc.method",
}

-- ตัวอย่างการใช้งาน
local function instrument_db_query(span, query, db_name)
    span:set_attribute(SemanticAttributes.DB_SYSTEM,    "mysql")
    span:set_attribute(SemanticAttributes.DB_NAME,      db_name)
    span:set_attribute(SemanticAttributes.DB_STATEMENT, query)
    span:set_attribute(SemanticAttributes.DB_OPERATION, "SELECT")
end
```

### 69.5.2 Child Span สำหรับ Database Calls

```lua
-- สร้าง Child Span สำหรับ Database Operation
local function query_with_tracing(parent_span, sql, params)
    local span = Span.new(parent_span.tracer, "db.query", {
        trace_id = parent_span.trace_id,
        span_id  = parent_span.span_id,
    })
    
    span:set_attribute("db.system",    "mysql")
    span:set_attribute("db.statement", sql)
    
    -- Execute query
    local ok, result, err = pcall(function()
        local db = require("resty.mysql")
        local conn = db:new()
        -- ... connection setup ...
        return conn:query(sql, params)
    end)
    
    if not ok then
        span:record_error(result)
        span:finish("ERROR")
        return nil, result
    end
    
    if err then
        span:record_error(err)
        span:finish("ERROR")
        return nil, err
    end
    
    span:set_attribute("db.rows_affected", result.affected_rows or 0)
    span:finish("OK")
    
    return result, nil
end
```

---

## 69.6 Zipkin Integration

### 69.6.1 Zipkin Exporter

Zipkin ใช้ JSON format ในการรับ Span data

```lua
-- lib/zipkin_exporter.lua
local http = require("resty.http")
local cjson = require("cjson.safe")

local ZipkinExporter = {}
ZipkinExporter.__index = ZipkinExporter

function ZipkinExporter.new(config)
    return setmetatable({
        endpoint     = config.endpoint or "http://localhost:9411/api/v2/spans",
        timeout_ms   = config.timeout_ms or 5000,
        batch_size   = config.batch_size or 100,
        _buffer      = {},
    }, ZipkinExporter)
end

-- แปลง Span เป็น Zipkin format
function ZipkinExporter:span_to_zipkin(span)
    local zipkin_span = {
        traceId       = span.trace_id,
        id            = span.span_id,
        name          = span.name,
        timestamp     = math.floor(span.start_time * 1000),  -- microseconds
        duration      = math.floor((span.end_time - span.start_time) * 1000),
        localEndpoint = {
            serviceName = span.tracer.service_name,
            ipv4        = ngx.var.server_addr,
            port        = tonumber(ngx.var.server_port),
        },
        tags = {},
        annotations = {},
    }
    
    -- Parent span
    if span.parent_id then
        zipkin_span.parentId = span.parent_id
    end
    
    -- Attributes -> Tags
    for k, v in pairs(span.attributes) do
        zipkin_span.tags[k] = tostring(v)
    end
    
    -- Status
    if span.status_code == "ERROR" then
        zipkin_span.tags["error"] = span.status_msg or "true"
    end
    
    -- Events -> Annotations
    for _, event in ipairs(span.events) do
        table.insert(zipkin_span.annotations, {
            timestamp = math.floor(event.timestamp * 1000),
            value     = event.name,
        })
    end
    
    return zipkin_span
end

-- ส่ง Spans ไปยัง Zipkin
function ZipkinExporter:send_batch(spans)
    local zipkin_spans = {}
    for _, span in ipairs(spans) do
        table.insert(zipkin_spans, self:span_to_zipkin(span))
    end
    
    local body, err = cjson.encode(zipkin_spans)
    if not body then
        ngx.log(ngx.ERR, "failed to encode spans: ", err)
        return false
    end
    
    local httpc = http.new()
    httpc:set_timeout(self.timeout_ms)
    
    local res, err = httpc:request_uri(self.endpoint, {
        method  = "POST",
        body    = body,
        headers = {
            ["Content-Type"] = "application/json",
        },
    })
    
    if not res then
        ngx.log(ngx.ERR, "failed to send spans to Zipkin: ", err)
        return false
    end
    
    if res.status ~= 202 then
        ngx.log(ngx.WARN, "Zipkin returned status: ", res.status)
        return false
    end
    
    return true
end

function ZipkinExporter:export(span)
    table.insert(self._buffer, span)
    
    if #self._buffer >= self.batch_size then
        self:flush()
    end
end

function ZipkinExporter:flush()
    if #self._buffer == 0 then return end
    
    local batch = self._buffer
    self._buffer = {}
    
    -- ส่งใน background (ไม่ block request)
    local ok = ngx.timer.at(0, function()
        self:send_batch(batch)
    end)
    
    if not ok then
        ngx.log(ngx.ERR, "failed to create timer for Zipkin export")
    end
end
```

---

## 69.7 Jaeger Integration

### 69.7.1 Jaeger UDP Exporter (Thrift format)

```lua
-- lib/jaeger_exporter.lua
-- Jaeger รับข้อมูลผ่าน UDP ด้วย Thrift binary protocol
-- หรือผ่าน HTTP Collector

local JaegerExporter = {}
JaegerExporter.__index = JaegerExporter

function JaegerExporter.new(config)
    return setmetatable({
        -- HTTP Collector endpoint (ง่ายกว่า UDP)
        endpoint     = config.endpoint or "http://localhost:14268/api/traces",
        service_name = config.service_name or "unknown",
        timeout_ms   = config.timeout_ms or 5000,
    }, JaegerExporter)
end

-- Jaeger รองรับ OpenTelemetry Protocol (OTLP) ด้วย
-- แนะนำให้ใช้ OTLP HTTP แทน
function JaegerExporter:span_to_otlp(span)
    -- OTLP format (JSON)
    return {
        traceId    = span.trace_id,
        spanId     = span.span_id,
        parentSpanId = span.parent_id,
        name       = span.name,
        kind       = 2,  -- SPAN_KIND_SERVER
        startTimeUnixNano = tostring(math.floor(span.start_time * 1e6)),
        endTimeUnixNano   = tostring(math.floor(span.end_time * 1e6)),
        attributes = self:convert_attributes(span.attributes),
        status = {
            code    = span.status_code == "ERROR" and 2 or 1,
            message = span.status_msg or "",
        },
        events = span.events,
    }
end

function JaegerExporter:convert_attributes(attrs)
    local result = {}
    for k, v in pairs(attrs) do
        local attr = { key = k }
        if type(v) == "boolean" then
            attr.value = { boolValue = v }
        elseif type(v) == "number" then
            if math.floor(v) == v then
                attr.value = { intValue = tostring(v) }
            else
                attr.value = { doubleValue = v }
            end
        else
            attr.value = { stringValue = tostring(v) }
        end
        table.insert(result, attr)
    end
    return result
end
```

---

## 69.8 Sampling Strategies

### 69.8.1 ประเภทของ Sampling

Sampling ช่วยลดปริมาณข้อมูล Trace ที่ต้องเก็บและส่ง

```lua
-- lib/sampler.lua
local Sampler = {}
Sampler.__index = Sampler

-- 1. Always Sample
function Sampler.always_on()
    return { should_sample = function() return true end }
end

-- 2. Never Sample (ใช้ใน dev)
function Sampler.always_off()
    return { should_sample = function() return false end }
end

-- 3. Trace ID Ratio Sampling
function Sampler.ratio(rate)
    assert(rate >= 0 and rate <= 1, "sample rate must be between 0 and 1")
    
    return {
        should_sample = function(trace_id)
            -- ใช้ trace_id เพื่อให้ consistent sampling
            -- (ถ้า sampled ที่ Service A ต้องได้ sampled ที่ Service B ด้วย)
            local id_int = tonumber(trace_id:sub(1, 15), 16)
            local threshold = math.floor(rate * 0xFFFFFFFFFFFFFF)
            return id_int < threshold
        end
    }
end

-- 4. Rate Limiting Sampler
function Sampler.rate_limit(max_per_second)
    local count = 0
    local last_reset = ngx.now()
    
    return {
        should_sample = function()
            local now = ngx.now()
            if now - last_reset >= 1.0 then
                count = 0
                last_reset = now
            end
            
            if count < max_per_second then
                count = count + 1
                return true
            end
            return false
        end
    }
end

-- 5. Parent-based Sampling (ปฏิบัติตามการตัดสินใจของ Parent)
function Sampler.parent_based(root_sampler)
    return {
        should_sample = function(trace_id, parent_sampled)
            if parent_sampled ~= nil then
                return parent_sampled  -- เชื่อฟัง parent
            end
            return root_sampler.should_sample(trace_id)
        end
    }
end
```

### 69.8.2 Adaptive Sampling

```lua
-- Adaptive Sampler: ปรับ Rate ตาม Error Rate
local AdaptiveSampler = {}
AdaptiveSampler.__index = AdaptiveSampler

function AdaptiveSampler.new(config)
    return setmetatable({
        base_rate     = config.base_rate or 0.1,
        error_rate    = config.error_rate or 1.0,  -- sample all errors
        window_size   = config.window_size or 60,  -- seconds
        _error_count  = 0,
        _total_count  = 0,
        _window_start = ngx.now(),
    }, AdaptiveSampler)
end

function AdaptiveSampler:should_sample(is_error)
    local now = ngx.now()
    
    -- Reset window
    if now - self._window_start > self.window_size then
        self._error_count = 0
        self._total_count = 0
        self._window_start = now
    end
    
    self._total_count = self._total_count + 1
    
    -- Always sample errors
    if is_error then
        self._error_count = self._error_count + 1
        return true
    end
    
    -- ถ้า error rate สูง เพิ่ม sampling rate
    local current_error_rate = self._total_count > 0
        and (self._error_count / self._total_count)
        or 0
    
    local effective_rate = self.base_rate
    if current_error_rate > 0.05 then  -- > 5% error rate
        effective_rate = math.min(1.0, self.base_rate * 5)
    end
    
    return math.random() < effective_rate
end
```

---

## 69.9 Correlation ID Patterns

### 69.9.1 Correlation ID ใน Log

```lua
-- Correlation ID middleware
local function setup_correlation()
    local headers = ngx.req.get_headers()
    
    -- สนับสนุนหลาย Header formats
    local correlation_id = 
        headers["x-correlation-id"] or
        headers["x-request-id"] or
        headers["x-trace-id"] or
        generate_id_secure(16)
    
    -- เก็บใน ngx.ctx
    ngx.ctx.correlation_id = correlation_id
    ngx.ctx.request_id = generate_id_secure(8)  -- unique ต่อ request นี้
    
    -- ส่งกลับผ่าน response
    ngx.header["X-Correlation-ID"] = correlation_id
    ngx.header["X-Request-ID"] = ngx.ctx.request_id
    
    return correlation_id
end

-- Structured logging พร้อม correlation
local function log_with_context(level, msg, data)
    local ctx = ngx.ctx
    local log_entry = {
        timestamp      = ngx.now(),
        level          = level,
        message        = msg,
        correlation_id = ctx.correlation_id,
        request_id     = ctx.request_id,
        trace_id       = ctx.trace_id,
        span_id        = ctx.span_id,
        service        = "api-gateway",
        host           = ngx.var.hostname,
    }
    
    if data then
        for k, v in pairs(data) do
            log_entry[k] = v
        end
    end
    
    local cjson = require("cjson.safe")
    ngx.log(ngx[level], cjson.encode(log_entry))
end
```

---

## 69.10 Parent/Child Span Relationships

### 69.10.1 Span Tree

```lua
-- SpanContext สำหรับ thread-safe context passing
local SpanContext = {}
SpanContext.__index = SpanContext

function SpanContext.from_span(span)
    return setmetatable({
        trace_id  = span.trace_id,
        span_id   = span.span_id,
        sampled   = span.sampled,
        baggage   = span.baggage or {},
    }, SpanContext)
end

-- สร้าง child span จาก parent
local function new_child_span(parent_span, operation_name)
    local child = Span.new(parent_span.tracer, operation_name, {
        trace_id = parent_span.trace_id,
        span_id  = parent_span.span_id,
    })
    
    -- inherit baggage
    child.baggage = {}
    if parent_span.baggage then
        for k, v in pairs(parent_span.baggage) do
            child.baggage[k] = v
        end
    end
    
    return child
end

-- ตัวอย่าง: ติดตาม HTTP call ไปยัง upstream service
local function trace_upstream_call(parent_span, url, options)
    local child = new_child_span(parent_span, "http.client " .. url)
    
    child:set_attribute("http.method", options.method or "GET")
    child:set_attribute("http.url", url)
    child:set_attribute("span.kind", "CLIENT")
    
    -- Inject trace context ใน outgoing headers
    local headers = options.headers or {}
    headers["traceparent"] = create_traceparent(
        child.trace_id,
        child.span_id,
        true
    )
    headers["X-Trace-ID"] = child.trace_id
    
    options.headers = headers
    
    -- ทำ HTTP call
    local httpc = require("resty.http").new()
    local res, err = httpc:request_uri(url, options)
    
    if not res then
        child:record_error(err or "connection failed")
        child:finish("ERROR")
        return nil, err
    end
    
    child:set_attribute("http.status_code", res.status)
    
    if res.status >= 500 then
        child:record_error("upstream error: " .. res.status)
        child:finish("ERROR")
    else
        child:finish("OK")
    end
    
    return res, nil
end
```

---

## 69.11 Error Tracking ใน Spans

### 69.11.1 Error Recording Patterns

```lua
-- Error tracking helper
local function with_span(tracer, name, attrs, fn)
    local span = tracer:new_span(name)
    
    if attrs then
        for k, v in pairs(attrs) do
            span:set_attribute(k, v)
        end
    end
    
    local ok, result, err = pcall(fn, span)
    
    if not ok then
        -- Lua error (exception)
        span:record_error(result)
        span:add_event("exception", {
            ["exception.message"]    = tostring(result),
            ["exception.stacktrace"] = debug.traceback(),
        })
        span:finish("ERROR")
        return nil, result
    end
    
    if err then
        -- Application error
        span:record_error(err)
        span:finish("ERROR")
        return result, err
    end
    
    span:finish("OK")
    return result, nil
end

-- ตัวอย่างการใช้งาน
local result, err = with_span(tracer, "process_payment", {
    ["payment.method"] = "credit_card",
    ["payment.amount"] = 1000,
}, function(span)
    -- validate
    span:add_event("validation_started")
    local ok = validate_payment(data)
    if not ok then
        return nil, "invalid payment data"
    end
    span:add_event("validation_passed")
    
    -- charge
    span:add_event("charge_started")
    local charge_result = charge_card(data)
    span:set_attribute("payment.transaction_id", charge_result.id)
    span:add_event("charge_completed")
    
    return charge_result
end)
```

---

## 69.12 Complete Tracing Library Implementation

นี่คือ Implementation ที่สมบูรณ์และพร้อมใช้งาน Production:

```lua
-- lib/otel.lua - OpenTelemetry-compatible tracing library for OpenResty
local cjson    = require("cjson.safe")
local http     = require("resty.http")
local resty_random = require("resty.random")
local resty_str    = require("resty.string")

local _M = { _VERSION = "2.0.0" }

-- ==================== Utilities ====================

local function random_hex(bytes)
    return resty_str.to_hex(resty_random.bytes(bytes))
end

local function now_ms()
    return math.floor(ngx.now() * 1000)
end

-- ==================== Span ====================

local Span = {}
Span.__index = Span

function Span.new(opts)
    return setmetatable({
        trace_id   = opts.trace_id or random_hex(16),
        span_id    = opts.span_id or random_hex(8),
        parent_id  = opts.parent_id,
        name       = opts.name or "unnamed",
        kind       = opts.kind or "SERVER",
        start_ms   = now_ms(),
        end_ms     = nil,
        status     = "UNSET",
        status_msg = nil,
        attrs      = {},
        events     = {},
        links      = {},
        _tracer    = opts.tracer,
    }, Span)
end

function Span:attr(key, value)
    self.attrs[key] = value
    return self
end

function Span:event(name, timestamp, attrs)
    table.insert(self.events, {
        name  = name,
        ts    = timestamp or now_ms(),
        attrs = attrs or {},
    })
    return self
end

function Span:error(msg, stack)
    self.status    = "ERROR"
    self.status_msg = msg
    self:attr("error", true)
    self:attr("error.message", tostring(msg))
    if stack then
        self:attr("error.stack", stack)
    end
    self:event("exception", nil, {
        ["exception.message"] = tostring(msg),
        ["exception.stacktrace"] = stack or debug.traceback(2),
    })
    return self
end

function Span:ok()
    if self.status == "UNSET" then
        self.status = "OK"
    end
    return self
end

function Span:finish()
    if self.end_ms then return self end
    self.end_ms = now_ms()
    if self.status == "UNSET" then self.status = "OK" end
    if self._tracer then
        self._tracer:_export(self)
    end
    return self
end

function Span:duration()
    return (self.end_ms or now_ms()) - self.start_ms
end

function Span:traceparent()
    local flags = "01"
    return string.format("00-%s-%s-%s", self.trace_id, self.span_id, flags)
end

-- ==================== Tracer ====================

local Tracer = {}
Tracer.__index = Tracer

function Tracer.new(opts)
    return setmetatable({
        service      = opts.service or "unknown",
        version      = opts.version,
        exporter     = opts.exporter,
        sampler      = opts.sampler or { should_sample = function() return true end },
        propagator   = opts.propagator or "w3c",
        _spans_sent  = 0,
    }, Tracer)
end

function Tracer:extract(headers)
    if self.propagator == "b3" then
        return self:_extract_b3(headers)
    end
    return self:_extract_w3c(headers)
end

function Tracer:_extract_w3c(headers)
    local tp = headers["traceparent"]
    if not tp then return nil end
    
    local ver, tid, pid, flags = tp:match(
        "^(%x%x)-(%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x)-(%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x)-(%x%x)$"
    )
    if not tid then return nil end
    
    return {
        trace_id = tid,
        span_id  = pid,
        sampled  = (tonumber(flags, 16) & 1) == 1,
    }
end

function Tracer:_extract_b3(headers)
    local single = headers["b3"]
    if single then
        if single == "0" then return { sampled = false } end
        local tid, sid, sampled = single:match("^(%x+)-(%x+)-?([01]?)$")
        if tid then
            return {
                trace_id = tid,
                span_id  = sid,
                sampled  = sampled ~= "0",
            }
        end
    end
    
    return {
        trace_id = headers["x-b3-traceid"],
        span_id  = headers["x-b3-spanid"],
        sampled  = headers["x-b3-sampled"] ~= "0",
    }
end

function Tracer:inject(span, headers)
    headers["traceparent"] = span:traceparent()
    headers["X-Trace-ID"]  = span.trace_id
    if self.propagator == "b3" then
        headers["X-B3-TraceId"] = span.trace_id
        headers["X-B3-SpanId"]  = span.span_id
        headers["X-B3-Sampled"] = "1"
    end
    return headers
end

function Tracer:start(name, opts)
    opts = opts or {}
    local parent = opts.parent or ngx.ctx._current_span
    
    local span = Span.new({
        name      = name,
        trace_id  = parent and parent.trace_id or (opts.trace_id or random_hex(16)),
        parent_id = parent and parent.span_id or opts.parent_id,
        kind      = opts.kind or "INTERNAL",
        tracer    = self,
    })
    
    -- Set standard resource attributes
    span:attr("service.name",    self.service)
    if self.version then
        span:attr("service.version", self.version)
    end
    
    ngx.ctx._current_span = span
    return span
end

function Tracer:start_server_span(name)
    local incoming = self:extract(ngx.req.get_headers())
    
    local span = Span.new({
        name      = name,
        trace_id  = incoming and incoming.trace_id or random_hex(16),
        parent_id = incoming and incoming.span_id,
        kind      = "SERVER",
        tracer    = self,
    })
    
    span:attr("service.name", self.service)
    ngx.ctx._trace_id      = span.trace_id
    ngx.ctx._current_span  = span
    
    -- Propagate in response
    ngx.header["X-Trace-ID"] = span.trace_id
    
    return span
end

function Tracer:_export(span)
    if not self.exporter then return end
    self._spans_sent = self._spans_sent + 1
    
    -- Non-blocking export
    local ok, err = ngx.timer.at(0, function()
        local success, export_err = pcall(function()
            self.exporter:export({ span })
        end)
        if not success then
            ngx.log(ngx.ERR, "trace export error: ", export_err)
        end
    end)
    
    if not ok then
        ngx.log(ngx.ERR, "failed to schedule trace export: ", err)
    end
end

-- ==================== Zipkin Exporter ====================

local ZipkinExporter = {}
ZipkinExporter.__index = ZipkinExporter

function ZipkinExporter.new(opts)
    return setmetatable({
        url     = opts.url or "http://localhost:9411/api/v2/spans",
        timeout = opts.timeout or 3000,
    }, ZipkinExporter)
end

function ZipkinExporter:export(spans)
    local payload = {}
    for _, s in ipairs(spans) do
        table.insert(payload, {
            traceId       = s.trace_id,
            id            = s.span_id,
            parentId      = s.parent_id,
            name          = s.name,
            timestamp     = s.start_ms * 1000,
            duration      = s:duration() * 1000,
            localEndpoint = { serviceName = s.attrs["service.name"] or "unknown" },
            tags          = (function()
                local t = {}
                for k, v in pairs(s.attrs) do t[k] = tostring(v) end
                if s.status == "ERROR" then t["error"] = s.status_msg or "true" end
                return t
            end)(),
        })
    end
    
    local body = cjson.encode(payload)
    local c = http.new()
    c:set_timeout(self.timeout)
    local res, err = c:request_uri(self.url, {
        method  = "POST",
        body    = body,
        headers = { ["Content-Type"] = "application/json" },
    })
    
    return res and res.status == 202, err
end

-- ==================== Module exports ====================

_M.Tracer         = Tracer
_M.Span           = Span
_M.ZipkinExporter = ZipkinExporter

return _M
```

---

## 69.13 ตัวอย่างการใช้งาน Complete

```lua
-- init_by_lua_block
local otel = require("lib.otel")

_G.tracer = otel.Tracer.new({
    service  = "order-service",
    version  = "1.2.0",
    exporter = otel.ZipkinExporter.new({
        url = os.getenv("ZIPKIN_URL") or "http://zipkin:9411/api/v2/spans"
    }),
    sampler  = {
        should_sample = function(trace_id)
            -- 5% sampling ปกติ, 100% สำหรับ error path
            return math.random() < 0.05
        end
    },
})

-- access_by_lua_block
local span = tracer:start_server_span(
    ngx.req.get_method() .. " " .. ngx.var.uri
)
span:attr("http.method",  ngx.req.get_method())
span:attr("http.target",  ngx.var.uri)
span:attr("http.scheme",  ngx.var.scheme)
span:attr("net.peer.ip",  ngx.var.remote_addr)
span:attr("server.port",  tonumber(ngx.var.server_port))

-- log_by_lua_block
local span = ngx.ctx._current_span
if span then
    span:attr("http.status_code", ngx.status)
    span:attr("http.response_size", tonumber(ngx.var.bytes_sent) or 0)
    if ngx.status >= 500 then
        span:error("HTTP " .. ngx.status)
    end
    span:finish()
end
```

---

## 69.14 แบบฝึกหัด

### แบบฝึกหัดที่ 1: พื้นฐาน
สร้าง Tracing Middleware อย่างง่ายที่:
- รับ `X-Trace-ID` จาก Header หรือสร้าง UUID ใหม่
- บันทึก Request method, URI, IP
- วัดเวลาใน Response Header ว่าใช้เวลากี่ milliseconds
- Log ข้อมูลเป็น JSON

### แบบฝึกหัดที่ 2: W3C Traceparent
เขียนฟังก์ชัน `parse_traceparent(header)` ที่:
- Validate format ตาม W3C spec
- Return table พร้อม trace_id, parent_id, flags, sampled
- Return nil, error message ถ้า format ผิด
- เขียน Unit Test ครอบคลุม edge cases

### แบบฝึกหัดที่ 3: Child Spans
สร้าง wrapper สำหรับ `resty.mysql` ที่:
- สร้าง Child Span อัตโนมัติสำหรับทุก Query
- บันทึก SQL statement, rows affected/returned
- Track query time
- Report error ถ้า query ล้มเหลว

### แบบฝึกหัดที่ 4: Sampling
Implement `RateLimitSampler` ที่:
- รับ parameter `max_traces_per_minute`
- ใช้ sliding window algorithm
- Thread-safe (ใน OpenResty context)
- Test ว่า rate limiting ทำงานถูกต้อง

### แบบฝึกหัดที่ 5: Integration
สร้าง Complete tracing setup สำหรับ API Gateway ที่มี:
- W3C traceparent extraction/injection
- Zipkin/Jaeger export
- Structured JSON logging พร้อม trace_id
- Parent-based sampling ที่เคารพ upstream sampling decision
- Dashboard ใน Zipkin แสดง Service Map

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **OpenTelemetry concepts** - Trace, Span, Context และความสัมพันธ์
2. **W3C Traceparent** - มาตรฐาน Header สำหรับ context propagation
3. **Span lifecycle** - การสร้าง, บันทึก attributes, events และ finish span
4. **Zipkin/Jaeger** - การส่ง trace data ไปยัง backend systems
5. **Sampling strategies** - วิธีต่างๆ ในการเลือกว่าจะ trace Request ไหน
6. **Error tracking** - การบันทึก errors และ exceptions ใน spans
7. **Production-ready library** - Complete implementation พร้อมใช้งาน

Distributed Tracing เป็นเครื่องมือสำคัญในการ Debug และ Optimize ระบบ Microservices ขนาดใหญ่ ความเข้าใจอย่างลึกซึ้งจะช่วยให้คุณสร้างระบบที่ Observable และ Maintainable ได้ดียิ่งขึ้น
