---
layout: blog
published: false
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




Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
