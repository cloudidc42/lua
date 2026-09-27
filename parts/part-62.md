# บทที่ 62: Game Engine Concepts

## บทนำ

ในบทนี้เราจะเรียนรู้ concept สำคัญในการพัฒนา game engine ตั้งแต่ Entity Component System (ECS), Scene Management, Object Pooling, ไปจนถึง A* Pathfinding และ AI State Machines

## 1. Entity Component System (ECS) Architecture

ECS แบ่งออกเป็น 3 ส่วนหลัก:
- **Entity** = ID เปล่าๆ (แค่ตัวเลข)
- **Component** = ข้อมูล (struct/table)
- **System** = logic ที่ประมวลผล components

```lua
-- Example 1: Basic ECS foundation
local ECS = {}
ECS.__index = ECS

function ECS.new()
    local self = setmetatable({}, ECS)
    self.nextId = 1
    self.entities = {}          -- set of active entity IDs
    self.components = {}        -- componentType -> {entityId -> data}
    self.systems = {}           -- ordered list of systems
    self.toDestroy = {}
    return self
end

-- สร้าง Entity ใหม่
function ECS:createEntity()
    local id = self.nextId
    self.nextId = self.nextId + 1
    self.entities[id] = true
    return id
end

-- ลบ Entity (lazy deletion)
function ECS:destroyEntity(id)
    table.insert(self.toDestroy, id)
end

function ECS:_flushDestroyed()
    for _, id in ipairs(self.toDestroy) do
        self.entities[id] = nil
        for compType, compStore in pairs(self.components) do
            compStore[id] = nil
        end
    end
    self.toDestroy = {}
end

-- เพิ่ม Component ให้ Entity
function ECS:addComponent(entityId, componentType, data)
    if not self.components[componentType] then
        self.components[componentType] = {}
    end
    self.components[componentType][entityId] = data
    return data
end

-- ดึง Component
function ECS:getComponent(entityId, componentType)
    local store = self.components[componentType]
    if store then return store[entityId] end
    return nil
end

-- ลบ Component
function ECS:removeComponent(entityId, componentType)
    if self.components[componentType] then
        self.components[componentType][entityId] = nil
    end
end

-- ตรวจสอบว่า Entity มี Component
function ECS:hasComponent(entityId, componentType)
    return self.components[componentType] ~= nil and
           self.components[componentType][entityId] ~= nil
end

-- query entities ที่มี components ทั้งหมดที่ระบุ
function ECS:query(...)
    local types = {...}
    local result = {}
    
    for id in pairs(self.entities) do
        local hasAll = true
        for _, t in ipairs(types) do
            if not self:hasComponent(id, t) then
                hasAll = false
                break
            end
        end
        if hasAll then
            table.insert(result, id)
        end
    end
    return result
end

-- เพิ่ม System
function ECS:addSystem(system)
    table.insert(self.systems, system)
    if system.init then system:init(self) end
end

-- update ทุก systems
function ECS:update(dt)
    for _, system in ipairs(self.systems) do
        if system.update then system:update(self, dt) end
    end
    self:_flushDestroyed()
end

-- draw
function ECS:draw()
    for _, system in ipairs(self.systems) do
        if system.draw then system:draw(self) end
    end
end

-- การใช้งาน
local world = ECS.new()

local player = world:createEntity()
world:addComponent(player, "transform", {x = 100, y = 200, rotation = 0, scaleX = 1, scaleY = 1})
world:addComponent(player, "velocity", {vx = 0, vy = 0})
world:addComponent(player, "health", {hp = 100, maxHp = 100})
world:addComponent(player, "sprite", {image = nil, w = 32, h = 32, color = {1, 0.5, 0, 1}})
world:addComponent(player, "player_input", {})

print("Player entity ID:", player)
```

## 2. Transform Component

```lua
-- Example 2: Transform component with hierarchy
local TransformSystem = {}
TransformSystem.__index = TransformSystem

function TransformSystem.newComponent(x, y, rotation, scaleX, scaleY)
    return {
        x = x or 0,
        y = y or 0,
        rotation = rotation or 0,
        scaleX = scaleX or 1,
        scaleY = scaleY or 1,
        parent = nil,
        -- World space (computed)
        worldX = 0,
        worldY = 0,
        worldRotation = 0,
        worldScaleX = 1,
        worldScaleY = 1,
        dirty = true
    }
end

function TransformSystem:update(world, dt)
    -- Update world transforms
    for _, id in ipairs(world:query("transform")) do
        local t = world:getComponent(id, "transform")
        if t.dirty then
            if t.parent then
                local p = world:getComponent(t.parent, "transform")
                if p then
                    -- Combine parent and local transforms
                    local cos = math.cos(p.worldRotation)
                    local sin = math.sin(p.worldRotation)
                    local lx = t.x * p.worldScaleX
                    local ly = t.y * p.worldScaleY
                    t.worldX = p.worldX + cos * lx - sin * ly
                    t.worldY = p.worldY + sin * lx + cos * ly
                    t.worldRotation = p.worldRotation + t.rotation
                    t.worldScaleX = p.worldScaleX * t.scaleX
                    t.worldScaleY = p.worldScaleY * t.scaleY
                end
            else
                t.worldX = t.x
                t.worldY = t.y
                t.worldRotation = t.rotation
                t.worldScaleX = t.scaleX
                t.worldScaleY = t.scaleY
            end
            t.dirty = false
        end
    end
end

-- Matrix helper
local function makeMatrix(tx, ty, rot, sx, sy)
    local cos = math.cos(rot)
    local sin = math.sin(rot)
    return {
        cos * sx, -sin * sy, tx,
        sin * sx,  cos * sy, ty,
        0, 0, 1
    }
end

local function transformPoint(matrix, x, y)
    return matrix[1]*x + matrix[2]*y + matrix[3],
           matrix[4]*x + matrix[5]*y + matrix[6]
end

-- ตัวอย่าง parent-child hierarchy
local parent = world:createEntity()
world:addComponent(parent, "transform", TransformSystem.newComponent(400, 300, 0))

local child = world:createEntity()
local childTransform = TransformSystem.newComponent(50, 0, 0)
childTransform.parent = parent
world:addComponent(child, "transform", childTransform)

print("Child follows parent automatically")
```

## 3. Sprite Component และ Render System

```lua
-- Example 3: Sprite component and render system
local SpriteComponent = {}

function SpriteComponent.new(options)
    return {
        image = options.image,
        quad = options.quad,
        width = options.width or 32,
        height = options.height or 32,
        originX = options.originX,  -- nil = center
        originY = options.originY,
        color = options.color or {1, 1, 1, 1},
        visible = true,
        layer = options.layer or 0,  -- สำหรับ sorting
        flipX = false,
        flipY = false
    }
end

local RenderSystem = {}
RenderSystem.__index = RenderSystem

function RenderSystem:draw(world)
    -- รวบรวม entities ที่มีทั้ง transform และ sprite
    local drawList = {}
    
    for _, id in ipairs(world:query("transform", "sprite")) do
        local sprite = world:getComponent(id, "sprite")
        if sprite.visible then
            table.insert(drawList, id)
        end
    end
    
    -- เรียงตาม layer
    table.sort(drawList, function(a, b)
        local sa = world:getComponent(a, "sprite")
        local sb = world:getComponent(b, "sprite")
        return sa.layer < sb.layer
    end)
    
    -- วาด
    for _, id in ipairs(drawList) do
        local t = world:getComponent(id, "transform")
        local s = world:getComponent(id, "sprite")
        
        local ox = s.originX or s.width / 2
        local oy = s.originY or s.height / 2
        local sx = s.flipX and -t.worldScaleX or t.worldScaleX
        local sy = s.flipY and -t.worldScaleY or t.worldScaleY
        
        love.graphics.setColor(s.color[1], s.color[2], s.color[3], s.color[4] or 1)
        
        if s.image then
            if s.quad then
                love.graphics.draw(s.image, s.quad, t.worldX, t.worldY, t.worldRotation, sx, sy, ox, oy)
            else
                love.graphics.draw(s.image, t.worldX, t.worldY, t.worldRotation, sx, sy, ox, oy)
            end
        else
            -- วาด placeholder rectangle
            love.graphics.rectangle("fill",
                t.worldX - ox * math.abs(sx),
                t.worldY - oy * math.abs(sy),
                s.width * math.abs(sx),
                s.height * math.abs(sy)
            )
        end
    end
    
    love.graphics.setColor(1, 1, 1, 1)
end

-- Debug render
local DebugRenderSystem = {}

function DebugRenderSystem:draw(world)
    if not debugMode then return end
    
    for _, id in ipairs(world:query("transform")) do
        local t = world:getComponent(id, "transform")
        love.graphics.setColor(0, 1, 0, 0.5)
        love.graphics.circle("line", t.worldX, t.worldY, 5)
        
        -- Draw axes
        love.graphics.setColor(1, 0, 0, 0.8)
        local ax = t.worldX + math.cos(t.worldRotation) * 20
        local ay = t.worldY + math.sin(t.worldRotation) * 20
        love.graphics.line(t.worldX, t.worldY, ax, ay)
        
        -- Velocity if exists
        local vel = world:getComponent(id, "velocity")
        if vel then
            love.graphics.setColor(0, 0, 1, 0.8)
            love.graphics.line(t.worldX, t.worldY,
                t.worldX + vel.vx * 0.1,
                t.worldY + vel.vy * 0.1)
        end
    end
    love.graphics.setColor(1, 1, 1, 1)
end

debugMode = false
```

## 4. Physics Component

```lua
-- Example 4: Physics component
local PhysicsComponent = {}

function PhysicsComponent.new(options)
    return {
        vx = options.vx or 0,
        vy = options.vy or 0,
        ax = options.ax or 0,        -- acceleration
        ay = options.ay or 0,
        mass = options.mass or 1,
        friction = options.friction or 0.98,  -- damping
        restitution = options.restitution or 0.3,
        isStatic = options.isStatic or false,
        gravityScale = options.gravityScale or 1,
        collider = options.collider  -- "circle" or "rect"
    }
end

local PhysicsSystem = {}
PhysicsSystem.__index = PhysicsSystem

local GRAVITY = 980

function PhysicsSystem:update(world, dt)
    local entities = world:query("transform", "physics")
    
    -- Update velocities
    for _, id in ipairs(entities) do
        local physics = world:getComponent(id, "physics")
        if physics.isStatic then goto continue end
        
        -- Apply gravity
        physics.vy = physics.vy + GRAVITY * physics.gravityScale * dt
        
        -- Apply acceleration
        physics.vx = physics.vx + physics.ax * dt
        physics.vy = physics.vy + physics.ay * dt
        
        -- Apply friction/damping
        physics.vx = physics.vx * physics.friction
        
        -- Update position
        local t = world:getComponent(id, "transform")
        t.x = t.x + physics.vx * dt
        t.y = t.y + physics.vy * dt
        t.dirty = true
        
        ::continue::
    end
    
    -- Simple circle-circle collision
    for i, id1 in ipairs(entities) do
        for j = i + 1, #entities do
            local id2 = entities[j]
            local p1 = world:getComponent(id1, "physics")
            local p2 = world:getComponent(id2, "physics")
            local t1 = world:getComponent(id1, "transform")
            local t2 = world:getComponent(id2, "transform")
            
            if p1.collider == "circle" and p2.collider == "circle" then
                local r1 = p1.collider_r or 16
                local r2 = p2.collider_r or 16
                local dx = t2.x - t1.x
                local dy = t2.y - t1.y
                local dist = math.sqrt(dx*dx + dy*dy)
                local minDist = r1 + r2
                
                if dist < minDist and dist > 0 then
                    -- Resolve collision
                    local nx = dx / dist
                    local ny = dy / dist
                    local overlap = minDist - dist
                    
                    if not p1.isStatic and not p2.isStatic then
                        t1.x = t1.x - nx * overlap * 0.5
                        t1.y = t1.y - ny * overlap * 0.5
                        t2.x = t2.x + nx * overlap * 0.5
                        t2.y = t2.y + ny * overlap * 0.5
                    elseif not p1.isStatic then
                        t1.x = t1.x - nx * overlap
                        t1.y = t1.y - ny * overlap
                    elseif not p2.isStatic then
                        t2.x = t2.x + nx * overlap
                        t2.y = t2.y + ny * overlap
                    end
                    
                    -- Reflect velocities
                    if not p1.isStatic and not p2.isStatic then
                        local relVx = p1.vx - p2.vx
                        local relVy = p1.vy - p2.vy
                        local relDotN = relVx * nx + relVy * ny
                        local impulse = -(1 + math.min(p1.restitution, p2.restitution)) * relDotN
                        impulse = impulse / (1/p1.mass + 1/p2.mass)
                        p1.vx = p1.vx + impulse / p1.mass * nx
                        p1.vy = p1.vy + impulse / p1.mass * ny
                        p2.vx = p2.vx - impulse / p2.mass * nx
                        p2.vy = p2.vy - impulse / p2.mass * ny
                    end
                    
                    t1.dirty = true
                    t2.dirty = true
                    
                    -- Fire collision event
                    local events = world:getComponent(id1, "event_listener")
                    if events and events.onCollide then
                        events.onCollide(id1, id2)
                    end
                end
            end
        end
    end
end
```

## 5. Input System

```lua
-- Example 5: Input system with action mapping
local InputSystem = {}
InputSystem.__index = InputSystem

function InputSystem.new()
    local self = setmetatable({}, InputSystem)
    
    -- Action -> keys mapping
    self.bindings = {
        move_left  = {"left", "a"},
        move_right = {"right", "d"},
        move_up    = {"up", "w"},
        move_down  = {"down", "s"},
        jump       = {"space"},
        fire       = {"z", "lctrl"},
        pause      = {"escape", "p"},
        interact   = {"e", "return"},
    }
    
    -- Gamepad bindings
    self.gpBindings = {
        move_left  = {"leftx<-0.3"},
        move_right = {"leftx>0.3"},
        jump       = {"a"},
        fire       = {"x"},
        pause      = {"start"},
    }
    
    self.pressed = {}    -- pressed this frame
    self.released = {}   -- released this frame
    self.held = {}       -- currently held
    self.axes = {}       -- analog axes
    
    return self
end

function InputSystem:isDown(action)
    local keys = self.bindings[action]
    if not keys then return false end
    for _, key in ipairs(keys) do
        if love.keyboard.isDown(key) then return true end
    end
    return false
end

function InputSystem:wasPressed(action)
    return self.pressed[action] == true
end

function InputSystem:wasReleased(action)
    return self.released[action] == true
end

function InputSystem:getAxis(action)
    -- Returns -1 to 1
    local left = self:isDown(action .. "_left") and 1 or 0
    local right = self:isDown(action .. "_right") and 1 or 0
    return right - left
end

function InputSystem:update(world, dt)
    self.pressed = {}
    self.released = {}
end

function InputSystem:keypressed(key)
    for action, keys in pairs(self.bindings) do
        for _, k in ipairs(keys) do
            if k == key then
                self.pressed[action] = true
                self.held[action] = true
            end
        end
    end
end

function InputSystem:keyreleased(key)
    for action, keys in pairs(self.bindings) do
        for _, k in ipairs(keys) do
            if k == key then
                self.released[action] = true
                self.held[action] = false
            end
        end
    end
end

-- Player Input System ใน ECS
local PlayerInputSystem = {}

function PlayerInputSystem:update(world, dt)
    local input = world.input  -- shared input system
    
    for _, id in ipairs(world:query("player_input", "physics", "transform")) do
        local physics = world:getComponent(id, "physics")
        local speed = 300
        
        if input:isDown("move_left") then physics.vx = -speed
        elseif input:isDown("move_right") then physics.vx = speed
        else physics.vx = physics.vx * 0.7 end
        
        if input:wasPressed("jump") then
            physics.vy = -600
        end
        
        -- Update sprite flip
        local sprite = world:getComponent(id, "sprite")
        if sprite then
            if physics.vx < 0 then sprite.flipX = true
            elseif physics.vx > 0 then sprite.flipX = false end
        end
    end
end

-- Demo usage
local inputSys = InputSystem.new()

function love.load()
    world2 = ECS.new()
    world2.input = inputSys
    
    local e = world2:createEntity()
    world2:addComponent(e, "transform", {x=400, y=300, rotation=0, scaleX=1, scaleY=1, worldX=400, worldY=300, worldRotation=0, worldScaleX=1, worldScaleY=1, dirty=false})
    world2:addComponent(e, "physics", {vx=0, vy=0, ax=0, ay=0, mass=1, friction=0.98, isStatic=false, gravityScale=1, restitution=0.3, collider="circle", collider_r=20})
    world2:addComponent(e, "player_input", {})
    world2:addComponent(e, "sprite", {width=40, height=40, color={0,0.7,1,1}, visible=true, layer=0, flipX=false, flipY=false})
    
    world2:addSystem(PlayerInputSystem)
    world2:addSystem(PhysicsSystem)
    world2:addSystem(RenderSystem)
end

function love.update(dt)
    inputSys:update(world2, dt)
    world2:update(dt)
end

function love.draw()
    world2:draw()
end

function love.keypressed(key)
    inputSys:keypressed(key)
end

function love.keyreleased(key)
    inputSys:keyreleased(key)
end
```

## 6. Scene Management

```lua
-- Example 6: Scene management system
local SceneManager = {}
SceneManager.__index = SceneManager

function SceneManager.new()
    local self = setmetatable({}, SceneManager)
    self.scenes = {}
    self.current = nil
    self.next = nil
    self.transitioning = false
    self.transition = nil
    return self
end

function SceneManager:register(name, scene)
    self.scenes[name] = scene
end

function SceneManager:switch(name, params)
    if not self.scenes[name] then
        error("Scene not found: " .. name)
    end
    
    if self.current and self.scenes[self.current] then
        local scene = self.scenes[self.current]
        if scene.exit then scene:exit() end
    end
    
    self.current = name
    local scene = self.scenes[name]
    scene.params = params
    if scene.enter then scene:enter(params) end
end

function SceneManager:switchWithTransition(name, transitionType, params)
    self.next = name
    self.nextParams = params
    self.transitioning = true
    self.transition = {
        type = transitionType,
        progress = 0,
        duration = 0.5
    }
end

function SceneManager:update(dt)
    if self.transitioning then
        self.transition.progress = self.transition.progress + dt / self.transition.duration
        
        if self.transition.progress >= 0.5 and self.next then
            self:switch(self.next, self.nextParams)
            self.next = nil
        end
        
        if self.transition.progress >= 1 then
            self.transitioning = false
            self.transition = nil
        end
    end
    
    if self.current and self.scenes[self.current] then
        local scene = self.scenes[self.current]
        if scene.update then scene:update(dt) end
    end
end

function SceneManager:draw()
    if self.current and self.scenes[self.current] then
        self.scenes[self.current]:draw()
    end
    
    -- Draw transition overlay
    if self.transitioning and self.transition then
        local p = self.transition.progress
        local alpha
        if p < 0.5 then
            alpha = p * 2
        else
            alpha = (1 - p) * 2
        end
        love.graphics.setColor(0, 0, 0, alpha)
        love.graphics.rectangle("fill", 0, 0, love.graphics.getWidth(), love.graphics.getHeight())
        love.graphics.setColor(1, 1, 1, 1)
    end
end

-- Define scenes
local MenuScene = {
    enter = function(self)
        self.title = "MY GAME"
        self.menuItems = {"Play", "Options", "Quit"}
        self.selected = 1
    end,
    update = function(self, dt) end,
    draw = function(self)
        love.graphics.setColor(0.1, 0.1, 0.3, 1)
        love.graphics.rectangle("fill", 0, 0, 800, 600)
        love.graphics.setColor(1, 1, 0, 1)
        love.graphics.printf(self.title, 0, 150, 800, "center")
        for i, item in ipairs(self.menuItems) do
            if i == self.selected then
                love.graphics.setColor(1, 1, 1, 1)
            else
                love.graphics.setColor(0.6, 0.6, 0.6, 1)
            end
            love.graphics.printf(item, 0, 280 + i * 50, 800, "center")
        end
    end,
    keypressed = function(self, key)
        if key == "up" then self.selected = math.max(1, self.selected - 1) end
        if key == "down" then self.selected = math.min(#self.menuItems, self.selected + 1) end
        if key == "return" then
            if self.selected == 1 then sceneManager:switch("game") end
            if self.selected == 3 then love.event.quit() end
        end
    end
}

local GameScene = {
    enter = function(self)
        self.score = 0
    end,
    update = function(self, dt)
        self.score = self.score + dt
    end,
    draw = function(self)
        love.graphics.setColor(0, 0.3, 0, 1)
        love.graphics.rectangle("fill", 0, 0, 800, 600)
        love.graphics.setColor(1, 1, 1, 1)
        love.graphics.printf("PLAYING\nScore: " .. math.floor(self.score), 0, 280, 800, "center")
        love.graphics.printf("ESC = Back to menu", 0, 400, 800, "center")
    end,
    keypressed = function(self, key)
        if key == "escape" then
            sceneManager:switchWithTransition("menu", "fade")
        end
    end
}

function love.load()
    sceneManager = SceneManager.new()
    sceneManager:register("menu", MenuScene)
    sceneManager:register("game", GameScene)
    sceneManager:switch("menu")
end

function love.update(dt)
    sceneManager:update(dt)
end

function love.draw()
    sceneManager:draw()
end

function love.keypressed(key)
    if sceneManager.current then
        local scene = sceneManager.scenes[sceneManager.current]
        if scene and scene.keypressed then
            scene:keypressed(key)
        end
    end
end
```

## 7. Object Pooling

```lua
-- Example 7: Object pooling system
local ObjectPool = {}
ObjectPool.__index = ObjectPool

function ObjectPool.new(factory, reset, initialSize)
    local self = setmetatable({}, ObjectPool)
    self.factory = factory      -- สร้าง object ใหม่
    self.reset = reset          -- reset object กลับสู่ default
    self.available = {}         -- available objects
    self.active = {}            -- currently active objects
    self.created = 0
    
    -- Pre-allocate
    for i = 1, initialSize or 0 do
        local obj = factory()
        table.insert(self.available, obj)
        self.created = self.created + 1
    end
    
    return self
end

function ObjectPool:acquire()
    local obj
    if #self.available > 0 then
        obj = table.remove(self.available)
    else
        obj = self.factory()
        self.created = self.created + 1
    end
    
    if self.reset then self.reset(obj) end
    table.insert(self.active, obj)
    obj._poolIndex = #self.active
    return obj
end

function ObjectPool:release(obj)
    -- ค้นหาใน active list
    for i, a in ipairs(self.active) do
        if a == obj then
            table.remove(self.active, i)
            -- อัปเดต indices
            for j = i, #self.active do
                self.active[j]._poolIndex = j
            end
            break
        end
    end
    
    obj._poolIndex = nil
    table.insert(self.available, obj)
end

function ObjectPool:releaseAll()
    for _, obj in ipairs(self.active) do
        obj._poolIndex = nil
        table.insert(self.available, obj)
    end
    self.active = {}
end

function ObjectPool:getStats()
    return {
        created = self.created,
        active = #self.active,
        available = #self.available
    }
end

-- ตัวอย่าง: Bullet pool
local function createBullet()
    return {
        x = 0, y = 0,
        vx = 0, vy = 0,
        w = 4, h = 12,
        damage = 1,
        active = false,
        color = {1, 1, 0, 1}
    }
end

local function resetBullet(b)
    b.x = 0; b.y = 0
    b.vx = 0; b.vy = 0
    b.active = true
end

-- ตัวอย่าง: Particle pool
local function createParticle()
    return {x=0, y=0, vx=0, vy=0, life=0, maxLife=1, size=1, color={1,1,1,1}}
end

local function resetParticle(p)
    p.life = 1; p.maxLife = 1
end

function love.load()
    bulletPool = ObjectPool.new(createBullet, resetBullet, 100)
    particlePool = ObjectPool.new(createParticle, resetParticle, 500)
    
    playerX, playerY = 400, 500
    shootTimer = 0
end

function love.update(dt)
    -- Shoot
    shootTimer = shootTimer + dt
    if love.keyboard.isDown("space") and shootTimer > 0.1 then
        shootTimer = 0
        local b = bulletPool:acquire()
        b.x = playerX
        b.y = playerY
        b.vy = -500
    end
    
    -- Update active bullets
    for i = #bulletPool.active, 1, -1 do
        local b = bulletPool.active[i]
        b.y = b.y + b.vy * dt
        if b.y < -20 then
            bulletPool:release(b)
        end
    end
    
    -- Update particles
    for i = #particlePool.active, 1, -1 do
        local p = particlePool.active[i]
        p.x = p.x + p.vx * dt
        p.y = p.y + p.vy * dt
        p.life = p.life - dt
        if p.life <= 0 then
            particlePool:release(p)
        end
    end
end

function love.keypressed(key)
    if key == "e" then
        -- Spawn explosion particles
        for i = 1, 20 do
            local p = particlePool:acquire()
            local angle = math.random() * math.pi * 2
            local speed = math.random(50, 200)
            p.x = playerX; p.y = playerY
            p.vx = math.cos(angle) * speed
            p.vy = math.sin(angle) * speed
            p.life = 0.5 + math.random() * 0.5
            p.maxLife = p.life
            p.size = math.random(2, 6)
            p.color = {1, math.random(), 0, 1}
        end
    end
end

function love.draw()
    love.graphics.setColor(0.05, 0.05, 0.1, 1)
    love.graphics.rectangle("fill", 0, 0, 800, 600)
    
    -- Draw bullets
    for _, b in ipairs(bulletPool.active) do
        love.graphics.setColor(b.color)
        love.graphics.rectangle("fill", b.x - b.w/2, b.y, b.w, b.h)
    end
    
    -- Draw particles
    for _, p in ipairs(particlePool.active) do
        local alpha = p.life / p.maxLife
        love.graphics.setColor(p.color[1], p.color[2], p.color[3], alpha)
        love.graphics.circle("fill", p.x, p.y, p.size * alpha)
    end
    
    -- Player
    love.graphics.setColor(0.2, 0.7, 1, 1)
    love.graphics.rectangle("fill", playerX - 15, playerY - 20, 30, 40)
    
    -- Stats
    local bStats = bulletPool:getStats()
    local pStats = particlePool:getStats()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print(string.format("Bullets: %d active, %d available, %d created",
        bStats.active, bStats.available, bStats.created), 10, 10)
    love.graphics.print(string.format("Particles: %d active, %d available, %d created",
        pStats.active, pStats.available, pStats.created), 10, 30)
    love.graphics.print("Space=Shoot, E=Explosion", 10, 50)
end
```

## 8. Quadtree สำหรับ Spatial Queries

```lua
-- Example 8: Quadtree for efficient spatial queries
local Quadtree = {}
Quadtree.__index = Quadtree

local MAX_OBJECTS = 8
local MAX_LEVELS = 5

function Quadtree.new(level, bounds)
    return setmetatable({
        level = level or 0,
        objects = {},
        bounds = bounds,  -- {x, y, w, h}
        nodes = {}        -- 4 children
    }, Quadtree)
end

function Quadtree:_split()
    local hw = self.bounds.w / 2
    local hh = self.bounds.h / 2
    local x = self.bounds.x
    local y = self.bounds.y
    
    self.nodes[1] = Quadtree.new(self.level + 1, {x = x + hw, y = y,      w = hw, h = hh})  -- NE
    self.nodes[2] = Quadtree.new(self.level + 1, {x = x,      y = y,      w = hw, h = hh})  -- NW
    self.nodes[3] = Quadtree.new(self.level + 1, {x = x,      y = y + hh, w = hw, h = hh})  -- SW
    self.nodes[4] = Quadtree.new(self.level + 1, {x = x + hw, y = y + hh, w = hw, h = hh})  -- SE
end

function Quadtree:_getIndex(obj)
    -- obj ต้องมี x, y, w, h
    local midX = self.bounds.x + self.bounds.w / 2
    local midY = self.bounds.y + self.bounds.h / 2
    
    local top    = obj.y < midY and obj.y + (obj.h or 0) < midY
    local bottom = obj.y >= midY
    
    if obj.x < midX and obj.x + (obj.w or 0) < midX then
        if top    then return 2 end
        if bottom then return 3 end
    elseif obj.x >= midX then
        if top    then return 1 end
        if bottom then return 4 end
    end
    
    return -1  -- spans multiple quadrants
end

function Quadtree:insert(obj)
    if #self.nodes > 0 then
        local idx = self:_getIndex(obj)
        if idx ~= -1 then
            self.nodes[idx]:insert(obj)
            return
        end
    end
    
    table.insert(self.objects, obj)
    
    if #self.objects > MAX_OBJECTS and self.level < MAX_LEVELS then
        if #self.nodes == 0 then self:_split() end
        
        local i = 1
        while i <= #self.objects do
            local idx = self:_getIndex(self.objects[i])
            if idx ~= -1 then
                local removed = table.remove(self.objects, i)
                self.nodes[idx]:insert(removed)
            else
                i = i + 1
            end
        end
    end
end

function Quadtree:retrieve(obj, result)
    result = result or {}
    
    if #self.nodes > 0 then
        local idx = self:_getIndex(obj)
        if idx ~= -1 then
            self.nodes[idx]:retrieve(obj, result)
        else
            -- Spans multiple, check all
            for _, node in ipairs(self.nodes) do
                node:retrieve(obj, result)
            end
        end
    end
    
    for _, o in ipairs(self.objects) do
        table.insert(result, o)
    end
    
    return result
end

function Quadtree:clear()
    self.objects = {}
    for _, node in ipairs(self.nodes) do
        node:clear()
    end
    self.nodes = {}
end

-- การใช้งาน
function love.load()
    qt = Quadtree.new(0, {x = 0, y = 0, w = 800, h = 600})
    
    enemies_qt = {}
    for i = 1, 200 do
        table.insert(enemies_qt, {
            x = math.random(10, 790),
            y = math.random(10, 590),
            w = 20, h = 20,
            id = i
        })
    end
    
    queryX, queryY = 400, 300
    queryW, queryH = 100, 100
    nearbyCount = 0
    
    font = love.graphics.newFont(14)
end

function love.update(dt)
    queryX = love.mouse.getX() - queryW/2
    queryY = love.mouse.getY() - queryH/2
    
    -- Rebuild quadtree each frame (หรือจะทำ dirty flag)
    qt:clear()
    for _, e in ipairs(enemies_qt) do
        qt:insert(e)
    end
    
    -- Query nearby
    local query = {x = queryX, y = queryY, w = queryW, h = queryH}
    local candidates = qt:retrieve(query)
    
    -- Mark nearby
    nearbyCount = 0
    for _, e in ipairs(enemies_qt) do e.nearby = false end
    for _, c in ipairs(candidates) do
        -- AABB check
        if c.x < queryX + queryW and c.x + c.w > queryX and
           c.y < queryY + queryH and c.y + c.h > queryY then
            c.nearby = true
            nearbyCount = nearbyCount + 1
        end
    end
end

function love.draw()
    -- Draw enemies
    for _, e in ipairs(enemies_qt) do
        if e.nearby then
            love.graphics.setColor(1, 0, 0, 1)
        else
            love.graphics.setColor(0.4, 0.4, 0.4, 0.8)
        end
        love.graphics.rectangle("fill", e.x, e.y, e.w, e.h)
    end
    
    -- Draw query region
    love.graphics.setColor(0, 1, 0, 0.3)
    love.graphics.rectangle("fill", queryX, queryY, queryW, queryH)
    love.graphics.setColor(0, 1, 0, 1)
    love.graphics.rectangle("line", queryX, queryY, queryW, queryH)
    
    love.graphics.setFont(font)
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Enemies: " .. #enemies_qt, 10, 10)
    love.graphics.print("Nearby: " .. nearbyCount, 10, 28)
    love.graphics.print("Move mouse to query", 10, 46)
end
```

## 9. A* Pathfinding

```lua
-- Example 9: A* pathfinding
local function heuristic(a, b)
    -- Manhattan distance
    return math.abs(a.x - b.x) + math.abs(a.y - b.y)
end

local AStar = {}
AStar.__index = AStar

function AStar.new(grid, walkable)
    -- grid[row][col] = tile value
    -- walkable = function(tile) return bool
    return setmetatable({
        grid = grid,
        walkable = walkable or function(t) return t == 0 end,
        rows = #grid,
        cols = #grid[1]
    }, AStar)
end

function AStar:findPath(startX, startY, goalX, goalY)
    -- Coordinates are grid cells (1-indexed)
    local openSet = {}
    local closedSet = {}
    
    local start = {x = startX, y = startY, g = 0, f = 0, parent = nil}
    start.h = heuristic(start, {x = goalX, y = goalY})
    start.f = start.g + start.h
    
    table.insert(openSet, start)
    
    while #openSet > 0 do
        -- Find node with lowest f score
        local lowestIdx = 1
        for i = 2, #openSet do
            if openSet[i].f < openSet[lowestIdx].f then
                lowestIdx = i
            end
        end
        
        local current = table.remove(openSet, lowestIdx)
        
        -- Goal reached
        if current.x == goalX and current.y == goalY then
            local path = {}
            local node = current
            while node do
                table.insert(path, 1, {x = node.x, y = node.y})
                node = node.parent
            end
            return path
        end
        
        -- Add to closed set
        closedSet[current.y .. "," .. current.x] = true
        
        -- Check neighbors (4-directional)
        local dirs = {{0,-1},{0,1},{-1,0},{1,0}}
        -- Diagonal: {{-1,-1},{1,-1},{-1,1},{1,1}}
        
        for _, dir in ipairs(dirs) do
            local nx = current.x + dir[1]
            local ny = current.y + dir[2]
            
            -- Bounds check
            if nx >= 1 and nx <= self.cols and ny >= 1 and ny <= self.rows then
                -- Walkable check
                if self.walkable(self.grid[ny][nx]) then
                    local key = ny .. "," .. nx
                    
                    if not closedSet[key] then
                        local gCost = current.g + 1
                        
                        -- Check if in open set
                        local existing = nil
                        for _, n in ipairs(openSet) do
                            if n.x == nx and n.y == ny then
                                existing = n
                                break
                            end
                        end
                        
                        if not existing or gCost < existing.g then
                            local neighbor = {
                                x = nx, y = ny,
                                g = gCost,
                                h = heuristic({x=nx,y=ny}, {x=goalX,y=goalY}),
                                parent = current
                            }
                            neighbor.f = neighbor.g + neighbor.h
                            
                            if not existing then
                                table.insert(openSet, neighbor)
                            else
                                existing.g = gCost
                                existing.f = gCost + existing.h
                                existing.parent = current
                            end
                        end
                    end
                end
            end
        end
    end
    
    return nil  -- No path found
end

-- การใช้งาน
function love.load()
    CELL = 32
    
    astarGrid = {
        {0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
        {0,0,0,1,0,0,0,0,0,1,1,1,1,0,0,0,0,0,0,0},
        {0,0,0,1,0,0,0,0,0,1,0,0,1,0,0,0,0,0,0,0},
        {0,0,0,0,0,0,0,0,0,1,0,0,1,0,0,1,1,1,0,0},
        {0,0,0,1,1,1,1,0,0,0,0,0,1,0,0,0,0,1,0,0},
        {0,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,1,0,0},
        {0,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0},
        {0,0,0,0,0,0,0,0,0,0,0,1,1,1,1,0,0,0,0,0},
        {0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0},
        {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
    }
    
    pathfinder = AStar.new(astarGrid)
    
    startCell = {x = 1, y = 1}
    goalCell = {x = 20, y = 10}
    
    path = pathfinder:findPath(startCell.x, startCell.y, goalCell.x, goalCell.y)
    
    font = love.graphics.newFont(12)
    
    -- Moving agent
    agentX, agentY = 1, 1
    agentPos = 1
    agentTimer = 0
    agentSpeed = 5  -- cells per second
end

function love.update(dt)
    if path and agentPos < #path then
        agentTimer = agentTimer + dt * agentSpeed
        if agentTimer >= 1 then
            agentTimer = 0
            agentPos = agentPos + 1
            agentX = path[agentPos].x
            agentY = path[agentPos].y
        end
    end
end

function love.mousepressed(x, y, button)
    local col = math.floor(x / CELL) + 1
    local row = math.floor(y / CELL) + 1
    
    if col >= 1 and col <= #astarGrid[1] and row >= 1 and row <= #astarGrid then
        if button == 1 then
            goalCell = {x = col, y = row}
            if astarGrid[row][col] ~= 1 then
                path = pathfinder:findPath(startCell.x, startCell.y, goalCell.x, goalCell.y)
                agentPos = 1
                agentX, agentY = startCell.x, startCell.y
            end
        elseif button == 2 then
            -- Toggle wall
            astarGrid[row][col] = astarGrid[row][col] == 1 and 0 or 1
            path = pathfinder:findPath(startCell.x, startCell.y, goalCell.x, goalCell.y)
        end
    end
end

function love.draw()
    -- Draw grid
    for row = 1, #astarGrid do
        for col = 1, #astarGrid[row] do
            local px = (col - 1) * CELL
            local py = (row - 1) * CELL
            
            if astarGrid[row][col] == 1 then
                love.graphics.setColor(0.3, 0.3, 0.35, 1)
            else
                love.graphics.setColor(0.15, 0.15, 0.2, 1)
            end
            love.graphics.rectangle("fill", px, py, CELL - 1, CELL - 1)
        end
    end
    
    -- Draw path
    if path then
        for i, node in ipairs(path) do
            local px = (node.x - 1) * CELL
            local py = (node.y - 1) * CELL
            local t = i / #path
            love.graphics.setColor(t, 1 - t, 0, 0.5)
            love.graphics.rectangle("fill", px + 8, py + 8, CELL - 17, CELL - 17)
        end
    else
        love.graphics.setColor(1, 0, 0, 0.5)
        love.graphics.printf("No path found!", 0, 280, 640, "center")
    end
    
    -- Start
    love.graphics.setColor(0, 1, 0, 1)
    love.graphics.circle("fill", (startCell.x - 0.5) * CELL, (startCell.y - 0.5) * CELL, 10)
    
    -- Goal
    love.graphics.setColor(1, 0, 0, 1)
    love.graphics.circle("fill", (goalCell.x - 0.5) * CELL, (goalCell.y - 0.5) * CELL, 10)
    
    -- Agent
    love.graphics.setColor(0, 0.7, 1, 1)
    love.graphics.circle("fill", (agentX - 0.5) * CELL, (agentY - 0.5) * CELL, 12)
    
    love.graphics.setFont(font)
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Left click=set goal | Right click=toggle wall", 10, 330)
    if path then
        love.graphics.print("Path length: " .. #path, 10, 346)
    end
end
```

## 10. Finite State Machine สำหรับ AI

```lua
-- Example 10: Finite State Machine (FSM) for AI
local FSM = {}
FSM.__index = FSM

function FSM.new(owner)
    return setmetatable({
        owner = owner,
        currentState = nil,
        previousState = nil,
        globalState = nil,
        states = {}
    }, FSM)
end

function FSM:addState(name, state)
    self.states[name] = state
end

function FSM:setState(name)
    if self.currentState then
        local state = self.states[self.currentState]
        if state and state.exit then state:exit(self.owner) end
    end
    
    self.previousState = self.currentState
    self.currentState = name
    
    local newState = self.states[name]
    if newState and newState.enter then
        newState:enter(self.owner)
    end
end

function FSM:revertToPrevious()
    if self.previousState then
        self:setState(self.previousState)
    end
end

function FSM:update(dt)
    if self.globalState then
        local gs = self.states[self.globalState]
        if gs and gs.update then gs:update(self.owner, dt) end
    end
    
    if self.currentState then
        local cs = self.states[self.currentState]
        if cs and cs.update then cs:update(self.owner, dt) end
    end
end

function FSM:isInState(name)
    return self.currentState == name
end

-- Enemy AI States
local EnemyStates = {}

EnemyStates.Idle = {
    enter = function(owner)
        owner.timer = 0
        print("Enemy entering Idle")
    end,
    update = function(owner, dt)
        owner.timer = owner.timer + dt
        
        -- Detect player
        local dx = owner.targetX - owner.x
        local dy = owner.targetY - owner.y
        local dist = math.sqrt(dx*dx + dy*dy)
        
        if dist < owner.detectionRange then
            owner.fsm:setState("Chase")
        elseif owner.timer > 3 then
            owner.fsm:setState("Patrol")
        end
    end,
    exit = function(owner) print("Enemy leaving Idle") end
}

EnemyStates.Patrol = {
    enter = function(owner)
        owner.patrolDir = 1
        owner.patrolTimer = 0
        print("Enemy entering Patrol")
    end,
    update = function(owner, dt)
        -- Move back and forth
        owner.x = owner.x + owner.patrolDir * 80 * dt
        owner.patrolTimer = owner.patrolTimer + dt
        
        if owner.patrolTimer > 2 then
            owner.patrolTimer = 0
            owner.patrolDir = -owner.patrolDir
        end
        
        -- Detect player
        local dx = owner.targetX - owner.x
        local dy = owner.targetY - owner.y
        local dist = math.sqrt(dx*dx + dy*dy)
        
        if dist < owner.detectionRange then
            owner.fsm:setState("Chase")
        end
    end,
    exit = function(owner) print("Enemy leaving Patrol") end
}

EnemyStates.Chase = {
    enter = function(owner)
        owner.chaseSpeed = 150
        print("Enemy entering Chase!")
    end,
    update = function(owner, dt)
        local dx = owner.targetX - owner.x
        local dy = owner.targetY - owner.y
        local dist = math.sqrt(dx*dx + dy*dy)
        
        if dist < 1 then return end
        
        -- Move toward player
        owner.x = owner.x + (dx/dist) * owner.chaseSpeed * dt
        owner.y = owner.y + (dy/dist) * owner.chaseSpeed * dt
        
        -- Attack range
        if dist < owner.attackRange then
            owner.fsm:setState("Attack")
        end
        
        -- Lost player
        if dist > owner.detectionRange * 1.5 then
            owner.fsm:setState("Idle")
        end
    end,
    exit = function(owner) print("Enemy leaving Chase") end
}

EnemyStates.Attack = {
    enter = function(owner)
        owner.attackTimer = 0
        owner.attackCooldown = 1.5
        print("Enemy ATTACKING!")
    end,
    update = function(owner, dt)
        owner.attackTimer = owner.attackTimer + dt
        
        local dx = owner.targetX - owner.x
        local dy = owner.targetY - owner.y
        local dist = math.sqrt(dx*dx + dy*dy)
        
        if owner.attackTimer >= owner.attackCooldown then
            owner.attackTimer = 0
            -- Do attack
            owner.lastAttackTime = love.timer.getTime()
        end
        
        if dist > owner.attackRange * 1.2 then
            owner.fsm:setState("Chase")
        end
    end,
    exit = function(owner) print("Enemy leaving Attack") end
}

EnemyStates.Dead = {
    enter = function(owner)
        owner.deathTimer = 0
        print("Enemy DEAD!")
    end,
    update = function(owner, dt)
        owner.deathTimer = owner.deathTimer + dt
    end
}

-- Enemy factory
local function createEnemy(x, y)
    local enemy = {
        x = x, y = y,
        hp = 100,
        targetX = 400, targetY = 300,
        detectionRange = 200,
        attackRange = 50,
        timer = 0,
        patrolDir = 1,
        patrolTimer = 0,
        lastAttackTime = 0
    }
    
    enemy.fsm = FSM.new(enemy)
    for name, state in pairs(EnemyStates) do
        enemy.fsm:addState(name, state)
    end
    enemy.fsm:setState("Idle")
    
    return enemy
end

function love.load()
    aiEnemies = {
        createEnemy(100, 300),
        createEnemy(600, 200),
        createEnemy(400, 450),
    }
    
    targetX, targetY = 400, 300
    font = love.graphics.newFont(12)
end

function love.update(dt)
    targetX = love.mouse.getX()
    targetY = love.mouse.getY()
    
    for _, enemy in ipairs(aiEnemies) do
        enemy.targetX = targetX
        enemy.targetY = targetY
        enemy.fsm:update(dt)
    end
end

function love.draw()
    love.graphics.setColor(0.1, 0.1, 0.15, 1)
    love.graphics.rectangle("fill", 0, 0, 800, 600)
    
    -- Player (mouse)
    love.graphics.setColor(0, 1, 0, 1)
    love.graphics.circle("fill", targetX, targetY, 15)
    love.graphics.setColor(0, 0.6, 0, 0.3)
    love.graphics.circle("line", targetX, targetY, 200)
    
    -- Enemies
    local stateColors = {
        Idle = {0.5, 0.5, 0.5},
        Patrol = {0, 0.5, 1},
        Chase = {1, 0.5, 0},
        Attack = {1, 0, 0},
        Dead = {0.2, 0.2, 0.2}
    }
    
    love.graphics.setFont(font)
    for i, enemy in ipairs(aiEnemies) do
        local state = enemy.fsm.currentState
        local color = stateColors[state] or {1, 1, 1}
        
        -- Detection range
        love.graphics.setColor(color[1], color[2], color[3], 0.1)
        love.graphics.circle("line", enemy.x, enemy.y, enemy.detectionRange)
        
        -- Body
        love.graphics.setColor(color[1], color[2], color[3], 1)
        love.graphics.circle("fill", enemy.x, enemy.y, 20)
        
        -- Attack indicator
        if state == "Attack" then
            love.graphics.setColor(1, 0, 0, 0.5)
            love.graphics.circle("line", enemy.x, enemy.y, enemy.attackRange)
        end
        
        -- State label
        love.graphics.setColor(1, 1, 1, 1)
        love.graphics.print(state or "?", enemy.x - 20, enemy.y - 30)
    end
    
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Move mouse to control player position", 10, 10)
end
```

## 11. Behavior Trees

```lua
-- Example 11: Behavior tree for complex AI
-- Node status
local SUCCESS = "success"
local FAILURE = "failure"
local RUNNING  = "running"

-- Base node
local BTNode = {}
BTNode.__index = BTNode
function BTNode:tick(context) return FAILURE end

-- Selector (OR) - tries children until one succeeds
local Selector = setmetatable({}, {__index = BTNode})
Selector.__index = Selector

function Selector.new(...)
    return setmetatable({children = {...}, current = 1}, Selector)
end

function Selector:tick(ctx)
    for i = self.current, #self.children do
        local status = self.children[i]:tick(ctx)
        if status == SUCCESS then
            self.current = 1
            return SUCCESS
        elseif status == RUNNING then
            self.current = i
            return RUNNING
        end
    end
    self.current = 1
    return FAILURE
end

-- Sequence (AND) - runs children in order, stops on failure
local Sequence = setmetatable({}, {__index = BTNode})
Sequence.__index = Sequence

function Sequence.new(...)
    return setmetatable({children = {...}, current = 1}, Sequence)
end

function Sequence:tick(ctx)
    for i = self.current, #self.children do
        local status = self.children[i]:tick(ctx)
        if status == FAILURE then
            self.current = 1
            return FAILURE
        elseif status == RUNNING then
            self.current = i
            return RUNNING
        end
    end
    self.current = 1
    return SUCCESS
end

-- Inverter
local Inverter = setmetatable({}, {__index = BTNode})
Inverter.__index = Inverter

function Inverter.new(child)
    return setmetatable({child = child}, Inverter)
end

function Inverter:tick(ctx)
    local status = self.child:tick(ctx)
    if status == SUCCESS then return FAILURE
    elseif status == FAILURE then return SUCCESS
    else return status end
end

-- Action node
local Action = setmetatable({}, {__index = BTNode})
Action.__index = Action

function Action.new(fn)
    return setmetatable({fn = fn}, Action)
end

function Action:tick(ctx)
    return self.fn(ctx)
end

-- Condition node
local Condition = setmetatable({}, {__index = BTNode})
Condition.__index = Condition

function Condition.new(fn)
    return setmetatable({fn = fn}, Condition)
end

function Condition:tick(ctx)
    return self.fn(ctx) and SUCCESS or FAILURE
end

-- ตัวอย่าง: Guard AI
local function makeGuardBT()
    return Selector.new(
        -- Combat branch
        Sequence.new(
            Condition.new(function(ctx)
                return ctx.enemy.canSeePlayer
            end),
            Selector.new(
                -- Attack if close
                Sequence.new(
                    Condition.new(function(ctx)
                        local dx = ctx.enemy.x - ctx.playerX
                        local dy = ctx.enemy.y - ctx.playerY
                        return math.sqrt(dx*dx+dy*dy) < 60
                    end),
                    Action.new(function(ctx)
                        ctx.enemy.attacking = true
                        ctx.enemy.lastAttack = love.timer.getTime()
                        return SUCCESS
                    end)
                ),
                -- Chase if far
                Action.new(function(ctx)
                    ctx.enemy.attacking = false
                    local dx = ctx.playerX - ctx.enemy.x
                    local dy = ctx.playerY - ctx.enemy.y
                    local dist = math.sqrt(dx*dx+dy*dy)
                    if dist > 0 then
                        ctx.enemy.x = ctx.enemy.x + dx/dist * ctx.enemy.speed * ctx.dt
                        ctx.enemy.y = ctx.enemy.y + dy/dist * ctx.enemy.speed * ctx.dt
                    end
                    return RUNNING
                end)
            )
        ),
        -- Patrol if no player
        Action.new(function(ctx)
            ctx.enemy.attacking = false
            ctx.enemy.patrolTimer = (ctx.enemy.patrolTimer or 0) + ctx.dt
            if ctx.enemy.patrolTimer > 2 then
                ctx.enemy.patrolTimer = 0
                ctx.enemy.patrolDir = -(ctx.enemy.patrolDir or 1)
            end
            ctx.enemy.x = ctx.enemy.x + (ctx.enemy.patrolDir or 1) * 50 * ctx.dt
            return RUNNING
        end)
    )
end

function love.load()
    btEnemy = {
        x = 200, y = 300,
        speed = 120,
        canSeePlayer = false,
        attacking = false,
        patrolDir = 1
    }
    btEnemy.bt = makeGuardBT()
    
    playerX_bt, playerY_bt = 400, 300
    font = love.graphics.newFont(14)
end

function love.update(dt)
    playerX_bt = love.mouse.getX()
    playerY_bt = love.mouse.getY()
    
    -- Check visibility (simplified - line of sight would be more complex)
    local dx = playerX_bt - btEnemy.x
    local dy = playerY_bt - btEnemy.y
    local dist = math.sqrt(dx*dx+dy*dy)
    btEnemy.canSeePlayer = dist < 250
    
    -- Tick behavior tree
    local ctx = {
        enemy = btEnemy,
        playerX = playerX_bt,
        playerY = playerY_bt,
        dt = dt
    }
    btEnemy.bt:tick(ctx)
end

function love.draw()
    love.graphics.setColor(0.05, 0.05, 0.1, 1)
    love.graphics.rectangle("fill", 0, 0, 800, 600)
    
    -- Detection range
    love.graphics.setColor(0.3, 0.3, 0, 0.2)
    love.graphics.circle("fill", btEnemy.x, btEnemy.y, 250)
    
    -- Enemy
    local ec = btEnemy.attacking and {1, 0, 0} or (btEnemy.canSeePlayer and {1, 0.5, 0} or {0.5, 0.5, 1})
    love.graphics.setColor(ec[1], ec[2], ec[3], 1)
    love.graphics.rectangle("fill", btEnemy.x - 15, btEnemy.y - 20, 30, 40)
    
    -- Player
    love.graphics.setColor(0, 1, 0, 1)
    love.graphics.circle("fill", playerX_bt, playerY_bt, 12)
    
    love.graphics.setFont(font)
    love.graphics.setColor(1, 1, 1, 1)
    local stateText = btEnemy.attacking and "ATTACKING!" or (btEnemy.canSeePlayer and "CHASING" or "PATROLLING")
    love.graphics.print("Enemy state: " .. stateText, 10, 10)
    love.graphics.print("Move mouse to control player", 10, 30)
end
```

## 12. Asset Management

```lua
-- Example 12: Asset manager
local AssetManager = {}
AssetManager.__index = AssetManager

function AssetManager.new()
    return setmetatable({
        images = {},
        sounds = {},
        fonts = {},
        music = {},
        shaders = {},
        loading = false,
        queue = {},
        loaded = 0,
        total = 0
    }, AssetManager)
end

-- Load image (cached)
function AssetManager:loadImage(key, path, filter)
    if not self.images[key] then
        local ok, img = pcall(love.graphics.newImage, path)
        if ok then
            if filter then img:setFilter(filter, filter) end
            self.images[key] = img
        else
            print("Failed to load image: " .. path .. " - " .. tostring(img))
            -- Create fallback 1x1 white image
            local d = love.image.newImageData(1, 1)
            d:setPixel(0, 0, 1, 0, 1, 1)
            self.images[key] = love.graphics.newImage(d)
        end
    end
    return self.images[key]
end

function AssetManager:getImage(key)
    return self.images[key]
end

function AssetManager:loadSound(key, path)
    if not self.sounds[key] then
        local ok, snd = pcall(love.audio.newSource, path, "static")
        if ok then
            self.sounds[key] = snd
        else
            print("Failed to load sound: " .. path)
        end
    end
    return self.sounds[key]
end

function AssetManager:loadFont(key, path, size)
    local fontKey = key .. "_" .. (size or "default")
    if not self.fonts[fontKey] then
        local ok, fnt
        if path then
            ok, fnt = pcall(love.graphics.newFont, path, size or 16)
        else
            ok, fnt = true, love.graphics.newFont(size or 16)
        end
        if ok then
            self.fonts[fontKey] = fnt
        end
    end
    return self.fonts[fontKey]
end

function AssetManager:loadMusic(key, path)
    if not self.music[key] then
        local ok, mus = pcall(love.audio.newSource, path, "stream")
        if ok then
            mus:setLooping(true)
            self.music[key] = mus
        end
    end
    return self.music[key]
end

-- Async loading queue
function AssetManager:queueImage(key, path, filter)
    table.insert(self.queue, {type="image", key=key, path=path, filter=filter})
    self.total = self.total + 1
end

function AssetManager:queueSound(key, path)
    table.insert(self.queue, {type="sound", key=key, path=path})
    self.total = self.total + 1
end

function AssetManager:processQueue(countPerFrame)
    countPerFrame = countPerFrame or 2
    local processed = 0
    
    while #self.queue > 0 and processed < countPerFrame do
        local item = table.remove(self.queue, 1)
        
        if item.type == "image" then
            self:loadImage(item.key, item.path, item.filter)
        elseif item.type == "sound" then
            self:loadSound(item.key, item.path)
        end
        
        self.loaded = self.loaded + 1
        processed = processed + 1
    end
    
    return #self.queue == 0  -- return true when done
end

function AssetManager:getLoadProgress()
    if self.total == 0 then return 1 end
    return self.loaded / self.total
end

function AssetManager:unload(type, key)
    if type == "image" and self.images[key] then
        self.images[key]:release()
        self.images[key] = nil
    elseif type == "sound" and self.sounds[key] then
        self.sounds[key]:release()
        self.sounds[key] = nil
    end
end

-- Global asset manager
assets = AssetManager.new()

-- Loading screen example
function love.load()
    -- Queue assets to load
    assets:queueImage("player", "assets/player.png", "nearest")
    assets:queueImage("tileset", "assets/tileset.png", "nearest")
    assets:queueImage("ui", "assets/ui.png")
    
    loadingFont = love.graphics.newFont(20)
    loadingDone = false
end

function love.update(dt)
    if not loadingDone then
        loadingDone = assets:processQueue(3)
    end
end

function love.draw()
    if not loadingDone then
        -- Loading screen
        love.graphics.setColor(0, 0, 0, 1)
        love.graphics.rectangle("fill", 0, 0, 800, 600)
        
        local progress = assets:getLoadProgress()
        local barW = 500
        local barH = 30
        local barX = (800 - barW) / 2
        local barY = 280
        
        love.graphics.setColor(0.2, 0.2, 0.2, 1)
        love.graphics.rectangle("fill", barX, barY, barW, barH)
        love.graphics.setColor(0, 0.8, 0.4, 1)
        love.graphics.rectangle("fill", barX, barY, barW * progress, barH)
        love.graphics.setColor(1, 1, 1, 1)
        love.graphics.rectangle("line", barX, barY, barW, barH)
        
        love.graphics.setFont(loadingFont)
        love.graphics.printf(string.format("Loading... %d%%", math.floor(progress * 100)),
            0, barY + 40, 800, "center")
    else
        love.graphics.setColor(1, 1, 1, 1)
        love.graphics.printf("All assets loaded!", 0, 280, 800, "center")
    end
end
```

## 13. Event System ใน Games

```lua
-- Example 13: Event system (Observer pattern)
local EventSystem = {}
EventSystem.__index = EventSystem

function EventSystem.new()
    return setmetatable({
        listeners = {},
        queue = {},
        processing = false
    }, EventSystem)
end

-- ลงทะเบียน listener
function EventSystem:on(eventType, callback, owner)
    if not self.listeners[eventType] then
        self.listeners[eventType] = {}
    end
    local id = #self.listeners[eventType] + 1
    table.insert(self.listeners[eventType], {
        callback = callback,
        owner = owner,
        id = id,
        active = true
    })
    return id
end

-- ลบ listener
function EventSystem:off(eventType, id)
    if self.listeners[eventType] then
        for i, listener in ipairs(self.listeners[eventType]) do
            if listener.id == id then
                table.remove(self.listeners[eventType], i)
                return
            end
        end
    end
end

-- ลบ listeners ของ owner ทั้งหมด
function EventSystem:offOwner(owner)
    for _, listeners in pairs(self.listeners) do
        for i = #listeners, 1, -1 do
            if listeners[i].owner == owner then
                table.remove(listeners, i)
            end
        end
    end
end

-- Fire event ทันที
function EventSystem:emit(eventType, data)
    if not self.listeners[eventType] then return end
    
    for _, listener in ipairs(self.listeners[eventType]) do
        if listener.active then
            listener.callback(data)
        end
    end
end

-- Queue event (สำหรับ deferred processing)
function EventSystem:queue(eventType, data, delay)
    table.insert(self.queue, {
        type = eventType,
        data = data,
        delay = delay or 0,
        timer = 0
    })
end

function EventSystem:update(dt)
    for i = #self.queue, 1, -1 do
        local event = self.queue[i]
        event.timer = event.timer + dt
        
        if event.timer >= event.delay then
            self:emit(event.type, event.data)
            table.remove(self.queue, i)
        end
    end
end

-- One-time listener
function EventSystem:once(eventType, callback, owner)
    local id
    id = self:on(eventType, function(data)
        callback(data)
        self:off(eventType, id)
    end, owner)
    return id
end

-- ตัวอย่างการใช้งาน
function love.load()
    events = EventSystem.new()
    
    score = 0
    combo = 0
    messages = {}
    
    -- Listen for score events
    events:on("enemy_killed", function(data)
        combo = combo + 1
        score = score + data.points * combo
        table.insert(messages, {
            text = "+" .. data.points * combo .. " (x" .. combo .. " combo!)",
            x = data.x,
            y = data.y,
            life = 1.5
        })
    end)
    
    -- Listen for combo reset
    events:on("combo_reset", function(data)
        combo = 0
    end)
    
    -- Listen for power-up
    events:on("powerup_collected", function(data)
        if data.type == "double_points" then
            print("Double points active!")
        end
    end)
    
    -- One-time first kill bonus
    events:once("enemy_killed", function(data)
        table.insert(messages, {
            text = "FIRST KILL BONUS! +500",
            x = 400, y = 200, life = 3
        })
        score = score + 500
    end)
    
    comboTimer = 0
    font = love.graphics.newFont(16)
    smallFont = love.graphics.newFont(12)
end

function love.update(dt)
    events:update(dt)
    
    -- Combo timer
    if combo > 0 then
        comboTimer = comboTimer + dt
        if comboTimer > 2 then
            comboTimer = 0
            events:emit("combo_reset", {})
        end
    end
    
    -- Update floating messages
    for i = #messages, 1, -1 do
        local m = messages[i]
        m.y = m.y - 50 * dt
        m.life = m.life - dt
        if m.life <= 0 then
            table.remove(messages, i)
        end
    end
end

function love.mousepressed(x, y, button)
    if button == 1 then
        -- Simulate killing enemy
        comboTimer = 0
        events:emit("enemy_killed", {
            x = x, y = y,
            points = 100,
            enemyType = "basic"
        })
    elseif button == 2 then
        -- Simulate collecting powerup
        events:emit("powerup_collected", {
            type = "double_points",
            x = x, y = y
        })
    end
end

function love.draw()
    love.graphics.setColor(0.1, 0.1, 0.15, 1)
    love.graphics.rectangle("fill", 0, 0, 800, 600)
    
    love.graphics.setFont(font)
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Score: " .. math.floor(score), 10, 10)
    love.graphics.print("Combo: x" .. combo, 10, 35)
    
    if combo > 0 then
        -- Combo bar
        love.graphics.setColor(0.2, 0.2, 0.2, 0.8)
        love.graphics.rectangle("fill", 10, 60, 200, 10)
        love.graphics.setColor(1, 0.5, 0, 1)
        local comboProgress = 1 - (comboTimer / 2)
        love.graphics.rectangle("fill", 10, 60, 200 * comboProgress, 10)
    end
    
    -- Floating messages
    love.graphics.setFont(smallFont)
    for _, m in ipairs(messages) do
        local alpha = math.min(1, m.life)
        love.graphics.setColor(1, 1, 0, alpha)
        love.graphics.printf(m.text, m.x - 100, m.y, 200, "center")
    end
    
    love.graphics.setColor(0.6, 0.6, 0.6, 1)
    love.graphics.print("Left click = kill enemy | Right click = powerup", 10, 570)
end
```

## สรุป Game Engine Concepts

ใน Game Engine Development หลักการสำคัญที่เรียนในบทนี้:

1. **ECS (Entity Component System)** - แยก data (components) ออกจาก logic (systems) ทำให้ยืดหยุ่นและขยายได้ง่าย
2. **Transform Hierarchy** - parent-child relationships สำหรับ scene graph
3. **Render System** - draw call batching, layer sorting
4. **Physics System** - velocity, acceleration, collision resolution
5. **Input System** - action mapping, buffering
6. **Scene Management** - transitions, state stacking
7. **Object Pooling** - ลด GC pressure สำหรับ frequently created/destroyed objects
8. **Quadtree** - spatial partitioning เพื่อ optimize collision detection O(n²) → O(n log n)
9. **A* Pathfinding** - heuristic search algorithm สำหรับ navigation
10. **FSM (Finite State Machine)** - simple AI behavior
11. **Behavior Trees** - complex hierarchical AI decisions
12. **Asset Manager** - centralized loading, caching, async loading
13. **Event System** - decoupled communication ระหว่าง game objects
