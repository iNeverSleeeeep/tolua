## tolua#

从原版上增加了以下功能：
- [逻辑热加载](#逻辑热加载) 

# 逻辑热加载
通过重新指定lua函数proto的方式实现逻辑代码的热加载，不重启游戏即可修改逻辑( *function* )并保留数据以及各种函数的绑定。
通常在游戏开发中我们的逻辑为
```lua
-- file LocalClass.lua
local LocalClass = {}
function LocalClass:Hello()
end
return LocalClass
```
```lua
-- file GlobalClass.lua
GlobalClass = {}
function GlobalClass:Hello()
end
```
通常的逻辑热加载方式是 *require(path)* 后调用 *package.loaded[path] = nil* 之后再执行 *require(path)* 的地方会加载新的逻辑。
这种实现方式如果规划不好的话会有两个问题：
- 类中的数据会丢失
- 绑定到其他lua或c#中的回调函数没有办法使用新的逻辑。

我在tolua_runtime库中新增加了一个 *hot.swaplfunc()* 方法，可以切换lua函数指向的函数指针，而在tolua工程的LuaHotReload类中增加了检测lua文件变更的代码。lua文件变更时，会自动加载新的lua逻辑，然后将新的function原型(*proto*)绑定到旧的table function中，这样就实现了一个比较易用的逻辑热加载逻辑。
对于类似LocalClass的写法，不需要做任何更改，对于类似GlobalClass.lua的代码，需要做如下修改
```lua
-- file GlobalClass.lua
GlobalClass = {}
function GlobalClass:Hello()
end
return 'GlobalClass' -- 这里是需要新加的代码。
```

目前存在的问题：
- 只应该在开发期，在编辑器中使用，因为切换了函数原型不确定会不会引来gc的问题（比如内存泄漏）。**只在开发中提高工作效率用。**
- 没有经过丰富的测试，这个我只写了一些简单的测试用例（见TestHotReload）。
- 不支持局部函数的热加载

比如你有下面这样的写法：
```lua
function Class:Init()
    EventManager.ListenEvent("EVENT_NAME", function()
        -- 处理的逻辑代码
    end)
end
```
需要改成下面这种写法
```lua
function Class:NewInit()
    EventManager.ListenEvent("EVENT_NAME", Bind(self.EventHandler, self))
end
function Class:EventHandler()
    -- 处理的逻辑代码
end

-- 这个是其他地方的定义的函数
function Bind(func, s)
    return function()
        func(s)
    end
end
```

要看实现方法的话，具体看一下github.com/iNeverSleeeeep/tolua_runtime的 *lua_hot.c* 和本项目中的LuaHotLoader类。

