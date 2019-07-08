---
layout: blog
published: true
---
## sLua-unreal 实现分析
本文从源码分析入手，主要关注点在功能覆盖度、实现原理和效率问题。
## LuaVar
LuaVar 用于c++测包装任何的lua值的对象，针对不同的lua类型提供了一系列的数据获取和设置方法。
1. 直接获取简单类型的值
2. 根据路径访问和设置table
3. 调用lua测闭包
### 内部值表示
LuaVar内部采用了和LuaVM一致的值表示，使用一个union + type的方式用一个结构体表示所有类型。也就是说定义的变量中不包含类型的信息，只有相应的值才会包含类型信息。
```
 typedef struct {
            union {
                RefRef* ref;
                lua_Integer i;
                lua_Number d;
                RefStr* s;
                void* ptr;
                bool b;
            };
            Type luatype;
        } lua_var;
```
参考Lua源码，可以发现大致是一致的。
```
typedef union Value {
  GCObject *gc;    /* collectable objects */
  void *p;         /* light userdata */
  int b;           /* booleans */
  lua_CFunction f; /* light C functions */
  lua_Integer i;   /* integer numbers */
  lua_Number n;    /* float numbers */
} Value;
```
这里我们通过分析LuaVar的通用构造函数来看LuaVar的值是如何对应到luaVM中的值的。
对于NIL、INT、STRING、BOOL等，是直接讲值复制到C++侧，对于FUNCTION、TABLE、USERDATA 则是利用LuaL_ref(l, LUA_REGISTRYINDEX) 保存了相对应值的引用。
```
	void LuaVar::init(lua_State* l, int p, LuaVar::Type type) {
		auto state = LuaState::get(l);
		stateIndex = state->stateIndex();
		switch (type) {
		case LV_NIL:
			break;
		case LV_INT:
			set(lua_tointeger(l, p));    
			break;
		case LV_NUMBER:
			set(lua_tonumber(l, p));      
			break;
		case LV_STRING: {
			size_t len;
			const char* buf = lua_tolstring(l, p, &len);
			set(buf, len);
			break;
		}
		case LV_BOOL:
			set(!!lua_toboolean(l, p));
			break;
		case LV_LIGHTUD:
			alloc(1);
			vars[0].ptr = lua_touserdata(l, p);
			vars[0].luatype = type;
			break;
		case LV_FUNCTION:
		case LV_TABLE:
		case LV_USERDATA:
			alloc(1);
			lua_pushvalue(l, p);
			vars[0].ref = new RefRef(l);
			vars[0].luatype = type;
			break;
		case LV_TUPLE:
			ensure(p > 0 && lua_gettop(l) >= p);
			initTuple(l, p);
			break;
		default:
			break;
		}
	}
```




Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
