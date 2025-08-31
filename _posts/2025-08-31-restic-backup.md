---
title: 差点和毕业论文说再见？构建WSL下的备份方案
date: 2025-08-31 10:51:30 +0800
categories: [编程, 备份]
tags: [restic]
description: 一块服役半年的SSD突然闹罢工，差点让我明年毕不了业。痛定思痛，决定用开源备份工具restic配合Windows任务计划程序，组合出了一套“狡兔三窟”的WSL全自动备份方案。
---

Windows最近更新[KB5063878](https://learn.microsoft.com/en-us/answers/questions/5534483/update-issue-kb5063878-(os-build-26100-4946))似乎造成了大面积的固态硬盘故障，前阵子在做实验出现大量写入的时候也遇到了假死的问题（硬盘占用100%，响应时间上万毫秒，写入速度几百KB每秒），一度以为这块服役不到半年的固态硬盘要归西了。这让我打了个激灵，想起来有许多重要数据没有备份，万一这块固态硬盘挂掉了，大概率明年是很难毕业了……

## 对备份方案的需求

最近这半年多以来，主要在WSL2环境中开发，所有的实验代码、实验结果、论文手稿都在WSL2子系统中。我需要一个稳定易用的备份系统，周期性的把重要数据进行备份。基本的要求包括：

- 适用于WSL2中linux的文件系统
- 具有恢复功能
- 周期性执行备份计划
- 配置简单，上手容易，方便迁移
- 时间、空间代价适中（二者不可能兼得）
- （可选）具有加密功能

## 构建备份方案

最终我发现了一个不错的备份方案，使用[restic](https://restic.net/)作为主要备份工具，它是支持Linux、BSD、Mac、Windows等多平台的开源备份工具，使用Go语言实现，备份的基本原理是将数据去重分块压缩，保存快照，因此运行起来十分高效且空间占用较小。它的GitHub仓库有将近30K Star，并且有十分详细的文档说明。

因此，restic作为主要的备份工具，可以满足绝大部分需求，唯一需要解决的问题是，如何进行周期性的备份，以及如何恰当地把WSL2中的数据备份到备份盘当中。在Linux Host系统中，解决这一问题的方案并不复杂，可以使用cron或者[autorestic](https://autorestic.vercel.app/)等工具来实现定期的备份，只要备份盘一直处于挂载状态即可。但在WSL2中，这一方案并不可行，主要因为以下原因：

1. 备份计划的执行时间一般设置在半夜或者凌晨等不使用计算机的时间段，这种情况下，如果WSL2中没有任务运行，Windows会自动关闭WSL2以节省资源，因此cron等计划很可能无法被正常执行；
2. 其次，默认配置下打开WSL2子系统时，备份盘是没有被挂载的，而挂载备份盘又必须要在Windows中以管理员权限执行命令`wsl.exe --unmount \\.\PHYSICALDRIVE0`（其中PHYSICALDRIVE0的末尾数字编号取决于具体平台）。

梳理一下，要实现每天自动化的备份任务执行，需要按次序完成以下子任务：

1. 在Powershell中以管理员权限执行命令，给WSL2子系统挂载备份盘；
2. 启动WSL2，在子系统中挂载备份盘到挂载点；
3. 执行备份任务（restic命令），写入备份日志；
4. 完成备份后，卸载备份盘。

## 备份方案的实现

这些任务的出发和自动执行可以通过Windows的任务计划程序来实现。

### 核心任务：WSL Daily Backup

首先在任务计划程序中创建任务"WSL Daily Backup"，配置如下：

- 勾选“使用最高权限运行”
- 触发器设置为每天凌晨2点
- 操作为启动程序，执行`wsl.exe -u username -e /home/username/backup/backup.sh`（填写备份脚本路径）

这一任务确保在指定的任务执行时间，可以启动备份脚本，但我们必须要确保任务执行前完成备份盘的挂载。

### 先导任务：Mount WSL Disks

为确保备份盘被正确挂载，在任务计划程序中创建新的任务"Mount DSL Disks"，配置如下：

- 勾选“使用最高权限运行”
- 无触发器设置
- 操作为启动程序，执行`C:\Windows\System32\wsl.exe --mount \\.\PHYSICALDRIVE0 --partition 1 -t ext4`
- 在设置中，勾选“允许按需运行任务”

这一任务执行Windows侧的挂载操作，无需设置触发器的原因是，我们将会在WSL2的启动过程中直接执行该任务。

其中的`--partition 1 -t ext4`取决于备份盘的具体分区设置和文件系统，设备号PHYSICALDRIVE0可以通过在Powershell中执行`Get-Disk`命令查看。

### WSL的启动脚本

配置好Mount WSL Disks任务后，参考[Medium一篇博客](https://medium.com/@stefan.berkner/automatically-starting-an-external-encrypted-ssd-in-windows-subsystem-wsl-6403c34e9680)分享的启动脚本，创建启动时执行的脚本`/mount.sh`

```bash
#!/bin/bash

echo $(date -u) "Mounting WSL disks"

task_name="Mount WSL Disks"
windows_sys_path="/mnt/c/Windows/system32"
task_command="schtasks.exe /run /tn"
timeout=30
interval=1
target_drive_uuid="9724a17b-26c9-46a0-ad0a-68be88bad9a9"
mount_point="/mnt/backup_hdd"

$windows_sys_path/$task_command "$task_name"

elapsed=0
while ! blkid --uuid $target_drive_uuid; do
  if [ $elapsed -ge $timeout ]; then
    echo $(date -u) "Timed out waiting for $target_drive_uuid to be ready."
    exit 1
  fi

  echo $(date -u) "Waiting for $target_drive_uuid to be ready..."
  sleep $interval
  elapsed=$((elapsed + interval))
done

target_drive=$(blkid --uuid $target_drive_uuid)
echo $(date -u) "Drive $target_drive ($target_drive_uuid) is ready"

mount -t ext4 $target_drive $mount_point && echo $(date -u) "Mounted $target_drive as $mount_point"
```

其中的`task_name`必须与任务计划程序中的任务名称相同，`target_drive_uuid`设置为启动盘的UUID（例如通过`sudo blkid -o list`查看）。在启动WSL2时执行该脚本，`$window_sys_path/$task_command "$task_name"`会执行挂载备份盘任务，随后进入一个timeout设置为30秒的循环中，每一秒检查一次具有指定UUID的设备是否在线，检查到上线之后，就执行脚本中最后的挂载命令，否则超时就异常退出。

准备好这份脚本之后，再编辑`/etc/wsl.conf`，添加以下内容，在WSL2启动时执行`/mount.sh`脚本：

```
[boot]
command="bash /mount.sh"
```

这样一来，每次启动WSL2的时候，都会执行`mount.sh`脚本，执行Windows中的挂载备份盘任务，并在子系统中挂载到指定挂载点。完成挂载后，restic命令就可以向备份盘中的仓库写入备份了。

### restic的备份脚本

启动restic备份的脚本放置于`$HOME/backup/backup.sh`，内容如下：

```bash
#!/bin/bash

export RESTIC_REPOSITORY="/mnt/wsl/PHYSICALDRIVE0p1/workstation-wsl"
export RESTIC_PASSWORD_FILE="$HOME/backup/.restic_password"

SOURCE_DIR="$HOME$"
EXCLUDE_FILE="$HOME/backup/exclude_list"
LOG_FILE="${RESTIC_REPOSITORY}/logs/backup_$(date +'%Y-%m-%d').log"
RESTIC="$HOME/.local/bin/restic"

# abort in case of error
set -e

mkdir -p $(dirname ${LOG_FILE})

log_message() {
  echo "$(date +'%Y-%m-%d %H:%M:%S') - $1" >>${LOG_FILE}
}

log_message "--- Backup Task Started ---"
log_message "Source: ${SOURCE_DIR}"
log_message "Destination Repo: ${RESTIC_REPOSITORY}"
log_message "Starting 'restic backup' ..."

$RESTIC backup ${SOURCE_DIR} \
  --verbose \
  --exclude-file=${EXCLUDE_FILE} \
  --tag "automated" >>${LOG_FILE} 2>&1

log_message "'restic backup' completed."
log_message "Starting 'restic forget & prune' policy..."

$RESTIC forget \
  --keep-daily 14 \
  --keep-weekly 8 \
  --keep-monthly 12 \
  --keep-last 10 \
  --prune >>${LOG_FILE} 2>&1

log_message "--- Backup Task Finished Successfully ---"

echo "" >>${LOG_FILE}

exit 0
```

其中的`SOURCE_DIR`是要备份的内容，`RESTIC_REPOSITORY`是在备份盘中存储备份数据的仓库路径，`RESTIC_PASSWORD_FILE`是备份仓库的密码，`RESTIC`是restic二进制文件的路径，`EXCLUDE_FILE`类似gitignore文件，用来排除文件。

在执行完backup命令之后，立刻执行forget命令，按照参数指定的policy清理不需要的备份数据，节省硬盘空间。在我的设置中，清理备份数据的时候会保留最近14天、8个周、12个月，以及最近的10次备份快照。

## 总结

数据备份的重要性太容易被忽视，经常在重要数据丢失之后才追悔莫及。构建一套适合自己的备份方案，可能只需要半天时间加上几百块钱，但是却能够极大概率防止“毕业前夕数据一夜间消失”的悲惨事故发生。

虽然我的关键源码和手稿都在GitHub有托管，但关键的实验数据因为提及太过庞大并没有托管。下一步的计划是找到合适的云备份方案并且融入到现有的restic备份计划中，真正实现狡兔三窟。
