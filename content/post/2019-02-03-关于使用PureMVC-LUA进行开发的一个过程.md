---
title: 关于使用PureMVC-LUA进行开发的一个过程
categories:
  - Cocos2d-X
date: 2019-02-03 21:04:18
updated: 2019-02-03 21:04:18
tags: 
  - Cocos2d-X
  - Lua
---
不要问我为什么，业务需要，有用到，所以来了解一下这么个例子。暂不说其优劣。

<!--more-->

# 简介

[PureMVC官方网站地址](http://puremvc.org)

官方网站上没有 Lua 的实现，属于个人作者做的。

[puremvc-lua项目地址](https://github.com/themoonbear/puremvc-lua)


[最佳实践中文版](http://puremvc.org/docs/PureMVC_IIBP_Chinese.pdf)  

说道 MVC 自然就少不了 **Model, View, Controler**。但在不同的框架中其表现的意义是不一样的。

## 关系图

![](../res/puremvc-concept-diagram.png)

> Model, View, Controler, Facade 类都是单例模式。 Model 缓存了对 Proxies 的引用，View 缓存对 Mediators 的引用，Controler 维护对 Command 类的映射，这些 Command 是无状态的，只有在需要的时候才建立。  
> 由 Facede 来初始化 Model, View, Controler，同时提供一个类来访问三着所有公共的方法。


>Mediators 操作  View Components（视图组件），Proxy 操作数据模型（包括服务），Command 负责比较复杂的活动（比如APP启动，关闭等）。Proxies，Mediators，Commands 都会用到 Facade 来用其他两者进行通信。



## Model

管理 Proxies，Proxies 进行实际的数据操作。 Proxies 可以通知 View 和 Controler。
## View
管理 Mediators，同时操作从 View Components 来的事件，及将数据传递给 视图组件。
## Controler

管理 Commands， Commands 是业务逻辑所在。 Commands 可以通知 View， 更新 Model。

# 实现

PureMVC 的入口，是 Facade，用在 Cococs-2dx Lua ，那么就是在 main.lua 中获取一个 Facade 实例。

我们通过继承 Facade 来自定义一个 AppFacade。



```lua
AppFacade = class("AppFacade", puremvc.Facade)

function AppFacade:ctor(key)
    self.super.ctor(self, key)
    --invote base method
end

function AppFacade:initializeController()
    --invote base method
    self.super.initializeController(self)
    self:initCommand()
end

--注册游戏Command
function AppFacade:initCommand()
    local StartupCommand = require("client.src.controller.command.StartupCommand")
    local LoadViewCommand = require("client.src.controller.command.LoadViewCommand")
    local UnLoadViewCommand = require("client.src.controller.command.UnLoadViewCommand")

    self:registerCommand(GAME_COMMAMD.START_UP, StartupCommand)
    self:registerCommand(GAME_COMMAMD.LOAD_VIEW, LoadViewCommand)
    self:registerCommand(GAME_COMMAMD.UNLOAD_VIEW, UnLoadViewCommand)
end

function AppFacade:startup()
    self:sendNotification(GAME_COMMAMD.START_UP)
end

--脚本重启
function AppFacade.restartup(key)
    local key = key or "DefaultKey"
    AppFacade.removeCore(key)
    AppFacade:getInstance():sendNotification(GAME_COMMAMD.START_UP)
end

function AppFacade:getInstance(key)
    local key = key or "DefaultKey"
    local instance = self.instanceMap[key]
    if nil ~= instance then
        return instance
    end
    if nil == self.instanceMap[key] then
        self.instanceMap[key] = AppFacade.new(key)
    end

    return self.instanceMap[key]
end

```

## Command 注册

在我们自定义的 Facade 中，我们注册了三个通知的处理类。事实上， Facade 作为一个全局的容器，容纳所有的对象，通过容器获取想要的对象进行操作。

注册通知，实际上就是在  Facade 的 **Controler** 内增加一个映射。


```lua
function Facade:registerCommand(notificationName, commandClassRef)
	self.controller:registerCommand(notificationName, commandClassRef)	
end
```

```lua
function Controller:registerCommand(notificationName, commandClassRef)
	assert(type(notificationName) == "string", "notificationName expected string")
	assert(type(commandClassRef) == "table", "commandClassRef expected table")
	if(self.commandMap[notificationName] == nil) then
		self.view:registerObserver(notificationName, Observer.new(self.executeCommand, self));
	end
	self.commandMap[notificationName] = commandClassRef
end
```
这里需要注意的是 `Observer.new(self.executeCommand, self));` 观察者的通知方法(notifyMethod)，就是 Controler的 executeCommand 方法：

```lua
function Controller:executeCommand(note)
	local commandClassRef = self.commandMap[note:getName()]
	if(commandClassRef == nil) then
		return
	end
	local commandInstance = commandClassRef.new()
	commandInstance:initializeNotifier(self.multitonKey)
	commandInstance:execute(note)
end
```

通知上下文，就是 Controler 本身。

```lua
function View:registerObserver(notificationName, observer)
	if self.observerMap[notificationName] ~= nil then
		table.insert(self.observerMap[notificationName], observer)
	else
		self.observerMap[notificationName] = {observer}
	end
end
```

除了在 Controler 中增加一个通知类型到处理类的映射，还会在 View 中注册观察者。

在 View 中，有个类型的通知，可能有多个观察者。

## 启动

在我们的 main.lua 中，调用  AppFacade 的 startup() 方法。

```lua
local function main()
	AppFacade:getInstance():startup()
end

local status, msg = xpcall(main, __G__TRACKBACK__)
if not status then
	print(msg)
end
```

其本质，也就是发送了一个通知出去：

```lua
function AppFacade:startup()
    self:sendNotification(GAME_COMMAMD.START_UP)
end
```

我们的通知，将会由我们注册的映射类处理，在前面的代码中，我们可以知道，其是 **StartupCommand**。

## 通知处理

`self:sendNotification(GAME_COMMAMD.START_UP)` 调用的是 Facade 类中的方法：

```lua
function Facade:sendNotification(notificationName, body, type)
	self:notifyObservers(Notification.new(notificationName, body, type))
end

function Facade:notifyObservers(notification)
	if self.view ~= nil then
		self.view:notifyObservers(notification)
	end
end
```

其本质，是在 View 中，将所有此通知类型观察者都进行通知。

```lua
function View:notifyObservers(notification)
	if self.observerMap[notification:getName()] ~= nil then
		local observers_ref = self.observerMap[notification:getName()]
		for _, o in pairs(observers_ref) do
			o:notifyObserver(notification)
		end
	end
end
```

```lua
function Observer:notifyObserver(notification)
	self.notify(self.context, notification)
end
```

前文说到，观察者的通知方法，已经被设置为 Controler 的 executeCommand 方法，

```lua
function Controller:executeCommand(note)
	local commandClassRef = self.commandMap[note:getName()]
	if(commandClassRef == nil) then
		return
	end
	local commandInstance = commandClassRef.new()
	commandInstance:initializeNotifier(self.multitonKey)
	commandInstance:execute(note)
end
```

此方法所做的就是根据通知类型，获取对应的类进行实例化，然后调用其 execute 方法。在我们这里，其实据执行的 StartupCommand:execute() 方法：

```lua
function StartupCommand:execute(note)
    self.super.execute(self, note)

    --@parm1 消息命令
    --@parm2 合并到 context.data 中，作为状态被传递到 Mediator 使用，使用 context.data 在场景之间传递信息非常方便
    --@parm3 消息命令附带信息 ? 这个信息是发到哪里去了？
    self:sendNotification(GAME_COMMAMD.PUSH_VIEW, {}, VIEW_LIST.WELLCOME_SCENE)
    --self:sendNotification(GAME_COMMAMD.PUSH_VIEW, {}, VIEW_LIST.WELLCOME_SCENE)
end
```

### SubCommand
其会先执行父类中的 execute 方法，再执行自定义的方法。由于我们之前添加了三个子 Command ，所以会优先执行：

```lua
    local PrepModelCommand = require("client.src.controller.command.PrepModelCommand")
    local PrepControllerCommand = require("client.src.controller.command.PrepControllerCommand")
    local PrepViewCommand = require("client.src.controller.command.PrepViewCommand")
```

```lua
function MacroCommand:execute(note)
    -- SIC- TODO optimize
    while(#self.subCommands > 0) do
        local ref= table.remove(self.subCommands, 1)
        local cmd= ref.new()
        cmd:initializeNotifier(self.multitonKey)
        cmd:execute(note)
    end
end
```

以 PrepViewCommand 为例：

```lua
function PrepViewCommand:execute(note)
	self.super.ctor(self)
	
	local ViewMediator = require("client.src.view.mediator.ViewMediator")
	
    self.facade:registerMediator(ViewMediator.new())
end
```

其向 Facade 注册了一个 ViewMediator。

我们简单看一下其注册过程：

```lua
function Facade:registerMediator(mediator)
	if self.view ~= nil then
		self.view:registerMediator(mediator)
	end
end
```

```lua
function View:registerMediator(mediator)
	if self.mediatorMap[mediator:getMediatorName()] ~= nil then
		return
	end
	mediator:initializeNotifier(self.multitonKey)
	self.mediatorMap[mediator:getMediatorName()] = mediator
	local interests = mediator:listNotificationInterests()
	if #interests > 0 then
		local observer = Observer.new(mediator.handleNotification, mediator)
		for _, i in pairs(interests) do
			self:registerObserver(i, observer)
		end
	end
	mediator:onRegister()
end
```

在这里，比较需要重点注意的是，每个 ViewMediator 会关注多个通知类型，那么就要为每个通知类型建立一个观察者，每个观察者的处理函数，及上下文都一样。


在我们这个 ViewMediator 中其关注的通知类型有几种：

```lua
function ViewMediator:listNotificationInterests()
	self.super.listNotificationInterests(self)
	--该mediator关心的消息
	return {
		GAME_COMMAMD.PUSH_VIEW,
		GAME_COMMAMD.POP_VIEW,
		GAME_COMMAMD.PLAY_EFFECT,
		GAME_COMMAMD.PLAY_ONE_EFFECT
	}
end
```

当其中一种通知到来的时候，就会调用其通知处理函数：

```lua
function ViewMediator:handleNotification(notification)
	self.super.handleNotification(self, notification)

	local msgName = notification:getName()
	local msgData = notification:getBody()
	local msgType = notification:getType()

	--local contextProxy = AppFacade:getInstance():retrieveProxy("ContextProxy")
	--处理收到的消息
	printf("msgName:%s Line:%d Command:%s", msgName, debug.getinfo(1).currentline, msgName)
	if (msgName == GAME_COMMAMD.PUSH_VIEW) then
		--self:sendNotification(GAME_COMMAMD.LOAD_VIEW, )
		--local context = Context.new() or contextProxy:findContextByName(msgType)

		-- load context nData
		--[[		if (type(msgData) == "table") then
			for k, v in pairs(msgData) do
				context.data[k] = v
			end
		end--]]
		assert(msgType ~= nil, "PushView Name expected not nil")
		local mediatorClass = nil
		local viewClass = nil

		--创建上下文环境
		if (msgType == VIEW_LIST.WELLCOME_SCENE) then
			mediatorClass = nil
			viewClass = require("client.src.view.component.WelcomeScene")
		elseif (msgType == VIEW_LIST.LOGON_SCENE) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.LogonScene")
		elseif (msgType == VIEW_LIST.CLIENT_SCENE) then
			mediatorClass = require("client.src.view.mediator.ClientSceneMediator")
			viewClass = require("client.src.plaza.views.ClientScene")
		elseif (msgType == VIEW_LIST.ROOM_LIST_LAYER) then
			mediatorClass = require("client.src.view.mediator.RoomListMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.RoomListLayer")
		elseif (msgType == VIEW_LIST.GONGGAO_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.GongGaoLayer")
		elseif (msgType == VIEW_LIST.VIP_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.VIPLayer")
		elseif (msgType == VIEW_LIST.GONGGAO_IOS_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.GongGaoLayerIOS")
		elseif (msgType == VIEW_LIST.PERSON_LAYER) then
			mediatorClass = require("client.src.view.mediator.UserInfoMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.UserInfoLayer")
		elseif (msgType == VIEW_LIST.POPWAIT_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.app.views.layer.other.PopWait")
		elseif (msgType == VIEW_LIST.QUERY_DIALOG_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.app.views.layer.other.QueryDialog")
		elseif (msgType == VIEW_LIST.SELECT_SYSTEM_HEAD_LAYER) then
			mediatorClass = require("client.src.view.mediator.SelectSystemHeadMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.SelectSystemHeadLayer")
		elseif (msgType == VIEW_LIST.SHARE_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.PromoterInputLayer")
		elseif (msgType == VIEW_LIST.GAME_LAYER) then
			mediatorClass = nil
			assert(msgData.viewClassPath ~= nil, "viewClassPath is nil please check game path")
			viewClass = require(msgData.viewClassPath)
		elseif (msgType == VIEW_LIST.ACTIVITY_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.ActivityLayer")
		elseif (msgType == VIEW_LIST.ACTIVITY_IOS_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.ActivityLayerIOS")
		elseif (msgType == VIEW_LIST.MODIFY_ACCOUNT_PASS_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.ModifyAccountPasswdLayer")
		elseif (msgType == VIEW_LIST.MODIFY_BANK_PASS_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.ModifyBankPasswdLayer")
		elseif (msgType == VIEW_LIST.TASKLAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.TaskLayer")
		elseif (msgType == VIEW_LIST.BANK_LAYER) then
			mediatorClass = require("client.src.view.mediator.BankMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.BankLayer")
		elseif (msgType == VIEW_LIST.BANK_OPEN_LAYER) then
			mediatorClass = require("client.src.view.mediator.BankOpenMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.BankOpenLayer")
		elseif (msgType == VIEW_LIST.BANK_MODIFY_LAYER) then
			mediatorClass = require("client.src.view.mediator.BankModifyMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.BankModifyLayer")
		elseif (msgType == VIEW_LIST.GAME_WAIT_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.app.views.layer.other.PopGameWait")
		elseif (msgType == VIEW_LIST.SETTING_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.other.OptionLayer")
		elseif (msgType == VIEW_LIST.SHOP_LAYER) then
			mediatorClass = require("client.src.view.mediator.ShopMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.ShopLayer")
		elseif (msgType == VIEW_LIST.SHOP_APPSTORE_LAYER) then
			mediatorClass = require("client.src.view.mediator.ShopMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.ShopAppstoreLayer")
		elseif (msgType == VIEW_LIST.EARN_LAYER) then
			mediatorClass = require("client.src.view.mediator.ShopMediator")
			 --require("client.src.view.mediator.EarnMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.EarnLayer")
		elseif (msgType == VIEW_LIST.EARN_MONEY) then
			mediatorClass = require("client.src.view.mediator.ShopMediator")
			 --require("client.src.view.mediator.EarnMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.EarnMoney")
		elseif (msgType == VIEW_LIST.VISITOR_BIND_LAYER) then
			mediatorClass = nil --require("client.src.view.mediator.EarnMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.VisitorBindLayer")
		elseif (msgType == VIEW_LIST.AGENT_RECHARGE_LAYER) then
			mediatorClass = nil --require("client.src.view.mediator.EarnMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.AgentRechargeLayer")
		elseif (msgType == VIEW_LIST.REGISTER_AGREATMENT) then
			mediatorClass = nil --require("client.src.view.mediator.EarnMediator")
			viewClass = require("client.src.plaza.views.layer.logon.Agreatment")
		elseif (msgType == VIEW_LIST.ACCOUNT_REGISTER_LAYER) then
			mediatorClass = nil --require("client.src.view.mediator.EarnMediator")
			viewClass = require("client.src.plaza.views.layer.logon.AccountRegisterView")
		elseif (msgType == VIEW_LIST.SHOW_POP_TIPS) then
			mediatorClass = nil
			viewClass = require("client.src.app.views.layer.other.PopTips")
		elseif (msgType == VIEW_LIST.SHOP_SHENGQING_DAILI) then
			mediatorClass = nil --require("client.src.view.mediator.EarnMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.AgentShengQingLayer")
		elseif (msgType == VIEW_LIST.SHOP_SHENGQING_DAILI_COMFIRM) then
			mediatorClass = nil --require("client.src.view.mediator.EarnMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.AgentShengQingConfirmLayer")
		elseif (msgType == VIEW_LIST.SHOP_TOUSU_DAILI) then
			mediatorClass = require("client.src.view.mediator.AgentTouSuMediator")
			viewClass = require("client.src.plaza.views.layer.plaza.AgentTouSuLayer")
		elseif (msgType == VIEW_LIST.AGENT_AGREATMENT) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.AgentAgreatment")
		elseif (msgType == VIEW_LIST.GAME_RULE) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.GameRule")
		elseif (msgType == VIEW_LIST.RECHARGE_RIGHT_NOW) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.RechargeRightNow")
		elseif (msgType == VIEW_LIST.ROATEWAIT_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.view.component.PopRoateWait")
		elseif (msgType == VIEW_LIST.SELECT_LINK_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.SelectLinkLayer")
		elseif (msgType == VIEW_LIST.HALL_MESSAGE_LAYER) then
			mediatorClass = nil
			viewClass = require("client.src.plaza.views.layer.plaza.HallMessageLayer")
		else
			assert(false, "not support view type")
		end

		local genid = guid()

		self:sendNotification(
			GAME_COMMAMD.LOAD_VIEW,
			{id = genid, viewClass = viewClass, mediatorClass = mediatorClass, parm = msgData},
			msgType
		)
	elseif (msgName == GAME_COMMAMD.POP_VIEW) then
		if (msgData ~= nil) then
			-- load context nData
			self:sendNotification(GAME_COMMAMD.UNLOAD_VIEW, msgData)
		else
			self:sendNotification(GAME_COMMAMD.UNLOAD_VIEW)
		end
	elseif (msgName == GAME_COMMAMD.PLAY_ONE_EFFECT) then
		--只播放一次的 音效
		local musicPath = msgType
		assert(type(msgType) == "string")
		--获取音乐播放配置
		local musicProxy = AppFacade:getInstance():retrieveProxy("MusicRecordProxy")
		local refCount = musicProxy:addRef(musicPath)
		if (refCount == 1) then
			AudioEngine.playEffect(cc.FileUtils:getInstance():fullPathForFilename(musicPath), false)
		end
	elseif (msgName == GAME_COMMAMD.PLAY_EFFECT) then
		local musicPath = msgType
		assert(type(msgType) == "string")
		--获取音乐播放配置
		AudioEngine.playEffect(cc.FileUtils:getInstance():fullPathForFilename(musicPath), false)
	else
		assert(false, "Command not support now!")
	end
end
```

## StartupCommand:execute

```lua
function StartupCommand:execute(note)
    self.super.execute(self, note)

    --@parm1 消息命令
    --@parm2 合并到 context.data 中，作为状态被传递到 Mediator 使用，使用 context.data 在场景之间传递信息非常方便
    --@parm3 消息命令附带信息 ? 这个信息是发到哪里去了？
    self:sendNotification(GAME_COMMAMD.PUSH_VIEW, {}, VIEW_LIST.WELLCOME_SCENE)
    --self:sendNotification(GAME_COMMAMD.PUSH_VIEW, {}, VIEW_LIST.WELLCOME_SCENE)
end

```

最终，当我们的调用 `    self:sendNotification(GAME_COMMAMD.PUSH_VIEW, {}, VIEW_LIST.WELLCOME_SCENE)`
，将会由 ViewMediator:handleNotification(notification) 处理：

```lua
	if (msgName == GAME_COMMAMD.PUSH_VIEW) then
		--self:sendNotification(GAME_COMMAMD.LOAD_VIEW, )
		--local context = Context.new() or contextProxy:findContextByName(msgType)

		-- load context nData
		--[[		if (type(msgData) == "table") then
			for k, v in pairs(msgData) do
				context.data[k] = v
			end
		end--]]
		assert(msgType ~= nil, "PushView Name expected not nil")
		local mediatorClass = nil
		local viewClass = nil

		--创建上下文环境
		if (msgType == VIEW_LIST.WELLCOME_SCENE) then
			mediatorClass = nil
			viewClass = require("client.src.view.component.WelcomeScene")
    ..........
    
    		local genid = guid()

		self:sendNotification(
			GAME_COMMAMD.LOAD_VIEW,
			{id = genid, viewClass = viewClass, mediatorClass = mediatorClass, parm = msgData},
			msgType
		)
```

## LOAD_VIEW 事件

此事件在 Facade 中注册：

```lua
    self:registerCommand(GAME_COMMAMD.LOAD_VIEW, LoadViewCommand)
```

```lua
function LoadViewCommand:execute(note)
    local msgData = note:getBody()
	local msgType = note:getType()

	--总视图上下文环境栈
	local contextProxy = AppFacade:getInstance():retrieveProxy("ContextProxy")
	
--[[	--查找要添加的视图
	local willAddContext = msgData.context
	
	if (willAddContext:getParent() ~= nil) then
		assert(false, "this context already has parent context")
	end	--]]
	
	--suspend、running、dead、normal
	local eventCoroutine = nil
	eventCoroutine = coroutine.create(function()
		local willAddContext = nil
		
		local Context = require("client.src.model.base.Context")
		if (msgData.parm.canrepeat == false) then
			willAddContext = contextProxy:findContextByName(msgType) or Context.new()
		else
			willAddContext = Context.new()
		end

		--判断是否为新创建的 Context 界面
		if (willAddContext:getParent() == nil) then
			willAddContext:setName(msgType)
			willAddContext:setViewClass(msgData.viewClass)
			willAddContext:setMediatorClass(msgData.mediatorClass)			
		end	
		
		--附加数据重新赋值
		if (type(msgData.parm) == "table") then
			willAddContext.data = {}
			for k, v in pairs(msgData.parm) do
				willAddContext.data[k] = v
			end
		end
		
		--引用次数 +1
		local refCount = willAddContext:retain()
		--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
		if (willAddContext.data.canrepeat == false) then
			if (refCount > 1) then
				printf("PUSH_VIEW FAILED,the view is find, attribute is not repeat!", debug.getinfo(1).currentline)
			else
				--上下文环境入栈
				--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
				self:recursiveAddView(willAddContext, function (context)
					if (context == willAddContext) then
						if (willAddContext.data.viewcallback ~= nil) then
							willAddContext.data.viewcallback(context:getView(), "enter")
						end
					end
				end, eventCoroutine)
				--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
				coroutine.yield()
			end
		else
			--上下文环境入栈
			--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
			self:recursiveAddView(willAddContext, function (context)
				if (context == willAddContext) then
					if (willAddContext.data.viewcallback ~= nil) then
						willAddContext.data.viewcallback(context:getView(), "enter")
					end
				end
			end, eventCoroutine)
			
			coroutine.yield()
		end
		--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
		local frontEventCoroutine = contextProxy:frontEvent()
		if (frontEventCoroutine) then
			--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
			if (frontEventCoroutine ~= eventCoroutine) then
				--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
				local result, msg = coroutine.resume(frontEventCoroutine)
				assert(result, msg)
			else
				--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
				contextProxy:popFrontEvent()
				local nxtEventCoroutine = contextProxy:frontEvent()
				if (nxtEventCoroutine) then
					--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
					local result, msg = coroutine.resume(nxtEventCoroutine)
					assert(result, msg)					
				end							
			end
		end
	end)
	
	local frontCoroutine = contextProxy:frontEvent()	
	
	contextProxy:pushBackEvent(eventCoroutine)
	--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
	if (frontCoroutine == nil) then
		--printf("PUSH_VIEW %d", debug.getinfo(1).currentline)
		local result, msg = coroutine.resume(eventCoroutine)
		assert(result, msg)				
	end
end
```

通过上下文来操作这个问题比较复杂，有空再解释。


