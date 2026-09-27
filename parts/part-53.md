# บทที่ 53: REST API Development ด้วย Lua

## บทนำ

REST (Representational State Transfer) คือ architectural style สำหรับการออกแบบ web APIs ที่ใช้กันอย่างแพร่หลาย ในบทนี้เราจะเรียนรู้หลักการออกแบบ REST API และการ implement ด้วย Lua บน OpenResty/Lapis

---

## 53.1 REST Principles หลักๆ

```
REST ยึดหลัก 6 ข้อ:
1. Client-Server        - แยก client และ server ออกจากกัน
2. Stateless            - แต่ละ request มีข้อมูลครบในตัวเอง
3. Cacheable            - Response ระบุได้ว่า cache ได้หรือไม่
4. Uniform Interface    - Interface สม่ำเสมอ (resource-based)
5. Layered System       - Client ไม่รู้ว่า server อยู่ layer ไหน
6. Code on Demand       - Optional: server ส่ง executable code ได้
```

```lua
-- สิ่งที่ REST ไม่ใช่:
-- ❌ /getUser?id=1        (verb ใน URL)
-- ❌ /createPost          (verb ใน URL)
-- ❌ /deleteUser/5        (verb ใน URL)

-- REST ที่ถูกต้อง:
-- ✅ GET    /users/1       (read user)
-- ✅ POST   /users         (create user)
-- ✅ PUT    /users/1       (replace user)
-- ✅ PATCH  /users/1       (update user partially)
-- ✅ DELETE /users/1       (delete user)
```

---

## 53.2 URL Structure Design

```lua
-- Resource Naming Rules:
-- 1. ใช้ nouns ไม่ใช่ verbs
-- 2. ใช้ plural nouns
-- 3. Lowercase, hyphen-separated
-- 4. ไม่มี trailing slash

-- Collections
-- GET    /api/v1/posts          - list all posts
-- POST   /api/v1/posts          - create post

-- Individual resources
-- GET    /api/v1/posts/123      - get post 123
-- PUT    /api/v1/posts/123      - replace post 123
-- PATCH  /api/v1/posts/123      - update post 123 partially
-- DELETE /api/v1/posts/123      - delete post 123

-- Nested resources
-- GET    /api/v1/posts/123/comments     - comments of post 123
-- POST   /api/v1/posts/123/comments     - add comment to post 123
-- GET    /api/v1/posts/123/comments/5   - specific comment

-- Search/Filter (query params)
-- GET    /api/v1/posts?status=published&author=alice&page=2
-- GET    /api/v1/posts?sort=created_at&order=desc&limit=20

-- Actions (non-resource actions)
-- POST   /api/v1/posts/123/publish      - publish a post
-- POST   /api/v1/users/123/activate     - activate user
-- POST   /api/v1/auth/login             - login
-- POST   /api/v1/auth/logout            - logout
```

---

## 53.3 HTTP Methods

```lua
-- GET - ดึงข้อมูล (idempotent, safe)
-- POST - สร้างข้อมูลใหม่ (not idempotent)
-- PUT - แทนที่ resource ทั้งหมด (idempotent)
-- PATCH - อัพเดทบางส่วน (idempotent)
-- DELETE - ลบ (idempotent)
-- HEAD - เหมือน GET แต่ไม่มี body (safe)
-- OPTIONS - ถาม server ว่า support method อะไร

-- ตัวอย่าง Implementation ใน Lapis/OpenResty
local routes = {
    -- Users CRUD
    {method = "GET",    path = "/api/v1/users",       handler = "list_users"},
    {method = "POST",   path = "/api/v1/users",       handler = "create_user"},
    {method = "GET",    path = "/api/v1/users/:id",   handler = "get_user"},
    {method = "PUT",    path = "/api/v1/users/:id",   handler = "replace_user"},
    {method = "PATCH",  path = "/api/v1/users/:id",   handler = "update_user"},
    {method = "DELETE", path = "/api/v1/users/:id",   handler = "delete_user"},
    
    -- Auth
    {method = "POST",   path = "/api/v1/auth/login",  handler = "login"},
    {method = "POST",   path = "/api/v1/auth/logout", handler = "logout"},
    {method = "POST",   path = "/api/v1/auth/refresh", handler = "refresh_token"},
}
```

---

## 53.4 HTTP Status Codes

```lua
-- สรุป Status Codes ที่ใช้บ่อย

-- 2xx Success
local HTTP_200 = 200  -- OK - GET, PUT, PATCH สำเร็จ
local HTTP_201 = 201  -- Created - POST สร้างสำเร็จ
local HTTP_204 = 204  -- No Content - DELETE สำเร็จ (ไม่มี body)

-- 3xx Redirection
local HTTP_301 = 301  -- Moved Permanently
local HTTP_302 = 302  -- Found (Temporary Redirect)
local HTTP_304 = 304  -- Not Modified (cache)

-- 4xx Client Errors
local HTTP_400 = 400  -- Bad Request - request ไม่ถูกต้อง
local HTTP_401 = 401  -- Unauthorized - ต้อง authenticate ก่อน
local HTTP_403 = 403  -- Forbidden - ไม่มีสิทธิ์ (แม้ authenticate แล้ว)
local HTTP_404 = 404  -- Not Found - ไม่พบ resource
local HTTP_405 = 405  -- Method Not Allowed
local HTTP_409 = 409  -- Conflict - ข้อมูลซ้ำ
local HTTP_422 = 422  -- Unprocessable Entity - validation failed
local HTTP_429 = 429  -- Too Many Requests - rate limited

-- 5xx Server Errors
local HTTP_500 = 500  -- Internal Server Error
local HTTP_502 = 502  -- Bad Gateway - upstream error
local HTTP_503 = 503  -- Service Unavailable
local HTTP_504 = 504  -- Gateway Timeout

-- ตัวอย่างการใช้
local function handle_create(self)
    local data = self.params
    
    -- Validation error
    if not data.name then
        self.status = 422  -- Unprocessable Entity
        return {render = "json", json = {error = "name is required"}}
    end
    
    -- Duplicate check
    local existing = Model:find({email = data.email})
    if existing then
        self.status = 409  -- Conflict
        return {render = "json", json = {error = "Email already exists"}}
    end
    
    -- Create
    local record = Model:create(data)
    if not record then
        self.status = 500
        return {render = "json", json = {error = "Database error"}}
    end
    
    -- Success
    self.status = 201  -- Created
    return {render = "json", json = {data = record}}
end
```

---

## 53.5 Request/Response Structure

```lua
-- Standard Request format
-- Headers:
-- Content-Type: application/json
-- Authorization: Bearer <token>
-- Accept: application/json
-- X-Request-ID: uuid-here

-- Standard Response format (JSON API inspired)
-- GET /api/v1/posts/1
{
  "data": {
    "id": 1,
    "type": "posts",
    "attributes": {
      "title": "Hello World",
      "content": "...",
      "published": true,
      "created_at": "2024-01-15T10:30:00Z"
    },
    "relationships": {
      "author": {"data": {"id": 1, "type": "users"}}
    }
  }
}

-- GET /api/v1/posts (collection)
{
  "data": [...],
  "meta": {
    "total": 100,
    "page": 1,
    "per_page": 20,
    "total_pages": 5
  },
  "links": {
    "self":  "/api/v1/posts?page=1",
    "next":  "/api/v1/posts?page=2",
    "prev":  null,
    "first": "/api/v1/posts?page=1",
    "last":  "/api/v1/posts?page=5"
  }
}

-- Error response
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Validation failed",
    "details": {
      "name": ["is required", "must be at least 2 characters"],
      "email": ["is invalid"]
    },
    "request_id": "req_abc123"
  }
}
```

```lua
-- Lua implementation ของ response helpers
local cjson = require "cjson"

local Response = {}

function Response.success(data, meta, links)
    local body = {data = data}
    if meta  then body.meta  = meta end
    if links then body.links = links end
    return body
end

function Response.error(code, message, details)
    return {
        error = {
            code       = code,
            message    = message,
            details    = details,
            request_id = ngx.var.request_id or ngx.md5(tostring(ngx.now()))
        }
    }
end

function Response.paginate(data, page, per_page, total, base_url)
    local total_pages = math.ceil(total / per_page)
    
    local function make_url(p)
        return base_url .. "?page=" .. p .. "&per_page=" .. per_page
    end
    
    return Response.success(data, {
        total       = total,
        page        = page,
        per_page    = per_page,
        total_pages = total_pages
    }, {
        self  = make_url(page),
        next  = page < total_pages and make_url(page + 1) or cjson.null,
        prev  = page > 1 and make_url(page - 1) or cjson.null,
        first = make_url(1),
        last  = make_url(total_pages)
    })
end

return Response
```

---

## 53.6 JSON API Format

```lua
-- ตัวอย่าง JSON API implementation
local Response = require "helpers.response"
local cjson    = require "cjson"

-- Serializer สำหรับ User
local function serialize_user(user, include_email)
    local attrs = {
        name       = user.name,
        role       = user.role,
        active     = user.active,
        created_at = user.created_at
    }
    if include_email then
        attrs.email = user.email
    end
    
    return {
        id   = user.id,
        type = "users",
        attributes = attrs,
        links = {
            self = "/api/v1/users/" .. user.id
        }
    }
end

-- Serializer สำหรับ Post
local function serialize_post(post, options)
    options = options or {}
    
    local attrs = {
        title      = post.title,
        slug       = post.slug,
        content    = options.include_content and post.content or nil,
        excerpt    = post.excerpt,
        published  = post.published,
        views      = post.views,
        created_at = post.created_at,
        updated_at = post.updated_at
    }
    
    local relationships = {}
    if post.author then
        relationships.author = {
            data = {id = post.author.id, type = "users"}
        }
    end
    
    return {
        id            = post.id,
        type          = "posts",
        attributes    = attrs,
        relationships = relationships,
        links         = {
            self = "/api/v1/posts/" .. post.id
        }
    }
end

-- ใช้งาน
app:get("/api/v1/users/:id", function(self)
    local Users = require "models.users"
    local user  = Users:find(self.params.id)
    
    if not user then
        self.status = 404
        return {render = "json", json = Response.error("NOT_FOUND", "User not found")}
    end
    
    self.status = 200
    return {render = "json", json = Response.success(serialize_user(user, true))}
end)
```

---

## 53.7 Error Responses

```lua
-- Error codes ที่เป็นระบบ
local ErrorCodes = {
    -- Auth errors
    AUTH_REQUIRED    = "AUTH_REQUIRED",
    INVALID_TOKEN    = "INVALID_TOKEN",
    TOKEN_EXPIRED    = "TOKEN_EXPIRED",
    PERMISSION_DENIED = "PERMISSION_DENIED",
    
    -- Validation errors
    VALIDATION_FAILED = "VALIDATION_FAILED",
    INVALID_INPUT     = "INVALID_INPUT",
    MISSING_FIELD     = "MISSING_FIELD",
    
    -- Resource errors
    NOT_FOUND         = "NOT_FOUND",
    ALREADY_EXISTS    = "ALREADY_EXISTS",
    
    -- Rate limiting
    RATE_LIMIT_EXCEEDED = "RATE_LIMIT_EXCEEDED",
    
    -- Server errors
    INTERNAL_ERROR    = "INTERNAL_ERROR",
    DATABASE_ERROR    = "DATABASE_ERROR",
    UPSTREAM_ERROR    = "UPSTREAM_ERROR"
}

-- Error handler
local function api_error(self, http_status, error_code, message, details)
    self.status = http_status
    ngx.header["Content-Type"] = "application/json"
    
    local error_body = {
        error = {
            code       = error_code,
            message    = message,
            request_id = ngx.md5(ngx.var.remote_addr .. tostring(ngx.now()))
        }
    }
    
    if details then
        error_body.error.details = details
    end
    
    -- Log server errors
    if http_status >= 500 then
        ngx.log(ngx.ERR, "API Error [" .. error_code .. "]: " .. message)
    end
    
    return {render = "json", json = error_body}
end

-- ตัวอย่างการใช้
app:post("/api/v1/users", function(self)
    -- Missing required field
    if not self.params.email then
        return api_error(self, 422, ErrorCodes.VALIDATION_FAILED,
            "Validation failed",
            {email = {"is required"}}
        )
    end
    
    -- Invalid format
    if not self.params.email:match("^[^@]+@[^@]+%.[^@]+$") then
        return api_error(self, 422, ErrorCodes.INVALID_INPUT,
            "Invalid email format",
            {email = {"must be a valid email address"}}
        )
    end
    
    -- Duplicate
    local Users = require "models.users"
    if Users:find({email = self.params.email}) then
        return api_error(self, 409, ErrorCodes.ALREADY_EXISTS,
            "Email already registered"
        )
    end
    
    -- Server error
    local ok, user = pcall(function()
        return Users:create({email = self.params.email, name = self.params.name})
    end)
    
    if not ok then
        return api_error(self, 500, ErrorCodes.DATABASE_ERROR,
            "Failed to create user"
        )
    end
    
    self.status = 201
    return {render = "json", json = {data = user}}
end)
```

---

## 53.8 Pagination

```lua
-- Pagination helpers
local Paginator = {}

function Paginator.new(params)
    local page     = math.max(1, tonumber(params.page) or 1)
    local per_page = math.min(100, math.max(1, tonumber(params.per_page or params.limit) or 20))
    
    return {
        page     = page,
        per_page = per_page,
        offset   = (page - 1) * per_page,
        limit    = per_page
    }
end

function Paginator.response(data, total, paginator, base_url)
    local total_pages = math.ceil(total / paginator.per_page)
    local page        = paginator.page
    local per_page    = paginator.per_page
    
    local function link(p)
        return base_url .. "?page=" .. p .. "&per_page=" .. per_page
    end
    
    return {
        data  = data,
        meta  = {
            total        = total,
            page         = page,
            per_page     = per_page,
            total_pages  = total_pages,
            count        = #data
        },
        links = {
            self  = link(page),
            first = link(1),
            last  = link(math.max(1, total_pages)),
            prev  = page > 1 and link(page - 1) or cjson.null,
            next  = page < total_pages and link(page + 1) or cjson.null,
        }
    }
end

-- ใช้งาน
app:get("/api/v1/posts", function(self)
    local Posts = require "models.posts"
    local p     = Paginator.new(self.params)
    
    local total = Posts:count("where published = true")
    local posts = Posts:select(
        "where published = true order by created_at desc limit ? offset ?",
        p.limit, p.offset
    )
    
    local base_url = "/api/v1/posts"
    self.status = 200
    return {render = "json", json = Paginator.response(posts, total, p, base_url)}
end)
```

---

## 53.9 Filtering และ Sorting

```lua
-- Filter และ Sort helpers
local function build_query(params, allowed_filters, allowed_sorts)
    local where_parts = {"1=1"}  -- เริ่มด้วย condition จริง
    local where_args  = {}
    local order_parts = {}
    
    -- Filter
    for _, filter in ipairs(allowed_filters) do
        local field    = filter.field
        local param    = filter.param or field
        local operator = filter.operator or "="
        local value    = params[param]
        
        if value and value ~= "" then
            if operator == "like" then
                table.insert(where_parts, field .. " ILIKE ?")
                table.insert(where_args, "%" .. value .. "%")
            elseif operator == "in" then
                -- value เป็น comma-separated list
                local vals = {}
                for v in value:gmatch("[^,]+") do
                    table.insert(vals, "?")
                    table.insert(where_args, v)
                end
                table.insert(where_parts, field .. " IN (" .. table.concat(vals, ",") .. ")")
            elseif operator == "bool" then
                table.insert(where_parts, field .. " = ?")
                table.insert(where_args, value == "true" or value == "1")
            else
                table.insert(where_parts, field .. " " .. operator .. " ?")
                table.insert(where_args, value)
            end
        end
    end
    
    -- Sort
    allowed_sorts = allowed_sorts or {}
    local sort_field = params.sort or "created_at"
    local sort_order = (params.order or "desc"):upper()
    
    -- Validate sort
    local sort_allowed = false
    for _, s in ipairs(allowed_sorts) do
        if s == sort_field then
            sort_allowed = true
            break
        end
    end
    
    if sort_allowed and (sort_order == "ASC" or sort_order == "DESC") then
        table.insert(order_parts, sort_field .. " " .. sort_order)
    else
        table.insert(order_parts, "created_at DESC")
    end
    
    local where = table.concat(where_parts, " AND ")
    local order = table.concat(order_parts, ", ")
    
    return where, where_args, order
end

-- ใช้งาน
app:get("/api/v1/posts", function(self)
    local Posts = require "models.posts"
    
    local filters = {
        {field = "published",   param = "published",   operator = "bool"},
        {field = "user_id",     param = "author_id",   operator = "="},
        {field = "title",       param = "search",      operator = "like"},
        {field = "category",    param = "category",    operator = "="},
    }
    
    local sorts = {"created_at", "updated_at", "title", "views"}
    
    local where, args, order = build_query(self.params, filters, sorts)
    
    local p = Paginator.new(self.params)
    
    -- Build query
    local query = "where " .. where .. " order by " .. order .. " limit ? offset ?"
    table.insert(args, p.limit)
    table.insert(args, p.offset)
    
    local total = Posts:count("where " .. where, unpack(args, 1, #args - 2))
    local posts = Posts:select(query, unpack(args))
    
    self.status = 200
    return {render = "json", json = Paginator.response(posts, total, p, "/api/v1/posts")}
end)
```

---

## 53.10 API Versioning

```lua
-- Strategy 1: URL versioning (most common)
-- /api/v1/users
-- /api/v2/users

-- nginx.conf
-- location /api/v1/ { ... }
-- location /api/v2/ { ... }

-- Strategy 2: Header versioning
-- Accept: application/vnd.myapi.v1+json
-- Accept: application/vnd.myapi.v2+json

-- Strategy 3: Query parameter
-- /api/users?version=1

-- Implementation ใน OpenResty
local function get_api_version(self)
    -- ลอง URL version ก่อน
    local url_version = ngx.var.uri:match("^/api/v(%d+)/")
    if url_version then
        return tonumber(url_version)
    end
    
    -- ลอง Accept header
    local accept = ngx.var.http_accept or ""
    local header_version = accept:match("vnd%.myapi%.v(%d+)")
    if header_version then
        return tonumber(header_version)
    end
    
    -- Default version
    return 1
end

-- Router ที่รองรับ versioning
local handlers = {
    v1 = {
        list_users  = require "handlers.v1.users".list,
        create_user = require "handlers.v1.users".create,
    },
    v2 = {
        list_users  = require "handlers.v2.users".list,
        create_user = require "handlers.v2.users".create,
    }
}

-- ใน content_by_lua_block
local version = get_api_version()
local version_handlers = handlers["v" .. version]

if not version_handlers then
    ngx.status = 400
    ngx.say('{"error": "Unsupported API version"}')
    return
end

-- Dispatch to version-specific handler
local handler = version_handlers[action_name]
if handler then handler() end
```

---

## 53.11 Authentication - API Keys

```lua
-- API Key Authentication
local db = require "lapis.db"

local function verify_api_key(key)
    if not key then return nil end
    
    -- ค้นหา key ใน database
    local result = db.query(
        "SELECT user_id, permissions, expires_at FROM api_keys WHERE key_hash = ? AND active = true",
        hash_key(key)  -- hash key ก่อน store
    )
    
    if not result or #result == 0 then
        return nil, "Invalid API key"
    end
    
    local api_key = result[1]
    
    -- ตรวจสอบ expiry
    if api_key.expires_at and api_key.expires_at < os.time() then
        return nil, "API key expired"
    end
    
    return api_key
end

-- Middleware สำหรับ API key auth
local function api_key_auth(fn)
    return function(self)
        -- รับ key จาก header หรือ query param
        local key = self.req.headers["x-api-key"]
                 or self.params.api_key
        
        if not key then
            self.status = 401
            return {render = "json", json = {
                error = {code = "API_KEY_REQUIRED", message = "API key is required"}
            }}
        end
        
        local api_key, err = verify_api_key(key)
        if not api_key then
            self.status = 401
            return {render = "json", json = {
                error = {code = "INVALID_API_KEY", message = err or "Invalid API key"}
            }}
        end
        
        -- เก็บข้อมูลไว้ใน context
        self.api_key    = api_key
        self.current_user_id = api_key.user_id
        
        return fn(self)
    end
end

-- ใช้งาน
app:get("/api/v1/protected", api_key_auth(function(self)
    return {render = "json", json = {
        message = "Hello, authenticated user!",
        user_id = self.current_user_id
    }}
end))
```

---

## 53.12 Authentication - JWT

```lua
-- JWT (JSON Web Token) Authentication
-- ต้องติดตั้ง: luarocks install lua-resty-jwt

local jwt = require "resty.jwt"

-- JWT Secret (เก็บใน environment variable)
local JWT_SECRET = os.getenv("JWT_SECRET") or "your-secret-key"

-- สร้าง JWT token
local function create_token(user_id, role, expires_in)
    expires_in = expires_in or 3600  -- 1 hour
    
    local payload = {
        sub  = tostring(user_id),
        role = role,
        iat  = ngx.time(),
        exp  = ngx.time() + expires_in
    }
    
    local token = jwt:sign(JWT_SECRET, {
        header  = {typ = "JWT", alg = "HS256"},
        payload = payload
    })
    
    return token
end

-- ตรวจสอบ JWT token
local function verify_token(token)
    if not token then
        return nil, "No token provided"
    end
    
    local jwt_obj = jwt:verify(JWT_SECRET, token)
    
    if not jwt_obj.verified then
        return nil, jwt_obj.reason
    end
    
    local payload = jwt_obj.payload
    
    -- ตรวจสอบ expiry
    if payload.exp and payload.exp < ngx.time() then
        return nil, "Token expired"
    end
    
    return payload
end

-- Login endpoint
app:post("/api/v1/auth/login", function(self)
    local Users = require "models.users"
    local user  = Users:find({email = self.params.email})
    
    if not user or not verify_password(self.params.password, user.password) then
        self.status = 401
        return {render = "json", json = {
            error = {code = "INVALID_CREDENTIALS", message = "Invalid email or password"}
        }}
    end
    
    local token = create_token(user.id, user.role)
    local refresh_token = create_token(user.id, user.role, 86400 * 30)  -- 30 days
    
    -- เก็บ refresh token ใน database
    local db = require "lapis.db"
    db.query("UPDATE users SET refresh_token = ? WHERE id = ?", refresh_token, user.id)
    
    self.status = 200
    return {render = "json", json = {
        data = {
            access_token  = token,
            refresh_token = refresh_token,
            expires_in    = 3600,
            token_type    = "Bearer",
            user          = {id = user.id, name = user.name, role = user.role}
        }
    }}
end)

-- JWT auth middleware
local function jwt_auth(fn)
    return function(self)
        local auth   = self.req.headers["authorization"] or ""
        local token  = auth:match("^Bearer%s+(.+)$")
        
        local payload, err = verify_token(token)
        if not payload then
            self.status = 401
            return {render = "json", json = {
                error = {
                    code    = err == "Token expired" and "TOKEN_EXPIRED" or "INVALID_TOKEN",
                    message = err or "Authentication required"
                }
            }}
        end
        
        self.current_user_id   = tonumber(payload.sub)
        self.current_user_role = payload.role
        
        return fn(self)
    end
end

-- Token refresh
app:post("/api/v1/auth/refresh", function(self)
    local refresh_token = self.params.refresh_token
    
    local payload, err = verify_token(refresh_token)
    if not payload then
        self.status = 401
        return {render = "json", json = {error = {message = "Invalid refresh token"}}}
    end
    
    -- ตรวจสอบว่า refresh_token ตรงกับ database
    local db   = require "lapis.db"
    local user = db.query("SELECT id, role FROM users WHERE id = ? AND refresh_token = ?",
        tonumber(payload.sub), refresh_token)
    
    if not user or #user == 0 then
        self.status = 401
        return {render = "json", json = {error = {message = "Refresh token revoked"}}}
    end
    
    local new_token = create_token(user[1].id, user[1].role)
    
    self.status = 200
    return {render = "json", json = {
        data = {
            access_token = new_token,
            expires_in   = 3600,
            token_type   = "Bearer"
        }
    }}
end)
```

---

## 53.13 Rate Limiting Implementation

```lua
-- Rate Limiting ด้วย sliding window algorithm
local function create_rate_limiter(config)
    config = config or {}
    local max_requests = config.max_requests or 100
    local window_seconds = config.window_seconds or 60
    local key_prefix = config.key_prefix or "rl:"
    
    return function(self)
        local redis = require "resty.redis"
        local red = redis:new()
        red:set_timeouts(100, 100, 100)
        
        local ok, err = red:connect("127.0.0.1", 6379)
        if not ok then
            -- Redis ไม่ available, ข้าม rate limiting
            ngx.log(ngx.WARN, "Rate limiter: Redis unavailable: " .. (err or ""))
            return true
        end
        
        -- Key สำหรับ rate limiting
        local identifier = self.current_user_id 
                        and ("user:" .. self.current_user_id)
                        or  ("ip:" .. ngx.var.remote_addr)
        
        local key = key_prefix .. identifier
        local current_time = ngx.time()
        local window_start = current_time - window_seconds
        
        -- Sliding window ด้วย Sorted Set
        red:multi()
        red:zadd(key, current_time, current_time .. ":" .. math.random(1000000))
        red:zremrangebyscore(key, "-inf", window_start)
        red:zcard(key)
        red:expire(key, window_seconds + 1)
        
        local results, err = red:exec()
        red:set_keepalive(10000, 100)
        
        if not results then
            return true  -- Redis error, allow request
        end
        
        local count = results[3]
        
        -- Set rate limit headers
        ngx.header["X-RateLimit-Limit"]     = max_requests
        ngx.header["X-RateLimit-Remaining"] = math.max(0, max_requests - count)
        ngx.header["X-RateLimit-Reset"]     = current_time + window_seconds
        
        if count > max_requests then
            self.status = 429
            ngx.header["Retry-After"] = window_seconds
            return false, {
                error = {
                    code    = "RATE_LIMIT_EXCEEDED",
                    message = "Too many requests. Please try again later.",
                    retry_after = window_seconds
                }
            }
        end
        
        return true
    end
end

-- สร้าง rate limiters ต่างๆ
local global_limiter = create_rate_limiter({
    max_requests    = 100,
    window_seconds  = 60,
    key_prefix      = "global:"
})

local auth_limiter = create_rate_limiter({
    max_requests    = 5,      -- 5 login attempts per minute
    window_seconds  = 60,
    key_prefix      = "auth:"
})

local api_limiter = create_rate_limiter({
    max_requests    = 1000,
    window_seconds  = 3600,   -- per hour
    key_prefix      = "api:"
})

-- ใช้งาน
app:post("/api/v1/auth/login", function(self)
    -- Strict rate limit สำหรับ login
    local ok, error_response = auth_limiter(self)
    if not ok then
        return {render = "json", json = error_response}
    end
    
    -- ... login logic
end)

app:get("/api/v1/posts", jwt_auth(function(self)
    -- Per-user rate limit
    local ok, error_response = api_limiter(self)
    if not ok then
        return {render = "json", json = error_response}
    end
    
    -- ... list posts
end))
```

---

## 53.14 CORS Implementation

```lua
-- CORS Middleware
local function cors_middleware(config)
    config = config or {}
    local allowed_origins = config.origins or {"*"}
    local allowed_methods = config.methods or 
        "GET, POST, PUT, PATCH, DELETE, OPTIONS"
    local allowed_headers = config.headers or 
        "Content-Type, Authorization, X-Requested-With, X-API-Key"
    local expose_headers  = config.expose or 
        "X-RateLimit-Limit, X-RateLimit-Remaining, X-Request-ID"
    local max_age = config.max_age or 86400  -- 24 hours
    local allow_credentials = config.credentials ~= false  -- default true
    
    return function(self)
        local origin = self.req.headers["origin"]
        
        if origin then
            -- ตรวจสอบ allowed origins
            local origin_allowed = false
            for _, allowed in ipairs(allowed_origins) do
                if allowed == "*" or allowed == origin then
                    origin_allowed = true
                    break
                end
                -- Support wildcard subdomain: *.example.com
                if allowed:match("^%*%.") then
                    local domain = allowed:sub(3)
                    if origin:match(domain:gsub("(%p)", "%%%1") .. "$") then
                        origin_allowed = true
                        break
                    end
                end
            end
            
            if origin_allowed then
                ngx.header["Access-Control-Allow-Origin"] = 
                    (allowed_origins[1] == "*") and "*" or origin
                
                if allow_credentials and allowed_origins[1] ~= "*" then
                    ngx.header["Access-Control-Allow-Credentials"] = "true"
                end
                
                ngx.header["Access-Control-Expose-Headers"] = expose_headers
            end
        end
        
        -- Preflight request
        if self.req.method == "OPTIONS" then
            ngx.header["Access-Control-Allow-Methods"] = allowed_methods
            ngx.header["Access-Control-Allow-Headers"] = allowed_headers
            ngx.header["Access-Control-Max-Age"]       = max_age
            self.status = 204
            return ""
        end
    end
end

-- ใช้งาน
app:before_filter(cors_middleware({
    origins      = {"https://app.example.com", "https://admin.example.com"},
    credentials  = true,
    max_age      = 3600
}))
```

---

## 53.15 Complete Blog Posts API

```lua
-- Blog API ที่สมบูรณ์

-- ============================================
-- models/posts.lua
-- ============================================
local Model = require("lapis.db.model").Model

local Posts = Model:extend("posts", {
    -- Validations
    validate = function(self)
        local errs = {}
        if not self.title or self.title == "" then
            table.insert(errs, "title is required")
        elseif #self.title > 255 then
            table.insert(errs, "title is too long (max 255 characters)")
        end
        if not self.content or self.content == "" then
            table.insert(errs, "content is required")
        end
        return #errs == 0, errs
    end,
    
    -- Generate slug from title
    generate_slug = function(self)
        local slug = self.title:lower()
            :gsub("[^a-z0-9%s]", "")
            :gsub("%s+", "-")
            :gsub("^-+", "")
            :gsub("-+$", "")
        
        -- Ensure uniqueness
        local base_slug = slug
        local counter = 1
        while Posts:find({slug = slug}) do
            slug = base_slug .. "-" .. counter
            counter = counter + 1
        end
        return slug
    end
})

return Posts
```

```lua
-- ============================================
-- app.lua - Full Blog API
-- ============================================
local lapis = require "lapis"
local app   = lapis.Application()
local Posts = require "models.posts"
local cjson = require "cjson"

-- CORS
app:before_filter(function(self)
    ngx.header["Access-Control-Allow-Origin"]  = "*"
    ngx.header["Access-Control-Allow-Methods"] = "GET, POST, PUT, PATCH, DELETE, OPTIONS"
    ngx.header["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
    
    if self.req.method == "OPTIONS" then
        self.status = 204
        return ""
    end
    
    -- Parse JSON body
    local ct = self.req.headers["content-type"] or ""
    if ct:match("application/json") then
        local body = ngx.req.get_body_data()
        if body and body ~= "" then
            local ok, data = pcall(cjson.decode, body)
            if ok and type(data) == "table" then
                for k, v in pairs(data) do
                    self.params[k] = v
                end
            end
        end
    end
end)

-- Helper functions
local function send_json(self, status, data)
    self.status = status
    ngx.header["Content-Type"] = "application/json"
    return {render = "json", json = data}
end

-- ============================================
-- GET /api/v1/posts
-- ============================================
app:get("/api/v1/posts", function(self)
    local page     = math.max(1, tonumber(self.params.page) or 1)
    local per_page = math.min(50, math.max(1, tonumber(self.params.per_page) or 10))
    local offset   = (page - 1) * per_page
    
    -- Build WHERE clause
    local where = "WHERE 1=1"
    local args  = {}
    
    -- Filter by published status
    if self.params.published ~= nil then
        where = where .. " AND published = ?"
        table.insert(args, self.params.published == "true")
    else
        where = where .. " AND published = true"  -- default: show published only
    end
    
    -- Search
    if self.params.q and self.params.q ~= "" then
        where = where .. " AND (title ILIKE ? OR content ILIKE ?)"
        local search = "%" .. self.params.q .. "%"
        table.insert(args, search)
        table.insert(args, search)
    end
    
    -- Tag filter
    if self.params.tag and self.params.tag ~= "" then
        where = where .. " AND ? = ANY(tags)"
        table.insert(args, self.params.tag)
    end
    
    -- Count total
    local db = require "lapis.db"
    local count_args = {table.unpack(args)}
    local count_result = db.query("SELECT COUNT(*) as cnt FROM posts " .. where, 
        table.unpack(count_args))
    local total = tonumber(count_result[1].cnt)
    
    -- Fetch posts
    local sort_field = ({created_at=1, updated_at=1, title=1, views=1})[self.params.sort or ""] 
                       and self.params.sort or "created_at"
    local sort_order = (self.params.order or "desc"):upper()
    if sort_order ~= "ASC" and sort_order ~= "DESC" then sort_order = "DESC" end
    
    local query_args = {table.unpack(args)}
    table.insert(query_args, per_page)
    table.insert(query_args, offset)
    
    local posts = db.query(
        "SELECT id, title, slug, excerpt, published, views, created_at, updated_at "
        .. "FROM posts " .. where 
        .. " ORDER BY " .. sort_field .. " " .. sort_order
        .. " LIMIT ? OFFSET ?",
        table.unpack(query_args)
    )
    
    local total_pages = math.ceil(total / per_page)
    
    return send_json(self, 200, {
        data  = posts,
        meta  = {
            total        = total,
            page         = page,
            per_page     = per_page,
            total_pages  = total_pages
        },
        links = {
            self  = "/api/v1/posts?page=" .. page,
            next  = page < total_pages and "/api/v1/posts?page=" .. (page+1) or cjson.null,
            prev  = page > 1 and "/api/v1/posts?page=" .. (page-1) or cjson.null,
        }
    })
end)

-- ============================================
-- GET /api/v1/posts/:id
-- ============================================
app:get("/api/v1/posts/:id", function(self)
    local post
    
    -- Support both ID and slug
    local id = tonumber(self.params.id)
    if id then
        post = Posts:find(id)
    else
        post = Posts:find({slug = self.params.id})
    end
    
    if not post then
        return send_json(self, 404, {
            error = {code = "NOT_FOUND", message = "Post not found"}
        })
    end
    
    -- Increment views (async, don't block)
    local ok = pcall(function()
        post:update({views = (post.views or 0) + 1})
    end)
    
    return send_json(self, 200, {data = post})
end)

-- ============================================
-- POST /api/v1/posts
-- ============================================
app:post("/api/v1/posts", function(self)
    -- Validate required fields
    local errors = {}
    if not self.params.title or self.params.title == "" then
        errors.title = "is required"
    elseif #self.params.title > 255 then
        errors.title = "is too long (max 255 characters)"
    end
    if not self.params.content or self.params.content == "" then
        errors.content = "is required"
    end
    
    if next(errors) then
        return send_json(self, 422, {
            error = {
                code    = "VALIDATION_FAILED",
                message = "Validation failed",
                details = errors
            }
        })
    end
    
    -- Generate slug
    local slug = self.params.slug
    if not slug or slug == "" then
        slug = self.params.title:lower()
            :gsub("[^a-z0-9%s%-]", "")
            :gsub("%s+", "-")
            :gsub("^-+", ""):gsub("-+$", "")
    end
    
    -- Ensure unique slug
    local base_slug = slug
    local counter   = 1
    while Posts:find({slug = slug}) do
        slug    = base_slug .. "-" .. counter
        counter = counter + 1
    end
    
    local post, err = Posts:create({
        title     = self.params.title,
        slug      = slug,
        content   = self.params.content,
        excerpt   = self.params.excerpt,
        published = self.params.published == true or self.params.published == "true",
        tags      = self.params.tags,
        views     = 0
    })
    
    if not post then
        ngx.log(ngx.ERR, "Failed to create post: " .. tostring(err))
        return send_json(self, 500, {
            error = {code = "DATABASE_ERROR", message = "Failed to create post"}
        })
    end
    
    ngx.header["Location"] = "/api/v1/posts/" .. post.id
    return send_json(self, 201, {data = post})
end)

-- ============================================
-- PATCH /api/v1/posts/:id
-- ============================================
app:patch("/api/v1/posts/:id", function(self)
    local post = Posts:find(tonumber(self.params.id))
    
    if not post then
        return send_json(self, 404, {
            error = {code = "NOT_FOUND", message = "Post not found"}
        })
    end
    
    local updates = {}
    local allowed = {"title", "content", "excerpt", "published", "tags"}
    
    for _, field in ipairs(allowed) do
        if self.params[field] ~= nil then
            updates[field] = self.params[field]
        end
    end
    
    if self.params.title and self.params.title ~= post.title then
        -- Regenerate slug if title changed
        local new_slug = self.params.title:lower()
            :gsub("[^a-z0-9%s%-]", "")
            :gsub("%s+", "-")
        local base_slug = new_slug
        local counter   = 1
        while Posts:find({slug = new_slug}) do
            new_slug = base_slug .. "-" .. counter
            counter  = counter + 1
        end
        updates.slug = new_slug
    end
    
    if next(updates) then
        post:update(updates)
    end
    
    return send_json(self, 200, {data = post})
end)

-- ============================================
-- DELETE /api/v1/posts/:id
-- ============================================
app:delete("/api/v1/posts/:id", function(self)
    local post = Posts:find(tonumber(self.params.id))
    
    if not post then
        return send_json(self, 404, {
            error = {code = "NOT_FOUND", message = "Post not found"}
        })
    end
    
    post:delete()
    
    return send_json(self, 200, {
        message = "Post deleted successfully"
    })
end)

-- ============================================
-- POST /api/v1/posts/:id/publish
-- ============================================
app:post("/api/v1/posts/:id/publish", function(self)
    local post = Posts:find(tonumber(self.params.id))
    
    if not post then
        return send_json(self, 404, {
            error = {code = "NOT_FOUND", message = "Post not found"}
        })
    end
    
    if post.published then
        return send_json(self, 409, {
            error = {code = "ALREADY_PUBLISHED", message = "Post is already published"}
        })
    end
    
    post:update({published = true, published_at = ngx.time()})
    
    return send_json(self, 200, {
        data    = post,
        message = "Post published successfully"
    })
end)

-- ============================================
-- GET /api/v1/posts/:id/related
-- ============================================
app:get("/api/v1/posts/:id/related", function(self)
    local post = Posts:find(tonumber(self.params.id))
    if not post then
        return send_json(self, 404, {error = {message = "Post not found"}})
    end
    
    local db = require "lapis.db"
    
    -- หา related posts ด้วย tags
    local related = db.query([[
        SELECT id, title, slug, excerpt, created_at
        FROM posts
        WHERE id != ?
        AND published = true
        AND tags && ?
        ORDER BY created_at DESC
        LIMIT 5
    ]], post.id, post.tags or "{}")
    
    return send_json(self, 200, {data = related})
end)

-- ============================================
-- Error Handlers
-- ============================================
app:handle_404(function(self)
    return send_json(self, 404, {
        error = {code = "NOT_FOUND", message = "The requested resource was not found"}
    })
end)

app:handle_error(function(self, err, trace)
    ngx.log(ngx.ERR, "Unhandled error: " .. tostring(err))
    return send_json(self, 500, {
        error = {code = "INTERNAL_ERROR", message = "An internal server error occurred"}
    })
end)

return app
```

---

## 53.16 API Documentation

```lua
-- ตัวอย่าง API Documentation endpoint (self-documenting API)

app:get("/api/v1/docs", function(self)
    local docs = {
        openapi = "3.0.0",
        info = {
            title   = "Blog API",
            version = "1.0.0",
            description = "Blog Posts REST API"
        },
        servers = {{url = "/api/v1"}},
        paths = {
            ["/posts"] = {
                get = {
                    summary     = "List all posts",
                    description = "Returns a paginated list of published posts",
                    parameters  = {
                        {name="page",    in_="query", schema={type="integer"}, description="Page number"},
                        {name="per_page",in_="query", schema={type="integer"}, description="Items per page"},
                        {name="q",       in_="query", schema={type="string"},  description="Search query"},
                        {name="sort",    in_="query", schema={type="string"},  description="Sort field"},
                        {name="order",   in_="query", schema={type="string"},  description="asc or desc"},
                    },
                    responses = {
                        ["200"] = {
                            description = "Successful response",
                            content = {
                                ["application/json"] = {
                                    schema = {
                                        type       = "object",
                                        properties = {
                                            data = {type = "array"},
                                            meta = {type = "object"}
                                        }
                                    }
                                }
                            }
                        }
                    }
                },
                post = {
                    summary    = "Create a new post",
                    requestBody = {
                        required = true,
                        content  = {
                            ["application/json"] = {
                                schema = {
                                    type     = "object",
                                    required = {"title", "content"},
                                    properties = {
                                        title     = {type = "string"},
                                        content   = {type = "string"},
                                        published = {type = "boolean"}
                                    }
                                }
                            }
                        }
                    },
                    responses = {
                        ["201"] = {description = "Post created"},
                        ["422"] = {description = "Validation failed"}
                    }
                }
            }
        }
    }
    
    self.status = 200
    ngx.header["Content-Type"] = "application/json"
    return {render = "json", json = docs}
end)
```

---

## 53.17 Request Validation Library

```lua
-- validators.lua
local Validator = {}

function Validator.new(params)
    return {
        params  = params,
        errors  = {},
        
        required = function(self, field, message)
            if not self.params[field] or self.params[field] == "" then
                table.insert(self.errors, {field = field, message = message or field .. " is required"})
            end
            return self
        end,
        
        string = function(self, field, opts)
            local val = self.params[field]
            if val then
                if opts.min and #val < opts.min then
                    table.insert(self.errors, {field=field, message=field.." is too short (min "..opts.min..")"})
                end
                if opts.max and #val > opts.max then
                    table.insert(self.errors, {field=field, message=field.." is too long (max "..opts.max..")"})
                end
                if opts.pattern and not val:match(opts.pattern) then
                    table.insert(self.errors, {field=field, message=opts.pattern_message or field.." format invalid"})
                end
            end
            return self
        end,
        
        number = function(self, field, opts)
            local val = tonumber(self.params[field])
            if self.params[field] and not val then
                table.insert(self.errors, {field=field, message=field.." must be a number"})
            elseif val then
                if opts.min and val < opts.min then
                    table.insert(self.errors, {field=field, message=field.." must be >= "..opts.min})
                end
                if opts.max and val > opts.max then
                    table.insert(self.errors, {field=field, message=field.." must be <= "..opts.max})
                end
            end
            return self
        end,
        
        email = function(self, field)
            local val = self.params[field]
            if val and not val:match("^[^@]+@[^@]+%.[^@]+$") then
                table.insert(self.errors, {field=field, message=field.." must be a valid email"})
            end
            return self
        end,
        
        enum = function(self, field, values)
            local val = self.params[field]
            if val then
                local valid = false
                for _, v in ipairs(values) do
                    if v == val then valid = true break end
                end
                if not valid then
                    table.insert(self.errors, {field=field, 
                        message=field.." must be one of: "..table.concat(values, ", ")})
                end
            end
            return self
        end,
        
        is_valid = function(self)
            return #self.errors == 0, self.errors
        end
    }
end

return Validator

-- ใช้งาน
app:post("/api/v1/posts", function(self)
    local Validator = require "validators"
    local v = Validator.new(self.params)
    
    v:required("title")
     :string("title", {min=2, max=255})
     :required("content")
     :string("content", {min=10})
     :enum("status", {"draft", "published", "archived"})
    
    local ok, errors = v:is_valid()
    if not ok then
        local details = {}
        for _, e in ipairs(errors) do
            details[e.field] = details[e.field] or {}
            table.insert(details[e.field], e.message)
        end
        return send_json(self, 422, {
            error = {code="VALIDATION_FAILED", message="Validation failed", details=details}
        })
    end
    
    -- Create post...
end)
```

---

## สรุป

ในบทนี้เราเรียนรู้ REST API Development ด้วย Lua ครบถ้วน:

1. **REST Principles** - หลักการ 6 ข้อของ REST
2. **URL Structure** - การออกแบบ URL ที่ถูกต้อง
3. **HTTP Methods** - GET, POST, PUT, PATCH, DELETE
4. **Status Codes** - การใช้ status codes ที่เหมาะสม
5. **Request/Response** - โครงสร้าง JSON API format
6. **Error Responses** - structured error responses
7. **Pagination, Filtering, Sorting** - การจัดการข้อมูลจำนวนมาก
8. **API Versioning** - strategies ต่างๆ
9. **Authentication** - API Keys และ JWT
10. **Rate Limiting** - sliding window algorithm
11. **CORS** - cross-origin resource sharing
12. **Complete CRUD API** - Blog Posts API ที่สมบูรณ์
13. **Validation Library** - reusable validation system
14. **API Documentation** - OpenAPI spec generation

การออกแบบ REST API ที่ดีต้องสม่ำเสมอ, ชัดเจน, และมี error handling ที่ดี!
