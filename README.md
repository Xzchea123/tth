local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local CoreGui = game:GetService("CoreGui")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

-- Destroy existing UI if already there
if CoreGui:FindFirstChild("TWR_GUI") then
    CoreGui.TWR_GUI:Destroy()
end

local gui = Instance.new("ScreenGui")
gui.Name = "TWR_GUI"
gui.Parent = CoreGui
gui.ResetOnSpawn = false

-- Main Frame
local mainFrame = Instance.new("Frame")
mainFrame.Parent = gui
mainFrame.Size = UDim2.new(0, 320, 0, 370)
mainFrame.Position = UDim2.new(0, 100, 0, 100)
mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
mainFrame.BorderSizePixel = 0
mainFrame.ClipsDescendants = true

-- UICorner for rounded frame
local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 8)
corner.Parent = mainFrame

-- Title Bar
local titleBar = Instance.new("TextLabel")
titleBar.Parent = mainFrame
titleBar.Size = UDim2.new(1, 0, 0, 30)
titleBar.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
titleBar.Text = "Those Who Remain – Avoid"
titleBar.Font = Enum.Font.GothamBold
titleBar.TextSize = 18
titleBar.TextColor3 = Color3.fromRGB(255, 255, 255)
titleBar.TextXAlignment = Enum.TextXAlignment.Center

-- Dragging
local dragging, dragInput, dragStart, startPos
titleBar.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = true
		dragStart = input.Position
		startPos = mainFrame.Position
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then dragging = false end
		end)
	end
end)
titleBar.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement then dragInput = input end
end)
UserInputService.InputChanged:Connect(function(input)
	if input == dragInput and dragging then
		local delta = input.Position - dragStart
		mainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
	end
end)

-- Toggle Button Factory
local function createToggle(name, yOffset)
	local btn = Instance.new("TextButton")
	btn.Parent = mainFrame
	btn.Size = UDim2.new(0, 140, 0, 30)
	btn.Position = UDim2.new(0, 10, 0, yOffset)
	btn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	btn.Text = name .. ": OFF"
	btn.Font = Enum.Font.Code
	btn.TextColor3 = Color3.new(1,1,1)
	btn.TextSize = 14
	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 6)
	corner.Parent = btn
	return btn
end

-- Slider Factory - Returns [label, slider, fill]
local function createSlider(name, yOffset, min, max, default, callback)
	local label = Instance.new("TextLabel")
	label.Parent = mainFrame
	label.Position = UDim2.new(0, 10, 0, yOffset)
	label.Size = UDim2.new(0, 140, 0, 20)
	label.Text = name .. ": " .. tostring(default)
	label.Font = Enum.Font.Code
	label.TextSize = 14
	label.TextColor3 = Color3.new(1, 1, 1)
	label.BackgroundTransparency = 1

	local slider = Instance.new("Frame")
	slider.Parent = mainFrame
	slider.Position = UDim2.new(0, 10, 0, yOffset + 20)
	slider.Size = UDim2.new(0, 140, 0, 8)
	slider.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
	Instance.new("UICorner", slider).CornerRadius = UDim.new(0, 5)

	local fill = Instance.new("Frame")
	fill.Parent = slider
	fill.Size = UDim2.new((default - min)/(max - min), 0, 1, 0)
	fill.BackgroundColor3 = Color3.fromRGB(120, 120, 255)
	fill.BorderSizePixel = 0
	fill.ZIndex = 2
	Instance.new("UICorner", fill).CornerRadius = UDim.new(0, 5)

	local dragging = false
	slider.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			dragging = true
		end
	end)
	slider.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			dragging = false
		end
	end)
	UserInputService.InputChanged:Connect(function(input)
		if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
			local scale = math.clamp((input.Position.X - slider.AbsolutePosition.X) / slider.AbsoluteSize.X, 0, 1)
			local val = math.floor(min + (max - min) * scale)
			fill.Size = UDim2.new(scale, 0, 1, 0)
			label.Text = name .. ": " .. val
			callback(val)
		end
	end)
	return label, slider, fill
end

local espBtn = createToggle("ESP", 40)
local espEnabled = false
local espTargets = {
    Ammo = Color3.fromRGB(255, 255, 255),
    Medkit = Color3.fromRGB(255, 0, 0),
    ["Body Armor"] = Color3.fromRGB(0, 255, 255),
    ["Gas Mask"] = Color3.fromRGB(150, 255, 150),
}
local itemConnections = {}

local function addESP(item)
    if espTargets[item.Name] and not item:FindFirstChildOfClass("Highlight") then
        local hl = Instance.new("Highlight")
        hl.FillColor = espTargets[item.Name]
        hl.OutlineColor = espTargets[item.Name]
        hl.FillTransparency = 0.2
        hl.OutlineTransparency = 0.5
        hl.Adornee = item
        hl.Parent = item
    end
end

local function removeESP(item)
    for _, h in pairs(item:GetChildren()) do
        if h:IsA("Highlight") then h:Destroy() end
    end
end

local function clearAllESP()
    -- Disconnect all connections
    for _, con in pairs(itemConnections) do
        con:Disconnect()
    end
    table.clear(itemConnections)
    -- Remove all highlights
    local items = Workspace:FindFirstChild("Ignore") and Workspace.Ignore:FindFirstChild("Items")
    if items then
        for _, item in pairs(items:GetChildren()) do
            removeESP(item)
        end
    end
end

local function enableESP()
    local items = Workspace:FindFirstChild("Ignore") and Workspace.Ignore:FindFirstChild("Items")
    if not items then return end
    -- Add ESP to all current items
    for _, item in pairs(items:GetChildren()) do
        addESP(item)
    end
    -- Listen for new items
    itemConnections["childAdded"] = items.ChildAdded:Connect(function(item)
        addESP(item)
    end)
    -- Clean up ESP when items are removed
    itemConnections["childRemoved"] = items.ChildRemoved:Connect(function(item)
        removeESP(item)
    end)
end

espBtn.MouseButton1Click:Connect(function()
    espEnabled = not espEnabled
    espBtn.Text = "ESP: " .. (espEnabled and "ON" or "OFF")
    clearAllESP()
    if espEnabled then
        enableESP()
    end
end)
-- Silent Aim
local saBtn = createToggle("Silent Aim", 80)
local saEnabled = false
local saTransparency = 0.4
local saLoop

local function applySilentAim()
	local folder = Workspace:FindFirstChild("Entities") and Workspace.Entities:FindFirstChild("Infected")
	if not folder then return end
	for _, ent in pairs(folder:GetChildren()) do
		if ent:IsA("Model") and ent:FindFirstChild("Head") then
			local head = ent.Head
			head.Size = Vector3.new(100, 100, 100)
			head.CanCollide = false
			head.Material = Enum.Material.Neon
			head.BrickColor = BrickColor.new("Institutional white")
			head.Transparency = saTransparency
			head.Massless = true
		end
	end
end

saBtn.MouseButton1Click:Connect(function()
	saEnabled = not saEnabled
	saBtn.Text = "Silent Aim: " .. (saEnabled and "ON" or "OFF")
	if saLoop then saLoop:Disconnect() saLoop = nil end
	if saEnabled then
		saLoop = RunService.Heartbeat:Connect(function()
			applySilentAim()
		end)
	else
		-- Reset heads to invisible
		local folder = Workspace:FindFirstChild("Entities") and Workspace.Entities:FindFirstChild("Infected")
		if folder then
			for _, ent in pairs(folder:GetChildren()) do
				if ent:IsA("Model") and ent:FindFirstChild("Head") then
					ent.Head.Transparency = 1
				end
			end
		end
	end
end)

-- FOV Control (Slider + Loop)
local currentFOV = 70
local fovLabel, fovSlider, fovFill = createSlider("FOV", 120, 70, 120, 70, function(val)
	currentFOV = val
end)
local fovLoop
if fovLoop then fovLoop:Disconnect() end
fovLoop = RunService.RenderStepped:Connect(function()
	if workspace.CurrentCamera and workspace.CurrentCamera.FieldOfView ~= currentFOV then
		workspace.CurrentCamera.FieldOfView = currentFOV
	end
end)

-- Silent Aim Transparency Control (Slider)
createSlider("SA Transparency", 180, 0, 100, 40, function(val)
	saTransparency = val / 100
end)

-- Full Bright (Slider + Loop)
local fullBrightBtn = createToggle("Full Bright", 240)
local fullBrightEnabled = false
local originalAmbient = Lighting.Ambient
local originalBrightness = Lighting.Brightness
local fullBrightLoop

fullBrightBtn.MouseButton1Click:Connect(function()
    fullBrightEnabled = not fullBrightEnabled
    fullBrightBtn.Text = "Full Bright: " .. (fullBrightEnabled and "ON" or "OFF")
    if fullBrightEnabled then
        if fullBrightLoop then fullBrightLoop:Disconnect() end
        fullBrightLoop = RunService.RenderStepped:Connect(function()
            Lighting.Ambient = Color3.new(1, 1, 1)
            Lighting.Brightness = 10
        end)
    else
        if fullBrightLoop then fullBrightLoop:Disconnect() end
        Lighting.Ambient = originalAmbient
        Lighting.Brightness = originalBrightness
    end
end)

-- No Fog (Slider + Loop)
local fogBtn = createToggle("No Fog", 280)
local fogEnabled = false
local originalFogEnd = Lighting.FogEnd
local fogLoop

fogBtn.MouseButton1Click:Connect(function()
    fogEnabled = not fogEnabled
    fogBtn.Text = "No Fog: " .. (fogEnabled and "ON" or "OFF")
    if fogEnabled then
        if fogLoop then fogLoop:Disconnect() end
        fogLoop = RunService.RenderStepped:Connect(function()
            Lighting.FogEnd = 1000000
        end)
    else
        if fogLoop then fogLoop:Disconnect() end
        Lighting.FogEnd = originalFogEnd
    end
end)

-- Mouse Lock/Unlock Keybind (')
local mouseFree = false
local mouseLoop

UserInputService.InputBegan:Connect(function(input, gpe)
    if input.KeyCode == Enum.KeyCode.Quote and not gpe then
        mouseFree = not mouseFree
        if mouseFree then
            if mouseLoop then mouseLoop:Disconnect() end
            mouseLoop = RunService.RenderStepped:Connect(function()
                UserInputService.MouseBehavior = Enum.MouseBehavior.Default
                UserInputService.MouseIconEnabled = true
            end)
        else
            if mouseLoop then mouseLoop:Disconnect() end
            UserInputService.MouseBehavior = Enum.MouseBehavior.LockCurrentPosition
            UserInputService.MouseIconEnabled = false
        end
    -- GUI hide/show keybind (])
    elseif input.KeyCode == Enum.KeyCode.RightBracket and not gpe then
        gui.Enabled = not gui.Enabled
    end
end)

gui.AncestryChanged:Connect(function()
    if not gui:IsDescendantOf(game) then
        if espCon then espCon:Disconnect() end
        if zombieAddedCon then zombieAddedCon:Disconnect() end
        if saLoop then saLoop:Disconnect() end
        if fovLoop then fovLoop:Disconnect() end
        if fullBrightLoop then fullBrightLoop:Disconnect() end
        if fogLoop then fogLoop:Disconnect() end
        if mouseLoop then mouseLoop:Disconnect() end
        UserInputService.MouseBehavior = Enum.MouseBehavior.Default
        UserInputService.MouseIconEnabled = true
    end
end)
