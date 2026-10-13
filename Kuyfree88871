--[[
    Dumper (No CoreGui) | UI: WindUI
    - Dump All (รวมไฟล์เดียว) + แยก Tree / Remote / GUI / Script
    - Script ทั้งหมดถูกรวมเป็นข้อความเดียวกันในไฟล์เดียว
    - ไม่แตะ CoreGui (ทั้งการสแกนและการ dump)
    - ต้องใช้ executor ที่รองรับ: writefile, decompile (ถ้าต้องการ dump script)
    ใช้กับเกมของตัวเอง / เพื่อศึกษาเท่านั้น
]]

local WindUI = loadstring(game:HttpGet("https://github.com/Footagesus/WindUI/releases/latest/download/main.lua"))()

local Players = game:GetService("Players")
local MarketplaceService = game:GetService("MarketplaceService")
local LP = Players.LocalPlayer
local CoreGui = game:GetService("CoreGui")

local FOLDER = "Dumper"

----------------------------------------------------------------
-- Settings (ปรับผ่าน UI)
----------------------------------------------------------------
local ALL_SERVICES = {
    "Workspace", "ReplicatedStorage", "ReplicatedFirst", "StarterGui",
    "StarterPack", "StarterPlayer", "Players", "Lighting",
    "SoundService", "Chat", "Teams", "TextChatService",
}

local S = {
    Services = { "Workspace", "ReplicatedStorage", "ReplicatedFirst", "StarterGui", "StarterPack", "StarterPlayer", "Players" },
    -- ส่วนที่จะรวมใน Dump All
    PartTree = true,
    PartRemote = true,
    PartGui = true,
    PartScript = true,
    -- ตัวเลือก
    IncludeNil = false,
    IncludeBindable = false,
    GuiProps = true,
    Clipboard = false,
}

local busy = false

----------------------------------------------------------------
-- Helpers
----------------------------------------------------------------
local function notify(title, content, dur)
    WindUI:Notify({ Title = title, Content = content, Duration = dur or 4 })
end

local function inCoreGui(inst)
    local ok, res = pcall(function()
        return inst == CoreGui or inst:IsDescendantOf(CoreGui)
    end)
    return ok and res
end

local function getService(name)
    local ok, svc = pcall(game.GetService, game, name)
    return ok and svc or nil
end

local function getRoots()
    local roots = {}
    for _, name in ipairs(S.Services) do
        if name ~= "CoreGui" then
            local svc = getService(name)
            if svc then
                if name == "Players" then
                    if LP then roots[#roots + 1] = LP end -- เฉพาะ LocalPlayer
                else
                    roots[#roots + 1] = svc
                end
            end
        end
    end
    return roots
end

local function esc(s)
    return string.format("%q", s)
end

local function luaPath(inst)
    local chain, cur = {}, inst
    while cur and cur ~= game do
        table.insert(chain, 1, cur)
        cur = cur.Parent
    end
    if cur ~= game then
        return "nil --[[ " .. inst.Name .. " (nil parent) ]]"
    end
    local out
    local startIdx = 2
    if chain[1] == LP then
        out = 'game:GetService("Players").LocalPlayer'
    else
        out = 'game:GetService(' .. esc(chain[1].ClassName) .. ')'
    end
    for i = startIdx, #chain do
        out = out .. "[" .. esc(chain[i].Name) .. "]"
    end
    return out
end

local counter = 0
local function tick_yield()
    counter += 1
    if counter % 400 == 0 then task.wait() end
end

-- รวม instance ทั้งหมดตาม service ที่เลือก (ตัด CoreGui ออกเสมอ)
local function collect()
    local list, seen = {}, {}
    local function add(i)
        if not seen[i] and not inCoreGui(i) then
            seen[i] = true
            list[#list + 1] = i
        end
    end
    for _, root in ipairs(getRoots()) do
        add(root)
        local ok, desc = pcall(root.GetDescendants, root)
        if ok then
            for _, v in ipairs(desc) do
                add(v)
                tick_yield()
            end
        end
    end
    if S.IncludeNil and getnilinstances then
        local ok, nils = pcall(getnilinstances)
        if ok then
            for _, v in ipairs(nils) do
                add(v)
                tick_yield()
            end
        end
    end
    return list
end

----------------------------------------------------------------
-- Dumpers (แต่ละอันคืนค่าเป็น string เดียว)
----------------------------------------------------------------
local function dumpTree()
    local buf = {}
    local function walk(inst, depth)
        if inCoreGui(inst) then return end
        buf[#buf + 1] = string.rep("  ", depth) .. inst.Name .. " [" .. inst.ClassName .. "]"
        tick_yield()
        local ok, kids = pcall(inst.GetChildren, inst)
        if ok then
            for _, c in ipairs(kids) do walk(c, depth + 1) end
        end
    end
    for _, root in ipairs(getRoots()) do walk(root, 0) end
    return table.concat(buf, "\n")
end

local function dumpRemotes(instances)
    local buf, n = {}, 0
    for _, i in ipairs(instances) do
        local c = i.ClassName
        local line
        if c == "RemoteEvent" or c == "UnreliableRemoteEvent" then
            line = ("[%s] %s\n    %s:FireServer(...)"):format(c, i:GetFullName(), luaPath(i))
        elseif c == "RemoteFunction" then
            line = ("[%s] %s\n    %s:InvokeServer(...)"):format(c, i:GetFullName(), luaPath(i))
        elseif S.IncludeBindable and (c == "BindableEvent" or c == "BindableFunction") then
            line = ("[%s] %s\n    %s"):format(c, i:GetFullName(), luaPath(i))
        end
        if line then
            n += 1
            buf[#buf + 1] = line
        end
    end
    return ("-- Total: %d\n\n"):format(n) .. table.concat(buf, "\n\n")
end

local function readProp(inst, prop)
    local ok, v = pcall(function() return inst[prop] end)
    if ok then return tostring(v) end
end

local function dumpGui(instances)
    local buf, n = {}, 0
    for _, i in ipairs(instances) do
        if i:IsA("LayerCollector") or i:IsA("GuiObject") then
            n += 1
            local line = ("[%s] %s"):format(i.ClassName, i:GetFullName())
            if S.GuiProps then
                local props = {}
                for _, p in ipairs({ "Visible", "Enabled", "Text", "Image", "Position", "Size", "AnchorPoint", "ZIndex" }) do
                    local v = readProp(i, p)
                    if v ~= nil then props[#props + 1] = p .. "=" .. v end
                end
                if #props > 0 then
                    line = line .. "\n    " .. table.concat(props, " | ")
                end
            end
            buf[#buf + 1] = line
            tick_yield()
        end
    end
    return ("-- Total: %d\n\n"):format(n) .. table.concat(buf, "\n")
end

-- Script ทั้งหมดรวมเป็นข้อความเดียว
local function dumpScripts(instances)
    if not decompile then
        return "-- executor นี้ไม่รองรับ decompile"
    end
    local buf, ok_n, fail_n = {}, 0, 0
    for _, i in ipairs(instances) do
        if i:IsA("LocalScript") or i:IsA("ModuleScript") then
            local ok, src = pcall(decompile, i)
            local head = ("-- ===== [%s] %s ====="):format(i.ClassName, i:GetFullName())
            if ok and type(src) == "string" and #src > 0 then
                ok_n += 1
                buf[#buf + 1] = head .. "\n" .. src
            else
                fail_n += 1
                buf[#buf + 1] = head .. "\n-- (decompile ไม่สำเร็จ)"
            end
            task.wait()
        end
    end
    return ("-- Scripts OK: %d | Failed: %d\n\n"):format(ok_n, fail_n) .. table.concat(buf, "\n\n")
end

----------------------------------------------------------------
-- Save
----------------------------------------------------------------
local placeName = tostring(game.PlaceId)
pcall(function()
    placeName = MarketplaceService:GetProductInfo(game.PlaceId).Name
end)
placeName = placeName:gsub("[^%w_%-]", "_")

local function save(kind, text)
    if not writefile then
        notify("Error", "executor ไม่รองรับ writefile")
        return
    end
    if makefolder and not (isfolder and isfolder(FOLDER)) then
        pcall(makefolder, FOLDER)
    end
    local path = ("%s/%s_%s_%s.txt"):format(FOLDER, kind, placeName, os.date("%Y%m%d_%H%M%S"))
    local ok, err = pcall(writefile, path, text)
    if ok then
        if S.Clipboard and setclipboard then pcall(setclipboard, text) end
        notify("Dump สำเร็จ", path)
    else
        notify("Save ล้มเหลว", tostring(err), 6)
    end
end

local function header()
    return table.concat({
        "-- Game: " .. placeName,
        "-- PlaceId: " .. game.PlaceId,
        "-- JobId: " .. game.JobId,
        "-- Services: " .. table.concat(S.Services, ", "),
        "-- CoreGui: excluded",
        "-- Date: " .. os.date("%Y-%m-%d %H:%M:%S"),
    }, "\n")
end

local function section(title, body)
    local bar = string.rep("=", 60)
    return ("%s\n%s\n%s\n%s"):format(bar, "## " .. title, bar, body)
end

local function run(kind)
    if busy then return notify("รอสักครู่", "กำลัง dump อยู่") end
    if #S.Services == 0 then return notify("ยังไม่ได้เลือก", "เลือก Service อย่างน้อย 1 อัน") end
    busy = true
    task.spawn(function()
        local ok, err = pcall(function()
            notify("เริ่ม Dump", kind)
            counter = 0
            local parts = { header() }

            local needInst = kind == "All" or kind == "Remote" or kind == "GUI" or kind == "Script"
            local instances = needInst and collect() or nil

            if kind == "Tree" or (kind == "All" and S.PartTree) then
                parts[#parts + 1] = section("TREE", dumpTree())
            end
            if kind == "Remote" or (kind == "All" and S.PartRemote) then
                parts[#parts + 1] = section("REMOTES", dumpRemotes(instances))
            end
            if kind == "GUI" or (kind == "All" and S.PartGui) then
                parts[#parts + 1] = section("GUI", dumpGui(instances))
            end
            if kind == "Script" or (kind == "All" and S.PartScript) then
                parts[#parts + 1] = section("SCRIPTS", dumpScripts(instances))
            end

            save(kind, table.concat(parts, "\n\n"))
        end)
        if not ok then notify("Error", tostring(err), 6) end
        busy = false
    end)
end

----------------------------------------------------------------
-- UI (WindUI)
----------------------------------------------------------------
local Window = WindUI:CreateWindow({
    Title = "Dumper",
    Icon = "file-down",
    Author = "No CoreGui",
    Folder = "DumperUI",
    Size = UDim2.fromOffset(560, 440),
    Theme = "Dark",
})

local TabDump = Window:Tab({ Title = "Dump", Icon = "download" })
local TabSet = Window:Tab({ Title = "Settings", Icon = "settings" })

-- ===== Dump tab =====
TabDump:Section({ Title = "รวมทั้งหมด (ไฟล์เดียว)" })

TabDump:Toggle({ Title = "Tree", Default = S.PartTree, Callback = function(v) S.PartTree = v end })
TabDump:Toggle({ Title = "Remote", Default = S.PartRemote, Callback = function(v) S.PartRemote = v end })
TabDump:Toggle({ Title = "GUI", Default = S.PartGui, Callback = function(v) S.PartGui = v end })
TabDump:Toggle({ Title = "Script (รวมเป็นข้อความเดียว)", Default = S.PartScript, Callback = function(v) S.PartScript = v end })

TabDump:Button({
    Title = "Dump All",
    Desc = "รวมส่วนที่เลือกไว้ด้านบนเป็นไฟล์เดียว",
    Callback = function() run("All") end,
})

TabDump:Section({ Title = "แยกทีละอย่าง" })

TabDump:Button({ Title = "Dump Tree", Desc = "โครงสร้าง instance", Callback = function() run("Tree") end })
TabDump:Button({ Title = "Dump Remote", Desc = "RemoteEvent / RemoteFunction", Callback = function() run("Remote") end })
TabDump:Button({ Title = "Dump GUI", Desc = "ScreenGui และ GuiObject (ไม่รวม CoreGui)", Callback = function() run("GUI") end })
TabDump:Button({ Title = "Dump Script", Desc = "LocalScript + ModuleScript รวมเป็นข้อความเดียว", Callback = function() run("Script") end })

-- ===== Settings tab =====
TabSet:Dropdown({
    Title = "Services ที่จะ Dump",
    Desc = "CoreGui ถูกตัดออกเสมอ",
    Values = ALL_SERVICES,
    Value = S.Services,
    Multi = true,
    AllowNone = true,
    Callback = function(opt)
        S.Services = type(opt) == "table" and opt or { opt }
    end,
})

TabSet:Toggle({ Title = "รวม Nil Instances", Desc = "ต้องรองรับ getnilinstances", Default = S.IncludeNil, Callback = function(v) S.IncludeNil = v end })
TabSet:Toggle({ Title = "รวม Bindable ใน Remote", Default = S.IncludeBindable, Callback = function(v) S.IncludeBindable = v end })
TabSet:Toggle({ Title = "บันทึก Properties ของ GUI", Default = S.GuiProps, Callback = function(v) S.GuiProps = v end })
TabSet:Toggle({ Title = "คัดลอกผลลัพธ์ไป Clipboard", Default = S.Clipboard, Callback = function(v) S.Clipboard = v end })

notify("Dumper", "พร้อมใช้งาน — ไฟล์เซฟที่โฟลเดอร์ workspace/" .. FOLDER)
