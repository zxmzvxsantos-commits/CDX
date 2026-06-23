--[[
    THE CODEX V3.2 - UI FORÇADA
    - UI criada em 3 containers diferentes (CoreGui, PlayerGui, StarterGui)
    - Diagnóstico completo
]]

local VERSION = "3.2.0"

print("========== THE CODEX V3.2 ==========")
print("Carregando...")

-- ==================== SERVIÇOS ====================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local CoreGui = game:GetService("CoreGui")
local StarterGui = game:GetService("StarterGui")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

print("[1] Serviços carregados")

-- ==================== CONFIGURAÇÕES ====================
local Settings = {
    Ashura = {
        Enabled = false,
        Range = 100,
        AutoAim = true,
        SilentAim = true,
        AutoAttack = false,
        AttackSpeed = 0.5,
        HitboxExpand = true,
        HitboxSize = 15,
        ShowRange = true,
        FOV = 90,
        TeleportToTarget = true,
        AntiStuck = true,
    },
    CopySet = {
        Scanned = false,
        SetData = {
            Bloodline = nil,
            SubAbility = nil,
            Element = nil,
            Companions = {},
            Tools = {}
        }
    }
}

print("[2] Configurações carregadas")

-- ==================== LISTA DE BLOODLINES ====================
local ALL_BLOODLINES = {
    "Sound", "Ray-Kerada", "Rykan-Shizen", "Kerada", "Tetsuo-Kaijin", "Kokotsu",
    "Forged-Rengoku", "Kamaki-Amethyst", "Smoke", "Emerald", "Tengoku", "Shiver-Akuma",
    "Xeno-Dokei", "Strange", "Kamaki", "Aizden", "Zero-Glacier", "Ice",
    "Black-Lightning", "Bolt", "Koncho", "Ghost-Korashi", "Renshiki-Ruby", "Crystal",
    "Pika-Senko", "Ashen-Storm", "Head-Less", "Frost", "Narumaki", "Akuma",
    "Minakami", "Kamaki-Inferno", "Bankai-Inferno", "Boil", "Bubble", "Code-Gaiden",
    "Satori-Rengoku", "Hair", "Azim-Senko", "Bankai-Akuma", "Satori-Akuma", "Seishin",
    "Menza", "Shindai-Akuma", "Tsunami", "Shado", "RELL", "Mud",
    "Shiro-Glacier", "Scorch", "Cobra", "Aidens-Son-Mud", "Shizen", "Dokei",
    "Doom-Shado", "Xeno-Azure", "Kagoku", "Dangan", "Eternal", "Jinshiki",
    "Paper", "Azarashi", "Ragnar", "Magma", "Web", "Arahaki-Jokei",
    "Glacier", "Dust", "Borumaki", "Fume", "Indra-Akuma", "Jokei",
    "Mecha-Spirit", "Storm", "Shindai-Rengoku-Yang", "Apol-Sand", "Renshiki", "SnakeMan",
    "Surge", "Raion-Rengoku", "Explosion", "Okami", "Octo-Ink", "Nectar",
    "Saberu", "Wanziame", "Sand", "Getsuga-Black", "Raion-Gaiden", "Rengoku",
    "Kenichi", "Six-Paths-Narumaki", "Vanhelsing", "Blood", "Borumaki-Gaiden",
    "Dark-Jokei", "Riser-Akuma", "Jayramaki", "Giovanni-Shizen", "Vine", "Apollo-Sand",
    "Powder", "Senko", "Raiden-Saberu", "Alphirama-Shizen", "Dio-Senko", "Lava",
    "Ryuji-Kenichi", "Fizz", "Ashura-Shizen", "Eastwood-Korashi", "Raion-Akuma",
    "Minakaze", "Doku-Tengoku", "Bruce-Kenichi", "Odin-Saberu", "Inferno",
    "Gold-Sand", "Ink", "Clay", "Rune-Koncho", "Kaijin", "Deva-Rengoku",
    "Sarachia-Akuma"
}

local function isGameBloodline(name)
    if not name then return false end
    local lower = name:lower()
    for _, bl in ipairs(ALL_BLOODLINES) do
        if lower:find(bl:lower()) then
            return true
        end
    end
    return false
end

local SUB_KEYWORDS = {"mode", "awakening", "transform", "rage", "berserk", "demon", "spirit", "ghost", "flash", "teleport", "shield", "barrier", "heal", "regen"}
local COMPANION_KEYWORDS = {"companion", "summon", "pet", "familiar", "ally", "partner", "mount"}

print("[3] " .. #ALL_BLOODLINES .. " bloodlines carregadas")

-- ==================== SISTEMA ASHURA (RESUMIDO) ====================
local AshuraSystem = {}
AshuraSystem.__index = AshuraSystem

function AshuraSystem.new()
    local self = setmetatable({}, AshuraSystem)
    self.Target = nil
    self.LastAttack = 0
    self.Connections = {}
    self.RangeCircle = nil
    self.Attack2Ready = true
    self.CurrentTool = nil
    self.AttackRemote = nil
    return self
end

function AshuraSystem:FindAshuraAttack()
    local character = LocalPlayer.Character
    if not character then return nil, nil end
    local searchPlaces = {character, LocalPlayer:FindFirstChild("Backpack")}
    for _, container in ipairs(searchPlaces) do
        if not container then continue end
        for _, tool in ipairs(container:GetChildren()) do
            if tool:IsA("Tool") then
                local name = tool.Name:lower()
                if name:find("ashura") or name:find("asura") then
                    local remote = self:FindAttackRemote(tool)
                    if remote then
                        return tool, remote
                    end
                end
            end
        end
    end
    return nil, nil
end

function AshuraSystem:FindAttackRemote(tool)
    for _, child in ipairs(tool:GetDescendants()) do
        if child:IsA("RemoteEvent") or child:IsA("RemoteFunction") then
            local name = child.Name:lower()
            if name:find("attack") or name:find("ability") or name:find("m1") or name:find("click") then
                return child
            end
        end
    end
    for _, child in ipairs(ReplicatedStorage:GetDescendants()) do
        if child:IsA("RemoteEvent") and child.Name:lower():find("ashura") then
            return child
        end
    end
    for _, child in ipairs(tool:GetDescendants()) do
        if child:IsA("RemoteEvent") then
            return child
        end
    end
    return nil
end

function AshuraSystem:FireAttack(target)
    if not self.AttackRemote then
        local tool, remote = self:FindAshuraAttack()
        if tool and remote then
            self.CurrentTool = tool
            self.AttackRemote = remote
        else
            if self.CurrentTool then
                self.CurrentTool:Activate()
                return true
            end
            return false
        end
    end
    if not self.CurrentTool or not self.AttackRemote then return false end
    local character = LocalPlayer.Character
    if character and character:FindFirstChild(self.CurrentTool.Name) == nil then
        local backpack = LocalPlayer:FindFirstChild("Backpack")
        if backpack and backpack:FindFirstChild(self.CurrentTool.Name) then
            local humanoid = character:FindFirstChild("Humanoid")
            if humanoid then
                humanoid:EquipTool(backpack[self.CurrentTool.Name])
                task.wait(0.15)
                self.CurrentTool = character:FindFirstChild(self.CurrentTool.Name)
            end
        end
    end
    if self.CurrentTool and self.CurrentTool.Parent == character then
        local args = {}
        if target then
            local targetRoot = target:IsA("Player") and target.Character and target.Character:FindFirstChild("HumanoidRootPart")
            if targetRoot then
                table.insert(args, targetRoot.Position)
                table.insert(args, targetRoot.CFrame)
            end
        end
        pcall(function()
            self.AttackRemote:FireServer(unpack(args))
            if not args or #args == 0 then
                self.AttackRemote:FireServer()
            end
        end)
        return true
    end
    return false
end

function AshuraSystem:ExpandHitbox(targetRoot)
    if not Settings.Ashura.HitboxExpand then return end
    local char = LocalPlayer.Character
    if not char then return end
    local hitbox = Instance.new("Part")
    hitbox.Name = "AshuraExpandedHitbox"
    hitbox.Size = Vector3.new(Settings.Ashura.HitboxSize, Settings.Ashura.HitboxSize, Settings.Ashura.HitboxSize)
    hitbox.Anchored = true
    hitbox.CanCollide = false
    hitbox.Transparency = 0.3
    hitbox.Color = Color3.fromRGB(255, 50, 0)
    hitbox.Material = Enum.Material.Neon
    hitbox.Position = targetRoot.Position
    hitbox.Parent = workspace
    local damageRemote = nil
    for _, child in ipairs(ReplicatedStorage:GetDescendants()) do
        if child:IsA("RemoteEvent") and (child.Name:lower():find("damage") or child.Name:lower():find("hit") or child.Name:lower():find("combat")) then
            damageRemote = child
            break
        end
    end
    local connection
    connection = hitbox.Touched:Connect(function(hit)
        local parent = hit.Parent
        if parent and parent:IsA("Model") then
            local humanoid = parent:FindFirstChild("Humanoid")
            if humanoid and humanoid.Health > 0 and parent ~= LocalPlayer.Character then
                if damageRemote then
                    pcall(function()
                        damageRemote:FireServer(parent, 25)
                    end)
                else
                    humanoid:TakeDamage(25)
                end
            end
        end
    end)
    task.delay(0.5, function()
        if connection then connection:Disconnect() end
        hitbox:Destroy()
    end)
end

function AshuraSystem:GetNearestTarget()
    local nearest = nil
    local shortestDistance = Settings.Ashura.Range
    local myRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil end
    local players = Players:GetPlayers()
    for _, player in ipairs(players) do
        if player == LocalPlayer then continue end
        local char = player.Character
        if not char then continue end
        local root = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChild("Humanoid")
        if not root or not hum or hum.Health <= 0 then continue end
        local distance = (root.Position - myRoot.Position).Magnitude
        if distance > Settings.Ashura.Range then continue end
        if Settings.Ashura.AutoAim then
            local screenPos, onScreen = Camera:WorldToScreenPoint(root.Position)
            if not onScreen then continue end
            local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
            local screenDistance = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
            local maxScreenDistance = Camera.ViewportSize.X * (Settings.Ashura.FOV / 360)
            if screenDistance > maxScreenDistance then continue end
        end
        if distance < shortestDistance then
            shortestDistance = distance
            nearest = player
        end
    end
    if not nearest then
        for _, obj in ipairs(workspace:GetDescendants()) do
            if obj:IsA("Model") and obj:FindFirstChild("Humanoid") and obj:FindFirstChild("HumanoidRootPart") then
                local hum = obj.Humanoid
                local root = obj.HumanoidRootPart
                if hum.Health > 0 and not Players:GetPlayerFromCharacter(obj) then
                    local distance = (root.Position - myRoot.Position).Magnitude
                    if distance < shortestDistance then
                        shortestDistance = distance
                        nearest = obj
                    end
                end
            end
        end
    end
    return nearest
end

function AshuraSystem:ExecuteAttack(target)
    if not target or not self.Attack2Ready then return end
    local character = LocalPlayer.Character
    if not character then return end
    local targetRoot = nil
    if target:IsA("Player") and target.Character then
        targetRoot = target.Character:FindFirstChild("HumanoidRootPart")
    elseif target:IsA("Model") then
        targetRoot = target:FindFirstChild("HumanoidRootPart")
    end
    if not targetRoot then return end
    self.Attack2Ready = false
    self.LastAttack = tick()
    if Settings.Ashura.TeleportToTarget then
        local myRoot = character:FindFirstChild("HumanoidRootPart")
        if myRoot then
            local distance = (targetRoot.Position - myRoot.Position).Magnitude
            if distance > 15 then
                local dir = (targetRoot.Position - myRoot.Position).Unit
                local teleportPos = targetRoot.Position - dir * 8
                if Settings.Ashura.AntiStuck then
                    local ray = Ray.new(teleportPos + Vector3.new(0, 5, 0), Vector3.new(0, -10, 0))
                    local hit, pos = workspace:FindPartOnRay(ray, character)
                    if hit then
                        teleportPos = pos + Vector3.new(0, 3, 0)
                    end
                end
                myRoot.CFrame = CFrame.new(teleportPos)
                task.wait(0.1)
            end
        end
    end
    if not Settings.Ashura.SilentAim and Settings.Ashura.AutoAim then
        Camera.CFrame = CFrame.new(Camera.CFrame.Position, targetRoot.Position)
    end
    local success = self:FireAttack(target)
    if success then
        task.spawn(function()
            self:ExpandHitbox(targetRoot)
        end)
    end
    task.delay(Settings.Ashura.AttackSpeed, function()
        self.Attack2Ready = true
    end)
end

function AshuraSystem:CreateRangeCircle()
    if self.RangeCircle then
        self.RangeCircle:Destroy()
    end
    local circle = Instance.new("Part")
    circle.Name = "AshuraAttack2Range"
    circle.Shape = Enum.PartType.Cylinder
    circle.Size = Vector3.new(Settings.Ashura.Range * 2, 0.1, Settings.Ashura.Range * 2)
    circle.Anchored = true
    circle.CanCollide = false
    circle.Transparency = 0.5
    circle.Color = Color3.fromRGB(255, 100, 0)
    circle.Material = Enum.Material.Neon
    circle.Parent = workspace
    local att = Instance.new("Attachment", circle)
    local beam = Instance.new("Beam", circle)
    beam.Texture = "rbxassetid://10827797626"
    beam.Width0 = 0.5
    beam.Width1 = 0.5
    self.RangeCircle = circle
end

function AshuraSystem:Start()
    if self.Enabled then return end
    self.Enabled = true
    if Settings.Ashura.ShowRange then
        self:CreateRangeCircle()
    end
    local tool, remote = self:FindAshuraAttack()
    if tool then
        self.CurrentTool = tool
        self.AttackRemote = remote
    end
    self.Connections.MainLoop = RunService.Heartbeat:Connect(function()
        if not Settings.Ashura.Enabled then return end
        local char = LocalPlayer.Character
        if not char then return end
        if self.RangeCircle then
            local root = char:FindFirstChild("HumanoidRootPart")
            if root then
                self.RangeCircle.Position = root.Position
            end
        end
        if not self.Target or not self:IsTargetValid() then
            self.Target = self:GetNearestTarget()
        end
        if Settings.Ashura.AutoAttack and self.Target then
            if tick() - self.LastAttack >= Settings.Ashura.AttackSpeed then
                self:ExecuteAttack(self.Target)
            end
        end
    end)
end

function AshuraSystem:Stop()
    self.Enabled = false
    self.Target = nil
    for _, conn in pairs(self.Connections) do
        conn:Disconnect()
    end
    self.Connections = {}
    if self.RangeCircle then
        self.RangeCircle:Destroy()
        self.RangeCircle = nil
    end
end

function AshuraSystem:IsTargetValid()
    if not self.Target then return false end
    local root = self.Target:IsA("Player") and self.Target.Character and self.Target.Character:FindFirstChild("HumanoidRootPart")
    if not root then return false end
    local hum = root.Parent and root.Parent:FindFirstChild("Humanoid")
    if not hum or hum.Health <= 0 then return false end
    local myRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not myRoot then return false end
    return (root.Position - myRoot.Position).Magnitude <= Settings.Ashura.Range
end

print("[4] Sistema Ashura carregado")

-- ==================== SISTEMA COPY SET ====================
local CopySetSystem = {}

local function getPlayerByName(name)
    if not name or name == "" then return nil end
    local lower = name:lower()
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Name:lower() == lower or (p.DisplayName and p.DisplayName:lower() == lower) then
            return p
        end
    end
    return nil
end

function CopySetSystem.ScanCharacter(char)
    if not char then return nil end
    local data = {
        Bloodline = nil,
        SubAbility = nil,
        Element = nil,
        Companions = {},
        Tools = {}
    }
    local tools = {}
    for _, obj in ipairs(char:GetChildren()) do
        if obj:IsA("Tool") then
            table.insert(tools, obj)
        end
    end
    for _, tool in ipairs(tools) do
        local name = tool.Name
        local lower = name:lower()
        local classified = false
        if isGameBloodline(name) then
            data.Bloodline = name
            classified = true
        end
        if not classified then
            for _, kw in ipairs(SUB_KEYWORDS) do
                if lower:find(kw) then
                    data.SubAbility = name
                    classified = true
                    break
                end
            end
        end
        if not classified then
            for _, kw in ipairs(COMPANION_KEYWORDS) do
                if lower:find(kw) then
                    table.insert(data.Companions, name)
                    classified = true
                    break
                end
            end
        end
        if not classified then
            table.insert(data.Tools, name)
        end
    end
    for _, child in ipairs(char:GetDescendants()) do
        if child:IsA("StringValue") or child:IsA("NumberValue") or child:IsA("ObjectValue") then
            local val = child.Value
            if type(val) == "string" then
                local lower = val:lower()
                local elementKeywords = {"fire", "water", "wind", "earth", "lightning", "ice", "lava", "wood", "sand", "dark", "light"}
                for _, el in ipairs(elementKeywords) do
                    if lower:find(el) then
                        data.Element = val
                        break
                    end
                end
            end
        end
    end
    return data
end

function CopySetSystem.ScanTarget(target)
    local char = nil
    if type(target) == "string" then
        local player = getPlayerByName(target)
        if player then
            char = player.Character
        else
            return nil, "Jogador não encontrado"
        end
    elseif target:IsA("Player") then
        char = target.Character
    elseif target:IsA("Model") then
        char = target
    else
        return nil, "Alvo inválido"
    end
    if not char then
        return nil, "Personagem não encontrado"
    end
    local data = CopySetSystem.ScanCharacter(char)
    if data then
        Settings.CopySet.Scanned = true
        Settings.CopySet.SetData = data
        return data, "OK"
    else
        return nil, "Falha ao escanear"
    end
end

function CopySetSystem.ApplySet()
    if not Settings.CopySet.Scanned then
        return false, "Nenhum set escaneado"
    end
    local data = Settings.CopySet.SetData
    local character = LocalPlayer.Character
    local backpack = LocalPlayer:FindFirstChild("Backpack")
    if not character or not backpack then
        return false, "Personagem ou mochila não encontrados"
    end
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid then return false, "Humanoid não encontrado" end
    local function equipTool(toolName)
        local tool = backpack:FindFirstChild(toolName)
        if not tool then
            local starter = LocalPlayer:FindFirstChild("StarterGear")
            if starter then tool = starter:FindFirstChild(toolName) end
        end
        if tool and tool:IsA("Tool") then
            if tool.Parent == character then return true end
            humanoid:EquipTool(tool)
            task.wait(0.1)
            return true
        end
        return false
    end
    local results = {
        Bloodline = data.Bloodline and equipTool(data.Bloodline) or false,
        SubAbility = data.SubAbility and equipTool(data.SubAbility) or false,
        Companions = {},
        Tools = {}
    }
    for _, comp in ipairs(data.Companions) do
        table.insert(results.Companions, {name = comp, success = equipTool(comp)})
    end
    for _, t in ipairs(data.Tools) do
        table.insert(results.Tools, {name = t, success = equipTool(t)})
    end
    local msg = "Set aplicado:\n"
    msg = msg .. "Bloodline: " .. (data.Bloodline or "N/A") .. (results.Bloodline and " ✓" or " ✗") .. "\n"
    msg = msg .. "Sub: " .. (data.SubAbility or "N/A") .. (results.SubAbility and " ✓" or " ✗") .. "\n"
    if data.Element then msg = msg .. "Elemento: " .. data.Element .. " (não equipável)\n" end
    if #data.Companions > 0 then
        msg = msg .. "Companions: "
        for _, r in ipairs(results.Companions) do
            msg = msg .. r.name .. (r.success and "✓" or "✗") .. " "
        end
        msg = msg .. "\n"
    end
    if #data.Tools > 0 then
        msg = msg .. "Outros: "
        for _, r in ipairs(results.Tools) do
            msg = msg .. r.name .. (r.success and "✓" or "✗") .. " "
        end
        msg = msg .. "\n"
    end
    return true, msg
end

print("[5] Sistema Copy Set carregado")

-- ==================== UI FORÇADA (MÚLTIPLOS CONTAINERS) ====================
local function createUI()
    print("[6] Criando UI...")

    -- Tenta encontrar o melhor container
    local containers = {
        {container = CoreGui, name = "CoreGui"},
        {container = LocalPlayer:FindFirstChild("PlayerGui"), name = "PlayerGui"},
        {container = StarterGui, name = "StarterGui"}
    }

    local uiCreated = false
    local attempts = 0

    for _, c in ipairs(containers) do
        if c.container then
            attempts = attempts + 1
            print("[6." .. attempts .. "] Tentando criar UI no: " .. c.name)

            -- Remove UI antiga se existir
            local oldUI = c.container:FindFirstChild("THE_CODEX_UI")
            if oldUI then
                oldUI:Destroy()
                print("[6." .. attempts .. "] UI antiga removida de: " .. c.name)
            end

            -- Cria a nova UI
            local screenGui = Instance.new("ScreenGui")
            screenGui.Name = "THE_CODEX_UI"
            screenGui.ResetOnSpawn = false
            screenGui.Parent = c.container

            -- Verifica se foi criada
            if screenGui.Parent then
                print("[6." .. attempts .. "] ✅ UI criada com sucesso em: " .. c.name)
                uiCreated = true

                -- ===== CRIA O CONTEÚDO DA UI =====
                local mainFrame = Instance.new("Frame")
                mainFrame.Size = UDim2.new(0, 420, 0, 520)
                mainFrame.Position = UDim2.new(0.5, -210, 0.5, -260)
                mainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
                mainFrame.BackgroundTransparency = 0.15
                mainFrame.BorderSizePixel = 0
                mainFrame.ClipsDescendants = true
                mainFrame.Parent = screenGui

                local title = Instance.new("TextLabel", mainFrame)
                title.Size = UDim2.new(1, 0, 0, 40)
                title.Text = "THE CODEX V3.2"
                title.TextColor3 = Color3.fromRGB(255, 170, 0)
                title.BackgroundTransparency = 1
                title.Font = Enum.Font.GothamBold
                title.TextSize = 22
                title.TextScaled = true

                local tabContainer = Instance.new("Frame", mainFrame)
                tabContainer.Size = UDim2.new(1, 0, 1, -45)
                tabContainer.Position = UDim2.new(0, 0, 0.08, 0)
                tabContainer.BackgroundTransparency = 1

                local tabButtons = {}
                local tabContents = {}

                local function createTab(name, content)
                    local btn = Instance.new("TextButton", mainFrame)
                    btn.Size = UDim2.new(0.5, 0, 0, 35)
                    btn.Position = UDim2.new(#tabButtons * 0.5, 0, 0.08, 0)
                    btn.Text = name
                    btn.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
                    btn.TextColor3 = Color3.new(1, 1, 1)
                    btn.Font = Enum.Font.GothamBold
                    btn.TextSize = 16
                    btn.BorderSizePixel = 0
                    btn.Parent = mainFrame

                    local frame = Instance.new("Frame", tabContainer)
                    frame.Size = UDim2.new(1, -20, 1, -10)
                    frame.Position = UDim2.new(0, 10, 0, 5)
                    frame.BackgroundTransparency = 1
                    frame.Visible = false
                    content(frame)

                    table.insert(tabButtons, btn)
                    table.insert(tabContents, frame)

                    btn.MouseButton1Click:Connect(function()
                        for i, f in ipairs(tabContents) do
                            f.Visible = (i == #tabContents)
                        end
                        for i, b in ipairs(tabButtons) do
                            b.BackgroundColor3 = (i == #tabContents) and Color3.fromRGB(80, 80, 100) or Color3.fromRGB(45, 45, 55)
                        end
                    end)
                end

                -- Aba Ashura
                createTab("Ashura", function(parent)
                    local yPos = 5
                    local function addToggle(text, default, callback)
                        local btn = Instance.new("TextButton", parent)
                        btn.Size = UDim2.new(0.95, 0, 0, 30)
                        btn.Position = UDim2.new(0.025, 0, 0, yPos)
                        btn.Text = text .. (default and " [ON]" or " [OFF]")
                        btn.BackgroundColor3 = default and Color3.fromRGB(30, 80, 30) or Color3.fromRGB(80, 30, 30)
                        btn.TextColor3 = Color3.new(1, 1, 1)
                        btn.Font = Enum.Font.Gotham
                        btn.TextSize = 14
                        btn.BorderSizePixel = 0
                        btn.Parent = parent

                        local state = default
                        btn.MouseButton1Click:Connect(function()
                            state = not state
                            btn.Text = text .. (state and " [ON]" or " [OFF]")
                            btn.BackgroundColor3 = state and Color3.fromRGB(30, 80, 30) or Color3.fromRGB(80, 30, 30)
                            if callback then callback(state) end
                        end)
                        yPos = yPos + 32
                        return btn
                    end

                    local function addSlider(text, min, max, default, suffix, callback)
                        local label = Instance.new("TextLabel", parent)
                        label.Size = UDim2.new(0.95, 0, 0, 20)
                        label.Position = UDim2.new(0.025, 0, 0, yPos)
                        label.Text = text .. ": " .. default .. (suffix or "")
                        label.TextColor3 = Color3.new(0.8, 0.8, 0.8)
                        label.BackgroundTransparency = 1
                        label.Font = Enum.Font.Gotham
                        label.TextSize = 13
                        label.Parent = parent
                        yPos = yPos + 22

                        local slider = Instance.new("Frame", parent)
                        slider.Size = UDim2.new(0.9, 0, 0, 6)
                        slider.Position = UDim2.new(0.05, 0, 0, yPos)
                        slider.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
                        slider.Parent = parent
                        yPos = yPos + 10

                        local fill = Instance.new("Frame", slider)
                        fill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)
                        fill.BackgroundColor3 = Color3.fromRGB(255, 170, 0)
                        fill.Parent = slider

                        local dragging = false
                        local function update(pos)
                            local relative = (pos.X - slider.AbsolutePosition.X) / slider.AbsoluteSize.X
                            relative = math.clamp(relative, 0, 1)
                            local value = math.round((min + relative * (max - min)) * 10) / 10
                            fill.Size = UDim2.new(relative, 0, 1, 0)
                            label.Text = text .. ": " .. value .. (suffix or "")
                            if callback then callback(value) end
                        end

                        slider.InputBegan:Connect(function(input)
                            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                                dragging = true
                                update(input.Position)
                            end
                        end)
                        slider.InputEnded:Connect(function(input)
                            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                                dragging = false
                            end
                        end)
                        slider.InputChanged:Connect(function(input)
                            if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
                                update(input.Position)
                            end
                        end)
                        yPos = yPos + 6
                    end

                    addToggle("Enable Ashura", false, function(v)
                        Settings.Ashura.Enabled = v
                        if v then AshuraSystem:Start() else AshuraSystem:Stop() end
                    end)
                    addToggle("Teleport to Target", true, function(v) Settings.Ashura.TeleportToTarget = v end)
                    addToggle("Anti-Stuck", true, function(v) Settings.Ashura.AntiStuck = v end)
                    addToggle("Auto Aim", true, function(v) Settings.Ashura.AutoAim = v end)
                    addToggle("Silent Aim", true, function(v) Settings.Ashura.SilentAim = v end)
                    addToggle("Auto Attack", false, function(v) Settings.Ashura.AutoAttack = v end)
                    addSlider("Range", 10, 300, 100, " studs", function(v)
                        Settings.Ashura.Range = v
                        if AshuraSystem.RangeCircle then
                            AshuraSystem.RangeCircle.Size = Vector3.new(v * 2, 0.1, v * 2)
                        end
                    end)
                    addSlider("Attack Speed", 0.1, 3, 0.5, "s", function(v) Settings.Ashura.AttackSpeed = v end)
                    addToggle("Expand Hitbox", true, function(v) Settings.Ashura.HitboxExpand = v end)
                    addSlider("Hitbox Size", 5, 50, 15, " studs", function(v) Settings.Ashura.HitboxSize = v end)
                    addToggle("Show Range Circle", true, function(v)
                        Settings.Ashura.ShowRange = v
                        if v and Settings.Ashura.Enabled then
                            AshuraSystem:CreateRangeCircle()
                        elseif not v and AshuraSystem.RangeCircle then
                            AshuraSystem.RangeCircle:Destroy()
                            AshuraSystem.RangeCircle = nil
                        end
                    end)
                    addSlider("FOV", 30, 180, 90, "°", function(v) Settings.Ashura.FOV = v end)
                end)

                -- Aba Copy Set
                createTab("Copy Set", function(parent)
                    local yPos = 5

                    local nickBox = Instance.new("TextBox", parent)
                    nickBox.Size =
