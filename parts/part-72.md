# บทที่ 72: GraphQL Server ด้วย Lua

## บทนำ

GraphQL เป็น query language สำหรับ API ที่พัฒนาโดย Facebook (Meta) ในปี 2015 ให้ client สามารถระบุได้อย่างชัดเจนว่าต้องการข้อมูลอะไรบ้าง แก้ปัญหา over-fetching และ under-fetching ที่พบใน REST API

## GraphQL vs REST

```lua
-- เปรียบเทียบ REST vs GraphQL

-- REST: หลาย endpoints, ข้อมูลถูกกำหนดโดย server
-- GET /users/1
-- GET /users/1/posts
-- GET /users/1/followers

-- GraphQL: endpoint เดียว, client กำหนดข้อมูลที่ต้องการ
-- POST /graphql
-- {
--   user(id: 1) {
--     name
--     email
--     posts { title, createdAt }
--     followers { name }
--   }
-- }

local comparison = {
    {aspect = "Endpoints", rest = "หลาย endpoints", graphql = "Endpoint เดียว (/graphql)"},
    {aspect = "Data fetching", rest = "Server กำหนด response", graphql = "Client กำหนดข้อมูลที่ต้องการ"},
    {aspect = "Over-fetching", rest = "มักเกิดขึ้น", graphql = "ไม่เกิด"},
    {aspect = "Under-fetching", rest = "ต้องเรียกหลาย requests", graphql = "request เดียวได้ทุกอย่าง"},
    {aspect = "Type system", rest = "Optional (OpenAPI)", graphql = "Built-in"},
    {aspect = "Versioning", rest = "v1, v2, v3...", graphql = "Evolve schema"},
    {aspect = "Introspection", rest = "ต้องใช้ tools", graphql = "Built-in"},
    {aspect = "Real-time", rest = "WebSocket/SSE แยก", graphql = "Subscriptions built-in"},
}

print("GraphQL vs REST:")
print(string.format("%-20s %-30s %-30s", "Aspect", "REST", "GraphQL"))
print(string.rep("-", 80))
for _, row in ipairs(comparison) do
    print(string.format("%-20s %-30s %-30s", row.aspect, row.rest, row.graphql))
end
```

## Schema Definition Language (SDL)

```lua
-- GraphQL SDL (Schema Definition Language)
-- กำหนด types, queries, mutations, subscriptions

--[[
# Scalar types
scalar DateTime
scalar JSON
scalar Upload

# Enum type
enum UserRole {
  ADMIN
  USER
  MODERATOR
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

# Object type
type User {
  id: ID!
  name: String!
  email: String!
  role: UserRole!
  createdAt: DateTime!
  posts: [Post!]!
  followers: [User!]!
  followingCount: Int!
}

type Post {
  id: ID!
  title: String!
  content: String!
  status: PostStatus!
  author: User!
  tags: [String!]!
  createdAt: DateTime!
  comments: [Comment!]!
  likeCount: Int!
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
  createdAt: DateTime!
}

# Input types (สำหรับ mutations)
input CreateUserInput {
  name: String!
  email: String!
  password: String!
  role: UserRole = USER
}

input CreatePostInput {
  title: String!
  content: String!
  tags: [String!]
  status: PostStatus = DRAFT
}

# Root types
type Query {
  # Users
  user(id: ID!): User
  users(page: Int, perPage: Int): UserConnection!
  me: User
  
  # Posts
  post(id: ID!): Post
  posts(filter: PostFilter, page: Int): PostConnection!
  featuredPosts: [Post!]!
}

type Mutation {
  # User mutations
  createUser(input: CreateUserInput!): AuthPayload!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  
  # Post mutations
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  publishPost(id: ID!): Post!
  deletePost(id: ID!): Boolean!
  
  # Auth
  login(email: String!, password: String!): AuthPayload!
  logout: Boolean!
}

type Subscription {
  postCreated: Post!
  postUpdated(id: ID!): Post!
  commentAdded(postId: ID!): Comment!
  userOnline(ids: [ID!]!): User!
}
]]

-- จำลอง schema structure ใน Lua
local schema_types = {
    scalars = {"ID", "String", "Int", "Float", "Boolean", "DateTime", "JSON"},
    enums = {
        UserRole = {"ADMIN", "USER", "MODERATOR"},
        PostStatus = {"DRAFT", "PUBLISHED", "ARCHIVED"},
    },
    types = {
        "User", "Post", "Comment", "AuthPayload",
        "UserConnection", "PostConnection",
    }
}

print("GraphQL Schema Types:")
print("Scalars: " .. table.concat(schema_types.scalars, ", "))
print("Enums:")
for name, values in pairs(schema_types.enums) do
    print(string.format("  %s: [%s]", name, table.concat(values, ", ")))
end
```

## graphql-lua Library

```lua
-- graphql-lua library (โดย bjornbytes)
-- luarocks install graphql

-- ตัวอย่างการ setup graphql-lua
-- local graphql = require("graphql")
-- local types = require("graphql.types")

-- จำลอง graphql-lua API
local types = {}

-- Scalar types
types.string = {kind = "SCALAR", name = "String"}
types.int = {kind = "SCALAR", name = "Int"}
types.float = {kind = "SCALAR", name = "Float"}
types.boolean = {kind = "SCALAR", name = "Boolean"}
types.id = {kind = "SCALAR", name = "ID"}

-- NonNull wrapper
function types.nonNull(t)
    return {kind = "NON_NULL", ofType = t, name = t.name .. "!"}
end

-- List wrapper
function types.list(t)
    return {kind = "LIST", ofType = t, name = "[" .. t.name .. "]"}
end

-- Object type builder
function types.object(config)
    return {
        kind = "OBJECT",
        name = config.name,
        description = config.description,
        fields = config.fields,
        interfaces = config.interfaces or {}
    }
end

-- Enum type builder
function types.enum(config)
    return {
        kind = "ENUM",
        name = config.name,
        values = config.values,
        description = config.description
    }
end

-- Input type builder
function types.inputObject(config)
    return {
        kind = "INPUT_OBJECT",
        name = config.name,
        fields = config.fields,
        description = config.description
    }
end

-- ทดสอบ type building
local UserType = types.object({
    name = "User",
    description = "ผู้ใช้งานระบบ",
    fields = {
        id = {type = types.nonNull(types.id), description = "User ID"},
        name = {type = types.nonNull(types.string), description = "ชื่อผู้ใช้"},
        email = {type = types.nonNull(types.string), description = "อีเมล"},
        age = {type = types.int, description = "อายุ"},
    }
})

print("Type: " .. UserType.name)
print("Description: " .. UserType.description)
print("Fields:")
for field_name, field in pairs(UserType.fields) do
    print(string.format("  %s: %s", field_name, field.type.name))
end
```

## การสร้าง GraphQL Schema

### Database จำลอง

```lua
-- Database จำลองสำหรับตัวอย่าง

local db = {
    users = {
        [1] = {id = "1", name = "สมชาย ใจดี", email = "somchai@example.com", role = "USER", created_at = "2024-01-15"},
        [2] = {id = "2", name = "สุมาลี รักดี", email = "sumalee@example.com", role = "ADMIN", created_at = "2024-01-10"},
        [3] = {id = "3", name = "วิชัย มั่นคง", email = "wichai@example.com", role = "MODERATOR", created_at = "2024-02-01"},
        [4] = {id = "4", name = "ประทีป วิมล", email = "prateep@example.com", role = "USER", created_at = "2024-02-15"},
    },
    posts = {
        [1] = {id = "1", title = "เรียน GraphQL ครั้งแรก", content = "บทความแนะนำ GraphQL...", author_id = "1", status = "PUBLISHED", tags = {"graphql", "tutorial"}, created_at = "2024-03-01"},
        [2] = {id = "2", title = "Lua สำหรับ Backend", content = "การใช้ Lua พัฒนา backend...", author_id = "1", status = "PUBLISHED", tags = {"lua", "backend"}, created_at = "2024-03-10"},
        [3] = {id = "3", title = "Draft Post", content = "งานที่ยังไม่เสร็จ...", author_id = "2", status = "DRAFT", tags = {}, created_at = "2024-03-15"},
        [4] = {id = "4", title = "Performance Tips", content = "เทคนิคเพิ่มประสิทธิภาพ...", author_id = "3", status = "PUBLISHED", tags = {"performance", "tips"}, created_at = "2024-03-20"},
    },
    comments = {
        [1] = {id = "1", text = "บทความดีมาก!", post_id = "1", author_id = "2", created_at = "2024-03-02"},
        [2] = {id = "2", text = "ขอบคุณสำหรับข้อมูล", post_id = "1", author_id = "3", created_at = "2024-03-03"},
        [3] = {id = "3", text = "น่าสนใจมาก", post_id = "2", author_id = "4", created_at = "2024-03-11"},
    },
    follows = {
        -- user_id -> [follower_ids]
        ["1"] = {"2", "3"},
        ["2"] = {"1", "4"},
    }
}

-- Helper functions
local function find_user(id)
    return db.users[tonumber(id)]
end

local function find_post(id)
    return db.posts[tonumber(id)]
end

local function get_user_posts(user_id)
    local posts = {}
    for _, post in pairs(db.posts) do
        if post.author_id == user_id then
            table.insert(posts, post)
        end
    end
    return posts
end

local function get_post_comments(post_id)
    local comments = {}
    for _, comment in pairs(db.comments) do
        if comment.post_id == post_id then
            table.insert(comments, comment)
        end
    end
    return comments
end

-- ทดสอบ helper functions
print("User 1: " .. (find_user("1") and find_user("1").name or "not found"))
print("Posts by user 1: " .. #get_user_posts("1") .. " posts")
print("Comments on post 1: " .. #get_post_comments("1") .. " comments")
```

### Resolvers

```lua
-- Resolvers คือฟังก์ชันที่ resolve แต่ละ field ใน schema

local resolvers = {}

-- Root Query resolvers
resolvers.Query = {
    -- user(id: ID!): User
    user = function(root, args, context)
        local user = find_user(args.id)
        if not user then
            error({message = "User not found", code = "USER_NOT_FOUND"})
        end
        return user
    end,
    
    -- users(page: Int, perPage: Int): UserConnection
    users = function(root, args, context)
        local page = args.page or 1
        local per_page = args.perPage or 10
        
        local all_users = {}
        for _, user in pairs(db.users) do
            table.insert(all_users, user)
        end
        
        -- Sort by created_at
        table.sort(all_users, function(a, b)
            return a.created_at < b.created_at
        end)
        
        -- Pagination
        local start = (page - 1) * per_page + 1
        local paged_users = {}
        for i = start, math.min(start + per_page - 1, #all_users) do
            table.insert(paged_users, all_users[i])
        end
        
        return {
            nodes = paged_users,
            total = #all_users,
            page = page,
            perPage = per_page,
            hasNextPage = start + per_page - 1 < #all_users
        }
    end,
    
    -- me: User (ใช้ context)
    me = function(root, args, context)
        if not context.user_id then
            error({message = "Not authenticated", code = "UNAUTHENTICATED"})
        end
        return find_user(context.user_id)
    end,
    
    -- post(id: ID!): Post
    post = function(root, args, context)
        local post = find_post(args.id)
        if not post then
            error({message = "Post not found", code = "POST_NOT_FOUND"})
        end
        return post
    end,
    
    -- posts(filter: PostFilter): PostConnection
    posts = function(root, args, context)
        local filter = args.filter or {}
        local posts = {}
        
        for _, post in pairs(db.posts) do
            local include = true
            
            if filter.status and post.status ~= filter.status then
                include = false
            end
            
            if filter.authorId and post.author_id ~= filter.authorId then
                include = false
            end
            
            if filter.tag then
                local has_tag = false
                for _, tag in ipairs(post.tags) do
                    if tag == filter.tag then has_tag = true; break end
                end
                if not has_tag then include = false end
            end
            
            if include then
                table.insert(posts, post)
            end
        end
        
        return {nodes = posts, total = #posts}
    end,
}

-- Type resolvers (field-level)
resolvers.User = {
    -- Resolve user's posts
    posts = function(user, args, context)
        return get_user_posts(user.id)
    end,
    
    -- Resolve user's followers
    followers = function(user, args, context)
        local follower_ids = db.follows[user.id] or {}
        local followers = {}
        for _, fid in ipairs(follower_ids) do
            local follower = find_user(fid)
            if follower then
                table.insert(followers, follower)
            end
        end
        return followers
    end,
    
    -- Computed field
    followingCount = function(user, args, context)
        return #(db.follows[user.id] or {})
    end,
}

resolvers.Post = {
    -- Resolve post's author
    author = function(post, args, context)
        return find_user(post.author_id)
    end,
    
    -- Resolve post's comments
    comments = function(post, args, context)
        return get_post_comments(post.id)
    end,
    
    -- Computed field
    likeCount = function(post, args, context)
        -- จำลอง like count
        return math.random(0, 100)
    end,
}

resolvers.Comment = {
    author = function(comment, args, context)
        return find_user(comment.author_id)
    end,
    
    post = function(comment, args, context)
        return find_post(comment.post_id)
    end,
}

-- ทดสอบ resolvers
local mock_context = {user_id = "1"}

print("=== Query Resolver Tests ===")
local user = resolvers.Query.user(nil, {id = "1"}, mock_context)
print("User: " .. user.name)

local users_result = resolvers.Query.users(nil, {page = 1, perPage = 3}, mock_context)
print(string.format("Users: %d/%d (hasNext: %s)", 
    #users_result.nodes, users_result.total, tostring(users_result.hasNextPage)))

local posts_result = resolvers.Query.posts(nil, {filter = {status = "PUBLISHED"}}, mock_context)
print("Published posts: " .. #posts_result.nodes)

print("\n=== Type Resolver Tests ===")
local user_posts = resolvers.User.posts(user, {}, mock_context)
print(string.format("User '%s' has %d posts", user.name, #user_posts))

local followers = resolvers.User.followers(user, {}, mock_context)
print(string.format("User has %d followers", #followers))
```

## Mutation Resolvers

```lua
-- Mutation resolvers สำหรับ create/update/delete

resolvers.Mutation = {
    -- createUser
    createUser = function(root, args, context)
        local input = args.input
        
        -- Validation
        if not input.name or #input.name < 2 then
            error({message = "Name must be at least 2 characters", 
                   code = "VALIDATION_ERROR",
                   field = "name"})
        end
        
        if not input.email or not input.email:match("@") then
            error({message = "Invalid email format",
                   code = "VALIDATION_ERROR",
                   field = "email"})
        end
        
        -- ตรวจสอบ email ซ้ำ
        for _, user in pairs(db.users) do
            if user.email == input.email then
                error({message = "Email already exists",
                       code = "DUPLICATE_EMAIL"})
            end
        end
        
        -- สร้าง user ใหม่
        local new_id = tostring(#db.users + 1)
        local new_user = {
            id = new_id,
            name = input.name,
            email = input.email,
            role = input.role or "USER",
            created_at = os.date("%Y-%m-%d"),
        }
        db.users[tonumber(new_id)] = new_user
        
        -- สร้าง JWT token (จำลอง)
        local token = "jwt." .. new_id .. "." .. os.time()
        
        return {
            user = new_user,
            token = token
        }
    end,
    
    -- createPost
    createPost = function(root, args, context)
        if not context.user_id then
            error({message = "Must be logged in", code = "UNAUTHENTICATED"})
        end
        
        local input = args.input
        
        if not input.title or #input.title < 3 then
            error({message = "Title must be at least 3 characters",
                   code = "VALIDATION_ERROR"})
        end
        
        local new_id = tostring(#db.posts + 1)
        local new_post = {
            id = new_id,
            title = input.title,
            content = input.content or "",
            author_id = context.user_id,
            status = input.status or "DRAFT",
            tags = input.tags or {},
            created_at = os.date("%Y-%m-%d"),
        }
        db.posts[tonumber(new_id)] = new_post
        
        print(string.format("Post created: '%s' by user %s", 
            new_post.title, context.user_id))
        
        return new_post
    end,
    
    -- publishPost
    publishPost = function(root, args, context)
        if not context.user_id then
            error({message = "Must be logged in", code = "UNAUTHENTICATED"})
        end
        
        local post = find_post(args.id)
        if not post then
            error({message = "Post not found", code = "NOT_FOUND"})
        end
        
        -- ตรวจสอบว่าเป็น author
        if post.author_id ~= context.user_id then
            local user = find_user(context.user_id)
            if not user or user.role ~= "ADMIN" then
                error({message = "Not authorized to publish this post",
                       code = "FORBIDDEN"})
            end
        end
        
        post.status = "PUBLISHED"
        post.published_at = os.date("%Y-%m-%dT%H:%M:%SZ")
        
        return post
    end,
    
    -- deletePost
    deletePost = function(root, args, context)
        if not context.user_id then
            error({message = "Must be logged in", code = "UNAUTHENTICATED"})
        end
        
        local post = find_post(args.id)
        if not post then
            error({message = "Post not found", code = "NOT_FOUND"})
        end
        
        if post.author_id ~= context.user_id then
            error({message = "Not authorized", code = "FORBIDDEN"})
        end
        
        db.posts[tonumber(args.id)] = nil
        return true
    end,
}

-- ทดสอบ mutations
print("=== Mutation Tests ===")

-- createUser
local ok, result = pcall(function()
    return resolvers.Mutation.createUser(nil, {
        input = {
            name = "ผู้ใช้ใหม่",
            email = "newuser@example.com",
            password = "secret123",
        }
    }, {})
end)
if ok then
    print("Created user: " .. result.user.name .. ", token: " .. result.token:sub(1, 20) .. "...")
else
    print("Error: " .. (type(result) == "table" and result.message or tostring(result)))
end

-- createPost
ok, result = pcall(function()
    return resolvers.Mutation.createPost(nil, {
        input = {
            title = "บทความทดสอบ GraphQL",
            content = "เนื้อหาบทความ...",
            tags = {"test", "graphql"},
        }
    }, {user_id = "1"})
end)
if ok then
    print("Created post: " .. result.title)
end
```

## Query Execution Engine (จำลอง)

```lua
-- จำลอง GraphQL query execution

local function parse_selection(fields_str)
    -- Parser อย่างง่าย
    local fields = {}
    for field in fields_str:gmatch("[%w_]+") do
        table.insert(fields, field)
    end
    return fields
end

-- Simple query executor
local function execute_query(query_type, operation, args, context, schema_resolvers)
    local resolver = schema_resolvers[query_type]
    if not resolver then
        return nil, "No resolver for " .. query_type
    end
    
    local field_resolver = resolver[operation]
    if not field_resolver then
        return nil, "No resolver for " .. query_type .. "." .. operation
    end
    
    local ok, result = pcall(field_resolver, nil, args, context)
    if not ok then
        if type(result) == "table" then
            return nil, result
        end
        return nil, {message = tostring(result), code = "INTERNAL_ERROR"}
    end
    
    return result, nil
end

-- ทดสอบ query execution
print("=== Query Execution ===")

-- สำเร็จ
local result, err = execute_query("Query", "user", {id = "2"}, mock_context, resolvers)
if result then
    print("user(id: 2) = " .. result.name .. " [" .. result.role .. "]")
else
    print("Error: " .. (type(err) == "table" and err.message or tostring(err)))
end

-- ไม่พบ user
result, err = execute_query("Query", "user", {id = "999"}, mock_context, resolvers)
if not result then
    print("user(id: 999) error: " .. (type(err) == "table" and err.message or tostring(err)))
end

-- Posts with filter
result, err = execute_query("Query", "posts", 
    {filter = {status = "PUBLISHED"}}, mock_context, resolvers)
if result then
    print(string.format("posts(status: PUBLISHED) = %d posts", #result.nodes))
    for _, post in ipairs(result.nodes) do
        print("  - " .. post.title)
    end
end
```

## Variables ใน GraphQL

```lua
-- Variables ทำให้ queries เป็น reusable และปลอดภัยจาก injection

-- ตัวอย่าง query ที่ใช้ variables
--[[
query GetUser($userId: ID!) {
    user(id: $userId) {
        name
        email
        posts {
            title
            status
        }
    }
}
]]

-- Variables จะถูกส่งแยกจาก query
local query_variables = {
    userId = "1"
}

-- ตัวอย่าง mutation ที่ใช้ variables
--[[
mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
        id
        title
        status
        author {
            name
        }
    }
}
]]

local mutation_variables = {
    input = {
        title = "บทความใหม่",
        content = "เนื้อหา...",
        tags = {"lua", "tutorial"},
        status = "DRAFT"
    }
}

-- Variable validation
local function validate_variables(variables, required_vars)
    local errors = {}
    for var_name, var_def in pairs(required_vars) do
        local value = variables[var_name]
        
        if var_def.required and value == nil then
            table.insert(errors, string.format(
                "Variable '%s' is required but not provided", var_name))
        elseif value ~= nil then
            -- Type checking (simplified)
            if var_def.type == "ID" or var_def.type == "String" then
                if type(value) ~= "string" then
                    table.insert(errors, string.format(
                        "Variable '%s' must be a string", var_name))
                end
            elseif var_def.type == "Int" then
                if type(value) ~= "number" or math.floor(value) ~= value then
                    table.insert(errors, string.format(
                        "Variable '%s' must be an integer", var_name))
                end
            elseif var_def.type == "Boolean" then
                if type(value) ~= "boolean" then
                    table.insert(errors, string.format(
                        "Variable '%s' must be a boolean", var_name))
                end
            end
        end
    end
    return errors
end

-- ทดสอบ variable validation
local required_vars = {
    userId = {type = "ID", required = true},
    page = {type = "Int", required = false},
}

print("=== Variable Validation ===")

local errors = validate_variables({userId = "123", page = 1}, required_vars)
print("Valid variables: " .. (#errors == 0 and "OK" or table.concat(errors, ", ")))

errors = validate_variables({page = 1.5}, required_vars)  -- missing userId, invalid page
print("Invalid variables:")
for _, e in ipairs(errors) do
    print("  - " .. e)
end
```

## Fragments

```lua
-- Fragments ช่วยลดการซ้ำซ้อนใน queries

--[[
# Fragment definition
fragment UserBasic on User {
    id
    name
    email
}

fragment UserFull on User {
    ...UserBasic
    role
    createdAt
    followingCount
}

fragment PostSummary on Post {
    id
    title
    status
    createdAt
    author {
        ...UserBasic
    }
}

# ใช้ fragments ใน query
query GetUserWithPosts($id: ID!) {
    user(id: $id) {
        ...UserFull
        posts {
            ...PostSummary
            tags
        }
    }
}
]]

-- Fragment system ใน Lua
local FragmentRegistry = {}
FragmentRegistry.__index = FragmentRegistry

function FragmentRegistry.new()
    return setmetatable({fragments = {}}, FragmentRegistry)
end

function FragmentRegistry:register(name, type_name, fields)
    self.fragments[name] = {
        name = name,
        on_type = type_name,
        fields = fields
    }
    print(string.format("Fragment '%s' registered on type '%s'", name, type_name))
end

function FragmentRegistry:get(name)
    return self.fragments[name]
end

function FragmentRegistry:spread(name, object)
    local fragment = self:get(name)
    if not fragment then
        error("Fragment '" .. name .. "' not found")
    end
    
    local result = {}
    for _, field_name in ipairs(fragment.fields) do
        if field_name:match("^%.%.%.") then
            -- Nested fragment spread
            local nested_name = field_name:sub(4)
            local nested = self:spread(nested_name, object)
            for k, v in pairs(nested) do
                result[k] = v
            end
        else
            result[field_name] = object[field_name]
        end
    end
    return result
end

-- ทดสอบ fragments
local registry = FragmentRegistry.new()

registry:register("UserBasic", "User", {"id", "name", "email"})
registry:register("UserFull", "User", {"...UserBasic", "role", "created_at"})

local user = find_user("1")
print("\nFull user data:")
for k, v in pairs(user) do
    print(string.format("  %s: %s", k, tostring(v)))
end

local basic_fragment = registry:spread("UserBasic", user)
print("\nUserBasic fragment spread:")
for k, v in pairs(basic_fragment) do
    print(string.format("  %s: %s", k, tostring(v)))
end
```

## Directives

```lua
-- Directives ใน GraphQL ควบคุมการ execution

--[[
# Built-in directives
query GetUser($id: ID!, $showEmail: Boolean!, $preview: Boolean = false) {
    user(id: $id) {
        name
        email @include(if: $showEmail)      # รวม field นี้ถ้า showEmail = true
        role @skip(if: $preview)             # ข้าม field นี้ถ้า preview = true
        bio @deprecated(reason: "Use description instead")
    }
}
]]

-- Directive processor
local DirectiveProcessor = {}

-- @include directive
function DirectiveProcessor.include(field_value, args)
    if args["if"] == false then
        return nil  -- ไม่รวม field
    end
    return field_value
end

-- @skip directive
function DirectiveProcessor.skip(field_value, args)
    if args["if"] == true then
        return nil  -- ข้าม field
    end
    return field_value
end

-- Custom @auth directive
function DirectiveProcessor.auth(field_value, args, context)
    local required_role = args.requires or "USER"
    
    if not context.user_id then
        error({message = "Authentication required", code = "UNAUTHENTICATED"})
    end
    
    local user = find_user(context.user_id)
    if not user then
        error({message = "User not found", code = "NOT_FOUND"})
    end
    
    -- Role hierarchy: ADMIN > MODERATOR > USER
    local role_level = {USER = 1, MODERATOR = 2, ADMIN = 3}
    
    if (role_level[user.role] or 0) < (role_level[required_role] or 0) then
        error({message = "Insufficient permissions", code = "FORBIDDEN"})
    end
    
    return field_value
end

-- Custom @rateLimit directive
local rate_limit_state = {}
function DirectiveProcessor.rateLimit(field_value, args, context)
    local max = args.max or 10
    local window = args.window or 60  -- seconds
    
    local key = (context.user_id or "anonymous") .. ":rateLimit"
    local now = os.time()
    
    if not rate_limit_state[key] then
        rate_limit_state[key] = {count = 0, reset_at = now + window}
    end
    
    local state = rate_limit_state[key]
    
    if now > state.reset_at then
        state.count = 0
        state.reset_at = now + window
    end
    
    state.count = state.count + 1
    
    if state.count > max then
        error({message = "Rate limit exceeded", code = "RATE_LIMITED"})
    end
    
    return field_value
end

-- จำลองการ apply directives
local function apply_directives(field_value, directives, context)
    for _, directive in ipairs(directives) do
        if DirectiveProcessor[directive.name] then
            field_value = DirectiveProcessor[directive.name](
                field_value, directive.args or {}, context or {})
            if field_value == nil then
                return nil  -- Field excluded
            end
        end
    end
    return field_value
end

-- ทดสอบ directives
print("=== Directive Tests ===")

local user_data = find_user("2")  -- ADMIN user

-- @include(if: true)
local email = apply_directives(user_data.email, 
    {{name = "include", args = {["if"] = true}}}, mock_context)
print("@include(if: true) email: " .. (email or "excluded"))

-- @include(if: false)
email = apply_directives(user_data.email,
    {{name = "include", args = {["if"] = false}}}, mock_context)
print("@include(if: false) email: " .. (email or "excluded"))

-- @skip(if: true)
local role = apply_directives(user_data.role,
    {{name = "skip", args = {["if"] = true}}}, mock_context)
print("@skip(if: true) role: " .. (role or "skipped"))

-- @auth directive
local ok, result = pcall(apply_directives, "sensitive_data",
    {{name = "auth", args = {requires = "ADMIN"}}},
    {user_id = "1"})  -- USER role
if not ok then
    print("@auth(ADMIN) with USER role: " .. 
        (type(result) == "table" and result.message or tostring(result)))
end

ok, result = pcall(apply_directives, "sensitive_data",
    {{name = "auth", args = {requires = "ADMIN"}}},
    {user_id = "2"})  -- ADMIN role
if ok then
    print("@auth(ADMIN) with ADMIN role: " .. result)
end
```

## N+1 Problem และ DataLoader

```lua
-- N+1 Problem: เกิดเมื่อ resolver เรียก database หลายครั้งโดยไม่จำเป็น

-- ปัญหา: ถ้ามี 10 posts แต่ละ post เรียก author resolver
-- จะเกิด 1 (posts query) + 10 (author queries) = 11 queries

-- DataLoader แก้ปัญหาโดย batch queries

local DataLoader = {}
DataLoader.__index = DataLoader

function DataLoader.new(batch_fn, options)
    local self = setmetatable({}, DataLoader)
    self.batch_fn = batch_fn
    self.options = options or {}
    self.max_batch_size = options.max_batch_size or 1000
    self.cache_enabled = options.cache ~= false
    self.cache = {}
    self.queue = {}
    self.scheduled = false
    return self
end

function DataLoader:load(key)
    -- ตรวจสอบ cache
    if self.cache_enabled and self.cache[key] then
        return self.cache[key]
    end
    
    -- เพิ่ม key เข้า queue
    table.insert(self.queue, key)
    
    -- จะ dispatch batch เมื่อ queue ไม่ว่าง
    if not self.scheduled then
        self.scheduled = true
        -- ใน production จะใช้ event loop / coroutine
        -- ที่นี่จะ dispatch ทันที
        self:dispatch()
    end
    
    return self.cache[key]
end

function DataLoader:load_many(keys)
    local results = {}
    for _, key in ipairs(keys) do
        table.insert(results, self:load(key))
    end
    return results
end

function DataLoader:dispatch()
    if #self.queue == 0 then
        self.scheduled = false
        return
    end
    
    -- รวบรวม unique keys
    local unique_keys = {}
    local seen = {}
    for _, key in ipairs(self.queue) do
        if not seen[key] then
            seen[key] = true
            table.insert(unique_keys, key)
        end
    end
    
    print(string.format("DataLoader: Batching %d keys (from %d requests)", 
        #unique_keys, #self.queue))
    
    -- เรียก batch function
    local results = self.batch_fn(unique_keys)
    
    -- เก็บใน cache
    for _, key in ipairs(unique_keys) do
        self.cache[key] = results[key]
    end
    
    self.queue = {}
    self.scheduled = false
end

-- User loader
local function batch_users(ids)
    print(string.format("DB Query: SELECT * FROM users WHERE id IN (%s)",
        table.concat(ids, ", ")))
    
    local results = {}
    for _, id in ipairs(ids) do
        results[id] = db.users[tonumber(id)]
    end
    return results
end

local userLoader = DataLoader.new(batch_users, {cache = true})

-- จำลองการแก้ N+1 problem
print("=== N+1 Problem Demo ===")
print("\nWithout DataLoader (N+1):")
local post_count = 0
for _, post in pairs(db.posts) do
    -- แต่ละ post เรียก DB แยกกัน
    local author = db.users[tonumber(post.author_id)]  -- 1 query per post
    post_count = post_count + 1
    -- print(string.format("  Post: %s, Author: %s", post.title, author and author.name or "?"))
end
print(string.format("Total DB queries: 1 (posts) + %d (authors) = %d", 
    post_count, post_count + 1))

print("\nWith DataLoader (batched):")
-- Pre-collect all author IDs
local author_ids = {}
local seen_ids = {}
for _, post in pairs(db.posts) do
    if not seen_ids[post.author_id] then
        seen_ids[post.author_id] = true
        table.insert(author_ids, post.author_id)
    end
end

-- Load all at once
local authors = userLoader:load_many(author_ids)
print(string.format("Total DB queries: 1 (posts) + 1 (authors batch) = 2"))
print(string.format("Loaded %d authors in 1 query", #authors))
```

## Authentication ใน GraphQL

```lua
-- Authentication strategies สำหรับ GraphQL

local Auth = {}

-- Token validation
function Auth.verify_token(token)
    if not token then
        return nil, "No token provided"
    end
    
    -- จำลอง JWT verification
    if token:match("^Bearer ") then
        token = token:sub(8)
    end
    
    -- ใน production จะ verify JWT signature
    local parts = {}
    for part in token:gmatch("[^.]+") do
        table.insert(parts, part)
    end
    
    if #parts < 2 then
        return nil, "Invalid token format"
    end
    
    -- จำลอง decode payload
    local user_id = parts[2]
    local user = find_user(user_id)
    
    if not user then
        return nil, "User not found"
    end
    
    return user, nil
end

-- Context builder (middleware)
function Auth.build_context(request)
    local context = {
        request = request,
        user = nil,
        user_id = nil,
        permissions = {},
    }
    
    -- Extract token from header
    local auth_header = request.headers and request.headers["Authorization"]
    
    if auth_header then
        local user, err = Auth.verify_token(auth_header)
        if user then
            context.user = user
            context.user_id = user.id
            
            -- Set permissions based on role
            if user.role == "ADMIN" then
                context.permissions = {"read", "write", "delete", "admin"}
            elseif user.role == "MODERATOR" then
                context.permissions = {"read", "write", "moderate"}
            else
                context.permissions = {"read", "write:own"}
            end
        else
            print("Auth warning: " .. err)
        end
    end
    
    return context
end

-- Permission checking
function Auth.check_permission(context, permission)
    for _, perm in ipairs(context.permissions) do
        if perm == permission or perm == "admin" then
            return true
        end
    end
    return false
end

-- Protected resolver wrapper
function Auth.protected(resolver, required_permission)
    return function(root, args, context)
        if not context.user_id then
            error({
                message = "You must be logged in",
                code = "UNAUTHENTICATED",
                extensions = {code = "UNAUTHENTICATED"}
            })
        end
        
        if required_permission and not Auth.check_permission(context, required_permission) then
            error({
                message = string.format(
                    "You don't have '%s' permission", required_permission),
                code = "FORBIDDEN",
                extensions = {code = "FORBIDDEN"}
            })
        end
        
        return resolver(root, args, context)
    end
end

-- ทดสอบ authentication
print("=== Authentication Tests ===")

-- สร้าง context จาก request
local admin_context = Auth.build_context({
    headers = {Authorization = "Bearer 2"}  -- user ID 2 = ADMIN
})
print(string.format("Admin context: user=%s, role=%s, perms=[%s]",
    admin_context.user and admin_context.user.name or "none",
    admin_context.user and admin_context.user.role or "none",
    table.concat(admin_context.permissions, ", ")))

local user_context = Auth.build_context({
    headers = {Authorization = "Bearer 1"}  -- user ID 1 = USER
})
print(string.format("User context: user=%s, role=%s, perms=[%s]",
    user_context.user and user_context.user.name or "none",
    user_context.user and user_context.user.role or "none",
    table.concat(user_context.permissions, ", ")))

-- Protected resolver
local protected_delete = Auth.protected(
    function(root, args, context)
        return {success = true, id = args.id}
    end,
    "delete"
)

print("\nTesting protected delete:")
local ok, result = pcall(protected_delete, nil, {id = "1"}, user_context)
if not ok then
    print("USER: " .. (type(result) == "table" and result.message or tostring(result)))
end

ok, result = pcall(protected_delete, nil, {id = "1"}, admin_context)
if ok then
    print("ADMIN: Success (id=" .. result.id .. ")")
end
```

## Cursor-based Pagination

```lua
-- Cursor-based pagination (Relay style)

local Pagination = {}

-- Encode/decode cursor
function Pagination.encode_cursor(id, created_at)
    -- ใน production จะใช้ base64
    return string.format("cursor:%s:%s", id, created_at)
end

function Pagination.decode_cursor(cursor)
    local id, created_at = cursor:match("^cursor:(.+):(.+)$")
    return id, created_at
end

-- Connection type
function Pagination.create_connection(items, args)
    local first = args.first or 10
    local after = args.after
    local last = args.last
    local before = args.before
    
    -- หา start position จาก cursor
    local start_idx = 1
    if after then
        local after_id = Pagination.decode_cursor(after)
        for i, item in ipairs(items) do
            if item.id == after_id then
                start_idx = i + 1
                break
            end
        end
    end
    
    -- ดึงข้อมูล
    local paged_items = {}
    local end_idx = math.min(start_idx + first - 1, #items)
    
    for i = start_idx, end_idx do
        table.insert(paged_items, items[i])
    end
    
    -- สร้าง edges และ cursors
    local edges = {}
    for _, item in ipairs(paged_items) do
        table.insert(edges, {
            node = item,
            cursor = Pagination.encode_cursor(item.id, item.created_at or "")
        })
    end
    
    -- PageInfo
    local page_info = {
        has_next_page = end_idx < #items,
        has_previous_page = start_idx > 1,
        start_cursor = #edges > 0 and edges[1].cursor or nil,
        end_cursor = #edges > 0 and edges[#edges].cursor or nil,
    }
    
    return {
        edges = edges,
        page_info = page_info,
        total_count = #items,
    }
end

-- ทดสอบ cursor pagination
print("=== Cursor-based Pagination ===")

-- สร้าง list ของ posts ที่ sort แล้ว
local all_posts = {}
for _, post in pairs(db.posts) do
    if post.status == "PUBLISHED" then
        table.insert(all_posts, post)
    end
end
table.sort(all_posts, function(a, b) return a.created_at < b.created_at end)

-- First page
local connection = Pagination.create_connection(all_posts, {first = 2})
print(string.format("First 2 items (total: %d):", connection.total_count))
for _, edge in ipairs(connection.edges) do
    print(string.format("  - %s [cursor: %s]", 
        edge.node.title, edge.cursor:sub(1, 30) .. "..."))
end
print(string.format("  hasNextPage: %s, hasPrevPage: %s",
    tostring(connection.page_info.has_next_page),
    tostring(connection.page_info.has_previous_page)))

-- Next page using cursor
if connection.page_info.end_cursor then
    local next_connection = Pagination.create_connection(all_posts, {
        first = 2,
        after = connection.page_info.end_cursor
    })
    print(string.format("\nNext 2 items (after cursor):"))
    for _, edge in ipairs(next_connection.edges) do
        print(string.format("  - %s", edge.node.title))
    end
    print(string.format("  hasNextPage: %s", 
        tostring(next_connection.page_info.has_next_page)))
end
```

## Subscriptions

```lua
-- GraphQL Subscriptions สำหรับ real-time updates

local SubscriptionManager = {}
SubscriptionManager.__index = SubscriptionManager

function SubscriptionManager.new()
    local self = setmetatable({}, SubscriptionManager)
    self.subscriptions = {}
    self.topic_subscribers = {}
    return self
end

function SubscriptionManager:subscribe(subscription_id, topic, filter_fn, callback)
    self.subscriptions[subscription_id] = {
        id = subscription_id,
        topic = topic,
        filter = filter_fn,
        callback = callback,
        created_at = os.time(),
    }
    
    if not self.topic_subscribers[topic] then
        self.topic_subscribers[topic] = {}
    end
    table.insert(self.topic_subscribers[topic], subscription_id)
    
    print(string.format("Subscribed: id=%s, topic=%s", subscription_id, topic))
end

function SubscriptionManager:unsubscribe(subscription_id)
    local sub = self.subscriptions[subscription_id]
    if sub then
        -- Remove from topic list
        local topic_subs = self.topic_subscribers[sub.topic] or {}
        for i, sid in ipairs(topic_subs) do
            if sid == subscription_id then
                table.remove(topic_subs, i)
                break
            end
        end
        
        self.subscriptions[subscription_id] = nil
        print("Unsubscribed: " .. subscription_id)
    end
end

function SubscriptionManager:publish(topic, data)
    local subscribers = self.topic_subscribers[topic] or {}
    local delivered = 0
    
    print(string.format("Publishing to topic '%s': %d subscribers", topic, #subscribers))
    
    for _, sub_id in ipairs(subscribers) do
        local sub = self.subscriptions[sub_id]
        if sub then
            -- Apply filter
            if not sub.filter or sub.filter(data) then
                sub.callback(data)
                delivered = delivered + 1
            end
        end
    end
    
    return delivered
end

-- Subscription resolvers
resolvers.Subscription = {
    -- postCreated: ส่งทุกครั้งที่มี post ใหม่
    postCreated = {
        subscribe = function(root, args, context)
            return "POST_CREATED"  -- topic
        end,
        resolve = function(event, args, context)
            return event.post
        end,
    },
    
    -- commentAdded: filter by postId
    commentAdded = {
        subscribe = function(root, args, context)
            return "COMMENT_ADDED:" .. args.postId
        end,
        resolve = function(event, args, context)
            return event.comment
        end,
        filter = function(event, args)
            return event.post_id == args.postId
        end,
    },
    
    -- userOnline
    userOnline = {
        subscribe = function(root, args, context)
            return "USER_STATUS"
        end,
        resolve = function(event, args, context)
            return event.user
        end,
        filter = function(event, args)
            for _, id in ipairs(args.ids or {}) do
                if id == event.user.id then return true end
            end
            return false
        end,
    },
}

-- ทดสอบ subscriptions
local sub_manager = SubscriptionManager.new()
print("=== Subscription Tests ===")

-- Subscribe ให้ client หลายคน
local received = {}

sub_manager:subscribe("sub-1", "POST_CREATED", nil, function(data)
    table.insert(received, {client = "sub-1", post = data.post})
    print(string.format("  Client sub-1 received: new post '%s'", data.post.title))
end)

sub_manager:subscribe("sub-2", "POST_CREATED", nil, function(data)
    table.insert(received, {client = "sub-2", post = data.post})
    print(string.format("  Client sub-2 received: new post '%s'", data.post.title))
end)

-- Publish event
print("\nPublishing new post event...")
local new_post_event = {
    post = {id = "99", title = "Breaking: New Article!", status = "PUBLISHED"}
}
local count = sub_manager:publish("POST_CREATED", new_post_event)
print(string.format("Delivered to %d subscribers", count))

-- Unsubscribe
sub_manager:unsubscribe("sub-1")

print("\nPublishing after unsubscribe...")
count = sub_manager:publish("POST_CREATED", {
    post = {id = "100", title = "Another Article", status = "PUBLISHED"}
})
print(string.format("Delivered to %d subscribers", count))
```

## Error Handling ใน GraphQL

```lua
-- GraphQL Error handling

local GraphQLError = {}

function GraphQLError.new(message, options)
    options = options or {}
    return {
        message = message,
        locations = options.locations,  -- [{line, column}]
        path = options.path,            -- path ใน response
        extensions = options.extensions or {},
    }
end

-- Error formatter
function GraphQLError.format(err)
    if type(err) == "string" then
        return GraphQLError.new(err)
    elseif type(err) == "table" then
        return {
            message = err.message or "Unknown error",
            extensions = {
                code = err.code or "INTERNAL_ERROR",
                details = err.details,
            }
        }
    end
    return GraphQLError.new("Unknown error")
end

-- Response format
local function create_response(data, errors)
    local response = {}
    
    if data ~= nil then
        response.data = data
    end
    
    if errors and #errors > 0 then
        local formatted_errors = {}
        for _, err in ipairs(errors) do
            table.insert(formatted_errors, GraphQLError.format(err))
        end
        response.errors = formatted_errors
    end
    
    return response
end

-- Partial response (data + errors)
-- GraphQL อนุญาตให้ return ข้อมูลบางส่วนพร้อม errors

local function execute_with_error_handling(operations, context)
    local data = {}
    local errors = {}
    
    for field_name, operation in pairs(operations) do
        local ok, result = pcall(function()
            return operation(context)
        end)
        
        if ok then
            data[field_name] = result
        else
            -- เพิ่ม null สำหรับ field ที่ error
            data[field_name] = nil
            table.insert(errors, GraphQLError.format(
                type(result) == "table" and result or {
                    message = tostring(result),
                    path = {field_name}
                }
            ))
        end
    end
    
    return create_response(data, errors)
end

-- ทดสอบ error handling
print("=== Error Handling Tests ===")

local response = execute_with_error_handling({
    user = function(ctx)
        return resolvers.Query.user(nil, {id = "1"}, ctx)
    end,
    nonExistentUser = function(ctx)
        return resolvers.Query.user(nil, {id = "999"}, ctx)
    end,
    posts = function(ctx)
        return resolvers.Query.posts(nil, {}, ctx)
    end,
}, mock_context)

print("Data fields:")
for k, v in pairs(response.data) do
    if v then
        if type(v) == "table" and v.name then
            print(string.format("  %s: %s", k, v.name))
        elseif type(v) == "table" and v.nodes then
            print(string.format("  %s: %d items", k, #v.nodes))
        else
            print(string.format("  %s: nil (error)", k))
        end
    else
        print(string.format("  %s: null (error occurred)", k))
    end
end

if response.errors then
    print("Errors:")
    for _, err in ipairs(response.errors) do
        print(string.format("  - %s (code: %s)", 
            err.message, 
            err.extensions and err.extensions.code or "unknown"))
    end
end
```

## Introspection

```lua
-- GraphQL Introspection ช่วยให้ client query เพื่อรู้ schema

local IntrospectionSchema = {}

-- Type representations
local type_defs = {
    User = {
        kind = "OBJECT",
        name = "User",
        description = "ผู้ใช้งานในระบบ",
        fields = {
            {name = "id", type = {kind = "NON_NULL", ofType = {kind = "SCALAR", name = "ID"}}, description = "User ID"},
            {name = "name", type = {kind = "NON_NULL", ofType = {kind = "SCALAR", name = "String"}}, description = "ชื่อผู้ใช้"},
            {name = "email", type = {kind = "NON_NULL", ofType = {kind = "SCALAR", name = "String"}}, description = "อีเมล"},
            {name = "role", type = {kind = "NON_NULL", ofType = {kind = "ENUM", name = "UserRole"}}, description = "บทบาท"},
            {name = "posts", type = {kind = "LIST", ofType = {kind = "OBJECT", name = "Post"}}, description = "โพสต์ของผู้ใช้"},
        }
    },
    Post = {
        kind = "OBJECT",
        name = "Post",
        description = "บทความ",
        fields = {
            {name = "id", type = {kind = "NON_NULL", ofType = {kind = "SCALAR", name = "ID"}}},
            {name = "title", type = {kind = "NON_NULL", ofType = {kind = "SCALAR", name = "String"}}},
            {name = "author", type = {kind = "NON_NULL", ofType = {kind = "OBJECT", name = "User"}}},
            {name = "status", type = {kind = "ENUM", name = "PostStatus"}},
        }
    },
    UserRole = {
        kind = "ENUM",
        name = "UserRole",
        enumValues = {"ADMIN", "USER", "MODERATOR"},
    }
}

-- __schema introspection
function IntrospectionSchema.get_schema()
    return {
        queryType = {name = "Query"},
        mutationType = {name = "Mutation"},
        subscriptionType = {name = "Subscription"},
        types = type_defs,
    }
end

-- __type introspection
function IntrospectionSchema.get_type(type_name)
    return type_defs[type_name]
end

-- ทดสอบ introspection
print("=== Schema Introspection ===")

local schema = IntrospectionSchema.get_schema()
print("Root types:")
print("  Query: " .. schema.queryType.name)
print("  Mutation: " .. schema.mutationType.name)
print("  Subscription: " .. schema.subscriptionType.name)

local user_type = IntrospectionSchema.get_type("User")
if user_type then
    print("\nUser type fields:")
    for _, field in ipairs(user_type.fields) do
        local type_str
        if field.type.kind == "NON_NULL" then
            type_str = field.type.ofType.name .. "!"
        else
            type_str = field.type.name or (field.type.ofType and field.type.ofType.name) or "?"
        end
        print(string.format("  %s: %s", field.name, type_str))
    end
end
```

## GraphQL Playground Setup

```lua
-- Setup GraphQL Playground สำหรับ development

local function create_playground_html(endpoint_url)
    return string.format([[
<!DOCTYPE html>
<html>
<head>
    <title>GraphQL Playground</title>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/graphql-playground-react/build/static/css/index.css" />
    <link rel="shortcut icon" href="https://cdn.jsdelivr.net/npm/graphql-playground-react/build/favicon.png" />
    <script src="https://cdn.jsdelivr.net/npm/graphql-playground-react/build/static/js/middleware.js"></script>
</head>
<body>
    <div id="root" />
    <script>
        window.addEventListener('load', function(event) {
            GraphQLPlayground.init(document.getElementById('root'), {
                endpoint: '%s',
                settings: {
                    'editor.theme': 'dark',
                    'editor.cursorShape': 'block',
                    'request.credentials': 'include',
                }
            });
        });
    </script>
</body>
</html>]], endpoint_url)
end

-- HTTP server handler สำหรับ GraphQL
local function handle_graphql_request(method, path, body, headers)
    -- Serve playground
    if method == "GET" and path == "/graphql" then
        return {
            status = 200,
            headers = {["Content-Type"] = "text/html"},
            body = create_playground_html("http://localhost:8080/graphql")
        }
    end
    
    -- Handle GraphQL queries
    if method == "POST" and path == "/graphql" then
        -- Parse request body (ใน production จะ parse JSON)
        local query = body.query
        local variables = body.variables or {}
        local operation_name = body.operationName
        
        print(string.format("GraphQL request: operation=%s", 
            operation_name or "unnamed"))
        
        -- Execute query
        local context = Auth.build_context({headers = headers})
        
        -- จำลอง response
        return {
            status = 200,
            headers = {["Content-Type"] = "application/json"},
            body = {
                data = {
                    __typename = "Query"
                }
            }
        }
    end
    
    return {status = 404, body = "Not Found"}
end

print("=== GraphQL Server Setup ===")
local html_size = #create_playground_html("http://localhost:8080/graphql")
print(string.format("Playground HTML size: %d bytes", html_size))

local response = handle_graphql_request("GET", "/graphql", nil, {})
print(string.format("GET /graphql -> status: %d, content-type: %s",
    response.status, response.headers["Content-Type"]))
```

## สรุปบทที่ 72

```lua
-- สรุปสิ่งที่ได้เรียนรู้ในบทที่ 72

local topics = {
    "GraphQL vs REST - เปรียบเทียบข้อดีข้อเสีย",
    "Schema Definition Language - กำหนด types, queries, mutations",
    "Types: Query, Mutation, Subscription - 3 root operations",
    "Resolvers - ฟังก์ชัน resolve แต่ละ field",
    "graphql-lua library - การใช้งาน",
    "Query Execution - การ execute queries",
    "Variables - reusable และปลอดภัย",
    "Fragments - ลดความซ้ำซ้อน",
    "Directives - @include, @skip, custom directives",
    "N+1 Problem - ปัญหาและวิธีแก้ด้วย DataLoader",
    "Authentication - context-based auth",
    "Cursor-based Pagination - Relay style",
    "Subscriptions - real-time updates",
    "Error Handling - partial responses",
    "Introspection - schema discovery",
    "Playground - development tools",
}

print("=" .. string.rep("=", 50))
print("บทที่ 72: GraphQL Server ด้วย Lua")
print("=" .. string.rep("=", 50))
for i, topic in ipairs(topics) do
    print(string.format("  %2d. %s", i, topic))
end
```
