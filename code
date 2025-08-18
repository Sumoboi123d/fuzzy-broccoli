-- COMBINED: Player ESP + First-Floor Base Timer ESP + Highest-Earnings Brainrot ESP
-- Fixes in this build:
--   • Highest brainrot instantly re-picks when the top leaves or drops to 0
--   • Text is anchored right above the *machine* using an Attachment (bounding-box top)
--   • Bold text (GothamBlack + black stroke), Name=RED, $/s=WHITE
--   • Low-lag: we watch only the rate *source*; scans are filtered; light watchdog
--   • Shares the same UI toggle as before (hotkey T or button)

------------------------------------------------------------
-- Services
------------------------------------------------------------
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer      = Players.LocalPlayer
local Camera           = workspace.CurrentCamera

------------------------------------------------------------
-- Config
------------------------------------------------------------
local HOTKEY_TOGGLE = Enum.KeyCode.T

-- Player name tag
local NAME_TEXT_SIZE     = 14
local NAME_FONT          = Enum.Font.GothamSemibold
local NAME_COLOR         = Color3.fromRGB(255,255,255)
local NAME_STROKE        = Color3.fromRGB(10,10,10)
local NAME_MAX_DISTANCE  = 600

-- Player highlight
local HIGHLIGHT_FILL_COLOR           = Color3.fromRGB(65, 105, 225)
local HIGHLIGHT_FILL_TRANSPARENCY    = 0.10
local HIGHLIGHT_OUTLINE_COLOR        = Color3.fromRGB(65, 105, 225)
local HIGHLIGHT_OUTLINE_TRANSPARENCY = 0.15

-- Panel
local PANEL_PRIMARY = Color3.fromRGB(28, 28, 33)
local PANEL_ACCENT  = Color3.fromRGB(235, 64, 52)
local PANEL_TEXT    = Color3.fromRGB(235, 235, 235)

------------------------------------------------------------
-- State (players)
------------------------------------------------------------
local ESPEnabled = true
local PerPlayer  = {}    -- [Player] = { Highlight, Billboard, Label, Head, HRP, Character, ProxyRig, ProxyMap, ProxyActive, CoreParts, LastShownMeters }
local Connections= {}    -- [Player] = { RBXScriptConnection, ... }

-- Proxy rigs folder
local PROXY_FOLDER = workspace:FindFirstChild("_ESP_ProxyRigs")
if not PROXY_FOLDER then
    PROXY_FOLDER = Instance.new("Folder")
    PROXY_FOLDER.Name = "_ESP_ProxyRigs"
    PROXY_FOLDER.Parent = workspace
end

------------------------------------------------------------
-- UI
------------------------------------------------------------
local controlGui = Instance.new("ScreenGui")
controlGui.Name = "ESPControlGui"
controlGui.IgnoreGuiInset = false
controlGui.ResetOnSpawn = false
controlGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
controlGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local panel = Instance.new("Frame")
panel.Name = "ESPPanel"
panel.Size = UDim2.fromOffset(200, 60)
panel.Position = UDim2.fromOffset(24, 120)
panel.BackgroundColor3 = PANEL_PRIMARY
panel.BorderSizePixel = 0
panel.Parent = controlGui

local uiCorner = Instance.new("UICorner")
uiCorner.CornerRadius = UDim.new(0, 10)
uiCorner.Parent = panel

local stroke = Instance.new("UIStroke")
stroke.Thickness = 1
stroke.Color = Color3.fromRGB(60, 60, 68)
stroke.Transparency = 0.25
stroke.Parent = panel

local title = Instance.new("TextLabel")
title.BackgroundTransparency = 1
title.Size = UDim2.new(1, -12, 0, 20)
title.Position = UDim2.fromOffset(10, 6)
title.Font = Enum.Font.GothamBold
title.TextSize = 15
title.TextXAlignment = Enum.TextXAlignment.Left
title.Text = "ESP"
title.TextColor3 = PANEL_TEXT
title.Parent = panel

local hint = Instance.new("TextLabel")
hint.BackgroundTransparency = 1
hint.Size = UDim2.new(1, -12, 0, 16)
hint.Position = UDim2.fromOffset(10, 26)
hint.Font = Enum.Font.Gotham
hint.TextSize = 12
hint.TextXAlignment = Enum.TextXAlignment.Left
hint.TextTransparency = 0.2
hint.Text = "T to toggle"
hint.TextColor3 = Color3.fromRGB(200, 200, 210)
hint.Parent = panel

local toggleBtn = Instance.new("TextButton")
toggleBtn.Name = "Toggle"
toggleBtn.Size = UDim2.fromOffset(84, 28)
toggleBtn.Position = UDim2.new(1, -10 - 84, 1, -8 - 28)
toggleBtn.BackgroundColor3 = PANEL_ACCENT
toggleBtn.BorderSizePixel = 0
toggleBtn.AutoButtonColor = true
toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
toggleBtn.Font = Enum.Font.GothamSemibold
toggleBtn.TextSize = 14
toggleBtn.Text = "ON"
toggleBtn.Parent = panel

local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(0, 8)
toggleCorner.Parent = toggleBtn

local dragHandle = Instance.new("Frame")
dragHandle.Name = "DragHandle"
dragHandle.BackgroundTransparency = 1
dragHandle.Size = UDim2.new(1, -8, 0, 34)
dragHandle.Position = UDim2.fromOffset(4, 4)
dragHandle.ZIndex = panel.ZIndex + 1
dragHandle.Parent = panel

do
    local dragging = false
    local dragOffset = Vector2.zero
    local function clampToViewport(px, py)
        local vp = Camera.ViewportSize
        local sz = panel.AbsoluteSize
        local x = math.clamp(px, 0, math.max(0, vp.X - sz.X))
        local y = math.clamp(py, 0, math.max(0, vp.Y - sz.Y))
        return x, y
    end
    dragHandle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragOffset = input.Position - panel.AbsolutePosition
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local rawX = input.Position.X - dragOffset.X
            local rawY = input.Position.Y - dragOffset.Y
            local x, y = clampToViewport(rawX, rawY)
            panel.Position = UDim2.fromOffset(x, y)
        end
    end)
    Camera:GetPropertyChangedSignal("ViewportSize"):Connect(function()
        local pos = panel.AbsolutePosition
        local x, y = clampToViewport(pos.X, pos.Y)
        panel.Position = UDim2.fromOffset(x, y)
    end)
end

------------------------------------------------------------
-- Player ESP helpers
------------------------------------------------------------
local CORE_NAMES = {
    "Head","UpperTorso","LowerTorso","Torso",
    "LeftUpperArm","LeftLowerArm","LeftHand",
    "RightUpperArm","RightLowerArm","RightHand",
    "LeftUpperLeg","LeftLowerLeg","LeftFoot",
    "RightUpperLeg","RightLowerLeg","RightFoot",
    "Left Arm","Right Arm","Left Leg","Right Leg",
    "HumanoidRootPart",
}

local function collectCoreParts(character)
    local core = {}
    for _, name in ipairs(CORE_NAMES) do
        local p = character:FindFirstChild(name)
        if p and p:IsA("BasePart") then table.insert(core, p) end
    end
    if #core == 0 then
        for _, d in ipairs(character:GetChildren()) do
            if d:IsA("BasePart") then table.insert(core, d) end
        end
    end
    return core
end

local function isCharacterInvisibleFast(coreParts)
    for _, part in ipairs(coreParts) do
        if part.Parent then
            local lt = (part.LocalTransparencyModifier or 0)
            local t  = part.Transparency
            if (lt < 0.95) and (t < 0.95) then return false end
        end
    end
    return true
end

local function makeNameBillboard(head, player)
    local bb = Instance.new("BillboardGui")
    bb.Name = "ESP_NameTag"
    bb.Size = UDim2.new(0, 140, 0, 22)
    bb.StudsOffset = Vector3.new(0, 3.0, 0)
    bb.AlwaysOnTop = true
    bb.MaxDistance = NAME_MAX_DISTANCE
    bb.Parent = head

    local label = Instance.new("TextLabel")
    label.BackgroundTransparency = 1
    label.Size = UDim2.new(1, 0, 1, 0)
    label.TextColor3 = NAME_COLOR
    label.TextStrokeColor3 = NAME_STROKE
    label.TextStrokeTransparency = 0.45
    label.Font = NAME_FONT
    label.TextScaled = false
    label.TextSize = NAME_TEXT_SIZE
    label.Text = (player.DisplayName ~= "" and player.DisplayName or player.Name)
    label.Parent = bb

    return bb, label
end

local function createProxyRigFor(character)
    local proxy = Instance.new("Model")
    proxy.Name = "_ESPProxy_" .. (character:GetDebugId():gsub("%W",""))
    proxy.Parent = PROXY_FOLDER

    local map = {}
    for _, part in ipairs(collectCoreParts(character)) do
        local p = Instance.new("Part")
        p.Anchored = true
        p.CanCollide = false
        p.CanTouch = false
        p.CanQuery = false
        p.CastShadow = false
        p.Size = part.Size
        p.CFrame = part.CFrame
        p.Transparency = 0.999
        p.Material = Enum.Material.SmoothPlastic
        p.Name = "Proxy_" .. part.Name
        p.Parent = proxy
        map[part] = p
    end
    return proxy, map
end

local function setProxyActive(pkt, active)
    pkt.ProxyActive = active
    if active then
        if not pkt.ProxyRig or not pkt.ProxyRig.Parent then
            local pr, map = createProxyRigFor(pkt.Character)
            pkt.ProxyRig = pr
            pkt.ProxyMap = map
        end
    end
end

local function cleanupPlayer(player)
    if Connections[player] then
        for _, c in ipairs(Connections[player]) do pcall(function() c:Disconnect() end) end
        Connections[player] = nil
    end
    local pkt = PerPlayer[player]
    if not pkt then return end
    if pkt.Highlight then pkt.Highlight:Destroy() end
    if pkt.Billboard then pkt.Billboard:Destroy() end
    if pkt.ProxyRig then pkt.ProxyRig:Destroy() end
    PerPlayer[player] = nil
end

local function attachForCharacter(player, character)
    if player == LocalPlayer then return end
    local head = character:FindFirstChild("Head")
    local hrp  = character:FindFirstChild("HumanoidRootPart")
    if not head or not hrp then
        head = character:WaitForChild("Head", 5)
        hrp  = character:WaitForChild("HumanoidRootPart", 5)
        if not head or not hrp then return end
    end

    local hl = Instance.new("Highlight")
    hl.Name = "ESP_Highlight"
    hl.Adornee = character
    hl.FillColor = HIGHLIGHT_FILL_COLOR
    hl.FillTransparency = HIGHLIGHT_FILL_TRANSPARENCY
    hl.OutlineColor = HIGHLIGHT_OUTLINE_COLOR
    hl.OutlineTransparency = HIGHLIGHT_OUTLINE_TRANSPARENCY
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.Enabled = ESPEnabled
    hl.Parent = character

    local bb, label = makeNameBillboard(head, player)
    if not ESPEnabled then bb.Parent = nil end

    PerPlayer[player] = {
        Highlight = hl,
        Billboard = bb,
        Label = label,
        Head = head,
        HRP = hrp,
        Character = character,
        ProxyRig = nil,
        ProxyMap = nil,
        ProxyActive = false,
        CoreParts = collectCoreParts(character),
        LastShownMeters = nil,
    }
end

local function enableESPForPlayer(player, enable)
    local pkt = PerPlayer[player]
    if not pkt then return end
    if pkt.Highlight then pkt.Highlight.Enabled = enable end
    if pkt.Billboard then
        local ok = pcall(function() pkt.Billboard.Enabled = enable end)
        if not ok then
            if enable and pkt.Head and pkt.Billboard.Parent ~= pkt.Head then
                pkt.Billboard.Parent = pkt.Head
            elseif not enable then
                pkt.Billboard.Parent = nil
            end
        end
    end
    if not enable then setProxyActive(pkt, false) end
end

------------------------------------------------------------
-- First-floor RemainingTime (ONE per plot)
------------------------------------------------------------
local PLOTS = workspace:FindFirstChild("Plots")
local FIRST_FLOOR_MAX_DISTANCE = 99999
local FIRST_FLOOR_SIZE = UDim2.new(30, 0, 30, 0)
local plotState, plotNextAllowed, PLOT_COOLDOWN = {}, {}, 0.15

local function refreshSinglePlot(plot)
    local candidates, lowestY, lowestGui = {}, math.huge, nil
    for _, d in ipairs(plot:GetDescendants()) do
        if d:IsA("TextLabel") and d.Name == "RemainingTime" then
            local gui = d.Parent
            if gui and gui:IsA("BillboardGui") then
                table.insert(candidates, gui)
                local y
                if gui.Adornee and gui.Adornee:IsA("BasePart") then
                    y = gui.Adornee.Position.Y
                elseif gui.Parent and gui.Parent:IsA("BasePart") then
                    y = gui.Parent.Position.Y
                end
                if y and y < lowestY then lowestY, lowestGui = y, gui end
            end
        end
    end
    for _, gui in ipairs(candidates) do
        if gui == lowestGui then
            gui.Enabled = ESPEnabled
            gui.MaxDistance = FIRST_FLOOR_MAX_DISTANCE
            gui.AlwaysOnTop = true
            gui.Size = FIRST_FLOOR_SIZE
        else
            gui.Enabled = false
        end
    end
    plotState[plot] = plotState[plot] or {}
    plotState[plot].selected = lowestGui
end

local function schedulePlotRefresh(plot)
    if not PLOTS then return end
    local now = os.clock()
    local nextAllowed = plotNextAllowed[plot] or 0
    if now < nextAllowed then return end
    plotNextAllowed[plot] = now + PLOT_COOLDOWN
    task.defer(function()
        if ESPEnabled and plot.Parent == PLOTS then refreshSinglePlot(plot) end
    end)
end

local function firstFloorHideAll()
    if not PLOTS then return end
    for _, plot in ipairs(PLOTS:GetChildren()) do
        local st = plotState[plot]
        if st and st.selected and st.selected.Parent then st.selected.Enabled = false end
    end
end

local function firstFloorInit()
    if not PLOTS then return end
    local kids = PLOTS:GetChildren()
    for i = 1, #kids do
        refreshSinglePlot(kids[i])
        if (i % 8) == 0 then task.wait() end
    end
    PLOTS.ChildAdded:Connect(function(ch) schedulePlotRefresh(ch) end)
    PLOTS.ChildRemoved:Connect(function(ch) plotState[ch] = nil end)
    PLOTS.DescendantAdded:Connect(function(inst)
        local plot = inst:FindFirstAncestorOfClass("Model") or inst:FindFirstAncestorWhichIsA("Folder")
        if plot and plot.Parent == PLOTS then schedulePlotRefresh(plot) end
    end)
    PLOTS.DescendantRemoving:Connect(function(inst)
        local plot = inst:FindFirstAncestorOfClass("Model") or inst:FindFirstAncestorWhichIsA("Folder")
        if plot and plot.Parent == PLOTS then schedulePlotRefresh(plot) end
    end)
end
if PLOTS then task.spawn(firstFloorInit) end

------------------------------------------------------------
-- Highest-Earnings Brainrot — bold, top-anchored, robust reselection
------------------------------------------------------------
local BR_GUI_NAME = "EarningsESP"
local BR_VALUE_NAMES = {
    "MoneyPerSecond","CashPerSecond","EarningsPerSecond","IncomePerSecond",
    "PerSecond","CashRate","Rate","PerSec","IncomeRate",
}
local BR_PER_MINUTE_NAMES = { "MoneyPerMinute","CashPerMinute","EarningsPerMinute","IncomePerMinute" }
local BR_EXCLUDE_LABEL_NAMES = { RemainingTime=true, Timer=true, Countdown=true, TimeLeft=true }
local BR_EXCLUDE_MODEL_SUBSTR = { "timer","time","countdown","remainingtime","clock" }

local BR_candidatesByRoot = {}  -- [Model] = rec
local BR_candidateList = {}     -- list of rec
local BR_currentTop = nil       -- rec
local BR_highlight = nil
local BR_espByPart = {}         -- [BasePart] = BillboardGui

-- perf knobs
local BR_Enabled = true
local BR_recomputeQueued = false
local BR_nextRecomputeAt = 0
local BR_RECOMPUTE_DELAY = 0.10
local BR_WATCHDOG_INTERVAL = 0.6
local BR_STALE_TTL = 8.0

-- ---------- helpers ----------
local function looksTimerName(name)
    local n = (type(name)=="string" and string.lower(name) or "")
    for _, sub in ipairs(BR_EXCLUDE_MODEL_SUBSTR) do
        if string.find(n, sub, 1, true) then return true end
    end
    return false
end

local RATE_TEXT_PATTERNS = { "/s","/sec"," per s"," per sec"," per-second"," per second"," p/s","/ 1s"," per 1s" }
local MIN_TEXT_PATTERNS  = { "/min"," per min"," per-minute"," per minute" }
local function textLooksRate(t)
    if type(t) ~= "string" then return false end
    local l = string.lower(t)
    for _, pat in ipairs(RATE_TEXT_PATTERNS) do if string.find(l, pat, 1, true) then return true end end
    for _, pat in ipairs(MIN_TEXT_PATTERNS)  do if string.find(l, pat, 1, true) then return true end end
    return false
end

local function parseNumberWithSuffix(text)
    if type(text)~="string" then return nil end
    local cleaned = text:gsub("%$", ""):gsub(",", ""):gsub("%s","")
    cleaned = cleaned:gsub("/sec",""):gsub("/s",""):gsub("persecond",""):gsub("persec","")
    cleaned = cleaned:gsub("/min",""):gsub("perminute","")
    local numPart, suffix = cleaned:match("([%d%.]+)%s*([KMBTkmbt]?)")
    if not numPart then return nil end
    local num = tonumber(numPart); if not num then return nil end
    suffix = (suffix or ""):upper()
    if     suffix=="K" then num = num*1e3
    elseif suffix=="M" then num = num*1e6
    elseif suffix=="B" then num = num*1e9
    elseif suffix=="T" then num = num*1e12 end
    return num
end

local function formatCurrencyPerSec(v)
    local n = tonumber(v) or 0
    if n>=1e12 then return string.format("$%.1fT/s", n/1e12)
    elseif n>=1e9 then return string.format("$%.1fB/s", n/1e9)
    elseif n>=1e6 then return string.format("$%.1fM/s", n/1e6)
    elseif n>=1e3 then return string.format("$%.1fK/s", n/1e3)
    else return string.format("$%d/s", math.floor(n+0.5)) end
end

local function findAnyBasePartFrom(inst)
    if not inst then return nil end
    if inst:IsA("BasePart") then return inst end
    if inst:IsA("BillboardGui") then
        if inst.Adornee and inst.Adornee:IsA("BasePart") then return inst.Adornee end
        if inst.Parent and inst.Parent:IsA("BasePart") then return inst.Parent end
    end
    local model = inst:FindFirstAncestorOfClass("Model") or (inst:IsA("Model") and inst)
    if model and model:IsA("Model") then
        if model.PrimaryPart and model.PrimaryPart:IsA("BasePart") then return model.PrimaryPart end
        for _, d in ipairs(model:GetDescendants()) do if d:IsA("BasePart") then return d end end
    end
    local cursor = inst.Parent
    while cursor do
        if cursor:IsA("BasePart") then return cursor end
        cursor = cursor.Parent
    end
    return nil
end

local function ensureHighlight()
    if BR_highlight and BR_highlight.Parent then return BR_highlight end
    local h = Instance.new("Highlight")
    h.Name = "TopBrainrotHighlight"
    h.FillTransparency = 1
    h.OutlineColor = Color3.fromRGB(0,162,255)
    h.OutlineTransparency = 0
    h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    h.Parent = workspace
    BR_highlight = h
    return h
end

-- placement directly above the machine
local function computeTopWorldPos(modelForBounds, defaultPart)
    local margin = 1.4
    local ok, cf, size = pcall(modelForBounds.GetBoundingBox, modelForBounds)
    if ok and cf then return cf.Position + Vector3.new(0, size.Y/2 + margin, 0) end
    return defaultPart.Position + Vector3.new(0, 3.5, 0)
end

local function getOrMakeTopAttachment(modelForBounds, basePart)
    local att = basePart:FindFirstChild("BR_TopAnchor")
    if not att then
        att = Instance.new("Attachment")
        att.Name = "BR_TopAnchor"
        att.Parent = basePart
    end
    local worldPos = computeTopWorldPos(modelForBounds, basePart)
    att.CFrame = basePart.CFrame:ToObjectSpace(CFrame.new(worldPos))
    return att
end

local function ensureESP(boundsModel, anchorPart)
    local gui = BR_espByPart[anchorPart]
    local att = getOrMakeTopAttachment(boundsModel, anchorPart)
    if gui and gui.Parent then
        gui.Adornee = att
        return gui
    end

    gui = Instance.new("BillboardGui")
    gui.Name = BR_GUI_NAME
    gui.AlwaysOnTop = true
    gui.Size = UDim2.new(0, 260, 0, 70)
    gui.MaxDistance = 999999
    gui.Adornee = att
    gui.Parent = anchorPart

    local frame = Instance.new("Frame")
    frame.BackgroundTransparency = 1
    frame.Size = UDim2.new(1,0,1,0)
    frame.Parent = gui

    local list = Instance.new("UIListLayout")
    list.FillDirection = Enum.FillDirection.Vertical
    list.HorizontalAlignment = Enum.HorizontalAlignment.Center
    list.VerticalAlignment = Enum.VerticalAlignment.Center
    list.SortOrder = Enum.SortOrder.LayoutOrder
    list.Parent = frame

    local title = Instance.new("TextLabel")
    title.Name = "Title"
    title.BackgroundTransparency = 1
    title.Size = UDim2.new(1, 0, 0.55, 0)
    title.TextScaled = true
    title.Font = Enum.Font.GothamBlack
    title.TextColor3 = Color3.fromRGB(255, 64, 64) -- RED
    title.TextStrokeColor3 = Color3.new(0,0,0)
    title.TextStrokeTransparency = 0
    local tStroke = Instance.new("UIStroke")
    tStroke.Thickness = 1.6
    tStroke.Color = Color3.new(0,0,0)
    tStroke.Parent = title
    title.Parent = frame

    local value = Instance.new("TextLabel")
    value.Name = "Value"
    value.BackgroundTransparency = 1
    value.Size = UDim2.new(1, 0, 0.45, 0)
    value.TextScaled = true
    value.Font = Enum.Font.GothamBlack
    value.TextColor3 = Color3.fromRGB(255,255,255) -- WHITE
    value.TextStrokeColor3 = Color3.new(0,0,0)
    value.TextStrokeTransparency = 0
    local vStroke = Instance.new("UIStroke")
    vStroke.Thickness = 1.6
    vStroke.Color = Color3.new(0,0,0)
    vStroke.Parent = value
    value.Parent = frame

    BR_espByPart[anchorPart] = gui
    return gui
end

local function trim(s) s = tostring(s or ""); return s:gsub("^%s+",""):gsub("%s+$","") end

local function nameFromNearbyUI(rateSrc, root)
    if not rateSrc then return nil end
    local container = rateSrc:FindFirstAncestorWhichIsA("BillboardGui") or rateSrc:FindFirstAncestorWhichIsA("GuiObject")
    local bestText, bestScore
    local function considerLabel(lbl)
        if not (lbl and (lbl:IsA("TextLabel") or lbl:IsA("TextBox"))) then return end
        if lbl == rateSrc then return end
        local t = trim(lbl.Text); if t=="" then return end
        if textLooksRate(t) then return end
        local score = #t
        local n = string.lower(lbl.Name or "")
        if n=="title" or n=="name" or n=="displayname" then score = score + 100 end
        if not bestScore or score > bestScore then bestScore, bestText = score, t end
    end
    if container then
        for _, d in ipairs(container:GetDescendants()) do
            if d:IsA("TextLabel") or d:IsA("TextBox") then considerLabel(d) end
        end
        if bestText then return bestText end
    end
    return nil
end

local function deriveDisplayName(root, rateSrc)
    for _, key in ipairs({"BrainrotName","DisplayName","Name"}) do
        local ok, v = pcall(function() return root:GetAttribute(key) end)
        if ok and type(v)=="string" then
            local t = trim(v); if t ~= "" then return t end
        end
    end
    local near = nameFromNearbyUI(rateSrc, root)
    if near and near ~= "" then return near end
    for _, d in ipairs(root:GetChildren()) do
        if d:IsA("StringValue") then
            local n = string.lower(d.Name)
            if n=="name" or n=="displayname" or n=="title" then
                local t = trim(d.Value); if t ~= "" then return t end
            end
        end
    end
    return root.Name or "Brainrot"
end

-- compute EPS & rate source
local BR_VALUE_NAMES_SET = {}; for _, n in ipairs(BR_VALUE_NAMES) do BR_VALUE_NAMES_SET[n]=true end
local function computeEPSAndSource(root)
    local best, src, srcType, srcKey = 0, nil, nil, nil
    for _, nm in ipairs(BR_VALUE_NAMES) do
        local ok, v = pcall(function() return root:GetAttribute(nm) end)
        if ok and type(v)=="number" and v>best then best,src,srcType,srcKey = v, root, "attr", nm end
    end
    for _, nm in ipairs(BR_PER_MINUTE_NAMES) do
        local ok, v = pcall(function() return root:GetAttribute(nm) end)
        if ok and type(v)=="number" and (v/60)>best then best,src,srcType,srcKey = v/60, root, "attrmin", nm end
    end
    for _, d in ipairs(root:GetDescendants()) do
        if d:IsA("NumberValue") or d:IsA("IntValue") then
            if BR_VALUE_NAMES_SET[d.Name] then
                local v = tonumber(d.Value); if v and v>best then best,src,srcType,srcKey = v, d, "num", nil end
            elseif d.Name:match("PerMinute") then
                local v = tonumber(d.Value); if v and (v/60)>best then best,src,srcType,srcKey = v/60, d, "num", nil end
            end
        elseif d:IsA("TextLabel") or d:IsA("TextBox") then
            if not BR_EXCLUDE_LABEL_NAMES[d.Name] then
                local t = d.Text
                if type(t)=="string" and t~="" and textLooksRate(t) then
                    local v = parseNumberWithSuffix(t)
                    if v then
                        local l = string.lower(t)
                        if string.find(l,"/min",1,true) or string.find(l,"per min",1,true) or string.find(l,"per minute",1,true) or string.find(l,"per-minute",1,true) then v = v/60 end
                        if v>best then best,src,srcType,srcKey = v, d, "text", nil end
                    end
                end
            end
        end
    end
    return best, src, srcType, srcKey
end

-- targeted watcher for the chosen source (low-lag)
local function attachSourceWatcher(rec)
    if rec.srcConn then pcall(function() rec.srcConn:Disconnect() end); rec.srcConn = nil end
    local function refresh()
        rec.eps, rec.rateSrc, rec.srcType, rec.srcKey = computeEPSAndSource(rec.root)
        rec.name = deriveDisplayName(rec.root, rec.rateSrc)
        rec.lastUpdate = os.clock()
        BR_recomputeQueued = false
        BR_nextRecomputeAt = 0
        BR_queueRecomputeTop()
    end
    if rec.srcType == "attr" or rec.srcType == "attrmin" then
        local sigOk, sig = pcall(function() return rec.root:GetAttributeChangedSignal(rec.srcKey) end)
        if sigOk and sig then rec.srcConn = sig:Connect(refresh) end
    elseif rec.srcType == "num" then
        if rec.rateSrc and rec.rateSrc.Parent then rec.srcConn = rec.rateSrc:GetPropertyChangedSignal("Value"):Connect(refresh) end
    elseif rec.srcType == "text" then
        if rec.rateSrc and rec.rateSrc.Parent then rec.srcConn = rec.rateSrc:GetPropertyChangedSignal("Text"):Connect(refresh) end
    end
end

local function getAnchorModelFromRateSrc(rateSrc, fallbackModel)
    if not rateSrc then return fallbackModel end
    local gui = rateSrc:FindFirstAncestorWhichIsA("BillboardGui")
    if gui then
        local bp = (gui.Adornee and gui.Adornee:IsA("BasePart") and gui.Adornee) or (gui.Parent and gui.Parent:IsA("BasePart") and gui.Parent)
        if bp then
            local m = bp:FindFirstAncestorOfClass("Model"); if m then return m end
        end
    end
    return fallbackModel
end

local function retargetPlacement(rec)
    local anchorModel = getAnchorModelFromRateSrc(rec.rateSrc, rec.root) or rec.root
    local anchorPart  = findAnyBasePartFrom(rec.rateSrc) or (anchorModel.PrimaryPart or findAnyBasePartFrom(anchorModel)) or rec.part
    rec.anchorModel, rec.anchorPart = anchorModel, anchorPart
end

local function setBrainrotESP(rec)
    for p, g in pairs(BR_espByPart) do
        if g and g.Parent then g.Enabled = BR_Enabled and (p == rec.anchorPart) else BR_espByPart[p] = nil end
    end
    local gui = ensureESP(rec.anchorModel, rec.anchorPart)
    local title = gui:FindFirstChild("Title", true)
    local value = gui:FindFirstChild("Value", true)
    if title and title:IsA("TextLabel") then title.Text = tostring(rec.name or "Brainrot") end
    if value and value:IsA("TextLabel") then value.Text = formatCurrencyPerSec(rec.eps or 0) end
    gui.Enabled = BR_Enabled
end

-- recompute top (debounced)
function BR_queueRecomputeTop()
    if BR_recomputeQueued then return end
    local now = os.clock()
    BR_recomputeQueued = true
    local delay = math.max(0, BR_nextRecomputeAt - now)
    task.delay(delay, function()
        BR_recomputeQueued = false
        BR_nextRecomputeAt = os.clock() + BR_RECOMPUTE_DELAY

        -- prune
        local now2 = os.clock()
        for i = #BR_candidateList, 1, -1 do
            local rec = BR_candidateList[i]
            if not (rec.root and rec.root.Parent) then
                if rec.srcConn then pcall(function() rec.srcConn:Disconnect() end) end
                BR_candidatesByRoot[rec.root] = nil
                table.remove(BR_candidateList, i)
            elseif rec.lastUpdate and (now2 - rec.lastUpdate > BR_STALE_TTL) and (rec.eps or 0) <= 0 then
                if rec.srcConn then pcall(function() rec.srcConn:Disconnect() end) end
                BR_candidatesByRoot[rec.root] = nil
                table.remove(BR_candidateList, i)
            end
        end

        -- choose
        local best = nil
        for _, rec in ipairs(BR_candidateList) do
            if rec.root and rec.root.Parent and (rec.eps or 0) > 0 then
                if (not best) or (rec.eps or 0) > (best.eps or 0) then best = rec end
            end
        end

        if best ~= BR_currentTop then
            BR_currentTop = best
            if BR_currentTop then
                retargetPlacement(BR_currentTop)
                setBrainrotESP(BR_currentTop)
                local h = ensureHighlight(); h.Adornee = BR_currentTop.anchorModel; h.Enabled = BR_Enabled
            else
                for _, g in pairs(BR_espByPart) do if g and g.Parent then g.Enabled = false end end
                if BR_highlight then BR_highlight.Adornee = nil end
            end
        elseif BR_currentTop then
            retargetPlacement(BR_currentTop)
            setBrainrotESP(BR_currentTop)
            local h = ensureHighlight(); h.Adornee = BR_currentTop.anchorModel; h.Enabled = BR_Enabled
        end
    end)
end

local function BR_connectRootSignals(rec)
    rec.conns = rec.conns or {}

    table.insert(rec.conns, rec.root.DescendantAdded:Connect(function(d)
        if d:IsA("NumberValue") or d:IsA("IntValue") or d:IsA("TextLabel") or d:IsA("TextBox") then
            rec.eps, rec.rateSrc, rec.srcType, rec.srcKey = computeEPSAndSource(rec.root)
            rec.name = deriveDisplayName(rec.root, rec.rateSrc)
            rec.lastUpdate = os.clock()
            attachSourceWatcher(rec)
            BR_queueRecomputeTop()
        end
    end))

    table.insert(rec.conns, rec.root.DescendantRemoving:Connect(function(d)
        if d == rec.rateSrc then
            rec.eps, rec.rateSrc, rec.srcType, rec.srcKey = computeEPSAndSource(rec.root)
            rec.name = deriveDisplayName(rec.root, rec.rateSrc)
            rec.lastUpdate = os.clock()
            attachSourceWatcher(rec)
            BR_queueRecomputeTop()
        end
    end))

    table.insert(rec.conns, rec.root.AncestryChanged:Connect(function()
        if not rec.root:IsDescendantOf(workspace) then
            if rec.srcConn then pcall(function() rec.srcConn:Disconnect() end) end
            BR_candidatesByRoot[rec.root] = nil
            for i = #BR_candidateList, 1, -1 do if BR_candidateList[i] == rec then table.remove(BR_candidateList, i) break end end
            if BR_currentTop == rec then BR_currentTop = nil end
            BR_queueRecomputeTop()
        end
    end))

    attachSourceWatcher(rec)
end

local function BR_registerRoot(root)
    if not root or BR_candidatesByRoot[root] then return end
    if looksTimerName(root.Name) then return end

    local part = findAnyBasePartFrom(root)
    if not part then return end

    local eps, src, srcType, srcKey = computeEPSAndSource(root)
    -- Register if we found any plausible *source* (even if eps==0) so future changes are caught
    if not src then return end

    local rec = {
        root = root, part = part,
        eps = eps or 0, rateSrc = src, srcType = srcType, srcKey = srcKey,
        name = deriveDisplayName(root, src),
        conns = {}, srcConn = nil, lastUpdate = os.clock(),
        anchorModel = nil, anchorPart = nil,
    }
    retargetPlacement(rec)
    BR_candidatesByRoot[root] = rec
    table.insert(BR_candidateList, rec)
    BR_connectRootSignals(rec)
    BR_queueRecomputeTop()
end

local function BR_tryRegisterFromGui(gui)
    if not gui or not gui:IsA("BillboardGui") then return end
    if looksTimerName(gui.Name) then return end
    local hasRate = false
    for _, d in ipairs(gui:GetDescendants()) do
        if (d:IsA("TextLabel") or d:IsA("TextBox")) and not BR_EXCLUDE_LABEL_NAMES[d.Name] then
            if textLooksRate(d.Text) then hasRate = true break end
        end
    end
    if not hasRate then return end
    local anchor = findAnyBasePartFrom(gui); if not anchor then return end
    local root = anchor:FindFirstAncestorOfClass("Model"); if not root then return end
    BR_registerRoot(root)
end

local function BR_initialScan(searchRoot)
    -- Register via GUIs that show rates (fast & reliable)
    for _, inst in ipairs(searchRoot:GetDescendants()) do
        if inst:IsA("BillboardGui") then BR_tryRegisterFromGui(inst) end
    end
    -- Extra: models that already expose a rate signal (attributes/values)
    for _, inst in ipairs(searchRoot:GetDescendants()) do
        if inst:IsA("Model") and not looksTimerName(inst.Name) then
            local eps, src = computeEPSAndSource(inst)
            if src then BR_registerRoot(inst) end
        end
    end
end

local function BR_startIncremental(root)
    root.DescendantAdded:Connect(function(inst)
        if inst:IsA("BillboardGui") then
            BR_tryRegisterFromGui(inst)
        elseif inst:IsA("Model") and not looksTimerName(inst.Name) then
            local eps, src = computeEPSAndSource(inst)
            if src then BR_registerRoot(inst) end
        end
    end)
    root.DescendantRemoving:Connect(function(inst)
        local r = inst:IsA("Model") and inst or inst:FindFirstAncestorOfClass("Model")
        if r and BR_candidatesByRoot[r] then
            local rec = BR_candidatesByRoot[r]
            if rec.srcConn then pcall(function() rec.srcConn:Disconnect() end) end
            BR_candidatesByRoot[r] = nil
            for i=#BR_candidateList,1,-1 do if BR_candidateList[i]==rec then table.remove(BR_candidateList,i) break end end
            if BR_currentTop == rec then BR_currentTop = nil end
            BR_queueRecomputeTop()
        end
    end)
end

local function BR_SetEnabled(on)
    BR_Enabled = on and true or false
    if BR_highlight then
        BR_highlight.Enabled = BR_Enabled
        if not BR_Enabled then BR_highlight.Adornee = nil end
    end
    for _, g in pairs(BR_espByPart) do
        if g and g.Parent then g.Enabled = BR_Enabled end
    end
    if BR_Enabled then BR_queueRecomputeTop() end
end

_G = _G or {}
_G.Brainrot_SetEnabled = BR_SetEnabled

local function BR_init()
    local searchRoot = workspace:FindFirstChild("Plots") or workspace
    task.spawn(function() BR_initialScan(searchRoot); BR_queueRecomputeTop() end)
    BR_startIncremental(searchRoot)
    BR_SetEnabled(ESPEnabled)

    -- Watchdog: keep anchor pinned and reselect if top vanishes or drops to 0
    task.spawn(function()
        while true do
            task.wait(BR_WATCHDOG_INTERVAL)
            if not BR_Enabled then continue end

            for p, g in pairs(BR_espByPart) do
                if (not p) or (not p.Parent) or (not g.Parent) then BR_espByPart[p] = nil end
            end

            local top = BR_currentTop
            if not top then
                BR_queueRecomputeTop()
            else
                if (not top.root) or (not top.root.Parent) then
                    BR_currentTop = nil
                    BR_queueRecomputeTop()
                else
                    local epsNow, srcNow, typeNow, keyNow = computeEPSAndSource(top.root)
                    if (epsNow or 0) <= 0 then
                        BR_currentTop = nil
                        BR_queueRecomputeTop()
                    else
                        if srcNow ~= top.rateSrc or typeNow ~= top.srcType or keyNow ~= top.srcKey then
                            top.eps, top.rateSrc, top.srcType, top.srcKey = epsNow, srcNow, typeNow, keyNow
                            top.name = deriveDisplayName(top.root, top.rateSrc)
                            top.lastUpdate = os.clock()
                            attachSourceWatcher(top)
                        else
                            top.eps = epsNow
                            top.lastUpdate = os.clock()
                        end
                        retargetPlacement(top)
                        local gui = BR_espByPart[top.anchorPart]
                        if gui then
                            gui.Adornee = getOrMakeTopAttachment(top.anchorModel, top.anchorPart)
                            local v = gui:FindFirstChild("Value", true)
                            if v and v:IsA("TextLabel") then
                                local txt = formatCurrencyPerSec(top.eps or 0)
                                if v.Text ~= txt then v.Text = txt end
                            end
                            local t = gui:FindFirstChild("Title", true)
                            if t and t:IsA("TextLabel") and t.Text ~= tostring(top.name or "Brainrot") then
                                t.Text = tostring(top.name or "Brainrot")
                            end
                        end
                    end
                end
            end
        end
    end)
end
pcall(BR_init)

------------------------------------------------------------
-- Player wiring
------------------------------------------------------------
local function onPlayerAdded(player)
    local nameConn = player:GetPropertyChangedSignal("DisplayName"):Connect(function()
        local pkt = PerPlayer[player]
        if pkt and pkt.Label then
            local base = (player.DisplayName ~= "" and player.DisplayName or player.Name)
            if pkt.LastShownMeters then
                pkt.Label.Text = base .. "  [" .. pkt.LastShownMeters .. "m]"
            else
                pkt.Label.Text = base
            end
        end
    end)

    local charConn = player.CharacterAdded:Connect(function(char)
        cleanupPlayer(player)
        task.defer(function()
            attachForCharacter(player, char)
            enableESPForPlayer(player, ESPEnabled)
        end)
    end)

    Connections[player] = Connections[player] or {}
    table.insert(Connections[player], nameConn)
    table.insert(Connections[player], charConn)

    if player.Character then
        attachForCharacter(player, player.Character)
        enableESPForPlayer(player, ESPEnabled)
    end
end

local function onPlayerRemoving(player)
    cleanupPlayer(player)
end

for _, plr in ipairs(Players:GetPlayers()) do
    if plr ~= LocalPlayer then onPlayerAdded(plr) end
end
Players.PlayerAdded:Connect(onPlayerAdded)
Players.PlayerRemoving:Connect(onPlayerRemoving)

-- Distance + invisibility updater (0.35s cadence)
task.spawn(function()
    while true do
        task.wait(0.35)
        if not ESPEnabled then continue end

        local lpChar = LocalPlayer.Character
        local lpRoot = lpChar and lpChar:FindFirstChild("HumanoidRootPart")

        for plr, pkt in pairs(PerPlayer) do
            if lpRoot and pkt and pkt.Label and pkt.HRP then
                local meters = math.floor((lpRoot.Position - pkt.HRP.Position).Magnitude + 0.5)
                if pkt.LastShownMeters ~= meters then
                    pkt.LastShownMeters = meters
                    local base = (plr.DisplayName ~= "" and plr.DisplayName or plr.Name)
                    pkt.Label.Text = string.format("%s  [%dm]", base, meters)
                end
            end

            local char = pkt.Character
            if char and pkt.Highlight and pkt.CoreParts then
                local invisible = isCharacterInvisibleFast(pkt.CoreParts)
                if invisible then
                    if not pkt.ProxyActive then setProxyActive(pkt, true) end
                    if pkt.ProxyRig and pkt.Highlight.Adornee ~= pkt.ProxyRig then
                        pkt.Highlight.Adornee = pkt.ProxyRig
                    end
                else
                    if pkt.Highlight.Adornee ~= char then pkt.Highlight.Adornee = char end
                    if pkt.ProxyActive then setProxyActive(pkt, false) end
                end
            end
        end
    end
end)

-- Throttled proxy sync (20 Hz)
local PROXY_UPDATE_INTERVAL = 1/20
local accum = 0
RunService.Heartbeat:Connect(function(dt)
    accum += dt
    if accum >= PROXY_UPDATE_INTERVAL then
        accum -= PROXY_UPDATE_INTERVAL
        for _, pkt in pairs(PerPlayer) do
            if pkt and pkt.ProxyActive and pkt.ProxyRig and pkt.ProxyMap and pkt.Character then
                for src, proxyPart in pairs(pkt.ProxyMap) do
                    if src.Parent and proxyPart.Parent then
                        local cf = src.CFrame
                        if proxyPart.CFrame ~= cf then proxyPart.CFrame = cf end
                        local sz = src.Size
                        if proxyPart.Size ~= sz then proxyPart.Size = sz end
                    end
                end
            end
        end
    end
end)

------------------------------------------------------------
-- Unified Toggle (GUI + hotkey) — includes Brainrot
------------------------------------------------------------
local function setESPEnabled(on)
    ESPEnabled = on
    if on then
        toggleBtn.Text = "ON";  toggleBtn.BackgroundColor3 = PANEL_ACCENT
    else
        toggleBtn.Text = "OFF"; toggleBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 78)
    end

    for plr, _ in pairs(PerPlayer) do enableESPForPlayer(plr, on) end

    if PLOTS then
        if on then for _, plot in ipairs(PLOTS:GetChildren()) do schedulePlotRefresh(plot) end
        else firstFloorHideAll() end
    end

    if _G and _G.Brainrot_SetEnabled then _G.Brainrot_SetEnabled(on) end
end

toggleBtn.MouseButton1Click:Connect(function() setESPEnabled(not ESPEnabled) end)
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == HOTKEY_TOGGLE then setESPEnabled(not ESPEnabled) end
end)

toggleBtn.Text = "ON"; toggleBtn.BackgroundColor3 = PANEL_ACCENT
_G.SetESPEnabled = function(on) setESPEnabled(on and true or false) end
