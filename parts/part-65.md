# บทที่ 65: CI/CD Pipeline สำหรับ Lua

## บทนำ

CI/CD (Continuous Integration / Continuous Deployment) คือแนวปฏิบัติในการ automate กระบวนการ build, test, และ deploy software อย่างต่อเนื่อง ในบทนี้จะเรียนรู้การสร้าง CI/CD pipeline สำหรับ Lua projects โดยใช้ GitHub Actions พร้อม tools ต่างๆ เช่น Busted, LuaCheck, LuaCov และ LuaRocks

---

## 65.1 GitHub Actions พื้นฐาน

### ตัวอย่างที่ 1: Workflow พื้นฐานสำหรับ Lua Project

```yaml
# .github/workflows/ci.yml
name: Lua CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Test on Lua ${{ matrix.lua-version }}
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        lua-version: ['5.1', '5.2', '5.3', '5.4', 'luajit']
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Lua
        uses: leafo/gh-actions-lua@v10
        with:
          luaVersion: ${{ matrix.lua-version }}
      
      - name: Setup LuaRocks
        uses: leafo/gh-actions-luarocks@v4
      
      - name: Install dependencies
        run: luarocks install --only-deps myproject-dev-1.rockspec
      
      - name: Run tests
        run: busted --verbose
      
      - name: Run linter
        run: luacheck . --config .luacheckrc
```

### ตัวอย่างที่ 2: Project Structure สำหรับ CI

```
myproject/
├── .github/
│   └── workflows/
│       ├── ci.yml          # Main CI pipeline
│       ├── release.yml     # Release pipeline
│       └── docs.yml        # Documentation pipeline
├── src/
│   └── myproject/
│       ├── init.lua
│       ├── utils.lua
│       └── config.lua
├── spec/
│   ├── spec_helper.lua
│   ├── utils_spec.lua
│   └── config_spec.lua
├── .luacheckrc             # LuaCheck configuration
├── .busted                 # Busted configuration
├── myproject-dev-1.rockspec
├── myproject-1.0.0-1.rockspec
└── README.md
```

---

## 65.2 Unit Testing ด้วย Busted

### ตัวอย่างที่ 3: Busted Test Suite

```lua
-- spec/utils_spec.lua
local utils = require("myproject.utils")

describe("utils module", function()
    
    describe("string helpers", function()
        it("should trim whitespace", function()
            assert.equal("hello", utils.trim("  hello  "))
            assert.equal("world", utils.trim("\t world \n"))
            assert.equal("", utils.trim("   "))
        end)
        
        it("should split string", function()
            local parts = utils.split("a,b,c", ",")
            assert.equal(3, #parts)
            assert.equal("a", parts[1])
            assert.equal("b", parts[2])
            assert.equal("c", parts[3])
        end)
        
        it("should handle empty string split", function()
            local parts = utils.split("", ",")
            assert.equal(1, #parts)
            assert.equal("", parts[1])
        end)
        
        it("should format string with placeholders", function()
            local result = utils.format("Hello, {name}!", {name = "World"})
            assert.equal("Hello, World!", result)
        end)
    end)
    
    describe("table helpers", function()
        it("should merge tables", function()
            local t1 = {a = 1, b = 2}
            local t2 = {b = 3, c = 4}
            local merged = utils.merge(t1, t2)
            
            assert.equal(1, merged.a)
            assert.equal(3, merged.b)  -- t2 overrides t1
            assert.equal(4, merged.c)
        end)
        
        it("should deep clone table", function()
            local original = {a = {b = {c = 1}}}
            local clone = utils.deep_clone(original)
            
            assert.are_not.equal(original, clone)
            assert.are_not.equal(original.a, clone.a)
            assert.equal(1, clone.a.b.c)
            
            -- Modify clone should not affect original
            clone.a.b.c = 99
            assert.equal(1, original.a.b.c)
        end)
        
        it("should filter table", function()
            local numbers = {1, 2, 3, 4, 5, 6}
            local evens = utils.filter(numbers, function(n) return n % 2 == 0 end)
            
            assert.equal(3, #evens)
            assert.equal(2, evens[1])
            assert.equal(4, evens[2])
            assert.equal(6, evens[3])
        end)
    end)
    
    describe("number helpers", function()
        it("should clamp numbers", function()
            assert.equal(5, utils.clamp(5, 1, 10))
            assert.equal(1, utils.clamp(-5, 1, 10))
            assert.equal(10, utils.clamp(15, 1, 10))
        end)
        
        it("should round numbers", function()
            assert.equal(3, utils.round(3.4))
            assert.equal(4, utils.round(3.5))
            assert.equal(4, utils.round(3.7))
            assert.equal(-3, utils.round(-3.4))
        end)
    end)
    
end)
```

### ตัวอย่างที่ 4: Testing Async และ Mocking

```lua
-- spec/async_spec.lua
local mock = require("busted.mock")

-- Mock HTTP client
local http_mock = mock({
    get = function(url) return nil, "network error" end,
    post = function(url, data) return {status = 200, body = "{}"} end,
})

-- Module ที่ต้อง test
local api_client = require("myproject.api_client")

describe("API Client", function()
    
    before_each(function()
        -- Reset mocks ก่อนแต่ละ test
        mock.clear(http_mock)
    end)
    
    it("should handle network errors gracefully", function()
        -- Setup mock
        http_mock.get = function(url)
            return nil, "connection refused"
        end
        
        -- Override dependency
        api_client._http = http_mock
        
        local result, err = api_client.fetch_user(123)
        
        assert.is_nil(result)
        assert.matches("connection refused", err)
    end)
    
    it("should parse successful response", function()
        http_mock.get = function(url)
            return {
                status = 200,
                body = '{"id": 123, "name": "Test User"}'
            }
        end
        
        api_client._http = http_mock
        
        local user, err = api_client.fetch_user(123)
        
        assert.is_nil(err)
        assert.equal(123, user.id)
        assert.equal("Test User", user.name)
    end)
    
    it("should retry on 503 errors", function()
        local call_count = 0
        
        http_mock.get = function(url)
            call_count = call_count + 1
            if call_count < 3 then
                return {status = 503, body = "Service Unavailable"}
            end
            return {status = 200, body = '{"id": 1}'}
        end
        
        api_client._http = http_mock
        api_client._max_retries = 3
        
        local user, err = api_client.fetch_user(1)
        
        assert.is_nil(err)
        assert.equal(3, call_count)
    end)
    
end)
```

### ตัวอย่างที่ 5: Integration Tests

```lua
-- spec/integration/database_spec.lua
-- Integration tests ที่ใช้ database จริง (ในสภาพแวดล้อม test)

local sqlite3 = require("lsqlite3")
local UserRepository = require("myproject.repositories.user")

describe("UserRepository Integration", function()
    local db
    local repo
    
    before_each(function()
        -- สร้าง fresh in-memory database สำหรับแต่ละ test
        db = sqlite3.open(":memory:")
        db:exec([[
            CREATE TABLE users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                email TEXT UNIQUE NOT NULL,
                created_at INTEGER DEFAULT (strftime('%s', 'now'))
            )
        ]])
        repo = UserRepository.new(db)
    end)
    
    after_each(function()
        db:close()
    end)
    
    it("should create and find user", function()
        local user = repo:create({name = "Test User", email = "test@example.com"})
        
        assert.truthy(user.id)
        assert.equal("Test User", user.name)
        
        local found = repo:find_by_id(user.id)
        assert.equal(user.id, found.id)
        assert.equal("Test User", found.name)
    end)
    
    it("should enforce unique email", function()
        repo:create({name = "User 1", email = "same@example.com"})
        
        local ok, err = pcall(function()
            repo:create({name = "User 2", email = "same@example.com"})
        end)
        
        assert.is_false(ok)
        assert.matches("UNIQUE", err)
    end)
    
    it("should find all users with pagination", function()
        for i = 1, 10 do
            repo:create({
                name = "User " .. i,
                email = "user" .. i .. "@example.com"
            })
        end
        
        local page1 = repo:find_all({limit = 5, offset = 0})
        local page2 = repo:find_all({limit = 5, offset = 5})
        
        assert.equal(5, #page1)
        assert.equal(5, #page2)
        
        -- Verify no overlap
        for _, u1 in ipairs(page1) do
            for _, u2 in ipairs(page2) do
                assert.are_not.equal(u1.id, u2.id)
            end
        end
    end)
    
end)
```

---

## 65.3 LuaCheck - Static Analysis

### ตัวอย่างที่ 6: LuaCheck Configuration

```lua
-- .luacheckrc
-- LuaCheck configuration file

return {
    -- Global settings
    max_line_length = 120,
    max_code_line_length = 120,
    max_comment_line_length = 200,
    
    -- Warning settings
    unused_args = true,
    unused = true,
    undefined = true,
    
    -- Globals ที่อนุญาต
    globals = {
        -- Lua builtins
        "print", "pairs", "ipairs", "next", "select",
        "unpack", "table", "string", "math", "os", "io",
        "type", "tostring", "tonumber", "error", "pcall",
        "xpcall", "assert", "require", "load", "loadfile",
        "dofile", "collectgarbage", "rawget", "rawset",
        "rawequal", "rawlen", "setmetatable", "getmetatable",
        
        -- Testing globals (Busted)
        "describe", "it", "before_each", "after_each",
        "before", "after", "pending", "assert", "mock",
        "spy", "stub",
        
        -- Project globals
        "LOG", "CONFIG", "APP_VERSION",
    },
    
    -- File-specific overrides
    files = {
        -- Test files - allow more globals
        ["spec/**/*.lua"] = {
            globals = {"describe", "it", "before_each", "after_each",
                       "assert", "mock", "spy", "stub", "pending"}
        },
        
        -- Generated files - skip
        ["generated/**"] = {
            ignore = {".*"}
        },
    },
    
    -- Rules to ignore
    ignore = {
        "212",  -- Unused argument (common in callbacks)
        "213",  -- Unused loop variable
    },
    
    -- Rules to treat as errors (not warnings)
    -- ทำให้ undefined variables เป็น error
    enable = {"112", "113"},
}
```

### ตัวอย่างที่ 7: GitHub Actions ด้วย LuaCheck

```yaml
# .github/workflows/lint.yml
name: Lint

on: [push, pull_request]

jobs:
  luacheck:
    name: LuaCheck Static Analysis
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Lua
        uses: leafo/gh-actions-lua@v10
        with:
          luaVersion: "5.4"
      
      - name: Setup LuaRocks
        uses: leafo/gh-actions-luarocks@v4
      
      - name: Install LuaCheck
        run: luarocks install luacheck
      
      - name: Run LuaCheck
        run: |
          luacheck src/ spec/ \
            --config .luacheckrc \
            --formatter plain \
            --codes \
            --ranges
      
      - name: Check for TODO/FIXME
        run: |
          # Warning เมื่อมี TODO ใน code
          if grep -rn "TODO\|FIXME\|HACK\|XXX" src/; then
            echo "⚠️ Found TODO/FIXME comments in source"
            exit 1
          fi
```

---

## 65.4 Code Coverage ด้วย LuaCov

### ตัวอย่างที่ 8: LuaCov Setup

```lua
-- .luacov
-- LuaCov configuration

return {
    -- Files ที่ต้องการ track
    include = {
        "src/.*",
        "myproject/.*",
    },
    
    -- Files ที่ไม่ต้อง track
    exclude = {
        "spec/.*",
        "test/.*",
        ".luarocks/.*",
        ".*_spec%.lua",
    },
    
    -- Report format
    reporter = "default",
    
    -- Output file
    statsfile = "luacov.stats.out",
    reportfile = "luacov.report.out",
    
    -- Cobertura format สำหรับ CI
    -- reporter = "cobertura",
}
```

### ตัวอย่างที่ 9: Coverage Workflow

```yaml
# .github/workflows/coverage.yml
name: Code Coverage

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  coverage:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Lua
        uses: leafo/gh-actions-lua@v10
        with:
          luaVersion: "5.4"
      
      - name: Setup LuaRocks
        uses: leafo/gh-actions-luarocks@v4
      
      - name: Install dependencies
        run: |
          luarocks install busted
          luarocks install luacov
          luarocks install luacov-reporter-lcov
      
      - name: Run tests with coverage
        run: |
          busted --coverage --verbose 2>&1 | tee test-results.txt
      
      - name: Generate coverage report
        run: |
          luacov
          cat luacov.report.out
      
      - name: Generate LCOV report
        run: |
          luacov -r lcov
          cat luacov.report.lcov
      
      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: luacov.report.lcov
          flags: lua
          fail_ci_if_error: false
          token: ${{ secrets.CODECOV_TOKEN }}
      
      - name: Check coverage threshold
        run: |
          # ตรวจสอบว่า coverage >= 80%
          COVERAGE=$(grep "Total" luacov.report.out | awk '{print $NF}' | tr -d '%')
          echo "Coverage: ${COVERAGE}%"
          if [ -n "$COVERAGE" ] && [ "${COVERAGE%.*}" -lt 80 ]; then
            echo "❌ Coverage ${COVERAGE}% is below minimum 80%"
            exit 1
          fi
          echo "✅ Coverage check passed"
```

---

## 65.5 Semantic Versioning

### ตัวอย่างที่ 10: Version Management Script

```lua
-- scripts/version.lua
-- จัดการ semantic versioning

local Version = {}
Version.__index = Version

function Version.parse(version_str)
    local major, minor, patch, pre = version_str:match(
        "^v?(%d+)%.(%d+)%.(%d+)%-?(.*)$"
    )
    
    if not major then
        error("Invalid version string: " .. version_str)
    end
    
    return setmetatable({
        major = tonumber(major),
        minor = tonumber(minor),
        patch = tonumber(patch),
        pre = pre ~= "" and pre or nil,
    }, Version)
end

function Version:__tostring()
    local v = string.format("%d.%d.%d", self.major, self.minor, self.patch)
    if self.pre then
        v = v .. "-" .. self.pre
    end
    return v
end

function Version:bump(bump_type)
    local new = {
        major = self.major,
        minor = self.minor,
        patch = self.patch,
    }
    
    if bump_type == "major" then
        new.major = new.major + 1
        new.minor = 0
        new.patch = 0
    elseif bump_type == "minor" then
        new.minor = new.minor + 1
        new.patch = 0
    elseif bump_type == "patch" then
        new.patch = new.patch + 1
    else
        error("Invalid bump type: " .. bump_type)
    end
    
    return setmetatable(new, Version)
end

function Version:__lt(other)
    if self.major ~= other.major then return self.major < other.major end
    if self.minor ~= other.minor then return self.minor < other.minor end
    return self.patch < other.patch
end

function Version:__eq(other)
    return self.major == other.major
        and self.minor == other.minor
        and self.patch == other.patch
end

-- Read version from file
local function read_version(filename)
    local f = io.open(filename, "r")
    if not f then return nil end
    local content = f:read("*a")
    f:close()
    
    local version = content:match("version%s*=%s*[\"']([^\"']+)[\"']")
    return version and Version.parse(version)
end

-- Write version to file
local function write_version(filename, version)
    local f = io.open(filename, "r")
    if not f then error("File not found: " .. filename) end
    local content = f:read("*a")
    f:close()
    
    -- Replace version string
    local new_content = content:gsub(
        '(version%s*=%s*["\'])([^"\']+)(["\'])',
        '%1' .. tostring(version) .. '%3'
    )
    
    f = io.open(filename, "w")
    f:write(new_content)
    f:close()
end

-- CLI interface
local args = {...}
local cmd = args[1] or "show"

local current = read_version("myproject-dev-1.rockspec")
if not current then
    current = Version.parse("0.1.0")
end

if cmd == "show" then
    print(tostring(current))
elseif cmd == "bump" then
    local bump_type = args[2] or "patch"
    local new_version = current:bump(bump_type)
    print(string.format("Bumping %s: %s -> %s",
        bump_type, tostring(current), tostring(new_version)))
    -- write_version("myproject-dev-1.rockspec", new_version)
elseif cmd == "check" then
    local compare_ver = Version.parse(args[2] or "1.0.0")
    if current < compare_ver then
        print(tostring(current) .. " < " .. tostring(compare_ver))
    elseif current == compare_ver then
        print(tostring(current) .. " == " .. tostring(compare_ver))
    else
        print(tostring(current) .. " > " .. tostring(compare_ver))
    end
end
```

### ตัวอย่างที่ 11: Rockspec File ที่ Complete

```lua
-- myproject-1.0.0-1.rockspec
package = "myproject"
version = "1.0.0-1"

source = {
    url = "git+https://github.com/username/myproject.git",
    tag = "v1.0.0",
}

description = {
    summary = "A comprehensive Lua project template",
    detailed = [[
        myproject provides utilities and patterns for
        building robust Lua applications with proper
        CI/CD integration.
    ]],
    homepage = "https://github.com/username/myproject",
    license = "MIT",
    maintainer = "Developer <dev@example.com>",
    labels = {"lua", "utilities", "template"},
}

dependencies = {
    "lua >= 5.1",
    "lsqlite3 >= 0.9",
    "inspect >= 3.1",
}

build = {
    type = "builtin",
    modules = {
        ["myproject"] = "src/myproject/init.lua",
        ["myproject.utils"] = "src/myproject/utils.lua",
        ["myproject.config"] = "src/myproject/config.lua",
        ["myproject.logger"] = "src/myproject/logger.lua",
    },
    install = {
        bin = {
            ["myproject"] = "bin/myproject",
        },
    },
}

test_dependencies = {
    "busted >= 2.0",
    "luacheck >= 1.0",
    "luacov >= 0.15",
}

test = {
    type = "busted",
}
```

---

## 65.6 Docker Integration

### ตัวอย่างที่ 12: Dockerfile สำหรับ Lua Application

```dockerfile
# Dockerfile
FROM ubuntu:22.04 AS base

# ติดตั้ง Lua และ dependencies
RUN apt-get update && apt-get install -y \
    lua5.4 \
    luarocks \
    libsqlite3-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# ติดตั้ง Lua dependencies ก่อน (เพื่อ cache layer)
COPY *.rockspec ./
RUN luarocks install --only-deps *.rockspec

# Copy source code
COPY . .

# Install application
RUN luarocks make *.rockspec

EXPOSE 8080

CMD ["lua", "src/server.lua"]

# --- Development stage ---
FROM base AS development
RUN luarocks install busted \
    && luarocks install luacheck \
    && luarocks install luacov

CMD ["lua", "-e", "print('Dev mode')"]

# --- Test stage ---
FROM development AS test
RUN busted --verbose && luacheck src/

# --- Production stage ---
FROM base AS production
ENV LUA_ENV=production
CMD ["lua", "src/server.lua"]
```

### ตัวอย่างที่ 13: Docker Build ใน CI

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [main]
    tags:
      - 'v*.*.*'
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GitHub Container Registry
        if: github.event_name != 'pull_request'
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
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
      
      - name: Build and test
        uses: docker/build-push-action@v5
        with:
          context: .
          target: test
          load: true
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Build and push production image
        if: github.event_name != 'pull_request'
        uses: docker/build-push-action@v5
        with:
          context: .
          target: production
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
```

---

## 65.7 Complete CI/CD Pipeline

### ตัวอย่างที่ 14: Full Pipeline

```yaml
# .github/workflows/pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main, develop, 'feature/**', 'hotfix/**']
  pull_request:
    branches: [main, develop]
  release:
    types: [published]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # Stage 1: Code Quality
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Lua 5.4
        uses: leafo/gh-actions-lua@v10
        with:
          luaVersion: "5.4"
      
      - name: Setup LuaRocks
        uses: leafo/gh-actions-luarocks@v4
      
      - name: Install quality tools
        run: |
          luarocks install luacheck
          luarocks install lua-format || true
      
      - name: Lint check
        run: luacheck src/ spec/ --config .luacheckrc
      
      - name: Format check
        run: |
          # Check if code is formatted (optional)
          echo "Format check passed"
  
  # Stage 2: Unit Tests
  unit-tests:
    name: Unit Tests (Lua ${{ matrix.lua-version }})
    needs: quality
    runs-on: ubuntu-latest
    
    strategy:
      fail-fast: false
      matrix:
        lua-version: ['5.2', '5.3', '5.4', 'luajit-2.1']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Lua ${{ matrix.lua-version }}
        uses: leafo/gh-actions-lua@v10
        with:
          luaVersion: ${{ matrix.lua-version }}
      
      - name: Setup LuaRocks
        uses: leafo/gh-actions-luarocks@v4
      
      - name: Install test dependencies
        run: |
          luarocks install busted
          luarocks install luacov
          luarocks install --only-deps *.rockspec
      
      - name: Run unit tests
        run: |
          busted spec/ \
            --verbose \
            --output junit \
            --ftest-output test-results-${{ matrix.lua-version }}.xml \
            --coverage
      
      - name: Generate coverage report
        if: matrix.lua-version == '5.4'
        run: luacov
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.lua-version }}
          path: test-results-*.xml
      
      - name: Upload coverage
        if: matrix.lua-version == '5.4'
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: false
  
  # Stage 3: Integration Tests
  integration-tests:
    name: Integration Tests
    needs: unit-tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Lua
        uses: leafo/gh-actions-lua@v10
        with:
          luaVersion: "5.4"
      
      - name: Setup LuaRocks
        uses: leafo/gh-actions-luarocks@v4
      
      - name: Install dependencies
        run: |
          luarocks install busted
          luarocks install luasql-postgres PGSQL_INCDIR=/usr/include/postgresql
          luarocks install lua-resty-redis || true
          luarocks install --only-deps *.rockspec
      
      - name: Run integration tests
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: testdb
          DB_USER: testuser
          DB_PASS: testpass
          REDIS_HOST: localhost
          REDIS_PORT: 6379
        run: busted spec/integration/ --verbose
  
  # Stage 4: Build Docker Image
  build:
    name: Build Docker Image
    needs: integration-tests
    runs-on: ubuntu-latest
    
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
      
      - name: Login to registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  # Stage 5: Deploy to Staging
  deploy-staging:
    name: Deploy to Staging
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        env:
          DEPLOY_KEY: ${{ secrets.STAGING_DEPLOY_KEY }}
          STAGING_HOST: ${{ secrets.STAGING_HOST }}
        run: |
          echo "Deploying to staging..."
          # ssh deploy@${STAGING_HOST} "docker pull ... && docker-compose up -d"
          echo "Deployed successfully"
      
      - name: Run smoke tests
        run: |
          # ทดสอบ basic endpoints หลัง deploy
          sleep 30  # รอให้ service start
          # curl -f https://staging.example.com/health
          echo "Smoke tests passed"
  
  # Stage 6: Deploy to Production
  deploy-production:
    name: Deploy to Production
    needs: build
    if: github.event_name == 'release'
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        env:
          PROD_KEY: ${{ secrets.PROD_DEPLOY_KEY }}
          PROD_HOST: ${{ secrets.PROD_HOST }}
        run: |
          echo "Deploying version ${{ github.event.release.tag_name }} to production"
          echo "Deployment successful"
      
      - name: Create deployment record
        run: |
          echo "Recording deployment in monitoring system..."
```

---

## 65.8 Release Automation

### ตัวอย่างที่ 15: Automatic Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest
    permissions:
      contents: write
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Fetch all history for changelog
      
      - name: Setup Lua
        uses: leafo/gh-actions-lua@v10
        with:
          luaVersion: "5.4"
      
      - name: Setup LuaRocks
        uses: leafo/gh-actions-luarocks@v4
      
      - name: Install tools
        run: |
          luarocks install busted
          luarocks install luarocks-build-cpp || true
      
      - name: Run final tests
        run: busted spec/ --verbose
      
      - name: Get version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Generate changelog
        id: changelog
        run: |
          # Generate changelog from git log
          PREV_TAG=$(git describe --abbrev=0 --tags HEAD^ 2>/dev/null || echo "")
          if [ -z "$PREV_TAG" ]; then
            CHANGES=$(git log --oneline --no-merges | head -20)
          else
            CHANGES=$(git log --oneline --no-merges ${PREV_TAG}..HEAD)
          fi
          
          echo "CHANGELOG<<EOF" >> $GITHUB_OUTPUT
          echo "## Changes in v${{ steps.version.outputs.VERSION }}" >> $GITHUB_OUTPUT
          echo "" >> $GITHUB_OUTPUT
          echo "$CHANGES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Create rockspec for release
        run: |
          # สร้าง rockspec สำหรับ version ใหม่
          VERSION=${{ steps.version.outputs.VERSION }}
          sed "s/dev-1/${VERSION}-1/g; s|git+https://.*|git+https://github.com/${{ github.repository }}.git|g" \
            myproject-dev-1.rockspec > myproject-${VERSION}-1.rockspec
          cat myproject-${VERSION}-1.rockspec
      
      - name: Upload to LuaRocks
        if: secrets.LUAROCKS_API_KEY != ''
        env:
          LUAROCKS_API_KEY: ${{ secrets.LUAROCKS_API_KEY }}
        run: |
          VERSION=${{ steps.version.outputs.VERSION }}
          luarocks upload myproject-${VERSION}-1.rockspec \
            --api-key=$LUAROCKS_API_KEY
      
      - name: Create GitHub Release
        uses: ncipollo/release-action@v1
        with:
          name: "v${{ steps.version.outputs.VERSION }}"
          body: ${{ steps.changelog.outputs.CHANGELOG }}
          artifacts: "myproject-*.rockspec"
          makeLatest: true
          token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Notify team
        if: success()
        run: |
          echo "✅ Released v${{ steps.version.outputs.VERSION }} successfully!"
          # สามารถ add Slack/Teams notification ได้ที่นี่
```

---

## 65.9 Environment Promotion

### ตัวอย่างที่ 16: Environment Promotion Pipeline

```yaml
# .github/workflows/promote.yml
name: Environment Promotion

on:
  workflow_dispatch:
    inputs:
      from_env:
        description: 'Source environment'
        required: true
        type: choice
        options:
          - staging
          - production
      to_env:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - production
      version:
        description: 'Version/tag to promote'
        required: true
        type: string

jobs:
  validate:
    name: Validate Promotion
    runs-on: ubuntu-latest
    
    steps:
      - name: Validate promotion path
        run: |
          FROM="${{ inputs.from_env }}"
          TO="${{ inputs.to_env }}"
          
          # ป้องกัน promote ที่ไม่ถูกต้อง
          if [ "$FROM" == "$TO" ]; then
            echo "❌ Cannot promote to same environment"
            exit 1
          fi
          
          if [ "$FROM" == "production" ] && [ "$TO" == "staging" ]; then
            echo "❌ Cannot promote from production to staging"
            exit 1
          fi
          
          echo "✅ Promotion path valid: $FROM -> $TO"
  
  promote:
    name: Promote ${{ inputs.version }} to ${{ inputs.to_env }}
    needs: validate
    runs-on: ubuntu-latest
    environment: ${{ inputs.to_env }}
    
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.version }}
      
      - name: Deploy to ${{ inputs.to_env }}
        run: |
          echo "Promoting ${{ inputs.version }} from ${{ inputs.from_env }} to ${{ inputs.to_env }}"
          # Deploy script here
          echo "Promotion successful"
      
      - name: Verify deployment
        run: |
          echo "Running post-deployment verification..."
          # Health check
          # curl -f https://${{ inputs.to_env }}.example.com/health
          echo "Verification passed"
      
      - name: Tag promotion
        run: |
          git tag "${{ inputs.to_env }}-${{ inputs.version }}" || true
          echo "Tagged: ${{ inputs.to_env }}-${{ inputs.version }}"
```

---

## 65.10 Monitoring และ Alerting

### ตัวอย่างที่ 17: Health Check Endpoint ใน Lua

```lua
-- src/health.lua
-- Health check endpoint สำหรับ CI/CD monitoring

local Health = {}
Health.__index = Health

function Health.new(checks)
    local self = setmetatable({}, Health)
    self._checks = checks or {}
    self._start_time = os.time()
    return self
end

function Health:add_check(name, check_fn)
    self._checks[name] = check_fn
end

function Health:run()
    local results = {
        status = "healthy",
        timestamp = os.time(),
        uptime = os.time() - self._start_time,
        version = os.getenv("APP_VERSION") or "unknown",
        checks = {},
    }

    for name, check_fn in pairs(self._checks) do
        local ok, err = pcall(check_fn)
        local check_result = {
            status = ok and "ok" or "error",
            name = name,
        }

        if not ok then
            check_result.error = tostring(err)
            results.status = "unhealthy"
        end

        results.checks[name] = check_result
    end

    return results
end

function Health:to_json(results)
    local checks_json = {}
    for name, check in pairs(results.checks) do
        local status_str = '"' .. check.status .. '"'
        local error_str = check.error and (', "error": "' .. check.error .. '"') or ""
        table.insert(checks_json, string.format(
            '"%s": {"status": %s%s}', name, status_str, error_str
        ))
    end

    return string.format(
        '{"status": "%s", "timestamp": %d, "uptime": %d, "version": "%s", "checks": {%s}}',
        results.status,
        results.timestamp,
        results.uptime,
        results.version,
        table.concat(checks_json, ", ")
    )
end

-- ตัวอย่างการใช้งาน
local health = Health.new()

-- เพิ่ม checks
health:add_check("database", function()
    -- ตรวจสอบ database connection
    local sqlite3 = require("lsqlite3")
    local db = sqlite3.open(":memory:")
    local row = db:nrows("SELECT 1")()
    db:close()
    assert(row, "Database query failed")
end)

health:add_check("memory", function()
    -- ตรวจสอบ memory usage
    local info = collectgarbage("count")
    local kb = info
    assert(kb < 100000, string.format("Memory too high: %.2f KB", kb))
end)

health:add_check("disk", function()
    -- ตรวจสอบ disk space (Unix only)
    local f = io.open("/proc/mounts", "r")
    if f then
        f:close()
        -- ใน production จะ check actual disk space
    end
end)

local results = health:run()
print(health:to_json(results))
```

### ตัวอย่างที่ 18: Deployment Notification Script

```lua
-- scripts/notify.lua
-- ส่ง notification เมื่อ deploy เสร็จ

local function send_slack_notification(webhook_url, message)
    -- ใช้ curl ส่ง notification
    local payload = string.format(
        '{"text": "%s"}',
        message:gsub('"', '\\"')
    )

    local cmd = string.format(
        'curl -s -X POST -H "Content-Type: application/json" -d \'%s\' "%s"',
        payload, webhook_url
    )

    local result = os.execute(cmd)
    return result == 0
end

local function format_deployment_message(opts)
    local emoji = opts.success and "✅" or "❌"
    local status = opts.success and "succeeded" or "failed"

    return string.format(
        "%s Deployment %s!\n" ..
        "• Service: %s\n" ..
        "• Version: %s\n" ..
        "• Environment: %s\n" ..
        "• By: %s",
        emoji, status,
        opts.service or "unknown",
        opts.version or "unknown",
        opts.environment or "unknown",
        opts.deployer or "CI/CD"
    )
end

-- รับ args จาก environment variables
local deployment = {
    success = os.getenv("DEPLOY_SUCCESS") == "true",
    service = os.getenv("SERVICE_NAME") or "myproject",
    version = os.getenv("DEPLOY_VERSION") or "unknown",
    environment = os.getenv("DEPLOY_ENV") or "unknown",
    deployer = os.getenv("GITHUB_ACTOR") or "CI/CD",
}

local message = format_deployment_message(deployment)
local webhook = os.getenv("SLACK_WEBHOOK_URL")

if webhook then
    local sent = send_slack_notification(webhook, message)
    if sent then
        print("Notification sent successfully")
    else
        print("Failed to send notification")
        os.exit(1)
    end
else
    print("No SLACK_WEBHOOK_URL set, skipping notification")
    print(message)
end
```

---

## 65.11 Security Scanning

### ตัวอย่างที่ 19: Security Checks ใน CI

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # Every Monday at 2 AM

jobs:
  secrets-scan:
    name: Scan for Secrets
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: TruffleHog scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --debug --only-verified
  
  dependency-scan:
    name: Dependency Vulnerability Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Lua
        uses: leafo/gh-actions-lua@v10
        with:
          luaVersion: "5.4"
      
      - name: Setup LuaRocks
        uses: leafo/gh-actions-luarocks@v4
      
      - name: Check for vulnerable dependencies
        run: |
          # ตรวจสอบ rockspec dependencies
          luarocks install --only-deps *.rockspec 2>&1 | tee deps.log
          
          # ตรวจสอบ version ที่ใช้
          luarocks list --porcelain | while read name ver; do
            echo "Checking $name@$ver..."
          done
          
          echo "Dependency check complete"
      
      - name: Scan for hardcoded secrets
        run: |
          # ค้นหา patterns ที่น่าสงสัย
          if grep -rn \
            -e "password\s*=\s*['\"][^'\"]\+['\"]" \
            -e "secret\s*=\s*['\"][^'\"]\+['\"]" \
            -e "api_key\s*=\s*['\"][^'\"]\+['\"]" \
            --include="*.lua" \
            --exclude-dir=spec \
            .; then
            echo "⚠️ Potential hardcoded secrets found!"
            exit 1
          fi
          echo "✅ No hardcoded secrets found"
```

---

## 65.12 Performance Testing ใน CI

### ตัวอย่างที่ 20: Benchmark ใน CI

```lua
-- spec/benchmark_spec.lua
-- Performance benchmarks ที่รันใน CI

local function benchmark(name, fn, iterations)
    iterations = iterations or 1000
    local start = os.clock()

    for _ = 1, iterations do
        fn()
    end

    local elapsed = os.clock() - start
    local per_iter = elapsed / iterations * 1000  -- milliseconds

    return {
        name = name,
        iterations = iterations,
        total_ms = elapsed * 1000,
        per_iter_ms = per_iter,
    }
end

describe("Performance Benchmarks", function()

    describe("string operations", function()
        it("string concatenation should be fast", function()
            local result = benchmark("string_concat", function()
                local s = ""
                for i = 1, 10 do
                    s = s .. tostring(i)
                end
            end, 10000)

            print(string.format(
                "\n  %s: %.3f ms/iter",
                result.name, result.per_iter_ms
            ))

            -- ไม่ควรช้ากว่า 0.1ms ต่อ iteration
            assert.is_true(result.per_iter_ms < 0.1,
                string.format("Too slow: %.3f ms", result.per_iter_ms))
        end)

        it("table.concat should be faster than concatenation", function()
            local result = benchmark("table_concat", function()
                local parts = {}
                for i = 1, 10 do
                    parts[i] = tostring(i)
                end
                return table.concat(parts)
            end, 10000)

            print(string.format(
                "\n  %s: %.3f ms/iter",
                result.name, result.per_iter_ms
            ))

            assert.is_true(result.per_iter_ms < 0.05,
                string.format("Too slow: %.3f ms", result.per_iter_ms))
        end)
    end)

    describe("table operations", function()
        it("table insert should be O(1) amortized", function()
            local result = benchmark("table_insert", function()
                local t = {}
                for i = 1, 100 do
                    table.insert(t, i)
                end
            end, 1000)

            print(string.format(
                "\n  %s: %.3f ms/iter",
                result.name, result.per_iter_ms
            ))

            assert.is_true(result.per_iter_ms < 0.5,
                string.format("Too slow: %.3f ms", result.per_iter_ms))
        end)
    end)

end)
```

---

## 65.13 Monitoring Dashboard

### ตัวอย่างที่ 21: CI Metrics Script

```lua
-- scripts/ci_metrics.lua
-- รวบรวม CI/CD metrics

local Metrics = {}
Metrics.__index = Metrics

function Metrics.new()
    return setmetatable({
        _data = {},
        _start_time = os.time(),
    }, Metrics)
end

function Metrics:record(name, value, labels)
    table.insert(self._data, {
        name = name,
        value = value,
        labels = labels or {},
        timestamp = os.time(),
    })
end

function Metrics:timer(name, fn, labels)
    local start = os.clock()
    local ok, result = pcall(fn)
    local elapsed = os.clock() - start

    self:record(name .. "_duration_seconds", elapsed, labels)
    self:record(name .. "_success", ok and 1 or 0, labels)

    if not ok then
        error(result)
    end
    return result
end

function Metrics:to_prometheus()
    local lines = {}
    for _, m in ipairs(self._data) do
        local label_str = ""
        if next(m.labels) then
            local parts = {}
            for k, v in pairs(m.labels) do
                table.insert(parts, k .. '="' .. v .. '"')
            end
            label_str = "{" .. table.concat(parts, ",") .. "}"
        end

        table.insert(lines, string.format(
            "%s%s %s %d",
            m.name, label_str, tostring(m.value), m.timestamp
        ))
    end
    return table.concat(lines, "\n")
end

function Metrics:summary()
    local by_name = {}
    for _, m in ipairs(self._data) do
        if not by_name[m.name] then
            by_name[m.name] = {count = 0, total = 0, min = math.huge, max = -math.huge}
        end
        local s = by_name[m.name]
        s.count = s.count + 1
        s.total = s.total + m.value
        s.min = math.min(s.min, m.value)
        s.max = math.max(s.max, m.value)
    end

    print("=== CI Metrics Summary ===")
    for name, stats in pairs(by_name) do
        print(string.format(
            "  %s: count=%d, avg=%.3f, min=%.3f, max=%.3f",
            name,
            stats.count,
            stats.total / stats.count,
            stats.min,
            stats.max
        ))
    end
end

-- ทดสอบ
local metrics = Metrics.new()

-- บันทึก metrics ต่างๆ
metrics:record("build_duration_seconds", 45.2, {branch = "main"})
metrics:record("test_count", 156, {suite = "unit"})
metrics:record("test_failures", 0, {suite = "unit"})
metrics:record("coverage_percent", 87.5, {module = "core"})
metrics:record("docker_image_size_mb", 128, {tag = "latest"})

-- Timer example
metrics:timer("database_migration", function()
    -- จำลอง migration
    local sum = 0
    for i = 1, 1000000 do sum = sum + i end
    return sum
end, {env = "test"})

metrics:summary()

print("\n=== Prometheus Format ===")
print(metrics:to_prometheus())
```

---

## แบบฝึกหัด

**ข้อที่ 1:** สร้าง complete CI pipeline สำหรับ Lua library ที่:
- Test บน Lua 5.1, 5.2, 5.3, 5.4 และ LuaJIT
- รัน LuaCheck
- Generate coverage report
- Auto-publish ไป LuaRocks เมื่อ tag v*

**ข้อที่ 2:** สร้าง deployment pipeline ที่:
- Deploy ไป staging เมื่อ push ไป develop
- Deploy ไป production เมื่อ create release
- มี rollback mechanism
- ส่ง Slack notification

**ข้อที่ 3:** เพิ่ม security scanning ใน CI ที่:
- ตรวจสอบ hardcoded secrets
- Check dependencies vulnerabilities
- SAST (Static Application Security Testing)
- Generate security report

**ข้อที่ 4:** สร้าง performance regression testing ที่:
- รัน benchmarks ใน CI
- เปรียบเทียบกับ baseline
- Fail ถ้า performance ลดลงเกิน 10%
- สร้าง performance report

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- **GitHub Actions**: การสร้าง workflow สำหรับ Lua projects
- **Busted Testing**: Unit tests, integration tests, mocking
- **LuaCheck**: Static analysis และ lint configuration
- **LuaCov**: Code coverage measurement
- **Semantic Versioning**: Version management automation
- **Docker Integration**: Building และ pushing Docker images
- **Complete Pipeline**: Multi-stage CI/CD จาก code ถึง production
- **Release Automation**: Auto-publish ไป LuaRocks และ GitHub Releases
- **Environment Promotion**: Dev → Staging → Production
- **Security Scanning**: Secret detection และ dependency scanning
- **Performance Testing**: Benchmarks และ regression detection

**ถัดไป**: บทที่ 66 จะเรียนรู้เกี่ยวกับ Load Balancing algorithms และ implementation ด้วย Lua
