# บทที่ 87: Production Systems Engineering

## บทนำ: SRE (Site Reliability Engineering)

SRE คือวิธีการวิศวกรรมซอฟต์แวร์ที่ Google พัฒนาขึ้นเพื่อจัดการกับระบบ production ขนาดใหญ่ ในบทนี้เราจะสำรวจแนวคิด SRE และวิธีการนำไปใช้ด้วย Lua

### SLA, SLO, SLI คืออะไร

- **SLA (Service Level Agreement)**: สัญญาที่ทำกับลูกค้าว่าจะให้บริการในระดับไหน
- **SLO (Service Level Objective)**: เป้าหมายภายในที่ต้องการบรรลุ (เช่น 99.9% uptime)
- **SLI (Service Level Indicator)**: ตัวชี้วัดจริงที่ใช้วัด (เช่น request success rate)

```lua
-- ตัวอย่างที่ 1: SLI Tracker
local SLITracker = {}
SLITracker.__index = SLITracker

function SLITracker.new(name, window_seconds)
    return setmetatable({
        name = name,
        window = window_seconds or 3600,  -- 1 hour default
        events = {},
        good_events = 0,
        total_events = 0
    }, SLITracker)
end

function SLITracker:record(is_good, timestamp)
    timestamp = timestamp or os.time()
    
    -- Cleanup old events outside window
    local cutoff = timestamp - self.window
    while #self.events > 0 and self.events[1].timestamp < cutoff do
        local old_event = table.remove(self.events, 1)
        self.total_events = self.total_events - 1
        if old_event.good then
            self.good_events = self.good_events - 1
        end
    end
    
    -- Record new event
    table.insert(self.events, {timestamp = timestamp, good = is_good})
    self.total_events = self.total_events + 1
    if is_good then
        self.good_events = self.good_events + 1
    end
end

function SLITracker:current_rate()
    if self.total_events == 0 then return 1.0 end
    return self.good_events / self.total_events
end

function SLITracker:report()
    local rate = self:current_rate()
    print(string.format("[SLI] %s: %.4f%% (%d/%d good events in last %ds)",
                       self.name,
                       rate * 100,
                       self.good_events,
                       self.total_events,
                       self.window))
    return rate
end
```

## Error Budget

```lua
-- ตัวอย่างที่ 2: Error Budget Calculator
local ErrorBudget = {}
ErrorBudget.__index = ErrorBudget

function ErrorBudget.new(slo_percentage, window_days)
    local self = setmetatable({
        slo = slo_percentage / 100,
        window_seconds = window_days * 86400,
        sli = SLITracker.new("availability", window_days * 86400),
        burn_rate_alerts = {}
    }, ErrorBudget)
    
    -- Calculate total allowed downtime
    self.total_budget_seconds = (1 - self.slo) * self.window_seconds
    
    return self
end

function ErrorBudget:remaining_seconds()
    local current_error_rate = 1 - self.sli:current_rate()
    local elapsed_seconds = math.min(
        self.window_seconds,
        self.sli.total_events > 0 and 
        (os.time() - (self.sli.events[1] and self.sli.events[1].timestamp or os.time())) 
        or 0
    )
    
    local used_seconds = current_error_rate * elapsed_seconds
    return self.total_budget_seconds - used_seconds
end

function ErrorBudget:burn_rate()
    local current_error_rate = 1 - self.sli:current_rate()
    local ideal_error_rate = 1 - self.slo
    
    if ideal_error_rate == 0 then return 0 end
    return current_error_rate / ideal_error_rate
end

function ErrorBudget:add_burn_rate_alert(name, threshold, window_minutes, action)
    table.insert(self.burn_rate_alerts, {
        name = name,
        threshold = threshold,
        window = window_minutes * 60,
        action = action
    })
end

function ErrorBudget:check_alerts()
    local burn_rate = self:burn_rate()
    
    for _, alert in ipairs(self.burn_rate_alerts) do
        if burn_rate >= alert.threshold then
            print(string.format("[ALERT] %s: burn rate %.2fx (threshold: %.2fx)",
                               alert.name, burn_rate, alert.threshold))
            if alert.action then
                alert.action(burn_rate)
            end
        end
    end
end

function ErrorBudget:report()
    local remaining = self:remaining_seconds()
    local burn = self:burn_rate()
    local percentage_remaining = (remaining / self.total_budget_seconds) * 100
    
    print(string.format([[
Error Budget Report:
  SLO: %.2f%%
  Current SLI: %.4f%%
  Budget remaining: %.0f seconds (%.1f%%)
  Burn rate: %.2fx
  Status: %s]],
        self.slo * 100,
        self.sli:current_rate() * 100,
        math.max(0, remaining),
        math.max(0, percentage_remaining),
        burn,
        remaining > 0 and "OK" or "BUDGET EXHAUSTED"
    ))
end

-- ตัวอย่างการใช้งาน
local budget = ErrorBudget.new(99.9, 30)  -- 99.9% SLO, 30-day window

budget:add_burn_rate_alert("page", 14.4, 60, function(rate)
    print("PagerDuty: Critical burn rate!")
end)

budget:add_burn_rate_alert("ticket", 6.0, 360, function(rate)
    print("Jira: High burn rate ticket created")
end)

-- Simulate some requests
local timestamp = os.time()
for i = 1, 1000 do
    local success = math.random() > 0.001  -- 99.9% success rate
    budget.sli:record(success, timestamp + i)
end

budget:report()
budget:check_alerts()
```

## Toil Automation

```lua
-- ตัวอย่างที่ 3: Toil Detector and Automator
local ToilTracker = {}
ToilTracker.__index = ToilTracker

function ToilTracker.new()
    return setmetatable({
        tasks = {},
        automations = {}
    }, ToilTracker)
end

function ToilTracker:record_task(name, duration_minutes, is_automated, category)
    if not self.tasks[name] then
        self.tasks[name] = {
            total_time = 0,
            count = 0,
            is_automated = false,
            category = category or "manual"
        }
    end
    
    local task = self.tasks[name]
    task.total_time = task.total_time + duration_minutes
    task.count = task.count + 1
    task.is_automated = is_automated
end

function ToilTracker:calculate_toil_percentage()
    local total_time = 0
    local toil_time = 0
    
    for name, task in pairs(self.tasks) do
        total_time = total_time + task.total_time
        if not task.is_automated then
            toil_time = toil_time + task.total_time
        end
    end
    
    return total_time > 0 and (toil_time / total_time * 100) or 0
end

function ToilTracker:top_toil_tasks(n)
    local tasks_list = {}
    for name, task in pairs(self.tasks) do
        if not task.is_automated then
            table.insert(tasks_list, {name = name, task = task})
        end
    end
    
    table.sort(tasks_list, function(a, b)
        return a.task.total_time > b.task.total_time
    end)
    
    local result = {}
    for i = 1, math.min(n or 5, #tasks_list) do
        table.insert(result, tasks_list[i])
    end
    return result
end

function ToilTracker:report()
    local toil_pct = self:calculate_toil_percentage()
    print(string.format("Toil percentage: %.1f%% (target: <50%%)", toil_pct))
    
    if toil_pct > 50 then
        print("WARNING: Toil exceeds 50% of engineering time!")
        print("Top toil tasks to automate:")
        for _, item in ipairs(self:top_toil_tasks(5)) do
            print(string.format("  - %s: %d min/week (%d occurrences)",
                               item.name, 
                               item.task.total_time, 
                               item.task.count))
        end
    end
end

-- ตัวอย่างที่ 4: Automation Runner
local AutomationRunner = {}
AutomationRunner.__index = AutomationRunner

function AutomationRunner.new()
    return setmetatable({
        automations = {},
        results = {}
    }, AutomationRunner)
end

function AutomationRunner:register(name, check_fn, action_fn, metadata)
    self.automations[name] = {
        check = check_fn,
        action = action_fn,
        metadata = metadata or {},
        last_run = nil,
        run_count = 0
    }
end

function AutomationRunner:run_all(dry_run)
    print(string.format("\n=== Automation Runner %s===", dry_run and "[DRY RUN] " or ""))
    
    local triggered = 0
    local skipped = 0
    
    for name, automation in pairs(self.automations) do
        local should_run, reason = automation.check()
        
        if should_run then
            triggered = triggered + 1
            print(string.format("[TRIGGER] %s: %s", name, reason or "condition met"))
            
            if not dry_run then
                local ok, result = pcall(automation.action)
                automation.last_run = os.time()
                automation.run_count = automation.run_count + 1
                
                if ok then
                    print(string.format("[SUCCESS] %s completed", name))
                    self.results[name] = {status = "success", result = result}
                else
                    print(string.format("[FAILED] %s: %s", name, result))
                    self.results[name] = {status = "failed", error = result}
                end
            end
        else
            skipped = skipped + 1
        end
    end
    
    print(string.format("\nSummary: %d triggered, %d skipped", triggered, skipped))
end
```

## Runbooks as Code

```lua
-- ตัวอย่างที่ 5: Runbook Framework
local Runbook = {}
Runbook.__index = Runbook

function Runbook.new(name, description)
    return setmetatable({
        name = name,
        description = description,
        steps = {},
        rollback_steps = {},
        executed_steps = {}
    }, Runbook)
end

function Runbook:step(name, description, fn, rollback_fn)
    table.insert(self.steps, {
        name = name,
        description = description,
        fn = fn,
        rollback = rollback_fn
    })
end

function Runbook:execute(context)
    print(string.format("\n=== Runbook: %s ===", self.name))
    print("Description:", self.description)
    print(string.rep("-", 50))
    
    context = context or {}
    local success = true
    
    for i, step in ipairs(self.steps) do
        print(string.format("\nStep %d/%d: %s", i, #self.steps, step.name))
        if step.description then
            print("  " .. step.description)
        end
        
        local ok, result = pcall(step.fn, context)
        
        if ok then
            print(string.format("  [OK] %s", tostring(result or "")))
            table.insert(self.executed_steps, {step = step, result = result})
            context[step.name] = result
        else
            print(string.format("  [FAILED] %s", result))
            success = false
            
            -- Attempt rollback
            print("\nInitiating rollback...")
            self:rollback(context)
            break
        end
    end
    
    if success then
        print("\n=== Runbook completed successfully ===")
    else
        print("\n=== Runbook failed ===")
    end
    
    return success, context
end

function Runbook:rollback(context)
    -- Rollback in reverse order
    for i = #self.executed_steps, 1, -1 do
        local item = self.executed_steps[i]
        if item.step.rollback then
            print(string.format("  Rolling back: %s", item.step.name))
            local ok, err = pcall(item.step.rollback, context, item.result)
            if not ok then
                print(string.format("  Rollback failed: %s", err))
            else
                print(string.format("  Rollback OK"))
            end
        end
    end
end

-- ตัวอย่าง: Database Migration Runbook
local db_migration = Runbook.new(
    "Database Schema Migration v2.1",
    "Adds new indexes and modifies user table for performance"
)

db_migration:step(
    "backup_database",
    "Create full database backup before migration",
    function(ctx)
        -- Simulate backup
        local backup_id = string.format("backup_%d", os.time())
        print("    Creating backup:", backup_id)
        ctx.backup_id = backup_id
        return backup_id
    end,
    function(ctx, backup_id)
        print("    Backup", backup_id, "preserved (no restore needed)")
    end
)

db_migration:step(
    "check_disk_space",
    "Verify sufficient disk space for migration",
    function(ctx)
        local available_gb = 50  -- simulated
        if available_gb < 10 then
            error("Insufficient disk space: " .. available_gb .. "GB available, need 10GB")
        end
        return available_gb .. "GB available"
    end
)

db_migration:step(
    "run_migration",
    "Execute SQL migration script",
    function(ctx)
        print("    Running migration SQL...")
        -- Simulate migration
        return "Migration applied: 3 indexes created, 1 column added"
    end,
    function(ctx)
        print("    Rolling back migration SQL...")
    end
)

db_migration:step(
    "verify_migration",
    "Run post-migration verification",
    function(ctx)
        print("    Verifying schema integrity...")
        return "All 47 tables verified"
    end
)

db_migration:execute()
```

## Chaos Engineering

```lua
-- ตัวอย่างที่ 6: Chaos Engineering Framework
local ChaosEngine = {}
ChaosEngine.__index = ChaosEngine

function ChaosEngine.new(options)
    options = options or {}
    return setmetatable({
        enabled = options.enabled or false,
        experiments = {},
        active_experiments = {},
        blast_radius = options.blast_radius or 0.1,  -- 10% of requests
        audit_log = {}
    }, ChaosEngine)
end

function ChaosEngine:add_experiment(name, conditions, fn)
    self.experiments[name] = {
        name = name,
        conditions = conditions,
        fn = fn,
        triggered_count = 0,
        enabled = true
    }
end

function ChaosEngine:should_inject(experiment_name)
    if not self.enabled then return false end
    
    local exp = self.experiments[experiment_name]
    if not exp or not exp.enabled then return false end
    
    -- Random sampling based on blast radius
    return math.random() < self.blast_radius
end

function ChaosEngine:inject(experiment_name, context)
    if not self:should_inject(experiment_name) then
        return false
    end
    
    local exp = self.experiments[experiment_name]
    exp.triggered_count = exp.triggered_count + 1
    
    table.insert(self.audit_log, {
        timestamp = os.time(),
        experiment = experiment_name,
        context = context
    })
    
    print(string.format("[CHAOS] Injecting: %s", experiment_name))
    
    local ok, err = pcall(exp.fn, context)
    if not ok then
        -- The experiment itself failed (intended behavior)
        return true, err
    end
    
    return true
end

function ChaosEngine:report()
    print("\n=== Chaos Engineering Report ===")
    print(string.format("Status: %s", self.enabled and "ENABLED" or "DISABLED"))
    print(string.format("Blast radius: %.0f%%", self.blast_radius * 100))
    print("\nExperiments:")
    
    for name, exp in pairs(self.experiments) do
        print(string.format("  %s: %d injections, %s",
                           name, exp.triggered_count,
                           exp.enabled and "enabled" or "disabled"))
    end
end

-- Define chaos experiments
local chaos = ChaosEngine.new({enabled = true, blast_radius = 0.3})

chaos:add_experiment("latency_injection", 
    {service = "database"},
    function(ctx)
        local delay_ms = math.random(100, 2000)
        -- Simulate latency by busy waiting
        local start = os.clock()
        while (os.clock() - start) * 1000 < delay_ms do end
        error(string.format("Chaos: Injected %dms latency", delay_ms))
    end
)

chaos:add_experiment("error_injection",
    {service = "api"},
    function(ctx)
        local errors = {"500 Internal Server Error", "503 Service Unavailable", "timeout"}
        local err = errors[math.random(1, #errors)]
        error("Chaos: " .. err)
    end
)

chaos:add_experiment("memory_pressure",
    {resource = "memory"},
    function(ctx)
        -- Simulate memory allocation
        local big_table = {}
        for i = 1, 10000 do
            big_table[i] = string.rep("x", 1000)
        end
        error("Chaos: Memory pressure applied")
    end
)

-- ตัวอย่างการใช้งาน Chaos
math.randomseed(42)
for i = 1, 5 do
    local injected, err = chaos:inject("latency_injection", {request_id = i})
    if injected then
        print(string.format("Request %d affected by chaos: %s", i, err or "ok"))
    else
        print(string.format("Request %d normal", i))
    end
end

chaos:report()
```

## Load Testing

```lua
-- ตัวอย่างที่ 7: Load Testing Framework
local LoadTester = {}
LoadTester.__index = LoadTester

function LoadTester.new(options)
    options = options or {}
    return setmetatable({
        target_rps = options.rps or 100,
        duration_seconds = options.duration or 60,
        warmup_seconds = options.warmup or 10,
        metrics = {
            requests = 0,
            success = 0,
            errors = 0,
            latencies = {}
        },
        percentiles = {}
    }, LoadTester)
end

function LoadTester:run(fn)
    print(string.format("Load Test: %d RPS for %ds (warmup: %ds)",
                       self.target_rps, self.duration_seconds, self.warmup_seconds))
    
    local interval_ms = 1000 / self.target_rps
    local start_time = os.clock()
    local end_time = start_time + self.duration_seconds
    local tick_time = start_time
    
    -- Simulate load test ticks
    local ticks = self.duration_seconds * self.target_rps
    
    for tick = 1, ticks do
        local req_start = os.clock()
        
        local ok, result = pcall(fn, {
            tick = tick,
            elapsed = tick * interval_ms / 1000
        })
        
        local latency_ms = (os.clock() - req_start) * 1000
        
        self.metrics.requests = self.metrics.requests + 1
        table.insert(self.metrics.latencies, latency_ms)
        
        if ok then
            self.metrics.success = self.metrics.success + 1
        else
            self.metrics.errors = self.metrics.errors + 1
        end
    end
    
    self:calculate_percentiles()
    self:report()
end

function LoadTester:calculate_percentiles()
    local latencies = self.metrics.latencies
    table.sort(latencies)
    
    local function percentile(p)
        local idx = math.ceil(#latencies * p / 100)
        return latencies[idx] or 0
    end
    
    self.percentiles = {
        p50 = percentile(50),
        p75 = percentile(75),
        p90 = percentile(90),
        p95 = percentile(95),
        p99 = percentile(99),
        p999 = percentile(99.9)
    }
end

function LoadTester:report()
    local total = self.metrics.requests
    local success_rate = total > 0 and (self.metrics.success / total * 100) or 0
    local actual_rps = total / self.duration_seconds
    
    -- Calculate average
    local sum = 0
    for _, l in ipairs(self.metrics.latencies) do sum = sum + l end
    local avg = #self.metrics.latencies > 0 and sum / #self.metrics.latencies or 0
    
    print(string.format([[
=== Load Test Results ===
Requests:     %d
Success rate: %.2f%%
Actual RPS:   %.1f (target: %d)

Latency (ms):
  Average: %.2f
  p50:     %.2f
  p75:     %.2f
  p90:     %.2f
  p95:     %.2f
  p99:     %.2f
  p99.9:   %.2f

Errors: %d]],
        total, success_rate, actual_rps, self.target_rps,
        avg,
        self.percentiles.p50, self.percentiles.p75,
        self.percentiles.p90, self.percentiles.p95,
        self.percentiles.p99, self.percentiles.p999,
        self.metrics.errors
    ))
end

-- ตัวอย่างการทดสอบ
local tester = LoadTester.new({rps = 100, duration = 1})

tester:run(function(ctx)
    -- Simulate variable latency
    local latency = math.random(1, 100)
    if math.random() < 0.01 then
        error("Simulated error")
    end
    return true
end)
```

## Capacity Planning

```lua
-- ตัวอย่างที่ 8: Capacity Planning Model
local CapacityPlanner = {}
CapacityPlanner.__index = CapacityPlanner

function CapacityPlanner.new()
    return setmetatable({
        current_metrics = {},
        growth_model = nil,
        resources = {}
    }, CapacityPlanner)
end

function CapacityPlanner:set_current_load(metric_name, value, unit)
    self.current_metrics[metric_name] = {
        value = value,
        unit = unit or ""
    }
end

function CapacityPlanner:set_growth_rate(daily_growth_percent)
    self.growth_model = {
        daily_rate = daily_growth_percent / 100
    }
end

function CapacityPlanner:add_resource(name, current_capacity, unit, cost_per_unit)
    self.resources[name] = {
        current = current_capacity,
        unit = unit,
        cost_per_unit = cost_per_unit or 0
    }
end

function CapacityPlanner:project(days)
    if not self.growth_model then
        error("Growth model not set")
    end
    
    local projections = {}
    local growth = self.growth_model.daily_rate
    
    for day = 1, days, math.floor(days/10) do
        local multiplier = (1 + growth) ^ day
        local projection = {
            day = day,
            metrics = {},
            resource_needs = {}
        }
        
        for metric_name, metric in pairs(self.current_metrics) do
            projection.metrics[metric_name] = {
                value = metric.value * multiplier,
                unit = metric.unit
            }
        end
        
        table.insert(projections, projection)
    end
    
    return projections
end

function CapacityPlanner:when_capacity_exhausted(resource_name, metric_name)
    if not self.growth_model then return nil end
    
    local resource = self.resources[resource_name]
    local metric = self.current_metrics[metric_name]
    
    if not resource or not metric then return nil end
    
    -- Find when metric will exceed resource capacity
    local day = 0
    local value = metric.value
    
    while value < resource.current and day < 3650 do
        day = day + 1
        value = value * (1 + self.growth_model.daily_rate)
    end
    
    return day
end

function CapacityPlanner:report(forecast_days)
    forecast_days = forecast_days or 90
    
    print("\n=== Capacity Planning Report ===")
    print(string.format("Growth rate: %.1f%%/day", self.growth_model.daily_rate * 100))
    print(string.format("Forecast period: %d days", forecast_days))
    
    print("\nResource exhaustion timeline:")
    for resource_name, resource in pairs(self.resources) do
        for metric_name in pairs(self.current_metrics) do
            local days = self:when_capacity_exhausted(resource_name, metric_name)
            if days and days <= forecast_days then
                print(string.format("  %s exhausted in %d days (based on %s growth)",
                                   resource_name, days, metric_name))
            end
        end
    end
    
    print("\n30/60/90 day projections:")
    for _, days in ipairs({30, 60, 90}) do
        if days <= forecast_days then
            local multiplier = (1 + self.growth_model.daily_rate) ^ days
            print(string.format("\n  Day %d (%.1fx growth):", days, multiplier))
            for metric_name, metric in pairs(self.current_metrics) do
                print(string.format("    %s: %.0f %s",
                                   metric_name,
                                   metric.value * multiplier,
                                   metric.unit))
            end
        end
    end
end

-- ตัวอย่างการใช้งาน
local planner = CapacityPlanner.new()
planner:set_current_load("requests_per_second", 1000, "RPS")
planner:set_current_load("storage_gb", 5000, "GB")
planner:set_current_load("active_users", 100000, "users")
planner:set_growth_rate(2)  -- 2% daily growth

planner:add_resource("web_servers", 5000, "RPS", 100)  -- current capacity
planner:add_resource("storage", 20000, "GB", 0.02)

planner:report(90)
```

## Incident Response Automation

```lua
-- ตัวอย่างที่ 9: Incident Response System
local IncidentManager = {}
IncidentManager.__index = IncidentManager

IncidentManager.SEVERITY = {
    P1 = {level = 1, name = "Critical", response_time = 5},   -- 5 minutes
    P2 = {level = 2, name = "High", response_time = 30},
    P3 = {level = 3, name = "Medium", response_time = 120},
    P4 = {level = 4, name = "Low", response_time = 480}
}

function IncidentManager.new()
    return setmetatable({
        incidents = {},
        incident_counter = 0,
        escalation_rules = {},
        notification_channels = {}
    }, IncidentManager)
end

function IncidentManager:create(title, severity, description)
    self.incident_counter = self.incident_counter + 1
    
    local incident = {
        id = string.format("INC-%04d", self.incident_counter),
        title = title,
        severity = severity,
        description = description,
        status = "open",
        created_at = os.time(),
        updated_at = os.time(),
        timeline = {},
        responders = {},
        actions_taken = {}
    }
    
    self.incidents[incident.id] = incident
    
    -- Auto-notify
    self:notify(incident, "incident_created")
    
    -- Start response timer
    self:schedule_escalation(incident)
    
    return incident
end

function IncidentManager:update(incident_id, status, message, responder)
    local incident = self.incidents[incident_id]
    if not incident then
        error("Incident not found: " .. incident_id)
    end
    
    incident.status = status
    incident.updated_at = os.time()
    
    table.insert(incident.timeline, {
        timestamp = os.time(),
        status = status,
        message = message,
        responder = responder
    })
    
    if status == "resolved" then
        incident.resolved_at = os.time()
        incident.ttd = incident.resolved_at - incident.created_at  -- Time to detect
        self:notify(incident, "incident_resolved")
        self:generate_pir_template(incident)
    end
end

function IncidentManager:notify(incident, event_type)
    local sev = IncidentManager.SEVERITY[incident.severity]
    
    print(string.format("[NOTIFY] [%s] Incident %s: %s",
                       event_type, incident.id, incident.title))
    
    -- Simulate different notification channels based on severity
    if sev.level <= 2 then
        print(string.format("  -> PagerDuty: @oncall (P%d)", sev.level))
        print(string.format("  -> Slack: #incidents-critical"))
        if sev.level == 1 then
            print("  -> Phone: On-call engineer")
            print("  -> Phone: Engineering manager")
        end
    else
        print(string.format("  -> Slack: #incidents"))
        print(string.format("  -> Email: team@company.com"))
    end
end

function IncidentManager:schedule_escalation(incident)
    local sev = IncidentManager.SEVERITY[incident.severity]
    print(string.format("[ESCALATION] If not acknowledged in %d min: escalate %s",
                       sev.response_time, incident.id))
end

function IncidentManager:generate_pir_template(incident)
    local ttd = incident.resolved_at - incident.created_at
    
    print(string.format([[

=== Post-Incident Review Template ===
Incident: %s
Title: %s
Severity: %s
Duration: %d minutes

1. Summary
   [What happened in 1-2 sentences]

2. Impact
   - Users affected: [number]
   - Services affected: [list]
   - Revenue impact: [amount]

3. Timeline
]],
        incident.id,
        incident.title,
        incident.severity,
        math.floor(ttd / 60)
    ))
    
    for _, event in ipairs(incident.timeline) do
        print(string.format("   %s: %s", 
                           os.date("%H:%M", event.timestamp),
                           event.message))
    end
    
    print([[
4. Root Cause
   [Technical root cause]

5. Contributing Factors
   [List of factors]

6. Action Items
   - [ ] Fix: [what to fix]
   - [ ] Improve: [monitoring/alerting improvements]
   - [ ] Prevent: [process improvements]

7. Lessons Learned
   [Key takeaways]
]])
end

-- ตัวอย่างการใช้งาน Incident Manager
local im = IncidentManager.new()

local incident = im:create(
    "API latency spike >500ms",
    "P2",
    "Users experiencing slow responses from /api/v1/search endpoint"
)

print("\n--- Incident Response Simulation ---")

-- Simulate response
im:update(incident.id, "acknowledged", 
          "Team notified, investigating", "alice@company.com")

im:update(incident.id, "investigating",
          "Found high CPU usage on db-primary, running slow queries", "bob@company.com")

im:update(incident.id, "mitigating",
          "Adding index to resolve slow query", "bob@company.com")

im:update(incident.id, "resolved",
          "Index added, latency back to normal <50ms", "bob@company.com")
```

## Health Check Automation

```lua
-- ตัวอย่างที่ 10: Health Check System
local HealthChecker = {}
HealthChecker.__index = HealthChecker

function HealthChecker.new()
    return setmetatable({
        checks = {},
        results = {},
        history = {}
    }, HealthChecker)
end

function HealthChecker:add_check(name, fn, options)
    options = options or {}
    self.checks[name] = {
        fn = fn,
        timeout = options.timeout or 5,
        critical = options.critical ~= false,
        tags = options.tags or {}
    }
end

function HealthChecker:run_check(name)
    local check = self.checks[name]
    if not check then
        return {status = "unknown", message = "Check not found"}
    end
    
    local start = os.clock()
    local ok, result = pcall(check.fn)
    local duration = os.clock() - start
    
    local health_result = {
        name = name,
        timestamp = os.time(),
        duration_ms = duration * 1000,
        critical = check.critical
    }
    
    if ok then
        if type(result) == "table" then
            health_result.status = result.status or "healthy"
            health_result.message = result.message
            health_result.details = result.details
        elseif result == true then
            health_result.status = "healthy"
        else
            health_result.status = "degraded"
            health_result.message = tostring(result)
        end
    else
        health_result.status = "unhealthy"
        health_result.message = result
    end
    
    return health_result
end

function HealthChecker:run_all()
    local results = {
        timestamp = os.time(),
        overall = "healthy",
        checks = {}
    }
    
    for name in pairs(self.checks) do
        local result = self:run_check(name)
        results.checks[name] = result
        
        if result.status == "unhealthy" and result.critical then
            results.overall = "unhealthy"
        elseif result.status == "degraded" and results.overall == "healthy" then
            results.overall = "degraded"
        end
    end
    
    -- Store in history
    table.insert(self.history, results)
    if #self.history > 100 then
        table.remove(self.history, 1)
    end
    
    return results
end

function HealthChecker:report(results)
    local icons = {healthy = "✓", degraded = "~", unhealthy = "✗", unknown = "?"}
    
    print(string.format("\n=== Health Check Report [%s] ===", 
                       os.date("%Y-%m-%d %H:%M:%S")))
    print(string.format("Overall: %s", results.overall:upper()))
    print()
    
    for name, check in pairs(results.checks) do
        local icon = icons[check.status] or "?"
        print(string.format("  [%s] %s (%.1fms)",
                           icon, name, check.duration_ms))
        if check.message then
            print(string.format("      %s", check.message))
        end
    end
end

-- ตัวอย่างการ Register Health Checks
local hc = HealthChecker.new()

hc:add_check("database", function()
    -- Simulate DB check
    local ping_time = math.random(1, 50)
    if ping_time > 40 then
        return {status = "degraded", message = string.format("Slow ping: %dms", ping_time)}
    end
    return {status = "healthy", message = string.format("Ping: %dms", ping_time)}
end, {critical = true})

hc:add_check("redis", function()
    return {status = "healthy", message = "Connected, 1234 keys"}
end, {critical = true})

hc:add_check("external_api", function()
    if math.random() < 0.1 then
        error("Connection timeout")
    end
    return {status = "healthy", message = "API responding"}
end, {critical = false})

hc:add_check("disk_space", function()
    local used_pct = math.random(60, 95)
    if used_pct > 90 then
        return {status = "unhealthy", message = used_pct .. "% used"}
    elseif used_pct > 80 then
        return {status = "degraded", message = used_pct .. "% used"}
    end
    return {status = "healthy", message = used_pct .. "% used"}
end)

math.randomseed(12345)
local results = hc:run_all()
hc:report(results)
```

## Deployment Verification

```lua
-- ตัวอย่างที่ 11: Deployment Verification Framework
local DeploymentVerifier = {}
DeploymentVerifier.__index = DeploymentVerifier

function DeploymentVerifier.new(deployment_id)
    return setmetatable({
        deployment_id = deployment_id,
        version = nil,
        checks = {},
        smoke_tests = {},
        passed = 0,
        failed = 0
    }, DeploymentVerifier)
end

function DeploymentVerifier:add_smoke_test(name, fn)
    table.insert(self.smoke_tests, {name = name, fn = fn})
end

function DeploymentVerifier:verify(options)
    options = options or {}
    print(string.format("\n=== Deployment Verification: %s ===", self.deployment_id))
    
    self.passed = 0
    self.failed = 0
    
    -- Run smoke tests
    print("\nRunning smoke tests...")
    for _, test in ipairs(self.smoke_tests) do
        local ok, err = pcall(test.fn)
        if ok then
            self.passed = self.passed + 1
            print(string.format("  [PASS] %s", test.name))
        else
            self.failed = self.failed + 1
            print(string.format("  [FAIL] %s: %s", test.name, err))
        end
    end
    
    local total = self.passed + self.failed
    local success_rate = total > 0 and (self.passed / total * 100) or 0
    
    print(string.format("\nResults: %d/%d passed (%.0f%%)", 
                       self.passed, total, success_rate))
    
    local threshold = options.success_threshold or 100
    if success_rate >= threshold then
        print("Deployment verification: PASSED")
        return true
    else
        print("Deployment verification: FAILED")
        return false
    end
end

-- ตัวอย่างที่ 12: Canary Analysis
local CanaryAnalyzer = {}
CanaryAnalyzer.__index = CanaryAnalyzer

function CanaryAnalyzer.new(options)
    options = options or {}
    return setmetatable({
        baseline_metrics = {},
        canary_metrics = {},
        thresholds = options.thresholds or {
            error_rate_increase = 0.01,      -- max 1% increase
            latency_p99_increase = 0.20,     -- max 20% increase
            success_rate_decrease = 0.005    -- max 0.5% decrease
        }
    }, CanaryAnalyzer)
end

function CanaryAnalyzer:record_baseline(metric, value)
    if not self.baseline_metrics[metric] then
        self.baseline_metrics[metric] = {}
    end
    table.insert(self.baseline_metrics[metric], value)
end

function CanaryAnalyzer:record_canary(metric, value)
    if not self.canary_metrics[metric] then
        self.canary_metrics[metric] = {}
    end
    table.insert(self.canary_metrics[metric], value)
end

function CanaryAnalyzer:analyze()
    local function avg(arr)
        if #arr == 0 then return 0 end
        local sum = 0
        for _, v in ipairs(arr) do sum = sum + v end
        return sum / #arr
    end
    
    local report = {
        passed = true,
        issues = {}
    }
    
    -- Compare error rates
    local baseline_err = avg(self.baseline_metrics.error_rate or {0})
    local canary_err = avg(self.canary_metrics.error_rate or {0})
    local err_increase = canary_err - baseline_err
    
    if err_increase > self.thresholds.error_rate_increase then
        report.passed = false
        table.insert(report.issues, string.format(
            "Error rate increased by %.2f%% (baseline: %.2f%%, canary: %.2f%%)",
            err_increase * 100, baseline_err * 100, canary_err * 100
        ))
    end
    
    -- Compare p99 latency
    local baseline_lat = avg(self.baseline_metrics.latency_p99 or {0})
    local canary_lat = avg(self.canary_metrics.latency_p99 or {0})
    
    if baseline_lat > 0 then
        local lat_increase = (canary_lat - baseline_lat) / baseline_lat
        if lat_increase > self.thresholds.latency_p99_increase then
            report.passed = false
            table.insert(report.issues, string.format(
                "p99 latency increased by %.0f%% (baseline: %.0fms, canary: %.0fms)",
                lat_increase * 100, baseline_lat, canary_lat
            ))
        end
    end
    
    return report
end

function CanaryAnalyzer:verdict()
    local report = self:analyze()
    
    print("\n=== Canary Analysis ===")
    
    if report.passed then
        print("Verdict: PASS - Canary looks healthy, proceed with rollout")
    else
        print("Verdict: FAIL - Issues detected in canary:")
        for _, issue in ipairs(report.issues) do
            print("  - " .. issue)
        end
        print("Recommendation: ROLLBACK canary deployment")
    end
    
    return report.passed
end

-- ตัวอย่างการใช้งาน Canary
local canary = CanaryAnalyzer.new()

-- Simulate baseline metrics
for _ = 1, 100 do
    canary:record_baseline("error_rate", math.random(0, 2) / 1000)
    canary:record_baseline("latency_p99", math.random(80, 120))
end

-- Simulate canary metrics (with slight regression)
for _ = 1, 20 do
    canary:record_canary("error_rate", math.random(5, 15) / 1000)  -- higher error rate
    canary:record_canary("latency_p99", math.random(90, 150))      -- slightly higher latency
end

canary:verdict()
```

## Feature Flags

```lua
-- ตัวอย่างที่ 13: Feature Flag System
local FeatureFlags = {}
FeatureFlags.__index = FeatureFlags

function FeatureFlags.new()
    return setmetatable({
        flags = {},
        overrides = {},
        evaluations = {}
    }, FeatureFlags)
end

function FeatureFlags:define(name, config)
    self.flags[name] = {
        name = name,
        enabled = config.enabled or false,
        rollout_percentage = config.rollout_percentage or 0,
        targeting = config.targeting or {},
        variants = config.variants,
        metadata = config.metadata or {}
    }
end

function FeatureFlags:evaluate(flag_name, context)
    local flag = self.flags[flag_name]
    if not flag then
        return false, nil
    end
    
    -- Check override for specific user/context
    local override_key = flag_name .. ":" .. (context.user_id or "")
    if self.overrides[override_key] ~= nil then
        return self.overrides[override_key], nil
    end
    
    -- Check if globally disabled
    if not flag.enabled then
        return false, nil
    end
    
    -- Targeting rules
    for _, rule in ipairs(flag.targeting) do
        if self:_evaluate_rule(rule, context) then
            return rule.value ~= false, rule.variant
        end
    end
    
    -- Percentage rollout
    if flag.rollout_percentage > 0 then
        local hash = self:_hash(flag_name, context.user_id or "")
        local bucket = hash % 100
        if bucket < flag.rollout_percentage then
            -- Return variant if defined
            if flag.variants and #flag.variants > 0 then
                local variant_idx = (hash % #flag.variants) + 1
                return true, flag.variants[variant_idx]
            end
            return true, nil
        end
    end
    
    return false, nil
end

function FeatureFlags:_evaluate_rule(rule, context)
    if rule.user_ids and context.user_id then
        for _, id in ipairs(rule.user_ids) do
            if id == context.user_id then return true end
        end
    end
    
    if rule.countries and context.country then
        for _, country in ipairs(rule.countries) do
            if country == context.country then return true end
        end
    end
    
    if rule.plan and context.plan then
        if rule.plan == context.plan then return true end
    end
    
    return false
end

function FeatureFlags:_hash(flag_name, user_id)
    -- Simple hash function
    local str = flag_name .. ":" .. user_id
    local hash = 0
    for i = 1, #str do
        hash = (hash * 31 + str:byte(i)) % (2^32)
    end
    return hash
end

function FeatureFlags:override(flag_name, user_id, value)
    self.overrides[flag_name .. ":" .. user_id] = value
end

-- ตัวอย่างการใช้งาน Feature Flags
local ff = FeatureFlags.new()

ff:define("new_checkout_flow", {
    enabled = true,
    rollout_percentage = 20,  -- 20% of users
    targeting = {
        {user_ids = {"beta_user_1", "beta_user_2"}, value = true},
        {plan = "enterprise", value = true}
    },
    variants = {"variant_a", "variant_b"}
})

ff:define("dark_mode", {
    enabled = true,
    rollout_percentage = 50
})

-- Test with different users
local test_users = {
    {user_id = "beta_user_1", plan = "pro", country = "US"},
    {user_id = "user_12345", plan = "free", country = "TH"},
    {user_id = "enterprise_user", plan = "enterprise", country = "UK"},
}

for _, user in ipairs(test_users) do
    local enabled, variant = ff:evaluate("new_checkout_flow", user)
    print(string.format("User %s: new_checkout=%s (variant=%s)",
                       user.user_id, tostring(enabled), tostring(variant)))
end
```

## Gradual Rollout

```lua
-- ตัวอย่างที่ 14: Progressive Deployment Controller
local ProgressiveDeployment = {}
ProgressiveDeployment.__index = ProgressiveDeployment

function ProgressiveDeployment.new(options)
    options = options or {}
    return setmetatable({
        stages = options.stages or {1, 5, 10, 25, 50, 100},  -- percentages
        current_stage = 0,
        current_percentage = 0,
        metrics_window = options.metrics_window or 300,  -- 5 minutes
        auto_advance = options.auto_advance ~= false,
        metrics = {
            error_rate = 0,
            latency_p99 = 0,
            success_rate = 1
        },
        thresholds = {
            max_error_rate = 0.01,
            max_latency_increase = 0.2
        },
        callbacks = {}
    }, ProgressiveDeployment)
end

function ProgressiveDeployment:on(event, fn)
    if not self.callbacks[event] then
        self.callbacks[event] = {}
    end
    table.insert(self.callbacks[event], fn)
end

function ProgressiveDeployment:emit(event, data)
    if self.callbacks[event] then
        for _, fn in ipairs(self.callbacks[event]) do
            fn(data)
        end
    end
end

function ProgressiveDeployment:advance()
    if self.current_stage >= #self.stages then
        print("Deployment already at 100%")
        return false
    end
    
    self.current_stage = self.current_stage + 1
    local new_pct = self.stages[self.current_stage]
    
    print(string.format("Advancing deployment to %d%%", new_pct))
    self.current_percentage = new_pct
    
    self:emit("stage_changed", {
        stage = self.current_stage,
        percentage = new_pct
    })
    
    if new_pct == 100 then
        self:emit("deployment_complete", {})
    end
    
    return true
end

function ProgressiveDeployment:rollback()
    if self.current_stage <= 0 then
        print("Nothing to rollback")
        return false
    end
    
    local old_pct = self.stages[self.current_stage]
    self.current_stage = 0
    self.current_percentage = 0
    
    print(string.format("ROLLING BACK from %d%% to 0%%", old_pct))
    self:emit("rollback", {from_percentage = old_pct})
    
    return true
end

function ProgressiveDeployment:update_metrics(metrics)
    self.metrics = metrics
    
    -- Check if we should auto-advance or rollback
    if self.auto_advance then
        if metrics.error_rate > self.thresholds.max_error_rate then
            print(string.format("ERROR: Error rate %.2f%% exceeds threshold %.2f%%",
                               metrics.error_rate * 100,
                               self.thresholds.max_error_rate * 100))
            self:rollback()
        elseif self.current_stage < #self.stages then
            self:advance()
        end
    end
end

function ProgressiveDeployment:status()
    print(string.format([[
=== Progressive Deployment Status ===
Current percentage: %d%%
Stage: %d/%d
Error rate: %.2f%%
Latency p99: %.0fms
Success rate: %.2f%%]],
        self.current_percentage,
        self.current_stage,
        #self.stages,
        self.metrics.error_rate * 100,
        self.metrics.latency_p99,
        self.metrics.success_rate * 100
    ))
end

-- ตัวอย่างการใช้งาน
local pd = ProgressiveDeployment.new({
    stages = {1, 5, 20, 50, 100},
    auto_advance = true
})

pd:on("stage_changed", function(data)
    print(string.format("Stage changed: %d%% traffic", data.percentage))
end)

pd:on("rollback", function(data)
    print(string.format("ROLLBACK: Alert team! Rolling back from %d%%", data.from_percentage))
end)

pd:on("deployment_complete", function()
    print("Deployment complete! 100% traffic on new version.")
end)

-- Simulate deployment
pd:advance()  -- 1%
pd:update_metrics({error_rate = 0.001, latency_p99 = 95, success_rate = 0.999})
pd:update_metrics({error_rate = 0.001, latency_p99 = 98, success_rate = 0.999})
pd:update_metrics({error_rate = 0.001, latency_p99 = 100, success_rate = 0.999})
pd:update_metrics({error_rate = 0.001, latency_p99 = 102, success_rate = 0.999})
pd:update_metrics({error_rate = 0.001, latency_p99 = 98, success_rate = 0.999})

pd:status()
```

## On-Call Tooling

```lua
-- ตัวอย่างที่ 15: On-Call Management System
local OnCallManager = {}
OnCallManager.__index = OnCallManager

function OnCallManager.new()
    return setmetatable({
        rotations = {},
        current_oncall = {},
        escalation_policies = {},
        notifications = {}
    }, OnCallManager)
end

function OnCallManager:add_rotation(name, members, rotation_days)
    self.rotations[name] = {
        name = name,
        members = members,
        rotation_days = rotation_days or 7,
        start_epoch = os.time()
    }
end

function OnCallManager:get_current_oncall(rotation_name)
    local rotation = self.rotations[rotation_name]
    if not rotation then return nil end
    
    local elapsed_days = math.floor((os.time() - rotation.start_epoch) / 86400)
    local member_idx = (math.floor(elapsed_days / rotation.rotation_days) % #rotation.members) + 1
    
    return rotation.members[member_idx]
end

function OnCallManager:add_escalation_policy(name, levels)
    self.escalation_policies[name] = {
        name = name,
        levels = levels
    }
end

function OnCallManager:escalate(policy_name, incident, level)
    local policy = self.escalation_policies[policy_name]
    if not policy then
        print("Unknown policy:", policy_name)
        return
    end
    
    level = level or 1
    if level > #policy.levels then
        print("Max escalation level reached")
        return
    end
    
    local target = policy.levels[level]
    print(string.format("[ESCALATION L%d] Notifying %s for incident %s",
                       level, target, incident.id))
    
    table.insert(self.notifications, {
        timestamp = os.time(),
        level = level,
        target = target,
        incident = incident.id
    })
    
    return level + 1  -- next escalation level
end

function OnCallManager:generate_oncall_report()
    print("\n=== On-Call Report ===")
    print(string.format("Generated: %s", os.date("%Y-%m-%d %H:%M")))
    print()
    
    for name, rotation in pairs(self.rotations) do
        local current = self:get_current_oncall(name)
        print(string.format("Rotation: %s", name))
        print(string.format("  Current on-call: %s", current or "none"))
        print(string.format("  Members: %s", table.concat(rotation.members, ", ")))
        print()
    end
end

-- ตัวอย่างการใช้งาน
local ocm = OnCallManager.new()

ocm:add_rotation("backend", {
    "alice@company.com",
    "bob@company.com", 
    "charlie@company.com"
}, 7)

ocm:add_rotation("frontend", {
    "dave@company.com",
    "eve@company.com"
}, 5)

ocm:add_escalation_policy("default", {
    "primary_oncall@company.com",
    "secondary_oncall@company.com",
    "engineering_manager@company.com",
    "cto@company.com"
})

ocm:generate_oncall_report()

-- Simulate escalation
local test_incident = {id = "INC-0001", severity = "P1"}
local next_level = ocm:escalate("default", test_incident, 1)
-- Wait 5 minutes...
next_level = ocm:escalate("default", test_incident, next_level)
```

## Auto-Scaling Triggers

```lua
-- ตัวอย่างที่ 16: Auto-Scaling Controller
local AutoScaler = {}
AutoScaler.__index = AutoScaler

function AutoScaler.new(options)
    options = options or {}
    return setmetatable({
        min_instances = options.min or 2,
        max_instances = options.max or 50,
        current_instances = options.initial or 2,
        scale_up_threshold = options.scale_up or 0.7,    -- 70% CPU
        scale_down_threshold = options.scale_down or 0.3,  -- 30% CPU
        cooldown_seconds = options.cooldown or 120,
        last_scale_time = 0,
        metrics_history = {},
        scale_events = {}
    }, AutoScaler)
end

function AutoScaler:update_metrics(metrics)
    table.insert(self.metrics_history, {
        timestamp = os.time(),
        cpu = metrics.cpu or 0,
        memory = metrics.memory or 0,
        rps = metrics.rps or 0,
        latency = metrics.latency or 0
    })
    
    -- Keep only last 100 data points
    if #self.metrics_history > 100 then
        table.remove(self.metrics_history, 1)
    end
    
    self:evaluate()
end

function AutoScaler:evaluate()
    if #self.metrics_history < 3 then return end
    
    -- Check cooldown
    if os.time() - self.last_scale_time < self.cooldown_seconds then
        return
    end
    
    -- Calculate average of recent metrics
    local recent = {}
    local count = math.min(5, #self.metrics_history)
    for i = #self.metrics_history - count + 1, #self.metrics_history do
        table.insert(recent, self.metrics_history[i])
    end
    
    local avg_cpu = 0
    for _, m in ipairs(recent) do avg_cpu = avg_cpu + m.cpu end
    avg_cpu = avg_cpu / #recent
    
    -- Make scaling decision
    if avg_cpu > self.scale_up_threshold then
        self:scale_up(avg_cpu)
    elseif avg_cpu < self.scale_down_threshold then
        self:scale_down(avg_cpu)
    end
end

function AutoScaler:scale_up(trigger_value)
    local new_count = math.min(
        self.max_instances,
        math.ceil(self.current_instances * 1.5)  -- 50% increase
    )
    
    if new_count <= self.current_instances then
        print("Already at max instances:", self.max_instances)
        return
    end
    
    local old_count = self.current_instances
    self.current_instances = new_count
    self.last_scale_time = os.time()
    
    local event = {
        type = "scale_up",
        from = old_count,
        to = new_count,
        trigger = trigger_value,
        timestamp = os.time()
    }
    table.insert(self.scale_events, event)
    
    print(string.format("[SCALE UP] %d -> %d instances (CPU: %.0f%%)",
                       old_count, new_count, trigger_value * 100))
end

function AutoScaler:scale_down(trigger_value)
    local new_count = math.max(
        self.min_instances,
        math.floor(self.current_instances * 0.75)  -- 25% decrease
    )
    
    if new_count >= self.current_instances then
        return  -- Already at min
    end
    
    local old_count = self.current_instances
    self.current_instances = new_count
    self.last_scale_time = os.time()
    
    local event = {
        type = "scale_down",
        from = old_count,
        to = new_count,
        trigger = trigger_value,
        timestamp = os.time()
    }
    table.insert(self.scale_events, event)
    
    print(string.format("[SCALE DOWN] %d -> %d instances (CPU: %.0f%%)",
                       old_count, new_count, trigger_value * 100))
end

function AutoScaler:report()
    print(string.format([[
=== Auto-Scaler Report ===
Current instances: %d (min: %d, max: %d)
Scale events: %d
Last scale: %s]],
        self.current_instances,
        self.min_instances,
        self.max_instances,
        #self.scale_events,
        self.last_scale_time > 0 and os.date("%H:%M:%S", self.last_scale_time) or "never"
    ))
    
    if #self.scale_events > 0 then
        print("\nRecent scale events:")
        local start = math.max(1, #self.scale_events - 5)
        for i = start, #self.scale_events do
            local e = self.scale_events[i]
            print(string.format("  %s: %d -> %d (%.0f%% CPU)",
                               e.type, e.from, e.to, e.trigger * 100))
        end
    end
end

-- ตัวอย่างการ simulate load
local scaler = AutoScaler.new({min = 2, max = 20, initial = 3})

-- Simulate traffic spike
local cpu_pattern = {0.3, 0.4, 0.6, 0.75, 0.85, 0.9, 0.85, 0.7, 0.5, 0.3, 0.2}

for _, cpu in ipairs(cpu_pattern) do
    scaler:update_metrics({cpu = cpu + (math.random() - 0.5) * 0.1})
    scaler.last_scale_time = scaler.last_scale_time - 130  -- bypass cooldown for demo
end

scaler:report()
```

## Monitoring and Alerting

```lua
-- ตัวอย่างที่ 17: Alert Rule Engine
local AlertEngine = {}
AlertEngine.__index = AlertEngine

function AlertEngine.new()
    return setmetatable({
        rules = {},
        active_alerts = {},
        alert_history = {}
    }, AlertEngine)
end

function AlertEngine:add_rule(name, condition_fn, options)
    options = options or {}
    self.rules[name] = {
        name = name,
        condition = condition_fn,
        severity = options.severity or "warning",
        message_fn = options.message or function(v) return name .. ": " .. tostring(v) end,
        for_duration = options.for_duration or 0,  -- seconds
        first_triggered = nil,
        annotations = options.annotations or {}
    }
end

function AlertEngine:evaluate(metrics)
    local timestamp = os.time()
    
    for rule_name, rule in pairs(self.rules) do
        local triggered, value = rule.condition(metrics)
        
        if triggered then
            if not rule.first_triggered then
                rule.first_triggered = timestamp
            end
            
            -- Check if condition persists long enough
            local duration = timestamp - rule.first_triggered
            if duration >= rule.for_duration then
                -- Fire alert
                if not self.active_alerts[rule_name] then
                    self.active_alerts[rule_name] = {
                        rule = rule_name,
                        severity = rule.severity,
                        message = rule.message_fn(value),
                        started_at = rule.first_triggered,
                        annotations = rule.annotations
                    }
                    self:fire_alert(self.active_alerts[rule_name])
                end
            end
        else
            rule.first_triggered = nil
            if self.active_alerts[rule_name] then
                self:resolve_alert(self.active_alerts[rule_name])
                self.active_alerts[rule_name] = nil
            end
        end
    end
end

function AlertEngine:fire_alert(alert)
    alert.fired_at = os.time()
    table.insert(self.alert_history, alert)
    
    local icons = {critical = "[!]", warning = "[~]", info = "[i]"}
    print(string.format("%s ALERT [%s]: %s",
                       icons[alert.severity] or "[?]",
                       alert.severity:upper(),
                       alert.message))
end

function AlertEngine:resolve_alert(alert)
    alert.resolved_at = os.time()
    local duration = alert.resolved_at - alert.fired_at
    print(string.format("[OK] Resolved [%s]: %s (after %ds)",
                       alert.severity:upper(),
                       alert.rule,
                       duration))
end

-- ตัวอย่างการตั้ง Alert Rules
local alerting = AlertEngine.new()

alerting:add_rule("high_error_rate", function(metrics)
    return metrics.error_rate > 0.05, metrics.error_rate
end, {
    severity = "critical",
    for_duration = 60,
    message = function(v)
        return string.format("Error rate is %.1f%% (threshold: 5%%)", v * 100)
    end
})

alerting:add_rule("high_latency", function(metrics)
    return metrics.latency_p99 > 500, metrics.latency_p99
end, {
    severity = "warning",
    for_duration = 120,
    message = function(v)
        return string.format("p99 latency is %dms (threshold: 500ms)", v)
    end
})

alerting:add_rule("low_success_rate", function(metrics)
    return metrics.success_rate < 0.99, metrics.success_rate
end, {
    severity = "critical",
    for_duration = 0,  -- immediate
    message = function(v)
        return string.format("Success rate dropped to %.1f%%", v * 100)
    end
})

-- Simulate metrics changes
local scenarios = {
    {error_rate = 0.01, latency_p99 = 100, success_rate = 0.99},  -- normal
    {error_rate = 0.06, latency_p99 = 550, success_rate = 0.94},  -- degraded
    {error_rate = 0.06, latency_p99 = 100, success_rate = 0.99},  -- recovering
    {error_rate = 0.01, latency_p99 = 100, success_rate = 0.99},  -- normal again
}

for _, metrics in ipairs(scenarios) do
    print(string.format("\n--- Metrics: err=%.0f%% lat=%dms success=%.1f%% ---",
                       metrics.error_rate * 100, metrics.latency_p99, 
                       metrics.success_rate * 100))
    alerting:evaluate(metrics)
end
```

## สรุปบทที่ 87

ในบทนี้เราได้เรียนรู้ Production Systems Engineering ครอบคลุม:

1. **SLA/SLO/SLI** - การวัดและติดตาม service reliability
2. **Error Budgets** - การจัดการ budget สำหรับ downtime
3. **Toil Automation** - การลด toil ด้วยการ automate
4. **Runbooks as Code** - Executable runbooks พร้อม rollback
5. **Chaos Engineering** - การทดสอบ resilience
6. **Load Testing** - การวัด performance ภายใต้ load
7. **Capacity Planning** - การวางแผนขยายระบบ
8. **Incident Response** - การจัดการ incidents อย่างเป็นระบบ
9. **Health Checks** - การตรวจสอบสุขภาพระบบอย่างต่อเนื่อง
10. **Deployment Verification** - การตรวจสอบ deployment
11. **Canary Analysis** - การวิเคราะห์ canary deployments
12. **Feature Flags** - การควบคุม feature releases
13. **Gradual Rollout** - Progressive deployment with auto-rollback
14. **On-Call Tooling** - การจัดการ on-call rotations
15. **Auto-Scaling** - การปรับขนาดระบบอัตโนมัติ
16. **Alert Rule Engine** - การสร้างและจัดการ alerts

SRE ที่ดีต้องมีทั้ง engineering rigor และ operational excellence รวมกัน
