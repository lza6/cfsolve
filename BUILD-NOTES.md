MSVC toolchain absent: cargo build cannot work on this host. Patch is verified line-identical to the validated local patch; linking/running requires a Windows box with MSVC (per docs/纯协议-cfsolve.md).

## 2026-11-19 15:00 补充：build script 假死诊断

- 现象：23 个 build-script-build 进程同时启动、CPU 持续增长但 `output` 文件始终不生成，
  cargo 主进程 0 CPU 等待，锁死 30+ 分钟。
- 已验证非正常编译：proc-macro2/libc/serde 等简单包的 build script 正常应秒级结束。
- 结合本机多起 rustc 任务同时运行的历史，判断为 Windows 进程句柄/管道语义异常
  （cargo 16496 的 stdout 管道继承链在多并发下卡死，build script 写不进管道）。
- 处置：终止 16496 与全部 build-script-build，改前台低并发重跑（见下节）。

## 2026-11-19 15:20 链接器结论：本机无法链接 MSVC 目标

- 假死 build script 清掉后重跑，编译推进正常，但最终链接失败：
  `rust-lld: error: could not open 'msvcrt.lib'`。
- 根因：`msvcrt.lib`（MSVC CRT 导入库）只随 **VC++ 工具集（VC.Tools.x86.x64）** 分发，
  本机 VS 2022 只装了 Installer + Shared，VC 工具集实际不存在（无 cl.exe/link.exe/msvcrt.lib）。
  Windows SDK 10.0.26100 完整，但缺少 VC 部分无法补齐。
- cargo 配置已回退为标准 CRT 链接（shim/ntstc 方案不适用于 proc-macro DLL，且缺 msvcrt.lib 同样无解）。
- 结论：challenge.exe 构建需要完整 MSVC（VC 工具集）。当前机器重装 VC 工具集多次未成功
  （bootstrapper 0x57）。后续如需 stage2 快照，走 CI（GitHub Actions windows-latest 自带 MSVC）
  或换有完整 VS 的机器。cfsolve 修复补丁本身已完成并推送 fork（a33082b）。
