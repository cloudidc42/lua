# บทที่ 61: LÖVE2D - 2D Game Development

## บทนำ

LÖVE2D (หรือเรียกว่า Love2D) คือ framework สำหรับพัฒนาเกม 2D โดยใช้ภาษา Lua เป็น open source และรองรับ Windows, macOS, Linux, Android, iOS LÖVE2D มี API ที่ง่ายต่อการใช้งาน เหมาะสำหรับผู้เริ่มต้นและนักพัฒนาเกม indie

## การติดตั้ง LÖVE2D

ดาวน์โหลดจาก https://love2d.org/ แล้วติดตั้งตามระบบปฏิบัติการของคุณ

โครงสร้างโปรเจค LÖVE2D พื้นฐาน:
```
mygame/
  main.lua      -- จุดเริ่มต้นของเกม
  conf.lua      -- การตั้งค่า (optional)
  assets/
    images/
    sounds/
    fonts/
```

รัน LÖVE2D:
```bash
love mygame/
# หรือ
love main.lua
```

## 1. Game Loop พื้นฐาน

LÖVE2D มี callback หลัก 3 ตัว:

```lua
-- Example 1: Game loop พื้นฐาน
function love.load()
    -- เรียกครั้งเดียวตอนเริ่มเกม
    -- โหลด assets, ตั้งค่าตัวแปร
    print("Game loaded!")
    playerX = 100
    playerY = 100
    speed = 200
end

function love.update(dt)
    -- เรียกทุก frame, dt = delta time (วินาที)
    -- อัปเดต game logic
    if love.keyboard.isDown("right") then
        playerX = playerX + speed * dt
    end
    if love.keyboard.isDown("left") then
        playerX = playerX - speed * dt
    end
end

function love.draw()
    -- เรียกทุก frame หลัง update
    -- วาดทุกอย่างที่จอ
    love.graphics.rectangle("fill", playerX, playerY, 50, 50)
end
```

## 2. conf.lua - การตั้งค่าเกม

```lua
-- Example 2: conf.lua
function love.conf(t)
    t.title = "My Awesome Game"
    t.version = "11.4"
    t.window.width = 1280
    t.window.height = 720
    t.window.resizable = false
    t.window.vsync = 1
    t.window.fullscreen = false
    t.window.icon = "assets/icon.png"
    
    -- เปิด/ปิด modules
    t.modules.audio = true
    t.modules.joystick = true
    t.modules.physics = true
    t.modules.sound = true
    t.modules.video = false  -- ปิดถ้าไม่ใช้
    
    -- สำหรับ mobile
    t.window.highdpi = true
end
```

## 3. การวาดรูปทรงพื้นฐาน (Drawing Shapes)

```lua
-- Example 3: Drawing shapes
function love.draw()
    -- สี: R, G, B, A (0-1)
    love.graphics.setColor(1, 0, 0, 1)  -- แดง
    love.graphics.rectangle("fill", 50, 50, 100, 80)
    
    love.graphics.setColor(0, 1, 0, 1)  -- เขียว
    love.graphics.rectangle("line", 200, 50, 100, 80)
    
    love.graphics.setColor(0, 0, 1, 1)  -- น้ำเงิน
    love.graphics.circle("fill", 400, 90, 50)
    
    love.graphics.setColor(1, 1, 0, 1)  -- เหลือง
    love.graphics.circle("line", 550, 90, 50)
    
    -- เส้น
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.line(50, 200, 600, 200)
    
    -- สามเหลี่ยม (polygon)
    love.graphics.setColor(1, 0.5, 0, 1)
    love.graphics.polygon("fill", 300, 250, 350, 350, 250, 350)
    
    -- Ellipse
    love.graphics.setColor(0.5, 0, 1, 1)
    love.graphics.ellipse("fill", 500, 300, 80, 40)
    
    -- Arc
    love.graphics.setColor(0, 1, 1, 1)
    love.graphics.arc("fill", 100, 350, 50, 0, math.pi)
    
    -- Point
    love.graphics.setPointSize(10)
    love.graphics.points(200, 400, 210, 400, 220, 400)
    
    -- reset สี
    love.graphics.setColor(1, 1, 1, 1)
end
```

## 4. การวาดข้อความ (Drawing Text)

```lua
-- Example 4: Text rendering
function love.load()
    -- โหลด font
    font = love.graphics.newFont(24)
    bigFont = love.graphics.newFont("assets/fonts/myfont.ttf", 36)
    
    -- สร้าง Text object (ประสิทธิภาพดีกว่า)
    myText = love.graphics.newText(font, "Hello, LÖVE2D!")
end

function love.draw()
    -- วาดข้อความพื้นฐาน
    love.graphics.setFont(font)
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Hello World!", 50, 50)
    
    -- วาดพร้อม rotation, scale
    love.graphics.print("Rotated!", 300, 100, math.pi/6, 1.5, 1.5)
    
    -- printf (จัดตำแหน่ง)
    love.graphics.printf("Centered Text", 0, 200, love.graphics.getWidth(), "center")
    love.graphics.printf("Right aligned", 0, 250, love.graphics.getWidth(), "right")
    
    -- วาด Text object
    love.graphics.draw(myText, 50, 300)
    
    -- แสดง FPS
    love.graphics.setColor(0, 1, 0, 1)
    love.graphics.print("FPS: " .. love.timer.getFPS(), 10, 10)
end
```

## 5. การโหลดและวาดรูปภาพ (Images)

```lua
-- Example 5: Image loading and drawing
function love.load()
    -- โหลด image
    playerImg = love.graphics.newImage("assets/player.png")
    backgroundImg = love.graphics.newImage("assets/background.png")
    
    -- ตั้งค่า filter
    playerImg:setFilter("nearest", "nearest")  -- pixel art style
    
    -- สร้าง Quad สำหรับ sprite sheet
    spriteSheet = love.graphics.newImage("assets/sprites.png")
    -- quad(x, y, width, height, imageWidth, imageHeight)
    walkFrame1 = love.graphics.newQuad(0, 0, 32, 32, spriteSheet:getDimensions())
    walkFrame2 = love.graphics.newQuad(32, 0, 32, 32, spriteSheet:getDimensions())
    
    imgWidth = playerImg:getWidth()
    imgHeight = playerImg:getHeight()
end

function love.draw()
    -- วาด background
    love.graphics.draw(backgroundImg, 0, 0)
    
    -- วาด image พื้นฐาน
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(playerImg, 100, 100)
    
    -- วาดพร้อม transformation
    -- draw(image, x, y, rotation, scaleX, scaleY, originX, originY)
    love.graphics.draw(playerImg, 300, 100, 0, 2, 2, imgWidth/2, imgHeight/2)
    
    -- วาดแบบ flip (scale = -1)
    love.graphics.draw(playerImg, 500, 100, 0, -1, 1, imgWidth/2, imgHeight/2)
    
    -- วาดด้วย Quad (sprite sheet)
    love.graphics.draw(spriteSheet, walkFrame1, 100, 200)
    love.graphics.draw(spriteSheet, walkFrame2, 150, 200)
end
```

## 6. Sprite Sheets และ Animation

```lua
-- Example 6: Sprite animation system
local Animation = {}
Animation.__index = Animation

function Animation.new(image, frameWidth, frameHeight, fps)
    local self = setmetatable({}, Animation)
    self.image = image
    self.frameWidth = frameWidth
    self.frameHeight = frameHeight
    self.fps = fps or 12
    self.currentFrame = 1
    self.timer = 0
    self.frames = {}
    self.playing = true
    self.looping = true
    
    -- สร้าง quads อัตโนมัติ
    local imgW, imgH = image:getDimensions()
    local cols = math.floor(imgW / frameWidth)
    local rows = math.floor(imgH / frameHeight)
    
    for row = 0, rows - 1 do
        for col = 0, cols - 1 do
            table.insert(self.frames, love.graphics.newQuad(
                col * frameWidth,
                row * frameHeight,
                frameWidth,
                frameHeight,
                imgW, imgH
            ))
        end
    end
    
    self.totalFrames = #self.frames
    return self
end

function Animation:update(dt)
    if not self.playing then return end
    
    self.timer = self.timer + dt
    if self.timer >= 1 / self.fps then
        self.timer = self.timer - 1 / self.fps
        self.currentFrame = self.currentFrame + 1
        
        if self.currentFrame > self.totalFrames then
            if self.looping then
                self.currentFrame = 1
            else
                self.currentFrame = self.totalFrames
                self.playing = false
            end
        end
    end
end

function Animation:draw(x, y, rotation, scaleX, scaleY)
    local ox = self.frameWidth / 2
    local oy = self.frameHeight / 2
    love.graphics.draw(
        self.image,
        self.frames[self.currentFrame],
        x, y,
        rotation or 0,
        scaleX or 1,
        scaleY or 1,
        ox, oy
    )
end

function Animation:setFrame(frame)
    self.currentFrame = math.max(1, math.min(frame, self.totalFrames))
end

function Animation:reset()
    self.currentFrame = 1
    self.timer = 0
    self.playing = true
end

-- การใช้งาน
function love.load()
    local sheet = love.graphics.newImage("assets/player_sheet.png")
    walkAnim = Animation.new(sheet, 32, 32, 8)
    runAnim = Animation.new(sheet, 32, 32, 12)
    
    playerX, playerY = 400, 300
    currentAnim = walkAnim
end

function love.update(dt)
    currentAnim:update(dt)
    
    if love.keyboard.isDown("shift") then
        currentAnim = runAnim
    else
        currentAnim = walkAnim
    end
end

function love.draw()
    currentAnim:draw(playerX, playerY)
end
```

## 7. Input Handling - Keyboard

```lua
-- Example 7: Keyboard input
local keys = {}

function love.load()
    playerX = 400
    playerY = 300
    speed = 300
    jumpPower = -500
    velocityY = 0
    onGround = false
    groundY = 550
end

function love.update(dt)
    -- Continuous input (isDown)
    if love.keyboard.isDown("left", "a") then
        playerX = playerX - speed * dt
    end
    if love.keyboard.isDown("right", "d") then
        playerX = playerX + speed * dt
    end
    
    -- Physics
    velocityY = velocityY + 980 * dt  -- gravity
    playerY = playerY + velocityY * dt
    
    if playerY >= groundY then
        playerY = groundY
        velocityY = 0
        onGround = true
    else
        onGround = false
    end
    
    -- clamp position
    playerX = math.max(0, math.min(playerX, love.graphics.getWidth() - 50))
end

-- Event-based input (keypressed/keyreleased)
function love.keypressed(key, scancode, isrepeat)
    if key == "space" and onGround then
        velocityY = jumpPower
    end
    
    if key == "escape" then
        love.event.quit()
    end
    
    if key == "f11" then
        local fullscreen = love.window.getFullscreen()
        love.window.setFullscreen(not fullscreen)
    end
    
    keys[key] = true
end

function love.keyreleased(key)
    keys[key] = false
end

function love.draw()
    love.graphics.setColor(0.2, 0.6, 1, 1)
    love.graphics.rectangle("fill", playerX - 25, playerY - 50, 50, 50)
    
    -- แสดง key state
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Space=Jump, Arrow=Move", 10, 10)
    
    -- Ground
    love.graphics.setColor(0.4, 0.8, 0.2, 1)
    love.graphics.rectangle("fill", 0, groundY, love.graphics.getWidth(), 50)
end
```

## 8. Input Handling - Mouse

```lua
-- Example 8: Mouse input
function love.load()
    mouseX = 0
    mouseY = 0
    clicks = {}
    drawing = false
    points = {}
    
    love.mouse.setVisible(true)
    love.mouse.setCursor(love.mouse.getSystemCursor("hand"))
end

function love.update(dt)
    mouseX = love.mouse.getX()
    mouseY = love.mouse.getY()
    
    -- ตรวจสอบปุ่มขณะกด
    if love.mouse.isDown(1) then  -- left click
        if drawing then
            table.insert(points, mouseX)
            table.insert(points, mouseY)
        end
    end
end

function love.mousepressed(x, y, button, istouch, presses)
    if button == 1 then  -- left
        drawing = true
        points = {x, y}
    elseif button == 2 then  -- right
        points = {}
        drawing = false
    elseif button == 3 then  -- middle
        print("Middle click at " .. x .. ", " .. y)
    end
end

function love.mousereleased(x, y, button)
    if button == 1 then
        drawing = false
        table.insert(clicks, {x = x, y = y, time = love.timer.getTime()})
    end
end

function love.mousemoved(x, y, dx, dy, istouch)
    -- dx, dy คือ delta (การเปลี่ยนแปลงจาก frame ก่อน)
end

function love.wheelmoved(x, y)
    -- y > 0 = scroll up, y < 0 = scroll down
    print("Wheel: " .. y)
end

function love.draw()
    -- วาดเส้น
    if #points >= 4 then
        love.graphics.setColor(1, 1, 0, 1)
        love.graphics.line(points)
    end
    
    -- แสดงตำแหน่ง mouse
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print(string.format("Mouse: %d, %d", mouseX, mouseY), 10, 10)
    love.graphics.print("Left click: draw | Right click: clear", 10, 30)
    
    -- custom cursor
    love.graphics.setColor(1, 0, 0, 1)
    love.graphics.circle("line", mouseX, mouseY, 10)
    love.graphics.line(mouseX - 15, mouseY, mouseX + 15, mouseY)
    love.graphics.line(mouseX, mouseY - 15, mouseX, mouseY + 15)
end
```

## 9. Input Handling - Gamepad

```lua
-- Example 9: Gamepad/Joystick input
function love.load()
    joysticks = love.joystick.getJoysticks()
    gamepad = joysticks[1]  -- controller แรก
    
    playerX = 400
    playerY = 300
    speed = 300
    
    deadzone = 0.2  -- ป้องกัน stick drift
end

function love.update(dt)
    if gamepad and gamepad:isConnected() then
        -- Analog stick
        local axisX = gamepad:getGamepadAxis("leftx")
        local axisY = gamepad:getGamepadAxis("lefty")
        
        -- Apply deadzone
        if math.abs(axisX) < deadzone then axisX = 0 end
        if math.abs(axisY) < deadzone then axisY = 0 end
        
        playerX = playerX + axisX * speed * dt
        playerY = playerY + axisY * speed * dt
        
        -- Button check
        if gamepad:isGamepadDown("a") then
            -- Jump หรือ action
        end
        
        -- Trigger (0-1)
        local rightTrigger = gamepad:getGamepadAxis("triggerleft")
        if rightTrigger > 0.5 then
            -- Shoot หรือ action
        end
    end
end

function love.gamepadpressed(joystick, button)
    print("Button pressed: " .. button)
    if button == "start" then
        -- Pause game
    end
    if button == "b" then
        -- Back/cancel
    end
end

function love.gamepadreleased(joystick, button)
    print("Button released: " .. button)
end

function love.joystickadded(joystick)
    print("Controller connected: " .. joystick:getName())
    gamepad = joystick
end

function love.joystickremoved(joystick)
    print("Controller disconnected")
    gamepad = nil
end

function love.draw()
    love.graphics.setColor(0, 0.7, 1, 1)
    love.graphics.circle("fill", playerX, playerY, 25)
    
    if gamepad then
        love.graphics.setColor(1, 1, 1, 1)
        love.graphics.print("Gamepad: " .. gamepad:getName(), 10, 10)
    else
        love.graphics.setColor(1, 0, 0, 1)
        love.graphics.print("No gamepad connected", 10, 10)
    end
end
```

## 10. เสียง (Sound)

```lua
-- Example 10: Sound system
function love.load()
    -- โหลด sound
    -- "static" = โหลดทั้งหมดเข้า RAM (สำหรับ sound effects สั้น)
    jumpSound = love.audio.newSource("assets/sounds/jump.wav", "static")
    hitSound = love.audio.newSource("assets/sounds/hit.wav", "static")
    
    -- "stream" = stream จาก disk (สำหรับเพลงยาว)
    bgMusic = love.audio.newSource("assets/sounds/music.mp3", "stream")
    bgMusic:setLooping(true)
    bgMusic:setVolume(0.7)
    
    -- เล่นเพลง
    love.audio.play(bgMusic)
    
    -- ตั้งค่า master volume
    love.audio.setVolume(1.0)
    
    musicPlaying = true
    jumpCooldown = 0
end

function love.update(dt)
    jumpCooldown = math.max(0, jumpCooldown - dt)
end

function love.keypressed(key)
    if key == "space" and jumpCooldown <= 0 then
        -- Clone source เพื่อเล่นหลาย instance พร้อมกัน
        local s = jumpSound:clone()
        s:setPitch(0.9 + math.random() * 0.2)  -- random pitch
        love.audio.play(s)
        jumpCooldown = 0.1
    end
    
    if key == "m" then
        if musicPlaying then
            bgMusic:pause()
        else
            bgMusic:play()
        end
        musicPlaying = not musicPlaying
    end
    
    if key == "up" then
        local vol = math.min(1, love.audio.getVolume() + 0.1)
        love.audio.setVolume(vol)
    end
    
    if key == "down" then
        local vol = math.max(0, love.audio.getVolume() - 0.1)
        love.audio.setVolume(vol)
    end
end

function love.draw()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Space=Jump Sound, M=Toggle Music", 10, 10)
    love.graphics.print("Up/Down=Volume: " .. string.format("%.1f", love.audio.getVolume()), 10, 30)
    love.graphics.print("Music: " .. (musicPlaying and "Playing" or "Paused"), 10, 50)
end
```

## 11. Physics ด้วย love.physics (Box2D)

```lua
-- Example 11: Physics world
function love.load()
    -- สร้าง physics world (gravity x, y)
    world = love.physics.newWorld(0, 800, true)
    
    -- Ground
    groundBody = love.physics.newBody(world, 400, 580, "static")
    groundShape = love.physics.newRectangleShape(800, 20)
    groundFixture = love.physics.newFixture(groundBody, groundShape)
    groundFixture:setFriction(0.5)
    
    -- Player box
    playerBody = love.physics.newBody(world, 400, 300, "dynamic")
    playerBody:setLinearDamping(0.5)
    playerShape = love.physics.newRectangleShape(40, 40)
    playerFixture = love.physics.newFixture(playerBody, playerShape)
    playerFixture:setDensity(1)
    playerFixture:setFriction(0.3)
    playerFixture:setRestitution(0.2)  -- bounce
    playerBody:resetMassData()
    
    -- Circle
    ballBody = love.physics.newBody(world, 200, 100, "dynamic")
    ballShape = love.physics.newCircleShape(25)
    ballFixture = love.physics.newFixture(ballBody, ballShape)
    ballFixture:setDensity(0.5)
    ballFixture:setRestitution(0.8)
    ballBody:resetMassData()
    
    objects = {}
    
    -- Collision callbacks
    world:setCallbacks(beginContact, endContact, preSolve, postSolve)
end

function beginContact(a, b, coll)
    -- a, b คือ fixtures ที่ชน
    print("Contact begin!")
end

function endContact(a, b, coll)
    print("Contact end!")
end

function preSolve(a, b, coll)
    -- ก่อนคำนวณ collision
end

function postSolve(a, b, coll, normalImpulse1, tangentImpulse1)
    -- หลังคำนวณ collision
end

function love.update(dt)
    world:update(dt)
    
    -- Player control
    if love.keyboard.isDown("left") then
        playerBody:applyForce(-5000, 0)
    end
    if love.keyboard.isDown("right") then
        playerBody:applyForce(5000, 0)
    end
    
    -- Spawn objects
    if love.keyboard.isDown("space") then
        spawnTimer = (spawnTimer or 0) + dt
        if spawnTimer > 0.3 then
            spawnTimer = 0
            local body = love.physics.newBody(world, love.mouse.getX(), love.mouse.getY(), "dynamic")
            local shape = love.physics.newCircleShape(10 + math.random(20))
            local fix = love.physics.newFixture(body, shape)
            fix:setDensity(1)
            fix:setRestitution(0.5)
            body:resetMassData()
            table.insert(objects, {body = body, shape = shape})
        end
    end
end

function love.keypressed(key)
    if key == "up" then
        local vx, vy = playerBody:getLinearVelocity()
        if math.abs(vy) < 1 then  -- on ground check (simplified)
            playerBody:setLinearVelocity(vx, -600)
        end
    end
end

function love.draw()
    -- Ground
    love.graphics.setColor(0.4, 0.8, 0.2, 1)
    love.graphics.polygon("fill", groundBody:getWorldPoints(groundShape:getPoints()))
    
    -- Player
    love.graphics.setColor(0.2, 0.4, 1, 1)
    love.graphics.polygon("fill", playerBody:getWorldPoints(playerShape:getPoints()))
    
    -- Ball
    love.graphics.setColor(1, 0.3, 0.3, 1)
    local bx, by = ballBody:getPosition()
    love.graphics.circle("fill", bx, by, 25)
    
    -- Objects
    love.graphics.setColor(1, 0.8, 0, 1)
    for _, obj in ipairs(objects) do
        local x, y = obj.body:getPosition()
        local r = obj.shape:getRadius()
        love.graphics.circle("fill", x, y, r)
    end
    
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Arrows=Move, Up=Jump, Hold Space=Spawn balls", 10, 10)
end
```

## 12. Collision Detection แบบ Manual (AABB)

```lua
-- Example 12: Manual collision detection
local function AABB(a, b)
    return a.x < b.x + b.w and
           a.x + a.w > b.x and
           a.y < b.y + b.h and
           a.y + a.h > b.y
end

local function circleAABB(circle, rect)
    local nearX = math.max(rect.x, math.min(circle.x, rect.x + rect.w))
    local nearY = math.max(rect.y, math.min(circle.y, rect.y + rect.h))
    local dx = circle.x - nearX
    local dy = circle.y - nearY
    return (dx * dx + dy * dy) < (circle.r * circle.r)
end

local function circleCircle(a, b)
    local dx = a.x - b.x
    local dy = a.y - b.y
    local dist = math.sqrt(dx * dx + dy * dy)
    return dist < a.r + b.r
end

function love.load()
    player = {x = 100, y = 300, w = 40, h = 40, speed = 200}
    
    walls = {
        {x = 300, y = 200, w = 20, h = 200},
        {x = 500, y = 100, w = 200, h = 20},
        {x = 0, y = 550, w = 800, h = 20},
    }
    
    ball = {x = 600, y = 300, r = 30}
    colliding = false
end

function love.update(dt)
    local prevX = player.x
    local prevY = player.y
    
    if love.keyboard.isDown("left") then player.x = player.x - player.speed * dt end
    if love.keyboard.isDown("right") then player.x = player.x + player.speed * dt end
    if love.keyboard.isDown("up") then player.y = player.y - player.speed * dt end
    if love.keyboard.isDown("down") then player.y = player.y + player.speed * dt end
    
    -- ตรวจสอบ collision กับ walls
    for _, wall in ipairs(walls) do
        if AABB(player, wall) then
            -- resolve collision แบบ simple
            player.x = prevX
            player.y = prevY
        end
    end
    
    -- ตรวจสอบ circle collision
    colliding = circleAABB(ball, player)
end

function love.draw()
    -- Player
    love.graphics.setColor(colliding and 1 or 0, colliding and 0 or 0.5, 1, 1)
    love.graphics.rectangle("fill", player.x, player.y, player.w, player.h)
    
    -- Walls
    love.graphics.setColor(0.5, 0.5, 0.5, 1)
    for _, wall in ipairs(walls) do
        love.graphics.rectangle("fill", wall.x, wall.y, wall.w, wall.h)
    end
    
    -- Ball
    love.graphics.setColor(1, colliding and 0 or 1, 0, 1)
    love.graphics.circle("fill", ball.x, ball.y, ball.r)
    
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("WASD/Arrows to move", 10, 10)
    if colliding then
        love.graphics.setColor(1, 0, 0, 1)
        love.graphics.print("COLLISION!", 350, 10)
    end
end
```

## 13. Particle Systems

```lua
-- Example 13: Particle systems
function love.load()
    -- สร้าง particle image
    local canvas = love.graphics.newCanvas(8, 8)
    love.graphics.setCanvas(canvas)
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.circle("fill", 4, 4, 4)
    love.graphics.setCanvas()
    
    -- Fire particles
    firePS = love.graphics.newParticleSystem(canvas, 500)
    firePS:setEmissionRate(100)
    firePS:setLifetime(0.5, 1.5)
    firePS:setParticleLifetime(0.3, 0.8)
    firePS:setLinearAcceleration(-30, -200, 30, -400)
    firePS:setSizeVariation(0.5)
    firePS:setSizes(1.5, 0.5)
    firePS:setColors(
        1, 0.5, 0, 1,    -- orange (birth)
        1, 0.2, 0, 0.8,  -- red
        0.5, 0.1, 0, 0   -- dark (death)
    )
    firePS:setSpread(math.pi / 6)
    firePS:setSpeed(50, 100)
    firePS:setRotation(0, math.pi * 2)
    firePS:setDirection(-math.pi / 2)  -- upward
    
    -- Explosion particles
    explosionPS = love.graphics.newParticleSystem(canvas, 300)
    explosionPS:setEmissionRate(0)  -- burst only
    explosionPS:setParticleLifetime(0.5, 1.0)
    explosionPS:setLinearAcceleration(0, 100, 0, 200)  -- gravity
    explosionPS:setSpeed(200, 400)
    explosionPS:setSizes(2, 0)
    explosionPS:setColors(1, 1, 0, 1, 1, 0, 0, 0)
    explosionPS:setSpread(math.pi * 2)  -- all directions
    explosionPS:setPosition(400, 300)
    
    fireX = 400
    fireY = 500
end

function love.update(dt)
    firePS:setPosition(fireX, fireY)
    firePS:update(dt)
    explosionPS:update(dt)
    
    fireX = love.mouse.getX()
    fireY = love.mouse.getY()
end

function love.keypressed(key)
    if key == "space" then
        -- Burst explosion
        explosionPS:setPosition(love.mouse.getX(), love.mouse.getY())
        explosionPS:emit(100)
    end
end

function love.draw()
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(firePS)
    love.graphics.draw(explosionPS)
    
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Move mouse=fire, Space=explosion", 10, 10)
end
```

## 14. Tilemaps

```lua
-- Example 14: Tilemap system
local TILE_SIZE = 32

-- Map data (0=empty, 1=ground, 2=wall, 3=water)
local mapData = {
    {1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1},
    {1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,0,0,0,2,2,2,0,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,0,0,0,3,3,3,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,0,0,0,3,3,3,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,2,0,0,0,0,0,0,0,2,2,2,0,0,1},
    {1,0,0,0,0,0,2,0,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1},
}

local tileColors = {
    [0] = {0.1, 0.1, 0.15},  -- empty (sky)
    [1] = {0.4, 0.8, 0.2},   -- ground (grass)
    [2] = {0.5, 0.5, 0.5},   -- wall (stone)
    [3] = {0.1, 0.4, 0.9},   -- water
}

local TileMap = {}
TileMap.__index = TileMap

function TileMap.new(data, tileSize)
    local self = setmetatable({}, TileMap)
    self.data = data
    self.tileSize = tileSize
    self.rows = #data
    self.cols = #data[1]
    return self
end

function TileMap:getTile(row, col)
    if row < 1 or row > self.rows or col < 1 or col > self.cols then
        return 1  -- border = solid
    end
    return self.data[row][col]
end

function TileMap:isSolid(row, col)
    local tile = self:getTile(row, col)
    return tile == 1 or tile == 2
end

function TileMap:worldToTile(x, y)
    local col = math.floor(x / self.tileSize) + 1
    local row = math.floor(y / self.tileSize) + 1
    return row, col
end

function TileMap:draw(offsetX, offsetY)
    offsetX = offsetX or 0
    offsetY = offsetY or 0
    
    for row = 1, self.rows do
        for col = 1, self.cols do
            local tile = self.data[row][col]
            local color = tileColors[tile] or {0.5, 0.5, 0.5}
            love.graphics.setColor(color[1], color[2], color[3], 1)
            love.graphics.rectangle("fill",
                (col - 1) * self.tileSize + offsetX,
                (row - 1) * self.tileSize + offsetY,
                self.tileSize - 1,
                self.tileSize - 1
            )
        end
    end
end

function love.load()
    tilemap = TileMap.new(mapData, TILE_SIZE)
    
    player = {
        x = 2 * TILE_SIZE,
        y = 1 * TILE_SIZE,
        w = TILE_SIZE - 4,
        h = TILE_SIZE - 4,
        vx = 0, vy = 0,
        speed = 200,
        jumpPower = -400,
        onGround = false
    }
end

function love.update(dt)
    local gravity = 800
    
    if love.keyboard.isDown("left") then
        player.vx = -player.speed
    elseif love.keyboard.isDown("right") then
        player.vx = player.speed
    else
        player.vx = player.vx * 0.8
    end
    
    player.vy = player.vy + gravity * dt
    
    -- X movement
    player.x = player.x + player.vx * dt
    local row1, col1 = tilemap:worldToTile(player.x, player.y + 1)
    local row2, col2 = tilemap:worldToTile(player.x + player.w, player.y + 1)
    local row3, col3 = tilemap:worldToTile(player.x, player.y + player.h - 1)
    local row4, col4 = tilemap:worldToTile(player.x + player.w, player.y + player.h - 1)
    
    if player.vx > 0 and (tilemap:isSolid(row2, col2) or tilemap:isSolid(row4, col4)) then
        player.x = (col2 - 1) * TILE_SIZE - player.w
        player.vx = 0
    elseif player.vx < 0 and (tilemap:isSolid(row1, col1) or tilemap:isSolid(row3, col3)) then
        player.x = col1 * TILE_SIZE
        player.vx = 0
    end
    
    -- Y movement
    player.y = player.y + player.vy * dt
    player.onGround = false
    
    local rB1, cB1 = tilemap:worldToTile(player.x + 2, player.y + player.h)
    local rB2, cB2 = tilemap:worldToTile(player.x + player.w - 2, player.y + player.h)
    local rT1, cT1 = tilemap:worldToTile(player.x + 2, player.y)
    local rT2, cT2 = tilemap:worldToTile(player.x + player.w - 2, player.y)
    
    if player.vy > 0 and (tilemap:isSolid(rB1, cB1) or tilemap:isSolid(rB2, cB2)) then
        player.y = (rB1 - 1) * TILE_SIZE - player.h
        player.vy = 0
        player.onGround = true
    elseif player.vy < 0 and (tilemap:isSolid(rT1, cT1) or tilemap:isSolid(rT2, cT2)) then
        player.y = rT1 * TILE_SIZE
        player.vy = 0
    end
end

function love.keypressed(key)
    if key == "space" and player.onGround then
        player.vy = player.jumpPower
    end
end

function love.draw()
    tilemap:draw()
    
    love.graphics.setColor(1, 0.5, 0, 1)
    love.graphics.rectangle("fill", player.x, player.y, player.w, player.h)
    
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Arrow/WASD + Space to move and jump", 10, 10)
end
```

## 15. Camera System

```lua
-- Example 15: Camera system
local Camera = {}
Camera.__index = Camera

function Camera.new()
    local self = setmetatable({}, Camera)
    self.x = 0
    self.y = 0
    self.scaleX = 1
    self.scaleY = 1
    self.rotation = 0
    self.shakeX = 0
    self.shakeY = 0
    self.shakeDuration = 0
    self.shakeIntensity = 0
    return self
end

function Camera:attach()
    love.graphics.push()
    love.graphics.translate(
        love.graphics.getWidth() / 2 + self.shakeX,
        love.graphics.getHeight() / 2 + self.shakeY
    )
    love.graphics.rotate(self.rotation)
    love.graphics.scale(self.scaleX, self.scaleY)
    love.graphics.translate(-self.x, -self.y)
end

function Camera:detach()
    love.graphics.pop()
end

function Camera:follow(x, y, lerp)
    lerp = lerp or 0.1
    self.x = self.x + (x - self.x) * lerp
    self.y = self.y + (y - self.y) * lerp
end

function Camera:shake(duration, intensity)
    self.shakeDuration = duration
    self.shakeIntensity = intensity
end

function Camera:update(dt)
    if self.shakeDuration > 0 then
        self.shakeDuration = self.shakeDuration - dt
        local intensity = self.shakeIntensity * (self.shakeDuration / 0.5)
        self.shakeX = (math.random() * 2 - 1) * intensity
        self.shakeY = (math.random() * 2 - 1) * intensity
    else
        self.shakeX = 0
        self.shakeY = 0
    end
end

function Camera:worldToScreen(x, y)
    local sx = (x - self.x) * self.scaleX + love.graphics.getWidth() / 2
    local sy = (y - self.y) * self.scaleY + love.graphics.getHeight() / 2
    return sx, sy
end

function Camera:screenToWorld(x, y)
    local wx = (x - love.graphics.getWidth() / 2) / self.scaleX + self.x
    local wy = (y - love.graphics.getHeight() / 2) / self.scaleY + self.y
    return wx, wy
end

-- การใช้งาน
function love.load()
    camera = Camera.new()
    
    player = {x = 0, y = 0, speed = 300}
    
    -- สร้าง world objects
    objects = {}
    for i = 1, 50 do
        table.insert(objects, {
            x = math.random(-2000, 2000),
            y = math.random(-1000, 1000),
            w = math.random(20, 80),
            h = math.random(20, 80),
            color = {math.random(), math.random(), math.random()}
        })
    end
end

function love.update(dt)
    if love.keyboard.isDown("left") then player.x = player.x - player.speed * dt end
    if love.keyboard.isDown("right") then player.x = player.x + player.speed * dt end
    if love.keyboard.isDown("up") then player.y = player.y - player.speed * dt end
    if love.keyboard.isDown("down") then player.y = player.y + player.speed * dt end
    
    camera:follow(player.x, player.y, 0.08)
    camera:update(dt)
    
    -- zoom
    if love.keyboard.isDown("q") then camera.scaleX = camera.scaleX - dt; camera.scaleY = camera.scaleY - dt end
    if love.keyboard.isDown("e") then camera.scaleX = camera.scaleX + dt; camera.scaleY = camera.scaleY + dt end
    camera.scaleX = math.max(0.2, math.min(3, camera.scaleX))
    camera.scaleY = camera.scaleX
end

function love.keypressed(key)
    if key == "space" then
        camera:shake(0.3, 15)
    end
end

function love.draw()
    camera:attach()
    
    -- Draw world
    love.graphics.setColor(0.1, 0.1, 0.15, 1)
    love.graphics.rectangle("fill", -5000, -3000, 10000, 6000)
    
    for _, obj in ipairs(objects) do
        love.graphics.setColor(obj.color[1], obj.color[2], obj.color[3], 1)
        love.graphics.rectangle("fill", obj.x, obj.y, obj.w, obj.h)
    end
    
    -- Player
    love.graphics.setColor(1, 0.8, 0, 1)
    love.graphics.rectangle("fill", player.x - 15, player.y - 20, 30, 40)
    
    camera:detach()
    
    -- HUD (ไม่ได้รับผล camera)
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Arrows=Move, Q/E=Zoom, Space=Shake", 10, 10)
    love.graphics.print(string.format("Pos: %.0f, %.0f | Zoom: %.1f", player.x, player.y, camera.scaleX), 10, 30)
end
```

## 16. Game States

```lua
-- Example 16: Game state management
local GameState = {}
GameState.__index = GameState

function GameState.new()
    local self = setmetatable({}, GameState)
    self.states = {}
    self.current = nil
    self.stack = {}
    return self
end

function GameState:add(name, state)
    self.states[name] = state
end

function GameState:switch(name)
    if self.current and self.states[self.current] and self.states[self.current].exit then
        self.states[self.current]:exit()
    end
    self.current = name
    if self.states[name] and self.states[name].enter then
        self.states[name]:enter()
    end
end

function GameState:push(name)
    if self.current then
        table.insert(self.stack, self.current)
        if self.states[self.current] and self.states[self.current].pause then
            self.states[self.current]:pause()
        end
    end
    self.current = name
    if self.states[name] and self.states[name].enter then
        self.states[name]:enter()
    end
end

function GameState:pop()
    if #self.stack > 0 then
        if self.states[self.current] and self.states[self.current].exit then
            self.states[self.current]:exit()
        end
        self.current = table.remove(self.stack)
        if self.states[self.current] and self.states[self.current].resume then
            self.states[self.current]:resume()
        end
    end
end

function GameState:update(dt)
    if self.current and self.states[self.current] and self.states[self.current].update then
        self.states[self.current]:update(dt)
    end
end

function GameState:draw()
    if self.current and self.states[self.current] and self.states[self.current].draw then
        self.states[self.current]:draw()
    end
end

function GameState:keypressed(key)
    if self.current and self.states[self.current] and self.states[self.current].keypressed then
        self.states[self.current]:keypressed(key)
    end
end

-- States
local MenuState = {
    enter = function(self)
        print("Entering Menu")
    end,
    update = function(self, dt) end,
    draw = function(self)
        love.graphics.setColor(0, 0, 0.5, 1)
        love.graphics.rectangle("fill", 0, 0, love.graphics.getWidth(), love.graphics.getHeight())
        love.graphics.setColor(1, 1, 1, 1)
        love.graphics.printf("MAIN MENU", 0, 200, love.graphics.getWidth(), "center")
        love.graphics.printf("Press ENTER to play\nPress ESC to quit", 0, 300, love.graphics.getWidth(), "center")
    end,
    keypressed = function(self, key)
        if key == "return" then
            gsm:switch("game")
        elseif key == "escape" then
            love.event.quit()
        end
    end
}

local PlayState = {
    enter = function(self)
        print("Entering Game")
        self.score = 0
        self.player = {x = 400, y = 300}
    end,
    update = function(self, dt)
        self.score = self.score + dt * 10
        if love.keyboard.isDown("left") then self.player.x = self.player.x - 200 * dt end
        if love.keyboard.isDown("right") then self.player.x = self.player.x + 200 * dt end
        if love.keyboard.isDown("up") then self.player.y = self.player.y - 200 * dt end
        if love.keyboard.isDown("down") then self.player.y = self.player.y + 200 * dt end
    end,
    draw = function(self)
        love.graphics.setColor(0.1, 0.1, 0.2, 1)
        love.graphics.rectangle("fill", 0, 0, love.graphics.getWidth(), love.graphics.getHeight())
        love.graphics.setColor(0, 1, 0, 1)
        love.graphics.rectangle("fill", self.player.x - 20, self.player.y - 20, 40, 40)
        love.graphics.setColor(1, 1, 1, 1)
        love.graphics.print("Score: " .. math.floor(self.score), 10, 10)
        love.graphics.print("ESC=Pause", 10, 30)
    end,
    keypressed = function(self, key)
        if key == "escape" then
            gsm:push("pause")
        end
    end
}

local PauseState = {
    enter = function(self) print("Paused") end,
    draw = function(self)
        -- วาด game state ที่อยู่ด้านล่างก่อน
        gsm.states["game"]:draw()
        
        -- Overlay
        love.graphics.setColor(0, 0, 0, 0.7)
        love.graphics.rectangle("fill", 0, 0, love.graphics.getWidth(), love.graphics.getHeight())
        love.graphics.setColor(1, 1, 1, 1)
        love.graphics.printf("PAUSED\nPress ESC to resume\nPress M for menu", 0, 250, love.graphics.getWidth(), "center")
    end,
    keypressed = function(self, key)
        if key == "escape" then
            gsm:pop()
        elseif key == "m" then
            gsm:switch("menu")
        end
    end
}

function love.load()
    gsm = GameState.new()
    gsm:add("menu", MenuState)
    gsm:add("game", PlayState)
    gsm:add("pause", PauseState)
    gsm:switch("menu")
end

function love.update(dt) gsm:update(dt) end
function love.draw() gsm:draw() end
function love.keypressed(key) gsm:keypressed(key) end
```

## 17. Save/Load Game

```lua
-- Example 17: Save and load system
local SaveSystem = {}

function SaveSystem.save(data, filename)
    filename = filename or "save.dat"
    
    -- serialize data to string
    local serialized = SaveSystem.serialize(data)
    
    -- love.filesystem เขียนไฟล์ใน save directory
    local success, err = love.filesystem.write(filename, serialized)
    if not success then
        print("Failed to save: " .. tostring(err))
    end
    return success
end

function SaveSystem.load(filename)
    filename = filename or "save.dat"
    
    if not love.filesystem.getInfo(filename) then
        return nil, "File not found"
    end
    
    local content, err = love.filesystem.read(filename)
    if not content then
        return nil, err
    end
    
    return SaveSystem.deserialize(content)
end

function SaveSystem.serialize(data, indent)
    indent = indent or 0
    local t = type(data)
    
    if t == "number" then
        return tostring(data)
    elseif t == "string" then
        return string.format("%q", data)
    elseif t == "boolean" then
        return tostring(data)
    elseif t == "table" then
        local result = "{\n"
        for k, v in pairs(data) do
            result = result .. string.rep("  ", indent + 1)
            if type(k) == "string" then
                result = result .. "[" .. string.format("%q", k) .. "]"
            else
                result = result .. "[" .. tostring(k) .. "]"
            end
            result = result .. " = " .. SaveSystem.serialize(v, indent + 1) .. ",\n"
        end
        result = result .. string.rep("  ", indent) .. "}"
        return result
    else
        return "nil"
    end
end

function SaveSystem.deserialize(str)
    local fn, err = load("return " .. str)
    if fn then
        return fn()
    else
        return nil, err
    end
end

-- การใช้งาน
function love.load()
    gameData = {
        playerName = "Hero",
        level = 1,
        score = 0,
        position = {x = 100, y = 100},
        inventory = {"sword", "shield", "potion"},
        settings = {
            musicVolume = 0.8,
            sfxVolume = 1.0,
            fullscreen = false
        }
    }
    
    -- พยายามโหลด save
    local loaded = SaveSystem.load("savegame.dat")
    if loaded then
        gameData = loaded
        print("Game loaded!")
    else
        print("No save found, starting new game")
    end
    
    font = love.graphics.newFont(16)
end

function love.update(dt)
    gameData.score = gameData.score + dt
end

function love.keypressed(key)
    if key == "s" then
        gameData.timestamp = os.time()
        if SaveSystem.save(gameData, "savegame.dat") then
            saveMessage = "Game Saved! " .. os.date("%H:%M:%S")
        end
    end
    if key == "l" then
        local loaded, err = SaveSystem.load("savegame.dat")
        if loaded then
            gameData = loaded
            saveMessage = "Game Loaded!"
        else
            saveMessage = "Load failed: " .. tostring(err)
        end
    end
end

function love.draw()
    love.graphics.setFont(font)
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Player: " .. gameData.playerName, 50, 50)
    love.graphics.print("Level: " .. gameData.level, 50, 75)
    love.graphics.print("Score: " .. math.floor(gameData.score), 50, 100)
    
    if gameData.inventory then
        love.graphics.print("Inventory:", 50, 130)
        for i, item in ipairs(gameData.inventory) do
            love.graphics.print("  - " .. item, 50, 130 + i * 20)
        end
    end
    
    love.graphics.print("Press S to save, L to load", 50, 300)
    
    if saveMessage then
        love.graphics.setColor(0, 1, 0, 1)
        love.graphics.print(saveMessage, 50, 330)
    end
end
```

## 18. Basic Platformer

```lua
-- Example 18: Complete platformer game
-- platformer.lua

local GRAVITY = 900
local PLAYER_SPEED = 250
local JUMP_POWER = -500
local TILE_SIZE = 32

-- Tile map
local map = {
    {1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1},
    {1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,0,0,1,1,1,0,0,0,0,0,0,0,0,1,1,1,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,0,0,0,1,1,1,1,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1},
    {1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,1,1,1,0,1},
    {1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1},
}

local function isSolid(row, col)
    if row < 1 or row > #map or col < 1 or col > #map[1] then return true end
    return map[row][col] == 1
end

local function tileAt(x, y)
    local col = math.floor(x / TILE_SIZE) + 1
    local row = math.floor(y / TILE_SIZE) + 1
    return row, col
end

-- Enemy class
local Enemy = {}
Enemy.__index = Enemy

function Enemy.new(x, y)
    return setmetatable({
        x = x, y = y,
        w = 28, h = 28,
        vx = 60, vy = 0,
        alive = true
    }, Enemy)
end

function Enemy:update(dt)
    self.vy = self.vy + GRAVITY * dt
    self.x = self.x + self.vx * dt
    self.y = self.y + self.vy * dt
    
    -- Turn around at walls
    local r1, c1 = tileAt(self.x, self.y + self.h)
    local r2, c2 = tileAt(self.x + self.w, self.y + self.h)
    
    if self.vy > 0 and (isSolid(r1, c1) or isSolid(r2, c2)) then
        self.y = (math.max(r1, r2) - 1) * TILE_SIZE - self.h
        self.vy = 0
    end
    
    if isSolid(tileAt(self.x + self.w, self.y + self.h/2)) and self.vx > 0 then
        self.vx = -self.vx
    end
    if isSolid(tileAt(self.x, self.y + self.h/2)) and self.vx < 0 then
        self.vx = -self.vx
    end
end

function Enemy:draw()
    if self.alive then
        love.graphics.setColor(1, 0, 0, 1)
        love.graphics.rectangle("fill", self.x, self.y, self.w, self.h)
        -- Eyes
        love.graphics.setColor(1, 1, 1, 1)
        love.graphics.circle("fill", self.x + 8, self.y + 8, 4)
        love.graphics.circle("fill", self.x + 20, self.y + 8, 4)
        love.graphics.setColor(0, 0, 0, 1)
        love.graphics.circle("fill", self.x + (self.vx > 0 and 9 or 7), self.y + 8, 2)
        love.graphics.circle("fill", self.x + (self.vx > 0 and 21 or 19), self.y + 8, 2)
    end
end

-- Coin class
local Coin = {}
Coin.__index = Coin

function Coin.new(x, y)
    return setmetatable({x = x, y = y, r = 8, collected = false, timer = 0}, Coin)
end

function Coin:update(dt)
    self.timer = self.timer + dt
end

function Coin:draw()
    if not self.collected then
        local bob = math.sin(self.timer * 3) * 3
        love.graphics.setColor(1, 0.9, 0, 1)
        love.graphics.circle("fill", self.x, self.y + bob, self.r)
        love.graphics.setColor(0.8, 0.7, 0, 1)
        love.graphics.circle("line", self.x, self.y + bob, self.r)
    end
end

function love.load()
    -- Player
    player = {
        x = TILE_SIZE * 2,
        y = TILE_SIZE * 7,
        w = 28, h = 36,
        vx = 0, vy = 0,
        onGround = false,
        facing = 1,
        lives = 3,
        score = 0,
        invincible = 0,
        jumpBuffer = 0,
        coyoteTime = 0
    }
    
    -- Enemies
    enemies = {
        Enemy.new(TILE_SIZE * 8, TILE_SIZE * 8),
        Enemy.new(TILE_SIZE * 15, TILE_SIZE * 8),
        Enemy.new(TILE_SIZE * 18, TILE_SIZE * 7),
    }
    
    -- Coins
    coins = {
        Coin.new(TILE_SIZE * 3 + 16, TILE_SIZE * 7),
        Coin.new(TILE_SIZE * 5 + 16, TILE_SIZE * 7),
        Coin.new(TILE_SIZE * 10 + 16, TILE_SIZE * 5),
        Coin.new(TILE_SIZE * 12 + 16, TILE_SIZE * 5),
        Coin.new(TILE_SIZE * 20 + 16, TILE_SIZE * 8),
    }
    
    camera = {x = 0, y = 0}
    gameOver = false
    font = love.graphics.newFont(16)
end

function love.update(dt)
    if gameOver then return end
    
    -- Update timers
    player.invincible = math.max(0, player.invincible - dt)
    player.jumpBuffer = math.max(0, player.jumpBuffer - dt)
    
    -- Player movement
    local moveX = 0
    if love.keyboard.isDown("left", "a") then moveX = -1; player.facing = -1 end
    if love.keyboard.isDown("right", "d") then moveX = 1; player.facing = 1 end
    
    player.vx = moveX * PLAYER_SPEED
    player.vy = player.vy + GRAVITY * dt
    
    -- Coyote time
    if player.onGround then
        player.coyoteTime = 0.1
    else
        player.coyoteTime = math.max(0, player.coyoteTime - dt)
    end
    
    -- Jump
    if player.jumpBuffer > 0 and player.coyoteTime > 0 then
        player.vy = JUMP_POWER
        player.jumpBuffer = 0
        player.coyoteTime = 0
    end
    
    -- X movement and collision
    player.x = player.x + player.vx * dt
    player.onGround = false
    
    local checkPoints = {
        {player.x, player.y + 1},
        {player.x + player.w, player.y + 1},
        {player.x, player.y + player.h - 1},
        {player.x + player.w, player.y + player.h - 1}
    }
    
    if player.vx > 0 then
        if isSolid(tileAt(player.x + player.w, player.y + 2)) or
           isSolid(tileAt(player.x + player.w, player.y + player.h - 2)) then
            local col = math.floor((player.x + player.w) / TILE_SIZE)
            player.x = col * TILE_SIZE - player.w
            player.vx = 0
        end
    elseif player.vx < 0 then
        if isSolid(tileAt(player.x, player.y + 2)) or
           isSolid(tileAt(player.x, player.y + player.h - 2)) then
            local col = math.floor(player.x / TILE_SIZE) + 1
            player.x = col * TILE_SIZE
            player.vx = 0
        end
    end
    
    -- Y movement and collision
    player.y = player.y + player.vy * dt
    
    if player.vy > 0 then
        if isSolid(tileAt(player.x + 2, player.y + player.h)) or
           isSolid(tileAt(player.x + player.w - 2, player.y + player.h)) then
            local row = math.floor((player.y + player.h) / TILE_SIZE)
            player.y = row * TILE_SIZE - player.h
            player.vy = 0
            player.onGround = true
        end
    elseif player.vy < 0 then
        if isSolid(tileAt(player.x + 2, player.y)) or
           isSolid(tileAt(player.x + player.w - 2, player.y)) then
            local row = math.floor(player.y / TILE_SIZE) + 1
            player.y = row * TILE_SIZE
            player.vy = 0
        end
    end
    
    -- Update enemies
    for _, enemy in ipairs(enemies) do
        enemy:update(dt)
        
        -- Player vs Enemy collision
        if enemy.alive and player.invincible <= 0 then
            if player.x < enemy.x + enemy.w and player.x + player.w > enemy.x and
               player.y < enemy.y + enemy.h and player.y + player.h > enemy.y then
                -- Stomping
                if player.vy > 0 and player.y + player.h < enemy.y + enemy.h / 2 + 10 then
                    enemy.alive = false
                    player.vy = -300
                    player.score = player.score + 100
                else
                    -- Take damage
                    player.lives = player.lives - 1
                    player.invincible = 2.0
                    player.vx = player.facing * -200
                    player.vy = -300
                    if player.lives <= 0 then
                        gameOver = true
                    end
                end
            end
        end
    end
    
    -- Coin collection
    for _, coin in ipairs(coins) do
        coin:update(dt)
        if not coin.collected then
            local dx = player.x + player.w/2 - coin.x
            local dy = player.y + player.h/2 - coin.y
            if math.sqrt(dx*dx + dy*dy) < player.w/2 + coin.r then
                coin.collected = true
                player.score = player.score + 10
            end
        end
    end
    
    -- Fall off screen
    if player.y > #map * TILE_SIZE then
        player.lives = player.lives - 1
        player.x = TILE_SIZE * 2
        player.y = TILE_SIZE * 7
        player.vy = 0
        if player.lives <= 0 then gameOver = true end
    end
    
    -- Camera follow
    local targetX = player.x - love.graphics.getWidth() / 2 + player.w / 2
    camera.x = math.max(0, math.min(targetX, #map[1] * TILE_SIZE - love.graphics.getWidth()))
end

function love.keypressed(key)
    if key == "space" or key == "up" or key == "w" then
        player.jumpBuffer = 0.15  -- buffer for responsiveness
    end
    if key == "r" and gameOver then
        love.load()
    end
    if key == "escape" then love.event.quit() end
end

function love.draw()
    love.graphics.push()
    love.graphics.translate(-camera.x, 0)
    
    -- Background
    love.graphics.setColor(0.4, 0.6, 1, 1)
    love.graphics.rectangle("fill", camera.x, 0, love.graphics.getWidth(), love.graphics.getHeight())
    
    -- Tiles
    for row = 1, #map do
        for col = 1, #map[row] do
            if map[row][col] == 1 then
                local tx = (col - 1) * TILE_SIZE
                local ty = (row - 1) * TILE_SIZE
                
                -- Only draw visible tiles
                if tx + TILE_SIZE > camera.x and tx < camera.x + love.graphics.getWidth() then
                    love.graphics.setColor(0.4, 0.3, 0.2, 1)
                    love.graphics.rectangle("fill", tx, ty, TILE_SIZE, TILE_SIZE)
                    love.graphics.setColor(0.5, 0.4, 0.3, 1)
                    love.graphics.rectangle("line", tx, ty, TILE_SIZE, TILE_SIZE)
                end
            end
        end
    end
    
    -- Coins
    for _, coin in ipairs(coins) do
        coin:draw()
    end
    
    -- Enemies
    for _, enemy in ipairs(enemies) do
        enemy:draw()
    end
    
    -- Player
    local flash = player.invincible > 0 and math.floor(player.invincible * 10) % 2 == 0
    if not flash then
        love.graphics.setColor(0.2, 0.5, 1, 1)
        love.graphics.rectangle("fill", player.x, player.y, player.w, player.h)
        -- Eyes
        love.graphics.setColor(1, 1, 1, 1)
        if player.facing == 1 then
            love.graphics.circle("fill", player.x + 20, player.y + 8, 4)
            love.graphics.setColor(0, 0, 0, 1)
            love.graphics.circle("fill", player.x + 21, player.y + 8, 2)
        else
            love.graphics.circle("fill", player.x + 8, player.y + 8, 4)
            love.graphics.setColor(0, 0, 0, 1)
            love.graphics.circle("fill", player.x + 7, player.y + 8, 2)
        end
        -- Feet (show when on ground)
        if player.onGround then
            love.graphics.setColor(0.1, 0.3, 0.8, 1)
            love.graphics.rectangle("fill", player.x, player.y + player.h - 8, 12, 8)
            love.graphics.rectangle("fill", player.x + player.w - 12, player.y + player.h - 8, 12, 8)
        end
    end
    
    love.graphics.pop()
    
    -- HUD
    love.graphics.setFont(font)
    love.graphics.setColor(0, 0, 0, 0.5)
    love.graphics.rectangle("fill", 0, 0, love.graphics.getWidth(), 40)
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Score: " .. player.score, 10, 10)
    
    -- Lives
    for i = 1, player.lives do
        love.graphics.setColor(1, 0.3, 0.3, 1)
        love.graphics.circle("fill", 200 + i * 25, 20, 8)
    end
    
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Lives:", 160, 10)
    
    if gameOver then
        love.graphics.setColor(0, 0, 0, 0.7)
        love.graphics.rectangle("fill", 0, 0, love.graphics.getWidth(), love.graphics.getHeight())
        love.graphics.setColor(1, 0, 0, 1)
        love.graphics.printf("GAME OVER\nScore: " .. player.score .. "\nPress R to restart", 0, 250, love.graphics.getWidth(), "center")
    end
end
```

## 19. Basic Space Shooter

```lua
-- Example 19: Space shooter game
function love.load()
    love.window.setTitle("Space Shooter")
    
    W = love.graphics.getWidth()
    H = love.graphics.getHeight()
    
    player = {
        x = W / 2,
        y = H - 100,
        w = 30, h = 40,
        speed = 350,
        shootCooldown = 0,
        shootRate = 0.15,
        lives = 3,
        score = 0,
        invincible = 0
    }
    
    bullets = {}
    enemyBullets = {}
    enemies = {}
    particles = {}
    stars = {}
    
    -- Create stars background
    for i = 1, 100 do
        table.insert(stars, {
            x = math.random(W),
            y = math.random(H),
            speed = math.random(1, 4) * 30,
            size = math.random(1, 3)
        })
    end
    
    -- Spawn enemies
    spawnTimer = 0
    spawnRate = 1.5
    gameOver = false
    wave = 1
    
    font = love.graphics.newFont(16)
end

local function spawnExplosion(x, y, color)
    for i = 1, 15 do
        local angle = math.random() * math.pi * 2
        local speed = math.random(50, 200)
        table.insert(particles, {
            x = x, y = y,
            vx = math.cos(angle) * speed,
            vy = math.sin(angle) * speed,
            life = 0.5 + math.random() * 0.5,
            maxLife = 1,
            color = color or {1, 0.5, 0, 1},
            size = math.random(2, 5)
        })
    end
end

function love.update(dt)
    if gameOver then return end
    
    -- Update stars
    for _, star in ipairs(stars) do
        star.y = star.y + star.speed * dt
        if star.y > H then star.y = 0; star.x = math.random(W) end
    end
    
    -- Player movement
    if love.keyboard.isDown("left", "a") then
        player.x = math.max(player.w/2, player.x - player.speed * dt)
    end
    if love.keyboard.isDown("right", "d") then
        player.x = math.min(W - player.w/2, player.x + player.speed * dt)
    end
    if love.keyboard.isDown("up", "w") then
        player.y = math.max(player.h/2, player.y - player.speed * dt)
    end
    if love.keyboard.isDown("down", "s") then
        player.y = math.min(H - player.h/2, player.y + player.speed * dt)
    end
    
    -- Shooting
    player.shootCooldown = player.shootCooldown - dt
    if love.keyboard.isDown("space") and player.shootCooldown <= 0 then
        player.shootCooldown = player.shootRate
        table.insert(bullets, {
            x = player.x, y = player.y - player.h/2,
            w = 4, h = 12,
            vy = -600, damage = 1
        })
    end
    
    player.invincible = math.max(0, player.invincible - dt)
    
    -- Spawn enemies
    spawnTimer = spawnTimer + dt
    if spawnTimer >= spawnRate then
        spawnTimer = 0
        spawnRate = math.max(0.5, spawnRate - 0.02)
        
        local eType = math.random(3)
        if eType == 1 then
            -- Normal enemy
            table.insert(enemies, {
                x = math.random(20, W - 20),
                y = -30,
                w = 30, h = 30,
                vy = 100 + wave * 20,
                hp = 1,
                color = {1, 0, 0.5, 1},
                type = "normal",
                shootTimer = math.random() * 2
            })
        elseif eType == 2 then
            -- Zigzag enemy
            table.insert(enemies, {
                x = math.random(20, W - 20),
                y = -30,
                w = 25, h = 25,
                vx = 150, vy = 80,
                hp = 1,
                color = {1, 0.5, 0, 1},
                type = "zigzag",
                shootTimer = 3
            })
        else
            -- Tank enemy
            table.insert(enemies, {
                x = math.random(50, W - 50),
                y = -40,
                w = 40, h = 40,
                vy = 60,
                hp = 3,
                color = {0.5, 0, 1, 1},
                type = "tank",
                shootTimer = 1
            })
        end
    end
    
    -- Update bullets
    for i = #bullets, 1, -1 do
        local b = bullets[i]
        b.y = b.y + b.vy * dt
        if b.y < -20 then table.remove(bullets, i) end
    end
    
    -- Update enemy bullets
    for i = #enemyBullets, 1, -1 do
        local b = enemyBullets[i]
        b.x = b.x + b.vx * dt
        b.y = b.y + b.vy * dt
        if b.y > H + 20 or b.x < -20 or b.x > W + 20 then
            table.remove(enemyBullets, i)
        end
    end
    
    -- Update enemies
    for i = #enemies, 1, -1 do
        local e = enemies[i]
        e.y = e.y + e.vy * dt
        
        if e.type == "zigzag" then
            e.x = e.x + e.vx * dt
            if e.x < 20 or e.x > W - 20 then e.vx = -e.vx end
        end
        
        -- Enemy shooting
        e.shootTimer = e.shootTimer - dt
        if e.shootTimer <= 0 then
            e.shootTimer = 1.5 + math.random()
            local dx = player.x - e.x
            local dy = player.y - e.y
            local len = math.sqrt(dx*dx + dy*dy)
            table.insert(enemyBullets, {
                x = e.x, y = e.y,
                vx = dx/len * 200,
                vy = dy/len * 200,
                w = 5, h = 5
            })
        end
        
        -- Remove if off screen
        if e.y > H + 50 then
            table.remove(enemies, i)
        end
    end
    
    -- Bullet vs Enemy collision
    for bi = #bullets, 1, -1 do
        local b = bullets[bi]
        local hit = false
        for ei = #enemies, 1, -1 do
            local e = enemies[ei]
            if b.x > e.x - e.w/2 and b.x < e.x + e.w/2 and
               b.y > e.y - e.h/2 and b.y < e.y + e.h/2 then
                e.hp = e.hp - 1
                hit = true
                if e.hp <= 0 then
                    spawnExplosion(e.x, e.y, e.color)
                    player.score = player.score + (e.type == "tank" and 30 or 10)
                    table.remove(enemies, ei)
                end
                break
            end
        end
        if hit then table.remove(bullets, bi) end
    end
    
    -- Enemy bullets vs Player
    if player.invincible <= 0 then
        for i = #enemyBullets, 1, -1 do
            local b = enemyBullets[i]
            if b.x > player.x - player.w/2 and b.x < player.x + player.w/2 and
               b.y > player.y - player.h/2 and b.y < player.y + player.h/2 then
                player.lives = player.lives - 1
                player.invincible = 2.0
                spawnExplosion(player.x, player.y, {0, 0.5, 1, 1})
                table.remove(enemyBullets, i)
                if player.lives <= 0 then gameOver = true end
                break
            end
        end
    end
    
    -- Enemies vs Player
    if player.invincible <= 0 then
        for i = #enemies, 1, -1 do
            local e = enemies[i]
            if math.abs(e.x - player.x) < (e.w + player.w)/2 and
               math.abs(e.y - player.y) < (e.h + player.h)/2 then
                player.lives = player.lives - 1
                player.invincible = 2.0
                spawnExplosion(e.x, e.y, e.color)
                table.remove(enemies, i)
                if player.lives <= 0 then gameOver = true end
            end
        end
    end
    
    -- Update particles
    for i = #particles, 1, -1 do
        local p = particles[i]
        p.x = p.x + p.vx * dt
        p.y = p.y + p.vy * dt
        p.vy = p.vy + 100 * dt
        p.life = p.life - dt
        if p.life <= 0 then table.remove(particles, i) end
    end
    
    -- Wave progression
    if player.score >= wave * 500 then
        wave = wave + 1
    end
end

function love.keypressed(key)
    if key == "r" and gameOver then love.load() end
    if key == "escape" then love.event.quit() end
end

function love.draw()
    -- Background
    love.graphics.setColor(0.02, 0.02, 0.1, 1)
    love.graphics.rectangle("fill", 0, 0, W, H)
    
    -- Stars
    love.graphics.setColor(1, 1, 1, 0.8)
    for _, star in ipairs(stars) do
        love.graphics.circle("fill", star.x, star.y, star.size)
    end
    
    -- Bullets
    love.graphics.setColor(0, 1, 1, 1)
    for _, b in ipairs(bullets) do
        love.graphics.rectangle("fill", b.x - b.w/2, b.y - b.h/2, b.w, b.h)
    end
    
    -- Enemy bullets
    love.graphics.setColor(1, 0, 0, 1)
    for _, b in ipairs(enemyBullets) do
        love.graphics.circle("fill", b.x, b.y, 4)
    end
    
    -- Enemies
    for _, e in ipairs(enemies) do
        love.graphics.setColor(e.color[1], e.color[2], e.color[3], e.color[4])
        love.graphics.polygon("fill",
            e.x, e.y - e.h/2,
            e.x - e.w/2, e.y + e.h/2,
            e.x, e.y + e.h/4,
            e.x + e.w/2, e.y + e.h/2
        )
        -- HP bar for tank
        if e.type == "tank" then
            love.graphics.setColor(0.2, 0.2, 0.2, 0.8)
            love.graphics.rectangle("fill", e.x - 20, e.y - e.h/2 - 8, 40, 4)
            love.graphics.setColor(0, 1, 0, 1)
            love.graphics.rectangle("fill", e.x - 20, e.y - e.h/2 - 8, 40 * (e.hp/3), 4)
        end
    end
    
    -- Player (flash when invincible)
    local flash = player.invincible > 0 and math.floor(player.invincible * 8) % 2 == 0
    if not flash and not gameOver then
        love.graphics.setColor(0, 0.7, 1, 1)
        love.graphics.polygon("fill",
            player.x, player.y - player.h/2,
            player.x - player.w/2, player.y + player.h/2,
            player.x, player.y + player.h/4,
            player.x + player.w/2, player.y + player.h/2
        )
        love.graphics.setColor(0, 1, 1, 0.7)
        love.graphics.polygon("line",
            player.x, player.y - player.h/2,
            player.x - player.w/2, player.y + player.h/2,
            player.x, player.y + player.h/4,
            player.x + player.w/2, player.y + player.h/2
        )
    end
    
    -- Particles
    for _, p in ipairs(particles) do
        local alpha = p.life / p.maxLife
        love.graphics.setColor(p.color[1], p.color[2], p.color[3], alpha)
        love.graphics.circle("fill", p.x, p.y, p.size * alpha)
    end
    
    -- HUD
    love.graphics.setFont(font)
    love.graphics.setColor(0, 0, 0, 0.5)
    love.graphics.rectangle("fill", 0, 0, W, 35)
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.print("Score: " .. player.score, 10, 8)
    love.graphics.print("Wave: " .. wave, 200, 8)
    love.graphics.print("WASD+Space", W - 120, 8)
    
    for i = 1, player.lives do
        love.graphics.setColor(0, 0.7, 1, 1)
        love.graphics.polygon("fill",
            350 + i * 25, 15,
            345 + i * 25, 30,
            350 + i * 25, 26,
            355 + i * 25, 30
        )
    end
    
    if gameOver then
        love.graphics.setColor(0, 0, 0, 0.7)
        love.graphics.rectangle("fill", 0, 0, W, H)
        love.graphics.setColor(1, 0, 0, 1)
        love.graphics.printf("GAME OVER\nScore: " .. player.score .. "\nWave: " .. wave .. "\nPress R to restart", 0, 250, W, "center")
    end
end
```

## 20. Performance Tips

```lua
-- Example 20: Performance optimization techniques

-- 1. SpriteBatch - วาด sprites หลายตัวพร้อมกัน (เร็วกว่า draw loop มาก)
function love.load()
    img = love.graphics.newImage("assets/particle.png")
    
    -- SpriteBatch รองรับสูงสุด N sprites
    batch = love.graphics.newSpriteBatch(img, 10000, "dynamic")
    
    -- ตัวอย่าง: particle system ด้วย SpriteBatch
    particles = {}
    for i = 1, 5000 do
        table.insert(particles, {
            x = math.random(0, 800),
            y = math.random(0, 600),
            vx = (math.random() - 0.5) * 100,
            vy = (math.random() - 0.5) * 100,
            id = 0  -- SpriteBatch ID
        })
    end
    
    -- 2. Canvas caching
    staticCanvas = love.graphics.newCanvas(800, 600)
    love.graphics.setCanvas(staticCanvas)
    -- วาด static content ครั้งเดียว
    for i = 1, 100 do
        love.graphics.setColor(math.random(), math.random(), math.random(), 0.5)
        love.graphics.rectangle("fill", math.random(800), math.random(600), 20, 20)
    end
    love.graphics.setCanvas()
    
    -- 3. Object pool
    bulletPool = {}
    for i = 1, 200 do
        bulletPool[i] = {x = 0, y = 0, active = false}
    end
end

local function getBullet()
    for _, b in ipairs(bulletPool) do
        if not b.active then
            b.active = true
            return b
        end
    end
    return nil
end

function love.update(dt)
    -- Update SpriteBatch
    batch:clear()
    for _, p in ipairs(particles) do
        p.x = p.x + p.vx * dt
        p.y = p.y + p.vy * dt
        if p.x < 0 then p.x = 800 elseif p.x > 800 then p.x = 0 end
        if p.y < 0 then p.y = 600 elseif p.y > 600 then p.y = 0 end
        batch:add(p.x, p.y)
    end
end

function love.draw()
    -- วาด cached background
    love.graphics.setColor(1, 1, 1, 1)
    love.graphics.draw(staticCanvas, 0, 0)
    
    -- วาด SpriteBatch ครั้งเดียว (เร็วมาก)
    love.graphics.setColor(1, 1, 0.5, 0.8)
    love.graphics.draw(batch)
    
    -- Performance stats
    love.graphics.setColor(0, 0, 0, 0.7)
    love.graphics.rectangle("fill", 0, 0, 250, 80)
    love.graphics.setColor(0, 1, 0, 1)
    love.graphics.print("FPS: " .. love.timer.getFPS(), 10, 10)
    love.graphics.print("Particles: " .. #particles, 10, 30)
    love.graphics.print("Draw calls: minimal (SpriteBatch)", 10, 50)
end

-- 4. Mesa tips
-- ใช้ love.graphics.newMesh() สำหรับ custom geometry ที่ซับซ้อน
-- ใช้ love.graphics.setScissor() เพื่อ clip drawing area
-- หลีกเลี่ยงการเปลี่ยน setColor บ่อยๆ
-- ใช้ love.math.noise() แทน math.random() สำหรับ coherent noise
-- ใช้ local variables แทน global ใน hot paths

-- 5. Profile ด้วย love.timer
local function measureTime(name, fn)
    local t = love.timer.getTime()
    fn()
    print(name .. ": " .. (love.timer.getTime() - t) * 1000 .. "ms")
end
```

## สรุป LÖVE2D

LÖVE2D เป็น framework ที่ทรงพลังและใช้งานง่ายสำหรับการพัฒนาเกม 2D ด้วย Lua โดยมีฟีเจอร์หลักได้แก่:

1. **Game Loop** - love.load, love.update, love.draw
2. **Graphics** - shapes, images, text, canvas, shaders
3. **Input** - keyboard, mouse, gamepad
4. **Audio** - static และ stream sources
5. **Physics** - Box2D integration
6. **Math** - vectors, random, noise
7. **Filesystem** - save/load data
8. **Window** - fullscreen, resize, icons

ข้อดีของ LÖVE2D:
- เรียนรู้ง่าย, API ชัดเจน
- Cross-platform
- Open source ฟรี
- Community ใหญ่
- เหมาะสำหรับ game jams และ indie games

แหล่งเรียนรู้เพิ่มเติม:
- https://love2d.org/wiki/
- https://github.com/love2d-community/awesome-love2d
