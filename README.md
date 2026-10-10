--[[
	Private Server Finder v2 (nâng cấp từ bản GlazeOnTop)
	- Lọc server theo số người của SERVER ĐÍCH (bản cũ chọn ngẫu nhiên, không lọc)
	- Quét nhiều trang, bỏ server đầy và server đã thử
	- Xử lý TeleportInitFailed + lỗi 772 (Flooded), có timeout
	- Chống chạy chồng nhiều vòng lặp
	- Kéo thả được cả trên điện thoại, có nút nổi thu nhỏ, phím RightShift
	- Tùy chọn tự chạy lại sau khi sang server mới (queue_on_teleport)
]]

local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local TweenService = game:GetService("TweenService")
local UIS = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local PlaceId = game.PlaceId

local CONFIG = {
	Pages = 5,             -- số trang server tối đa mỗi lần quét (100 server/trang)
	MinFree = 2,           -- server đích phải còn ít nhất N chỗ trống
	TeleportTimeout = 12,  -- chờ tối đa N giây cho mỗi lần dịch chuyển
	FloodCooldown = 4,     -- nghỉ N giây khi gặp lỗi 772
	-- Dán link RAW chứa chính script này để tự chạy lại sau khi sang server mới.
	-- Để trống = không tự chạy lại.
	ScriptUrl = "",
}

local env = (getgenv and getgenv()) or _G
if env.PSF_Stop then pcall(env.PSF_Stop) end -- dọn bản cũ nếu chạy lại

-- ==============================
-- TRẠNG THÁI
-- ==============================
local alive = true
local autoEnabled = false
local busy = false
local teleporting = false
local flood = false
local queued = false
local tried = {}
local conns = {}

local function track(c)
	table.insert(conns, c)
	return c
end

local COLORS = {
	cyan = Color3.fromRGB(0, 230, 255),
	green = Color3.fromRGB(0, 230, 150),
	amber = Color3.fromRGB(255, 180, 50),
	red = Color3.fromRGB(255, 80, 80),
	grey = Color3.fromRGB(160, 160, 180),
}

-- ==============================
-- GUI
-- ==============================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "PrivateServerFinder"
ScreenGui.ResetOnSpawn = false
ScreenGui.DisplayOrder = 999
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

if syn and syn.protect_gui then pcall(syn.protect_gui, ScreenGui) end
local parented = false
if gethui then parented = pcall(function() ScreenGui.Parent = gethui() end) end
if not parented then parented = pcall(function() ScreenGui.Parent = game:GetService("CoreGui") end) end
if not parented or not ScreenGui.Parent then
	ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
end

local function make(class, props, parent)
	local o = Instance.new(class)
	for k, v in pairs(props) do o[k] = v end
	o.Parent = parent
	return o
end

local function corner(o, r)
	make("UICorner", {CornerRadius = UDim.new(0, r)}, o)
end

local MainFrame = make("Frame", {
	BackgroundColor3 = Color3.fromRGB(15, 12, 25),
	BorderSizePixel = 0,
	Position = UDim2.new(0.5, -135, 0.5, -146),
	Size = UDim2.new(0, 270, 0, 292),
	Active = true,
	ClipsDescendants = true,
}, ScreenGui)
corner(MainFrame, 12)
make("UIStroke", {Color = Color3.fromRGB(60, 50, 90), Thickness = 1.5}, MainFrame)

local Header = make("Frame", {
	BackgroundTransparency = 1,
	Size = UDim2.new(1, 0, 0, 48),
	Active = true,
}, MainFrame)

make("TextLabel", {
	Position = UDim2.new(0, 15, 0, 12),
	Size = UDim2.new(0, 180, 0, 18),
	BackgroundTransparency = 1,
	Font = Enum.Font.FredokaOne,
	Text = "PRIVATE SERVER FINDER",
	TextColor3 = COLORS.cyan,
	TextSize = 13,
	TextXAlignment = Enum.TextXAlignment.Left,
}, Header)

make("TextLabel", {
	Position = UDim2.new(0, 15, 0, 30),
	Size = UDim2.new(0, 180, 0, 14),
	BackgroundTransparency = 1,
	Font = Enum.Font.GothamMedium,
	Text = "v2 - upgraded from GlazeOnTop",
	TextColor3 = Color3.fromRGB(120, 120, 150),
	TextSize = 10,
	TextXAlignment = Enum.TextXAlignment.Left,
}, Header)

local CloseBtn = make("TextButton", {
	BackgroundColor3 = Color3.fromRGB(35, 30, 50),
	Position = UDim2.new(1, -32, 0, 12),
	Size = UDim2.new(0, 22, 0, 22),
	Font = Enum.Font.GothamBold,
	Text = "X",
	TextColor3 = Color3.fromRGB(200, 200, 220),
	TextSize = 11,
	ZIndex = 3,
}, MainFrame)
corner(CloseBtn, 6)

local MinBtn = make("TextButton", {
	BackgroundColor3 = Color3.fromRGB(35, 30, 50),
	Position = UDim2.new(1, -58, 0, 12),
	Size = UDim2.new(0, 22, 0, 22),
	Font = Enum.Font.GothamBold,
	Text = "-",
	TextColor3 = Color3.fromRGB(200, 200, 220),
	TextSize = 14,
	ZIndex = 3,
}, MainFrame)
corner(MinBtn, 6)

local StatusCard = make("Frame", {
	Position = UDim2.new(0, 15, 0, 52),
	Size = UDim2.new(1, -30, 0, 76),
	BackgroundColor3 = Color3.fromRGB(24, 20, 40),
	BorderSizePixel = 0,
}, MainFrame)
corner(StatusCard, 8)

local NowLabel = make("TextLabel", {
	Position = UDim2.new(0, 10, 0, 6),
	Size = UDim2.new(1, -20, 0, 18),
	BackgroundTransparency = 1,
	Font = Enum.Font.GothamBold,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextColor3 = COLORS.cyan,
	TextSize = 12,
	Text = "",
}, StatusCard)

local StatusLabel = make("TextLabel", {
	Position = UDim2.new(0, 10, 0, 24),
	Size = UDim2.new(1, -20, 0, 30),
	BackgroundTransparency = 1,
	Font = Enum.Font.GothamMedium,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextColor3 = COLORS.grey,
	TextSize = 10,
	TextWrapped = true,
	Text = "Set max players & click Hop.",
}, StatusCard)

local StatLabel = make("TextLabel", {
	Position = UDim2.new(0, 10, 0, 56),
	Size = UDim2.new(1, -20, 0, 14),
	BackgroundTransparency = 1,
	Font = Enum.Font.GothamMedium,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextColor3 = Color3.fromRGB(110, 110, 140),
	TextSize = 9,
	Text = "Scanned: 0 | Tried: 0",
}, StatusCard)

local MaxBox = make("TextBox", {
	Position = UDim2.new(0, 15, 0, 136),
	Size = UDim2.new(1, -30, 0, 32),
	BackgroundColor3 = Color3.fromRGB(24, 20, 40),
	BorderSizePixel = 0,
	Font = Enum.Font.GothamBold,
	Text = "5",
	PlaceholderText = "Max players (including you)",
	TextColor3 = Color3.new(1, 1, 1),
	TextSize = 13,
	ClearTextOnFocus = false,
}, MainFrame)
corner(MaxBox, 8)

local JoinBtn = make("TextButton", {
	Position = UDim2.new(0, 15, 0, 176),
	Size = UDim2.new(1, -30, 0, 34),
	BackgroundColor3 = Color3.fromRGB(0, 140, 255),
	BorderSizePixel = 0,
	Font = Enum.Font.GothamBold,
	Text = "HOP SERVER NOW",
	TextColor3 = Color3.new(1, 1, 1),
	TextSize = 11,
}, MainFrame)
corner(JoinBtn, 8)

local AutoBtn = make("TextButton", {
	Position = UDim2.new(0, 15, 0, 216),
	Size = UDim2.new(1, -30, 0, 34),
	BackgroundColor3 = Color3.fromRGB(35, 30, 50),
	BorderSizePixel = 0,
	Font = Enum.Font.GothamBold,
	Text = "AUTO HOP: DISABLED",
	TextColor3 = Color3.fromRGB(180, 180, 200),
	TextSize = 11,
}, MainFrame)
corner(AutoBtn, 8)

local ResetBtn = make("TextButton", {
	Position = UDim2.new(0, 15, 0, 256),
	Size = UDim2.new(1, -30, 0, 26),
	BackgroundColor3 = Color3.fromRGB(24, 20, 40),
	BorderSizePixel = 0,
	Font = Enum.Font.GothamMedium,
	Text = "RESET TRIED LIST",
	TextColor3 = Color3.fromRGB(150, 150, 175),
	TextSize = 10,
}, MainFrame)
corner(ResetBtn, 8)

-- Nút nổi để ẩn/hiện
local Orb = make("TextButton", {
	Size = UDim2.fromOffset(46, 46),
	Position = UDim2.new(0, 15, 0.5, 0),
	BackgroundColor3 = Color3.fromRGB(15, 12, 25),
	Font = Enum.Font.GothamBold,
	Text = "PS",
	TextColor3 = COLORS.cyan,
	TextSize = 13,
	Active = true,
}, ScreenGui)
corner(Orb, 23)
make("UIStroke", {Color = COLORS.cyan, Thickness = 1.5}, Orb)

-- Kéo thả (chuột + cảm ứng)
local function makeDraggable(handle, target)
	local state = {moved = false}
	local dragging, dragStart, startPos = false, nil, nil
	track(handle.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
			or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			state.moved = false
			dragStart = input.Position
			startPos = target.Position
			local c
			c = input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
					c:Disconnect()
				end
			end)
		end
	end))
	track(UIS.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
			or input.UserInputType == Enum.UserInputType.Touch) then
			local d = input.Position - dragStart
			if d.Magnitude > 6 then state.moved = true end
			target.Position = UDim2.new(
				startPos.X.Scale, startPos.X.Offset + d.X,
				startPos.Y.Scale, startPos.Y.Offset + d.Y
			)
		end
	end))
	return state
end

makeDraggable(Header, MainFrame)
local orbDrag = makeDraggable(Orb, Orb)

local function addHover(btn, normal, hover)
	track(btn.MouseEnter:Connect(function()
		TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = hover}):Play()
	end))
	track(btn.MouseLeave:Connect(function()
		TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = normal}):Play()
	end))
end
addHover(JoinBtn, Color3.fromRGB(0, 140, 255), Color3.fromRGB(30, 160, 255))
addHover(CloseBtn, Color3.fromRGB(35, 30, 50), Color3.fromRGB(220, 50, 70))
addHover(MinBtn, Color3.fromRGB(35, 30, 50), Color3.fromRGB(55, 50, 80))
addHover(ResetBtn, Color3.fromRGB(24, 20, 40), Color3.fromRGB(40, 34, 64))

-- ==============================
-- TIỆN ÍCH
-- ==============================
local function setStatus(text, color)
	if not alive then return end
	StatusLabel.Text = text
	StatusLabel.TextColor3 = color or COLORS.grey
end

local function updateStats(scanned)
	local n = 0
	for _ in pairs(tried) do n = n + 1 end
	StatLabel.Text = ("Scanned: %d | Tried: %d"):format(scanned or 0, n)
end

local function getThreshold()
	return math.max(1, math.floor(tonumber(MaxBox.Text) or 1))
end

local function httpGet(url)
	local ok, body = pcall(function() return game:HttpGet(url) end)
	if ok and type(body) == "string" and body ~= "" then return body end
	local req = (syn and syn.request) or (http and http.request) or http_request or request
		or (fluxus and fluxus.request)
	if req then
		local ok2, res = pcall(req, {Url = url, Method = "GET"})
		if ok2 and res and type(res.Body) == "string" and res.Body ~= ""
			and (res.StatusCode == nil or res.StatusCode == 200) then
			return res.Body
		end
	end
	return nil
end

-- Lấy danh sách server đích. Trả về: danh sách id, có phải fallback không, số server đã quét
local function fetchServers(threshold)
	local pool, cursor, scanned = {}, "", 0
	for page = 1, CONFIG.Pages do
		if not alive then break end
		setStatus(("Scanning servers... page %d/%d"):format(page, CONFIG.Pages), COLORS.amber)
		local url = "https://games.roblox.com/v1/games/" .. PlaceId
			.. "/servers/Public?sortOrder=Asc&excludeFullGames=true&limit=100"
			.. (cursor ~= "" and ("&cursor=" .. cursor) or "")
		local body
		for attempt = 1, 3 do
			body = httpGet(url)
			if body then break end
			task.wait(1.5 * attempt) -- có thể bị rate limit (429)
		end
		if not body then break end
		local ok, data = pcall(HttpService.JSONDecode, HttpService, body)
		if not ok or type(data) ~= "table" or type(data.data) ~= "table" then break end

		for _, s in ipairs(data.data) do
			scanned = scanned + 1
			if s.id and s.id ~= game.JobId and not tried[s.id]
				and s.playing and s.maxPlayers
				and (s.maxPlayers - s.playing) >= CONFIG.MinFree then
				table.insert(pool, {id = s.id, playing = s.playing})
			end
		end
		cursor = data.nextPageCursor or ""
		if cursor == "" then break end
		task.wait(0.3)
	end
	updateStats(scanned)

	-- server đích sau khi vào sẽ có (playing + 1) người
	local good = {}
	for _, s in ipairs(pool) do
		if s.playing + 1 <= threshold then table.insert(good, s) end
	end

	local source, fallback = good, false
	if #good == 0 then
		table.sort(pool, function(a, b) return a.playing < b.playing end)
		source, fallback = {}, true
		for i = 1, math.min(15, #pool) do source[i] = pool[i] end
	end

	local ids = {}
	for _, s in ipairs(source) do table.insert(ids, s.id) end
	for i = #ids, 2, -1 do
		local j = math.random(i)
		ids[i], ids[j] = ids[j], ids[i]
	end
	return ids, fallback, scanned
end

track(TeleportService.TeleportInitFailed:Connect(function(player, result, msg)
	if player ~= LocalPlayer then return end
	teleporting = false
	local m = tostring(msg or "")
	if result == Enum.TeleportResult.Flooded or m:find("772") then
		flood = true
	end
	setStatus("Teleport failed: " .. tostring(result), COLORS.red)
end))

local function teleportTo(id)
	tried[id] = true
	teleporting = true
	flood = false
	local ok = pcall(TeleportService.TeleportToPlaceInstance, TeleportService, PlaceId, id, LocalPlayer)
	if not ok then
		teleporting = false
		return
	end
	local t = 0
	while teleporting and alive and t < CONFIG.TeleportTimeout do
		task.wait(0.25)
		t = t + 0.25
	end
end

-- Tự chạy lại script sau khi sang server mới (cần executor có queue_on_teleport)
local function queueAuto()
	if queued or CONFIG.ScriptUrl == "" then return end
	local q = queue_on_teleport or (syn and syn.queue_on_teleport) or (fluxus and fluxus.queue_on_teleport)
	if not q then return end
	local src = ("getgenv().PSF_AUTO=true;getgenv().PSF_MAX=%d;loadstring(game:HttpGet(%q))()")
		:format(getThreshold(), CONFIG.ScriptUrl)
	if pcall(q, src) then queued = true end
end

-- ==============================
-- LOGIC HOP
-- ==============================
local function runHop(shouldContinue)
	if busy then
		setStatus("Already hopping, please wait...", COLORS.amber)
		return
	end
	busy = true
	local ok, err = pcall(function()
		local threshold = getThreshold()
		local current = #Players:GetPlayers()
		if current <= threshold then
			setStatus(("Server OK (%d <= %d). Staying here."):format(current, threshold), COLORS.green)
			return
		end

		local list, fallback = fetchServers(threshold)
		if #list == 0 then
			tried = {}
			updateStats(0)
			setStatus("No server found. Tried list reset, retrying...", COLORS.red)
			task.wait(2)
			return
		end

		if fallback then
			setStatus(("None <= %d players. Trying the emptiest servers."):format(threshold), COLORS.amber)
			task.wait(1)
		end

		queueAuto()
		for i, id in ipairs(list) do
			if not (alive and shouldContinue()) then break end
			setStatus(("Teleporting %d/%d..."):format(i, #list), COLORS.cyan)
			updateStats()
			teleportTo(id)
			if flood then
				setStatus("Error 772 (flooded). Cooling down...", COLORS.amber)
				task.wait(CONFIG.FloodCooldown)
			else
				task.wait(1)
			end
		end
	end)
	busy = false
	if not ok then setStatus("Error: " .. tostring(err), COLORS.red) end
end

local function setAuto(on)
	autoEnabled = on
	if on then
		AutoBtn.Text = "AUTO HOP: ENABLED"
		AutoBtn.BackgroundColor3 = Color3.fromRGB(0, 180, 120)
		AutoBtn.TextColor3 = Color3.new(1, 1, 1)
		task.spawn(function()
			while autoEnabled and alive do
				if #Players:GetPlayers() <= getThreshold() then
					setStatus("Optimal server. Monitoring...", COLORS.green)
					task.wait(3)
				else
					runHop(function() return autoEnabled end)
					task.wait(2)
				end
			end
		end)
	else
		AutoBtn.Text = "AUTO HOP: DISABLED"
		AutoBtn.BackgroundColor3 = Color3.fromRGB(35, 30, 50)
		AutoBtn.TextColor3 = Color3.fromRGB(180, 180, 200)
		setStatus("Auto hop stopped.", COLORS.grey)
	end
end

-- ==============================
-- SỰ KIỆN
-- ==============================
local function toggle() MainFrame.Visible = not MainFrame.Visible end

local function shutdown()
	alive = false
	autoEnabled = false
	for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
	pcall(function() ScreenGui:Destroy() end)
end
env.PSF_Stop = shutdown

track(CloseBtn.MouseButton1Click:Connect(shutdown))
track(MinBtn.MouseButton1Click:Connect(function() MainFrame.Visible = false end))
track(Orb.Activated:Connect(function()
	if not orbDrag.moved then toggle() end
end))
track(UIS.InputBegan:Connect(function(i, gp)
	if not gp and i.KeyCode == Enum.KeyCode.RightShift then toggle() end
end))

track(JoinBtn.MouseButton1Click:Connect(function()
	task.spawn(runHop, function() return true end)
end))

track(AutoBtn.MouseButton1Click:Connect(function()
	setAuto(not autoEnabled)
end))

track(ResetBtn.MouseButton1Click:Connect(function()
	tried = {}
	updateStats(0)
	setStatus("Tried list cleared.", COLORS.grey)
end))

task.spawn(function()
	while alive do
		NowLabel.Text = "Current Server: " .. #Players:GetPlayers() .. " player(s)"
		task.wait(1)
	end
end)

-- Khôi phục cài đặt sau khi được queue_on_teleport chạy lại
if env.PSF_MAX then
	MaxBox.Text = tostring(env.PSF_MAX)
	env.PSF_MAX = nil
end
if env.PSF_AUTO then
	env.PSF_AUTO = nil
	task.spawn(function()
		if not game:IsLoaded() then game.Loaded:Wait() end
		task.wait(3)
		if alive then setAuto(true) end
	end)
end
