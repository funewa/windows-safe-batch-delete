---
name: windows-safe-batch-delete
description: 在 Windows 上安全批量删除大目录（优先移入回收站、可验证、可回滚）。当用户要求"删掉某个临时/构建目录""清理 C 盘某个文件夹""这些文件可以删除吗"且涉及 GB 级或多达上百个条目时使用。内置：只读盘点 → 危险内容识别（WSL vhdx / 虚拟磁盘）→ 逐项回收并校验 → 部分删除后的还原。
agent_created: true
---

# Windows 安全批量删除

## 0. 铁律

1. **先只读盘点，绝不直接删**。产出：条目数、总体积、按体积排序的清单、最近 N 天有改动的条目。
2. **先扫危险物**：`ext4.vhdx` / `.vhdx` / `.vhd` / `.iso`（**WSL 发行版虚拟磁盘，删了等于销毁一个 Linux 环境**）、`.git`（未推送的本地改动）、近 14 天改动的条目。命中即**停下来问用户**，不要自作主张。
3. 删除一律走**回收站**，且**逐项校验**是否真的进了回收站。
4. 每批 ≤ 10 项；出现"消失了但回收站里没有"立即停止并报警。

## 1. 只读盘点

```python
import os, time
for e in sorted(os.scandir(ROOT), key=lambda x: x.name.lower()):
    st = e.stat()
    size = dirsize(e.path) if e.is_dir() else st.st_size   # 目录递归求和
```
顺带确认 `TMP/TEMP` 环境变量是否指向该目录（指向则不要删）。

## 2. 回收站 API（ctypes，无需第三方库）

```python
import ctypes
from ctypes import wintypes

class SHFILEOPSTRUCTW(ctypes.Structure):
    _fields_ = [("hwnd", wintypes.HWND), ("wFunc", wintypes.UINT),
                ("pFrom", wintypes.LPCWSTR), ("pTo", wintypes.LPCWSTR),
                ("fFlags", ctypes.c_uint), ("fAnyOperationsAborted", wintypes.BOOL),
                ("hNameMappings", ctypes.c_void_p), ("lpszProgressTitle", wintypes.LPCWSTR)]

def recycle(path):
    op = SHFILEOPSTRUCTW(None, 3, path + "\0\0", None,
                         0x40 | 0x10 | 0x400 | 0x4, False, None, None)  # ALLOWUNDO|NOCONFIRM|NOERRORUI|SILENT
    return ctypes.windll.shell32.SHFileOperationW(ctypes.byref(op))
```

### 三个必知的坑

- **返回值不可信**：成功也常返回 `rc=2`。判成败只看：源文件是否消失 **且** 回收站新增对应 `$I` 记录。
- **同进程连续调用会挂死** → 每项派一个独立子进程（`subprocess.run([PY, worker, path], timeout=25)`）。
- **大目录会挂起 → 被中断后"删了一半"**（实测 671 MB 目录掉到 338 MB，数千文件散落进回收站）。
  **正确做法**：先 `os.rename` 移动到暂存区（同盘瞬间完成），再对暂存区副本回收（1 秒级）；超时 ≤ 25 s。

## 3. 校验是否真进回收站

```python
BIN = r"C:\$Recycle.Bin\<SID>"        # 逐个 SID 试 os.listdir，取能读的那个
for fn in os.listdir(BIN):
    if not fn.startswith("$I"): continue
    raw = open(os.path.join(BIN, fn), "rb").read()
    size = struct.unpack("<Q", raw[8:16])[0]          # 0x08: int64 文件大小
    idx  = raw.find("C:".encode("utf-16-le"))          # 原始路径从 0x1C(28) 开始，用 find 定位最稳
    src  = raw[idx:].decode("utf-16-le","ignore").split("\x00")[0] if idx>=0 else ""
```
校验逻辑：删除前快照 `$I` 文件名集合，删除后找**新增**项，且其 `src` 以被删名字结尾 → 确认入站。

## 4. 回收站配额（避免"过大而被永久删除"）

- 注册表 `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\BitBucket\Volume\{卷GUID}\MaxCapacity`（单位 MB）。
- 卷 GUID → 盘符：`GetVolumePathNamesForVolumeNameW(r"\\?\Volume{GUID}\")`。
- 若单项目体积 > 配额，不要走回收站，改为暂存或征求用户意见。

## 5. 部分删除后的还原（救命用）

每个 `$I` 都有原始路径，配套 `$R`（`$I` 去掉前缀换 `$R`）就是内容，按原路径写回即可完整还原目录树。
所以"删了一半"并不等于丢失 —— 先统计 `$I` 按顶层目录分组，报给用户，再决定是否整体还原。

## 5b. 还原与合并的实现要点

- 还原 = 把 `$R<随机名>` rename 回 `$I` 里记录的原始路径；**不要删 `$I`**（沙箱 safe-delete 钩子：一轮内 ≥50 次文件删除会被拦截，报 `SAFE_DELETE_BULK_CONFIRM_REQUIRED`）。残留的孤儿 `$I` 让用户清空回收站即可。
- 目标路径已存在 → 跳过（视为重复）；`$R` 已不存在 → 记为失败。
- 把暂存区目录并回原处时，**按整目录 rename**（`os.scandir` 顶层逐个试，目标不存在就整个 rename；存在才递归），不要逐文件搬：本机逐文件 rename 只有 ~14 个/秒，万级文件会跑到小时级。
- 还原后必须做**体积对账**：拿清理前的目录体积表逐一比对，输出"清理前 / 现在 / 差"，差值应为 0 或极小。

## 6. 不要用的方案

- `send2trash`（pip）：会把路径转 8.3 短名，短名功能关闭的机器上一律 `FileNotFoundError`。
- `rm -rf` / `shutil.rmtree` 直接删：不可逆，只在用户明确接受永久删除时用。
- `powershell.exe`：本环境 Bash 里被拦截，不要依赖。

## 7. 报告模板

体积 Top 列表 + 近 14 天改动项 + 危险物（vhdx 等）+ 已回收/已跳过/卡住三栏 + 可用空间前后对比 + **诚实的损失说明**（任何被永久删除的内容必须点名）。
