---
date: '2026-08-19T14:00:00+08:00'
draft: false
title: 'Godot引擎生成MonoGlue时崩溃的问题'
tags: ["Godot", "C++"]
---

## 问题描述

最近刚更新到`Godot4.7.3rc dotnet`版本，本机环境为windows11，发现在手动编译引擎后的`generate-mono-glue`的最后会崩溃：
```
ERROR: FATAL: Condition "!is_main_thread_assigned.clear_if_set()" is true.
   at: Thread::release_main_thread (core\os\thread.cpp:101)

================================================================
CrashHandlerException: Program crashed
Engine version: Godot Engine v4.7.3.rc.mono.custom_build (afdda6ef2c7717efb49f9cf1a412f6d74154fd21)
Dumping the backtrace.
Load address: 7ff697d50000

[0] 7ff69d15f3a7 (main+540f3a7) - Thread::release_main_thread (D:\GameDev\GodotEngine\godot\core\os\thread.cpp:101)
[1] 7ff69d15f3a7 (main+540f3a7) - Thread::release_main_thread (D:\GameDev\GodotEngine\godot\core\os\thread.cpp:101)
[2] 7ff697df6422 (main+a6422) - Main::cleanup (D:\GameDev\GodotEngine\godot\main\main.cpp:5406)
[3] 7ff698e365ab (main+10e65ab) - cleanup_and_exit_godot (D:\GameDev\GodotEngine\godot\modules\mono\editor\bindings_generator.cpp:5273)
[4] 7ff698e3907d (main+10e907d) - BindingsGenerator::handle_cmdline_args (D:\GameDev\GodotEngine\godot\modules\mono\editor\bindings_generator.cpp:5309)
[5] 7ff697e0696b (main+b696b) - Main::setup2 (D:\GameDev\GodotEngine\godot\main\main.cpp:3892)
[6] 7ff697e1802d (main+c802d) - Main::setup (D:\GameDev\GodotEngine\godot\main\main.cpp:2936)
[7] 7ff697dda6cb (main+8a6cb) - widechar_main (D:\GameDev\GodotEngine\godot\platform\windows\godot_windows.cpp:90)
[8] 7ff697dda537 (main+8a537) - _main (D:\GameDev\GodotEngine\godot\platform\windows\godot_windows.cpp:134)
[9] 7ff697dda7a9 (main+8a7a9) - main (D:\GameDev\GodotEngine\godot\platform\windows\godot_windows.cpp:146)
[10] 7ff69dbf0a89 (main+5ea0a89) - __scrt_common_main_seh (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288)
[11] 7ff8a08eccb7 (kernel32.dll+2ccb7) - BaseThreadInitThunk
-- END OF C++ BACKTRACE --
================================================================
```

## 深入调查

查看源码后发现，似乎是`Thread::release_main_thread()`里触发assert，`is_main_thread_assigned` 标志在释放主线程时没有被set。随后我翻阅git commit，发现有一个相关的提交：`Make it impossible to have more than one main thread, and don't release unnecessarily - #121161#121161`。分析以后发现，极有可能是`make_main_thread()`里没有正确set`is_main_thread_assigned`。

在`make_main_thread` & `release_main_thread`两个函数中输出日志
```cpp
void Thread::make_main_thread() {
	fprintf(stderr, "[PROBE] make_main_thread: caller_id=%llu flag=%d id_counter=%llu\n",
			(unsigned long long)caller_id, is_main_thread_assigned.is_set() ? 1 : 0,
			(unsigned long long)id_counter.get());
	if (caller_id == MAIN_ID) {
		return; // We're already the main thread
	}
	CRASH_COND_MSG(!is_main_thread_assigned.set_if_clear(), "A second thread attempted to become the main thread.");
	caller_id = MAIN_ID;
}

void Thread::release_main_thread() {
	fprintf(stderr, "[PROBE] release_main_thread: caller_id=%llu flag=%d id_counter=%llu\n",
			(unsigned long long)caller_id, is_main_thread_assigned.is_set() ? 1 : 0,
			(unsigned long long)id_counter.get());
	CRASH_COND_MSG(caller_id != MAIN_ID, "Trying to release main thread from a thread that isn't main.");
	CRASH_COND(!is_main_thread_assigned.clear_if_set());
	caller_id = id_counter.increment();
}
```
验证结果如下：
```
[PROBE] make_main_thread: caller_id=1 flag=0 id_counter=42
[PROBE] release_main_thread: caller_id=1 flag=0 id_counter=42
ERROR: FATAL: Condition "!is_main_thread_assigned.clear_if_set()" is true.
   at: Thread::release_main_thread (core\os\thread.cpp:107)
```

继续结合`Thread.cpp`里的代码：
```cpp
SafeNumeric<uint64_t> Thread::id_counter(1); // The first value after .increment() is 2, hence by default the main thread ID should be 1.
thread_local Thread::ID Thread::caller_id = Thread::id_counter.increment();
```

为什么`caller_id=1`？按理说应该是2，然后在`make_main_thread`中执行`is_main_thread_assigned.set_if_clear()`操作，这样就完美符合预期。

经过deepseek老师的帮助，推测可能是`caller_id`先于`id_counter`初始化导致的问题。先修改代码：

```cpp
uint64_t _get_id_counter_inc(SafeNumeric<uint64_t>& idc) {
	return idc.increment();
}
SafeNumeric<uint64_t> Thread::id_counter(1);
thread_local Thread::ID Thread::caller_id = _get_id_counter_inc(id_counter);

```

然后在main函数第一行断点，结果如下：

```
[breakpoint-hit at main first line]

thread:
main thread

variables:
Thread::caller_id	1	unsigned __int64
Thread::id_counter	{value=1 }	SafeNumeric<unsigned __int64>


[breakpoint-hit when id_counter value change to 2]

thread:
Not Flagged	>	34260	0	Worker Thread	ntdll.dll thread	godot.windows.editor.x86_64.mono.exe!`dynamic initializer for 'Thread::caller_id''

variables:
Thread::caller_id	0	unsigned __int64
Thread::id_counter	{value=2 }	SafeNumeric<unsigned __int64>

callstack:
>	godot.windows.editor.x86_64.mono.exe!`dynamic initializer for 'Thread::caller_id''()	C++
 	godot.windows.editor.x86_64.mono.exe!__dyn_tls_init(void * __formal, unsigned long dwReason, void * __formal) Line 95	C++
 	ntdll.dll!00007ff8a168471b()	Unknown
 	ntdll.dll!00007ff8a16fe06a()	Unknown
 	ntdll.dll!00007ff8a15df723()	Unknown
 	ntdll.dll!00007ff8a15dface()	Unknown
 	ntdll.dll!00007ff8a15df5c2()	Unknown
 	ntdll.dll!00007ff8a164eb8b()	Unknown
 	ntdll.dll!00007ff8a164ea3a()	Unknown
 	ntdll.dll!00007ff8a15cc21e()	Unknown


[breakpoint-hit when id_counter value change to 3]

thread:
Not Flagged	>	9952	0	Worker Thread	InputHost.dll thread	godot.windows.editor.x86_64.mono.exe!`dynamic initializer for 'Thread::caller_id''

variables:
Thread::caller_id	0	unsigned __int64
Thread::id_counter	{value=3 }	SafeNumeric<unsigned __int64>

callstack:
>	godot.windows.editor.x86_64.mono.exe!`dynamic initializer for 'Thread::caller_id''()	C++
 	godot.windows.editor.x86_64.mono.exe!__dyn_tls_init(void * __formal, unsigned long dwReason, void * __formal) Line 95	C++
 	ntdll.dll!00007ff8a168471b()	Unknown
 	ntdll.dll!00007ff8a16fe06a()	Unknown
 	ntdll.dll!00007ff8a15df723()	Unknown
 	ntdll.dll!00007ff8a15dface()	Unknown
 	ntdll.dll!00007ff8a15df5c2()	Unknown
 	ntdll.dll!00007ff8a164eb8b()	Unknown
 	ntdll.dll!00007ff8a164ea3a()	Unknown
 	ntdll.dll!00007ff8a15cc21e()	Unknown


[breakpoint-hit when id_counter value change to 4]

thread:
main thread

variables:
Thread::caller_id	1	unsigned __int64
Thread::id_counter	{value=4 }	SafeNumeric<unsigned __int64>

callstack:
>	[Inline Frame] godot.windows.editor.x86_64.mono.exe!SafeNumeric<unsigned __int64>::increment() Line 77	C++
 	[Inline Frame] godot.windows.editor.x86_64.mono.exe!_get_id_counter_inc(SafeNumeric<unsigned __int64> &) Line 44	C++
 	godot.windows.editor.x86_64.mono.exe!Thread::start(void(*)(void *) p_callback, void * p_user, const Thread::Settings & p_settings) Line 80	C++
 	godot.windows.editor.x86_64.mono.exe!IP::IP() Line 346	C++
 	[Inline Frame] godot.windows.editor.x86_64.mono.exe!IPWindows::{ctor}() Line 155	C++
 	godot.windows.editor.x86_64.mono.exe!IPWindows::_create_unix() Line 152	C++
 	godot.windows.editor.x86_64.mono.exe!register_core_types() Line 333	C++
 	godot.windows.editor.x86_64.mono.exe!Main::setup(const char * execpath, int argc, char * * argv, bool p_second_phase) Line 1068	C++
 	godot.windows.editor.x86_64.mono.exe!widechar_main(int argc, wchar_t * * argv) Line 90	C++
 	godot.windows.editor.x86_64.mono.exe!_main() Line 134	C++
 	godot.windows.editor.x86_64.mono.exe!main(int argc, char * * argv) Line 148	C++
 	[External Code]
```

在第一个断点位置，就有问题：`caller_id`和`id_counter`都是1，按理说都应该是2才对。这个情况进一步印证我的假设：`id_counter`初始化前为0，此时`caller_id`先初始化为1，然后`id_counter`初始化赋值为1。

继续验证，修改代码如下：
```cpp
static uint64_t _get_id_counter_inc(SafeNumeric<uint64_t> &idc) {
	fprintf(stderr, "[Test] id_counter=%llu\n", (unsigned long long)idc.get());
	idc.increment();
	idc.increment();
	idc.increment();
	idc.increment();
	return idc.increment();
}
SafeNumeric<uint64_t> Thread::id_counter(99); // The first value after .increment() is 2, hence by default the main thread ID should be 1.
//thread_local Thread::ID Thread::caller_id = Thread::id_counter.increment();
thread_local Thread::ID Thread::caller_id = _get_id_counter_inc(id_counter);
```

运行后在main第一行断点发现变量结果如下，完美验证猜想：
```
Thread::caller_id	5	unsigned __int64
Thread::id_counter	{value=99 }	SafeNumeric<unsigned __int64>
```

现在的问题在于，为什么`caller_id`先于`id_counter`初始化？我尝试给`id_counter`改为用函数的返回值初始化，然后在这个函数里断点。遗憾的是因为开启了编译优化，导致断点无效。我又尝试关闭编译优化，继续断点，此时发现这个bug不会复现。这个问题只能先搁置了。

## 后续发现

经过检查对比，最终发现问题出在tracy上，只要我将tracy加入编译，就会导致初始化顺序出错。复现如下：
```
scons platform=windows vsproj=yes dev_build=yes debug_symbols=yes d3d12=yes module_mono_enabled=yes profiler=tracy profiler_path=D:\GameDev\GodotEngine\godot\tracy_src accesskit=no angle=no use
msbuild godot.sln /p:Configuration=editor /p:Platform=x64
bin\godot.windows.editor.x86_64.mono.exe --headless --generate-mono-glue modules/mono/glue
```

有了这个结论以后，我进行如下修改，把id_counter包装到get_id_counter函数里然后打断点：
```cpp
SafeNumeric<uint64_t> Thread::id_counter(1); // The first value after .increment() is 2, hence by default the main thread ID should be 1.
SafeNumeric<uint64_t> &Thread::get_id_counter() {
	return Thread::id_counter;
}
thread_local Thread::ID Thread::caller_id = Thread::get_id_counter().increment();
```
可以看到断点第一次hit是这个调用栈：
```
>	godot.windows.editor.dev.x86_64.mono.exe!Thread::get_id_counter() Line 43	C++
 	godot.windows.editor.dev.x86_64.mono.exe!`dynamic initializer for 'Thread::caller_id''()	C++
 	[Inline Frame] godot.windows.editor.dev.x86_64.mono.exe!__dyn_tls_init(void *) Line 98	C++
 	godot.windows.editor.dev.x86_64.mono.exe!__dyn_tls_on_demand_init() Line 130	C++
 	godot.windows.editor.dev.x86_64.mono.exe!tracy::Profiler::Profiler() Line 1489	C++
 	godot.windows.editor.dev.x86_64.mono.exe!tracy::`dynamic initializer for 's_profiler''()	C++
 	godot.windows.editor.dev.x86_64.mono.exe!_initterm(void(*)() * first, void(*)() * last) Line 16	C++
 	godot.windows.editor.dev.x86_64.mono.exe!__scrt_common_main_seh() Line 258	C++
 	godot.windows.editor.dev.x86_64.mono.exe!ShimMainCRTStartup(...) Line 74	C
 	kernel32.dll!00007ffd9921ccb7()	Unknown
 	ntdll.dll!00007ffd9af0ad6c()	Unknown
```
以及此时`id_counter`还没有初始化为1：
```
-		Thread::id_counter	{value=0 }	SafeNumeric<unsigned __int64>
```
查看调用栈，发现是`tracy::Profiler::Profiler()`过来的，为了初始化`s_profiler`。
搜索了一下，在TracyProfiler.cpp里找到了：
```cpp
#ifdef __APPLE__
#  ifndef TRACY_DELAYED_INIT
#    define TRACY_DELAYED_INIT
#  endif
#else
#  ifdef __GNUC__
#    define init_order( val ) __attribute__ ((init_priority(val)))
#  else
#    define init_order(x)
#  endif
#endif

...

static Profiler init_order(105) s_profiler;

...

Profiler::Profiler()
{
	...
	#ifndef TRACY_DELAYED_INIT
	#  ifdef _MSC_VER
		// 3. But these variables need to be initialized in main thread within the .CRT$XCB section. Do it here.
		s_token_detail = moodycamel::ProducerToken( s_queue );  // <- line 1489
		s_token = ProducerWrapper { s_queue.get_explicit_producer( s_token_detail ) };
		s_threadHandle = ThreadHandleWrapper { m_mainThread };
	#  endif
	#endif
	...
}

```

出错步骤如下：
1. _initterm 执行 s_profiler 的动态初始化器 → 进入 Profiler::Profiler()。
2. 1489 行给 s_token_detail 赋值 → 主线程首次触发 TLS 守卫 → __dyn_tls_on_demand_init → __dyn_tls_init 批量执行该模块所有 TLS 初始化器。
3. 其中 Thread::caller_id 的初始化器（get_id_counter().increment()）被执行。
4. 此时 Thread::id_counter（当时是普通静态成员，其 _initterm 初始化器排得更靠后）还是 0 → caller_id = 1，恰好撞上MAIN_ID。
5. 后续 Main::setup 里 make_main_thread() 发现 caller_id == MAIN_ID 直接 return，没 set is_main_thread_assigned；最后release_main_thread() 的 clear_if_set() 失败 → crash。
（没有 tracy 时，主线程第一次碰 caller_id 在 main() 之后的正常代码里，那时 _initterm 已全部跑完，id_counter 已是 1，caller_id = 2，一切正常——所以只有开 tracy 才复现。）

另外，我也测试了mac版本，不会有这个bug，原因是1489~1491这三行只在windows下编译，因为MSVC的TLS惰性初始化无法用 init_order()控制，作者只能单独处理。在mac上由于不会跑1489~1491这三行，所以TLS初始化延后了。

## 修复

为了修复初始化的顺序，我采用的方法是在函数内部声明static变量，将初始化的顺序交给函数调用顺序控制。

```cpp
SafeNumeric<uint64_t> &Thread::get_id_counter() {
	static SafeNumeric<uint64_t> id_counter(1); // The first value after .increment() is 2, hence by default the main thread ID should be 1.
	return id_counter;
}
thread_local Thread::ID Thread::caller_id = Thread::get_id_counter().increment();

```