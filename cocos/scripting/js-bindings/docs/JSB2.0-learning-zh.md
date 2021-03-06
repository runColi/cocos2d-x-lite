# JSB2.0 绑定教程

## 抽象层

### 架构

![](JSB2.0-Architecture.png)

### 宏(macro)

抽象层必然会比直接使用JS引擎API的方式多占用一些CPU执行时间，如何把抽象层本身的开销降到最低成为设计的第一目标。

JS绑定的大部分工作其实就是设定JS相关操作的CPP回调，在回调函数中关联CPP对象。其实主要包含如下两种类型：

* 注册JS函数（包含全局函数，类构造函数、类析构函数、类成员函数，类静态成员函数），绑定一个CPP回调
* 注册JS对象的属性读写访问器，分别绑定读与写的CPP回调

如何做到抽象层开销最小而且暴露统一的API供上层使用？

以注册JS函数的回调定义为例，JavaScriptCore, SpiderMoneky, V8, ChakraCore的定义各不相同，具体如下：

**JavaScriptCore:**

```c++
JSValueRef JSB_foo_func(
		JSContextRef _cx,
		JSObjectRef _function,
		JSObjectRef _thisObject,
		size_t argc,
		const JSValueRef _argv[],
		JSValueRef* _exception
	);
```

**SpiderMonkey:**

```c++
bool JSB_foo_func(
		JSContext* _cx,
		unsigned argc,
		JS::Value* _vp
	);
```

**V8:**

```c++
void JSB_foo_func(
		const v8::FunctionCallbackInfo<v8::Value>& v8args
	);
```

**ChakraCore:**

```c++
JsValueRef JSB_foo_func(
		JsValueRef _callee,
		bool _isConstructCall,
		JsValueRef* _argv,
		unsigned short argc,
		void* _callbackState
	);
```

我们评估了几种方案，最终确定使用`宏`来抹平不同JS引擎回调函数定义与参数类型的不同，不管底层是使用什么引擎，开发者统一使用一种回调函数的定义。我们借鉴了lua的回调函数定义方式，抽象层所有的JS到CPP的回调函数的定义为：

```c++
bool foo(se::State& s)
{
	...
	...
}
SE_BIND_FUNC(foo) // 此处以回调函数的定义为例
```

开发者编写完回调函数后，记住使用`SE_BIND_XXX`系列的宏对回调函数进行包装。目前提供了如下几个宏：

* SE\_BIND\_PROP_GET: 包装一个JS对象属性读取的回调函数
* SE\_BIND\_PROP_SET: 包装一个JS对象属性写入的回调函数
* SE\_BIND_FUNC: 包装一个JS函数，可用于全局函数、类成员函数、类静态函数
* SE\_DECLARE_FUNC: 声明一个JS函数，一般在.h头文件中使用
* SE\_BIND_CTOR: 包装一个JS构造函数
* SE\_BIND\_SUB\_CLS\_CTOR: 包装一个JS子类的构造函数，此子类使用cc.Class.extend继承Native绑定类
* SE\_FINALIZE_FUNC: 包装一个JS对象被GC回收后的回调函数
* SE\_DECLARE\_FINALIZE_FUNC: 声明一个JS对象被GC回收后的回调函数
* _SE: 包装回调函数的名称，转义为每个JS引擎能够识别的回调函数的定义，注意，第一个字符为下划线，类似Windows下用的`_T("xxx")`来包装Unicode或者MultiBytes字符串

## API

### CPP命名空间(namespace)

CPP抽象层所有的类型都在`se`命名空间下，其为ScriptEngine的缩写。

### 类型

#### se::ScriptEngine

se::ScriptEngine为JS引擎的管理员，掌管JS引擎初始化、销毁、重启、Native模块注册、加载脚本、强制垃圾回收、JS异常清理、是否启用调试器。
它是一个单例，可通过se::ScriptEngine::getInstance()得到对应的实例。

#### se::Value

se::Value可以被理解为JS变量在CPP层的引用。JS变量有`object`, `number`, `string`, `boolean`, `null`, `undefined`六种类型，因此se::Value使用`union`包含`object`, `number`, `string`, `boolean`4种`有值类型`，`无值类型`: `null`, `undefined`可由`_type`直接表示。

```c++
namespace se {
	class Value {
		enum class Type : char
		{
		    Undefined = 0,
		    Null,
		    Number,
		    Boolean,
		    String,
		    Object
		};
		...
		...
	private:
		union {
		    bool _boolean;
		    double _number;
		    std::string* _string;
		    Object* _object;
		} _u;
		
		Type _type;
		...
		...
	};
}
```

如果se::Value中保存基础数据类型，比如`number`，`string`，`boolean`，其内部是直接存储一份值副本。
`object`的存储比较特殊，是通过`se::Object*`对JS对象的弱引用(weak reference)。

#### se::Object

se::Object继承于se::RefCounter引用计数管理类。目前抽象层中只有se::Object继承于se::RefCounter。
上一小节我们说到，se::Object是保存了对JS对象的弱引用，这里笔者有必要解释一下为什么是弱引用。

* 原因一：JS对象控制CPP对象的生命周期的需要

当在脚本层中通过`var sp = new cc.Sprite("a.png");`创建了一个Sprite后，在构造回调函数绑定中我们会创建一个se::Object并保留在一个全局的map(NativePtrToObjectMap)中，此map用于查询`cocos2d::Sprite*`指针获取对应的JS对象`se::Object*`。

```c++
static bool js_cocos2d_Sprite_finalize(se::State& s)
{
    CCLOG("jsbindings: finalizing JS object %p (cocos2d::Sprite)", s.nativeThisObject());
    cocos2d::Sprite* cobj = (cocos2d::Sprite*)s.nativeThisObject();
    if (cobj->getReferenceCount() == 1)
        cobj->autorelease();
    else
        cobj->release();
    return true;
}
SE_BIND_FINALIZE_FUNC(js_cocos2d_Sprite_finalize)

static bool js_cocos2dx_Sprite_constructor(se::State& s)
{
    cocos2d::Sprite* cobj = new (std::nothrow) cocos2d::Sprite(); // cobj将在finalize函数中被释放
    s.thisObject()->setPrivateData(cobj); // setPrivateData内部会去保存cobj到NativePtrToObjectMap中
    return true;
}
SE_BIND_CTOR(js_cocos2dx_Sprite_constructor, __jsb_cocos2d_Sprite_class, js_cocos2d_Sprite_finalize)
```

设想如果强制要求se::Object为JS对象的强引用(strong reference)，即让JS对象不受GC控制，由于se::Object一直存在于map中，finalize回调将永远无法被触发，从而导致内存泄露。

正是由于se::Object保存的是JS对象的弱引用，JS对象控制CPP对象的生命周期才能够实现。以上代码中，当JS对象被释放后，会触发finalize回调，开发者只需要在`js_cocos2d_Sprite_finalize`中释放对应的c++对象即可，se::Object的释放已经被包含在`SE_BIND_FINALIZE_FUNC`宏中自动处理，开发者无需管理在`JS对象控制CPP对象`模式中se::Object的释放，但是在`CPP对象控制JS对象`模式中，开发者需要管理对se::Object的释放，具体下一节中会举例说明。

* 原因二：更加灵活，手动调用root方法以支持强引用

se::Object中提供了root/unroot方法供开发者调用，root会把JS对象放入到不受GC扫描到的区域，调用root后，se::Object就强引用了JS对象，只有当unroot被调用，或者se::Object被释放后，JS对象才会放回到受GC扫描到的区域。

一般情况下，如果对象是非`cocos2d::Ref`的子类，会采用CPP对象控制JS对象的生命周期的方式去绑定。引擎内spine, dragonbones, box2d，anysdk等第三方库的绑定就是采用此方式。当CPP对象被释放的时候，需要在NativePtrToObjectMap中查找对应的se::Object，然后手动unroot和decRef。以spine中spTrackEntry的绑定为例：

```c++
spTrackEntry_setDisposeCallback([](spTrackEntry* entry){
        // spTrackEntry的销毁回调
        auto cleanup = [entry](){

            if (!se::ScriptEngine::getInstance()->isValid())
                return;

            se::AutoHandleScope hs;
            se::ScriptEngine::getInstance()->clearException();

            auto iter = se::NativePtrToObjectMap::find(entry);
            if (iter != se::NativePtrToObjectMap::end())
            {
                CCLOG("spTrackEntry %p was recycled!", entry);
                se::Object* seObj = iter->second;
                seObj->clearPrivateData(); // 解除mapping关系
                seObj->unroot(); // unroot，使JS对象受GC管理
                seObj->decRef(); // 释放se::Object
            }
        };

        // 确保不再垃圾回收中去操作JS引擎的API
        if (!se::ScriptEngine::getInstance()->isGarbageCollecting())
        {
            cleanup();
        }
        else
        { // 如果在垃圾回收，把清理任务放在帧结束中进行
            CleanupTask::pushTaskToAutoReleasePool(cleanup);
        }
    });
```


__对象类型__

绑定对象的创建已经被隐藏在对应的`SE_BIND_CTOR`和`SE_BIND_SUB_CLS_CTOR`函数中，开发者在绑定回调中如果需要用到当前对象对应的se::Object，只需要通过s.thisObject()即可获取。其中s为se::State类型，具体会在后续章节中说明。

此外，se::Object目前支持以下几种对象的手动创建：

* Plain Object : 通过se::Object::createPlainObject创建, 类似JS中的`var a = {};`
* Array Object : 通过se::Object::createArrayObject创建，类似JS中的`var a = [];`
* Uint8 Typed Array Object : 通过se::Object::createUint8TypedArray创建，类似JS中的`var a = new Uint8Array(buffer);`
* Array Buffer Object : 通过se::Object::createArrayBufferObject，类似JS中的`var a = new ArrayBuffer(len);`

__手动创建对象的释放__

se::Object::createXXX方法与cocos2d-x中的create方法不同，抽象层是完全独立的一个模块，并不依赖与cocos2d-x的autorelease机制。虽然se::Object也是继承引用计数类，但开发者需要处理**手动创建出来的对象**的释放。

```c++
se::Object* obj = se::Object::createPlainObject();
...
...
obj->decRef(); // 释放引用，避免内存泄露
```

#### se::HandleObject (推荐的管理手动创建对象的辅助类)

* 在比较复杂的逻辑中使用手动创建对象，开发者往往会忘记在不同的逻辑中处理decRef

```c++
bool foo()
{
	se::Object* obj = se::Object::createPlainObject();
	if (var1)
		return false; // 这里直接返回了，忘记做decRef释放操作
	
	if (var2)
		return false; // 这里直接返回了，忘记做decRef释放操作
	...
	...
	obj->decRef();
	return true;
}
```

就算在不同的返回条件分支中加上了decRef也会导致逻辑复杂，难以维护，如果后期加入另外一个返回分支，很容易忘记decRef。

* JS引擎在se::Object::createXXX后，如果由于某种原因JS引擎做了GC操作，导致后续使用的se::Object内部引用了一个非法指针，引发程序崩溃

为了解决上述两个问题，抽象层定义了一个辅助管理**手动创建对象**的类型，即se::handleObject。

se::HandleObject是一个辅助类，用于更加简单地管理手动创建的se::Object对象的释放、root和unroot操作。
以下两种代码写法是等价的，使用se::HandleObject的代码量明显少很多，而且更加安全。

```c++
    {
        se::HandleObject obj(se::Object::createPlainObject());
        obj->setProperty(...);
        otherObject->setProperty("foo", se::Value(obj));
    }
 
	等价于：

    {
        se::Object* obj = se::Object::createPlainObject();
        obj->root(); // 在手动创建完对象后立马root，防止对象被GC

        obj->setProperty(...);
        otherObject->setProperty("foo", se::Value(obj));
        
        obj->unroot(); // 当对象被使用完后，调用unroot
        obj->decRef(); // 引用计数减一，避免内存泄露
    }
```

注意：

* 不要尝试使用se::HandleObject创建一个native与JS的绑定对象，在JS控制CPP的模式中，绑定对象的释放会被抽象层自动处理，在CPP控制JS的模式中，前一章节中已经有描述了。
* se::HandleObject对象只能够在栈上被分配，而且栈上构造的时候必须传入一个se::Object指针。


#### se::Class

se::Class用于暴露CPP类到JS中，它会在JS中创建一个对应名称的constructor function。

它有如下方法：

* `static se::Class* create(className, obj, parentProto, ctor)`: 创建一个Class，注册成功后，在JS层中可以通过`var xxx = new SomeClass();`的方式创建一个对象
* `bool defineFunction(name, func)`: 定义Class中的成员函数
* `bool defineProperty(name, getter, setter)`: 定义Class属性读写器
* `bool defineStaticFunction(name, func)`: 定义Class的静态成员函数，可通过SomeClass.foo()这种非new的方式访问，与类实例对象无关
* `bool defineStaticProperty(name, getter, setter)`: 定义Class的静态属性读写器，可通过SomeClass.propertyA直接读写，与类实例对象无关
* `bool defineFinalizeFunction(func)`: 定义JS对象被GC后的CPP回调
* `bool install()`: 注册此类到JS虚拟机中
* `Object* getProto()`: 获取注册到JS中的类(其实是JS的constructor)的prototype对象，类似function Foo(){}的Foo.prototype
* `const char* getName() const`: 获取当前Class的名称

**注意：**

Class类型创建后，不需要手动释放内存，它会被封装层自动处理。

更具体API说明可以翻看API文档或者代码注释

#### se::AutoHandleScope

se::AutoHandleScope对象类型完全是为了解决V8的兼容问题而引入的概念。
V8中，当有CPP函数中需要触发JS相关操作，比如调用JS函数，访问JS属性等任何调用v8::Local<>的操作，V8强制要求在调用这些操作前必须存在一个v8::HandleScope作用域，否则会引发程序崩溃。

因此抽象层中引入了se::AutoHandleScope的概念，其只在V8上有实现，其他JS引擎目前都只是空实现。

开发者需要记住，在任何代码执行中，需要调用JS的逻辑前，声明一个se::AutoHandleScope即可，比如：

```c++
class SomeClass {
	void update(float dt) {
		se::ScriptEngine::getInstance()->clearException();
		se::AutoHandleScope hs;
		
		se::Object* obj = ...;
		obj->setProperty(...);
		...
		...
		obj->call(...);
	}
};
```

#### se::State

之前章节我们有提及State类型，它是绑定回调中的一个环境，我们通过se::State可以取得当前的CPP指针、se::Object对象指针、参数列表、返回值引用。

```c++
bool foo(se::State& s)
{
	// 获取native对象指针
	SomeClass* cobj = (SomeClass*)s.nativeThisObject();
	// 获取se::Object对象指针
	se::Object* thisObject = s.thisObject();
	// 获取参数列表
	const se::ValueArray& args = s.args();
	// 设置返回值
	s.rval().setInt32(100);
	return true;
}
SE_BIND_FUNC(foo)
```

## 抽象层依赖Cocos引擎么？

不依赖。

ScriptEngine这层设计之初就将其定义为一个独立模块，完全不依赖Cocos引擎。开发者完整可以通过copy、paste把cocos/scripting/js-bindings/jswrapper下的所有抽象层源码拷贝到其他项目中直接使用。

## 手动绑定

### 回调函数声明

```c++
static bool Foo_balabala(se::State& s)
{
	const auto& args = s.args();
	int argc = (int)args.size();
	
	if (argc >= 2) // 这里约定参数个数必须大于等于2，否则抛出错误到JS层且返回false
	{
		...
		...
		return true;
	}
	
	SE_REPORT_ERROR("wrong number of arguments: %d, was expecting %d", argc, 2);
	return false;
}

// 如果是绑定函数，则用SE_BIND_FUNC，构造函数、析构函数、子类构造函数等类似
SE_BIND_FUNC(Foo_balabala)
```

### 为JS对象设置一个属性值

```c++
se::Object* globalObj = se::ScriptEngine::getInstance()->getGlobalObject(); // 这里为了演示方便，获取全局对象
globalObj->setProperty("foo", se::Value(100)); // 给全局对象设置一个foo属性，值为100
```

在JS中就可以直接使用foo这个全局变量了

```js
cc.log("foo value: " + foo); // 打印出 foo value: 100
```

### 为JS对象定义一个属性读写回调

```c++
// 全局对象的foo属性的读回调
static bool Global_get_foo(se::State& s)
{
	NativeObj* cobj = (NativeObj*)s.nativeThisObject();
	int32_t ret = cobj->getValue();
	s.rval().setInt32(ret);
	return true;
}
SE_BIND_PROP_GET(Global_get_foo)

// 全局对象的foo属性的写回调
static bool Global_set_foo(se::State& s)
{
	const auto& args = s.args();
	int argc = (int)args.size();
	if (argc >= 1)
	{
		NativeObj* cobj = (NativeObj*)s.nativeThisObject();
		int32_t arg1 = args[0].toInt32();
		cobj->setValue(arg1);
		// void类型的函数，无需设置s.rval，未设置默认返回undefined给JS层
		return true;
	}

	SE_REPORT_ERROR("wrong number of arguments: %d, was expecting %d", argc, 1);
	return false;
}
SE_BIND_PROP_SET(Global_set_foo)

void some_func()
{
	se::Object* globalObj = se::ScriptEngine::getInstance()->getGlobalObject(); // 这里为了演示方便，获取全局对象
	globalObj->defineProperty("foo", _SE(Global_get_foo), _SE(Global_set_foo)); // 使用_SE宏包装一下具体的函数名称
}
```

### 为JS对象设置一个函数

```c++
static bool Foo_function(se::State& s)
{
	...
	...
}
SE_BIND_FUNC(Foo_function)

void some_func()
{
	se::Object* globalObj = se::ScriptEngine::getInstance()->getGlobalObject(); // 这里为了演示方便，获取全局对象
	globalObj->defineFunction("foo", _SE(Foo_function)); // 使用_SE宏包装一下具体的函数名称
}

```

### 注册一个CPP类到JS虚拟机中

```c++
static se::Object* __jsb_ns_SomeClass_proto = nullptr;
static se::Class* __jsb_ns_SomeClass_class = nullptr;

namespace ns {
    class SomeClass
    {
    public:
        SomeClass()
        : xxx(0)
        {}

        void foo() {
            printf("SomeClass::foo\n");
            
            Director::getInstance()->getScheduler()->schedule([this](float dt){
                static int counter = 0;
                ++counter;
                if (_cb != nullptr)
                    _cb(counter);
            }, this, 1.0f, CC_REPEAT_FOREVER, 0.0f, false, "iamkey");
        }

        static void static_func() {
            printf("SomeClass::static_func\n");
        }

        void setCallback(const std::function<void(int)>& cb) {
            _cb = cb;
            if (_cb != nullptr)
            {
                printf("setCallback(cb)\n");
            }
            else
            {
                printf("setCallback(nullptr)\n");
            }
        }

        int xxx;
    private:
        std::function<void(int)> _cb;
    };
} // namespace ns {

static bool js_SomeClass_finalize(se::State& s)
{
    return true;
}
SE_BIND_FINALIZE_FUNC(js_SomeClass_finalize)

static bool js_SomeClass_constructor(se::State& s)
{
    ns::SomeClass* cobj = new ns::SomeClass();
    s.thisObject()->setPrivateData(cobj);
    return true;
}
SE_BIND_CTOR(js_SomeClass_constructor, __jsb_ns_SomeClass_class, js_SomeClass_finalize)

static bool js_SomeClass_foo(se::State& s)
{
    ns::SomeClass* cobj = (ns::SomeClass*)s.nativeThisObject();
    cobj->foo();
    return true;
}
SE_BIND_FUNC(js_SomeClass_foo)

static bool js_SomeClass_get_xxx(se::State& s)
{
    ns::SomeClass* cobj = (ns::SomeClass*)s.nativeThisObject();
    s.rval().setInt32(cobj->xxx);
    return true;
}
SE_BIND_PROP_GET(js_SomeClass_get_xxx)

static bool js_SomeClass_set_xxx(se::State& s)
{
    const auto& args = s.args();
    int argc = (int)args.size();
    if (argc > 0)
    {
        ns::SomeClass* cobj = (ns::SomeClass*)s.nativeThisObject();
        cobj->xxx = args[0].toInt32();
        return true;
    }

    SE_REPORT_ERROR("wrong number of arguments: %d, was expecting %d", argc, 1);
    return false;
}
SE_BIND_PROP_SET(js_SomeClass_set_xxx)

static bool js_SomeClass_static_func(se::State& s)
{
    ns::SomeClass::static_func();
    return true;
}
SE_BIND_FUNC(js_SomeClass_static_func)

bool js_register_ns_SomeClass(se::Object* global)
{
    // 保证namespace对象存在
    se::Value nsVal;
    if (!global->getProperty("ns", &nsVal))
    {
        // 不存在则创建一个JS对象，相当于 var ns = {};
        se::HandleObject jsobj(se::Object::createPlainObject());
        nsVal.setObject(jsobj);

        // 将ns对象挂载到global对象中，名称为ns
        global->setProperty("ns", nsVal);
    }
    se::Object* ns = nsVal.toObject();

    // 创建一个Class对象，开发者无需考虑Class对象的释放，其交由ScriptEngine内部自动处理
    auto cls = se::Class::create("SomeClass", ns, nullptr, _SE(js_SomeClass_constructor)); // 如果无构造函数，最后一个参数可传入nullptr，则这个类在JS中无法被new SomeClass()出来

    // 为这个Class对象定义成员函数、属性、静态函数、析构函数
    cls->defineFunction("foo", _SE(js_SomeClass_foo));
    cls->defineProperty("xxx", _SE(js_SomeClass_get_xxx), _SE(js_SomeClass_set_xxx));

    cls->defineFinalizeFunction(_SE(js_SomeClass_finalize));

    // 注册类型到JS VirtualMachine的操作
    cls->install();

    // JSBClassType为Cocos引擎绑定层封装的类型注册的辅助函数，此函数不属于ScriptEngine这层
    JSBClassType::registerClass<ns::SomeClass>(cls);

    // 保存注册的结果，便于其他地方使用，比如类继承
    __jsb_ns_SomeClass_proto = cls->getProto();
    __jsb_ns_SomeClass_class = cls;

    // 为每个此Class实例化出来的对象附加一个属性
    __jsb_ns_SomeClass_proto->setProperty("yyy", se::Value("helloyyy"));

    // 注册静态成员变量和静态成员函数
    se::Value ctorVal;
    if (ns->getProperty("SomeClass", &ctorVal) && ctorVal.isObject())
    {
        ctorVal.toObject()->setProperty("static_val", se::Value(200));
        ctorVal.toObject()->defineFunction("static_func", _SE(js_SomeClass_static_func));
    }

    // 清空异常
    se::ScriptEngine::getInstance()->clearException();
    return true;
}
```

### 如何绑定CPP接口中的回调函数？

```c++
static bool js_SomeClass_setCallback(se::State& s)
{
    const auto& args = s.args();
    int argc = (int)args.size();
    if (argc >= 1)
    {
        ns::SomeClass* cobj = (ns::SomeClass*)s.nativeThisObject();

        se::Value jsFunc = args[0];
        se::Value jsTarget = argc > 1 ? args[1] : se::Value::Undefined;

        if (jsFunc.isNullOrUndefined())
        {
            cobj->setCallback(nullptr);
        }
        else
        {
            assert(jsFunc.isObject() && jsFunc.toObject()->isFunction());

            // 如果当前SomeClass是可以被new出来的类，我们 使用se::Object::attachObject把jsFunc和jsTarget关联到当前对象中
            s.thisObject()->attachObject(jsFunc.toObject());
            s.thisObject()->attachObject(jsTarget.toObject());

            // 如果当前SomeClass类是一个单例类，或者永远只有一个实例的类，我们不能用se::Object::attachObject去关联
            // 必须使用se::Object::root，开发者无需关系unroot，unroot的操作会随着lambda的销毁触发jsFunc的析构，在se::Object的析构函数中进行unroot操作。
            // js_cocos2dx_EventDispatcher_addCustomEventListener的绑定代码就是使用此方式，因为EventDispatcher始终只有一个实例，
            // 如果使用s.thisObject->attachObject(jsFunc.toObject);会导致对应的func和target永远无法被释放，引发内存泄露。

            // jsFunc.toObject()->root();
            // jsTarget.toObject()->root();

            cobj->setCallback([jsFunc, jsTarget](int counter){

                // CPP回调函数中要传递数据给JS或者调用JS函数，在回调函数开始需要添加如下两行代码。
                se::ScriptEngine::getInstance()->clearException();
                se::AutoHandleScope hs;

                se::ValueArray args;
                args.push_back(se::Value(counter));

                se::Object* target = jsTarget.isObject() ? jsTarget.toObject() : nullptr;
                jsFunc.toObject()->call(args, target);
            });
        }

        return true;
    }

    SE_REPORT_ERROR("wrong number of arguments: %d, was expecting %d", argc, 1);
    return false;
}
SE_BIND_FUNC(js_SomeClass_setCallback)
```

SomeClass类注册后，就可以在JS中这样使用了：

```js
 var myObj = new ns.SomeClass();
 myObj.foo();
 ns.SomeClass.static_func();
 cc.log("ns.SomeClass.static_val: " + ns.SomeClass.static_val);
 cc.log("Old myObj.xxx:" + myObj.xxx);
 myObj.xxx = 1234;
 cc.log("New myObj.xxx:" + myObj.xxx);
 cc.log("myObj.yyy: " + myObj.yyy);

 var delegateObj = {
     onCallback: function(counter) {
         cc.log("Delegate obj, onCallback: " + counter + ", this.myVar: " + this.myVar);
         this.setVar();
     },

     setVar: function() {
         this.myVar++;
     },

     myVar: 100
 };

 myObj.setCallback(delegateObj.onCallback, delegateObj);

 setTimeout(function(){
    myObj.setCallback(null);
 }, 6000); // 6秒后清空callback
```

Console中会输出：

```
SomeClass::foo
SomeClass::static_func
ns.SomeClass.static_val: 200
Old myObj.xxx:0
New myObj.xxx:1234
myObj.yyy: helloyyy
setCallback(cb)
Delegate obj, onCallback: 1, this.myVar: 100
Delegate obj, onCallback: 2, this.myVar: 101
Delegate obj, onCallback: 3, this.myVar: 102
Delegate obj, onCallback: 4, this.myVar: 103
Delegate obj, onCallback: 5, this.myVar: 104
Delegate obj, onCallback: 6, this.myVar: 105
setCallback(nullptr)
```

### 

## 自动绑定

### 配置模块ini文件

配置方法与1.6中的方法相同，主要注意的是：1.7中废弃了`script_control_cpp`，因为`script_control_cpp`字段会影响到整个模块，如果模块中需要绑定cocos2d::Ref子类和非cocos::Ref子类，原来的绑定配置则无法满足需求。1.7中取而代之的新字段为`classes_owned_by_cpp`，表示哪些类是需要由CPP来控制JS对象的生命周期。

1.7中另外加入的一个配置字段为`persistent_classes`, 用于表示哪些类是在游戏运行中一直存在的，比如：TextureCache SpriteFrameCache FileUtils EventDispatcher ActionManager Scheduler

其他字段与1.6一致。

具体可以参考引擎目录下的tools/tojs/cocos2dx.ini等ini配置。

### 理解ini文件中每个字段的意义

```
# 模块名称
[cocos2d-x] 

# 绑定回调函数的前缀，也是生成的自动绑定文件的前缀
prefix = cocos2dx

# 绑定的类挂载在JS中的哪个对象中，类似命名空间
target_namespace = cc

# 自动绑定工具基于Android编译环境，此处配置Android头文件搜索路径
android_headers = -I%(androidndkdir)s/platforms/android-14/arch-arm/usr/include -I%(androidndkdir)s/sources/cxx-stl/gnu-libstdc++/4.8/libs/armeabi-v7a/include -I%(androidndkdir)s/sources/cxx-stl/gnu-libstdc++/4.8/include -I%(androidndkdir)s/sources/cxx-stl/gnu-libstdc++/4.9/libs/armeabi-v7a/include -I%(androidndkdir)s/sources/cxx-stl/gnu-libstdc++/4.9/include

# 配置Android编译参数
android_flags = -D_SIZE_T_DEFINED_

# 配置clang头文件搜索路径
clang_headers = -I%(clangllvmdir)s/%(clang_include)s

# 配置clang编译参数
clang_flags = -nostdinc -x c++ -std=c++11 -U __SSE__

# 配置引擎的头文件搜索路径
cocos_headers = -I%(cocosdir)s/cocos -I%(cocosdir)s/cocos/platform/android -I%(cocosdir)s/external/sources

# 配置引擎编译参数
cocos_flags = -DANDROID

# 配置额外的编译参数
extra_arguments = %(android_headers)s %(clang_headers)s %(cxxgenerator_headers)s %(cocos_headers)s %(android_flags)s %(clang_flags)s %(cocos_flags)s %(extra_flags)s
 
# 需要自动绑定工具解析哪些头文件
headers = %(cocosdir)s/cocos/cocos2d.h %(cocosdir)s/cocos/scripting/js-bindings/manual/BaseJSAction.h

# 在生成的绑定代码中，重命名头文件
replace_headers=CCProtectedNode.h::2d/CCProtectedNode.h,CCAsyncTaskPool.h::base/CCAsyncTaskPool.h

# 需要绑定哪些类，可以使用正则表达式，以空格为间隔
classes = 

# 哪些类需要在JS层通过cc.Class.extend，以空格为间隔
classes_need_extend = 

# 需要为哪些类绑定属性，以逗号为间隔
field = Acceleration::[x y z timestamp]

# 需要忽略绑定哪些类，以逗号为间隔
skip = AtlasNode::[getTextureAtlas],
       ParticleBatchNode::[getTextureAtlas],

# 重命名函数，以逗号为间隔
rename_functions = ComponentContainer::[get=getComponent],
                   LayerColor::[initWithColor=init],

# 重命名类，以逗号为间隔
rename_classes = SimpleAudioEngine::AudioEngine,
                 SAXParser::PlistParser,


# 配置哪些类不需要搜索其父类
classes_have_no_parents = Node Director SimpleAudioEngine FileUtils TMXMapInfo Application GLViewProtocol SAXParser Configuration

# 配置哪些父类需要被忽略
base_classes_to_skip = Ref Clonable

# 配置哪些类是抽象类，抽象类没有构造函数，即在js层无法通过var a = new SomeClass();的方式构造JS对象
abstract_classes = Director SpriteFrameCache Set SimpleAudioEngine

# 配置哪些类是始终以一个实例的方式存在的，游戏运行过程中不会被销毁
persistent_classes = TextureCache SpriteFrameCache FileUtils EventDispatcher ActionManager Scheduler

# 配置哪些类是需要由CPP对象来控制JS对象生命周期的，未配置的类，默认采用JS控制CPP对象生命周期
classes_owned_by_cpp = 
```

## 远程调试

### Chrome远程调试V8

TBD

Windows:

Android:

### Safari远程调试JavaScriptCore

TBD

Mac:

iOS:


## Q & A

### se::ScriptEngine与ScriptingCore的区别，为什么还要保留ScriptingCore?

在1.7中，抽象层被设计为一个与引擎没有关系的独立模块，对JS引擎的管理从ScriptingCore被移动到了se::ScriptEngine类中，ScriptingCore被保留下来是希望通过它把引擎的一些事件传递给封装层，充当适配器的角色。

ScriptingCore只需要在AppDelegate中被使用一次即可, 之后的所有操作都只需要用到se::ScriptEngine。

```c++
bool AppDelegate::applicationDidFinishLaunching()
{
	...
	...
    director->setAnimationInterval(1.0 / 60);

    // 这两行把ScriptingCore这个适配器设置给引擎，用于传递引擎的一些事件，
    // 比如Node的onEnter, onExit, Action的update，JS对象的持有与解除持有
    ScriptingCore* sc = ScriptingCore::getInstance();
    ScriptEngineManager::getInstance()->setScriptEngine(sc);
    
    se::ScriptEngine* se = se::ScriptEngine::getInstance();
    ...
    ...
}
```
### se::Object::root/unroot 与 se::Object::incRef/decRef的区别?

TBD

### 对象生命周期的关联与解除关联

se::Object::attachObject/dettachObject

`objA->attachObject(objB);`类似于JS中执行`objA.__nativeRefs[index] = objB`，只有当objA被GC后，objB才有可能被GC
`objA->dettachObject(objB);`类似于JS中执行`delete objA.__nativeRefs[index];`，这样objB的生命周期就不受objA控制了

### cocos2d::Ref子类与非cocos2d::Ref子类JS/CPP对象生命周期管理有何不同？

目前引擎中cocos2d::Ref子类的绑定采用JS对象控制CPP对象生命周期的方式，这样做的好处是，解决了一直以来被诟病的需要在JS层retain，release对象的烦恼。

非cocos2d::Ref子类采用CPP对象控制JS对象生命周期的方式。此方式要求，CPP对象销毁后需要通知绑定层去调用对应se::Object的clearPrivateData, unroot, decRef的方法。JS代码中一定要慎重操作对象，当有可能出现非法对象的逻辑中，使用cc.sys.isObjectValid来判断CPP对象是否被释放了。

### 绑定cocos2d::Ref子类的析构函数需要注意的事项

如果在JS对象的finalize回调中调用任何JS引擎的API，可能导致崩溃。因为当前引擎正在进行垃圾回收的流程，无法被打断处理其他操作。
finalize回调中是告诉CPP层是否对应的CPP对象的内存，不能在CPP对象的析构中又去操作JS引擎API。

那如果必须调用，应该如何处理？

cocos2d-x的绑定中，如果引用计数为1了，我们不使用release，而是使用autorelease延时CPP类的析构到帧结束去执行。

```c++
static bool js_cocos2d_Sprite_finalize(se::State& s)
{
    CCLOG("jsbindings: finalizing JS object %p (cocos2d::Sprite)", s.nativeThisObject());
    cocos2d::Sprite* cobj = (cocos2d::Sprite*)s.nativeThisObject();
    if (cobj->getReferenceCount() == 1)
        cobj->autorelease();
    else
        cobj->release();
    return true;
}
SE_BIND_FINALIZE_FUNC(js_cocos2d_Sprite_finalize)
```

### 如何监听脚本错误

在AppDelegate.cpp中通过se::ScriptEngine::getInstance()->setExceptionCallback(...)设置JS层异常回调。

```c++
bool AppDelegate::applicationDidFinishLaunching()
{
	...
	...
	se::ScriptEngine* se = se::ScriptEngine::getInstance();

    se->setExceptionCallback([](const char* location, const char* message, const char* stack){
        // Send exception information to server like Tencent Bugly.
        // ...
        // ...
    });

    jsb_register_all_modules();
	...
	...
	return true;
}

```




