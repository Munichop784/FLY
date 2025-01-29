-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

-- Player setup
local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local playerGui = player:WaitForChild("PlayerGui")

-- Config
local FLIGHT_SPEED = 50
local TOGGLE_KEY = Enum.KeyCode.L
local ASCEND_KEY = Enum.KeyCode.Space
local DESCEND_KEY = Enum.KeyCode.LeftShift
local SPEED_INCREMENT = 20
local MAX_SPEED = 500

-- Variables
local flying = false
local camera = workspace.CurrentCamera
local humanoid, rootPart

-- Create Status UI for Flight
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FlightGui"
screenGui.Parent = playerGui

local statusFrame = Instance.new("Frame")
statusFrame.Size = UDim2.new(0, 200, 0, 100)
statusFrame.Position = UDim2.new(0.85, -100, 0.1, 0)
statusFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
statusFrame.BackgroundTransparency = 0.5
statusFrame.BorderSizePixel = 0
statusFrame.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent = statusFrame

local statusText = Instance.new("TextLabel")
statusText.Size = UDim2.new(1, 0, 0.5, 0)
statusText.BackgroundTransparency = 1
statusText.TextColor3 = Color3.fromRGB(255, 255, 255)
statusText.TextSize = 18
statusText.Font = Enum.Font.GothamBold
statusText.Text = "Flight: OFF"
statusText.Parent = statusFrame

local speedText = Instance.new("TextLabel")
speedText.Size = UDim2.new(1, 0, 0.5, 0)
speedText.Position = UDim2.new(0, 0, 0.5, 0)
speedText.BackgroundTransparency = 1
speedText.TextColor3 = Color3.fromRGB(255, 255, 255)
speedText.TextSize = 18
speedText.Font = Enum.Font.GothamBold
speedText.Text = "Speed: " .. FLIGHT_SPEED
speedText.Parent = statusFrame

-- Functions
local function updateStatus(isFlying)
    statusText.TextColor3 = isFlying and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    statusText.Text = "Flight: " .. (isFlying and "ON" or "OFF")
    
    local tween = TweenService:Create(statusFrame, TweenInfo.new(0.3), {
        BackgroundTransparency = isFlying and 0.3 or 0.5
    })
    tween:Play()
end

local function updateSpeedDisplay()
    speedText.Text = "Speed: " .. FLIGHT_SPEED
end

local function getCharacter()
    character = player.Character or player.CharacterAdded:Wait()
    humanoid = character:WaitForChild("Humanoid")
    rootPart = character:WaitForChild("HumanoidRootPart")
end

local function updateFlight()
    if not flying or not rootPart then return end
    
    local moveDirection = Vector3.new(0, 0, 0)
    local lookVector = camera.CFrame.LookVector
    local rightVector = camera.CFrame.RightVector
    
    -- Calculate movement direction
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then
        moveDirection = moveDirection + lookVector
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then
        moveDirection = moveDirection - lookVector
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then
        moveDirection = moveDirection - rightVector
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then
        moveDirection = moveDirection + rightVector
    end
    if UserInputService:IsKeyDown(ASCEND_KEY) then
        moveDirection = moveDirection + Vector3.new(0, 1, 0)
    end
    if UserInputService:IsKeyDown(DESCEND_KEY) then
        moveDirection = moveDirection - Vector3.new(0, 1, 0)
    end
    
    -- Normalize and apply speed
    if moveDirection.Magnitude > 0 then
        moveDirection = moveDirection.Unit * (FLIGHT_SPEED / 10)
    end
    
    -- Update position using CFrame
    if rootPart then
        rootPart.CFrame = rootPart.CFrame + moveDirection
        -- ป้องกัน fall damage
        rootPart.Velocity = Vector3.new(0, 0, 0)
    end
end

local function toggleFlight()
    flying = not flying
    
    if flying then
        getCharacter()
        if humanoid and rootPart then
            -- ป้องกันการตกและ animation ต่างๆ
            humanoid:ChangeState(Enum.HumanoidStateType.Swimming)
            rootPart.Velocity = Vector3.new(0, 0, 0)
        end
    else
        if humanoid then
            humanoid:ChangeState(Enum.HumanoidStateType.Landing)
            task.wait(0.1)
            humanoid:ChangeState(Enum.HumanoidStateType.Running)
        end
    end
    
    updateStatus(flying)
end

local function adjustSpeed(increment)
    FLIGHT_SPEED = math.clamp(FLIGHT_SPEED + increment, 10, MAX_SPEED)
    updateSpeedDisplay()
    
    local flashTween = TweenService:Create(speedText, TweenInfo.new(0.2), {
        TextColor3 = Color3.fromRGB(255, 255, 0)
    })
    flashTween:Play()
    
    task.wait(0.2)
    speedText.TextColor3 = Color3.fromRGB(255, 255, 255)
end

-- Input handling
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if input.KeyCode == TOGGLE_KEY then
        toggleFlight()
    elseif input.KeyCode == Enum.KeyCode.Q and flying then
        adjustSpeed(-SPEED_INCREMENT)
    elseif input.KeyCode == Enum.KeyCode.E and flying then
        adjustSpeed(SPEED_INCREMENT)
    end
end)

-- Main loop
RunService.Heartbeat:Connect(function()
    if flying and rootPart then
        -- ป้องกัน fall damage อย่างต่อเนื่อง
        rootPart.Velocity = Vector3.new(0, 0, 0)
        updateFlight()
    end
end)

-- Character handling
player.CharacterAdded:Connect(function(newCharacter)
    character = newCharacter
    humanoid = character:WaitForChild("Humanoid")
    rootPart = character:WaitForChild("HumanoidRootPart")
    
    if flying then
        task.wait(0.1)
        humanoid:ChangeState(Enum.HumanoidStateType.Swimming)
        rootPart.Velocity = Vector3.new(0, 0, 0)
    end
end)

-- Initial setup
getCharacter()
updateStatus(false)
updateSpeedDisplay()

-- Create Controls GUI
local controlsGui = Instance.new("ScreenGui")
controlsGui.Parent = playerGui

local controlsFrame = Instance.new("Frame")
controlsFrame.Size = UDim2.new(0, 250, 0, 300)
controlsFrame.Position = UDim2.new(0, 10, 0.6, 0)
controlsFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
controlsFrame.BackgroundTransparency = 0.5
controlsFrame.BorderSizePixel = 0
controlsFrame.Parent = controlsGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent = controlsFrame

local controlsText = Instance.new("TextLabel")
controlsText.Size = UDim2.new(1, -20, 1, -20)
controlsText.Position = UDim2.new(0, 10, 0, 10)
controlsText.BackgroundTransparency = 1
controlsText.TextColor3 = Color3.fromRGB(255, 255, 255)
controlsText.TextSize = 14
controlsText.Text = [[Flight Controls:
L - Toggle Flight
WASD - Movement
Space - Up
Shift - Down
Q/E - Adjust Speed]]
controlsText.Parent = controlsFrame

-- Flight Button
local flightButton = Instance.new("TextButton")
flightButton.Parent = controlsFrame
flightButton.Size = UDim2.new(0.8, 0, 0.15, 0)
flightButton.Position = UDim2.new(0.1, 0, 0.8, 0)
flightButton.Text = "🕊️ Flight (L)"
flightButton.BackgroundColor3 = Color3.fromRGB(0, 255, 255)
flightButton.TextColor3 = Color3.fromRGB(0, 0, 0)
flightButton.TextSize = 20
flightButton.Font = Enum.Font.GothamBold
flightButton.MouseButton1Click:Connect(function()
    toggleFlight()
end)

-- Auto-hide hint
task.wait(5)
controlsGui:Destroy()
