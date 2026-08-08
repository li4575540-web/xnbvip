-- =====================================================================
-- 👑【小木HUB v7.14 · 绝对纯净复活版】
-- =====================================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local plr = Players.LocalPlayer
local camera = Workspace.CurrentCamera

local Flight = { Enabled = false, Speed = 60, Up = false, Down = false }
local FlyCar = { Enabled = false, Speed = 80, Up = false, Down = false }
local Speed = { Enabled = false, Value = 60 }
local Jump = { Enabled = false, Value = 50 }
local Misc = { AntiStun = true, ESP = true, Noclip = false }
local Aimbot = { Enabled = false, FOV = 120, Smooth = 0.7, TeamCheck = true }

local character, rootPart, humanoid
local espCache = {}
local currentTarget = nil
local renderConnection = nil
local noclipApplied = false
local tpTarget = nil
local screenGui = nil
local refreshPlayerListRunning = true

local carUpPressed = false
local carDownPressed = false

local function setupChar(char)
    character = char
    for _, v in pairs(espCache) do
        pcall(function()
            if v and v.Frame then v.Frame:Destroy() end
            if v and v.Label then v.Label:Destroy() end
        end)
    end
    espCache = {}
    Flight.Enabled = false
    FlyCar.Enabled = false
    Flight.Up = false
    Flight.Down = false
    FlyCar.Up = false
    FlyCar.Down = false
    carUpPressed = false
    carDownPressed = false
    Aimbot.Enabled = false
    currentTarget = nil
    Misc.ESP = false
    noclipApplied = false
    if currentWindow and currentWindow.main then currentWindow.main.Visible = false end
    task.spawn(function()
        rootPart = char:WaitForChild("HumanoidRootPart", 5)
        humanoid = char:WaitForChild("Humanoid", 5)
    end)
end

if plr.Character then setupChar(plr.Character) end
plr.CharacterAdded:Connect(setupChar)

local function cleanup()
    refreshPlayerListRunning = false
    if Flight.Enabled then
        Flight.Enabled = false
        pcall(function() if rootPart and rootPart.Parent then rootPart.AssemblyLinearVelocity = Vector3.new(0, 0, 0) end end)
    end
    FlyCar.Enabled = false
    FlyCar.Up = false
    FlyCar.Down = false
    Flight.Up = false
    Flight.Down = false
    carUpPressed = false
    carDownPressed = false
    if humanoid then pcall(function() humanoid.PlatformStand = false end) end
    if character then pcall(function() for _, p in ipairs(character:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = true end end end) end
    noclipApplied = false
    for _, v in pairs(espCache) do
        pcall(function()
            if v and v.Frame then v.Frame:Destroy() end
            if v and v.Label then v.Label:Destroy() end
        end)
    end
    espCache = {}
    if renderConnection then renderConnection:Disconnect(); renderConnection = nil end
    if screenGui then screenGui:Destroy(); screenGui = nil end
end

local UI = {}
local currentWindow = nil

function UI:CreateWindow(title)
    screenGui = Instance.new("ScreenGui")
    screenGui.Name = "XiaoMuUI"
    screenGui.ResetOnSpawn = false
    screenGui.IgnoreGuiInset = true
    screenGui.Parent = plr:WaitForChild("PlayerGui")
    local main = Instance.new("Frame")
    main.Parent = screenGui
    main.Size = UDim2.new(0, 520, 0, 400)
    main.Position = UDim2.new(0.5, 0, 0.5, 0)
    main.AnchorPoint = Vector2.new(0.5, 0.5)
    main.BackgroundColor3 = Color3.fromRGB(18, 20, 28)
    main.BackgroundTransparency = 0.1
    main.Visible = false
    Instance.new("UICorner", main).CornerRadius = UDim.new(0, 12)
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(130, 80, 255)
    stroke.Thickness = 2
    stroke.Parent = main
    local titleLbl = Instance.new("TextLabel")
    titleLbl.Parent = main
    titleLbl.BackgroundTransparency = 1
    titleLbl.Position = UDim2.new(0.04, 0, 0.03, 0)
    titleLbl.Size = UDim2.new(0, 120, 0, 30)
    titleLbl.Font = Enum.Font.GothamBold
    titleLbl.Text = title
    titleLbl.TextColor3 = Color3.fromRGB(220, 220, 255)
    titleLbl.TextSize = 16
    titleLbl.TextXAlignment = Enum.TextXAlignment.Left
    local closeBtn = Instance.new("TextButton")
    closeBtn.Parent = main
    closeBtn.BackgroundTransparency = 1
    closeBtn.Position = UDim2.new(0.94, 0, 0.02, 0)
    closeBtn.Size = UDim2.new(0, 30, 0, 30)
    closeBtn.Text = "✕"
    closeBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
    closeBtn.TextSize = 14
    closeBtn.MouseButton1Click:Connect(function() cleanup() end)
    local dragging = false
    local dragInput, mousePos, framePos
    titleLbl.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            mousePos = input.Position
            framePos = main.Position
        end
    end)
    titleLbl.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - mousePos
            local newX = math.clamp(framePos.X.Offset + delta.X, 0, camera.ViewportSize.X - main.AbsoluteSize.X)
            local newY = math.clamp(framePos.Y.Offset + delta.Y, 0, camera.ViewportSize.Y - main.AbsoluteSize.Y)
            main.Position = UDim2.new(framePos.X.Scale, newX, framePos.Y.Scale, newY)
        end
    end)
    local tabContainer = Instance.new("Frame")
    tabContainer.Parent = main
    tabContainer.BackgroundTransparency = 1
    tabContainer.Position = UDim2.new(0, 0, 0, 45)
    tabContainer.Size = UDim2.new(1, 0, 0, 35)
    local contentContainer = Instance.new("Frame")
    contentContainer.Parent = main
    contentContainer.BackgroundTransparency = 1
    contentContainer.Position = UDim2.new(0, 0, 0, 85)
    contentContainer.Size = UDim2.new(1, 0, 1, -85)
    local windowObj = { screenGui = screenGui, main = main, tabContainer = tabContainer, contentContainer = contentContainer, tabs = {}, currentTab = nil }
    currentWindow = windowObj
    return windowObj
end

function UI:CreateTab(windowObj, name, icon)
    local btn = Instance.new("TextButton")
    btn.Parent = windowObj.tabContainer
    btn.BackgroundTransparency = 1
    btn.Size = UDim2.new(0, 80, 1, 0)
    btn.Position = UDim2.new(0, (#windowObj.tabs * 85), 0, 0)
    btn.Text = icon .. " " .. name
    btn.TextColor3 = Color3.fromRGB(160, 160, 180)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 12
    local panel = Instance.new("ScrollingFrame")
    panel.Parent = windowObj.contentContainer
    panel.BackgroundTransparency = 1
    panel.Size = UDim2.new(1, 0, 1, 0)
    panel.CanvasSize = UDim2.new(0, 0, 0, 300)
    panel.ScrollBarThickness = 2
    panel.Visible = false
    local tabObj = { button = btn, panel = panel, yOffset = 10 }
    btn.MouseButton1Click:Connect(function()
        if windowObj.currentTab then
            windowObj.currentTab.button.TextColor3 = Color3.fromRGB(160, 160, 180)
            windowObj.currentTab.panel.Visible = false
        end
        btn.TextColor3 = Color3.fromRGB(46, 204, 113)
        panel.Visible = true
        windowObj.currentTab = tabObj
    end)
    table.insert(windowObj.tabs, tabObj)
    return tabObj
end

function UI:CreateGroupbox(tabObj, title)
    local group = Instance.new("Frame")
    group.Parent = tabObj.panel
    group.BackgroundColor3 = Color3.fromRGB(25, 27, 35)
    group.BackgroundTransparency = 0.3
    group.Size = UDim2.new(1, -10, 0, 0)
    group.Position = UDim2.new(0.05, 0, 0, 0)
    Instance.new("UICorner", group).CornerRadius = UDim.new(0, 6)
    local lbl = Instance.new("TextLabel")
    lbl.Parent = group
    lbl.BackgroundTransparency = 1
    lbl.Position = UDim2.new(0.05, 0, 0.05, 0)
    lbl.Size = UDim2.new(1, 0, 0, 20)
    lbl.Font = Enum.Font.GothamBold
    lbl.Text = title
    lbl.TextColor3 = Color3.fromRGB(200, 200, 200)
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    local layout = Instance.new("UIListLayout")
    layout.Parent = group
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Padding = UDim.new(0, 6)
    local pad = Instance.new("UIPadding")
    pad.Parent = group
    pad.PaddingTop = UDim.new(0, 30)
    local groupObj = { frame = group, yOffset = 0 }
    return groupObj
end

function UI:CreateToggle(groupObj, title, def, cb)
    local frame = Instance.new("Frame")
    frame.Parent = groupObj.frame
    frame.BackgroundColor3 = Color3.fromRGB(25, 27, 35)
    frame.Size = UDim2.new(0.9, 0, 0, 36)
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)
    local lbl = Instance.new("TextLabel")
    lbl.Parent = frame
    lbl.BackgroundTransparency = 1
    lbl.Position = UDim2.new(0.06, 0, 0, 0)
    lbl.Size = UDim2.new(0.6, 0, 1, 0)
    lbl.Font = Enum.Font.Gotham
    lbl.Text = title
    lbl.TextColor3 = Color3.fromRGB(230, 230, 240)
    lbl.TextSize = 13
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    local bg = Instance.new("TextButton")
    bg.Parent = frame
    bg.BackgroundColor3 = def and Color3.fromRGB(46, 204, 113) or Color3.fromRGB(50, 52, 65)
    bg.Position = UDim2.new(0.82, 0, 0.15, 0)
    bg.Size = UDim2.new(0, 38, 0, 24)
    bg.Text = ""
    Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)
    local dot = Instance.new("Frame")
    dot.Parent = bg
    dot.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    dot.Position = def and UDim2.new(0.52, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0)
    dot.Size = UDim2.new(0, 16, 0, 16)
    Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)
    local state = def
    bg.MouseButton1Click:Connect(function()
        state = not state
        bg.BackgroundColor3 = state and Color3.fromRGB(46, 204, 113) or Color3.fromRGB(50, 52, 65)
        dot.Position = state and UDim2.new(0.52, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0)
        cb(state)
    end)
    return frame
end

function UI:CreateSlider(groupObj, title, min, max, def, cb)
    local frame = Instance.new("Frame")
    frame.Parent = groupObj.frame
    frame.BackgroundColor3 = Color3.fromRGB(25, 27, 35)
    frame.Size = UDim2.new(0.9, 0, 0, 36)
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)
    local lbl = Instance.new("TextLabel")
    lbl.Parent = frame
    lbl.BackgroundTransparency = 1
    lbl.Position = UDim2.new(0.06, 0, 0, 0)
    lbl.Size = UDim2.new(0.5, 0, 1, 0)
    lbl.Font = Enum.Font.Gotham
    lbl.Text = title
    lbl.TextColor3 = Color3.fromRGB(230, 230, 240)
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    local val = Instance.new("TextLabel")
    val.Parent = frame
    val.BackgroundTransparency = 1
    val.Position = UDim2.new(0.52, 0, 0, 0)
    val.Size = UDim2.new(0.1, 0, 1, 0)
    val.Font = Enum.Font.GothamBold
    val.Text = tostring(def)
    val.TextColor3 = Color3.fromRGB(46, 204, 113)
    val.TextSize = 12
    local bar = Instance.new("Frame")
    bar.Parent = frame
    bar.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
    bar.Position = UDim2.new(0.64, 0, 0.35, 0)
    bar.Size = UDim2.new(0.32, 0, 0, 6)
    Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)
    local fill = Instance.new("Frame")
    fill.Parent = bar
    fill.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
    fill.Size = UDim2.new((def - min) / (max - min), 0, 1, 0)
    Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)
    local knob = Instance.new("TextButton")
    knob.Parent = bar
    knob.Size = UDim2.new(0, 16, 0, 16)
    knob.Position = UDim2.new((def - min) / (max - min), 0, 0.5, 0)
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
            local posX = input.Position.X - bar.AbsolutePosition.X
            local pos = math.clamp(posX / bar.AbsoluteSize.X, 0, 1)
            local v = math.floor(min + (max - min) * pos)
            val.Text = tostring(v)
            fill.Size = UDim2.new(pos, 0, 1, 0)
            knob.Position = UDim2.new(pos, 0, 0.5, 0)
            cb(v)
        end
    end)
    return frame
end

function UI:CreateButton(groupObj, title, cb)
    local btn = Instance.new("TextButton")
    btn.Parent = groupObj.frame
    btn.BackgroundColor3 = Color3.fromRGB(50, 52, 65)
    btn.Size = UDim2.new(0.9, 0, 0, 32)
    btn.Text = title
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.Gotham
    btn.TextSize = 12
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    btn.MouseButton1Click:Connect(function() cb() end)
    return btn
end

function UI:CreateLongPressButton(groupObj, title, onStart, onStop)
    local btn = Instance.new("TextButton")
    btn.Parent = groupObj.frame
    btn.BackgroundColor3 = Color3.fromRGB(50, 52, 65)
    btn.Size = UDim2.new(0.9, 0, 0, 32)
    btn.Text = title
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.Gotham
    btn.TextSize = 12
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    btn.MouseButton1Down:Connect(function() onStart() end)
    btn.MouseButton1Up:Connect(function() onStop() end)
    btn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then onStart() end
    end)
    btn.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then onStop() end
    end)
    return btn
end

function UI:CreateDropdown(groupObj, title, items, onSelect)
    local frame = Instance.new("Frame")
    frame.Parent = groupObj.frame
    frame.BackgroundColor3 = Color3.fromRGB(50, 52, 65)
    frame.Size = UDim2.new(0.9, 0, 0, 32)
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)
    local lbl = Instance.new("TextLabel")
    lbl.Parent = frame
    lbl.BackgroundTransparency = 1
    lbl.Size = UDim2.new(1, 0, 1, 0)
    lbl.Font = Enum.Font.Gotham
    lbl.Text = title
    lbl.TextColor3 = Color3.fromRGB(200, 200, 200)
    lbl.TextSize = 12
    lbl.Position = UDim2.new(0.05, 0, 0, 0)
    local dropdownObj = { frame = frame, label = lbl, items = items, onSelect = onSelect }
    function dropdownObj:SetItems(newItems) self.items = newItems end
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            local popup = Instance.new("Frame")
            popup.Parent = screenGui
            popup.Size = UDim2.new(0, 180, 0, math.min(#dropdownObj.items * 32, 200))
            popup.Position = UDim2.new(0.5, 0, 0.5, 0)
            popup.AnchorPoint = Vector2.new(0.5, 0.5)
            popup.BackgroundColor3 = Color3.fromRGB(30, 32, 40)
            Instance.new("UICorner", popup).CornerRadius = UDim.new(0, 6)
            local scroll = Instance.new("ScrollingFrame")
            scroll.Parent = popup
            scroll.Size = UDim2.new(1, 0, 1, 0)
            scroll.BackgroundTransparency = 1
            scroll.CanvasSize = UDim2.new(0, 0, 0, #dropdownObj.items * 32)
            local yPos = 0
            for _, item in ipairs(dropdownObj.items) do
                local opt = Instance.new("TextButton")
                opt.Parent = scroll
                opt.Size = UDim2.new(1, 0, 0, 32)
                opt.Position = UDim2.new(0, 0, 0, yPos)
                opt.Text = item
                opt.TextColor3 = Color3.fromRGB(255, 255, 255)
                opt.BackgroundColor3 = Color3.fromRGB(40, 42, 50)
                opt.Font = Enum.Font.Gotham
                opt.TextSize = 12
                opt.MouseButton1Click:Connect(function()
                    lbl.Text = item
                    if dropdownObj.onSelect then dropdownObj.onSelect(item) end
                    popup:Destroy()
                end)
                yPos = yPos + 32
            end
            local closeBtn = Instance.new("TextButton")
            closeBtn.Parent = popup
            closeBtn.Size = UDim2.new(0, 20, 0, 20)
            closeBtn.Position = UDim2.new(1, -25, 0, 5)
            closeBtn.Text = "✕"
            closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            closeBtn.BackgroundTransparency = 1
            closeBtn.MouseButton1Click:Connect(function() popup:Destroy() end)
            local connection = UserInputService.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                    if not popup:IsAncestorOf(input) then
                        popup:Destroy(); connection:Disconnect()
                    end
                end
            end)
        end
    end)
    return dropdownObj
end

local openBtn = Instance.new("TextButton")
openBtn.Parent = screenGui
openBtn.Size = UDim2.new(0, 55, 0, 55)
openBtn.Position = UDim2.new(0.08, 0, 0.5, 0)
openBtn.AnchorPoint = Vector2.new(0.5, 0.5)
openBtn.BackgroundColor3 = Color3.fromRGB(60, 50, 120)
openBtn.Text = "⚡"
openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
openBtn.TextSize = 26
Instance.new("UICorner", openBtn).CornerRadius = UDim.new(0, 16)
openBtn.MouseButton1Click:Connect(function()
    if currentWindow then currentWindow.main.Visible = not currentWindow.main.Visible end
end)

local window = UI:CreateWindow("小木HUB v7.14")
local tab1 = UI:CreateTab(window, "移动", "⚡")
local tab2 = UI:CreateTab(window, "战斗", "🎯")
local tab3 = UI:CreateTab(window, "视觉", "👁️")
local tab4 = UI:CreateTab(window, "飞车", "🚗")
local tab5 = UI:CreateTab(window, "传送", "🌀")
tab1.button.TextColor3 = Color3.fromRGB(46, 204, 113)
tab1.panel.Visible = true
window.currentTab = tab1

local g1 = UI:CreateGroupbox(tab1, "人物控制")
local g2 = UI:CreateGroupbox(tab2, "自瞄设置")
local g3 = UI:CreateGroupbox(tab3, "视觉辅助")
local g4 = UI:CreateGroupbox(tab4, "飞车系统")
local g5 = UI:CreateGroupbox(tab5, "传送操作")

UI:CreateToggle(g1, "速度加快", Speed.Enabled, function(v) Speed.Enabled = v end)
UI:CreateSlider(g1, "移动速度", 16, 200, Speed.Value, function(v) Speed.Value = v end)
UI:CreateToggle(g1, "跳跃加强", Jump.Enabled, function(v) Jump.Enabled = v end)
UI:CreateSlider(g1, "跳跃高度", 50, 300, Jump.Value, function(v) Jump.Value = v end)
UI:CreateToggle(g1, "穿墙", Misc.Noclip, function(v) Misc.Noclip = v; noclipApplied = false end)
UI:CreateToggle(g1, "人物飞行", Flight.Enabled, function(v) Flight.Enabled = v; if not v then cleanup() end end)

UI:CreateToggle(g2, "超维自瞄", Aimbot.Enabled, function(v) Aimbot.Enabled = v; if not v then currentTarget = nil end end)
UI:CreateToggle(g2, "战队保护", Aimbot.TeamCheck, function(v) Aimbot.TeamCheck = v end)
UI:CreateSlider(g2, "FOV范围", 50, 300, Aimbot.FOV, function(v) Aimbot.FOV = v end)
UI:CreateSlider(g2, "平滑度", 10, 100, Aimbot.Smooth * 100, function(v) Aimbot.Smooth = v / 100 end)

UI:CreateToggle(g3, "全图透视 ESP", Misc.ESP, function(v) Misc.ESP = v end)

UI:CreateToggle(g4, "飞车模式", FlyCar.Enabled, function(v) FlyCar.Enabled = v; FlyCar.Up = false; FlyCar.Down = false end)
UI:CreateSlider(g4, "飞车速度", 20, 200, FlyCar.Speed, function(v) FlyCar.Speed = v end)
UI:CreateLongPressButton(g4, "▲ 上升 (长按)", function() FlyCar.Up = true end, function() FlyCar.Up = false end)
UI:CreateLongPressButton(g4, "▼ 下降 (长按)", function() FlyCar.Down = true end, function() FlyCar.Down = false end)

local dropdown = UI:CreateDropdown(g5, "选择目标", {}, function(name)
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Name == name then tpTarget = p; break end
    end
end)

task.spawn(function()
    while refreshPlayerListRunning do
        local list = {}
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= plr then table.insert(list, p.Name) end
        end
        if dropdown then dropdown:SetItems(list) end
        task.wait(3)
    end
end)

UI:CreateButton(g5, "直接传送", function()
    if not tpTarget or not tpTarget.Character then return end
    local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart")
    if hrp and rootPart then rootPart.CFrame = hrp.CFrame end
end)
UI:CreateButton(g5, "前面传送", function()
    if not tpTarget or not tpTarget.Character then return end
    local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart")
    if hrp and rootPart then rootPart.CFrame = hrp.CFrame + hrp.CFrame.LookVector * 0.5 end
end)
UI:CreateButton(g5, "后面传送", function()
    if not tpTarget or not tpTarget.Character then return end
    local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart")
    if hrp and rootPart then rootPart.CFrame = hrp.CFrame - hrp.CFrame.LookVector * 0.5 end
end)
UI:CreateButton(g5, "💥 甩飞一次", function()
    if tpTarget and tpTarget.Character then
        local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart")
        if hrp then hrp.AssemblyLinearVelocity = Vector3.new(0, 120, 0) end
    end
end)

renderConnection = RunService.RenderStepped:Connect(function()
    if not currentWindow then return end
    if not humanoid or humanoid.Health <= 0 then return end
    if Aimbot.Enabled then
        local closestDist = Aimbot.FOV
        local closestTarget = nil
        local center = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= plr then
                if Aimbot.TeamCheck and p.Team == plr.Team then continue end
                local c = p.Character
                if c and c:FindFirstChild("HumanoidRootPart") and c:FindFirstChild("Humanoid") and c.Humanoid.Health > 0 then
                    local hrp = c.HumanoidRootPart
                    local s, on = camera:WorldToViewportPoint(hrp.Position)
                    if on then
                        local params = RaycastParams.new()
                        params.FilterDescendantsInstances = {plr.Character}
                        params.FilterType = Enum.RaycastFilterType.Exclude
                        local result = workspace:Raycast(camera.CFrame.Position, (hrp.Position - camera.CFrame.Position).Unit * 999, params)
                        if not result then
                            local d = (Vector2.new(s.X, s.Y) - center).Magnitude
                            if d < closestDist then closestDist = d; closestTarget = p end
                        end
                    end
                end
            end
        end
        if closestTarget then
            currentTarget = closestTarget
            local c = closestTarget.Character
            if c and c:FindFirstChild("HumanoidRootPart") then
                local tPos = c.HumanoidRootPart.Position
                local lDir = (tPos - camera.CFrame.Position).Unit
                camera.CFrame = camera.CFrame:Lerp(CFrame.lookAt(camera.CFrame.Position, camera.CFrame.Position + lDir), Aimbot.Smooth)
            end
        else
            currentTarget = nil
        end
    end
    if Misc.ESP then
        for userId, v in pairs(espCache) do
            local p = Players:GetPlayerByUserId(userId)
            if not p then
                pcall(function()
                    if v and v.Frame then v.Frame:Destroy() end
                    if v and v.Label then v.Label:Destroy() end
                end)
                espCache[userId] = nil
            end
        end
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= plr and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                local hrp = p.Character.HumanoidRootPart
                local s, on = camera:WorldToViewportPoint(hrp.Position)
                if on and s.Z > 0 and s.Z < 1000 then
                    local size = 50 + (200 - s.Z) * 0.3
                    if not espCache[p.UserId] then
                        local f = Instance.new("Frame"); f.Parent = screenGui; f.BackgroundTransparency = 1; f.Size = UDim2.new(0, size, 0, size*2); f.Position = UDim2.new(0, s.X - size/2, 0, s.Y - size); f.ZIndex = 20; local st = Instance.new("UIStroke"); st.Color = Color3.fromRGB(255,255,255); st.Thickness = 1; st.Parent = f; local l = Instance.new("TextLabel"); l.Parent = f; l.BackgroundTransparency = 1; l.Size = UDim2.new(0, 100, 0, 16); l.Position = UDim2.new(0, -25, 0, -20); l.Text = p.Name; l.TextColor3 = Color3.fromRGB(255,255,255); l.TextSize = 10; l.ZIndex = 21; espCache[p.UserId] = {Frame=f, Label=l}
                    else
                        local e = espCache[p.UserId]; e.Frame.Size = UDim2.new(0, size, 0, size*2); e.Frame.Position = UDim2.new(0, s.X - size/2, 0, s.Y - size); e.Frame.Visible = true; e.Label.Visible = true
                    end
                else
                    if espCache[p.UserId] then espCache[p.UserId].Frame.Visible = false; espCache[p.UserId].Label.Visible = false end
                end
            end
        end
    else
        for _, v in pairs(espCache) do
            pcall(function()
                if v and v.Frame then v.Frame:Destroy() end
                if v and v.Label then v.Label:Destroy() end
            end)
        end
        espCache = {}
    end
    if FlyCar.Enabled then
        local seat = plr.Character and (plr.Character:FindFirstChild("Seat") or plr.Character:FindFirstChild("VehicleSeat"))
        if seat and seat.Occupant and seat.Occupant.Parent == plr.Character then
            local vr = seat.Parent:FindFirstChild("VehicleRootPart") or seat.Parent:FindFirstChild("PrimaryPart") or seat.Parent:FindFirstChild("Body")
            if vr then
                local v = Vector3.new(0,0,0)
                local h = plr.Character:FindFirstChild("Humanoid")
                if h then
                    local d = h.MoveDirection
                    if d.Magnitude > 0.1 then
                        local r = camera.CFrame:VectorToObjectSpace(d)
                        v = (camera.CFrame.LookVector * -r.Z + camera.CFrame.RightVector * r.X) * FlyCar.Speed
                    end
                end
                if FlyCar.Up then v = v + Vector3.new(0, FlyCar.Speed, 0) end
                if FlyCar.Down then v = v - Vector3.new(0, FlyCar.Speed, 0) end
                vr.AssemblyLinearVelocity = v
            end
        end
    end
    pcall(function()
        if character and humanoid and humanoid.Parent then
            if Speed.Enabled then humanoid.WalkSpeed = Speed.Value
            elseif humanoid.WalkSpeed ~= 16 then humanoid.WalkSpeed = 16 end
            if Jump.Enabled then humanoid.JumpPower = Jump.Value; humanoid.UseJumpPower = true
            else humanoid.UseJumpPower = false; humanoid.JumpPower = 50 end
            if Misc.AntiStun and humanoid.PlatformStand and not Flight.Enabled then humanoid.PlatformStand = false end
        end
    end)
    if Misc.Noclip and not noclipApplied and character then
        pcall(function() for _, p in ipairs(character:GetDescendants()) do if p:IsA("BasePart") and p.Name ~= "HumanoidRootPart" then p.CanCollide = false end end; noclipApplied = true end)
    elseif not Misc.Noclip and noclipApplied and character then
        pcall(function() for _, p in ipairs(character:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = true end end; noclipApplied = false end)
    end
    if Flight.Enabled then
        pcall(function()
            if not character or not rootPart or not humanoid then return end
            humanoid.PlatformStand = true
            local moveDir = humanoid.MoveDirection
            local velocity = Vector3.new(0,0,0)
            if moveDir.Magnitude > 0 then
                local rel = camera.CFrame:VectorToObjectSpace(moveDir)
                velocity = (camera.CFrame.LookVector * -rel.Z + camera.CFrame.RightVector * rel.X) * Flight.Speed
            end
            if Flight.Up then velocity = velocity + Vector3.new(0, Flight.Speed, 0) end
            if Flight.Down then velocity = velocity - Vector3.new(0, Flight.Speed, 0) end
            rootPart.AssemblyLinearVelocity = velocity
        end)
    end
end)

print("✅【小木HUB v7.14】已复活！")
