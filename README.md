--[[ Hub clean rebuild ]]
pcall(function()
	local pg = game.Players.LocalPlayer:FindFirstChild("PlayerGui")
	if pg then
		for _, n in ipairs({"HubRev", "HubRevBoot", "HubH", "HubV3"}) do
			local o = pg:FindFirstChild(n)
			if o then o:Destroy() end
		end
	end
end)

getgenv()._HUB_REV = true

local Players = game:GetService("Players")
local RS = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local LP = Players.LocalPlayer
local Cam = Workspace.CurrentCamera

local S = {
	Aimbot = false,
	AimSmooth = 0.3,
	AimFOV = 300,
	SilentAim = false,
	Triggerbot = false,
	TrigFOV = 120,
	ShowFOV = false,
	TargetPriority = "Closest", -- Closest | Crosshair | LowHP
	Rage = false,
	VoidSpam = false,
	ESP = false,
	LevelSpoof = false,
	Level = 9999,
	WinStreakSpoof = false,
	WinStreak = 9999,
	DeviceSpoof = false,
	DeviceName = "Console",
	Noclip = false,
	UnlockAll = false,
	Fly = false,
	FlySpeed = 60,
	TeamCheck = true,
	WallCheck = true,
	RageInterval = 0.06,
	RapidFire = false,
	NoRecoil = false,
	NoSpread = false,
	WalkSpeed = false,
	WalkSpeedValue = 45,
	Freecam = false,
	FreecamSpeed = 50,
	ESP_Name = true,
	ESP_HealthBar = true,
	ESP_Distance = true,
	ESP_Box = true,
	ESP_Skeleton = true,
}
_G.HUB = S

local function notify(t)
	pcall(function()
		game:GetService("StarterGui"):SetCore("SendNotification", {
			Title = "Hub", Text = tostring(t), Duration = 2
		})
	end)
end

-- Device spoof (Lunara: SetControls remote)
local DEVICE_CFGS = {
	Mobile = { Code = "Touch" },
	Console = { Code = "Gamepad" },
	VR = { Code = "VR" },
	PC = { Code = "MouseKeyboard" },
	Computer = { Code = "MouseKeyboard" },
}
local lastDeviceApply = 0
local function getSetControlsRemote()
	local remotes = RS:FindFirstChild("Remotes")
	remotes = remotes and remotes:FindFirstChild("Replication")
	remotes = remotes and remotes:FindFirstChild("Fighter")
	return remotes and remotes:FindFirstChild("SetControls")
end

local function fireControls(code)
	local sc = getSetControlsRemote()
	if not sc then return false end
	local ok = pcall(function()
		sc:FireServer(code)
	end)
	return ok
end

local function realDeviceCode()
	local it = UIS:GetLastInputType()
	if it == Enum.UserInputType.Touch then
		return "Touch"
	elseif it == Enum.UserInputType.Gamepad1 or it == Enum.UserInputType.Gamepad2 then
		return "Gamepad"
	elseif it == Enum.UserInputType.Focus then -- unlikely
		return "MouseKeyboard"
	else
		-- mobile executors often still report mouse; prefer Touch if TouchEnabled
		if UIS.TouchEnabled and not UIS.KeyboardEnabled then
			return "Touch"
		end
		return "MouseKeyboard"
	end
end

function applyDeviceSpoof(force)
	-- ONLY when enabled
	if not S.DeviceSpoof then
		return
	end
	local now = tick()
	if not force and now - lastDeviceApply < 0.5 then
		return
	end
	lastDeviceApply = now
	local cfg = DEVICE_CFGS[S.DeviceName]
	if not cfg then return end
	for _ = 1, 3 do
		if fireControls(cfg.Code) then
			break
		end
		task.wait(0.2)
	end
end

function restoreRealDevice()
	-- call when spoof disabled so server gets real controls back
	local code = realDeviceCode()
	fireControls(code)
	print("[hub] device restored:", code)
end

task.spawn(function()
	task.wait(2)
	if S.DeviceSpoof then applyDeviceSpoof(true) end
	while true do
		task.wait(10)
		if S.DeviceSpoof then applyDeviceSpoof(false) end
	end
end)

LP.CharacterAdded:Connect(function()
	task.wait(1)
	if S.DeviceSpoof then
		task.wait(0.5)
		applyDeviceSpoof(true)
	end
end)

-- Lunara Unlock All (exact remote they use)
local unlockAllExecuted = false
function runLunaraUnlockAll()
	if unlockAllExecuted then
		notify("Unlock All already on")
		return
	end
	unlockAllExecuted = true
	task.spawn(function()
		local ok, err = pcall(function()
			-- same source as Lunara hub Misc > Unlock All
			loadstring(game:HttpGet("https://pastefy.app/6ElsMLeb/raw", true))()
		end)
		if ok then
			notify("Lunara Unlock All on")
			print("[hub] Lunara Unlock All loaded")
		else
			unlockAllExecuted = false
			S.UnlockAll = false
			warn("[hub] Unlock All fail:", err)
			notify("Unlock All fail")
		end
	end)
end

-- ========== AC (Lunara full) ==========
-- setmetatable kv trap (MiscellaneousController)
pcall(function()
	local _stbl
	_stbl = hookfunction(getrenv().setmetatable, newcclosure(function(tbl, mt)
		if mt and typeof(mt) == "table" and rawget(mt, "__mode") == "kv" then
			local tr = debug.traceback()
			if tr:find("MiscellaneousController") or tr:find("LocalScript3") then
				return _stbl({1, 2, 3}, {})
			end
		end
		return _stbl(tbl, mt)
	end))
end)

-- disable anticheat-named scripts + network client junk
task.spawn(function()
	pcall(function()
		local tags = {"anticheat", "ac", "detection", "ban", "kick", "security", "moderation"}
		local function proc(o)
			pcall(function()
				if o:IsA("LocalScript") or o:IsA("ModuleScript") then
					local ok, nm = pcall(function() return string.lower(o.Name) end)
					if not ok or not nm then return end
					for i = 1, #tags do
						if string.find(nm, tags[i], 1, true) then
							pcall(function() o.Disabled = true end)
							break
						end
					end
				end
			end)
		end
		pcall(function()
			local desc = game:GetDescendants()
			for i = 1, #desc do
				if i % 400 == 0 then task.wait() end
				proc(desc[i])
			end
		end)
		pcall(function()
			game.DescendantAdded:Connect(proc)
		end)
	end)
	pcall(function()
		local nc = game:GetService("NetworkClient")
		if not nc then return end
		nc.ChildAdded:Connect(function(ch)
			pcall(function()
				local ok, n = pcall(function() return string.lower(ch.Name) end)
				if ok and n then
					if string.find(n, "anticheat") or string.find(n, "detection") then
						pcall(function() ch:Destroy() end)
					end
				end
			end)
		end)
	end)
end)

-- ClientAlert fake + kick block
pcall(function()
	local fake = Instance.new("RemoteEvent")
	fake.Name = "ClientAlert"
	fake.Parent = LP

	local pmt = getrawmetatable(LP)
	if pmt then
		local oldnc = pmt.__namecall
		setreadonly(pmt, false)
		pmt.__namecall = newcclosure(function(self, ...)
			if getnamecallmethod() == "WaitForChild" and select(1, ...) == "ClientAlert" then
				return fake
			end
			return oldnc(self, ...)
		end)
		setreadonly(pmt, true)
	end

	local mt = getrawmetatable(game)
	if mt then
		local old = mt.__namecall
		setreadonly(mt, false)
		mt.__namecall = newcclosure(function(self, ...)
			local m = getnamecallmethod()
			if self == LP and (m == "Kick" or m == "kick") then
				return
			end
			if type(m) == "string" then
				local ml = string.lower(m)
				if string.find(ml, "kick") or m == "Shutdown" then
					return
				end
			end
			if m == "FireServer" and self == fake then
				return
			end
			return old(self, ...)
		end)
		setreadonly(mt, true)
	end
	print("bypass started")
end)

-- neuter LocalScript3 / LoadingScreen ban functions (Lunara getgc)
task.spawn(function()
	task.wait(0.5)
	pcall(function()
		local rf = game:GetService("ReplicatedFirst")
		local tgt = rf:FindFirstChild("LocalScript3") or rf:WaitForChild("LocalScript3", 10)
		local ct = 0
		local gc = getgc(false)
		for i = 1, #gc do
			if i % 300 == 0 then task.wait() end
			local fn = gc[i]
			if type(fn) == "function" then
				local ok1, env = pcall(getfenv, fn)
				if ok1 and type(env) == "table" then
					local ok2, scr = pcall(function()
						return rawget(env, "script")
					end)
					if ok2 and scr and typeof(scr) == "Instance" then
						local ok3, ss = pcall(tostring, scr)
						if ok3 and (scr == tgt or (type(ss) == "string" and string.find(ss, "LoadingScreen"))) then
							local ok4, consts = pcall(debug.getconstants, fn)
							if ok4 and type(consts) == "table" then
								for j = 1, #consts do
									local c = consts[j]
									if type(c) == "string" and (string.find(c, "TakeTheL") or string.find(c, "ban") or string.find(c, "kick")) then
										pcall(function()
											hookfunction(fn, function() end)
											ct = ct + 1
										end)
										break
									end
								end
							end
						end
					end
				end
			end
		end
		print("[hub] AC neutered:", ct)
	end)
end)

-- ========== UI FIRST (always) ==========
local function makeToggle(parent, text, get, set)
	local row = Instance.new("TextButton")
	row.Size = UDim2.new(1, -8, 0, 28)
	row.BackgroundColor3 = Color3.fromRGB(26, 26, 32)
	row.Text = ""
	row.AutoButtonColor = true
	row.Parent = parent
	Instance.new("UICorner", row).CornerRadius = UDim.new(0, 4)

	local label = Instance.new("TextLabel")
	label.Size = UDim2.new(1, -50, 1, 0)
	label.Position = UDim2.new(0, 8, 0, 0)
	label.BackgroundTransparency = 1
	label.Text = text
	label.TextColor3 = Color3.fromRGB(220, 220, 230)
	label.Font = Enum.Font.Gotham
	label.TextSize = 13
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.Parent = row

	local state = Instance.new("TextLabel")
	state.Size = UDim2.new(0, 40, 1, 0)
	state.Position = UDim2.new(1, -44, 0, 0)
	state.BackgroundTransparency = 1
	state.Font = Enum.Font.GothamBold
	state.TextSize = 12
	state.Parent = row

	local function refresh()
		local on = get()
		state.Text = on and "ON" or "OFF"
		state.TextColor3 = on and Color3.fromRGB(80, 255, 140) or Color3.fromRGB(120, 120, 130)
	end
	refresh()
	row.MouseButton1Click:Connect(function()
		set(not get())
		refresh()
	end)
	return refresh
end

local function makeInput(parent, text, get, set, minV, maxV)
	local row = Instance.new("Frame")
	row.Size = UDim2.new(1, -8, 0, 28)
	row.BackgroundTransparency = 1
	row.Parent = parent

	local label = Instance.new("TextLabel")
	label.Size = UDim2.new(0.45, 0, 1, 0)
	label.BackgroundTransparency = 1
	label.Text = text
	label.TextColor3 = Color3.fromRGB(160, 160, 170)
	label.Font = Enum.Font.Gotham
	label.TextSize = 12
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.Parent = row

	local box = Instance.new("TextBox")
	box.Size = UDim2.new(0.5, 0, 1, 0)
	box.Position = UDim2.new(0.48, 0, 0, 0)
	box.BackgroundColor3 = Color3.fromRGB(28, 28, 36)
	box.Text = tostring(get())
	box.TextColor3 = Color3.new(1, 1, 1)
	box.Font = Enum.Font.GothamBold
	box.TextSize = 12
	box.ClearTextOnFocus = false
	box.Parent = row
	Instance.new("UICorner", box).CornerRadius = UDim.new(0, 4)
	box.FocusLost:Connect(function()
		local v = tonumber(box.Text)
		if v then
			v = math.clamp(math.floor(v), minV, maxV)
			set(v)
			box.Text = tostring(v)
		else
			box.Text = tostring(get())
		end
	end)
end

local function section(parent, title)
	local f = Instance.new("Frame")
	f.Size = UDim2.new(1, 0, 0, 0)
	f.AutomaticSize = Enum.AutomaticSize.Y
	f.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
	f.Parent = parent
	Instance.new("UICorner", f).CornerRadius = UDim.new(0, 5)
	local st = Instance.new("UIStroke")
	st.Color = Color3.fromRGB(45, 45, 55)
	st.Parent = f

	local h = Instance.new("TextLabel")
	h.Size = UDim2.new(1, 0, 0, 22)
	h.BackgroundTransparency = 1
	h.Text = "  " .. title
	h.TextColor3 = Color3.fromRGB(180, 180, 200)
	h.Font = Enum.Font.GothamBold
	h.TextSize = 12
	h.TextXAlignment = Enum.TextXAlignment.Left
	h.Parent = f

	local body = Instance.new("Frame")
	body.Size = UDim2.new(1, -6, 0, 0)
	body.Position = UDim2.new(0, 3, 0, 22)
	body.AutomaticSize = Enum.AutomaticSize.Y
	body.BackgroundTransparency = 1
	body.Parent = f
	local lay = Instance.new("UIListLayout")
	lay.Padding = UDim.new(0, 4)
	lay.Parent = body
	local pad = Instance.new("UIPadding")
	pad.PaddingBottom = UDim.new(0, 6)
	pad.Parent = f
	return body
end

local function buildUI()
	local pg = LP:WaitForChild("PlayerGui", 15)
	if not pg then
		warn("[hub] no PlayerGui")
		return
	end
	for _, n in ipairs({"HubRev", "HubRevBoot"}) do
		local o = pg:FindFirstChild(n)
		if o then o:Destroy() end
	end

	local sg = Instance.new("ScreenGui")
	sg.Name = "HubRev"
	sg.ResetOnSpawn = false
	sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	sg.DisplayOrder = 999
	sg.Parent = pg

	-- Toggle button (draggable)
	local toggle = Instance.new("TextButton")
	toggle.Name = "ToggleBtn"
	toggle.Size = UDim2.new(0, 100, 0, 32)
	toggle.Position = UDim2.new(0, 12, 0, 60)
	toggle.BackgroundTransparency = 0
	toggle.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
	toggle.Text = "Toggle UI"
	toggle.TextColor3 = Color3.fromRGB(255, 255, 255)
	toggle.Font = Enum.Font.Code
	toggle.TextSize = 13
	toggle.TextStrokeTransparency = 1
	toggle.BorderSizePixel = 1
	toggle.BorderColor3 = Color3.fromRGB(140, 100, 255)
	toggle.ZIndex = 10
	toggle.Parent = sg
	Instance.new("UICorner", toggle).CornerRadius = UDim.new(0, 0)

	local dragging, d0, p0 = false, nil, nil
	toggle.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			d0 = input.Position
			p0 = toggle.Position
			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
				end
			end)
		end
	end)
	UIS.InputChanged:Connect(function(input)
		if not dragging then return end
		if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
			local d = input.Position - d0
			toggle.Position = UDim2.new(p0.X.Scale, p0.X.Offset + d.X, p0.Y.Scale, p0.Y.Offset + d.Y)
		end
	end)

	-- Main panel
	local main = Instance.new("Frame")
	main.Name = "Main"
	main.Size = UDim2.new(0, 580, 0, 400)
	main.Position = UDim2.new(0.5, -290, 0.5, -200)
	main.BackgroundColor3 = Color3.fromRGB(14, 14, 18)
	main.Active = true
	main.Draggable = true
	main.Visible = true
	main.ZIndex = 5
	main.Parent = sg
	Instance.new("UICorner", main).CornerRadius = UDim.new(0, 0)
	local ms = Instance.new("UIStroke")
	ms.Color = Color3.fromRGB(140, 100, 255)
	ms.Thickness = 1.5
	ms.Transparency = 0.35
	ms.Parent = main

	local title = Instance.new("TextLabel")
	title.Size = UDim2.new(1, 0, 0, 32)
	title.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
	title.Text = "  CuteGirl Hub"
	title.TextColor3 = Color3.fromRGB(255, 255, 255)
	title.Font = Enum.Font.Code
	title.TextSize = 14
	title.TextXAlignment = Enum.TextXAlignment.Left
	title.Parent = main
	Instance.new("UICorner", title).CornerRadius = UDim.new(0, 0)

	-- Tabs
	local tabBar = Instance.new("Frame")
	tabBar.Size = UDim2.new(1, -12, 0, 28)
	tabBar.Position = UDim2.new(0, 6, 0, 32)
	tabBar.BackgroundTransparency = 1
	tabBar.Parent = main
	local tLay = Instance.new("UIListLayout")
	tLay.FillDirection = Enum.FillDirection.Horizontal
	tLay.Padding = UDim.new(0, 4)
	tLay.Parent = tabBar

	local pages = {}
	local tabNames = {"Combat", "Character", "Visuals", "Misc"}

	for _, name in ipairs(tabNames) do
		local page = Instance.new("Frame")
		page.Name = name
		page.Size = UDim2.new(1, -12, 1, -68)
		page.Position = UDim2.new(0, 6, 0, 64)
		page.BackgroundTransparency = 1
		page.Visible = false
		page.Parent = main

		local left = Instance.new("ScrollingFrame")
		left.Name = "Left"
		left.Size = UDim2.new(0.5, -4, 1, 0)
		left.BackgroundTransparency = 1
		left.BorderSizePixel = 0
		left.ScrollBarThickness = 3
		left.CanvasSize = UDim2.new(0, 0, 0, 0)
		left.AutomaticCanvasSize = Enum.AutomaticSize.Y
		left.Parent = page
		Instance.new("UIListLayout", left).Padding = UDim.new(0, 6)

		local right = left:Clone()
		right.Name = "Right"
		right.Position = UDim2.new(0.5, 4, 0, 0)
		right.Parent = page

		pages[name] = page
	end

	local function showTab(name)
		for n, page in pairs(pages) do
			page.Visible = (n == name)
		end
		for _, child in ipairs(tabBar:GetChildren()) do
			if child:IsA("TextButton") then
				child.BackgroundColor3 = (child.Name == name)
					and Color3.fromRGB(50, 40, 75)
					or Color3.fromRGB(18, 18, 24)
				if child.Name == name then
					child.TextColor3 = Color3.fromRGB(255, 255, 255)
				else
					child.TextColor3 = Color3.fromRGB(160, 160, 160)
				end
			end
		end
	end

	for _, name in ipairs(tabNames) do
		local btn = Instance.new("TextButton")
		btn.Name = name
		btn.Size = UDim2.new(0, 88, 1, 0)
		btn.BackgroundColor3 = Color3.fromRGB(22, 22, 28)
		btn.Text = name
		btn.TextColor3 = Color3.fromRGB(200, 200, 200)
		btn.Font = Enum.Font.Code
		btn.TextSize = 12
		btn.BorderSizePixel = 0
		btn.Parent = tabBar
		Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 0)
		btn.MouseButton1Click:Connect(function()
			showTab(name)
		end)
	end

	-- Combat
	local cL, cR = pages.Combat.Left, pages.Combat.Right
	local aimBody = section(cL, "Aimbot")
	makeToggle(aimBody, "Enable (head lock)", function() return S.Aimbot end, function(v) S.Aimbot = v end)
	makeToggle(aimBody, "Silent Aim", function() return S.SilentAim end, function(v) S.SilentAim = v end)
	makeToggle(aimBody, "Team Check", function() return S.TeamCheck end, function(v) S.TeamCheck = v end)
	makeToggle(aimBody, "Wall Check", function() return S.WallCheck end, function(v) S.WallCheck = v end)
	makeInput(aimBody, "Aim FOV", function() return S.AimFOV end, function(v) S.AimFOV = v end, 50, 800)
	makeToggle(aimBody, "Show FOV", function() return S.ShowFOV end, function(v) S.ShowFOV = v end)
	local trigBody = section(cL, "Triggerbot")
	makeToggle(trigBody, "Enable (auto on target)", function() return S.Triggerbot end, function(v) S.Triggerbot = v end)
	makeInput(trigBody, "Trig FOV", function() return S.TrigFOV end, function(v) S.TrigFOV = v end, 20, 400)
	local gunBody = section(cL, "Gun Mods")
	makeToggle(gunBody, "Rapid Fire", function() return S.RapidFire end, function(v) S.RapidFire = v end)
	makeToggle(gunBody, "No Recoil (no kick)", function() return S.NoRecoil end, function(v) S.NoRecoil = v end)
	makeToggle(gunBody, "No Spread (laser)", function() return S.NoSpread end, function(v) S.NoSpread = v end)
	local rageBody = section(cR, "Ragebot")
	makeToggle(rageBody, "Enable", function() return S.Rage end, function(v) S.Rage = v end)
	local voidBody = section(cR, "Void Spam")
	makeToggle(voidBody, "Far Void (safe TP)", function() return S.VoidSpam end, function(v) S.VoidSpam = v end)
	local priBody = section(cR, "Target")
	-- cycle priority button
	local priBtn = Instance.new("TextButton")
	priBtn.Size = UDim2.new(1, -8, 0, 28)
	priBtn.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
	priBtn.Text = "Priority: " .. tostring(S.TargetPriority)
	priBtn.TextColor3 = Color3.fromRGB(210, 210, 210)
	priBtn.Font = Enum.Font.Code
	priBtn.TextSize = 12
	priBtn.Parent = priBody
	Instance.new("UICorner", priBtn).CornerRadius = UDim.new(0, 0)
	local priList = {"Closest", "Crosshair", "LowHP"}
	priBtn.MouseButton1Click:Connect(function()
		local idx = 1
		for j = 1, #priList do
			if priList[j] == S.TargetPriority then idx = j break end
		end
		idx = idx % #priList + 1
		S.TargetPriority = priList[idx]
		priBtn.Text = "Priority: " .. S.TargetPriority
	end)

	-- Character
	local chL, chR = pages.Character.Left, pages.Character.Right
	local prof = section(chL, "Profile")
	makeToggle(prof, "Level Spoof", function() return S.LevelSpoof end, function(v) S.LevelSpoof = v end)
	makeInput(prof, "Level", function() return S.Level end, function(v) S.Level = v end, 1, 99999)
	makeToggle(prof, "Win Streak", function() return S.WinStreakSpoof end, function(v) S.WinStreakSpoof = v end)
	makeInput(prof, "Streak", function() return S.WinStreak end, function(v) S.WinStreak = v end, 0, 99999)
	local mov = section(chR, "Movement")
	makeToggle(mov, "Fly", function() return S.Fly end, function(v)
		S.Fly = v
		if not v then
			pcall(function()
				local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
				if hum then hum.PlatformStand = false end
				local hrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
				if hrp then
					local bv = hrp:FindFirstChild("HubFly")
					if bv then bv:Destroy() end
				end
			end)
		end
	end)
	makeInput(mov, "Fly speed", function() return S.FlySpeed end, function(v) S.FlySpeed = v end, 10, 200)
	makeToggle(mov, "Noclip", function() return S.Noclip end, function(v) S.Noclip = v end)
	makeToggle(mov, "Walk Speed (Velocity)", function() return S.WalkSpeed end, function(v) S.WalkSpeed = v end)
	makeInput(mov, "Speed value", function() return S.WalkSpeedValue end, function(v) S.WalkSpeedValue = v end, 1, 200)
	makeToggle(mov, "Freecam", function() return S.Freecam end, function(v)
		S.Freecam = v
		if not v then pcall(stopFreecam) end
		if v then pcall(startFreecam) end
	end)
	makeInput(mov, "Freecam speed", function() return S.FreecamSpeed end, function(v) S.FreecamSpeed = v end, 10, 200)
	
	-- Visuals
	local vis = section(pages.Visuals.Left, "ESP")
	makeToggle(vis, "Enable", function() return S.ESP end, function(v) S.ESP = v end)
	makeToggle(vis, "Name", function() return S.ESP_Name end, function(v) S.ESP_Name = v end)
	makeToggle(vis, "Health Bar", function() return S.ESP_HealthBar end, function(v) S.ESP_HealthBar = v end)
	makeToggle(vis, "Distance", function() return S.ESP_Distance end, function(v) S.ESP_Distance = v end)
	makeToggle(vis, "Box", function() return S.ESP_Box end, function(v) S.ESP_Box = v end)
	makeToggle(vis, "Skeleton", function() return S.ESP_Skeleton end, function(v) S.ESP_Skeleton = v end)

	-- Misc
	local dev = section(pages.Misc.Left, "Device Spoof")
	makeToggle(dev, "Enable", function() return S.DeviceSpoof end, function(v)
		S.DeviceSpoof = v
		if v then
			applyDeviceSpoof(true)
			notify("Device ON: " .. tostring(S.DeviceName))
		else
			pcall(restoreRealDevice)
			notify("Device OFF (restored)")
		end
	end)

	local curDev = Instance.new("TextLabel")
	curDev.Name = "CurDev"
	curDev.Size = UDim2.new(1, -8, 0, 20)
	curDev.BackgroundTransparency = 1
	curDev.Text = "  Now: " .. tostring(S.DeviceName)
	curDev.TextColor3 = Color3.fromRGB(190, 190, 210)
	curDev.Font = Enum.Font.GothamBold
	curDev.TextSize = 12
	curDev.TextXAlignment = Enum.TextXAlignment.Left
	curDev.Parent = dev

	-- Lunara codes: Touch / Gamepad / VR / MouseKeyboard
	local devices = {
		{"PC", "PC"},
		{"Console", "Console"},
		{"Mobile", "Mobile"},
		{"VR", "VR"},
	}
	for _, pair in ipairs(devices) do
		local label, value = pair[1], pair[2]
		local b = Instance.new("TextButton")
		b.Size = UDim2.new(1, -8, 0, 26)
		b.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
		b.Text = label
		b.TextColor3 = Color3.fromRGB(210, 210, 210)
		b.Font = Enum.Font.Code
		b.TextSize = 12
		b.BorderSizePixel = 0
		b.Parent = dev
		Instance.new("UICorner", b).CornerRadius = UDim.new(0, 0)
		b.MouseButton1Click:Connect(function()
			S.DeviceName = value
			curDev.Text = "  Now: " .. value
			if S.DeviceSpoof then
				applyDeviceSpoof(true)
				notify("Device: " .. value)
			else
				notify("Device set: " .. value .. " (Enable OFF)")
			end
		end)
	end

	local unlockBody = section(pages.Misc.Right, "Unlock")
	makeToggle(unlockBody, "Unlock All (Lunara)", function() return S.UnlockAll end, function(v)
		S.UnlockAll = v
		if v then
			runLunaraUnlockAll()
		end
	end)
	local cfgBody = section(pages.Misc.Right, "Config")
	local nameBox = Instance.new("TextBox")
	nameBox.Size = UDim2.new(1, -8, 0, 28)
	nameBox.BackgroundColor3 = Color3.fromRGB(10, 10, 14)
	nameBox.Text = "default"
	nameBox.PlaceholderText = "config name"
	nameBox.TextColor3 = Color3.fromRGB(255, 255, 255)
	nameBox.Font = Enum.Font.Code
	nameBox.TextSize = 12
	nameBox.ClearTextOnFocus = false
	nameBox.Parent = cfgBody
	nameBox.BorderSizePixel = 1
	nameBox.BorderColor3 = Color3.fromRGB(55, 55, 70)
	local function cfgBtn(text, fn)
		local b = Instance.new("TextButton")
		b.Size = UDim2.new(1, -8, 0, 26)
		b.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
		b.Text = text
		b.TextColor3 = Color3.fromRGB(210, 210, 210)
		b.Font = Enum.Font.Code
		b.TextSize = 12
		b.Parent = cfgBody
		b.MouseButton1Click:Connect(function()
			fn(nameBox.Text)
		end)
		return b
	end
	cfgBtn("Create / Save", function(n) saveConfig(n) end)
	cfgBtn("Load", function(n) loadConfig(n) end)
	cfgBtn("List configs", function()
		listConfigs()
	end)

	local note = section(pages.Misc.Right, "Info")
	local info = Instance.new("TextLabel")
	info.Size = UDim2.new(1, -8, 0, 40)
	info.BackgroundTransparency = 1
	info.Text = "AC always on\nVoid = Combat toggle\nFly = Lunara"
	info.TextColor3 = Color3.fromRGB(140, 140, 150)
	info.Font = Enum.Font.Gotham
	info.TextSize = 11
	info.TextXAlignment = Enum.TextXAlignment.Left
	info.Parent = note

	showTab("Combat")

	local open = true
	toggle.MouseButton1Click:Connect(function()
		-- ignore if was dragging far
		open = not open
		main.Visible = open
	end)

	UIS.InputBegan:Connect(function(input, gp)
		if gp then return end
		if input.KeyCode == Enum.KeyCode.RightShift then
			open = not open
			main.Visible = open
		end
	end)

	print("[hub] UI created")
	notify("UI ready")
end

-- Build UI immediately
local uiOk, uiErr = pcall(buildUI)
if not uiOk then
	warn("[hub] UI FAIL:", uiErr)
	notify("UI FAIL: " .. tostring(uiErr))
end

-- ========== Modules ==========
local function sreq(inst)
	if not inst then return nil end
	local ok, m = pcall(require, inst)
	if ok then return m end
	return nil
end

local Utility, EnumLibrary, FC, GunModule, UseItem

local function refreshMods()
	local mod = RS:FindFirstChild("Modules")
	Utility = Utility or sreq(mod and mod:FindFirstChild("Utility"))
	EnumLibrary = EnumLibrary or sreq(mod and mod:FindFirstChild("EnumLibrary"))
	local ps = LP:FindFirstChild("PlayerScripts")
	local ctr = ps and ps:FindFirstChild("Controllers")
	FC = FC or sreq(ctr and ctr:FindFirstChild("FighterController"))
	local it = ps and ps:FindFirstChild("Modules")
	it = it and it:FindFirstChild("ItemTypes")
	GunModule = GunModule or sreq(it and it:FindFirstChild("Gun"))
	local r = RS:FindFirstChild("Remotes")
	r = r and r:FindFirstChild("Replication")
	r = r and r:FindFirstChild("Fighter")
	UseItem = UseItem or (r and r:FindFirstChild("UseItem"))
end
refreshMods()
task.spawn(function()
	for _ = 1, 20 do
		task.wait(0.5)
		refreshMods()
		if Utility and EnumLibrary and UseItem then break end
	end
	print("[hub] mods U", Utility ~= nil, "E", EnumLibrary ~= nil, "FC", FC ~= nil, "G", GunModule ~= nil, "UI", UseItem ~= nil)
end)

local controls
pcall(function()
	controls = require(LP.PlayerScripts:WaitForChild("PlayerModule", 5)):GetControls()
end)

local function getFighter()
	refreshMods()
	if not FC then return nil end
	if FC.LocalFighter then return FC.LocalFighter end
	if type(FC.GetFighter) == "function" then
		local ok, f = pcall(FC.GetFighter, FC, LP)
		if ok then return f end
	end
	return nil
end

local function getRootPart()
	local lf = getFighter()
	if lf and lf.Entity and lf.Entity.RootPart then
		return lf.Entity.RootPart
	end
	return LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
end

local function getMoveWorld()
	local cf = Cam.CFrame
	local look = Vector3.new(cf.LookVector.X, 0, cf.LookVector.Z)
	local right = Vector3.new(cf.RightVector.X, 0, cf.RightVector.Z)
	if look.Magnitude > 0 then look = look.Unit end
	if right.Magnitude > 0 then right = right.Unit end

	if controls and controls.GetMoveVector then
		local ok, mv = pcall(function() return controls:GetMoveVector() end)
		if ok and typeof(mv) == "Vector3" and mv.Magnitude > 0.05 then
			local dir = right * mv.X + look * mv.Z
			if dir.Magnitude > 0.05 then return dir.Unit end
		end
	end

	local m = Vector3.zero
	if UIS:IsKeyDown(Enum.KeyCode.W) then m = m + look end
	if UIS:IsKeyDown(Enum.KeyCode.S) then m = m - look end
	if UIS:IsKeyDown(Enum.KeyCode.A) then m = m - right end
	if UIS:IsKeyDown(Enum.KeyCode.D) then m = m + right end
	if m.Magnitude > 0 then return m.Unit end

	local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
	if hum and hum.MoveDirection.Magnitude > 0.05 then
		local md = hum.MoveDirection
		local flat = Vector3.new(md.X, 0, md.Z)
		if flat.Magnitude > 0 then return flat.Unit end
	end
	return Vector3.zero
end

local function validEnemy(plr)
	if plr == LP then return false end
	local char = plr.Character
	if not char then return false end
	local hum = char:FindFirstChildOfClass("Humanoid")
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if not hum or not hrp or hum.Health <= 0 then return false end
	if hrp:FindFirstChild("Attachment") then return false end
	if S.TeamCheck then
		local a, b = LP:GetAttribute("TeamID"), plr:GetAttribute("TeamID")
		if a and b and a == b then return false end
	end
	return true
end


local function hasLOS(fromPos, toPos, targetChar)
	if not S.WallCheck then return true end
	local origin = fromPos
	local dir = toPos - origin
	local dist = dir.Magnitude
	if dist < 1 then return true end
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	local filter = { LP.Character }
	if targetChar then table.insert(filter, targetChar) end
	params.FilterDescendantsInstances = filter
	params.IgnoreWater = true
	local result = Workspace:Raycast(origin, dir.Unit * dist, params)
	if not result then return true end
	-- hit something that is not the target
	return false
end

local function getHead(plr)
	local c = plr.Character
	if not c then return nil end
	return c:FindFirstChild("HitboxHead") or c:FindFirstChild("Head")
end

local function getNearest(ignoreWall)
	local root = getRootPart()
	if not root then return nil end
	local best, bd = nil, math.huge
	local from = root.Position
	for _, plr in ipairs(Players:GetPlayers()) do
		if validEnemy(plr) then
			local head = getHead(plr)
			local hrp = plr.Character and plr.Character:FindFirstChild("HumanoidRootPart")
			if head and hrp then
				local okWall = ignoreWall or hasLOS(from, head.Position, plr.Character)
				if okWall then
					local d = (from - hrp.Position).Magnitude
					if d < bd then
						bd = d
						best = head
					end
				end
			end
		end
	end
	return best
end

local function getCrosshair(fov)
	local best, bd = nil, fov or 120
	local center = Cam.ViewportSize / 2
	local root = getRootPart()
	local from = root and root.Position or Cam.CFrame.Position
	for _, plr in ipairs(Players:GetPlayers()) do
		if validEnemy(plr) then
			local head = getHead(plr)
			if head then
				local sp, on = Cam:WorldToViewportPoint(head.Position)
				if on and sp.Z > 0 then
					local dist = (Vector2.new(sp.X, sp.Y) - center).Magnitude
					if dist < bd and hasLOS(from, head.Position, plr.Character) then
						bd = dist
						best = head
					end
				end
			end
		end
	end
	return best
end


local function getTarget(ignoreWall, fov)
	local mode = S.TargetPriority or "Closest"
	if mode == "Crosshair" then
		return getCrosshair(fov or S.AimFOV or 300) or getNearest(ignoreWall)
	elseif mode == "LowHP" then
		local root = getRootPart()
		if not root then return nil end
		local best, bestHp = nil, math.huge
		local from = root.Position
		for _, plr in ipairs(Players:GetPlayers()) do
			if validEnemy(plr) then
				local head = getHead(plr)
				local hum = plr.Character and plr.Character:FindFirstChildOfClass("Humanoid")
				if head and hum then
					local okWall = ignoreWall or hasLOS(from, head.Position, plr.Character)
					if okWall and hum.Health < bestHp then
						bestHp = hum.Health
						best = head
					end
				end
			end
		end
		return best
	end
	return getNearest(ignoreWall)
end

local function encodeShot(from, to, head)
	if not Utility then return nil end
	local e = Utility:EncodeCFrame(CFrame.new(from, to))
	return {
		[utf8.char(1)] = {
			[utf8.char(0)] = e,
			[utf8.char(1)] = e,
			[utf8.char(2)] = head,
			[utf8.char(3)] = Utility:EncodeCFrame(CFrame.new(0.43, 0.25, 0.42)),
		},
	}
end

-- ========== RAGE / VOID (stable, don't touch when editing UI) ==========
local csync = { active = false, realCF = nil, serverPos = nil }

-- Visual NEVER stays at server shot pos. RenderStep always restores client CF.
pcall(function()
	RunService:BindToRenderStep("hub_csync", Enum.RenderPriority.Camera.Value - 1, function()
		if not csync.realCF then return end
		local root = getRootPart()
		if not root then return end
		-- always keep what you see at realCF while rage/csync running
		if csync.active or S.Rage then
			pcall(function()
				root.CFrame = csync.realCF
			end)
		end
	end)
end)

local function csyncCaptureVisual()
	local root = getRootPart()
	if root then
		csync.realCF = root.CFrame
	end
end

local function csyncBegin(pos)
	local root = getRootPart()
	if not root then return end
	-- capture current visual BEFORE any snap (do not overwrite with under-target)
	if not csync.realCF then
		csync.realCF = root.CFrame
	end
	csync.serverPos = pos
	csync.active = true
	-- brief server-side snap only (visual restored same frame by RenderStep)
	pcall(function()
		root.CFrame = CFrame.new(pos)
		root.AssemblyLinearVelocity = Vector3.zero
	end)
end

local function csyncEnd()
	local root = getRootPart()
	if root and csync.realCF then
		pcall(function()
			root.CFrame = csync.realCF
			root.AssemblyLinearVelocity = Vector3.zero
		end)
	end
	csync.active = false
	csync.serverPos = nil
	-- keep realCF while Rage on so next shot still has visual anchor
	if not S.Rage then
		csync.realCF = nil
	end
end

local function rageFire(head)
	if not head then return false end
	refreshMods()
	if not (Utility and EnumLibrary and UseItem) then return false end

	local lf = getFighter()
	if not lf or not lf.EquippedItem then return false end

	local ok, oid = pcall(function()
		return lf.EquippedItem:Get("ObjectID")
	end)
	local okE, se = pcall(function()
		return EnumLibrary:ToEnum("StartShooting")
	end)
	if not (ok and oid and okE and se) then return false end

	local pos = head.Position
	local root = getRootPart()
	local from = root and root.Position or Cam.CFrame.Position

	-- under-target for SERVER shot only â€” screen stays put
	csyncCaptureVisual()
	local under = pos + Vector3.new(0, -5, 0)
	csyncBegin(under)
	from = under

	local data = encodeShot(from, pos, head)
	if not data then
		csyncEnd()
		return false
	end

	local fired = false
	for _ = 1, 2 do
		local okF = pcall(function()
			UseItem:FireServer(oid, se, data, nil)
		end)
		if okF then fired = true end
	end

	task.delay(0.05, function()
		csyncEnd()
	end)
	return fired
end

-- Silent aim / trigger redirect on real shots
task.spawn(function()
	for _ = 1, 15 do
		refreshMods()
		if GunModule and GunModule.StartShooting then break end
		task.wait(0.4)
	end
	if not (GunModule and GunModule.StartShooting) then
		warn("[hub] gun hook failed")
		return
	end
	local old = GunModule.StartShooting
	GunModule.StartShooting = function(self, ...)
		local a, b, c, d, e = old(self, ...)
		if not S.Rage and not S.Triggerbot and not S.SilentAim then
			return a, b, c, d, e
		end
		local isLocal = false
		pcall(function()
			local cf = self.ClientFighter
			if cf then
				if cf.IsLocalPlayer then
					isLocal = true
				elseif cf.Player == LP then
					isLocal = true
				end
			end
		end)
		if not isLocal or type(c) ~= "table" then
			return a, b, c, d, e
		end
		local head
		if S.Rage then
			head = getNearest(true)
		elseif S.SilentAim then
			head = getTarget(false, S.AimFOV or 300)
		else
			head = getCrosshair(S.TrigFOV or 120)
		end
		if head and Utility then
			local pos = head.Position
			local root = getRootPart()
			local from = root and root.Position or Cam.CFrame.Position
			if S.Rage then
				from = pos + Vector3.new(0, -5, 0)
			end
			pcall(function()
				local data = encodeShot(from, pos, head)
				if data then
					c[utf8.char(1)] = data[utf8.char(1)]
					c[utf8.char(0)] = data[utf8.char(1)][utf8.char(0)]
					c[utf8.char(2)] = head
				end
			end)
			d = true
		end
		return a, b, c, d, e
	end
	print("[hub] gun hooked")
end)

-- Aimbot: hard lock onto enemy HEAD (team + wall check kept)
pcall(function()
	RunService:UnbindFromRenderStep("hub_aimbot")
end)
RunService:BindToRenderStep("hub_aimbot", Enum.RenderPriority.Camera.Value + 1, function()
	if not S.Aimbot then return end
	-- wall/team via getCrosshair / getNearest (hasLOS + validEnemy)
	local head = getTarget(false, S.AimFOV or 300)
	if not head or not head.Parent then return end
	-- prefer actual Head / HitboxHead part
	local target = head
	local char = head.Parent
	if char then
		local h = char:FindFirstChild("HitboxHead") or char:FindFirstChild("Head")
		if h then target = h end
	end
	local cam = Workspace.CurrentCamera
	if not cam then return end
	-- exact snap (no lerp) â€” crosshair on head
	cam.CFrame = CFrame.lookAt(cam.CFrame.Position, target.Position)
end)


-- Silent Aim (Lunara: UseItem while shoot held, no camera move)
local lastSilent = 0
RunService.Heartbeat:Connect(function()
	if not S.SilentAim then return end
	if S.Rage then return end
	local holding = false
	pcall(function()
		holding = UIS:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
			or UIS:IsKeyDown(Enum.KeyCode.ButtonR2)
	end)
	if not holding then return end
	if tick() - lastSilent < 0.05 then return end
	local head = getTarget(false, S.AimFOV or 300)
	if not head then return end
	local char = head.Parent
	if char then
		local h = char:FindFirstChild("HitboxHead") or char:FindFirstChild("Head")
		if h then head = h end
	end
	refreshMods()
	if not (Utility and EnumLibrary and UseItem) then return end
	local lf = getFighter()
	if not lf or not lf.EquippedItem then return end
	local ok, oid = pcall(function() return lf.EquippedItem:Get("ObjectID") end)
	local okE, se = pcall(function() return EnumLibrary:ToEnum("StartShooting") end)
	if not (ok and oid and okE and se) then return end
	local root = getRootPart()
	local from = root and root.Position or Cam.CFrame.Position
	local data = encodeShot(from, head.Position, head)
	if data then
		pcall(function() UseItem:FireServer(oid, se, data, nil) end)
		lastSilent = tick()
	end
end)

-- Triggerbot: shoot when target near crosshair
local lastTrig = 0
RunService.Heartbeat:Connect(function()
	if not S.Triggerbot or S.Rage then return end
	if tick() - lastTrig < 0.1 then return end
	local lf = getFighter()
	if not lf or not lf.EquippedItem then return end
	local head = getCrosshair(math.max(S.TrigFOV or 120, 80))
	if not head then return end
	refreshMods()
	if not (Utility and EnumLibrary and UseItem) then return end
	local ok, oid = pcall(function() return lf.EquippedItem:Get("ObjectID") end)
	local okE, se = pcall(function() return EnumLibrary:ToEnum("StartShooting") end)
	if not (ok and oid and okE and se) then return end
	local root = getRootPart()
	local from = root and root.Position or Cam.CFrame.Position
	local data = encodeShot(from, head.Position, head)
	if data then
		pcall(function() UseItem:FireServer(oid, se, data, nil) end)
		lastTrig = tick()
	end
end)

-- Ragebot loop
local lastRage = 0
RunService.Heartbeat:Connect(function()
	if not S.Rage then
		if csync.active then csyncEnd() end
		csync.realCF = nil
		return
	end
	-- refresh visual anchor when not mid-shot
	if not csync.active then
		csyncCaptureVisual()
	end
	if tick() - lastRage < (S.RageInterval or 0.06) then return end
	local lf = getFighter()
	if not lf or not lf.EquippedItem then return end
	local head = getNearest(true) -- rage: no wall check
	if not head then return end
	if rageFire(head) then
		lastRage = tick()
	end
end)



-- Void Spam: continuous far safe teleport (not onto enemy)
local farVoidConn = nil
local farVoidSaved = nil
local function stopFarVoid()
	if farVoidConn then
		farVoidConn:Disconnect()
		farVoidConn = nil
	end
	local root = getRootPart()
	if root and farVoidSaved then
		pcall(function()
			root.CFrame = farVoidSaved
			root.AssemblyLinearVelocity = Vector3.zero
		end)
	end
	farVoidSaved = nil
end

local function startFarVoid()
	stopFarVoid()
	local root = getRootPart()
	if root then
		farVoidSaved = root.CFrame
	end
	farVoidConn = RunService.Heartbeat:Connect(function()
		if not S.VoidSpam then
			stopFarVoid()
			return
		end
		-- if rage is actively shooting, don't fight rage csync every frame
		if S.Rage and csync.active then
			return
		end
		local root2 = getRootPart()
		if not root2 then return end
		if not farVoidSaved then
			farVoidSaved = root2.CFrame
		end
		-- very far, high enough Y so you don't die (void kill is usually low Y)
		local p = Vector3.new(
			math.random(-25000, 25000),
			5000 + math.random(0, 2000),
			math.random(-25000, 25000)
		)
		pcall(function()
			root2.CFrame = CFrame.new(p)
			root2.AssemblyLinearVelocity = Vector3.zero
		end)
	end)
	-- visual restore each frame so you don't see void (client view)
	RunService:BindToRenderStep("hub_farvoid_vis", Enum.RenderPriority.Camera.Value, function()
		if not S.VoidSpam or not farVoidSaved then return end
		if S.Rage and csync.active then return end
		local root2 = getRootPart()
		if root2 then
			pcall(function()
				root2.CFrame = farVoidSaved
			end)
		end
	end)
end

task.spawn(function()
	local was = false
	while true do
		task.wait(0.2)
		if S.VoidSpam and not was then
			startFarVoid()
		elseif (not S.VoidSpam) and was then
			stopFarVoid()
			pcall(function()
				RunService:UnbindFromRenderStep("hub_farvoid_vis")
			end)
		end
		was = S.VoidSpam
	end
end)

-- Rapid Fire (auto + semi pistol/shotgun + melee)
local function forceZeroInfo(info)
	if type(info) ~= "table" then return end
	for k, v in pairs(info) do
		if type(v) == "number" then
			local lk = string.lower(tostring(k))
			if string.find(lk, "cool") or string.find(lk, "delay") or string.find(lk, "rate")
				or string.find(lk, "interval") or string.find(lk, "ready") or string.find(lk, "wait")
				or string.find(lk, "shoot") or string.find(lk, "attack") or string.find(lk, "swing")
				or string.find(lk, "fire") or string.find(lk, "shot") then
				info[k] = 0
			end
		end
	end
	-- explicit
	for _, k in ipairs({
		"ShootCooldown", "ShotCooldown", "FireCooldown", "FireRate", "Cooldown",
		"AttackCooldown", "SwingCooldown", "HitCooldown", "DelayBetweenShots",
		"TimeBetweenShots", "MinShootInterval", "NextShotTime",
	}) do
		if type(info[k]) == "number" then
			info[k] = 0
		end
	end
end

task.spawn(function()
	for _ = 1, 25 do
		refreshMods()
		if GunModule and GunModule.StartShooting then break end
		task.wait(0.25)
	end

	if GunModule and GunModule.StartShooting then
		local prev = GunModule.StartShooting
		GunModule.StartShooting = function(self, ...)
			if S.RapidFire then
				pcall(function()
					if self.Info then forceZeroInfo(self.Info) end
				end)
			end
			local results = { prev(self, ...) }
			-- do NOT restore cooldowns while RapidFire (semi-auto needs them stuck at 0)
			if S.RapidFire then
				pcall(function()
					if self.Info then forceZeroInfo(self.Info) end
				end)
			end
			return unpack(results)
		end
		print("[hub] rapid fire gun ok")
	end

	local MeleeModule
	pcall(function()
		MeleeModule = require(LP.PlayerScripts.Modules.ItemTypes.Melee)
	end)
	if MeleeModule and MeleeModule.StartShooting then
		local prevM = MeleeModule.StartShooting
		MeleeModule.StartShooting = function(self, ...)
			if S.RapidFire then
				pcall(function()
					if self.Info then forceZeroInfo(self.Info) end
				end)
			end
			local results = { prevM(self, ...) }
			if S.RapidFire then
				pcall(function()
					if self.Info then forceZeroInfo(self.Info) end
				end)
			end
			return unpack(results)
		end
		print("[hub] rapid melee ok")
	end
end)

-- continuous zero + semi auto re-fire while holding shoot
local lastSemiFire = 0
RunService.Heartbeat:Connect(function()
	if not S.RapidFire then return end
	local lf = getFighter()
	if not lf or not lf.EquippedItem then return end

	pcall(function()
		local item = lf.EquippedItem
		if item.Info then forceZeroInfo(item.Info) end
		for _, k in ipairs({"ShootCooldown", "AttackCooldown", "LastShoot", "LastAttack", "NextShot"}) do
			if type(item[k]) == "number" then item[k] = 0 end
		end
	end)

	-- semi-auto assist: while LMB / touch held, keep calling StartShooting path via UseItem
	local holding = false
	pcall(function()
		holding = UIS:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
			or UIS:IsKeyDown(Enum.KeyCode.ButtonR2)
	end)
	-- mobile: if not keyboard, treat as holding when any touch after short debounce is flaky;
	-- rely on cooldown zero primarily for mobile

	if holding and tick() - lastSemiFire >= 0.05 then
		refreshMods()
		if Utility and EnumLibrary and UseItem then
			local ok, oid = pcall(function() return lf.EquippedItem:Get("ObjectID") end)
			local okE, se = pcall(function() return EnumLibrary:ToEnum("StartShooting") end)
			if ok and oid and okE and se then
				local root = getRootPart()
				local from = root and root.Position or Cam.CFrame.Position
				local to = Cam.CFrame.Position + Cam.CFrame.LookVector * 200
				local head = nil
				-- aim at where you're looking; optional nearest if rage
				pcall(function()
					local data = encodeShot(from, to, workspace.CurrentCamera)
					-- better: use look direction empty hit
					if Utility then
						local e = Utility:EncodeCFrame(CFrame.new(from, to))
						data = {
							[utf8.char(1)] = {
								[utf8.char(0)] = e,
								[utf8.char(1)] = e,
								[utf8.char(2)] = nil,
								[utf8.char(3)] = Utility:EncodeCFrame(CFrame.new(0.43, 0.25, 0.42)),
							},
						}
						UseItem:FireServer(oid, se, data, nil)
						lastSemiFire = tick()
					end
				end)
			end
		end
	end
end)


-- WalkSpeed (Lunara Velocity: AssemblyLinearVelocity * moveDir)
RunService.Heartbeat:Connect(function()
	if not S.WalkSpeed then return end
	local char = LP.Character
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hrp or not hum then return end
	local moveDir = hum.MoveDirection
	if moveDir.Magnitude > 0.05 then
		local spd = S.WalkSpeedValue or 45
		hrp.AssemblyLinearVelocity = Vector3.new(
			moveDir.X * spd,
			hrp.AssemblyLinearVelocity.Y,
			moveDir.Z * spd
		)
	end
end)

-- ESP (reference style: name + thin HP + distance + thin box + skeleton)
local function getEspFolder()
	local parent = nil
	pcall(function()
		parent = gethui and gethui() or game:GetService("CoreGui")
	end)
	if not parent then
		parent = LP:FindFirstChild("PlayerGui") or LP:WaitForChild("PlayerGui", 5)
	end
	local f = parent and parent:FindFirstChild("HubESP")
	if f then return f end
	f = Instance.new("Folder")
	f.Name = "HubESP"
	f.Parent = parent
	return f
end

local function part(char, names)
	for _, n in ipairs(names) do
		local p = char:FindFirstChild(n)
		if p and p:IsA("BasePart") then return p end
	end
	return nil
end

local function skelPairs(char)
	local pairsList = {}
	local function add(aNames, bNames)
		local a = part(char, aNames)
		local b = part(char, bNames)
		if a and b then table.insert(pairsList, {a, b}) end
	end
	add({"Head"}, {"UpperTorso", "Torso"})
	add({"UpperTorso"}, {"LowerTorso"})
	add({"UpperTorso", "Torso"}, {"LeftUpperArm", "Left Arm"})
	add({"LeftUpperArm"}, {"LeftLowerArm"})
	add({"LeftLowerArm"}, {"LeftHand"})
	add({"UpperTorso", "Torso"}, {"RightUpperArm", "Right Arm"})
	add({"RightUpperArm"}, {"RightLowerArm"})
	add({"RightLowerArm"}, {"RightHand"})
	add({"LowerTorso", "Torso"}, {"LeftUpperLeg", "Left Leg"})
	add({"LeftUpperLeg"}, {"LeftLowerLeg"})
	add({"LeftLowerLeg"}, {"LeftFoot"})
	add({"LowerTorso", "Torso"}, {"RightUpperLeg", "Right Leg"})
	add({"RightUpperLeg"}, {"RightLowerLeg"})
	add({"RightLowerLeg"}, {"RightFoot"})
	add({"Torso"}, {"Left Arm"})
	add({"Torso"}, {"Right Arm"})
	add({"Torso"}, {"Left Leg"})
	add({"Torso"}, {"Right Leg"})
	return pairsList
end

task.spawn(function()
	while true do
		task.wait(0.25)
		pcall(function()
			local espFolder = getEspFolder()
			for _, c in ipairs(espFolder:GetChildren()) do
				c:Destroy()
			end
			if not S.ESP then return end
			local myRoot = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
			for _, plr in ipairs(Players:GetPlayers()) do
				if validEnemy(plr) then
					local char = plr.Character
					local hrp = char and char:FindFirstChild("HumanoidRootPart")
					local head = char and char:FindFirstChild("Head")
					local hum = char and char:FindFirstChildOfClass("Humanoid")
					if hrp and hum and hum.Health > 0 then
						local ratio = math.clamp(hum.Health / math.max(hum.MaxHealth, 1), 0, 1)
						local dist = 0
						if myRoot then
							dist = math.floor((myRoot.Position - hrp.Position).Magnitude)
						end

						-- Compact tag like reference (name + thin HP + distance)
						if S.ESP_Name or S.ESP_HealthBar or S.ESP_Distance then
							local bb = Instance.new("BillboardGui")
							bb.Name = "HubTag"
							bb.Size = UDim2.new(0, 120, 0, 36)
							bb.StudsOffset = Vector3.new(0, 2.8, 0)
							bb.AlwaysOnTop = true
							bb.Adornee = head or hrp
							bb.Parent = espFolder

							local lay = Instance.new("UIListLayout")
							lay.FillDirection = Enum.FillDirection.Vertical
							lay.HorizontalAlignment = Enum.HorizontalAlignment.Center
							lay.Padding = UDim.new(0, 1)
							lay.Parent = bb

							if S.ESP_Name then
								local name = Instance.new("TextLabel")
								name.Size = UDim2.new(1, 0, 0, 14)
								name.BackgroundTransparency = 1
								name.Text = plr.DisplayName or plr.Name
								name.TextColor3 = Color3.fromRGB(255, 255, 255)
								name.TextStrokeTransparency = 0.4
								name.Font = Enum.Font.GothamBold
								name.TextSize = 12
								name.Parent = bb
							end

							if S.ESP_HealthBar then
								-- thin horizontal bar under name (reference style)
								local barHold = Instance.new("Frame")
								barHold.Size = UDim2.new(0, 40, 0, 3)
								barHold.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
								barHold.BorderSizePixel = 0
								barHold.Parent = bb
								local fill = Instance.new("Frame")
								fill.Size = UDim2.new(ratio, 0, 1, 0)
								fill.BackgroundColor3 = Color3.fromRGB(0, 255, 100)
								fill.BorderSizePixel = 0
								fill.Parent = barHold
							end

							if S.ESP_Distance then
								local dlab = Instance.new("TextLabel")
								dlab.Size = UDim2.new(1, 0, 0, 12)
								dlab.BackgroundTransparency = 1
								dlab.Text = tostring(dist) .. "m"
								dlab.TextColor3 = Color3.fromRGB(200, 200, 210)
								dlab.TextStrokeTransparency = 0.5
								dlab.Font = Enum.Font.Gotham
								dlab.TextSize = 11
								dlab.Parent = bb
							end
						end

						-- Thin box (SelectionBox light)
						if S.ESP_Box then
							local box = Instance.new("BoxHandleAdornment")
							box.Name = "HubBox"
							box.Adornee = hrp
							box.Size = Vector3.new(2.2, 5.2, 2.2)
							box.Color3 = Color3.fromRGB(255, 255, 255)
							box.Transparency = 0.85
							box.AlwaysOnTop = true
							box.ZIndex = 5
							box.Parent = espFolder
							-- outline feel with SelectionBox thinner
							local sb = Instance.new("SelectionBox")
							sb.Adornee = char
							sb.Color3 = Color3.fromRGB(255, 255, 255)
							sb.LineThickness = 0.015
							sb.Transparency = 0.35
							sb.SurfaceTransparency = 1
							sb.Parent = espFolder
						end

						-- Skeleton fitted to body (thin beams)
						if S.ESP_Skeleton then
							for _, ab in ipairs(skelPairs(char)) do
								local a, b = ab[1], ab[2]
								local att0 = Instance.new("Attachment")
								att0.Parent = a
								local att1 = Instance.new("Attachment")
								att1.Parent = b
								local beam = Instance.new("Beam")
								beam.Attachment0 = att0
								beam.Attachment1 = att1
								beam.Width0 = 0.04
								beam.Width1 = 0.04
								beam.FaceCamera = true
								beam.LightEmission = 0.3
								beam.Color = ColorSequence.new(Color3.fromRGB(255, 255, 255))
								beam.Transparency = NumberSequence.new(0.15)
								beam.Parent = espFolder
							end
						end
					end
				end
			end
		end)
	end
end)

-- Spoofs
-- Level + WinStreak (Lunara path)
task.spawn(function()
	while true do
		task.wait(0.4)
		if S.LevelSpoof then
			pcall(function()
				LP:SetAttribute("Level", S.Level)
			end)
		end
		if S.WinStreakSpoof then
			pcall(function()
				local ls = LP:FindFirstChild("CustomLeaderstats")
				if ls then
					local ws = ls:FindFirstChild("Win Streak") or ls:FindFirstChild("WinStreak")
					if ws then
						ws.Value = S.WinStreak
					end
				end
				LP:SetAttribute("WinStreak", S.WinStreak)
			end)
		end
	end
end)

-- Noclip
RunService.Stepped:Connect(function()
	if not S.Noclip then return end
	local char = LP.Character
	if not char then return end
	for _, p in ipairs(char:GetDescendants()) do
		if p:IsA("BasePart") then
			p.CanCollide = false
		end
	end
end)

-- Fly (Lunara-style BodyVelocity + BodyGyro)
local flyConn = nil
local flyBV = nil
local flyBG = nil

local function cleanupFly()
	if flyConn then
		flyConn:Disconnect()
		flyConn = nil
	end
	if flyBV then
		pcall(function() flyBV:Destroy() end)
		flyBV = nil
	end
	if flyBG then
		pcall(function() flyBG:Destroy() end)
		flyBG = nil
	end
	local char = LP.Character
	local hrp = char and char:FindFirstChild("HumanoidRootPart")
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	if hrp then
		pcall(function()
			hrp.AssemblyLinearVelocity = Vector3.zero
		end)
	end
	if hum then
		pcall(function()
			hum.PlatformStand = false
			hum:ChangeState(Enum.HumanoidStateType.Running)
		end)
	end
end

local function startFly()
	cleanupFly()
	local char = LP.Character
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hrp or not hum then return end

	flyBV = Instance.new("BodyVelocity")
	flyBV.Name = "HubFly"
	flyBV.MaxForce = Vector3.new(1e6, 1e6, 1e6)
	flyBV.Velocity = Vector3.zero
	flyBV.Parent = hrp

	flyBG = Instance.new("BodyGyro")
	flyBG.Name = "HubFlyGyro"
	flyBG.MaxTorque = Vector3.new(1e6, 1e6, 1e6)
	flyBG.P = 1000
	flyBG.Parent = hrp

	pcall(function()
		hum:ChangeState(Enum.HumanoidStateType.PlatformStanding)
	end)

	flyConn = RunService.Heartbeat:Connect(function()
		if not S.Fly then
			cleanupFly()
			return
		end
		local char2 = LP.Character
		if not char2 or not char2.Parent then
			cleanupFly()
			return
		end
		local hrp2 = char2:FindFirstChild("HumanoidRootPart")
		local hum2 = char2:FindFirstChildOfClass("Humanoid")
		if not hrp2 or not hum2 then
			cleanupFly()
			return
		end

		-- recreate movers if lost (respawn / reset)
		if not flyBV or flyBV.Parent ~= hrp2 then
			if flyBV then pcall(function() flyBV:Destroy() end) end
			flyBV = Instance.new("BodyVelocity")
			flyBV.Name = "HubFly"
			flyBV.MaxForce = Vector3.new(1e6, 1e6, 1e6)
			flyBV.Parent = hrp2
		end
		if not flyBG or flyBG.Parent ~= hrp2 then
			if flyBG then pcall(function() flyBG:Destroy() end) end
			flyBG = Instance.new("BodyGyro")
			flyBG.Name = "HubFlyGyro"
			flyBG.MaxTorque = Vector3.new(1e6, 1e6, 1e6)
			flyBG.P = 1000
			flyBG.Parent = hrp2
		end

		local cam = Workspace.CurrentCamera
		local look = cam.CFrame.LookVector
		local right = cam.CFrame.RightVector
		local move = Vector3.zero

		-- PC (Lunara exact)
		if UIS:IsKeyDown(Enum.KeyCode.W) then move = move + look end
		if UIS:IsKeyDown(Enum.KeyCode.S) then move = move - look end
		if UIS:IsKeyDown(Enum.KeyCode.A) then move = move - right end
		if UIS:IsKeyDown(Enum.KeyCode.D) then move = move + right end
		if UIS:IsKeyDown(Enum.KeyCode.Space) then move = move + Vector3.new(0, 1, 0) end
		if UIS:IsKeyDown(Enum.KeyCode.LeftShift) or UIS:IsKeyDown(Enum.KeyCode.LeftControl) then
			move = move - Vector3.new(0, 1, 0)
		end

		-- Mobile: camera-relative stick (stick up = look forward)
		if move.Magnitude < 0.05 then
			local mv = nil
			if controls and controls.GetMoveVector then
				local ok, v = pcall(function() return controls:GetMoveVector() end)
				if ok then mv = v end
			end
			if typeof(mv) == "Vector3" and mv.Magnitude > 0.1 then
				-- standard: -Z forward in camera space for control modules
				move = (right * mv.X) + (look * -mv.Z)
			end
		end

		if move.Magnitude > 0.1 then
			flyBV.Velocity = move.Unit * S.FlySpeed
		else
			flyBV.Velocity = Vector3.zero
		end
		-- gyro follows full camera (Lunara)
		flyBG.CFrame = cam.CFrame

		if hum2:GetState() ~= Enum.HumanoidStateType.PlatformStanding then
			pcall(function()
				hum2:ChangeState(Enum.HumanoidStateType.PlatformStanding)
			end)
		end
	end)
end

-- watch S.Fly toggles (polled lightly)
task.spawn(function()
	local was = false
	while true do
		task.wait(0.15)
		if S.Fly and not was then
			startFly()
		elseif (not S.Fly) and was then
			cleanupFly()
		end
		was = S.Fly
	end
end)

LP.CharacterAdded:Connect(function()
	if S.Fly then
		task.wait(0.5)
		if S.Fly then startFly() end
	end
end)


-- FOV circle
task.spawn(function()
	local pg = LP:WaitForChild("PlayerGui", 10)
	if not pg then return end
	local sg = Instance.new("ScreenGui")
	sg.Name = "HubFOV"
	sg.ResetOnSpawn = false
	sg.IgnoreGuiInset = true
	sg.Parent = pg
	local ring = Instance.new("Frame")
	ring.AnchorPoint = Vector2.new(0.5, 0.5)
	ring.Position = UDim2.fromScale(0.5, 0.5)
	ring.BackgroundTransparency = 1
	ring.Parent = sg
	local stroke = Instance.new("UIStroke")
	stroke.Color = Color3.fromRGB(255, 255, 255)
	stroke.Thickness = 1
	stroke.Transparency = 0.3
	stroke.Parent = ring
	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(1, 0)
	corner.Parent = ring
	RunService.RenderStepped:Connect(function()
		if not S.ShowFOV then
			ring.Visible = false
			return
		end
		ring.Visible = true
		local fov = S.Aimbot and (S.AimFOV or 300) or (S.TrigFOV or 120)
		ring.Size = UDim2.fromOffset(fov * 2, fov * 2)
	end)
end)

-- No Recoil / No Spread (Lunara style)
task.spawn(function()
	for _ = 1, 20 do
		refreshMods()
		if GunModule then break end
		task.wait(0.3)
	end
	if GunModule and GunModule._Recoil then
		local oldR = GunModule._Recoil
		GunModule._Recoil = function(self, mult)
			if S.NoRecoil then return end
			return oldR(self, mult)
		end
		print("[hub] no recoil hooked")
	end
	pcall(function()
		local GU = require(RS.Modules.GameplayUtility)
		if GU and GU.GetSpread then
			local oldS = GU.GetSpread
			GU.GetSpread = function(...)
				if S.NoSpread then
					return CFrame.new()
				end
				return oldS(...)
			end
			print("[hub] no spread hooked")
		end
	end)
end)

-- Freecam
local freecam = { cam = nil, conn = nil, pos = nil }
function stopFreecam()
	if freecam.conn then freecam.conn:Disconnect(); freecam.conn = nil end
	pcall(function()
		local cam = Workspace.CurrentCamera
		if cam then cam.CameraType = Enum.CameraType.Custom end
	end)
	freecam.pos = nil
end
function startFreecam()
	stopFreecam()
	local cam = Workspace.CurrentCamera
	freecam.pos = cam.CFrame
	cam.CameraType = Enum.CameraType.Scriptable
	freecam.conn = RunService.RenderStepped:Connect(function(dt)
		if not S.Freecam then stopFreecam() return end
		local cam2 = Workspace.CurrentCamera
		cam2.CameraType = Enum.CameraType.Scriptable
		local look = cam2.CFrame.LookVector
		local right = cam2.CFrame.RightVector
		local move = Vector3.zero
		if UIS:IsKeyDown(Enum.KeyCode.W) then move = move + look end
		if UIS:IsKeyDown(Enum.KeyCode.S) then move = move - look end
		if UIS:IsKeyDown(Enum.KeyCode.A) then move = move - right end
		if UIS:IsKeyDown(Enum.KeyCode.D) then move = move + right end
		if UIS:IsKeyDown(Enum.KeyCode.Space) then move = move + Vector3.new(0, 1, 0) end
		if UIS:IsKeyDown(Enum.KeyCode.LeftShift) then move = move - Vector3.new(0, 1, 0) end
		if controls and controls.GetMoveVector then
			local ok, mv = pcall(function() return controls:GetMoveVector() end)
			if ok and typeof(mv) == "Vector3" and mv.Magnitude > 0.05 then
				move = move + right * mv.X + look * (-mv.Z)
			end
		end
		if move.Magnitude > 0 then
			freecam.pos = freecam.pos + move.Unit * (S.FreecamSpeed or 50) * dt
		end
		-- mouse look approx: keep scriptable at freecam.pos with current look from last cam
		freecam.pos = CFrame.new(freecam.pos.Position, freecam.pos.Position + cam2.CFrame.LookVector)
		cam2.CFrame = freecam.pos
	end)
end

-- Config save/load (named)
local CFG_FOLDER = "cutegirl_configs"
function saveConfig(name)
	name = tostring(name or "default"):gsub("[^%w_%-]", "")
	if name == "" then name = "default" end
	pcall(function()
		if not writefile then
			notify("writefile N/A")
			return
		end
		if makefolder then pcall(makefolder, CFG_FOLDER) end
		local HttpService = game:GetService("HttpService")
		writefile(CFG_FOLDER .. "/" .. name .. ".json", HttpService:JSONEncode(S))
		notify("Saved: " .. name)
	end)
end
function loadConfig(name)
	name = tostring(name or "default"):gsub("[^%w_%-]", "")
	if name == "" then name = "default" end
	pcall(function()
		local path = CFG_FOLDER .. "/" .. name .. ".json"
		if not (readfile and isfile and isfile(path)) then
			notify("No config: " .. name)
			return
		end
		local HttpService = game:GetService("HttpService")
		local data = HttpService:JSONDecode(readfile(path))
		if type(data) == "table" then
			for k, v in pairs(data) do
				if S[k] ~= nil then S[k] = v end
			end
			notify("Loaded: " .. name)
		end
	end)
end
function listConfigs()
	pcall(function()
		if not listfiles then
			notify("listfiles N/A")
			return
		end
		local files = listfiles(CFG_FOLDER)
		local names = {}
		for _, f in ipairs(files or {}) do
			local n = tostring(f):match("([^/\\]+)%.json$")
			if n then table.insert(names, n) end
		end
		if #names == 0 then
			notify("No configs")
		else
			notify("Configs: " .. table.concat(names, ", "))
			print("[hub] configs:", table.concat(names, ", "))
		end
	end)
end
pcall(function() loadConfig("default") end)



print("[hub] clean rebuild loaded")
