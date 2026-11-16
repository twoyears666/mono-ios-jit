# mono-ios-jit

给 iOS（TrollStore + JIT）编一个**开着 JIT** 的 Mono 运行时，用来在 iOS 上跑 .NET 托管模组。

## 为什么需要它

`microsoft.netcore.app.runtime.mono.ios-arm64` 这份官方运行时是带 `-DDISABLE_JIT` 编出来的（App Store 不允许 JIT）。于是 `src/mono/mono/mini/tramp-arm64.c` 里那些 JIT trampoline 全落在 `#else /* DISABLE_JIT */` 分支，只剩 `g_assert_not_reached()`：

```
mono_jit_init_version → mini_init → mono_arch_init
  → mini_get_breakpoint_trampoline → mono_arch_create_sdb_trampoline
  → g_assert_not_reached  (tramp-arm64.c:864)
```

所以嵌进去一调 `mono_jit_init` 就 SIGABRT。这不是 iOS 权限问题——PojavLauncher 的 JRE 能在 iOS 上 JIT，Mono 同样可以。

## 做法

1. 在 macOS arm64 runner 上原生编 `dotnet/runtime` 的 mono 运行库（macOS 版默认带 JIT）。
2. 用 `vtool` 把 `LC_BUILD_VERSION` 从 macOS 改成 iOS 15（CPU/ABI 相同，只登记平台号不同），再 adhoc 签名。
3. 托管 BCL 直接沿用官方 ios-arm64 运行时包（dll 与平台无关），保证 CoreLib 版本一致。

产物：`Mono-IOS-JIT.zip`（`libmonosgen-ios-arm64-jit.dylib` + `bcl/`）。

## 产物验收

- `otool -l` 里 `LC_BUILD_VERSION platform IOS`
- `nm` 能查到 `mono_arch_create_sdb_trampoline` 等 trampoline 符号
- 真机上 `dlopen` + `mono_jit_init_version` 不再 abort
