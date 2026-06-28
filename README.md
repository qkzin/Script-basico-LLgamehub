-- LLGameHub - Sistema de Voo e ESP

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local StarterGui = game:GetService("StarterGui")

-- Variáveis principais
local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local rootPart = character:WaitForChild("HumanoidRootPart")

-- Estado do sistema
local flightEnabled = false
local flightActive = false
local espEnabled = false
local flightSpeed = 50
local panelMinimized = false
local dragging = false
local dragStart, panelStart

-- Criar ScreenGui
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "LLGameHub"
screenGui.Parent = player:WaitForChild("PlayerGui")
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

-- Função para criar toggle moderno
local function createToggle(parent, x, y, label, callback)
	local frame = Instance.new("Frame")
	frame.Size = UDim2.new(0, 200, 0, 40)
	frame.Position = UDim2.new(0, x, 0, y)
	frame.BackgroundTransparency = 1
	frame.Parent = parent
	
	local labelObj = Instance.new("TextLabel")
	labelObj.Size = UDim2.new(0, 100, 1, 0)
	labelObj.BackgroundTransparency = 1
	labelObj.Text = label
	labelObj.TextColor3 = Color3.fromRGB(200, 220, 255)
	labelObj.Font = Enum.Font.GothamBold
	labelObj.TextSize = 16
	labelObj.TextXAlignment = Enum.TextXAlignment.Left
	labelObj.Parent = frame
	
	local toggleFrame = Instance.new("Frame")
	toggleFrame.Size = UDim2.new(0, 60, 0, 30)
	toggleFrame.Position = UDim2.new(0, 130, 0, 5)
	toggleFrame.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
	toggleFrame.BorderSizePixel = 0
	toggleFrame.ClipsDescendants = true
	toggleFrame.Parent = frame
	
	local UICorner = Instance.new("UICorner")
	UICorner.CornerRadius = UDim.new(0, 15)
	UICorner.Parent = toggleFrame
	
	local toggleButton = Instance.new("Frame")
	toggleButton.Size = UDim2.new(0, 26, 0, 26)
	toggleButton.Position = UDim2.new(0, 2, 0, 2)
	toggleButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
	toggleButton.BorderSizePixel = 0
	toggleButton.Parent = toggleFrame
	
	local UICorner2 = Instance.new("UICorner")
	UICorner2.CornerRadius = UDim.new(0, 13)
	UICorner2.Parent = toggleButton
	
	local isOn = false
	
	local function updateToggle()
		local targetColor = isOn and Color3.fromRGB(50, 200, 50) or Color3.fromRGB(200, 50, 50)
		local targetPosition = isOn and UDim2.new(0, 32, 0, 2) or UDim2.new(0, 2, 0, 2)
		
		TweenService:Create(toggleFrame, TweenInfo.new(0.2), {BackgroundColor3 = targetColor}):Play()
		TweenService:Create(toggleButton, TweenInfo.new(0.2), {Position = targetPosition}):Play()
	end
	
	local function toggle()
		isOn = not isOn
		updateToggle()
		if callback then callback(isOn) end
	end
	
	toggleFrame.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			toggle()
		end
	end)
	
	return {
		frame = frame,
		setState = function(state)
			isOn = state
			updateToggle()
		end,
		getState = function() return isOn end
	}
end

-- Criar painel principal
local mainPanel = Instance.new("Frame")
mainPanel.Size = UDim2.new(0, 300, 0, 250)
mainPanel.Position = UDim2.new(0.5, -150, 0.5, -125)
mainPanel.BackgroundColor3 = Color3.fromRGB(10, 30, 60)
mainPanel.BorderSizePixel = 0
mainPanel.Active = true
mainPanel.Draggable = true
mainPanel.Parent = screenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 10)
UICorner.Parent = mainPanel

-- Barra de título
local titleBar = Instance.new("Frame")
titleBar.Size = UDim2.new(1, 0, 0, 40)
titleBar.BackgroundColor3 = Color3.fromRGB(20, 50, 90)
titleBar.BorderSizePixel = 0
titleBar.Parent = mainPanel

local UICorner3 = Instance.new("UICorner")
UICorner3.CornerRadius = UDim.new(0, 10)
UICorner3.Parent = titleBar

local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(1, -40, 1, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "LLGameHub"
titleLabel.TextColor3 = Color3.fromRGB(100, 200, 255)
titleLabel.Font = Enum.Font.GothamBold
titleLabel.TextSize = 20
titleLabel.Parent = titleBar

-- Botão minimizar
local minimizeBtn = Instance.new("ImageButton")
minimizeBtn.Size = UDim2.new(0, 30, 0, 30)
minimizeBtn.Position = UDim2.new(1, -35, 0, 5)
minimizeBtn.BackgroundColor3 = Color3.fromRGB(30, 60, 100)
minimizeBtn.BorderSizePixel = 0
minimizeBtn.Image = "rbxassetid://3926305904"
minimizeBtn.ImageColor3 = Color3.fromRGB(100, 200, 255)
minimizeBtn.Parent = titleBar

local UICorner4 = Instance.new("UICorner")
UICorner4.CornerRadius = UDim.new(0, 8)
UICorner4.Parent = minimizeBtn

-- Conteúdo do painel
local contentFrame = Instance.new("Frame")
contentFrame.Size = UDim2.new(1, -20, 1, -50)
contentFrame.Position = UDim2.new(0, 10, 0, 45)
contentFrame.BackgroundTransparency = 1
contentFrame.Parent = mainPanel

-- Toggles
local flightToggle = createToggle(contentFrame, 0, 0, "Voo", function(state)
	flightActive = state
	if state then
		enableFlight()
	else
		disableFlight()
	end
end)

local espToggle = createToggle(contentFrame, 0, 50, "ESP", function(state)
	espEnabled = state
	if state then
		enableESP()
	else
		disableESP()
	end
end)

-- Controle de velocidade
local speedLabel = Instance.new("TextLabel")
speedLabel.Size = UDim2.new(0, 100, 0, 30)
speedLabel.Position = UDim2.new(0, 0, 0, 110)
speedLabel.BackgroundTransparency = 1
speedLabel.Text = "Velocidade: 50"
speedLabel.TextColor3 = Color3.fromRGB(200, 220, 255)
speedLabel.Font = Enum.Font.GothamBold
speedLabel.TextSize = 14
speedLabel.TextXAlignment = Enum.TextXAlignment.Left
speedLabel.Parent = contentFrame

local speedSlider = Instance.new("Frame")
speedSlider.Size = UDim2.new(0, 200, 0, 10)
speedSlider.Position = UDim2.new(0, 0, 0, 145)
speedSlider.BackgroundColor3 = Color3.fromRGB(30, 60, 100)
speedSlider.BorderSizePixel = 0
speedSlider.Parent = contentFrame

local UICorner5 = Instance.new("UICorner")
UICorner5.CornerRadius = UDim.new(0, 5)
UICorner5.Parent = speedSlider

local sliderFill = Instance.new("Frame")
sliderFill.Size = UDim2.new(0.5, 0, 1, 0)
sliderFill.BackgroundColor3 = Color3.fromRGB(50, 150, 255)
sliderFill.BorderSizePixel = 0
sliderFill.Parent = speedSlider

local UICorner6 = Instance.new("UICorner")
UICorner6.CornerRadius = UDim.new(0, 5)
UICorner6.Parent = sliderFill

local sliderButton = Instance.new("Frame")
sliderButton.Size = UDim2.new(0, 20, 0, 20)
sliderButton.Position = UDim2.new(0.5, -10, 0, -5)
sliderButton.BackgroundColor3 = Color3.fromRGB(100, 200, 255)
sliderButton.BorderSizePixel = 0
sliderButton.Parent = speedSlider

local UICorner7 = Instance.new("UICorner")
UICorner7.CornerRadius = UDim.new(0, 10)
UICorner7.Parent = sliderButton

-- Sistema de arrasto do slider
local sliderDragging = false
sliderButton.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		sliderDragging = true
	end
end)

sliderButton.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		sliderDragging = false
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if sliderDragging and input.UserInputType == Enum.UserInputType.MouseMovement then
		local mousePos = UserInputService:GetMouseLocation()
		local sliderPos = speedSlider.AbsolutePosition
		local sliderSize = speedSlider.AbsoluteSize.X
		local relativeX = math.clamp(mousePos.X - sliderPos.X, 0, sliderSize)
		local percentage = relativeX / sliderSize
		
		sliderFill.Size = UDim2.new(percentage, 0, 1, 0)
		sliderButton.Position = UDim2.new(percentage, -10, 0, -5)
		
		flightSpeed = math.floor(percentage * 100)
		speedLabel.Text = "Velocidade: " .. flightSpeed
	end
end)

-- Ícone minimizado (arrastável)
local minimizedIcon = Instance.new("ImageButton")
minimizedIcon.Size = UDim2.new(0, 50, 0, 50)
minimizedIcon.Position = UDim2.new(0.5, -25, 0.5, -25)
minimizedIcon.BackgroundColor3 = Color3.fromRGB(20, 50, 90)
minimizedIcon.BorderSizePixel = 0
minimizedIcon.Image = "rbxassetid://3926305904"
minimizedIcon.ImageColor3 = Color3.fromRGB(100, 200, 255)
minimizedIcon.Visible = false
minimizedIcon.Active = true  -- Permite drag
minimizedIcon.Parent = screenGui

local UICorner8 = Instance.new("UICorner")
UICorner8.CornerRadius = UDim.new(0, 10)
UICorner8.Parent = minimizedIcon

-- Sistema de drag para o ícone minimizado
local iconDragging = false
local iconDragStart = Vector2.new(0, 0)
local iconStartPos = UDim2.new(0, 0, 0, 0)

minimizedIcon.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		iconDragging = true
		iconDragStart = UserInputService:GetMouseLocation()
		iconStartPos = minimizedIcon.Position
		
		-- Feedback visual ao começar a arrastar
		TweenService:Create(minimizedIcon, TweenInfo.new(0.15), {
			BackgroundColor3 = Color3.fromRGB(30, 70, 120),
			Size = UDim2.new(0, 55, 0, 55)
		}):Play()
	end
end)

minimizedIcon.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		iconDragging = false
		
		-- Restaurar tamanho normal
		TweenService:Create(minimizedIcon, TweenInfo.new(0.15), {
			BackgroundColor3 = Color3.fromRGB(20, 50, 90),
			Size = UDim2.new(0, 50, 0, 50)
		}):Play()
		
		-- Se não houve movimento, considera como clique (reabrir)
		local mousePos = UserInputService:GetMouseLocation()
		local distance = (mousePos - iconDragStart).Magnitude
		if distance < 10 then
			panelMinimized = false
			mainPanel.Visible = true
			minimizedIcon.Visible = false
		end
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if iconDragging and input.UserInputType == Enum.UserInputType.MouseMovement then
		local mousePos = UserInputService:GetMouseLocation()
		local delta = mousePos - iconDragStart
		
		-- Calcular nova posição
		local newX = iconStartPos.X.Offset + delta.X
		local newY = iconStartPos.Y.Offset + delta.Y
		
		-- Limitar à tela (opcional)
		local viewportSize = workspace.CurrentCamera.ViewportSize
		newX = math.clamp(newX, 0, viewportSize.X - minimizedIcon.AbsoluteSize.X)
		newY = math.clamp(newY, 0, viewportSize.Y - minimizedIcon.AbsoluteSize.Y)
		
		minimizedIcon.Position = UDim2.new(0, newX, 0, newY)
	end
end)

-- Botão minimizar (painel principal)
minimizeBtn.MouseButton1Click:Connect(function()
	panelMinimized = true
	mainPanel.Visible = false
	minimizedIcon.Visible = true
	
	-- Animar aparecimento do ícone
	TweenService:Create(minimizedIcon, TweenInfo.new(0.3), {
		ImageTransparency = 0,
		BackgroundTransparency = 0
	}):Play()
end)

-- Sistema de voo
local bodyVelocity, bodyGyro
local flightConnection

function enableFlight()
	if not flightEnabled then
		flightEnabled = true
		humanoid.PlatformStand = true
		
		bodyVelocity = Instance.new("BodyVelocity")
		bodyVelocity.MaxForce = Vector3.new(1e5, 1e5, 1e5)
		bodyVelocity.Velocity = Vector3.new(0, 0, 0)
		bodyVelocity.Parent = rootPart
		
		bodyGyro = Instance.new("BodyGyro")
		bodyGyro.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
		bodyGyro.P = 1000
		bodyGyro.D = 100
		bodyGyro.CFrame = rootPart.CFrame
		bodyGyro.Parent = rootPart
		
		-- Loop de voo
		flightConnection = RunService.RenderStepped:Connect(function()
			if flightActive and flightEnabled then
				local moveDirection = Vector3.new(
					UserInputService:IsKeyDown(Enum.KeyCode.D) and 1 or UserInputService:IsKeyDown(Enum.KeyCode.A) and -1 or 0,
					UserInputService:IsKeyDown(Enum.KeyCode.Space) and 1 or UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) and -1 or 0,
					UserInputService:IsKeyDown(Enum.KeyCode.S) and -1 or UserInputService:IsKeyDown(Enum.KeyCode.W) and 1 or 0
				)
				
				if moveDirection.Magnitude > 0 then
					moveDirection = moveDirection.Unit
					local camera = workspace.CurrentCamera
					local relativeDirection = camera.CFrame:VectorToWorldSpace(moveDirection)
					bodyVelocity.Velocity = relativeDirection * flightSpeed
					bodyGyro.CFrame = CFrame.lookAt(rootPart.Position, rootPart.Position + relativeDirection)
				else
					bodyVelocity.Velocity = Vector3.new(0, 0, 0)
				end
			end
		end)
	end
end

function disableFlight()
	flightEnabled = false
	humanoid.PlatformStand = false
	
	if flightConnection then
		flightConnection:Disconnect()
		flightConnection = nil
	end
	
	if bodyVelocity then
		bodyVelocity:Destroy()
		bodyVelocity = nil
	end
	
	if bodyGyro then
		bodyGyro:Destroy()
		bodyGyro = nil
	end
end

-- Sistema ESP
local espObjects = {}
local playerAddedConnection

function enableESP()
	for _, otherPlayer in ipairs(Players:GetPlayers()) do
		if otherPlayer ~= player then
			addESPToPlayer(otherPlayer)
		end
	end
	
	playerAddedConnection = Players.PlayerAdded:Connect(function(newPlayer)
		if espEnabled then
			addESPToPlayer(newPlayer)
		end
	end)
end

function addESPToPlayer(targetPlayer)
	local highlight = Instance.new("Highlight")
	highlight.FillColor = Color3.fromRGB(50, 150, 255)
	highlight.FillTransparency = 0.5
	highlight.OutlineColor = Color3.fromRGB(50, 150, 255)
	highlight.OutlineTransparency = 0.3
	highlight.Adornee = targetPlayer.Character
	highlight.Parent = targetPlayer.Character
	
	espObjects[targetPlayer] = highlight
	
	targetPlayer.CharacterAdded:Connect(function(newCharacter)
		if espEnabled then
			if espObjects[targetPlayer] then
				espObjects[targetPlayer]:Destroy()
			end
			
			local newHighlight = Instance.new("Highlight")
			newHighlight.FillColor = Color3.fromRGB(50, 150, 255)
			newHighlight.FillTransparency = 0.5
			newHighlight.OutlineColor = Color3.fromRGB(50, 150, 255)
			newHighlight.OutlineTransparency = 0.3
			newHighlight.Adornee = newCharacter
			newHighlight.Parent = newCharacter
			
			espObjects[targetPlayer] = newHighlight
		end
	end)
end

function disableESP()
	if playerAddedConnection then
		playerAddedConnection:Disconnect()
		playerAddedConnection = nil
	end
	
	for _, highlight in pairs(espObjects) do
		highlight:Destroy()
	end
	espObjects = {}
end

-- Feedback visual nos botões
local function addButtonFeedback(button)
	button.MouseEnter:Connect(function()
		TweenService:Create(button, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(40, 80, 130)}):Play()
	end)
	
	button.MouseLeave:Connect(function()
		TweenService:Create(button, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(30, 60, 100)}):Play()
	end)
end

addButtonFeedback(minimizeBtn)
addButtonFeedback(minimizedIcon)

-- Limpeza ao sair
player.OnTeleport:Connect(function()
	screenGui:Destroy()
end)

-- Notificação inicial
StarterGui:SetCore("SendNotification", {
	Title = "LLGameHub",
	Text = "Sistema carregado! Pressione o ícone para abrir.",
	Duration = 5
})
