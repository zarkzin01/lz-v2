--[[
    LZ HUB v1.2 - COMPLETO
    Abas:
    - Drift / Ré
    - Ângulo da Roda
    - Aceleração
    SHIFT/S = ré+drift | W = acelerar | ESPAÇO = drift | A/D = ângulo | F1 = painel
]]

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local LP = Players.LocalPlayer

-- ===================== CONFIG =====================
local cfg = {
	-- Drift / Ré
	speed = 1,
	force = 120,
	driftPower = 40,
	autoDrift = true,

	-- Aceleração
	accelSpeed = 1,
	accelForce = 120,
	accelDriftPower = 40,
	accelAutoDrift = false,
}

local steerCfg = {
	enabled = false,
	maxAngle = 0.40,
	speed = 0.50,
	current = 0,
	isA = false,
	isD = false,
}

local isRev, isAccel, isDrift = false, false, false
local curCar = nil
local running = false
local loopThread = nil
local currentTab = "Drift"

-- ===== DETECÇÃO DE CARRO =====
local function findCar()
	local char = LP.Character
	if not char or not char.Parent then return nil end
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hum then return nil end
	local seat = hum.SeatPart
	if not seat or not seat:IsA("VehicleSeat") then return nil end

	local p = seat.Parent
	while p and p ~= workspace do
		if p:IsA("Model") and (p.PrimaryPart or p:FindFirstChildWhichIsA("BasePart")) then
			return p, seat
		end
		p = p.Parent
	end
	return seat.Parent, seat
end

local function findPlayerCarCDT()
	local folder = workspace:FindFirstChild("Cars")
	if not folder then return nil end
	for _, car in ipairs(folder:GetChildren()) do
		local stats = car:FindFirstChild("Stats")
		if stats then
			local owner = stats:FindFirstChild("Owner")
			if owner and (owner.Value == LP.Name or owner.Value == LP or owner.Value == LP.UserId) then
				return car
			end
		end
	end
	return nil
end

-- ===== EIXO TRASEIRO =====
local function getRearAxle(car)
	if not car then return nil end

	local base = car.PrimaryPart
	if not base then
		for _, p in ipairs(car:GetDescendants()) do
			if p:IsA("BasePart") then base = p break end
		end
	end
	if not base then return nil end

	local parts = {}
	for _, p in ipairs(car:GetDescendants()) do
		if p:IsA("BasePart") then table.insert(parts, p) end
	end

	local rearNames = {"rear", "back", "traseira", "traseiro", "rwheel", "bwheel", "rl", "rr"}
	local rearParts = {}
	for _, p in ipairs(parts) do
		local n = p.Name:lower()
		for _, rn in ipairs(rearNames) do
			if n:find(rn) then
				table.insert(rearParts, p)
				break
			end
		end
	end
	if #rearParts > 0 then return rearParts end

	local look = base.CFrame.LookVector
	local basePos = base.Position
	local bestScore = -math.huge
	local rearPart = nil

	for _, p in ipairs(parts) do
		local size = p.Size.Magnitude
		if size > 2 and size < 30 then
			local offset = p.Position - basePos
			local score = -(offset:Dot(look))
			if score > bestScore then
				bestScore = score
				rearPart = p
			end
		end
	end

	if rearPart then return {rearPart} end
	return {base}
end

-- ===== ÂNGULO DAS RODAS =====
local function applySteer(car, angle)
	if not car then return end
	local wheels = {
		car:FindFirstChild("FR", true),
		car:FindFirstChild("FL", true)
	}
	for _, wheel in pairs(wheels) do
		if wheel then
			local axel = wheel:FindFirstChild("Axel") or wheel:FindFirstChild("Axle")
			if axel then
				local att = axel:FindFirstChild("Attachment0")
				if att then
					local cur = att.Axis
					att.Axis = Vector3.new(cur.X, cur.Y, angle)
				end
			end
		end
	end
end

-- ===== LOOP PRINCIPAL (Ré + Aceleração) =====
local function stopLoop()
	running = false
	if loopThread then
		task.cancel(loopThread)
		loopThread = nil
	end
end

local function startLoop()
	if running then return end
	running = true
	loopThread = task.spawn(function()
		while running do
			local char = LP.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			local seat = hum and hum.SeatPart

			if not seat or not seat:IsA("VehicleSeat") then
				running = false
				break
			end

			local car = findCar()
			if not car then
				running = false
				break
			end

			curCar = car

			local base = car.PrimaryPart
			if not base then
				for _, p in ipairs(car:GetDescendants()) do
					if p:IsA("BasePart") then base = p break end
				end
			end

			if base and base.AssemblyMass > 0 then
				local mass = base.AssemblyMass
				local interval = 0.1
				local rearParts = getRearAxle(car)

				-- ===== RÉ =====
				if isRev and cfg.speed > 0 then
					local back = -base.CFrame.LookVector
					local impulse = back * cfg.speed * cfg.force * mass * interval

					if rearParts then
						for _, p in ipairs(rearParts) do
							if p and p.Parent and p.AssemblyMass > 0 then
								p:ApplyImpulse(impulse / #rearParts)
							end
						end
					end

					if isDrift and cfg.driftPower > 0 then
						local velocity = base.AssemblyLinearVelocity.Magnitude
						local gripFactor = math.clamp(1 - (velocity / 100) * (cfg.driftPower / 100), 0.1, 0.9)
						for _, p in ipairs(rearParts or {}) do
							if p and p.Parent then
								p.CustomPhysicalProperties = PhysicalProperties.new(0.7, gripFactor, 0.3, 1, 1)
							end
						end
					end
				end

				-- ===== ACELERAÇÃO =====
				if isAccel and cfg.accelSpeed > 0 then
					local forward = base.CFrame.LookVector
					local impulse = forward * cfg.accelSpeed * cfg.accelForce * mass * interval

					if rearParts then
						for _, p in ipairs(rearParts) do
							if p and p.Parent and p.AssemblyMass > 0 then
								p:ApplyImpulse(impulse / #rearParts)
							end
						end
					end

					if isDrift and cfg.accelDriftPower > 0 then
						local velocity = base.AssemblyLinearVelocity.Magnitude
						local gripFactor = math.clamp(1 - (velocity / 100) * (cfg.accelDriftPower / 100), 0.1, 0.9)
						for _, p in ipairs(rearParts or {}) do
							if p and p.Parent then
								p.CustomPhysicalProperties = PhysicalProperties.new(0.7, gripFactor, 0.3, 1, 1)
							end
						end
					end
				end

				-- Restaura grip se não estiver driftando
				if not isDrift then
					for _, p in ipairs(rearParts or {}) do
						if p and p.Parent then
							p.CustomPhysicalProperties = nil
						end
					end
				end
			end

			task.wait(0.1)
		end
	end)
end

-- ===== LOOP ÂNGULO =====
RunService.RenderStepped:Connect(function(dt)
	local cdtCar = findPlayerCarCDT()
	if cdtCar then curCar = cdtCar end

	if not steerCfg.enabled or not curCar then return end

	local dir = 0
	if steerCfg.isA and not steerCfg.isD then dir = -1
	elseif steerCfg.isD and not steerCfg.isA then dir = 1 end

	if dir ~= 0 then
		steerCfg.current = math.clamp(steerCfg.current + (dir * steerCfg.speed * dt), -steerCfg.maxAngle, steerCfg.maxAngle)
	else
		if steerCfg.current > 0 then
			steerCfg.current = math.max(0, steerCfg.current - steerCfg.speed * dt)
		elseif steerCfg.current < 0 then
			steerCfg.current = math.min(0, steerCfg.current + steerCfg.speed * dt)
		end
	end

	applySteer(curCar, steerCfg.current)
end)

-- ===== INPUTS =====
UIS.InputBegan:Connect(function(i, gp)
	if gp then return end
	local k = i.KeyCode

	if k == Enum.KeyCode.S or k == Enum.KeyCode.Down or k == Enum.KeyCode.LeftShift or k == Enum.KeyCode.RightShift then
		isRev = true
		if cfg.autoDrift then isDrift = true end
	elseif k == Enum.KeyCode.W or k == Enum.KeyCode.Up then
		isAccel = true
		if cfg.accelAutoDrift then isDrift = true end
	elseif k == Enum.KeyCode.Space then
		isDrift = true
	elseif k == Enum.KeyCode.A then
		steerCfg.isA = true
	elseif k == Enum.KeyCode.D then
		steerCfg.isD = true
	elseif k == Enum.KeyCode.F1 then
		if _G.lzGui then _G.lzGui.Enabled = not _G.lzGui.Enabled end
	end
end)

UIS.InputEnded:Connect(function(i, gp)
	if gp then return end
	local k = i.KeyCode

	if k == Enum.KeyCode.S or k == Enum.KeyCode.Down or k == Enum.KeyCode.LeftShift or k == Enum.KeyCode.RightShift then
		isRev = false
		if not UIS:IsKeyDown(Enum.KeyCode.Space) and not (cfg.accelAutoDrift and isAccel) then
			isDrift = false
		end
	elseif k == Enum.KeyCode.W or k == Enum.KeyCode.Up then
		isAccel = false
		if not UIS:IsKeyDown(Enum.KeyCode.Space) and not (cfg.autoDrift and isRev) then
			isDrift = false
		end
	elseif k == Enum.KeyCode.Space then
		if not ((cfg.autoDrift and isRev) or (cfg.accelAutoDrift and isAccel)) then
			isDrift = false
		end
	elseif k == Enum.KeyCode.A then
		steerCfg.isA = false
	elseif k == Enum.KeyCode.D then
		steerCfg.isD = false
	end
end)

-- ===== DETECÇÃO DE SENTAR =====
local function hookCharacter(char)
	local hum = char:WaitForChild("Humanoid", 5)
	if not hum then return end

	hum.Seated:Connect(function(active, seat)
		if active and seat and seat:IsA("VehicleSeat") then
			task.wait(0.1)
			startLoop()
		else
			stopLoop()
			curCar = nil
			steerCfg.current = 0
		end
	end)
end

if LP.Character then hookCharacter(LP.Character) end
LP.CharacterAdded:Connect(hookCharacter)

-- ===== ARRASTAR =====
local function makeDraggable(frame)
	local dragging, dragStart, startPos = false, nil, nil
	frame.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			dragStart = input.Position
			startPos = frame.Position
		end
	end)
	frame.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end)
	UIS.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			local delta = input.Position - dragStart
			frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
		end
	end)
end

-- ===== UI =====
local gui = Instance.new("ScreenGui")
gui.Name = "LZ_HUB_v1"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.Parent = LP:WaitForChild("PlayerGui")
_G.lzGui = gui

local f = Instance.new("Frame")
f.Size = UDim2.new(0, 260, 0, 340)
f.Position = UDim2.new(0.5, -130, 0.12, 0)
f.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
f.BorderSizePixel = 0
f.Active = true
f.Parent = gui
Instance.new("UICorner", f).CornerRadius = UDim.new(0, 10)
makeDraggable(f)

-- Título
local title = Instance.new("TextLabel", f)
title.Size = UDim2.new(1, -30, 0, 24)
title.Position = UDim2.new(0, 8, 0, 2)
title.BackgroundTransparency = 1
title.Text = "⚡ LZ HUB v1.2"
title.TextColor3 = Color3.fromRGB(150, 100, 255)
title.Font = Enum.Font.GothamBold
title.TextSize = 14
title.TextXAlignment = Enum.TextXAlignment.Left

-- Status
local status = Instance.new("TextLabel", f)
status.Size = UDim2.new(1, -10, 0, 14)
status.Position = UDim2.new(0, 5, 0, 24)
status.BackgroundTransparency = 1
status.Text = "🔴 Sem carro"
status.TextColor3 = Color3.fromRGB(255, 100, 100)
status.Font = Enum.Font.Gotham
status.TextSize = 11
status.TextXAlignment = Enum.TextXAlignment.Left

task.spawn(function()
	while true do
		task.wait(0.3)
		if curCar and curCar.Parent then
			status.Text = "🟢 " .. curCar.Name
			status.TextColor3 = Color3.fromRGB(100, 255, 100)
		else
			status.Text = "🔴 Sem carro"
			status.TextColor3 = Color3.fromRGB(255, 100, 100)
		end
	end
end)

-- ===== ABAS (3) =====
local tabDrift = Instance.new("TextButton", f)
tabDrift.Size = UDim2.new(0.31, 0, 0, 24)
tabDrift.Position = UDim2.new(0.02, 0, 0, 44)
tabDrift.BackgroundColor3 = Color3.fromRGB(150, 100, 255)
tabDrift.Text = "Drift/Ré"
tabDrift.TextColor3 = Color3.fromRGB(255, 255, 255)
tabDrift.Font = Enum.Font.GothamBold
tabDrift.TextSize = 11
Instance.new("UICorner", tabDrift).CornerRadius = UDim.new(0, 5)

local tabAngulo = Instance.new("TextButton", f)
tabAngulo.Size = UDim2.new(0.31, 0, 0, 24)
tabAngulo.Position = UDim2.new(0.345, 0, 0, 44)
tabAngulo.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
tabAngulo.Text = "Ângulo"
tabAngulo.TextColor3 = Color3.fromRGB(255, 255, 255)
tabAngulo.Font = Enum.Font.GothamBold
tabAngulo.TextSize = 11
Instance.new("UICorner", tabAngulo).CornerRadius = UDim.new(0, 5)

local tabAccel = Instance.new("TextButton", f)
tabAccel.Size = UDim2.new(0.31, 0, 0, 24)
tabAccel.Position = UDim2.new(0.67, 0, 0, 44)
tabAccel.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
tabAccel.Text = "Aceleração"
tabAccel.TextColor3 = Color3.fromRGB(255, 255, 255)
tabAccel.Font = Enum.Font.GothamBold
tabAccel.TextSize = 11
Instance.new("UICorner", tabAccel).CornerRadius = UDim.new(0, 5)

-- Containers
local contentDrift = Instance.new("Frame", f)
contentDrift.Size = UDim2.new(1, -10, 1, -80)
contentDrift.Position = UDim2.new(0, 5, 0, 74)
contentDrift.BackgroundTransparency = 1

local contentAngulo = Instance.new("Frame", f)
contentAngulo.Size = UDim2.new(1, -10, 1, -80)
contentAngulo.Position = UDim2.new(0, 5, 0, 74)
contentAngulo.BackgroundTransparency = 1
contentAngulo.Visible = false

local contentAccel = Instance.new("Frame", f)
contentAccel.Size = UDim2.new(1, -10, 1, -80)
contentAccel.Position = UDim2.new(0, 5, 0, 74)
contentAccel.BackgroundTransparency = 1
contentAccel.Visible = false

-- ========== ABA DRIFT / RÉ ==========
local function createTextBox(parent, pos, placeholder)
	local box = Instance.new("TextBox", parent)
	box.Size = UDim2.new(0.9, 0, 0, 28)
	box.Position = pos
	box.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	box.TextColor3 = Color3.fromRGB(255, 255, 255)
	box.PlaceholderText = placeholder
	box.Text = ""
	box.Font = Enum.Font.Gotham
	box.TextSize = 13
	box.ClearTextOnFocus = true
	Instance.new("UICorner", box).CornerRadius = UDim.new(0, 6)
	return box
end

local function createButton(parent, pos, text, color)
	local btn = Instance.new("TextButton", parent)
	btn.Size = UDim2.new(0.9, 0, 0, 28)
	btn.Position = pos
	btn.BackgroundColor3 = color
	btn.Text = text
	btn.TextColor3 = Color3.fromRGB(255, 255, 255)
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 13
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
	return btn
end

local inputDrift = createTextBox(contentDrift, UDim2.new(0.05, 0, 0, 0), "Velocidade (ex: 1.2)")
local applyDrift = createButton(contentDrift, UDim2.new(0.05, 0, 0, 34), "Aplicar", Color3.fromRGB(150, 100, 255))

applyDrift.MouseButton1Click:Connect(function()
	local n = tonumber(inputDrift.Text)
	if n and n > 0 then
		cfg.speed = n
		applyDrift.Text = "✓ " .. n
		task.delay(1, function() applyDrift.Text = "Aplicar" end)
	else
		applyDrift.Text = "Inválido"
		task.delay(1, function() applyDrift.Text = "Aplicar" end)
	end
end)

local bDrift = createButton(contentDrift, UDim2.new(0.05, 0, 0, 70), "Drift Auto: ON", Color3.fromRGB(0, 150, 80))
bDrift.MouseButton1Click:Connect(function()
	cfg.autoDrift = not cfg.autoDrift
	bDrift.Text = "Drift Auto: " .. (cfg.autoDrift and "ON" or "OFF")
	bDrift.BackgroundColor3 = cfg.autoDrift and Color3.fromRGB(0, 150, 80) or Color3.fromRGB(150, 50, 50)
end)

local flabel = Instance.new("TextLabel", contentDrift)
flabel.Size = UDim2.new(0.9, 0, 0, 14)
flabel.Position = UDim2.new(0.05, 0, 0, 106)
flabel.BackgroundTransparency = 1
flabel.Text = "Força impulso: " .. cfg.force
flabel.TextColor3 = Color3.fromRGB(180, 180, 180)
flabel.Font = Enum.Font.Gotham
flabel.TextSize = 11
flabel.TextXAlignment = Enum.TextXAlignment.Left

local fm = Instance.new("TextButton", contentDrift)
fm.Size = UDim2.new(0.15, 0, 0, 26)
fm.Position = UDim2.new(0.05, 0, 0, 122)
fm.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
fm.Text = "−"
fm.TextColor3 = Color3.fromRGB(255, 255, 255)
fm.Font = Enum.Font.GothamBold
fm.TextSize = 16
Instance.new("UICorner", fm).CornerRadius = UDim.new(0, 6)

local fp = Instance.new("TextButton", contentDrift)
fp.Size = UDim2.new(0.15, 0, 0, 26)
fp.Position = UDim2.new(0.80, 0, 0, 122)
fp.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
fp.Text = "+"
fp.TextColor3 = Color3.fromRGB(255, 255, 255)
fp.Font = Enum.Font.GothamBold
fp.TextSize = 16
Instance.new("UICorner", fp).CornerRadius = UDim.new(0, 6)

local fv = Instance.new("TextLabel", contentDrift)
fv.Size = UDim2.new(0.55, 0, 0, 26)
fv.Position = UDim2.new(0.225, 0, 0, 122)
fv.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
fv.Text = tostring(cfg.force)
fv.TextColor3 = Color3.fromRGB(255, 255, 255)
fv.Font = Enum.Font.GothamBold
fv.TextSize = 13
Instance.new("UICorner", fv).CornerRadius = UDim.new(0, 6)

fm.MouseButton1Click:Connect(function()
	cfg.force = math.max(10, cfg.force - 10)
	fv.Text = tostring(cfg.force)
	flabel.Text = "Força impulso: " .. cfg.force
end)
fp.MouseButton1Click:Connect(function()
	cfg.force = math.min(1000, cfg.force + 10)
	fv.Text = tostring(cfg.force)
	flabel.Text = "Força impulso: " .. cfg.force
end)

local dlabel = Instance.new("TextLabel", contentDrift)
dlabel.Size = UDim2.new(0.9, 0, 0, 14)
dlabel.Position = UDim2.new(0.05, 0, 0, 156)
dlabel.BackgroundTransparency = 1
dlabel.Text = "Drift (grip): " .. cfg.driftPower
dlabel.TextColor3 = Color3.fromRGB(180, 180, 180)
dlabel.Font = Enum.Font.Gotham
dlabel.TextSize = 11
dlabel.TextXAlignment = Enum.TextXAlignment.Left

local dm = Instance.new("TextButton", contentDrift)
dm.Size = UDim2.new(0.15, 0, 0, 26)
dm.Position = UDim2.new(0.05, 0, 0, 172)
dm.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
dm.Text = "−"
dm.TextColor3 = Color3.fromRGB(255, 255, 255)
dm.Font = Enum.Font.GothamBold
dm.TextSize = 16
Instance.new("UICorner", dm).CornerRadius = UDim.new(0, 6)

local dp = Instance.new("TextButton", contentDrift)
dp.Size = UDim2.new(0.15, 0, 0, 26)
dp.Position = UDim2.new(0.80, 0, 0, 172)
dp.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
dp.Text = "+"
dp.TextColor3 = Color3.fromRGB(255, 255, 255)
dp.Font = Enum.Font.GothamBold
dp.TextSize = 16
Instance.new("UICorner", dp).CornerRadius = UDim.new(0, 6)

local dv = Instance.new("TextLabel", contentDrift)
dv.Size = UDim2.new(0.55, 0, 0, 26)
dv.Position = UDim2.new(0.225, 0, 0, 172)
dv.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
dv.Text = tostring(cfg.driftPower)
dv.TextColor3 = Color3.fromRGB(255, 255, 255)
dv.Font = Enum.Font.GothamBold
dv.TextSize = 13
Instance.new("UICorner", dv).CornerRadius = UDim.new(0, 6)

dm.MouseButton1Click:Connect(function()
	cfg.driftPower = math.max(10, cfg.driftPower - 5)
	dv.Text = tostring(cfg.driftPower)
	dlabel.Text = "Drift (grip): " .. cfg.driftPower
end)
dp.MouseButton1Click:Connect(function()
	cfg.driftPower = math.min(200, cfg.driftPower + 5)
	dv.Text = tostring(cfg.driftPower)
	dlabel.Text = "Drift (grip): " .. cfg.driftPower
end)

-- ========== ABA ÂNGULO ==========
local steerToggle = createButton(contentAngulo, UDim2.new(0.05, 0, 0, 0), "Ângulo: OFF", Color3.fromRGB(80, 80, 80))
steerToggle.MouseButton1Click:Connect(function()
	steerCfg.enabled = not steerCfg.enabled
	steerToggle.Text = "Ângulo: " .. (steerCfg.enabled and "ON" or "OFF")
	steerToggle.BackgroundColor3 = steerCfg.enabled and Color3.fromRGB(0, 150, 80) or Color3.fromRGB(80, 80, 80)
	if not steerCfg.enabled then
		steerCfg.current = 0
		if curCar then applySteer(curCar, 0) end
	end
end)

local angleLabel = Instance.new("TextLabel", contentAngulo)
angleLabel.Size = UDim2.new(0.9, 0, 0, 14)
angleLabel.Position = UDim2.new(0.05, 0, 0, 38)
angleLabel.BackgroundTransparency = 1
angleLabel.Text = "Ângulo Máximo:"
angleLabel.TextColor3 = Color3.fromRGB(180, 180, 180)
angleLabel.Font = Enum.Font.Gotham
angleLabel.TextSize = 12
angleLabel.TextXAlignment = Enum.TextXAlignment.Left

local angleBox = createTextBox(contentAngulo, UDim2.new(0.05, 0, 0, 54), "0.40")
angleBox.Text = "0.40"

local speedLabel = Instance.new("TextLabel", contentAngulo)
speedLabel.Size = UDim2.new(0.9, 0, 0, 14)
speedLabel.Position = UDim2.new(0.05, 0, 0, 90)
speedLabel.BackgroundTransparency = 1
speedLabel.Text = "Velocidade do Giro:"
speedLabel.TextColor3 = Color3.fromRGB(180, 180, 180)
speedLabel.Font = Enum.Font.Gotham
speedLabel.TextSize = 12
speedLabel.TextXAlignment = Enum.TextXAlignment.Left

local speedBox = createTextBox(contentAngulo, UDim2.new(0.05, 0, 0, 106), "0.50")
speedBox.Text = "0.50"

local applySteerBtn = createButton(contentAngulo, UDim2.new(0.05, 0, 0, 144), "Aplicar", Color3.fromRGB(150, 100, 255))
applySteerBtn.MouseButton1Click:Connect(function()
	local a = tonumber((angleBox.Text:gsub(",", ".")))
	local s = tonumber((speedBox.Text:gsub(",", ".")))
	if a and s then
		steerCfg.maxAngle = a
		steerCfg.speed = s
		if not steerCfg.enabled then
			steerCfg.enabled = true
			steerToggle.Text = "Ângulo: ON"
			steerToggle.BackgroundColor3 = Color3.fromRGB(0, 150, 80)
		end
		applySteerBtn.Text = "✓ Aplicado"
		task.delay(1, function() applySteerBtn.Text = "Aplicar" end)
	else
		applySteerBtn.Text = "Inválido"
		task.delay(1, function() applySteerBtn.Text = "Aplicar" end)
	end
end)

local infoAngulo = Instance.new("TextLabel", contentAngulo)
infoAngulo.Size = UDim2.new(0.9, 0, 0, 40)
infoAngulo.Position = UDim2.new(0.05, 0, 0, 184)
infoAngulo.BackgroundTransparency = 1
infoAngulo.Text = "Use A / D para girar as rodas\nenquanto o Ângulo estiver ON"
infoAngulo.TextColor3 = Color3.fromRGB(140, 140, 140)
infoAngulo.Font = Enum.Font.Gotham
infoAngulo.TextSize = 11
infoAngulo.TextXAlignment = Enum.TextXAlignment.Left

-- ========== ABA ACELERAÇÃO ==========
local inputAccel = createTextBox(contentAccel, UDim2.new(0.05, 0, 0, 0), "Velocidade (ex: 1.2)")
local applyAccel = createButton(contentAccel, UDim2.new(0.05, 0, 0, 34), "Aplicar", Color3.fromRGB(100, 200, 255))

applyAccel.MouseButton1Click:Connect(function()
	local n = tonumber(inputAccel.Text)
	if n and n > 0 then
		cfg.accelSpeed = n
		applyAccel.Text = "✓ " .. n
		task.delay(1, function() applyAccel.Text = "Aplicar" end)
	else
		applyAccel.Text = "Inválido"
		task.delay(1, function() applyAccel.Text = "Aplicar" end)
	end
end)

local bAccel = createButton(contentAccel, UDim2.new(0.05, 0, 0, 70), "Drift Auto: OFF", Color3.fromRGB(150, 50, 50))
bAccel.MouseButton1Click:Connect(function()
	cfg.accelAutoDrift = not cfg.accelAutoDrift
	bAccel.Text = "Drift Auto: " .. (cfg.accelAutoDrift and "ON" or "OFF")
	bAccel.BackgroundColor3 = cfg.accelAutoDrift and Color3.fromRGB(0, 150, 80) or Color3.fromRGB(150, 50, 50)
end)

local flabelA = Instance.new("TextLabel", contentAccel)
flabelA.Size = UDim2.new(0.9, 0, 0, 14)
flabelA.Position = UDim2.new(0.05, 0, 0, 106)
flabelA.BackgroundTransparency = 1
flabelA.Text = "Força impulso: " .. cfg.accelForce
flabelA.TextColor3 = Color3.fromRGB(180, 180, 180)
flabelA.Font = Enum.Font.Gotham
flabelA.TextSize = 11
flabelA.TextXAlignment = Enum.TextXAlignment.Left

local fmA = Instance.new("TextButton", contentAccel)
fmA.Size = UDim2.new(0.15, 0, 0, 26)
fmA.Position = UDim2.new(0.05, 0, 0, 122)
fmA.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
fmA.Text = "−"
fmA.TextColor3 = Color3.fromRGB(255, 255, 255)
fmA.Font = Enum.Font.GothamBold
fmA.TextSize = 16
Instance.new("UICorner", fmA).CornerRadius = UDim.new(0, 6)

local fpA = Instance.new("TextButton", contentAccel)
fpA.Size = UDim2.new(0.15, 0, 0, 26)
fpA.Position = UDim2.new(0.80, 0, 0, 122)
fpA.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
fpA.Text = "+"
fpA.TextColor3 = Color3.fromRGB(255, 255, 255)
fpA.Font = Enum.Font.GothamBold
fpA.TextSize = 16
Instance.new("UICorner", fpA).CornerRadius = UDim.new(0, 6)

local fvA = Instance.new("TextLabel", contentAccel)
fvA.Size = UDim2.new(0.55, 0, 0, 26)
fvA.Position = UDim2.new(0.225, 0, 0, 122)
fvA.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
fvA.Text = tostring(cfg.accelForce)
fvA.TextColor3 = Color3.fromRGB(255, 255, 255)
fvA.Font = Enum.Font.GothamBold
fvA.TextSize = 13
Instance.new("UICorner", fvA).CornerRadius = UDim.new(0, 6)

fmA.MouseButton1Click:Connect(function()
	cfg.accelForce = math.max(10, cfg.accelForce - 10)
	fvA.Text = tostring(cfg.accelForce)
	flabelA.Text = "Força impulso: " .. cfg.accelForce
end)
fpA.MouseButton1Click:Connect(function()
	cfg.accelForce = math.min(1000, cfg.accelForce + 10)
	fvA.Text = tostring(cfg.accelForce)
	flabelA.Text = "Força impulso: " .. cfg.accelForce
end)

local dlabelA = Instance.new("TextLabel", contentAccel)
dlabelA.Size = UDim2.new(0.9, 0, 0, 14)
dlabelA.Position = UDim2.new(0.05, 0, 0, 156)
dlabelA.BackgroundTransparency = 1
dlabelA.Text = "Drift (grip): " .. cfg.accelDriftPower
dlabelA.TextColor3 = Color3.fromRGB(180, 180, 180)
dlabelA.Font = Enum.Font.Gotham
dlabelA.TextSize = 11
dlabelA.TextXAlignment = Enum.TextXAlignment.Left

local dmA = Instance.new("TextButton", contentAccel)
dmA.Size = UDim2.new(0.15, 0, 0, 26)
dmA.Position = UDim2.new(0.05, 0, 0, 172)
dmA.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
dmA.Text = "−"
dmA.TextColor3 = Color3.fromRGB(255, 255, 255)
dmA.Font = Enum.Font.GothamBold
dmA.TextSize = 16
Instance.new("UICorner", dmA).CornerRadius = UDim.new(0, 6)

local dpA = Instance.new("TextButton", contentAccel)
dpA.Size = UDim2.new(0.15, 0, 0, 26)
dpA.Position = UDim2.new(0.80, 0, 0, 172)
dpA.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
dpA.Text = "+"
dpA.TextColor3 = Color3.fromRGB(255, 255, 255)
dpA.Font = Enum.Font.GothamBold
dpA.TextSize = 16
Instance.new("UICorner", dpA).CornerRadius = UDim.new(0, 6)

local dvA = Instance.new("TextLabel", contentAccel)
dvA.Size = UDim2.new(0.55, 0, 0, 26)
dvA.Position = UDim2.new(0.225, 0, 0, 172)
dvA.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
dvA.Text = tostring(cfg.accelDriftPower)
dvA.TextColor3 = Color3.fromRGB(255, 255, 255)
dvA.Font = Enum.Font.GothamBold
dvA.TextSize = 13
Instance.new("UICorner", dvA).CornerRadius = UDim.new(0, 6)

dmA.MouseButton1Click:Connect(function()
	cfg.accelDriftPower = math.max(10, cfg.accelDriftPower - 5)
	dvA.Text = tostring(cfg.accelDriftPower)
	dlabelA.Text = "Drift (grip): " .. cfg.accelDriftPower
end)
dpA.MouseButton1Click:Connect(function()
	cfg.accelDriftPower = math.min(200, cfg.accelDriftPower + 5)
	dvA.Text = tostring(cfg.accelDriftPower)
	dlabelA.Text = "Drift (grip): " .. cfg.accelDriftPower
end)

local infoAccel = Instance.new("TextLabel", contentAccel)
infoAccel.Size = UDim2.new(0.9, 0, 0, 30)
infoAccel.Position = UDim2.new(0.05, 0, 0, 210)
infoAccel.BackgroundTransparency = 1
infoAccel.Text = "W / Seta Cima = Acelerar\nESPAÇO = Drift"
infoAccel.TextColor3 = Color3.fromRGB(140, 140, 140)
infoAccel.Font = Enum.Font.Gotham
infoAccel.TextSize = 11
infoAccel.TextXAlignment = Enum.TextXAlignment.Left

-- ===== TROCA DE ABAS =====
local function switchTab(tab)
	currentTab = tab
	contentDrift.Visible = (tab == "Drift")
	contentAngulo.Visible = (tab == "Angulo")
	contentAccel.Visible = (tab == "Accel")

	tabDrift.BackgroundColor3 = (tab == "Drift") and Color3.fromRGB(150, 100, 255) or Color3.fromRGB(50, 50, 50)
	tabAngulo.BackgroundColor3 = (tab == "Angulo") and Color3.fromRGB(150, 100, 255) or Color3.fromRGB(50, 50, 50)
	tabAccel.BackgroundColor3 = (tab == "Accel") and Color3.fromRGB(100, 200, 255) or Color3.fromRGB(50, 50, 50)
end

tabDrift.MouseButton1Click:Connect(function() switchTab("Drift") end)
tabAngulo.MouseButton1Click:Connect(function() switchTab("Angulo") end)
tabAccel.MouseButton1Click:Connect(function() switchTab("Accel") end)

-- Fechar / Abrir
local closeBtn = Instance.new("TextButton", f)
closeBtn.Size = UDim2.new(0, 24, 0, 24)
closeBtn.Position = UDim2.new(1, -28, 0, 2)
closeBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
closeBtn.Text = "✕"
closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 14
Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 6)

local openBtn = Instance.new("TextButton", gui)
openBtn.Size = UDim2.new(0, 45, 0, 45)
openBtn.Position = UDim2.new(0, 20, 0.5, -22)
openBtn.BackgroundColor3 = Color3.fromRGB(150, 100, 255)
openBtn.Text = "LZ"
openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
openBtn.Font = Enum.Font.GothamBold
openBtn.TextSize = 16
openBtn.Visible = false
openBtn.Active = true
Instance.new("UICorner", openBtn).CornerRadius = UDim.new(1, 0)
makeDraggable(openBtn)

closeBtn.MouseButton1Click:Connect(function()
	f.Visible = false
	openBtn.Visible = true
end)

openBtn.MouseButton1Click:Connect(function()
	f.Visible = true
	openBtn.Visible = false
end)

print("[LZ HUB v1.2] W=Acelerar | S/SHIFT=Ré+Drift | ESPAÇO=Drift | A/D=Ângulo | F1=Painel")
