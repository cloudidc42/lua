# บทที่ 71: gRPC กับ Lua

## บทนำ

gRPC (Google Remote Procedure Call) เป็น framework สำหรับการสื่อสารระหว่าง services ที่พัฒนาโดย Google ใช้ Protocol Buffers เป็น interface definition language และ HTTP/2 เป็น transport layer ทำให้มีประสิทธิภาพสูงกว่า REST API ในหลาย ๆ กรณี

## gRPC vs REST

### ความแตกต่างหลัก

```lua
-- ตัวอย่างเปรียบเทียบแนวคิด gRPC vs REST

-- REST API: ใช้ HTTP methods และ JSON
-- GET /users/123
-- POST /users { "name": "Alice", "age": 30 }
-- PUT /users/123 { "name": "Alice Updated" }
-- DELETE /users/123

-- gRPC: ใช้ method calls และ Protocol Buffers
-- userService.GetUser({ id: 123 })
-- userService.CreateUser({ name: "Alice", age: 30 })
-- userService.UpdateUser({ id: 123, name: "Alice Updated" })
-- userService.DeleteUser({ id: 123 })

-- ข้อดีของ gRPC:
-- 1. ประสิทธิภาพสูงกว่า (binary protocol vs JSON text)
-- 2. Strongly typed interfaces
-- 3. รองรับ streaming แบบ bidirectional
-- 4. Code generation อัตโนมัติ
-- 5. HTTP/2 multiplexing

local comparison = {
    rest = {
        protocol = "HTTP/1.1 หรือ HTTP/2",
        format = "JSON (text-based)",
        typing = "ไม่มี strict typing",
        streaming = "ไม่รองรับ native streaming",
        performance = "ปานกลาง"
    },
    grpc = {
        protocol = "HTTP/2",
        format = "Protocol Buffers (binary)",
        typing = "Strongly typed",
        streaming = "รองรับ 4 รูปแบบ",
        performance = "สูงมาก"
    }
}

for service, props in pairs(comparison) do
    print("=== " .. service:upper() .. " ===")
    for key, value in pairs(props) do
        print(string.format("  %-15s: %s", key, value))
    end
end
```

### เปรียบเทียบ Performance

```lua
-- จำลองการเปรียบเทียบ payload size
local json = require("cjson")  -- หรือ dkjson

-- ข้อมูล user แบบ JSON
local user_json = '{"id":12345,"name":"สมชาย ใจดี","email":"somchai@example.com","age":30,"active":true}'

-- Protocol Buffers จะ encode ข้อมูลเดียวกันได้ประมาณ 30-50% เล็กกว่า
-- (ตัวเลขจริงขึ้นกับโครงสร้างข้อมูล)

local function measure_size(data)
    return #data
end

print("JSON size: " .. measure_size(user_json) .. " bytes")
print("Protobuf estimated: ~" .. math.floor(measure_size(user_json) * 0.6) .. " bytes")
print("Compression ratio: ~40% smaller with Protobuf")
```

## Protocol Buffers

### โครงสร้าง .proto File

```lua
-- ตัวอย่าง .proto file syntax (แสดงเป็น comment)
--[[
syntax = "proto3";

package userservice;

// Message definitions
message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    bool active = 5;
    repeated string tags = 6;
    UserStatus status = 7;
}

enum UserStatus {
    UNKNOWN = 0;
    ACTIVE = 1;
    INACTIVE = 2;
    BANNED = 3;
}

message GetUserRequest {
    int32 id = 1;
}

message GetUserResponse {
    User user = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
}

message ListUsersResponse {
    repeated User users = 1;
    int32 total = 2;
}

// Service definition
service UserService {
    // Unary RPC
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
    
    // Server streaming
    rpc ListUsers(ListUsersRequest) returns (stream GetUserResponse);
    
    // Client streaming
    rpc CreateUsers(stream User) returns (ListUsersResponse);
    
    // Bidirectional streaming
    rpc SyncUsers(stream User) returns (stream GetUserResponse);
}
]]

-- จำลองโครงสร้าง proto definition ใน Lua
local proto_schema = {
    syntax = "proto3",
    package = "userservice",
    messages = {
        User = {
            fields = {
                {number = 1, name = "id", type = "int32"},
                {number = 2, name = "name", type = "string"},
                {number = 3, name = "email", type = "string"},
                {number = 4, name = "age", type = "int32"},
                {number = 5, name = "active", type = "bool"},
            }
        }
    }
}

-- แสดงโครงสร้าง schema
print("Package: " .. proto_schema.package)
print("Message: User")
for _, field in ipairs(proto_schema.messages.User.fields) do
    print(string.format("  Field %d: %s (%s)", field.number, field.name, field.type))
end
```

### Protocol Buffers Encoding

```lua
-- จำลองการ encode/decode Protocol Buffers
-- ในการใช้งานจริงจะใช้ library เช่น lua-protobuf

local protobuf = {}

-- Wire types
protobuf.WIRE_TYPES = {
    VARINT = 0,
    FIXED64 = 1,
    LENGTH_DELIMITED = 2,
    START_GROUP = 3,
    END_GROUP = 4,
    FIXED32 = 5
}

-- Encode varint
function protobuf.encode_varint(value)
    local bytes = {}
    while value > 127 do
        table.insert(bytes, (value & 0x7F) | 0x80)
        value = value >> 7
    end
    table.insert(bytes, value & 0x7F)
    return bytes
end

-- Encode field tag
function protobuf.encode_tag(field_number, wire_type)
    return protobuf.encode_varint((field_number << 3) | wire_type)
end

-- ทดสอบ encoding
local tag = protobuf.encode_tag(1, protobuf.WIRE_TYPES.VARINT)
print("Field 1 tag bytes:")
for i, b in ipairs(tag) do
    io.write(string.format("0x%02X ", b))
end
print()

-- Encode string field
function protobuf.encode_string(field_number, value)
    local result = {}
    -- Tag
    local tag_bytes = protobuf.encode_tag(field_number, protobuf.WIRE_TYPES.LENGTH_DELIMITED)
    for _, b in ipairs(tag_bytes) do
        table.insert(result, b)
    end
    -- Length
    local len_bytes = protobuf.encode_varint(#value)
    for _, b in ipairs(len_bytes) do
        table.insert(result, b)
    end
    -- Value
    for i = 1, #value do
        table.insert(result, string.byte(value, i))
    end
    return result
end

local encoded = protobuf.encode_string(2, "Alice")
print("Encoded 'name: Alice':")
for _, b in ipairs(encoded) do
    io.write(string.format("0x%02X ", b))
end
print()
print("Total bytes: " .. #encoded)
```

## กรอบงาน grpc-lua

### การติดตั้งและตั้งค่า

```lua
-- การติดตั้ง grpc-lua
-- luarocks install grpc-lua
-- หรือ compile จาก source

-- ในตัวอย่างนี้จะจำลองโครงสร้าง grpc-lua API
-- เนื่องจากต้องการ C library ในการรัน

-- โครงสร้าง grpc-lua module
local grpc_mock = {
    -- Channel สำหรับเชื่อมต่อกับ server
    channel = {},
    -- Stub สำหรับเรียก service
    stub = {},
    -- Status codes
    status_code = {
        OK = 0,
        CANCELLED = 1,
        UNKNOWN = 2,
        INVALID_ARGUMENT = 3,
        NOT_FOUND = 5,
        ALREADY_EXISTS = 6,
        PERMISSION_DENIED = 7,
        RESOURCE_EXHAUSTED = 8,
        FAILED_PRECONDITION = 9,
        ABORTED = 10,
        DEADLINE_EXCEEDED = 4,
        UNAUTHENTICATED = 16,
        UNAVAILABLE = 14,
        INTERNAL = 13,
    }
}

print("gRPC Status Codes:")
for name, code in pairs(grpc_mock.status_code) do
    print(string.format("  %s = %d", name, code))
end
```

### การสร้าง gRPC Channel

```lua
-- การสร้าง channel เพื่อเชื่อมต่อกับ gRPC server
-- ใน production จะใช้ library จริง

local function create_channel_config(host, port, options)
    options = options or {}
    return {
        target = string.format("%s:%d", host, port),
        credentials = options.credentials or "insecure",
        options = {
            -- Keep alive settings
            ["grpc.keepalive_time_ms"] = options.keepalive_time or 30000,
            ["grpc.keepalive_timeout_ms"] = options.keepalive_timeout or 5000,
            ["grpc.keepalive_permit_without_calls"] = options.keepalive_without_calls and 1 or 0,
            -- Max message sizes
            ["grpc.max_receive_message_length"] = options.max_recv_msg or 4 * 1024 * 1024,
            ["grpc.max_send_message_length"] = options.max_send_msg or 4 * 1024 * 1024,
            -- Load balancing
            ["grpc.lb_policy_name"] = options.lb_policy or "round_robin",
        }
    }
end

-- ตัวอย่างการสร้าง channel configurations
local configs = {
    -- Insecure channel (สำหรับ development)
    dev = create_channel_config("localhost", 50051),
    
    -- Channel with TLS
    prod = create_channel_config("api.example.com", 443, {
        credentials = "tls",
        keepalive_time = 60000,
    }),
    
    -- Channel with custom options
    custom = create_channel_config("service.internal", 50051, {
        max_recv_msg = 8 * 1024 * 1024,
        lb_policy = "round_robin",
    })
}

for name, config in pairs(configs) do
    print(string.format("Channel '%s': %s (%s)", name, config.target, config.credentials))
end
```

## Unary RPC ใน Lua

### การสร้าง gRPC Server (จำลอง)

```lua
-- จำลอง gRPC server implementation

-- Service handler
local UserServiceImpl = {}

-- Database จำลอง
local users_db = {
    [1] = {id = 1, name = "สมชาย ใจดี", email = "somchai@example.com", age = 30, active = true},
    [2] = {id = 2, name = "สุมาลี รักดี", email = "sumalee@example.com", age = 25, active = true},
    [3] = {id = 3, name = "วิชัย มั่นคง", email = "wichai@example.com", age = 35, active = false},
}

-- Unary RPC: GetUser
function UserServiceImpl.GetUser(request, context)
    local user_id = request.id
    
    -- Validate input
    if not user_id or user_id <= 0 then
        context:set_status({
            code = 3,  -- INVALID_ARGUMENT
            message = "User ID ต้องมีค่ามากกว่า 0"
        })
        return nil
    end
    
    local user = users_db[user_id]
    if not user then
        context:set_status({
            code = 5,  -- NOT_FOUND
            message = string.format("ไม่พบ user ที่มี ID %d", user_id)
        })
        return nil
    end
    
    return {user = user}
end

-- Unary RPC: CreateUser
function UserServiceImpl.CreateUser(request, context)
    -- Validate required fields
    if not request.name or request.name == "" then
        context:set_status({
            code = 3,
            message = "Name is required"
        })
        return nil
    end
    
    if not request.email or not request.email:match("@") then
        context:set_status({
            code = 3,
            message = "Valid email is required"
        })
        return nil
    end
    
    -- สร้าง user ใหม่
    local new_id = #users_db + 1
    local new_user = {
        id = new_id,
        name = request.name,
        email = request.email,
        age = request.age or 0,
        active = true
    }
    
    users_db[new_id] = new_user
    print(string.format("สร้าง user ใหม่: ID=%d, Name=%s", new_id, new_user.name))
    
    return {user = new_user}
end

-- ทดสอบ service handlers
local mock_context = {
    status = nil,
    set_status = function(self, status)
        self.status = status
        print(string.format("Status set: code=%d, message=%s", status.code, status.message))
    end
}

print("=== ทดสอบ GetUser ===")
local result = UserServiceImpl.GetUser({id = 1}, mock_context)
if result then
    print("Found user: " .. result.user.name)
end

print("\n=== ทดสอบ GetUser ที่ไม่มีอยู่ ===")
mock_context.status = nil
result = UserServiceImpl.GetUser({id = 999}, mock_context)
if not result then
    print("User not found (expected)")
end

print("\n=== ทดสอบ CreateUser ===")
mock_context.status = nil
result = UserServiceImpl.CreateUser({
    name = "ประทีป วิมล",
    email = "prateep@example.com",
    age = 28
}, mock_context)
if result then
    print("Created user: " .. result.user.name .. " (ID: " .. result.user.id .. ")")
end
```

### การสร้าง gRPC Client (จำลอง)

```lua
-- จำลอง gRPC client implementation

local GrpcClient = {}
GrpcClient.__index = GrpcClient

function GrpcClient.new(host, port, options)
    local self = setmetatable({}, GrpcClient)
    self.host = host
    self.port = port
    self.options = options or {}
    self.connected = false
    self.timeout = options and options.timeout or 30
    return self
end

function GrpcClient:connect()
    -- จำลองการเชื่อมต่อ
    print(string.format("Connecting to %s:%d...", self.host, self.port))
    self.connected = true
    print("Connected!")
    return true
end

function GrpcClient:call_unary(service, method, request, metadata)
    if not self.connected then
        return nil, {code = 14, message = "Not connected"}
    end
    
    metadata = metadata or {}
    
    -- จำลองการส่ง request
    print(string.format("Calling %s.%s with request: %s", 
        service, method, 
        require("cjson") and require("cjson").encode(request) or tostring(request)))
    
    -- จำลอง response
    if service == "UserService" and method == "GetUser" then
        local response = UserServiceImpl.GetUser(request, {
            set_status = function(self, s) end,
            get_metadata = function() return metadata end
        })
        return response, nil
    end
    
    return nil, {code = 12, message = "Method not implemented"}
end

function GrpcClient:close()
    self.connected = false
    print("Connection closed")
end

-- ทดสอบ client
local client = GrpcClient.new("localhost", 50051)
client:connect()

-- ส่ง unary request
local response, err = client:call_unary("UserService", "GetUser", {id = 1})
if err then
    print("Error: " .. err.message)
elseif response then
    print("Response received: user=" .. (response.user and response.user.name or "nil"))
end

client:close()
```

## Server Streaming RPC

```lua
-- Server Streaming: client ส่ง 1 request, server ส่ง หลาย responses

-- Server-side streaming handler
function UserServiceImpl.ListUsers(request, stream_context)
    local page = request.page or 1
    local page_size = request.page_size or 10
    
    print(string.format("ListUsers: page=%d, page_size=%d", page, page_size))
    
    -- ส่ง users ทีละคน (streaming)
    local count = 0
    for id, user in pairs(users_db) do
        -- เช็คว่า stream ยังเปิดอยู่
        if not stream_context:is_cancelled() then
            -- จำลองการส่ง response ผ่าน stream
            stream_context:write({user = user})
            count = count + 1
            
            -- จำลอง delay
            -- os.execute("sleep 0.1")
        else
            print("Stream cancelled by client")
            break
        end
    end
    
    print(string.format("Sent %d users", count))
end

-- Mock stream context
local function create_stream_context()
    local ctx = {
        cancelled = false,
        written = {},
        status = nil
    }
    
    function ctx:is_cancelled()
        return self.cancelled
    end
    
    function ctx:write(response)
        table.insert(self.written, response)
        if response.user then
            print("  Streaming user: " .. response.user.name)
        end
    end
    
    function ctx:set_status(s)
        self.status = s
    end
    
    return ctx
end

print("=== ทดสอบ Server Streaming ===")
local stream_ctx = create_stream_context()
UserServiceImpl.ListUsers({page = 1, page_size = 10}, stream_ctx)
print(string.format("รับทั้งหมด %d responses จาก stream", #stream_ctx.written))
```

## Client Streaming RPC

```lua
-- Client Streaming: client ส่งหลาย requests, server ส่ง 1 response

-- Client-side streaming handler
function UserServiceImpl.CreateUsers(stream_reader, context)
    local created_users = {}
    local error_count = 0
    
    -- อ่าน requests จาก stream ทีละรายการ
    local request = stream_reader:read()
    while request do
        -- Validate และสร้าง user
        if request.name and request.name ~= "" then
            local new_id = #users_db + 1
            local new_user = {
                id = new_id,
                name = request.name,
                email = request.email or "",
                age = request.age or 0,
                active = true
            }
            users_db[new_id] = new_user
            table.insert(created_users, new_user)
            print("  Created: " .. new_user.name)
        else
            error_count = error_count + 1
            print("  Skipped invalid user")
        end
        
        request = stream_reader:read()
    end
    
    -- ส่ง response เดียว
    return {
        users = created_users,
        total = #created_users,
        errors = error_count
    }
end

-- Mock stream reader
local function create_stream_reader(requests)
    local idx = 0
    return {
        read = function(self)
            idx = idx + 1
            return requests[idx]
        end
    }
end

print("=== ทดสอบ Client Streaming ===")
local stream_reader = create_stream_reader({
    {name = "ผู้ใช้ใหม่ 1", email = "new1@example.com", age = 22},
    {name = "ผู้ใช้ใหม่ 2", email = "new2@example.com", age = 23},
    {name = "", email = "invalid@example.com"},  -- invalid
    {name = "ผู้ใช้ใหม่ 3", email = "new3@example.com", age = 24},
})

local result = UserServiceImpl.CreateUsers(stream_reader, mock_context)
print(string.format("สร้างสำเร็จ: %d users, Errors: %d", result.total, result.errors))
```

## Bidirectional Streaming RPC

```lua
-- Bidirectional Streaming: ทั้งสองฝั่งส่งได้พร้อมกัน

-- Bidirectional streaming handler
function UserServiceImpl.SyncUsers(stream, context)
    local synced = 0
    local deleted = 0
    
    -- อ่านและตอบพร้อมกัน
    local request = stream:read()
    while request do
        if request.operation == "upsert" then
            -- Create or update user
            if request.user then
                local user = request.user
                users_db[user.id] = user
                synced = synced + 1
                
                -- ส่ง acknowledgment กลับ
                stream:write({
                    operation = "acked",
                    user_id = user.id,
                    status = "synced"
                })
            end
        elseif request.operation == "delete" then
            -- Delete user
            if users_db[request.user_id] then
                users_db[request.user_id] = nil
                deleted = deleted + 1
                
                stream:write({
                    operation = "acked",
                    user_id = request.user_id,
                    status = "deleted"
                })
            end
        end
        
        request = stream:read()
    end
    
    print(string.format("Sync complete: %d synced, %d deleted", synced, deleted))
end

-- Mock bidirectional stream
local function create_bidi_stream(requests)
    local idx = 0
    local written = {}
    
    return {
        read = function(self)
            idx = idx + 1
            return requests[idx]
        end,
        write = function(self, response)
            table.insert(written, response)
            print(string.format("  Server -> Client: op=%s, user_id=%s, status=%s",
                response.operation or "?",
                tostring(response.user_id or "?"),
                response.status or "?"))
        end,
        get_written = function(self)
            return written
        end
    }
end

print("=== ทดสอบ Bidirectional Streaming ===")
local bidi_stream = create_bidi_stream({
    {operation = "upsert", user = {id = 10, name = "Sync User 1", email = "sync1@example.com"}},
    {operation = "upsert", user = {id = 11, name = "Sync User 2", email = "sync2@example.com"}},
    {operation = "delete", user_id = 1},
    {operation = "upsert", user = {id = 12, name = "Sync User 3", email = "sync3@example.com"}},
})

UserServiceImpl.SyncUsers(bidi_stream, mock_context)
print("Written responses: " .. #bidi_stream:get_written())
```

## Error Handling ใน gRPC

### gRPC Status Codes

```lua
-- gRPC Status Code definitions
local GrpcStatus = {}

GrpcStatus.codes = {
    OK = {code = 0, description = "สำเร็จ"},
    CANCELLED = {code = 1, description = "ถูกยกเลิกโดย caller"},
    UNKNOWN = {code = 2, description = "ข้อผิดพลาดที่ไม่รู้จัก"},
    INVALID_ARGUMENT = {code = 3, description = "ข้อมูล input ไม่ถูกต้อง"},
    DEADLINE_EXCEEDED = {code = 4, description = "หมดเวลา"},
    NOT_FOUND = {code = 5, description = "ไม่พบ resource"},
    ALREADY_EXISTS = {code = 6, description = "มีอยู่แล้ว"},
    PERMISSION_DENIED = {code = 7, description = "ไม่มีสิทธิ์"},
    RESOURCE_EXHAUSTED = {code = 8, description = "ใช้ resource หมดแล้ว"},
    FAILED_PRECONDITION = {code = 9, description = "เงื่อนไขไม่ตรงตามที่กำหนด"},
    ABORTED = {code = 10, description = "การดำเนินการถูกยกเลิก"},
    OUT_OF_RANGE = {code = 11, description = "ค่าอยู่นอกช่วงที่กำหนด"},
    UNIMPLEMENTED = {code = 12, description = "ยังไม่ได้ implement"},
    INTERNAL = {code = 13, description = "ข้อผิดพลาดภายใน server"},
    UNAVAILABLE = {code = 14, description = "Service ไม่พร้อมให้บริการ"},
    DATA_LOSS = {code = 15, description = "ข้อมูลสูญหาย"},
    UNAUTHENTICATED = {code = 16, description = "ยังไม่ได้ authenticate"},
}

function GrpcStatus.new(code_name, message, details)
    local status_def = GrpcStatus.codes[code_name]
    if not status_def then
        error("Unknown status code: " .. tostring(code_name))
    end
    
    return {
        code = status_def.code,
        code_name = code_name,
        description = status_def.description,
        message = message or "",
        details = details or {}
    }
end

function GrpcStatus.is_ok(status)
    return status.code == 0
end

function GrpcStatus.to_string(status)
    return string.format("[%s(%d)] %s", 
        status.code_name, status.code, status.message)
end

-- ทดสอบ status codes
local statuses = {
    GrpcStatus.new("OK", "Request processed successfully"),
    GrpcStatus.new("NOT_FOUND", "User with ID 999 not found"),
    GrpcStatus.new("INVALID_ARGUMENT", "Email format is invalid"),
    GrpcStatus.new("UNAUTHENTICATED", "JWT token expired"),
    GrpcStatus.new("PERMISSION_DENIED", "Insufficient permissions to delete user"),
}

for _, status in ipairs(statuses) do
    local ok_marker = GrpcStatus.is_ok(status) and "✓" or "✗"
    print(string.format("%s %s", ok_marker, GrpcStatus.to_string(status)))
end
```

### Error Details และ Rich Error Model

```lua
-- gRPC Rich Error Model
local RichError = {}

-- Error detail types
function RichError.bad_request(field_violations)
    return {
        type = "BadRequest",
        field_violations = field_violations
    }
end

function RichError.retry_info(retry_delay_seconds)
    return {
        type = "RetryInfo",
        retry_delay = retry_delay_seconds
    }
end

function RichError.error_info(reason, domain, metadata)
    return {
        type = "ErrorInfo",
        reason = reason,
        domain = domain,
        metadata = metadata or {}
    }
end

function RichError.request_info(request_id, serving_data)
    return {
        type = "RequestInfo",
        request_id = request_id,
        serving_data = serving_data
    }
end

-- สร้าง error response ที่มี details
local function create_validation_error(violations)
    return {
        status = GrpcStatus.new("INVALID_ARGUMENT", "Validation failed"),
        details = {
            RichError.bad_request(violations),
            RichError.request_info("req-" .. tostring(os.time()), "user-service-v1")
        }
    }
end

-- ตัวอย่างการใช้งาน
local error_response = create_validation_error({
    {field = "email", description = "รูปแบบ email ไม่ถูกต้อง"},
    {field = "age", description = "อายุต้องอยู่ระหว่าง 1-150"},
    {field = "name", description = "ชื่อต้องมีความยาวอย่างน้อย 2 ตัวอักษร"},
})

print("Error: " .. error_response.status.message)
for _, detail in ipairs(error_response.details) do
    print("Detail type: " .. detail.type)
    if detail.field_violations then
        for _, v in ipairs(detail.field_violations) do
            print(string.format("  Field '%s': %s", v.field, v.description))
        end
    end
end
```

## Metadata ใน gRPC

```lua
-- Metadata คือ key-value pairs ที่ส่งไปกับ RPC calls
-- คล้ายกับ HTTP headers

local GrpcMetadata = {}
GrpcMetadata.__index = GrpcMetadata

function GrpcMetadata.new()
    local self = setmetatable({}, GrpcMetadata)
    self._data = {}
    return self
end

function GrpcMetadata:add(key, value)
    -- Keys ต้องเป็น lowercase
    key = key:lower()
    if not self._data[key] then
        self._data[key] = {}
    end
    table.insert(self._data[key], value)
    return self
end

function GrpcMetadata:get(key)
    key = key:lower()
    local values = self._data[key]
    if values and #values > 0 then
        return values[1]
    end
    return nil
end

function GrpcMetadata:get_all(key)
    key = key:lower()
    return self._data[key] or {}
end

function GrpcMetadata:to_table()
    return self._data
end

-- Binary metadata (keys ที่ลงท้ายด้วย -bin)
function GrpcMetadata:add_binary(key, bytes)
    if not key:match("%-bin$") then
        key = key .. "-bin"
    end
    self:add(key, bytes)
end

-- ตัวอย่างการใช้ metadata
local request_metadata = GrpcMetadata.new()
    :add("authorization", "Bearer eyJhbGciOiJSUzI1NiJ9...")
    :add("x-request-id", "req-uuid-12345")
    :add("x-user-agent", "lua-grpc-client/1.0")
    :add("accept-language", "th-TH")
    :add("x-trace-id", "trace-abc123")

print("Request Metadata:")
for key, values in pairs(request_metadata:to_table()) do
    for _, val in ipairs(values) do
        -- Truncate long values for display
        local display_val = #val > 30 and val:sub(1, 30) .. "..." or val
        print(string.format("  %s: %s", key, display_val))
    end
end

-- Server ส่ง trailing metadata กลับมา
local response_metadata = GrpcMetadata.new()
    :add("x-request-id", "req-uuid-12345")  -- echo back
    :add("x-response-time", "42ms")
    :add("x-server-version", "user-service-v2.1.0")
    :add("x-cache", "MISS")

print("\nResponse Metadata:")
for key, values in pairs(response_metadata:to_table()) do
    print(string.format("  %s: %s", key, values[1]))
end
```

## Deadlines และ Timeouts

```lua
-- Deadlines ใน gRPC กำหนดเวลาสูงสุดที่ RPC call จะรอ

local GrpcDeadline = {}

function GrpcDeadline.from_now(seconds)
    return {
        deadline = os.time() + seconds,
        timeout_seconds = seconds
    }
end

function GrpcDeadline.is_expired(deadline_obj)
    return os.time() > deadline_obj.deadline
end

function GrpcDeadline.remaining_seconds(deadline_obj)
    return math.max(0, deadline_obj.deadline - os.time())
end

-- Wrapper ที่ใช้ deadline
local function call_with_deadline(fn, deadline_obj, ...)
    if GrpcDeadline.is_expired(deadline_obj) then
        return nil, GrpcStatus.new("DEADLINE_EXCEEDED", 
            "Deadline exceeded before call could start")
    end
    
    -- จำลองการตรวจสอบ deadline ระหว่าง execution
    local remaining = GrpcDeadline.remaining_seconds(deadline_obj)
    print(string.format("Calling with %d seconds remaining", remaining))
    
    -- Execute function
    local result = fn(...)
    
    -- ตรวจสอบ deadline อีกครั้งหลัง execution
    if GrpcDeadline.is_expired(deadline_obj) then
        return nil, GrpcStatus.new("DEADLINE_EXCEEDED",
            "Deadline exceeded during execution")
    end
    
    return result, nil
end

-- ตัวอย่าง timeout handling
local function slow_operation(delay)
    -- จำลอง operation ที่ช้า
    local start = os.clock()
    while os.clock() - start < delay do
        -- busy wait (ใน production อย่าทำแบบนี้)
    end
    return {success = true, data = "operation result"}
end

print("=== ทดสอบ Deadline ===")

-- Deadline 5 วินาที
local deadline = GrpcDeadline.from_now(5)
print(string.format("Deadline set: %d seconds from now", deadline.timeout_seconds))

local result, err = call_with_deadline(slow_operation, deadline, 0.1)  -- 0.1 second operation
if err then
    print("Error: " .. err.message)
else
    print("Success: " .. (result and result.data or "no data"))
end

print(string.format("Remaining after call: %.1f seconds", 
    GrpcDeadline.remaining_seconds(deadline)))
```

## Authentication กับ gRPC

### JWT Authentication

```lua
-- JWT-based authentication สำหรับ gRPC

local function base64_encode(data)
    local b64 = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
    local result = {}
    local padding = 0
    
    for i = 1, #data, 3 do
        local b1 = string.byte(data, i) or 0
        local b2 = string.byte(data, i+1) or 0
        local b3 = string.byte(data, i+2) or 0
        
        if i+1 > #data then padding = 2
        elseif i+2 > #data then padding = 1
        end
        
        local idx1 = (b1 >> 2) + 1
        local idx2 = ((b1 & 3) << 4 | (b2 >> 4)) + 1
        local idx3 = ((b2 & 15) << 2 | (b3 >> 6)) + 1
        local idx4 = (b3 & 63) + 1
        
        table.insert(result, b64:sub(idx1, idx1))
        table.insert(result, b64:sub(idx2, idx2))
        if padding < 2 then table.insert(result, b64:sub(idx3, idx3)) else table.insert(result, "=") end
        if padding < 1 then table.insert(result, b64:sub(idx4, idx4)) else table.insert(result, "=") end
    end
    
    return table.concat(result)
end

-- สร้าง JWT token (จำลอง)
local function create_jwt(payload, secret)
    local header = base64_encode('{"alg":"HS256","typ":"JWT"}')
    local payload_encoded = base64_encode(table.concat({
        '{"sub":"',
        tostring(payload.sub or ""),
        '","name":"',
        tostring(payload.name or ""),
        '","exp":',
        tostring(payload.exp or (os.time() + 3600)),
        '}'
    }))
    
    -- ใน production จะใช้ HMAC-SHA256 จริง
    local signature = base64_encode("signature-" .. secret)
    
    return string.format("%s.%s.%s", header, payload_encoded, signature)
end

-- gRPC Interceptor สำหรับ authentication
local function auth_interceptor(method, request, metadata, next_handler)
    -- ตรวจสอบ token
    local token = metadata:get("authorization")
    
    if not token then
        return nil, GrpcStatus.new("UNAUTHENTICATED", 
            "Missing authorization header")
    end
    
    -- ตรวจสอบรูปแบบ Bearer token
    if not token:match("^Bearer ") then
        return nil, GrpcStatus.new("UNAUTHENTICATED",
            "Authorization header must start with 'Bearer '")
    end
    
    local jwt_token = token:sub(8)  -- Remove "Bearer "
    
    -- ใน production จะ verify JWT signature ด้วย
    print(string.format("Authenticating request for method: %s", method))
    print(string.format("Token: %s...", jwt_token:sub(1, 20)))
    
    -- เรียก handler ต่อ
    return next_handler(method, request, metadata)
end

-- ทดสอบ authentication
local token = create_jwt({
    sub = "user-123",
    name = "สมชาย ใจดี",
    exp = os.time() + 3600
}, "secret-key")

print("Generated JWT: " .. token:sub(1, 50) .. "...")

local auth_metadata = GrpcMetadata.new()
    :add("authorization", "Bearer " .. token)

-- จำลองการเรียก interceptor
local result, err = auth_interceptor(
    "GetUser",
    {id = 1},
    auth_metadata,
    function(method, req, meta)
        return {user = users_db[req.id]}, nil
    end
)

if err then
    print("Auth error: " .. err.message)
else
    print("Auth success, response: " .. (result and result.user and result.user.name or "nil"))
end
```

### TLS/mTLS Authentication

```lua
-- TLS Configuration สำหรับ gRPC

local TLSConfig = {}

function TLSConfig.client_tls(options)
    return {
        type = "tls",
        ca_cert = options.ca_cert,
        verify_server = options.verify_server ~= false,
        server_name_override = options.server_name_override,
    }
end

function TLSConfig.mutual_tls(options)
    return {
        type = "mtls",
        ca_cert = options.ca_cert,
        client_cert = options.client_cert,
        client_key = options.client_key,
        verify_client = true,
        verify_server = true,
    }
end

-- ตัวอย่าง TLS configurations
local tls_configs = {
    -- Client TLS (ตรวจสอบ server cert เท่านั้น)
    client_only = TLSConfig.client_tls({
        ca_cert = "/etc/ssl/certs/ca.crt",
        verify_server = true,
    }),
    
    -- Mutual TLS (ทั้งสองฝั่งตรวจสอบกัน)
    mutual = TLSConfig.mutual_tls({
        ca_cert = "/etc/ssl/certs/ca.crt",
        client_cert = "/etc/ssl/client/client.crt",
        client_key = "/etc/ssl/client/client.key",
    }),
}

for name, config in pairs(tls_configs) do
    print(string.format("TLS Config '%s': type=%s", name, config.type))
    if config.client_cert then
        print(string.format("  Client cert: %s", config.client_cert))
    end
end
```

## gRPC Gateway

### REST to gRPC Translation

```lua
-- gRPC Gateway แปลง REST requests เป็น gRPC calls

-- กำหนด HTTP rules สำหรับ .proto
--[[
service UserService {
    rpc GetUser(GetUserRequest) returns (GetUserResponse) {
        option (google.api.http) = {
            get: "/v1/users/{id}"
        };
    }
    
    rpc CreateUser(User) returns (GetUserResponse) {
        option (google.api.http) = {
            post: "/v1/users"
            body: "*"
        };
    }
    
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse) {
        option (google.api.http) = {
            get: "/v1/users"
        };
    }
}
]]

-- Gateway routing table
local gateway_routes = {
    {method = "GET", pattern = "/v1/users/{id}", 
     service = "UserService", rpc = "GetUser",
     path_params = {"id"}},
    
    {method = "POST", pattern = "/v1/users",
     service = "UserService", rpc = "CreateUser",
     body_mapping = "*"},
    
    {method = "GET", pattern = "/v1/users",
     service = "UserService", rpc = "ListUsers",
     query_params = {"page", "page_size"}},
     
    {method = "DELETE", pattern = "/v1/users/{id}",
     service = "UserService", rpc = "DeleteUser",
     path_params = {"id"}},
}

-- Pattern matching
local function match_route(http_method, http_path)
    for _, route in ipairs(gateway_routes) do
        if route.method == http_method then
            -- แปลง pattern เป็น regex
            local pattern = route.pattern:gsub("{(%w+)}", "([^/]+)")
            local captures = {http_path:match("^" .. pattern .. "$")}
            
            if #captures > 0 then
                -- สร้าง path params
                local params = {}
                if route.path_params then
                    for i, param_name in ipairs(route.path_params) do
                        params[param_name] = captures[i]
                    end
                end
                return route, params
            end
        end
    end
    return nil, {}
end

-- Test routing
local test_requests = {
    {method = "GET", path = "/v1/users/123"},
    {method = "POST", path = "/v1/users"},
    {method = "GET", path = "/v1/users"},
    {method = "DELETE", path = "/v1/users/456"},
    {method = "GET", path = "/v1/orders/789"},  -- no match
}

print("=== gRPC Gateway Routing ===")
for _, req in ipairs(test_requests) do
    local route, params = match_route(req.method, req.path)
    if route then
        print(string.format("%s %s -> %s.%s", req.method, req.path, route.service, route.rpc))
        for k, v in pairs(params) do
            print(string.format("  Param: %s = %s", k, v))
        end
    else
        print(string.format("%s %s -> 404 Not Found", req.method, req.path))
    end
end
```

## gRPC Reflection

```lua
-- gRPC Server Reflection ช่วยให้ client รู้ว่า server มี services อะไรบ้าง
-- โดยไม่ต้องรู้ .proto file ล่วงหน้า

local ServerReflection = {}

-- จำลอง service descriptors
local service_registry = {
    ["userservice.UserService"] = {
        name = "UserService",
        full_name = "userservice.UserService",
        methods = {
            {
                name = "GetUser",
                input_type = "userservice.GetUserRequest",
                output_type = "userservice.GetUserResponse",
                client_streaming = false,
                server_streaming = false,
            },
            {
                name = "ListUsers",
                input_type = "userservice.ListUsersRequest",
                output_type = "userservice.GetUserResponse",
                client_streaming = false,
                server_streaming = true,  -- server streaming
            },
            {
                name = "CreateUsers",
                input_type = "userservice.User",
                output_type = "userservice.ListUsersResponse",
                client_streaming = true,  -- client streaming
                server_streaming = false,
            },
            {
                name = "SyncUsers",
                input_type = "userservice.User",
                output_type = "userservice.GetUserResponse",
                client_streaming = true,  -- bidirectional
                server_streaming = true,
            },
        }
    }
}

function ServerReflection.list_services()
    local services = {}
    for full_name, _ in pairs(service_registry) do
        table.insert(services, full_name)
    end
    return services
end

function ServerReflection.describe_service(full_name)
    return service_registry[full_name]
end

function ServerReflection.get_rpc_type(service_name, method_name)
    local service = service_registry[service_name]
    if not service then return nil end
    
    for _, method in ipairs(service.methods) do
        if method.name == method_name then
            if method.client_streaming and method.server_streaming then
                return "BIDIRECTIONAL_STREAMING"
            elseif method.client_streaming then
                return "CLIENT_STREAMING"
            elseif method.server_streaming then
                return "SERVER_STREAMING"
            else
                return "UNARY"
            end
        end
    end
    return nil
end

-- ทดสอบ reflection
print("=== gRPC Server Reflection ===")
print("Available services:")
for _, svc in ipairs(ServerReflection.list_services()) do
    print("  " .. svc)
    
    local desc = ServerReflection.describe_service(svc)
    if desc then
        for _, method in ipairs(desc.methods) do
            local rpc_type = ServerReflection.get_rpc_type(svc, method.name)
            print(string.format("    %s (%s)", method.name, rpc_type))
        end
    end
end
```

## Health Checking

```lua
-- gRPC Health Checking Protocol

local HealthService = {}

-- Health statuses
HealthService.SERVING_STATUS = {
    UNKNOWN = 0,
    SERVING = 1,
    NOT_SERVING = 2,
    SERVICE_UNKNOWN = 3,
}

-- Service health state
local health_state = {}

function HealthService.set_serving_status(service_name, status)
    health_state[service_name] = status
    print(string.format("Health status for '%s': %s",
        service_name,
        status == HealthService.SERVING_STATUS.SERVING and "SERVING" or "NOT_SERVING"))
end

-- Check health RPC handler
function HealthService.Check(request)
    local service = request.service or ""
    
    if service == "" then
        -- ตรวจสอบ overall server health
        local any_not_serving = false
        for _, status in pairs(health_state) do
            if status ~= HealthService.SERVING_STATUS.SERVING then
                any_not_serving = true
                break
            end
        end
        
        return {
            status = any_not_serving and 
                HealthService.SERVING_STATUS.NOT_SERVING or
                HealthService.SERVING_STATUS.SERVING
        }
    end
    
    local status = health_state[service]
    if status == nil then
        return {status = HealthService.SERVING_STATUS.SERVICE_UNKNOWN}
    end
    
    return {status = status}
end

-- Watch health (streaming)
function HealthService.Watch(request, stream)
    local service = request.service or ""
    local last_status = nil
    
    -- ส่งสถานะปัจจุบัน
    local current = health_state[service]
    if current ~= last_status then
        stream:write({status = current or HealthService.SERVING_STATUS.SERVICE_UNKNOWN})
        last_status = current
    end
    
    -- ใน production จะ watch สำหรับ status changes
    -- และส่ง update เมื่อ status เปลี่ยน
end

-- ตั้งค่า health states
HealthService.set_serving_status("userservice.UserService", HealthService.SERVING_STATUS.SERVING)
HealthService.set_serving_status("orderservice.OrderService", HealthService.SERVING_STATUS.SERVING)
HealthService.set_serving_status("paymentservice.PaymentService", HealthService.SERVING_STATUS.NOT_SERVING)

-- ทดสอบ health check
print("\n=== Health Checks ===")
local check_result = HealthService.Check({service = ""})
print("Overall health: " .. (check_result.status == 1 and "SERVING" or "NOT_SERVING"))

for _, svc in ipairs({"userservice.UserService", "paymentservice.PaymentService", "unknownservice.Svc"}) do
    local result = HealthService.Check({service = svc})
    local status_name
    for name, code in pairs(HealthService.SERVING_STATUS) do
        if code == result.status then status_name = name end
    end
    print(string.format("  %s: %s", svc, status_name or "UNKNOWN"))
end
```

## Load Balancing สำหรับ gRPC

```lua
-- Load Balancing strategies สำหรับ gRPC

local LoadBalancer = {}
LoadBalancer.__index = LoadBalancer

-- Round Robin Load Balancer
function LoadBalancer.round_robin(endpoints)
    local self = setmetatable({}, LoadBalancer)
    self.endpoints = endpoints
    self.current_idx = 0
    self.policy = "round_robin"
    return self
end

-- Weighted Round Robin
function LoadBalancer.weighted_round_robin(endpoints)
    -- endpoints = {{host, port, weight}, ...}
    local self = setmetatable({}, LoadBalancer)
    self.policy = "weighted_round_robin"
    
    -- สร้าง weighted list
    self.weighted_endpoints = {}
    for _, ep in ipairs(endpoints) do
        for i = 1, (ep.weight or 1) do
            table.insert(self.weighted_endpoints, {
                host = ep.host,
                port = ep.port
            })
        end
    end
    
    self.current_idx = 0
    return self
end

-- Random Load Balancer
function LoadBalancer.random(endpoints)
    local self = setmetatable({}, LoadBalancer)
    self.endpoints = endpoints
    self.policy = "random"
    return self
end

function LoadBalancer:pick()
    if self.policy == "round_robin" then
        self.current_idx = (self.current_idx % #self.endpoints) + 1
        return self.endpoints[self.current_idx]
    elseif self.policy == "weighted_round_robin" then
        self.current_idx = (self.current_idx % #self.weighted_endpoints) + 1
        return self.weighted_endpoints[self.current_idx]
    elseif self.policy == "random" then
        return self.endpoints[math.random(1, #self.endpoints)]
    end
end

function LoadBalancer:get_stats()
    local ep_count = self.policy == "weighted_round_robin" and 
        #self.weighted_endpoints or #(self.endpoints or {})
    return {
        policy = self.policy,
        endpoint_count = ep_count,
        current_idx = self.current_idx
    }
end

-- ทดสอบ load balancing
math.randomseed(42)  -- สำหรับ reproducible output

local endpoints = {
    {host = "service-1.internal", port = 50051},
    {host = "service-2.internal", port = 50051},
    {host = "service-3.internal", port = 50051},
}

local weighted_endpoints = {
    {host = "service-1.internal", port = 50051, weight = 3},  -- รับ traffic 3x
    {host = "service-2.internal", port = 50051, weight = 2},  -- รับ traffic 2x
    {host = "service-3.internal", port = 50051, weight = 1},  -- รับ traffic 1x
}

print("=== Round Robin ===")
local rr_lb = LoadBalancer.round_robin(endpoints)
for i = 1, 6 do
    local ep = rr_lb:pick()
    print(string.format("Request %d -> %s:%d", i, ep.host, ep.port))
end

print("\n=== Weighted Round Robin ===")
local wrr_lb = LoadBalancer.weighted_round_robin(weighted_endpoints)
local hit_count = {}
for i = 1, 12 do
    local ep = wrr_lb:pick()
    hit_count[ep.host] = (hit_count[ep.host] or 0) + 1
end
print("Hit distribution after 12 requests:")
for host, count in pairs(hit_count) do
    print(string.format("  %s: %d hits (%.0f%%)", host, count, count/12*100))
end
```

## Interceptors (Middleware)

```lua
-- gRPC Interceptors สำหรับ cross-cutting concerns

-- Chain of interceptors
local InterceptorChain = {}

function InterceptorChain.new()
    return {interceptors = {}}
end

function InterceptorChain:add(interceptor)
    table.insert(self.interceptors, interceptor)
    return self
end

function InterceptorChain:build(handler)
    -- สร้าง chain จาก interceptors (reversed)
    local current_handler = handler
    for i = #self.interceptors, 1, -1 do
        local interceptor = self.interceptors[i]
        local next_handler = current_handler
        current_handler = function(ctx, req)
            return interceptor(ctx, req, next_handler)
        end
    end
    return current_handler
end

-- ตัวอย่าง interceptors

-- Logging interceptor
local function logging_interceptor(ctx, request, next)
    local start_time = os.clock()
    print(string.format("[LOG] %s -> %s", 
        ctx.method or "unknown", 
        require("cjson") and require("cjson").encode(request) or tostring(request)))
    
    local result, err = next(ctx, request)
    
    local elapsed = (os.clock() - start_time) * 1000
    print(string.format("[LOG] %s <- %s (%.2fms)", 
        ctx.method or "unknown",
        err and "ERROR: " .. err.message or "OK",
        elapsed))
    
    return result, err
end

-- Rate limiting interceptor
local rate_limiter = {}
local function rate_limit_interceptor(ctx, request, next)
    local client_id = ctx.client_id or "anonymous"
    local now = os.time()
    
    if not rate_limiter[client_id] then
        rate_limiter[client_id] = {count = 0, reset_at = now + 60}
    end
    
    local limiter = rate_limiter[client_id]
    
    -- Reset counter ทุกนาที
    if now > limiter.reset_at then
        limiter.count = 0
        limiter.reset_at = now + 60
    end
    
    -- ตรวจสอบ rate limit
    local max_requests = 100  -- 100 requests per minute
    if limiter.count >= max_requests then
        return nil, GrpcStatus.new("RESOURCE_EXHAUSTED",
            string.format("Rate limit exceeded: %d requests per minute", max_requests))
    end
    
    limiter.count = limiter.count + 1
    return next(ctx, request)
end

-- Recovery interceptor (catch panics)
local function recovery_interceptor(ctx, request, next)
    local ok, result_or_err = pcall(next, ctx, request)
    if not ok then
        print("[PANIC] Recovered from error: " .. tostring(result_or_err))
        return nil, GrpcStatus.new("INTERNAL", 
            "Internal server error")
    end
    return result_or_err
end

-- ทดสอบ interceptor chain
print("=== Interceptor Chain ===")
local chain = InterceptorChain.new()
    :add(recovery_interceptor)
    :add(logging_interceptor)
    :add(rate_limit_interceptor)

local handler = chain:build(function(ctx, req)
    return UserServiceImpl.GetUser(req, {
        set_status = function() end
    }), nil
end)

local ctx = {method = "GetUser", client_id = "client-123"}
local result, err = handler(ctx, {id = 2})
if result then
    print("Result: " .. (result.user and result.user.name or "nil"))
end
```

## การจัดการ Connection Pooling

```lua
-- Connection Pool สำหรับ gRPC

local ConnectionPool = {}
ConnectionPool.__index = ConnectionPool

function ConnectionPool.new(options)
    local self = setmetatable({}, ConnectionPool)
    self.min_size = options.min_size or 2
    self.max_size = options.max_size or 10
    self.target = options.target
    self.connections = {}
    self.available = {}
    self.waiting = {}
    self.created = 0
    return self
end

function ConnectionPool:acquire()
    -- ตรวจสอบ connection ที่พร้อมใช้งาน
    if #self.available > 0 then
        local conn = table.remove(self.available)
        print(string.format("Pool: Reusing connection (available: %d)", #self.available))
        return conn
    end
    
    -- สร้าง connection ใหม่ถ้ายังไม่เกิน max_size
    if self.created < self.max_size then
        self.created = self.created + 1
        local conn = {
            id = self.created,
            target = self.target,
            created_at = os.time(),
            in_use = true,
        }
        table.insert(self.connections, conn)
        print(string.format("Pool: Created new connection #%d (total: %d/%d)", 
            conn.id, self.created, self.max_size))
        return conn
    end
    
    -- Pool เต็ม - รอ connection ว่าง
    print("Pool: Waiting for available connection...")
    return nil, "pool exhausted"
end

function ConnectionPool:release(conn)
    conn.in_use = false
    table.insert(self.available, conn)
    print(string.format("Pool: Released connection #%d (available: %d)", 
        conn.id, #self.available))
end

function ConnectionPool:get_stats()
    return {
        total = self.created,
        available = #self.available,
        in_use = self.created - #self.available,
        max_size = self.max_size
    }
end

-- ทดสอบ connection pool
print("=== Connection Pool ===")
local pool = ConnectionPool.new({
    min_size = 2,
    max_size = 5,
    target = "user-service:50051"
})

-- Acquire connections
local connections = {}
for i = 1, 4 do
    local conn, err = pool:acquire()
    if conn then
        table.insert(connections, conn)
    end
end

-- แสดง stats
local stats = pool:get_stats()
print(string.format("\nPool Stats: total=%d, in_use=%d, available=%d, max=%d",
    stats.total, stats.in_use, stats.available, stats.max_size))

-- Release บางส่วน
print("\nReleasing 2 connections...")
pool:release(connections[1])
pool:release(connections[2])

stats = pool:get_stats()
print(string.format("After release: in_use=%d, available=%d", 
    stats.in_use, stats.available))

-- Re-acquire
print("\nAcquiring connection again...")
local conn = pool:acquire()
if conn then
    print("Got connection #" .. conn.id)
end
```

## Advanced: Retry Policy

```lua
-- Retry policy สำหรับ gRPC

local RetryPolicy = {}

-- กำหนด retry conditions
RetryPolicy.RETRYABLE_STATUS_CODES = {
    [14] = true,  -- UNAVAILABLE
    [4]  = true,  -- DEADLINE_EXCEEDED
    [13] = true,  -- INTERNAL (บางกรณี)
}

function RetryPolicy.new(options)
    return {
        max_attempts = options.max_attempts or 3,
        initial_backoff = options.initial_backoff or 0.1,  -- seconds
        max_backoff = options.max_backoff or 10,
        backoff_multiplier = options.backoff_multiplier or 2,
        retryable_status_codes = options.retryable_status_codes or RetryPolicy.RETRYABLE_STATUS_CODES
    }
end

function RetryPolicy.should_retry(policy, attempt, status)
    if attempt >= policy.max_attempts then
        return false
    end
    
    if not policy.retryable_status_codes[status.code] then
        return false
    end
    
    return true
end

function RetryPolicy.get_backoff(policy, attempt)
    local backoff = policy.initial_backoff * (policy.backoff_multiplier ^ (attempt - 1))
    -- Add jitter
    backoff = backoff * (0.8 + math.random() * 0.4)
    return math.min(backoff, policy.max_backoff)
end

-- Execute with retry
local function execute_with_retry(fn, policy, ...)
    local attempt = 0
    
    while true do
        attempt = attempt + 1
        
        local result, err = fn(...)
        
        if not err then
            if attempt > 1 then
                print(string.format("Succeeded after %d attempts", attempt))
            end
            return result, nil
        end
        
        if not RetryPolicy.should_retry(policy, attempt, err) then
            print(string.format("Not retrying after attempt %d: %s", attempt, err.message))
            return nil, err
        end
        
        local backoff = RetryPolicy.get_backoff(policy, attempt)
        print(string.format("Attempt %d failed (%s), retrying in %.2fs...", 
            attempt, err.message, backoff))
        
        -- ใน production จะ sleep จริง
        -- os.execute(string.format("sleep %g", backoff))
    end
end

-- ทดสอบ retry
math.randomseed(123)
local retry_policy = RetryPolicy.new({
    max_attempts = 4,
    initial_backoff = 0.1,
    backoff_multiplier = 2,
    max_backoff = 5,
})

local call_count = 0
local function flaky_service(user_id)
    call_count = call_count + 1
    -- จำลอง service ที่ล้มเหลว 2 ครั้งแรก
    if call_count <= 2 then
        return nil, {code = 14, message = "Service temporarily unavailable"}
    end
    return {user = users_db[user_id]}, nil
end

print("=== Retry Policy Test ===")
call_count = 0
local result, err = execute_with_retry(flaky_service, retry_policy, 1)
if result then
    print("Final result: " .. result.user.name)
else
    print("Final error: " .. err.message)
end
```

## สรุปบทที่ 71

```lua
-- สรุปสิ่งที่ได้เรียนรู้ในบทที่ 71

local summary = {
    title = "บทที่ 71: gRPC กับ Lua",
    topics = {
        "gRPC vs REST - ความแตกต่างและข้อดีข้อเสีย",
        "Protocol Buffers - การ define schema และ encoding",
        "gRPC Service Types - Unary, Server/Client/Bidirectional Streaming",
        "grpc-lua library - การติดตั้งและใช้งาน",
        "Error Handling - Status codes และ Rich Error Model",
        "Metadata - การส่ง key-value data กับ RPC calls",
        "Deadlines/Timeouts - การกำหนดเวลาสูงสุด",
        "Authentication - JWT, TLS, mTLS",
        "gRPC Gateway - แปลง REST เป็น gRPC",
        "Server Reflection - introspection",
        "Health Checking - ตรวจสอบสถานะ service",
        "Load Balancing - Round Robin, Weighted, Random",
        "Interceptors - Middleware สำหรับ cross-cutting concerns",
        "Connection Pooling - การจัดการ connections",
        "Retry Policy - การ retry เมื่อเกิดข้อผิดพลาด",
    },
    key_points = {
        "gRPC ใช้ HTTP/2 + Protocol Buffers ทำให้เร็วกว่า REST",
        "Strongly typed interfaces ลด bugs จาก wrong data format",
        "Streaming support ทั้ง 4 รูปแบบ",
        "gRPC Gateway ช่วย bridge REST clients กับ gRPC services",
        "Interceptors เป็น pattern สำคัญสำหรับ middleware",
    }
}

print("=" .. string.rep("=", 50))
print(summary.title)
print("=" .. string.rep("=", 50))

print("\nหัวข้อที่เรียน:")
for i, topic in ipairs(summary.topics) do
    print(string.format("  %2d. %s", i, topic))
end

print("\nประเด็นสำคัญ:")
for _, point in ipairs(summary.key_points) do
    print("  * " .. point)
end
```
