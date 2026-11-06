 --[[
=========================================================
  SORIANO AIMBOT — v4.0 (Arrastar + Fechar Consertados)
  LocalScript — coloque em StarterPlayerScripts
  Teclas: N=Aim | M=ESP | C=Circulo | F=Fly | G=SuperTiro | H=Painel
=========================================================
]]

local IMAGE_ID = "rbxassetid://96279184559501"

--===================== SERVIÇOS =====================--
local Players       = game:GetService("Players")
local RunService    = game:GetService("RunService")
local UIS           = game:GetService("UserInputService")
local VIM           = game:GetService("VirtualInputManager")
local TweenService  = game:GetService("TweenService")

local Camera = workspace.CurrentCamera
local LP     = Players.LocalPlayer
local PG     = LP:WaitForChild("PlayerGui")

--===================== ESTADO =====================--
local lockedTarget = nil
local camActive    = false
local yaw, pitch   = 0, 0
local bursting     = false
local flying       = false
local flyObjects   = {}
local panelOpen    = true

--===================== CONFIG =====================--
local Config = {
    Aim = { Enabled = false, FOV = 260, Smoothness = 0.28, Part = "Head", MaxDist = 2000 },
    Cam = { Sens = 0.003, Distance = 12, Height = 1.4 },
    Circle = { Enabled = true, Color = Color3.fromRGB(0, 255, 140), Thickness = 2 },
    ESP = { Enabled = false, Color = Color3.fromRGB(0, 255, 140), MaxDistance = 2500 },
    SuperShot = { Enabled = false, Mult = 500, Batch = 20 },
    Fly = { Speed = 65 },
    TeamCheck = true,
    SkipFriend = true,
}

--===================== HELPERS =====================--
local function getMyRoot()
    local char = LP.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

--===================== FILTROS =====================--
local friendCache, teamCache, espData = {}, {}, {}
local buildESP

local function removeESP(plr)
    local d = espData[plr]
    if not d then return end
    if d.highlight then d.highlight:Destroy() end
    if d.billboard then d.billboard:Destroy() end
    espData[plr] = nil
end

local function clearAllESP()
    for plr in pairs(espData) do removeESP(plr) end
end

local function hookTeam(plr)
    teamCache[plr] = plr.Team
    plr.Changed:Connect(function(prop)
        if prop == "Team" then
            teamCache[plr] = plr.Team
            if espData[plr] then
                task.spawn(function() removeESP(plr); if buildESP then buildESP(plr) end end)
            end
        end
    end)
end

for _, plr in ipairs(Players:GetPlayers()) do hookTeam(plr) end
Players.PlayerAdded:Connect(hookTeam)

local function isFriend(plr)
    if plr == LP then return true end
    if not Config.SkipFriend then return false end
    local cached = friendCache[plr.UserId]
    if cached == true or cached == false then return cached end
    friendCache[plr.UserId] = false
    task.spawn(function()
        local ok, res = pcall(function() return plr:IsFriendsWith(LP.UserId) end)
        if ok then friendCache[plr.UserId] = res else friendCache[plr.UserId] = nil end
    end)
    return false
end

local function isAlly(plr)
    if not Config.TeamCheck or plr == LP then return false end
    local myTeam, theirTeam = LP.Team, teamCache[plr] or plr.Team
    if not myTeam or not theirTeam then return false end
    return theirTeam == myTeam
end

local function validTarget(plr)
    if not plr or plr == LP or not plr.Parent then return false end
    if isFriend(plr) or isAlly(plr) then return false end
    local char = plr.Character
    if not char then return false end
    local hum, part = char:FindFirstChildOfClass("Humanoid"), char:FindFirstChild(Config.Aim.Part)
    if not hum or not part or hum.Health <= 0 or hum:GetState() == Enum.HumanoidStateType.Dead then return false end
    local myRoot = getMyRoot()
    if not myRoot or (part.Position - myRoot.Position).Magnitude > Config.Aim.MaxDist then return false end
    return true
end

--===================== CÍRCULO =====================--
local overlay = Instance.new("ScreenGui")
overlay.IgnoreGuiInset, overlay.ResetOnSpawn, overlay.DisplayOrder, overlay.Parent = true, false, 20, PG

local circle = Instance.new("Frame")
circle.AnchorPoint, circle.BackgroundTransparency, circle.BorderSizePixel, circle.Visible, circle.Parent = Vector2.new(0.5, 0.5), 1, 0, false, overlay
local circleCorner = Instance.new("UICorner"); circleCorner.CornerRadius = UDim.new(1, 0); circleCorner.Parent = circle
local circleStroke = Instance.new("UIStroke"); circleStroke.Color = Config.Circle.Color; circleStroke.Thickness = Config.Circle.Thickness; circleStroke.Transparency = 0.1; circleStroke.Parent = circle

local function refreshCircle()
    local d = Config.Aim.FOV * 2
    circle.Size = UDim2.fromOffset(d, d)
    circle.Position = UDim2.fromOffset(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    circleStroke.Color, circleStroke.Thickness = Config.Circle.Color, Config.Circle.Thickness
end
refreshCircle()

--===================== SUPER TIRO =====================--
local function fireOnce(tool)
    if tool and tool.Parent and tool:IsA("Tool") then pcall(function() tool:Activate() end); return end
    pcall(function()
        VIM:SendMouseButtonEvent(math.floor(Camera.ViewportSize.X/2), math.floor(Camera.ViewportSize.Y/2), 0, true,  game, 1)
        VIM:SendMouseButtonEvent(math.floor(Camera.ViewportSize.X/2), math.floor(Camera.ViewportSize.Y/2), 0, false, game, 1)
    end)
end

local function superShot()
    if bursting or not LP.Character then return end
    local total, batch = math.max(1, math.floor(Config.SuperShot.Mult)), math.max(1, math.floor(Config.SuperShot.Batch))
    bursting = true
    task.spawn(function()
        pcall(function()
            for i = 1, total do
                if not Config.SuperShot.Enabled or not LP.Character then break end
                local tool = (i % batch == 1) and LP.Character:FindFirstChildOfClass("Tool") or nil
                fireOnce(tool)
                if i % batch == 0 then task.wait() end
            end
        end)
        bursting = false
    end)
end

--===================== ESP =====================--
function buildESP(plr)
    removeESP(plr)
    if not Config.ESP.Enabled or plr == LP then return end
    local char = plr.Character
    local hrp, hum = char and char:FindFirstChild("HumanoidRootPart"), char and char:FindFirstChildOfClass("Humanoid")
    if not hrp or not hum then return end

    local hl = Instance.new("Highlight")
    hl.Adornee, hl.FillColor, hl.OutlineColor, hl.FillTransparency, hl.DepthMode, hl.Parent = char, Config.ESP.Color, Color3.new(1,1,1), 0.55, Enum.HighlightDepthMode.AlwaysOnTop, char

    local bb = Instance.new("BillboardGui")
    bb.Size, bb.StudsOffset, bb.AlwaysOnTop, bb.Parent = UDim2.new(0, 220, 0, 56), Vector3.new(0, 3.2, 0), true, hrp

    local name = Instance.new("TextLabel")
    name.Size, name.BackgroundTransparency, name.Font, name.TextSize, name.TextColor3, name.TextStrokeTransparency, name.Text, name.Parent = UDim2.new(1,0,0.45,0), 1, Enum.Font.GothamBold, 14, Config.ESP.Color, 0.4, plr.Name, bb

    local info = Instance.new("TextLabel")
    info.Size, info.Position, info.BackgroundTransparency, info.Font, info.TextSize, info.TextColor3, info.TextStrokeTransparency, info.Parent = UDim2.new(1,0,0.35,0), UDim2.new(0,0,0.45,0), 1, Enum.Font.Gotham, 12, Color3.new(1,1,1), 0.4, bb

    espData[plr] = { highlight = hl, billboard = bb, info = info }
end

task.spawn(function()
    while task.wait(0.15) do
        if not Config.ESP.Enabled then clearAllESP() continue end
        local myRoot = getMyRoot()
        for plr, d in pairs(espData) do
            local char = plr.Character
            local hrp, hum = char and char:FindFirstChild("HumanoidRootPart"), char and char:FindFirstChildOfClass("Humanoid")
            if not hrp or not hum or hum.Health <= 0 or not myRoot then removeESP(plr)
            else
                local dist = math.floor((hrp.Position - myRoot.Position).Magnitude)
                local show = dist <= Config.ESP.MaxDistance and not isFriend(plr) and not isAlly(plr)
                if d.highlight then d.highlight.Enabled = show end
                if d.billboard then d.billboard.Enabled = show end
                if show then d.info.Text = string.format("%d HP | %dm", math.floor(hum.Health), dist) end
            end
        end
        for _, plr in ipairs(Players:GetPlayers()) do if plr ~= LP and not espData[plr] and plr.Character then buildESP(plr) end end
    end
end)

local function hookPlayer(plr)
    if plr == LP then return end
    plr.CharacterAdded:Connect(function() task.spawn(function() task.wait(0.5) buildESP(plr) end) end)
    if plr.Character then task.spawn(function() task.wait(0.5) buildESP(plr) end) end
end

for _, plr in ipairs(Players:GetPlayers()) do hookPlayer(plr) end
Players.PlayerAdded:Connect(hookPlayer)
Players.PlayerRemoving:Connect(function(plr) removeESP(plr); teamCache[plr] = nil; if lockedTarget == plr then lockedTarget = nil end end)

--===================== AIMBOT =====================--
local function acquireTarget()
    local best, bestDist = nil, Config.Aim.FOV
    local center = Camera.ViewportSize / 2
    for _, plr in ipairs(Players:GetPlayers()) do
        if validTarget(plr) then
            local part = plr.Character[Config.Aim.Part]
            local sp, onScreen = Camera:WorldToViewportPoint(part.Position)
            if onScreen then
                local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
                if d <= bestDist then best, bestDist = plr, d end
            end
        end
    end
    return best
end

local function syncAngles()
    local look = Camera.CFrame.LookVector
    yaw, pitch = math.atan2(-look.X, -look.Z), math.asin(math.clamp(look.Y, -1, 1))
end

local function setAim(state)
    if state then
        if not (LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")) then return end
        syncAngles()
        Camera.CameraType, camActive, Config.Aim.Enabled = Enum.CameraType.Scriptable, true, true
    else
        Config.Aim.Enabled, camActive, lockedTarget = false, false, nil
        Camera.CameraType = Enum.CameraType.Custom
        local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
        if hum then Camera.CameraSubject = hum end
    end
end

local function aimStep()
    if not camActive then circle.Visible = false return end
    local char = LP.Character
    local hum, root = char and char:FindFirstChildOfClass("Humanoid"), char and char:FindFirstChild("HumanoidRootPart")
    if not hum or not root or hum.Health <= 0 then return end

    circle.Visible = Config.Circle.Enabled
    if Config.Circle.Enabled then circle.Position = UDim2.fromOffset(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2) end

    if not validTarget(lockedTarget) then
        lockedTarget = acquireTarget() 
    end

    local originPos = root.Position + Vector3.new(0, Config.Cam.Height, 0)

    if lockedTarget then
        local part = lockedTarget.Character and lockedTarget.Character[Config.Aim.Part]
        if part then
            local dir = part.Position - originPos
            if dir.Magnitude > 0.5 then
                local wantYaw, wantPitch = math.atan2(-dir.X, -dir.Z), math.asin(math.clamp(dir.Unit.Y, -1, 1))
                local dy = (wantYaw - yaw + math.pi) % (2 * math.pi) - math.pi
                local a = math.clamp(1 - Config.Aim.Smoothness, 0.05, 1)
                yaw, pitch = yaw + dy * a, pitch + (wantPitch - pitch) * a
            end
        end
    end

    pitch = math.clamp(pitch, -1.5, 1.5)
    local rot = CFrame.fromOrientation(pitch, yaw, 0)
    Camera.CFrame = CFrame.new(originPos - rot.LookVector * Config.Cam.Distance) * rot
end

pcall(function() RunService:UnbindFromRenderStep("sorianoAim") end)
RunService:BindToRenderStep("sorianoAim", Enum.RenderPriority.Camera.Value + 1, aimStep)

UIS.InputChanged:Connect(function(input)
    if not camActive then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        yaw, pitch = yaw - input.Delta.X * Config.Cam.Sens, math.clamp(pitch - input.Delta.Y * Config.Cam.Sens, -1.5, 1.5)
    elseif input.UserInputType == Enum.UserInputType.Touch then
        yaw, pitch = yaw - input.Delta.X * Config.Cam.Sens * 1.8, math.clamp(pitch - input.Delta.Y * Config.Cam.Sens * 1.8, -1.5, 1.5)
    end
end)

LP.CharacterAdded:Connect(function()
    if camActive then task.wait(0.8); if camActive then local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid"); if hum then Camera.CameraType = Enum.CameraType.Scriptable; syncAngles() end end end
end)

--===================== FLY =====================--
local function setFly(state)
    flying = state
    local char = LP.Character
    local hrp, hum = char and char:FindFirstChild("HumanoidRootPart"), char and char:FindFirstChildOfClass("Humanoid")
    for _, obj in ipairs(flyObjects) do obj:Destroy() end; flyObjects = {}
    if not flying then if hum then hum.PlatformStand = false end; return end
    if not hrp or not hum then flying = false; return end

    local att = Instance.new("Attachment"); att.Parent = hrp
    local lv = Instance.new("LinearVelocity")
    lv.Attachment0, lv.MaxForce, lv.VelocityConstraintMode, lv.VectorVelocity, lv.Parent = att, math.huge, Enum.VelocityConstraintMode.Vector, Vector3.zero, hrp
    local ao = Instance.new("AlignOrientation")
    ao.Mode, ao.Attachment0, ao.MaxTorque, ao.CFrame, ao.Parent = Enum.OrientationAlignmentMode.OneAttachment, att, math.huge, Camera.CFrame, hrp
    flyObjects = { lv, ao, att }
    hum.PlatformStand = true
end

RunService.RenderStepped:Connect(function()
    if not flying then return end
    local lv, ao = flyObjects[1], flyObjects[2]
    if not lv or not ao then return end
    local dir = Vector3.zero
    if UIS:IsKeyDown(Enum.KeyCode.W) then dir += Camera.CFrame.LookVector end
    if UIS:IsKeyDown(Enum.KeyCode.S) then dir -= Camera.CFrame.LookVector end
    if UIS:IsKeyDown(Enum.KeyCode.A) then dir -= Camera.CFrame.RightVector end
    if UIS:IsKeyDown(Enum.KeyCode.D) then dir += Camera.CFrame.RightVector end
    if UIS:IsKeyDown(Enum.KeyCode.Space) then dir += Vector3.new(0, 1, 0) end
    if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then dir -= Vector3.new(0, 1, 0) end
    lv.VectorVelocity = dir.Magnitude > 0 and dir.Unit * Config.Fly.Speed or Vector3.zero
    ao.CFrame = Camera.CFrame
end)

LP.CharacterAdded:Connect(function() if flying then task.wait(0.6); setFly(false); setFly(true) end end)

--===================== PAINEL UI =====================--
local panelGui = Instance.new("ScreenGui")
panelGui.ResetOnSpawn, panelGui.DisplayOrder, panelGui.Parent = false, 50, PG

local panel = Instance.new("Frame")
panel.Size, panel.Position, panel.BackgroundColor3, panel.BackgroundTransparency, panel.BorderSizePixel, panel.Parent = UDim2.new(0, 232, 0, 520), UDim2.new(0, 18, 0, 70), Color3.fromRGB(12, 14, 18), 0.28, 0, panelGui

local corner = Instance.new("UICorner"); corner.CornerRadius = UDim.new(0, 10); corner.Parent = panel
local stroke = Instance.new("UIStroke"); stroke.Color = Config.Circle.Color; stroke.Transparency = 0.55; stroke.Parent = panel

-- Título
local title = Instance.new("TextLabel")
title.Size, title.BackgroundColor3, title.BackgroundTransparency, title.BorderSizePixel, title.Font, title.TextSize, title.TextColor3, title.Text, title.Active, title.Parent = UDim2.new(1, 0, 0, 34), Color3.new(0, 0, 0), 0.45, 0, Enum.Font.GothamBold, 13, Config.Circle.Color, "SORIANO AIMBOT v4.0", true, panel
local titleCorner = Instance.new("UICorner"); titleCorner.CornerRadius = UDim.new(0, 10); titleCorner.Parent = title

-- Botão Fechar (Zona Cega - Fica em cima do título, mas não arrasta)
local closeBtn = Instance.new("TextButton")
closeBtn.Size, closeBtn.Position, closeBtn.BackgroundColor3, closeBtn.BackgroundTransparency, closeBtn.BorderSizePixel, closeBtn.Font, closeBtn.TextSize, closeBtn.TextColor3, closeBtn.Text, closeBtn.ZIndex, closeBtn.Parent = UDim2.new(0, 26, 0, 26), UDim2.new(1, -30, 0, 4), Color3.fromRGB(150, 45, 45), 0.15, 0, Enum.Font.GothamBold, 14, Color3.new(1, 1, 1), "✕", 5, panel
local closeCorner = Instance.new("UICorner"); closeCorner.CornerRadius = UDim.new(0, 6); closeCorner.Parent = closeBtn

local status = Instance.new("TextLabel")
status.Size, status.Position, status.BackgroundTransparency, status.Font, status.TextSize, status.TextXAlignment, status.TextColor3, status.Text, status.Parent = UDim2.new(1, -20, 0, 16), UDim2.new(0, 10, 0, 36), 1, Enum.Font.Code, 10, Enum.TextXAlignment.Left, Color3.fromRGB(120, 220, 170), "alvo: nenhum", panel

local statusFilters = Instance.new("TextLabel")
statusFilters.Size, statusFilters.Position, statusFilters.BackgroundTransparency, statusFilters.Font, statusFilters.TextSize, statusFilters.TextXAlignment, statusFilters.TextColor3, statusFilters.Text, statusFilters.Parent = UDim2.new(1, -20, 0, 16), UDim2.new(0, 10, 0, 52), 1, Enum.Font.Code, 10, Enum.TextXAlignment.Left, Color3.fromRGB(180, 180, 180), "filtros: ...", panel

-- Drag (Arrastar) - O título não cobre o botão, então não tem mais conflito!
local dragStart, startPos = nil, nil
title.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then 
        dragStart = input.Position
        startPos = panel.Position
    end
end)
UIS.InputChanged:Connect(function(input)
    if dragStart and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        panel.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UIS.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then 
        dragStart = nil 
    end
end)

local refreshers = {}
local function makeToggle(text, y, getter, setter)
    local btn = Instance.new("TextButton")
    btn.Size, btn.Position, btn.BackgroundColor3, btn.BackgroundTransparency, btn.BorderSizePixel, btn.AutoButtonColor, btn.Font, btn.TextSize, btn.TextColor3, btn.Parent = UDim2.new(1, -20, 0, 28), UDim2.new(0, 10, 0, y), Color3.fromRGB(40, 44, 52), 0.2, 0, false, Enum.Font.GothamBold, 12, Color3.new(1, 1, 1), panel
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, 6); c.Parent = btn
    local function refresh()
        local on = getter()
        btn.BackgroundColor3, btn.Text, btn.TextColor3 = on and Color3.fromRGB(0, 170, 95) or Color3.fromRGB(40, 44, 52), text .. (on and "  [ON]" or "  [OFF]"), on and Color3.new(1, 1, 1) or Color3.fromRGB(190, 190, 190)
    end
    btn.MouseButton1Click:Connect(function() setter(not getter()); refresh() end)
    refreshers[text] = refresh; refresh(); return refresh
end

local function makeStepper(label, y, getter, setter, step, min, max, fmt, onChange)
    local box = Instance.new("Frame")
    box.Size, box.Position, box.BackgroundTransparency, box.Parent = UDim2.new(1, -20, 0, 26), UDim2.new(0, 10, 0, y), 1, panel
    local lbl = Instance.new("TextLabel")
    lbl.Size, lbl.BackgroundTransparency, lbl.Font, lbl.TextSize, lbl.TextColor3, lbl.TextXAlignment, lbl.Parent = UDim2.new(0.6, 0, 1, 0), 1, Enum.Font.Gotham, 11, Color3.fromRGB(200, 200, 200), Enum.TextXAlignment.Left, box
    local minus = Instance.new("TextButton")
    minus.Size, minus.Position, minus.BackgroundColor3, minus.BorderSizePixel, minus.Font, minus.TextColor3, minus.Text, minus.Parent = UDim2.new(0, 26, 1, 0), UDim2.new(0.6, 0, 0, 0), Color3.fromRGB(40, 44, 52), 0, Enum.Font.GothamBold, Color3.new(1, 1, 1), "-", box
    local plus = minus:Clone(); plus.Position, plus.Text, plus.Parent = UDim2.new(1, -26, 0, 0), "+", box
    for _, b in ipairs({ minus, plus }) do local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, 6); c.Parent = b end
    local function refresh() lbl.Text = string.format("%s: %s", label, fmt(getter())) end
    minus.MouseButton1Click:Connect(function() setter(math.clamp(getter() - step, min, max)); refresh(); if onChange then onChange() end end)
    plus.MouseButton1Click:Connect(function() setter(math.clamp(getter() + step, min, max)); refresh(); if onChange then onChange() end end)
    refresh()
end

makeToggle("AIMBOT (N)", 74, function() return Config.Aim.Enabled end, function(v) setAim(v) end)
local refreshEsp = makeToggle("ESP (M)", 108, function() return Config.ESP.Enabled end, function(v)
    Config.ESP.Enabled = v
    if v then for _, plr in ipairs(Players:GetPlayers()) do buildESP(plr) end else clearAllESP() end
end)

makeToggle("CIRCULO (C)", 142, function() return Config.Circle.Enabled end, function(v) Config.Circle.Enabled = v end)
makeToggle("VOAR (F)", 176, function() return flying end, function(v) setFly(v) end)
makeToggle("SUPER TIRO (G)", 210, function() return Config.SuperShot.Enabled end, function(v) Config.SuperShot.Enabled = v end)
makeToggle("IGNORAR AMIGOS", 244, function() return Config.SkipFriend end, function(v) Config.SkipFriend = v; if v then friendCache = {} end end)
makeToggle("IGNORAR TIME", 278, function() return Config.TeamCheck end, function(v) Config.TeamCheck = v end)

makeStepper("RAIO DO CIRCULO", 314, function() return Config.Aim.FOV end, function(v) Config.Aim.FOV = v end, 20, 40, 500, function(v) return tostring(v) end, refreshCircle)
makeStepper("SUAVIDADE", 344, function() return Config.Aim.Smoothness end, function(v) Config.Aim.Smoothness = v end, 0.05, 0.05, 0.95, function(v) return string.format("%.2f", v) end)
makeStepper("ALCANCE (STUDS)", 374, function() return Config.Aim.MaxDist end, function(v) Config.Aim.MaxDist = v end, 100, 100, 2500, function(v) return tostring(v) end)
makeStepper("DISTANCIA CAM", 404, function() return Config.Cam.Distance end, function(v) Config.Cam.Distance = v end, 2, 4, 12, function(v) return tostring(v) end)
makeStepper("TIROS POR CLIQUE", 434, function() return Config.SuperShot.Mult end, function(v) Config.SuperShot.Mult = v end, 50, 10, 1000, function(v) return tostring(v) end)

local reset = Instance.new("TextButton")
reset.Size, reset.Position, reset.BackgroundColor3, reset.BackgroundTransparency, reset.BorderSizePixel, reset.Font, reset.TextSize, reset.TextColor3, reset.Text, reset.Parent = UDim2.new(1, -20, 0, 26), UDim2.new(0, 10, 0, 470), Color3.fromRGB(150, 45, 45), 0.15, 0, Enum.Font.GothamBold, 11, Color3.new(1, 1, 1), "PARAR TUDO", panel
local rc = Instance.new("UICorner"); rc.CornerRadius = UDim.new(0, 6); rc.Parent = reset
reset.MouseButton1Click:Connect(function()
    setAim(false); Config.SuperShot.Enabled = false; bursting = false; Config.ESP.Enabled = false; clearAllESP(); if flying then setFly(false) end
    friendCache, teamCache = {}, {}
    for _, r in pairs(refreshers) do r() end
end)

--===================== BOTÃO FLUTUANTE (COBRA + FALLBACK) =====================--
local openBtn = Instance.new("ImageButton")
openBtn.Size                   = UDim2.new(0, 54, 0, 54)
openBtn.Position               = UDim2.new(0, 18, 0, 70)
openBtn.BackgroundColor3       = Color3.fromRGB(12, 14, 18)
openBtn.BackgroundTransparency = 0.15
openBtn.BorderSizePixel        = 0
openBtn.Image                  = IMAGE_ID
openBtn.ScaleType              = Enum.ScaleType.Fit
openBtn.Visible                = false
openBtn.Parent                 = panelGui

local openCorner = Instance.new("UICorner")
openCorner.CornerRadius = UDim.new(1, 0)
openCorner.Parent       = openBtn

local openStroke = Instance.new("UIStroke")
openStroke.Color        = Config.Circle.Color
openStroke.Thickness    = 2
openStroke.Transparency = 0.25
openStroke.Parent       = openBtn

local fallbackEmoji = Instance.new("TextLabel")
fallbackEmoji.Size = UDim2.new(1, 0, 1, 0)
fallbackEmoji.BackgroundTransparency = 1
fallbackEmoji.Text = "🐍"
fallbackEmoji.TextScaled = true
fallbackEmoji.TextColor3 = Config.Circle.Color
fallbackEmoji.Parent = openBtn

openBtn.ImageLabel:GetPropertyChangedSignal("IsLoaded"):Connect(function()
    if openBtn.ImageLabel.IsLoaded then fallbackEmoji.Visible = false end
end)

openBtn.MouseEnter:Connect(function()
    TweenService:Create(openBtn, TweenInfo.new(0.2), { BackgroundColor3 = Color3.fromRGB(0, 60, 35), BackgroundTransparency = 0.1 }):Play()
    TweenService:Create(openStroke, TweenInfo.new(0.2), { Thickness = 3, Transparency = 0 }):Play()
end)

openBtn.MouseLeave:Connect(function()
    TweenService:Create(openBtn, TweenInfo.new(0.2), { BackgroundColor3 = Color3.fromRGB(12, 14, 18), BackgroundTransparency = 0.15 }):Play()
    TweenService:Create(openStroke, TweenInfo.new(0.2), { Thickness = 2, Transparency = 0.25 }):Play()
end)

--===================== SISTEMA FECHAR/ABRIR (BLINDADO) =====================--
local function setPanelOpen(state)
    panelOpen = state
    panel.Visible = state
    openBtn.Visible = not state
end

closeBtn.MouseButton1Click:Connect(function() 
    setPanelOpen(false) 
end)

openBtn.MouseButton1Click:Connect(function() 
    setPanelOpen(true) 
end)

--===================== LOOP DE STATUS =====================--
task.spawn(function()
    while task.wait(0.1) do
        if lockedTarget and lockedTarget.Parent and validTarget(lockedTarget) then
            local ok, dist = pcall(function()
                local part = lockedTarget.Character and lockedTarget.Character[Config.Aim.Part]
                return part and math.floor((part.Position - Camera.CFrame.Position).Magnitude) or nil
            end)
            if ok and dist then status.Text, status.TextColor3 = "TRAVADO: " .. lockedTarget.Name .. " | " .. dist, Color3.fromRGB(120, 255, 150)
            else status.Text, status.TextColor3 = "alvo perdido: " .. lockedTarget.Name, Color3.fromRGB(255, 180, 120) end
        else
            status.Text, status.TextColor3 = Config.Aim.Enabled and "alvo: procurando no circulo..." or "alvo: nenhum", Color3.fromRGB(180, 180, 120)
        end

        local timeStr = Config.TeamCheck and "TIME: ON" or "TIME: OFF"
        local friendStr = Config.SkipFriend and "AMIGOS: ON" or "AMIGOS: OFF"
        local color = (Config.TeamCheck or Config.SkipFriend) and Color3.fromRGB(100, 220, 255) or Color3.fromRGB(180, 180, 180)
        statusFilters.Text, statusFilters.TextColor3 = string.format("filtros: %s | %s", timeStr, friendStr), color
    end
end)

--===================== INPUT =====================--
UIS.InputBegan:Connect(function(input, processed)
    if input.UserInputType == Enum.UserInputType.MouseButton1 and Config.SuperShot.Enabled and not processed then superShot() end
    if processed then return end
    
    if input.KeyCode == Enum.KeyCode.M then
        Config.ESP.Enabled = not Config.ESP.Enabled
        if Config.ESP.Enabled then for _, plr in ipairs(Players:GetPlayers()) do buildESP(plr) end else clearAllESP() end
        if refreshEsp then refreshEsp() end
    elseif input.KeyCode == Enum.KeyCode.N then setAim(not Config.Aim.Enabled); if refreshers["AIMBOT (N)"] then refreshers["AIMBOT (N)"]() end
    elseif input.KeyCode == Enum.KeyCode.C then Config.Circle.Enabled = not Config.Circle.Enabled; if refreshers["CIRCULO (C)"] then refreshers["CIRCULO (C)"]() end
    elseif input.KeyCode == Enum.KeyCode.F then setFly(not flying); if refreshers["VOAR (F)"] then refreshers["VOAR (F)"]() end
    elseif input.KeyCode == Enum.KeyCode.G then Config.SuperShot.Enabled = not Config.SuperShot.Enabled; if refreshers["SUPER TIRO (G)"] then refreshers["SUPER TIRO (G)"]() end
    elseif input.KeyCode == Enum.KeyCode.H then setPanelOpen(not panelOpen) end
end)

print("[SORIANO AIMBOT v4.0] carregado | Arrastar e Fechar perfeitos!")
