-- ==========================================
-- ✈️【小飞HUB v2 - 防倒 + 防自转 + 易点版】
-- ==========================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")

local plr = Players.LocalPlayer
local camera = Workspace.CurrentCamera

local Flight = { Enabled = false, Speed = 60, Up = false, Down = false }
local character, rootPart, humanoid
local renderConnection = nil

local function setupChar(char)
    character = char
    task.spawn(function()
        rootPart = char:WaitForChild("HumanoidRootPart", 5)
        humanoid = char:WaitForChild("Humanoid", 5)
    end)
end

if plr.Character then setupChar(plr.Character) end
plr.CharacterAdded:Connect(setupChar)

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "XiaoFeiHub"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
pcall(function() screenGui.Parent = CoreGui end)
if not screenGui.Parent then screenGui.Parent = plr:WaitForChild("PlayerGui") end

-- ========== 主窗口 ==========
local mainFrame = Instance.new("Frame")
mainFrame.Parent = screenGui
mainFrame.Size = UDim2.new(0, 280, 0, 160)
mainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
mainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
mainFrame.BackgroundColor3 = Color3.fromRGB(16, 17, 22)
mainFrame.Visible = true
mainFrame.Active = true
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 10)
local mainStroke = Instance.new("UIStroke")
mainStroke.Color = Color3.fromRGB(80, 160, 255)
mainStroke.Thickness = 1.5
mainStroke.Parent = mainFrame

-- ========== 顶部标题栏 ==========
local titleBar = Instance.new("Frame")
titleBar.Parent = mainFrame
titleBar.Size = UDim2.new(1, 0, 0, 30)
titleBar.BackgroundColor3 = Color3.fromRGB(18, 19, 26)
titleBar.BackgroundTransparency = 0.3
Instance.new("UICorner", titleBar).CornerRadius = UDim.new(0, 10)

local title = Instance.new("TextLabel")
title.Parent = titleBar
title.BackgroundTransparency = 1
title.Position = UDim2.new(0.05, 0, 0, 0)
title.Size = UDim2.new(0.6, 0, 1, 0)
title.Font = Enum.Font.GothamBold
title.Text = "✈️ 飞行控制"
title.TextColor3 = Color3.fromRGB(80, 160, 255)
title.TextSize = 13
title.TextXAlignment = Enum.TextXAlignment.Left

-- ========== 修复：放大关闭和最小化按钮 ==========
local closeBtn = Instance.new("TextButton")
closeBtn.Parent = titleBar
closeBtn.BackgroundTransparency = 1
closeBtn.Position = UDim2.new(1, -36, 0, 0) -- 放大到 30x30
closeBtn.Size = UDim2.new(0, 30, 0, 30)
closeBtn.Font = Enum.Font.GothamBold
closeBtn.Text = "✕"
closeBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
closeBtn.TextSize = 16
closeBtn.MouseButton1Click:Connect(function()
    Flight.Enabled = false
    if renderConnection then renderConnection:Disconnect() end
    screenGui:Destroy()
end)

local miniBtn = Instance.new("TextButton")
miniBtn.Parent = titleBar
miniBtn.BackgroundTransparency = 1
miniBtn.Position = UDim2.new(0.9, 0, 0, 0)
miniBtn.Size = UDim2.new(0, 30, 0, 30)
miniBtn.Font = Enum.Font.GothamBold
miniBtn.Text = "-"
miniBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
miniBtn.TextSize = 16

-- ========== 悬浮小飞机 ==========
local miniSquare = Instance.new("TextButton")
miniSquare.Name = "MiniSquare"
miniSquare.Parent = screenGui
miniSquare.Size = UDim2.new(0, 50, 0, 50)
miniSquare.Position = UDim2.new(0.02, 0, 0.2, 0)
miniSquare.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
miniSquare.Text = "✈️"
miniSquare.TextColor3 = Color3.fromRGB(40, 40, 50)
miniSquare.TextSize = 22
miniSquare.Visible = false
miniSquare.Active = true
miniSquare.Draggable = true
Instance.new("UICorner", miniSquare).CornerRadius = UDim.new(1, 0)

local miniStroke = Instance.new("UIStroke")
miniStroke.Color = Color3.fromRGB(80, 160, 255)
miniStroke.Thickness = 2
miniStroke.Parent = miniSquare

local shadow = Instance.new("Frame")
shadow.Name = "Shadow"
shadow.Parent = miniSquare
shadow.Size = UDim2.new(1, 0, 1, 0)
shadow.Position = UDim2.new(0, 0, 0.04, 0)
shadow.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
shadow.BackgroundTransparency = 0.5
shadow.ZIndex = -1
Instance.new("UICorner", shadow).CornerRadius = UDim.new(1, 0)

miniBtn.MouseButton1Click:Connect(function()
    mainFrame.Visible = false
    miniSquare.Visible = true
end)

miniSquare.MouseButton1Click:Connect(function()
    miniSquare.Visible = false
    mainFrame.Visible = true
    TweenService:Create(mainFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {Size = UDim2.new(0, 300, 0, 170)}):Play()
    task.wait(0.2)
    TweenService:Create(mainFrame, TweenInfo.new(0.2), {Size = UDim2.new(0, 280, 0, 160)}):Play()
end)

-- ========== 可拖拽逻辑 ==========
local dragging = false
local dragInput, mousePos, framePos

titleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        mousePos = input.Position
        framePos = mainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)

titleBar.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - mousePos
        mainFrame.Position = UDim2.new(
            framePos.X.Scale,
            framePos.X.Offset + delta.X,
            framePos.Y.Scale,
            framePos.Y.Offset + delta.Y
        )
    end
end)

-- ========== 控制区 ==========
local controlArea = Instance.new("Frame")
controlArea.Parent = mainFrame
controlArea.BackgroundTransparency = 1
controlArea.Position = UDim2.new(0.05, 0, 0.3, 0)
controlArea.Size = UDim2.new(0.9, 0, 0.65, 0)

local toggleFrame = Instance.new("Frame")
toggleFrame.Parent = controlArea
toggleFrame.Size = UDim2.new(1, 0, 0, 36)
toggleFrame.BackgroundColor3 = Color3.fromRGB(24, 26, 35)
Instance.new("UICorner", toggleFrame).CornerRadius = UDim.new(0, 6)

local toggleLabel = Instance.new("TextLabel")
toggleLabel.Parent = toggleFrame
toggleLabel.BackgroundTransparency = 1
toggleLabel.Position = UDim2.new(0.05, 0, 0, 0)
toggleLabel.Size = UDim2.new(0.6, 0, 1, 0)
toggleLabel.Font = Enum.Font.GothamBold
toggleLabel.Text = "人物飞行"
toggleLabel.TextColor3 = Color3.fromRGB(220, 220, 230)
toggleLabel.TextSize = 11
toggleLabel.TextXAlignment = Enum.TextXAlignment.Left

local toggleBtn = Instance.new("TextButton")
toggleBtn.Parent = toggleFrame
toggleBtn.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
toggleBtn.Position = UDim2.new(0.82, 0, 0.15, 0)
toggleBtn.Size = UDim2.new(0, 36, 0, 20)
toggleBtn.Text = ""
Instance.new("UICorner", toggleBtn).CornerRadius = UDim.new(1, 0)

local toggleDot = Instance.new("Frame")
toggleDot.Parent = toggleBtn
toggleDot.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
toggleDot.Position = UDim2.new(0.08, 0, 0.1, 0)
toggleDot.Size = UDim2.new(0, 14, 0, 14)
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
sliderFrame.Position = UDim2.new(0, 0, 0, 44)
sliderFrame.Size = UDim2.new(1, 0, 0, 36)
sliderFrame.BackgroundColor3 = Color3.fromRGB(24, 26, 35)
Instance.new("UICorner", sliderFrame).CornerRadius = UDim.new(0, 6)

local sliderLabel = Instance.new("TextLabel")
sliderLabel.Parent = sliderFrame
sliderLabel.BackgroundTransparency = 1
sliderLabel.Position = UDim2.new(0.05, 0, 0, 0)
sliderLabel.Size = UDim2.new(0.4, 0, 1, 0)
sliderLabel.Font = Enum.Font.GothamBold
sliderLabel.Text = "飞行速度"
sliderLabel.TextColor3 = Color3.fromRGB(220, 220, 230)
sliderLabel.TextSize = 11
sliderLabel.TextXAlignment = Enum.TextXAlignment.Left

local sliderVal = Instance.new("TextLabel")
sliderVal.Parent = sliderFrame
sliderVal.BackgroundTransparency = 1
sliderVal.Position = UDim2.new(0.52, 0, 0, 0)
sliderVal.Size = UDim2.new(0.1, 0, 1, 0)
sliderVal.Font = Enum.Font.GothamBold
sliderVal.Text = tostring(Flight.Speed)
sliderVal.TextColor3 = Color3.fromRGB(80, 160, 255)
sliderVal.TextSize = 11

local sliderBg = Instance.new("Frame")
sliderBg.Parent = sliderFrame
sliderBg.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
sliderBg.Position = UDim2.new(0.66, 0, 0.35, 0)
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

-- 升降按钮
local liftFrame = Instance.new("Frame")
liftFrame.Parent = controlArea
liftFrame.Position = UDim2.new(0, 0, 0, 88)
liftFrame.Size = UDim2.new(1, 0, 0, 36)
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
upBtn.Position = UDim2.new(0.65, 0, 0.05, 0)
upBtn.Size = UDim2.new(0.12, 0, 0, 28)
upBtn.Font = Enum.Font.GothamBold
upBtn.Text = "▲"
upBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
upBtn.TextSize = 14
Instance.new("UICorner", upBtn).CornerRadius = UDim.new(0, 6)

local downBtn = Instance.new("TextButton")
downBtn.Parent = liftFrame
downBtn.BackgroundColor3 = Color3.fromRGB(231, 76, 60)
downBtn.Position = UDim2.new(0.79, 0, 0.05, 0)
downBtn.Size = UDim2.new(0.12, 0, 0, 28)
downBtn.Font = Enum.Font.GothamBold
downBtn.Text = "▼"
downBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
downBtn.TextSize = 14
Instance.new("UICorner", downBtn).CornerRadius = UDim.new(0, 6)

local function bindHold(btn, setter)
    btn.MouseButton1Down:Connect(function() setter(true) end)
    btn.MouseButton1Up:Connect(function() setter(false) end)
    btn.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then setter(false) end
    end)
end
bindHold(upBtn, function(s) Flight.Up = s end)
bindHold(downBtn, function(s) Flight.Down = s end)

-- ========== 飞行主循环（修复版） ==========
renderConnection = RunService.RenderStepped:Connect(function()
    if not Flight.Enabled then return end
    if not character or not rootPart or not humanoid then return end
    if not rootPart.Parent then return end

    humanoid.PlatformStand = true
    
    -- 🔥 修复：每帧强制把人物拉正，防止倒地
    rootPart.CFrame = CFrame.new(rootPart.Position)
    
    local moveDir = humanoid.MoveDirection
    local velocity = Vector3.new(0, 0, 0)

    if moveDir.Magnitude > 0 then
        local rel = camera.CFrame:VectorToObjectSpace(moveDir)
        velocity = (camera.CFrame.LookVector * -rel.Z + camera.CFrame.RightVector * rel.X) * Flight.Speed
    end

    if Flight.Up then velocity = velocity + Vector3.new(0, Flight.Speed, 0) end
    if Flight.Down then velocity = velocity - Vector3.new(0, Flight.Speed, 0) end

    -- 🔥 修复：每次移动时，强制锁定角度，防止撞墙后乱旋转
    local currentPos = rootPart.Position
    rootPart.AssemblyLinearVelocity = velocity
    rootPart.CFrame = CFrame.new(currentPos + velocity * 0.1)
end)

print("✅【小飞HUB v2稳定版】加载成功！")
