local player = game.Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local screenGui = Instance.new("ScreenGui")
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

local openBtn = Instance.new("TextButton")
openBtn.Size = UDim2.new(0,50,0,50)
openBtn.Position = UDim2.new(1,-60,0,10)
openBtn.Text = "LOG"
openBtn.Parent = screenGui

local main = Instance.new("Frame")
main.Size = UDim2.new(0,350,0,400)
main.Position = UDim2.new(0.5,-175,0.5,-200)
main.Visible = false
main.Parent = screenGui

local hideBtn = Instance.new("TextButton")
hideBtn.Size = UDim2.new(0,50,0,30)
hideBtn.Position = UDim2.new(1,-60,0,0)
hideBtn.Text = "X"
hideBtn.Parent = main

local list = Instance.new("ScrollingFrame")
list.Size = UDim2.new(1,0,1,-40)
list.Position = UDim2.new(0,0,0,40)
list.CanvasSize = UDim2.new(0,0,0,0)
list.Parent = main

local layout = Instance.new("UIListLayout")
layout.Parent = list

local function addLog(text)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1,0,0,25)
    lbl.Text = text
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.TextWrapped = true
    lbl.Parent = list
end

layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    list.CanvasSize = UDim2.new(0,0,0,layout.AbsoluteContentSize.Y)
end)

local function stringify(v, depth)
    depth = depth or 0
    if depth > 2 then return "..." end

    if typeof(v) == "Vector3" then
        return "Vector3("..math.floor(v.X)..","..math.floor(v.Y)..","..math.floor(v.Z)..")"
    elseif typeof(v) == "CFrame" then
        local p = v.Position
        return "CFrame("..math.floor(p.X)..","..math.floor(p.Y)..","..math.floor(p.Z)..")"
    elseif typeof(v) == "table" then
        local s = "{"
        for k,val in pairs(v) do
            s = s .. tostring(k).."="..stringify(val, depth+1)..", "
        end
        return s.."}"
    else
        return tostring(v)
    end
end

local function stringifyArgs(...)
    local t = {}
    for i,v in ipairs({...}) do
        table.insert(t, stringify(v))
    end
    return table.concat(t,", ")
end

-- 핵심: 완전한 namecall 후킹
local mt = getrawmetatable(game)
local old = mt.__namecall
setreadonly(mt,false)

mt.__namecall = newcclosure(function(self,...)
    local method = getnamecallmethod()

    if method == "FireServer" or method == "InvokeServer" then
        addLog("[SEND] "..self:GetFullName().." ("..stringifyArgs(...)..")")
    end

    return old(self,...)
end)

setreadonly(mt,true)

local function hookRemote(v)
    if v:IsA("RemoteEvent") then
        v.OnClientEvent:Connect(function(...)
            addLog("[RECEIVE] "..v:GetFullName().." ("..stringifyArgs(...)..")")
        end)
    elseif v:IsA("RemoteFunction") then
        v.OnClientInvoke = function(...)
            addLog("[INVOKE] "..v:GetFullName().." ("..stringifyArgs(...)..")")
        end
    end
end

for _,v in pairs(game:GetDescendants()) do
    hookRemote(v)
end

game.DescendantAdded:Connect(function(v)
    hookRemote(v)
end)

openBtn.MouseButton1Click:Connect(function()
    main.Visible = not main.Visible
end)

hideBtn.MouseButton1Click:Connect(function()
    main.Visible = false
end)
