---
date: '2026-06-15T14:00:40+08:00'
draft: false
title: 'Godot C# 在 Shutdown 时的 GC 问题'
tags: ["Godot", "C#"]
---

## 问题描述

关闭游戏窗口时，有一定概率在此处触发内存泄漏问题：

```cpp
_FORCE_INLINE_ ~List() {
	// A self list must be empty on destruction.
	DEV_ASSERT(_first == nullptr);
}
```

## 深入调查

该问题出现在 `Windows11` 上的 `Godot 4.6.3 dotnet`。就我的情况而言，我发现这个 `List` 实际上是 `CSharpScript` 的容器，而 `CSharpScript` 在该时点尚未被释放。于是我在 `RefCounted` 中记录了 ref / unref 的调用栈，得到如下结果。

在 `DEV_ASSERT(_first == nullptr);` 处，检查该 `List` 第一个元素的 ref & unref 记录：

```
-		_self	0x00000220db3a54b0 {type_info={class_name=U"AudioRay" native_base_name={_data=0x000002208c658f08 {...} } ...} ...}	CSharpScript *
-		Script	{...}	Script
-		Resource	{name=[empty] path_cache=U"res://addons/raytraced_audio_csharp/AudioRay.cs" scene_unique_id=[empty] ...}	Resource
-		RefCounted	{refcount={count={value={...} } } refcount_init={count={value={...} } } dereference_count={value=0 } ...}	RefCounted
+		Object	{_extension=0x0000000000000000 <NULL> _extension_instance=0x0000000000000000 signal_mutex=0x00000220db6b9140 {...} ...}	Object
+		refcount	{count={value=1 } }	SafeRefCount
+		refcount_init	{count={value=0 } }	SafeRefCount
+		dereference_count	{value=0 }	SafeNumeric<unsigned int>
-		debug_ref_frames	{[size]=44 [0]={frames=0x00000220db16ae18 {0x00007ff6d23d399e {...}, 0x00007ff6d23d3c28 {...}, 0x00007ff6d23d3aa3 {...}, ...} ...} ...}	Vector<RefCounted::DebugFrame>
		[size]	44	unsigned __int64
-		[0]	{frames=0x00000220db16ae18 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
-		frames	0x00000220db16ae18 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...}	const void *[8]
		[0]	0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}	const void *
		[1]	0x00007ff6d23d3c28 {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::reference(void), Line 91}	const void *
		[2]	0x00007ff6d23d3aa3 {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::init_ref(void), Line 42}	const void *
		[3]	0x00007ff6cd96bfcb {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::ref_pointer<1>(CSharpScript * p_refcounted), Line 92}	const void *
		[4]	0x00007ff6cd96e0fd {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::operator=(CSharpScript * p_from), Line 166}	const void *
		[5]	0x00007ff6cd96cfd9 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::Ref<CSharpScript>(CSharpScript * p_from), Line 203}	const void *
		[6]	0x00007ff6cd978be3 {godot.windows.editor.dev.x86_64.mono.exe!godotsharp_internal_new_csharp_script(Ref<CSharpScript> * r_dest), Line 320}	const void *
		[7]	0x00007ffa1389a09d	const void *
		count	8	unsigned int
		......
		......
-		[42]	{frames=0x00000220db16b9e8 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
-		frames	0x00000220db16b9e8 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...}	const void *[8]
		[0]	0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}	const void *
		[1]	0x00007ff6d23d3c28 {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::reference(void), Line 91}	const void *
		[2]	0x00007ff6cc3c57bb {godot.windows.editor.dev.x86_64.mono.exe!Ref<Script>::ref_pointer<0>(Script * p_refcounted), Line 96}	const void *
		[3]	0x00007ff6ccc1fa85 {godot.windows.editor.dev.x86_64.mono.exe!Ref<Script>::operator=(const Variant & p_variant), Line 176}	const void *
		[4]	0x00007ff6ccc1e459 {godot.windows.editor.dev.x86_64.mono.exe!Ref<Script>::Ref<Script>(const Variant & p_from), Line 207}	const void *
		[5]	0x00007ff6ccd93720 {godot.windows.editor.dev.x86_64.mono.exe!ContainerTypeValidate::_internal_validate_object(const Variant & p_variant, const char * p_operation, bool p_output_errors), Line 124}	const void *
		[6]	0x00007ff6ccd932ba {godot.windows.editor.dev.x86_64.mono.exe!ContainerTypeValidate::_internal_validate(Variant & inout_variant, const char * p_operation, bool p_output_errors), Line 83}	const void *
		[7]	0x00007ff6d23655ca {godot.windows.editor.dev.x86_64.mono.exe!ContainerTypeValidate::validate(Variant & inout_variant, const char * p_operation), Line 148}	const void *
		count	8	unsigned int
-		[43]	{frames=0x00000220db16ba30 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
-		frames	0x00000220db16ba30 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...}	const void *[8]
		[0]	0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}	const void *
		[1]	0x00007ff6d23d3c28 {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::reference(void), Line 91}	const void *
		[2]	0x00007ff6cd96c04b {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::ref_pointer<0>(CSharpScript * p_refcounted), Line 96}	const void *
		[3]	0x00007ff6cd96b00d {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::operator=<Script>(const Ref<Script> & p_from), Line 162}	const void *
		[4]	0x00007ff6cd96af09 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::Ref<CSharpScript><Script>(const Ref<Script> & p_from), Line 199}	const void *
		[5]	0x00007ff6cd96007d {godot.windows.editor.dev.x86_64.mono.exe!CSharpScript::inherits_script(const Ref<Script> & p_script), Line 2708}	const void *
		[6]	0x00007ff6ccd93915 {godot.windows.editor.dev.x86_64.mono.exe!ContainerTypeValidate::_internal_validate_object(const Variant & p_variant, const char * p_operation, bool p_output_errors), Line 134}	const void *
		[7]	0x00007ff6ccd932ba {godot.windows.editor.dev.x86_64.mono.exe!ContainerTypeValidate::_internal_validate(Variant & inout_variant, const char * p_operation, bool p_output_errors), Line 83}	const void *
		count	8	unsigned int
+		[Raw View]	{write={...} _cowdata={_ptr=0x00000220db16ae18 {frames=... count=8 } } }	Vector<RefCounted::DebugFrame>
-		debug_unref_frames	{[size]=44 [0]={frames=0x00000240d7a66838 {0x00007ff6d23d399e {...}, 0x00007ff6d23d3e4d {...}, 0x00007ff6d23d3adf {...}, ...} ...} ...}	Vector<RefCounted::DebugFrame>
		[size]	44	unsigned __int64
-		[0]	{frames=0x00000240d7a66838 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
-		frames	0x00000240d7a66838 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...}	const void *[8]
		[0]	0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}	const void *
		[1]	0x00007ff6d23d3e4d {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::unreference(void), Line 134}	const void *
		[2]	0x00007ff6d23d3adf {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::init_ref(void), Line 44}	const void *
		[3]	0x00007ff6cd96bfcb {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::ref_pointer<1>(CSharpScript * p_refcounted), Line 92}	const void *
		[4]	0x00007ff6cd96e0fd {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::operator=(CSharpScript * p_from), Line 166}	const void *
		[5]	0x00007ff6cd96cfd9 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::Ref<CSharpScript>(CSharpScript * p_from), Line 203}	const void *
		[6]	0x00007ff6cd978be3 {godot.windows.editor.dev.x86_64.mono.exe!godotsharp_internal_new_csharp_script(Ref<CSharpScript> * r_dest), Line 320}	const void *
		[7]	0x00007ffa1389a09d	const void *
		count	8	unsigned int
		......
		......
		......
-		[42]	{frames=0x00000240d7a67408 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
-		frames	0x00000240d7a67408 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...}	const void *[8]
		[0]	0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}	const void *
		[1]	0x00007ff6d23d3e4d {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::unreference(void), Line 134}	const void *
		[2]	0x00007ff6cd977a71 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::unref(void), Line 222}	const void *
		[3]	0x00007ff6cd96dd83 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::~Ref<CSharpScript>(void), Line 239}	const void *
		[4]	0x00007ff6cd96359a {godot.windows.editor.dev.x86_64.mono.exe!CSharpInstance::~CSharpInstance(void), Line 2055}	const void *
		[5]	0x00007ff6cd96f027 {Inside godot.windows.editor.dev.x86_64.mono.exe!CSharpInstance::`scalar deleting destructor'(unsigned int)}	const void *
		[6]	0x00007ff6d2399afb {godot.windows.editor.dev.x86_64.mono.exe!memdelete<ScriptInstance>(ScriptInstance * p_class), Line 152}	const void *
		[7]	0x00007ff6d2389d93 {godot.windows.editor.dev.x86_64.mono.exe!Object::set_script_instance(ScriptInstance * p_instance), Line 1128}	const void *
		count	8	unsigned int
-		[43]	{frames=0x00000240d7a67450 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
-		frames	0x00000240d7a67450 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...}	const void *[8]
		[0]	0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}	const void *
		[1]	0x00007ff6d23d3e4d {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::unreference(void), Line 134}	const void *
		[2]	0x00007ff6cd977a71 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::unref(void), Line 222}	const void *
		[3]	0x00007ff6cd96dd83 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::~Ref<CSharpScript>(void), Line 239}	const void *
		[4]	0x00007ff6cd96359a {godot.windows.editor.dev.x86_64.mono.exe!CSharpInstance::~CSharpInstance(void), Line 2055}	const void *
		[5]	0x00007ff6cd96f027 {Inside godot.windows.editor.dev.x86_64.mono.exe!CSharpInstance::`scalar deleting destructor'(unsigned int)}	const void *
		[6]	0x00007ff6d2399afb {godot.windows.editor.dev.x86_64.mono.exe!memdelete<ScriptInstance>(ScriptInstance * p_class), Line 152}	const void *
		[7]	0x00007ff6d2389d93 {godot.windows.editor.dev.x86_64.mono.exe!Object::set_script_instance(ScriptInstance * p_instance), Line 1128}	const void *
		count	8	unsigned int
+		[Raw View]	{write={...} _cowdata={_ptr=0x00000240d7a66838 {frames=... count=8 } } }	Vector<RefCounted::DebugFrame>
```

对照 `debug_ref_frames` 和 `debug_unref_frames`，两者的长度都是 44。这提示我：这个 `CSharpScript` 还需要一次 unref 才能被释放（因为 `RefCounted` 的初始 refcount 是 1）。

那么到底是谁还持有这个引用？接下来检查问题未发生的情况：
我选择了这条代码路径来设置断点，条件是当这个 `CSharpScript` 的名字是 `AudioRay` 时（与上一个案例相同）：`CSharpScript::~CSharpScript() Line 2806`

在 `CSharpScript::~CSharpScript()` 处，检查该 `CSharpScript` 的 ref & unref 记录：

```
-		Script	{...}	Script
-		Resource	{name=[empty] path_cache=U"res://addons/raytraced_audio_csharp/AudioRay.cs" scene_unique_id=[empty] ...}	Resource
-		RefCounted	{refcount={count={value={...} } } refcount_init={count={value={...} } } dereference_count={value=0 } ...}	RefCounted
+		Object	{_extension=0x0000000000000000 <NULL> _extension_instance=0x0000000000000000 signal_mutex=0x000001d7572d4a30 {...} ...}	Object
+		refcount	{count={value=0 } }	SafeRefCount
+		refcount_init	{count={value=0 } }	SafeRefCount
+		dereference_count	{value=0 }	SafeNumeric<unsigned int>
+		debug_ref_frames	{[size]=44 [0]={frames=0x000001d757742298 {0x00007ff6d23d399e {...}, 0x00007ff6d23d3c28 {...}, 0x00007ff6d23d3aa3 {...}, ...} ...} ...}	Vector<RefCounted::DebugFrame>
-		debug_unref_frames	{[size]=45 [0]={frames=0x000001d75774d738 {0x00007ff6d23d399e {...}, 0x00007ff6d23d3e4d {...}, 0x00007ff6d23d3adf {...}, ...} ...} ...}	Vector<RefCounted::DebugFrame>
		[size]	45	unsigned __int64
+		[0]	{frames=0x000001d75774d738 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
+		[1]	{frames=0x000001d75774d780 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
		......
		......
		......
-		[42]	{frames=0x000001d75774e308 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
-		frames	0x000001d75774e308 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...}	const void *[8]
		[0]	0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}	const void *
		[1]	0x00007ff6d23d3e4d {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::unreference(void), Line 134}	const void *
		[2]	0x00007ff6cd977a71 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::unref(void), Line 222}	const void *
		[3]	0x00007ff6cd96dd83 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::~Ref<CSharpScript>(void), Line 239}	const void *
		[4]	0x00007ff6cd96359a {godot.windows.editor.dev.x86_64.mono.exe!CSharpInstance::~CSharpInstance(void), Line 2055}	const void *
		[5]	0x00007ff6cd96f027 {Inside godot.windows.editor.dev.x86_64.mono.exe!CSharpInstance::`scalar deleting destructor'(unsigned int)}	const void *
		[6]	0x00007ff6d2399afb {godot.windows.editor.dev.x86_64.mono.exe!memdelete<ScriptInstance>(ScriptInstance * p_class), Line 152}	const void *
		[7]	0x00007ff6d2389d93 {godot.windows.editor.dev.x86_64.mono.exe!Object::set_script_instance(ScriptInstance * p_instance), Line 1128}	const void *
		count	8	unsigned int
-		[43]	{frames=0x000001d75774e350 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
-		frames	0x000001d75774e350 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...}	const void *[8]
		[0]	0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}	const void *
		[1]	0x00007ff6d23d3e4d {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::unreference(void), Line 134}	const void *
		[2]	0x00007ff6cd977a71 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::unref(void), Line 222}	const void *
		[3]	0x00007ff6cd96dd83 {godot.windows.editor.dev.x86_64.mono.exe!Ref<CSharpScript>::~Ref<CSharpScript>(void), Line 239}	const void *
		[4]	0x00007ff6cd96359a {godot.windows.editor.dev.x86_64.mono.exe!CSharpInstance::~CSharpInstance(void), Line 2055}	const void *
		[5]	0x00007ff6cd96f027 {Inside godot.windows.editor.dev.x86_64.mono.exe!CSharpInstance::`scalar deleting destructor'(unsigned int)}	const void *
		[6]	0x00007ff6d2399afb {godot.windows.editor.dev.x86_64.mono.exe!memdelete<ScriptInstance>(ScriptInstance * p_class), Line 152}	const void *
		[7]	0x00007ff6d2389d93 {godot.windows.editor.dev.x86_64.mono.exe!Object::set_script_instance(ScriptInstance * p_instance), Line 1128}	const void *
		count	8	unsigned int
-		[44]	{frames=0x000001d75774e398 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...} ...}	RefCounted::DebugFrame
-		frames	0x000001d75774e398 {0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}, ...}	const void *[8]
		[0]	0x00007ff6d23d399e {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::_capture_debug_frame(void), Line 64}	const void *
		[1]	0x00007ff6d23d3e4d {godot.windows.editor.dev.x86_64.mono.exe!RefCounted::unreference(void), Line 134}	const void *
		[2]	0x00007ff6cc3dc021 {godot.windows.editor.dev.x86_64.mono.exe!Ref<Script>::unref(void), Line 222}	const void *
		[3]	0x00007ff6cc3c95b3 {godot.windows.editor.dev.x86_64.mono.exe!Ref<Script>::~Ref<Script>(void), Line 239}	const void *
		[4]	0x00007ff6d236390a {Inside godot.windows.editor.dev.x86_64.mono.exe!ContainerTypeValidate::~ContainerTypeValidate(void)}	const void *
		[5]	0x00007ff6d243489a {Inside godot.windows.editor.dev.x86_64.mono.exe!ArrayPrivate::~ArrayPrivate(void)}	const void *
		[6]	0x00007ff6d24349f7 {Inside godot.windows.editor.dev.x86_64.mono.exe!ArrayPrivate::`scalar deleting destructor'(unsigned int)}	const void *
		[7]	0x00007ff6d24345c8 {godot.windows.editor.dev.x86_64.mono.exe!memdelete<ArrayPrivate>(ArrayPrivate * p_class), Line 152}	const void *
		count	8	unsigned int
```

我发现了关键点 `debug_unref_frames[44]`，它就是该 `CSharpScript` 的最后一个引用持有者。

在 `CSharpScript::~CSharpScript()` 处，检查调用栈：

```
 	godot.windows.editor.dev.x86_64.mono.exe!CSharpScript::~CSharpScript() Line 2806	C++
 	[External Code]	
 	godot.windows.editor.dev.x86_64.mono.exe!memdelete<RefCounted>(RefCounted * p_class) Line 152	C++
 	godot.windows.editor.dev.x86_64.mono.exe!Ref<Script>::unref() Line 223	C++
 	godot.windows.editor.dev.x86_64.mono.exe!Ref<Script>::~Ref<Script>() Line 239	C++
 	[External Code]	
 	godot.windows.editor.dev.x86_64.mono.exe!memdelete<ArrayPrivate>(ArrayPrivate * p_class) Line 152	C++
 	godot.windows.editor.dev.x86_64.mono.exe!Array::_unref() Line 83	C++
 	godot.windows.editor.dev.x86_64.mono.exe!Array::~Array() Line 969	C++
 	[External Code]	
	godot.windows.editor.dev.x86_64.mono.exe!godotsharp_array_destroy(Array * p_self) Line 1048	C++
 	[External Code]	
 	godot.windows.editor.dev.x86_64.mono.exe!CSharpLanguage::finalize() Line 144	C++
 	godot.windows.editor.dev.x86_64.mono.exe!CSharpLanguage::finish() Line 135	C++
 	godot.windows.editor.dev.x86_64.mono.exe!ScriptServer::finish_languages() Line 345	C++
 	godot.windows.editor.dev.x86_64.mono.exe!Main::cleanup(bool p_force) Line 5172	C++
 	godot.windows.editor.dev.x86_64.mono.exe!widechar_main(int argc, wchar_t * * argv) Line 103	C++
 	godot.windows.editor.dev.x86_64.mono.exe!_main() Line 126	C++
 	godot.windows.editor.dev.x86_64.mono.exe!main(int argc, char * * argv) Line 140	C++
 	godot.windows.editor.dev.x86_64.mono.exe!WinMain(HINSTANCE__ * hInstance, HINSTANCE__ * hPrevInstance, char * lpCmdLine, int nCmdShow) Line 154	C++
 	[External Code]	
 	godot.windows.editor.dev.x86_64.mono.exe!ShimMainCRTStartup(...) Line 74	C
```

OK，`CSharpLanguage::finalize() Line 144` 表明它来自这里：

```cpp
void CSharpLanguage::finalize() {
	if (finalized) {
		return;
	}

	if (gdmono && gdmono->is_runtime_initialized() && GDMonoCache::godot_api_cache_updated) {
		GDMonoCache::managed_callbacks.DisposablesTracker_OnGodotShuttingDown();   // <- here!!!
	}
......
}
```

在 `godot\modules\mono\glue\GodotSharp\GodotSharp\Core\DisposablesTracker.cs` 中找到了对应的 C# 代码，其中的 `OnGodotShuttingDownImpl` 函数似乎是手动调用 `Dispose()` 来最终 unref 这些 `CSharpScript` 的。

## 分析与修复

我查看了 `OnGodotShuttingDownImpl`，经过一番 API 检索后，我认为问题可能就在这里。以下是我**猜测**的出错时间线（buggy timeline）：
- Step1：dotnet runtime 的 GC 回收（collected）了某些 `RefCounted` 对象，但它们的 finalizer 并不会立即被调用。
- Step2：`OnGodotShuttingDownImpl` 手动 `Dispose()` 了所有 "disposable" 对象，但 Step1 中的对象不在此列——因为它们已被 GC 回收，其 `WeakReference` 应当已失效（invalid）。
- Step3：`DEV_ASSERT(_first == nullptr);` 触发。
- Step4：dotnet 的 Finalizer 线程最终才调用 Step1 中那些对象的 finalizer。

因此我修改了 `OnGodotShuttingDownImpl`，在该函数末尾手动触发 GC 并等待所有 finalizer 执行完毕；在此之后，该问题不再出现（我已测试了大约 10 次）。

```csharp
private static void OnGodotShuttingDownImpl()
{
    bool isStdoutVerbose;

    try
    {
        isStdoutVerbose = OS.IsStdOutVerbose();
    }
    catch (ObjectDisposedException)
    {
        // OS singleton already disposed. Maybe OnUnloading was called twice.
        isStdoutVerbose = false;
    }

    if (isStdoutVerbose)
        GD.Print("Unloading: Disposing tracked instances...");

    // Dispose Godot Objects first, and only then dispose other disposables
    // like StringName, NodePath, Godot.Collections.Array/Dictionary, etc.
    // The Godot Object Dispose() method may need any of the later instances.

    foreach (WeakReference<GodotObject> item in GodotObjectInstances.Keys)
    {
        if (item.TryGetTarget(out GodotObject? self))
            self.Dispose();
    }

    foreach (WeakReference<IDisposable> item in OtherInstances.Keys)
    {
        if (item.TryGetTarget(out IDisposable? self))
            self.Dispose();
    }

    if (isStdoutVerbose)
         GD.Print("Unloading: Finished disposing tracked instances.");
   
    // MY FIX: waiting for all finalizers finished
    GC.Collect(GC.MaxGeneration, GCCollectionMode.Forced);
    GC.WaitForPendingFinalizers(); 
    // MY FIX END
}
```

另外，通过mixed debugging和多次测试，我还发现最后一个引用的持有者是一个 `TypedArray`：它的 `_p->typed.script` 与上述地址完全相同。

```
&_p->typed.script	0x0000023061165bc0 {type_info={class_name=U"AudioRay" native_base_name={_data=0x0000021041172ea8 {refcount={count={value=...} } ...} } ...} ...}	Ref<Script> *
this	0x0000023061165bc0 {type_info={class_name=U"AudioRay" native_base_name={_data=0x0000021041172ea8 {refcount={count={value=...} } ...} } ...} ...}	Ref<Script> *
```