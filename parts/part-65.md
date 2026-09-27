# บทที่ 65: CI/CD Pipeline สำหรับ Lua

## บทนำ

CI/CD (Continuous Integration / Continuous Deployment) คือแนวปฏิบัติที่ช่วยให้ทีม development สามารถ deliver code ได้รวดเร็ว มั่นคง และอัตโนมัติ ในบทนี้จะครอบคลุมการสร้าง CI/CD pipeline สำหรับ Lua projects โดยใช้ GitHub Actions ตั้งแต่การรัน tests, ตรวจสอบ code quality, สร้าง Docker images, ไปจนถึงการ deploy ไปยัง production

---

## 65.1 GitHub Actions สำหรับ Lua Projects

### 65.1.1 โครงสร้าง GitHub Actions

GitHub Actions ใช้ YAML files ใน `.github/workflows/` directory โดยแต่ละ workflow ประกอบด้วย:

- **Events**: สิ่งที่ trigger workflow (push, pull_request, schedule, etc.)
- **Jobs**: กลุ่มของ steps ที่รันบน runner เดียว
- **Steps**: คำสั่งแต่ละขั้นตอนใน job

```yaml
# ตัวอย่างที่ 1: .github/workflows/hello-world.yml - Workflow พื้นฐาน
name: Hello World Workflow

# กำหนด events ที่ trigger workflow
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:    # รันด้วยมือได้

jobs:
  greet:
    name: Greet Job
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Setup Lua
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: "5.4"
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Print Lua version
      run: |
        lua -v
        luarocks --version
    
    - name: Run simple Lua script
      run: |
        lua -e "print('Hello from GitHub Actions!')"
        lua -e "print('Lua version: ' .. _VERSION)"
```

### 65.1.2 Workflow สำหรับ Lua Library

```yaml
# ตัวอย่างที่ 2: .github/workflows/library-ci.yml - CI สำหรับ Lua library
name: Lua Library CI

on:
  push:
    branches: [main, develop, 'feature/**', 'hotfix/**']
  pull_request:
    branches: [main, develop]

# ใช้ concurrency เพื่อยกเลิก runs เก่าเมื่อ push ใหม่
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  LUA_VERSION: "5.4"
  LUAROCKS_VERSION: "3.9.2"

jobs:
  # Job 1: ตรวจสอบ syntax และ style
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Lua
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: ${{ env.LUA_VERSION }}
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Install luacheck
      run: luarocks install luacheck
    
    - name: Run luacheck
      run: |
        luacheck src/ --codes --ranges \
          --globals ngx cjson redis \
          --max-line-length 120 \
          --ignore 212 213  # unused self/vararg
    
    - name: Check formatting (StyLua)
      run: |
        # Download StyLua
        curl -sL https://github.com/JohnnyMorganz/StyLua/releases/latest/download/stylua-linux-x86_64.zip -o stylua.zip
        unzip stylua.zip
        chmod +x stylua
        
        # ตรวจสอบว่า code formatting ถูกต้อง
        ./stylua --check src/ spec/
```

---

## 65.2 Testing ด้วย Busted ใน CI

### 65.2.1 Busted Test Runner

Busted คือ test framework สำหรับ Lua ที่มีความสามารถสูง รองรับ BDD-style tests

```yaml
# ตัวอย่างที่ 3: .github/workflows/test.yml - Test workflow
name: Tests

on: [push, pull_request]

jobs:
  test:
    name: Test on Lua ${{ matrix.lua-version }}
    runs-on: ubuntu-latest
    
    # Matrix testing: ทดสอบกับ Lua versions หลาย version
    strategy:
      fail-fast: false
      matrix:
        lua-version: ["5.1", "5.2", "5.3", "5.4", "luajit-2.1"]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Lua ${{ matrix.lua-version }}
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: ${{ matrix.lua-version }}
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    # Cache LuaRocks dependencies เพื่อเพิ่มความเร็ว
    - name: Cache LuaRocks
      uses: actions/cache@v4
      with:
        path: ~/.luarocks
        key: ${{ runner.os }}-lua${{ matrix.lua-version }}-${{ hashFiles('*.rockspec') }}
        restore-keys: |
          ${{ runner.os }}-lua${{ matrix.lua-version }}-
    
    - name: Install dependencies
      run: |
        luarocks install --only-deps myproject-dev-1.rockspec
        luarocks install busted
        luarocks install luacov
        luarocks install busted-htest  # HTML test reporter
    
    - name: Run tests
      run: |
        busted spec/ \
          --coverage \
          --output=TAP \
          --verbose \
          2>&1 | tee test-results.txt
      env:
        LUA_PATH: "./src/?.lua;./src/?/init.lua;$LUA_PATH"
    
    - name: Upload test results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-results-lua${{ matrix.lua-version }}
        path: test-results.txt
```

### 65.2.2 Busted Test Files

```lua
-- ตัวอย่างที่ 4: spec/calculator_spec.lua - Busted test file
-- BDD-style tests สำหรับ calculator module

local Calculator = require "myproject.calculator"

describe("Calculator", function()
    local calc
    
    -- Setup ก่อนแต่ละ test
    before_each(function()
        calc = Calculator.new()
    end)
    
    -- Teardown หลังจากแต่ละ test
    after_each(function()
        calc = nil
    end)
    
    describe("add", function()
        it("should add two positive numbers", function()
            assert.equals(5, calc:add(2, 3))
        end)
        
        it("should add negative numbers", function()
            assert.equals(-1, calc:add(2, -3))
        end)
        
        it("should handle zero", function()
            assert.equals(0, calc:add(0, 0))
        end)
    end)
    
    describe("divide", function()
        it("should divide correctly", function()
            assert.equals(2, calc:divide(10, 5))
        end)
        
        it("should raise error for division by zero", function()
            assert.has_error(function()
                calc:divide(10, 0)
            end, "Division by zero")
        end)
        
        it("should return float for non-integer division", function()
            local result = calc:divide(7, 2)
            assert.near(3.5, result, 0.001)
        end)
    end)
    
    describe("history", function()
        it("should track operations", function()
            calc:add(1, 2)
            calc:add(3, 4)
            
            local history = calc:get_history()
            assert.equals(2, #history)
            assert.equals("add(1, 2) = 3", history[1])
        end)
        
        it("should clear history", function()
            calc:add(1, 2)
            calc:clear_history()
            
            assert.equals(0, #calc:get_history())
        end)
    end)
    
    -- Async test (สำหรับ LuaJIT/OpenResty)
    describe("async operations", function()
        it("should handle async calculation", function(done)
            calc:async_add(1, 2, function(result)
                assert.equals(3, result)
                done()
            end)
        end)
    end)
end)
```

### 65.2.3 Mock และ Stub ใน Tests

```lua
-- ตัวอย่างที่ 5: spec/service_spec.lua - Testing with mocks
local UserService = require "myproject.user_service"

-- Mock object
local mock_db = {
    find = function(self, id)
        if id == 1 then
            return {id = 1, name = "สมชาย", email = "somchai@test.com"}
        end
        return nil
    end,
    save = function(self, user)
        user.id = math.random(1000)
        return user
    end,
    call_count = {find = 0, save = 0}
}

-- Spy บน mock
local original_find = mock_db.find
mock_db.find = function(self, id)
    mock_db.call_count.find = mock_db.call_count.find + 1
    return original_find(self, id)
end

describe("UserService", function()
    local service
    
    before_each(function()
        -- reset call counts
        mock_db.call_count = {find = 0, save = 0}
        service = UserService.new(mock_db)
    end)
    
    describe("get_user", function()
        it("should return user when found", function()
            local user = service:get_user(1)
            
            assert.is_not_nil(user)
            assert.equals("สมชาย", user.name)
            assert.equals(1, mock_db.call_count.find)
        end)
        
        it("should return nil when not found", function()
            local user = service:get_user(999)
            assert.is_nil(user)
        end)
        
        it("should validate input", function()
            assert.has_error(function()
                service:get_user(-1)
            end)
            
            assert.has_error(function()
                service:get_user("not_a_number")
            end)
        end)
    end)
    
    describe("create_user", function()
        it("should create and return new user with id", function()
            local new_user = service:create_user({
                name = "สมหญิง",
                email = "somying@test.com"
            })
            
            assert.is_not_nil(new_user.id)
            assert.equals("สมหญิง", new_user.name)
        end)
        
        it("should validate required fields", function()
            -- Missing name
            assert.has_error(function()
                service:create_user({email = "test@test.com"})
            end, "name is required")
            
            -- Missing email
            assert.has_error(function()
                service:create_user({name = "Test"})
            end, "email is required")
            
            -- Invalid email format
            assert.has_error(function()
                service:create_user({name = "Test", email = "not-an-email"})
            end, "invalid email format")
        end)
    end)
end)
```

---

## 65.3 Luacheck ใน Pipeline

### 65.3.1 Configuration ของ Luacheck

```lua
-- ตัวอย่างที่ 6: .luacheckrc - Luacheck configuration file
-- Configuration สำหรับ luacheck static analyzer

-- Standard globals ที่อนุญาต
std = "lua54"

-- Global variables ที่กำหนดเพิ่มเติม
globals = {
    -- OpenResty globals
    "ngx",
    "ndk",
    -- Testing globals (Busted)
    "describe",
    "it",
    "before_each",
    "after_each",
    "before_all",
    "after_all",
    "assert",
    "spy",
    "stub",
    "mock",
    "pending",
    "done"
}

-- ปิด warnings บางอย่าง
ignore = {
    "212",  -- Unused argument self
    "213",  -- Unused loop variable
}

-- ไม่ตรวจสอบ files เหล่านี้
exclude_files = {
    "spec/fixtures/**",
    "vendor/**",
    "*.min.lua"
}

-- กำหนด max line length
max_line_length = 120
max_string_line_length = 160

-- ตั้งค่าตามแต่ละ file pattern
files["spec/**"] = {
    -- ใน test files อนุญาต globals เพิ่มเติม
    globals = {"assert", "describe", "it", "before_each", "after_each"},
    ignore = {"211"}  -- อนุญาต unused variables ใน test files
}

files["src/migrations/**"] = {
    -- Migration files อาจมี unused globals
    ignore = {"111", "112"}
}
```

### 65.3.2 Luacheck ใน GitHub Actions

```yaml
# ตัวอย่างที่ 7: .github/workflows/quality.yml - Code quality checks
name: Code Quality

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  luacheck:
    name: Static Analysis (luacheck)
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Lua
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: "5.4"
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Install luacheck
      run: luarocks install luacheck
    
    - name: Run luacheck
      id: luacheck
      run: |
        luacheck src/ spec/ \
          --formatter TAP \
          --codes \
          --ranges \
          2>&1 | tee luacheck-results.txt
        
        # สร้าง summary
        echo "## Luacheck Results" >> $GITHUB_STEP_SUMMARY
        echo '```' >> $GITHUB_STEP_SUMMARY
        cat luacheck-results.txt >> $GITHUB_STEP_SUMMARY
        echo '```' >> $GITHUB_STEP_SUMMARY
    
    - name: Upload luacheck results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: luacheck-results
        path: luacheck-results.txt
    
    - name: Check for critical issues
      run: |
        # Fail ถ้ามี errors (ไม่ใช่แค่ warnings)
        luacheck src/ spec/ --quiet --no-warnings
```

---

## 65.4 Code Coverage ด้วย LuaCov

### 65.4.1 LuaCov Configuration

```lua
-- ตัวอย่างที่ 8: .luacov - LuaCov configuration
-- LuaCov configuration file

return {
    -- ไฟล์ที่ต้องการ coverage
    include = {
        "src/.*",
    },
    
    -- ไฟล์ที่ไม่ต้องการ coverage
    exclude = {
        "spec/.*",
        "vendor/.*",
        ".*/init%.lua$",  -- Skip init files
    },
    
    -- Output file
    statsfile = "luacov.stats.out",
    
    -- Report file
    reportfile = "luacov.report.out",
    
    -- Threshold สำหรับ fail (%)
    -- ถ้า coverage ต่ำกว่านี้จะถือว่า fail
    -- (ใช้ใน CI script ไม่ใช่ config โดยตรง)
    threshold = 80
}
```

### 65.4.2 Coverage ใน GitHub Actions

```yaml
# ตัวอย่างที่ 9: coverage section ใน CI workflow
  coverage:
    name: Code Coverage
    runs-on: ubuntu-latest
    needs: test   # รันหลังจาก test ผ่านแล้ว
    
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
        luarocks install luacov-reporter-lcov  # LCOV format สำหรับ Codecov
    
    - name: Run tests with coverage
      run: |
        busted spec/ \
          --coverage \
          -c luacov.lua  # ใช้ custom coverage config
      env:
        LUA_PATH: "./src/?.lua;$LUA_PATH"
    
    - name: Generate coverage report
      run: |
        lua -e "require('luacov.reporter').report()"
        
        # แสดง summary
        cat luacov.report.out
        
        # ตรวจสอบ coverage threshold
        COVERAGE=$(grep -E "^Total" luacov.report.out | awk '{print $NF}' | tr -d '%')
        echo "Coverage: ${COVERAGE}%"
        
        if [ $(echo "$COVERAGE < 80" | bc) -eq 1 ]; then
          echo "Coverage ${COVERAGE}% is below threshold of 80%"
          exit 1
        fi
    
    - name: Convert to LCOV format
      run: |
        lua -e "require('luacov.reporter.lcov').report()"
        mv luacov.report.lcov coverage.lcov
    
    - name: Upload to Codecov
      uses: codecov/codecov-action@v4
      with:
        file: coverage.lcov
        flags: lua
        name: lua-coverage
        token: ${{ secrets.CODECOV_TOKEN }}
    
    - name: Upload coverage artifacts
      uses: actions/upload-artifact@v4
      with:
        name: coverage-report
        path: |
          luacov.report.out
          coverage.lcov
```

---

## 65.5 Semantic Versioning

### 65.5.1 Version Management

```bash
# ตัวอย่างที่ 10: scripts/version.sh - Version management script
#!/bin/bash
set -e

# อ่าน current version จาก rockspec หรือ version file
get_current_version() {
    if [ -f "VERSION" ]; then
        cat VERSION
    elif ls *.rockspec 1> /dev/null 2>&1; then
        grep -m1 'version = ' *.rockspec | sed "s/.*version = '\(.*\)'.*/\1/"
    else
        echo "0.0.0"
    fi
}

# Parse version components
parse_version() {
    local version=$1
    IFS='.' read -r MAJOR MINOR PATCH <<< "$version"
    echo "$MAJOR $MINOR $PATCH"
}

# Bump version
bump_version() {
    local bump_type=$1  # major, minor, patch
    local current=$(get_current_version)
    read MAJOR MINOR PATCH <<< $(parse_version "$current")
    
    case $bump_type in
        major)
            MAJOR=$((MAJOR + 1))
            MINOR=0
            PATCH=0
            ;;
        minor)
            MINOR=$((MINOR + 1))
            PATCH=0
            ;;
        patch)
            PATCH=$((PATCH + 1))
            ;;
        *)
            echo "Usage: $0 bump [major|minor|patch]"
            exit 1
            ;;
    esac
    
    local new_version="${MAJOR}.${MINOR}.${PATCH}"
    echo "$new_version"
}

# อัปเดต version ใน files ต่าง ๆ
update_version_files() {
    local new_version=$1
    
    # อัปเดต VERSION file
    echo "$new_version" > VERSION
    
    # อัปเดต rockspec
    for rockspec in *.rockspec; do
        if [ -f "$rockspec" ]; then
            sed -i "s/version = '[0-9]*\.[0-9]*\.[0-9]*'/version = '$new_version'/" "$rockspec"
            
            # Rename rockspec file
            local name=$(echo "$rockspec" | sed 's/-[0-9].*\.rockspec//')
            mv "$rockspec" "${name}-${new_version}-1.rockspec"
        fi
    done
    
    # อัปเดต version ใน main Lua file
    if [ -f "src/init.lua" ]; then
        sed -i "s/_VERSION = '[0-9]*\.[0-9]*\.[0-9]*'/_VERSION = '$new_version'/" src/init.lua
    fi
    
    echo "Updated version to $new_version"
}

# Main
case "$1" in
    get)
        get_current_version
        ;;
    bump)
        new_version=$(bump_version "$2")
        update_version_files "$new_version"
        ;;
    set)
        update_version_files "$2"
        ;;
    *)
        echo "Usage: $0 [get|bump|set] [major|minor|patch|version]"
        exit 1
        ;;
esac
```

### 65.5.2 Conventional Commits สำหรับ Auto-versioning

```yaml
# ตัวอย่างที่ 11: .github/workflows/release.yml - Automatic release
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  packages: write

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # ต้องการ full history สำหรับ changelog
        token: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Setup Node.js (สำหรับ semantic-release)
      uses: actions/setup-node@v4
      with:
        node-version: "20"
    
    - name: Install semantic-release
      run: |
        npm install -g \
          semantic-release \
          @semantic-release/changelog \
          @semantic-release/git \
          @semantic-release/github \
          @semantic-release/exec
    
    - name: Setup Lua
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: "5.4"
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Run semantic-release
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        LUAROCKS_API_KEY: ${{ secrets.LUAROCKS_API_KEY }}
      run: npx semantic-release
```

### 65.5.3 .releaserc.json Configuration

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    [
      "@semantic-release/changelog",
      {
        "changelogFile": "CHANGELOG.md"
      }
    ],
    [
      "@semantic-release/exec",
      {
        "prepareCmd": "scripts/version.sh set ${nextRelease.version}",
        "publishCmd": "luarocks upload *.rockspec --api-key=${LUAROCKS_API_KEY}"
      }
    ],
    [
      "@semantic-release/git",
      {
        "assets": ["CHANGELOG.md", "VERSION", "*.rockspec", "src/init.lua"],
        "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
      }
    ],
    "@semantic-release/github"
  ]
}
```

---

## 65.6 Automatic Release ไปยัง LuaRocks

### 65.6.1 Rockspec สำหรับ LuaRocks

```lua
-- ตัวอย่างที่ 12: myproject-1.0.0-1.rockspec - LuaRocks package spec
package = "myproject"
version = "1.0.0-1"
source = {
   url = "git+https://github.com/username/myproject.git",
   tag = "v1.0.0"
}
description = {
   summary = "A useful Lua library",
   detailed = [[
      myproject เป็น Lua library สำหรับ...
      มีความสามารถในการ...
   ]],
   homepage = "https://github.com/username/myproject",
   license = "MIT"
}
dependencies = {
   "lua >= 5.1",
   "lua-cjson >= 2.1.0",
   "luasocket >= 3.0"
}
build = {
   type = "builtin",
   modules = {
      ["myproject"] = "src/init.lua",
      ["myproject.utils"] = "src/utils.lua",
      ["myproject.config"] = "src/config.lua",
      ["myproject.validator"] = "src/validator.lua"
   },
   copy_directories = {
      "docs"
   }
}
```

### 65.6.2 LuaRocks Upload Workflow

```yaml
# ตัวอย่างที่ 13: .github/workflows/publish-luarocks.yml
name: Publish to LuaRocks

on:
  release:
    types: [published]  # trigger เมื่อ create release

jobs:
  publish:
    name: Publish to LuaRocks
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
      with:
        ref: ${{ github.event.release.tag_name }}
    
    - name: Setup Lua
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: "5.4"
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Extract version from tag
      id: version
      run: |
        TAG="${{ github.event.release.tag_name }}"
        VERSION="${TAG#v}"  # ลบ 'v' prefix
        echo "version=${VERSION}" >> $GITHUB_OUTPUT
        echo "rockspec=myproject-${VERSION}-1.rockspec" >> $GITHUB_OUTPUT
    
    - name: Validate rockspec
      run: |
        luarocks lint ${{ steps.version.outputs.rockspec }}
    
    - name: Upload to LuaRocks
      run: |
        luarocks upload \
          ${{ steps.version.outputs.rockspec }} \
          --api-key=${{ secrets.LUAROCKS_API_KEY }}
    
    - name: Verify upload
      run: |
        sleep 30  # รอให้ LuaRocks อัปเดต
        luarocks search myproject ${{ steps.version.outputs.version }}
```

---

## 65.7 Docker Image Building และ Pushing

### 65.7.1 Docker Build และ Push Workflow

```yaml
# ตัวอย่างที่ 14: .github/workflows/docker.yml - Docker build and push
name: Docker Build & Push

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  docker:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
      security-events: write
    
    steps:
    - uses: actions/checkout@v4
    
    # Setup Docker Buildx สำหรับ multi-platform builds
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Set up QEMU (สำหรับ multi-platform)
      uses: docker/setup-qemu-action@v3
    
    # Login ไปยัง registries
    - name: Login to GitHub Container Registry
      if: github.event_name != 'pull_request'
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Login to Docker Hub
      if: github.event_name != 'pull_request'
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    # สร้าง metadata สำหรับ Docker tags
    - name: Extract metadata for Docker
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          ${{ secrets.DOCKERHUB_USERNAME }}/lua-app
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=semver,pattern={{major}}
          type=sha,prefix=sha-
          type=raw,value=latest,enable={{is_default_branch}}
    
    # Build image เพื่อรัน tests ก่อน
    - name: Build test image
      uses: docker/build-push-action@v5
      with:
        context: .
        target: tester
        load: true  # load ลง local Docker
        tags: lua-app:test
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    - name: Run tests in Docker
      run: |
        docker run --rm lua-app:test \
          busted spec/ --output=TAP
    
    # Security scan
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: lua-app:test
        format: sarif
        output: trivy-results.sarif
        severity: HIGH,CRITICAL
    
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: trivy-results.sarif
    
    # Build และ Push production image
    - name: Build and push production image
      uses: docker/build-push-action@v5
      with:
        context: .
        target: production
        platforms: linux/amd64,linux/arm64
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        build-args: |
          APP_VERSION=${{ github.ref_name }}
          BUILD_DATE=${{ github.event.repository.updated_at }}
          GIT_COMMIT=${{ github.sha }}
    
    - name: Inspect image
      if: github.event_name != 'pull_request'
      run: |
        docker buildx imagetools inspect \
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```

---

## 65.8 Integration Testing

### 65.8.1 Integration Test Setup

```yaml
# ตัวอย่างที่ 15: .github/workflows/integration-tests.yml
name: Integration Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  integration:
    name: Integration Tests
    runs-on: ubuntu-latest
    
    services:
      # Redis service
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      # PostgreSQL service
      postgres:
        image: postgres:15-alpine
        ports:
          - 5432:5432
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
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
        luarocks install lua-resty-redis  
        luarocks install luasql-postgres
        luarocks install http  # Pure Lua HTTP client
    
    - name: Run database migrations
      env:
        DB_HOST: localhost
        DB_PORT: 5432
        DB_NAME: testdb
        DB_USER: testuser
        DB_PASSWORD: testpass
      run: |
        lua scripts/migrate.lua up
    
    - name: Run integration tests
      env:
        APP_ENV: test
        REDIS_HOST: localhost
        REDIS_PORT: 6379
        DB_HOST: localhost
        DB_PORT: 5432
        DB_NAME: testdb
        DB_USER: testuser
        DB_PASSWORD: testpass
      run: |
        busted spec/integration/ \
          --output=TAP \
          --verbose \
          --tags=integration
    
    - name: Run API tests with curl
      run: |
        # Start the application
        nginx -c $(pwd)/nginx.conf -p $(pwd) &
        sleep 3
        
        # Test health endpoint
        curl -f http://localhost:8080/health || exit 1
        
        # Test API endpoints
        bash spec/api/test_users_api.sh
        bash spec/api/test_auth_api.sh
        
        # Cleanup
        nginx -s stop
```

### 65.8.2 API Integration Test Script

```bash
# ตัวอย่างที่ 16: spec/api/test_users_api.sh - API integration tests
#!/bin/bash
set -e

BASE_URL="${API_URL:-http://localhost:8080}"
PASS=0
FAIL=0

# Helper functions
assert_status() {
    local expected=$1
    local actual=$2
    local test_name=$3
    
    if [ "$expected" == "$actual" ]; then
        echo "  [PASS] $test_name (status: $actual)"
        ((PASS++))
    else
        echo "  [FAIL] $test_name (expected: $expected, got: $actual)"
        ((FAIL++))
    fi
}

assert_json_field() {
    local json=$1
    local field=$2
    local expected=$3
    local test_name=$4
    
    local actual=$(echo "$json" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('$field',''))")
    
    if [ "$expected" == "$actual" ]; then
        echo "  [PASS] $test_name (field '$field': $actual)"
        ((PASS++))
    else
        echo "  [FAIL] $test_name (expected '$field'='$expected', got '$actual')"
        ((FAIL++))
    fi
}

echo "=== API Integration Tests: Users ==="
echo ""

# Test 1: Health check
echo "Test: Health check"
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/health")
assert_status "200" "$STATUS" "GET /health returns 200"

# Test 2: Get all users (empty)
echo ""
echo "Test: Get all users"
RESPONSE=$(curl -s -w "\n%{http_code}" "$BASE_URL/api/users")
STATUS=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | head -1)
assert_status "200" "$STATUS" "GET /api/users returns 200"

# Test 3: Create user
echo ""
echo "Test: Create user"
RESPONSE=$(curl -s -w "\n%{http_code}" \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{"name":"สมชาย","email":"somchai@test.com"}' \
    "$BASE_URL/api/users")
STATUS=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | head -1)
assert_status "201" "$STATUS" "POST /api/users returns 201"
assert_json_field "$BODY" "name" "สมชาย" "Response contains user name"

# Test 4: Validation error
echo ""
echo "Test: Validation error"
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{"name":"NoEmail"}' \
    "$BASE_URL/api/users")
assert_status "422" "$STATUS" "POST /api/users without email returns 422"

# Test 5: Rate limiting
echo ""
echo "Test: Rate limiting"
for i in {1..110}; do
    curl -s -o /dev/null "$BASE_URL/api/users"
done
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/api/users")
assert_status "429" "$STATUS" "Rate limit returns 429 after threshold"

echo ""
echo "=== Summary ==="
echo "Passed: $PASS"
echo "Failed: $FAIL"

if [ $FAIL -gt 0 ]; then
    exit 1
fi
```

---

## 65.9 Environment Promotion (dev→staging→prod)

### 65.9.1 Multi-Environment Deployment Workflow

```yaml
# ตัวอย่างที่ 17: .github/workflows/deploy.yml - Multi-environment deployment
name: Deploy

on:
  push:
    branches:
      - develop    # → staging
      - main       # → production
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  # กำหนด environment ที่จะ deploy
  set-environment:
    name: Set Environment
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    
    steps:
    - name: Determine environment
      id: set-env
      run: |
        if [ "${{ github.event_name }}" == "workflow_dispatch" ]; then
          echo "environment=${{ inputs.environment }}" >> $GITHUB_OUTPUT
        elif [ "${{ github.ref_name }}" == "main" ]; then
          echo "environment=production" >> $GITHUB_OUTPUT
        else
          echo "environment=staging" >> $GITHUB_OUTPUT
        fi

  # Deploy ไปยัง Development (auto)
  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    if: github.ref_name == 'develop' || github.event_name == 'pull_request'
    environment: development
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to dev cluster
      uses: appleboy/ssh-action@v1
      with:
        host: ${{ secrets.DEV_HOST }}
        username: deploy
        key: ${{ secrets.DEV_SSH_KEY }}
        script: |
          cd /srv/lua-app
          git pull origin develop
          docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
          docker-compose exec app nginx -s reload
    
    - name: Run smoke tests on dev
      run: |
        sleep 15  # รอให้ deploy เสร็จ
        curl -f https://dev.example.com/health

  # Deploy ไปยัง Staging (auto after tests)
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    if: github.ref_name == 'develop'
    needs: [deploy-dev]
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Login to registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Deploy to staging
      uses: appleboy/ssh-action@v1
      with:
        host: ${{ secrets.STAGING_HOST }}
        username: deploy
        key: ${{ secrets.STAGING_SSH_KEY }}
        envs: GITHUB_SHA,GITHUB_REPOSITORY
        script: |
          export IMAGE="ghcr.io/${GITHUB_REPOSITORY}:sha-${GITHUB_SHA:0:7}"
          
          # Update image ใน docker-compose
          sed -i "s|image:.*lua-app.*|image: ${IMAGE}|g" /srv/lua-app/docker-compose.yml
          
          cd /srv/lua-app
          docker-compose pull app
          docker-compose up -d --no-build app
          
          # Wait for health check
          for i in {1..30}; do
            if curl -sf https://staging.example.com/health; then
              echo "Deployment successful"
              break
            fi
            echo "Waiting... ($i/30)"
            sleep 5
          done
    
    - name: Run staging acceptance tests
      run: |
        npm install -g newman
        newman run spec/postman/acceptance_tests.json \
          --environment spec/postman/staging_env.json \
          --reporters cli,junit \
          --reporter-junit-export staging-test-results.xml
    
    - name: Upload test results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: staging-acceptance-tests
        path: staging-test-results.xml

  # Deploy ไปยัง Production (manual approval required)
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    if: github.ref_name == 'main'
    needs: []
    environment:
      name: production
      url: https://api.example.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Notify deployment start
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "🚀 Starting production deployment for ${{ github.repository }}",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Production Deployment Started*\n• Repo: ${{ github.repository }}\n• Commit: `${{ github.sha }}`\n• By: ${{ github.actor }}"
                }
              }
            ]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    
    - name: Deploy to production (Blue/Green)
      run: |
        # Deploy to green environment
        ssh deploy@${{ secrets.PROD_HOST }} << 'EOF'
          cd /srv/lua-app
          
          # Pull new image
          docker pull ghcr.io/${{ github.repository }}:${{ github.sha }}
          
          # Scale up green deployment
          docker-compose -f docker-compose.green.yml up -d
          
          # Wait for green to be healthy
          for i in {1..60}; do
            if docker-compose -f docker-compose.green.yml exec app curl -sf http://localhost/health; then
              echo "Green deployment healthy"
              break
            fi
            sleep 5
          done
          
          # Switch traffic to green
          nginx -s reload
          
          # Scale down blue
          sleep 30  # drain existing connections
          docker-compose -f docker-compose.blue.yml down
        EOF
    
    - name: Notify deployment success
      if: success()
      uses: slackapi/slack-github-action@v1
      with:
        payload: '{"text": "Production deployment successful!"}'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    
    - name: Rollback on failure
      if: failure()
      run: |
        ssh deploy@${{ secrets.PROD_HOST }} "cd /srv/lua-app && docker-compose -f docker-compose.blue.yml up -d && nginx -s reload"
        
        # Notify failure
        curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
          -d '{"text": "Production deployment FAILED - rolled back!"}'
```

---

## 65.10 Complete CI/CD Pipeline

### 65.10.1 Full .github/workflows/ci.yml

```yaml
# ตัวอย่างที่ 18: .github/workflows/ci.yml - Complete CI/CD pipeline
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop, 'feature/**', 'hotfix/**', 'release/**']
    tags: ['v*.*.*']
  pull_request:
    branches: [main, develop]
  schedule:
    # รัน full test suite ทุกวันจันทร์เวลา 2:00 (UTC)
    - cron: '0 2 * * 1'
  workflow_dispatch:
    inputs:
      skip_tests:
        description: 'Skip tests (emergency deploy)'
        required: false
        default: 'false'
        type: boolean
      deploy_env:
        description: 'Force deploy to environment'
        required: false
        type: choice
        options: ['', 'staging', 'production']

# Concurrency: ยกเลิก run เก่าสำหรับ branch เดียวกัน
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

# Default permissions (principle of least privilege)
permissions:
  contents: read

env:
  LUA_VERSION: "5.4"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============================================================
  # STAGE 1: Code Quality
  # ============================================================
  
  lint:
    name: Lint & Style Check
    runs-on: ubuntu-latest
    if: ${{ !inputs.skip_tests }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Lua
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: ${{ env.LUA_VERSION }}
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Cache LuaRocks
      uses: actions/cache@v4
      with:
        path: ~/.luarocks
        key: ${{ runner.os }}-luarocks-${{ hashFiles('*.rockspec') }}
    
    - name: Install luacheck
      run: luarocks install luacheck
    
    - name: Run luacheck
      run: |
        luacheck src/ spec/ \
          --codes --ranges \
          --formatter TAP \
          --max-line-length 120
    
    - name: Check YAML syntax
      uses: ibiqlik/action-yamllint@v3
      with:
        file_or_dir: .github/
        config_data: |
          extends: default
          rules:
            line-length:
              max: 160
    
    - name: Lint Dockerfile
      uses: hadolint/hadolint-action@v3.1.0
      with:
        dockerfile: Dockerfile
        failure-threshold: warning
  
  # ============================================================
  # STAGE 2: Tests
  # ============================================================
  
  unit-tests:
    name: Unit Tests (Lua ${{ matrix.lua-version }})
    runs-on: ubuntu-latest
    if: ${{ !inputs.skip_tests }}
    needs: lint
    
    strategy:
      fail-fast: false
      matrix:
        lua-version: ["5.1", "5.4", "luajit-2.1"]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Lua ${{ matrix.lua-version }}
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: ${{ matrix.lua-version }}
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Cache LuaRocks
      uses: actions/cache@v4
      with:
        path: ~/.luarocks
        key: ${{ runner.os }}-lua${{ matrix.lua-version }}-${{ hashFiles('*.rockspec') }}
    
    - name: Install dependencies
      run: |
        luarocks install --only-deps *.rockspec
        luarocks install busted
        luarocks install luacov
    
    - name: Run unit tests
      run: |
        busted spec/unit/ \
          --coverage \
          --output=TAP \
          --verbose \
          2>&1 | tee unit-test-results.txt
      env:
        LUA_PATH: "./src/?.lua;./src/?/init.lua;$LUA_PATH"
    
    - name: Generate coverage report
      run: |
        lua -e "require('luacov.reporter').report()"
        cat luacov.report.out
    
    - name: Check coverage threshold
      run: |
        COVERAGE=$(grep "^Total" luacov.report.out | grep -oP '\d+\.\d+(?=%)' | tail -1)
        echo "Coverage: ${COVERAGE}%"
        if awk "BEGIN {exit !($COVERAGE < 75)}"; then
          echo "::error::Coverage ${COVERAGE}% is below threshold of 75%"
          exit 1
        fi
    
    - name: Upload test artifacts
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: unit-test-results-lua${{ matrix.lua-version }}
        path: |
          unit-test-results.txt
          luacov.report.out
  
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    if: ${{ !inputs.skip_tests }}
    needs: lint
    
    services:
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
        options: --health-cmd "redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 5
      
      postgres:
        image: postgres:15-alpine
        ports: ["5432:5432"]
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Lua
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: ${{ env.LUA_VERSION }}
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Install dependencies
      run: |
        luarocks install --only-deps *.rockspec
        luarocks install busted
    
    - name: Run integration tests
      env:
        APP_ENV: test
        REDIS_HOST: localhost
        REDIS_PORT: 6379
        DB_HOST: localhost
        DB_PORT: 5432
        DB_NAME: testdb
        DB_USER: testuser
        DB_PASSWORD: testpass
      run: |
        busted spec/integration/ --tags=integration --output=TAP
    
    - name: Upload integration test results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: integration-test-results
        path: integration-test-results.txt
  
  # ============================================================
  # STAGE 3: Security Scan
  # ============================================================
  
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: [unit-tests]
    permissions:
      security-events: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run Semgrep
      uses: returntocorp/semgrep-action@v1
      with:
        config: >-
          p/lua
          p/owasp-top-ten
    
    - name: Scan secrets
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: ${{ github.event.repository.default_branch }}
  
  # ============================================================
  # STAGE 4: Build Docker Image
  # ============================================================
  
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    permissions:
      contents: read
      packages: write
    
    outputs:
      image_digest: ${{ steps.build.outputs.digest }}
      image_tag: ${{ steps.meta.outputs.version }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to GHCR
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
          type=semver,pattern={{version}}
          type=sha,prefix=sha-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        target: production
        platforms: linux/amd64,linux/arm64
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
  
  # ============================================================
  # STAGE 5: Deploy
  # ============================================================
  
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build, security]
    if: github.ref_name == 'develop' || (github.event_name == 'workflow_dispatch' && inputs.deploy_env == 'staging')
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to staging
      uses: appleboy/ssh-action@v1
      env:
        IMAGE_TAG: ${{ needs.build.outputs.image_tag }}
      with:
        host: ${{ secrets.STAGING_HOST }}
        username: deploy
        key: ${{ secrets.STAGING_SSH_KEY }}
        envs: IMAGE_TAG,REGISTRY,IMAGE_NAME
        script: |
          export IMAGE="${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
          cd /srv/lua-app
          docker pull "$IMAGE"
          docker-compose up -d --no-build
    
    - name: Run smoke tests
      run: |
        sleep 20
        curl -f https://staging.example.com/health
    
    - name: Run acceptance tests
      run: |
        curl -sf https://staging.example.com/api/users
  
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, security, deploy-staging]
    if: |
      github.ref_name == 'main' || 
      startsWith(github.ref, 'refs/tags/v') ||
      (github.event_name == 'workflow_dispatch' && inputs.deploy_env == 'production')
    environment:
      name: production
      url: https://api.example.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to production
      uses: appleboy/ssh-action@v1
      env:
        IMAGE_TAG: ${{ needs.build.outputs.image_tag }}
      with:
        host: ${{ secrets.PROD_HOST }}
        username: deploy
        key: ${{ secrets.PROD_SSH_KEY }}
        envs: IMAGE_TAG,REGISTRY,IMAGE_NAME
        script: |
          export IMAGE="${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
          cd /srv/lua-app
          docker pull "$IMAGE"
          docker-compose up -d --no-build
    
    - name: Verify production health
      run: |
        for i in {1..12}; do
          if curl -sf https://api.example.com/health; then
            echo "Production is healthy"
            exit 0
          fi
          echo "Waiting for production... ($i/12)"
          sleep 10
        done
        echo "Production health check failed!"
        exit 1
```

---

## 65.11 Advanced CI/CD Patterns

### 65.11.1 Reusable Workflows

```yaml
# ตัวอย่างที่ 19: .github/workflows/reusable-test.yml - Reusable workflow
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      lua-version:
        required: true
        type: string
      run-coverage:
        required: false
        type: boolean
        default: false
    secrets:
      codecov-token:
        required: false
    outputs:
      coverage:
        description: "Coverage percentage"
        value: ${{ jobs.test.outputs.coverage }}

jobs:
  test:
    runs-on: ubuntu-latest
    outputs:
      coverage: ${{ steps.coverage.outputs.percent }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Lua ${{ inputs.lua-version }}
      uses: leafo/gh-actions-lua@v10
      with:
        luaVersion: ${{ inputs.lua-version }}
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Install and run tests
      run: |
        luarocks install busted
        luarocks install luacov
        busted spec/ --coverage
    
    - name: Get coverage
      id: coverage
      if: ${{ inputs.run-coverage }}
      run: |
        lua -e "require('luacov.reporter').report()"
        COVERAGE=$(grep "^Total" luacov.report.out | grep -oP '\d+\.\d+(?=%)' | tail -1)
        echo "percent=${COVERAGE}" >> $GITHUB_OUTPUT
```

### 65.11.2 Composite Actions

```yaml
# ตัวอย่างที่ 20: .github/actions/setup-lua/action.yml - Composite action
name: Setup Lua Environment
description: Sets up Lua with LuaRocks and common packages

inputs:
  lua-version:
    description: Lua version to install
    required: false
    default: '5.4'
  install-busted:
    description: Install Busted test framework
    required: false
    default: 'true'
  install-luacheck:
    description: Install luacheck
    required: false
    default: 'true'

outputs:
  lua-path:
    description: LUA_PATH for installed packages
    value: ${{ steps.setup.outputs.lua-path }}

runs:
  using: composite
  steps:
  - name: Setup Lua
    uses: leafo/gh-actions-lua@v10
    with:
      luaVersion: ${{ inputs.lua-version }}
  
  - name: Setup LuaRocks
    uses: leafo/gh-actions-luarocks@v4
  
  - name: Cache LuaRocks packages
    uses: actions/cache@v4
    with:
      path: ~/.luarocks
      key: ${{ runner.os }}-lua${{ inputs.lua-version }}-${{ hashFiles('*.rockspec') }}
  
  - name: Install test tools
    shell: bash
    run: |
      if [ "${{ inputs.install-busted }}" == "true" ]; then
        luarocks install busted || true
        luarocks install luacov || true
      fi
      
      if [ "${{ inputs.install-luacheck }}" == "true" ]; then
        luarocks install luacheck || true
      fi
  
  - name: Set LUA_PATH
    id: setup
    shell: bash
    run: |
      LUA_PATH="./src/?.lua;./src/?/init.lua;$(luarocks path --lr-path)"
      echo "lua-path=${LUA_PATH}" >> $GITHUB_OUTPUT
      echo "LUA_PATH=${LUA_PATH}" >> $GITHUB_ENV
```

### 65.11.3 Dependabot Configuration

```yaml
# ตัวอย่างที่ 21: .github/dependabot.yml - Automated dependency updates
version: 2

updates:
  # GitHub Actions updates
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
      day: monday
      time: "09:00"
      timezone: "Asia/Bangkok"
    commit-message:
      prefix: "chore(deps)"
    labels:
      - "dependencies"
      - "github-actions"
    reviewers:
      - "your-github-username"
  
  # Docker base image updates
  - package-ecosystem: docker
    directory: /
    schedule:
      interval: weekly
    commit-message:
      prefix: "chore(docker)"
    labels:
      - "dependencies"
      - "docker"
  
  # npm dependencies (สำหรับ semantic-release)
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    commit-message:
      prefix: "chore(deps)"
    ignore:
      - dependency-name: "*"
        update-types: ["version-update:semver-patch"]
```

---

## 65.12 Monitoring และ Alerting ใน CI/CD

### 65.12.1 GitHub Actions Notifications

```yaml
# ตัวอย่างที่ 22: .github/workflows/notify.yml - Notifications workflow
name: Notifications

on:
  workflow_run:
    workflows: ["CI/CD Pipeline"]
    types: [completed]

jobs:
  notify:
    runs-on: ubuntu-latest
    
    steps:
    - name: Check workflow status
      id: status
      run: |
        if [ "${{ github.event.workflow_run.conclusion }}" == "success" ]; then
          echo "emoji=✅" >> $GITHUB_OUTPUT
          echo "color=#36a64f" >> $GITHUB_OUTPUT
        else
          echo "emoji=❌" >> $GITHUB_OUTPUT
          echo "color=#ff0000" >> $GITHUB_OUTPUT
        fi
    
    - name: Send Slack notification
      uses: slackapi/slack-github-action@v1
      if: github.event.workflow_run.head_branch == 'main'
      with:
        payload: |
          {
            "attachments": [
              {
                "color": "${{ steps.status.outputs.color }}",
                "title": "${{ steps.status.outputs.emoji }} CI/CD Pipeline: ${{ github.event.workflow_run.conclusion }}",
                "fields": [
                  {
                    "title": "Repository",
                    "value": "${{ github.repository }}",
                    "short": true
                  },
                  {
                    "title": "Branch",
                    "value": "${{ github.event.workflow_run.head_branch }}",
                    "short": true
                  },
                  {
                    "title": "Commit",
                    "value": "${{ github.event.workflow_run.head_sha }}",
                    "short": true
                  },
                  {
                    "title": "Run URL",
                    "value": "${{ github.event.workflow_run.html_url }}",
                    "short": false
                  }
                ]
              }
            ]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 65.12.2 Pipeline Performance Monitoring

```lua
-- ตัวอย่างที่ 23: scripts/ci-metrics.lua - CI/CD metrics collection
-- Script สำหรับ collect metrics จาก CI pipeline

local cjson = require "cjson"

local function parse_time(time_str)
    -- Parse ISO8601 duration หรือ seconds
    if type(time_str) == "number" then return time_str end
    local h, m, s = time_str:match("(%d+):(%d+):(%d+)")
    if h then
        return tonumber(h) * 3600 + tonumber(m) * 60 + tonumber(s)
    end
    return 0
end

local function read_github_event()
    local event_file = os.getenv("GITHUB_EVENT_PATH")
    if not event_file then return {} end
    
    local f = io.open(event_file, "r")
    if not f then return {} end
    
    local content = f:read("*all")
    f:close()
    
    local ok, data = pcall(cjson.decode, content)
    return ok and data or {}
end

local function collect_metrics()
    local event = read_github_event()
    
    return {
        workflow = os.getenv("GITHUB_WORKFLOW") or "",
        run_id = os.getenv("GITHUB_RUN_ID") or "",
        run_number = tonumber(os.getenv("GITHUB_RUN_NUMBER")) or 0,
        repository = os.getenv("GITHUB_REPOSITORY") or "",
        branch = os.getenv("GITHUB_REF_NAME") or "",
        sha = os.getenv("GITHUB_SHA") or "",
        actor = os.getenv("GITHUB_ACTOR") or "",
        event_name = os.getenv("GITHUB_EVENT_NAME") or "",
        timestamp = os.time()
    }
end

-- Main
local metrics = collect_metrics()
print(cjson.encode(metrics))

-- ส่ง metrics ไปยัง monitoring service
local http_cmd = string.format(
    "curl -s -X POST '%s/metrics' -H 'Content-Type: application/json' -d '%s'",
    os.getenv("METRICS_ENDPOINT") or "http://metrics.internal",
    cjson.encode(metrics)
)
os.execute(http_cmd)
```

---

## 65.13 Best Practices และ Optimization

### 65.13.1 Caching Strategy

```yaml
# ตัวอย่างที่ 24: Caching strategy สำหรับ Lua CI
# ใน job steps:

    - name: Cache LuaRocks packages
      id: cache-luarocks
      uses: actions/cache@v4
      with:
        path: |
          ~/.luarocks
          /usr/local/lib/lua
          /usr/local/share/lua
        # Cache key ที่ดี: combine OS + Lua version + lockfile hash
        key: luarocks-${{ runner.os }}-lua${{ env.LUA_VERSION }}-${{ hashFiles('**/*.rockspec', '**/luarocks.lock') }}
        restore-keys: |
          luarocks-${{ runner.os }}-lua${{ env.LUA_VERSION }}-
          luarocks-${{ runner.os }}-
    
    - name: Cache Docker layers
      uses: actions/cache@v4
      with:
        path: /tmp/.buildx-cache
        key: docker-${{ runner.os }}-${{ hashFiles('**/Dockerfile', '*.rockspec') }}
        restore-keys: |
          docker-${{ runner.os }}-
    
    - name: Install dependencies (use cache if available)
      if: steps.cache-luarocks.outputs.cache-hit != 'true'
      run: |
        luarocks install --only-deps *.rockspec
        luarocks install busted
        luarocks install luacheck
        luarocks install luacov
```

### 65.13.2 Parallel Test Execution

```yaml
# ตัวอย่างที่ 25: Parallel test execution
  parallel-tests:
    name: Test Suite ${{ matrix.suite }}
    runs-on: ubuntu-latest
    
    strategy:
      fail-fast: false
      matrix:
        suite:
          - unit/models
          - unit/services
          - unit/utils
          - integration/api
          - integration/database
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup environment
      uses: ./.github/actions/setup-lua
      with:
        lua-version: "5.4"
    
    - name: Run test suite
      run: |
        busted spec/${{ matrix.suite }}/ \
          --output=TAP \
          2>&1 | tee test-${{ matrix.suite }}-results.txt
    
    - name: Upload results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-${{ matrix.suite }}-results
        path: test-${{ matrix.suite }}-results.txt
  
  # รวม results จาก parallel tests
  aggregate-results:
    name: Aggregate Test Results
    runs-on: ubuntu-latest
    needs: parallel-tests
    if: always()
    
    steps:
    - name: Download all results
      uses: actions/download-artifact@v4
      with:
        pattern: test-*-results
        merge-multiple: true
    
    - name: Summarize results
      run: |
        echo "## Test Results Summary" >> $GITHUB_STEP_SUMMARY
        echo "" >> $GITHUB_STEP_SUMMARY
        
        TOTAL_PASS=0
        TOTAL_FAIL=0
        
        for file in *.txt; do
          PASS=$(grep -c "^ok" "$file" || true)
          FAIL=$(grep -c "^not ok" "$file" || true)
          TOTAL_PASS=$((TOTAL_PASS + PASS))
          TOTAL_FAIL=$((TOTAL_FAIL + FAIL))
          
          echo "### $file" >> $GITHUB_STEP_SUMMARY
          echo "- Passed: $PASS" >> $GITHUB_STEP_SUMMARY
          echo "- Failed: $FAIL" >> $GITHUB_STEP_SUMMARY
        done
        
        echo "" >> $GITHUB_STEP_SUMMARY
        echo "**Total: $TOTAL_PASS passed, $TOTAL_FAIL failed**" >> $GITHUB_STEP_SUMMARY
        
        if [ $TOTAL_FAIL -gt 0 ]; then
          exit 1
        fi
```

---

## 65.14 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic CI Pipeline
สร้าง GitHub Actions workflow สำหรับ Lua library ที่:
- รัน luacheck บน code ทั้งหมดใน `src/`
- รัน Busted tests ใน `spec/`
- ทดสอบกับ Lua 5.1, 5.4, และ LuaJIT
- Upload test results เป็น artifacts

### แบบฝึกหัดที่ 2: Coverage Gate
เพิ่ม code coverage ใน pipeline จากแบบฝึกหัดที่ 1:
- รัน LuaCov พร้อมกับ tests
- Generate HTML coverage report
- Fail pipeline ถ้า coverage ต่ำกว่า 80%
- Upload report ไปยัง Codecov

### แบบฝึกหัดที่ 3: Docker Build Pipeline
สร้าง workflow สำหรับ build และ push Docker image:
- รัน tests ใน Docker container
- ทำ security scan ด้วย Trivy
- Build multi-platform image (amd64 + arm64)
- Push ไปยัง GitHub Container Registry
- Tag image ด้วย version และ commit SHA

### แบบฝึกหัดที่ 4: Multi-Environment Deployment
สร้าง deployment pipeline ที่:
- Auto-deploy ไปยัง dev เมื่อ push ไปยัง feature branch
- Auto-deploy ไปยัง staging เมื่อ merge ไปยัง develop
- Deploy ไปยัง production ต้องมี manual approval
- มี rollback mechanism เมื่อ health check ไม่ผ่าน

### แบบฝึกหัดที่ 5: Complete LuaRocks Release
สร้าง release automation ที่:
- ใช้ Conventional Commits สำหรับ auto-versioning
- Generate CHANGELOG จาก commits
- สร้าง GitHub Release พร้อม release notes
- Upload package ไปยัง LuaRocks โดยอัตโนมัติ
- ส่ง Slack notification เมื่อ release สำเร็จ

---

## สรุป

ในบทนี้เราได้เรียนรู้การสร้าง CI/CD Pipeline ที่สมบูรณ์สำหรับ Lua projects:

- **GitHub Actions** - สร้าง workflows สำหรับ CI/CD ด้วย YAML
- **Busted** - รัน unit tests และ integration tests ใน CI environment
- **Luacheck** - ทำ static analysis และ style checking อัตโนมัติ
- **LuaCov** - วัด code coverage และบังคับใช้ coverage threshold
- **Semantic Versioning** - จัดการ version แบบอัตโนมัติด้วย Conventional Commits
- **LuaRocks Publishing** - Publish packages ไปยัง LuaRocks อัตโนมัติ
- **Docker Integration** - Build, scan, และ push Docker images ใน pipeline
- **Integration Testing** - รัน tests กับ real services (Redis, PostgreSQL)
- **Environment Promotion** - Deploy แบบ dev → staging → production พร้อม approval gates
- **Optimization** - Caching, parallel execution, reusable workflows

การมี CI/CD pipeline ที่ดีช่วยให้ทีมสามารถ deliver code ได้อย่างรวดเร็วและมั่นใจว่า code quality อยู่ในระดับสูงตลอดเวลา
