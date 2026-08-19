---
date: '2026-08-19T14:00:00+08:00'
draft: false
title: 'Godot engine crashes while generating MonoGlue'
tags: ["Godot", "C++"]
---

## Problem Description

I recently updated to `Godot 4.7.3rc dotnet` on a Windows 11 machine, and found that after manually compiling the engine, `generate-mono-glue` crashes at the very end:

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

## In-Depth Investigation

Looking at the source code, it seems the assert is triggered in `Thread::release_main_thread()` — the `is_main_thread_assigned` flag was not set when releasing the main thread. I then went through the git commits and found a related one: `Make it impossible to have more than one main thread, and don't release unnecessarily - #121161#121161`. After analyzing it, the most likely cause is that `make_main_thread()` did not correctly set `is_main_thread_assigned`.

I added logging to the `make_main_thread` and `release_main_thread` functions:

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

The verification results are as follows:

```
[PROBE] make_main_thread: caller_id=1 flag=0 id_counter=42
[PROBE] release_main_thread: caller_id=1 flag=0 id_counter=42
ERROR: FATAL: Condition "!is_main_thread_assigned.clear_if_set()" is true.
   at: Thread::release_main_thread (core\os\thread.cpp:107)
```

Combining this with the code in `Thread.cpp`:

```cpp
SafeNumeric<uint64_t> Thread::id_counter(1); // The first value after .increment() is 2, hence by default the main thread ID should be 1.
thread_local Thread::ID Thread::caller_id = Thread::id_counter.increment();
```

Why is `caller_id=1`? It should be 2, and then the `is_main_thread_assigned.set_if_clear()` call in `make_main_thread` would work exactly as expected.

With the help of DeepSeek, I speculated that the problem might be `caller_id` being initialized before `id_counter`. First I modified the code:

```cpp
uint64_t _get_id_counter_inc(SafeNumeric<uint64_t>& idc) {
	return idc.increment();
}
SafeNumeric<uint64_t> Thread::id_counter(1);
thread_local Thread::ID Thread::caller_id = _get_id_counter_inc(id_counter);
```

Then I set a breakpoint at the first line of `main`. The results are as follows:

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

There is already a problem at the first breakpoint: both `caller_id` and `id_counter` are 1, when both should be 2. This further confirms my hypothesis: before `id_counter` is initialized it is 0, at which point `caller_id` is initialized to 1 first, and then `id_counter` is initialized and assigned 1.

To verify further, I modified the code as follows:

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

After running, the breakpoint at the first line of `main` shows the variable values below, which perfectly verifies the hypothesis:

```
Thread::caller_id	5	unsigned __int64
Thread::id_counter	{value=99 }	SafeNumeric<unsigned __int64>
```

The remaining question is why `caller_id` is initialized before `id_counter`. I tried initializing `id_counter` with a function return value and setting a breakpoint inside that function. Unfortunately, because compilation optimizations were enabled, the breakpoint had no effect. I then tried disabling compilation optimizations and breaking again, only to find that the bug no longer reproduced. I had to set this question aside for now.

## Fix

To fix the initialization order, my approach was to declare the variable as `static` inside a function, letting the function call order control the initialization order.

```cpp
SafeNumeric<uint64_t>& Thread::get_id_counter() {
	static SafeNumeric<uint64_t> id_counter(1); // The first value after .increment() is 2, hence by default the main thread ID should be 1.
	return id_counter;
}

Thread::ID& Thread::_caller_id_slot() {
	static thread_local ID caller_id = get_id_counter().increment();
	return caller_id;
}

static void Thread::set_caller_id(Thread::ID id) {
	_caller_id_slot() = id;
}

static Thread::ID Thread::get_caller_id() {
	return _caller_id_slot();
}
```