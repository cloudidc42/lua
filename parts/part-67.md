# บทที่ 67: Monitoring และ Metrics

## บทนำ

Observability คือความสามารถในการเข้าใจสถานะภายในของระบบจากข้อมูลภายนอก ประกอบด้วย 3 เสาหลัก (Three Pillars):
- **Logs** - บันทึกเหตุการณ์ที่เกิดขึ้น
- **Metrics** - ตัวเลขวัดผลการทำงาน
- **Traces** - ติดตาม request ข้ามระบบ

---

## 67.1 Prometheus Metrics Format

Prometheus ใช้ text format ง่ายๆ ในการ expose metrics

```lua
-- ตัวอย่างที่ 1: Prometheus text format generator
local PrometheusFormatter = {}
PrometheusFormatter.__index = PrometheusFormatter

function PrometheusFormatter.new()
    local self   = setmetatable({}, PrometheusFormatter)
    self.metrics = {}
    return self
end

function PrometheusFormatter:addMetric(name, help, type_, samples)
    table.insert(self.metrics, {
        name    = name,
        help    = help,
        type    = type_,
        samples = samples,
    })
end

function PrometheusFormatter:formatLabels(labels)
    if not labels or next(labels) == nil then return "" end
    local parts = {}
    for k, v in pairs(labels) do
        table.insert(parts, string.format('%s="%s"', k, tostring(v)))
    end
    table.sort(parts)
    return "{" .. table.concat(parts, ",") .. "}"
end

function PrometheusFormatter:render()
    local lines = {}
    for _, m in ipairs(self.metrics) do
        table.insert(lines, "# HELP " .. m.name .. " " .. m.help)
        table.insert(lines, "# TYPE " .. m.name .. " " .. m.type)
        for _, sample in ipairs(m.samples) do
            local labelStr = self:formatLabels(sample.labels)
            local line     = m.name .. labelStr .. " " .. tostring(sample.value)
            if sample.timestamp then
                line = line .. " " .. tostring(sample.timestamp)
            end
            table.insert(lines, line)
        end
    end
    return table.concat(lines, "\n")
end

local pf = PrometheusFormatter.new()

pf:addMetric(
    "http_requests_total",
    "Total HTTP requests",
    "counter",
    {
        {labels = {method = "GET",  status = "200"}, value = 1523},
        {labels = {method = "POST", status = "200"}, value = 456},
        {labels = {method = "GET",  status = "404"}, value = 23},
        {labels = {method = "POST", status = "500"}, value = 7},
    }
)

pf:addMetric(
    "http_request_duration_seconds",
    "HTTP request duration in seconds",
    "histogram",
    {
        {labels = {le = "0.005"}, value = 123},
        {labels = {le = "0.01"},  value = 456},
        {labels = {le = "0.025"}, value = 789},
        {labels = {le = "+Inf"},  value = 1000},
        {labels = {},              value = 1000, name_suffix = "_count"},
        {labels = {},              value = 8.45, name_suffix = "_sum"},
    }
)

print(pf:render())
```

---

## 67.2 Counter

Counter คือ metric ที่เพิ่มขึ้นเสมอ ไม่ลดลง

```lua
-- ตัวอย่างที่ 2: Counter implementation
local Counter = {}
Counter.__index = Counter

function Counter.new(name, help, labelNames)
    local self       = setmetatable({}, Counter)
    self.name        = name
    self.help        = help
    self.labelNames  = labelNames or {}
    self.values      = {}  -- key -> value
    self.createdAt   = os.time()
    return self
end

function Counter:_key(labels)
    if not labels or #self.labelNames == 0 then return "__default__" end
    local parts = {}
    for _, name in ipairs(self.labelNames) do
        table.insert(parts, tostring(labels[name] or ""))
    end
    return table.concat(parts, "|")
end

function Counter:inc(labels, amount)
    local key = self:_key(labels)
    self.values[key] = (self.values[key] or 0) + (amount or 1)
end

function Counter:get(labels)
    return self.values[self:_key(labels)] or 0
end

function Counter:reset(labels)
    self.values[self:_key(labels)] = 0
end

function Counter:collect()
    local samples = {}
    for key, value in pairs(self.values) do
        local labels = {}
        if key ~= "__default__" then
            local parts = {}
            for p in key:gmatch("[^|]+") do table.insert(parts, p) end
            for i, name in ipairs(self.labelNames) do
                labels[name] = parts[i]
            end
        end
        table.insert(samples, {labels = labels, value = value})
    end
    return samples
end

-- ทดสอบ
local reqCounter = Counter.new(
    "http_requests_total",
    "Total HTTP requests",
    {"method", "status", "path"}
)

reqCounter:inc({method = "GET",  status = "200", path = "/api"})
reqCounter:inc({method = "GET",  status = "200", path = "/api"})
reqCounter:inc({method = "POST", status = "201", path = "/api/users"})
reqCounter:inc({method = "GET",  status = "404", path = "/missing"})

print("Counter values:")
local samples = reqCounter:collect()
for _, s in ipairs(samples) do
    local labelParts = {}
    for k, v in pairs(s.labels) do
        table.insert(labelParts, k .. "=" .. v)
    end
    print(string.format("  {%s} = %d", table.concat(labelParts, ", "), s.value))
end
```

---

## 67.3 Gauge

Gauge คือ metric ที่เพิ่มหรือลดได้ ใช้วัดค่าปัจจุบัน

```lua
-- ตัวอย่างที่ 3: Gauge implementation
local Gauge = {}
Gauge.__index = Gauge

function Gauge.new(name, help, labelNames)
    local self       = setmetatable({}, Gauge)
    self.name        = name
    self.help        = help
    self.labelNames  = labelNames or {}
    self.values      = {}
    return self
end

function Gauge:_key(labels)
    if not labels or #self.labelNames == 0 then return "__default__" end
    local parts = {}
    for _, n in ipairs(self.labelNames) do
        table.insert(parts, tostring(labels[n] or ""))
    end
    return table.concat(parts, "|")
end

function Gauge:set(labels, value)
    if type(labels) == "number" then
        -- Shorthand: gauge:set(42)
        self.values["__default__"] = labels
        return
    end
    self.values[self:_key(labels)] = value
end

function Gauge:inc(labels, amount)
    local key = self:_key(labels)
    self.values[key] = (self.values[key] or 0) + (amount or 1)
end

function Gauge:dec(labels, amount)
    local key = self:_key(labels)
    self.values[key] = (self.values[key] or 0) - (amount or 1)
end

function Gauge:get(labels)
    return self.values[self:_key(labels)] or 0
end

function Gauge:setToCurrentTime(labels)
    self:set(labels, os.time())
end

function Gauge:collect()
    local samples = {}
    for key, value in pairs(self.values) do
        local labels = {}
        if key ~= "__default__" then
            local parts = {}
            for p in key:gmatch("[^|]+") do table.insert(parts, p) end
            for i, n in ipairs(self.labelNames) do labels[n] = parts[i] end
        end
        table.insert(samples, {labels = labels, value = value})
    end
    return samples
end

-- ทดสอบ
local connGauge = Gauge.new("active_connections", "Active connections", {"service"})

connGauge:set({service = "api"},      10)
connGauge:set({service = "database"}, 5)

connGauge:inc({service = "api"}, 3)
connGauge:dec({service = "database"}, 2)

print("Gauge values:")
for _, s in ipairs(connGauge:collect()) do
    for k, v in pairs(s.labels) do
        print(string.format("  service=%s -> %d", v, s.value))
    end
end

-- Memory gauge example
local memGauge = Gauge.new("memory_bytes_used", "Memory used in bytes")
memGauge:set(nil, 1024 * 1024 * 512)  -- 512 MB
print(string.format("\nMemory: %.0f bytes (%.1f MB)",
    memGauge:get(), memGauge:get() / 1024 / 1024))
```

---

## 67.4 Histogram

Histogram วัดการกระจายตัวของค่า เช่น request latency

```lua
-- ตัวอย่างที่ 4: Histogram implementation
local Histogram = {}
Histogram.__index = Histogram

local DEFAULT_BUCKETS = {0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10}

function Histogram.new(name, help, buckets, labelNames)
    local self       = setmetatable({}, Histogram)
    self.name        = name
    self.help        = help
    self.labelNames  = labelNames or {}
    self.buckets     = buckets or DEFAULT_BUCKETS
    self.data        = {}  -- key -> {buckets, sum, count}
    return self
end

function Histogram:_key(labels)
    if not labels or #self.labelNames == 0 then return "__default__" end
    local parts = {}
    for _, n in ipairs(self.labelNames) do
        table.insert(parts, tostring(labels[n] or ""))
    end
    return table.concat(parts, "|")
end

function Histogram:_initKey(key)
    if self.data[key] then return end
    local buckets = {}
    for _, b in ipairs(self.buckets) do
        buckets[b] = 0
    end
    buckets[math.huge] = 0  -- +Inf bucket
    self.data[key] = {buckets = buckets, sum = 0, count = 0}
end

function Histogram:observe(labels, value)
    local key
    if type(labels) == "number" then
        -- Shorthand: observe(value)
        value  = labels
        labels = nil
        key    = "__default__"
    else
        key = self:_key(labels)
    end

    self:_initKey(key)
    local d = self.data[key]
    d.sum   = d.sum + value
    d.count = d.count + 1

    for _, b in ipairs(self.buckets) do
        if value <= b then
            d.buckets[b] = d.buckets[b] + 1
        end
    end
    d.buckets[math.huge] = d.count  -- +Inf always = count
end

function Histogram:percentile(labels, p)
    local key = labels and self:_key(labels) or "__default__"
    local d   = self.data[key]
    if not d or d.count == 0 then return 0 end

    local target = math.ceil(p / 100 * d.count)
    for _, b in ipairs(self.buckets) do
        if d.buckets[b] >= target then
            return b
        end
    end
    return self.buckets[#self.buckets]
end

function Histogram:mean(labels)
    local key = labels and self:_key(labels) or "__default__"
    local d   = self.data[key]
    if not d or d.count == 0 then return 0 end
    return d.sum / d.count
end

function Histogram:collect(labels)
    local key = labels and self:_key(labels) or "__default__"
    local d   = self.data[key]
    if not d then return nil end
    return {
        buckets = d.buckets,
        sum     = d.sum,
        count   = d.count,
    }
end

-- ทดสอบ
math.randomseed(42)
local latencyHist = Histogram.new(
    "http_request_duration_seconds",
    "HTTP request latency",
    {0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5}
)

-- Simulate 200 requests with varying latency
for i = 1, 200 do
    -- Log-normal distribution simulation
    local u1     = math.random()
    local u2     = math.random()
    local normal = math.sqrt(-2 * math.log(u1)) * math.cos(2 * math.pi * u2)
    local latency = math.exp(normal * 0.5 - 2)  -- mean ~0.15s
    latencyHist:observe(math.max(0.001, latency))
end

local d = latencyHist:collect()
print(string.format("Histogram: count=%d, sum=%.3f, mean=%.3fs",
    d.count, d.sum, d.sum / d.count))
print(string.format("  p50=%.3fs, p95=%.3fs, p99=%.3fs",
    latencyHist:percentile(nil, 50),
    latencyHist:percentile(nil, 95),
    latencyHist:percentile(nil, 99)))
```

---

## 67.5 Summary

Summary คล้าย Histogram แต่คำนวณ quantile โดยตรง

```lua
-- ตัวอย่างที่ 5: Summary implementation
local Summary = {}
Summary.__index = Summary

function Summary.new(name, help, quantiles, maxAge)
    local self      = setmetatable({}, Summary)
    self.name       = name
    self.help       = help
    self.quantiles  = quantiles or {0.5, 0.9, 0.95, 0.99}
    self.maxAge     = maxAge or 600  -- 10 minutes sliding window
    self.samples    = {}
    self.sum        = 0
    self.count      = 0
    return self
end

function Summary:observe(value)
    local now = os.time()
    table.insert(self.samples, {value = value, time = now})
    self.sum   = self.sum + value
    self.count = self.count + 1

    -- Remove old samples
    local cutoff = now - self.maxAge
    local new    = {}
    for _, s in ipairs(self.samples) do
        if s.time >= cutoff then
            table.insert(new, s)
        end
    end
    self.samples = new
end

function Summary:calculateQuantile(q)
    if #self.samples == 0 then return 0 end
    local sorted = {}
    for _, s in ipairs(self.samples) do
        table.insert(sorted, s.value)
    end
    table.sort(sorted)
    local idx = math.ceil(q * #sorted)
    return sorted[math.max(1, idx)]
end

function Summary:collect()
    local result = {
        quantiles = {},
        sum       = self.sum,
        count     = self.count,
    }
    for _, q in ipairs(self.quantiles) do
        result.quantiles[q] = self:calculateQuantile(q)
    end
    return result
end

function Summary:prometheusText()
    local lines = {}
    table.insert(lines, "# HELP " .. self.name .. " " .. self.help)
    table.insert(lines, "# TYPE " .. self.name .. " summary")
    local data = self:collect()
    for q, v in pairs(data.quantiles) do
        table.insert(lines, string.format('%s{quantile="%g"} %g', self.name, q, v))
    end
    table.insert(lines, self.name .. "_sum " .. data.sum)
    table.insert(lines, self.name .. "_count " .. data.count)
    return table.concat(lines, "\n")
end

math.randomseed(123)
local summary = Summary.new(
    "rpc_duration_seconds",
    "RPC duration in seconds",
    {0.5, 0.9, 0.99}
)

for i = 1, 100 do
    summary:observe(math.random() * 2)
end

print(summary:prometheusText())
```

---

## 67.6 Metrics Registry

```lua
-- ตัวอย่างที่ 6: Metrics Registry
local Registry = {}
Registry.__index = Registry

function Registry.new()
    local self    = setmetatable({}, Registry)
    self.metrics  = {}
    return self
end

function Registry:register(metric)
    if self.metrics[metric.name] then
        error("Metric already registered: " .. metric.name)
    end
    self.metrics[metric.name] = metric
    return metric
end

function Registry:get(name)
    return self.metrics[name]
end

function Registry:newCounter(name, help, labelNames)
    return self:register(Counter.new(name, help, labelNames))
end

function Registry:newGauge(name, help, labelNames)
    return self:register(Gauge.new(name, help, labelNames))
end

function Registry:newHistogram(name, help, buckets, labelNames)
    return self:register(Histogram.new(name, help, buckets, labelNames))
end

function Registry:renderPrometheus()
    local lines = {}
    for name, m in pairs(self.metrics) do
        table.insert(lines, "# HELP " .. name .. " " .. (m.help or ""))
        -- Determine type
        local mtype = "untyped"
        if getmetatable(m) == Counter then
            mtype = "counter"
        elseif getmetatable(m) == Gauge then
            mtype = "gauge"
        elseif getmetatable(m) == Histogram then
            mtype = "histogram"
        end
        table.insert(lines, "# TYPE " .. name .. " " .. mtype)

        if m.collect then
            local samples = m:collect()
            if type(samples) == "table" and samples.count then
                -- Histogram
                table.insert(lines, name .. "_sum " .. tostring(samples.sum))
                table.insert(lines, name .. "_count " .. tostring(samples.count))
            elseif type(samples) == "table" then
                for _, s in ipairs(samples) do
                    local labelStr = ""
                    if s.labels and next(s.labels) then
                        local parts = {}
                        for k, v in pairs(s.labels) do
                            table.insert(parts, string.format('%s="%s"', k, v))
                        end
                        labelStr = "{" .. table.concat(parts, ",") .. "}"
                    end
                    table.insert(lines, name .. labelStr .. " " .. tostring(s.value))
                end
            end
        end
    end
    return table.concat(lines, "\n")
end

-- ทดสอบ Registry
local registry = Registry.new()

local httpReqs  = registry:newCounter("http_requests_total", "HTTP requests", {"method", "status"})
local activeConns = registry:newGauge("active_connections", "Active connections", {"backend"})
local reqDur    = registry:newHistogram("http_request_duration_seconds", "Request duration",
    {0.01, 0.05, 0.1, 0.5, 1.0})

httpReqs:inc({method = "GET",  status = "200"}, 150)
httpReqs:inc({method = "POST", status = "201"}, 45)
httpReqs:inc({method = "GET",  status = "500"}, 3)

activeConns:set({backend = "api"},    12)
activeConns:set({backend = "worker"}, 4)

for _ = 1, 50 do
    reqDur:observe(math.random() * 0.5)
end

print("=== Prometheus /metrics output ===")
print(registry:renderPrometheus())
```

---

## 67.7 Exposing /metrics Endpoint

```lua
-- ตัวอย่างที่ 7: HTTP /metrics endpoint (Pure Lua simulation)
local MetricsServer = {}
MetricsServer.__index = MetricsServer

function MetricsServer.new(registry, port)
    local self     = setmetatable({}, MetricsServer)
    self.registry  = registry
    self.port      = port or 9090
    self.routes    = {}
    return self
end

function MetricsServer:addRoute(path, handler)
    self.routes[path] = handler
end

function MetricsServer:handle(request)
    -- Simulate HTTP request handling
    local path    = request.path or "/"
    local method  = request.method or "GET"

    if path == "/metrics" and method == "GET" then
        return {
            status  = 200,
            headers = {
                ["Content-Type"] = "text/plain; version=0.0.4; charset=utf-8",
            },
            body = self.registry:renderPrometheus(),
        }
    elseif path == "/health" then
        return {
            status  = 200,
            headers = {["Content-Type"] = "application/json"},
            body    = '{"status":"healthy","timestamp":' .. os.time() .. '}',
        }
    elseif path == "/ready" then
        return {
            status  = 200,
            headers = {["Content-Type"] = "application/json"},
            body    = '{"ready":true}',
        }
    else
        local handler = self.routes[path]
        if handler then
            return handler(request)
        end
        return {status = 404, body = "Not Found"}
    end
end

-- Simulate server
local reg = Registry.new()
local counter = reg:newCounter("demo_requests", "Demo requests", {"endpoint"})
counter:inc({endpoint = "/api"}, 100)
counter:inc({endpoint = "/health"}, 50)

local server = MetricsServer.new(reg, 9090)

-- Simulate requests
local responses = {
    server:handle({method = "GET", path = "/metrics"}),
    server:handle({method = "GET", path = "/health"}),
    server:handle({method = "GET", path = "/unknown"}),
}

for i, resp in ipairs(responses) do
    print(string.format("\nRequest %d: HTTP %d", i, resp.status))
    if resp.headers then
        for k, v in pairs(resp.headers) do
            print(string.format("  %s: %s", k, v))
        end
    end
    -- Only print first 3 lines of body
    local lines = {}
    for line in resp.body:gmatch("[^\n]+") do
        table.insert(lines, line)
        if #lines >= 3 then break end
    end
    print("  Body: " .. table.concat(lines, " | "))
end
```

---

## 67.8 Application Performance Monitoring

```lua
-- ตัวอย่างที่ 8: APM - Application Performance Monitor
local APM = {}
APM.__index = APM

function APM.new(serviceName)
    local self        = setmetatable({}, APM)
    self.service      = serviceName
    self.spans        = {}
    self.activeSpans  = {}
    self.counters     = {}
    self.timers       = {}
    return self
end

function APM:startSpan(operationName, tags)
    local spanId = string.format("%08x", math.random(0x10000000, 0x7FFFFFFF))
    local span   = {
        id        = spanId,
        operation = operationName,
        tags      = tags or {},
        startTime = os.clock(),
        logs      = {},
    }
    self.activeSpans[spanId] = span
    return span
end

function APM:finishSpan(span, error)
    span.duration = os.clock() - span.startTime
    span.error    = error
    self.activeSpans[span.id] = nil
    table.insert(self.spans, span)

    -- Update metrics
    local key = span.operation
    if not self.timers[key] then
        self.timers[key] = {count = 0, total = 0, errors = 0, max = 0}
    end
    local t     = self.timers[key]
    t.count     = t.count + 1
    t.total     = t.total + span.duration
    t.max       = math.max(t.max, span.duration)
    if error then t.errors = t.errors + 1 end
end

function APM:logToSpan(span, message, fields)
    table.insert(span.logs, {
        time    = os.clock() - span.startTime,
        message = message,
        fields  = fields,
    })
end

function APM:incrementCounter(name, amount)
    self.counters[name] = (self.counters[name] or 0) + (amount or 1)
end

function APM:report()
    print(string.format("=== APM Report: %s ===", self.service))
    print("\nOperation timing:")
    for op, t in pairs(self.timers) do
        local avg    = t.count > 0 and (t.total / t.count) or 0
        local errPct = t.count > 0 and (t.errors / t.count * 100) or 0
        print(string.format(
            "  %-30s calls=%d avg=%.3fms max=%.3fms err=%.1f%%",
            op, t.count, avg * 1000, t.max * 1000, errPct))
    end
    print("\nCounters:")
    for name, v in pairs(self.counters) do
        print(string.format("  %s = %d", name, v))
    end
end

-- ทดสอบ
local apm = APM.new("my-service")

-- Simulate database queries
for i = 1, 10 do
    local span = apm:startSpan("db.query", {table = "users"})
    -- Simulate work
    local t = os.clock()
    while os.clock() - t < 0.001 do end  -- 1ms sleep simulation
    local isError = i == 5  -- 5th query fails
    apm:logToSpan(span, "SQL executed", {rows = math.random(1, 100)})
    apm:finishSpan(span, isError and "timeout" or nil)
    apm:incrementCounter("db.queries")
    if isError then apm:incrementCounter("db.errors") end
end

-- Simulate HTTP calls
for i = 1, 5 do
    local span = apm:startSpan("http.call", {url = "/api/data"})
    local t = os.clock()
    while os.clock() - t < 0.002 do end  -- 2ms
    apm:finishSpan(span, nil)
    apm:incrementCounter("http.calls")
end

apm:report()
```

---

## 67.9 Error Rate Tracking

```lua
-- ตัวอย่างที่ 9: Error Rate Tracker
local ErrorRateTracker = {}
ErrorRateTracker.__index = ErrorRateTracker

function ErrorRateTracker.new(windowSecs)
    local self       = setmetatable({}, ErrorRateTracker)
    self.windowSecs  = windowSecs or 60
    self.buckets     = {}  -- time buckets
    self.bucketSize  = 1   -- 1 second per bucket
    return self
end

function ErrorRateTracker:_bucket()
    return math.floor(os.time() / self.bucketSize)
end

function ErrorRateTracker:_cleanup()
    local cutoff = self:_bucket() - math.floor(self.windowSecs / self.bucketSize)
    for k in pairs(self.buckets) do
        if tonumber(k) < cutoff then
            self.buckets[k] = nil
        end
    end
end

function ErrorRateTracker:record(isError)
    local key = tostring(self:_bucket())
    if not self.buckets[key] then
        self.buckets[key] = {total = 0, errors = 0}
    end
    self.buckets[key].total  = self.buckets[key].total + 1
    if isError then
        self.buckets[key].errors = self.buckets[key].errors + 1
    end
    self:_cleanup()
end

function ErrorRateTracker:rate()
    local total  = 0
    local errors = 0
    for _, b in pairs(self.buckets) do
        total  = total + b.total
        errors = errors + b.errors
    end
    if total == 0 then return 0, 0, 0 end
    return errors / total, errors, total
end

function ErrorRateTracker:isBreaching(threshold)
    local rate = self:rate()
    return rate > threshold
end

local tracker = ErrorRateTracker.new(60)

-- Simulate traffic with some errors
math.randomseed(99)
for i = 1, 100 do
    local isError = math.random() < 0.07  -- 7% error rate
    tracker:record(isError)
end

local rate, errors, total = tracker:rate()
print(string.format("Error rate: %.2f%% (%d errors / %d total)",
    rate * 100, errors, total))

if tracker:isBreaching(0.05) then
    print("ALERT: Error rate exceeds 5% threshold!")
end
```

---

## 67.10 Latency Percentiles

```lua
-- ตัวอย่างที่ 10: Latency Percentile Calculator
local LatencyTracker = {}
LatencyTracker.__index = LatencyTracker

function LatencyTracker.new(maxSamples)
    local self         = setmetatable({}, LatencyTracker)
    self.maxSamples    = maxSamples or 1000
    self.samples       = {}
    self.sorted        = false
    self.sum           = 0
    self.count         = 0
    self.min           = math.huge
    self.max           = 0
    return self
end

function LatencyTracker:record(ms)
    table.insert(self.samples, ms)
    self.sorted = false
    self.sum    = self.sum + ms
    self.count  = self.count + 1
    self.min    = math.min(self.min, ms)
    self.max    = math.max(self.max, ms)

    -- Keep bounded
    if #self.samples > self.maxSamples then
        table.remove(self.samples, 1)
        -- Recalculate sum/min/max (simplified)
    end
end

function LatencyTracker:_sort()
    if not self.sorted then
        table.sort(self.samples)
        self.sorted = true
    end
end

function LatencyTracker:percentile(p)
    self:_sort()
    if #self.samples == 0 then return 0 end
    local idx = math.ceil(p / 100 * #self.samples)
    return self.samples[math.max(1, math.min(idx, #self.samples))]
end

function LatencyTracker:mean()
    if self.count == 0 then return 0 end
    return self.sum / self.count
end

function LatencyTracker:stddev()
    if #self.samples < 2 then return 0 end
    local mean = self:mean()
    local variance = 0
    for _, v in ipairs(self.samples) do
        variance = variance + (v - mean)^2
    end
    return math.sqrt(variance / #self.samples)
end

function LatencyTracker:report(name)
    print(string.format("=== Latency Report: %s ===", name or "default"))
    print(string.format("  count: %d", self.count))
    print(string.format("  min:   %.2f ms", self.min == math.huge and 0 or self.min))
    print(string.format("  max:   %.2f ms", self.max))
    print(string.format("  mean:  %.2f ms", self:mean()))
    print(string.format("  std:   %.2f ms", self:stddev()))
    print(string.format("  p50:   %.2f ms", self:percentile(50)))
    print(string.format("  p90:   %.2f ms", self:percentile(90)))
    print(string.format("  p95:   %.2f ms", self:percentile(95)))
    print(string.format("  p99:   %.2f ms", self:percentile(99)))
    print(string.format("  p999:  %.2f ms", self:percentile(99.9)))
end

-- ทดสอบ
math.randomseed(42)
local lt = LatencyTracker.new(500)

for i = 1, 200 do
    -- Normal distribution centered around 50ms with outliers
    local base = 50 + math.random(-20, 30)
    if math.random() < 0.05 then base = base + math.random(200, 500) end  -- outliers
    lt:record(math.max(1, base))
end

lt:report("HTTP API")
```

---

## 67.11 Throughput Measurement

```lua
-- ตัวอย่างที่ 11: Throughput/RPS Meter
local RateMeter = {}
RateMeter.__index = RateMeter

-- Exponential Moving Average
function RateMeter.new()
    local self       = setmetatable({}, RateMeter)
    self.count       = 0
    self.startTime   = os.clock()
    self.lastTime    = os.clock()
    self.lastCount   = 0
    -- EMA rates
    self.rate1m      = 0   -- 1-minute rate
    self.rate5m      = 0   -- 5-minute rate
    self.rate15m     = 0   -- 15-minute rate
    self._ticker     = 0
    return self
end

function RateMeter:mark(n)
    n = n or 1
    self.count = self.count + n
    self:_tick()
end

function RateMeter:_tick()
    local now     = os.clock()
    local elapsed = now - self.lastTime

    if elapsed >= 5 then  -- Update every 5 seconds
        local instantRate = (self.count - self.lastCount) / elapsed
        self.lastCount = self.count
        self.lastTime  = now

        -- EMA formula: rate = rate * alpha + instant * (1 - alpha)
        local alpha1m  = math.exp(-elapsed / 60)
        local alpha5m  = math.exp(-elapsed / 300)
        local alpha15m = math.exp(-elapsed / 900)

        if self.rate1m == 0 then
            self.rate1m  = instantRate
            self.rate5m  = instantRate
            self.rate15m = instantRate
        else
            self.rate1m  = self.rate1m  * alpha1m  + instantRate * (1 - alpha1m)
            self.rate5m  = self.rate5m  * alpha5m  + instantRate * (1 - alpha5m)
            self.rate15m = self.rate15m * alpha15m + instantRate * (1 - alpha15m)
        end
    end
end

function RateMeter:meanRate()
    local elapsed = os.clock() - self.startTime
    if elapsed == 0 then return 0 end
    return self.count / elapsed
end

function RateMeter:report()
    print(string.format("Total: %d, Mean rate: %.2f/s, 1m: %.2f/s, 5m: %.2f/s, 15m: %.2f/s",
        self.count,
        self:meanRate(),
        self.rate1m,
        self.rate5m,
        self.rate15m))
end

local meter = RateMeter.new()

-- Simulate burst traffic
for i = 1, 500 do
    meter:mark()
end

meter:report()
```

---

## 67.12 Custom Metrics in OpenResty

```lua
-- ตัวอย่างที่ 12: OpenResty custom metrics (simulation)
--[[
-- ใน nginx.conf:
-- lua_shared_dict metrics 10m;
--]]

-- Simulate ngx.shared.DICT API
local SharedDict = {}
SharedDict.__index = SharedDict

function SharedDict.new()
    local self  = setmetatable({}, SharedDict)
    self.store  = {}
    self.expiry = {}
    return self
end

function SharedDict:set(key, value, exptime)
    self.store[key]  = value
    if exptime then
        self.expiry[key] = os.time() + exptime
    end
end

function SharedDict:get(key)
    if self.expiry[key] and os.time() > self.expiry[key] then
        self.store[key]  = nil
        self.expiry[key] = nil
        return nil
    end
    return self.store[key]
end

function SharedDict:incr(key, value, initValue)
    local current = self:get(key)
    if current == nil then
        current = initValue or 0
    end
    local new = current + (value or 1)
    self.store[key] = new
    return new
end

function SharedDict:delete(key)
    self.store[key]  = nil
    self.expiry[key] = nil
end

-- OpenResty-style metrics using shared dict
local ngx_shared_metrics = SharedDict.new()

-- Simulate access_log_by_lua_block
local function recordRequestMetrics(method, status, latencyMs, upstream)
    local prefix = "req"
    ngx_shared_metrics:incr(prefix .. ":total", 1, 0)
    ngx_shared_metrics:incr(prefix .. ":method:" .. method, 1, 0)
    ngx_shared_metrics:incr(prefix .. ":status:" .. tostring(status), 1, 0)
    if upstream then
        ngx_shared_metrics:incr(prefix .. ":upstream:" .. upstream, 1, 0)
    end
    -- Accumulate latency
    ngx_shared_metrics:incr(prefix .. ":latency_sum", latencyMs, 0)
    ngx_shared_metrics:incr(prefix .. ":latency_count", 1, 0)
end

-- Simulate content_by_lua_block for /metrics
local function metricsHandler()
    local lines = {}
    local total = ngx_shared_metrics:get("req:total") or 0
    table.insert(lines, "# HELP openresty_requests_total Total requests")
    table.insert(lines, "# TYPE openresty_requests_total counter")
    table.insert(lines, "openresty_requests_total " .. total)

    local latSum   = ngx_shared_metrics:get("req:latency_sum") or 0
    local latCount = ngx_shared_metrics:get("req:latency_count") or 0
    if latCount > 0 then
        table.insert(lines, "# HELP openresty_latency_avg_ms Average latency")
        table.insert(lines, "# TYPE openresty_latency_avg_ms gauge")
        table.insert(lines, string.format("openresty_latency_avg_ms %.2f",
            latSum / latCount))
    end
    return table.concat(lines, "\n")
end

-- Simulate requests
local methods  = {"GET", "POST", "PUT", "DELETE"}
local statuses = {200, 200, 200, 201, 404, 500}
local upstreams = {"app-1", "app-2", "app-3"}

math.randomseed(77)
for i = 1, 50 do
    recordRequestMetrics(
        methods[math.random(#methods)],
        statuses[math.random(#statuses)],
        math.random(5, 200),
        upstreams[math.random(#upstreams)]
    )
end

print("=== OpenResty /metrics simulation ===")
print(metricsHandler())
```

---

## 67.13 Health Check Endpoints

```lua
-- ตัวอย่างที่ 13: Comprehensive Health Check
local HealthCheck = {}
HealthCheck.__index = HealthCheck

local STATUS = {
    PASS    = "pass",
    FAIL    = "fail",
    WARN    = "warn",
    UNKNOWN = "unknown",
}

function HealthCheck.new(serviceName, version)
    local self     = setmetatable({}, HealthCheck)
    self.service   = serviceName
    self.version   = version
    self.checks    = {}
    self.startTime = os.time()
    return self
end

function HealthCheck:addCheck(name, fn, critical)
    table.insert(self.checks, {
        name     = name,
        fn       = fn,
        critical = critical ~= false,
    })
end

function HealthCheck:run()
    local results  = {}
    local overall  = STATUS.PASS
    local startMs  = os.clock()

    for _, check in ipairs(self.checks) do
        local t0   = os.clock()
        local ok, result = pcall(check.fn)
        local dur  = (os.clock() - t0) * 1000

        local status, message, data
        if ok then
            if type(result) == "table" then
                status  = result.status  or STATUS.PASS
                message = result.message or "ok"
                data    = result.data
            elseif result == false then
                status  = STATUS.FAIL
                message = "check returned false"
            else
                status  = STATUS.PASS
                message = tostring(result)
            end
        else
            status  = STATUS.FAIL
            message = tostring(result)
        end

        results[check.name] = {
            status         = status,
            message        = message,
            durationMs     = dur,
            data           = data,
        }

        if status == STATUS.FAIL and check.critical then
            overall = STATUS.FAIL
        elseif status == STATUS.WARN and overall == STATUS.PASS then
            overall = STATUS.WARN
        end
    end

    local totalMs = (os.clock() - startMs) * 1000
    return {
        status    = overall,
        service   = self.service,
        version   = self.version,
        uptime    = os.time() - self.startTime,
        checks    = results,
        durationMs = totalMs,
    }
end

function HealthCheck:toJSON(report)
    local checkParts = {}
    for name, c in pairs(report.checks) do
        table.insert(checkParts, string.format(
            '"%s":{"status":"%s","message":"%s","durationMs":%.2f}',
            name, c.status, c.message, c.durationMs))
    end
    return string.format(
        '{"status":"%s","service":"%s","version":"%s","uptime":%d,"checks":{%s},"totalMs":%.2f}',
        report.status, report.service, report.version or "unknown",
        report.uptime, table.concat(checkParts, ","), report.durationMs)
end

-- ทดสอบ
local hc = HealthCheck.new("order-service", "1.2.3")

hc:addCheck("database", function()
    -- Simulate DB check
    return {status = "pass", message = "Connected to PostgreSQL"}
end)

hc:addCheck("cache", function()
    -- Simulate Redis check
    return {status = "pass", message = "Redis ping OK", data = {latencyMs = 0.5}}
end)

hc:addCheck("payment-api", function()
    -- Simulate external API check
    local latency = math.random(50, 300)
    if latency > 250 then
        return {status = "warn", message = string.format("High latency: %dms", latency)}
    end
    return {status = "pass", message = string.format("OK (%dms)", latency)}
end)

hc:addCheck("disk_space", function()
    local free = math.random(5, 80)
    if free < 10 then
        return {status = "fail", message = string.format("Low disk: %d%% free", free)}
    end
    return {status = "pass", message = string.format("%d%% free", free)}
end, false)  -- not critical

local report = hc:run()
print(hc:toJSON(report))
print(string.format("\nOverall status: %s", report.status))
```

---

## 67.14 Alerting Concepts

```lua
-- ตัวอย่างที่ 14: Alert Manager
local AlertManager = {}
AlertManager.__index = AlertManager

local SEVERITY = {
    INFO     = 1,
    WARNING  = 2,
    CRITICAL = 3,
}

function AlertManager.new()
    local self    = setmetatable({}, AlertManager)
    self.rules    = {}
    self.active   = {}   -- active alerts
    self.handlers = {}   -- notification handlers
    return self
end

function AlertManager:addRule(rule)
    table.insert(self.rules, {
        name      = rule.name,
        condition = rule.condition,
        severity  = rule.severity or SEVERITY.WARNING,
        message   = rule.message or rule.name,
        for_      = rule.for_ or 0,  -- seconds before firing
        _pending  = nil,
    })
end

function AlertManager:onAlert(fn)
    table.insert(self.handlers, fn)
end

function AlertManager:evaluate(metrics)
    for _, rule in ipairs(self.rules) do
        local firing = rule.condition(metrics)

        if firing then
            if not rule._pending then
                rule._pending = os.time()
            end
            local pendingFor = os.time() - rule._pending
            if pendingFor >= rule.for_ then
                -- Alert fires
                if not self.active[rule.name] then
                    self.active[rule.name] = {
                        rule      = rule,
                        firedAt   = os.time(),
                        severity  = rule.severity,
                        message   = rule.message,
                    }
                    -- Notify handlers
                    for _, h in ipairs(self.handlers) do
                        h("FIRING", self.active[rule.name])
                    end
                end
            end
        else
            if self.active[rule.name] then
                -- Alert resolved
                local alert = self.active[rule.name]
                self.active[rule.name] = nil
                rule._pending = nil
                for _, h in ipairs(self.handlers) do
                    h("RESOLVED", alert)
                end
            else
                rule._pending = nil
            end
        end
    end
end

function AlertManager:activeAlerts()
    local result = {}
    for name, alert in pairs(self.active) do
        table.insert(result, alert)
    end
    return result
end

-- ทดสอบ
local am = AlertManager.new()

am:addRule({
    name      = "HighErrorRate",
    severity  = SEVERITY.CRITICAL,
    message   = "Error rate exceeds 5%",
    for_      = 0,
    condition = function(m) return (m.errorRate or 0) > 0.05 end,
})

am:addRule({
    name      = "HighLatency",
    severity  = SEVERITY.WARNING,
    message   = "P99 latency exceeds 500ms",
    for_      = 0,
    condition = function(m) return (m.p99Latency or 0) > 500 end,
})

am:addRule({
    name      = "LowThroughput",
    severity  = SEVERITY.INFO,
    message   = "RPS below expected minimum",
    for_      = 0,
    condition = function(m) return (m.rps or 0) < 10 end,
})

am:onAlert(function(state, alert)
    print(string.format("[ALERT %s] %s: %s",
        state, alert.rule.name, alert.message))
end)

print("=== Alert Evaluation ===")
am:evaluate({errorRate = 0.08, p99Latency = 300, rps = 100})  -- High error
am:evaluate({errorRate = 0.02, p99Latency = 600, rps = 100})  -- High latency
am:evaluate({errorRate = 0.02, p99Latency = 300, rps = 5})    -- Low throughput
am:evaluate({errorRate = 0.02, p99Latency = 300, rps = 100})  -- All clear

print("\nActive alerts: " .. #am:activeAlerts())
```

---

## 67.15 Grafana Dashboard Concepts

```lua
-- ตัวอย่างที่ 15: Dashboard Definition Generator (JSON structure)
local GrafanaDashboard = {}
GrafanaDashboard.__index = GrafanaDashboard

function GrafanaDashboard.new(title)
    local self   = setmetatable({}, GrafanaDashboard)
    self.title   = title
    self.panels  = {}
    self.uid     = string.format("%08x", math.random(0xFFFFFFFF))
    return self
end

function GrafanaDashboard:addPanel(panel)
    panel.id = #self.panels + 1
    table.insert(self.panels, panel)
    return self
end

function GrafanaDashboard:stat(title, expr, unit)
    return self:addPanel({
        type  = "stat",
        title = title,
        targets = {{expr = expr}},
        options = {unit = unit or "short"},
    })
end

function GrafanaDashboard:graph(title, targets)
    return self:addPanel({
        type    = "timeseries",
        title   = title,
        targets = targets,
    })
end

function GrafanaDashboard:table(title, targets)
    return self:addPanel({
        type    = "table",
        title   = title,
        targets = targets,
    })
end

function GrafanaDashboard:toJSON()
    -- Simplified JSON representation
    local panelLines = {}
    for _, p in ipairs(self.panels) do
        local targetLines = {}
        for _, t in ipairs(p.targets or {}) do
            table.insert(targetLines, string.format('{"expr":"%s"}', t.expr))
        end
        table.insert(panelLines, string.format(
            '{"id":%d,"type":"%s","title":"%s","targets":[%s]}',
            p.id, p.type, p.title, table.concat(targetLines, ",")))
    end
    return string.format(
        '{"title":"%s","uid":"%s","panels":[%s]}',
        self.title, self.uid, table.concat(panelLines, ","))
end

local dash = GrafanaDashboard.new("My Service Overview")

dash:stat("Request Rate",    'rate(http_requests_total[5m])',             "reqps")
dash:stat("Error Rate",      'rate(http_errors_total[5m])',               "reqps")
dash:stat("P99 Latency",     'histogram_quantile(0.99, rate(...))',        "ms")
dash:stat("Active Connections", 'active_connections',                     "short")

dash:graph("Request Rate Over Time", {
    {expr = 'rate(http_requests_total{status="200"}[5m])'},
    {expr = 'rate(http_requests_total{status="5xx"}[5m])'},
})

dash:graph("Latency Percentiles", {
    {expr = 'histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))'},
    {expr = 'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))'},
    {expr = 'histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))'},
})

print("Dashboard JSON preview:")
local json = dash:toJSON()
print(json:sub(1, 200) .. "...")
print(string.format("\nPanels: %d", #dash.panels))
```

---

## 67.16 OpenTelemetry Concepts

```lua
-- ตัวอย่างที่ 16: OpenTelemetry Trace/Span simulation
local Tracer = {}
Tracer.__index = Tracer

local Span = {}
Span.__index = Span

function Span.new(tracer, name, parentId)
    local self         = setmetatable({}, Span)
    self.tracer        = tracer
    self.name          = name
    self.traceId       = parentId and
        tracer:_findTrace(parentId) or
        tracer:_newTraceId()
    self.spanId        = tracer:_newSpanId()
    self.parentSpanId  = parentId
    self.startTime     = os.clock()
    self.endTime       = nil
    self.attributes    = {}
    self.events        = {}
    self.status        = "UNSET"
    self.statusMessage = ""
    return self
end

function Span:setAttribute(key, value)
    self.attributes[key] = value
    return self
end

function Span:addEvent(name, attributes)
    table.insert(self.events, {
        name       = name,
        time       = os.clock() - self.startTime,
        attributes = attributes or {},
    })
    return self
end

function Span:setStatus(status, message)
    self.status        = status  -- "OK", "ERROR", "UNSET"
    self.statusMessage = message or ""
    return self
end

function Span:finish()
    self.endTime = os.clock()
    self.tracer:_recordSpan(self)
    return self
end

function Span:duration()
    local endT = self.endTime or os.clock()
    return (endT - self.startTime) * 1000  -- ms
end

function Tracer.new(serviceName)
    local self        = setmetatable({}, Tracer)
    self.service      = serviceName
    self.spans        = {}
    self.traceMap     = {}  -- spanId -> traceId
    return self
end

function Tracer:_newTraceId()
    return string.format("%016x%016x",
        math.random(0, 0x7FFFFFFF), math.random(0, 0x7FFFFFFF))
end

function Tracer:_newSpanId()
    return string.format("%016x", math.random(0, 0x7FFFFFFF))
end

function Tracer:_findTrace(parentSpanId)
    return self.traceMap[parentSpanId]
end

function Tracer:_recordSpan(span)
    table.insert(self.spans, span)
    self.traceMap[span.spanId] = span.traceId
end

function Tracer:startSpan(name, parentSpanId)
    return Span.new(self, name, parentSpanId)
end

function Tracer:getTrace(traceId)
    local result = {}
    for _, s in ipairs(self.spans) do
        if s.traceId == traceId then
            table.insert(result, s)
        end
    end
    return result
end

function Tracer:printTrace(traceId)
    local spans = self:getTrace(traceId)
    if #spans == 0 then
        print("No spans found for trace " .. traceId)
        return
    end
    table.sort(spans, function(a, b) return a.startTime < b.startTime end)
    print(string.format("=== Trace: %s ===", traceId:sub(1, 16) .. "..."))
    for _, s in ipairs(spans) do
        local indent = s.parentSpanId and "  " or ""
        print(string.format("%s[%s] %s (%.2fms) [%s]",
            indent, s.spanId:sub(1, 8), s.name, s:duration(), s.status))
        for k, v in pairs(s.attributes) do
            print(string.format("%s  %s=%s", indent, k, tostring(v)))
        end
    end
end

-- ทดสอบ
local tracer = Tracer.new("checkout-service")

-- Root span: HTTP request
local rootSpan = tracer:startSpan("POST /api/checkout")
rootSpan:setAttribute("http.method", "POST")
rootSpan:setAttribute("http.url", "/api/checkout")
rootSpan:setAttribute("user.id", "user-1234")

    -- Child span: validate cart
    local validateSpan = tracer:startSpan("validate_cart", rootSpan.spanId)
    validateSpan:setAttribute("cart.items", 3)
    validateSpan:addEvent("cart_validated")
    validateSpan:setStatus("OK")
    validateSpan:finish()

    -- Child span: charge payment
    local paymentSpan = tracer:startSpan("charge_payment", rootSpan.spanId)
    paymentSpan:setAttribute("payment.method", "credit_card")
    paymentSpan:setAttribute("payment.amount", 99.99)
    paymentSpan:addEvent("payment_initiated")
    paymentSpan:setStatus("OK")
    paymentSpan:finish()

    -- Child span: update inventory
    local inventorySpan = tracer:startSpan("update_inventory", rootSpan.spanId)
    inventorySpan:setAttribute("items.count", 3)
    inventorySpan:finish()

rootSpan:setAttribute("http.status_code", 200)
rootSpan:setStatus("OK")
rootSpan:finish()

tracer:printTrace(rootSpan.traceId)
```

---

## 67.17 Metrics Middleware สำหรับ Web Framework

```lua
-- ตัวอย่างที่ 17: Metrics middleware
local MetricsMiddleware = {}
MetricsMiddleware.__index = MetricsMiddleware

function MetricsMiddleware.new(registry)
    local self      = setmetatable({}, MetricsMiddleware)
    self.registry   = registry or Registry.new()

    -- Register standard metrics
    self.requests = self.registry:newCounter(
        "http_requests_total",
        "Total HTTP requests",
        {"method", "path", "status"}
    )
    self.duration = self.registry:newHistogram(
        "http_request_duration_seconds",
        "HTTP request duration",
        {0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
        {"method", "path"}
    )
    self.active = self.registry:newGauge(
        "http_requests_in_flight",
        "Current in-flight requests"
    )
    return self
end

function MetricsMiddleware:wrap(handler)
    local self = self
    return function(req)
        local startTime = os.clock()
        self.active:inc()

        -- Call actual handler
        local ok, resp = pcall(handler, req)
        local duration = os.clock() - startTime

        local status = "500"
        if ok and resp then
            status = tostring(resp.status or 200)
        end

        -- Record metrics
        self.requests:inc({
            method = req.method or "GET",
            path   = req.path or "/",
            status = status,
        })
        self.duration:observe({
            method = req.method or "GET",
            path   = req.path or "/",
        }, duration)
        self.active:dec()

        if ok then return resp end
        return {status = 500, body = "Internal Server Error"}
    end
end

-- ทดสอบ
local mw  = MetricsMiddleware.new()
local reg = mw.registry

-- Define routes
local function usersHandler(req)
    return {status = 200, body = '{"users":[]}'}
end

local function ordersHandler(req)
    if math.random() < 0.1 then
        error("Database connection failed")
    end
    return {status = 200, body = '{"orders":[]}'}
end

-- Wrap with metrics
local wrappedUsers  = mw:wrap(usersHandler)
local wrappedOrders = mw:wrap(ordersHandler)

math.randomseed(55)
for i = 1, 30 do
    wrappedUsers({method = "GET", path = "/api/users"})
end
for i = 1, 20 do
    wrappedOrders({method = "GET", path = "/api/orders"})
end

print("=== Metrics after 50 requests ===")
print(reg:renderPrometheus())
```

---

## 67.18 SLO (Service Level Objectives) Tracking

```lua
-- ตัวอย่างที่ 18: SLO Tracker
local SLOTracker = {}
SLOTracker.__index = SLOTracker

function SLOTracker.new(name, config)
    local self          = setmetatable({}, SLOTracker)
    self.name           = name
    self.target         = config.target or 0.999   -- 99.9%
    self.latencyTarget  = config.latencyTarget      -- ms
    self.windowDays     = config.windowDays or 30
    self.good           = 0
    self.total          = 0
    self.errorBudget    = 1 - self.target
    return self
end

function SLOTracker:record(success, latencyMs)
    self.total = self.total + 1
    local isGood = success
    if latencyMs and self.latencyTarget then
        isGood = isGood and (latencyMs <= self.latencyTarget)
    end
    if isGood then
        self.good = self.good + 1
    end
end

function SLOTracker:compliance()
    if self.total == 0 then return 1.0 end
    return self.good / self.total
end

function SLOTracker:errorBudgetRemaining()
    local actualErrorRate  = 1 - self:compliance()
    local budgetConsumed   = actualErrorRate / self.errorBudget
    return math.max(0, 1 - budgetConsumed)
end

function SLOTracker:report()
    local compliance = self:compliance()
    local budgetLeft = self:errorBudgetRemaining()
    local status

    if compliance >= self.target then
        status = "MET"
    elseif budgetLeft > 0.5 then
        status = "AT RISK"
    else
        status = "BREACHED"
    end

    print(string.format("=== SLO: %s ===", self.name))
    print(string.format("  Target:         %.3f%%", self.target * 100))
    print(string.format("  Actual:         %.3f%%", compliance * 100))
    print(string.format("  Error budget:   %.2f%% remaining", budgetLeft * 100))
    print(string.format("  Requests:       %d total, %d good",
        self.total, self.good))
    print(string.format("  Status:         %s", status))
end

-- ทดสอบ
local slo = SLOTracker.new("API Availability", {
    target        = 0.999,
    latencyTarget = 200,
})

math.randomseed(88)
for i = 1, 10000 do
    local success  = math.random() > 0.001  -- 99.9% success
    local latency  = math.random(10, 500)
    slo:record(success, latency)
end

slo:report()
```

---

## 67.19 Distributed Metrics Aggregation

```lua
-- ตัวอย่างที่ 19: Metrics Aggregator
local MetricsAggregator = {}
MetricsAggregator.__index = MetricsAggregator

function MetricsAggregator.new()
    local self      = setmetatable({}, MetricsAggregator)
    self.nodes      = {}   -- node -> metrics snapshot
    self.aggregated = {}
    return self
end

function MetricsAggregator:push(nodeId, metrics)
    self.nodes[nodeId] = {
        metrics   = metrics,
        updatedAt = os.time(),
    }
end

function MetricsAggregator:aggregate()
    local result = {counters = {}, gauges = {}}

    for nodeId, node in pairs(self.nodes) do
        -- Sum counters
        if node.metrics.counters then
            for k, v in pairs(node.metrics.counters) do
                result.counters[k] = (result.counters[k] or 0) + v
            end
        end
        -- Average gauges
        if node.metrics.gauges then
            for k, v in pairs(node.metrics.gauges) do
                if not result.gauges[k] then
                    result.gauges[k] = {sum = 0, count = 0}
                end
                result.gauges[k].sum   = result.gauges[k].sum + v
                result.gauges[k].count = result.gauges[k].count + 1
            end
        end
    end

    -- Compute gauge averages
    for k, v in pairs(result.gauges) do
        result.gauges[k] = v.count > 0 and (v.sum / v.count) or 0
    end

    self.aggregated = result
    return result
end

function MetricsAggregator:report()
    local agg = self:aggregate()
    print("=== Aggregated Metrics ===")
    print("Counters:")
    for k, v in pairs(agg.counters) do
        print(string.format("  %s = %d", k, v))
    end
    print("Gauges (average):")
    for k, v in pairs(agg.gauges) do
        print(string.format("  %s = %.2f", k, v))
    end
end

local aggregator = MetricsAggregator.new()

-- Simulate 3 nodes reporting metrics
for nodeId = 1, 3 do
    aggregator:push("node-" .. nodeId, {
        counters = {
            requests = math.random(1000, 5000),
            errors   = math.random(10, 50),
        },
        gauges = {
            cpu_percent    = math.random(20, 80),
            memory_percent = math.random(40, 90),
            connections    = math.random(10, 200),
        }
    })
end

aggregator:report()
```

---

## 67.20 Metric Alert Rules (PromQL-like)

```lua
-- ตัวอย่างที่ 20: Alert Rule Engine
local AlertRuleEngine = {}
AlertRuleEngine.__index = AlertRuleEngine

function AlertRuleEngine.new()
    local self    = setmetatable({}, AlertRuleEngine)
    self.rules    = {}
    self.alerts   = {}
    return self
end

function AlertRuleEngine:addRule(rule)
    table.insert(self.rules, {
        name        = rule.name,
        expr        = rule.expr,
        for_        = rule.for_ or 0,
        labels      = rule.labels or {},
        annotations = rule.annotations or {},
        _pending    = nil,
    })
end

function AlertRuleEngine:evaluate(metricFn, currentTime)
    currentTime = currentTime or os.time()
    local fired = {}

    for _, rule in ipairs(self.rules) do
        local value = metricFn(rule.expr)
        local active = value ~= nil and value ~= false

        if active then
            if not rule._pending then
                rule._pending = currentTime
            end
            if (currentTime - rule._pending) >= rule.for_ then
                local alertKey = rule.name
                if not self.alerts[alertKey] then
                    self.alerts[alertKey] = {
                        name        = rule.name,
                        labels      = rule.labels,
                        annotations = rule.annotations,
                        firedAt     = currentTime,
                        value       = value,
                    }
                    table.insert(fired, self.alerts[alertKey])
                end
            end
        else
            rule._pending = nil
            if self.alerts[rule.name] then
                self.alerts[rule.name] = nil
            end
        end
    end
    return fired
end

-- ทดสอบ
local engine = AlertRuleEngine.new()

engine:addRule({
    name   = "HighCPU",
    expr   = "cpu_usage_percent > 80",
    for_   = 0,
    labels = {severity = "warning"},
    annotations = {summary = "CPU usage is above 80%"},
})

engine:addRule({
    name   = "ServiceDown",
    expr   = "up == 0",
    for_   = 0,
    labels = {severity = "critical"},
    annotations = {summary = "Service is down"},
})

-- Simulate metric values
local function getMetric(expr)
    local metrics = {
        ["cpu_usage_percent > 80"] = 85,
        ["up == 0"]                = nil,  -- service is up
    }
    return metrics[expr]
end

local fired = engine:evaluate(getMetric)
if #fired > 0 then
    print("Fired alerts:")
    for _, alert in ipairs(fired) do
        print(string.format("  [%s] %s: %s",
            alert.labels.severity or "info",
            alert.name,
            alert.annotations.summary or ""))
    end
else
    print("No alerts fired")
end
```

---

## 67.21 Complete Observability Setup

```lua
-- ตัวอย่างที่ 21: Observability Stack
local ObservabilityStack = {}
ObservabilityStack.__index = ObservabilityStack

function ObservabilityStack.new(serviceName)
    local self     = setmetatable({}, ObservabilityStack)
    self.service   = serviceName
    self.registry  = Registry.new()
    self.tracer    = Tracer.new(serviceName)
    self.logs      = {}
    return self
end

function ObservabilityStack:counter(name, help, labels)
    return self.registry:newCounter(name, help, labels)
end

function ObservabilityStack:gauge(name, help, labels)
    return self.registry:newGauge(name, help, labels)
end

function ObservabilityStack:histogram(name, help, buckets, labels)
    return self.registry:newHistogram(name, help, buckets, labels)
end

function ObservabilityStack:span(name, parentId)
    return self.tracer:startSpan(name, parentId)
end

function ObservabilityStack:log(level, message, fields)
    local entry = {
        timestamp = os.time(),
        level     = level,
        service   = self.service,
        message   = message,
        fields    = fields or {},
    }
    table.insert(self.logs, entry)
    print(string.format("[%s] %s - %s", level, self.service, message))
end

function ObservabilityStack:info(msg, fields)  return self:log("INFO",  msg, fields) end
function ObservabilityStack:warn(msg, fields)  return self:log("WARN",  msg, fields) end
function ObservabilityStack:error(msg, fields) return self:log("ERROR", msg, fields) end

-- ทดสอบ
local obs = ObservabilityStack.new("payment-service")

local requests  = obs:counter("payment_requests_total", "Payment requests", {"status"})
local latency   = obs:histogram("payment_duration_ms", "Payment duration", {10, 50, 100, 500})
local balance   = obs:gauge("payment_queue_size", "Payment queue size")

balance:set(nil, 0)

-- Process payments
for i = 1, 5 do
    local span = obs:span("process_payment")
    span:setAttribute("payment.id", "pay-" .. i)

    local t0    = os.clock()
    obs:info("Processing payment", {id = "pay-" .. i})

    -- Simulate processing
    local success = math.random() > 0.2
    local dur     = math.random(20, 200)

    if success then
        requests:inc({status = "success"})
        obs:info("Payment completed", {id = "pay-" .. i, duration_ms = dur})
        span:setStatus("OK")
    else
        requests:inc({status = "failed"})
        obs:error("Payment failed", {id = "pay-" .. i, reason = "insufficient_funds"})
        span:setStatus("ERROR", "insufficient_funds")
    end

    latency:observe(dur)
    balance:dec()
    span:finish()
end

print("\n=== Observability Summary ===")
print("Traces: " .. #obs.tracer.spans)
print("Logs:   " .. #obs.logs)
```

---

## 67.22 Metrics Export Formats

```lua
-- ตัวอย่างที่ 22: Multiple export formats
local MetricsExporter = {}
MetricsExporter.__index = MetricsExporter

function MetricsExporter.new(registry)
    local self      = setmetatable({}, MetricsExporter)
    self.registry   = registry
    return self
end

function MetricsExporter:toPrometheus()
    return self.registry:renderPrometheus()
end

function MetricsExporter:toInfluxLine(measurement, tags, fields, timestamp)
    -- InfluxDB line protocol: measurement,tag=val field=val timestamp
    local tagStr = {}
    for k, v in pairs(tags or {}) do
        table.insert(tagStr, k .. "=" .. tostring(v):gsub(" ", "_"))
    end
    local fieldStr = {}
    for k, v in pairs(fields or {}) do
        if type(v) == "number" then
            table.insert(fieldStr, k .. "=" .. tostring(v))
        else
            table.insert(fieldStr, k .. '="' .. tostring(v) .. '"')
        end
    end
    local line = measurement
    if #tagStr > 0 then
        line = line .. "," .. table.concat(tagStr, ",")
    end
    line = line .. " " .. table.concat(fieldStr, ",")
    if timestamp then
        line = line .. " " .. tostring(timestamp * 1000000000)  -- nanoseconds
    end
    return line
end

function MetricsExporter:toDatadog(name, value, tags)
    -- Datadog StatsD format
    local tagStr = ""
    if tags and next(tags) then
        local parts = {}
        for k, v in pairs(tags) do
            table.insert(parts, k .. ":" .. tostring(v))
        end
        tagStr = "|#" .. table.concat(parts, ",")
    end
    return string.format("%s:%s|g%s", name, tostring(value), tagStr)
end

local exporter = MetricsExporter.new(Registry.new())

print("=== InfluxDB Line Protocol ===")
print(exporter:toInfluxLine(
    "http_requests",
    {method = "GET", status = "200", host = "app-1"},
    {value = 1524, duration_ms = 45.2},
    os.time()
))

print("\n=== Datadog StatsD ===")
print(exporter:toDatadog("http.requests", 1524, {env = "prod", service = "api"}))
print(exporter:toDatadog("http.latency.p99", 230, {env = "prod"}))
```

---

## 67.23 สรุป Monitoring Patterns

```lua
-- ตัวอย่างที่ 23: USE Method monitoring
-- USE: Utilization, Saturation, Errors

local USEMonitor = {}
USEMonitor.__index = USEMonitor

function USEMonitor.new(resource)
    local self       = setmetatable({}, USEMonitor)
    self.resource    = resource
    self.utilization = 0  -- % busy
    self.saturation  = 0  -- queue length
    self.errors      = 0  -- error count
    self.total       = 0
    return self
end

function USEMonitor:update(util, sat, errors)
    self.utilization = util
    self.saturation  = sat
    self.errors      = self.errors + (errors or 0)
    self.total       = self.total + 1
end

function USEMonitor:report()
    print(string.format("USE Report - %s:", self.resource))
    print(string.format("  Utilization: %.1f%%", self.utilization))
    print(string.format("  Saturation:  %.1f (queue depth)", self.saturation))
    print(string.format("  Errors:      %d", self.errors))

    -- Diagnosis
    if self.utilization > 80 then
        print("  ⚠ High utilization - consider scaling")
    end
    if self.saturation > 10 then
        print("  ⚠ High saturation - requests queueing")
    end
    if self.errors > 0 then
        print(string.format("  ⚠ Errors detected: %d", self.errors))
    end
end

-- RED method: Rate, Errors, Duration
local function redReport(name, rate, errorRate, durationP99)
    print(string.format("RED Report - %s:", name))
    print(string.format("  Rate:     %.1f req/s", rate))
    print(string.format("  Errors:   %.2f%%", errorRate * 100))
    print(string.format("  Duration: %.1f ms (p99)", durationP99))
end

local cpu = USEMonitor.new("CPU")
cpu:update(72, 0.5, 0)
cpu:report()

print()
redReport("HTTP API", 450.5, 0.023, 187.3)
```

---

## สรุปบทที่ 67

ในบทนี้เราได้เรียนรู้:

1. **Prometheus Format** - text-based metrics exposition
2. **Counter/Gauge/Histogram/Summary** - metric types
3. **Metrics Registry** - จัดการ metrics collection
4. **APM** - Application Performance Monitoring
5. **Error Rate Tracking** - วัดอัตรา error
6. **Latency Percentiles** - p50, p95, p99, p999
7. **Throughput** - RPS measurement ด้วย EMA
8. **Health Check** - /health และ /ready endpoints
9. **Alerting** - Rule-based alerting
10. **Grafana** - Dashboard definition
11. **OpenTelemetry** - Distributed tracing
12. **SLO Tracking** - Error budget management
13. **USE/RED Methods** - Monitoring methodologies

---

*จบบทที่ 67 - Monitoring และ Metrics*
