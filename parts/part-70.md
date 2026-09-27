# บทที่ 70: Performance Testing กับ Lua

## บทนำ

**Performance Testing** คือกระบวนการวัดและประเมินประสิทธิภาพของระบบภายใต้ Load ต่างๆ เพื่อให้มั่นใจว่าระบบสามารถรองรับ Traffic ที่คาดหวังได้ ในบทนี้เราจะเรียนรู้การใช้ `wrk` และ `wrk2` ซึ่งเป็น Load Testing Tools ที่ใช้ Lua Script ในการปรับแต่งพฤติกรรมการทดสอบ

---

## 70.1 wrk และ wrk2 Overview

### 70.1.1 ความแตกต่างระหว่าง wrk และ wrk2

| Feature | wrk | wrk2 |
|---------|-----|------|
| Request rate | สูงสุดที่เป็นไปได้ | ควบคุม Rate ได้ (constant rate) |
| Latency reporting | ไม่ accurate สำหรับ high load | High Dynamic Range (HDR) Histogram |
| Percentile accuracy | ต่ำ | สูง (ใช้ HDR) |
| Use case | หา max throughput | วัด latency ที่ target rate |

```bash
# ติดตั้ง wrk
git clone https://github.com/wg/wrk.git
cd wrk && make

# ติดตั้ง wrk2
git clone https://github.com/giltene/wrk2.git
cd wrk2 && make

# รัน basic test
wrk -t4 -c100 -d30s http://localhost:8080/api/users
# -t = threads, -c = connections, -d = duration
```

### 70.1.2 Output ของ wrk

```
Running 30s test @ http://localhost:8080/api/users
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    45.23ms   12.34ms 234.56ms   89.23%
    Req/Sec    567.89    45.12   678.00     72.00%
  68000 requests in 30.10s, 45.23MB read
Requests/sec:   2259.80
Transfer/sec:      1.50MB
```

---

## 70.2 Custom wrk Scripts ใน Lua

### 70.2.1 โครงสร้าง wrk Script

wrk ให้เรา override hooks ต่างๆ ด้วย Lua:

```lua
-- basic.lua - Script พื้นฐานที่สุด

-- setup: เรียกครั้งเดียวต่อ thread ก่อนเริ่ม test
function setup(thread)
    -- thread object มี method: get/set(name, value), stop()
    thread:set("id", thread:get("id") or 0)
end

-- init: เรียกเมื่อ thread เริ่ม, args = command line args
function init(args)
    requests = 0
    responses = 0
end

-- request: เรียกทุกครั้งก่อนส่ง request, return request object
function request()
    requests = requests + 1
    return wrk.request()  -- ใช้ default request
end

-- response: เรียกเมื่อได้รับ response
function response(status, headers, body)
    responses = responses + 1
end

-- done: เรียกเมื่อ thread จบ (สำหรับ cleanup/reporting)
function done(summary, latency, requests_count)
    io.write(string.format(
        "Thread completed: %d requests, %d responses\n",
        requests, responses
    ))
end
```

### 70.2.2 Dynamic Request Generation

```lua
-- dynamic_request.lua - สร้าง Request แบบ Dynamic

local counter = 0
local user_ids = { 1, 2, 3, 4, 5, 42, 100, 999 }

-- สร้าง Request ที่หลากหลาย
function request()
    counter = counter + 1
    
    -- Rotate through different endpoints
    local endpoints = {
        "/api/users",
        "/api/products",
        "/api/orders",
        "/api/health",
    }
    local path = endpoints[(counter % #endpoints) + 1]
    
    -- บาง Request ใช้ Query Parameters
    if counter % 3 == 0 then
        local uid = user_ids[(counter % #user_ids) + 1]
        path = "/api/users/" .. uid
    end
    
    return wrk.format("GET", path, {
        ["Accept"]          = "application/json",
        ["X-Request-ID"]    = tostring(counter),
        ["Authorization"]   = "Bearer test-token-123",
    })
end

function response(status, headers, body)
    if status ~= 200 then
        -- Log non-200 responses
        io.write(string.format("Non-200: %d for request %d\n", status, counter))
    end
end
```

### 70.2.3 POST Request ด้วย JSON Body

```lua
-- post_test.lua - Test POST endpoints

local cjson = require("cjson")
local counter = 0

-- สร้าง JSON body ที่หลากหลาย
local function create_order_body()
    counter = counter + 1
    return cjson.encode({
        user_id  = math.random(1, 10000),
        items    = {
            { product_id = math.random(1, 100), quantity = math.random(1, 5) },
            { product_id = math.random(101, 200), quantity = math.random(1, 3) },
        },
        shipping = {
            address  = "123 Test Street",
            city     = "Bangkok",
            country  = "TH",
        },
        request_id = "test-" .. counter,
    })
end

function request()
    local body = create_order_body()
    local headers = {
        ["Content-Type"]   = "application/json",
        ["Content-Length"] = tostring(#body),
        ["Authorization"]  = "Bearer " .. os.getenv("API_TOKEN"),
    }
    return wrk.format("POST", "/api/orders", headers, body)
end

-- ติดตาม success/error rate
local success_count = 0
local error_count   = 0

function response(status, headers, body)
    if status >= 200 and status < 300 then
        success_count = success_count + 1
    else
        error_count = error_count + 1
        if error_count <= 10 then  -- log แค่ 10 errors แรก
            io.write(string.format("Error %d: %s\n", status, body:sub(1, 200)))
        end
    end
end

function done(summary, latency, req)
    print(string.format("\n=== Custom Summary ==="))
    print(string.format("Success: %d, Errors: %d", success_count, error_count))
    print(string.format("Error rate: %.2f%%",
        (error_count / (success_count + error_count)) * 100))
end
```

---

## 70.3 Benchmarking HTTP Endpoints

### 70.3.1 Authentication Flow Test

```lua
-- auth_benchmark.lua - Test authentication flow

local tokens = {}
local token_expiry = {}
local base_url = "http://localhost:8080"

-- Pre-populate tokens ก่อน test
function setup(thread)
    -- สร้าง tokens จำลอง (ใน production ต้อง authenticate จริง)
    for i = 1, 100 do
        tokens[i] = string.format("user-%d-token-%s",
            i, string.rep("x", 32))
        token_expiry[i] = os.time() + 3600
    end
    thread:set("token_idx", 1)
end

local request_count = 0

function request()
    request_count = request_count + 1
    
    -- Rotate tokens
    local idx = (request_count % #tokens) + 1
    local token = tokens[idx]
    
    -- Mix ของ API calls
    local api_calls = {
        { method = "GET",  path = "/api/profile" },
        { method = "GET",  path = "/api/dashboard" },
        { method = "POST", path = "/api/activity" },
    }
    local call = api_calls[(request_count % #api_calls) + 1]
    
    local headers = {
        ["Authorization"] = "Bearer " .. token,
        ["Accept"]        = "application/json",
        ["User-Agent"]    = "wrk-benchmark/1.0",
    }
    
    local body
    if call.method == "POST" then
        body = '{"event":"page_view","page":"/dashboard"}'
        headers["Content-Type"]   = "application/json"
        headers["Content-Length"] = tostring(#body)
    end
    
    return wrk.format(call.method, call.path, headers, body)
end
```

### 70.3.2 Read-Write Mix Benchmark

```lua
-- read_write_mix.lua - Test read/write ratio

-- Config: 80% reads, 20% writes
local READ_RATIO = 0.8

local reads  = 0
local writes = 0
local errors = 0

-- Pre-generated data pool
local write_data = {}
for i = 1, 1000 do
    write_data[i] = string.format(
        '{"name":"User %d","email":"user%d@test.com","age":%d}',
        i, i, 20 + (i % 50)
    )
end

local req_count = 0

function request()
    req_count = req_count + 1
    
    if math.random() < READ_RATIO then
        -- Read request
        reads = reads + 1
        local user_id = math.random(1, 10000)
        return wrk.format("GET", "/api/users/" .. user_id, {
            ["Accept"] = "application/json",
        })
    else
        -- Write request
        writes = writes + 1
        local body = write_data[(req_count % #write_data) + 1]
        return wrk.format("POST", "/api/users", {
            ["Content-Type"]   = "application/json",
            ["Content-Length"] = tostring(#body),
        }, body)
    end
end

function response(status, headers, body)
    if status >= 400 then
        errors = errors + 1
    end
end

function done(summary, latency, req)
    print(string.format("\n=== Read/Write Mix Results ==="))
    print(string.format("Reads: %d (%.1f%%)", reads, (reads/req_count)*100))
    print(string.format("Writes: %d (%.1f%%)", writes, (writes/req_count)*100))
    print(string.format("Errors: %d (%.2f%%)", errors, (errors/req_count)*100))
    print(string.format("Total RPS: %.2f", req_count / summary["duration"] * 1e6))
end
```

---

## 70.4 Connection Pooling Verification

### 70.4.1 ทดสอบ Connection Pool

```lua
-- connection_pool_test.lua - ตรวจสอบ connection pooling

-- ติดตาม connection header ใน response
local keep_alive_count   = 0
local close_count        = 0
local total_responses    = 0
local response_times     = {}

function request()
    return wrk.format("GET", "/api/health", {
        ["Connection"] = "keep-alive",
    })
end

function response(status, headers, body)
    total_responses = total_responses + 1
    
    -- ตรวจสอบ Connection header
    local conn_header = headers["Connection"] or headers["connection"] or ""
    if conn_header:lower():find("keep-alive") then
        keep_alive_count = keep_alive_count + 1
    elseif conn_header:lower():find("close") then
        close_count = close_count + 1
    end
end

function done(summary, latency, req)
    print("\n=== Connection Pooling Analysis ===")
    print(string.format("Total responses: %d", total_responses))
    print(string.format("Keep-Alive connections: %d (%.1f%%)",
        keep_alive_count, (keep_alive_count / total_responses) * 100))
    print(string.format("Connection Close: %d (%.1f%%)",
        close_count, (close_count / total_responses) * 100))
    print(string.format("\nRecommendation: %s",
        keep_alive_count / total_responses > 0.95
            and "Connection pooling is working well (>95%)"
            or "WARNING: Connection pooling may not be configured correctly"))
end
```

### 70.4.2 ทดสอบ Database Connection Pool

```lua
-- db_pool_test.lua - Test database query performance

-- Queries ที่ใช้ test
local queries = {
    "simple_select",
    "join_query",
    "aggregate_query",
    "write_query",
}

local query_stats = {}
for _, q in ipairs(queries) do
    query_stats[q] = { count = 0, errors = 0, total_time = 0 }
end

local request_times = {}
local req_count = 0

function request()
    req_count = req_count + 1
    local query_type = queries[(req_count % #queries) + 1]
    
    -- Encode query type ใน path หรือ header
    return wrk.format("GET", "/api/benchmark/" .. query_type, {
        ["Accept"] = "application/json",
        ["X-Query-Type"] = query_type,
    })
end

function response(status, headers, body)
    local query_type = "unknown"
    -- ดู response header เพื่อรู้ว่าเป็น query อะไร
    local qt = headers["X-Query-Type"] or "simple_select"
    
    if query_stats[qt] then
        query_stats[qt].count = query_stats[qt].count + 1
        if status >= 400 then
            query_stats[qt].errors = query_stats[qt].errors + 1
        end
    end
    
    -- บันทึก server-timing ถ้ามี
    local timing = headers["Server-Timing"] or headers["X-DB-Time"]
    if timing then
        local ms = tonumber(timing:match("dur=([%d%.]+)"))
        if ms and query_stats[qt] then
            query_stats[qt].total_time = query_stats[qt].total_time + ms
        end
    end
end

function done(summary, latency, req)
    print("\n=== Database Query Performance ===")
    for qt, stats in pairs(query_stats) do
        if stats.count > 0 then
            print(string.format("%-20s count: %6d  avg_time: %.2fms  errors: %d",
                qt, stats.count,
                stats.total_time / stats.count,
                stats.errors))
        end
    end
end
```

---

## 70.5 Percentile Latency Analysis

### 70.5.1 เข้าใจ Percentiles

Percentile latency บอกว่า X% ของ requests ใช้เวลาน้อยกว่าหรือเท่ากับค่านี้

```lua
-- percentile_analysis.lua - วิเคราะห์ latency distribution

-- HDR Histogram สำหรับ accurate percentiles
-- (wrk2 ใช้ HDR by default, wrk ต้องเก็บเอง)

local latencies = {}
local max_samples = 100000  -- เก็บสูงสุด 100k samples

-- ใช้ approximate histogram เพื่อประหยัด memory
local histogram_buckets = {}
local function init_histogram()
    -- Buckets: 0-1ms, 1-2ms, 2-5ms, 5-10ms, 10-25ms, 25-50ms, 50-100ms, 100ms+
    local boundaries = { 1, 2, 5, 10, 25, 50, 100, 250, 500, 1000 }
    for _, b in ipairs(boundaries) do
        histogram_buckets[b] = 0
    end
    histogram_buckets["1000+"] = 0
end

function init(args)
    init_histogram()
end

local start_times = {}
local req_idx = 0

function request()
    req_idx = req_idx + 1
    start_times[req_idx % 1000] = os.clock() * 1000  -- ms
    return wrk.request()
end

local resp_idx = 0
function response(status, headers, body)
    resp_idx = resp_idx + 1
    
    -- approximate latency
    local boundaries_sorted = { 1, 2, 5, 10, 25, 50, 100, 250, 500, 1000 }
    local latency_ms = math.random(1, 100)  -- จำลอง latency
    
    for _, b in ipairs(boundaries_sorted) do
        if latency_ms <= b then
            histogram_buckets[b] = histogram_buckets[b] + 1
            return
        end
    end
    histogram_buckets["1000+"] = histogram_buckets["1000+"] + 1
end

function done(summary, latency, req)
    print("\n=== Latency Distribution ===")
    print(string.format("%-12s %10s %12s %10s",
        "Latency", "Count", "Cumulative", "Percentile"))
    print(string.rep("-", 48))
    
    local total = 0
    for _, v in pairs(histogram_buckets) do total = total + v end
    
    local cumulative = 0
    local boundaries_sorted = { 1, 2, 5, 10, 25, 50, 100, 250, 500, 1000, "1000+" }
    
    for _, b in ipairs(boundaries_sorted) do
        local count = histogram_buckets[b] or 0
        cumulative = cumulative + count
        local pct = total > 0 and (cumulative / total * 100) or 0
        print(string.format("<= %-8s %10d %12d %9.1f%%",
            tostring(b) .. "ms", count, cumulative, pct))
    end
    
    -- wrk built-in latency stats
    print("\n=== wrk Latency Stats (from HDR) ===")
    print(string.format("Mean:   %.2fms", latency.mean / 1000))
    print(string.format("Stdev:  %.2fms", latency.stdev / 1000))
    print(string.format("Max:    %.2fms", latency.max / 1000))
    print(string.format("P50:    %.2fms", latency:percentile(50) / 1000))
    print(string.format("P75:    %.2fms", latency:percentile(75) / 1000))
    print(string.format("P90:    %.2fms", latency:percentile(90) / 1000))
    print(string.format("P95:    %.2fms", latency:percentile(95) / 1000))
    print(string.format("P99:    %.2fms", latency:percentile(99) / 1000))
    print(string.format("P99.9:  %.2fms", latency:percentile(99.9) / 1000))
    print(string.format("P99.99: %.2fms", latency:percentile(99.99) / 1000))
end
```

### 70.5.2 wrk2 สำหรับ Constant Rate Testing

```lua
-- constant_rate.lua - test at specific request rate with wrk2

-- ใช้กับ: wrk2 -t4 -c100 -d60s -R1000 -s constant_rate.lua <url>
-- -R1000 = 1000 requests per second

local errors_by_status = {}
local p99_threshold_ms = 100  -- SLA: p99 < 100ms

function response(status, headers, body)
    if status ~= 200 then
        errors_by_status[status] = (errors_by_status[status] or 0) + 1
    end
end

function done(summary, latency, req)
    print("\n=== wrk2 Constant Rate Results ===")
    print(string.format("Target rate:  %d RPS", 1000))
    print(string.format("Actual rate:  %.0f RPS",
        summary["requests"] / (summary["duration"] / 1e6)))
    
    local p99 = latency:percentile(99) / 1000  -- to ms
    print(string.format("\nP99 Latency: %.2fms (SLA: %dms) %s",
        p99, p99_threshold_ms,
        p99 <= p99_threshold_ms and "PASS" or "FAIL"))
    
    print(string.format("P95 Latency: %.2fms", latency:percentile(95) / 1000))
    print(string.format("P50 Latency: %.2fms", latency:percentile(50) / 1000))
    
    if next(errors_by_status) then
        print("\n=== Errors by Status Code ===")
        for status, count in pairs(errors_by_status) do
            print(string.format("  HTTP %d: %d requests (%.2f%%)",
                status, count, count / summary["requests"] * 100))
        end
    end
    
    -- SLA check
    local total_errors = summary["errors"]["status"] + 
                         summary["errors"]["connect"] +
                         summary["errors"]["read"]
    local error_rate = total_errors / summary["requests"] * 100
    
    print(string.format("\n=== SLA Report ==="))
    print(string.format("Error rate: %.4f%% (threshold: 0.1%%) %s",
        error_rate,
        error_rate <= 0.1 and "PASS" or "FAIL"))
end
```

---

## 70.6 Throughput vs Latency Tradeoffs

### 70.6.1 Sweep Test

```lua
-- sweep_test.lua - test ที่ concurrency levels ต่างๆ

-- รัน test นี้หลายครั้งด้วย concurrency ต่างๆ และเปรียบเทียบ
-- wrk -t$(nproc) -c10 -d30s -s sweep_test.lua <url>
-- wrk -t$(nproc) -c50 -d30s -s sweep_test.lua <url>
-- wrk -t$(nproc) -c100 -d30s -s sweep_test.lua <url>
-- wrk -t$(nproc) -c200 -d30s -s sweep_test.lua <url>

local test_start = os.time()
local concurrency_hint = tonumber(os.getenv("WRK_CONCURRENCY") or "0")

function init(args)
    if #args > 0 then
        concurrency_hint = tonumber(args[1]) or 0
    end
end

function request()
    return wrk.format("GET", "/api/compute-intensive", {
        ["Accept"] = "application/json",
    })
end

local total_reqs = 0
local total_errors = 0

function response(status, headers, body)
    total_reqs = total_reqs + 1
    if status >= 400 then
        total_errors = total_errors + 1
    end
end

function done(summary, latency, req)
    local duration_sec = summary["duration"] / 1e6
    local rps = summary["requests"] / duration_sec
    local p50 = latency:percentile(50) / 1000
    local p99 = latency:percentile(99) / 1000
    
    -- Output CSV สำหรับ plotting
    print(string.format("concurrency,rps,p50_ms,p99_ms,error_pct"))
    print(string.format("%d,%.0f,%.2f,%.2f,%.4f",
        concurrency_hint, rps, p50, p99,
        (total_errors / total_reqs) * 100))
end
```

### 70.6.2 Bash Script สำหรับ Sweep

```bash
#!/bin/bash
# sweep.sh - run sweep test ที่ concurrency levels ต่างๆ

OUTPUT="sweep_results.csv"
echo "concurrency,rps,p50_ms,p99_ms,error_pct" > "$OUTPUT"

for C in 1 5 10 25 50 100 200 500; do
    echo "Testing concurrency=$C..."
    
    RESULT=$(wrk -t4 -c$C -d30s \
        -s sweep_test.lua \
        "http://localhost:8080" \
        -- $C 2>/dev/null | tail -1)
    
    echo "$RESULT" >> "$OUTPUT"
    sleep 5  # cool-down
done

echo "Results saved to $OUTPUT"
# Plot with gnuplot or Python pandas
```

---

## 70.7 Finding Bottlenecks with Profiling

### 70.7.1 Lua Profiling ใน OpenResty

```lua
-- profiler.lua - Simple profiling สำหรับ OpenResty

local Profiler = {}
Profiler.__index = Profiler

function Profiler.new()
    return setmetatable({
        _timers = {},
        _counts = {},
        _totals = {},
    }, Profiler)
end

-- Timer สำหรับวัด function execution time
function Profiler:time(name, fn)
    local start = ngx.now() * 1000
    local result = { fn() }
    local elapsed = ngx.now() * 1000 - start
    
    self._counts[name] = (self._counts[name] or 0) + 1
    self._totals[name] = (self._totals[name] or 0) + elapsed
    
    -- Track max time
    self._timers[name] = math.max(self._timers[name] or 0, elapsed)
    
    return table.unpack(result)
end

-- Report
function Profiler:report()
    local results = {}
    for name, count in pairs(self._counts) do
        table.insert(results, {
            name    = name,
            count   = count,
            total   = self._totals[name],
            avg     = self._totals[name] / count,
            max     = self._timers[name],
        })
    end
    
    -- Sort by total time (descending)
    table.sort(results, function(a, b) return a.total > b.total end)
    
    local lines = { "=== Profiler Report ===" }
    table.insert(lines, string.format("%-30s %8s %8s %8s %8s",
        "Function", "Count", "Total(ms)", "Avg(ms)", "Max(ms)"))
    table.insert(lines, string.rep("-", 70))
    
    for _, r in ipairs(results) do
        table.insert(lines, string.format("%-30s %8d %9.2f %8.3f %8.3f",
            r.name, r.count, r.total, r.avg, r.max))
    end
    
    return table.concat(lines, "\n")
end

-- Global profiler instance
local profiler = Profiler.new()

-- ตัวอย่างการใช้งาน
local function handle_request()
    -- Profile ส่วนต่างๆ ของ request handling
    local user = profiler:time("auth.verify_token", function()
        return verify_jwt_token(ngx.req.get_headers()["Authorization"])
    end)
    
    local data = profiler:time("db.fetch_user", function()
        return db_query("SELECT * FROM users WHERE id = ?", user.id)
    end)
    
    local html = profiler:time("template.render", function()
        return render_template("user_profile", data)
    end)
    
    ngx.say(html)
end
```

### 70.7.2 Request Timing Breakdown

```lua
-- timing_breakdown.lua - วัด timing ของแต่ละ phase

-- wrk script ที่วิเคราะห์ Server-Timing header
-- Server ต้องส่ง: Server-Timing: db;dur=12.3,render;dur=5.6,total;dur=20.1

local phase_totals = {}
local phase_counts = {}
local request_count = 0

function response(status, headers, body)
    request_count = request_count + 1
    
    local timing = headers["Server-Timing"] or headers["server-timing"]
    if not timing then return end
    
    -- Parse Server-Timing header
    -- Format: name;dur=value, name2;dur=value2
    for entry in timing:gmatch("[^,]+") do
        local name = entry:match("^%s*([^;]+)")
        local dur  = entry:match("dur=([%d%.]+)")
        
        if name and dur then
            name = name:match("^%s*(.-)%s*$")
            local ms = tonumber(dur) or 0
            
            phase_totals[name] = (phase_totals[name] or 0) + ms
            phase_counts[name] = (phase_counts[name] or 0) + 1
        end
    end
end

function done(summary, latency, req)
    print("\n=== Server-Side Timing Breakdown ===")
    print(string.format("%-20s %10s %10s %10s",
        "Phase", "Requests", "Avg (ms)", "Total (ms)"))
    print(string.rep("-", 55))
    
    -- Sort by average time
    local phases = {}
    for name, total in pairs(phase_totals) do
        table.insert(phases, {
            name  = name,
            total = total,
            count = phase_counts[name] or 0,
            avg   = total / (phase_counts[name] or 1),
        })
    end
    table.sort(phases, function(a, b) return a.avg > b.avg end)
    
    for _, p in ipairs(phases) do
        print(string.format("%-20s %10d %10.2f %10.2f",
            p.name, p.count, p.avg, p.total))
    end
    
    print(string.format("\nTotal requests analyzed: %d", request_count))
end
```

---

## 70.8 Memory Leak Detection Under Load

### 70.8.1 Memory Monitoring Script

```lua
-- memory_leak.lua - ตรวจหา memory leak ระหว่าง load test

-- Endpoint ที่ expose memory stats (ต้องสร้างเอง)
-- GET /debug/memory -> {"rss_mb": 45.2, "heap_mb": 38.1, "lua_kb": 2048}

local memory_samples = {}
local sample_interval = 10  -- sample ทุก 10 requests
local request_count = 0

function request()
    request_count = request_count + 1
    
    -- ทุกๆ N requests, check memory
    if request_count % sample_interval == 0 then
        return wrk.format("GET", "/debug/memory")
    end
    
    return wrk.format("GET", "/api/workload")
end

function response(status, headers, body)
    if status == 200 and body:find('"rss_mb"') then
        -- Parse memory info
        local rss  = tonumber(body:match('"rss_mb":%s*([%d%.]+)'))
        local heap = tonumber(body:match('"heap_mb":%s*([%d%.]+)'))
        
        if rss then
            table.insert(memory_samples, {
                req_count = request_count,
                rss_mb    = rss,
                heap_mb   = heap or 0,
                timestamp = os.time(),
            })
        end
    end
end

function done(summary, latency, req)
    if #memory_samples < 2 then
        print("Not enough memory samples collected")
        return
    end
    
    print("\n=== Memory Usage Over Time ===")
    print(string.format("%-10s %10s %10s", "Requests", "RSS (MB)", "Heap (MB)"))
    print(string.rep("-", 35))
    
    local first = memory_samples[1]
    local last  = memory_samples[#memory_samples]
    
    -- Print สรุปทุก 10 samples
    for i, s in ipairs(memory_samples) do
        if i == 1 or i == #memory_samples or i % 10 == 0 then
            print(string.format("%-10d %10.1f %10.1f",
                s.req_count, s.rss_mb, s.heap_mb))
        end
    end
    
    -- Growth analysis
    local rss_growth = last.rss_mb - first.rss_mb
    local growth_pct = (rss_growth / first.rss_mb) * 100
    
    print(string.format("\n=== Memory Growth Analysis ==="))
    print(string.format("Initial RSS:  %.1f MB", first.rss_mb))
    print(string.format("Final RSS:    %.1f MB", last.rss_mb))
    print(string.format("Growth:       %.1f MB (%.1f%%)", rss_growth, growth_pct))
    
    -- ตรวจหา memory leak
    if growth_pct > 20 then
        print("\nWARNING: Significant memory growth detected!")
        print("Possible memory leak. Check:")
        print("  - Global variable accumulation")
        print("  - Event listener not removed")
        print("  - Cache without eviction policy")
        print("  - Connection leaks in pools")
    else
        print("\nMemory usage looks stable (growth < 20%)")
    end
end
```

---

## 70.9 Comparative Benchmarks

### 70.9.1 A/B Version Comparison

```lua
-- ab_compare.lua - เปรียบเทียบ 2 versions ของ API

-- ใช้: wrk -t4 -c100 -d60s -s ab_compare.lua <url>
-- ตั้งค่า environment variables:
-- VERSION_A_URL=http://api-v1:8080
-- VERSION_B_URL=http://api-v2:8080

local url_a = os.getenv("VERSION_A_URL") or "http://localhost:8080"
local url_b = os.getenv("VERSION_B_URL") or "http://localhost:8081"

local stats = {
    a = { requests = 0, errors = 0, total_ms = 0, max_ms = 0 },
    b = { requests = 0, errors = 0, total_ms = 0, max_ms = 0 },
}

local current_version = "a"
local req_count = 0

function setup(thread)
    -- Alternate versions per thread
    local tid = thread:get("id") or 0
    thread:set("version", tid % 2 == 0 and "a" or "b")
end

function init(args)
    -- Use thread's assigned version
end

function request()
    req_count = req_count + 1
    current_version = (req_count % 2 == 0) and "a" or "b"
    
    local base = current_version == "a" and url_a or url_b
    -- wrk ใช้ host จาก command line, path จาก request()
    return wrk.format("GET", "/api/compute", {
        ["Accept"] = "application/json",
        ["X-Version"] = current_version,
    })
end

local req_start = {}

function response(status, headers, body)
    local version = headers["X-API-Version"] or current_version
    local s = stats[version] or stats.a
    
    s.requests = s.requests + 1
    if status >= 400 then
        s.errors = s.errors + 1
    end
    
    -- Get timing from header
    local dur = tonumber(headers["X-Response-Time"]) or 0
    s.total_ms = s.total_ms + dur
    s.max_ms = math.max(s.max_ms, dur)
end

function done(summary, latency, req)
    print("\n=== A/B Version Comparison ===")
    print(string.format("%-12s %10s %10s %10s %10s",
        "Version", "Requests", "Errors", "Avg (ms)", "Max (ms)"))
    print(string.rep("-", 57))
    
    for _, ver in ipairs({"a", "b"}) do
        local s = stats[ver]
        local avg = s.requests > 0 and s.total_ms / s.requests or 0
        print(string.format("%-12s %10d %10d %10.2f %10.2f",
            "Version " .. ver:upper(),
            s.requests, s.errors, avg, s.max_ms))
    end
    
    -- Winner
    local avg_a = stats.a.requests > 0 and stats.a.total_ms / stats.a.requests or 0
    local avg_b = stats.b.requests > 0 and stats.b.total_ms / stats.b.requests or 0
    
    if avg_a < avg_b then
        print(string.format("\nVersion A is faster by %.1f%% (%.2fms vs %.2fms)",
            (avg_b - avg_a) / avg_b * 100, avg_a, avg_b))
    else
        print(string.format("\nVersion B is faster by %.1f%% (%.2fms vs %.2fms)",
            (avg_a - avg_b) / avg_a * 100, avg_b, avg_a))
    end
end
```

---

## 70.10 Complete Performance Test Suite

นี่คือ Test Suite ที่ครบครันพร้อมสำหรับการทดสอบ Production:

```lua
-- perf_suite.lua - Complete performance test suite

--[[
  ใช้งาน:
  wrk2 -t$(nproc) -c200 -d120s -R2000 --latency \
       -s perf_suite.lua \
       http://api.example.com \
       -- --scenario=full_flow
--]]

local cjson = require("cjson")

-- Configuration
local config = {
    scenario    = os.getenv("SCENARIO") or "read_only",
    sla = {
        p99_ms    = 200,
        p95_ms    = 100,
        p50_ms    = 50,
        error_pct = 0.5,
        min_rps   = 1000,
    }
}

-- Parse args
function init(args)
    for _, arg in ipairs(args) do
        local k, v = arg:match("^%-%-(%w+)=(.+)$")
        if k then config[k] = v end
    end
    
    print(string.format("=== Performance Test Suite ==="))
    print(string.format("Scenario: %s", config.scenario))
    print(string.format("SLA: p99<%dms, p95<%dms, errors<%.1f%%",
        config.sla.p99_ms, config.sla.p95_ms, config.sla.error_pct))
    print("==============================")
end

-- Scenario definitions
local scenarios = {}

-- Scenario 1: Read-only
scenarios["read_only"] = function()
    local paths = {
        "/api/products?page=1",
        "/api/products?page=2",
        "/api/categories",
        "/api/featured",
    }
    local idx = math.random(#paths)
    return wrk.format("GET", paths[idx], {
        ["Accept"] = "application/json",
        ["Cache-Control"] = "no-cache",
    })
end

-- Scenario 2: Full flow (browse -> add to cart -> checkout)
local flow_state = {}
scenarios["full_flow"] = function()
    local thread_id = 0  -- simplification
    local state = flow_state[thread_id] or "browse"
    
    if state == "browse" then
        flow_state[thread_id] = "add_cart"
        return wrk.format("GET", "/api/products/" .. math.random(1, 100), {
            ["Accept"] = "application/json",
        })
    elseif state == "add_cart" then
        flow_state[thread_id] = "checkout"
        local body = cjson.encode({
            product_id = math.random(1, 100),
            quantity   = math.random(1, 3),
        })
        return wrk.format("POST", "/api/cart/items", {
            ["Content-Type"] = "application/json",
            ["Content-Length"] = tostring(#body),
            ["Authorization"] = "Bearer test-user-token",
        }, body)
    else
        flow_state[thread_id] = "browse"
        return wrk.format("GET", "/api/cart", {
            ["Authorization"] = "Bearer test-user-token",
        })
    end
end

-- Scenario 3: Search-heavy
scenarios["search"] = function()
    local terms = { "laptop", "phone", "book", "headphone", "camera" }
    local term = terms[math.random(#terms)]
    local path = string.format("/api/search?q=%s&limit=20&offset=%d",
        term, math.random(0, 10) * 20)
    return wrk.format("GET", path, { ["Accept"] = "application/json" })
end

function request()
    local fn = scenarios[config.scenario] or scenarios["read_only"]
    return fn()
end

-- Stats tracking
local stats = {
    total     = 0,
    success   = 0,
    client_err = 0,
    server_err = 0,
    timeouts  = 0,
}

function response(status, headers, body)
    stats.total = stats.total + 1
    
    if status >= 200 and status < 300 then
        stats.success = stats.success + 1
    elseif status >= 400 and status < 500 then
        stats.client_err = stats.client_err + 1
    elseif status >= 500 then
        stats.server_err = stats.server_err + 1
    end
end

function done(summary, latency, req)
    local duration_sec = summary["duration"] / 1e6
    local actual_rps = summary["requests"] / duration_sec
    local error_pct = (stats.server_err / stats.total) * 100
    
    print("\n====== TEST RESULTS ======")
    print(string.format("Duration:       %.1fs", duration_sec))
    print(string.format("Total requests: %d", stats.total))
    print(string.format("Actual RPS:     %.0f", actual_rps))
    print(string.format("Success:        %d (%.1f%%)",
        stats.success, stats.success / stats.total * 100))
    print(string.format("Client errors:  %d", stats.client_err))
    print(string.format("Server errors:  %d", stats.server_err))
    
    print("\n====== LATENCY ======")
    local p50  = latency:percentile(50) / 1000
    local p75  = latency:percentile(75) / 1000
    local p90  = latency:percentile(90) / 1000
    local p95  = latency:percentile(95) / 1000
    local p99  = latency:percentile(99) / 1000
    local p999 = latency:percentile(99.9) / 1000
    
    print(string.format("P50:   %.2fms", p50))
    print(string.format("P75:   %.2fms", p75))
    print(string.format("P90:   %.2fms", p90))
    print(string.format("P95:   %.2fms", p95))
    print(string.format("P99:   %.2fms", p99))
    print(string.format("P99.9: %.2fms", p999))
    print(string.format("Max:   %.2fms", latency.max / 1000))
    
    -- SLA Evaluation
    print("\n====== SLA CHECK ======")
    local sla = config.sla
    local all_pass = true
    
    local function check(name, actual, threshold, invert)
        local pass = invert and (actual >= threshold) or (actual <= threshold)
        all_pass = all_pass and pass
        print(string.format("%-20s %.2f %s %s %s",
            name, actual,
            invert and ">=" or "<=",
            tostring(threshold),
            pass and "PASS" or "FAIL"))
    end
    
    check("P99 latency (ms)",   p99,        sla.p99_ms)
    check("P95 latency (ms)",   p95,        sla.p95_ms)
    check("P50 latency (ms)",   p50,        sla.p50_ms)
    check("Error rate (%)",     error_pct,  sla.error_pct)
    check("Throughput (RPS)",   actual_rps, sla.min_rps, true)
    
    print("\n====== VERDICT ======")
    print(all_pass and "ALL TESTS PASSED" or "SOME TESTS FAILED")
    
    -- Exit code (wrk2 support)
    if not all_pass then
        os.exit(1)
    end
end
```

---

## 70.11 Bash Automation สำหรับ Test Suite

```bash
#!/bin/bash
# run_perf_tests.sh - Automation script

set -e

API_URL="${API_URL:-http://localhost:8080}"
REPORT_DIR="perf_reports/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$REPORT_DIR"

echo "=== Performance Test Automation ==="
echo "Target: $API_URL"
echo "Report dir: $REPORT_DIR"

# ฟังก์ชัน run test scenario
run_scenario() {
    local name="$1"
    local scenario="$2"
    local threads="${3:-4}"
    local conns="${4:-100}"
    local rate="${5:-500}"
    local duration="${6:-60s}"
    
    echo "Running: $name (t=$threads c=$conns R=$rate d=$duration)"
    
    wrk2 -t$threads -c$conns -d$duration -R$rate --latency \
         -s perf_suite.lua \
         "$API_URL" \
         -- "--scenario=$scenario" \
    > "$REPORT_DIR/${name}.txt" 2>&1
    
    echo "Done: $name"
}

# Run scenarios
run_scenario "read_only_low"    "read_only" 4 50  200 60s
run_scenario "read_only_mid"    "read_only" 4 100 500 60s
run_scenario "read_only_high"   "read_only" 8 200 1000 60s
run_scenario "full_flow_low"    "full_flow" 4 50  100 60s
run_scenario "search_stress"    "search"    4 100 300 120s

# Generate summary
echo "=== Summary ==="
for f in "$REPORT_DIR"/*.txt; do
    name=$(basename "$f" .txt)
    rps=$(grep "Actual RPS:" "$f" | awk '{print $NF}')
    p99=$(grep "P99:" "$f" | head -1 | awk '{print $NF}')
    verdict=$(grep -E "PASSED|FAILED" "$f" | tail -1)
    echo "$name: RPS=$rps P99=$p99 $verdict"
done
```

---

## 70.12 แบบฝึกหัด

### แบบฝึกหัดที่ 1: wrk Script พื้นฐาน
เขียน wrk script ที่:
- ส่ง GET request ไปยัง `/api/products`
- ทุกๆ 5th request ส่ง POST ไปยัง `/api/cart`
- นับ success และ error responses
- แสดงสรุปเมื่อ test จบ

### แบบฝึกหัดที่ 2: Percentile Report
สร้าง wrk script ที่:
- เก็บ response time ใน histogram
- คำนวณ p50, p75, p90, p95, p99 เอง
- เปรียบเทียบกับค่า SLA ที่กำหนด
- Output เป็น JSON format สำหรับ CI integration

### แบบฝึกหัดที่ 3: Memory Leak Test
ออกแบบ test plan สำหรับ detect memory leak:
- รัน load test 5 นาที
- Sample memory ทุก 30 วินาที
- Plot กราฟ memory usage vs time
- ตั้ง threshold สำหรับ alert

### แบบฝึกหัดที่ 4: A/B Comparison
สร้าง framework สำหรับ A/B performance testing:
- Route % ของ traffic ไป version A/B
- เก็บสถิติแยกต่างหาก
- Statistical significance test
- สร้าง HTML report พร้อมกราฟ

### แบบฝึกหัดที่ 5: CI Integration
เขียน Pipeline สำหรับ performance regression testing:
- รัน baseline test ก่อน deploy
- รัน test หลัง deploy
- Compare results และ fail ถ้า performance ลดลง > 10%
- Send notification ผ่าน Slack/Email

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **wrk/wrk2 basics** - วิธีใช้ tools และความแตกต่าง
2. **Lua scripts** - การเขียน custom scenarios ที่ซับซ้อน
3. **Percentile analysis** - ทำไม p99 ถึงสำคัญกว่า average
4. **Connection pooling** - ตรวจสอบว่า pooling ทำงานถูกต้อง
5. **Memory leak detection** - วิธีหา memory leak ใน production
6. **A/B benchmarking** - เปรียบเทียบ versions อย่างมีระบบ
7. **SLA validation** - automated checking ว่าระบบผ่าน SLA

Performance testing ที่ดีต้องทำสม่ำเสมอ ไม่ใช่แค่ก่อน launch การ integrate เข้า CI/CD pipeline ช่วยให้ detect regression ได้เร็วก่อนถึง production
