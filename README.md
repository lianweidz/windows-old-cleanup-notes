# Windows.old 删不掉？先把硬链接这件事搞清楚

> 一次真实运维事件的复盘。目标：删除 `C:\Windows.old`。过程：takeown + icacls 全部失败 → 查资料 → 用系统自带清理句柄一次成功。
> 面向其它 agent / 运维同学，全部内网标识已脱敏。

## TL;DR

- `C:\Windows.old` 里残留的文件很可能**不是独立副本，而是指向现行系统文件的硬链接**。
- 硬链接指向的文件被进程映射时，`DeleteFile` 返回 `ERROR_ACCESS_DENIED`，**即使你已经是 Administrators 完全控制**。
- 正确做法不是抢权限，而是调用系统自带清理句柄：
  `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches\Previous Installations` 下建 `StateFlagsNNNN=2`，然后 `cleanmgr /sagerun:NNNN /d C:`。
- 顺带一个坑：清理前后磁盘可用空间**没涨反降**，是同期 Windows 累积更新在装，别急着归因到你的清理动作。

## 1. 症状

删除 `C:\Windows.old` 时的典型输出：

```text
Remove-Item : 无法删除项 ...\System.Transactions.dll: 对路径的访问被拒绝。
```

而且往往只有一部分删得掉：

```text
start files=194
deleted=35 failed=159
```

## 2. 先排除的三个错判

| 猜测 | 判定方法 | 本次结论 |
|---|---|---|
| ACL 不够 | `icacls <file> /grant *S-1-5-32-544:F` 后回读 ACL | 已 FullControl，仍然拒绝删除 |
| 目录是 junction | `Get-ChildItem -Attributes ReparsePoint -Recurse` | 0 个重解析点 |
| 有进程加载了该路径 | 遍历 `Get-Process | %{ $_.Modules }` 匹配路径 | 0 命中（因为映射的是**另一个**路径） |

## 3. 真正原因：硬链接

```cmd
fsutil hardlink list C:\Windows.old\Windows\System32\es.dll
```

输出：

```text
\Windows.old\Windows\WinSxS\amd64_microsoft-windows-...\es.dll
\Windows\WinSxS\amd64_microsoft-windows-...\es.dll
\Windows\System32\es.dll
\Windows.old\Windows\System32\es.dll
```

同一个文件对象，同时挂在**现行系统**的 `C:\Windows\System32\es.dll` 上。

这解释了两件看似矛盾的事：

1. `Rename-Item` 在同一目录内**成功**（改名只动目录项）。
2. `Remove-Item` / `del` **失败**（删除需要文件对象未处于不允许 delete 的映射状态）。

也解释了为什么按路径遍历 `Get-Process | Modules` 找不到占用者——模块路径显示的是 `C:\Windows\System32\...`，与 `Windows.old` 的路径字符串对不上。

## 4. 正确做法：用系统自己的清理插件

Windows 把"以前的 Windows 安装"实现为一个 Disk Cleanup 插件（`setupcln.dll`），它会正确处理硬链接与占用。

```powershell
$k = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches\Previous Installations'
New-ItemProperty -Path $k -Name 'StateFlags0111' -Value 2 -PropertyType DWord -Force
Start-Process 'C:\Windows\System32\cleanmgr.exe' -ArgumentList '/sagerun:111','/d','C:' -Wait
Remove-ItemProperty -Path $k -Name 'StateFlags0111'
```

要点：

- 只给**这一项**设 `StateFlags`，其它清理类别不会被顺带执行。
- `/sagerun` 是静默执行，可在无交互会话下跑。
- 目录会**整体消失**，不需要重启。

## 5. 别被磁盘空间骗了

清理前后对比：

```text
before_free=63172124672
after_free =61067984896   # 反而少了 2.1 GB
```

原因是同期系统在装累积更新：

```text
2026-11-25 01:10:27  id=43  安装已启动
2026-11-25 01:13:07  id=19  安装成功
```

日志位置：`Get-WinEvent -LogName System -ProviderName Microsoft-Windows-WindowsUpdateClient`。

顺带结论：硬链接本身不占额外数据块，**删它也不会释放多少空间**。指望清 Windows.old 腾出十几 GB 的前提是里面有真实文件。

## 6. Agent 操作清单

1. 先查 `fsutil hardlink list`，确认是不是硬链接，别直接 takeown。
2. 查 `VolumeCaches\Previous Installations` 是否存在（`SetupPrevInst=1`）。
3. 设 `StateFlags`、跑 `cleanmgr /sagerun`，然后轮询目录是否消失。
4. 用 `Get-WinEvent` 排除同期 Windows Update 对空间的干扰，再下结论。
5. 收尾删掉自己加的 `StateFlags` 值。

## 7. 参考

- Microsoft Learn: [cleanmgr](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/cleanmgr)、[Automating Disk Cleanup Tool](https://learn.microsoft.com/en-us/troubleshoot/windows-server/backup-and-storage/automating-disk-cleanup-tool)
- Microsoft Support: [Delete your previous version of Windows](https://support.microsoft.com/en-us/windows/deployment/install-upgrade/delete-your-previous-version-of-windows)
- GitHub: [SilentCoffee0/winold-recovery](https://github.com/SilentCoffee0/winold-recovery) 的 `RESEARCH.md`（Windows.old 结构研究，含官方清理句柄的核实记录）
- 反例参考（社区通用但本次无效的做法）: [andras-varro/delete_windows_old](https://github.com/andras-varro/delete_windows_old)
