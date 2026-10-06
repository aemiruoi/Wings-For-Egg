local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local VirtualUser = game:GetService("VirtualUser")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

--==================================================
-- SETTINGS
--==================================================

local ESP_ENABLED = false
local INSTANT_PROMPT_ENABLED = false
local FLY_ENABLED = false
local HIDE_MY_PET = false
local AUTO_COLLECT_ENABLED = false
local SPEED_ENABLED = false
local INVINCIBLE_ENABLED = false
local ANTI_GUARDIAN_ENABLED = false
local ANTI_AFK_ENABLED = false
local FPS_BOOST_ENABLED = false
local teleportGraceUntil = 0   -- selama waktu ini, teleport dari script tidak dianggap "terlempar"

local PROMPT_DISTANCE = 25
local FLY_SPEED = 50
local WALK_SPEED = 32

-- Folder berisi part berbahaya (path dari Workspace, dipisah titik).
-- Tambahkan path lain di sini kalau ada folder part berbahaya lainnya.
local KILL_PART_PATHS = {
	"Map.KillParts",
	"Map.Lava",
}

-- Folder berisi guardian (path dari Workspace)
local GUARDIAN_PATHS = {
	"LocalGuardians",
}

local COLLECT_DELAY = 0.15                   -- jeda tiap teleport ke coin (detik)
local COIN_OFFSET = Vector3.new(0, 0, 0)     -- offset posisi saat teleport ke coin
local RETURN_TO_START = true                 -- balik ke posisi awal saat dimatikan

local ESP_FILL_COLOR = Color3.fromRGB(255, 220, 70)
local ESP_OUTLINE_COLOR = Color3.fromRGB(255, 255, 255)

--==================================================
-- CHARACTER
--==================================================

-- Semua koneksi global disimpan di sini supaya bisa diputus saat Close
local connections = {}
local closed = false

local function getCharacter()
	return player.Character or player.CharacterAdded:Wait()
end

--==================================================
-- GET FRIENDS
--==================================================

local function getFriends()
	local live = Workspace:FindFirstChild("Live")

	if not live then
		return nil
	end

	return live:FindFirstChild("Friends")
end

--==================================================
-- GUI (WindUI): objek toggle dibuat di bagian akhir script
--==================================================

local WindUI
local Window
local espButton, promptButton, flyButton, speedButton
local collectButton, hidePetButton, godButton, guardButton
local afkButton, fpsButton

-- Sinkronkan tampilan toggle WindUI dengan status script.
-- Aman dipanggil berulang: callback toggle selalu mengecek status dulu.
local function applyToggle(toggle, on)

	if toggle and toggle.Set then

		pcall(function()
			toggle:Set(on)
		end)

	end

end

--==================================================
-- EGG ESP
--==================================================

local function getEggPart(egg)

	return egg.PrimaryPart
		or egg:FindFirstChild("RootPart", true)
		or egg:FindFirstChildWhichIsA("BasePart", true)

end

local function removeEggInfo(egg)

	-- Naikkan token supaya proses pembuatan info yang masih menunggu ikut batal
	egg:SetAttribute(
		"EggESPToken",
		(egg:GetAttribute("EggESPToken") or 0) + 1
	)

	local info =
		egg:FindFirstChild("EggESPInfo")

	if info then
		info:Destroy()
	end

end

local function copyUIState(original, clone)

	if original:IsA("TextLabel")
		or original:IsA("TextButton")
		or original:IsA("TextBox") then

		if clone:IsA("TextLabel")
			or clone:IsA("TextButton")
			or clone:IsA("TextBox") then

			clone.Text = original.Text
			clone.TextColor3 = original.TextColor3
			clone.TextTransparency = original.TextTransparency
			clone.TextStrokeColor3 = original.TextStrokeColor3
			clone.TextStrokeTransparency = original.TextStrokeTransparency
			clone.TextScaled = original.TextScaled
			clone.TextSize = original.TextSize
			clone.Font = original.Font
			clone.Visible = original.Visible

		end
	end

	if original:IsA("ImageLabel")
		or original:IsA("ImageButton") then

		if clone:IsA("ImageLabel")
			or clone:IsA("ImageButton") then

			clone.Image = original.Image
			clone.ImageColor3 = original.ImageColor3
			clone.ImageTransparency = original.ImageTransparency
			clone.Visible = original.Visible

		end
	end

	if original:IsA("GuiObject")
		and clone:IsA("GuiObject") then

		clone.Visible = original.Visible
		clone.BackgroundTransparency =
			original.BackgroundTransparency
		clone.BackgroundColor3 =
			original.BackgroundColor3

	end

end

local function syncBillboard(original, clone)

	for _, originalObject in ipairs(
		original:GetDescendants()
	) do

		local relativePath = {}
		local current = originalObject

		while current and current ~= original do

			table.insert(
				relativePath,
				1,
				current.Name
			)

			current = current.Parent

		end

		local cloneObject = clone

		for _, name in ipairs(relativePath) do

			cloneObject =
				cloneObject:FindFirstChild(name)

			if not cloneObject then
				break
			end

		end

		if cloneObject then
			copyUIState(
				originalObject,
				cloneObject
			)
		end

	end

end

-- Cari billboard asli: nama FriendBillboard (boleh di dalam part), atau billboard apa pun di egg
local function findOriginalBillboard(egg)

	local named =
		egg:FindFirstChild("FriendBillboard", true)

	if named and named:IsA("BillboardGui") then
		return named
	end

	for _, descendant in ipairs(egg:GetDescendants()) do

		if descendant:IsA("BillboardGui")
			and descendant.Name ~= "EggESPInfo" then

			return descendant

		end

	end

	return nil

end

-- Cetak struktur satu egg ke Output (sekali saja) untuk membantu diagnosa
local dumpedEggStructure = false

local function dumpEggStructure(egg)

	if dumpedEggStructure then
		return
	end

	dumpedEggStructure = true

	local fullName = egg:GetFullName()

	print("[EggESP] Struktur egg contoh:", fullName)

	for name, value in pairs(egg:GetAttributes()) do
		print("[EggESP]   attribute:", name, "=", tostring(value))
	end

	local count = 0

	for _, descendant in ipairs(egg:GetDescendants()) do

		count += 1

		if count > 40 then
			print("[EggESP]   ... (dipotong, terlalu banyak)")
			break
		end

		print(
			"[EggESP]  ",
			descendant.ClassName,
			string.sub(descendant:GetFullName(), #fullName + 2)
		)

	end

end

-- Teks cadangan kalau game tidak punya billboard sama sekali
local function createFallbackInfo(egg, rootPart)

	local billboard =
		Instance.new("BillboardGui")

	billboard.Name = "EggESPInfo"
	billboard.AlwaysOnTop = true
	billboard.Size = UDim2.fromOffset(180, 40)
	billboard.StudsOffset = Vector3.new(0, 4, 0)
	billboard.MaxDistance = 500
	billboard.Adornee = rootPart
	billboard.Parent = egg

	local label =
		Instance.new("TextLabel")

	label.Size = UDim2.fromScale(1, 1)
	label.BackgroundTransparency = 1
	label.Text = egg.Name
	label.TextColor3 = ESP_FILL_COLOR
	label.TextStrokeTransparency = 0
	label.TextSize = 15
	label.Font = Enum.Font.GothamBold
	label.TextWrapped = true
	label.Parent = billboard

	return billboard

end

local function createEggInfo(egg)

	if not egg:IsA("Model") then
		return
	end

	removeEggInfo(egg)

	local token =
		egg:GetAttribute("EggESPToken")

	task.spawn(function()

		local original
		local rootPart

		-- Tunggu sampai 5 detik kalau billboard / part belum muncul
		for _ = 1, 25 do

			if not egg.Parent
				or egg:GetAttribute("EggESPToken") ~= token then

				return

			end

			original = findOriginalBillboard(egg)
			rootPart = getEggPart(egg)

			if original and rootPart then
				break
			end

			task.wait(0.2)

		end

		if not egg.Parent
			or egg:GetAttribute("EggESPToken") ~= token then

			return

		end

		if not rootPart then
			warn("[EggESP] Tidak ada part di egg:", egg.Name)
			return
		end

		local clone

		if original then

			clone = original:Clone()
			clone.Name = "EggESPInfo"
			clone.Enabled = true
			clone.AlwaysOnTop = true
			clone.MaxDistance = 500
			clone.Adornee = rootPart
			clone.StudsOffset = Vector3.new(0, 4, 0)
			clone.Parent = egg

			syncBillboard(original, clone)

		else

			warn(
				"[EggESP] FriendBillboard tidak ditemukan di",
				egg.Name,
				"- pakai teks cadangan."
			)

			clone = createFallbackInfo(egg, rootPart)

			dumpEggStructure(egg)

			-- Kalau billboard asli muncul belakangan, ganti teks cadangan dengan yang asli
			task.spawn(function()

				while clone.Parent
					and egg.Parent
					and egg:GetAttribute("EggESPToken") == token do

					if findOriginalBillboard(egg) then
						createEggInfo(egg)
						return
					end

					task.wait(1)

				end

			end)

		end

		if original then

			while clone.Parent
				and egg.Parent
				and original.Parent
				and egg:GetAttribute("EggESPToken") == token do

				syncBillboard(original, clone)

				task.wait(0.5)

			end

		end

	end)

end

local function setEggESP(egg, enabled)

	if not egg:IsA("Model") then
		return
	end

	local highlight =
		egg:FindFirstChild("EggESP")

	if enabled then

		if not highlight then

			highlight =
				Instance.new("Highlight")

			highlight.Name =
				"EggESP"

			highlight.Adornee =
				egg

			highlight.FillColor =
				ESP_FILL_COLOR

			highlight.OutlineColor =
				ESP_OUTLINE_COLOR

			highlight.FillTransparency =
				0.65

			highlight.OutlineTransparency =
				0

			highlight.DepthMode =
				Enum.HighlightDepthMode.AlwaysOnTop

			highlight.Parent =
				egg

		end

		createEggInfo(egg)

	else

		if highlight then
			highlight:Destroy()
		end

		removeEggInfo(egg)

	end

end

local function updateAllESP()

	local friends =
		getFriends()

	if not friends then
		warn("Workspace.Live.Friends tidak ditemukan.")
		return
	end

	for _, egg in ipairs(
		friends:GetChildren()
	) do

		setEggESP(
			egg,
			ESP_ENABLED
		)

	end

end

--==================================================
-- INSTANT PROMPT
--==================================================

local originalPrompts = setmetatable({}, {__mode = "k"})

local function updatePrompt(object)

	if not object:IsA("ProximityPrompt") then
		return
	end

	if object.Name ~= "StealPrompt" then
		return
	end

	if INSTANT_PROMPT_ENABLED then

		if not originalPrompts[object] then
			originalPrompts[object] = {
				hold = object.HoldDuration,
				distance = object.MaxActivationDistance,
				lineOfSight = object.RequiresLineOfSight,
			}
		end

		object.HoldDuration = 0

		object.MaxActivationDistance =
			PROMPT_DISTANCE

		object.RequiresLineOfSight =
			false

	end

end

local function updateAllPrompts()

	local friends =
		getFriends()

	if not friends then
		return
	end

	for _, object in ipairs(
		friends:GetDescendants()
	) do

		updatePrompt(object)

	end

end

--==================================================
-- FLY SPEED
--==================================================

local function updateFlySpeed()
	-- FLY_SPEED diatur lewat slider di UI
end


--==================================================
-- FLY
--==================================================

local flyConnection = nil

local function stopFly()

	FLY_ENABLED = false

	if flyConnection then
		flyConnection:Disconnect()
		flyConnection = nil
	end

	local character =
		player.Character

	if character then

		local humanoid =
			character:FindFirstChildOfClass(
				"Humanoid"
			)

		local root =
			character:FindFirstChild(
				"HumanoidRootPart"
			)

		if humanoid then
			humanoid.PlatformStand = false
		end

		if root then

			local velocity =
				root:FindFirstChild(
					"FlyVelocity"
				)

			if velocity then
				velocity:Destroy()
			end

			local gyro =
				root:FindFirstChild(
					"FlyGyro"
				)

			if gyro then
				gyro:Destroy()
			end

		end

	end

	applyToggle(flyButton, false)

end

local function startFly()

	updateFlySpeed()

	local character =
		getCharacter()

	local humanoid =
		character:WaitForChild(
			"Humanoid"
		)

	local root =
		character:WaitForChild(
			"HumanoidRootPart"
		)

	FLY_ENABLED = true

	applyToggle(flyButton, true)

	local velocity =
		root:FindFirstChild(
			"FlyVelocity"
		)

	if not velocity then

		velocity =
			Instance.new("BodyVelocity")

		velocity.Name =
			"FlyVelocity"

		velocity.MaxForce =
			Vector3.new(
				math.huge,
				math.huge,
				math.huge
			)

		velocity.P = 10000
		velocity.Velocity = Vector3.zero
		velocity.Parent = root

	end

	local gyro =
		root:FindFirstChild(
			"FlyGyro"
		)

	if not gyro then

		gyro =
			Instance.new("BodyGyro")

		gyro.Name =
			"FlyGyro"

		gyro.MaxTorque =
			Vector3.new(
				math.huge,
				math.huge,
				math.huge
			)

		gyro.P = 10000
		gyro.D = 500
		gyro.Parent = root

	end

	humanoid.PlatformStand = true

	flyConnection =
		RunService.RenderStepped:Connect(function()

			if not FLY_ENABLED then
				return
			end

			if not character.Parent then
				stopFly()
				return
			end

			local camera =
				Workspace.CurrentCamera

			if not camera then
				return
			end

			local moveDirection =
				Vector3.zero

			if UserInputService:IsKeyDown(
				Enum.KeyCode.W
			) then

				moveDirection +=
					camera.CFrame.LookVector

			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.S
			) then

				moveDirection -=
					camera.CFrame.LookVector

			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.A
			) then

				moveDirection -=
					camera.CFrame.RightVector

			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.D
			) then

				moveDirection +=
					camera.CFrame.RightVector

			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.Space
			) then

				moveDirection +=
					Vector3.new(0, 1, 0)

			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.LeftControl
			)
			or UserInputService:IsKeyDown(
				Enum.KeyCode.RightControl
			) then

				moveDirection -=
					Vector3.new(0, 1, 0)

			end

			if moveDirection.Magnitude > 0 then

				moveDirection =
					moveDirection.Unit

				velocity.Velocity =
					moveDirection *
					FLY_SPEED

			else

				velocity.Velocity =
					Vector3.zero

			end

			local look =
				camera.CFrame.LookVector

			gyro.CFrame =
				CFrame.lookAt(
					root.Position,
					root.Position + look
				)

		end)

end

--==================================================
-- HIDE MY PET
--==================================================

local function isMyPet(object)

	if not object then
		return false
	end

	-- Karakter sendiri (dan isinya) bukan pet
	local myCharacter = player.Character

	if myCharacter
		and (object == myCharacter
			or object:IsDescendantOf(myCharacter)) then

		return false

	end

	-- Karakter pemain lain juga bukan pet
	if Players:GetPlayerFromCharacter(object) then
		return false
	end

	-- Attribute Owner
	local owner =
		object:GetAttribute("Owner")

	local ownerUserId =
		object:GetAttribute("OwnerUserId")

	local ownerPlayer =
		object:GetAttribute("Player")

	if owner then

		if owner == player.Name
			or owner == player.UserId
			or tostring(owner) ==
				tostring(player.UserId) then

			return true

		end

	end

	if ownerUserId then

		if tostring(ownerUserId) ==
			tostring(player.UserId) then

			return true

		end

	end

	if ownerPlayer then

		if ownerPlayer == player.Name
			or ownerPlayer == player.UserId
			or tostring(ownerPlayer) ==
				tostring(player.UserId) then

			return true

		end

	end

	-- ObjectValue Owner
	local ownerValue =
		object:FindFirstChild(
			"Owner",
			true
		)

	if ownerValue
		and ownerValue:IsA("ObjectValue") then

		if ownerValue.Value == player then
			return true
		end

	end

	-- Nama object mengandung nama player
	if string.find(
		string.lower(object.Name),
		string.lower(player.Name),
		1,
		true
	) then

		return true

	end

	return false

end

-- Nilai asli efek/decal disimpan supaya bisa dipulihkan persis seperti semula
local hiddenOriginals = setmetatable({}, {__mode = "k"})
local hiddenPets = setmetatable({}, {__mode = "k"})
local hideRun = 0

local function hasEnabledProperty(object)

	return object:IsA("ParticleEmitter")
		or object:IsA("Trail")
		or object:IsA("Beam")
		or object:IsA("Fire")
		or object:IsA("Smoke")
		or object:IsA("Sparkles")
		or object:IsA("Light")
		or object:IsA("Highlight")
		or object:IsA("BillboardGui")
		or object:IsA("SurfaceGui")

end

local function applyHidden(object, hide)

	if object:IsA("BasePart") then

		object.LocalTransparencyModifier =
			hide and 1 or 0

	elseif object:IsA("Decal")
		or object:IsA("Texture") then

		if hide then

			if hiddenOriginals[object] == nil then
				hiddenOriginals[object] = object.Transparency
			end

			object.Transparency = 1

		else

			local original = hiddenOriginals[object]

			if original ~= nil then
				object.Transparency = original
				hiddenOriginals[object] = nil
			end

		end

	elseif hasEnabledProperty(object) then

		if hide then

			if hiddenOriginals[object] == nil then
				hiddenOriginals[object] = object.Enabled
			end

			object.Enabled = false

		else

			local original = hiddenOriginals[object]

			if original ~= nil then
				object.Enabled = original
				hiddenOriginals[object] = nil
			end

		end

	end

end

local function setPetVisibility(object, visible)

	if not object:IsA("Model") then
		return
	end

	if not isMyPet(object) then
		return
	end

	hiddenPets[object] = (not visible) or nil

	for _, descendant in ipairs(
		object:GetDescendants()
	) do

		applyHidden(descendant, not visible)

	end

end

local function restoreCharacter()

	local myCharacter = player.Character

	if myCharacter then

		for _, descendant in ipairs(
			myCharacter:GetDescendants()
		) do

			if descendant:IsA("BasePart") then
				descendant.LocalTransparencyModifier = 0
			end

		end

	end

end

-- Model yang sudah dicek dan bukan milikmu dilewati selama 20 detik (hemat CPU)
local notMineCache = setmetatable({}, {__mode = "k"})

local function scanPets(force)

	local now = os.clock()

	for _, object in ipairs(
		Workspace:GetDescendants()
	) do

		if object:IsA("Model") then

			local checkedAt = notMineCache[object]

			if force
				or not checkedAt
				or now - checkedAt > 20 then

				if isMyPet(object) then

					notMineCache[object] = nil

					setPetVisibility(
						object,
						not HIDE_MY_PET
					)

				else

					notMineCache[object] = now

				end

			end

		end

	end

end

local function updateMyPets()

	-- Pulihkan karakter kalau sempat ke-hide
	restoreCharacter()

	scanPets(true)

end

-- Selama Hide My Pet ON: sembunyikan ulang tiap 0.5 detik (efek mutasi bisa
-- dinyalakan lagi oleh game) dan scan ulang penuh tiap ~3 detik untuk pet baru
local function startHideLoop()

	hideRun += 1

	local runId = hideRun

	task.spawn(function()

		local ticks = 0

		while HIDE_MY_PET
			and not closed
			and hideRun == runId do

			ticks += 1

			for pet in pairs(hiddenPets) do

				if pet.Parent then

					for _, descendant in ipairs(
						pet:GetDescendants()
					) do

						applyHidden(descendant, true)

					end

				else

					hiddenPets[pet] = nil

				end

			end

			if ticks % 5 == 0 then
				scanPets()
			end

			task.wait(1)

		end

	end)

end

--==================================================
-- WALK SPEED BOOST
--==================================================

local originalWalkSpeeds = setmetatable({}, {__mode = "k"})
local speedConnection = nil

local function getHumanoid()

	local character =
		player.Character

	return character
		and character:FindFirstChildOfClass(
			"Humanoid"
		)

end

local function updateWalkSpeed()
	-- WALK_SPEED diatur lewat slider di UI
end


local function stopSpeedBoost()

	SPEED_ENABLED = false

	if speedConnection then
		speedConnection:Disconnect()
		speedConnection = nil
	end

	local humanoid =
		getHumanoid()

	if humanoid
		and originalWalkSpeeds[humanoid] then

		humanoid.WalkSpeed =
			originalWalkSpeeds[humanoid]

		originalWalkSpeeds[humanoid] = nil

	end

	applyToggle(speedButton, false)

end

local function startSpeedBoost()

	updateWalkSpeed()

	SPEED_ENABLED = true

	applyToggle(speedButton, true)

	-- Heartbeat supaya tetap jalan walau game mereset WalkSpeed
	speedConnection =
		RunService.Heartbeat:Connect(function()

			local humanoid =
				getHumanoid()

			if not humanoid then
				return
			end

			if originalWalkSpeeds[humanoid] == nil then
				originalWalkSpeeds[humanoid] =
					humanoid.WalkSpeed
			end

			if humanoid.WalkSpeed ~= WALK_SPEED then
				humanoid.WalkSpeed = WALK_SPEED
			end

		end)

end

--==================================================
-- ANTI KILL PARTS
--==================================================

local originalCanTouch = setmetatable({}, {__mode = "k"})
local originalScriptEnabled = setmetatable({}, {__mode = "k"})
local killWatchConnections = {}
local watchedContainers = {}
local invincibleRun = 0

local function resolvePath(path)

	local current = Workspace

	for name in string.gmatch(path, "[^%.]+") do

		current =
			current:FindFirstChild(name)

		if not current then
			return nil
		end

	end

	return current

end

local function protectPart(part)

	if part:IsA("BasePart") then

		-- Simpan nilai asli sekali saja
		if originalCanTouch[part] == nil then
			originalCanTouch[part] = part.CanTouch
		end

		-- Kalau game mengaktifkannya lagi (misalnya saat respawn), matikan lagi
		if part.CanTouch then
			part.CanTouch = false
		end

	end

	-- Script di dalam part (misalnya "lava S") ikut dimatikan
	if part:IsA("LuaSourceContainer") then

		local ok, enabled = pcall(function()
			return part.Enabled
		end)

		if ok then

			if originalScriptEnabled[part] == nil then
				originalScriptEnabled[part] = enabled
			end

			if enabled then

				pcall(function()
					part.Enabled = false
				end)

			end

		end

	end

end

local function protectContainer(container)

	protectPart(container)

	for _, descendant in ipairs(
		container:GetDescendants()
	) do

		protectPart(descendant)

	end

end

-- Scan ulang semua path: tangani folder yang diganti game dan part baru
local function refreshInvincible()

	for _, path in ipairs(KILL_PART_PATHS) do

		local container =
			resolvePath(path)

		if container then

			if watchedContainers[path] ~= container then

				watchedContainers[path] = container

				killWatchConnections[#killWatchConnections + 1] =
					container.DescendantAdded:Connect(function(descendant)

						if INVINCIBLE_ENABLED then
							protectPart(descendant)
						end

					end)

			end

			protectContainer(container)

		end

	end

end

local function stopInvincible()

	INVINCIBLE_ENABLED = false

	invincibleRun += 1

	for _, connection in ipairs(killWatchConnections) do
		connection:Disconnect()
	end

	table.clear(killWatchConnections)
	table.clear(watchedContainers)

	for part, original in pairs(originalCanTouch) do

		if part.Parent then
			part.CanTouch = original
		end

		originalCanTouch[part] = nil

	end

	for scriptObject, enabled in pairs(originalScriptEnabled) do

		if scriptObject.Parent then

			pcall(function()
				scriptObject.Enabled = enabled
			end)

		end

		originalScriptEnabled[scriptObject] = nil

	end

	applyToggle(godButton, false)

end

local function startInvincible()

	INVINCIBLE_ENABLED = true

	invincibleRun += 1

	local runId = invincibleRun

	applyToggle(godButton, true)

	refreshInvincible()

	for _, path in ipairs(KILL_PART_PATHS) do

		if not resolvePath(path) then
			warn("[AntiKill] Path tidak ditemukan (akan dicoba lagi):", path)
		end

	end

	-- Scan ulang tiap detik supaya tetap aktif walau map/part di-reset game
	task.spawn(function()

		while INVINCIBLE_ENABLED
			and not closed
			and invincibleRun == runId do

			task.wait(2)

			if INVINCIBLE_ENABLED
				and invincibleRun == runId then

				refreshInvincible()

			end

		end

	end)

end

-- Setelah respawn, langsung terapkan ulang
connections[#connections + 1] =
	player.CharacterAdded:Connect(function()

		if INVINCIBLE_ENABLED then

			refreshInvincible()

			task.wait(0.5)

			if INVINCIBLE_ENABLED then
				refreshInvincible()
			end

		end

	end)

--==================================================
-- ANTI GUARDIAN
--==================================================

local GUARD_RADIUS = 45      -- jarak guardian yang dianggap "dekat" (studs)
local GUARD_MAX_STEP = 25    -- lompatan posisi per frame yang dianggap terlempar

local guardOriginalCanTouch = setmetatable({}, {__mode = "k"})
local guardWatchConnections = {}
local guardWatched = {}
local guardStepConnection = nil
local guardRun = 0
local guardLastCharacter = nil
local guardLastHumanoid = nil
local guardLastCFrame = nil
local guardAttributeConnection = nil
local guardPreStepConnection = nil
local guardHookInstalled = false
local guardOldNamecall = nil

local function guardianNear(position, radius)

	for _, path in ipairs(GUARDIAN_PATHS) do

		local container =
			resolvePath(path)

		if container then

			for _, guardian in ipairs(
				container:GetChildren()
			) do

				if guardian:IsA("PVInstance")
					and (guardian:GetPivot().Position - position).Magnitude <= radius then

					return true

				end

			end

		end

	end

	return false

end

local function protectGuardianPart(part)

	if part:IsA("BasePart") then

		if guardOriginalCanTouch[part] == nil then
			guardOriginalCanTouch[part] = part.CanTouch
		end

		if part.CanTouch then
			part.CanTouch = false
		end

	end

end

local function refreshGuardians()

	for _, path in ipairs(GUARDIAN_PATHS) do

		local container =
			resolvePath(path)

		if container then

			if guardWatched[path] ~= container then

				guardWatched[path] = container

				guardWatchConnections[#guardWatchConnections + 1] =
					container.DescendantAdded:Connect(function(descendant)

						if ANTI_GUARDIAN_ENABLED then
							protectGuardianPart(descendant)
						end

					end)

			end

			protectGuardianPart(container)

			for _, descendant in ipairs(
				container:GetDescendants()
			) do

				protectGuardianPart(descendant)

			end

		end

	end

end

local function setHumanoidHitStates(humanoid, enabled)

	pcall(function()
		humanoid:SetStateEnabled(
			Enum.HumanoidStateType.Ragdoll,
			enabled
		)
		humanoid:SetStateEnabled(
			Enum.HumanoidStateType.FallingDown,
			enabled
		)
	end)

end

local function isForceObject(instance)

	return instance:IsA("BodyVelocity")
		or instance:IsA("BodyForce")
		or instance:IsA("BodyThrust")
		or instance:IsA("BodyPosition")
		or instance:IsA("BodyAngularVelocity")
		or instance:IsA("BodyGyro")
		or instance:IsA("VectorForce")
		or instance:IsA("LinearVelocity")
		or instance:IsA("AngularVelocity")
		or instance:IsA("Torque")

end

-- Dari script Guardian: guardian hanya mengejar / menyerang kalau atribut karakter
-- "inDangerZone" bernilai true. Kita paksa false (hanya di sisi client).
local function suppressDangerZone()

	local character =
		player.Character

	if character
		and character:GetAttribute("inDangerZone") then

		character:SetAttribute("inDangerZone", false)

	end

end

local function isGuardianRemote(instance)

	if instance.ClassName ~= "RemoteEvent" then
		return false
	end

	local name =
		string.lower(instance.Name)

	return string.find(name, "guardian", 1, true) ~= nil
		and string.find(name, "caught", 1, true) ~= nil

end

-- Lapisan cadangan: kalau guardian tetap sempat menyerang, remote "Guardian Caught"
-- tidak dikirim ke server (hanya jalan kalau executor mendukung hookmetamethod)
local function installGuardianRemoteBlock()

	if guardHookInstalled then
		return
	end

	if not (hookmetamethod and getnamecallmethod) then

		warn(
			"[AntiGuardian] hookmetamethod tidak didukung executor ini,",
			"hanya memakai pencegahan target."
		)

		return

	end

	local ok, err = pcall(function()

		local wrap =
			newcclosure or function(callback)
				return callback
			end

		guardOldNamecall =
			hookmetamethod(
				game,
				"__namecall",
				wrap(function(self, ...)

					local method =
						getnamecallmethod()

					if method == "FireServer"
						and ANTI_GUARDIAN_ENABLED
						and not closed
						and typeof(self) == "Instance"
						and isGuardianRemote(self) then

						return nil

					end

					return guardOldNamecall(self, ...)

				end)
			)

	end)

	if ok then
		guardHookInstalled = true
	else
		warn("[AntiGuardian] Gagal memasang blokir remote:", err)
	end

end

local function guardStep()

	local character =
		player.Character

	local root =
		character
		and character:FindFirstChild("HumanoidRootPart")

	local humanoid =
		character
		and character:FindFirstChildOfClass("Humanoid")

	if not root or not humanoid then
		guardLastCFrame = nil
		return
	end

	suppressDangerZone()

	-- Karakter baru (respawn): jangan bandingkan dengan posisi lama
	if character ~= guardLastCharacter then

		guardLastCharacter = character
		guardLastCFrame = nil

		if guardAttributeConnection then
			guardAttributeConnection:Disconnect()
		end

		-- Kalau game menyalakan lagi atributnya, langsung matikan
		guardAttributeConnection =
			character:GetAttributeChangedSignal("inDangerZone"):Connect(function()

				if ANTI_GUARDIAN_ENABLED then
					suppressDangerZone()
				end

			end)

	end

	if humanoid ~= guardLastHumanoid then
		guardLastHumanoid = humanoid
		setHumanoidHitStates(humanoid, false)
	end

	-- Fly / auto collect memindahkan karakter dengan sengaja
	local selfMoving =
		FLY_ENABLED
		or AUTO_COLLECT_ENABLED
		or os.clock() < teleportGraceUntil

	-- Terlempar / ditarik jauh sekaligus dekat guardian: kembalikan ke posisi sebelumnya
	if not selfMoving and guardLastCFrame then

		local jump =
			(root.Position - guardLastCFrame.Position).Magnitude

		if jump > GUARD_MAX_STEP
			and guardianNear(guardLastCFrame.Position, GUARD_RADIUS) then

			root.CFrame = guardLastCFrame
			root.AssemblyLinearVelocity = Vector3.zero

			return

		end

	end

	guardLastCFrame = root.CFrame

	if not guardianNear(root.Position, GUARD_RADIUS) then
		return
	end

	-- Buang gaya dorong / tarik yang ditempel ke karakter (selain milik Fly)
	for _, child in ipairs(root:GetChildren()) do

		if isForceObject(child)
			and child.Name ~= "FlyVelocity"
			and child.Name ~= "FlyGyro" then

			child:Destroy()

		end

	end

	-- Tidak boleh pingsan / terkunci
	if humanoid.PlatformStand and not FLY_ENABLED then
		humanoid.PlatformStand = false
	end

	if root.Anchored and humanoid.Health > 0 then
		root.Anchored = false
	end

	-- Batalkan lonjakan kecepatan (knockback)
	if not selfMoving then

		local velocity =
			root.AssemblyLinearVelocity

		local horizontal =
			Vector3.new(velocity.X, 0, velocity.Z)

		local horizontalLimit =
			math.max(humanoid.WalkSpeed, 16) * 1.6 + 4

		local verticalLimit =
			math.max(humanoid.JumpPower, 50) * 1.8

		local newX = velocity.X
		local newY = velocity.Y
		local newZ = velocity.Z
		local changed = false

		if horizontal.Magnitude > horizontalLimit then

			local intended =
				humanoid.MoveDirection * humanoid.WalkSpeed

			newX = intended.X
			newZ = intended.Z
			changed = true

		end

		if velocity.Y > verticalLimit then
			newY = 0
			changed = true
		end

		if changed then
			root.AssemblyLinearVelocity =
				Vector3.new(newX, newY, newZ)
		end

	end

end

local function stopAntiGuardian()

	ANTI_GUARDIAN_ENABLED = false

	guardRun += 1

	if guardStepConnection then
		guardStepConnection:Disconnect()
		guardStepConnection = nil
	end

	if guardPreStepConnection then
		guardPreStepConnection:Disconnect()
		guardPreStepConnection = nil
	end

	if guardAttributeConnection then
		guardAttributeConnection:Disconnect()
		guardAttributeConnection = nil
	end

	for _, connection in ipairs(guardWatchConnections) do
		connection:Disconnect()
	end

	table.clear(guardWatchConnections)
	table.clear(guardWatched)

	for part, original in pairs(guardOriginalCanTouch) do

		if part.Parent then
			part.CanTouch = original
		end

		guardOriginalCanTouch[part] = nil

	end

	local humanoid =
		getHumanoid()

	if humanoid then
		setHumanoidHitStates(humanoid, true)
	end

	guardLastCharacter = nil
	guardLastHumanoid = nil
	guardLastCFrame = nil

	applyToggle(guardButton, false)

end

local function startAntiGuardian()

	ANTI_GUARDIAN_ENABLED = true

	guardRun += 1

	local runId = guardRun

	applyToggle(guardButton, true)

	refreshGuardians()

	for _, path in ipairs(GUARDIAN_PATHS) do

		if not resolvePath(path) then
			warn("[AntiGuardian] Path tidak ditemukan (akan dicoba lagi):", path)
		end

	end

	guardStepConnection =
		RunService.Heartbeat:Connect(guardStep)

	-- Stepped jalan sebelum Heartbeat, jadi atribut sudah false saat guardian mengecek
	guardPreStepConnection =
		RunService.Stepped:Connect(function()
			suppressDangerZone()
		end)

	installGuardianRemoteBlock()

	-- Scan ulang tiap detik: guardian baru / folder yang diganti game
	task.spawn(function()

		while ANTI_GUARDIAN_ENABLED
			and not closed
			and guardRun == runId do

			task.wait(1)

			if ANTI_GUARDIAN_ENABLED
				and guardRun == runId then

				refreshGuardians()

			end

		end

	end)

end

--==================================================
-- TELEPORT KE ISLAND
--==================================================

local ISLAND_FOLDER_PATH = "Map.Ground"   -- isi folder ini = daftar island
local ISLAND_HEIGHT_OFFSET = 6            -- tinggi di atas permukaan saat mendarat

-- Titik tengah dan ukuran island
local function getIslandBounds(island)

	if island:IsA("Model") then

		local ok, cframe, size = pcall(function()
			return island:GetBoundingBox()
		end)

		if ok then
			return cframe.Position, size
		end

	end

	-- Folder / Part: hitung manual dari semua part di dalamnya
	local minV
	local maxV

	local function include(part)

		local half = part.Size / 2
		local low = part.Position - half
		local high = part.Position + half

		if not minV then

			minV = low
			maxV = high

		else

			minV = Vector3.new(
				math.min(minV.X, low.X),
				math.min(minV.Y, low.Y),
				math.min(minV.Z, low.Z)
			)

			maxV = Vector3.new(
				math.max(maxV.X, high.X),
				math.max(maxV.Y, high.Y),
				math.max(maxV.Z, high.Z)
			)

		end

	end

	if island:IsA("BasePart") then
		include(island)
	end

	for _, descendant in ipairs(island:GetDescendants()) do

		if descendant:IsA("BasePart") then
			include(descendant)
		end

	end

	if not minV then
		return nil
	end

	return (minV + maxV) / 2, maxV - minV

end

local function teleportToIsland(island, button)

	if not island or not island.Parent then
		warn("[Teleport] Island sudah tidak ada, tekan Refresh.")
		return
	end

	local character =
		player.Character

	local root =
		character
		and character:FindFirstChild("HumanoidRootPart")

	local humanoid =
		character
		and character:FindFirstChildOfClass("Humanoid")

	if not root
		or not humanoid
		or humanoid.Health <= 0 then

		return

	end

	local center, size =
		getIslandBounds(island)

	if not center then
		warn("[Teleport] Tidak ada part di island:", island.Name)
		return
	end

	-- Cari permukaan tanah di tengah island dengan raycast dari atas
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = {character}
	params.IgnoreWater = true

	local origin =
		Vector3.new(
			center.X,
			center.Y + size.Y / 2 + 100,
			center.Z
		)

	local result =
		Workspace:Raycast(
			origin,
			Vector3.new(0, -(size.Y + 400), 0),
			params
		)

	local target

	if result then

		target =
			result.Position
			+ Vector3.new(0, ISLAND_HEIGHT_OFFSET, 0)

	else

		target =
			center
			+ Vector3.new(
				0,
				size.Y / 2 + ISLAND_HEIGHT_OFFSET,
				0
			)

	end

	-- Beri tahu fitur lain bahwa ini teleport yang disengaja
	teleportGraceUntil = os.clock() + 1

	root.AssemblyLinearVelocity = Vector3.zero
	root.CFrame = CFrame.new(target)

	if button then

		button.Text = "✓"

		task.delay(0.8, function()

			if button.Parent then
				button.Text = "GO"
			end

		end)

	end

end

local islandMap = {}

-- Kumpulkan island dari folder, kembalikan daftar nama (nama kembar diberi nomor)
local function collectIslands()

	islandMap = {}

	local names = {}

	local folder =
		resolvePath(ISLAND_FOLDER_PATH)

	if not folder then

		warn(
			"[Teleport] Folder island tidak ditemukan:",
			ISLAND_FOLDER_PATH
		)

		return names

	end

	local islands = {}

	for _, island in ipairs(folder:GetChildren()) do

		if island:IsA("Model")
			or island:IsA("Folder")
			or island:IsA("BasePart") then

			islands[#islands + 1] = island

		end

	end

	table.sort(islands, function(a, b)
		return a.Name < b.Name
	end)

	for _, island in ipairs(islands) do

		local uniqueName = island.Name
		local count = 1

		while islandMap[uniqueName] do
			count += 1
			uniqueName = island.Name .. " (" .. count .. ")"
		end

		islandMap[uniqueName] = island
		names[#names + 1] = uniqueName

	end

	return names

end


--==================================================
-- AUTO COLLECT BLOODMOON COINS (TELEPORT)
--==================================================

local collectId = 0

local function getCoinsFolder()

	local live =
		Workspace:FindFirstChild("Live")

	if not live then
		return nil
	end

	return live:FindFirstChild("BloodMoonCoins")

end

local function getCoinPosition(coin)

	if coin:IsA("BasePart") then
		return coin.Position
	end

	if coin:IsA("Model") then
		return coin:GetPivot().Position
	end

	local part =
		coin:FindFirstChildWhichIsA(
			"BasePart",
			true
		)

	if part then
		return part.Position
	end

	return nil

end

-- Bantuan tambahan kalau executor mendukung (aman kalau tidak ada)
local function triggerCoin(root, coin)

	pcall(function()

		local part =
			coin:IsA("BasePart")
			and coin
			or coin:FindFirstChildWhichIsA(
				"BasePart",
				true
			)

		if part and firetouchinterest then
			firetouchinterest(root, part, 0)
			firetouchinterest(root, part, 1)
		end

		local prompt =
			coin:FindFirstChildWhichIsA(
				"ProximityPrompt",
				true
			)

		if prompt and fireproximityprompt then
			fireproximityprompt(prompt)
		end

	end)

end

local function getRoot()

	local character =
		player.Character

	return character
		and character:FindFirstChild(
			"HumanoidRootPart"
		)

end

local function startAutoCollect()

	AUTO_COLLECT_ENABLED = true

	applyToggle(collectButton, true)

	collectId += 1

	local runId = collectId

	task.spawn(function()

		local startRoot = getRoot()

		local origin =
			startRoot and startRoot.CFrame

		while AUTO_COLLECT_ENABLED
			and not closed
			and collectId == runId do

			local folder =
				getCoinsFolder()

			if folder and getRoot() then

				for _, coin in ipairs(
					folder:GetChildren()
				) do

					if not AUTO_COLLECT_ENABLED
						or closed
						or collectId ~= runId then

						break

					end

					if coin.Parent then

						local position =
							getCoinPosition(coin)

						local root =
							getRoot()

						if position and root then

							root.CFrame =
								CFrame.new(
									position + COIN_OFFSET
								)

							root.AssemblyLinearVelocity =
								Vector3.zero

							triggerCoin(
								root,
								coin
							)

							task.wait(COLLECT_DELAY)

						end

					end

				end

			end

			task.wait(0.5)

		end

		-- Balik ke posisi awal
		local root =
			getRoot()

		if RETURN_TO_START
			and origin
			and root
			and collectId == runId then

			root.CFrame = origin

		end

	end)

end

local function stopAutoCollect()

	AUTO_COLLECT_ENABLED = false

	applyToggle(collectButton, false)

end

--==================================================
-- NEW EGG / PROMPT
--==================================================

task.spawn(function()

	local live =
		Workspace:WaitForChild(
			"Live"
		)

	local friends =
		live:WaitForChild(
			"Friends"
		)

	if closed then
		return
	end

	connections[#connections + 1] = friends.ChildAdded:Connect(function(egg)

		task.wait(0.2)

		if ESP_ENABLED then

			setEggESP(
				egg,
				true
			)

		end

		if INSTANT_PROMPT_ENABLED then

			for _, object in ipairs(
				egg:GetDescendants()
			) do

				updatePrompt(object)

			end

		end

	end)

	connections[#connections + 1] = friends.DescendantAdded:Connect(function(object)

		if INSTANT_PROMPT_ENABLED then

			task.wait()

			updatePrompt(object)

		end

	end)

end)

--==================================================
-- DETECT NEW PET
--==================================================

connections[#connections + 1] = Workspace.DescendantAdded:Connect(function(object)

	if not HIDE_MY_PET then
		return
	end

	if object:IsA("Model") then

		task.wait()

		if isMyPet(object) then

			setPetVisibility(
				object,
				false
			)

		end

		return

	end

	-- Part / efek baru (misalnya efek mutasi) di dalam pet yang sudah disembunyikan
	local current = object.Parent

	while current and current ~= Workspace do

		if hiddenPets[current] then
			applyHidden(object, true)
			return
		end

		current = current.Parent

	end

end)

--==================================================
-- CHARACTER RESPAWN
--==================================================

connections[#connections + 1] = player.CharacterAdded:Connect(function()

	if FLY_ENABLED then
		stopFly()
	end

	task.wait(1)

	if HIDE_MY_PET then
		updateMyPets()
	end

end)

--==================================================
-- ANTI AFK
--==================================================

local afkConnection = nil

local function stopAntiAfk()

	ANTI_AFK_ENABLED = false

	if afkConnection then
		afkConnection:Disconnect()
		afkConnection = nil
	end

	applyToggle(afkButton, false)

end

local function startAntiAfk()

	ANTI_AFK_ENABLED = true

	applyToggle(afkButton, true)

	if afkConnection then
		afkConnection:Disconnect()
	end

	-- Roblox menandai pemain AFK lewat event Idled; kirim input palsu supaya tidak ter-kick
	afkConnection =
		player.Idled:Connect(function()

			pcall(function()
				VirtualUser:CaptureController()
				VirtualUser:ClickButton2(Vector2.new(0, 0))
			end)

		end)

end

--==================================================
-- FPS BOOST
--==================================================

local fpsOriginals = setmetatable({}, {__mode = "k"})
local fpsSaved = {}
local fpsConnection = nil
local fpsRun = 0

local function isHeavyEffect(object)

	return object:IsA("ParticleEmitter")
		or object:IsA("Trail")
		or object:IsA("Beam")
		or object:IsA("Smoke")
		or object:IsA("Fire")
		or object:IsA("Sparkles")
		or object:IsA("PostEffect")

end

local function applyFpsBoost(object)

	if isHeavyEffect(object) then

		if fpsOriginals[object] == nil then
			fpsOriginals[object] = object.Enabled
		end

		if object.Enabled then
			object.Enabled = false
		end

	end

end

local function stopFpsBoost()

	FPS_BOOST_ENABLED = false

	fpsRun += 1

	if fpsConnection then
		fpsConnection:Disconnect()
		fpsConnection = nil
	end

	-- Kembalikan efek ke nilai aslinya
	for object, original in pairs(fpsOriginals) do

		if object.Parent then

			pcall(function()
				object.Enabled = original
			end)

		end

		fpsOriginals[object] = nil

	end

	-- Kembalikan pengaturan grafis
	if fpsSaved.quality ~= nil then

		pcall(function()
			settings().Rendering.QualityLevel = fpsSaved.quality
		end)

	end

	if fpsSaved.shadows ~= nil then

		pcall(function()
			Lighting.GlobalShadows = fpsSaved.shadows
		end)

	end

	if fpsSaved.decoration ~= nil then

		pcall(function()
			Workspace.Terrain.Decoration = fpsSaved.decoration
		end)

	end

	table.clear(fpsSaved)

	applyToggle(fpsButton, false)

end

local function startFpsBoost()

	FPS_BOOST_ENABLED = true

	applyToggle(fpsButton, true)

	fpsRun += 1

	local runId = fpsRun

	-- Simpan pengaturan asli, lalu turunkan grafis
	pcall(function()
		fpsSaved.quality = settings().Rendering.QualityLevel
	end)

	pcall(function()
		fpsSaved.shadows = Lighting.GlobalShadows
	end)

	pcall(function()
		fpsSaved.decoration = Workspace.Terrain.Decoration
	end)

	pcall(function()
		settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
	end)

	pcall(function()
		Lighting.GlobalShadows = false
	end)

	pcall(function()
		Workspace.Terrain.Decoration = false
	end)

	-- Matikan efek berat (partikel, trail, beam, post-processing).
	-- Dikerjakan bertahap supaya game tidak membeku.
	task.spawn(function()

		for _, object in ipairs(Lighting:GetDescendants()) do
			applyFpsBoost(object)
		end

		local count = 0

		for _, object in ipairs(Workspace:GetDescendants()) do

			if not FPS_BOOST_ENABLED or fpsRun ~= runId then
				return
			end

			applyFpsBoost(object)

			count += 1

			if count % 1500 == 0 then
				task.wait()
			end

		end

	end)

	-- Efek yang muncul belakangan ikut dimatikan
	fpsConnection =
		Workspace.DescendantAdded:Connect(function(object)

			if FPS_BOOST_ENABLED then
				applyFpsBoost(object)
			end

		end)

end

--==================================================
-- CLOSE SCRIPT
--==================================================

local function restorePrompts()

	for prompt, original in pairs(originalPrompts) do

		if prompt.Parent then

			prompt.HoldDuration = original.hold
			prompt.MaxActivationDistance = original.distance
			prompt.RequiresLineOfSight = original.lineOfSight

		end

	end

	table.clear(originalPrompts)

end

local function closeScript()

	if closed then
		return
	end

	closed = true

	-- Matikan anti AFK dan FPS boost
	if ANTI_AFK_ENABLED then
		stopAntiAfk()
	end

	if FPS_BOOST_ENABLED then
		stopFpsBoost()
	end

	-- Matikan fly
	if FLY_ENABLED then
		stopFly()
	end

	-- Matikan anti guardian
	if ANTI_GUARDIAN_ENABLED then
		stopAntiGuardian()
	end

	-- Matikan anti kill parts
	if INVINCIBLE_ENABLED then
		stopInvincible()
	end

	-- Matikan speed boost
	if SPEED_ENABLED then
		stopSpeedBoost()
	end

	-- Matikan auto collect
	AUTO_COLLECT_ENABLED = false

	-- Matikan ESP
	ESP_ENABLED = false

	local friends = getFriends()

	if friends then

		for _, egg in ipairs(friends:GetChildren()) do
			setEggESP(egg, false)
		end

	end

	-- Kembalikan prompt ke nilai asli
	INSTANT_PROMPT_ENABLED = false
	restorePrompts()

	-- Munculkan lagi pet dan karakter
	HIDE_MY_PET = false
	updateMyPets()

	-- Putus semua koneksi
	for _, connection in ipairs(connections) do
		connection:Disconnect()
	end

	table.clear(connections)

	-- Hapus GUI
	if Window then
		pcall(function()
			Window:Destroy()
		end)
	end

	print("Mire Hub closed.")

end


--==================================================
-- GUI (WindUI)
--==================================================

local WINDUI_VERSION = "1.6.66"

local function loadWindUI()

	local urls = {
		"https://github.com/Footagesus/WindUI/releases/download/"
			.. WINDUI_VERSION
			.. "/main.lua",
		"https://github.com/Footagesus/WindUI/releases/latest/download/main.lua",
	}

	for _, url in ipairs(urls) do

		local ok, result = pcall(function()
			return loadstring(game:HttpGet(url))()
		end)

		if ok and result then
			return result
		end

		warn("[Mire Hub] Gagal memuat WindUI dari:", url)

	end

	return nil

end

WindUI = loadWindUI()

if not WindUI then

	warn(
		"[Mire Hub] WindUI tidak bisa dimuat.",
		"Pastikan executor mendukung HttpGet dan GitHub bisa diakses."
	)

	closeScript()

	return

end

local function notify(title, content)

	pcall(function()
		WindUI:Notify({
			Title = title,
			Content = content,
			Duration = 3,
		})
	end)

end

Window = WindUI:CreateWindow({
	Title = "Wisteria KB Escrot Hub",
	Icon = "egg",
	Author = "Beta Release",
	Size = UDim2.fromOffset(580, 460),
	Theme = "Dark",
	ToggleKey = Enum.KeyCode.RightShift,
})

-- Window ditutup dari UI (tombol close): bersihkan semua fitur
Window:OnDestroy(function()
	closeScript()
end)

local MainTab = Window:Tab({ Title = "Main", Icon = "house" })
local PlayerTab = Window:Tab({ Title = "Player", Icon = "user" })
local ProtectTab = Window:Tab({ Title = "Protection", Icon = "shield" })
local TeleportTab = Window:Tab({ Title = "Teleport", Icon = "map-pin" })

local MiscTab = Window:Tab({ Title = "Misc", Icon = "settings" })

----------------------------------------------
-- TAB: MISC
----------------------------------------------

local UtilitySection = MiscTab:Section({
	Title = "Utility",
	Box = true,
	BoxBorder = true,
	Opened = true,
})

afkButton = UtilitySection:Toggle({
	Title = "Anti AFK",
	Value = false,
	Callback = function(state)

		if state then

			if not ANTI_AFK_ENABLED then
				startAntiAfk()
			end

		elseif ANTI_AFK_ENABLED then

			stopAntiAfk()

		end

	end,
})

fpsButton = UtilitySection:Toggle({
	Title = "FPS Boost",
	Value = false,
	Callback = function(state)

		if state then

			if not FPS_BOOST_ENABLED then
				startFpsBoost()
			end

		elseif FPS_BOOST_ENABLED then

			stopFpsBoost()

		end

	end,
})

local AppearanceSection = MiscTab:Section({
	Title = "Appearance",
	Box = true,
	BoxBorder = true,
	Opened = true,
})

local themeNames = {}
pcall(function()
	for themeName in pairs(WindUI:GetThemes()) do
		table.insert(themeNames, themeName)
	end
end)
table.sort(themeNames)

local themeDropdown
themeDropdown = AppearanceSection:Dropdown({
	Title = "Theme",
	Values = themeNames,
	Value = "Dark",
	SearchBarEnabled = true,
	MenuWidth = 280,
	Callback = function(theme)
		if theme and theme ~= "" then
			WindUI:SetTheme(theme)
		end
	end,
})

AppearanceSection:Button({
	Title = "Change Theme",
	Icon = "palette",
	Callback = function()
		WindUI:SetTheme("Indigo")
		pcall(function()
			themeDropdown:Select("Indigo")
		end)
	end,
})

----------------------------------------------
-- TAB: MAIN
----------------------------------------------

local EggSection = MainTab:Section({
	Title = "Eggs",
	Box = true,
	BoxBorder = true,
	Opened = true,
})

espButton = EggSection:Toggle({
	Title = "Egg ESP",
	Value = false,
	Callback = function(state)

		ESP_ENABLED = state

		updateAllESP()

	end,
})

promptButton = EggSection:Toggle({
	Title = "Instant Prompt",
	Value = false,
	Callback = function(state)

		INSTANT_PROMPT_ENABLED = state

		if state then
			updateAllPrompts()
		end

	end,
})

EggSection:Slider({
	Title = "Prompt Distance",
	Step = 1,
	Value = {
		Min = 5,
		Max = 100,
		Default = PROMPT_DISTANCE,
	},
	Callback = function(value)

		PROMPT_DISTANCE = tonumber(value) or PROMPT_DISTANCE

		if INSTANT_PROMPT_ENABLED then
			updateAllPrompts()
		end

	end,
})

local FarmSection = MainTab:Section({
	Title = "Farm",
	Box = true,
	BoxBorder = true,
	Opened = true,
})

collectButton = FarmSection:Toggle({
	Title = "Auto Collect Coins",
	Value = false,
	Callback = function(state)

		if state then

			if not AUTO_COLLECT_ENABLED then
				startAutoCollect()
			end

		elseif AUTO_COLLECT_ENABLED then

			stopAutoCollect()

		end

	end,
})

----------------------------------------------
-- TAB: PLAYER
----------------------------------------------

local FlySection = PlayerTab:Section({
	Title = "Fly",
	Box = true,
	BoxBorder = true,
	Opened = true,
})

flyButton = FlySection:Toggle({
	Title = "Fly",
	Value = false,
	Callback = function(state)

		if state then

			if not FLY_ENABLED then
				startFly()
			end

		elseif FLY_ENABLED then

			stopFly()

		end

	end,
})

FlySection:Slider({
	Title = "Fly Speed",
	Step = 1,
	Value = {
		Min = 1,
		Max = 5000,
		Default = FLY_SPEED,
	},
	Callback = function(value)
		FLY_SPEED = tonumber(value) or FLY_SPEED
	end,
})

local SpeedSection = PlayerTab:Section({
	Title = "Speed",
	Box = true,
	BoxBorder = true,
	Opened = true,
})

speedButton = SpeedSection:Toggle({
	Title = "Speed Boost",
	Value = false,
	Callback = function(state)

		if state then

			if not SPEED_ENABLED then
				startSpeedBoost()
			end

		elseif SPEED_ENABLED then

			stopSpeedBoost()

		end

	end,
})

SpeedSection:Slider({
	Title = "Walk Speed",
	Step = 1,
	Value = {
		Min = 16,
		Max = 500,
		Default = WALK_SPEED,
	},
	Callback = function(value)
		WALK_SPEED = tonumber(value) or WALK_SPEED
	end,
})

----------------------------------------------
-- TAB: PROTECTION
----------------------------------------------

local ProtectSection = ProtectTab:Section({
	Title = "Protection",
	Box = true,
	BoxBorder = true,
	Opened = true,
})

hidePetButton = ProtectSection:Toggle({
	Title = "Hide My Pet",
	Value = false,
	Callback = function(state)

		if state == HIDE_MY_PET then
			return
		end

		HIDE_MY_PET = state

		updateMyPets()

		if state then
			startHideLoop()
		else
			hideRun += 1
		end

	end,
})

godButton = ProtectSection:Toggle({
	Title = "Anti Kill Parts",
	Value = false,
	Callback = function(state)

		if state then

			if not INVINCIBLE_ENABLED then
				startInvincible()
			end

		elseif INVINCIBLE_ENABLED then

			stopInvincible()

		end

	end,
})

guardButton = ProtectSection:Toggle({
	Title = "Anti Guardian",
	Value = false,
	Callback = function(state)

		if state then

			if not ANTI_GUARDIAN_ENABLED then
				startAntiGuardian()
			end

		elseif ANTI_GUARDIAN_ENABLED then

			stopAntiGuardian()

		end

	end,
})

----------------------------------------------
-- TAB: TELEPORT
----------------------------------------------

local IslandSection = TeleportTab:Section({
	Title = "Island",
	Box = true,
	BoxBorder = true,
	Opened = true,
})

local PLACEHOLDER_ISLAND = "(belum ada island)"
local selectedIslandName = nil
local islandDropdown = nil

local initialNames = collectIslands()

if #initialNames == 0 then
	initialNames = { PLACEHOLDER_ISLAND }
end

islandDropdown = IslandSection:Dropdown({
	Title = "Pilih island",
	Values = initialNames,
	Callback = function(option)
		selectedIslandName = option
	end,
})

local function refreshIslands()

	local names = collectIslands()

	selectedIslandName = nil

	if #names == 0 then
		names = { PLACEHOLDER_ISLAND }
	end

	pcall(function()
		islandDropdown:Refresh(names)
	end)

end

local function teleportSelected()

	local island =
		selectedIslandName
		and islandMap[selectedIslandName]

	if not island then
		notify("Teleport", "Pilih island dulu.")
		return
	end

	teleportToIsland(island, nil)

end

IslandSection:Button({
	Title = "Teleport",
	Callback = teleportSelected,
})

IslandSection:Button({
	Title = "Refresh list",
	Callback = refreshIslands,
})

-- Isi daftar saat script dimuat (tunggu map sampai 10 detik)
task.spawn(function()

	for _ = 1, 20 do

		if closed or resolvePath(ISLAND_FOLDER_PATH) then
			break
		end

		task.wait(0.5)

	end

	if not closed then
		refreshIslands()
	end

end)

pcall(function()
	Window:SelectTab(MainTab)
end)

-- Anti AFK menyala otomatis (bisa dimatikan di tab Misc)
startAntiAfk()

notify("Mire Hub", "Siap. Tekan RightShift untuk menyembunyikan UI.")

--==================================================
-- DONE
--==================================================

print("Mire Hub loaded successfully!")
print("Prompt Distance:", PROMPT_DISTANCE)
print("Fly Speed:", FLY_SPEED)
