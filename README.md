--[[ 
Auto Perseguir, Revistar e Roubar com UI (Delta Roblox)
]]

-- Bypass sugerido anti-detections
local Namecall
Namecall = hookmetamethod(game, "__namecall", function(self, ...)
    if getnamecallmethod() == "FireServer" or getnamecallmethod() == "InvokeServer" then
        if tostring(self) == "nperdeseutempobro" or tostring(self) == "CheckAnti" then
            return nil
        end
    end
    return Namecall(self, ...)
end)

-- UI parent
local function get_ui_parent()
    return (gethui and gethui()) or game:GetService("CoreGui")
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AutoPersigaRevGUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = get_ui_parent()

local DragButton = Instance.new("TextButton")
DragButton.Size = UDim2.new(0,145,0,38)
DragButton.Position = UDim2.new(0.01,0,0.3,0)
DragButton.BackgroundColor3 = Color3.fromRGB(40,40,40)
DragButton.Text = "Ativar Script"
DragButton.TextColor3 = Color3.fromRGB(255,255,255)
DragButton.TextSize = 16
DragButton.Font = Enum.Font.GothamBold
DragButton.Parent = ScreenGui

-- Drag
do
    local dragging, dragStart, startPos
    DragButton.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = i.Position
            startPos = DragButton.Position
        end
    end)
    game:GetService("UserInputService").InputChanged:Connect(function(i)
        if dragging then
            local delta = i.Position - dragStart
            DragButton.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y
            )
        end
    end)
    DragButton.InputEnded:Connect(function()
        dragging = false
    end)
end

-- Services
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = Players.LocalPlayer

-- 🔒 SAFE CHECK (SafeGui no HumanoidRootPart)
local function is_safe_player(char)
    if not char then return true end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return true end
    return hrp:FindFirstChild("SafeGui") ~= nil
end

-- Itens
local itens = {
    "AK47","Uzi","Glock17","Parafal","Faca","IA2","G3",
    "Tratamento","Xbox","Hi Power","C4","AR-15","Escudo"
}

local args = { [1]="mudaInv",[2]="1",[4]="1" }

local function deletarNotifyGui()
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if not pg then return end
    for _,g in ipairs(pg:GetChildren()) do
        if g.Name == "NotifyGui" then g:Destroy() end
    end
end

local function try_roubar()
    deletarNotifyGui()
    for i,item in ipairs(itens) do
        if i <= 16 then
            args[2] = tostring(i)
            args[3] = item
            pcall(function()
                ReplicatedStorage.Modules.InvRemotes.InvRequest:InvokeServer(unpack(args))
            end)
            task.wait(0.02)
        end
    end
end

-- 🎯 HITBOX NA CABEÇA
local function aumentar_hitbox(player)
    if not player.Character then return end
    local head = player.Character:FindFirstChild("Head")
    if not head or head:FindFirstChild("HitboxAumentada") then return end

    head.Size = Vector3.new(6,6,6)
    head.CanCollide = false
    head.Massless = true

    local box = Instance.new("BoxHandleAdornment")
    box.Name = "HitboxAumentada"
    box.Adornee = head
    box.Size = Vector3.new(6,6,6)
    box.AlwaysOnTop = true
    box.Transparency = 0.6
    box.Color3 = Color3.fromRGB(255,0,80)
    box.Parent = head
end

-- Aim cabeça
local function auto_aim(target)
    task.spawn(function()
        while target and target.Character and target.Character:FindFirstChild("Head") do
            workspace.CurrentCamera.CFrame =
                CFrame.new(
                    workspace.CurrentCamera.CFrame.Position,
                    target.Character.Head.Position
                )
            task.wait(0.03)
        end
    end)
end

-- CONTROLE
local active = false
local perseguir_thread

local function perseguir()
    if perseguir_thread then return end

    perseguir_thread = task.spawn(function()
        while active do
            local target

            -- 🔍 escolhe player SEM SafeGui
            for _,p in ipairs(Players:GetPlayers()) do
                if p ~= LocalPlayer
                and p.Character
                and p.Character:FindFirstChild("Humanoid")
                and p.Character.Humanoid.Health > 0
                and p.Character:FindFirstChild("HumanoidRootPart")
                and not is_safe_player(p.Character) then
                    target = p
                    break
                end
            end

            if not target then
                task.wait(0.5)
                continue
            end

            aumentar_hitbox(target)
            auto_aim(target)

            local humanoid = target.Character.Humanoid
            local thrp = target.Character.HumanoidRootPart

            -- 🚶 PERSEGUIÇÃO (ATRÁS REAL)
            while active
            and humanoid.Health > 0
            and LocalPlayer.Character
            and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") do

                local hrp = LocalPlayer.Character.HumanoidRootPart

                -- 🔥 ATRÁS USANDO LOOKVECTOR
                local behindPos =
                    thrp.Position - (thrp.CFrame.LookVector * 2)

                hrp.CFrame = CFrame.new(behindPos, thrp.Position)

                task.wait(0.015)
            end

            -- ☠ MORREU
            if humanoid.Health <= 0
            and LocalPlayer.Character
            and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then

                local hrp = LocalPlayer.Character.HumanoidRootPart
                hrp.CFrame = thrp.CFrame * CFrame.new(0,2.5,0)

                task.wait(0.1)

                for i=1,2 do
                    pcall(function()
                        ReplicatedStorage.Remote:FireServer("/revistar")
                    end)
                    task.wait(0.06)
                end

                try_roubar()
            end

            task.wait(0.25)
        end
        perseguir_thread = nil
    end)
end

-- Botão
DragButton.MouseButton1Click:Connect(function()
    active = not active
    DragButton.Text = active and "Desativar Script" or "Ativar Script"
    if active then
        perseguir()
    end
end)

ScreenGui.DisplayOrder = math.random(200,400)
