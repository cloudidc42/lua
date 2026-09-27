# บทที่ 88: Global Scale Architecture

## บทนำ: สถาปัตยกรรมระดับโลก

การสร้างระบบที่รองรับผู้ใช้ทั่วโลกนั้นต้องพิจารณาหลายปัจจัย ตั้งแต่ latency ระหว่างทวีป ไปจนถึง data sovereignty และ compliance requirements

### ความท้าทายของ Global Scale

```lua
-- ตัวอย่างที่ 1: Global Region Configuration
local GlobalConfig = {}
GlobalConfig.__index = GlobalConfig

-- Region definitions with geographic coordinates
local REGIONS = {
    {
        name = "us-east-1",
        location = "N. Virginia",
        lat = 38.0,
        lon = -78.0,
        continent = "NA",
        primary = true
    },
    {
        name = "eu-west-1",
        location = "Ireland",
        lat = 53.0,
        lon = -8.0,
        continent = "EU",
        primary = true
    },
    {
        name = "ap-southeast-1",
        location = "Singapore",
        lat = 1.3,
        lon = 103.8,
        continent = "APAC",
        primary = true
    },
    {
        name = "ap-northeast-1",
        location = "Tokyo",
        lat = 35.7,
        lon = 139.7,
        continent = "APAC",
        primary = false
    },
    {
        name = "sa-east-1",
        location = "São Paulo",
        lat = -23.5,
        lon = -46.6,
        continent = "SA",
        primary = false
    }
}

-- Calculate approximate latency based on distance (simplified model)
local function estimate_latency_ms(lat1, lon1, lat2, lon2)
    -- Haversine formula for distance
    local R = 6371  -- Earth radius in km
    local dlat = math.rad(lat2 - lat1)
    local dlon = math.rad(lon2 - lon1)
    
    local a = math.sin(dlat/2)^2 + 
              math.cos(math.rad(lat1)) * math.cos(math.rad(lat2)) * 
              math.sin(dlon/2)^2
    local c = 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))
    local distance_km = R * c
    
    -- Speed of light in fiber: ~200,000 km/s
    -- Round trip: distance * 2 / speed + overhead
    local speed_km_per_ms = 200  -- km/ms
    local rtt_ms = (distance_km * 2 / speed_km_per_ms) + 10  -- +10ms overhead
    
    return math.floor(rtt_ms)
end

-- Build latency matrix
print("=== Inter-Region Latency Matrix (ms) ===")
io.write(string.format("%-20s", ""))
for _, r in ipairs(REGIONS) do
    io.write(string.format("%-15s", r.name:sub(1, 12)))
end
print()

for _, src in ipairs(REGIONS) do
    io.write(string.format("%-20s", src.name))
    for _, dst in ipairs(REGIONS) do
        if src.name == dst.name then
            io.write(string.format("%-15s", "0"))
        else
            local lat = estimate_latency_ms(src.lat, src.lon, dst.lat, dst.lon)
            io.write(string.format("%-15d", lat))
        end
    end
    print()
end
```

## Global Load Balancer

```lua
-- ตัวอย่างที่ 2: Global Load Balancer Simulation
local GlobalLoadBalancer = {}
GlobalLoadBalancer.__index = GlobalLoadBalancer

function GlobalLoadBalancer.new()
    return setmetatable({
        regions = {},
        routing_policy = "latency",  -- latency, geolocation, weighted
        health_status = {}
    }, GlobalLoadBalancer)
end

function GlobalLoadBalancer:add_region(config)
    config.health = true
    config.weight = config.weight or 100
    config.current_rps = 0
    config.capacity_rps = config.capacity or 10000
    table.insert(self.regions, config)
end

function GlobalLoadBalancer:find_nearest(client_lat, client_lon)
    local best_region = nil
    local best_latency = math.huge
    
    for _, region in ipairs(self.regions) do
        if region.health then
            local latency = estimate_latency_ms(
                client_lat, client_lon,
                region.lat, region.lon
            )
            
            if latency < best_latency then
                best_latency = latency
                best_region = region
            end
        end
    end
    
    return best_region, best_latency
end

function GlobalLoadBalancer:route(request)
    local policy = self.routing_policy
    local target = nil
    local reason = ""
    
    if policy == "latency" then
        target, reason = self:route_by_latency(request)
    elseif policy == "geolocation" then
        target, reason = self:route_by_geolocation(request)
    elseif policy == "weighted" then
        target, reason = self:route_by_weight(request)
    end
    
    if not target then
        -- Fallback to any healthy region
        for _, region in ipairs(self.regions) do
            if region.health then
                target = region
                reason = "fallback"
                break
            end
        end
    end
    
    return target, reason
end

function GlobalLoadBalancer:route_by_latency(request)
    return self:find_nearest(request.client_lat, request.client_lon)
end

function GlobalLoadBalancer:route_by_geolocation(request)
    -- Route based on client's continent/country
    for _, region in ipairs(self.regions) do
        if region.health and region.continent == request.continent then
            return region, "geolocation:" .. request.continent
        end
    end
    return nil
end

function GlobalLoadBalancer:route_by_weight(request)
    local total_weight = 0
    local healthy_regions = {}
    
    for _, region in ipairs(self.regions) do
        if region.health then
            total_weight = total_weight + region.weight
            table.insert(healthy_regions, region)
        end
    end
    
    local rand = math.random(1, total_weight)
    local cumulative = 0
    
    for _, region in ipairs(healthy_regions) do
        cumulative = cumulative + region.weight
        if rand <= cumulative then
            return region, "weighted"
        end
    end
    
    return healthy_regions[#healthy_regions], "weighted_fallback"
end

function GlobalLoadBalancer:mark_unhealthy(region_name)
    for _, region in ipairs(self.regions) do
        if region.name == region_name then
            region.health = false
            print(string.format("[GLB] Region %s marked UNHEALTHY", region_name))
        end
    end
end

function GlobalLoadBalancer:mark_healthy(region_name)
    for _, region in ipairs(self.regions) do
        if region.name == region_name then
            region.health = true
            print(string.format("[GLB] Region %s marked HEALTHY", region_name))
        end
    end
end

-- ตัวอย่างการใช้งาน
local glb = GlobalLoadBalancer.new()
glb.routing_policy = "latency"

for _, r in ipairs(REGIONS) do
    glb:add_region(r)
end

-- Simulate requests from different locations
local client_locations = {
    {name = "New York", lat = 40.7, lon = -74.0},
    {name = "London", lat = 51.5, lon = -0.1},
    {name = "Bangkok", lat = 13.7, lon = 100.5},
    {name = "Sydney", lat = -33.9, lon = 151.2},
    {name = "Mumbai", lat = 19.1, lon = 72.9}
}

print("\n=== Global Load Balancer Routing ===")
for _, client in ipairs(client_locations) do
    local target, latency = glb:route({
        client_lat = client.lat,
        client_lon = client.lon,
        continent = "APAC"
    })
    
    print(string.format("Client: %-15s -> Region: %-20s (~%dms)",
                       client.name, target and target.location or "none", 
                       type(latency) == "number" and latency or 0))
end
```

## GeoDNS Implementation

```lua
-- ตัวอย่างที่ 3: GeoDNS System
local GeoDNS = {}
GeoDNS.__index = GeoDNS

function GeoDNS.new()
    return setmetatable({
        zones = {},
        default_zone = nil,
        ttl = 60  -- seconds
    }, GeoDNS)
end

function GeoDNS:add_zone(name, config)
    self.zones[name] = {
        name = name,
        ip_addresses = config.ips or {},
        countries = config.countries or {},
        continents = config.continents or {},
        asn_ranges = config.asn_ranges or {},
        weight = config.weight or 100,
        health = true,
        ttl = config.ttl or self.ttl
    }
end

function GeoDNS:set_default(zone_name)
    self.default_zone = zone_name
end

function GeoDNS:resolve(domain, client_info)
    -- Try most specific match first
    local zone = self:find_zone_by_country(client_info.country)
               or self:find_zone_by_continent(client_info.continent)
               or self.zones[self.default_zone]
    
    if not zone or not zone.health then
        -- Return any healthy zone
        for _, z in pairs(self.zones) do
            if z.health then
                zone = z
                break
            end
        end
    end
    
    if not zone then
        return nil, "SERVFAIL: No healthy zones"
    end
    
    return {
        domain = domain,
        zone = zone.name,
        ips = zone.ip_addresses,
        ttl = zone.ttl
    }
end

function GeoDNS:find_zone_by_country(country)
    for _, zone in pairs(self.zones) do
        if zone.health then
            for _, c in ipairs(zone.countries) do
                if c == country then return zone end
            end
        end
    end
    return nil
end

function GeoDNS:find_zone_by_continent(continent)
    for _, zone in pairs(self.zones) do
        if zone.health then
            for _, c in ipairs(zone.continents) do
                if c == continent then return zone end
            end
        end
    end
    return nil
end

-- ตัวอย่างการตั้งค่า GeoDNS
local dns = GeoDNS.new()

dns:add_zone("us-east", {
    ips = {"1.2.3.10", "1.2.3.11"},
    continents = {"NA"},
    countries = {"US", "CA", "MX"}
})

dns:add_zone("eu-west", {
    ips = {"2.3.4.10", "2.3.4.11"},
    continents = {"EU"},
    countries = {"GB", "DE", "FR", "TH_EU"}
})

dns:add_zone("apac", {
    ips = {"3.4.5.10", "3.4.5.11"},
    continents = {"APAC"},
    countries = {"TH", "SG", "JP", "KR", "AU"}
})

dns:set_default("us-east")

-- Test DNS resolution
local test_clients = {
    {country = "TH", continent = "APAC"},
    {country = "DE", continent = "EU"},
    {country = "CA", continent = "NA"},
    {country = "BR", continent = "SA"}  -- No specific zone
}

print("\n=== GeoDNS Resolution ===")
for _, client in ipairs(test_clients) do
    local result = dns:resolve("api.example.com", client)
    if result then
        print(string.format("Country: %-5s -> Zone: %-10s IPs: %s",
                           client.country, result.zone,
                           table.concat(result.ips, ", ")))
    end
end
```

## Edge Computing with Cloudflare Workers Pattern

```lua
-- ตัวอย่างที่ 4: Cloudflare Workers Edge Logic Simulation
-- Note: In real CF Workers, this runs in JavaScript/WASM
-- This shows the equivalent logic in Lua

local EdgeWorker = {}
EdgeWorker.__index = EdgeWorker

function EdgeWorker.new(name)
    return setmetatable({
        name = name,
        middlewares = {},
        routes = {},
        kv_store = {},  -- Simulate KV storage
        cache = {}
    }, EdgeWorker)
end

function EdgeWorker:use(middleware)
    table.insert(self.middlewares, middleware)
end

function EdgeWorker:route(method, path, handler)
    table.insert(self.routes, {
        method = method:upper(),
        pattern = path,
        handler = handler
    })
end

function EdgeWorker:handle(request)
    -- Apply middlewares
    local ctx = {
        request = request,
        response = nil,
        worker = self,
        kv = self.kv_store
    }
    
    for _, middleware in ipairs(self.middlewares) do
        local result = middleware(ctx)
        if result then
            return result  -- Middleware short-circuited
        end
    end
    
    -- Match routes
    for _, route in ipairs(self.routes) do
        if route.method == request.method then
            local params = self:match_path(route.pattern, request.path)
            if params then
                request.params = params
                return route.handler(ctx)
            end
        end
    end
    
    return {status = 404, body = "Not Found"}
end

function EdgeWorker:match_path(pattern, path)
    -- Simple pattern matching with :param support
    local param_names = {}
    local regex = pattern:gsub(":([%w_]+)", function(name)
        table.insert(param_names, name)
        return "([^/]+)"
    end)
    regex = "^" .. regex .. "$"
    
    local captures = {path:match(regex)}
    if #captures == 0 and not path:match(regex) then
        return nil
    end
    
    local params = {}
    for i, name in ipairs(param_names) do
        params[name] = captures[i]
    end
    return params
end

-- ตัวอย่าง: Edge Cache with Regional Optimization
local function make_edge_cache_middleware(ttl)
    return function(ctx)
        local cache_key = ctx.request.method .. ":" .. ctx.request.path
        
        -- Check cache
        local cached = ctx.worker.cache[cache_key]
        if cached and (os.time() - cached.created_at) < ttl then
            print(string.format("[EDGE CACHE HIT] %s", cache_key))
            return cached.response
        end
        
        return nil  -- Cache miss, continue to handler
    end
end

-- ตัวอย่าง: Geo-blocking Middleware
local function make_geoblocking_middleware(blocked_countries)
    local blocked_set = {}
    for _, c in ipairs(blocked_countries) do
        blocked_set[c] = true
    end
    
    return function(ctx)
        local country = ctx.request.headers and ctx.request.headers["CF-IPCountry"]
        if country and blocked_set[country] then
            return {
                status = 403,
                body = "Access denied from your region",
                headers = {["X-Blocked-Reason"] = "geo-restriction"}
            }
        end
        return nil
    end
end

-- ตัวอย่าง: Rate Limiting Middleware at Edge
local function make_edge_ratelimit(requests_per_minute)
    local counters = {}
    
    return function(ctx)
        local ip = ctx.request.ip or "unknown"
        local minute_key = ip .. ":" .. math.floor(os.time() / 60)
        
        counters[minute_key] = (counters[minute_key] or 0) + 1
        
        if counters[minute_key] > requests_per_minute then
            return {
                status = 429,
                body = "Too Many Requests",
                headers = {
                    ["Retry-After"] = "60",
                    ["X-RateLimit-Limit"] = tostring(requests_per_minute),
                    ["X-RateLimit-Remaining"] = "0"
                }
            }
        end
        
        return nil
    end
end

-- ตัวอย่าง Worker Definition
local worker = EdgeWorker.new("global-api")

worker:use(make_geoblocking_middleware({"CN", "KP"}))
worker:use(make_edge_ratelimit(60))
worker:use(make_edge_cache_middleware(300))

worker:route("GET", "/api/v1/users/:id", function(ctx)
    return {
        status = 200,
        body = string.format('{"id":"%s","name":"User %s"}', 
                            ctx.request.params.id, ctx.request.params.id),
        headers = {["Content-Type"] = "application/json"}
    }
end)

worker:route("GET", "/health", function(ctx)
    return {status = 200, body = '{"status":"ok"}'}
end)

-- Test the edge worker
local test_requests = {
    {method = "GET", path = "/api/v1/users/123", ip = "1.2.3.4",
     headers = {["CF-IPCountry"] = "TH"}},
    {method = "GET", path = "/api/v1/users/456", ip = "5.6.7.8",
     headers = {["CF-IPCountry"] = "CN"}},  -- blocked
    {method = "GET", path = "/health", ip = "9.10.11.12", headers = {}},
}

print("\n=== Edge Worker Responses ===")
for _, req in ipairs(test_requests) do
    local response = worker:handle(req)
    print(string.format("%-40s -> %d", 
                       req.method .. " " .. req.path,
                       response.status))
end
```

## CDN Edge Logic

```lua
-- ตัวอย่างที่ 5: CDN Cache Control System
local CDNController = {}
CDNController.__index = CDNController

function CDNController.new()
    return setmetatable({
        cache_rules = {},
        purge_queue = {},
        edge_nodes = {}
    }, CDNController)
end

function CDNController:add_cache_rule(pattern, config)
    table.insert(self.cache_rules, {
        pattern = pattern,
        ttl = config.ttl or 86400,
        vary_by = config.vary_by or {},
        cache_control = config.cache_control,
        stale_while_revalidate = config.stale_while_revalidate or 0,
        stale_if_error = config.stale_if_error or 0,
        bypass_conditions = config.bypass or {}
    })
end

function CDNController:get_cache_config(path, request_headers)
    for _, rule in ipairs(self.cache_rules) do
        if path:match(rule.pattern) then
            -- Check bypass conditions
            local should_bypass = false
            for _, condition in ipairs(rule.bypass_conditions) do
                if condition(path, request_headers) then
                    should_bypass = true
                    break
                end
            end
            
            if should_bypass then
                return {ttl = 0, cache_control = "no-cache, no-store"}
            end
            
            -- Build cache key considering vary-by headers
            local cache_key = path
            for _, header in ipairs(rule.vary_by) do
                local value = request_headers[header]
                if value then
                    cache_key = cache_key .. ":" .. header .. "=" .. value
                end
            end
            
            return {
                ttl = rule.ttl,
                cache_control = rule.cache_control or 
                    string.format("public, max-age=%d, stale-while-revalidate=%d",
                                 rule.ttl, rule.stale_while_revalidate),
                cache_key = cache_key,
                stale_while_revalidate = rule.stale_while_revalidate,
                stale_if_error = rule.stale_if_error
            }
        end
    end
    
    return {ttl = 0, cache_control = "no-cache"}
end

function CDNController:purge(pattern, options)
    options = options or {}
    
    local purge_job = {
        id = string.format("purge_%d", os.time()),
        pattern = pattern,
        regions = options.regions or "all",
        timestamp = os.time()
    }
    
    table.insert(self.purge_queue, purge_job)
    print(string.format("[CDN PURGE] Job %s created for pattern: %s", 
                       purge_job.id, pattern))
    
    return purge_job.id
end

-- ตัวอย่างการตั้งค่า CDN Rules
local cdn = CDNController.new()

-- Static assets - long cache
cdn:add_cache_rule("^/static/", {
    ttl = 31536000,  -- 1 year
    cache_control = "public, max-age=31536000, immutable"
})

-- API responses - short cache
cdn:add_cache_rule("^/api/v1/products", {
    ttl = 300,  -- 5 minutes
    stale_while_revalidate = 60,
    stale_if_error = 86400,
    vary_by = {"Accept-Language", "Accept-Currency"}
})

-- User-specific API - no cache
cdn:add_cache_rule("^/api/v1/user", {
    ttl = 0,
    cache_control = "private, no-cache",
    bypass = {
        function(path, headers)
            return headers["Authorization"] ~= nil
        end
    }
})

-- Images with automatic format selection
cdn:add_cache_rule("^/images/", {
    ttl = 604800,  -- 1 week
    vary_by = {"Accept"}  -- For WebP/AVIF negotiation
})

-- Test cache configuration
local test_paths = {
    {path = "/static/app.js", headers = {}},
    {path = "/api/v1/products/123", headers = {["Accept-Language"] = "th"}},
    {path = "/api/v1/user/profile", headers = {["Authorization"] = "Bearer token"}},
    {path = "/images/hero.jpg", headers = {["Accept"] = "image/webp"}}
}

print("\n=== CDN Cache Configuration ===")
for _, test in ipairs(test_paths) do
    local config = cdn:get_cache_config(test.path, test.headers)
    print(string.format("%-40s TTL: %6ds | %s",
                       test.path, config.ttl, 
                       (config.cache_control or ""):sub(1, 50)))
end
```

## Data Locality and Replication

```lua
-- ตัวอย่างที่ 6: Global Database Replication Manager
local ReplicationManager = {}
ReplicationManager.__index = ReplicationManager

function ReplicationManager.new(options)
    options = options or {}
    return setmetatable({
        primary_region = options.primary or "us-east-1",
        replicas = {},
        replication_lag = {},
        consistency_model = options.consistency or "eventual"
    }, ReplicationManager)
end

function ReplicationManager:add_replica(region, options)
    options = options or {}
    self.replicas[region] = {
        region = region,
        lag_ms = 0,
        status = "syncing",
        read_enabled = options.read_enabled ~= false,
        write_enabled = options.write_enabled or false,
        priority = options.priority or 1
    }
end

function ReplicationManager:simulate_replication(data_size_kb)
    -- Simulate replication lag based on data size and network
    for region, replica in pairs(self.replicas) do
        local base_latency = estimate_latency_ms(
            REGIONS[1].lat, REGIONS[1].lon,
            -- Find region lat/lon
            self:get_region_coords(region)
        )
        
        -- Add throughput-based delay
        local throughput_delay = data_size_kb / 10  -- 10KB/ms throughput
        local lag = base_latency + throughput_delay
        
        replica.lag_ms = lag
        replica.status = lag > 100 and "lagging" or "synced"
    end
end

function ReplicationManager:get_region_coords(region_name)
    for _, r in ipairs(REGIONS) do
        if r.name == region_name then
            return r.lat, r.lon
        end
    end
    return 0, 0
end

function ReplicationManager:route_read(request_region, consistency_level)
    consistency_level = consistency_level or self.consistency_model
    
    if consistency_level == "strong" then
        -- Always read from primary
        return self.primary_region, "strong_consistency"
    end
    
    -- Find nearest replica with acceptable lag
    local max_lag = consistency_level == "bounded_staleness" and 500 or math.huge
    
    -- Check if local replica is available and fresh enough
    local local_replica = self.replicas[request_region]
    if local_replica and local_replica.read_enabled and 
       local_replica.status ~= "down" and
       local_replica.lag_ms <= max_lag then
        return request_region, "local_replica"
    end
    
    -- Find best available replica
    local best_region = self.primary_region
    local best_score = math.huge
    
    for region, replica in pairs(self.replicas) do
        if replica.read_enabled and replica.status ~= "down" and
           replica.lag_ms <= max_lag then
            local latency = estimate_latency_ms(
                0, 0,  -- simplified
                self:get_region_coords(region)
            )
            local score = latency + replica.lag_ms
            if score < best_score then
                best_score = score
                best_region = region
            end
        end
    end
    
    return best_region, "nearest_replica"
end

-- ตัวอย่างที่ 7: Multi-Region Active-Active Configuration
local ActiveActiveCluster = {}
ActiveActiveCluster.__index = ActiveActiveCluster

function ActiveActiveCluster.new()
    return setmetatable({
        nodes = {},
        conflict_resolver = nil,
        vector_clocks = {}
    }, ActiveActiveCluster)
end

function ActiveActiveCluster:add_node(region, options)
    self.nodes[region] = {
        region = region,
        data = {},
        vector_clock = {},
        pending_writes = {},
        replicated_writes = {}
    }
end

-- Vector clock for conflict detection
function ActiveActiveCluster:increment_clock(region)
    local node = self.nodes[region]
    if not node then error("Unknown region: " .. region) end
    
    node.vector_clock[region] = (node.vector_clock[region] or 0) + 1
    return node.vector_clock
end

function ActiveActiveCluster:compare_clocks(clock1, clock2)
    local c1_wins_any = false
    local c2_wins_any = false
    
    -- Check all known regions
    local all_regions = {}
    for r in pairs(clock1) do all_regions[r] = true end
    for r in pairs(clock2) do all_regions[r] = true end
    
    for region in pairs(all_regions) do
        local v1 = clock1[region] or 0
        local v2 = clock2[region] or 0
        
        if v1 > v2 then c1_wins_any = true end
        if v2 > v1 then c2_wins_any = true end
    end
    
    if c1_wins_any and not c2_wins_any then
        return "c1_dominates"
    elseif c2_wins_any and not c1_wins_any then
        return "c2_dominates"
    elseif not c1_wins_any and not c2_wins_any then
        return "equal"
    else
        return "concurrent"  -- Conflict!
    end
end

function ActiveActiveCluster:write(region, key, value)
    local node = self.nodes[region]
    if not node then error("Unknown region") end
    
    local clock = self:increment_clock(region)
    
    local entry = {
        key = key,
        value = value,
        clock = {table.unpack(clock)},  -- clone
        region = region,
        timestamp = os.time()
    }
    
    -- Store locally
    if not node.data[key] then
        node.data[key] = entry
    else
        -- Resolve conflict
        local result = self:compare_clocks(node.data[key].clock, entry.clock)
        if result == "c2_dominates" then
            node.data[key] = entry
        elseif result == "concurrent" then
            -- Apply conflict resolution strategy
            if self.conflict_resolver then
                node.data[key] = self.conflict_resolver(node.data[key], entry)
            else
                -- Last-write-wins by timestamp
                if entry.timestamp > node.data[key].timestamp then
                    node.data[key] = entry
                end
            end
        end
    end
    
    -- Queue for replication to other nodes
    table.insert(node.pending_writes, entry)
    
    return entry
end

function ActiveActiveCluster:replicate_to(source_region, target_region)
    local source = self.nodes[source_region]
    local target = self.nodes[target_region]
    
    if not source or not target then return end
    
    for _, write in ipairs(source.pending_writes) do
        -- Apply to target
        target.replicated_writes[write.key] = write
        
        -- Merge vector clocks
        for region, value in pairs(write.clock) do
            target.vector_clock[region] = math.max(
                target.vector_clock[region] or 0,
                value
            )
        end
    end
    
    source.pending_writes = {}  -- Clear after replication
end
```

## Global Rate Limiting

```lua
-- ตัวอย่างที่ 8: Distributed Rate Limiter
local DistributedRateLimiter = {}
DistributedRateLimiter.__index = DistributedRateLimiter

function DistributedRateLimiter.new(options)
    options = options or {}
    return setmetatable({
        global_limit = options.global_limit or 10000,  -- per minute
        per_region_limit = options.per_region_limit or 3000,
        per_user_limit = options.per_user_limit or 60,
        counters = {
            global = {},
            region = {},
            user = {}
        },
        sync_interval = options.sync_interval or 1000  -- ms
    }, DistributedRateLimiter)
end

function DistributedRateLimiter:get_window_key(granularity)
    -- Current minute key
    return tostring(math.floor(os.time() / 60))
end

function DistributedRateLimiter:check_and_increment(identifier, counter_type, limit)
    local window = self:get_window_key(60)
    local key = identifier .. ":" .. window
    
    if not self.counters[counter_type] then
        self.counters[counter_type] = {}
    end
    
    self.counters[counter_type][key] = (self.counters[counter_type][key] or 0) + 1
    local current = self.counters[counter_type][key]
    
    return current <= limit, current, limit
end

function DistributedRateLimiter:check(request)
    local results = {}
    
    -- Check global rate
    local global_ok, global_count, global_limit = 
        self:check_and_increment("global", "global", self.global_limit)
    results.global = {ok = global_ok, count = global_count, limit = global_limit}
    
    -- Check per-region rate
    if request.region then
        local region_ok, region_count, region_limit = 
            self:check_and_increment(request.region, "region", self.per_region_limit)
        results.region = {ok = region_ok, count = region_count, limit = region_limit}
    end
    
    -- Check per-user rate
    if request.user_id then
        local user_ok, user_count, user_limit = 
            self:check_and_increment(request.user_id, "user", self.per_user_limit)
        results.user = {ok = user_ok, count = user_count, limit = user_limit}
    end
    
    -- All checks must pass
    local allowed = global_ok and 
                   (not results.region or results.region.ok) and
                   (not results.user or results.user.ok)
    
    return allowed, results
end

function DistributedRateLimiter:get_headers(results)
    local headers = {}
    
    if results.user then
        headers["X-RateLimit-Limit"] = tostring(results.user.limit)
        headers["X-RateLimit-Remaining"] = tostring(
            math.max(0, results.user.limit - results.user.count))
        headers["X-RateLimit-Reset"] = tostring(
            (math.floor(os.time() / 60) + 1) * 60)
    end
    
    return headers
end

-- ตัวอย่างที่ 9: Geo-Based Rate Limiting
local GeoRateLimiter = {}
GeoRateLimiter.__index = GeoRateLimiter

function GeoRateLimiter.new()
    return setmetatable({
        rules = {},
        default_limit = 60
    }, GeoRateLimiter)
end

function GeoRateLimiter:add_rule(name, config)
    table.insert(self.rules, {
        name = name,
        countries = config.countries,
        continents = config.continents,
        limit = config.limit,
        window = config.window or 60,
        priority = config.priority or 0,
        action = config.action or "limit"  -- "limit", "block", "allow"
    })
    
    -- Sort by priority (higher = checked first)
    table.sort(self.rules, function(a, b)
        return a.priority > b.priority
    end)
end

function GeoRateLimiter:get_limit(country, continent)
    for _, rule in ipairs(self.rules) do
        local matches = false
        
        if rule.countries then
            for _, c in ipairs(rule.countries) do
                if c == country then matches = true; break end
            end
        end
        
        if not matches and rule.continents then
            for _, c in ipairs(rule.continents) do
                if c == continent then matches = true; break end
            end
        end
        
        if matches then
            if rule.action == "block" then
                return 0, rule.name
            elseif rule.action == "allow" then
                return math.huge, rule.name
            else
                return rule.limit, rule.name
            end
        end
    end
    
    return self.default_limit, "default"
end

-- ตัวอย่างการใช้งาน
local geo_rl = GeoRateLimiter.new()

-- Bot farms - strict limits
geo_rl:add_rule("known_bot_sources", {
    countries = {"XX"},  -- fictional
    limit = 10,
    priority = 100
})

-- Premium regions - higher limits
geo_rl:add_rule("premium_regions", {
    countries = {"US", "GB", "DE", "JP"},
    limit = 300,
    priority = 50
})

-- Emerging markets - medium limits
geo_rl:add_rule("emerging_markets", {
    continents = {"APAC", "SA", "AF"},
    limit = 60,
    priority = 10
})

local test_geos = {
    {country = "TH", continent = "APAC"},
    {country = "US", continent = "NA"},
    {country = "XX", continent = "EU"},
}

print("\n=== Geo Rate Limits ===")
for _, geo in ipairs(test_geos) do
    local limit, rule = geo_rl:get_limit(geo.country, geo.continent)
    print(string.format("Country: %-5s -> %4d req/min (rule: %s)",
                       geo.country, limit, rule))
end
```

## Time Zone Handling

```lua
-- ตัวอย่างที่ 10: Global Time Zone Handler
local TimeZoneHandler = {}
TimeZoneHandler.__index = TimeZoneHandler

-- IANA timezone offsets (simplified - UTC offsets in hours)
local TIMEZONE_DATA = {
    ["America/New_York"] = {offset = -5, dst_offset = -4, name = "Eastern"},
    ["America/Los_Angeles"] = {offset = -8, dst_offset = -7, name = "Pacific"},
    ["Europe/London"] = {offset = 0, dst_offset = 1, name = "GMT/BST"},
    ["Europe/Paris"] = {offset = 1, dst_offset = 2, name = "CET/CEST"},
    ["Asia/Bangkok"] = {offset = 7, dst_offset = 7, name = "ICT"},
    ["Asia/Tokyo"] = {offset = 9, dst_offset = 9, name = "JST"},
    ["Asia/Singapore"] = {offset = 8, dst_offset = 8, name = "SGT"},
    ["Australia/Sydney"] = {offset = 10, dst_offset = 11, name = "AEST/AEDT"},
    ["UTC"] = {offset = 0, dst_offset = 0, name = "UTC"}
}

-- Country to timezone mapping (simplified)
local COUNTRY_TIMEZONE = {
    TH = "Asia/Bangkok",
    US = "America/New_York",
    GB = "Europe/London",
    DE = "Europe/Paris",
    JP = "Asia/Tokyo",
    SG = "Asia/Singapore",
    AU = "Australia/Sydney"
}

function TimeZoneHandler.new()
    return setmetatable({
        timezones = TIMEZONE_DATA
    }, TimeZoneHandler)
end

function TimeZoneHandler:is_dst(timezone, timestamp)
    -- Simplified DST check (March-November for Northern Hemisphere)
    local tz = self.timezones[timezone]
    if not tz then return false end
    
    if tz.offset == tz.dst_offset then return false end  -- No DST
    
    local date = os.date("*t", timestamp)
    -- Very simplified: DST from March to November
    return date.month >= 3 and date.month <= 11
end

function TimeZoneHandler:get_offset(timezone, timestamp)
    local tz = self.timezones[timezone]
    if not tz then return 0 end
    
    if self:is_dst(timezone, timestamp) then
        return tz.dst_offset
    end
    return tz.offset
end

function TimeZoneHandler:convert(timestamp, from_tz, to_tz)
    -- Convert UTC offset
    local from_offset = self:get_offset(from_tz, timestamp) * 3600
    local to_offset = self:get_offset(to_tz, timestamp) * 3600
    
    local utc_time = timestamp - from_offset
    return utc_time + to_offset
end

function TimeZoneHandler:format(timestamp, timezone, format_str)
    format_str = format_str or "%Y-%m-%d %H:%M:%S"
    local offset_hours = self:get_offset(timezone, timestamp)
    local local_time = timestamp + offset_hours * 3600
    return os.date(format_str, local_time) .. " " .. timezone
end

function TimeZoneHandler:get_business_hours(timezone)
    local tz = self.timezones[timezone]
    return {
        start_hour = 9,   -- 9 AM
        end_hour = 17,    -- 5 PM
        timezone = timezone,
        offset = tz and (tz.offset) or 0
    }
end

function TimeZoneHandler:is_business_hours(timestamp, timezone)
    local offset = self:get_offset(timezone, timestamp) * 3600
    local local_time = os.date("*t", timestamp + offset)
    local hour = local_time.hour
    local wday = local_time.wday  -- 1=Sunday, 7=Saturday
    
    -- Monday-Friday 9AM-5PM
    return wday >= 2 and wday <= 6 and hour >= 9 and hour < 17
end

-- ตัวอย่างการใช้งาน
local tzh = TimeZoneHandler.new()
local now = os.time()

print("\n=== Time Zone Display ===")
local zones = {"Asia/Bangkok", "Europe/London", "America/New_York", "Asia/Tokyo"}
for _, tz in ipairs(zones) do
    local formatted = tzh:format(now, tz, "%Y-%m-%d %H:%M")
    local is_biz = tzh:is_business_hours(now, tz)
    print(string.format("%-25s %s (business hours: %s)",
                       tz, formatted, is_biz and "yes" or "no"))
end
```

## Currency and Pricing by Region

```lua
-- ตัวอย่างที่ 11: Global Pricing Engine
local PricingEngine = {}
PricingEngine.__index = PricingEngine

function PricingEngine.new()
    return setmetatable({
        base_prices = {},
        currencies = {},
        regional_adjustments = {},
        tax_rates = {}
    }, PricingEngine)
end

function PricingEngine:set_base_price(product_id, price_usd)
    self.base_prices[product_id] = price_usd
end

function PricingEngine:add_currency(code, rate_to_usd, symbol, format)
    self.currencies[code] = {
        code = code,
        rate = rate_to_usd,  -- 1 USD = rate units of this currency
        symbol = symbol,
        format = format or "%s %.2f"  -- symbol + amount
    }
end

function PricingEngine:add_regional_adjustment(country, multiplier, currency)
    self.regional_adjustments[country] = {
        multiplier = multiplier or 1.0,
        currency = currency
    }
end

function PricingEngine:add_tax(country, rate, name)
    self.tax_rates[country] = {
        rate = rate,
        name = name or "VAT"
    }
end

function PricingEngine:get_price(product_id, country, include_tax)
    local base_usd = self.base_prices[product_id]
    if not base_usd then
        return nil, "Product not found"
    end
    
    -- Get regional adjustment
    local adjustment = self.regional_adjustments[country] or {multiplier = 1.0}
    local currency_code = adjustment.currency or "USD"
    local currency = self.currencies[currency_code]
    
    if not currency then
        currency = self.currencies["USD"]
        currency_code = "USD"
    end
    
    -- Calculate price
    local price_usd = base_usd * adjustment.multiplier
    local price_local = price_usd * currency.rate
    
    -- Round to appropriate decimal places
    price_local = math.floor(price_local * 100 + 0.5) / 100
    
    -- Apply tax if requested
    local tax_amount = 0
    if include_tax and self.tax_rates[country] then
        local tax = self.tax_rates[country]
        tax_amount = price_local * tax.rate
        price_local = price_local + tax_amount
        price_local = math.floor(price_local * 100 + 0.5) / 100
    end
    
    return {
        product_id = product_id,
        country = country,
        currency = currency_code,
        price_usd = base_usd,
        price_local = price_local,
        tax_amount = math.floor(tax_amount * 100 + 0.5) / 100,
        tax_rate = (self.tax_rates[country] or {}).rate or 0,
        formatted = string.format("%s %.2f", currency.symbol, price_local)
    }
end

-- ตัวอย่างการตั้งค่า Pricing
local pricing = PricingEngine.new()

-- Base prices in USD
pricing:set_base_price("PRO_MONTHLY", 29.99)
pricing:set_base_price("ENTERPRISE_MONTHLY", 299.99)

-- Currencies
pricing:add_currency("USD", 1.0, "$")
pricing:add_currency("EUR", 0.92, "€")
pricing:add_currency("GBP", 0.79, "£")
pricing:add_currency("THB", 35.5, "฿")
pricing:add_currency("JPY", 149.0, "¥", "%s %.0f")
pricing:add_currency("SGD", 1.35, "S$")

-- Regional adjustments
pricing:add_regional_adjustment("US", 1.0, "USD")
pricing:add_regional_adjustment("GB", 0.85, "GBP")
pricing:add_regional_adjustment("DE", 0.90, "EUR")
pricing:add_regional_adjustment("TH", 0.70, "THB")  -- Lower for Thailand
pricing:add_regional_adjustment("JP", 1.10, "JPY")
pricing:add_regional_adjustment("SG", 1.0, "SGD")

-- Tax rates
pricing:add_tax("GB", 0.20, "VAT")
pricing:add_tax("DE", 0.19, "MwSt")
pricing:add_tax("TH", 0.07, "VAT")
pricing:add_tax("SG", 0.09, "GST")
pricing:add_tax("JP", 0.10, "消費税")

-- Display pricing table
print("\n=== Global Pricing Table ===")
print(string.format("%-10s %-10s %-12s %-12s %-15s",
                   "Country", "Currency", "Price (excl)", "Tax", "Price (incl)"))
print(string.rep("-", 65))

for _, country in ipairs({"US", "GB", "DE", "TH", "JP", "SG"}) do
    local excl = pricing:get_price("PRO_MONTHLY", country, false)
    local incl = pricing:get_price("PRO_MONTHLY", country, true)
    
    if excl then
        print(string.format("%-10s %-10s %-12s %-12s %-15s",
                           country,
                           excl.currency,
                           excl.formatted,
                           string.format("%.2f%%", (excl.tax_rate or 0) * 100),
                           incl and incl.formatted or "N/A"))
    end
end
```

## Language and Locale Routing

```lua
-- ตัวอย่างที่ 12: Locale Router
local LocaleRouter = {}
LocaleRouter.__index = LocaleRouter

function LocaleRouter.new()
    return setmetatable({
        supported_locales = {},
        country_locale_map = {},
        fallback_locale = "en-US"
    }, LocaleRouter)
end

function LocaleRouter:add_locale(code, config)
    self.supported_locales[code] = {
        code = code,
        language = config.language,
        country = config.country,
        direction = config.direction or "ltr",
        date_format = config.date_format or "YYYY-MM-DD",
        number_format = config.number_format or {decimal = ".", thousands = ","},
        currency_display = config.currency_display or "code"
    }
end

function LocaleRouter:map_country_to_locale(country, locale)
    if not self.country_locale_map[country] then
        self.country_locale_map[country] = {}
    end
    table.insert(self.country_locale_map[country], locale)
end

function LocaleRouter:negotiate(request_locales, country)
    -- Try to match request's Accept-Language with supported locales
    for _, requested in ipairs(request_locales) do
        -- Exact match
        if self.supported_locales[requested] then
            return self.supported_locales[requested], "exact_match"
        end
        
        -- Language-only match (e.g., "th" matches "th-TH")
        local lang = requested:match("^([a-z]+)") or requested
        for code, locale in pairs(self.supported_locales) do
            if locale.language == lang then
                return locale, "language_match"
            end
        end
    end
    
    -- Country-based fallback
    if country and self.country_locale_map[country] then
        local locale_code = self.country_locale_map[country][1]
        if self.supported_locales[locale_code] then
            return self.supported_locales[locale_code], "country_match"
        end
    end
    
    -- Final fallback
    return self.supported_locales[self.fallback_locale], "fallback"
end

function LocaleRouter:format_date(timestamp, locale_code)
    local locale = self.supported_locales[locale_code]
    if not locale then return os.date("%Y-%m-%d", timestamp) end
    
    local d = os.date("*t", timestamp)
    local fmt = locale.date_format
    
    return fmt
        :gsub("YYYY", string.format("%04d", d.year))
        :gsub("MM", string.format("%02d", d.month))
        :gsub("DD", string.format("%02d", d.day))
end

function LocaleRouter:format_number(number, locale_code)
    local locale = self.supported_locales[locale_code]
    if not locale then return tostring(number) end
    
    local fmt = locale.number_format
    local int_part = math.floor(math.abs(number))
    local dec_part = math.abs(number) - int_part
    
    -- Add thousands separators
    local result = tostring(int_part)
    local formatted = result:reverse():gsub("(%d%d%d)", "%1" .. fmt.thousands):reverse()
    if formatted:sub(1, 1) == fmt.thousands then
        formatted = formatted:sub(2)
    end
    
    -- Add decimal
    if dec_part > 0 then
        formatted = formatted .. fmt.decimal .. string.format("%.2f", dec_part):sub(3)
    end
    
    return (number < 0 and "-" or "") .. formatted
end

-- ตัวอย่างการตั้งค่า
local router = LocaleRouter.new()

router:add_locale("en-US", {
    language = "en", country = "US",
    date_format = "MM/DD/YYYY",
    number_format = {decimal = ".", thousands = ","}
})

router:add_locale("th-TH", {
    language = "th", country = "TH",
    date_format = "DD/MM/YYYY",
    number_format = {decimal = ".", thousands = ","}
})

router:add_locale("de-DE", {
    language = "de", country = "DE",
    date_format = "DD.MM.YYYY",
    number_format = {decimal = ",", thousands = "."}
})

router:add_locale("ja-JP", {
    language = "ja", country = "JP",
    date_format = "YYYY年MM月DD日",
    number_format = {decimal = ".", thousands = ","}
})

router:map_country_to_locale("TH", "th-TH")
router:map_country_to_locale("DE", "de-DE")
router:map_country_to_locale("JP", "ja-JP")
router:map_country_to_locale("US", "en-US")

-- Test locale negotiation
local test_requests = {
    {accept_language = {"th", "en"}, country = "TH"},
    {accept_language = {"de-DE", "de"}, country = "DE"},
    {accept_language = {"fr"}, country = "JP"},  -- French not supported
    {accept_language = {"ko"}, country = "KR"},   -- Neither supported
}

print("\n=== Locale Negotiation ===")
for _, req in ipairs(test_requests) do
    local locale, method = router:negotiate(req.accept_language, req.country)
    local date = router:format_date(os.time(), locale.code)
    print(string.format("Accept: %-20s -> %-8s (%s) | Date: %s",
                       table.concat(req.accept_language, ","),
                       locale.code, method, date))
end
```

## Cross-Region Consistency

```lua
-- ตัวอย่างที่ 13: Consistency Models Implementation
local ConsistencyModel = {}
ConsistencyModel.__index = ConsistencyModel

-- CRDT: Grow-only Counter
local GCounter = {}
GCounter.__index = GCounter

function GCounter.new(node_id)
    return setmetatable({
        node_id = node_id,
        counters = {}
    }, GCounter)
end

function GCounter:increment(amount)
    amount = amount or 1
    self.counters[self.node_id] = (self.counters[self.node_id] or 0) + amount
end

function GCounter:value()
    local total = 0
    for _, count in pairs(self.counters) do
        total = total + count
    end
    return total
end

function GCounter:merge(other)
    -- Take max of each counter
    for node_id, count in pairs(other.counters) do
        self.counters[node_id] = math.max(
            self.counters[node_id] or 0,
            count
        )
    end
end

-- CRDT: Last-Write-Wins Register
local LWWRegister = {}
LWWRegister.__index = LWWRegister

function LWWRegister.new(node_id)
    return setmetatable({
        node_id = node_id,
        value = nil,
        timestamp = 0
    }, LWWRegister)
end

function LWWRegister:write(value)
    local ts = os.time() * 1000 + math.random(0, 999)  -- millisecond precision
    self.value = value
    self.timestamp = ts
    return ts
end

function LWWRegister:read()
    return self.value
end

function LWWRegister:merge(other)
    if other.timestamp > self.timestamp then
        self.value = other.value
        self.timestamp = other.timestamp
    end
end

-- ตัวอย่างที่ 14: Multi-Region Session Management
local GlobalSessionStore = {}
GlobalSessionStore.__index = GlobalSessionStore

function GlobalSessionStore.new(options)
    options = options or {}
    return setmetatable({
        sessions = {},
        primary_region = options.primary or "us-east-1",
        replication_regions = options.replicas or {},
        session_ttl = options.ttl or 3600,
        consistency = options.consistency or "eventual"
    }, GlobalSessionStore)
end

function GlobalSessionStore:create_session(user_id, data)
    local session_id = string.format("sess_%d_%d", os.time(), math.random(100000, 999999))
    
    local session = {
        id = session_id,
        user_id = user_id,
        data = data or {},
        created_at = os.time(),
        updated_at = os.time(),
        expires_at = os.time() + self.session_ttl,
        region = self.primary_region
    }
    
    self.sessions[session_id] = session
    
    -- Replicate to other regions
    self:replicate_session(session)
    
    return session_id
end

function GlobalSessionStore:get_session(session_id, requesting_region)
    local session = self.sessions[session_id]
    
    if not session then
        return nil, "SESSION_NOT_FOUND"
    end
    
    if os.time() > session.expires_at then
        self:invalidate_session(session_id)
        return nil, "SESSION_EXPIRED"
    end
    
    -- For strong consistency, verify against primary
    if self.consistency == "strong" and requesting_region ~= self.primary_region then
        -- In real implementation, this would contact primary region
        print(string.format("[SESSION] Cross-region read: %s from %s",
                           session_id:sub(1, 20), requesting_region))
    end
    
    return session
end

function GlobalSessionStore:update_session(session_id, updates)
    local session = self.sessions[session_id]
    if not session then return false end
    
    for k, v in pairs(updates) do
        session.data[k] = v
    end
    session.updated_at = os.time()
    
    self:replicate_session(session)
    return true
end

function GlobalSessionStore:invalidate_session(session_id)
    self.sessions[session_id] = nil
    
    -- Notify all regions to invalidate
    for _, region in ipairs(self.replication_regions) do
        print(string.format("[SESSION] Invalidating %s in %s", 
                           session_id:sub(1, 20), region))
    end
end

function GlobalSessionStore:replicate_session(session)
    for _, region in ipairs(self.replication_regions) do
        -- In real implementation, this would be async replication
        print(string.format("[SESSION] Replicating to %s", region))
    end
end

-- ตัวอย่างการใช้งาน
local session_store = GlobalSessionStore.new({
    primary = "us-east-1",
    replicas = {"eu-west-1", "ap-southeast-1"},
    ttl = 86400
})

local session_id = session_store:create_session("user_123", {
    cart_items = 3,
    last_page = "/products"
})
print("Created session:", session_id)

local session = session_store:get_session(session_id, "ap-southeast-1")
if session then
    print("Session data:", session.data.last_page)
end
```

## Real Architecture: Global API with Lua

```lua
-- ตัวอย่างที่ 15: Complete Global API Architecture
local GlobalAPI = {}
GlobalAPI.__index = GlobalAPI

function GlobalAPI.new(config)
    return setmetatable({
        config = config,
        
        -- Core components
        load_balancer = GlobalLoadBalancer.new(),
        geo_dns = GeoDNS.new(),
        rate_limiter = DistributedRateLimiter.new(config.rate_limits),
        feature_flags = {},
        
        -- Data layer
        session_store = GlobalSessionStore.new(config.sessions),
        cache = {},
        
        -- Observability
        metrics = {
            requests_total = 0,
            requests_by_region = {},
            latency_histogram = {},
            errors_total = 0
        },
        
        -- Request handlers
        handlers = {}
    }, GlobalAPI)
end

function GlobalAPI:register_handler(path, method, fn)
    local key = method:upper() .. ":" .. path
    self.handlers[key] = fn
end

function GlobalAPI:handle_request(request)
    local start_time = os.clock()
    
    -- Update metrics
    self.metrics.requests_total = self.metrics.requests_total + 1
    self.metrics.requests_by_region[request.region or "unknown"] = 
        (self.metrics.requests_by_region[request.region or "unknown"] or 0) + 1
    
    -- Rate limiting
    local allowed, rate_results = self.rate_limiter:check(request)
    if not allowed then
        return self:error_response(429, "Too Many Requests", 
                                   self.rate_limiter:get_headers(rate_results))
    end
    
    -- Route to handler
    local key = (request.method or "GET"):upper() .. ":" .. (request.path or "/")
    local handler = self.handlers[key]
    
    local response
    if handler then
        local ok, result = pcall(handler, request, self)
        if ok then
            response = result
        else
            self.metrics.errors_total = self.metrics.errors_total + 1
            response = self:error_response(500, "Internal Server Error")
        end
    else
        response = self:error_response(404, "Not Found")
    end
    
    -- Record latency
    local latency = (os.clock() - start_time) * 1000
    table.insert(self.metrics.latency_histogram, latency)
    
    -- Add standard headers
    response.headers = response.headers or {}
    response.headers["X-Region"] = request.region or "unknown"
    response.headers["X-Response-Time"] = string.format("%.2fms", latency)
    
    return response
end

function GlobalAPI:error_response(status, message, headers)
    return {
        status = status,
        body = string.format('{"error":"%s","status":%d}', message, status),
        headers = headers or {["Content-Type"] = "application/json"}
    }
end

function GlobalAPI:health_summary()
    local total = self.metrics.requests_total
    local errors = self.metrics.errors_total
    local success_rate = total > 0 and ((total - errors) / total) or 1
    
    local sum = 0
    local count = #self.metrics.latency_histogram
    for _, l in ipairs(self.metrics.latency_histogram) do sum = sum + l end
    local avg_latency = count > 0 and (sum / count) or 0
    
    return {
        status = success_rate > 0.99 and "healthy" or "degraded",
        requests_total = total,
        success_rate = success_rate,
        avg_latency_ms = avg_latency,
        errors_total = errors,
        requests_by_region = self.metrics.requests_by_region
    }
end

-- Instantiate the global API
local api = GlobalAPI.new({
    rate_limits = {
        global_limit = 10000,
        per_user_limit = 60
    },
    sessions = {
        primary = "us-east-1",
        replicas = {"eu-west-1", "ap-southeast-1"}
    }
})

-- Register handlers
api:register_handler("/api/v1/products", "GET", function(req, app)
    -- Locale-aware response
    local locale = req.locale or "en-US"
    local products = {
        {id = 1, name = "Product A"},
        {id = 2, name = "Product B"},
    }
    
    return {
        status = 200,
        body = string.format('{"products":%d,"locale":"%s"}', #products, locale),
        headers = {
            ["Content-Type"] = "application/json",
            ["Cache-Control"] = "public, max-age=300"
        }
    }
end)

api:register_handler("/api/v1/users/:id", "GET", function(req, app)
    return {
        status = 200,
        body = '{"id":"' .. (req.params and req.params.id or "unknown") .. '"}',
        headers = {["Content-Type"] = "application/json"}
    }
end)

-- Simulate global traffic
print("\n=== Global API Traffic Simulation ===")
math.randomseed(42)

local regions = {"us-east-1", "eu-west-1", "ap-southeast-1"}
local paths = {"/api/v1/products", "/api/v1/users/123", "/api/v1/unknown"}

for i = 1, 20 do
    local region = regions[math.random(1, #regions)]
    local path = paths[math.random(1, #paths)]
    
    local request = {
        method = "GET",
        path = path,
        region = region,
        user_id = "user_" .. math.random(1, 10),
        ip = string.format("%d.%d.%d.%d", 
                          math.random(1,255), math.random(1,255),
                          math.random(1,255), math.random(1,255))
    }
    
    local response = api:handle_request(request)
    print(string.format("[%s] %-40s -> %d", region, path, response.status))
end

-- Print health summary
local health = api:health_summary()
print(string.format("\n=== API Health Summary ==="))
print(string.format("Status: %s", health.status))
print(string.format("Total requests: %d", health.requests_total))
print(string.format("Success rate: %.1f%%", health.success_rate * 100))
print(string.format("Avg latency: %.2fms", health.avg_latency_ms))
print("Requests by region:")
for region, count in pairs(health.requests_by_region) do
    print(string.format("  %s: %d", region, count))
end
```

## สรุปบทที่ 88

ในบทนี้เราได้เรียนรู้ Global Scale Architecture ครอบคลุม:

1. **Global Load Balancing** - Latency-based, geolocation, weighted routing
2. **GeoDNS** - Geographic DNS routing
3. **Edge Computing** - Cloudflare Workers pattern ใน Lua
4. **CDN Edge Logic** - Cache control และ purging
5. **Data Locality** - Global database replication
6. **Active-Active Multi-Region** - CRDT-based conflict resolution
7. **Global Rate Limiting** - Distributed rate limiting
8. **Geo-Based Rate Limiting** - Country/continent specific limits
9. **Time Zone Handling** - Global time zone management
10. **Currency/Pricing** - Regional pricing with tax support
11. **Locale Routing** - Language and date format negotiation
12. **Cross-Region Consistency** - CRDT data structures
13. **Global Session Management** - Multi-region sessions
14. **Complete Global API** - Full architecture implementation

Key takeaways:
- Latency มีผลกระทบสูงในระบบ global scale ต้องออกแบบเพื่อลด round trips
- CRDT ช่วยให้ active-active replication ทำงานได้โดยไม่มี conflicts
- Edge computing ช่วยลด latency ได้มากโดยการ process request ใกล้ user
- Consistency tradeoffs: strong consistency เพิ่ม latency แต่ eventual consistency ง่ายกว่า
- Data locality compliance (GDPR, PDPA) ต้องพิจารณาในการออกแบบ region architecture
