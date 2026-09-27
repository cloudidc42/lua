# บทที่ 69: Docker + Lua Applications

## บทนำ

Docker ช่วยให้เราสามารถ package แอปพลิเคชันพร้อม dependencies ทั้งหมดลงใน container ที่ทำงานได้เหมือนกันทุก environment ในบทนี้เราจะเรียนรู้การใช้ Docker กับ Lua applications และ OpenResty

---

## 69.1 Docker Basics สำหรับ Lua Developers

```lua
-- ตัวอย่างที่ 1: Docker concept simulator ใน Lua
-- (แสดงให้เห็น concepts ของ Docker)

local DockerConcept = {}
DockerConcept.__index = DockerConcept

function DockerConcept.new()
    local self    = setmetatable({}, DockerConcept)
    self.images   = {}
    self.containers = {}
    self.networks   = {}
    self.volumes    = {}
    return self
end

function DockerConcept:build(tag, config)
    local image = {
        id      = string.format("sha256:%032x", math.random(0x7FFFFFFF)),
        tag     = tag,
        base    = config.from,
        layers  = {},
        size    = 0,
        created = os.time(),
    }

    -- Simulate layer creation
    for _, instruction in ipairs(config.instructions or {}) do
        local layer = {
            type     = instruction.type,
            command  = instruction.command,
            size_mb  = math.random(1, 50),
        }
        table.insert(image.layers, layer)
        image.size = image.size + layer.size_mb
    end

    self.images[tag] = image
    print(string.format("[BUILD] Image '%s' built: %d layers, ~%dMB",
        tag, #image.layers, image.size))
    return image
end

function DockerConcept:run(imageName, config)
    local image = self.images[imageName]
    if not image then
        error("Image not found: " .. imageName)
    end

    local containerId = string.format("%12x", math.random(0x7FFFFFFFFFFFFFFF))
    local container   = {
        id       = containerId,
        name     = config.name or ("container_" .. containerId:sub(1, 6)),
        image    = imageName,
        status   = "running",
        ports    = config.ports or {},
        env      = config.env or {},
        volumes  = config.volumes or {},
        network  = config.network or "bridge",
        startTime = os.time(),
    }

    self.containers[containerId] = container
    print(string.format("[RUN] Container '%s' started from '%s'",
        container.name, imageName))
    return container
end

function DockerConcept:stop(containerId)
    local c = self.containers[containerId]
    if c then
        c.status = "stopped"
        print(string.format("[STOP] Container '%s' stopped", c.name))
    end
end

function DockerConcept:ps()
    print("CONTAINER ID   IMAGE           STATUS    NAMES")
    for id, c in pairs(self.containers) do
        print(string.format("%-14s %-15s %-9s %s",
            id:sub(1, 12), c.image:sub(1, 15), c.status, c.name))
    end
end

-- ทดสอบ
math.randomseed(42)
local docker = DockerConcept.new()

docker:build("my-lua-app:1.0", {
    from = "ubuntu:22.04",
    instructions = {
        {type = "RUN",  command = "apt-get install -y lua5.4"},
        {type = "COPY", command = "app.lua /app/"},
        {type = "CMD",  command = "lua /app/app.lua"},
    }
})

local c1 = docker:run("my-lua-app:1.0", {
    name    = "lua-app-1",
    ports   = {["8080/tcp"] = "8080"},
    env     = {APP_ENV = "production"},
})

local c2 = docker:run("my-lua-app:1.0", {
    name    = "lua-app-2",
    ports   = {["8080/tcp"] = "8081"},
    env     = {APP_ENV = "production"},
})

docker:ps()
docker:stop(c1.id)

print("\nAfter stop:")
docker:ps()
```

---

## 69.2 Dockerfile สำหรับ Lua Application

```lua
-- ตัวอย่างที่ 2: Dockerfile generator สำหรับ Lua app
local Dockerfile = {}
Dockerfile.__index = Dockerfile

function Dockerfile.new()
    local self = setmetatable({}, Dockerfile)
    self.instructions = {}
    return self
end

function Dockerfile:FROM(image, alias)
    local instr = "FROM " .. image
    if alias then instr = instr .. " AS " .. alias end
    table.insert(self.instructions, instr)
    return self
end

function Dockerfile:ARG(name, default)
    local instr = "ARG " .. name
    if default then instr = instr .. "=" .. default end
    table.insert(self.instructions, instr)
    return self
end

function Dockerfile:ENV(key, value)
    table.insert(self.instructions, string.format("ENV %s=%s", key, tostring(value)))
    return self
end

function Dockerfile:RUN(command)
    table.insert(self.instructions, "RUN " .. command)
    return self
end

function Dockerfile:COPY(src, dst, from)
    local instr = "COPY"
    if from then instr = instr .. " --from=" .. from end
    instr = instr .. string.format(" %s %s", src, dst)
    table.insert(self.instructions, instr)
    return self
end

function Dockerfile:WORKDIR(path)
    table.insert(self.instructions, "WORKDIR " .. path)
    return self
end

function Dockerfile:EXPOSE(port, protocol)
    local instr = "EXPOSE " .. tostring(port)
    if protocol then instr = instr .. "/" .. protocol end
    table.insert(self.instructions, instr)
    return self
end

function Dockerfile:CMD(cmd)
    if type(cmd) == "table" then
        local parts = {}
        for _, c in ipairs(cmd) do
            table.insert(parts, '"' .. c .. '"')
        end
        table.insert(self.instructions, "CMD [" .. table.concat(parts, ", ") .. "]")
    else
        table.insert(self.instructions, "CMD " .. cmd)
    end
    return self
end

function Dockerfile:ENTRYPOINT(cmd)
    if type(cmd) == "table" then
        local parts = {}
        for _, c in ipairs(cmd) do
            table.insert(parts, '"' .. c .. '"')
        end
        table.insert(self.instructions, "ENTRYPOINT [" .. table.concat(parts, ", ") .. "]")
    else
        table.insert(self.instructions, "ENTRYPOINT " .. cmd)
    end
    return self
end

function Dockerfile:HEALTHCHECK(config)
    local instr = "HEALTHCHECK"
    if config.interval then instr = instr .. " --interval=" .. config.interval end
    if config.timeout   then instr = instr .. " --timeout="  .. config.timeout  end
    if config.retries   then instr = instr .. " --retries="  .. config.retries  end
    instr = instr .. " CMD " .. config.cmd
    table.insert(self.instructions, instr)
    return self
end

function Dockerfile:USER(user)
    table.insert(self.instructions, "USER " .. user)
    return self
end

function Dockerfile:LABEL(labels)
    local parts = {}
    for k, v in pairs(labels) do
        table.insert(parts, string.format('%s="%s"', k, v))
    end
    table.insert(self.instructions, "LABEL " .. table.concat(parts, " \\\n      "))
    return self
end

function Dockerfile:render()
    return table.concat(self.instructions, "\n")
end

-- Dockerfile สำหรับ Lua CLI application
local luaDockerfile = Dockerfile.new()
    :FROM("ubuntu:22.04")
    :LABEL({
        maintainer  = "dev@example.com",
        version     = "1.0.0",
        description = "Lua application container",
    })
    :ARG("LUA_VERSION", "5.4")
    :ENV("LUA_PATH", "/app/?.lua;;")
    :ENV("APP_ENV",  "production")
    :RUN("apt-get update && apt-get install -y \\\n    lua5.4 \\\n    luarocks \\\n    && rm -rf /var/lib/apt/lists/*")
    :RUN("luarocks install luasocket")
    :RUN("luarocks install lua-cjson")
    :WORKDIR("/app")
    :COPY("*.lua", "./")
    :COPY("*.rockspec", "./")
    :RUN("adduser --disabled-password --gecos '' appuser")
    :USER("appuser")
    :EXPOSE(8080)
    :HEALTHCHECK({
        interval = "30s",
        timeout  = "5s",
        retries  = 3,
        cmd      = "lua /app/healthcheck.lua || exit 1",
    })
    :CMD({"lua", "main.lua"})

print("=== Dockerfile for Lua Application ===")
print(luaDockerfile:render())
```

---

## 69.3 Dockerfile สำหรับ OpenResty

```lua
-- ตัวอย่างที่ 3: OpenResty Dockerfile
local openrestyDockerfile = Dockerfile.new()
    :FROM("openresty/openresty:1.25.3.2-alpine")
    :LABEL({maintainer = "devops@example.com", version = "2.0.0"})
    :ENV("NGINX_WORKER_PROCESSES",  "auto")
    :ENV("NGINX_WORKER_CONNECTIONS", "1024")
    :RUN("apk add --no-cache \\\n    curl \\\n    luarocks \\\n    && luarocks install lua-resty-http")
    :COPY("nginx/nginx.conf",    "/usr/local/openresty/nginx/conf/nginx.conf")
    :COPY("nginx/conf.d/",       "/etc/nginx/conf.d/")
    :COPY("lua/",                "/usr/local/openresty/lualib/app/")
    :EXPOSE(80)
    :EXPOSE(443)
    :HEALTHCHECK({
        interval = "10s",
        timeout  = "3s",
        retries  = 3,
        cmd      = "curl -f http://localhost/health || exit 1",
    })
    :CMD({"/usr/local/openresty/bin/openresty", "-g", "daemon off;"})

print("=== Dockerfile for OpenResty ===")
print(openrestyDockerfile:render())
```

---

## 69.4 Multi-stage Build

```lua
-- ตัวอย่างที่ 4: Multi-stage Dockerfile
local function generateMultiStageDockerfile()
    local lines = {}

    -- Stage 1: Builder
    table.insert(lines, "# Stage 1: Builder - install dependencies")
    table.insert(lines, "FROM ubuntu:22.04 AS builder")
    table.insert(lines, "")
    table.insert(lines, "RUN apt-get update && apt-get install -y \\")
    table.insert(lines, "    lua5.4 \\")
    table.insert(lines, "    luarocks \\")
    table.insert(lines, "    build-essential \\")
    table.insert(lines, "    git \\")
    table.insert(lines, "    && rm -rf /var/lib/apt/lists/*")
    table.insert(lines, "")
    table.insert(lines, "WORKDIR /build")
    table.insert(lines, "COPY *.rockspec ./")
    table.insert(lines, "RUN luarocks install --tree /build/rocks --only-deps *.rockspec 2>/dev/null || true")
    table.insert(lines, "COPY . .")
    table.insert(lines, "")
    table.insert(lines, "# Run tests in builder stage")
    table.insert(lines, "RUN lua test/run_tests.lua")
    table.insert(lines, "")

    -- Stage 2: Production
    table.insert(lines, "# Stage 2: Production - minimal image")
    table.insert(lines, "FROM ubuntu:22.04 AS production")
    table.insert(lines, "")
    table.insert(lines, "RUN apt-get update && apt-get install -y \\")
    table.insert(lines, "    lua5.4 \\")
    table.insert(lines, "    libssl3 \\")
    table.insert(lines, "    && rm -rf /var/lib/apt/lists/*")
    table.insert(lines, "")
    table.insert(lines, "# Copy only what's needed")
    table.insert(lines, "COPY --from=builder /build/rocks /usr/local/lib/lua/rocks")
    table.insert(lines, "COPY --from=builder /build/src   /app/src")
    table.insert(lines, "COPY --from=builder /build/main.lua /app/")
    table.insert(lines, "")
    table.insert(lines, "# Security: non-root user")
    table.insert(lines, "RUN groupadd -r appgroup && useradd -r -g appgroup appuser")
    table.insert(lines, "RUN chown -R appuser:appgroup /app")
    table.insert(lines, "")
    table.insert(lines, "WORKDIR /app")
    table.insert(lines, "USER appuser")
    table.insert(lines, "")
    table.insert(lines, "ENV LUA_PATH=/app/src/?.lua;/usr/local/lib/lua/rocks/share/lua/5.4/?.lua;;")
    table.insert(lines, "ENV LUA_CPATH=/usr/local/lib/lua/rocks/lib/lua/5.4/?.so;;")
    table.insert(lines, "")
    table.insert(lines, "EXPOSE 8080")
    table.insert(lines, "HEALTHCHECK --interval=30s --timeout=5s --retries=3 \\")
    table.insert(lines, "    CMD lua /app/src/health.lua || exit 1")
    table.insert(lines, "")
    table.insert(lines, 'ENTRYPOINT ["lua"]')
    table.insert(lines, 'CMD ["/app/main.lua"]')

    return table.concat(lines, "\n")
end

print("=== Multi-stage Dockerfile ===")
print(generateMultiStageDockerfile())
```

---

## 69.5 Docker Compose สำหรับ Development

```lua
-- ตัวอย่างที่ 5: Docker Compose generator
local ComposeGenerator = {}
ComposeGenerator.__index = ComposeGenerator

function ComposeGenerator.new(version)
    local self    = setmetatable({}, ComposeGenerator)
    self.version  = version or "3.9"
    self.services = {}
    self.networks = {}
    self.volumes  = {}
    return self
end

function ComposeGenerator:addService(name, config)
    self.services[name] = config
    return self
end

function ComposeGenerator:addNetwork(name, config)
    self.networks[name] = config or {driver = "bridge"}
    return self
end

function ComposeGenerator:addVolume(name, config)
    self.volumes[name] = config or {}
    return self
end

function ComposeGenerator:_indent(str, spaces)
    local prefix = string.rep(" ", spaces)
    return str:gsub("\n", "\n" .. prefix)
end

function ComposeGenerator:_yamlValue(v, indent)
    indent = indent or 0
    local t = type(v)
    if t == "nil"     then return "null"
    elseif t == "boolean" then return tostring(v)
    elseif t == "number"  then return tostring(v)
    elseif t == "string"  then
        -- Quote if contains special chars
        if v:match("[:#{}%[%]]") or v == "" then
            return '"' .. v:gsub('"', '\\"') .. '"'
        end
        return v
    end
    return tostring(v)
end

function ComposeGenerator:render()
    local lines = {}
    table.insert(lines, "version: '" .. self.version .. "'")
    table.insert(lines, "")
    table.insert(lines, "services:")

    for name, svc in pairs(self.services) do
        table.insert(lines, "  " .. name .. ":")

        if svc.build then
            if type(svc.build) == "string" then
                table.insert(lines, "    build: " .. svc.build)
            else
                table.insert(lines, "    build:")
                table.insert(lines, "      context: " .. (svc.build.context or "."))
                if svc.build.dockerfile then
                    table.insert(lines, "      dockerfile: " .. svc.build.dockerfile)
                end
                if svc.build.target then
                    table.insert(lines, "      target: " .. svc.build.target)
                end
                if svc.build.args then
                    table.insert(lines, "      args:")
                    for k, v in pairs(svc.build.args) do
                        table.insert(lines, string.format("        %s: %s", k, tostring(v)))
                    end
                end
            end
        end

        if svc.image then
            table.insert(lines, "    image: " .. svc.image)
        end

        if svc.container_name then
            table.insert(lines, "    container_name: " .. svc.container_name)
        end

        if svc.restart then
            table.insert(lines, "    restart: " .. svc.restart)
        end

        if svc.ports and #svc.ports > 0 then
            table.insert(lines, "    ports:")
            for _, p in ipairs(svc.ports) do
                table.insert(lines, '      - "' .. p .. '"')
            end
        end

        if svc.environment then
            table.insert(lines, "    environment:")
            for k, v in pairs(svc.environment) do
                table.insert(lines, string.format("      %s: %s", k, tostring(v)))
            end
        end

        if svc.env_file then
            table.insert(lines, "    env_file:")
            for _, f in ipairs(svc.env_file) do
                table.insert(lines, "      - " .. f)
            end
        end

        if svc.volumes and #svc.volumes > 0 then
            table.insert(lines, "    volumes:")
            for _, v in ipairs(svc.volumes) do
                table.insert(lines, "      - " .. v)
            end
        end

        if svc.networks and #svc.networks > 0 then
            table.insert(lines, "    networks:")
            for _, n in ipairs(svc.networks) do
                table.insert(lines, "      - " .. n)
            end
        end

        if svc.depends_on and #svc.depends_on > 0 then
            table.insert(lines, "    depends_on:")
            for _, d in ipairs(svc.depends_on) do
                table.insert(lines, "      - " .. d)
            end
        end

        if svc.healthcheck then
            local hc = svc.healthcheck
            table.insert(lines, "    healthcheck:")
            table.insert(lines, '      test: ["CMD", ' ..
                table.concat(
                    (function()
                        local parts = {}
                        for p in hc.test:gmatch("%S+") do
                            table.insert(parts, '"' .. p .. '"')
                        end
                        return parts
                    end)(),
                ", ") .. "]")
            if hc.interval then
                table.insert(lines, "      interval: " .. hc.interval)
            end
            if hc.timeout then
                table.insert(lines, "      timeout: " .. hc.timeout)
            end
            if hc.retries then
                table.insert(lines, "      retries: " .. hc.retries)
            end
        end

        if svc.command then
            table.insert(lines, "    command: " .. svc.command)
        end

        if svc.deploy then
            table.insert(lines, "    deploy:")
            if svc.deploy.replicas then
                table.insert(lines, "      replicas: " .. svc.deploy.replicas)
            end
            if svc.deploy.resources then
                table.insert(lines, "      resources:")
                table.insert(lines, "        limits:")
                table.insert(lines, "          cpus: '" .. (svc.deploy.resources.cpus or "0.5") .. "'")
                table.insert(lines, "          memory: " .. (svc.deploy.resources.memory or "256M"))
            end
        end
    end

    -- Networks
    if next(self.networks) then
        table.insert(lines, "")
        table.insert(lines, "networks:")
        for name, net in pairs(self.networks) do
            table.insert(lines, "  " .. name .. ":")
            if net.driver then
                table.insert(lines, "    driver: " .. net.driver)
            end
        end
    end

    -- Volumes
    if next(self.volumes) then
        table.insert(lines, "")
        table.insert(lines, "volumes:")
        for name, vol in pairs(self.volumes) do
            table.insert(lines, "  " .. name .. ":")
            if vol.driver then
                table.insert(lines, "    driver: " .. vol.driver)
            end
        end
    end

    return table.concat(lines, "\n")
end

local compose = ComposeGenerator.new("3.9")

compose:addService("app", {
    build = {
        context    = ".",
        dockerfile = "Dockerfile",
        target     = "production",
        args       = {LUA_VERSION = "5.4"},
    },
    container_name = "lua-app",
    restart        = "unless-stopped",
    ports          = {"8080:8080"},
    environment    = {
        APP_ENV    = "development",
        LOG_LEVEL  = "debug",
        DB_HOST    = "postgres",
        REDIS_HOST = "redis",
    },
    volumes  = {
        "./src:/app/src",
        "./config:/app/config:ro",
        "app-logs:/app/logs",
    },
    networks    = {"app-network"},
    depends_on  = {"postgres", "redis"},
    healthcheck = {
        test     = "curl -f http://localhost:8080/health",
        interval = "30s",
        timeout  = "5s",
        retries  = 3,
    },
    deploy = {
        replicas  = 2,
        resources = {cpus = "0.5", memory = "256M"},
    },
})

compose:addService("postgres", {
    image          = "postgres:15-alpine",
    container_name = "lua-postgres",
    restart        = "unless-stopped",
    ports          = {"5432:5432"},
    environment    = {
        POSTGRES_DB       = "appdb",
        POSTGRES_USER     = "appuser",
        POSTGRES_PASSWORD = "secret",
    },
    volumes  = {"postgres-data:/var/lib/postgresql/data"},
    networks = {"app-network"},
    healthcheck = {
        test     = "pg_isready -U appuser -d appdb",
        interval = "10s",
        timeout  = "5s",
        retries  = 5,
    },
})

compose:addService("redis", {
    image          = "redis:7-alpine",
    container_name = "lua-redis",
    restart        = "unless-stopped",
    ports          = {"6379:6379"},
    command        = "redis-server --appendonly yes",
    volumes        = {"redis-data:/data"},
    networks       = {"app-network"},
    healthcheck    = {
        test     = "redis-cli ping",
        interval = "10s",
        timeout  = "3s",
        retries  = 3,
    },
})

compose:addService("nginx", {
    image          = "nginx:alpine",
    container_name = "lua-nginx",
    ports          = {"80:80", "443:443"},
    volumes        = {
        "./nginx/nginx.conf:/etc/nginx/nginx.conf:ro",
        "./nginx/ssl:/etc/nginx/ssl:ro",
    },
    networks    = {"app-network"},
    depends_on  = {"app"},
})

compose:addNetwork("app-network", {driver = "bridge"})
compose:addVolume("postgres-data", {})
compose:addVolume("redis-data",    {})
compose:addVolume("app-logs",      {})

print("=== docker-compose.yml ===")
print(compose:render())
```

---

## 69.6 Environment Variables Management

```lua
-- ตัวอย่างที่ 6: Environment variable management
local EnvManager = {}
EnvManager.__index = EnvManager

function EnvManager.new()
    local self       = setmetatable({}, EnvManager)
    self.required    = {}
    self.optional    = {}
    self.config      = {}
    return self
end

function EnvManager:require(name, description, validator)
    table.insert(self.required, {
        name        = name,
        description = description,
        validator   = validator,
    })
    return self
end

function EnvManager:optional(name, default, description)
    table.insert(self.optional, {
        name        = name,
        default     = default,
        description = description,
    })
    return self
end

function EnvManager:load()
    local errors = {}

    -- Load required variables
    for _, v in ipairs(self.required) do
        local value = os.getenv(v.name)
        if not value then
            table.insert(errors, string.format(
                "Required environment variable '%s' is not set: %s",
                v.name, v.description or ""))
        else
            if v.validator and not v.validator(value) then
                table.insert(errors, string.format(
                    "Invalid value for '%s': %s", v.name, value))
            else
                self.config[v.name] = value
            end
        end
    end

    if #errors > 0 then
        return nil, table.concat(errors, "\n")
    end

    -- Load optional variables
    for _, v in ipairs(self.optional) do
        self.config[v.name] = os.getenv(v.name) or v.default
    end

    return self.config, nil
end

function EnvManager:get(name)
    return self.config[name]
end

function EnvManager:getInt(name)
    return tonumber(self.config[name])
end

function EnvManager:getBool(name)
    local v = self.config[name]
    return v == "true" or v == "1" or v == "yes"
end

function EnvManager:generateDotenv()
    local lines = {}
    table.insert(lines, "# Required variables")
    for _, v in ipairs(self.required) do
        table.insert(lines, string.format("# %s (required) - %s",
            v.name, v.description or ""))
        table.insert(lines, v.name .. "=")
        table.insert(lines, "")
    end
    table.insert(lines, "# Optional variables (with defaults)")
    for _, v in ipairs(self.optional) do
        table.insert(lines, string.format("# %s - %s",
            v.name, v.description or ""))
        table.insert(lines, string.format("%s=%s", v.name, tostring(v.default or "")))
        table.insert(lines, "")
    end
    return table.concat(lines, "\n")
end

local env = EnvManager.new()

env:require("DATABASE_URL",    "PostgreSQL connection string",
    function(v) return v:match("^postgres://") ~= nil end)
env:require("SECRET_KEY",      "Application secret key (min 32 chars)",
    function(v) return #v >= 32 end)
env:require("REDIS_URL",       "Redis connection string")

env:optional("APP_ENV",        "production", "Application environment")
env:optional("PORT",           "8080",       "HTTP server port")
env:optional("LOG_LEVEL",      "INFO",       "Logging level")
env:optional("MAX_CONNECTIONS", "100",       "Database max connections")
env:optional("TIMEOUT_MS",     "5000",       "Request timeout in ms")
env:optional("DEBUG",          "false",      "Enable debug mode")

print("=== .env.example ===")
print(env:generateDotenv())

-- Simulate loading (without actual env vars set)
-- env:load() would fail without required vars being set
print("Config structure: DATABASE_URL, SECRET_KEY, REDIS_URL are required")
```

---

## 69.7 Volume Mounting

```lua
-- ตัวอย่างที่ 7: Volume configuration helper
local VolumeConfig = {}
VolumeConfig.__index = VolumeConfig

function VolumeConfig.new()
    local self   = setmetatable({}, VolumeConfig)
    self.volumes = {}
    return self
end

function VolumeConfig:bind(hostPath, containerPath, options)
    table.insert(self.volumes, {
        type     = "bind",
        source   = hostPath,
        target   = containerPath,
        readonly = options and options.readonly or false,
    })
    return self
end

function VolumeConfig:named(volumeName, containerPath, options)
    table.insert(self.volumes, {
        type     = "volume",
        source   = volumeName,
        target   = containerPath,
        readonly = options and options.readonly or false,
    })
    return self
end

function VolumeConfig:tmpfs(containerPath, size)
    table.insert(self.volumes, {
        type   = "tmpfs",
        target = containerPath,
        tmpfs  = {size = size},
    })
    return self
end

function VolumeConfig:toDockerFlags()
    local flags = {}
    for _, v in ipairs(self.volumes) do
        if v.type == "bind" then
            local flag = string.format("-v %s:%s", v.source, v.target)
            if v.readonly then flag = flag .. ":ro" end
            table.insert(flags, flag)
        elseif v.type == "volume" then
            local flag = string.format("--mount type=volume,source=%s,target=%s",
                v.source, v.target)
            if v.readonly then flag = flag .. ",readonly" end
            table.insert(flags, flag)
        elseif v.type == "tmpfs" then
            table.insert(flags, string.format("--mount type=tmpfs,target=%s", v.target))
        end
    end
    return table.concat(flags, " \\\n  ")
end

function VolumeConfig:toComposeYaml()
    local lines = {}
    table.insert(lines, "    volumes:")
    for _, v in ipairs(self.volumes) do
        if v.type == "bind" then
            local entry = string.format("      - %s:%s", v.source, v.target)
            if v.readonly then entry = entry .. ":ro" end
            table.insert(lines, entry)
        elseif v.type == "volume" then
            table.insert(lines, string.format("      - %s:%s", v.source, v.target))
        elseif v.type == "tmpfs" then
            table.insert(lines, string.format("      - type: tmpfs\n        target: %s", v.target))
        end
    end
    return table.concat(lines, "\n")
end

local vols = VolumeConfig.new()
    :bind("./src",       "/app/src")
    :bind("./config",    "/app/config", {readonly = true})
    :named("app-data",   "/app/data")
    :named("app-logs",   "/var/log/app")
    :tmpfs("/tmp/cache", "64m")

print("=== Docker run flags ===")
print("docker run " .. vols:toDockerFlags() .. " my-app:latest")
print("\n=== Docker Compose volumes section ===")
print(vols:toComposeYaml())
```

---

## 69.8 Health Checks in Docker

```lua
-- ตัวอย่างที่ 8: Health check script สำหรับ Lua app

-- health.lua - script ที่ใช้ใน Docker HEALTHCHECK
local function runHealthCheck()
    local checks = {}
    local overall = true

    -- Check 1: Process responsive
    local function checkProcess()
        -- ตรวจสอบว่า main process ยังทำงาน
        local pidFile = "/var/run/app.pid"
        local f       = io.open(pidFile, "r")
        if not f then return false, "PID file not found" end
        local pid = f:read("*n")
        f:close()
        if not pid then return false, "Invalid PID" end
        return true, "Process running (PID: " .. pid .. ")"
    end

    -- Check 2: HTTP endpoint
    local function checkHTTP()
        -- ใน production ใช้ socket หรือ curl
        -- สำหรับ demo simulate
        return true, "HTTP 200 OK"
    end

    -- Check 3: Database connectivity
    local function checkDatabase()
        -- ใน production เปิด connection จริง
        return true, "Database connected"
    end

    -- Check 4: Disk space
    local function checkDisk()
        -- ใน production ใช้ df command
        return true, "Disk space OK"
    end

    local checkList = {
        {name = "process",  fn = checkProcess},
        {name = "http",     fn = checkHTTP},
        {name = "database", fn = checkDatabase},
        {name = "disk",     fn = checkDisk},
    }

    for _, c in ipairs(checkList) do
        local ok, msg = c.fn()
        checks[c.name] = {ok = ok, message = msg}
        if not ok then overall = false end
    end

    return overall, checks
end

local ok, checks = runHealthCheck()

print("Health Check Result:")
for name, c in pairs(checks) do
    print(string.format("  %-15s [%s] %s",
        name, c.ok and "PASS" or "FAIL", c.message))
end
print("\nOverall: " .. (ok and "HEALTHY" or "UNHEALTHY"))

-- Exit code สำหรับ Docker: 0 = healthy, 1 = unhealthy
-- os.exit(ok and 0 or 1)
```

---

## 69.9 Docker Networking

```lua
-- ตัวอย่างที่ 9: Container networking concepts
local ContainerNetwork = {}
ContainerNetwork.__index = ContainerNetwork

function ContainerNetwork.new(name, config)
    local self     = setmetatable({}, ContainerNetwork)
    self.name      = name
    self.driver    = config.driver or "bridge"
    self.subnet    = config.subnet
    self.ipRange   = config.ipRange
    self.gateway   = config.gateway
    self.containers = {}
    return self
end

function ContainerNetwork:connect(containerName, config)
    self.containers[containerName] = {
        name     = containerName,
        aliases  = config.aliases  or {},
        ip       = config.ip,
    }
    print(string.format("[NET] Container '%s' connected to network '%s'",
        containerName, self.name))
end

function ContainerNetwork:resolve(name)
    -- Container name resolution
    for cName, c in pairs(self.containers) do
        if cName == name then return c.ip or cName end
        for _, alias in ipairs(c.aliases) do
            if alias == name then return c.ip or cName end
        end
    end
    return nil
end

function ContainerNetwork:generateDockerCommand()
    local lines = {}
    table.insert(lines, "# Create network")
    local cmd = string.format("docker network create \\\n  --driver %s", self.driver)
    if self.subnet then
        cmd = cmd .. string.format(" \\\n  --subnet %s", self.subnet)
    end
    if self.gateway then
        cmd = cmd .. string.format(" \\\n  --gateway %s", self.gateway)
    end
    cmd = cmd .. " \\\n  " .. self.name
    table.insert(lines, cmd)
    return table.concat(lines, "\n")
end

-- Service Discovery simulation
local ServiceDiscovery = {}
ServiceDiscovery.__index = ServiceDiscovery

function ServiceDiscovery.new()
    local self    = setmetatable({}, ServiceDiscovery)
    self.services = {}
    return self
end

function ServiceDiscovery:register(name, host, port, metadata)
    if not self.services[name] then
        self.services[name] = {}
    end
    table.insert(self.services[name], {
        host     = host,
        port     = port,
        metadata = metadata or {},
        healthy  = true,
    })
end

function ServiceDiscovery:discover(name)
    local instances = self.services[name] or {}
    local healthy   = {}
    for _, i in ipairs(instances) do
        if i.healthy then table.insert(healthy, i) end
    end
    return healthy
end

-- ทดสอบ
local appNetwork = ContainerNetwork.new("app-network", {
    driver  = "bridge",
    subnet  = "172.20.0.0/16",
    gateway = "172.20.0.1",
})

appNetwork:connect("api-service",    {aliases = {"api"}, ip = "172.20.0.10"})
appNetwork:connect("db-service",     {aliases = {"db", "postgres"}, ip = "172.20.0.20"})
appNetwork:connect("cache-service",  {aliases = {"cache", "redis"}, ip = "172.20.0.30"})
appNetwork:connect("nginx-proxy",    {ip = "172.20.0.2"})

print("=== Container Networking ===")
print(appNetwork:generateDockerCommand())
print("\nService Resolution:")
print("  'api'     -> " .. (appNetwork:resolve("api") or "not found"))
print("  'db'      -> " .. (appNetwork:resolve("db") or "not found"))
print("  'redis'   -> " .. (appNetwork:resolve("cache") or "not found"))
print("  'unknown' -> " .. (appNetwork:resolve("unknown") or "not found"))

local sd = ServiceDiscovery.new()
sd:register("payment-service", "172.20.0.11", 8080, {version = "1.0"})
sd:register("payment-service", "172.20.0.12", 8080, {version = "1.0"})
sd:register("user-service",    "172.20.0.21", 8080, {version = "2.0"})

local instances = sd:discover("payment-service")
print(string.format("\nDiscovered %d instances of payment-service", #instances))
for _, i in ipairs(instances) do
    print(string.format("  %s:%d", i.host, i.port))
end
```

---

## 69.10 Kubernetes Pod Spec

```lua
-- ตัวอย่างที่ 10: Kubernetes manifest generator
local K8sManifest = {}
K8sManifest.__index = K8sManifest

function K8sManifest.new()
    local self = setmetatable({}, K8sManifest)
    return self
end

function K8sManifest:deployment(config)
    local spec = {
        apiVersion = "apps/v1",
        kind       = "Deployment",
        metadata   = {
            name      = config.name,
            namespace = config.namespace or "default",
            labels    = config.labels or {app = config.name},
        },
        spec = {
            replicas = config.replicas or 2,
            selector = {
                matchLabels = config.labels or {app = config.name}
            },
            template = {
                metadata = {
                    labels = config.labels or {app = config.name}
                },
                spec = {
                    containers = {
                        {
                            name  = config.name,
                            image = config.image,
                            ports = config.ports and
                                (function()
                                    local ports = {}
                                    for _, p in ipairs(config.ports) do
                                        table.insert(ports, {containerPort = p})
                                    end
                                    return ports
                                end)() or nil,
                            env    = config.env,
                            resources = config.resources,
                            livenessProbe  = config.livenessProbe,
                            readinessProbe = config.readinessProbe,
                            volumeMounts   = config.volumeMounts,
                        }
                    },
                    volumes = config.volumes,
                }
            },
            strategy = {
                type           = "RollingUpdate",
                rollingUpdate  = {
                    maxSurge       = config.maxSurge       or 1,
                    maxUnavailable = config.maxUnavailable or 0,
                }
            }
        }
    }
    return spec
end

function K8sManifest:service(config)
    return {
        apiVersion = "v1",
        kind       = "Service",
        metadata   = {
            name      = config.name,
            namespace = config.namespace or "default",
        },
        spec = {
            selector = config.selector or {app = config.name},
            ports    = config.ports,
            type     = config.type or "ClusterIP",
        }
    }
end

function K8sManifest:configMap(name, namespace, data)
    return {
        apiVersion = "v1",
        kind       = "ConfigMap",
        metadata   = {name = name, namespace = namespace or "default"},
        data       = data,
    }
end

function K8sManifest:toYAML(manifest, indent)
    indent = indent or 0
    local lines = {}
    local prefix = string.rep("  ", indent)

    if type(manifest) == "table" then
        -- Check if it's an array
        local isArray = true
        local maxIdx  = 0
        for k in pairs(manifest) do
            if type(k) ~= "number" then isArray = false; break end
            maxIdx = math.max(maxIdx, k)
        end
        isArray = isArray and maxIdx == #manifest

        if isArray then
            for _, v in ipairs(manifest) do
                if type(v) == "table" then
                    local sub = self:toYAML(v, indent + 1)
                    -- First line gets '-' prefix
                    sub = sub:gsub("^" .. string.rep("  ", indent + 1), prefix .. "- ", 1)
                    table.insert(lines, sub)
                else
                    table.insert(lines, prefix .. "- " .. tostring(v))
                end
            end
        else
            for k, v in pairs(manifest) do
                if type(v) == "table" then
                    table.insert(lines, prefix .. k .. ":")
                    table.insert(lines, self:toYAML(v, indent + 1))
                elseif type(v) == "string" then
                    if v:match("[:#{}%[%]|>]") or v == "" then
                        table.insert(lines, string.format('%s%s: "%s"', prefix, k, v))
                    else
                        table.insert(lines, prefix .. k .. ": " .. v)
                    end
                elseif v ~= nil then
                    table.insert(lines, prefix .. k .. ": " .. tostring(v))
                end
            end
        end
    else
        table.insert(lines, prefix .. tostring(manifest))
    end

    return table.concat(lines, "\n")
end

local k8s = K8sManifest.new()

-- Deployment
local deployment = k8s:deployment({
    name      = "lua-api",
    namespace = "production",
    image     = "registry.example.com/lua-api:v2.1.0",
    replicas  = 3,
    labels    = {app = "lua-api", tier = "backend", version = "v2"},
    ports     = {8080},
    env       = {
        {name = "APP_ENV",     value = "production"},
        {name = "DB_HOST",     value = "postgres-service"},
        {name = "REDIS_HOST",  value = "redis-service"},
        {name  = "SECRET_KEY",
         valueFrom = {secretKeyRef = {name = "app-secrets", key = "secret-key"}}},
    },
    resources = {
        requests = {cpu = "100m",  memory = "128Mi"},
        limits   = {cpu = "500m",  memory = "512Mi"},
    },
    livenessProbe = {
        httpGet = {path = "/health", port = 8080},
        initialDelaySeconds = 30,
        periodSeconds       = 10,
    },
    readinessProbe = {
        httpGet = {path = "/ready", port = 8080},
        initialDelaySeconds = 5,
        periodSeconds       = 5,
    },
    maxSurge       = 1,
    maxUnavailable = 0,
})

print("=== Kubernetes Deployment ===")
print("apiVersion: " .. deployment.apiVersion)
print("kind: " .. deployment.kind)
print("metadata:")
print("  name: " .. deployment.metadata.name)
print("  namespace: " .. deployment.metadata.namespace)
print("spec:")
print("  replicas: " .. deployment.spec.replicas)
print("  strategy:")
print("    type: " .. deployment.spec.strategy.type)
print("    rollingUpdate:")
print("      maxSurge: " .. deployment.spec.strategy.rollingUpdate.maxSurge)
print("      maxUnavailable: " .. deployment.spec.strategy.rollingUpdate.maxUnavailable)
print("  template:")
print("    spec:")
print("      containers:")
for _, c in ipairs(deployment.spec.template.spec.containers) do
    print("        - name: " .. c.name)
    print("          image: " .. c.image)
    if c.resources then
        print("          resources:")
        print("            requests:")
        print("              cpu: " .. c.resources.requests.cpu)
        print("              memory: " .. c.resources.requests.memory)
        print("            limits:")
        print("              cpu: " .. c.resources.limits.cpu)
        print("              memory: " .. c.resources.limits.memory)
    end
end
```

---

## 69.11 ConfigMaps and Secrets

```lua
-- ตัวอย่างที่ 11: ConfigMap and Secret management
local function generateConfigMap(name, namespace, configData)
    local lines = {}
    table.insert(lines, "apiVersion: v1")
    table.insert(lines, "kind: ConfigMap")
    table.insert(lines, "metadata:")
    table.insert(lines, "  name: " .. name)
    table.insert(lines, "  namespace: " .. (namespace or "default"))
    table.insert(lines, "data:")
    for k, v in pairs(configData) do
        if type(v) == "string" and v:match("\n") then
            -- Multi-line value
            table.insert(lines, "  " .. k .. ": |")
            for line in v:gmatch("[^\n]+") do
                table.insert(lines, "    " .. line)
            end
        else
            table.insert(lines, string.format("  %s: %q", k, tostring(v)))
        end
    end
    return table.concat(lines, "\n")
end

local function generateSecret(name, namespace, secretData)
    -- ใน production ใช้ base64 encode
    local function base64Mock(s)
        return s:gsub(".", function(c)
            return string.format("%02x", c:byte())
        end)
    end

    local lines = {}
    table.insert(lines, "apiVersion: v1")
    table.insert(lines, "kind: Secret")
    table.insert(lines, "metadata:")
    table.insert(lines, "  name: " .. name)
    table.insert(lines, "  namespace: " .. (namespace or "default"))
    table.insert(lines, "type: Opaque")
    table.insert(lines, "data:")
    for k, v in pairs(secretData) do
        -- In real use: echo -n 'value' | base64
        table.insert(lines, string.format("  %s: <base64-encoded-%s>", k, k))
    end
    table.insert(lines, "# Generate with:")
    table.insert(lines, "# kubectl create secret generic " .. name .. " \\")
    for k, v in pairs(secretData) do
        table.insert(lines, string.format("#   --from-literal=%s='%s' \\", k, v:sub(1,3) .. "***"))
    end
    table.insert(lines, "#   -n " .. (namespace or "default"))
    return table.concat(lines, "\n")
end

print("=== ConfigMap ===")
print(generateConfigMap("app-config", "production", {
    APP_ENV          = "production",
    LOG_LEVEL        = "INFO",
    MAX_CONNECTIONS  = "100",
    TIMEOUT_MS       = "5000",
    ["nginx.conf"]   = "server {\n    listen 80;\n    location / {\n        proxy_pass http://app:8080;\n    }\n}",
}))

print("\n=== Secret ===")
print(generateSecret("app-secrets", "production", {
    ["database-url"] = "postgres://user:password@host/db",
    ["secret-key"]   = "super-secret-key-min-32-characters",
    ["api-token"]    = "tok-live-xxxxx",
}))
```

---

## 69.12 Horizontal Scaling

```lua
-- ตัวอย่างที่ 12: HPA (Horizontal Pod Autoscaler)
local function generateHPA(name, deploymentName, config)
    local lines = {}
    table.insert(lines, "apiVersion: autoscaling/v2")
    table.insert(lines, "kind: HorizontalPodAutoscaler")
    table.insert(lines, "metadata:")
    table.insert(lines, "  name: " .. name)
    table.insert(lines, "spec:")
    table.insert(lines, "  scaleTargetRef:")
    table.insert(lines, "    apiVersion: apps/v1")
    table.insert(lines, "    kind: Deployment")
    table.insert(lines, "    name: " .. deploymentName)
    table.insert(lines, string.format("  minReplicas: %d", config.minReplicas or 2))
    table.insert(lines, string.format("  maxReplicas: %d", config.maxReplicas or 10))
    table.insert(lines, "  metrics:")

    if config.cpu then
        table.insert(lines, "  - type: Resource")
        table.insert(lines, "    resource:")
        table.insert(lines, "      name: cpu")
        table.insert(lines, "      target:")
        table.insert(lines, "        type: Utilization")
        table.insert(lines, string.format("        averageUtilization: %d",
            config.cpu.targetPercent or 70))
    end

    if config.memory then
        table.insert(lines, "  - type: Resource")
        table.insert(lines, "    resource:")
        table.insert(lines, "      name: memory")
        table.insert(lines, "      target:")
        table.insert(lines, "        type: Utilization")
        table.insert(lines, string.format("        averageUtilization: %d",
            config.memory.targetPercent or 80))
    end

    if config.custom then
        for _, metric in ipairs(config.custom) do
            table.insert(lines, "  - type: Pods")
            table.insert(lines, "    pods:")
            table.insert(lines, "      metric:")
            table.insert(lines, "        name: " .. metric.name)
            table.insert(lines, "      target:")
            table.insert(lines, "        type: AverageValue")
            table.insert(lines, "        averageValue: " .. metric.targetValue)
        end
    end

    if config.behavior then
        table.insert(lines, "  behavior:")
        if config.behavior.scaleDown then
            table.insert(lines, "    scaleDown:")
            table.insert(lines, "      stabilizationWindowSeconds: " ..
                (config.behavior.scaleDown.stabilizationWindow or 300))
        end
        if config.behavior.scaleUp then
            table.insert(lines, "    scaleUp:")
            table.insert(lines, "      stabilizationWindowSeconds: " ..
                (config.behavior.scaleUp.stabilizationWindow or 60))
        end
    end

    return table.concat(lines, "\n")
end

print("=== HorizontalPodAutoscaler ===")
print(generateHPA("lua-api-hpa", "lua-api", {
    minReplicas = 2,
    maxReplicas = 20,
    cpu         = {targetPercent = 70},
    memory      = {targetPercent = 80},
    custom      = {
        {name = "http_requests_per_second", targetValue = "100"},
    },
    behavior    = {
        scaleDown = {stabilizationWindow = 300},
        scaleUp   = {stabilizationWindow = 60},
    }
}))
```

---

## 69.13 Rolling Deployments

```lua
-- ตัวอย่างที่ 13: Rolling deployment simulator
local RollingDeployment = {}
RollingDeployment.__index = RollingDeployment

function RollingDeployment.new(config)
    local self         = setmetatable({}, RollingDeployment)
    self.name          = config.name
    self.currentImage  = config.currentImage
    self.targetImage   = config.targetImage
    self.replicas      = config.replicas or 3
    self.maxSurge      = config.maxSurge or 1
    self.maxUnavailable = config.maxUnavailable or 0
    self.pods          = {}

    -- Initialize current pods
    for i = 1, self.replicas do
        self.pods[i] = {
            id      = "pod-" .. i,
            image   = self.currentImage,
            status  = "running",
            version = "old",
        }
    end
    return self
end

function RollingDeployment:currentStatus()
    local old     = 0
    local new     = 0
    local running = 0
    for _, p in ipairs(self.pods) do
        if p.version == "old"     then old     = old + 1 end
        if p.version == "new"     then new     = new + 1 end
        if p.status == "running"  then running = running + 1 end
    end
    return {old = old, new = new, running = running, total = #self.pods}
end

function RollingDeployment:printStatus()
    local s = self:currentStatus()
    print(string.format("  [%s] old=%d new=%d running=%d/%d",
        self.name, s.old, s.new, s.running, s.total))
end

function RollingDeployment:step()
    -- Find an old running pod to update
    for i, pod in ipairs(self.pods) do
        if pod.version == "old" and pod.status == "running" then
            -- Check if we can have unavailable pods
            local s = self:currentStatus()
            if s.running > self.replicas - self.maxUnavailable then
                print(string.format("  Updating %s: %s -> %s",
                    pod.id, self.currentImage, self.targetImage))
                pod.status  = "terminating"
                pod.version = "updating"
                pod.image   = self.targetImage
                -- Simulate: terminate old, start new
                pod.status  = "running"
                pod.version = "new"
                return true
            end
        end
    end
    return false  -- No more pods to update
end

function RollingDeployment:rollout()
    print(string.format("=== Rolling Update: %s ===", self.name))
    print(string.format("Old image: %s", self.currentImage))
    print(string.format("New image: %s", self.targetImage))
    print(string.format("Replicas: %d, MaxSurge: %d, MaxUnavailable: %d",
        self.replicas, self.maxSurge, self.maxUnavailable))
    print("\nInitial state:")
    self:printStatus()

    local step = 0
    while true do
        local s = self:currentStatus()
        if s.old == 0 then break end
        step = step + 1
        print(string.format("\nStep %d:", step))
        self:step()
        self:printStatus()
        if step > 20 then break end  -- Safety
    end

    print("\nRollout complete!")
    self:printStatus()
end

local deploy = RollingDeployment.new({
    name          = "lua-api",
    currentImage  = "lua-api:v1.0",
    targetImage   = "lua-api:v2.0",
    replicas      = 4,
    maxSurge      = 1,
    maxUnavailable = 1,
})

deploy:rollout()
```

---

## 69.14 Docker Image Layer Optimization

```lua
-- ตัวอย่างที่ 14: Dockerfile best practices checker
local DockerfileLinter = {}
DockerfileLinter.__index = DockerfileLinter

function DockerfileLinter.new()
    local self   = setmetatable({}, DockerfileLinter)
    self.rules   = {}
    self.issues  = {}
    return self
end

function DockerfileLinter:addRule(rule)
    table.insert(self.rules, rule)
end

function DockerfileLinter:check(dockerfile)
    self.issues = {}
    local instructions = {}
    for line in dockerfile:gmatch("[^\n]+") do
        local trimmed = line:match("^%s*(.-)%s*$")
        if #trimmed > 0 and not trimmed:match("^#") then
            table.insert(instructions, trimmed)
        end
    end

    for _, rule in ipairs(self.rules) do
        local issues = rule.check(instructions, dockerfile)
        for _, issue in ipairs(issues or {}) do
            table.insert(self.issues, {
                severity = rule.severity,
                rule     = rule.name,
                message  = issue,
            })
        end
    end
    return self.issues
end

-- Default rules
local linter = DockerfileLinter.new()

linter:addRule({
    name     = "NO_LATEST_TAG",
    severity = "ERROR",
    check    = function(instructions, raw)
        local issues = {}
        if raw:match("FROM%s+[^:]+:%s*latest") or raw:match("FROM%s+[^: \n]+%s*\n") then
            table.insert(issues, "Use specific image tag instead of 'latest'")
        end
        return issues
    end,
})

linter:addRule({
    name     = "NON_ROOT_USER",
    severity = "WARN",
    check    = function(instructions)
        local hasUser = false
        for _, i in ipairs(instructions) do
            if i:match("^USER%s+") then hasUser = true end
        end
        if not hasUser then
            return {"No USER instruction found - container will run as root"}
        end
    end,
})

linter:addRule({
    name     = "HAS_HEALTHCHECK",
    severity = "WARN",
    check    = function(instructions)
        local hasHC = false
        for _, i in ipairs(instructions) do
            if i:match("^HEALTHCHECK") then hasHC = true end
        end
        if not hasHC then
            return {"No HEALTHCHECK instruction found"}
        end
    end,
})

linter:addRule({
    name     = "CACHE_APT",
    severity = "INFO",
    check    = function(instructions, raw)
        local issues = {}
        if raw:match("apt%-get%s+update") and not raw:match("rm %-rf /var/lib/apt") then
            table.insert(issues, "Clean apt cache after install to reduce image size")
        end
        return issues
    end,
})

linter:addRule({
    name     = "COPY_NOT_ADD",
    severity = "INFO",
    check    = function(instructions)
        local issues = {}
        for _, i in ipairs(instructions) do
            if i:match("^ADD%s+") and not i:match("%.tar") then
                table.insert(issues,
                    "Use COPY instead of ADD for simple file copying")
            end
        end
        return issues
    end,
})

-- Test Dockerfile
local testDockerfile = [[
FROM ubuntu:latest
RUN apt-get update && apt-get install -y lua5.4
ADD config.json /app/config.json
COPY app.lua /app/
CMD lua /app/app.lua
]]

print("=== Dockerfile Linting ===")
print("Dockerfile:")
print(testDockerfile)
print("Issues found:")
local issues = linter:check(testDockerfile)
for _, issue in ipairs(issues) do
    print(string.format("  [%s] %s: %s",
        issue.severity, issue.rule, issue.message))
end
print(string.format("\nTotal: %d issues", #issues))
```

---

## 69.15 Container Orchestration

```lua
-- ตัวอย่างที่ 15: Simple container orchestrator
local Orchestrator = {}
Orchestrator.__index = Orchestrator

function Orchestrator.new()
    local self    = setmetatable({}, Orchestrator)
    self.services = {}
    self.nodes    = {}
    return self
end

function Orchestrator:addNode(name, resources)
    self.nodes[name] = {
        name      = name,
        cpu       = resources.cpu    or 4,     -- cores
        memory    = resources.memory or 8192,  -- MB
        usedCPU   = 0,
        usedMemory = 0,
        pods      = {},
    }
end

function Orchestrator:bestNode(cpuReq, memReq)
    local best = nil
    for _, node in pairs(self.nodes) do
        local freeCPU = node.cpu    - node.usedCPU
        local freeMem = node.memory - node.usedMemory
        if freeCPU >= cpuReq and freeMem >= memReq then
            if best == nil or (freeCPU > best.freeCPU) then
                best = {
                    node    = node,
                    freeCPU = freeCPU,
                    freeMem = freeMem,
                }
            end
        end
    end
    return best and best.node or nil
end

function Orchestrator:schedule(service, replicas)
    local svc = self.services[service]
    if not svc then
        return nil, "service not found"
    end

    local scheduled = {}
    for i = 1, replicas do
        local node = self:bestNode(svc.cpuRequest, svc.memRequest)
        if not node then
            return scheduled, "insufficient resources for replica " .. i
        end

        local podId = string.format("%s-%d-%04x",
            service, i, math.random(0xFFFF))
        local pod   = {
            id        = podId,
            service   = service,
            node      = node.name,
            cpuReq    = svc.cpuRequest,
            memReq    = svc.memRequest,
            status    = "running",
        }

        node.usedCPU    = node.usedCPU + svc.cpuRequest
        node.usedMemory = node.usedMemory + svc.memRequest
        table.insert(node.pods, pod)
        table.insert(scheduled, pod)
    end

    return scheduled, nil
end

function Orchestrator:defineService(name, config)
    self.services[name] = {
        name       = name,
        image      = config.image,
        cpuRequest = config.cpu    or 0.1,
        memRequest = config.memory or 128,
    }
end

function Orchestrator:printCluster()
    print("=== Cluster Status ===")
    for nodeName, node in pairs(self.nodes) do
        print(string.format("Node: %s (CPU: %.1f/%.1f, Memory: %d/%dMB)",
            nodeName,
            node.usedCPU, node.cpu,
            node.usedMemory, node.memory))
        for _, pod in ipairs(node.pods) do
            print(string.format("  - %s [%s]", pod.id, pod.status))
        end
    end
end

math.randomseed(77)
local orch = Orchestrator.new()
orch:addNode("node-1", {cpu = 4, memory = 8192})
orch:addNode("node-2", {cpu = 4, memory = 8192})
orch:addNode("node-3", {cpu = 2, memory = 4096})

orch:defineService("api",    {image = "api:v1",    cpu = 0.5, memory = 256})
orch:defineService("worker", {image = "worker:v1", cpu = 1.0, memory = 512})
orch:defineService("cache",  {image = "redis:7",   cpu = 0.25, memory = 256})

local pods, err = orch:schedule("api",    4)
print(string.format("Scheduled %d API pods%s", #pods, err and " (partial: "..err..")" or ""))

pods, err = orch:schedule("worker", 2)
print(string.format("Scheduled %d worker pods%s", #pods, err and " (partial: "..err..")" or ""))

pods, err = orch:schedule("cache",  1)
print(string.format("Scheduled %d cache pods%s", #pods, err and " (partial: "..err..")" or ""))

orch:printCluster()
```

---

## 69.16 สรุป Docker Best Practices

```lua
-- ตัวอย่างที่ 16: Dockerfile best practices summary
local BestPractices = {
    {
        category = "Security",
        practices = {
            "ใช้ USER instruction เพื่อ run ด้วย non-root user",
            "Scan image ด้วย trivy หรือ snyk",
            "ใช้ read-only filesystem: --read-only flag",
            "จำกัด capabilities: --cap-drop ALL",
            "ไม่ควร store secrets ใน image",
        }
    },
    {
        category = "Size Optimization",
        practices = {
            "ใช้ Alpine หรือ distroless base images",
            "Multi-stage builds เพื่อแยก builder และ runtime",
            "ล้าง package cache หลัง install",
            "ใช้ .dockerignore เพื่อ exclude unnecessary files",
            "Combine RUN commands เพื่อลด layers",
        }
    },
    {
        category = "Performance",
        practices = {
            "เรียง COPY จากที่เปลี่ยนน้อยก่อน เพื่อ cache layers",
            "ใช้ BuildKit (DOCKER_BUILDKIT=1)",
            "Mount cache สำหรับ package managers",
            "กำหนด resource limits (--memory, --cpus)",
        }
    },
    {
        category = "Reliability",
        practices = {
            "เพิ่ม HEALTHCHECK instruction",
            "Handle SIGTERM gracefully",
            "ใช้ tini หรือ dumb-init เป็น init process",
            "กำหนด restart policy ที่เหมาะสม",
            "Log to stdout/stderr เสมอ",
        }
    },
}

print("=== Docker Best Practices for Lua Applications ===")
for _, category in ipairs(BestPractices) do
    print(string.format("\n[%s]", category.category))
    for i, practice in ipairs(category.practices) do
        print(string.format("  %d. %s", i, practice))
    end
end

-- Example optimized Dockerfile
print("\n=== Optimized Dockerfile Example ===")
local optimized = [[
# ✓ Specific base image version
FROM openresty/openresty:1.25.3.2-alpine AS base

# ✓ Install with cache mount (BuildKit)
RUN --mount=type=cache,target=/var/cache/apk \
    apk add --no-cache curl luarocks

# ✓ Dependencies before app code (better caching)
WORKDIR /app
COPY *.rockspec ./
RUN luarocks install --tree /app/rocks --only-deps *.rockspec

# ✓ App code last (changes most frequently)
COPY lua/ ./lua/
COPY nginx/ ./nginx/

# ✓ Non-root user
RUN adduser -D -H -s /sbin/nologin appuser
USER appuser

EXPOSE 8080

# ✓ Health check
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
    CMD curl -sf http://localhost:8080/health || exit 1

# ✓ Use exec form for CMD
CMD ["/usr/local/openresty/bin/openresty", "-g", "daemon off;"]
]]

print(optimized)
```

---

## สรุปบทที่ 69

ในบทนี้เราได้เรียนรู้:

1. **Docker Basics** - Images, Containers, Networks, Volumes
2. **Dockerfile for Lua** - สร้าง image สำหรับ Lua app
3. **Dockerfile for OpenResty** - Nginx + LuaJIT container
4. **Multi-stage Builds** - Builder + Production stages
5. **Docker Compose** - Development environment
6. **Environment Variables** - Config management
7. **Volume Mounting** - Bind mounts, named volumes
8. **Health Checks** - Container health verification
9. **Docker Networking** - Bridge networks, DNS resolution
10. **Kubernetes Pod Spec** - Deployment, Service manifests
11. **ConfigMaps & Secrets** - Configuration management
12. **Horizontal Scaling** - HPA configuration
13. **Rolling Deployments** - Zero-downtime updates
14. **Dockerfile Linting** - Best practices checker
15. **Container Orchestration** - Scheduling concepts

---

*จบบทที่ 69 - Docker + Lua Applications*
