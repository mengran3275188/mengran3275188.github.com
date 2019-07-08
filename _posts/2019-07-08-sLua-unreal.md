---
layout: blog
published: true
---
# sLua-unreal 实现分析
本文从源码分析入手，主要关注点在实现原理、功能覆盖度和效率问题。
# LuaVar
LuaVar 用于c++测包装任何的lua值的对象，针对不同的lua类型提供了一系列的数据获取和设置方法。
1. 直接获取简单类型的值
2. 根据路径访问和设置table
3. 调用lua测闭包
## LuaVar内部值表示
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
对于NIL、INT、STRING、BOOL等，是直接将值复制到C++侧，
```
   // 为了方便阅读这里把初始化函数分成两段，这一段是将luaVM的值复制到c++
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
		... //此处代码省略
		default:
			break;
		}
	}
```
对于FUNCTION、TABLE、USERDATA 则是利用LuaL_ref(l, LUA_REGISTRYINDEX) 保存了相对应值的引用。
```
   // 为了方便阅读这里把初始化函数分成两段
	void LuaVar::init(lua_State* l, int p, LuaVar::Type type) {
		auto state = LuaState::get(l);
		stateIndex = state->stateIndex();
		switch (type) {
      ...   // 此处代码省略
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
lightuserdata和其他基本类型类似，初始化的过程是将指针的值从luaVM复制到c++。
还有一种新类型LV_TUPLE, 这个类型目前只用来处理调用lua方法时返回值大于一个的情况。LuaVar通过将内部的lua_var声明成数组，并定义了numOfVar来表示元素数量，这样初始化LV_TUPLE也就变成了初始化多个lua_var。
## LuaVar行为定义
这里我们只考虑对LuaVar封装的luaVM内的值的处理，FUNCTION、TABLE、USERDATA被包装成LUA_REGISTRYINDEX的整数索引。所以这里的数据类型判断（isXXX()）、设置值、提取值要么是和LuaVar交互，要么是和LUA_REGISTRYINDEX交互。对于sLua如何向LuaVM注册并push一个Userdata，将在下一章处理。
### 判断数据类型
根据lua_var.luatype判断。
```
    bool isNil() const;
    bool isFunction() const;
    bool isTuple() const;
    bool isTable() const;
    bool isInt() const;
    bool isNumber() const;
    bool isString() const;
    bool isBool() const;
    bool isUserdata(const char* t) const;
    bool isLightUserdata() const;
    bool isValid() const;
```
### 设置简单值类型
和上面init函数行为一致。
```
	void set(lua_State* L,int p);
	void set(lua_Integer v);
	void set(int v);
	void set(lua_Number v);
	void set(const char* v, size_t len);
	void set(const LuaLString& lstr);
	void set(bool b);
 ```
 ### 提取值
 提取简单值类型
 ```
	int asInt() const;
	int64 asInt64() const;
	float asFloat() const;
	double asDouble() const;
	const char* asString(size_t* outlen=nullptr) const;
	LuaLString asLString() const;
	bool asBool() const;
	void* asLightUD() const;
	template<typename T>
 	T* asUserdata(const char* t) const {
		auto L = getState();
		push(L);
		UserData<T*>* ud = reinterpret_cast<UserData<T*>*>(luaL_testudata(L, -1, t));
		lua_pop(L,1);
		return ud?ud->ud:nullptr;
	}
 ```
 ### Table相关操作
 ```
template<typename R,typename T>
R getFromTable(T key,bool rawget=false) const {
	ensure(isTable());
	auto L = getState();
	if (!L) return R();
	AutoStack as(L);
	push(L);
	LuaObject::push(L,key);
	if (rawget) lua_rawget(L, -2);
	else lua_gettable(L,-2);
	return ArgOperatorOpt::readArg<typename remove_cr<R>::type>(L,-1);
}
template<typename K,typename V>
void setToTable(K k,V v) {
	ensure(isTable());
	auto L = getState();
	push(L);
	LuaObject::push(L,k);
	LuaObject::push(L,v);
	lua_settable(L,-3);
	lua_pop(L,1);
}
 ```
 这里和我们通常通过c++代码设置lua table有三点不同，第一个是AutoStack的引入，第二个就是LuaObject::push()，第三个是ArgOperatorOpt::readArg。LuaObject::push 和 ArgOperatorOpt::readArg是一系列模板方法，负责c++内置类型、Unreal定义类型或者用户自定义类型和LuaVM的交互，根据偏特化或者特化对不同类型提供不同实现，这块会在下一章详细展开。AutoStack更像一个语法糖，在析构的时候重置Lua栈指针，这里我们将代码展开后就一目了然。
 ```
struct AutoStack {
	AutoStack(lua_State* l) {
	this->L = l;
	this->top = lua_gettop(L);
	}
~AutoStack() {
	lua_settop(this->L,this->top);
}
	lua_State* L;
	int top;
};
 ```
 
 
 
 


