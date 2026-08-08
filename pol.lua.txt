-- =====================================================================
-- 🚀【纯飞行面板 - 按键全集成版】
-- =====================================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")

local plr = Players.LocalPlayer
local camera = Workspace.CurrentCamera

-- ==========================================
-- 【飞行状态】
-- ==========================================
local Flight = {
    Enabled = false,
    Speed = 60,
    Up = false,
    Down = false
}

local character, rootPart, humanoid
local renderConnection = nil

-- ==========================================
-- 【角色监听】
-- ==========================================
local function setupChar(char)
    character = char
    task.spawn(function()
        rootPart = char:WaitForChild("HumanoidRootPart", 5)
        humanoid = char:WaitForChild("Humanoid", 5)
    end)
end

if plr.Character then setupChar(plr.Character) end
plr.CharacterAdded:Connect(setupChar)

-- ==========================================
-- 【UI 构建 - 极简版】
-- ==========================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "XiaoFeiHub"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
pcall(function() screenGui.Parent = CoreGui end)
if not screenGui.Parent then screenGui.Parent = plr:WaitForChild("PlayerGui") end

-- ==========================================
-- 【主窗口】
-- ==========================================
local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Parent = screenGui
mainFrame.Size = UDim2.new(0, 300, 0, 200)
mainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
mainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
mainFrame.BackgroundColor3 = Color3.fromRGB(16, 17, 22)
mainFrame.Visible = true
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 10)
local mainStroke = Instance.new("UIStroke")
mainStroke.Color = Color3.fromRGB(80, 160, 255)
mainStroke.Thickness = 1.5
mainStroke.Parent = mainFrame

-- 标题
local title = Instance.new("TextLabel")
title.Parent = mainFrame
title.BackgroundTransparency = 1
title.Position = UDim2.new(0, 0, 0.05, 0)
title.Size = UDim2.new(1, 0, 0, 30)
title.Font = Enum.Font.GothamBold
title.Text = "🕊️ 小飞飞行控制"
title.TextColor3 = Color3.fromRGB(80, 160, 255)
title.TextSize = 14
title.TextYAlignment = Enum.TextYAlignment.Center

-- 关闭按钮
local closeBtn = Instance.new("TextButton")
closeBtn.Parent = mainFrame
closeBtn.BackgroundTransparency = 1
closeBtn.Position = UDim2.new(0.92, 0, 0.05, 0)
closeBtn.Size = UDim2.new(0, 24, 0, 24)
closeBtn.Font = Enum.Font.GothamBold
closeBtn.Text = "×"
closeBtn.TextColor3 = Color3.fromRGB(160, 160, 180)
closeBtn.TextSize = 16
closeBtn.MouseButton1Click:Connect(function()
    Flight.Enabled = false
    if renderConnection then renderConnection:Disconnect() end
    screenGui:Destroy()
end)

-- ==========================================
-- 【控制区域】
-- ==========================================
local controlArea = Instance.new("Frame")
controlArea.Parent = mainFrame
controlArea.BackgroundTransparency = 1
controlArea.Position = UDim2.new(0.05, 0, 0.25, 0)
controlArea.Size = UDim2.new(0.9, 0, 0.7, 0)

-- 开关
local toggleFrame = Instance.new("Frame")
toggleFrame.Parent = controlArea
toggleFrame.Size = UDim2.new(1, 0, 0, 40)
toggleFrame.BackgroundColor3 = Color3.fromRGB(24, 26, 35)
Instance.new("UICorner", toggleFrame).CornerRadius = UDim.new(0, 6)

local toggleLabel = Instance.new("TextLabel")
toggleLabel.Parent = toggleFrame
toggleLabel.BackgroundTransparency = 1
toggleLabel.Position = UDim2.new(0.05, 0, 0, 0)
toggleLabel.Size = UDim2.new(0.6, 0, 1, 0)
toggleLabel.Font = Enum.Font.GothamBold
toggleLabel.Text = "飞行模式"
toggleLabel.TextColor3 = Color3.fromRGB(220, 220, 230)
toggleLabel.TextSize = 11
toggleLabel.TextXAlignment = Enum.TextXAlignment.Left

local toggleBtn = Instance.new("TextButton")
toggleBtn.Parent = toggleFrame
toggleBtn.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
toggleBtn.Position = UDim2.new(0.82, 0, 0.15, 0)
toggleBtn.Size = UDim2.new(0, 40, 0, 24)
toggleBtn.Text = ""
Instance.new("UICorner", toggleBtn).CornerRadius = UDim.new(1, 0)

local toggleDot = Instance.new("Frame")
toggleDot.Parent = toggleBtn
toggleDot.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
toggleDot.Position = UDim2.new(0.08, 0, 0.1, 0)
toggleDot.Size = UDim2.new(0, 18, 0, 18)
Instance.new("UICorner", toggleDot).CornerRadius = UDim.new(1, 0)

local state = false
toggleBtn.MouseButton1Click:Connect(function()
    state = not state
    Flight.Enabled = state
    local targetColor = state and Color3.fromRGB(46, 204, 113) or Color3.fromRGB(45, 48, 60)
    local targetPos = state and UDim2.new(0.52, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0)
    TweenService:Create(toggleBtn, TweenInfo.new(0.2), {BackgroundColor3 = targetColor}):Play()
    TweenService:Create(toggleDot, TweenInfo.new(0.2), {Position = targetPos}):Play()
end)

-- 速度滑块
local sliderFrame = Instance.new("Frame")
sliderFrame.Parent = controlArea
sliderFrame.Position = UDim2.new(0, 0, 0, 50)
sliderFrame.Size = UDim2.new(1, 0, 0, 40)
sliderFrame.BackgroundColor3 = Color3.fromRGB(24, 26, 35)
Instance.new("UICorner", sliderFrame).CornerRadius = UDim.new(0, 6)

local sliderLabel = Instance.new("TextLabel")
sliderLabel.Parent = sliderFrame
sliderLabel.BackgroundTransparency = 1
sliderLabel.Position = UDim2.new(0.05, 0, 0, 0)
sliderLabel.Size = UDim2.new(0.5, 0, 1, 0)
sliderLabel.Font = Enum.Font.GothamBold
sliderLabel.Text = "飞行速度"
sliderLabel.TextColor3 = Color3.fromRGB(220, 220, 230)
sliderLabel.TextSize = 11
sliderLabel.TextXAlignment = Enum.TextXAlignment.Left

local sliderVal = Instance.new("TextLabel")
sliderVal.Parent = sliderFrame
sliderVal.BackgroundTransparency = 1
sliderVal.Position = UDim2.new(0.52, 0, 0, 0)
sliderVal.Size = UDim2.new(0.12, 0, 1, 0)
sliderVal.Font = Enum.Font.GothamBold
sliderVal.Text = tostring(Flight.Speed)
sliderVal.TextColor3 = Color3.fromRGB(80, 160, 255)
sliderVal.TextSize = 11

local sliderBg = Instance.new("Frame")
sliderBg.Parent = sliderFrame
sliderBg.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
sliderBg.Position = UDim2.new(0.66, 0, 0.4, 0)
sliderBg.Size = UDim2.new(0.3, 0, 0, 6)
Instance.new("UICorner", sliderBg).CornerRadius = UDim.new(1, 0)

local sliderFill = Instance.new("Frame")
sliderFill.Parent = sliderBg
sliderFill.BackgroundColor3 = Color3.fromRGB(80, 160, 255)
sliderFill.Size = UDim2.new(0.5, 0, 1, 0)
Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(1, 0)

local knob = Instance.new("TextButton")
knob.Parent = sliderBg
knob.Size = UDim2.new(0, 14, 0, 14)
knob.Position = UDim2.new(0.5, 0, 0.5, 0)
knob.AnchorPoint = Vector2.new(0.5, 0.5)
knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

local dragging = false
knob.MouseButton1Down:Connect(function() dragging = true end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging then
        local posX = input.Position.X - sliderBg.AbsolutePosition.X
        local pos = math.clamp(posX / sliderBg.AbsoluteSize.X, 0, 1)
        local val = math.floor(20 + (200 - 20) * pos)
        sliderVal.Text = tostring(val)
        sliderFill.Size = UDim2.new(pos, 0, 1, 0)
        knob.Position = UDim2.new(pos, 0, 0.5, 0)
        Flight.Speed = val
    end
end)

-- 升降按钮（完全集成在UI内）
local liftFrame = Instance.new("Frame")
liftFrame.Parent = controlArea
liftFrame.Position = UDim2.new(0, 0, 0, 100)
liftFrame.Size = UDim2.new(1, 0, 0, 50)
liftFrame.BackgroundColor3 = Color3.fromRGB(24, 26, 35)
Instance.new("UICorner", liftFrame).CornerRadius = UDim.new(0, 6)

local liftLabel = Instance.new("TextLabel")
liftLabel.Parent = liftFrame
liftLabel.BackgroundTransparency = 1
liftLabel.Position = UDim2.new(0.05, 0, 0, 0)
liftLabel.Size = UDim2.new(0.4, 0, 1, 0)
liftLabel.Font = Enum.Font.GothamBold
liftLabel.Text = "升降控制"
liftLabel.TextColor3 = Color3.fromRGB(220, 220, 230)
liftLabel.TextSize = 11
liftLabel.TextXAlignment = Enum.TextXAlignment.Left

local upBtn = Instance.new("TextButton")
upBtn.Parent = liftFrame
upBtn.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
upBtn.Position = UDim2.new(0.65, 0, 0.1, 0)
upBtn.Size = UDim2.new(0.15, 0, 0, 30)
upBtn.Font = Enum.Font.GothamBold
upBtn.Text = "▲"
upBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
upBtn.TextSize = 14
Instance.new("UICorner", upBtn).CornerRadius = UDim.new(0, 6)

local downBtn = Instance.new("TextButton")
downBtn.Parent = liftFrame
downBtn.BackgroundColor3 = Color3.fromRGB(231, 76, 60)
downBtn.Position = UDim2.new(0.82, 0, 0.1, 0)
downBtn.Size = UDim2.new(0.15, 0, 0, 30)
downBtn.Font = Enum.Font.GothamBold
downBtn.Text = "▼"
downBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
downBtn.TextSize = 14
Instance.new("UICorner", downBtn).CornerRadius = UDim.new(0, 6)

-- 长按事件绑定
upBtn.MouseButton1Down:Connect(function() Flight.Up = true end)
upBtn.MouseButton1Up:Connect(function() Flight.Up = false end)
upBtn.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then Flight.Up = false end
end)

downBtn.MouseButton1Down:Connect(function() Flight.Down = true end)
downBtn.MouseButton1Up:Connect(function() Flight.Down = false end)
downBtn.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then Flight.Down = false end
end)

-- ==========================================
-- 【飞行主循环】
-- ==========================================
renderConnection = RunService.RenderStepped:Connect(function()
    if not Flight.Enabled then return end
    if not character or not rootPart or not humanoid then return end
    if not rootPart.Parent then return end

    humanoid.PlatformStand = true

    local moveDir = humanoid.MoveDirection
    local velocity = Vector3.new(0, 0, 0)

    if moveDir.Magnitude > 0 then
        local rel = camera.CFrame:VectorToObjectSpace(moveDir)
        velocity = (camera.CFrame.LookVector * -rel.Z + camera.CFrame.RightVector * rel.X) * Flight.Speed
    end

    if Flight.Up then velocity = velocity + Vector3.new(0, Flight.Speed, 0) end
    if Flight.Down then velocity = velocity - Vector3.new(0, Flight.Speed, 0) end

    rootPart.AssemblyLinearVelocity = velocity
end)

print("✅【不偷懒飞行面板】加载成功！长按升降已修复。")
