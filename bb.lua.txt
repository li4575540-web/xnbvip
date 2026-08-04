-- =====================================================================
-- 👑【小木HUB v6.8 - 稳定全功能直通车版】
-- =====================================================================
--
-- 📋 【更新日志】
-- 
-- v6.8 (2026-08-04)
-- 🔧 核心循环重构：彻底重写 RunService.RenderStepped 主循环，将自瞄、飞车、飞行、ESP全部整合进同一帧循环，
--    彻底修复“开关有效但功能不动”的问题。
-- ⚡ 执行逻辑解耦：移除了过度依赖 pcall 的冗余包裹，让底层功能直接挂钩游戏物理引擎，响应速度更快。
-- 🛡️ 模块独立加载：将功能逻辑与UI彻底分离，确保开启某个功能时，对应的代码块独立运行，互不干扰。
-- 📱 移动端适配：所有滑动条、按钮、长按事件完全兼容触摸屏操作。
-- 🎨 WindowUI 窗口：带有标题栏、拖拽功能和右上角关闭按钮，视觉更贴近桌面窗口。
-- ⚡ 快速启动按钮：屏幕左侧新增蓝色 ⚡ 悬浮按钮，无需寻找菜单，点一下就展开主界面。
--
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
-- 【核心状态】
-- ==========================================
local Flight = { Enabled = false, Speed = 60, Up = false, Down = false }
local FlyCar = { Enabled = false, Speed = 80 }
local Speed = { Enabled = false, Value = 60 }
local Jump = { Enabled = false, Value = 50 }
local Misc = { AntiStun = true, ESP = true, Noclip = false }
local Aimbot = { Enabled = false, FOV = 120, TeamCheck = true, AimPart = "UpperTorso" }
local character, rootPart, humanoid
local espCache = {}
local currentTarget = nil
local renderConnection = nil
local carUpP, carDownP = false, false
local noclipApplied = false

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
-- 【清理】
-- ==========================================
local function cleanup()
    Flight.Enabled = false; FlyCar.Enabled = false
    if humanoid then pcall(function() humanoid.PlatformStand = false end) end
    if character then pcall(function() for _, p in ipairs(character:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = true end end end) end
    noclipApplied = false
    for _, v in pairs(espCache) do pcall(function() v:Destroy() end) end
    espCache = {}
end

-- ==========================================
-- 【UI 构建】
-- ==========================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "XiaoMuHubV2"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
pcall(function() screenGui.Parent = CoreGui end)
if not screenGui.Parent then screenGui.Parent = plr:WaitForChild("PlayerGui") end

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Parent = screenGui
mainFrame.Size = UDim2.new(0, 520, 0, 400)
mainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
mainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
mainFrame.BackgroundColor3 = Color3.fromRGB(20, 22, 30)
mainFrame.Visible = false
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 8)
local mainStroke = Instance.new("UIStroke"); mainStroke.Color = Color3.fromRGB(45, 48, 60); mainStroke.Thickness = 1.5; mainStroke.Parent = mainFrame

local titleBar = Instance.new("Frame")
titleBar.Parent = mainFrame
titleBar.Size = UDim2.new(1, 0, 0, 40)
titleBar.BackgroundColor3 = Color3.fromRGB(28, 30, 40)
Instance.new("UICorner", titleBar).CornerRadius = UDim.new(0, 8)
local titleText = Instance.new("TextLabel"); titleText.Parent = titleBar; titleText.Size = UDim2.new(0.6, 0, 1, 0); titleText.Position = UDim2.new(0.05, 0, 0, 0); titleText.BackgroundTransparency = 1; titleText.Font = Enum.Font.GothamBold; titleText.Text = " 小木HUB v6.8"; titleText.TextColor3 = Color3.fromRGB(80, 160, 255); titleText.TextSize = 16; titleText.TextXAlignment = Enum.TextXAlignment.Left
local closeBtn = Instance.new("TextButton"); closeBtn.Parent = titleBar; closeBtn.Size = UDim2.new(0, 30, 0, 30); closeBtn.Position = UDim2.new(1, -36, 0, 5); closeBtn.BackgroundTransparency = 1; closeBtn.Text = "✕"; closeBtn.TextColor3 = Color3.fromRGB(180, 180, 180); closeBtn.TextSize = 14
closeBtn.MouseButton1Click:Connect(function() cleanup(); if renderConnection then renderConnection:Disconnect() end; screenGui:Destroy() end)

-- ==========================================
-- 【UI组件】
-- ==========================================
local function createToggle(parent, y, txt, init, cb)
    local card = Instance.new("Frame"); card.Parent = parent; card.BackgroundColor3 = Color3.fromRGB(24, 26, 35); card.Position = UDim2.new(0, 0, 0, y); card.Size = UDim2.new(1, 0, 0, 40); Instance.new("UICorner", card).CornerRadius = UDim.new(0, 6)
    local lbl = Instance.new("TextLabel"); lbl.Parent = card; lbl.BackgroundTransparency = 1; lbl.Position = UDim2.new(0.05, 0, 0, 0); lbl.Size = UDim2.new(0.6, 0, 1, 0); lbl.Font = Enum.Font.GothamBold; lbl.Text = txt; lbl.TextColor3 = Color3.fromRGB(220, 220, 230); lbl.TextSize = 12; lbl.TextXAlignment = Enum.TextXAlignment.Left
    local bg = Instance.new("TextButton"); bg.Parent = card; bg.BackgroundColor3 = init and Color3.fromRGB(46, 204, 113) or Color3.fromRGB(45, 48, 60); bg.Position = UDim2.new(0.82, 0, 0.2, 0); bg.Size = UDim2.new(0, 40, 0, 22); bg.Text = ""; Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)
    local dot = Instance.new("Frame"); dot.Parent = bg; dot.BackgroundColor3 = Color3.fromRGB(255, 255, 255); dot.Position = init and UDim2.new(0.52, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0); dot.Size = UDim2.new(0, 16, 0, 16); Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)
    local state = init
    bg.MouseButton1Click:Connect(function() state = not state; local c = state and Color3.fromRGB(46, 204, 113) or Color3.fromRGB(45, 48, 60); local p = state and UDim2.new(0.52, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0); TweenService:Create(bg, TweenInfo.new(0.2), {BackgroundColor3 = c}):Play(); TweenService:Create(dot, TweenInfo.new(0.2), {Position = p}):Play(); cb(state) end)
end

local function createSlider(parent, y, txt, mn, mx, cur, unit, cb)
    local card = Instance.new("Frame"); card.Parent = parent; card.BackgroundColor3 = Color3.fromRGB(24, 26, 35); card.Position = UDim2.new(0, 0, 0, y); card.Size = UDim2.new(1, 0, 0, 40); Instance.new("UICorner", card).CornerRadius = UDim.new(0, 6)
    local lbl = Instance.new("TextLabel"); lbl.Parent = card; lbl.BackgroundTransparency = 1; lbl.Position = UDim2.new(0.05, 0, 0, 0); lbl.Size = UDim2.new(0.6, 0, 1, 0); lbl.Font = Enum.Font.GothamBold; lbl.Text = txt; lbl.TextColor3 = Color3.fromRGB(220, 220, 230); lbl.TextSize = 12; lbl.TextXAlignment = Enum.TextXAlignment.Left
    local val = Instance.new("TextLabel"); val.Parent = card; val.BackgroundTransparency = 1; val.Position = UDim2.new(0.52, 0, 0, 0); val.Size = UDim2.new(0.12, 0, 1, 0); val.Font = Enum.Font.GothamBold; val.Text = tostring(cur); val.TextColor3 = Color3.fromRGB(80, 160, 255); val.TextSize = 12
    local bar = Instance.new("Frame"); bar.Parent = card; bar.BackgroundColor3 = Color3.fromRGB(45, 48, 60); bar.Position = UDim2.new(0.66, 0, 0.45, 0); bar.Size = UDim2.new(0.3, 0, 0, 6); Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)
    local fill = Instance.new("Frame"); fill.Parent = bar; fill.BackgroundColor3 = Color3.fromRGB(80, 160, 255); fill.Size = UDim2.new((cur - mn)/(mx - mn), 0, 1, 0); Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)
    local knob = Instance.new("Frame"); knob.Parent = bar; knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255); knob.AnchorPoint = Vector2.new(0.5, 0.5); knob.Position = UDim2.new((cur - mn)/(mx - mn), 0, 0.5, 0); knob.Size = UDim2.new(0, 14, 0, 14); Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)
    local dragging = false
    knob.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = true end end)
    UserInputService.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end end)
    UserInputService.InputChanged:Connect(function(input) if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then local posX = input.Position.X - bar.AbsolutePosition.X; local pos = math.clamp(posX / bar.AbsoluteSize.X, 0, 1); local v = math.floor(mn + (mx - mn) * pos); val.Text = tostring(v); fill.Size = UDim2.new(pos, 0, 1, 0); knob.Position = UDim2.new(pos, 0, 0.5, 0); cb(v) end end)
end

-- ==========================================
-- 【分类面板】
-- ==========================================
local tabScroll = Instance.new("ScrollingFrame"); tabScroll.Parent = mainFrame; tabScroll.BackgroundTransparency = 1; tabScroll.Position = UDim2.new(0.05, 0, 0.12, 0); tabScroll.Size = UDim2.new(0.9, 0, 0, 40); tabScroll.ScrollBarThickness = 0; tabScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
local currentTab = "移动"; local tabContentFrames = {}
local function createTabBtn(txt, icon)
    local btn = Instance.new("TextButton"); btn.Parent = tabScroll; btn.BackgroundTransparency = 1; btn.Size = UDim2.new(0, 80, 1, 0); btn.Text = icon .. txt; btn.TextColor3 = Color3.fromRGB(160, 160, 180); btn.Font = Enum.Font.GothamBold; btn.TextSize = 13; Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8); return btn
end
local function createContentPanel() local sf = Instance.new("ScrollingFrame"); sf.Parent = mainFrame; sf.BackgroundTransparency = 1; sf.Position = UDim2.new(0.05, 0, 0.20, 0); sf.Size = UDim2.new(0.9, 0, 0.70, 0); sf.CanvasSize = UDim2.new(0, 0, 0, 350); sf.ScrollBarThickness = 3; sf.Visible = false; return sf end

local tabBtns = {}
local function setupTabs()
    local names = {"移动", "战斗", "视觉", "飞车", "传送"}
    local icons = {"⚡", "🎯", "👁️", "🚗", "🌀"}
    local posX = 0
    for i, name in ipairs(names) do
        local btn = createTabBtn(name, icons[i]); btn.Position = UDim2.new(0, posX, 0, 0); posX = posX + 90; tabBtns[name] = btn
        local panel = createContentPanel(); panel.Name = name .. "Panel"; tabContentFrames[name] = panel
        btn.MouseButton1Click:Connect(function() currentTab = name; for _, b in pairs(tabBtns) do b.TextColor3 = Color3.fromRGB(160, 160, 180) end; btn.TextColor3 = Color3.fromRGB(80, 160, 255); for n, p in pairs(tabContentFrames) do p.Visible = (n == name) end end)
    end
    tabScroll.CanvasSize = UDim2.new(0, posX, 0, 0)
    if tabBtns["移动"] then tabBtns["移动"].TextColor3 = Color3.fromRGB(80, 160, 255) end
    if tabContentFrames["移动Panel"] then tabContentFrames["移动Panel"].Visible = true end
end
setupTabs()

-- ==========================================
-- 【功能填充 - 完整版】
-- ==========================================
local mP = tabContentFrames["移动Panel"]
createToggle(mP, 0, "速度加快", Speed.Enabled, function(s) Speed.Enabled = s end)
createSlider(mP, 50, "移动速度", 16, 200, Speed.Value, "速", function(v) Speed.Value = v end)
createToggle(mP, 100, "跳跃加强", Jump.Enabled, function(s) Jump.Enabled = s end)
createSlider(mP, 150, "跳跃高度", 50, 300, Jump.Value, "高", function(v) Jump.Value = v end)
createToggle(mP, 200, "穿墙 (Noclip)", Misc.Noclip, function(s) Misc.Noclip = s; noclipApplied = false end)
createToggle(mP, 250, "人物飞行", Flight.Enabled, function(s) Flight.Enabled = s; if not s then cleanup() end)

local bP = tabContentFrames["战斗Panel"]
createToggle(bP, 0, "超维自瞄", Aimbot.Enabled, function(s) Aimbot.Enabled = s; if not s then currentTarget = nil end)
createToggle(bP, 50, "战队保护", Aimbot.TeamCheck, function(s) Aimbot.TeamCheck = s end)
createSlider(bP, 100, "FOV范围", 50, 300, Aimbot.FOV, "度", function(v) Aimbot.FOV = v end)

local vP = tabContentFrames["视觉Panel"]
createToggle(vP, 0, "全图透视 ESP", Misc.ESP, function(s) Misc.ESP = s end)

local cP = tabContentFrames["飞车Panel"]
createToggle(cP, 0, "🚗 飞车模式", FlyCar.Enabled, function(s) FlyCar.Enabled = s; if not s then carUpP = false; carDownP = false end end)
createSlider(cP, 50, "飞车速度", 20, 200, FlyCar.Speed, "速", function(v) FlyCar.Speed = v end)
local carCtrl = Instance.new("Frame"); carCtrl.Parent = cP; carCtrl.BackgroundColor3 = Color3.fromRGB(24, 26, 35); carCtrl.Position = UDim2.new(0, 0, 0, 100); carCtrl.Size = UDim2.new(1, 0, 0, 50); Instance.new("UICorner", carCtrl).CornerRadius = UDim.new(0, 6)
local carLbl = Instance.new("TextLabel"); carLbl.Parent = carCtrl; carLbl.BackgroundTransparency = 1; carLbl.Position = UDim2.new(0.05, 0, 0.1, 0); carLbl.Size = UDim2.new(0.5, 0, 1, 0); carLbl.Font = Enum.Font.GothamBold; carLbl.Text = "飞车升降"; carLbl.TextColor3 = Color3.fromRGB(220, 220, 230); carLbl.TextSize = 12
local carUp = Instance.new("TextButton"); carUp.Parent = carCtrl; carUp.BackgroundColor3 = Color3.fromRGB(46, 204, 113); carUp.Position = UDim2.new(0.65, 0, 0.1, 0); carUp.Size = UDim2.new(0.15, 0, 0, 32); carUp.Text = "▲"; carUp.TextColor3 = Color3.fromRGB(255, 255, 255); carUp.TextSize = 14; Instance.new("UICorner", carUp).CornerRadius = UDim.new(0, 4)
local carDown = Instance.new("TextButton"); carDown.Parent = carCtrl; carDown.BackgroundColor3 = Color3.fromRGB(231, 76, 60); carDown.Position = UDim2.new(0.82, 0, 0.1, 0); carDown.Size = UDim2.new(0.15, 0, 0, 32); carDown.Text = "▼"; carDown.TextColor3 = Color3.fromRGB(255, 255, 255); carDown.TextSize = 14; Instance.new("UICorner", carDown).CornerRadius = UDim.new(0, 4)
local function bindCar(btn, setter) btn.MouseButton1Down:Connect(function() setter(true) end); btn.MouseButton1Up:Connect(function() setter(false) end); btn.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch then setter(false) end end) end
bindCar(carUp, function(s) carUpP = s end); bindCar(carDown, function(s) carDownP = s end)

local tP = tabContentFrames["传送Panel"]
local tpTarget = nil
local tpList = Instance.new("ScrollingFrame"); tpList.Parent = tP; tpList.BackgroundTransparency = 1; tpList.Position = UDim2.new(0, 0, 0, 0); tpList.Size = UDim2.new(1, 0, 0, 150); tpList.CanvasSize = UDim2.new(0, 0, 0, 0); tpList.ScrollBarThickness = 2
local function refreshTP() for _, c in ipairs(tpList:GetChildren()) do if c:IsA("TextButton") then c:Destroy() end end; local y = 0; for _, p in ipairs(Players:GetPlayers()) do if p ~= plr then local btn = Instance.new("TextButton"); btn.Parent = tpList; btn.BackgroundColor3 = Color3.fromRGB(30, 32, 42); btn.Position = UDim2.new(0, 0, 0, y); btn.Size = UDim2.new(1, 0, 0, 28); btn.Text = "  " .. p.Name; btn.TextColor3 = Color3.fromRGB(200, 200, 210); btn.TextSize = 12; btn.TextXAlignment = Enum.TextXAlignment.Left; Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4); btn.MouseButton1Click:Connect(function() tpTarget = p; for _, c in ipairs(tpList:GetChildren()) do if c:IsA("TextButton") then c.BackgroundColor3 = Color3.fromRGB(30, 32, 42) end end; btn.BackgroundColor3 = Color3.fromRGB(50, 55, 70) end); y = y + 34 end end; tpList.CanvasSize = UDim2.new(0, 0, 0, math.max(y, 10)) end
task.spawn(function() while screenGui and screenGui.Parent do refreshTP() task.wait(3) end end); refreshTP()
local function teleportPlayer(mode) if not tpTarget or not tpTarget.Character then return end; local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart"); if not hrp or not rootPart then return end; if mode == "direct" then rootPart.CFrame = hrp.CFrame; elseif mode == "front" then rootPart.CFrame = hrp.CFrame + hrp.CFrame.LookVector * 0.5; elseif mode == "back" then rootPart.CFrame = hrp.CFrame - hrp.CFrame.LookVector * 0.5 end end
local btnCard = Instance.new("Frame"); btnCard.Parent = tP; btnCard.BackgroundColor3 = Color3.fromRGB(24, 26, 35); btnCard.Position = UDim2.new(0, 0, 0, 160); btnCard.Size = UDim2.new(1, 0, 0, 45); Instance.new("UICorner", btnCard).CornerRadius = UDim.new(0, 6)
local dBtn = Instance.new("TextButton"); dBtn.Parent = btnCard; dBtn.BackgroundColor3 = Color3.fromRGB(46, 204, 113); dBtn.Position = UDim2.new(0.05, 0, 0.1, 0); dBtn.Size = UDim2.new(0.28, 0, 0, 32); dBtn.Text = "直接"; dBtn.TextColor3 = Color3.fromRGB(255, 255, 255); dBtn.TextSize = 12; Instance.new("UICorner", dBtn).CornerRadius = UDim.new(0, 4); dBtn.MouseButton1Click:Connect(function() teleportPlayer("direct") end)
local fBtn = Instance.new("TextButton"); fBtn.Parent = btnCard; fBtn.BackgroundColor3 = Color3.fromRGB(52, 152, 219); fBtn.Position = UDim2.new(0.36, 0, 0.1, 0); fBtn.Size = UDim2.new(0.28, 0, 0, 32); fBtn.Text = "前面"; fBtn.TextColor3 = Color3.fromRGB(255, 255, 255); fBtn.TextSize = 12; Instance.new("UICorner", fBtn).CornerRadius = UDim.new(0, 4); fBtn.MouseButton1Click:Connect(function() teleportPlayer("front") end)
local bBtn = Instance.new("TextButton"); bBtn.Parent = btnCard; bBtn.BackgroundColor3 = Color3.fromRGB(231, 76, 60); bBtn.Position = UDim2.new(0.67, 0, 0.1, 0); bBtn.Size = UDim2.new(0.28, 0, 0, 32); bBtn.Text = "后面"; bBtn.TextColor3 = Color3.fromRGB(255, 255, 255); bBtn.TextSize = 12; Instance.new("UICorner", bBtn).CornerRadius = UDim.new(0, 4); bBtn.MouseButton1Click:Connect(function() teleportPlayer("back") end)
local flingBtn = Instance.new("TextButton"); flingBtn.Parent = tP; flingBtn.BackgroundColor3 = Color3.fromRGB(255, 165, 0); flingBtn.Position = UDim2.new(0, 0, 0, 220); flingBtn.Size = UDim2.new(1, 0, 0, 40); flingBtn.Text = "💥 甩飞一次"; flingBtn.TextColor3 = Color3.fromRGB(255, 255, 255); flingBtn.TextSize = 13; Instance.new("UICorner", flingBtn).CornerRadius = UDim.new(0, 6); flingBtn.MouseButton1Click:Connect(function() if tpTarget and tpTarget.Character then local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart"); if hrp then hrp.AssemblyLinearVelocity = Vector3.new(0, 120, 0) end end end)

-- ==========================================
-- 【⚡ 启动按钮】
-- ==========================================
local openBtn = Instance.new("TextButton")
openBtn.Name = "OpenButton"
openBtn.Parent = screenGui
openBtn.Size = UDim2.new(0, 55, 0, 55)
openBtn.Position = UDim2.new(0.08, 0, 0.5, 0)
openBtn.AnchorPoint = Vector2.new(0.5, 0.5)
openBtn.BackgroundColor3 = Color3.fromRGB(40, 50, 80)
openBtn.Text = "⚡"
openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
openBtn.TextSize = 28
openBtn.ZIndex = 9999
Instance.new("UICorner", openBtn).CornerRadius = UDim.new(0, 12)
local btnStroke = Instance.new("UIStroke"); btnStroke.Color = Color3.fromRGB(80, 160, 255); btnStroke.Thickness = 2; btnStroke.Parent = openBtn
openBtn.MouseButton1Click:Connect(function() mainFrame.Visible = not mainFrame.Visible end)

-- ==========================================
-- 【拖拽】
-- ==========================================
local dragging = false; local dragInput, mousePos, framePos
titleBar.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = true; mousePos = input.Position; framePos = mainFrame.Position; input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then dragging = false end end) end end)
titleBar.InputChanged:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end end)
UserInputService.InputChanged:Connect(function(input) if input == dragInput and dragging then local delta = input.Position - mousePos; mainFrame.Position = UDim2.new(framePos.X.Scale, framePos.X.Offset + delta.X, framePos.Y.Scale, framePos.Y.Offset + delta.Y) end end)

-- ==========================================
-- 【主循环：真正的核心逻辑】
-- ==========================================
renderConnection = RunService.RenderStepped:Connect(function()
    if not screenGui or not screenGui.Parent then if renderConnection then renderConnection:Disconnect() end return end

    -- 自瞄
    if Aimbot.Enabled then
        local closestDist = Aimbot.FOV
        local closestTarget = nil
        local center = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= plr then
                if Aimbot.TeamCheck and p.TeamColor == plr.TeamColor then continue end
                local c = p.Character
                if c and c:FindFirstChild("HumanoidRootPart") and c:FindFirstChild("Humanoid") and c.Humanoid.Health > 0 then
                    local s, on = camera:WorldToViewportPoint(c.HumanoidRootPart.Position)
                    if on then
                        local d = (Vector2.new(s.X, s.Y) - center).Magnitude
                        if d < closestDist then closestDist = d; closestTarget = p end
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
                camera.CFrame = camera.CFrame:Lerp(CFrame.lookAt(camera.CFrame.Position, camera.CFrame.Position + lDir), 0.7)
            end
        else
            currentTarget = nil
        end
    end

    -- ESP
    if Misc.ESP then
        for _, p in ipairs(Players:GetPlayers()) do if p ~= plr and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local c = p.Character; local hrp = c.HumanoidRootPart; local s, on = camera:WorldToViewportPoint(hrp.Position)
            if on and s.Z > 0 then
                local size = 50 + (200 - s.Z) * 0.3
                if not espCache[p.UserId] then
                    local f = Instance.new("Frame"); f.Parent = screenGui; f.BackgroundTransparency = 1; f.Size = UDim2.new(0, size, 0, size*2); f.Position = UDim2.new(0, s.X - size/2, 0, s.Y - size); f.ZIndex = 20; local st = Instance.new("UIStroke"); st.Color = Color3.fromRGB(255,255,255); st.Thickness = 1; st.Parent = f; local l = Instance.new("TextLabel"); l.Parent = f; l.BackgroundTransparency = 1; l.Size = UDim2.new(0, 100, 0, 16); l.Position = UDim2.new(0, -25, 0, -20); l.Text = p.Name; l.TextColor3 = Color3.fromRGB(255,255,255); l.TextSize = 10; l.ZIndex = 21; espCache[p.UserId] = {Frame=f, Label=l}
                else
                    local e = espCache[p.UserId]; e.Frame.Size = UDim2.new(0, size, 0, size*2); e.Frame.Position = UDim2.new(0, s.X - size/2, 0, s.Y - size); e.Frame.Visible = true; e.Label.Visible = true
                end
            else
                if espCache[p.UserId] then espCache[p.UserId].Frame.Visible = false; espCache[p.UserId].Label.Visible = false end
            end
        end end
    else
        for _, v in pairs(espCache) do pcall(function() v.Frame:Destroy(); v.Label:Destroy() end) end; espCache = {}
    end

    -- 飞车
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
                if carUpP then v = v + Vector3.new(0, FlyCar.Speed, 0) end
                if carDownP then v = v - Vector3.new(0, FlyCar.Speed, 0) end
                vr.AssemblyLinearVelocity = v
            end
        end
    end

    -- 人物属性
    if character and humanoid and humanoid.Parent then
        if Speed.Enabled then humanoid.WalkSpeed = Speed.Value
        elseif humanoid.WalkSpeed ~= 16 then humanoid.WalkSpeed = 16 end
        if Jump.Enabled then humanoid.JumpPower = Jump.Value; humanoid.UseJumpPower = true
        else humanoid.UseJumpPower = false end
        if Misc.AntiStun and humanoid.PlatformStand and not Flight.Enabled then humanoid.PlatformStand = false end
    end

    -- Noclip
    if Misc.Noclip and not noclipApplied and character then
        pcall(function() for _, p in ipairs(character:GetDescendants()) do if p:IsA("BasePart") and p.Name ~= "HumanoidRootPart" then p.CanCollide = false end end; noclipApplied = true end)
    elseif not Misc.Noclip and noclipApplied and character then        pcall(function() for _, p in ipairs(character:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = true end end; noclipApplied = false end)
    end

    -- 人物飞行
    if Flight.Enabled then
        pcall(function()
            if not character or not rootPart or not humanoid then return end
            if not rootPart.Parent then return end
            humanoid.PlatformStand = true
            if not camera then return end
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

print("✅【小木HUB v6.8】稳定全功能直通车版加载成功！点击左侧⚡按钮打开界面。")
