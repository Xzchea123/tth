-- Solara GUI + ESP for "Those Who Remain"
local DISTANCE_TO_PICKUP = 10

-- Create GUI
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ESPGui"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = game:GetService("CoreGui")

local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Size = UDim2.new(0, 120, 0, 30)
ToggleBtn.Position = UDim2.new(0, 10, 0, 10)
ToggleBtn.Text = "Toggle ESP: OFF"
ToggleBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
ToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleBtn.Parent = ScreenGui

-- Setup references
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Workspace = game:GetService("Workspace")
local camera = Workspace.CurrentCamera
local itemsHolder = Workspace:WaitForChild("Ignore"):WaitForChild("Items")

-- ESP settings
local espEnabled = false
local espHighlights = {}

local highlightTargets = {
    Ammo = { Color = Color3.fromRGB(255, 255, 255) },
    Medkit = { Color = Color3.fromRGB(255, 0, 0) },
    ["Body Armor"] = { Color = Color3.fromRGB(0, 255, 255) },
}

-- Create highlight
local function createHighlight(itemName, color)
    local highlight = Instance.new("Highlight")
    highlight.Name = itemName .. "_ESP"
    highlight.FillColor = color
    highlight.OutlineColor = color
    highlight.FillTransparency = 0.2
    highlight.OutlineTransparency = 0.5
    return highlight
end

-- Toggle ESP
local function toggleESP()
    espEnabled = not espEnabled
    ToggleBtn.Text = espEnabled and "Toggle ESP: ON" or "Toggle ESP: OFF"

    for _, child in ipairs(itemsHolder:GetChildren()) do
        if highlightTargets[child.Name] then
            if espEnabled and not child:FindFirstChildOfClass("Highlight") then
                local hl = createHighlight(child.Name, highlightTargets[child.Name].Color)
                hl.Adornee = child
                hl.Parent = child
            elseif not espEnabled then
                local existing = child:FindFirstChildOfClass("Highlight")
                if existing then
                    existing:Destroy()
                end
            end
        end
    end
end

ToggleBtn.MouseButton1Click:Connect(toggleESP)

-- New item monitoring
itemsHolder.ChildAdded:Connect(function(item)
    task.wait(0.2)
    if espEnabled and highlightTargets[item.Name] then
        local hl = createHighlight(item.Name, highlightTargets[item.Name].Color)
        hl.Adornee = item
        hl.Parent = item
    end
end)
