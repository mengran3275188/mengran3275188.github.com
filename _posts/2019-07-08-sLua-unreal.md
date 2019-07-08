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
 ### 方法调用
 这里的方法调用，假设所有参数已经都在LuaVM栈上。
 ```
 	int LuaVar::docall(int argn) const {
		if (!isValid()) {
			Log::Error("State of lua function is invalid");
			return 0;
		}
		auto L = getState();
		int top = lua_gettop(L);
		top = top - argn + 1;
		LuaState::pushErrorHandler(L);
		lua_insert(L, top);
		vars[0].ref->push(L);

		{
			LuaScriptCallGuard g(L);
			lua_insert(L, top + 1);
			// top is err handler
			if (lua_pcallk(L, argn, LUA_MULTRET, top, NULL, NULL))
				lua_pop(L, 1);
			lua_remove(L, top); // remove err handler;
		}
		return lua_gettop(L) - top + 1;
	}
 ```
 LuaScriptCallGuard为额外线程检测c++调用lua方法是否死锁，单个方法调用超过5s会触发超时。
 # LuaObject
 ## 创建Userdata,并向luaVM注册metatable。
 ```
template<class T, ESPMode mode, bool F = IsUObject<T>::value>
static int pushType(lua_State* L, SharedRefUD<T, mode>* cls, const char* tn);

template<class T,ESPMode mode, bool F = IsUObject<T>::value>
static int pushType(lua_State* L, SharedPtrUD<T, mode>* cls, const char* tn);

template<class T, bool F = IsUObject<T>::value>
static int pushType(lua_State* L,T cls,const char* tn,lua_CFunction setupmt,int gc);

template<class T, bool F = IsUObject<T>::value >
static int pushType(lua_State* L,T cls,const char* tn,lua_CFunction setupmt=nullptr,lua_CFunction gc=nullptr);

template<>
inline int LuaObject::pushType<LuaStruct*, false>(lua_State* L, LuaStruct* cls,
		const char* tn, lua_CFunction setupmt, lua_CFunction gc);

 ```
 前两个函数用来处理SharedRef<T>或SharedPtr<T>类型，后面两个函数是两个模板函数，用来处理其他类型，最后一个函数是第三个函数的全特化版本，针对LuaStruct*类型做了特殊处理。虽然这里用五个函数来分别实现pushType操作，但是这五个函数的大部分代码都是一致的，这里列出其中一个函数的实现，注释里会说明其他函数的不同之处。
  ```
template<class T, bool F = IsUObject<T>::value >
static int pushType(lua_State* L,T cls,const char* tn,lua_CFunction setupmt=nullptr,lua_CFunction gc=nullptr) {
	if(!cls) {
		lua_pushnil(L);
		return 1;
	}
   // 这里UserData<T>不同的函数会采用不同的类型
   // LuaStruct 会使用 UserData<LuaStruct*>
   // SharedRef<T> 和 SharedPtr<T>会使用对应的包装类 UserData<BOXPUD*>，避免智能指针在LuaVM里传递。
	UserData<T>* ud = reinterpret_cast< UserData<T>* >(lua_newuserdata(L, sizeof(UserData<T>)));
	ud->parent = nullptr;
	ud->ud = cls;
	ud->flag = gc!=nullptr?UD_AUTOGC:UD_NOFLAG;
	if (F) ud->flag |= UD_UOBJECT;  //根据实际类型打上不同标签  UD_UOBJECT|UD_THREADSAFEPTR|UD_USTRUCT
	setupMetaTable(L,tn,setupmt,gc);
	return 1;
}
  ```
  ## 将值从c++压入LuaVM
  LuaObject类提供了34个同名的重载函数来实现，这里通过归纳为以下几类来分别说明：
  - c/c++基本类型（int,bool, void...）
  - Unreal string类型（FText, FString, FName）
  - Unreal UObject
  - TSharedPtr、TSharedRef
  - TBaseDelegate
  - TArray、 TMap
  
  
  

 
 
 
 
 


