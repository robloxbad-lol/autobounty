-- [[ Blox Fruits: PvP Destroyer (Custom Combat Logic) ]]

if not game:IsLoaded() then game.Loaded:Wait() end

local Players = game:GetService("Players")
repeat task.wait(0.5) until Players.LocalPlayer and Players.LocalPlayer:FindFirstChild("PlayerGui")
local Player = Players.LocalPlayer

local function WaitForChildPath(parent, path)
    local current = parent
    for _, name in pairs(path) do
        current = current:WaitForChild(name, 30)
        if not current then return nil end
    end
    return current
end

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local CommF = WaitForChildPath(ReplicatedStorage, {"Remotes", "CommF_"})
local Net = WaitForChildPath(ReplicatedStorage, {"Modules", "Net"})
local RE_Attack = Net and Net:WaitForChild("RE/RegisterAttack", 10)
local RE_Hit = Net and Net:WaitForChild("RE/RegisterHit", 10)

if not (CommF and RE_Attack and RE_Hit) then return end

-- 設定
_G.PvPHunterRunning = true
_G.AutoBuso = true 
_G.SkipTarget = false
_G.BlacklistedPlayers = {}
if not _G.ServerHopBlacklist then _G.ServerHopBlacklist = {} end

local VirtualInputManager = game:GetService("VirtualInputManager")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local CoreGui = game:GetService("CoreGui")
local currentJobId = game.JobId

-- サーバーキック時の自動ホップ
CoreGui.RobloxPromptGui.promptOverlay.ChildAdded:Connect(function(child)
    if child.Name == "ErrorPrompt" then
        local success, result = pcall(function()
            return HttpService:JSONDecode(game:HttpGet('https://games.roblox.com/v1/games/' .. game.PlaceId .. '/servers/Public?sortOrder=Asc&limit=100'))
        end)
        if success and result.data then
            TeleportService:TeleportToPlaceInstance(game.PlaceId, result.data[math.random(1, #result.data)].id, Player)
        end
    end
end)

-- 戦闘状態の超厳格チェック
local function isSafeToHop()
    local mainGui = Player.PlayerGui:FindFirstChild("Main")
    local combatLabel = mainGui and mainGui:FindFirstChild("BottomHUDList") and mainGui.BottomHUDList:FindFirstChild("InCombat")
    if combatLabel and combatLabel.Visible then
        local text = combatLabel.Text
        if string.find(text, "Bounty at risk") or string.find(text, "risk") then
            return false
        end
    end
    return true
end

-- 強制サーバーホップ
local function serverHop()
    _G.ServerHopBlacklist[currentJobId] = tick() + 600
    while not isSafeToHop() do task.wait(1) end
    local success, result = pcall(function()
        return HttpService:JSONDecode(game:HttpGet('https://games.roblox.com/v1/games/' .. game.PlaceId .. '/servers/Public?sortOrder=Asc&limit=100'))
    end)
    if success and result and result.data then
        for _, server in pairs(result.data) do
            if server.id ~= currentJobId and server.playing < server.maxPlayers and not _G.ServerHopBlacklist[server.id] then
                TeleportService:TeleportToPlaceInstance(game.PlaceId, server.id, Player)
                task.wait(1)
            end
        end
    end
    task.wait(2)
    TeleportService:Teleport(game.PlaceId)
end

-- パイレーツ選択後にHP0監視開始
task.spawn(function()
    repeat task.wait(1) until Player.Team ~= nil and Player.Team.Name == "Pirates"
    local deadTimer = 0
    while task.wait(0.5) do
        if Player.Character and Player.Character:FindFirstChild("Humanoid") then
            if Player.Character.Humanoid.Health <= 0 then
                deadTimer = deadTimer + 0.5
            else
                deadTimer = 0
            end
            if deadTimer >= 3 then serverHop() break end
        end
    end
end)

-- スキップボタン
if CoreGui:FindFirstChild("SkipGui") then CoreGui.SkipGui:Destroy() end
local sg = Instance.new("ScreenGui", CoreGui)
sg.Name = "SkipGui"
local skipBtn = Instance.new("TextButton", sg)
skipBtn.Size = UDim2.new(0, 150, 0, 50)
skipBtn.Position = UDim2.new(0.5, -75, 0, 70)
skipBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
skipBtn.Text = "Skip Target"
skipBtn.TextColor3 = Color3.new(1, 1, 1)
skipBtn.TextSize = 20
skipBtn.Font = Enum.Font.SourceSansBold
Instance.new("UICorner", skipBtn)
skipBtn.MouseButton1Click:Connect(function() _G.SkipTarget = true end)

-- 武装色の覇気 & チーム選択
task.spawn(function()
    while task.wait(1.5) do
        if _G.AutoBuso and Player.Character and not Player.Character:FindFirstChild("HasBuso") then
            pcall(function() CommF:InvokeServer("Buso") end)
        end
    end
end)
task.spawn(function()
    task.wait(3)
    repeat 
        if Player.Team == nil or Player.Team.Name ~= "Pirates" then
            pcall(function() CommF:InvokeServer("SetTeam", "Pirates") end)
        end
        task.wait(1)
    until Player.Team ~= nil
end)

-- セーフゾーン判定
local function isInsideSafeZone(char)
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return true end
    local folder = workspace:FindFirstChild("_WorldOrigin") and workspace._WorldOrigin:FindFirstChild("SafeZones")
    if not folder then return false end
    for _, obj in pairs(folder:GetDescendants()) do
        if obj:IsA("BasePart") then
            local sm = obj:FindFirstChildOfClass("SpecialMesh")
            local scale = sm and sm.Scale or Vector3.new(1, 1, 1)
            local finalSize = obj.Size * scale
            local dist = hrp.Position - obj.Position
            if math.abs(dist.X) < finalSize.X/2 and math.abs(dist.Z) < finalSize.Z/2 and math.abs(dist.Y) < finalSize.Y/2 + 10 then
                return true
            end
        end
    end
    return false
end

-- 【今回の修正箇所】セーフゾーン内で攻撃するかどうかの判定
local function shouldTargetInSafeZone(targetPlayer)
    local charInWorkspace = workspace:FindFirstChild("Characters") and workspace.Characters:FindFirstChild(targetPlayer.Name)
    if charInWorkspace and charInWorkspace:FindFirstChild("InCombat") then
        local combatVal = charInWorkspace.InCombat.Value
        -- 0という数字が入っている場合、またはチェックマーク(true)がある場合は狙う
        if type(combatVal) == "number" or combatVal == true then
            return true
        end
    end
    -- チェックマークがない(false)状態は狙わない
    return false
end

-- 攻撃エンジン
local function ExecuteFA(targetChar, targetRoot)
    RE_Attack:FireServer(0.01)
    RE_Hit:FireServer(targetRoot, {{targetChar, targetRoot}, {targetChar, targetRoot}})
    local tool = Player.Character and Player.Character:FindFirstChildOfClass("Tool")
    if tool and tool:FindFirstChild("LeftClickRemote") then
        tool.LeftClickRemote:FireServer((targetRoot.Position - Player.Character:GetPivot().Position).Unit, 1)
    end
end

-- メインループ
local firstAttackFound = false
task.spawn(function()
    while _G.PvPHunterRunning do
        if Player.Team == nil then task.wait(1) continue end
        local currentTargets = {}
        for _, p in pairs(Players:GetPlayers()) do
            if p ~= Player and not _G.BlacklistedPlayers[p.UserId] then
                local char = p.Character
                local targetHrp = char and char:FindFirstChild("HumanoidRootPart")
                local pvp = p:GetAttribute("PvpDisabled") or (p:FindFirstChild("PvpDisabled", true) and p:FindFirstChild("PvpDisabled", true).Value)
                
                if pvp ~= true and char and char:FindFirstChild("Humanoid") and char.Humanoid.Health > 0 and targetHrp then
                    local inSafe = isInsideSafeZone(char)
                    
                    -- セーフゾーン外ならOK。
                    -- セーフゾーン内なら「数字(0含む)またはチェックあり」の時だけターゲットにする。
                    if not inSafe or (inSafe and shouldTargetInSafeZone(p)) then
                        table.insert(currentTargets, p)
                    end
                end
            end
        end

        if #currentTargets == 0 then
            serverHop()
            task.wait(5)
        else
            for _, targetPlayer in pairs(currentTargets) do
                _G.SkipTarget = false
                while _G.PvPHunterRunning and targetPlayer.Parent and targetPlayer.Character and not _G.SkipTarget do
                    local char = targetPlayer.Character
                    local hum = char:FindFirstChild("Humanoid")
                    local root = char:FindFirstChild("HumanoidRootPart")
                    local myRoot = Player.Character and Player.Character:FindFirstChild("HumanoidRootPart")

                    if not hum or hum.Health <= 0 then break end
                    
                    -- 追撃中：セーフゾーンに入り、かつ「チェックマークがなくなり数字でもなくなった」ら即中止
                    if isInsideSafeZone(char) and not shouldTargetInSafeZone(targetPlayer) then
                        break
                    end

                    if myRoot and root then
                        if not firstAttackFound then
                            task.wait(0.5)
                            VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.Two, false, game)
                            task.wait(0.1)
                            VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.Two, false, game)
                            firstAttackFound = true
                        end
                        myRoot.CFrame = root.CFrame * CFrame.new(0, 5, 0)
                        pcall(function() ExecuteFA(char, root) end)
                    end
                    task.wait(0.01)
                end
                if _G.SkipTarget then _G.BlacklistedPlayers[targetPlayer.UserId] = true end
                task.wait(0.2)
            end
        end
        task.wait(1)
    end
end)
