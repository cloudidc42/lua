# บทที่ 70: CI/CD Pipeline

## บทนำ

CI/CD (Continuous Integration / Continuous Deployment) คือแนวทางการพัฒนาซอฟต์แวร์ที่ automate การ build, test และ deploy เพื่อให้ code ถูก deliver ไปยัง production ได้เร็วและน่าเชื่อถือ ในบทนี้เราจะเรียนรู้การสร้าง CI/CD pipeline สำหรับ Lua applications

---

## 70.1 CI/CD Concepts

```lua
-- ตัวอย่างที่ 1: CI/CD Pipeline Simulator
local Pipeline = {}
Pipeline.__index = Pipeline

local STAGE_STATUS = {
    PENDING  = "pending",
    RUNNING  = "running",
    SUCCESS  = "success",
    FAILED   = "failed",
    SKIPPED  = "skipped",
    CANCELED = "canceled",
}

function Pipeline.new(name, config)
    local self     = setmetatable({}, Pipeline)
    self.name      = name
    self.stages    = {}
    self.env       = config and config.env or {}
    self.onFailure = config and config.onFailure
    self.results   = {}
    return self
end

function Pipeline:addStage(stage)
    table.insert(self.stages, {
        name     = stage.name,
        jobs     = stage.jobs or {},
        needs    = stage.needs or {},
        status   = STAGE_STATUS.PENDING,
        duration = 0,
    })
    return self
end

function Pipeline:run()
    local startTime = os.clock()
    print(string.format("=== Pipeline: %s ===", self.name))

    for _, stage in ipairs(self.stages) do
        -- Check if all needed stages passed
        local canRun = true
        for _, needed in ipairs(stage.needs) do
            if self.results[needed] ~= STAGE_STATUS.SUCCESS then
                canRun = false
                break
            end
        end

        if not canRun then
            stage.status = STAGE_STATUS.SKIPPED
            print(string.format("[SKIP]  %s (dependencies not met)", stage.name))
            self.results[stage.name] = STAGE_STATUS.SKIPPED
        else
            stage.status = STAGE_STATUS.RUNNING
            print(string.format("[RUN]   %s", stage.name))
            local stageStart = os.clock()

            -- Run all jobs in stage
            local allPassed = true
            for _, job in ipairs(stage.jobs) do
                local ok, err = pcall(job.fn, self.env)
                if ok then
                    print(string.format("  [PASS] %s", job.name))
                else
                    print(string.format("  [FAIL] %s: %s", job.name, tostring(err)))
                    allPassed = false
                    if job.allowFailure then
                        print(string.format("         (allowed to fail)"))
                        allPassed = true
                    end
                end
            end

            stage.duration = os.clock() - stageStart
            stage.status   = allPassed and STAGE_STATUS.SUCCESS or STAGE_STATUS.FAILED
            self.results[stage.name] = stage.status

            local icon = stage.status == STAGE_STATUS.SUCCESS and "[OK]  " or "[FAIL]"
            print(string.format("%s %s (%.2fs)", icon, stage.name, stage.duration))

            if not allPassed then
                print("\nPipeline failed at stage: " .. stage.name)
                if self.onFailure then self.onFailure(stage) end
                return false
            end
        end
    end

    local totalTime = os.clock() - startTime
    print(string.format("\nPipeline completed successfully in %.2fs", totalTime))
    return true
end

-- ทดสอบ
local pipeline = Pipeline.new("lua-app-ci", {
    env = {
        APP_NAME = "lua-api",
        VERSION  = "2.0.0",
    }
})

pipeline:addStage({
    name = "lint",
    jobs = {
        {name = "luacheck", fn = function(env)
            -- Simulate luacheck
            print("      Running luacheck on " .. env.APP_NAME)
            -- if hasErrors then error("lint errors found") end
        end},
        {name = "stylua", fn = function(env)
            print("      Running stylua format check")
        end},
    }
})

pipeline:addStage({
    name  = "test",
    needs = {"lint"},
    jobs  = {
        {name = "unit-tests", fn = function(env)
            print("      Running unit tests...")
            -- Simulate test run
            local passed = math.random(45, 50)
            print(string.format("      Tests: %d passed, 0 failed", passed))
        end},
        {name = "integration-tests", fn = function(env)
            print("      Running integration tests...")
        end},
    }
})

pipeline:addStage({
    name  = "build",
    needs = {"test"},
    jobs  = {
        {name = "docker-build", fn = function(env)
            print(string.format("      Building %s:%s", env.APP_NAME, env.VERSION))
        end},
    }
})

pipeline:addStage({
    name  = "deploy-staging",
    needs = {"build"},
    jobs  = {
        {name = "deploy", fn = function(env)
            print("      Deploying to staging...")
        end},
        {name = "smoke-test", fn = function(env)
            print("      Running smoke tests on staging...")
        end},
    }
})

math.randomseed(42)
pipeline:run()
```

---

## 70.2 GitHub Actions Workflow สำหรับ Lua

```lua
-- ตัวอย่างที่ 2: GitHub Actions YAML generator
local ActionsWorkflow = {}
ActionsWorkflow.__index = ActionsWorkflow

function ActionsWorkflow.new(name)
    local self   = setmetatable({}, ActionsWorkflow)
    self.name    = name
    self.on      = {}
    self.env     = {}
    self.jobs    = {}
    return self
end

function ActionsWorkflow:onPush(branches, paths)
    self.on.push = {
        branches = branches,
        paths    = paths,
    }
    return self
end

function ActionsWorkflow:onPR(branches)
    self.on.pull_request = {branches = branches}
    return self
end

function ActionsWorkflow:onSchedule(cron)
    if not self.on.schedule then self.on.schedule = {} end
    table.insert(self.on.schedule, {cron = cron})
    return self
end

function ActionsWorkflow:setEnv(key, value)
    self.env[key] = value
    return self
end

function ActionsWorkflow:addJob(id, config)
    self.jobs[id] = {
        name        = config.name or id,
        runsOn      = config.runsOn or "ubuntu-latest",
        needs       = config.needs,
        env         = config.env,
        strategy    = config.strategy,
        steps       = config.steps or {},
        outputs     = config.outputs,
        ["if"]      = config["if"],
        permissions = config.permissions,
        timeout     = config.timeout,
        services    = config.services,
    }
    return self
end

function ActionsWorkflow:renderYAML()
    local lines = {}

    table.insert(lines, "name: " .. self.name)
    table.insert(lines, "")

    -- Triggers
    table.insert(lines, "on:")
    if self.on.push then
        table.insert(lines, "  push:")
        if self.on.push.branches then
            table.insert(lines, "    branches:")
            for _, b in ipairs(self.on.push.branches) do
                table.insert(lines, "      - " .. b)
            end
        end
        if self.on.push.paths then
            table.insert(lines, "    paths:")
            for _, p in ipairs(self.on.push.paths) do
                table.insert(lines, "      - " .. p)
            end
        end
    end
    if self.on.pull_request then
        table.insert(lines, "  pull_request:")
        if self.on.pull_request.branches then
            table.insert(lines, "    branches:")
            for _, b in ipairs(self.on.pull_request.branches) do
                table.insert(lines, "      - " .. b)
            end
        end
    end
    if self.on.schedule then
        table.insert(lines, "  schedule:")
        for _, s in ipairs(self.on.schedule) do
            table.insert(lines, '    - cron: "' .. s.cron .. '"')
        end
    end
    table.insert(lines, "")

    -- Global env
    if next(self.env) then
        table.insert(lines, "env:")
        for k, v in pairs(self.env) do
            table.insert(lines, string.format("  %s: %s", k, tostring(v)))
        end
        table.insert(lines, "")
    end

    -- Jobs
    table.insert(lines, "jobs:")
    for jobId, job in pairs(self.jobs) do
        table.insert(lines, "  " .. jobId .. ":")
        table.insert(lines, "    name: " .. job.name)
        table.insert(lines, "    runs-on: " .. job.runsOn)

        if job.timeout then
            table.insert(lines, "    timeout-minutes: " .. job.timeout)
        end

        if job["if"] then
            table.insert(lines, '    if: ' .. job["if"])
        end

        if job.permissions then
            table.insert(lines, "    permissions:")
            for k, v in pairs(job.permissions) do
                table.insert(lines, string.format("      %s: %s", k, v))
            end
        end

        if job.needs then
            if type(job.needs) == "string" then
                table.insert(lines, "    needs: " .. job.needs)
            else
                table.insert(lines, "    needs:")
                for _, n in ipairs(job.needs) do
                    table.insert(lines, "      - " .. n)
                end
            end
        end

        if job.env then
            table.insert(lines, "    env:")
            for k, v in pairs(job.env) do
                table.insert(lines, string.format("      %s: %s", k, tostring(v)))
            end
        end

        if job.strategy then
            table.insert(lines, "    strategy:")
            if job.strategy.failFast ~= nil then
                table.insert(lines, "      fail-fast: " .. tostring(job.strategy.failFast))
            end
            if job.strategy.matrix then
                table.insert(lines, "      matrix:")
                for k, vals in pairs(job.strategy.matrix) do
                    table.insert(lines, "        " .. k .. ":")
                    for _, v in ipairs(vals) do
                        table.insert(lines, "          - " .. tostring(v))
                    end
                end
            end
        end

        if job.services then
            table.insert(lines, "    services:")
            for svcName, svc in pairs(job.services) do
                table.insert(lines, "      " .. svcName .. ":")
                table.insert(lines, "        image: " .. svc.image)
                if svc.ports then
                    table.insert(lines, "        ports:")
                    for _, p in ipairs(svc.ports) do
                        table.insert(lines, '          - "' .. p .. '"')
                    end
                end
                if svc.env then
                    table.insert(lines, "        env:")
                    for k, v in pairs(svc.env) do
                        table.insert(lines, string.format("          %s: %s", k, tostring(v)))
                    end
                end
                if svc.options then
                    table.insert(lines, "        options: " .. svc.options)
                end
            end
        end

        -- Steps
        table.insert(lines, "    steps:")
        for _, step in ipairs(job.steps) do
            local prefix = "      - "
            if step.name then
                table.insert(lines, prefix .. 'name: "' .. step.name .. '"')
                prefix = "        "
            end
            if step.id then
                table.insert(lines, prefix .. "id: " .. step.id)
            end
            if step["if"] then
                table.insert(lines, prefix .. "if: " .. step["if"])
            end
            if step.uses then
                table.insert(lines, prefix .. "uses: " .. step.uses)
                if step.with then
                    table.insert(lines, prefix .. "with:")
                    for k, v in pairs(step.with) do
                        local val = tostring(v)
                        if val:match("\n") then
                            table.insert(lines, prefix .. "  " .. k .. ": |")
                            for line in val:gmatch("[^\n]+") do
                                table.insert(lines, prefix .. "    " .. line)
                            end
                        else
                            table.insert(lines, string.format("%s  %s: %s",
                                prefix, k, val))
                        end
                    end
                end
            elseif step.run then
                table.insert(lines, prefix .. "run: |")
                for line in step.run:gmatch("[^\n]+") do
                    table.insert(lines, prefix .. "  " .. line)
                end
            end
            if step.env then
                table.insert(lines, prefix .. "env:")
                for k, v in pairs(step.env) do
                    table.insert(lines, string.format("%s  %s: %s", prefix, k, tostring(v)))
                end
            end
            if step.continueOnError then
                table.insert(lines, prefix .. "continue-on-error: true")
            end
        end

        table.insert(lines, "")
    end

    return table.concat(lines, "\n")
end

-- Build workflow
local workflow = ActionsWorkflow.new("Lua CI/CD Pipeline")

workflow
    :onPush({"main", "develop"}, {"**.lua", "Dockerfile", ".github/**"})
    :onPR({"main", "develop"})
    :setEnv("REGISTRY",     "ghcr.io")
    :setEnv("IMAGE_NAME",   "myorg/lua-api")
    :setEnv("LUA_VERSION",  "5.4")

-- Lint job
workflow:addJob("lint", {
    name   = "Lint and Format Check",
    runsOn = "ubuntu-latest",
    steps  = {
        {
            name = "Checkout code",
            uses = "actions/checkout@v4",
        },
        {
            name = "Install luacheck",
            run  = "sudo apt-get install -y luarocks\nsudo luarocks install luacheck",
        },
        {
            name = "Run luacheck",
            run  = "luacheck src/ --config .luacheckrc",
        },
    }
})

-- Test job
workflow:addJob("test", {
    name   = "Run Tests",
    runsOn = "ubuntu-latest",
    needs  = "lint",
    strategy = {
        failFast = false,
        matrix   = {
            lua_version = {"5.4", "luajit-2.1"},
        }
    },
    services = {
        postgres = {
            image   = "postgres:15",
            ports   = {"5432:5432"},
            env     = {POSTGRES_PASSWORD = "testpass", POSTGRES_DB = "testdb"},
            options = "--health-cmd pg_isready --health-interval 10s",
        },
        redis = {
            image   = "redis:7",
            ports   = {"6379:6379"},
            options = "--health-cmd \"redis-cli ping\" --health-interval 10s",
        },
    },
    steps = {
        {name = "Checkout code", uses = "actions/checkout@v4"},
        {
            name = "Setup Lua ${{ matrix.lua_version }}",
            uses = "leafo/gh-actions-lua@v10",
            with = {luaVersion = "${{ matrix.lua_version }}"},
        },
        {
            name = "Setup LuaRocks",
            uses = "leafo/gh-actions-luarocks@v4",
        },
        {
            name = "Cache LuaRocks packages",
            uses = "actions/cache@v3",
            with = {
                path = "lua_modules",
                key  = "${{ runner.os }}-luarocks-${{ hashFiles('*.rockspec') }}",
            }
        },
        {
            name = "Install dependencies",
            run  = "luarocks install --tree lua_modules",
        },
        {
            name = "Run tests",
            run  = "lua test/run_all.lua",
            env  = {
                DB_URL    = "postgresql://postgres:testpass@localhost/testdb",
                REDIS_URL = "redis://localhost:6379",
                APP_ENV   = "test",
            },
        },
        {
            name = "Upload coverage",
            uses = "codecov/codecov-action@v3",
            with = {file = "coverage.out"},
            continueOnError = true,
        },
    }
})

-- Build job
workflow:addJob("build", {
    name    = "Build Docker Image",
    runsOn  = "ubuntu-latest",
    needs   = "test",
    permissions = {contents = "read", packages = "write"},
    steps   = {
        {name = "Checkout code", uses = "actions/checkout@v4"},
        {
            name = "Set up Docker Buildx",
            uses = "docker/setup-buildx-action@v3",
        },
        {
            name = "Login to Container Registry",
            uses = "docker/login-action@v3",
            with = {
                registry = "${{ env.REGISTRY }}",
                username = "${{ github.actor }}",
                password = "${{ secrets.GITHUB_TOKEN }}",
            }
        },
        {
            name = "Extract metadata",
            id   = "meta",
            uses = "docker/metadata-action@v5",
            with = {
                images = "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}",
                tags   = "type=ref,event=branch\ntype=semver,pattern={{version}}\ntype=sha",
            }
        },
        {
            name = "Build and push",
            uses = "docker/build-push-action@v5",
            with = {
                context      = ".",
                push         = "${{ github.event_name != 'pull_request' }}",
                tags         = "${{ steps.meta.outputs.tags }}",
                labels       = "${{ steps.meta.outputs.labels }}",
                cache_from   = "type=gha",
                cache_to     = "type=gha,mode=max",
            }
        },
    }
})

-- Deploy staging
workflow:addJob("deploy-staging", {
    name    = "Deploy to Staging",
    runsOn  = "ubuntu-latest",
    needs   = "build",
    ["if"]  = "github.ref == 'refs/heads/develop'",
    env     = {
        KUBECONFIG = "${{ secrets.STAGING_KUBECONFIG }}",
    },
    steps   = {
        {name = "Checkout code", uses = "actions/checkout@v4"},
        {
            name = "Setup kubectl",
            uses = "azure/setup-kubectl@v3",
            with = {version = "v1.28.0"},
        },
        {
            name = "Deploy to staging",
            run  = [[
kubectl set image deployment/lua-api \
  lua-api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
  -n staging
kubectl rollout status deployment/lua-api -n staging --timeout=5m]],
        },
        {
            name = "Run smoke tests",
            run  = "lua test/smoke_test.lua https://staging.example.com",
        },
    }
})

-- Deploy production
workflow:addJob("deploy-production", {
    name    = "Deploy to Production",
    runsOn  = "ubuntu-latest",
    needs   = {"build", "deploy-staging"},
    ["if"]  = "github.ref == 'refs/heads/main'",
    steps   = {
        {name = "Checkout code", uses = "actions/checkout@v4"},
        {
            name = "Deploy with Blue-Green",
            run  = [[
echo "Starting blue-green deployment..."
./scripts/deploy_bluegreen.sh \
  --image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
  --namespace production]],
            env = {KUBECONFIG = "${{ secrets.PROD_KUBECONFIG }}"},
        },
        {
            name = "Notify on success",
            uses = "slackapi/slack-github-action@v1",
            with = {
                payload = '{"text": "Deployment successful: ${{ github.sha }}"}',
            },
            env  = {SLACK_WEBHOOK_URL = "${{ secrets.SLACK_WEBHOOK }}"},
        },
    }
})

print("=== GitHub Actions Workflow ===")
print(workflow:renderYAML())
```

---

## 70.3 Running Tests in CI

```lua
-- ตัวอย่างที่ 3: Test runner สำหรับ CI
local TestRunner = {}
TestRunner.__index = TestRunner

function TestRunner.new(config)
    local self     = setmetatable({}, TestRunner)
    self.config    = config or {}
    self.suites    = {}
    self.results   = {
        passed  = 0,
        failed  = 0,
        skipped = 0,
        errors  = {},
    }
    self.startTime = os.clock()
    return self
end

function TestRunner:suite(name, fn)
    table.insert(self.suites, {name = name, fn = fn})
end

function TestRunner:run()
    print("Running test suites...")
    print(string.rep("-", 60))

    for _, suite in ipairs(self.suites) do
        print(string.format("\n  Suite: %s", suite.name))
        local tests = {}
        local t     = {
            test = function(name, fn)
                table.insert(tests, {name = name, fn = fn})
            end,
            skip = function(name)
                table.insert(tests, {name = name, skip = true})
            end,
        }
        suite.fn(t)

        for _, test in ipairs(tests) do
            if test.skip then
                print(string.format("    SKIP %s", test.name))
                self.results.skipped = self.results.skipped + 1
            else
                local ok, err = pcall(test.fn)
                if ok then
                    print(string.format("    PASS %s", test.name))
                    self.results.passed = self.results.passed + 1
                else
                    print(string.format("    FAIL %s", test.name))
                    print(string.format("         %s", tostring(err)))
                    self.results.failed = self.results.failed + 1
                    table.insert(self.results.errors, {
                        suite = suite.name,
                        test  = test.name,
                        error = tostring(err),
                    })
                end
            end
        end
    end

    local elapsed = os.clock() - self.startTime
    self:printSummary(elapsed)
    return self.results.failed == 0
end

function TestRunner:printSummary(elapsed)
    print("\n" .. string.rep("-", 60))
    print(string.format("Tests: %d passed, %d failed, %d skipped (%.3fs)",
        self.results.passed,
        self.results.failed,
        self.results.skipped,
        elapsed))

    if #self.results.errors > 0 then
        print("\nFailed tests:")
        for _, e in ipairs(self.results.errors) do
            print(string.format("  %s > %s", e.suite, e.test))
            print(string.format("  Error: %s", e.error))
        end
    end
end

function TestRunner:exitCode()
    return self.results.failed > 0 and 1 or 0
end

-- Assert helpers
local function assertEqual(actual, expected, msg)
    if actual ~= expected then
        error(string.format("Expected %s but got %s%s",
            tostring(expected), tostring(actual),
            msg and (" - " .. msg) or ""))
    end
end

local function assertNotNil(value, msg)
    if value == nil then
        error("Expected non-nil value" .. (msg and (" - " .. msg) or ""))
    end
end

local function assertTrue(value, msg)
    if not value then
        error("Expected true but got " .. tostring(value) ..
            (msg and (" - " .. msg) or ""))
    end
end

local function assertError(fn, msg)
    local ok = pcall(fn)
    if ok then
        error("Expected error but none was thrown" ..
            (msg and (" - " .. msg) or ""))
    end
end

-- Sample tests
local runner = TestRunner.new()

runner:suite("String utilities", function(t)
    t.test("should trim whitespace", function()
        local function trim(s) return s:match("^%s*(.-)%s*$") end
        assertEqual(trim("  hello  "), "hello")
        assertEqual(trim(""),          "")
        assertEqual(trim("no spaces"), "no spaces")
    end)

    t.test("should split by delimiter", function()
        local function split(s, sep)
            local parts = {}
            for p in s:gmatch("[^" .. sep .. "]+") do
                table.insert(parts, p)
            end
            return parts
        end
        local parts = split("a,b,c", ",")
        assertEqual(#parts, 3)
        assertEqual(parts[1], "a")
        assertEqual(parts[3], "c")
    end)

    t.skip("TODO: test unicode handling")
end)

runner:suite("Math utilities", function(t)
    t.test("should clamp values", function()
        local function clamp(v, min, max) return math.max(min, math.min(max, v)) end
        assertEqual(clamp(5,  0, 10),  5)
        assertEqual(clamp(-1, 0, 10),  0)
        assertEqual(clamp(15, 0, 10), 10)
    end)

    t.test("should calculate average", function()
        local function avg(t)
            local s = 0
            for _, v in ipairs(t) do s = s + v end
            return s / #t
        end
        assertEqual(avg({1, 2, 3}), 2)
        assertEqual(avg({10, 20}),  15)
    end)
end)

runner:suite("Error handling", function(t)
    t.test("should handle nil gracefully", function()
        local function safeGet(t, key)
            if t == nil then return nil end
            return t[key]
        end
        assertEqual(safeGet(nil,    "key"),  nil)
        assertEqual(safeGet({a=1}, "a"),    1)
        assertEqual(safeGet({},    "missing"), nil)
    end)

    t.test("should propagate errors", function()
        assertError(function()
            error("expected error")
        end)
    end)
end)

local success = runner:run()
print("\nTest runner exit code: " .. runner:exitCode())
```

---

## 70.4 Linting with Luacheck

```lua
-- ตัวอย่างที่ 4: Luacheck configuration generator
local LuacheckConfig = {}
LuacheckConfig.__index = LuacheckConfig

function LuacheckConfig.new()
    local self   = setmetatable({}, LuacheckConfig)
    self.options = {}
    return self
end

function LuacheckConfig:set(key, value)
    self.options[key] = value
    return self
end

function LuacheckConfig:render()
    local lines = {}
    table.insert(lines, "-- .luacheckrc")
    table.insert(lines, "-- Luacheck configuration for CI")
    table.insert(lines, "")

    -- Format each option
    for key, value in pairs(self.options) do
        if type(value) == "boolean" then
            table.insert(lines, key .. " = " .. tostring(value))
        elseif type(value) == "number" then
            table.insert(lines, key .. " = " .. tostring(value))
        elseif type(value) == "string" then
            table.insert(lines, key .. ' = "' .. value .. '"')
        elseif type(value) == "table" then
            local parts = {}
            for _, v in ipairs(value) do
                if type(v) == "string" then
                    table.insert(parts, '"' .. v .. '"')
                else
                    table.insert(parts, tostring(v))
                end
            end
            table.insert(lines, key .. " = {" .. table.concat(parts, ", ") .. "}")
        end
    end

    return table.concat(lines, "\n")
end

-- .luacheckrc สำหรับ OpenResty project
local config = LuacheckConfig.new()
config
    :set("std",      "luajit")
    :set("unused",   true)
    :set("redefined", true)
    :set("max_line_length", 120)
    :set("ignore", {
        "212",  -- unused argument
        "213",  -- unused loop variable
    })
    :set("globals", {
        "ngx",
        "jit",
        "bit",
        "table",
        "string",
        "math",
        "io",
        "os",
    })

print("=== .luacheckrc ===")
print(config:render())

-- Simulated luacheck run
local function simulateLuacheck(files)
    print("\n=== Luacheck Output Simulation ===")
    local issues = {}

    -- Simulate findings
    local simulatedIssues = {
        {file = "src/api.lua",    line = 45,  col = 10, code = "W112", msg = "implicit self"},
        {file = "src/utils.lua",  line = 23,  col = 5,  code = "E011", msg = "expected '=' near 'end'"},
        {file = "src/config.lua", line = 78,  col = 1,  code = "W611", msg = "line is too long"},
    }

    local errors  = 0
    local warnings = 0

    for _, issue in ipairs(simulatedIssues) do
        local severity = issue.code:sub(1,1) == "E" and "error" or "warning"
        print(string.format("%s:%d:%d: (%s) [%s] %s",
            issue.file, issue.line, issue.col,
            severity, issue.code, issue.msg))
        if severity == "error" then errors = errors + 1
        else warnings = warnings + 1 end
    end

    print(string.format("\nTotal: %d errors, %d warnings", errors, warnings))
    return errors == 0
end

simulateLuacheck({"src/*.lua"})
```

---

## 70.5 Code Coverage

```lua
-- ตัวอย่างที่ 5: Code Coverage Tracker
local Coverage = {}
Coverage.__index = Coverage

function Coverage.new()
    local self    = setmetatable({}, Coverage)
    self.files    = {}   -- filename -> {lines -> {hit_count, executable}}
    self.enabled  = true
    return self
end

function Coverage:trackLine(filename, lineNum)
    if not self.enabled then return end
    if not self.files[filename] then
        self.files[filename] = {}
    end
    local line = self.files[filename][lineNum]
    if not line then
        self.files[filename][lineNum] = {hits = 0, executable = true}
    end
    self.files[filename][lineNum].hits =
        self.files[filename][lineNum].hits + 1
end

function Coverage:markExecutable(filename, lines)
    if not self.files[filename] then
        self.files[filename] = {}
    end
    for _, lineNum in ipairs(lines) do
        if not self.files[filename][lineNum] then
            self.files[filename][lineNum] = {hits = 0, executable = true}
        end
    end
end

function Coverage:report()
    local totalLines  = 0
    local hitLines    = 0
    local fileReports = {}

    for filename, lines in pairs(self.files) do
        local fileTotalLines = 0
        local fileHitLines   = 0
        for _, line in pairs(lines) do
            if line.executable then
                fileTotalLines = fileTotalLines + 1
                if line.hits > 0 then
                    fileHitLines = fileHitLines + 1
                end
            end
        end
        local pct = fileTotalLines > 0 and
            (fileHitLines / fileTotalLines * 100) or 0

        table.insert(fileReports, {
            file  = filename,
            total = fileTotalLines,
            hit   = fileHitLines,
            pct   = pct,
        })

        totalLines = totalLines + fileTotalLines
        hitLines   = hitLines   + fileHitLines
    end

    table.sort(fileReports, function(a, b) return a.pct < b.pct end)

    return {
        files  = fileReports,
        total  = totalLines,
        hit    = hitLines,
        pct    = totalLines > 0 and (hitLines / totalLines * 100) or 0,
    }
end

function Coverage:printReport()
    local r = self:report()
    print("=== Code Coverage Report ===")
    print(string.format("%-35s %6s %6s %6s", "File", "Lines", "Hit", "Coverage"))
    print(string.rep("-", 60))
    for _, f in ipairs(r.files) do
        local bar = string.rep("█", math.floor(f.pct / 5))
        print(string.format("%-35s %6d %6d %5.1f%% %s",
            f.file, f.total, f.hit, f.pct, bar))
    end
    print(string.rep("-", 60))
    print(string.format("%-35s %6d %6d %5.1f%%",
        "TOTAL", r.total, r.hit, r.pct))
    return r.pct >= 80  -- pass if >= 80% coverage
end

function Coverage:toLCOV()
    local lines = {}
    table.insert(lines, "TN:")  -- test name
    for filename, fileLines in pairs(self.files) do
        table.insert(lines, "SF:" .. filename)
        for lineNum, line in pairs(fileLines) do
            if line.executable then
                table.insert(lines, string.format("DA:%d,%d",
                    lineNum, line.hits))
            end
        end
        -- Count lines hit
        local hit = 0
        local total = 0
        for _, line in pairs(fileLines) do
            if line.executable then
                total = total + 1
                if line.hits > 0 then hit = hit + 1 end
            end
        end
        table.insert(lines, string.format("LH:%d", hit))
        table.insert(lines, string.format("LF:%d", total))
        table.insert(lines, "end_of_record")
    end
    return table.concat(lines, "\n")
end

-- Simulate coverage data
local cov = Coverage.new()

cov:markExecutable("src/api.lua",    {10,11,12,15,18,20,25,30,35,40})
cov:markExecutable("src/utils.lua",  {5,6,7,10,12,15,18,20})
cov:markExecutable("src/config.lua", {3,4,5,8,10})

-- Simulate test execution hitting lines
for _, line in ipairs({10,11,12,15,18,20,25,30}) do
    cov:trackLine("src/api.lua", line)
end
for _, line in ipairs({5,6,7,10,12,15}) do
    cov:trackLine("src/utils.lua", line)
end
for _, line in ipairs({3,4,5}) do
    cov:trackLine("src/config.lua", line)
end

local passed = cov:printReport()
print("\nCoverage " .. (passed and "PASSED" or "FAILED") .. " (threshold: 80%)")
```

---

## 70.6 Building Docker Images in CI

```lua
-- ตัวอย่างที่ 6: Docker build script generator
local DockerBuildScript = {}
DockerBuildScript.__index = DockerBuildScript

function DockerBuildScript.new(config)
    local self    = setmetatable({}, DockerBuildScript)
    self.registry = config.registry or "ghcr.io"
    self.org      = config.org
    self.app      = config.app
    self.sha      = config.sha or "$(git rev-parse --short HEAD)"
    self.branch   = config.branch or "$(git rev-parse --abbrev-ref HEAD)"
    return self
end

function DockerBuildScript:imageName()
    return string.format("%s/%s/%s", self.registry, self.org, self.app)
end

function DockerBuildScript:tags()
    local imageName = self:imageName()
    return {
        string.format("%s:%s",   imageName, self.sha),
        string.format("%s:latest", imageName),
    }
end

function DockerBuildScript:generateBuildScript()
    local lines = {}
    table.insert(lines, "#!/bin/bash")
    table.insert(lines, "set -euo pipefail")
    table.insert(lines, "")
    table.insert(lines, "# Docker build script for CI")
    table.insert(lines, "")

    local imageName = self:imageName()
    table.insert(lines, string.format("IMAGE=%s", imageName))
    table.insert(lines, "SHA=$(git rev-parse --short HEAD)")
    table.insert(lines, "BRANCH=$(git rev-parse --abbrev-ref HEAD)")
    table.insert(lines, "DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)")
    table.insert(lines, "")

    table.insert(lines, "# Enable BuildKit")
    table.insert(lines, "export DOCKER_BUILDKIT=1")
    table.insert(lines, "")

    table.insert(lines, "echo \"Building image: ${IMAGE}:${SHA}\"")
    table.insert(lines, "")

    table.insert(lines, "docker build \\")
    table.insert(lines, "  --build-arg BUILD_DATE=\"${DATE}\" \\")
    table.insert(lines, "  --build-arg GIT_COMMIT=\"${SHA}\" \\")
    table.insert(lines, "  --build-arg GIT_BRANCH=\"${BRANCH}\" \\")
    table.insert(lines, "  --cache-from type=registry,ref=${IMAGE}:buildcache \\")
    table.insert(lines, "  --cache-to   type=registry,ref=${IMAGE}:buildcache,mode=max \\")
    table.insert(lines, "  --target production \\")
    table.insert(lines, "  --tag \"${IMAGE}:${SHA}\" \\")
    table.insert(lines, "  --tag \"${IMAGE}:latest\" \\")
    table.insert(lines, "  --file Dockerfile \\")
    table.insert(lines, "  .")
    table.insert(lines, "")

    table.insert(lines, "# Tag with branch name (sanitized)")
    table.insert(lines, 'SAFE_BRANCH=$(echo "${BRANCH}" | sed "s/[^a-zA-Z0-9._-]/-/g")')
    table.insert(lines, 'docker tag "${IMAGE}:${SHA}" "${IMAGE}:${SAFE_BRANCH}"')
    table.insert(lines, "")

    table.insert(lines, "# Scan for vulnerabilities")
    table.insert(lines, 'echo "Scanning image for vulnerabilities..."')
    table.insert(lines, 'trivy image --exit-code 1 --severity HIGH,CRITICAL "${IMAGE}:${SHA}" || {')
    table.insert(lines, '  echo "Security scan failed!"')
    table.insert(lines, '  exit 1')
    table.insert(lines, '}')
    table.insert(lines, "")

    table.insert(lines, "# Push images")
    table.insert(lines, 'echo "Pushing images..."')
    table.insert(lines, 'docker push "${IMAGE}:${SHA}"')
    table.insert(lines, 'docker push "${IMAGE}:latest"')
    table.insert(lines, 'docker push "${IMAGE}:${SAFE_BRANCH}"')
    table.insert(lines, "")

    table.insert(lines, 'echo "Build complete: ${IMAGE}:${SHA}"')

    return table.concat(lines, "\n")
end

local builder = DockerBuildScript.new({
    registry = "ghcr.io",
    org      = "mycompany",
    app      = "lua-api",
})

print("=== Docker Build Script ===")
print(builder:generateBuildScript())
```

---

## 70.7 Deployment Automation

```lua
-- ตัวอย่างที่ 7: Deployment script generator
local function generateDeployScript(config)
    local lines = {}
    table.insert(lines, "#!/bin/bash")
    table.insert(lines, "set -euo pipefail")
    table.insert(lines, "")
    table.insert(lines, "# Automated deployment script")
    table.insert(lines, string.format("# Deploy %s to %s", config.app, config.env))
    table.insert(lines, "")

    table.insert(lines, "APP_NAME=" .. config.app)
    table.insert(lines, "NAMESPACE=" .. config.namespace)
    table.insert(lines, "IMAGE=" .. config.image)
    table.insert(lines, "IMAGE_TAG=${1:-latest}")
    table.insert(lines, "TIMEOUT=" .. (config.timeout or "5m"))
    table.insert(lines, "")

    table.insert(lines, "# Validate inputs")
    table.insert(lines, 'if [[ -z "${IMAGE_TAG}" ]]; then')
    table.insert(lines, '  echo "Usage: $0 <image-tag>"')
    table.insert(lines, '  exit 1')
    table.insert(lines, 'fi')
    table.insert(lines, "")

    table.insert(lines, "# Check cluster connectivity")
    table.insert(lines, 'if ! kubectl cluster-info &>/dev/null; then')
    table.insert(lines, '  echo "ERROR: Cannot connect to Kubernetes cluster"')
    table.insert(lines, '  exit 1')
    table.insert(lines, 'fi')
    table.insert(lines, "")

    table.insert(lines, "# Record current state for rollback")
    table.insert(lines, 'CURRENT_IMAGE=$(kubectl get deployment "${APP_NAME}" \\')
    table.insert(lines, '  -n "${NAMESPACE}" \\')
    table.insert(lines, '  -o jsonpath="{.spec.template.spec.containers[0].image}" 2>/dev/null || echo "")')
    table.insert(lines, 'echo "Current image: ${CURRENT_IMAGE}"')
    table.insert(lines, 'echo "Target image: ${IMAGE}:${IMAGE_TAG}"')
    table.insert(lines, "")

    table.insert(lines, "# Update deployment")
    table.insert(lines, 'echo "Updating deployment..."')
    table.insert(lines, 'kubectl set image deployment/"${APP_NAME}" \\')
    table.insert(lines, '  "${APP_NAME}"="${IMAGE}:${IMAGE_TAG}" \\')
    table.insert(lines, '  -n "${NAMESPACE}"')
    table.insert(lines, "")

    table.insert(lines, "# Add deployment annotation")
    table.insert(lines, 'kubectl annotate deployment "${APP_NAME}" \\')
    table.insert(lines, '  kubernetes.io/change-cause="Deploy ${IMAGE}:${IMAGE_TAG} on $(date -u)" \\')
    table.insert(lines, '  -n "${NAMESPACE}" --overwrite')
    table.insert(lines, "")

    table.insert(lines, "# Wait for rollout")
    table.insert(lines, 'echo "Waiting for rollout (timeout: ${TIMEOUT})..."')
    table.insert(lines, 'if ! kubectl rollout status deployment/"${APP_NAME}" \\')
    table.insert(lines, '  -n "${NAMESPACE}" --timeout="${TIMEOUT}"; then')
    table.insert(lines, '  echo "ERROR: Rollout failed! Rolling back..."')
    table.insert(lines, '  kubectl rollout undo deployment/"${APP_NAME}" -n "${NAMESPACE}"')
    table.insert(lines, '  kubectl rollout status deployment/"${APP_NAME}" -n "${NAMESPACE}"')
    table.insert(lines, '  echo "Rollback complete"')
    table.insert(lines, '  exit 1')
    table.insert(lines, 'fi')
    table.insert(lines, "")

    table.insert(lines, "# Post-deployment health check")
    table.insert(lines, 'echo "Running health checks..."')
    table.insert(lines, "sleep 5")
    table.insert(lines, 'ENDPOINT="' .. (config.healthEndpoint or "http://app/health") .. '"')
    table.insert(lines, 'for i in $(seq 1 10); do')
    table.insert(lines, '  if curl -sf "${ENDPOINT}" > /dev/null 2>&1; then')
    table.insert(lines, '    echo "Health check passed on attempt ${i}"')
    table.insert(lines, '    break')
    table.insert(lines, '  fi')
    table.insert(lines, '  if [[ "${i}" == "10" ]]; then')
    table.insert(lines, '    echo "ERROR: Health check failed after 10 attempts"')
    table.insert(lines, '    exit 1')
    table.insert(lines, '  fi')
    table.insert(lines, '  echo "Attempt ${i}/10 failed, retrying in 5s..."')
    table.insert(lines, '  sleep 5')
    table.insert(lines, 'done')
    table.insert(lines, "")

    table.insert(lines, 'echo "Deployment successful: ${IMAGE}:${IMAGE_TAG}"')

    return table.concat(lines, "\n")
end

print("=== Deployment Script ===")
print(generateDeployScript({
    app            = "lua-api",
    env            = "production",
    namespace      = "production",
    image          = "ghcr.io/mycompany/lua-api",
    timeout        = "10m",
    healthEndpoint = "http://lua-api.production.svc/health",
}))
```

---

## 70.8 Blue-Green Deployment Script

```lua
-- ตัวอย่างที่ 8: Blue-Green deployment automation
local function generateBlueGreenScript()
    return [[
#!/bin/bash
set -euo pipefail

# Blue-Green Deployment Script
# Usage: ./deploy_bluegreen.sh --image <image:tag> --namespace <ns>

APP_NAME="lua-api"
NAMESPACE="production"
IMAGE=""

# Parse arguments
while [[ $# -gt 0 ]]; do
  case $1 in
    --image)     IMAGE="$2";     shift 2 ;;
    --namespace) NAMESPACE="$2"; shift 2 ;;
    *) echo "Unknown arg: $1"; exit 1 ;;
  esac
done

[[ -z "$IMAGE" ]] && { echo "ERROR: --image is required"; exit 1; }

# Determine current and next slot
CURRENT_SLOT=$(kubectl get service "${APP_NAME}" \
  -n "${NAMESPACE}" \
  -o jsonpath='{.spec.selector.slot}' 2>/dev/null || echo "blue")

if [[ "${CURRENT_SLOT}" == "blue" ]]; then
  NEXT_SLOT="green"
else
  NEXT_SLOT="blue"
fi

echo "Current slot: ${CURRENT_SLOT}"
echo "Deploying to: ${NEXT_SLOT}"
echo "Image: ${IMAGE}"

# Deploy to inactive slot
DEPLOY_NAME="${APP_NAME}-${NEXT_SLOT}"

echo "Updating deployment '${DEPLOY_NAME}'..."
kubectl set image deployment/"${DEPLOY_NAME}" \
  "${APP_NAME}"="${IMAGE}" \
  -n "${NAMESPACE}"

# Wait for next slot to be ready
echo "Waiting for ${NEXT_SLOT} slot to be ready..."
kubectl rollout status deployment/"${DEPLOY_NAME}" \
  -n "${NAMESPACE}" --timeout=10m

# Health check the inactive slot directly
INACTIVE_SVC="${APP_NAME}-${NEXT_SLOT}-internal"
echo "Health checking ${NEXT_SLOT} slot..."
for i in $(seq 1 15); do
  if kubectl exec -n "${NAMESPACE}" \
    $(kubectl get pod -n "${NAMESPACE}" -l "app=${DEPLOY_NAME}" \
      -o jsonpath='{.items[0].metadata.name}') \
    -- curl -sf http://localhost:8080/health > /dev/null 2>&1; then
    echo "Health check passed"
    break
  fi
  [[ $i -eq 15 ]] && { echo "Health check FAILED"; exit 1; }
  echo "Attempt $i/15 failed, retrying..."
  sleep 10
done

# Switch traffic to new slot
echo "Switching traffic to ${NEXT_SLOT}..."
kubectl patch service "${APP_NAME}" \
  -n "${NAMESPACE}" \
  --type='json' \
  -p="[{\"op\": \"replace\", \"path\": \"/spec/selector/slot\", \"value\": \"${NEXT_SLOT}\"}]"

echo "Traffic now pointing to ${NEXT_SLOT}"

# Wait a bit, then verify
sleep 5
echo "Verifying deployment..."
CURRENT_POD=$(kubectl get pod \
  -n "${NAMESPACE}" \
  -l "app=${APP_NAME},slot=${NEXT_SLOT}" \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)

echo "Active pod: ${CURRENT_POD}"
echo ""
echo "Blue-Green deployment complete!"
echo "Old slot (${CURRENT_SLOT}) is kept for quick rollback."
echo "To rollback: kubectl patch service ${APP_NAME} -n ${NAMESPACE} \\"
echo "  --type='json' -p='[{\"op\": \"replace\", \"path\": \"/spec/selector/slot\", \"value\": \"${CURRENT_SLOT}\"}]'"
]]
end

print("=== Blue-Green Deployment Script ===")
print(generateBlueGreenScript())
```

---

## 70.9 Rollback Automation

```lua
-- ตัวอย่างที่ 9: Rollback script
local function generateRollbackScript()
    return [[
#!/bin/bash
set -euo pipefail

# Automated Rollback Script
# Triggers on: failed health checks, error rate spike, manual trigger

APP_NAME="${1:-lua-api}"
NAMESPACE="${2:-production}"
REASON="${3:-manual}"

echo "=== Starting Rollback ==="
echo "App:       ${APP_NAME}"
echo "Namespace: ${NAMESPACE}"
echo "Reason:    ${REASON}"
echo ""

# Check if deployment exists
if ! kubectl get deployment "${APP_NAME}" -n "${NAMESPACE}" &>/dev/null; then
  echo "ERROR: Deployment '${APP_NAME}' not found in namespace '${NAMESPACE}'"
  exit 1
fi

# Show rollout history
echo "Rollout history:"
kubectl rollout history deployment/"${APP_NAME}" -n "${NAMESPACE}"
echo ""

# Get previous revision
PREV_REVISION=$(kubectl rollout history deployment/"${APP_NAME}" \
  -n "${NAMESPACE}" \
  -o jsonpath='{range .items[*]}{.revision}{"\n"}{end}' 2>/dev/null | \
  sort -n | tail -2 | head -1)

echo "Rolling back to revision: ${PREV_REVISION}"

# Perform rollback
kubectl rollout undo deployment/"${APP_NAME}" \
  -n "${NAMESPACE}"

# Wait for rollback to complete
echo "Waiting for rollback to complete..."
kubectl rollout status deployment/"${APP_NAME}" \
  -n "${NAMESPACE}" --timeout=5m

# Verify health
echo "Verifying health after rollback..."
sleep 10

HEALTH_URL="http://${APP_NAME}.${NAMESPACE}.svc/health"
if ! curl -sf "${HEALTH_URL}" > /dev/null 2>&1; then
  echo "WARNING: Health check still failing after rollback!"
  echo "Manual intervention may be required."
  exit 1
fi

# Record rollback event
echo "Recording rollback event..."
kubectl annotate deployment "${APP_NAME}" \
  -n "${NAMESPACE}" \
  rollback.kubernetes.io/time="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  rollback.kubernetes.io/reason="${REASON}" \
  --overwrite

echo ""
echo "=== Rollback Complete ==="
echo "Deployment has been rolled back successfully."
echo "Please investigate the cause: ${REASON}"
]]
end

print("=== Rollback Script ===")
print(generateRollbackScript())
```

---

## 70.10 Environment Promotion

```lua
-- ตัวอย่างที่ 10: Environment promotion pipeline
local PromotionPipeline = {}
PromotionPipeline.__index = PromotionPipeline

function PromotionPipeline.new()
    local self        = setmetatable({}, PromotionPipeline)
    self.environments = {}
    self.gatingChecks = {}
    return self
end

function PromotionPipeline:addEnvironment(name, config)
    table.insert(self.environments, {
        name     = name,
        cluster  = config.cluster,
        ns       = config.namespace or name,
        gates    = config.gates or {},   -- checks before promoting
        autoPromote = config.autoPromote ~= false,
    })
end

function PromotionPipeline:addGate(name, fn)
    self.gatingChecks[name] = fn
end

function PromotionPipeline:canPromote(env, image)
    local results = {}
    for _, gateName in ipairs(env.gates) do
        local check = self.gatingChecks[gateName]
        if not check then
            print(string.format("  WARNING: Gate '%s' not found", gateName))
        else
            local ok, reason = check(env, image)
            results[gateName] = {ok = ok, reason = reason}
            if not ok then
                print(string.format("  GATE FAILED [%s]: %s", gateName, reason))
                return false
            end
            print(string.format("  GATE PASSED [%s]", gateName))
        end
    end
    return true
end

function PromotionPipeline:promote(image, startFrom)
    print(string.format("=== Promotion Pipeline: %s ===", image))
    local started = startFrom == nil

    for _, env in ipairs(self.environments) do
        if not started then
            if env.name == startFrom then started = true end
        end

        if started then
            print(string.format("\n[%s] Environment: %s", env.name:upper(), env.name))

            -- Run gating checks
            if #env.gates > 0 then
                print("  Running gates...")
                if not self:canPromote(env, image) then
                    print(string.format("  Promotion blocked at %s", env.name))
                    return false, env.name
                end
            end

            -- Deploy
            print(string.format("  Deploying %s to %s/%s...", image, env.cluster, env.ns))
            -- In real script: kubectl set image deployment/...
            print(string.format("  Deployed successfully to %s", env.name))

            if not env.autoPromote and env ~= self.environments[#self.environments] then
                print("  Waiting for manual approval to continue...")
                -- In real pipeline: wait for approval
            end
        end
    end

    print("\nPromotion pipeline complete!")
    return true, nil
end

-- Setup promotion pipeline
local promotion = PromotionPipeline.new()

promotion:addEnvironment("dev", {
    cluster  = "dev-cluster",
    namespace = "dev",
    gates    = {},
    autoPromote = true,
})

promotion:addEnvironment("staging", {
    cluster  = "staging-cluster",
    namespace = "staging",
    gates    = {"tests_passed", "security_scan"},
    autoPromote = true,
})

promotion:addEnvironment("production", {
    cluster  = "prod-cluster",
    namespace = "production",
    gates    = {"staging_healthy", "approval_required"},
    autoPromote = false,
})

-- Define gates
promotion:addGate("tests_passed", function(env, image)
    -- Check CI test results
    local passed = math.random() > 0.1  -- 90% pass rate (demo)
    return passed, passed and "All tests passed" or "Test failures detected"
end)

promotion:addGate("security_scan", function(env, image)
    -- Check Trivy scan results
    return true, "No critical vulnerabilities found"
end)

promotion:addGate("staging_healthy", function(env, image)
    -- Check staging environment health
    return true, "Staging environment is healthy"
end)

promotion:addGate("approval_required", function(env, image)
    -- Manual approval gate
    -- In real CI: check for approval label/comment
    return true, "Deployment approved by @reviewer"
end)

math.randomseed(99)
promotion:promote("ghcr.io/mycompany/lua-api:v2.1.0")
```

---

## 70.11 Secrets Management in CI

```lua
-- ตัวอย่างที่ 11: Secrets management patterns
local SecretsManager = {}
SecretsManager.__index = SecretsManager

function SecretsManager.new()
    local self   = setmetatable({}, SecretsManager)
    self.secrets = {}
    return self
end

-- Secret rotation policy generator
function SecretsManager:generateRotationPolicy(secrets)
    local lines = {}
    table.insert(lines, "# Secrets Rotation Policy")
    table.insert(lines, "# Review and rotate these secrets regularly")
    table.insert(lines, "")
    table.insert(lines, "| Secret | Environment | Rotation Period | Last Rotated |")
    table.insert(lines, "|--------|-------------|-----------------|--------------|")
    for _, s in ipairs(secrets) do
        table.insert(lines, string.format("| %-20s | %-10s | %-15s | %-20s |",
            s.name, s.env, s.rotation, s.lastRotated or "Unknown"))
    end
    return table.concat(lines, "\n")
end

-- GitHub Actions secrets usage guide
function SecretsManager:generateActionsSecrets(config)
    local lines = {}
    table.insert(lines, "# GitHub Actions Secrets Configuration")
    table.insert(lines, "# Set these in: Settings > Secrets and variables > Actions")
    table.insert(lines, "")

    local categories = {
        {name = "Container Registry", secrets = {
            "REGISTRY_TOKEN    - Personal access token for container registry",
            "REGISTRY_USERNAME - Registry username",
        }},
        {name = "Kubernetes", secrets = {
            "STAGING_KUBECONFIG    - Kubeconfig for staging cluster (base64)",
            "PRODUCTION_KUBECONFIG - Kubeconfig for production cluster (base64)",
        }},
        {name = "Database", secrets = {
            "STAGING_DB_URL     - PostgreSQL connection string for staging",
            "PRODUCTION_DB_URL  - PostgreSQL connection string for production",
        }},
        {name = "Notifications", secrets = {
            "SLACK_WEBHOOK      - Slack webhook URL for deployment notifications",
            "PAGERDUTY_KEY      - PagerDuty integration key for alerts",
        }},
        {name = "External APIs", secrets = {
            "PAYMENT_API_KEY    - Payment gateway API key",
            "SMTP_PASSWORD      - SMTP server password",
        }},
    }

    for _, cat in ipairs(categories) do
        table.insert(lines, "## " .. cat.name)
        for _, s in ipairs(cat.secrets) do
            table.insert(lines, "  " .. s)
        end
        table.insert(lines, "")
    end

    return table.concat(lines, "\n")
end

-- Vault integration simulation
function SecretsManager:generateVaultConfig(config)
    return string.format([[
# HashiCorp Vault Integration
# vault.hcl

vault {
  address = "%s"
  token   = env("VAULT_TOKEN")
}

# Secret paths
secret "database" {
  path = "secret/data/%s/database"
}

secret "api_keys" {
  path = "secret/data/%s/api-keys"
}

# Dynamic secrets (auto-rotated)
dynamic_secret "db_creds" {
  path = "database/creds/my-role"
  renew_increment = "1h"
}
]], config.address or "https://vault.example.com",
    config.env or "production",
    config.env or "production")
end

local sm = SecretsManager.new()

print("=== Secrets Management ===")
print(sm:generateActionsSecrets({}))
print(sm:generateVaultConfig({address = "https://vault.company.com", env = "production"}))
print(sm:generateRotationPolicy({
    {name = "DATABASE_PASSWORD",  env = "production", rotation = "90 days",  lastRotated = "2024-01-01"},
    {name = "API_SECRET_KEY",     env = "production", rotation = "30 days",  lastRotated = "2024-03-01"},
    {name = "REGISTRY_TOKEN",     env = "all",        rotation = "180 days", lastRotated = "2023-12-01"},
    {name = "KUBECONFIG",         env = "production", rotation = "365 days", lastRotated = "2024-01-15"},
}))
```

---

## 70.12 Caching Dependencies in CI

```lua
-- ตัวอย่างที่ 12: Dependency caching strategies
local function generateCacheConfig()
    local configs = {}

    -- LuaRocks cache
    table.insert(configs, {
        name    = "LuaRocks Packages",
        config  = [[
    - name: Cache LuaRocks packages
      uses: actions/cache@v3
      with:
        path: |
          ~/.luarocks
          lua_modules
        key: ${{ runner.os }}-luarocks-${{ hashFiles('**/*.rockspec') }}
        restore-keys: |
          ${{ runner.os }}-luarocks-

    - name: Install dependencies
      run: |
        luarocks install --tree lua_modules]],
    })

    -- Docker layer cache
    table.insert(configs, {
        name   = "Docker Layer Cache",
        config = [[
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build with cache
      uses: docker/build-push-action@v5
      with:
        cache-from: type=gha
        cache-to: type=gha,mode=max
        # Alternative: registry cache
        # cache-from: type=registry,ref=${{ env.IMAGE }}:buildcache
        # cache-to: type=registry,ref=${{ env.IMAGE }}:buildcache,mode=max]],
    })

    -- APT packages cache
    table.insert(configs, {
        name   = "APT Package Cache",
        config = [[
    - name: Cache APT packages
      uses: awalsh128/cache-apt-pkgs-action@v1
      with:
        packages: lua5.4 luarocks libssl-dev
        version: 1.0]],
    })

    return configs
end

print("=== CI Dependency Caching ===")
for _, c in ipairs(generateCacheConfig()) do
    print(string.format("\n### %s ###", c.name))
    print(c.config)
end
```

---

## 70.13 Notification on Failure

```lua
-- ตัวอย่างที่ 13: CI failure notification system
local NotificationSystem = {}
NotificationSystem.__index = NotificationSystem

function NotificationSystem.new()
    local self     = setmetatable({}, NotificationSystem)
    self.channels  = {}
    return self
end

function NotificationSystem:addChannel(name, config)
    self.channels[name] = config
end

function NotificationSystem:notify(event)
    for channelName, channel in pairs(self.channels) do
        if self:shouldNotify(channel, event) then
            local message = self:formatMessage(channel.format, event)
            print(string.format("[%s] Sending notification: %s",
                channelName, message:sub(1, 80)))
        end
    end
end

function NotificationSystem:shouldNotify(channel, event)
    if not channel.events then return true end
    for _, e in ipairs(channel.events) do
        if e == event.type then return true end
    end
    return false
end

function NotificationSystem:formatMessage(format, event)
    if format == "slack" then
        return string.format(
            '{"text": "%s: %s", "blocks": [{"type": "section", "text": {"type": "mrkdwn", "text": "*%s*\\n%s\\nCommit: `%s`"}}]}',
            event.status == "success" and "Deployment succeeded" or "Deployment FAILED",
            event.app,
            event.app .. " - " .. event.status:upper(),
            event.message or "",
            event.commit or "unknown")
    elseif format == "email" then
        return string.format(
            "Subject: [CI] %s %s - %s\n\nDeployment %s for %s\n\nCommit: %s\nMessage: %s",
            event.status:upper(), event.app, event.env,
            event.status, event.app,
            event.commit or "unknown",
            event.message or "")
    elseif format == "pagerduty" then
        return string.format(
            '{"event_action": "%s", "payload": {"summary": "%s", "severity": "%s"}}',
            event.status == "failed" and "trigger" or "resolve",
            event.app .. " deployment " .. event.status,
            event.status == "failed" and "critical" or "info")
    end
    return tostring(event)
end

-- GitHub Actions notification step generator
function NotificationSystem:generateActionsSteps(config)
    local lines = {}

    -- Slack notification
    table.insert(lines, "    # Notification steps (add to end of deploy job)")
    table.insert(lines, "")
    table.insert(lines, "    - name: Notify Slack on success")
    table.insert(lines, "      if: success()")
    table.insert(lines, "      uses: slackapi/slack-github-action@v1")
    table.insert(lines, "      with:")
    table.insert(lines, "        payload: |")
    table.insert(lines, '          {')
    table.insert(lines, '            "text": "Deployment succeeded!",')
    table.insert(lines, '            "blocks": [{')
    table.insert(lines, '              "type": "section",')
    table.insert(lines, '              "text": {')
    table.insert(lines, '                "type": "mrkdwn",')
    table.insert(lines, '                "text": "*${{ github.repository }}* deployed to production\\nCommit: `${{ github.sha }}`\\nAuthor: ${{ github.actor }}"')
    table.insert(lines, '              }')
    table.insert(lines, '            }]')
    table.insert(lines, '          }')
    table.insert(lines, "      env:")
    table.insert(lines, "        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}")
    table.insert(lines, "")

    table.insert(lines, "    - name: Notify Slack on failure")
    table.insert(lines, "      if: failure()")
    table.insert(lines, "      uses: slackapi/slack-github-action@v1")
    table.insert(lines, "      with:")
    table.insert(lines, "        payload: |")
    table.insert(lines, '          {')
    table.insert(lines, '            "text": "Deployment FAILED!",')
    table.insert(lines, '            "blocks": [{')
    table.insert(lines, '              "type": "section",')
    table.insert(lines, '              "text": {')
    table.insert(lines, '                "type": "mrkdwn",')
    table.insert(lines, '                "text": "*FAILED* ${{ github.repository }}\\nCommit: `${{ github.sha }}`\\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View logs>"')
    table.insert(lines, '              }')
    table.insert(lines, '            }]')
    table.insert(lines, '          }')
    table.insert(lines, "      env:")
    table.insert(lines, "        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}")

    return table.concat(lines, "\n")
end

local ns = NotificationSystem.new()
ns:addChannel("slack", {
    format = "slack",
    events = {"success", "failed"},
})
ns:addChannel("email", {
    format = "email",
    events = {"failed"},
})
ns:addChannel("pagerduty", {
    format = "pagerduty",
    events = {"failed"},
})

print("=== CI Notification System ===")
ns:notify({
    type    = "success",
    app     = "lua-api",
    env     = "production",
    commit  = "abc1234",
    status  = "success",
    message = "v2.1.0 deployed",
})

ns:notify({
    type    = "failed",
    app     = "lua-api",
    env     = "staging",
    commit  = "def5678",
    status  = "failed",
    message = "Health check timed out",
})

print("\n--- Notification Steps for GitHub Actions ---")
print(ns:generateActionsSteps({}))
```

---

## 70.14 Complete CI/CD Pipeline YAML

```lua
-- ตัวอย่างที่ 14: Complete pipeline สำหรับ OpenResty project
local function generateOpenRestyPipeline()
    return [[
name: OpenResty Lua CI/CD

on:
  push:
    branches: [main, develop]
    paths: ['**.lua', 'nginx/**', 'Dockerfile', '.github/**']
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install luacheck
        run: |
          sudo apt-get install -y luarocks
          sudo luarocks install luacheck

      - name: Lint Lua code
        run: luacheck lua/ --config .luacheckrc

      - name: Check nginx config
        run: docker run --rm -v $PWD/nginx:/etc/nginx nginx:alpine nginx -t

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: lint
    services:
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup Lua
        uses: leafo/gh-actions-lua@v10
        with:
          luaVersion: "luajit-2.1"

      - name: Setup LuaRocks
        uses: leafo/gh-actions-luarocks@v4

      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: lua_modules
          key: ${{ runner.os }}-luarocks-${{ hashFiles('*.rockspec') }}

      - name: Install dependencies
        run: luarocks install --tree lua_modules

      - name: Run unit tests
        run: |
          export LUA_PATH="./lua/?.lua;./lua/?/init.lua;./lua_modules/share/lua/5.1/?.lua;;"
          lua test/unit/run.lua

      - name: Run integration tests
        run: lua test/integration/run.lua
        env:
          REDIS_URL: redis://localhost:6379
          APP_ENV: test

      - name: Check coverage
        run: lua test/coverage.lua
        continue-on-error: true

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: filesystem
          scan-ref: .
          severity: CRITICAL,HIGH
          exit-code: 1

  build:
    name: Build & Push
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    if: github.event_name != 'pull_request'
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    name: Deploy Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          kubectl set image deployment/lua-api \
            lua-api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/lua-api -n staging --timeout=5m
        env:
          KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}

      - name: Run smoke tests
        run: lua test/smoke_test.lua ${{ secrets.STAGING_URL }}

  deploy-production:
    name: Deploy Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Blue-Green Deploy
        run: ./scripts/deploy_bluegreen.sh
        env:
          KUBECONFIG: ${{ secrets.PROD_KUBECONFIG }}
          IMAGE: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}

      - name: Verify deployment
        run: lua test/smoke_test.lua https://api.example.com

      - name: Notify on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text":"Deployed ${{ github.sha }} to production"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text":"FAILED: Production deployment for ${{ github.sha }}"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
]]
end

print("=== Complete CI/CD Pipeline for OpenResty ===")
print(generateOpenRestyPipeline())
```

---

## 70.15 สรุป CI/CD Best Practices

```lua
-- ตัวอย่างที่ 15: CI/CD Best Practices Checklist
local CICDBestPractices = {
    {
        category = "Continuous Integration",
        items    = {
            "Run tests on every commit/PR",
            "Keep CI fast (< 10 minutes for PR feedback)",
            "Fail fast: lint before tests, tests before build",
            "Run tests in parallel where possible",
            "Cache dependencies to speed up runs",
            "Use matrix strategy for multiple Lua versions",
            "Block merging if CI fails",
        }
    },
    {
        category = "Code Quality",
        items    = {
            "Enforce code style with luacheck + stylua",
            "Maintain > 80% code coverage",
            "Security scanning with Trivy/snyk",
            "Dependency vulnerability checks",
            "Static analysis for common bugs",
        }
    },
    {
        category = "Docker / Container",
        items    = {
            "Use specific image tags (not :latest)",
            "Multi-stage builds for minimal images",
            "Scan images before pushing",
            "Cache Docker layers in CI",
            "Never store secrets in images",
            "Run as non-root user",
        }
    },
    {
        category = "Deployment Strategy",
        items    = {
            "Use staging environment before production",
            "Blue-green or rolling deployments",
            "Automated smoke tests after deploy",
            "Automatic rollback on health check failure",
            "Keep deployment artifacts immutable",
            "Tag images with commit SHA",
        }
    },
    {
        category = "Secrets & Security",
        items    = {
            "Never commit secrets to repository",
            "Use GitHub Actions secrets for sensitive values",
            "Rotate secrets regularly",
            "Use OIDC for cloud provider auth (no long-lived keys)",
            "Least privilege for deployment credentials",
        }
    },
    {
        category = "Observability",
        items    = {
            "Notify team on deployment success/failure",
            "Link deployment to commit/PR",
            "Track deployment frequency metrics",
            "Monitor error rate after deploy",
            "Set up alerts for failed CI runs",
        }
    },
}

print("=== CI/CD Best Practices for Lua Applications ===")
for _, cat in ipairs(CICDBestPractices) do
    print(string.format("\n[%s]", cat.category))
    for i, item in ipairs(cat.items) do
        print(string.format("  %d. %s", i, item))
    end
end

-- Generate checklist
print("\n\n=== Pre-deployment Checklist ===")
local checklist = {
    "[x] All CI checks pass",
    "[x] Code review approved",
    "[x] No critical security vulnerabilities",
    "[x] Tests pass (unit + integration)",
    "[x] Coverage >= 80%",
    "[ ] Staged in staging environment",
    "[ ] Smoke tests pass on staging",
    "[ ] Rollback plan documented",
    "[ ] Team notified of deployment",
    "[ ] Monitoring/alerts configured",
}

for _, item in ipairs(checklist) do
    print("  " .. item)
end
```

---

## สรุปบทที่ 70

ในบทนี้เราได้เรียนรู้:

1. **CI/CD Concepts** - Pipeline stages, jobs, gates
2. **GitHub Actions** - Workflow YAML สำหรับ Lua projects
3. **Test Running** - Test runner with assertions
4. **Luacheck** - Configuration และ linting
5. **Code Coverage** - LCOV format, threshold checks
6. **Docker Build** - Build scripts, caching, tagging
7. **Deployment Automation** - kubectl deployment scripts
8. **Blue-Green** - Zero-downtime deployment
9. **Rollback** - Automatic rollback on failure
10. **Environment Promotion** - dev → staging → production
11. **Secrets Management** - GitHub secrets, Vault integration
12. **Dependency Caching** - LuaRocks, Docker layer caches
13. **Notifications** - Slack, email, PagerDuty
14. **Complete Pipeline** - Production-ready OpenResty CI/CD
15. **Best Practices** - CI/CD checklist

---

*จบบทที่ 70 - CI/CD Pipeline*
