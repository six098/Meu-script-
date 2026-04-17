```lua
--[[
Source script taken from: https://github.com/Roblox/creator-docs/blob/main/content/en-us/characters/emotes.md

scriptblox: https://scriptblox.com/script/Universal-Script-7yd7-I-Emote-Script-48024

]]

if _G.EmotesGUIRunning then
getgenv().Notify({
Title = '7yd7 | Emote',
Content = '⚠️ It works It actually works',
Duration = 5
})
return
end
_G.EmotesGUIRunning = true

local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local GuiService = game:GetService("GuiService")
local ContentProvider = game:GetService("ContentProvider")
local StarterGui = game:GetService("StarterGui")
local request = http_request or (syn and syn.request) or request

local State = {
currentMode = "emote",
emotesWalkEnabled = false,
favoriteEnabled = false,
hudEditorActive = false,
speedEmoteEnabled = false,
isLoading = false,
favoriteSetVersion = 0,
favoriteSetBuiltVersion = -1,
emoteCacheVersion = 0,
animationCacheVersion = 0,
isGUICreated = false,
isMonitoringClicks = false,
lastRadialActionTime = 0,
lastWheelVisibleTime = 0,
lastActionTick = 0,
lastRandomEmoteId = nil,
lastRandomAnimationId = nil,
lastRandomVisualSpam = 0,
randomSpamConn = nil,
animImageSpamConn = nil,
animImageSpamMap = nil,
animImageSpamTicks = nil,
animImageSpamToken = 0,
animImageRetry = 0,
randomSlotBlockerConn = nil,
totalEmotesLoaded = 0,
currentPage = 1,
totalPages = 1,
itemsPerPage = 8,
emoteSearchTerm = "",
animationSearchTerm = "",
currentEmoteTrack = nil,
currentCharacter = nil,
emoteClickConnections = {},
guiConnections = {},
animationsData = {},
originalAnimationsData = {},
filteredAnimations = {},
favoriteAnimations = {},
favoriteAnimationsFileName = "FavoriteAnimations.json",
emotesData = {},
originalEmotesData = {},
filteredEmotes = {},
scannedEmotes = {},
favoriteEmotes = {},
favoriteFileName = "FavoriteEmotes.json",
speedEmoteConfigFile = "SpeedEmoteConfig.json",
favoriteEmoteSet = {},
favoriteAnimationSet = {},
emotePageCache = { version = nil, normal = {}, favorites = {} },
animationPageCache = { version = nil, normal = {}, favorites = {} },
suppressSearch = false,
emoteMonitorToken = 0,
animationMonitorToken = 0,
imageUpdateToken = 0,
defaultButtonImage = "rbxassetid://71408678974152",
enabledButtonImage = "rbxassetid://106798555684020",
favoriteIconId = "rbxassetid://97307461910825",
notFavoriteIconId = "rbxassetid://124025954365505",
EmoteTheme = nil,
isApplyingTheme = false,
targetImages = {},
AnimationCachePath = "7yd7/AnimationCache.json",
AnimationCache = {},
AnimationListCachePath = "7yd7/AnimationListCache.json",
EmoteListCachePath = "7yd7/EmoteListCache.json",
CustomAnimationPath = "7yd7/CustomAnimations.json",
CustomAnimations = {},
currentCustomAnimationName = "Default",
customAnimationEditorActive = false,
customAnimationEditingKey = nil,
customAnimationEditingName = nil
}

Config = {
NotifyEnabled = true,
SearchVisible = true,
FavVisible = true,
ModeVisible = true,
FreezeVisible = true,
SpeedVisible = true,
NavVisible = true,
EmoteSpeed = 1,
EmoteSpeedEnabled = false,
SelectedTheme = "Default",
EmotePage = 1,
AnimationPage = 1,
RandomEnabled = true,
RandomMode = "All",
AuthenticFirstPage = false,
HUDPositions = {},
HUDSizes = {},
HUDProperties = {},
CustomFrames = {},
AutoReloadEnabled = false,
LastPlayedAnimationData = nil
}

HUD = {
Connections = {},
IsUnlocked = false,
DefaultPositions = {},
DefaultSizes = {},
DefaultTexts = {},
DefaultPlaceholders = {},
Layouts = {},
LayoutsRemoved = {},
SelectionGui = nil,
SelectedElement = nil,
ResizeHandles = {},
ResizeConnections = {},
FriendlyNames = {
["Under.1left"] = "PrevPage",
["Under.9right"] = "NextPage",
["Under.4pages"] = "TotalPages",
["Under.3TextLabel"] = "Divider",
["Under.2Route-number"] = "CurrentPage",
["Top.Search"] = "Search",
["EmoteWalkButton"] = "Freeze",
["Favorite"] = "Favorite",
["SpeedEmote"] = "SpeedEmote",
["SpeedBox"] = "SpeedBox",
["Changepage"] = "ChangePage",
["Reload"] = "AutoReload",
["Top"] = "Top",
["Under"] = "Under"
}
}

function loadAnimationCache()
if isfile and isfile(State.AnimationCachePath) then
local success, decoded = pcall(function()
return HttpService:JSONDecode(readfile(State.AnimationCachePath))
end)
if success and type(decoded) == "table" then
State.AnimationCache = decoded
end
end
end

function saveAnimationCache()
if writefile then
pcall(function()
if not isfolder("7yd7") then makefolder("7yd7") end
writefile(State.AnimationCachePath, HttpService:JSONEncode(State.AnimationCache))
end)
end
end

function resolveAnimationMappings(bundledItems)
local mappings = {}
for _, assetIds in pairs(bundledItems) do
for _, assetId in pairs(assetIds) do
local success, objects = pcall(function()
return game:GetObjects("rbxassetid://" .. assetId)
end)
if success and objects then
local function searchTree(parent, parentPath)
for _, child in pairs(parent:GetChildren()) do
if child:IsA("Animation") then
local animationPath = parentPath .. "." .. child.Name
local pathParts = animationPath:split(".")
table.insert(mappings, {
category = pathParts[#pathParts - 1],
name = pathParts[#pathParts],
animationId = child.AnimationId
})
elseif #child:GetChildren() > 0 then
searchTree(child, parentPath .. "." .. child.Name)
end
end
end
for _, obj in pairs(objects) do
searchTree(obj, obj.Name)
obj.Parent = workspace
task.delay(1, function()
if obj then obj:Destroy() end
end)
end
end
end
end
return mappings
end

function buildCustomSetMappings(setName)
if type(setName) == "string" then
setName = setName:gsub("%s*%-.*$", "")
end
local set = State.CustomAnimations and State.CustomAnimations.Sets and State.CustomAnimations.Sets[setName]
if not set then return {} end
local mappings = {}
for cat, anims in pairs(set) do
if cat ~= "__meta" then
for name, id in pairs(anims) do
if tostring(id) ~= "0" then
table.insert(mappings, {category = cat, name = name, animationId = "rbxassetid://" .. id})
end
end
end
end
return mappings
end

loadAnimationCache()

local UI = {
CustomFrames = {},
Under = nil,
_1left = nil,
_9right = nil,
_4pages = nil,
_3TextLabel = nil,
_2Routenumber = nil,
Top = nil,
EmoteWalkButton = nil,
Search = nil,
Favorite = nil,
SpeedEmote = nil,
SpeedBox = nil,
Changepage = nil,
Reload = nil,
Background = nil
}

local HUD = {
Connections = {},
Strokes = {},
ResizeHandles = {},
ResizeConnections = {},
UndoStack = {},
SelectedElement = nil,
Overlay = nil,
IsUnlocked = false,
ForceVisibleConn = nil,
Layouts = {},
LayoutsRemoved = {},
FriendlyNames = {
["Under.1left"] = "Left Arrow",
["Under.9right"] = "Right Arrow",
["Under.4pages"] = "Total Pages",
["Under.3TextLabel"] = "Separator Label",
["Under.2Route-number"] = "Page Number Box",
["Top.Search"] = "Search/ID Box",
},
DefaultPositions = {
Top = UDim2.new(0.127499998, 0, -0.109999999, 0),
Under = UDim2.new(0.129999995, 0, 1, 0),
EmoteWalkButton = UDim2.new(0.889999986, 0, -0.107500002, 0),
Favorite = UDim2.new(0.0189999994, 0, -0.108000003, 0),
SpeedEmote = UDim2.new(0.888999999, 0, 0, 0),
SpeedBox = UDim2.new(0.0189999398, 0, -0.000499992399, 0),
Changepage = UDim2.new(0.019, 0, 1.021, 0),
Reload = UDim2.new(0.888999999, 0, 1.02100003, 0),
["Left Arrow"] = UDim2.new(0, 0, 0.028, 0),
["Right Arrow"] = UDim2.new(0.169, 0, 0.028, 0),
["Total Pages"] = UDim2.new(0.339, 0, 0.094, 0),
["Separator Label"] = UDim2.new(0.498, 0, 0.028, 0),
["Page Number Box"] = UDim2.new(0.837, 0, 0.094, 0),
["Search/ID Box"] = UDim2.new(0.01, 0, 0.092, 0),
},
DefaultSizes = {
Top = UDim2.new(0.737500012, 0, 0.0949999914, 0),
Under = UDim2.new(0.737500012, 0, 0.132499993, 0),
EmoteWalkButton = UDim2.new(0.0874999985, 0, 0.0874999985, 0),
Favorite = UDim2.new(0.0874999985, 0, 0.0874999985, 0),
SpeedEmote = UDim2.new(0.0874999985, 0, 0.0874999985, 0),
SpeedBox = UDim2.new(0.0874999985, 0, 0.0874999985, 0),
Changepage = UDim2.new(0.087, 0, 0.087, 0),
Reload = UDim2.new(0.0869999975, 0, 0.0869999975, 0),
["Left Arrow"] = UDim2.new(0.169491529, 0, 0.94339627, 0),
["Right Arrow"] = UDim2.new(0.169491529, 0, 0.94339627, 0),
["Total Pages"] = UDim2.new(0.159322038, 0, 0.811320841, 0),
["Separator Label"] = UDim2.new(0.338983059, 0, 0.94339627, 0),
["Page Number Box"] = UDim2.new(0.159322038, 0, 0.811320841, 0),
["Search/ID Box"] = UDim2.new(0.864406765, 0, 0.81578958, 0),
},
DefaultTexts = {
["Left Arrow"] = "",
["Right Arrow"] = "",
["Total Pages"] = "1",
["Separator Label"] = " ------ ",
["Page Number Box"] = "1",
["Search/ID Box"] = "",
["SpeedBox"] = "1",
},
DefaultPlaceholders = {
["Search/ID Box"] = "Search/ID",
}
}

function getAllHUDObjects()
local elems = {}
if UI.Top then elems["Top"] = UI.Top end
if UI.Under then elems["Under"] = UI.Under end
if UI.EmoteWalkButton then elems["EmoteWalkButton"] = UI.EmoteWalkButton end
if UI.Favorite then elems["Favorite"] = UI.Favorite end
if UI.SpeedEmote then elems["SpeedEmote"] = UI.SpeedEmote end
if UI.SpeedBox then elems["SpeedBox"] = UI.SpeedBox end
if UI.Changepage then elems["Changepage"] = UI.Changepage end
if UI.Reload then elems["Reload"] = UI.Reload end
if UI.CustomFrames then
for n, f in pairs(UI.CustomFrames) do
elems[n] = f
end
end
if UI.Top then
for _, child in pairs(UI.Top:GetChildren()) do
if child:IsA("GuiObject") and not child:IsA("UIListLayout") and not child:IsA("UICorner") then
local internalName = "Top." .. child.Name
elems[HUD.FriendlyNames[internalName] or internalName] = child
end
end
end
if UI.Under then
for _, child in pairs(UI.Under:GetChildren()) do
if child:IsA("GuiObject") and not child:IsA("UIListLayout") and not child:IsA("UICorner") then
local internalName = "Under." .. child.Name
elems[HUD.FriendlyNames[internalName] or internalName] = child
end
end
end
return elems
end

function getMovableElements()
local all = getAllHUDObjects()
local movable = {}
for name, el in pairs(all) do
local isChild = false
for _, friendly in pairs(HUD.FriendlyNames) do
if name == friendly then isChild = true; break end
end
if not isChild or HUD.IsUnlocked then
movable[name] = el
end
end
return movable
end

function ColorToTable(c) return {math.round(c.R*255), math.round(c.G*255), math.round(c.B*255)} end
function TableToColor(t)
if type(t) ~= "table" then
return Color3.fromRGB(255, 255, 255)
end
local r = tonumber(t[1]) or 255
local g = tonumber(t[2]) or 255
local b = tonumber(t[3]) or 255
return Color3.fromRGB(r, g, b)
end

local function isThemeDefaultRGB(r, g, b)
return r == 28 and g == 30 and b == 32
end

local AnimationSystem = {
Cache = {},
currentThemeName = "Default"
}

AnimationSystem.LooksLikeGif = function(url)
if not url then return false end
url = string.lower(tostring(url))
return url:find(".gif") or url:find("gif") or url:find("format=gif") or url:find("image/gif")
end

AnimationSystem.NormalizeUrl = function(url)
if not url or url == "" then return url end
local targetUrl = tostring(url)
targetUrl = targetUrl:gsub("%?raw=true", "")
if targetUrl:find("github.com") then
targetUrl = targetUrl:gsub("github.com", "raw.githubusercontent.com")
targetUrl = targetUrl:gsub("/blob/", "/")
targetUrl = targetUrl:gsub("/raw/", "/")
end
if targetUrl:find(" ") and not targetUrl:find("%%20") then
targetUrl = targetUrl:gsub(" ", "%%20")
end
if not targetUrl:find("://") then
local id = targetUrl:match("id=(%d+)") or targetUrl:match("^(%d+)$")
if id then return "rbxassetid://" .. id end
end
return targetUrl
end

AnimationSystem.ParseGifInfo = function(bytes)
if not bytes or #bytes < 13 then return nil end
if bytes:sub(1, 3) ~= "GIF" then return nil end
local function u16le(pos)
local b1 = bytes:byte(pos) or 0
local b2 = bytes:byte(pos + 1) or 0
return b1 + b2 * 256
end
local width = u16le(7)
local height = u16le(9)
local packed = bytes:byte(11) or 0
local gctFlag = bit32.band(packed, 0x80) ~= 0
local gctSize = bit32.band(packed, 0x07)
local offset = 13
if gctFlag then
offset = offset + (3 * (2 ^ (gctSize + 1)))
end
local frames = 0
local delays = {}
local pendingDelay = nil
local function skipSubBlocks(pos)
while pos <= #bytes do
local size = bytes:byte(pos) or 0
pos = pos + 1
if size == 0 then break end
pos = pos + size
end
return pos
end
while offset <= #bytes do
local b = bytes:byte(offset)
if not b then break end
if b == 0x3B then
break
elseif b == 0x21 then
local label = bytes:byte(offset + 1) or 0
if label == 0xF9 then
local delay = u16le(offset + 4)
pendingDelay = delay
offset = offset + 8
else
offset = skipSubBlocks(offset + 2)
end
elseif b == 0x2C then
frames = frames + 1
if pendingDelay then
table.insert(delays, pendingDelay)
pendingDelay = nil
end
local packedImg = bytes:byte(offset + 9) or 0
local lctFlag = bit32.band(packedImg, 0x80) ~= 0
local lctSize = bit32.band(packedImg, 0x07)
offset = offset + 10
if lctFlag then
offset = offset + (3 * (2 ^ (lctSize + 1)))
end
offset = offset + 1
offset = skipSubBlocks(offset)
else
offset = offset + 1
end
end
local totalDelay = 0
for _, d in ipairs(delays) do totalDelay = totalDelay + d end
local avgDelay = (#delays > 0) and (totalDelay / #delays) or 10
return {
width = width,
height = height,
frames = frames > 0 and frames or #delays,
totalDelayCs = totalDelay,
avgDelayCs = avgDelay
}
end

AnimationSystem.ParsePngInfo = function(bytes)
if not bytes or #bytes < 24 then return nil end
if bytes:sub(1, 8) ~= "\137PNG\r\n\26\n" then return nil end
local function u32be(pos)
local b1 = bytes:byte(pos) or 0
local b2 = bytes:byte(pos + 1) or 0
local b3 = bytes:byte(pos + 2) or 0
local b4 = bytes:byte(pos + 3) or 0
return ((b1 * 256 + b2) * 256 + b3) * 256 + b4
end
local width = u32be(17)
local height = u32be(21)
if width <= 0 or height <= 0 then return nil end
return { width = width, height = height }
end

AnimationSystem.StopGif = function()
if State.currentWheelAnimToken then
State.currentWheelAnimToken = State.currentWheelAnimToken + 1
end
end

AnimationSystem.SetImageMode = function(img, custom)
if not img then return end
if custom then
img.ScaleType = Enum.ScaleType.Stretch
img.SliceCenter = Rect.new(0, 0, 0, 0)
img.SliceScale = 1
else
img.ScaleType = Enum.ScaleType.Fit
end
end

AnimationSystem.StartGif = function(img, data)
AnimationSystem.StopGif()
if not img or not data or not data.sprite then return end
State.currentWheelAnimToken = (State.currentWheelAnimToken or 0) + 1
local token = State.currentWheelAnimToken
local frames = data.frames or 1
local frameW = data.frameW or 0
local frameH = data.frameH or 0
local cols = data.cols or 1
local delay = data.delay or 0.1
img.Image = data.sprite
img.ImageRectSize = Vector2.new(frameW, frameH)
local current = 0
local acc = 0
local connection
connection = RunService.Heartbeat:Connect(function(dt)
if token ~= State.currentWheelAnimToken then
connection:Disconnect()
return
end
acc = acc + dt
if acc < delay then return end
acc = 0
current = (current + 1) % frames
local col = current % cols
local row = math.floor(current / cols)
img.ImageRectOffset = Vector2.new(col * frameW, row * frameH)
end)
end

AnimationSystem.AreMetaEqual = function(a, b)
if not a or not b then return a == b end
return a.GifUrl == b.GifUrl and a.SheetUrl == b.SheetUrl and a.Enabled == b.Enabled
end

AnimationSystem.MakeKey = function(gif, sheet)
return tostring(gif) .. "|" .. tostring(sheet)
end

AnimationSystem.GetIconColor = function(key)
if themes and themes[AnimationSystem.currentThemeName] then
local theme = themes[AnimationSystem.currentThemeName]
if theme.IconColors and theme.IconColors[key] then
return TableToColor(theme.IconColors[key])
end
return TableToColor(theme.ImageColor or {255, 255, 255})
elseif State.EmoteTheme then
local theme = State.EmoteTheme
if theme.IconColors and theme.IconColors[key] then
return TableToColor(theme.IconColors[key])
end
return theme.ImageColor or Color3.new(1, 1, 1)
end
return Color3.fromRGB(255, 255, 255)
end

AnimationSystem.ResetRandomSlot = function(frontFrame)
if not frontFrame then return end
local slot = frontFrame:FindFirstChild("1")
if slot and slot:IsA("ImageLabel") then
slot.ImageColor3 = Color3.fromRGB(255, 255, 255)
slot.Image = ""
local idValue = slot:FindFirstChild("AnimationID")
if idValue then idValue:Destroy() end
end
end

-- SafeLoad corrigido
function SafeLoad(url, name)
local success, content
for i = 1, 3 do
success, content = pcall(function() return game:HttpGet(url) end)
if success and content and content ~= "" then break end
task.wait(1)
end
return success and content or nil
end

-- Botão de fechar (fora do SafeLoad)
task.spawn(function()
repeat task.wait() until UI and UI.Top

local CloseBtn = Instance.new("TextButton")
CloseBtn.Name = "CloseButton"
CloseBtn.Size = UDim2.new(0, 30, 1, 0)
CloseBtn.Position = UDim2.new(1, -35, 0, 0)
CloseBtn.Text = "X"
CloseBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.ZIndex = 999
CloseBtn.Parent = UI.Top

CloseBtn.MouseButton1Click:Connect(function()
pcall(function()
if State.currentEmoteTrack then State.currentEmoteTrack:Stop() end
end)
pcall(function()
for _, v in pairs(State.guiConnections) do
if typeof(v) == "RBXScriptConnection" then v:Disconnect() end
end
end)
pcall(function()
if State.randomSpamConn then State.randomSpamConn:Disconnect() end
if State.animImageSpamConn then State.animImageSpamConn:Disconnect() end
end)
pcall(function()
if UI.Background then UI.Background:Destroy()
elseif UI.Top then UI.Top.Parent:Destroy() end
end)
_G.EmotesGUIRunning = false
end)
end)
```
