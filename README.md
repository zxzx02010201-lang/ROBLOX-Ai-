--[[
    Swarm-OS: Racket Rivals AI (V39 - Zero-Jitter Overlay)
    更新說明：
    1. 死區邏輯 (Deadzone)：當角色與球的距離小於 1.5 格時，停止位移注入，消除前後抖動。
    2. 線性插值平滑 (Lerp)：使用 CFrame:Lerp 讓移動更絲滑，減少瞬移帶來的物理碰撞。
    3. 萬倍強度穩定：在遠距離使用萬倍速，近距離自動切換為高精度鎖定。
    4. 全方位攻擊：維持 360 度狂暴擊球，只要球進範圍就打。
]]

local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
local RunService = game:GetService("RunService")
local VIM = game:GetService("VirtualInputManager")

local Window = Rayfield:CreateWindow({
   Name = "Swarm-OS | 靜態鎖定 AI",
   LoadingTitle = "正在校準死區防抖算法...",
   LoadingSubtitle = "by Leo66656gA3",
})

local _G = {
    AutoSwing = false,
    AutoChase = false,
    AutoLook = false,
    Reach = 40,        
    AI_Strength = 3000, 
    MySide = nil
}

local function FindTheBall()
    for _, v in pairs(game.Workspace:GetDescendants()) do
        if v:IsA("BasePart") or v:IsA("MeshPart") then
            if (v.Name:find("Ball") or v.Name:find("Shuttle")) and not v:IsDescendantOf(game.Players.LocalPlayer.Character) then
                return v
            end
        end
    end
    return nil
end

-- === 移動線程 (防抖重疊邏輯) ===
task.spawn(function()
    RunService.Heartbeat:Connect(function()
        if not _G.AutoChase then return end
        local char = game.Players.LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end

        if _G.MySide == nil then _G.MySide = (hrp.Position.Z > 0) and "Positive" or "Negative" end

        local ball = FindTheBall()
        if ball then
            local ballPos = ball.Position
            local targetZ = ballPos.Z
            
            -- 邊界保護
            if _G.MySide == "Positive" then targetZ = math.max(1.5, targetZ) else targetZ = math.min(-1.5, targetZ) end

            local targetPos = Vector3.new(ballPos.X, hrp.Position.Y, targetZ)
            local diff = (targetPos - hrp.Position)
            local dist = diff.Magnitude
            
            -- 【核心修正：死區防抖】
            -- 如果距離已經小於 1.2 格，就認為已經抵達，停止強制 CFrame 移動
            if dist > 1.2 then 
                -- 使用強度縮放
                local step = (_G.AI_Strength / 1000) * 1.5
                
                -- 使用 Lerp (線性插值) 取代直接加法，這會讓移動有「減速感」，極大減少抖動
                local alpha = math.min(step / dist, 1) -- 確保不會衝過頭
                hrp.CFrame = hrp.CFrame:Lerp(CFrame.new(targetPos, hrp.Position + hrp.CFrame.LookVector), alpha)
                
                -- 同步基礎移動動畫
                char.Humanoid:MoveTo(targetPos)
                char.Humanoid.WalkSpeed = 16 + (_G.AI_Strength * 0.05)
            else
                -- 在死區內，將速度歸零，防止慣性導致的前後移動
                hrp.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                hrp.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
            end
        end
    end)
end)

-- === 攻擊線程 (全方位狂暴) ===
task.spawn(function()
    RunService.Heartbeat:Connect(function()
        if not _G.AutoSwing then return end
        local char = game.Players.LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end

        local ball = FindTheBall()
        local vpSize = workspace.CurrentCamera.ViewportSize

        if ball then
            local dist = (hrp.Position - ball.Position).Magnitude
            
            -- 面向對面
            local pushZ = (_G.MySide == "Positive") and -85 or 85
            if _G.AutoLook then
                hrp.CFrame = CFrame.lookAt(hrp.Position, Vector3.new(0, hrp.Position.Y, pushZ))
            end

            -- 只要在 Reach 內就狂刷
            if dist <= _G.Reach then
                VIM:SendMouseButtonEvent(vpSize.X/2, vpSize.Y/2, 0, true, game, 0)
                VIM:SendMouseButtonEvent(vpSize.X/2, vpSize.Y/2, 0, false, game, 0)
            end
        end
    end)
end)

-- --- UI ---
local Tab = Window:CreateTab("防抖同步 V39", 4483362458)

Tab:CreateSlider({
   Name = "同步強度 (1-10000)",
   Range = {1, 10000},
   Increment = 10,
   CurrentValue = 3000,
   Callback = function(v) _G.AI_Strength = v end,
})

Tab:CreateToggle({Name = "防抖座標同步", CurrentValue = false, Callback = function(v) _G.AutoChase = v end})
Tab:CreateToggle({Name = "全方位狂暴攻擊", CurrentValue = false, Callback = function(v) _G.AutoSwing = v end})
Tab:CreateToggle({Name = "固定面向球場", CurrentValue = false, Callback = function(v) _G.AutoLook = v end})

Tab:CreateSlider({
   Name = "擊球半徑 (Reach)",
   Range = {10, 100},
   Increment = 1,
   CurrentValue = 40,
   Callback = function(v) _G.Reach = v end,
})

Tab:CreateButton({Name = "重置半場", Callback = function() _G.MySide = nil end})
