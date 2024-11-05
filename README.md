getgenv().Rasma = {
    ["Preload"] = {
        Notifications = true,
    },
    ["SilentAim"] = {
        Enabled = true,
        UseKeybind = true,
        Keybind = "T",
        HitPart = "UpperTorso",
        Prediction = 0.1263,
    },
    ["FOV"] = {
        ["SilentAim"] = {
            Visible = true,
            Size = 34,
            Color = Color3.fromRGB(0, 0, 0),
            Filled = false,
            Transparency = 0.5,
            Position = "Center", -- "mouse" for pc or "center" for mobile 
        },
    },
    ["Safety"] = {
        AntiGroundShots = true,
        AntiAimViewer = true,
    },
    ["Checks"] = {
        ["SilentAim"] = {
            WallCheck = true,
            KnockedCheck = false,
            AliveCheck = false,
        },
    },
    ["AutoSettings"] = {
        LowGFX = false,
        MuteBoomboxes = true,
    },
}
-- // Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local SoundService = game:GetService("SoundService")
local Workspace = game:GetService("Workspace")
--
repeat task.wait() until game:IsLoaded() and (Players.LocalPlayer.Character or Players.LocalPlayer.CharacterAdded:Wait())
--
-- // Variables
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local Viewport = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
local SilentTarget = nil
local Connection = {nil, nil}
local AimPoint = nil
local CurrentGame, DHGames, RemoteEvent
--
local DHGames = {
	[1008451066] = {
		Name = "Da Hood",
		MouseArguments = "UpdateMousePos",
		Functions = {
			GetRemote = function()
				return game:GetService("ReplicatedStorage").MainEvent
			end,
			CheckKnocked = function(Player)
				if (Player) and Player.Character:FindFirstChild("BodyEffects") then
					return Player.Character.BodyEffects["K.O"].Value
				end
                --
				return false
			end
		},
	},
	[1958807588] = {
		Name = "Hood Modded",
		MouseArguments = "MousePos",
		Functions = {
			GetRemote = function()
				return game:GetService("ReplicatedStorage").Bullets
			end
		},
	},
	[3895585994] = {
		Name = "Hood Trainer",
		MouseArguments = "UpdateMousePos",
		Functions = {
			GetRemote = function()
				return game:GetService("ReplicatedStorage").MainRemote
			end
		},
	},
    ["Universal"] = {
        Name = "Universal",
        MouseArguments = "MOUSE",
        Functions = {
            GetRemote = function()
                return game:GetService("ReplicatedStorage").MAINEVENT
            end
        },
    }
}
--
do -- Preload
    if DHGames[game.GameId] then
        CurrentGame = DHGames[game.GameId]
    else
        CurrentGame = DHGames["Universal"]
    end
    --
    RemoteEvent = CurrentGame.Functions.GetRemote()
    --
    if Rasma.AutoSettings.LowGFX then
        for Index, Value in pairs(workspace:GetDescendants()) do
            if Value.ClassName == "Part" or Value.ClassName == "SpawnLocation" or Value.ClassName == "WedgePart" or Value.ClassName == "Terrain" or Value.ClassName == "MeshPart" then
                Value.Material = "Plastic"
            end
        end
    end
    --
    if Rasma.AutoSettings.MuteBoomboxes then
        for Index, Value in ipairs(workspace:GetDescendants()) do
            if Value:IsA("Sound") then
                Value:Stop()
                Value:Destroy()
            end
        end
    end
end
--
local SilentAimFOVCircle = Drawing.new("Circle")
SilentAimFOVCircle.Visible = true
SilentAimFOVCircle.Color = Color3.fromRGB(255, 255, 255)
SilentAimFOVCircle.Filled = false
--
do -- Functions
    function UpdateFOV()
        if not SilentAimFOVCircle then return end
        --
        SilentAimFOVCircle.Visible = Rasma.FOV.SilentAim.Visible
        SilentAimFOVCircle.Color = Rasma.FOV.SilentAim.Color
        SilentAimFOVCircle.Radius = Rasma.FOV.SilentAim.Size * 3
        SilentAimFOVCircle.Filled = Rasma.FOV.SilentAim.Filled
        SilentAimFOVCircle.Transparency = Rasma.FOV.SilentAim.Transparency
        --
        if Rasma.FOV.SilentAim.Position == "Center" then
            SilentAimFOVCircle.Position = Viewport
        else
            SilentAimFOVCircle.Position = UserInputService:GetMouseLocation()
        end
    end
    --
    function RayCast(Part, Origin, Ignore, Distance)
        local Ignore = Ignore or {}
        local Distance = Distance or 2000
        --
        local Cast = Ray.new(Origin, (Part.Position - Origin).Unit * Distance)
        local Hit = Workspace:FindPartOnRayWithIgnoreList(Cast, Ignore)
        if Hit and Hit:IsDescendantOf(Part.Parent) then
            return true, Hit
        else
            return false, Hit
        end
        return false, nil
    end
    --
    function GetTarget()
        local Target, Closest = nil, math.huge
        local MousePosition = UserInputService:GetMouseLocation()
        --
        for _, Player in ipairs(Players:GetPlayers()) do
            if Player == LocalPlayer then continue end
            --
            local Character = Player and Player.Character
            local Humanoid = Character and Player.Character:FindFirstChild("Humanoid")
            local RootPart = Character and Player.Character:FindFirstChild("HumanoidRootPart")
            --
            if not (Character and Humanoid and RootPart) then continue end
            --
            -- // Checks
            if not Rasma.SilentAim.Enabled then continue end
            if Rasma.Checks.SilentAim.KnockedCheck and CurrentGame == 1008451066 and Player.Character:FindFirstChild("BodyEffects") and Player.Character.BodyEffects["K.O"].Value or Player.Character:FindFirstChild("GRABBING_CONSTRAINT") ~= nil then continue end
            if Rasma.Checks.SilentAim.WallCheck and not RayCast(RootPart, Camera.CFrame.Position, {LocalPlayer.Character}) then continue end
            if Rasma.Checks.SilentAim.AliveCheck and not (Humanoid.Health > 0) then continue end
            --
            local Position, OnScreen = Camera:WorldToScreenPoint(RootPart.Position)
            local Distance
            --
            if Rasma.FOV.SilentAim.Position == "Center" then
                Distance = (Vector2.new(Position.X, Position.Y) - Viewport).Magnitude
            else
                Distance = (Vector2.new(Position.X, Position.Y) - Vector2.new(MousePosition.X, MousePosition.Y)).Magnitude
            end
            --
            if not (Distance <= SilentAimFOVCircle.Radius) then continue end
            --
            if (Distance < Closest) and OnScreen then
                Target = Player
                Closest = Distance
            end
        end
        SilentTarget = Target
    end
    --
    function CalculateAimPoint()
        if SilentTarget and SilentTarget.Character then
            local hitPart = SilentTarget.Character:FindFirstChild(Rasma.SilentAim.HitPart)
            if hitPart then
               
                local velocity = hitPart.Velocity
                local distance = (hitPart.Position - Camera.CFrame.Position).Magnitude


                local predictionTime = distance / 300 -- assuming projectile speed is 300, adjust as needed
                local predictedPosition = hitPart.Position + (velocity * predictionTime)

                -- (anti-ground shots fix)
                if Rasma.Safety.AntiGroundShots then
                    predictedPosition = Vector3.new(predictedPosition.X, math.max(predictedPosition.Y, 3), predictedPosition.Z)
                end

                AimPoint = predictedPosition
            end
        end
    end
end
--
do -- Main RunService
    RunService.PreRender:Connect(function()
        GetTarget()
        UpdateFOV()
        CalculateAimPoint()
    end)
end
--
do -- UIS
    UserInputService.InputBegan:Connect(function(Input, gpe)
        if gpe then return end
        --
        if Rasma.SilentAim.UseKeybind and Input.KeyCode == Enum.KeyCode[Rasma.SilentAim.Keybind] then
            Rasma.SilentAim.Enabled = not Rasma.SilentAim.Enabled
        end
    end)
end
--
do -- Main Silent Aim
    LocalPlayer.CharacterAdded:Connect(function(Character)
        Character.ChildAdded:Connect(function(child)
            if child:IsA("Tool") then
                if Connection[1] == nil then
                    Connection[1] = child 
                end
                --
                if Connection[1] ~= child and Connection[2] ~= nil then 
                    Connection[2]:Disconnect()
                    Connection[1] = child
                end
                --
                Connection[2] = child.Activated:Connect(function()
                    if SilentTarget and AimPoint and Rasma.Safety.AntiAimViewer then
                        RemoteEvent:FireServer(CurrentGame.MouseArguments, AimPoint)
                    end
                end)
            end
        end)
    end)
    --
    LocalPlayer.Character.ChildAdded:Connect(function(child)
        if child:IsA("Tool") then
            if Connection[1] == nil then
                Connection[1] = child 
            end
            --
            if Connection[1] ~= child and Connection[2] ~= nil then 
                Connection[2]:Disconnect()
                Connection[1] = child
            end
            --
            Connection[2] = child.Activated:Connect(function()
                if SilentTarget and AimPoint and Rasma.Safety.AntiAimViewer then
                    RemoteEvent:FireServer(CurrentGame.MouseArguments, AimPoint)
                end
            end)
        end
    end)
    --
    local oldNamecall
    --
    oldNamecall = hookmetamethod(game, "__namecall", function(...)
        local args = {...};
        local method = getnamecallmethod();
        if method == "FireServer" and args[2] == CurrentGame.MouseArguments then
            if SilentTarget and AimPoint and not Rasma.Safety.AntiAimViewer then
                args[3] = AimPoint
            end
            --
            return oldNamecall(unpack(args))
        end
        --
        return oldNamecall(...)
    end)
end
