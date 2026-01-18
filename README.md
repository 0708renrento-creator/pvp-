# pvp-
--// Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ProximityPromptService = game:GetService("ProximityPromptService")
local LP = Players.LocalPlayer

repeat task.wait() until LP:FindFirstChild("PlayerGui")

-- ===== GUI =====
local gui = Instance.new("ScreenGui", LP.PlayerGui)
gui.Name = "HirekatuHad"

local main = Instance.new("Frame", gui)
main.Size = UDim2.fromOffset(180,130)
main.Position = UDim2.fromScale(0.5,0.5)-UDim2.fromOffset(90,65)
main.BackgroundColor3 = Color3.fromRGB(25,25,25)
main.BorderSizePixel = 0
main.Active = true
main.Draggable = true
Instance.new("UICorner", main).CornerRadius = UDim.new(0,10)

local title = Instance.new("TextLabel", main)
title.Size = UDim2.new(1,0,0,24)
title.BackgroundTransparency = 1
title.Text = "Hirekatu Had"
title.Font = Enum.Font.GothamBold
title.TextSize = 16
title.TextColor3 = Color3.new(1,1,1)

-- ===== Utils =====
local function getChar()
	local c = LP.Character or LP.CharacterAdded:Wait()
	return c, c:WaitForChild("HumanoidRootPart"), c:FindFirstChildOfClass("Humanoid")
end

local function makeBtn(parent,text,x,y,w,h)
	local b = Instance.new("TextButton",parent)
	b.Size = UDim2.fromOffset(w,h)
	b.Position = UDim2.fromOffset(x,y)
	b.Text = text
	b.Font = Enum.Font.GothamBold
	b.TextSize = 14
	b.TextColor3 = Color3.new(1,1,1)
	b.BackgroundColor3 = Color3.fromRGB(255,0,0)
	b.BorderSizePixel = 0
	Instance.new("UICorner", b).CornerRadius = UDim.new(0,6)
	return b
end

local RED, GREEN, BLUE = Color3.fromRGB(255,0,0), Color3.fromRGB(0,255,0), Color3.fromRGB(0,120,255)

-- ===== Speed & Jump =====
local speedMode = 0 -- 0=OFF / 1=Speed / 2=Grab
local SPEED_FAST, SPEED_GRAB = 53.5, 27.5
local speedConn = nil
local lowG, vf, att = false

local btnW1, btnH1 = 70, 26
local spacing = 6
local totalWidth = btnW1*2+spacing
local startX = (180-totalWidth)/2
local topY = 35

local speedBtn = makeBtn(main,"Speed",startX,topY,btnW1,btnH1)
local jumpBtn  = makeBtn(main,"Jump",startX+btnW1+spacing,topY,btnW1,btnH1)
local jumpOn = false

local function startSpeed()
	if speedConn then return end
	speedConn = RunService.Heartbeat:Connect(function()
		local _, hrp, hum = getChar()
		if hum.MoveDirection.Magnitude>0 then
			local s = speedMode==2 and SPEED_GRAB or SPEED_FAST
			hrp.AssemblyLinearVelocity = Vector3.new(hum.MoveDirection.X*s, hrp.AssemblyLinearVelocity.Y, hum.MoveDirection.Z*s)
		end
	end)
end

local function stopSpeed()
	if speedConn then speedConn:Disconnect() speedConn=nil end
end

local function enableLowG()
	if lowG then return end
	lowG=true
	local _, hrp, hum = getChar()
	att = Instance.new("Attachment",hrp)
	vf = Instance.new("VectorForce",hrp)
	vf.Attachment0 = att
	vf.RelativeTo = Enum.ActuatorRelativeTo.World
	UserInputService.JumpRequest:Connect(function()
		if lowG and hum.FloorMaterial~=Enum.Material.Air then
			hrp.Velocity = Vector3.new(hrp.Velocity.X,63,hrp.Velocity.Z)
		end
	end)
	RunService.Heartbeat:Connect(function()
		if not lowG then return end
		local mass = hrp.AssemblyMass
		local hv = Vector3.new(hrp.Velocity.X,0,hrp.Velocity.Z).Magnitude
		local div = 4.6 - math.clamp(hv/25,0,4.6-3.2)
		vf.Force = Vector3.new(0,workspace.Gravity*mass/div,0)
	end)
end

local function disableLowG()
	lowG=false
	if vf then vf:Destroy() end
	if att then att:Destroy() end
end

local function updateSpeedBtn()
	if speedMode==0 then
		speedBtn.BackgroundColor3=RED
		stopSpeed()
	elseif speedMode==1 then
		speedBtn.BackgroundColor3=GREEN
		startSpeed()
	else
		speedBtn.BackgroundColor3=BLUE
		startSpeed()
	end
end

speedBtn.MouseButton1Click:Connect(function()
	speedMode=(speedMode+1)%3
	updateSpeedBtn()
end)

jumpBtn.MouseButton1Click:Connect(function()
	jumpOn=not jumpOn
	jumpBtn.BackgroundColor3 = jumpOn and GREEN or RED
	if jumpOn then enableLowG() else disableLowG() end
end)

-- ===== Hit & Auto =====
local btnW2, btnH2 = 140, 26
local bottomStartY = 65
local hitBtn  = makeBtn(main,"Hit",(180-btnW2)/2,bottomStartY,btnW2,btnH2)
local autoBtn = makeBtn(main,"Auto",(180-btnW2)/2,bottomStartY+btnH2+4,btnW2,btnH2)
local hitOn, autoOn, nextAttack = false, false, 0

hitBtn.MouseButton1Click:Connect(function()
	hitOn = not hitOn
	hitBtn.BackgroundColor3 = hitOn and GREEN or RED
end)

autoBtn.MouseButton1Click:Connect(function()
	autoOn = not autoOn
	autoBtn.BackgroundColor3 = autoOn and GREEN or RED
end)

-- ===== 攻撃処理 =====
local ATTACK_RANGE, ATTACK_MIN, ATTACK_MAX = 10, 0.7, 0.9
local AUTO_WEAPON_NAME="Bat"

local function getNearestTarget(hrp2)
	local nearest, dist = nil, ATTACK_RANGE
	for _,plr in ipairs(Players:GetPlayers()) do
		if plr~=LP and plr.Character then
			local trp = plr.Character:FindFirstChild("HumanoidRootPart")
			if trp then
				local d = (hrp2.Position-trp.Position).Magnitude
				if d<dist then
					dist=d
					nearest=plr
				end
			end
		end
	end
	return nearest
end

local function lookAt(hrp2,trp)
	hrp2.CFrame = CFrame.new(hrp2.Position, Vector3.new(trp.Position.X,hrp2.Position.Y,trp.Position.Z))
end

local function getTool(ch)
	for _,v in ipairs(ch:GetChildren()) do
		if v:IsA("Tool") then return v end
	end
end

local function getOrEquipBat(ch)
	for _,v in ipairs(ch:GetChildren()) do
		if v:IsA("Tool") and v.Name==AUTO_WEAPON_NAME then return v end
	end
	local bp = LP:FindFirstChild("Backpack")
	if not bp then return end
	for _,v in ipairs(bp:GetChildren()) do
		if v:IsA("Tool") and v.Name==AUTO_WEAPON_NAME then
			ch.Humanoid:EquipTool(v)
			return v
		end
	end
end

RunService.Heartbeat:Connect(function()
	if not (hitOn or autoOn) then return end
	if tick()<nextAttack then return end

	local ch = LP.Character
	if not ch then return end
	local hrp2 = ch:FindFirstChild("HumanoidRootPart")
	if not hrp2 then return end

	local target = getNearestTarget(hrp2)
	if not target or not target.Character then return end
	local trp = target.Character:FindFirstChild("HumanoidRootPart")
	if not trp then return end

	local tool = autoOn and getOrEquipBat(ch) or getTool(ch)
	if not tool then return end

	lookAt(hrp2,trp)
	pcall(function() tool:Activate() end)

	nextAttack = tick() + math.random()*(ATTACK_MAX-ATTACK_MIN)+ATTACK_MIN
end)

-- ===== Grab Speed検知 =====
ProximityPromptService.PromptButtonHoldBegan:Connect(function(prompt,plr)
	if plr~=LP then return end
	if speedMode==0 then return end
	speedMode=2
	updateSpeedBtn()
end)
