-- Solara-Compatible GUI with Title, ESP, Silent Aim, No Fog & Fullbright (with loops)
local Workspace   = game:GetService("Workspace")
local Lighting    = game:GetService("Lighting")
local CoreGui     = game:GetService("CoreGui")
local Players     = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

--=== GUI SETUP ===--
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name   = "ThoseWhoRemainGui"
ScreenGui.Parent = CoreGui

-- Title Label
local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size                   = UDim2.new(0, 300, 0, 30)
TitleLabel.Position               = UDim2.new(0, 10, 0, 10)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text                   = "Those Who Remain  –  By Avoid"
TitleLabel.Font                   = Enum.Font.GothamBold
TitleLabel.TextSize               = 20
TitleLabel.TextColor3             = Color3.new(1,1,1)
TitleLabel.Parent                 = ScreenGui

-- Button factory
local function makeButton(name, y)
    local b = Instance.new("TextButton")
    b.Size             = UDim2.new(0, 140, 0, 30)
    b.Position         = UDim2.new(0, 10, 0, y)
    b.Text             = name .. ": OFF"
    b.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    b.TextColor3       = Color3.new(1,1,1)
    b.Font             = Enum.Font.Code
    b.TextSize         = 16
    b.AutoButtonColor  = true
    b.Parent           = ScreenGui
    return b
end

--=== ESP ===--
local espBtn     = makeButton("ESP", 50)
local espEnabled = false
local espCon
local espTargets = {
    Ammo          = Color3.fromRGB(255, 255, 255),
    Medkit        = Color3.fromRGB(255,   0,   0),
    ["Body Armor"] = Color3.fromRGB(  0, 255, 255),
    ["Gas Mask"]   = Color3.fromRGB(150, 255, 150),
}

local function clearESP()
    local itemsHolder = Workspace:FindFirstChild("Ignore")
                        and Workspace.Ignore:FindFirstChild("Items")
    if not itemsHolder then return end
    for _, it in ipairs(itemsHolder:GetChildren()) do
        local h = it:FindFirstChildWhichIsA("Highlight")
        if h then h:Destroy() end
    end
    if espCon then espCon:Disconnect() end
    espCon = nil
end

local function updateESP()
    local itemsHolder = Workspace:WaitForChild("Ignore")
                         :WaitForChild("Items")
    for _, it in ipairs(itemsHolder:GetChildren()) do
        if espTargets[it.Name] and not it:FindFirstChildOfClass("Highlight") then
            local hl = Instance.new("Highlight")
            hl.FillColor        = espTargets[it.Name]
            hl.OutlineColor     = espTargets[it.Name]
            hl.FillTransparency = 0.2
            hl.OutlineTransparency = 0.5
            hl.Adornee          = it
            hl.Parent           = it
        end
    end
    espCon = itemsHolder.ChildAdded:Connect(function(it)
        task.wait(0.2)
        if espTargets[it.Name] then
            local hl = Instance.new("Highlight")
            hl.FillColor        = espTargets[it.Name]
            hl.OutlineColor     = espTargets[it.Name]
            hl.FillTransparency = 0.2
            hl.OutlineTransparency = 0.5
            hl.Adornee          = it
            hl.Parent           = it
        end
    end)
end

espBtn.MouseButton1Click:Connect(function()
    espEnabled = not espEnabled
    espBtn.Text = "ESP: " .. (espEnabled and "ON" or "OFF")
    clearESP()
    if espEnabled then updateESP() end
end)

--=== SILENT AIM ===--
local saBtn     = makeButton("Silent Aim", 90)
local saEnabled = false
local saThread

local function startSilentAim()
    local folder = Workspace:FindFirstChild("Entities")
                   and Workspace.Entities:FindFirstChild("Infected")
    if not folder then return warn("Entities.Infected not found") end
    saThread = task.spawn(function()
        while saEnabled do
            for _, ent in ipairs(folder:GetChildren()) do
                if ent:IsA("Model") and ent:FindFirstChild("Head") then
                    pcall(function()
                        local head = ent.Head
                        head.Size         = Vector3.new(100,100,100)
                        head.CanCollide   = false
                        head.Material     = Enum.Material.ForceField
                        head.BrickColor   = BrickColor.new("Really red")
                        head.Transparency = 0.4
                        head.Massless     = true
                    end)
                end
            end
            task.wait(0.5)
        end
    end)
end

saBtn.MouseButton1Click:Connect(function()
    saEnabled = not saEnabled
    saBtn.Text = "Silent Aim: " .. (saEnabled and "ON" or "OFF")
    if saEnabled then startSilentAim() else saEnabled = false end
end)

--=== NO FOG (with loop) ===--
local fogBtn     = makeButton("No Fog", 130)
local fogEnabled = false
local fogThread

local originalFogStart = Lighting.FogStart
local originalFogEnd   = Lighting.FogEnd

fogBtn.MouseButton1Click:Connect(function()
    fogEnabled = not fogEnabled
    fogBtn.Text = "No Fog: " .. (fogEnabled and "ON" or "OFF")
    if fogEnabled then
        fogThread = task.spawn(function()
            while fogEnabled do
                Lighting.FogStart = 0
                Lighting.FogEnd   = 1e6
                for _, v in ipairs(Lighting:GetDescendants()) do
                    if v:IsA("Atmosphere") then v:Destroy() end
                end
                task.wait(1)
            end
        end)
    else
        fogEnabled = false
        if fogThread then task.cancel(fogThread); fogThread = nil end
        Lighting.FogStart = originalFogStart
        Lighting.FogEnd   = originalFogEnd
    end
end)

--=== FULLBRIGHT (with loop) ===--
local fbBtn       = makeButton("Fullbright", 170)
local fbEnabled   = false
local fbThread

local originalProps = {
    Ambient        = Lighting.Ambient,
    OutdoorAmbient = Lighting.OutdoorAmbient,
    Brightness     = Lighting.Brightness,
    GlobalShadows  = Lighting.GlobalShadows
}
local effectStates = {}
for _, eff in ipairs(Lighting:GetDescendants()) do
    if eff:IsA("ColorCorrectionEffect")
    or eff:IsA("BloomEffect")
    or eff:IsA("SunRaysEffect")
    or eff:IsA("DepthOfFieldEffect")
    or eff:IsA("BlurEffect") then
        effectStates[eff] = eff.Enabled
    end
end

fbBtn.MouseButton1Click:Connect(function()
    fbEnabled = not fbEnabled
    fbBtn.Text = "Fullbright: " .. (fbEnabled and "ON" or "OFF")
    if fbEnabled then
        fbThread = task.spawn(function()
            while fbEnabled do
                Lighting.Ambient        = Color3.new(1,1,1)
                Lighting.OutdoorAmbient = Color3.new(1,1,1)
                Lighting.Brightness     = 2
                Lighting.GlobalShadows  = false
                for eff, _ in pairs(effectStates) do
                    if eff and eff.Parent then eff.Enabled = false end
                end
                task.wait(1)
            end
        end)
    else
        fbEnabled = false
        if fbThread then task.cancel(fbThread); fbThread = nil end
        Lighting.Ambient        = originalProps.Ambient
        Lighting.OutdoorAmbient = originalProps.OutdoorAmbient
        Lighting.Brightness     = originalProps.Brightness
        Lighting.GlobalShadows  = originalProps.GlobalShadows
        for eff, state in pairs(effectStates) do
            if eff and eff.Parent then eff.Enabled = state end
        end
    end
end)
