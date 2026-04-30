local Library = loadstring(game:HttpGet("https://raw.githubusercontent.com/Twwzy/GitMenu/Home/WindUI/Library.lua"))()

local AIMBOT_ENABLED = false
local FOV_ENABLED = true
local FOV_RADIUS = 120
local WEAPON_REQUIRED = true
local CHECK_WALL = true
local AIMBOT_SMOOTH = 0.35
local IGNORE_TEAM = true
local AIM_PART = "Head"
local NO_RECOIL_ENABLED = false

_G.H = false
_G.S = 10

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

local isAiming = false
local hitboxConnection = nil
local aimbotConnection = nil
local recoilConnection = nil
local fovCircle = nil
local originalRecoil = {}

local VALID_AIM_PARTS = {"Head", "HumanoidRootPart", "Torso", "UpperTorso", "LowerTorso"}

local function applyNoRecoil()
    if not NO_RECOIL_ENABLED then return end
    local char = LocalPlayer.Character
    if not char then return end
    local tool = char:FindFirstChildOfClass("Tool")
    if not tool then return end
    for _, v in pairs(tool:GetChildren()) do
        if v:IsA("NumberValue") and (v.Name:lower():find("recoil") or v.Name:lower():find("kick") or v.Name:lower():find("spread")) then
            if not originalRecoil[tool.Name..v.Name] then originalRecoil[tool.Name..v.Name] = v.Value end
            v.Value = 0
        elseif v:IsA("Vector3Value") and (v.Name:lower():find("recoil") or v.Name:lower():find("kick")) then
            if not originalRecoil[tool.Name..v.Name] then originalRecoil[tool.Name..v.Name] = v.Value end
            v.Value = Vector3.new(0, 0, 0)
        end
    end
end

local function resetRecoil()
    local char = LocalPlayer.Character
    if not char then return end
    local tool = char:FindFirstChildOfClass("Tool")
    if not tool then return end
    for key, val in pairs(originalRecoil) do
        pcall(function()
            local name = key:gsub(tool.Name, "")
            local obj = tool:FindFirstChild(name)
            if obj then
                if obj:IsA("NumberValue") then obj.Value = val
                elseif obj:IsA("Vector3Value") then obj.Value = val end
            end
        end)
    end
    originalRecoil = {}
end

local function resetHitboxes()
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("Head") then
            local h = p.Character.Head
            h.Size = Vector3.new(2, 1, 1)
            h.Transparency = 1
            h.CanCollide = true
            h.Massless = false
        end
    end
end

local function getAimPart(char)
    if not char then return nil end
    return char:FindFirstChild(AIM_PART) or char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
end

local function isSameTeam(plr)
    if not IGNORE_TEAM then return false end
    local lt, tt = LocalPlayer.Team, plr.Team
    return lt and tt and lt == tt
end

local function setupFOVCircle()
    if not Drawing then return end
    local s, c = pcall(function() return Drawing.new("Circle") end)
    if s and c then
        fovCircle = c
        fovCircle.Visible = FOV_ENABLED
        fovCircle.Radius = FOV_RADIUS
        fovCircle.Thickness = 2
        fovCircle.Color = Color3.fromRGB(255, 100, 150)
        fovCircle.Filled = false
        fovCircle.NumSides = 64
        fovCircle.Transparency = 0.7
    end
end
setupFOVCircle()

local function hasWeapon()
    local char = LocalPlayer.Character
    return char and char:FindFirstChildOfClass("Tool") ~= nil
end

local function isVisible(part)
    if not part then return false end
    local origin = Camera.CFrame.Position
    local ray = Workspace:Raycast(origin, (part.Position - origin), RaycastParams.new())
    return not ray or ray.Instance:IsDescendantOf(part.Parent)
end

local function getBestTarget()
    if not FOV_ENABLED then return nil end
    local best, bestAngle = nil, FOV_RADIUS
    local center = Camera.ViewportSize / 2
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and not isSameTeam(plr) and plr.Character then
            local aimPart = getAimPart(plr.Character)
            if aimPart then
                local vec, onScreen = Camera:WorldToViewportPoint(aimPart.Position)
                if onScreen then
                    local angle = (Vector2.new(vec.X, vec.Y) - center).Magnitude
                    if angle < bestAngle and (not CHECK_WALL or isVisible(aimPart)) then
                        best, bestAngle = aimPart, angle
                    end
                end
            end
        end
    end
    return best
end

local function doAimbot(target)
    if not target then return end
    if AIMBOT_SMOOTH > 0 then
        Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(Camera.CFrame.Position, target.Position), AIMBOT_SMOOTH)
    else
        Camera.CFrame = CFrame.new(Camera.CFrame.Position, target.Position)
    end
end

local function startAimbotLoop()
    if aimbotConnection then aimbotConnection:Disconnect() end
    aimbotConnection = RunService.RenderStepped:Connect(function()
        if not AIMBOT_ENABLED then return end
        if WEAPON_REQUIRED and not (hasWeapon() or isAiming) then return end
        local target = getBestTarget()
        if target then doAimbot(target) end
    end)
end

local function startNoRecoilLoop()
    if recoilConnection then recoilConnection:Disconnect() end
    recoilConnection = RunService.RenderStepped:Connect(function()
        if NO_RECOIL_ENABLED then applyNoRecoil() end
    end)
end

local function toggleHitbox()
    _G.H = not _G.H
    if _G.H then
        if not hitboxConnection then
            hitboxConnection = RunService.RenderStepped:Connect(function()
                if not _G.H then return end
                for _, p in pairs(Players:GetPlayers()) do
                    if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("Head") then
                        local h = p.Character.Head
                        h.Size = Vector3.new(_G.S, _G.S, _G.S)
                        h.Transparency = 0.8
                        h.CanCollide = false
                        h.Massless = true
                        h.BrickColor = BrickColor.new("Bright green")
                    end
                end
            end)
        end
    else
        if hitboxConnection then
            hitboxConnection:Disconnect()
            hitboxConnection = nil
        end
        resetHitboxes()
    end
end

local function toggleNoRecoil()
    NO_RECOIL_ENABLED = not NO_RECOIL_ENABLED
    if NO_RECOIL_ENABLED then
        startNoRecoilLoop()
    else
        if recoilConnection then
            recoilConnection:Disconnect()
            recoilConnection = nil
        end
        resetRecoil()
    end
end

UserInputService.InputBegan:Connect(function(inp, gp)
    if gp then return end
    if inp.UserInputType == Enum.UserInputType.MouseButton2 then isAiming = true end
end)

UserInputService.InputEnded:Connect(function(inp, gp)
    if gp then return end
    if inp.UserInputType == Enum.UserInputType.MouseButton2 then isAiming = false end
end)

RunService.RenderStepped:Connect(function()
    if fovCircle then
        fovCircle.Visible = FOV_ENABLED
        if FOV_ENABLED and Camera then
            fovCircle.Position = Camera.ViewportSize / 2
            fovCircle.Radius = FOV_RADIUS
        end
    end
end)

startAimbotLoop()

local Window = Library:CreateWindow({
    Title = "My Super Hub",
    Icon = "boxes",
    Author = "by Tycoon",
    Folder = "MySuperHub_Aimbot",
    Size = UDim2.fromOffset(580, 520),
    ToggleKey = Enum.KeyCode.LeftShift,
    Theme = "Dark",
    Resizable = true,
})

Window:Tag({
    Title = "v2.0",
    Icon = "sparkles",
    Color = Color3.fromRGB(139, 92, 246),
})

local AimbotTab = Window:Tab({ Title = "Aimbot", Icon = "target" })
local HitboxTab = Window:Tab({ Title = "Hitbox", Icon = "shield" })
local MiscTab = Window:Tab({ Title = "Misc", Icon = "settings" })

AimbotTab:Toggle({ Title = "Aimbot", Value = AIMBOT_ENABLED, Callback = function(v) AIMBOT_ENABLED = v end })
AimbotTab:Toggle({ Title = "Anti-Parede", Value = CHECK_WALL, Callback = function(v) CHECK_WALL = v end })
AimbotTab:Toggle({ Title = "Exigir Arma/Mira", Value = WEAPON_REQUIRED, Callback = function(v) WEAPON_REQUIRED = v end })
AimbotTab:Toggle({ Title = "Ignorar Time", Value = IGNORE_TEAM, Callback = function(v) IGNORE_TEAM = v end })
AimbotTab:Divider()
AimbotTab:Dropdown({ Title = "Parte do Corpo", Values = VALID_AIM_PARTS, Value = AIM_PART, Callback = function(v) AIM_PART = v end })
AimbotTab:Divider()
AimbotTab:Slider({ Title = "Raio do FOV", Min = 40, Max = 400, Value = FOV_RADIUS, Callback = function(v) FOV_RADIUS = v end })
AimbotTab:Slider({ Title = "Suavização", Min = 0, Max = 1, Value = AIMBOT_SMOOTH, Precision = 2, Callback = function(v) AIMBOT_SMOOTH = v end })
AimbotTab:Divider()
AimbotTab:Toggle({ Title = "Mostrar Círculo FOV", Value = FOV_ENABLED, Callback = function(v) FOV_ENABLED = v end })

HitboxTab:Toggle({ Title = "Hitbox", Value = _G.H, Callback = function() toggleHitbox() end })
HitboxTab:Divider()
HitboxTab:Slider({ Title = "Tamanho da Hitbox", Min = 5, Max = 20, Value = _G.S, Callback = function(v) _G.S = v end })

MiscTab:Toggle({ Title = "No Recoil", Value = NO_RECOIL_ENABLED, Callback = function(v) toggleNoRecoil() end })
MiscTab:Divider()
MiscTab:Label("Remove o recuo das armas ao atirar")
