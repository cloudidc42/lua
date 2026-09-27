# บทที่ 52: Lapis - Web Framework สำหรับ Lua

## บทนำ

Lapis คือ web framework สำหรับ Lua ที่ทำงานบน OpenResty (Nginx + LuaJIT) หรือ MoonScript มีฟีเจอร์ครบสำหรับการสร้าง web applications เช่น routing, templates, database ORM, sessions และอื่นๆ

### ข้อดีของ Lapis
- ทำงานบน OpenResty ทำให้ performance สูง
- มี ORM สำหรับ PostgreSQL และ MySQL
- Template engine (etlua) ที่ใช้งานง่าย
- Migrations system สำหรับ database
- Sessions และ authentication ครบ
- Test framework built-in

---

## 52.1 การติดตั้ง Lapis

```bash
# ติดตั้งด้วย LuaRocks
luarocks install lapis

# หรือติดตั้งพร้อม dependencies ทั้งหมด
luarocks install lapis
luarocks install lua-cjson
luarocks install pgmoon      # สำหรับ PostgreSQL
luarocks install luasql-mysql # สำหรับ MySQL

# ตรวจสอบการติดตั้ง
lapis help

# สร้าง project ใหม่
mkdir myapp && cd myapp
lapis new --lua  # สร้าง Lua project
# หรือ
lapis new        # สร้าง MoonScript project
```

---

## 52.2 โครงสร้าง Project

```
myapp/
├── app.lua           # Application definition
├── config.lua        # Configuration
├── models/           # Database models
│   ├── users.lua
│   └── posts.lua
├── views/            # HTML templates
│   ├── layout.html   # Base layout
│   ├── index.html
│   └── users/
│       ├── index.html
│       └── show.html
├── static/           # Static files
│   ├── css/
│   └── js/
├── spec/             # Tests
│   └── app_spec.lua
└── nginx.conf        # Generated nginx config
```

---

## 52.3 app.lua พื้นฐาน

```lua
-- app.lua
local lapis = require "lapis"
local app = lapis.Application()

-- Simple route
app:get("/", function(self)
    return self:html_res("<h1>Hello from Lapis!</h1>")
end)

-- Route with parameter
app:get("/users/:id", function(self)
    return "User ID: " .. self.params.id
end)

-- POST route
app:post("/submit", function(self)
    local name = self.params.name
    return "Hello, " .. (name or "stranger") .. "!"
end)

return app
```

---

## 52.4 config.lua

```lua
-- config.lua
local config = require "lapis.config"

config("development", {
    -- Database
    postgres = {
        host     = "127.0.0.1",
        port     = 5432,
        database = "myapp_dev",
        user     = "postgres",
        password = "password"
    },
    
    -- Server
    port          = 8080,
    num_workers   = 1,
    
    -- Logging
    logging = true,
    
    -- Secret key สำหรับ sessions
    secret = "my-super-secret-key-change-in-production",
    
    -- Session storage
    session_name = "myapp_session"
})

config("production", {
    postgres = {
        host     = os.getenv("DB_HOST") or "localhost",
        port     = tonumber(os.getenv("DB_PORT")) or 5432,
        database = os.getenv("DB_NAME") or "myapp",
        user     = os.getenv("DB_USER") or "postgres",
        password = os.getenv("DB_PASS") or ""
    },
    
    port        = 80,
    num_workers = "auto",
    logging     = false,
    secret      = os.getenv("SECRET_KEY") or error("SECRET_KEY required!")
})
```

---

## 52.5 Routes - การกำหนด URL Routes

```lua
local lapis = require "lapis"
local app = lapis.Application()

-- GET routes
app:get("/", function(self)
    return "Home page"
end)

app:get("/about", function(self)
    return "About page"
end)

-- Route parameters
app:get("/users/:id", function(self)
    return "User: " .. self.params.id
end)

app:get("/posts/:year/:month/:slug", function(self)
    local year  = self.params.year
    local month = self.params.month
    local slug  = self.params.slug
    return "Post: " .. year .. "/" .. month .. "/" .. slug
end)

-- POST routes
app:post("/users", function(self)
    -- สร้าง user ใหม่
    local name  = self.params.name
    local email = self.params.email
    return "Created user: " .. (name or "")
end)

-- PUT/PATCH routes
app:put("/users/:id", function(self)
    return "Updated user: " .. self.params.id
end)

-- DELETE routes
app:delete("/users/:id", function(self)
    return "Deleted user: " .. self.params.id
end)

-- Match multiple methods
app:match("users_route", "/users", function(self)
    if self.req.method == "GET" then
        return "List users"
    elseif self.req.method == "POST" then
        return "Create user"
    end
end)

-- Wildcard routes
app:get("/files/*", function(self)
    local path = self.params.splat
    return "File: " .. path
end)

return app
```

---

## 52.6 Request และ Response Objects

```lua
app:get("/request-info", function(self)
    -- self คือ Request object
    
    -- Method
    local method = self.req.method  -- "GET", "POST", etc.
    
    -- URL info
    local url  = self.req.parsed_url  -- parsed URL table
    local path = self.req.parsed_url.path
    
    -- Headers
    local headers = self.req.headers
    local ua = headers["user-agent"]
    local ct = headers["content-type"]
    
    -- Query parameters
    -- URL: /search?q=lua&page=2
    local q    = self.params.q     -- "lua"
    local page = self.params.page  -- "2"
    
    -- POST body params
    -- Form: name=John&email=john@example.com
    local name  = self.params.name
    local email = self.params.email
    
    -- IP Address
    local ip = self.req.headers["x-real-ip"] or "unknown"
    
    -- Return response
    return {
        render = "json",
        json = {
            method  = method,
            path    = path,
            query   = self.params,
            ip      = ip,
            ua      = ua
        }
    }
end)
```

---

## 52.7 Response Types

```lua
-- String response
app:get("/string", function(self)
    return "Plain text response"
end)

-- HTML response
app:get("/html", function(self)
    return self:html_res("<h1>Hello HTML!</h1>")
end)

-- JSON response
app:get("/json", function(self)
    return {
        render = "json",
        json = {
            name    = "Alice",
            age     = 30,
            active  = true
        }
    }
end)

-- JSON helper
app:get("/json2", function(self)
    return self:json_res({message = "Hello JSON!"})
end)

-- Redirect
app:get("/old-path", function(self)
    return {redirect_to = "/new-path"}
end)

app:get("/redirect-with-code", function(self)
    return self:redirect_to("/login", 302)
end)

-- Custom status code
app:get("/not-found", function(self)
    self.status = 404
    return "Not Found"
end)

-- Render template
app:get("/template", function(self)
    self.title = "My Page"
    self.users = {"Alice", "Bob", "Charlie"}
    return {render = "index"}  -- renders views/index.html
end)

-- File download (static file)
app:get("/download/:filename", function(self)
    local filename = self.params.filename
    return {
        layout = false,
        render = "download",
        filename = filename
    }
end)
```

---

## 52.8 HTML Templates ด้วย etlua

```lua
-- ใน app.lua
app:get("/page", function(self)
    self.title = "My Page Title"
    self.user  = {name = "Alice", email = "alice@example.com"}
    self.items = {"Apple", "Banana", "Cherry"}
    return {render = "page"}
end)
```

```html
<!-- views/layout.html -->
<!DOCTYPE html>
<html>
<head>
    <title><%= page_title or "My App" %></title>
    <meta charset="utf-8">
    <link rel="stylesheet" href="/static/css/app.css">
</head>
<body>
    <nav>
        <a href="/">Home</a>
        <a href="/about">About</a>
    </nav>
    
    <main>
        <%- content_for_inner %>
    </main>
    
    <footer>
        <p>&copy; 2024 My App</p>
    </footer>
</body>
</html>
```

```html
<!-- views/page.html -->
<h1><%= title %></h1>

<% if user then %>
    <p>Welcome, <%= user.name %>!</p>
    <p>Email: <%= user.email %></p>
<% else %>
    <p>Welcome, guest!</p>
<% end %>

<ul>
<% for i, item in ipairs(items or {}) do %>
    <li><%= i %>. <%= item %></li>
<% end %>
</ul>

<!-- etlua tags:
     <%= expr %>   - Output escaped HTML
     <%- expr %>   - Output raw HTML (unescaped)
     <% code %>    - Execute Lua code
     <%# comment %> - Comment (not output)
-->
```

---

## 52.9 etlua Template Examples

```html
<!-- views/users/index.html -->
<h1>Users (Total: <%= #users %>)</h1>

<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Email</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
    <% for _, user in ipairs(users) do %>
        <tr>
            <td><%= user.id %></td>
            <td><%= user.name %></td>
            <td><%= user.email %></td>
            <td>
                <a href="/users/<%= user.id %>">View</a>
                <a href="/users/<%= user.id %>/edit">Edit</a>
                <form method="post" action="/users/<%= user.id %>/delete" style="display:inline">
                    <button type="submit">Delete</button>
                </form>
            </td>
        </tr>
    <% end %>
    </tbody>
</table>

<!-- Pagination -->
<% if current_page > 1 then %>
    <a href="?page=<%= current_page - 1 %>">Previous</a>
<% end %>
<span>Page <%= current_page %> of <%= total_pages %></span>
<% if current_page < total_pages then %>
    <a href="?page=<%= current_page + 1 %>">Next</a>
<% end %>
```

```html
<!-- views/partials/flash.html -->
<% if flash and flash.success then %>
    <div class="alert alert-success"><%= flash.success %></div>
<% end %>
<% if flash and flash.error then %>
    <div class="alert alert-danger"><%= flash.error %></div>
<% end %>
```

---

## 52.10 Before/After Filters

```lua
local lapis = require "lapis"
local app = lapis.Application()

-- before_filter ทำงานก่อน action ทุกตัว
app:before_filter(function(self)
    -- ตรวจสอบ authentication
    if not self.session.user_id then
        -- ยกเว้น public routes
        local public_routes = {
            ["/"] = true,
            ["/login"] = true,
            ["/register"] = true
        }
        
        if not public_routes[self.req.parsed_url.path] then
            return self:redirect_to("/login")
        end
    else
        -- โหลด user จาก session
        local Users = require "models.users"
        self.current_user = Users:find(self.session.user_id)
    end
end)

-- before_filter เฉพาะกลุ่ม routes
local function require_admin(self)
    if not self.current_user or self.current_user.role ~= "admin" then
        self.status = 403
        return "Access Denied"
    end
end

app:get("/admin", require_admin, function(self)
    return "Admin panel"
end)

app:get("/admin/users", require_admin, function(self)
    return "Admin users list"
end)
```

---

## 52.11 Database Models - PostgreSQL

```lua
-- models/users.lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users", {
    -- primary_key = "id",  -- default
    -- timestamp = true,    -- auto created_at, updated_at
    
    -- Custom methods
    get_full_name = function(self)
        return self.first_name .. " " .. self.last_name
    end,
    
    is_admin = function(self)
        return self.role == "admin"
    end
})

return Users
```

```lua
-- ใช้งาน Users model ใน app.lua
local Users = require "models.users"

-- CREATE
app:post("/users", function(self)
    local user = Users:create({
        name     = self.params.name,
        email    = self.params.email,
        password = bcrypt_hash(self.params.password),
        role     = "user"
    })
    
    if user then
        return {redirect_to = "/users/" .. user.id}
    else
        self.error = "Failed to create user"
        return {render = "users/new"}
    end
end)

-- READ - Find by primary key
app:get("/users/:id", function(self)
    local user = Users:find(self.params.id)
    
    if not user then
        return self.app.handle_404(self)
    end
    
    self.user = user
    return {render = "users/show"}
end)

-- READ - Find by condition
app:get("/users", function(self)
    -- Find all users
    local all_users = Users:select()
    
    -- Find with WHERE clause
    local admins = Users:select("where role = ?", "admin")
    
    -- Find one by condition
    local alice = Users:find({name = "Alice"})
    
    -- Paginate
    local page  = tonumber(self.params.page) or 1
    local limit = 10
    local users = Users:paginated("order by created_at desc", {
        per_page = limit
    })
    
    self.users = users:get_page(page)
    self.total = users:total_items()
    return {render = "users/index"}
end)

-- UPDATE
app:put("/users/:id", function(self)
    local user = Users:find(self.params.id)
    if not user then
        self.status = 404
        return {render = "json", json = {error = "Not found"}}
    end
    
    user:update({
        name  = self.params.name,
        email = self.params.email
    })
    
    return {render = "json", json = {success = true}}
end)

-- DELETE
app:delete("/users/:id", function(self)
    local user = Users:find(self.params.id)
    if not user then
        self.status = 404
        return {render = "json", json = {error = "Not found"}}
    end
    
    user:delete()
    return {render = "json", json = {message = "Deleted"}}
end)
```

---

## 52.12 Advanced Model Queries

```lua
local db = require "lapis.db"
local Users = require "models.users"
local Posts = require "models.posts"

-- Raw SQL query
local results = db.query("SELECT * FROM users WHERE created_at > ?", "2024-01-01")

-- Count
local count = Users:count()
local admin_count = Users:count("role = ?", "admin")

-- Find with complex conditions
local active_users = Users:select("where active = ? and created_at > ? order by name", 
    true, "2024-01-01")

-- JOIN
local posts_with_authors = db.query([[
    SELECT p.*, u.name as author_name
    FROM posts p
    JOIN users u ON p.user_id = u.id
    WHERE p.published = ?
    ORDER BY p.created_at DESC
    LIMIT ?
]], true, 10)

-- Transaction
db.transaction(function()
    local user = Users:create({name = "New User", email = "new@example.com"})
    Posts:create({
        user_id = user.id,
        title   = "First Post",
        content = "Hello World!"
    })
end)

-- Bulk insert
db.insert("users", {
    {name = "User1", email = "user1@example.com"},
    {name = "User2", email = "user2@example.com"},
    {name = "User3", email = "user3@example.com"}
})

-- Scopes (ถ้า implement เอง)
function Users:active()
    return Users:select("where active = ?", true)
end

function Users:admins()
    return Users:select("where role = ?", "admin")
end
```

---

## 52.13 Migrations

```lua
-- migrations/001_create_users.lua
local schema = require "lapis.db.schema"

return {
    -- Up migration
    up = function()
        schema.create_table("users", {
            {name = "id",         type = schema.types.serial, primary_key = true},
            {name = "name",       type = schema.types.varchar(100), null = false},
            {name = "email",      type = schema.types.varchar(255), null = false, unique = true},
            {name = "password",   type = schema.types.varchar(255), null = false},
            {name = "role",       type = schema.types.varchar(50), default = "'user'"},
            {name = "active",     type = schema.types.boolean, default = "true"},
            {name = "created_at", type = schema.types.time, null = false},
            {name = "updated_at", type = schema.types.time, null = false},
        })
        
        schema.create_index("users", "email", {unique = true})
        schema.create_index("users", "role")
    end,
    
    -- Down migration
    down = function()
        schema.drop_table("users")
    end
}
```

```lua
-- migrations/002_create_posts.lua
local schema = require "lapis.db.schema"

return {
    up = function()
        schema.create_table("posts", {
            {name = "id",         type = schema.types.serial, primary_key = true},
            {name = "user_id",    type = schema.types.integer, null = false},
            {name = "title",      type = schema.types.varchar(255), null = false},
            {name = "slug",       type = schema.types.varchar(255), null = false, unique = true},
            {name = "content",    type = schema.types.text},
            {name = "published",  type = schema.types.boolean, default = "false"},
            {name = "views",      type = schema.types.integer, default = "0"},
            {name = "created_at", type = schema.types.time},
            {name = "updated_at", type = schema.types.time},
        })
        
        -- Foreign key
        schema.create_index("posts", "user_id")
        schema.create_index("posts", "slug", {unique = true})
        schema.create_index("posts", "published")
    end,
    
    down = function()
        schema.drop_table("posts")
    end
}
```

```bash
# รัน migrations
lapis migrate

# Rollback migration ล่าสุด
lapis rollback

# ดู migration status
lapis migration_status
```

---

## 52.14 Sessions

```lua
-- Sessions ใน Lapis ใช้ cookie-based storage
-- ต้องกำหนด secret ใน config.lua

-- เก็บข้อมูลใน session
app:post("/login", function(self)
    local Users = require "models.users"
    local user = Users:find({email = self.params.email})
    
    if user and verify_password(self.params.password, user.password) then
        -- เก็บ user_id ใน session
        self.session.user_id = user.id
        self.session.user_name = user.name
        self.session.login_time = os.time()
        
        return {redirect_to = "/dashboard"}
    else
        self.flash = {error = "Invalid email or password"}
        return {redirect_to = "/login"}
    end
end)

-- อ่าน session
app:get("/dashboard", function(self)
    if not self.session.user_id then
        return {redirect_to = "/login"}
    end
    
    self.user_name = self.session.user_name
    return {render = "dashboard"}
end)

-- ลบ session (logout)
app:post("/logout", function(self)
    self.session.user_id   = nil
    self.session.user_name = nil
    self.session.login_time = nil
    
    return {redirect_to = "/"}
end)

-- Flash messages (เก็บข้ามหน้าครั้งเดียว)
app:post("/form", function(self)
    -- ทำ action
    self.flash.success = "Form submitted successfully!"
    return {redirect_to = "/"}
end)

-- อ่าน flash ใน template
-- views/layout.html:
-- <% if flash and flash.success then %>
--   <div class="alert"><%= flash.success %></div>
-- <% end %>
```

---

## 52.15 Cookie Handling

```lua
-- Set cookie
app:get("/set-cookie", function(self)
    -- Cookie จะถูก set ผ่าน header
    local lapis_util = require "lapis.util"
    
    -- Using ngx directly
    ngx.header["Set-Cookie"] = {
        "user_pref=dark_mode; Path=/; Max-Age=86400",
        "lang=th; Path=/; Max-Age=2592000"
    }
    
    return "Cookie set!"
end)

-- อ่าน cookie
app:get("/read-cookie", function(self)
    local cookies = self.req.headers.cookie or ""
    
    -- Parse cookies
    local cookie_table = {}
    for name, value in cookies:gmatch("([^=;]+)=([^;]*)") do
        name = name:match("^%s*(.-)%s*$")  -- trim whitespace
        cookie_table[name] = value
    end
    
    local pref = cookie_table["user_pref"] or "default"
    local lang = cookie_table["lang"] or "en"
    
    return {
        render = "json",
        json = {
            preference = pref,
            language   = lang
        }
    }
end)
```

---

## 52.16 Form Handling และ Validation

```lua
-- สร้าง validation helper
local function validate(params, rules)
    local errors = {}
    
    for field, rule in pairs(rules) do
        local value = params[field]
        
        if rule.required and (not value or value == "") then
            table.insert(errors, field .. " is required")
        elseif value then
            if rule.min_length and #value < rule.min_length then
                table.insert(errors, 
                    field .. " must be at least " .. rule.min_length .. " characters")
            end
            if rule.max_length and #value > rule.max_length then
                table.insert(errors, 
                    field .. " must be at most " .. rule.max_length .. " characters")
            end
            if rule.pattern and not value:match(rule.pattern) then
                table.insert(errors, field .. " format is invalid")
            end
        end
    end
    
    return #errors == 0, errors
end

-- ใช้งาน validation
app:post("/register", function(self)
    local ok, errors = validate(self.params, {
        name = {
            required   = true,
            min_length = 2,
            max_length = 100
        },
        email = {
            required = true,
            pattern  = "^[^@]+@[^@]+%.[^@]+$"
        },
        password = {
            required   = true,
            min_length = 8
        }
    })
    
    if not ok then
        self.errors = errors
        self.params_copy = self.params  -- เก็บ form values ไว้
        return {render = "register"}
    end
    
    -- สร้าง user
    local Users = require "models.users"
    local existing = Users:find({email = self.params.email})
    if existing then
        self.errors = {"Email already registered"}
        return {render = "register"}
    end
    
    local user = Users:create({
        name     = self.params.name,
        email    = self.params.email,
        password = hash_password(self.params.password)
    })
    
    self.session.user_id = user.id
    return {redirect_to = "/dashboard"}
end)
```

---

## 52.17 Authentication Middleware

```lua
-- helpers/auth.lua
local M = {}

function M.require_login(fn)
    return function(self)
        if not self.session.user_id then
            self.session.return_to = self.req.parsed_url.path
            return self:redirect_to("/login")
        end
        return fn(self)
    end
end

function M.require_admin(fn)
    return function(self)
        if not self.current_user or self.current_user.role ~= "admin" then
            self.status = 403
            return "Access denied"
        end
        return fn(self)
    end
end

return M

-- ใช้งาน
local auth = require "helpers.auth"

app:get("/profile",    auth.require_login(function(self)
    self.user = self.current_user
    return {render = "profile"}
end))

app:get("/admin",      auth.require_admin(function(self)
    return {render = "admin/index"}
end))
```

---

## 52.18 Error Handling

```lua
-- Custom error handlers
app:handle_404(function(self)
    self.status = 404
    if self:is_json() then
        return {render = "json", json = {error = "Not Found", code = 404}}
    end
    return {render = "errors/404"}
end)

app:handle_error(function(self, err, trace)
    ngx.log(ngx.ERR, "Error: " .. tostring(err))
    ngx.log(ngx.ERR, "Trace: " .. tostring(trace))
    
    self.status = 500
    if self:is_json() then
        return {render = "json", json = {
            error = "Internal Server Error",
            code  = 500
        }}
    end
    
    self.error_message = err
    return {render = "errors/500"}
end)

-- Per-route error handling
app:get("/risky", function(self)
    local ok, result = pcall(function()
        -- code ที่อาจ error
        error("Something went wrong!")
    end)
    
    if not ok then
        self.status = 500
        return {render = "json", json = {error = tostring(result)}}
    end
    
    return {render = "json", json = {data = result}}
end)
```

---

## 52.19 URL Helpers

```lua
-- app.lua
local lapis = require "lapis"
local app = lapis.Application()

-- Named routes
app:get("user_show", "/users/:id", function(self)
    self.user = {id = self.params.id, name = "Alice"}
    return {render = "users/show"}
end)

app:get("post_index", "/posts", function(self)
    return {render = "posts/index"}
end)

-- URL generation
app:get("/nav-example", function(self)
    -- สร้าง URL จาก route name
    local user_url = self:url_for("user_show", {id = 42})
    -- ได้ "/users/42"
    
    local post_url = self:url_for("post_index")
    -- ได้ "/posts"
    
    -- URL กับ query string
    local search_url = self:url_for("post_index", nil, {q = "lua", page = 2})
    -- ได้ "/posts?q=lua&page=2"
    
    return {
        render = "json",
        json = {
            user_url  = user_url,
            post_url  = post_url,
            search_url = search_url
        }
    }
end)
```

---

## 52.20 Middleware Pattern

```lua
-- middleware/timing.lua
return function(app)
    app:before_filter(function(self)
        self.start_time = ngx.now()
    end)
    
    -- ต้องใช้ after_filter หรือ wrap content
    local original_respond = app.respond
    -- ... หรือใช้ log phase แทน
end

-- middleware/cors.lua
return function(app)
    app:before_filter(function(self)
        local origin = self.req.headers["origin"]
        if origin then
            ngx.header["Access-Control-Allow-Origin"] = "*"
            ngx.header["Access-Control-Allow-Methods"] = 
                "GET, POST, PUT, DELETE, OPTIONS"
            ngx.header["Access-Control-Allow-Headers"] = 
                "Content-Type, Authorization"
        end
        
        if self.req.method == "OPTIONS" then
            self.status = 204
            return ""  -- empty response
        end
    end)
end

-- ใช้ middleware ใน app.lua
local cors_mw = require "middleware.cors"
cors_mw(app)
```

---

## 52.21 Building a Complete CRUD API

```lua
-- app.lua - Blog Posts CRUD API
local lapis = require "lapis"
local db    = require "lapis.db"
local cjson = require "cjson"

local app = lapis.Application()

-- Models
local Posts = require "models.posts"
local Users = require "models.users"

-- Helper: JSON response
local function json(self, status, data)
    self.status = status
    return {render = "json", json = data}
end

-- Helper: paginate
local function paginate(model, where_clause, params, per_page)
    where_clause = where_clause or ""
    per_page = per_page or 10
    local page = tonumber(params.page) or 1
    
    local total = model:count(where_clause ~= "" and where_clause or nil)
    local items = model:select(
        where_clause .. " order by created_at desc limit ? offset ?",
        per_page, (page - 1) * per_page
    )
    
    return {
        data       = items,
        pagination = {
            page       = page,
            per_page   = per_page,
            total      = total,
            total_pages = math.ceil(total / per_page)
        }
    }
end

-- GET /api/posts - list all posts
app:get("/api/posts", function(self)
    local result = paginate(Posts, "where published = true", self.params)
    return json(self, 200, result)
end)

-- GET /api/posts/:id - get single post
app:get("/api/posts/:id", function(self)
    local post = Posts:find(self.params.id)
    if not post then
        return json(self, 404, {error = "Post not found"})
    end
    
    -- Increment view count
    post:update({views = (post.views or 0) + 1})
    
    return json(self, 200, {data = post})
end)

-- POST /api/posts - create post
app:post("/api/posts", function(self)
    -- Validate
    if not self.params.title or self.params.title == "" then
        return json(self, 422, {error = "title is required"})
    end
    
    local slug = self.params.title:lower()
                    :gsub("[^a-z0-9]+", "-")
                    :gsub("^-+", "")
                    :gsub("-+$", "")
    
    -- Check slug uniqueness
    local existing = Posts:find({slug = slug})
    if existing then
        slug = slug .. "-" .. tostring(os.time())
    end
    
    local post, err = Posts:create({
        user_id   = self.session.user_id or 1,
        title     = self.params.title,
        slug      = slug,
        content   = self.params.content or "",
        published = self.params.published == "true" or false
    })
    
    if not post then
        return json(self, 500, {error = "Failed to create post: " .. (err or "")})
    end
    
    return json(self, 201, {data = post})
end)

-- PUT /api/posts/:id - update post
app:put("/api/posts/:id", function(self)
    local post = Posts:find(self.params.id)
    if not post then
        return json(self, 404, {error = "Post not found"})
    end
    
    local updates = {}
    if self.params.title     then updates.title     = self.params.title end
    if self.params.content   then updates.content   = self.params.content end
    if self.params.published ~= nil then
        updates.published = self.params.published == "true"
    end
    
    if next(updates) then
        post:update(updates)
    end
    
    return json(self, 200, {data = post})
end)

-- DELETE /api/posts/:id - delete post
app:delete("/api/posts/:id", function(self)
    local post = Posts:find(self.params.id)
    if not post then
        return json(self, 404, {error = "Post not found"})
    end
    
    post:delete()
    return json(self, 200, {message = "Post deleted"})
end)

-- GET /api/posts/search?q=keyword
app:get("/api/posts/search", function(self)
    local q = self.params.q
    if not q or q == "" then
        return json(self, 400, {error = "Query parameter 'q' is required"})
    end
    
    local posts = Posts:select(
        "where published = true and (title ilike ? or content ilike ?) limit ?",
        "%" .. q .. "%",
        "%" .. q .. "%",
        20
    )
    
    return json(self, 200, {data = posts, query = q, count = #posts})
end)

return app
```

---

## 52.22 Testing Lapis Apps

```lua
-- spec/app_spec.lua
local use_test_env = require "lapis.spec".use_test_env
local request      = require "lapis.spec.server".request
local assert       = require "luassert"

describe("Blog API", function()
    use_test_env()
    
    before_each(function()
        -- Reset database
        local db = require "lapis.db"
        db.query("TRUNCATE posts RESTART IDENTITY")
        db.query("TRUNCATE users RESTART IDENTITY")
        
        -- Create test user
        db.query("INSERT INTO users (name, email, password) VALUES (?, ?, ?)",
            "Test User", "test@example.com", "hashed_password")
    end)
    
    -- Test GET /api/posts
    it("should return list of posts", function()
        local status, body = request("GET", "/api/posts")
        
        assert.same(200, status)
        
        local data = require("cjson").decode(body)
        assert.truthy(data.data)
        assert.truthy(data.pagination)
    end)
    
    -- Test POST /api/posts
    it("should create a new post", function()
        local status, body = request("POST", "/api/posts", {
            post = {title = "Test Post", content = "Test Content"}
        })
        
        assert.same(201, status)
        
        local data = require("cjson").decode(body)
        assert.same("Test Post", data.data.title)
    end)
    
    -- Test validation
    it("should reject post without title", function()
        local status, body = request("POST", "/api/posts", {
            post = {content = "No title here"}
        })
        
        assert.same(422, status)
        
        local data = require("cjson").decode(body)
        assert.truthy(data.error)
    end)
    
    -- Test 404
    it("should return 404 for nonexistent post", function()
        local status, body = request("GET", "/api/posts/99999")
        assert.same(404, status)
    end)
end)
```

```bash
# รัน tests
busted spec/

# รัน test เดียว
busted spec/app_spec.lua

# รัน พร้อม verbose output
busted --verbose spec/
```

---

## 52.23 Static Files และ Asset Pipeline

```nginx
# nginx.conf
server {
    listen 8080;
    
    # Static files
    location /static/ {
        alias /path/to/myapp/static/;
        expires 7d;
        add_header Cache-Control "public, immutable";
    }
    
    # App routes
    location / {
        default_type text/html;
        content_by_lua_block {
            require("lapis").serve("app")
        }
    }
}
```

```lua
-- ใน template, reference static files
-- views/layout.html
-- <link rel="stylesheet" href="/static/css/app.css">
-- <script src="/static/js/app.js"></script>

-- หรือใช้ asset_url helper
-- <img src="<%= self:asset_url('images/logo.png') %>">
```

---

## 52.24 JSON API Pattern

```lua
-- ตัวอย่าง JSON API ที่สมบูรณ์
local lapis  = require "lapis"
local app    = lapis.Application()
local cjson  = require "cjson"

-- Middleware: parse JSON body
app:before_filter(function(self)
    local ct = self.req.headers["content-type"] or ""
    if ct:match("application/json") then
        local body = self.req:read_body_as_string()
        if body and body ~= "" then
            local ok, data = pcall(cjson.decode, body)
            if ok then
                -- Merge JSON body into params
                for k, v in pairs(data) do
                    self.params[k] = v
                end
            end
        end
    end
end)

-- Standard JSON response helper
local function api_response(self, status, data, meta)
    self.status = status
    local response = {data = data}
    if meta then response.meta = meta end
    return {render = "json", json = response}
end

local function api_error(self, status, message, details)
    self.status = status
    return {render = "json", json = {
        error   = {
            message = message,
            details = details,
            status  = status
        }
    }}
end

-- Routes
app:get("/api/v1/users", function(self)
    local Users = require "models.users"
    local page  = tonumber(self.params.page) or 1
    local limit = tonumber(self.params.limit) or 20
    
    local total = Users:count()
    local users = Users:select("order by created_at desc limit ? offset ?",
        limit, (page - 1) * limit)
    
    return api_response(self, 200, users, {
        page       = page,
        per_page   = limit,
        total      = total,
        total_pages = math.ceil(total / limit)
    })
end)

app:post("/api/v1/users", function(self)
    -- Validation
    if not self.params.name then
        return api_error(self, 422, "Validation failed", {
            name = "is required"
        })
    end
    
    local Users = require "models.users"
    local user = Users:create({
        name  = self.params.name,
        email = self.params.email
    })
    
    return api_response(self, 201, user)
end)

return app
```

---

## 52.25 Lapis Application ขั้นสูง

```lua
-- Subclassing Application
local lapis = require "lapis"
local BaseApp = lapis.Application

local MyApp = BaseApp:extend()

-- Custom properties
MyApp.version = "1.0.0"
MyApp.api_prefix = "/api/v1"

-- Custom method
function MyApp:is_json()
    local ct = self.req.headers["content-type"] or ""
    local accept = self.req.headers["accept"] or ""
    return ct:match("application/json") or accept:match("application/json")
end

-- Include routes from other files
function MyApp:include_routes()
    local user_routes = require "routes.users"
    local post_routes = require "routes.posts"
    
    for _, route in ipairs(user_routes) do
        self:match(unpack(route))
    end
    for _, route in ipairs(post_routes) do
        self:match(unpack(route))
    end
end

return MyApp
```

---

## สรุป

ในบทนี้เราเรียนรู้เกี่ยวกับ Lapis web framework:

1. **การติดตั้งและโครงสร้าง** project
2. **Routes** - GET, POST, PUT, DELETE พร้อม parameters
3. **Request/Response** objects และ methods ต่างๆ
4. **HTML templates** ด้วย etlua
5. **Before/After filters** สำหรับ middleware
6. **Database models** ด้วย lapis.db.model
7. **Migrations** สำหรับ schema management
8. **Sessions** และ cookie handling
9. **Form validation** และ error handling
10. การสร้าง **CRUD API** ที่สมบูรณ์
11. **Testing** ด้วย busted

Lapis เป็น framework ที่ powerful และ production-ready สำหรับการสร้าง web applications ด้วย Lua บน OpenResty!
