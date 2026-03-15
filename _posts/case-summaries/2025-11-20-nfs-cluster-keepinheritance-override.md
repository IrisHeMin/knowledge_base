---
layout: post
title: "NFS 集群新文件丢失 NTFS 权限继承 — KeepInheritance 被集群注册表覆盖"
date: 2025-11-20
categories: [NFS, Failover-Cluster]
tags: [nfs, nfsv4, ntfs, permissions, inheritance, keepinheritance, failover-cluster, registry, windows-server-2019, smb]
---

# Case Summary: Windows Server 2019 NFS 集群新文件丢失 NTFS 权限继承 — KeepInheritance 被集群注册表覆盖

**Product/Service:** Windows Server 2019 / Server for NFS / Failover Clustering / SMB File Sharing

---

## 1. 症状 (Symptoms)

- 业务用户通过 SMB 共享访问文件时，所有**新生成的文件**提示没有权限，无法打开
- 新文件在 Windows 上查看 NTFS 安全属性时，**没有从父目录继承任何权限**，表现为"光杆司令"（只有创建者或无有效 ACE）
- 已有的旧文件不受影响，问题仅出现在 Linux 通过 NFS 写入的新文件上
- 问题在安装 2025 年 11 月安全补丁并重启服务器后出现

## 2. 背景 (Background / Environment)

- **文件处理流程**：
  1. 客户端通过 SFTP 将原始数据上传到 Linux 应用服务器
  2. Linux 处理完成后，通过 **NFSv4.1** 将结果文件写入 Windows Server 2019 NFS 集群
  3. 最终用户通过 **SMB 共享**访问这些结果文件
- **Windows 服务器配置**：
  - Windows Server 2019，部署了 **Server for NFS** 角色
  - NFS 服务运行在 **Windows Failover Cluster**（故障转移集群）上
  - NFS 共享目录配置了 NTFS 权限继承，父目录有正确的 ACL
- **近期变更**：安装了 2025 年 11 月的安全更新（如 KB5062557 等），安装后重启了服务器

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

1. **检查本机 NFS 注册表配置**
   - 检查了 NFS 服务的关键注册表项：
     ```
     HKEY_LOCAL_MACHINE\Software\Microsoft\Server for NFS\CurrentVersion\Mapping\KeepInheritance
     ```
   - 发现该值为 **0**（表示 NFS 创建文件时不保留 NTFS 权限继承）
   - 将其改为 **1** 后，测试 Linux 端创建新文件 → **权限恢复正常**，新文件成功从父目录继承 NTFS 权限
   - 初步结论：`KeepInheritance = 0` 是问题所在，修改后问题解决

2. **重启后问题复发**
   - 约一周后客户重启了服务器，问题再次出现
   - 再次检查注册表，发现 `KeepInheritance` 又**变回了 0**
   - 手动再改为 1，但这次**不再生效** —— 新文件依旧不继承 NTFS 权限
   - 关键疑问：是谁把值改回 0 的？是否与安全补丁有关？

3. **抓取 NFS Server ETL 日志**
   - 抓取了 NFS 服务端的 ETL trace
   - 日志确认 Linux 通过 NFSv4.1 协议创建文件（如 test8.txt）
   - 但日志中看不出是谁修改了注册表值
   - 线索卡住，需要从更底层排查

4. **检查集群注册表（关键发现）**
   - 由于 NFS 运行在 Failover Cluster 上，怀疑集群有独立的配置管理机制
   - 检查了**集群专用的注册表路径**：
     ```
     HKEY_LOCAL_MACHINE\Cluster\ServerForNFS\Parameters\KeepInheritance
     ```
   - 发现该值为 **0**
   - **关键发现**：在集群环境中，真正起决定作用的是 `HKLM\Cluster\...` 路径下的配置。每当 NFS 集群角色重新上线（重启、failover），集群服务会将这份配置**推送覆盖**到本机 `HKLM\Software\...` 路径
   - 这就解释了：
     - 第一次改本机注册表能生效 → 因为当时集群角色正在运行，还没被覆盖
     - 重启后值被改回 0 → 集群角色重新上线时把旧配置推回来了

5. **排查安全补丁影响**
   - 查阅了 2025 年 11 月安全补丁的公开文档（KB5062557 等）
   - 文档中没有任何关于 Server for NFS、KeepInheritance 或 NFS ACL 行为变更的说明
   - 确认：补丁本身未修改 NFS 配置，**重启才是触发器** —— 重启导致集群角色重新上线，旧的集群配置（`KeepInheritance = 0`）被重新下发
   - 结论：**Trigger ≠ Root Cause**（触发条件不等于根本原因）

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 第一次修改本机注册表后"成功"，导致误判根因已找到 | 浪费一周时间，直到重启后问题复发才重新排查 | 重启后复发暴露了"更上游"的配置源，转向排查集群注册表 |
| NFS Server ETL 日志看不出注册表被谁修改 | 无法从日志直接定位配置覆盖的来源 | 基于集群架构知识，推理出集群注册表是权威配置源，直接检查确认 |
| 客户怀疑是安全补丁导致问题，排查方向偏移 | 花时间排查补丁 changelog 是否修改了 NFS 相关配置 | 查阅补丁公开文档确认无 NFS 相关变更，明确重启是触发器而非根因 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

NFS 服务运行在 Windows Failover Cluster 上，集群注册表路径 `HKLM\Cluster\ServerForNFS\Parameters\KeepInheritance` 的值为 **0**（默认值，表示 NFS 创建文件时不保留 NTFS 权限继承）。

每当服务器重启或 NFS 集群角色发生 failover/重新上线时，集群服务会将其配置**推送覆盖**到本机注册表 `HKLM\Software\Microsoft\Server for NFS\CurrentVersion\Mapping\KeepInheritance`，导致之前手动修改的值被还原为 0。

安全补丁本身未修改任何 NFS 配置，但安装补丁后的重启触发了集群角色重新上线，使旧的集群配置重新生效，表现为"打完补丁就坏了"。

### Resolution

1. 在**集群注册表**中修改 `KeepInheritance` 为 1：
   ```
   HKEY_LOCAL_MACHINE\Cluster\ServerForNFS\Parameters\KeepInheritance = 1
   ```

2. 同时将**本机注册表**也设为 1（保持一致）：
   ```
   HKEY_LOCAL_MACHINE\Software\Microsoft\Server for NFS\CurrentVersion\Mapping\KeepInheritance = 1
   ```

3. 将 NFS 集群角色 **Offline → Online** 重新加载配置

4. 从 Linux 端创建新文件，在 Windows 上验证 NTFS 安全属性确认新文件正确继承了父目录权限

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - Windows NFS 集群环境中，`KeepInheritance` 注册表有**两个位置**：本机路径（`HKLM\Software\...`）和集群路径（`HKLM\Cluster\...`）。集群路径是权威配置源，会在角色上线时覆盖本机路径
  - `KeepInheritance = 1` 控制 NFS 创建文件时是否保留 NTFS 权限继承。默认值 0 意味着 NFS 创建的文件不继承父目录 ACL
  - 集群环境的配置管理机制：集群服务维护自己的注册表，角色上线时将配置推送到本机注册表

- **排查方法**：
  - **"修好了"不等于"找到根因了"**：第一次修改生效后不要急于结案，要考虑配置是否会被其他机制覆盖（集群、GPO、scheduled task 等）
  - **区分 Trigger 和 Root Cause**：补丁重启只是触发条件，真正的根因是集群配置基线不正确。排查时要分清"是什么暴露了问题"和"问题本身是什么"
  - **集群环境排查要检查集群注册表**：当本机注册表修改不持久时，立即检查 `HKLM\Cluster\` 下对应的配置

- **预防措施**：
  - 在集群环境中修改 NFS 配置时，**必须同时修改集群注册表**，否则下次 failover 或重启时会被覆盖
  - 建立配置基线文档，记录集群注册表中的关键配置项及其期望值
  - 重大变更（如安装补丁）前后，对比关键注册表项是否发生变化

## 7. 参考文档 (References)

- [Deploy Network File System](https://learn.microsoft.com/en-us/windows-server/storage/nfs/deploy-nfs) — Windows Server NFS 部署指南，包括 NFS 认证和共享配置
- [Network File System overview](https://learn.microsoft.com/en-us/windows-server/storage/nfs/nfs-overview) — NFS 功能概述，支持的版本和跨平台文件共享场景
- [Failover Clustering overview](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-clustering-overview) — Windows Server 故障转移集群架构和工作原理

---
---

# Case Summary: Windows Server 2019 NFS Cluster — New Files Lose NTFS Permission Inheritance Due to Cluster Registry Overriding KeepInheritance

**Product/Service:** Windows Server 2019 / Server for NFS / Failover Clustering / SMB File Sharing

---

## 1. Symptoms

- Business users accessing files through SMB share are **denied access** to all newly generated files
- New files show **no inherited NTFS permissions** from the parent directory when viewed in Windows Security properties — they appear as "bare" with no effective ACEs
- Existing older files are unaffected; the issue only affects new files written by Linux via NFS
- The problem appeared after installing November 2025 security patches and rebooting the server

## 2. Background / Environment

- **File processing workflow**:
  1. Clients upload raw data via SFTP to a Linux application server
  2. Linux processes the data and writes result files to the Windows Server 2019 NFS Cluster via **NFSv4.1**
  3. End users access these result files through **SMB shares**
- **Windows server configuration**:
  - Windows Server 2019 with **Server for NFS** role installed
  - NFS service runs on a **Windows Failover Cluster**
  - NFS share directories have NTFS permission inheritance configured with proper parent directory ACLs
- **Recent changes**: November 2025 security updates (e.g., KB5062557) were installed, requiring a server reboot

## 3. Investigation & Troubleshooting

1. **Checked the local NFS registry configuration**
   - Examined the key NFS service registry entry:
     ```
     HKEY_LOCAL_MACHINE\Software\Microsoft\Server for NFS\CurrentVersion\Mapping\KeepInheritance
     ```
   - Found the value was **0** (meaning NFS-created files do not preserve NTFS permission inheritance)
   - Changed it to **1**, then tested file creation from Linux → **Permissions restored**, new files correctly inherited parent directory NTFS permissions
   - Initial conclusion: `KeepInheritance = 0` was the issue, fixed after modification

2. **Problem recurred after reboot**
   - Approximately one week later, the customer rebooted the server and the problem returned
   - Re-checked the registry — `KeepInheritance` had **reverted to 0**
   - Manually changed it back to 1, but this time **it did not take effect** — new files still lacked NTFS permission inheritance
   - Key question: Who changed the value back to 0? Was it related to the security patches?

3. **Captured NFS Server ETL logs**
   - Captured NFS server-side ETL traces
   - Logs confirmed Linux was creating files via NFSv4.1 protocol (e.g., test8.txt)
   - However, the logs did not reveal who was modifying the registry value
   - Investigation stalled — needed to dig deeper

4. **Checked the cluster registry (critical discovery)**
   - Since NFS runs on a Failover Cluster, suspected the cluster had its own configuration management
   - Examined the **cluster-specific registry path**:
     ```
     HKEY_LOCAL_MACHINE\Cluster\ServerForNFS\Parameters\KeepInheritance
     ```
   - Found the value was **0**
   - **Critical finding**: In a cluster environment, the authoritative configuration resides under `HKLM\Cluster\...`. Whenever the NFS cluster role comes online (reboot, failover), the Cluster Service **pushes and overwrites** this configuration to the local `HKLM\Software\...` path
   - This explained:
     - First fix worked → because the cluster role was running and hadn't been overwritten yet
     - Value reverted to 0 after reboot → cluster role came back online and pushed the old configuration

5. **Investigated security patch impact**
   - Reviewed public documentation for November 2025 security patches (KB5062557, etc.)
   - Found no mentions of Server for NFS, KeepInheritance, or NFS ACL behavior changes
   - Confirmed: Patches did not modify NFS configuration. **The reboot was the trigger** — it caused the cluster role to come back online, re-pushing the old cluster configuration (`KeepInheritance = 0`)
   - Conclusion: **Trigger ≠ Root Cause**

## 4. Blockers & How They Were Resolved

| Blocker | Impact | How Resolved |
|---------|--------|-------------|
| First local registry fix "worked," creating a false sense of resolution | Wasted a week until the reboot caused the problem to resurface | Recurrence after reboot exposed an "upstream" configuration source; pivoted to investigating cluster registry |
| NFS Server ETL logs didn't show who modified the registry | Could not directly pinpoint the source of configuration override from logs | Used cluster architecture knowledge to reason that the cluster registry is the authoritative source; verified directly |
| Customer suspected security patches caused the issue, diverting investigation | Time spent analyzing patch changelogs for NFS-related changes | Reviewed public patch documentation confirming no NFS changes; clarified that the reboot was the trigger, not the root cause |

## 5. Root Cause & Resolution

### Root Cause

The NFS service was running on a Windows Failover Cluster, and the cluster registry path `HKLM\Cluster\ServerForNFS\Parameters\KeepInheritance` was set to **0** (default value, meaning NFS-created files do not preserve NTFS permission inheritance).

Whenever the server rebooted or the NFS cluster role experienced a failover/restart, the Cluster Service **pushed and overwrote** this configuration to the local registry at `HKLM\Software\Microsoft\Server for NFS\CurrentVersion\Mapping\KeepInheritance`, reverting any manual modifications back to 0.

The security patches themselves did not modify any NFS configuration. However, the post-patch reboot triggered the cluster role to come back online, causing the old cluster configuration to take effect again — creating the appearance that "patches broke it."

### Resolution

1. Set `KeepInheritance` to 1 in the **cluster registry**:
   ```
   HKEY_LOCAL_MACHINE\Cluster\ServerForNFS\Parameters\KeepInheritance = 1
   ```

2. Also set the **local registry** to 1 for consistency:
   ```
   HKEY_LOCAL_MACHINE\Software\Microsoft\Server for NFS\CurrentVersion\Mapping\KeepInheritance = 1
   ```

3. Cycle the NFS cluster role **Offline → Online** to reload configuration

4. Create new files from the Linux side and verify on Windows that NTFS Security properties show correct inherited permissions from the parent directory

## 6. Lessons Learned

- **Technical Knowledge**:
  - In a Windows NFS cluster environment, `KeepInheritance` exists in **two registry locations**: the local path (`HKLM\Software\...`) and the cluster path (`HKLM\Cluster\...`). The cluster path is the authoritative source and overwrites the local path when the role comes online
  - `KeepInheritance = 1` controls whether NFS-created files preserve NTFS permission inheritance. The default value of 0 means NFS-created files do not inherit parent directory ACLs
  - Cluster configuration management mechanism: the Cluster Service maintains its own registry and pushes configurations to local registry when roles come online

- **Troubleshooting Methodology**:
  - **"Fixed" doesn't mean "root cause found"**: After a first fix works, don't rush to close the case — consider whether the configuration might be overwritten by other mechanisms (cluster, GPO, scheduled tasks, etc.)
  - **Distinguish Trigger from Root Cause**: The patch reboot was merely the trigger; the real root cause was an incorrect cluster configuration baseline. Always differentiate "what exposed the problem" from "what the problem actually is"
  - **In cluster environments, always check cluster registry**: When local registry modifications don't persist, immediately check corresponding entries under `HKLM\Cluster\`

- **Prevention**:
  - When modifying NFS configuration in a cluster environment, **always update the cluster registry** as well; otherwise the next failover or reboot will overwrite the changes
  - Establish configuration baseline documentation recording key cluster registry entries and their expected values
  - Before and after major changes (e.g., patch installation), compare critical registry entries for unexpected modifications

## 7. References

- [Deploy Network File System](https://learn.microsoft.com/en-us/windows-server/storage/nfs/deploy-nfs) — Windows Server NFS deployment guide, including NFS authentication and share configuration
- [Network File System overview](https://learn.microsoft.com/en-us/windows-server/storage/nfs/nfs-overview) — NFS feature overview, supported versions, and cross-platform file sharing scenarios
- [Failover Clustering overview](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-clustering-overview) — Windows Server Failover Cluster architecture and operational principles
