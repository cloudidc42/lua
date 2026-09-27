# บทที่ 74: Machine Learning กับ Lua

## บทนำ

Lua มีบทบาทสำคัญในประวัติศาสตร์ของ Machine Learning โดยเฉพาะผ่าน Torch framework ที่พัฒนาขึ้นในช่วงปี 2000s และถูกใช้งานอย่างแพร่หลายในงานวิจัย deep learning ก่อนที่ PyTorch (Python port ของ Torch) จะเข้ามาแทนที่ ปัจจุบัน Lua ยังคงถูกใช้ในงาน ML บนระบบ embedded และงานที่ต้องการ performance สูง

## Torch/Lua สำหรับ ML (บริบทประวัติศาสตร์)

```lua
-- ประวัติ Torch Framework
local torch_history = {
    {year = 2002, event = "Torch เริ่มพัฒนาที่ IDIAP Research Institute"},
    {year = 2008, event = "Torch 3 - C++ backend, Lua scripting"},
    {year = 2011, event = "Torch 7 - ปรับปรุงสำคัญ, TH C library"},
    {year = 2015, event = "Facebook, Google, NYU ใช้ Torch อย่างแพร่หลาย"},
    {year = 2016, event = "PyTorch เริ่มพัฒนาบน Python"},
    {year = 2017, event = "PyTorch เปิดตัว - ค่อยๆ แทนที่ Torch/Lua"},
    {year = 2018, event = "Torch/Lua ML ลดความนิยมลง"},
    {year = 2024, event = "Lua ยังใช้ใน embedded ML, game AI, scripting"},
}

print("=== Torch/Lua ML History ===")
for _, event in ipairs(torch_history) do
    print(string.format("  %d: %s", event.year, event.event))
end

-- ทำไม Lua ดีสำหรับ ML
local advantages = {
    "Fast C extension integration",
    "Small memory footprint",
    "Easy embedding in C/C++ applications",
    "Clean syntax สำหรับ mathematical operations",
    "Dynamic typing สำหรับ prototyping",
}

print("\nLua advantages for ML:")
for i, adv in ipairs(advantages) do
    print(string.format("  %d. %s", i, adv))
end
```

## Tensors ใน Lua

```lua
-- Tensor คือโครงสร้างข้อมูลหลักใน ML
-- n-dimensional array of numbers

-- สร้าง Tensor class ใน Lua
local Tensor = {}
Tensor.__index = Tensor

function Tensor.new(data, shape)
    local self = setmetatable({}, Tensor)
    
    if type(data) == "table" then
        -- จาก nested table
        self.data = {}
        self.shape = shape or {}
        
        if type(data[1]) == "table" then
            -- 2D tensor
            self.shape = {#data, #data[1]}
            for i, row in ipairs(data) do
                for j, val in ipairs(row) do
                    table.insert(self.data, val)
                end
            end
        else
            -- 1D tensor
            self.shape = {#data}
            for _, v in ipairs(data) do
                table.insert(self.data, v)
            end
        end
    else
        -- สร้าง tensor ว่าง
        self.data = {}
        self.shape = shape or {0}
    end
    
    return self
end

-- สร้าง tensor เต็มไปด้วยค่าเดียวกัน
function Tensor.fill(shape, value)
    local self = setmetatable({}, Tensor)
    self.shape = shape
    self.data = {}
    
    local total = 1
    for _, dim in ipairs(shape) do
        total = total * dim
    end
    
    for i = 1, total do
        self.data[i] = value
    end
    
    return self
end

-- Tensor ที่เต็มไปด้วยศูนย์
function Tensor.zeros(shape)
    return Tensor.fill(shape, 0)
end

-- Tensor ที่เต็มไปด้วยหนึ่ง
function Tensor.ones(shape)
    return Tensor.fill(shape, 1)
end

-- Random tensor
function Tensor.randn(shape)
    local self = setmetatable({}, Tensor)
    self.shape = shape
    self.data = {}
    
    local total = 1
    for _, dim in ipairs(shape) do
        total = total * dim
    end
    
    -- Box-Muller transform สำหรับ normal distribution
    for i = 1, total, 2 do
        local u1 = math.random()
        local u2 = math.random()
        local z1 = math.sqrt(-2 * math.log(u1)) * math.cos(2 * math.pi * u2)
        local z2 = math.sqrt(-2 * math.log(u1)) * math.sin(2 * math.pi * u2)
        self.data[i] = z1
        if i + 1 <= total then
            self.data[i + 1] = z2
        end
    end
    
    return self
end

-- math.pi ไม่มีใน Lua โดยตรง
math.pi = math.pi or 3.14159265358979323846

-- Random tensor (uniform)
function Tensor.rand(shape, low, high)
    low = low or 0
    high = high or 1
    local self = setmetatable({}, Tensor)
    self.shape = shape
    self.data = {}
    
    local total = 1
    for _, dim in ipairs(shape) do total = total * dim end
    
    for i = 1, total do
        self.data[i] = low + math.random() * (high - low)
    end
    
    return self
end

-- Properties
function Tensor:ndim()
    return #self.shape
end

function Tensor:size()
    local total = 1
    for _, dim in ipairs(self.shape) do
        total = total * dim
    end
    return total
end

function Tensor:numel()
    return self:size()
end

-- Element access
function Tensor:get(...)
    local indices = {...}
    
    if #indices == 1 then
        -- 1D access
        return self.data[indices[1]]
    elseif #indices == 2 then
        -- 2D access: row, col
        local row, col = indices[1], indices[2]
        local cols = self.shape[2]
        return self.data[(row - 1) * cols + col]
    end
end

function Tensor:set(value, ...)
    local indices = {...}
    
    if #indices == 1 then
        self.data[indices[1]] = value
    elseif #indices == 2 then
        local row, col = indices[1], indices[2]
        local cols = self.shape[2]
        self.data[(row - 1) * cols + col] = value
    end
end

-- Display
function Tensor:__tostring()
    local rows = self.shape[1] or 1
    local cols = self.shape[2] or 1
    
    if self:ndim() == 1 then
        local parts = {}
        for _, v in ipairs(self.data) do
            table.insert(parts, string.format("%.4f", v))
        end
        return "tensor([" .. table.concat(parts, ", ") .. "])"
    else
        local lines = {"tensor(["}
        for i = 1, rows do
            local row_parts = {}
            for j = 1, cols do
                table.insert(row_parts, string.format("%8.4f", self:get(i, j)))
            end
            local prefix = i == 1 and "  [" or "  ["
            local suffix = i == rows and "]" or "],"
            table.insert(lines, prefix .. table.concat(row_parts, ", ") .. suffix)
        end
        table.insert(lines, "])")
        return table.concat(lines, "\n")
    end
end

setmetatable(Tensor, {__call = function(cls, ...) return cls.new(...) end})

-- ทดสอบ Tensor creation
math.randomseed(42)
print("=== Tensor Basics ===")

local t1 = Tensor({1, 2, 3, 4, 5})
print("1D tensor: " .. tostring(t1))
print("Shape: " .. t1.shape[1])

local t2 = Tensor({{1, 2, 3}, {4, 5, 6}, {7, 8, 9}})
print("\n2D tensor (3x3):")
print(tostring(t2))
print("Element [2,3] = " .. t2:get(2, 3))

local zeros = Tensor.zeros({3, 4})
print("\nZeros (3x4): shape=" .. zeros.shape[1] .. "x" .. zeros.shape[2])

local rand_t = Tensor.rand({2, 3})
print("\nRandom (2x3):")
print(tostring(rand_t))
```

## Linear Algebra Operations

```lua
-- การดำเนินการ Linear Algebra พื้นฐาน

-- Element-wise operations
function Tensor:add(other)
    assert(self:size() == other:size(), "Tensor sizes must match")
    local result = Tensor.zeros(self.shape)
    for i = 1, self:size() do
        result.data[i] = self.data[i] + other.data[i]
    end
    return result
end

function Tensor:sub(other)
    assert(self:size() == other:size(), "Tensor sizes must match")
    local result = Tensor.zeros(self.shape)
    for i = 1, self:size() do
        result.data[i] = self.data[i] - other.data[i]
    end
    return result
end

function Tensor:mul_scalar(scalar)
    local result = Tensor.zeros(self.shape)
    for i = 1, self:size() do
        result.data[i] = self.data[i] * scalar
    end
    return result
end

-- Element-wise multiply
function Tensor:mul(other)
    if type(other) == "number" then
        return self:mul_scalar(other)
    end
    assert(self:size() == other:size(), "Tensor sizes must match")
    local result = Tensor.zeros(self.shape)
    for i = 1, self:size() do
        result.data[i] = self.data[i] * other.data[i]
    end
    return result
end

-- Matrix multiply
function Tensor:mm(other)
    -- self: (m, k), other: (k, n) -> result: (m, n)
    assert(self:ndim() == 2 and other:ndim() == 2, "Both must be 2D tensors")
    local m = self.shape[1]
    local k = self.shape[2]
    local n = other.shape[2]
    assert(k == other.shape[1], 
        string.format("Dimension mismatch: (%d,%d) x (%d,%d)", 
            m, k, other.shape[1], n))
    
    local result = Tensor.zeros({m, n})
    for i = 1, m do
        for j = 1, n do
            local sum = 0
            for l = 1, k do
                sum = sum + self:get(i, l) * other:get(l, j)
            end
            result:set(sum, i, j)
        end
    end
    return result
end

-- Matrix transpose
function Tensor:t()
    assert(self:ndim() == 2, "Transpose requires 2D tensor")
    local rows, cols = self.shape[1], self.shape[2]
    local result = Tensor.zeros({cols, rows})
    for i = 1, rows do
        for j = 1, cols do
            result:set(self:get(i, j), j, i)
        end
    end
    return result
end

-- Reduction operations
function Tensor:sum()
    local total = 0
    for _, v in ipairs(self.data) do
        total = total + v
    end
    return total
end

function Tensor:mean()
    return self:sum() / self:size()
end

function Tensor:max()
    local max_val = -math.huge
    for _, v in ipairs(self.data) do
        if v > max_val then max_val = v end
    end
    return max_val
end

function Tensor:min()
    local min_val = math.huge
    for _, v in ipairs(self.data) do
        if v < min_val then min_val = v end
    end
    return min_val
end

function Tensor:std()
    local m = self:mean()
    local variance = 0
    for _, v in ipairs(self.data) do
        variance = variance + (v - m)^2
    end
    return math.sqrt(variance / self:size())
end

-- Apply function to each element
function Tensor:apply(fn)
    local result = Tensor.zeros(self.shape)
    for i, v in ipairs(self.data) do
        result.data[i] = fn(v)
    end
    return result
end

-- ทดสอบ linear algebra
print("=== Linear Algebra ===")
math.randomseed(10)

local A = Tensor({{1, 2}, {3, 4}})
local B = Tensor({{5, 6}, {7, 8}})

print("A =")
print(tostring(A))
print("\nB =")
print(tostring(B))

local C = A:mm(B)
print("\nA @ B =")
print(tostring(C))

print("\nA + B =")
print(tostring(A:add(B)))

print("\nA transposed =")
print(tostring(A:t()))

local v = Tensor({1, 2, 3, 4, 5})
print("\nVector statistics:")
print(string.format("  sum=%.2f, mean=%.2f, std=%.2f, max=%.2f, min=%.2f",
    v:sum(), v:mean(), v:std(), v:max(), v:min()))
```

## Activation Functions

```lua
-- Activation functions สำหรับ neural networks

local Activations = {}

-- Sigmoid: f(x) = 1 / (1 + e^(-x))
function Activations.sigmoid(x)
    return 1 / (1 + math.exp(-x))
end

function Activations.sigmoid_derivative(x)
    local s = Activations.sigmoid(x)
    return s * (1 - s)
end

-- ReLU: f(x) = max(0, x)
function Activations.relu(x)
    return math.max(0, x)
end

function Activations.relu_derivative(x)
    return x > 0 and 1 or 0
end

-- Leaky ReLU
function Activations.leaky_relu(x, alpha)
    alpha = alpha or 0.01
    return x > 0 and x or alpha * x
end

-- Tanh
function Activations.tanh(x)
    return math.tanh(x)
end

function Activations.tanh_derivative(x)
    local t = math.tanh(x)
    return 1 - t * t
end

-- Softmax: สำหรับ multi-class classification
function Activations.softmax(tensor)
    local max_val = tensor:max()
    local exp_sum = 0
    local exp_vals = {}
    
    for i, v in ipairs(tensor.data) do
        local e = math.exp(v - max_val)  -- numerical stability
        exp_vals[i] = e
        exp_sum = exp_sum + e
    end
    
    local result = Tensor.zeros(tensor.shape)
    for i, e in ipairs(exp_vals) do
        result.data[i] = e / exp_sum
    end
    
    return result
end

-- GELU (Gaussian Error Linear Unit)
function Activations.gelu(x)
    return 0.5 * x * (1 + math.tanh(math.sqrt(2/math.pi) * (x + 0.044715 * x^3)))
end

-- Apply activation to tensor
function Activations.apply_to_tensor(activation_fn, tensor)
    return tensor:apply(activation_fn)
end

-- ทดสอบ activations
print("=== Activation Functions ===")

local test_values = {-3, -2, -1, -0.5, 0, 0.5, 1, 2, 3}
print(string.format("%-8s %-10s %-10s %-10s %-10s",
    "x", "sigmoid", "relu", "tanh", "leaky_relu"))
print(string.rep("-", 52))

for _, x in ipairs(test_values) do
    print(string.format("%-8.2f %-10.4f %-10.4f %-10.4f %-10.4f",
        x,
        Activations.sigmoid(x),
        Activations.relu(x),
        Activations.tanh(x),
        Activations.leaky_relu(x)))
end

-- Softmax
print("\nSoftmax example:")
local logits = Tensor({2.0, 1.0, 0.1})
local probs = Activations.softmax(logits)
print("Logits: " .. tostring(logits))
print("Probs:  " .. tostring(probs))
print(string.format("Sum of probs: %.4f", probs:sum()))

-- Sigmoid applied to tensor
print("\nSigmoid on tensor [-2, -1, 0, 1, 2]:")
local t = Tensor({-2, -1, 0, 1, 2})
local s = Activations.apply_to_tensor(Activations.sigmoid, t)
print(tostring(s))
```

## Neural Network จาก Scratch

### Layer Implementation

```lua
-- Neural network layer implementation

local Layer = {}
Layer.__index = Layer

function Layer.dense(input_size, output_size, activation)
    local self = setmetatable({}, Layer)
    self.type = "dense"
    self.input_size = input_size
    self.output_size = output_size
    self.activation = activation or "relu"
    
    -- Xavier/Glorot initialization
    local scale = math.sqrt(2.0 / (input_size + output_size))
    self.weights = Tensor.rand({input_size, output_size}, -scale, scale)
    self.biases = Tensor.zeros({1, output_size})
    
    -- Gradients
    self.dW = Tensor.zeros({input_size, output_size})
    self.db = Tensor.zeros({1, output_size})
    
    -- Cache for backprop
    self.input_cache = nil
    self.z_cache = nil
    
    return self
end

function Layer:forward(input)
    self.input_cache = input
    
    -- Z = X @ W + b
    local z = input:mm(self.weights)
    
    -- Add biases to each row
    local result = Tensor.zeros({input.shape[1], self.output_size})
    for i = 1, input.shape[1] do
        for j = 1, self.output_size do
            result:set(z:get(i, j) + self.biases:get(1, j), i, j)
        end
    end
    
    self.z_cache = result
    
    -- Apply activation
    if self.activation == "relu" then
        return result:apply(Activations.relu)
    elseif self.activation == "sigmoid" then
        return result:apply(Activations.sigmoid)
    elseif self.activation == "tanh" then
        return result:apply(Activations.tanh)
    elseif self.activation == "softmax" then
        return Activations.softmax(result)
    elseif self.activation == "linear" or self.activation == "none" then
        return result
    end
    
    return result
end

-- Simple Neural Network
local NeuralNetwork = {}
NeuralNetwork.__index = NeuralNetwork

function NeuralNetwork.new(layers)
    local self = setmetatable({}, NeuralNetwork)
    self.layers = layers
    self.learning_rate = 0.01
    return self
end

function NeuralNetwork:forward(input)
    local x = input
    for _, layer in ipairs(self.layers) do
        x = layer:forward(x)
    end
    return x
end

function NeuralNetwork:predict(input)
    return self:forward(input)
end

-- Loss functions
local Loss = {}

-- Mean Squared Error
function Loss.mse(predictions, targets)
    local diff = predictions:sub(targets)
    local squared = diff:apply(function(x) return x * x end)
    return squared:mean()
end

-- Binary Cross Entropy
function Loss.binary_cross_entropy(predictions, targets)
    local eps = 1e-7  -- numerical stability
    local total = 0
    local n = predictions:size()
    
    for i = 1, n do
        local p = math.max(eps, math.min(1 - eps, predictions.data[i]))
        local y = targets.data[i]
        total = total - (y * math.log(p) + (1 - y) * math.log(1 - p))
    end
    
    return total / n
end

-- Categorical Cross Entropy
function Loss.categorical_cross_entropy(predictions, targets)
    local eps = 1e-7
    local total = 0
    local batch_size = predictions.shape[1]
    local num_classes = predictions.shape[2]
    
    for i = 1, batch_size do
        for j = 1, num_classes do
            local p = math.max(eps, predictions:get(i, j))
            local y = targets:get(i, j)
            total = total - y * math.log(p)
        end
    end
    
    return total / batch_size
end

-- ทดสอบ neural network
print("=== Simple Neural Network ===")
math.randomseed(42)

-- สร้าง network: 4 input -> 8 hidden (ReLU) -> 1 output (sigmoid)
local network = NeuralNetwork.new({
    Layer.dense(4, 8, "relu"),
    Layer.dense(8, 4, "relu"),
    Layer.dense(4, 1, "sigmoid"),
})

-- สร้าง test data
local X = Tensor.rand({3, 4})  -- 3 samples, 4 features
local y = Tensor({{1}, {0}, {1}})  -- labels

print("Input shape: " .. X.shape[1] .. "x" .. X.shape[2])
print("Input data:")
print(tostring(X))

-- Forward pass
local output = network:forward(X)
print("\nNetwork output (probabilities):")
print(tostring(output))

-- Compute loss
local bce_loss = Loss.binary_cross_entropy(output, y)
print(string.format("\nBinary Cross Entropy Loss: %.4f", bce_loss))
```

## Forward Pass

```lua
-- Forward pass รายละเอียด

local function detailed_forward_pass()
    math.randomseed(100)
    
    print("=== Detailed Forward Pass ===")
    
    -- Simple 2-layer network: 2 -> 3 -> 1
    local W1 = Tensor({{0.1, 0.2, 0.3}, {-0.1, 0.4, -0.2}})  -- 2x3
    local b1 = Tensor({{0.1, -0.1, 0.05}})  -- 1x3
    local W2 = Tensor({{0.5}, {-0.3}, {0.8}})  -- 3x1
    local b2 = Tensor({{-0.2}})  -- 1x1
    
    -- Input
    local x = Tensor({{0.5, -0.3}})  -- 1 sample, 2 features
    
    print("Input x:")
    print(tostring(x))
    
    -- Layer 1: Z1 = x @ W1 + b1
    local Z1 = x:mm(W1)
    -- Add bias
    local Z1_biased = Tensor.zeros({1, 3})
    for j = 1, 3 do
        Z1_biased:set(Z1:get(1, j) + b1:get(1, j), 1, j)
    end
    print("\nZ1 (before activation):")
    print(tostring(Z1_biased))
    
    -- A1 = ReLU(Z1)
    local A1 = Z1_biased:apply(Activations.relu)
    print("\nA1 (after ReLU):")
    print(tostring(A1))
    
    -- Layer 2: Z2 = A1 @ W2 + b2
    local Z2 = A1:mm(W2)
    local Z2_biased = Tensor.zeros({1, 1})
    Z2_biased:set(Z2:get(1, 1) + b2:get(1, 1), 1, 1)
    print("\nZ2 (before activation):")
    print(tostring(Z2_biased))
    
    -- A2 = Sigmoid(Z2)
    local A2 = Z2_biased:apply(Activations.sigmoid)
    print("\nA2 (output, after Sigmoid):")
    print(tostring(A2))
    
    print(string.format("\nPrediction: %.4f (%.0f%%)", 
        A2:get(1, 1), A2:get(1, 1) * 100))
    
    return A2
end

detailed_forward_pass()
```

## Loss Function และ Backpropagation

```lua
-- Backpropagation implementation

local function backpropagation_example()
    print("=== Backpropagation Example ===")
    
    -- Simple single neuron example
    -- y_pred = sigmoid(w * x + b)
    -- loss = BCE(y_pred, y_true)
    
    local w = 0.5
    local b = -0.1
    local x = 0.8
    local y_true = 1.0
    
    -- Forward pass
    local z = w * x + b
    local y_pred = Activations.sigmoid(z)
    local loss = -(y_true * math.log(y_pred) + (1 - y_true) * math.log(1 - y_pred))
    
    print(string.format("Forward: z=%.4f, y_pred=%.4f, loss=%.4f", z, y_pred, loss))
    
    -- Backward pass (chain rule)
    -- dL/dy_pred = -(y_true/y_pred - (1-y_true)/(1-y_pred))
    local dL_dy = -(y_true / y_pred - (1 - y_true) / (1 - y_pred))
    
    -- dy_pred/dz = sigmoid'(z) = sigmoid(z) * (1 - sigmoid(z))
    local dy_dz = Activations.sigmoid_derivative(z)
    
    -- dL/dz = dL/dy * dy/dz
    local dL_dz = dL_dy * dy_dz
    
    -- dz/dw = x, dz/db = 1
    local dL_dw = dL_dz * x
    local dL_db = dL_dz
    
    print(string.format("Gradients: dL/dw=%.4f, dL/db=%.4f", dL_dw, dL_db))
    
    -- Update parameters (gradient descent)
    local learning_rate = 0.1
    local w_new = w - learning_rate * dL_dw
    local b_new = b - learning_rate * dL_db
    
    print(string.format("Updated: w: %.4f -> %.4f, b: %.4f -> %.4f",
        w, w_new, b, b_new))
    
    -- ตรวจสอบว่า loss ลดลง
    local z_new = w_new * x + b_new
    local y_pred_new = Activations.sigmoid(z_new)
    local loss_new = -(y_true * math.log(y_pred_new) + (1-y_true) * math.log(1-y_pred_new))
    
    print(string.format("Loss improved: %.4f -> %.4f (diff: %.4f)",
        loss, loss_new, loss - loss_new))
    
    return w_new, b_new
end

backpropagation_example()
```

## Gradient Descent

```lua
-- Gradient Descent variants

local Optimizer = {}

-- Stochastic Gradient Descent (SGD)
function Optimizer.sgd(params, grads, learning_rate)
    for i, param in ipairs(params) do
        params[i] = param - learning_rate * grads[i]
    end
    return params
end

-- SGD with Momentum
function Optimizer.sgd_momentum(params, grads, velocities, learning_rate, momentum)
    momentum = momentum or 0.9
    for i, param in ipairs(params) do
        velocities[i] = momentum * velocities[i] - learning_rate * grads[i]
        params[i] = param + velocities[i]
    end
    return params, velocities
end

-- Adam Optimizer
function Optimizer.adam(params, grads, m, v, t, lr, beta1, beta2, eps)
    lr = lr or 0.001
    beta1 = beta1 or 0.9
    beta2 = beta2 or 0.999
    eps = eps or 1e-8
    
    for i, grad in ipairs(grads) do
        m[i] = beta1 * m[i] + (1 - beta1) * grad
        v[i] = beta2 * v[i] + (1 - beta2) * grad * grad
        
        local m_hat = m[i] / (1 - beta1^t)
        local v_hat = v[i] / (1 - beta2^t)
        
        params[i] = params[i] - lr * m_hat / (math.sqrt(v_hat) + eps)
    end
    
    return params, m, v
end

-- ทดสอบ gradient descent บน simple function
-- Minimize f(x) = x^2 + 2x + 1 = (x+1)^2
-- Minimum at x = -1
print("=== Gradient Descent Optimization ===")
print("Minimizing f(x) = x^2 + 2x + 1")
print("Theoretical minimum: x = -1, f(-1) = 0")
print()

local function f(x)
    return x^2 + 2*x + 1
end

local function df(x)  -- gradient
    return 2*x + 2
end

-- SGD
local x = 5.0  -- starting point
local lr = 0.1
print("SGD (lr=0.1):")
print(string.format("  Start: x=%.4f, f(x)=%.4f", x, f(x)))
for i = 1, 20 do
    local grad = df(x)
    x = x - lr * grad
end
print(string.format("  After 20 steps: x=%.4f, f(x)=%.6f", x, f(x)))

-- Training loop
local function train_simple_model(X_data, y_data, epochs, lr)
    -- y = w * x (no bias, simple case)
    -- minimize MSE
    
    local w = math.random()  -- initial weight
    
    print(string.format("\nTraining linear model y = w*x"))
    print(string.format("Initial w = %.4f", w))
    
    local history = {}
    
    for epoch = 1, epochs do
        -- Forward pass
        local y_pred = {}
        for i, x in ipairs(X_data) do
            y_pred[i] = w * x
        end
        
        -- Compute MSE loss
        local loss = 0
        for i, y_p in ipairs(y_pred) do
            loss = loss + (y_p - y_data[i])^2
        end
        loss = loss / #X_data
        
        -- Compute gradient
        local dw = 0
        for i, x in ipairs(X_data) do
            dw = dw + 2 * (y_pred[i] - y_data[i]) * x
        end
        dw = dw / #X_data
        
        -- Update weight
        w = w - lr * dw
        
        if epoch % 10 == 0 then
            table.insert(history, {epoch = epoch, loss = loss, w = w})
            print(string.format("  Epoch %3d: loss=%.6f, w=%.4f", epoch, loss, w))
        end
    end
    
    return w, history
end

-- Training data: y = 2x + noise
math.randomseed(42)
local X_train = {}
local y_train = {}
for i = 1, 20 do
    local x = (math.random() - 0.5) * 10
    local y = 2 * x + (math.random() - 0.5) * 0.5  -- true w=2 + noise
    table.insert(X_train, x)
    table.insert(y_train, y)
end

local final_w = train_simple_model(X_train, y_train, 100, 0.01)
print(string.format("Final w = %.4f (target: 2.0)", final_w))
```

## Logistic Regression

```lua
-- Logistic Regression สำหรับ binary classification

local LogisticRegression = {}
LogisticRegression.__index = LogisticRegression

function LogisticRegression.new(learning_rate, iterations)
    local self = setmetatable({}, LogisticRegression)
    self.lr = learning_rate or 0.01
    self.n_iter = iterations or 1000
    self.weights = nil
    self.bias = 0
    self.loss_history = {}
    return self
end

function LogisticRegression:fit(X, y)
    local n_samples = #X
    local n_features = #X[1]
    
    -- Initialize weights
    self.weights = {}
    for i = 1, n_features do
        self.weights[i] = 0
    end
    self.bias = 0
    
    for iter = 1, self.n_iter do
        -- Forward pass
        local y_pred = {}
        for i = 1, n_samples do
            local z = self.bias
            for j = 1, n_features do
                z = z + self.weights[j] * X[i][j]
            end
            y_pred[i] = Activations.sigmoid(z)
        end
        
        -- Compute loss (BCE)
        local loss = 0
        local eps = 1e-7
        for i = 1, n_samples do
            local p = math.max(eps, math.min(1 - eps, y_pred[i]))
            loss = loss - (y[i] * math.log(p) + (1 - y[i]) * math.log(1 - p))
        end
        loss = loss / n_samples
        
        if iter % 100 == 0 then
            table.insert(self.loss_history, {iter = iter, loss = loss})
        end
        
        -- Compute gradients
        local dw = {}
        for j = 1, n_features do
            dw[j] = 0
        end
        local db = 0
        
        for i = 1, n_samples do
            local error = y_pred[i] - y[i]
            for j = 1, n_features do
                dw[j] = dw[j] + error * X[i][j]
            end
            db = db + error
        end
        
        -- Update parameters
        for j = 1, n_features do
            self.weights[j] = self.weights[j] - self.lr * dw[j] / n_samples
        end
        self.bias = self.bias - self.lr * db / n_samples
    end
end

function LogisticRegression:predict_proba(X)
    local probs = {}
    for i = 1, #X do
        local z = self.bias
        for j = 1, #self.weights do
            z = z + self.weights[j] * X[i][j]
        end
        probs[i] = Activations.sigmoid(z)
    end
    return probs
end

function LogisticRegression:predict(X, threshold)
    threshold = threshold or 0.5
    local probs = self:predict_proba(X)
    local predictions = {}
    for i, p in ipairs(probs) do
        predictions[i] = p >= threshold and 1 or 0
    end
    return predictions
end

-- สร้าง dataset จำลอง
local function generate_classification_data(n_samples)
    math.randomseed(42)
    local X = {}
    local y = {}
    
    for i = 1, n_samples do
        if i <= n_samples // 2 then
            -- Class 0: centered at (-1, -1)
            X[i] = {-1 + math.random(), -1 + math.random()}
            y[i] = 0
        else
            -- Class 1: centered at (1, 1)
            X[i] = {1 + math.random(), 1 + math.random()}
            y[i] = 1
        end
    end
    
    return X, y
end

print("=== Logistic Regression ===")

local X_train, y_train = generate_classification_data(100)
local X_test, y_test = generate_classification_data(20)

local lr_model = LogisticRegression.new(0.1, 500)
print("Training...")
lr_model:fit(X_train, y_train)

print("Training loss history:")
for _, record in ipairs(lr_model.loss_history) do
    print(string.format("  Iter %3d: loss = %.4f", record.iter, record.loss))
end

-- Evaluate
local y_pred = lr_model:predict(X_test)
local correct = 0
for i = 1, #y_test do
    if y_pred[i] == y_test[i] then
        correct = correct + 1
    end
end
print(string.format("\nTest Accuracy: %d/%d = %.1f%%",
    correct, #y_test, correct/#y_test * 100))

-- Weights
print(string.format("Learned weights: w1=%.4f, w2=%.4f, bias=%.4f",
    lr_model.weights[1], lr_model.weights[2], lr_model.bias))
```

## k-Nearest Neighbors (kNN)

```lua
-- kNN Algorithm

local KNN = {}
KNN.__index = KNN

function KNN.new(k)
    local self = setmetatable({}, KNN)
    self.k = k or 3
    self.X_train = nil
    self.y_train = nil
    return self
end

function KNN:fit(X, y)
    self.X_train = X
    self.y_train = y
    print(string.format("KNN fitted with %d training samples, k=%d", #X, self.k))
end

function KNN:euclidean_distance(a, b)
    local sum = 0
    for i = 1, #a do
        sum = sum + (a[i] - b[i])^2
    end
    return math.sqrt(sum)
end

function KNN:predict(X_test)
    local predictions = {}
    
    for i, test_point in ipairs(X_test) do
        -- คำนวณ distance กับทุก training point
        local distances = {}
        for j, train_point in ipairs(self.X_train) do
            local dist = self:euclidean_distance(test_point, train_point)
            table.insert(distances, {idx = j, dist = dist, label = self.y_train[j]})
        end
        
        -- Sort by distance
        table.sort(distances, function(a, b) return a.dist < b.dist end)
        
        -- นับ votes จาก k neighbors
        local vote_counts = {}
        for n = 1, math.min(self.k, #distances) do
            local label = distances[n].label
            vote_counts[label] = (vote_counts[label] or 0) + 1
        end
        
        -- หา majority vote
        local best_label = nil
        local best_count = 0
        for label, count in pairs(vote_counts) do
            if count > best_count then
                best_count = count
                best_label = label
            end
        end
        
        predictions[i] = best_label
    end
    
    return predictions
end

function KNN:predict_proba(X_test)
    local all_probs = {}
    local classes = {}
    
    -- Get unique classes
    local class_set = {}
    for _, y in ipairs(self.y_train) do
        class_set[y] = true
    end
    for c in pairs(class_set) do
        table.insert(classes, c)
    end
    table.sort(classes)
    
    for i, test_point in ipairs(X_test) do
        local distances = {}
        for j, train_point in ipairs(self.X_train) do
            table.insert(distances, {dist = self:euclidean_distance(test_point, train_point),
                                    label = self.y_train[j]})
        end
        table.sort(distances, function(a, b) return a.dist < b.dist end)
        
        local vote_counts = {}
        for _, c in ipairs(classes) do
            vote_counts[c] = 0
        end
        
        for n = 1, math.min(self.k, #distances) do
            vote_counts[distances[n].label] = vote_counts[distances[n].label] + 1
        end
        
        local probs = {}
        for _, c in ipairs(classes) do
            probs[c] = vote_counts[c] / self.k
        end
        
        all_probs[i] = probs
    end
    
    return all_probs
end

-- ทดสอบ kNN
print("=== k-Nearest Neighbors ===")

-- Iris-like dataset (simplified)
math.randomseed(123)
local function generate_iris_like(n)
    local X, y = {}, {}
    local classes = {
        {center = {1, 1}, label = 0},
        {center = {3, 3}, label = 1},
        {center = {5, 1}, label = 2},
    }
    
    for i = 1, n do
        local cls = classes[(i % 3) + 1]
        X[i] = {
            cls.center[1] + (math.random() - 0.5),
            cls.center[2] + (math.random() - 0.5),
        }
        y[i] = cls.label
    end
    
    return X, y
end

local X_train_knn, y_train_knn = generate_iris_like(60)
local X_test_knn, y_test_knn = generate_iris_like(15)

local knn = KNN.new(5)
knn:fit(X_train_knn, y_train_knn)

local y_pred_knn = knn:predict(X_test_knn)
local correct_knn = 0
for i = 1, #y_test_knn do
    if y_pred_knn[i] == y_test_knn[i] then
        correct_knn = correct_knn + 1
    end
end

print(string.format("KNN (k=5) Test Accuracy: %d/%d = %.1f%%",
    correct_knn, #y_test_knn, correct_knn/#y_test_knn * 100))

-- Test point prediction with probabilities
local test_point = {{2, 2}}
local probs = knn:predict_proba(test_point)
print("\nProbabilities for point (2, 2):")
for class, prob in pairs(probs[1]) do
    print(string.format("  Class %s: %.2f%%", tostring(class), prob * 100))
end
```

## k-Means Clustering

```lua
-- k-Means Clustering

local KMeans = {}
KMeans.__index = KMeans

function KMeans.new(k, max_iter, tolerance)
    local self = setmetatable({}, KMeans)
    self.k = k
    self.max_iter = max_iter or 100
    self.tolerance = tolerance or 1e-4
    self.centroids = nil
    self.labels = nil
    self.inertia = nil
    return self
end

function KMeans:euclidean_distance(a, b)
    local sum = 0
    for i = 1, #a do
        sum = sum + (a[i] - b[i])^2
    end
    return math.sqrt(sum)
end

function KMeans:fit(X)
    local n_samples = #X
    local n_features = #X[1]
    
    -- Initialize centroids (k-means++ style, simplified)
    self.centroids = {}
    -- เลือก centroid แรกแบบ random
    table.insert(self.centroids, {table.unpack(X[math.random(1, n_samples)])})
    
    -- เลือก centroids ที่เหลือโดยอิงจาก distance
    for c = 2, self.k do
        local distances = {}
        for i = 1, n_samples do
            local min_dist = math.huge
            for _, centroid in ipairs(self.centroids) do
                local d = self:euclidean_distance(X[i], centroid)
                if d < min_dist then min_dist = d end
            end
            distances[i] = min_dist
        end
        
        -- เลือก point ที่มี distance สูงสุด (deterministic variant)
        local max_dist_idx = 1
        for i, d in ipairs(distances) do
            if d > distances[max_dist_idx] then max_dist_idx = i end
        end
        table.insert(self.centroids, {table.unpack(X[max_dist_idx])})
    end
    
    -- Iteration
    for iter = 1, self.max_iter do
        -- Assign clusters
        local old_centroids = {}
        for c, centroid in ipairs(self.centroids) do
            old_centroids[c] = {table.unpack(centroid)}
        end
        
        self.labels = {}
        local cluster_sums = {}
        local cluster_counts = {}
        
        for c = 1, self.k do
            cluster_sums[c] = {}
            for j = 1, n_features do
                cluster_sums[c][j] = 0
            end
            cluster_counts[c] = 0
        end
        
        -- Assign each point to nearest centroid
        self.inertia = 0
        for i = 1, n_samples do
            local min_dist = math.huge
            local nearest = 1
            
            for c, centroid in ipairs(self.centroids) do
                local d = self:euclidean_distance(X[i], centroid)
                if d < min_dist then
                    min_dist = d
                    nearest = c
                end
            end
            
            self.labels[i] = nearest
            self.inertia = self.inertia + min_dist^2
            cluster_counts[nearest] = cluster_counts[nearest] + 1
            
            for j = 1, n_features do
                cluster_sums[nearest][j] = cluster_sums[nearest][j] + X[i][j]
            end
        end
        
        -- Update centroids
        local max_shift = 0
        for c = 1, self.k do
            if cluster_counts[c] > 0 then
                for j = 1, n_features do
                    self.centroids[c][j] = cluster_sums[c][j] / cluster_counts[c]
                end
                
                -- ตรวจสอบ convergence
                local shift = self:euclidean_distance(self.centroids[c], old_centroids[c])
                if shift > max_shift then max_shift = shift end
            end
        end
        
        if max_shift < self.tolerance then
            print(string.format("KMeans converged at iteration %d", iter))
            break
        end
    end
    
    return self
end

function KMeans:predict(X)
    local labels = {}
    for i, point in ipairs(X) do
        local min_dist = math.huge
        local nearest = 1
        
        for c, centroid in ipairs(self.centroids) do
            local d = self:euclidean_distance(point, centroid)
            if d < min_dist then
                min_dist = d
                nearest = c
            end
        end
        labels[i] = nearest
    end
    return labels
end

-- ทดสอบ k-Means
print("=== k-Means Clustering ===")
math.randomseed(42)

-- สร้าง clustered data
local cluster_data = {}
local cluster_centers = {{1, 1}, {4, 4}, {7, 2}}
for _, center in ipairs(cluster_centers) do
    for i = 1, 20 do
        table.insert(cluster_data, {
            center[1] + (math.random() - 0.5) * 1.5,
            center[2] + (math.random() - 0.5) * 1.5,
        })
    end
end

local kmeans = KMeans.new(3)
kmeans:fit(cluster_data)

print(string.format("Inertia (within-cluster sum of squares): %.2f", kmeans.inertia))
print("Cluster centroids:")
for c, centroid in ipairs(kmeans.centroids) do
    print(string.format("  Cluster %d: (%.2f, %.2f)", c, centroid[1], centroid[2]))
end

-- Count samples per cluster
local cluster_counts = {}
for i = 1, 3 do cluster_counts[i] = 0 end
for _, label in ipairs(kmeans.labels) do
    cluster_counts[label] = cluster_counts[label] + 1
end

print("Samples per cluster:")
for c, count in ipairs(cluster_counts) do
    print(string.format("  Cluster %d: %d samples", c, count))
end
```

## Feature Scaling

```lua
-- Feature Scaling สำคัญมากสำหรับ ML algorithms

local Scaler = {}

-- Min-Max Scaler: scale to [0, 1]
function Scaler.min_max(X)
    local n_features = #X[1]
    local min_vals = {}
    local max_vals = {}
    
    -- คำนวณ min, max สำหรับแต่ละ feature
    for j = 1, n_features do
        min_vals[j] = math.huge
        max_vals[j] = -math.huge
        for i = 1, #X do
            if X[i][j] < min_vals[j] then min_vals[j] = X[i][j] end
            if X[i][j] > max_vals[j] then max_vals[j] = X[i][j] end
        end
    end
    
    -- Scale ข้อมูล
    local X_scaled = {}
    for i = 1, #X do
        X_scaled[i] = {}
        for j = 1, n_features do
            local range = max_vals[j] - min_vals[j]
            if range == 0 then
                X_scaled[i][j] = 0
            else
                X_scaled[i][j] = (X[i][j] - min_vals[j]) / range
            end
        end
    end
    
    return X_scaled, {min = min_vals, max = max_vals}
end

-- Standard Scaler: zero mean, unit variance
function Scaler.standard(X)
    local n_features = #X[1]
    local means = {}
    local stds = {}
    
    -- คำนวณ mean
    for j = 1, n_features do
        local sum = 0
        for i = 1, #X do
            sum = sum + X[i][j]
        end
        means[j] = sum / #X
    end
    
    -- คำนวณ std
    for j = 1, n_features do
        local sum_sq = 0
        for i = 1, #X do
            sum_sq = sum_sq + (X[i][j] - means[j])^2
        end
        stds[j] = math.sqrt(sum_sq / #X)
    end
    
    -- Scale
    local X_scaled = {}
    for i = 1, #X do
        X_scaled[i] = {}
        for j = 1, n_features do
            if stds[j] == 0 then
                X_scaled[i][j] = 0
            else
                X_scaled[i][j] = (X[i][j] - means[j]) / stds[j]
            end
        end
    end
    
    return X_scaled, {mean = means, std = stds}
end

-- Robust Scaler (ใช้ median และ IQR)
function Scaler.robust(X)
    local n_features = #X[1]
    local medians = {}
    local iqrs = {}
    
    for j = 1, n_features do
        local col = {}
        for i = 1, #X do
            table.insert(col, X[i][j])
        end
        table.sort(col)
        
        local n = #col
        medians[j] = col[math.ceil(n / 2)]
        iqrs[j] = col[math.ceil(n * 0.75)] - col[math.ceil(n * 0.25)]
    end
    
    local X_scaled = {}
    for i = 1, #X do
        X_scaled[i] = {}
        for j = 1, n_features do
            if iqrs[j] == 0 then
                X_scaled[i][j] = 0
            else
                X_scaled[i][j] = (X[i][j] - medians[j]) / iqrs[j]
            end
        end
    end
    
    return X_scaled, {median = medians, iqr = iqrs}
end

-- ทดสอบ Feature Scaling
print("=== Feature Scaling ===")

local data = {
    {100, 0.5, 30000},
    {200, 1.0, 50000},
    {150, 0.8, 40000},
    {300, 2.0, 80000},
    {50,  0.2, 20000},
}

print("Original data (first 3 rows):")
for i = 1, 3 do
    print(string.format("  [%.0f, %.1f, %.0f]", data[i][1], data[i][2], data[i][3]))
end

local mm_scaled, mm_params = Scaler.min_max(data)
print("\nMin-Max Scaled:")
for i = 1, 3 do
    print(string.format("  [%.4f, %.4f, %.4f]", mm_scaled[i][1], mm_scaled[i][2], mm_scaled[i][3]))
end

local std_scaled, std_params = Scaler.standard(data)
print("\nStandard Scaled:")
for i = 1, 3 do
    print(string.format("  [%6.3f, %6.3f, %6.3f]", std_scaled[i][1], std_scaled[i][2], std_scaled[i][3]))
end

print("\nStandard scaler params:")
print(string.format("  means: [%.2f, %.3f, %.0f]",
    std_params.mean[1], std_params.mean[2], std_params.mean[3]))
print(string.format("  stds:  [%.2f, %.3f, %.0f]",
    std_params.std[1], std_params.std[2], std_params.std[3]))
```

## Model Evaluation

```lua
-- Metrics สำหรับ evaluate model performance

local Metrics = {}

-- Accuracy
function Metrics.accuracy(y_true, y_pred)
    local correct = 0
    for i = 1, #y_true do
        if y_true[i] == y_pred[i] then
            correct = correct + 1
        end
    end
    return correct / #y_true
end

-- Confusion Matrix (binary)
function Metrics.confusion_matrix(y_true, y_pred)
    local tp, tn, fp, fn = 0, 0, 0, 0
    
    for i = 1, #y_true do
        if y_true[i] == 1 and y_pred[i] == 1 then tp = tp + 1
        elseif y_true[i] == 0 and y_pred[i] == 0 then tn = tn + 1
        elseif y_true[i] == 0 and y_pred[i] == 1 then fp = fp + 1
        elseif y_true[i] == 1 and y_pred[i] == 0 then fn = fn + 1
        end
    end
    
    return {tp = tp, tn = tn, fp = fp, fn = fn}
end

-- Precision, Recall, F1
function Metrics.precision_recall_f1(y_true, y_pred)
    local cm = Metrics.confusion_matrix(y_true, y_pred)
    
    local precision = cm.tp + cm.fp > 0 and cm.tp / (cm.tp + cm.fp) or 0
    local recall = cm.tp + cm.fn > 0 and cm.tp / (cm.tp + cm.fn) or 0
    local f1 = precision + recall > 0 and 
        2 * precision * recall / (precision + recall) or 0
    
    return {
        precision = precision,
        recall = recall,
        f1 = f1,
        confusion_matrix = cm
    }
end

-- Mean Absolute Error
function Metrics.mae(y_true, y_pred)
    local sum = 0
    for i = 1, #y_true do
        sum = sum + math.abs(y_true[i] - y_pred[i])
    end
    return sum / #y_true
end

-- Root Mean Squared Error
function Metrics.rmse(y_true, y_pred)
    local sum = 0
    for i = 1, #y_true do
        sum = sum + (y_true[i] - y_pred[i])^2
    end
    return math.sqrt(sum / #y_true)
end

-- R-squared
function Metrics.r_squared(y_true, y_pred)
    local y_mean = 0
    for _, y in ipairs(y_true) do
        y_mean = y_mean + y
    end
    y_mean = y_mean / #y_true
    
    local ss_res = 0  -- residual sum of squares
    local ss_tot = 0  -- total sum of squares
    
    for i = 1, #y_true do
        ss_res = ss_res + (y_true[i] - y_pred[i])^2
        ss_tot = ss_tot + (y_true[i] - y_mean)^2
    end
    
    if ss_tot == 0 then return 1 end
    return 1 - ss_res / ss_tot
end

-- Cross-validation
function Metrics.cross_validate(X, y, model_class, k_folds, model_params)
    local n = #X
    local fold_size = math.floor(n / k_folds)
    local scores = {}
    
    for fold = 1, k_folds do
        -- แบ่ง train/test
        local test_start = (fold - 1) * fold_size + 1
        local test_end = fold == k_folds and n or fold * fold_size
        
        local X_train, y_train = {}, {}
        local X_test, y_test = {}, {}
        
        for i = 1, n do
            if i >= test_start and i <= test_end then
                table.insert(X_test, X[i])
                table.insert(y_test, y[i])
            else
                table.insert(X_train, X[i])
                table.insert(y_train, y[i])
            end
        end
        
        -- Train model
        local model = model_class.new(table.unpack(model_params or {}))
        model:fit(X_train, y_train)
        
        -- Evaluate
        local y_pred = model:predict(X_test)
        local acc = Metrics.accuracy(y_test, y_pred)
        table.insert(scores, acc)
        
        print(string.format("  Fold %d: accuracy = %.4f", fold, acc))
    end
    
    -- คำนวณ mean และ std
    local mean_score = 0
    for _, s in ipairs(scores) do mean_score = mean_score + s end
    mean_score = mean_score / #scores
    
    local std_score = 0
    for _, s in ipairs(scores) do
        std_score = std_score + (s - mean_score)^2
    end
    std_score = math.sqrt(std_score / #scores)
    
    return {scores = scores, mean = mean_score, std = std_score}
end

-- ทดสอบ metrics
print("=== Model Evaluation Metrics ===")

-- Classification metrics
local y_true_cls = {1, 0, 1, 1, 0, 1, 0, 0, 1, 0}
local y_pred_cls = {1, 0, 1, 0, 0, 1, 1, 0, 1, 0}

print("Classification Report:")
local acc = Metrics.accuracy(y_true_cls, y_pred_cls)
print(string.format("  Accuracy:  %.4f (%.1f%%)", acc, acc * 100))

local prf = Metrics.precision_recall_f1(y_true_cls, y_pred_cls)
print(string.format("  Precision: %.4f", prf.precision))
print(string.format("  Recall:    %.4f", prf.recall))
print(string.format("  F1-Score:  %.4f", prf.f1))
print("  Confusion Matrix:")
print(string.format("    TP=%d, TN=%d, FP=%d, FN=%d",
    prf.confusion_matrix.tp, prf.confusion_matrix.tn,
    prf.confusion_matrix.fp, prf.confusion_matrix.fn))

-- Regression metrics
local y_true_reg = {1.5, 2.0, 3.5, 4.0, 5.0}
local y_pred_reg = {1.3, 2.2, 3.3, 4.5, 4.8}

print("\nRegression Report:")
print(string.format("  MAE:   %.4f", Metrics.mae(y_true_reg, y_pred_reg)))
print(string.format("  RMSE:  %.4f", Metrics.rmse(y_true_reg, y_pred_reg)))
print(string.format("  R²:    %.4f", Metrics.r_squared(y_true_reg, y_pred_reg)))

-- Cross-validation
print("\nCross-Validation (5-fold KNN):")
local X_cv, y_cv = generate_classification_data(50)
local cv_results = Metrics.cross_validate(X_cv, y_cv, KNN, 5, {3})
print(string.format("  Mean: %.4f ± %.4f", cv_results.mean, cv_results.std))
```

## สรุปบทที่ 74

```lua
-- สรุป Machine Learning กับ Lua

print("=" .. string.rep("=", 55))
print("บทที่ 74: Machine Learning กับ Lua")
print("=" .. string.rep("=", 55))

local topics = {
    "Torch/Lua History - บริบทประวัติศาสตร์",
    "Tensors - n-dimensional arrays",
    "Linear Algebra - matrix operations",
    "Activation Functions - sigmoid, relu, tanh, softmax",
    "Neural Network - layer, forward pass",
    "Loss Functions - MSE, BCE, CCE",
    "Backpropagation - chain rule",
    "Gradient Descent - SGD, momentum, Adam",
    "Logistic Regression - binary classification",
    "k-Nearest Neighbors - non-parametric",
    "k-Means Clustering - unsupervised",
    "Feature Scaling - min-max, standard, robust",
    "Model Evaluation - accuracy, precision, recall, F1",
    "Cross-Validation - k-fold",
}

for i, topic in ipairs(topics) do
    print(string.format("  %2d. %s", i, topic))
end

print("\nKey Takeaways:")
print("  * Lua ถูกใช้ใน Torch framework สำหรับ deep learning")
print("  * Tensors เป็นโครงสร้างข้อมูลหลักใน ML")
print("  * Backpropagation ใช้ chain rule คำนวณ gradients")
print("  * Feature scaling สำคัญสำหรับ distance-based algorithms")
print("  * Cross-validation ช่วย estimate model performance อย่างน่าเชื่อถือ")
```
