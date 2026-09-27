# บทที่ 68: Advanced Logging

## บทนำ

Logging คือรากฐานของ Observability การทำ Structured Logging ช่วยให้ค้นหา วิเคราะห์ และ alert บน log ได้อย่างมีประสิทธิภาพ ในบทนี้เราจะเรียนรู้เทคนิค logging ขั้นสูงด้วย Lua

---

## 68.1 Structured Logging (JSON)

Structured Logging เก็บข้อมูลเป็น key-value แทนที่จะเป็น plain text

```lua
-- ตัวอย่างที่ 1: JSON Logger พื้นฐาน
local function jsonEncode(value, indent)
    local t = type(value)
    if t == "nil" then
        return "null"
    elseif t == "boolean" then
        return tostring(value)
    elseif t == "number" then
        if value ~= value then return "null" end          -- NaN
        if value == math.huge or value == -math.huge then return "null" end
        if math.floor(value) == value and math.abs(value) < 2^53 then
            return string.format("%d", value)
        end
        return string.format("%.6g", value)
    elseif t == "string" then
        return '"' .. value
            :gsub('\\', '\\\\')
            :gsub('"',  '\\"')
            :gsub('\n', '\\n')
            :gsub('\r', '\\r')
            :gsub('\t', '\\t') .. '"'
    elseif t == "table" then
        -- Check if array
        local isArray = true
        local maxIdx  = 0
        for k in pairs(value) do
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                isArray = false
                break
            end
            maxIdx = math.max(maxIdx, k)
        end
        isArray = isArray and maxIdx == #value

        local parts = {}
        if isArray then
            for _, v in ipairs(value) do
                table.insert(parts, jsonEncode(v))
            end
            return "[" .. table.concat(parts, ",") .. "]"
        else
            for k, v in pairs(value) do
                table.insert(parts, '"' .. tostring(k) .. '":' .. jsonEncode(v))
            end
            table.sort(parts)
            return "{" .. table.concat(parts, ",") .. "}"
        end
    end
    return '"[' .. t .. ']"'
end

local StructuredLogger = {}
StructuredLogger.__index = StructuredLogger

local LOG_LEVELS = {
    DEBUG = 10, INFO = 20, WARN = 30, ERROR = 40, FATAL = 50,
}

function StructuredLogger.new(config)
    local self     = setmetatable({}, StructuredLogger)
    config         = config or {}
    self.service   = config.service  or "app"
    self.version   = config.version  or "1.0.0"
    self.env       = config.env      or "production"
    self.level     = LOG_LEVELS[config.level] or LOG_LEVELS.INFO
    self.output    = config.output   or io.stdout
    self.fields    = config.fields   or {}
    return self
end

function StructuredLogger:log(level, message, fields)
    local levelNum = LOG_LEVELS[level] or LOG_LEVELS.INFO
    if levelNum < self.level then return end

    local entry = {
        timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
        level     = level,
        service   = self.service,
        version   = self.version,
        env       = self.env,
        message   = message,
    }

    -- Merge base fields
    for k, v in pairs(self.fields) do
        entry[k] = v
    end

    -- Merge log-specific fields
    if fields then
        for k, v in pairs(fields) do
            entry[k] = v
        end
    end

    self.output:write(jsonEncode(entry) .. "\n")
end

function StructuredLogger:debug(msg, fields) self:log("DEBUG", msg, fields) end
function StructuredLogger:info(msg, fields)  self:log("INFO",  msg, fields) end
function StructuredLogger:warn(msg, fields)  self:log("WARN",  msg, fields) end
function StructuredLogger:error(msg, fields) self:log("ERROR", msg, fields) end
function StructuredLogger:fatal(msg, fields)
    self:log("FATAL", msg, fields)
    os.exit(1)
end

function StructuredLogger:withFields(fields)
    local child = StructuredLogger.new({
        service = self.service,
        version = self.version,
        env     = self.env,
        output  = self.output,
    })
    for k, v in pairs(self.fields) do child.fields[k] = v end
    for k, v in pairs(fields) do child.fields[k] = v end
    return child
end

-- ทดสอบ
local logger = StructuredLogger.new({
    service = "order-service",
    version = "2.1.0",
    env     = "production",
    level   = "DEBUG",
})

logger:info("Service started", {port = 8080})
logger:debug("Processing request", {
    request_id = "req-001",
    method     = "POST",
    path       = "/api/orders",
})
logger:warn("High memory usage", {
    memory_mb  = 450,
    threshold  = 400,
})
logger:error("Database query failed", {
    error      = "connection timeout",
    query      = "SELECT * FROM orders",
    duration_ms = 5000,
})

-- Logger with pre-set fields
local requestLogger = logger:withFields({
    request_id = "req-abc-123",
    user_id    = "user-456",
})
requestLogger:info("User authenticated")
requestLogger:info("Order created", {order_id = "ord-789"})
```

---

## 68.2 Log Fields: Timestamp, Level, Service, Request ID

```lua
-- ตัวอย่างที่ 2: Standard Log Fields
local StandardFields = {}

function StandardFields.requestContext(requestId, userId, sessionId)
    return {
        request_id = requestId,
        user_id    = userId,
        session_id = sessionId,
    }
end

function StandardFields.httpRequest(method, path, statusCode, durationMs, bodyBytes)
    return {
        http_method       = method,
        http_path         = path,
        http_status       = statusCode,
        http_duration_ms  = durationMs,
        http_bytes        = bodyBytes,
    }
end

function StandardFields.error(err, stack)
    return {
        error       = tostring(err),
        error_type  = type(err) == "table" and (err.type or "unknown") or "string",
        stack_trace = stack,
    }
end

function StandardFields.database(op, table_, durationMs, rows)
    return {
        db_operation  = op,
        db_table      = table_,
        db_duration_ms = durationMs,
        db_rows       = rows,
    }
end

-- Request ID generator
local function newRequestId()
    return string.format("%08x-%04x-%04x-%04x-%012x",
        math.random(0xFFFFFFFF),
        math.random(0xFFFF),
        math.random(0xFFFF),
        math.random(0xFFFF),
        math.random(0xFFFFFFFFFFFF))
end

-- Simulate request processing
local logger2 = StructuredLogger.new({service = "api-gateway", level = "DEBUG"})

local function handleRequest(method, path)
    local reqId = newRequestId()
    local reqLogger = logger2:withFields(StandardFields.requestContext(
        reqId, "user-" .. math.random(1000, 9999), "sess-" .. math.random(1000)))

    reqLogger:info("Request received", StandardFields.httpRequest(method, path, nil, nil, nil))

    local t0 = os.clock()
    -- Simulate processing
    local ok = math.random() > 0.1
    local status = ok and 200 or 500
    local dur    = math.random(10, 300)

    if not ok then
        reqLogger:error("Request failed", StandardFields.error("downstream timeout"))
    end

    reqLogger:info("Request completed", StandardFields.httpRequest(
        method, path, status, dur, math.random(100, 10000)))
    return status
end

math.randomseed(42)
for _, req in ipairs({
    {"GET",  "/api/users"},
    {"POST", "/api/orders"},
    {"GET",  "/api/products"},
}) do
    handleRequest(req[1], req[2])
end
```

---

## 68.3 Correlation IDs across Services

```lua
-- ตัวอย่างที่ 3: Correlation ID Propagation
local CorrelationContext = {}
CorrelationContext.__index = CorrelationContext

-- Thread-local-like context (Lua coroutine or global table simulation)
local _contexts = {}

function CorrelationContext.new(correlationId, requestId, parentSpanId)
    return {
        correlation_id  = correlationId  or newRequestId(),
        request_id      = requestId      or newRequestId(),
        parent_span_id  = parentSpanId,
        span_id         = newRequestId(),
        baggage         = {},
    }
end

function CorrelationContext.fromHeaders(headers)
    return CorrelationContext.new(
        headers["X-Correlation-ID"],
        headers["X-Request-ID"],
        headers["X-Parent-Span-ID"]
    )
end

function CorrelationContext.toHeaders(ctx)
    return {
        ["X-Correlation-ID"]  = ctx.correlation_id,
        ["X-Request-ID"]      = ctx.request_id,
        ["X-Span-ID"]         = ctx.span_id,
        ["X-Parent-Span-ID"]  = ctx.parent_span_id,
    }
end

-- Service A receives external request
local function serviceA_handler(incomingHeaders)
    local ctx    = CorrelationContext.fromHeaders(incomingHeaders)
    local logger = StructuredLogger.new({service = "service-a"})
    local ctxLogger = logger:withFields({
        correlation_id = ctx.correlation_id,
        request_id     = ctx.request_id,
        span_id        = ctx.span_id,
    })

    ctxLogger:info("Handling request in Service A")

    -- Call Service B with propagated context
    local outHeaders = CorrelationContext.toHeaders(ctx)
    ctxLogger:info("Calling Service B", {
        upstream        = "service-b",
        outgoing_headers = jsonEncode(outHeaders),
    })

    return ctx, outHeaders
end

-- Service B receives request from A
local function serviceB_handler(incomingHeaders)
    local ctx    = CorrelationContext.fromHeaders(incomingHeaders)
    local logger = StructuredLogger.new({service = "service-b"})
    local ctxLogger = logger:withFields({
        correlation_id = ctx.correlation_id,
        request_id     = ctx.request_id,
        parent_span_id = ctx.parent_span_id,
        span_id        = ctx.span_id,
    })

    ctxLogger:info("Processing in Service B (correlated)")
    return ctx
end

print("=== Correlation ID Propagation ===")
local ctx, headers = serviceA_handler({
    ["X-Correlation-ID"] = "corr-" .. string.format("%08x", math.random(0xFFFFFFFF)),
    ["X-Request-ID"]     = "req-" .. string.format("%08x", math.random(0xFFFFFFFF)),
})
serviceB_handler(headers)
print("\nCorrelation ID flows through both services: " .. ctx.correlation_id)
```

---

## 68.4 Distributed Tracing with Logs

```lua
-- ตัวอย่างที่ 4: Trace-aware Logger
local TraceLogger = {}
TraceLogger.__index = TraceLogger

function TraceLogger.new(baseLogger)
    local self      = setmetatable({}, TraceLogger)
    self.base       = baseLogger
    self.traceId    = nil
    self.spanId     = nil
    self.parentSpanId = nil
    return self
end

function TraceLogger:withTrace(traceId, spanId, parentSpanId)
    local child = TraceLogger.new(self.base)
    child.traceId      = traceId
    child.spanId       = spanId
    child.parentSpanId = parentSpanId
    return child
end

function TraceLogger:_fields(extra)
    local fields = extra or {}
    if self.traceId then
        fields.trace_id      = self.traceId
        fields.span_id       = self.spanId
        fields.parent_span_id = self.parentSpanId
    end
    return fields
end

function TraceLogger:info(msg, fields)
    self.base:info(msg, self:_fields(fields))
end

function TraceLogger:error(msg, fields)
    self.base:error(msg, self:_fields(fields))
end

function TraceLogger:warn(msg, fields)
    self.base:warn(msg, self:_fields(fields))
end

-- Simulate distributed trace
local base   = StructuredLogger.new({service = "checkout"})
local tracer = TraceLogger.new(base)

local traceId  = string.format("trace-%016x", math.random(0x7FFFFFFF))
local rootSpan = string.format("span-%08x", math.random(0xFFFFFF))

local rootLogger = tracer:withTrace(traceId, rootSpan, nil)
rootLogger:info("Checkout started", {cart_id = "cart-123", items = 3})

local dbSpan   = string.format("span-%08x", math.random(0xFFFFFF))
local dbLogger = tracer:withTrace(traceId, dbSpan, rootSpan)
dbLogger:info("Querying inventory", {items = 3})
dbLogger:info("Inventory confirmed", {available = true})

local paySpan   = string.format("span-%08x", math.random(0xFFFFFF))
local payLogger = tracer:withTrace(traceId, paySpan, rootSpan)
payLogger:info("Charging payment", {amount = 99.99, currency = "THB"})
payLogger:info("Payment successful", {transaction_id = "txn-456"})

rootLogger:info("Checkout completed", {order_id = "ord-789"})
```

---

## 68.5 Log Levels และ Filtering

```lua
-- ตัวอย่างที่ 5: Dynamic Log Level Configuration
local DynamicLogger = {}
DynamicLogger.__index = DynamicLogger

local ALL_LEVELS = {"TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL", "OFF"}
local LEVEL_NUM  = {}
for i, l in ipairs(ALL_LEVELS) do LEVEL_NUM[l] = i end

function DynamicLogger.new(config)
    local self      = setmetatable({}, DynamicLogger)
    self.service    = config.service or "app"
    self.level      = "INFO"
    self.filters    = {}
    self.sinks      = {}
    self.fields     = config.fields or {}
    return self
end

function DynamicLogger:setLevel(level)
    if not LEVEL_NUM[level] then
        error("Invalid level: " .. tostring(level))
    end
    self.level = level
end

function DynamicLogger:addFilter(name, fn)
    self.filters[name] = fn
end

function DynamicLogger:addSink(name, fn)
    self.sinks[name] = fn
end

function DynamicLogger:shouldLog(level, entry)
    if LEVEL_NUM[level] < LEVEL_NUM[self.level] then return false end
    for name, filter in pairs(self.filters) do
        if not filter(entry) then return false end
    end
    return true
end

function DynamicLogger:log(level, message, fields)
    local entry = {
        timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
        level     = level,
        service   = self.service,
        message   = message,
        fields    = fields or {},
    }
    for k, v in pairs(self.fields) do entry[k] = v end
    for k, v in pairs(entry.fields) do entry[k] = v end
    entry.fields = nil

    if not self:shouldLog(level, entry) then return end

    for _, sink in pairs(self.sinks) do
        sink(entry)
    end
end

-- Sinks
local function consoleSink(entry)
    io.write(string.format("[%s] %s: %s\n", entry.level, entry.service, entry.message))
end

local function jsonSink(entry)
    -- Write to "file" (stdout for demo)
    -- io.write(jsonEncode(entry) .. "\n")
end

local function errorSink(entry)
    if entry.level == "ERROR" or entry.level == "FATAL" then
        io.stderr:write(string.format("ERROR: %s - %s\n", entry.service, entry.message))
    end
end

local dl = DynamicLogger.new({service = "api"})
dl:setLevel("DEBUG")
dl:addSink("console", consoleSink)
dl:addSink("stderr_errors", errorSink)

-- Filter: exclude health check logs
dl:addFilter("no_healthcheck", function(entry)
    return not (entry.path and entry.path:match("/health"))
end)

dl:log("INFO",  "Server started", {port = 8080})
dl:log("DEBUG", "Config loaded",  {config_file = "/etc/app.yaml"})
dl:log("WARN",  "Cache miss",     {key = "user:1234"})
dl:log("ERROR", "DB timeout",     {query = "SELECT 1"})

-- Dynamic level change at runtime
print("\n--- Changing log level to WARN ---")
dl:setLevel("WARN")
dl:log("INFO",  "This won't appear")
dl:log("DEBUG", "This won't appear either")
dl:log("WARN",  "This will appear", {reason = "high load"})
```

---

## 68.6 Log Sampling for High Traffic

```lua
-- ตัวอย่างที่ 6: Log Sampling
local SampledLogger = {}
SampledLogger.__index = SampledLogger

function SampledLogger.new(logger, rate)
    local self   = setmetatable({}, SampledLogger)
    self.logger  = logger
    self.rate    = rate    -- 0.0 to 1.0
    self.counter = 0
    self.logged  = 0
    self.skipped = 0
    return self
end

function SampledLogger:shouldSample(level)
    -- Always log errors
    if level == "ERROR" or level == "FATAL" or level == "WARN" then
        return true
    end
    self.counter = self.counter + 1
    -- Deterministic sampling: every 1/rate-th request
    local period = math.floor(1 / self.rate)
    return (self.counter % period) == 0
end

function SampledLogger:log(level, message, fields)
    if self:shouldSample(level) then
        fields         = fields or {}
        fields._sampled = true
        fields._rate    = self.rate
        self.logger:log(level, message, fields)
        self.logged = self.logged + 1
    else
        self.skipped = self.skipped + 1
    end
end

function SampledLogger:info(msg, f)  self:log("INFO",  msg, f) end
function SampledLogger:warn(msg, f)  self:log("WARN",  msg, f) end
function SampledLogger:error(msg, f) self:log("ERROR", msg, f) end

function SampledLogger:stats()
    local total = self.logged + self.skipped
    local pct   = total > 0 and (self.logged / total * 100) or 0
    return string.format("logged=%d, skipped=%d, rate=%.1f%%",
        self.logged, self.skipped, pct)
end

-- ทดสอบ
local base_logger = StructuredLogger.new({service = "high-traffic-api"})
-- Sample 10% of INFO logs
local sampled = SampledLogger.new(base_logger, 0.10)

print("=== Log Sampling (10% of INFO) ===")
for i = 1, 100 do
    sampled:info("Request processed", {request_id = i, duration_ms = math.random(10, 200)})
end

-- Errors always logged
for i = 1, 5 do
    sampled:error("Critical error", {code = i})
end

print("Sampling stats: " .. sampled:stats())
```

---

## 68.7 Audit Logging

```lua
-- ตัวอย่างที่ 7: Audit Log
local AuditLogger = {}
AuditLogger.__index = AuditLogger

local AUDIT_ACTIONS = {
    CREATE = "CREATE",
    READ   = "READ",
    UPDATE = "UPDATE",
    DELETE = "DELETE",
    LOGIN  = "LOGIN",
    LOGOUT = "LOGOUT",
    EXPORT = "EXPORT",
    ADMIN  = "ADMIN",
}

function AuditLogger.new(output)
    local self   = setmetatable({}, AuditLogger)
    self.output  = output or io.stdout
    self.events  = {}
    return self
end

function AuditLogger:log(event)
    local entry = {
        id          = string.format("audit-%016x", math.random(0x7FFFFFFFFFFFFFFF)),
        timestamp   = os.date("!%Y-%m-%dT%H:%M:%SZ"),
        event_type  = "audit",
        action      = event.action,
        actor       = event.actor,          -- who performed the action
        resource    = event.resource,       -- what was acted upon
        resource_id = event.resource_id,
        outcome     = event.outcome or "success",
        ip_address  = event.ip_address,
        user_agent  = event.user_agent,
        changes     = event.changes,        -- diff for UPDATE
        reason      = event.reason,
        metadata    = event.metadata,
    }

    table.insert(self.events, entry)
    self.output:write(jsonEncode(entry) .. "\n")
    return entry.id
end

function AuditLogger:create(actor, resource, id, metadata)
    return self:log({
        action      = AUDIT_ACTIONS.CREATE,
        actor       = actor,
        resource    = resource,
        resource_id = id,
        metadata    = metadata,
    })
end

function AuditLogger:update(actor, resource, id, changes)
    return self:log({
        action      = AUDIT_ACTIONS.UPDATE,
        actor       = actor,
        resource    = resource,
        resource_id = id,
        changes     = changes,
    })
end

function AuditLogger:delete(actor, resource, id, reason)
    return self:log({
        action      = AUDIT_ACTIONS.DELETE,
        actor       = actor,
        resource    = resource,
        resource_id = id,
        reason      = reason,
    })
end

function AuditLogger:login(actor, ip, userAgent, outcome)
    return self:log({
        action      = AUDIT_ACTIONS.LOGIN,
        actor       = actor,
        resource    = "auth",
        resource_id = actor,
        ip_address  = ip,
        user_agent  = userAgent,
        outcome     = outcome or "success",
    })
end

function AuditLogger:query(filter)
    local result = {}
    for _, e in ipairs(self.events) do
        local match = true
        if filter.actor    and e.actor    ~= filter.actor    then match = false end
        if filter.resource and e.resource ~= filter.resource then match = false end
        if filter.action   and e.action   ~= filter.action   then match = false end
        if match then table.insert(result, e) end
    end
    return result
end

-- ทดสอบ
local audit = AuditLogger.new()

print("=== Audit Log Demo ===")

audit:login("alice@example.com", "192.168.1.100", "Mozilla/5.0", "success")
audit:create("alice@example.com", "order", "ord-001", {total = 500, items = 3})
audit:update("alice@example.com", "order", "ord-001", {
    status = {from = "pending", to = "confirmed"}
})
audit:login("bob@example.com", "10.0.0.50", "curl/7.68", "failure")
audit:delete("admin@example.com", "user", "user-deprecated", "account merger")

-- Query
print("\nAlice's actions:")
for _, e in ipairs(audit:query({actor = "alice@example.com"})) do
    print(string.format("  %s %s/%s", e.action, e.resource, e.resource_id))
end

print("\nFailed logins:")
for _, e in ipairs(audit:query({action = "LOGIN"})) do
    if e.outcome == "failure" then
        print(string.format("  %s from %s", e.actor, e.ip_address))
    end
end
```

---

## 68.8 Security Event Logging

```lua
-- ตัวอย่างที่ 8: Security Event Logger
local SecurityLogger = {}
SecurityLogger.__index = SecurityLogger

local SEC_EVENTS = {
    AUTH_FAILURE       = "AUTH_FAILURE",
    AUTH_SUCCESS       = "AUTH_SUCCESS",
    PERMISSION_DENIED  = "PERMISSION_DENIED",
    RATE_LIMIT         = "RATE_LIMIT",
    INJECTION_ATTEMPT  = "INJECTION_ATTEMPT",
    SUSPICIOUS_PATTERN = "SUSPICIOUS_PATTERN",
    BRUTE_FORCE        = "BRUTE_FORCE",
    DATA_EXFILTRATION  = "DATA_EXFILTRATION",
}

function SecurityLogger.new(logger)
    local self           = setmetatable({}, SecurityLogger)
    self.logger          = logger
    self.threatMap       = {}  -- ip -> {count, firstSeen, lastSeen}
    self.brute_threshold = 5
    return self
end

function SecurityLogger:recordThreat(ip, event)
    if not self.threatMap[ip] then
        self.threatMap[ip] = {count = 0, firstSeen = os.time(), events = {}}
    end
    local t = self.threatMap[ip]
    t.count     = t.count + 1
    t.lastSeen  = os.time()
    table.insert(t.events, event)
end

function SecurityLogger:authFailure(ip, username, reason)
    self:recordThreat(ip, SEC_EVENTS.AUTH_FAILURE)
    local threat = self.threatMap[ip]

    local severity = "WARN"
    local isBrute  = threat and threat.count >= self.brute_threshold

    if isBrute then
        severity = "ERROR"
        self.logger:error("Brute force attack detected", {
            event      = SEC_EVENTS.BRUTE_FORCE,
            ip         = ip,
            username   = username,
            attempts   = threat.count,
            first_seen = threat.firstSeen,
        })
    end

    self.logger:log(severity, "Authentication failure", {
        event    = SEC_EVENTS.AUTH_FAILURE,
        ip       = ip,
        username = username,
        reason   = reason,
        attempts = threat and threat.count or 1,
    })
end

function SecurityLogger:permissionDenied(userId, resource, action, ip)
    self.logger:warn("Permission denied", {
        event    = SEC_EVENTS.PERMISSION_DENIED,
        user_id  = userId,
        resource = resource,
        action   = action,
        ip       = ip,
    })
end

function SecurityLogger:injectionAttempt(ip, input, pattern)
    self:recordThreat(ip, SEC_EVENTS.INJECTION_ATTEMPT)
    self.logger:error("Injection attempt detected", {
        event   = SEC_EVENTS.INJECTION_ATTEMPT,
        ip      = ip,
        pattern = pattern,
        -- Input sanitized for logging
        input_hash = string.format("%08x", #input),
        input_len  = #input,
    })
end

function SecurityLogger:rateLimitExceeded(ip, endpoint, limit)
    self.logger:warn("Rate limit exceeded", {
        event    = SEC_EVENTS.RATE_LIMIT,
        ip       = ip,
        endpoint = endpoint,
        limit    = limit,
    })
end

function SecurityLogger:threatReport()
    print("=== Threat Report ===")
    local threats = {}
    for ip, t in pairs(self.threatMap) do
        table.insert(threats, {ip = ip, count = t.count, t = t})
    end
    table.sort(threats, function(a, b) return a.count > b.count end)
    for _, t in ipairs(threats) do
        print(string.format("  %s: %d incidents", t.ip, t.count))
    end
end

-- ทดสอบ
local secLog = SecurityLogger.new(StructuredLogger.new({service = "auth-service"}))

print("=== Security Events ===")
-- Simulate brute force
for i = 1, 7 do
    secLog:authFailure("192.168.1.50", "admin", "invalid_password")
end

secLog:permissionDenied("user-123", "/admin/users", "DELETE", "10.0.0.1")
secLog:injectionAttempt("1.2.3.4", "'; DROP TABLE users; --", "sql_injection")
secLog:rateLimitExceeded("5.6.7.8", "/api/login", 60)

print()
secLog:threatReport()
```

---

## 68.9 Log Aggregation (ELK Stack Concepts)

```lua
-- ตัวอย่างที่ 9: Logstash-compatible Log Format
local LogstashFormatter = {}
LogstashFormatter.__index = LogstashFormatter

function LogstashFormatter.new(config)
    local self       = setmetatable({}, LogstashFormatter)
    self.type        = config.type    or "application"
    self.host        = config.host    or "localhost"
    self.tags        = config.tags    or {}
    return self
end

function LogstashFormatter:format(entry)
    -- Logstash JSON format with @timestamp and @version
    local doc = {
        ["@timestamp"] = entry.timestamp or os.date("!%Y-%m-%dT%H:%M:%S.000Z"),
        ["@version"]   = "1",
        ["type"]       = self.type,
        ["host"]       = self.host,
        ["level"]      = entry.level,
        ["message"]    = entry.message,
        ["service"]    = entry.service,
        ["tags"]       = self.tags,
    }

    -- Flatten all extra fields
    for k, v in pairs(entry) do
        if k ~= "timestamp" and k ~= "level" and k ~= "message" and k ~= "service" then
            doc[k] = v
        end
    end

    return jsonEncode(doc)
end

-- Elasticsearch index mapping simulation
local ElasticMapping = {}

function ElasticMapping.generate(sampleDoc)
    local mapping = {mappings = {properties = {}}}
    for k, v in pairs(sampleDoc) do
        local t = type(v)
        if t == "number" then
            if math.floor(v) == v then
                mapping.mappings.properties[k] = {type = "long"}
            else
                mapping.mappings.properties[k] = {type = "float"}
            end
        elseif t == "boolean" then
            mapping.mappings.properties[k] = {type = "boolean"}
        elseif t == "string" then
            if k:match("_id$") or k:match("_code$") then
                mapping.mappings.properties[k] = {type = "keyword"}
            elseif k == "message" then
                mapping.mappings.properties[k] = {
                    type   = "text",
                    fields = {keyword = {type = "keyword"}}
                }
            else
                mapping.mappings.properties[k] = {type = "keyword"}
            end
        end
    end
    return mapping
end

local lsFormatter = LogstashFormatter.new({
    type = "nginx-access",
    host = "web-server-01",
    tags = {"nginx", "production"},
})

local sample = {
    level      = "INFO",
    message    = "GET /api/users 200 45ms",
    service    = "nginx",
    timestamp  = os.date("!%Y-%m-%dT%H:%M:%S.000Z"),
    http_method = "GET",
    http_path   = "/api/users",
    http_status = 200,
    duration_ms = 45,
    bytes_sent  = 1234,
    client_ip   = "192.168.1.100",
    user_agent  = "Mozilla/5.0",
}

print("=== Logstash JSON Format ===")
print(lsFormatter:format(sample))

print("\n=== Elasticsearch Mapping ===")
local mapping = ElasticMapping.generate(sample)
print("Properties: " .. #(function(t) local r={} for k in pairs(t) do r[#r+1]=k end return r end)(
    mapping.mappings.properties) .. " fields")
```

---

## 68.10 Fluentd Compatible Logging

```lua
-- ตัวอย่างที่ 10: Fluentd Forward Protocol
local FluentdLogger = {}
FluentdLogger.__index = FluentdLogger

function FluentdLogger.new(tag, config)
    local self   = setmetatable({}, FluentdLogger)
    self.tag     = tag
    self.config  = config or {}
    self.buffer  = {}
    self.maxBuf  = config.bufferSize or 100
    return self
end

function FluentdLogger:emit(record)
    local event = {
        tag       = self.tag,
        time      = os.time(),
        record    = record,
    }
    table.insert(self.buffer, event)

    if #self.buffer >= self.maxBuf then
        self:flush()
    end
end

function FluentdLogger:flush()
    if #self.buffer == 0 then return end

    -- In production: send via TCP to Fluentd
    -- Format: msgpack [tag, time, record]
    -- Here we simulate with JSON
    local entries = {}
    for _, e in ipairs(self.buffer) do
        table.insert(entries, string.format(
            "[%q, %d, %s]", e.tag, e.time, jsonEncode(e.record)))
    end

    print(string.format("[Fluentd] Flushing %d events for tag '%s'",
        #self.buffer, self.tag))
    -- print(table.concat(entries, "\n"))

    self.buffer = {}
end

function FluentdLogger:withTag(tag)
    return FluentdLogger.new(self.tag .. "." .. tag, self.config)
end

-- Fluentd configuration (simulation)
local function generateFluentdConfig(inputs, filters, outputs)
    local lines = {}

    -- Input
    for _, input in ipairs(inputs) do
        table.insert(lines, "<source>")
        for k, v in pairs(input) do
            table.insert(lines, string.format("  %s %s", k, tostring(v)))
        end
        table.insert(lines, "</source>")
        table.insert(lines, "")
    end

    -- Filter
    for _, filter in ipairs(filters or {}) do
        table.insert(lines, string.format('<filter %s>', filter.tag or "**"))
        for k, v in pairs(filter) do
            if k ~= "tag" then
                table.insert(lines, string.format("  %s %s", k, tostring(v)))
            end
        end
        table.insert(lines, "</filter>")
        table.insert(lines, "")
    end

    -- Output
    for _, output in ipairs(outputs) do
        table.insert(lines, string.format('<match %s>', output.tag or "**"))
        for k, v in pairs(output) do
            if k ~= "tag" then
                table.insert(lines, string.format("  %s %s", k, tostring(v)))
            end
        end
        table.insert(lines, "</match>")
        table.insert(lines, "")
    end

    return table.concat(lines, "\n")
end

local fluentdConfig = generateFluentdConfig(
    {
        {["@type"] = "tail",
         path      = "/var/log/app/*.log",
         tag       = "app.*",
         format    = "json",
         time_key  = "timestamp"},
    },
    {
        {tag = "app.**",
         ["@type"] = "record_transformer",
         ["<record>"] = "hostname #{Socket.gethostname}"},
    },
    {
        {tag = "app.**",
         ["@type"] = "elasticsearch",
         host      = "elasticsearch",
         port      = 9200,
         index_name = "app-logs",
         type_name  = "_doc"},
    }
)

print("=== Fluentd Configuration ===")
print(fluentdConfig)

-- Logger usage
local flLogger = FluentdLogger.new("app.api")
for i = 1, 5 do
    flLogger:emit({
        level      = "INFO",
        message    = "Request handled",
        request_id = "req-" .. i,
        status     = 200,
    })
end
flLogger:flush()
```

---

## 68.11 Cloud Logging

```lua
-- ตัวอย่างที่ 11: Cloud Logging Formats (GCP / AWS)
local CloudLogger = {}
CloudLogger.__index = CloudLogger

-- Google Cloud Logging format
function CloudLogger.toGCP(entry)
    local severity_map = {
        DEBUG   = "DEBUG",
        INFO    = "INFO",
        WARN    = "WARNING",
        WARNING = "WARNING",
        ERROR   = "ERROR",
        FATAL   = "CRITICAL",
    }
    return {
        severity  = severity_map[entry.level] or "DEFAULT",
        message   = entry.message,
        timestamp = {
            seconds = os.time(),
            nanos   = 0,
        },
        labels    = {
            service = entry.service,
            version = entry.version,
        },
        httpRequest = entry.http and {
            requestMethod = entry.http.method,
            requestUrl    = entry.http.url,
            status        = entry.http.status,
            latency       = entry.http.duration_ms and
                string.format("%.3fs", entry.http.duration_ms / 1000),
        } or nil,
        jsonPayload = entry,
    }
end

-- AWS CloudWatch Logs format
function CloudLogger.toCloudWatch(entry, logGroup, logStream)
    return {
        logGroupName  = logGroup  or "/app/production",
        logStreamName = logStream or entry.service or "default",
        logEvents     = {
            {
                timestamp = os.time() * 1000,  -- milliseconds
                message   = jsonEncode({
                    level     = entry.level,
                    message   = entry.message,
                    service   = entry.service,
                    fields    = entry,
                }),
            }
        }
    }
end

-- Azure Monitor Logs (Application Insights)
function CloudLogger.toAppInsights(entry)
    local type_map = {
        INFO    = "trace",
        DEBUG   = "trace",
        WARN    = "trace",
        ERROR   = "exception",
        FATAL   = "exception",
    }
    return {
        name       = "Microsoft.ApplicationInsights." ..
            (entry.service or "App") .. "." ..
            (type_map[entry.level] or "trace"),
        time       = os.date("!%Y-%m-%dT%H:%M:%S.000Z"),
        iKey       = "instrumentation-key-here",
        data       = {
            baseType = (type_map[entry.level] or "trace") == "trace"
                and "MessageData" or "ExceptionData",
            baseData = {
                message    = entry.message,
                severityLevel = entry.level,
                properties = entry,
            }
        }
    }
end

local testEntry = {
    level     = "ERROR",
    service   = "payment-service",
    version   = "1.0.0",
    message   = "Payment processing failed",
    error     = "gateway_timeout",
    amount    = 99.99,
    http      = {method = "POST", url = "/api/charge", status = 504, duration_ms = 5000},
}

print("=== GCP Cloud Logging ===")
print(jsonEncode(CloudLogger.toGCP(testEntry)))

print("\n=== AWS CloudWatch ===")
print(jsonEncode(CloudLogger.toCloudWatch(testEntry, "/payment/prod", "payment-service")))
```

---

## 68.12 Log Retention Policy

```lua
-- ตัวอย่างที่ 12: Log Retention Manager
local RetentionManager = {}
RetentionManager.__index = RetentionManager

function RetentionManager.new()
    local self    = setmetatable({}, RetentionManager)
    self.policies = {}
    return self
end

function RetentionManager:addPolicy(name, config)
    self.policies[name] = {
        name        = name,
        retainDays  = config.retainDays  or 30,
        maxSizeMB   = config.maxSizeMB,
        compress    = config.compress    ~= false,
        archive     = config.archive,
        tags        = config.tags        or {},
    }
end

function RetentionManager:getPolicy(level, tags)
    -- Match most specific policy
    local matched = nil
    for _, policy in pairs(self.policies) do
        local tagMatch = true
        for _, t in ipairs(policy.tags) do
            local found = false
            for _, lt in ipairs(tags or {}) do
                if lt == t then found = true; break end
            end
            if not found then tagMatch = false; break end
        end
        if tagMatch then
            if matched == nil or #policy.tags > #matched.tags then
                matched = policy
            end
        end
    end
    return matched or self.policies["default"]
end

function RetentionManager:generateRotationConfig()
    local lines = {}
    for name, policy in pairs(self.policies) do
        table.insert(lines, string.format("# Policy: %s", name))
        table.insert(lines, "/var/log/app/" .. name .. "/*.log {")
        table.insert(lines, string.format("    rotate %d", policy.retainDays))
        table.insert(lines, "    daily")
        table.insert(lines, "    missingok")
        table.insert(lines, "    notifempty")
        if policy.compress then
            table.insert(lines, "    compress")
            table.insert(lines, "    delaycompress")
        end
        if policy.maxSizeMB then
            table.insert(lines, string.format("    maxsize %dM", policy.maxSizeMB))
        end
        table.insert(lines, "    postrotate")
        table.insert(lines, "        kill -HUP $(cat /var/run/app.pid 2>/dev/null) 2>/dev/null || true")
        table.insert(lines, "    endscript")
        table.insert(lines, "}")
        table.insert(lines, "")
    end
    return table.concat(lines, "\n")
end

local rm = RetentionManager.new()
rm:addPolicy("default", {retainDays = 30, maxSizeMB = 100, compress = true})
rm:addPolicy("audit",   {retainDays = 365, maxSizeMB = 500, compress = true, archive = "s3://logs-archive"})
rm:addPolicy("security", {retainDays = 90, compress = true, tags = {"security"}})
rm:addPolicy("debug",    {retainDays = 7,  maxSizeMB = 50,  compress = false})

print("=== Log Rotation Configuration (logrotate) ===")
print(rm:generateRotationConfig())
```

---

## 68.13 Real-time Log Analysis

```lua
-- ตัวอย่างที่ 13: Real-time Log Analyzer
local LogAnalyzer = {}
LogAnalyzer.__index = LogAnalyzer

function LogAnalyzer.new(windowSecs)
    local self        = setmetatable({}, LogAnalyzer)
    self.windowSecs   = windowSecs or 60
    self.events       = {}
    self.patterns     = {}
    self.alerts       = {}
    return self
end

function LogAnalyzer:addPattern(name, config)
    table.insert(self.patterns, {
        name      = name,
        match     = config.match,      -- function(entry) -> bool
        threshold = config.threshold,  -- count within window
        action    = config.action,     -- function(count, entries)
    })
end

function LogAnalyzer:ingest(entry)
    entry._time = os.time()
    table.insert(self.events, entry)
    self:_cleanup()
    self:_analyze(entry)
end

function LogAnalyzer:_cleanup()
    local cutoff = os.time() - self.windowSecs
    local new    = {}
    for _, e in ipairs(self.events) do
        if e._time >= cutoff then
            table.insert(new, e)
        end
    end
    self.events = new
end

function LogAnalyzer:_windowEvents(pattern)
    local matched = {}
    for _, e in ipairs(self.events) do
        if pattern.match(e) then
            table.insert(matched, e)
        end
    end
    return matched
end

function LogAnalyzer:_analyze(newEntry)
    for _, pattern in ipairs(self.patterns) do
        if pattern.match(newEntry) then
            local matched = self:_windowEvents(pattern)
            if #matched >= pattern.threshold then
                local alertKey = pattern.name
                -- Debounce: only alert once per window
                if not self.alerts[alertKey] or
                   (os.time() - self.alerts[alertKey]) > self.windowSecs then
                    self.alerts[alertKey] = os.time()
                    pattern.action(#matched, matched)
                end
            end
        end
    end
end

function LogAnalyzer:stats()
    local counts = {}
    for _, e in ipairs(self.events) do
        local k = e.level or "UNKNOWN"
        counts[k] = (counts[k] or 0) + 1
    end
    return counts
end

-- ทดสอบ
local analyzer = LogAnalyzer.new(60)

analyzer:addPattern("error_spike", {
    match     = function(e) return e.level == "ERROR" end,
    threshold = 5,
    action    = function(count, events)
        print(string.format("[ALERT] Error spike: %d errors in last 60s!", count))
    end,
})

analyzer:addPattern("auth_failure_burst", {
    match     = function(e)
        return e.event_type == "AUTH_FAILURE"
    end,
    threshold = 3,
    action    = function(count, events)
        print(string.format("[ALERT] Auth failure burst: %d failures!", count))
    end,
})

-- Feed logs
local testLogs = {
    {level = "INFO",  message = "Request OK",        event_type = "HTTP"},
    {level = "ERROR", message = "DB timeout",         event_type = "DB"},
    {level = "INFO",  message = "Request OK",         event_type = "HTTP"},
    {level = "ERROR", message = "Connection refused",  event_type = "DB"},
    {level = "INFO",  message = "Auth failed",         event_type = "AUTH_FAILURE"},
    {level = "INFO",  message = "Auth failed",         event_type = "AUTH_FAILURE"},
    {level = "ERROR", message = "NullPointer",         event_type = "APP"},
    {level = "INFO",  message = "Auth failed",         event_type = "AUTH_FAILURE"},
    {level = "ERROR", message = "Memory OOM",          event_type = "SYSTEM"},
    {level = "ERROR", message = "Disk full",           event_type = "SYSTEM"},
    {level = "ERROR", message = "Queue overflow",      event_type = "QUEUE"},
}

print("=== Real-time Log Analysis ===")
for _, log in ipairs(testLogs) do
    analyzer:ingest(log)
end

local stats = analyzer:stats()
print("\nLog stats (last 60s):")
for level, count in pairs(stats) do
    print(string.format("  %s: %d", level, count))
end
```

---

## 68.14 Log Masking / PII Protection

```lua
-- ตัวอย่างที่ 14: PII Masking in Logs
local PIIMasker = {}
PIIMasker.__index = PIIMasker

function PIIMasker.new()
    local self    = setmetatable({}, PIIMasker)
    self.rules    = {}
    return self
end

function PIIMasker:addRule(name, pattern, replacement)
    table.insert(self.rules, {
        name        = name,
        pattern     = pattern,
        replacement = replacement,
    })
end

function PIIMasker:mask(text)
    if type(text) ~= "string" then return text end
    local result = text
    for _, rule in ipairs(self.rules) do
        result = result:gsub(rule.pattern, rule.replacement)
    end
    return result
end

function PIIMasker:maskTable(t, depth)
    depth = depth or 0
    if depth > 5 then return t end
    local result = {}
    for k, v in pairs(t) do
        if type(v) == "string" then
            result[k] = self:mask(v)
        elseif type(v) == "table" then
            result[k] = self:maskTable(v, depth + 1)
        else
            result[k] = v
        end
    end
    return result
end

-- Field-level masking
local SENSITIVE_FIELDS = {
    "password", "token", "secret", "api_key", "credit_card",
    "ssn", "passport", "phone", "email",
}

function PIIMasker:maskFields(entry)
    local result = {}
    for k, v in pairs(entry) do
        local isSensitive = false
        for _, field in ipairs(SENSITIVE_FIELDS) do
            if k:lower():match(field) then
                isSensitive = true
                break
            end
        end
        if isSensitive and type(v) == "string" then
            -- Show only first/last chars
            if #v > 4 then
                result[k] = v:sub(1, 2) .. string.rep("*", #v - 4) .. v:sub(-2)
            else
                result[k] = string.rep("*", #v)
            end
        else
            result[k] = v
        end
    end
    return result
end

local masker = PIIMasker.new()

-- Common PII patterns
masker:addRule("email",       "[%w.]+@[%w.]+%.[%a]+",
    function(s) return s:sub(1,2) .. "***@***.***" end)
masker:addRule("thai_id",     "%d%d%d%d%d%d%d%d%d%d%d%d%d",
    function(s) return s:sub(1,1) .. "-***-" .. s:sub(-4) end)
masker:addRule("credit_card", "%d%d%d%d[%s%-]?%d%d%d%d[%s%-]?%d%d%d%d[%s%-]?%d%d%d%d",
    "****-****-****-XXXX")

print("=== PII Masking ===")
local sensitiveLog = {
    user_id      = "user-123",
    email        = "john.doe@example.com",
    password     = "secretpass123",
    credit_card  = "4532015112830366",
    api_key      = "sk-live-xxxxxxxxxxx",
    message      = "User login: john.doe@example.com from 192.168.1.1",
}

local masked = masker:maskFields(sensitiveLog)
masked.message = masker:mask(masked.message)

print("Original fields:")
for k, v in pairs(sensitiveLog) do
    print(string.format("  %s: %s", k, tostring(v)))
end

print("\nMasked fields:")
for k, v in pairs(masked) do
    print(string.format("  %s: %s", k, tostring(v)))
end
```

---

## 68.15 Multi-output Logger

```lua
-- ตัวอย่างที่ 15: Logger with multiple outputs
local MultiOutputLogger = {}
MultiOutputLogger.__index = MultiOutputLogger

function MultiOutputLogger.new(config)
    local self   = setmetatable({}, MultiOutputLogger)
    self.outputs = {}
    self.fields  = config and config.fields or {}
    self.service = config and config.service or "app"
    self.masker  = config and config.masker
    return self
end

function MultiOutputLogger:addOutput(name, output)
    self.outputs[name] = output
    return self
end

function MultiOutputLogger:removeOutput(name)
    self.outputs[name] = nil
end

function MultiOutputLogger:_write(entry)
    -- Apply PII masking
    if self.masker then
        entry = self.masker:maskFields(entry)
    end

    for name, output in pairs(self.outputs) do
        local ok, err = pcall(function()
            output:write(entry)
        end)
        if not ok then
            -- Log to stderr but don't crash
            io.stderr:write(string.format("[LOGGER] Output '%s' failed: %s\n",
                name, tostring(err)))
        end
    end
end

function MultiOutputLogger:log(level, message, fields)
    local entry = {
        timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
        level     = level,
        service   = self.service,
        message   = message,
    }
    for k, v in pairs(self.fields) do entry[k] = v end
    if fields then
        for k, v in pairs(fields) do entry[k] = v end
    end
    self:_write(entry)
end

function MultiOutputLogger:info(msg, f)  self:log("INFO",  msg, f) end
function MultiOutputLogger:warn(msg, f)  self:log("WARN",  msg, f) end
function MultiOutputLogger:error(msg, f) self:log("ERROR", msg, f) end

-- Output implementations
local ConsoleOutput = {}
ConsoleOutput.__index = ConsoleOutput
function ConsoleOutput.new(format)
    local self    = setmetatable({}, ConsoleOutput)
    self.format   = format or "text"
    return self
end
function ConsoleOutput:write(entry)
    if self.format == "json" then
        io.write(jsonEncode(entry) .. "\n")
    else
        io.write(string.format("[%s] %s: %s\n",
            entry.level, entry.service, entry.message))
    end
end

local BufferOutput = {}
BufferOutput.__index = BufferOutput
function BufferOutput.new()
    local self  = setmetatable({}, BufferOutput)
    self.buffer = {}
    return self
end
function BufferOutput:write(entry)
    table.insert(self.buffer, entry)
end
function BufferOutput:flush()
    local entries = self.buffer
    self.buffer = {}
    return entries
end

local FileOutput = {}
FileOutput.__index = FileOutput
function FileOutput.new(path)
    local self  = setmetatable({}, FileOutput)
    self.path   = path
    self.count  = 0
    return self
end
function FileOutput:write(entry)
    self.count = self.count + 1
    -- In production: append to file
    -- local f = io.open(self.path, "a")
    -- f:write(jsonEncode(entry) .. "\n")
    -- f:close()
end

-- ทดสอบ
local bufOutput = BufferOutput.new()
local logger    = MultiOutputLogger.new({service = "multi-output-demo"})

logger:addOutput("console", ConsoleOutput.new("text"))
logger:addOutput("buffer",  bufOutput)

logger:info("Application started", {pid = 12345})
logger:warn("Memory usage high",   {usage_mb = 800})
logger:error("Connection failed",  {host = "db-master"})

print("\nBuffered entries:")
for _, entry in ipairs(bufOutput:flush()) do
    print(string.format("  [%s] %s", entry.level, entry.message))
end
```

---

## 68.16 Log Format Comparison

```lua
-- ตัวอย่างที่ 16: เปรียบเทียบ log formats
local function demoFormats()
    local entry = {
        timestamp  = "2024-01-15T10:30:00Z",
        level      = "ERROR",
        service    = "payment-service",
        message    = "Payment gateway timeout",
        request_id = "req-abc123",
        user_id    = "user-456",
        amount     = 99.99,
        duration_ms = 5000,
        error      = "gateway_timeout",
    }

    -- 1. Plain text (legacy)
    print("=== Plain Text ===")
    print(string.format("[%s] %s ERROR: %s (req=%s)",
        entry.timestamp, entry.service, entry.message, entry.request_id))

    -- 2. JSON
    print("\n=== JSON ===")
    print(jsonEncode(entry))

    -- 3. logfmt
    print("\n=== logfmt ===")
    local parts = {}
    for k, v in pairs(entry) do
        if type(v) == "string" and v:match("%s") then
            table.insert(parts, string.format('%s="%s"', k, v))
        else
            table.insert(parts, string.format('%s=%s', k, tostring(v)))
        end
    end
    table.sort(parts)
    print(table.concat(parts, " "))

    -- 4. CEF (Common Event Format) for SIEM
    print("\n=== CEF (SIEM) ===")
    print(string.format("CEF:0|Company|payment-service|1.0|GATEWAY_TIMEOUT|%s|5|" ..
        "rt=%d requestId=%s userId=%s durationMs=%d",
        entry.message,
        os.time(),
        entry.request_id,
        entry.user_id,
        entry.duration_ms))

    -- 5. GELF (Graylog Extended Log Format)
    print("\n=== GELF ===")
    local gelf = {
        version      = "1.1",
        host         = "web-01",
        short_message = entry.message,
        full_message  = jsonEncode(entry),
        timestamp     = os.time(),
        level         = 3,  -- 0=EMERG, 3=ERROR, 7=DEBUG
        _service      = entry.service,
        _request_id   = entry.request_id,
        _duration_ms  = entry.duration_ms,
    }
    print(jsonEncode(gelf))
end

demoFormats()
```

---

## 68.17 Log Context Propagation

```lua
-- ตัวอย่างที่ 17: Context-aware logging middleware
local LogContext = {}
LogContext.__index = LogContext

-- Global context stack (simplified, use coroutine-local in production)
local _contextStack = {}

function LogContext.push(ctx)
    table.insert(_contextStack, ctx)
end

function LogContext.pop()
    return table.remove(_contextStack)
end

function LogContext.current()
    return _contextStack[#_contextStack] or {}
end

function LogContext.with(ctx, fn)
    LogContext.push(ctx)
    local ok, result = pcall(fn)
    LogContext.pop()
    if not ok then error(result) end
    return result
end

-- Context-aware logger
local ContextLogger = {}
ContextLogger.__index = ContextLogger

function ContextLogger.new(base)
    local self = setmetatable({}, ContextLogger)
    self.base  = base
    return self
end

function ContextLogger:log(level, message, extraFields)
    local ctx    = LogContext.current()
    local fields = {}
    for k, v in pairs(ctx) do fields[k] = v end
    if extraFields then
        for k, v in pairs(extraFields) do fields[k] = v end
    end
    self.base:log(level, message, fields)
end

function ContextLogger:info(msg, f)  self:log("INFO",  msg, f) end
function ContextLogger:error(msg, f) self:log("ERROR", msg, f) end
function ContextLogger:warn(msg, f)  self:log("WARN",  msg, f) end

-- ทดสอบ
local baseLogger = StructuredLogger.new({service = "context-demo"})
local ctxLogger  = ContextLogger.new(baseLogger)

ctxLogger:info("Outside context - no extra fields")

LogContext.with({request_id = "req-001", user_id = "user-100"}, function()
    ctxLogger:info("Inside request context")
    ctxLogger:info("Processing order", {order_id = "ord-500"})

    LogContext.with({db_transaction = "txn-abc"}, function()
        ctxLogger:info("Inside DB transaction context")
    end)

    ctxLogger:info("Back to request context only")
end)

ctxLogger:info("Outside context again")
```

---

## 68.18 สรุป Logging Patterns

```lua
-- ตัวอย่างที่ 18: Complete logging setup
local function createProductionLogger(serviceName, version)
    local logger = MultiOutputLogger.new({
        service = serviceName,
        fields  = {
            version  = version,
            env      = os.getenv("APP_ENV") or "production",
            hostname = os.getenv("HOSTNAME") or "unknown",
        },
        masker = (function()
            local m = PIIMasker.new()
            m:addRule("email", "[%w.]+@[%w.]+%.[%a]+",
                function(s) return s:sub(1,2) .. "***@***.***" end)
            return m
        end)(),
    })

    -- Console output (JSON for log aggregators)
    logger:addOutput("console", ConsoleOutput.new("json"))

    -- Buffer for async flush
    local buf = BufferOutput.new()
    logger:addOutput("buffer", buf)

    return logger, buf
end

local prodLogger, logBuffer = createProductionLogger("order-service", "2.0.0")

-- Standard request logging
local function logRequest(logger, req, resp, duration)
    local level = resp.status >= 500 and "ERROR" or
                  resp.status >= 400 and "WARN"  or "INFO"
    logger:log(level, "HTTP request", {
        http_method   = req.method,
        http_path     = req.path,
        http_status   = resp.status,
        duration_ms   = duration,
        request_id    = req.id,
        user_id       = req.userId,
        user_email    = req.userEmail,  -- will be masked
    })
end

logRequest(prodLogger,
    {method = "POST", path = "/api/orders", id = "req-001",
     userId = "u123", userEmail = "test@example.com"},
    {status = 201},
    45
)

logRequest(prodLogger,
    {method = "GET", path = "/api/users", id = "req-002",
     userId = "u456", userEmail = "user@domain.com"},
    {status = 500},
    3200
)

print("\nBuffered log count: " .. #logBuffer:flush())
```

---

## สรุปบทที่ 68

ในบทนี้เราได้เรียนรู้:

1. **Structured Logging** - JSON format พร้อม standard fields
2. **Log Fields** - timestamp, level, service, request_id, user_id
3. **Correlation IDs** - การส่ง context ข้าม services
4. **Distributed Tracing** - trace_id และ span_id ใน logs
5. **Log Levels** - Dynamic level configuration
6. **Log Sampling** - ลด volume สำหรับ high-traffic
7. **Audit Logging** - บันทึกการกระทำของผู้ใช้
8. **Security Logging** - Threat detection และ security events
9. **ELK Stack** - Logstash, Elasticsearch formats
10. **Fluentd** - Configuration และ emit
11. **Cloud Logging** - GCP, AWS, Azure formats
12. **Retention Policy** - Log rotation configuration
13. **Real-time Analysis** - Pattern matching และ alerting
14. **PII Masking** - ปกป้อง sensitive data
15. **Multi-output** - Console, file, buffer sinks
16. **Context Propagation** - Thread-local logging context

---

*จบบทที่ 68 - Advanced Logging*
