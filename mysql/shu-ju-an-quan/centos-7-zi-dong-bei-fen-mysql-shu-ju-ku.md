---
cover: >-
  https://images.unsplash.com/photo-1542903660-eedba2cda473?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHwyfHxkYXRhfGVufDB8fHx8MTcwOTkzNzM1OXww&ixlib=rb-4.0.3&q=85
coverY: 0
---

# CentOS 7 自动备份 MySQL 数据库

在这篇文章中，我们将探讨如何在 CentOS 7 系统上为 MySQL 数据库设置自动备份。这对于确保您的数据安全和完整性至关重要，特别是对于重要的业务数据。我们将使用 `mysqldump` 工具来备份测试的 `halo` 数据库，并通过 `cron` 实现自动化。

## 前置条件

* 拥有 CentOS 7 服务器。
* 安装并运行 MySQL 数据库。
* 有足够的权限来创建备份脚本并设置 `cron` 任务。

## 创建备份脚本

使用文本编辑器（如 `nano` 或 `vi`），创建一个新的备份脚本：

```bash
sudo vi /usr/local/bin/mysql_backup.sh
```

然后，复制并粘贴以下脚本：

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/path/to/backup"
DB_USER="your_username"
DB_PASSWORD="your_password"
DB_NAME="halo"

mysqldump -u $DB_USER -p$DB_PASSWORD $DB_NAME > "$BACKUP_DIR/halo_backup_$DATE.sql"
```

替换 `BACKUP_DIR`、`DB_USER` 和 `DB_PASSWORD` 为你的实际备份目录和 MySQL 凭据。

可以为 `BACKUP_DIR` 添加一下执行权限：

```bash
sudo chmod -R 700 /path/to/backup
```

使脚本可执行：

```bash
sudo chmod +x /usr/local/bin/mysql_backup.sh
```

## 设置 Cron 任务

编辑 `cron` 表：

```bash
sudo crontab -e
```

在 `cron` 表的底部添加一行来定时运行备份脚本。例如，要每天凌晨 2 点运行备份，添加：

```bash
0 2 * * * /usr/local/bin/mysql_backup.sh
```

这行指令意味着每天的 02:00 执行脚本。

## 测试备份脚本

手动运行备份脚本：如果执行失败请看【常见问题一节】

```bash
/usr/local/bin/mysql_backup.sh
```

检查备份文件，确保在指定的 `BACKUP_DIR` 中出现了新的 SQL 备份文件。

![](https://javgo-images.oss-cn-beijing.aliyuncs.com/2024-02-24-074151.png)

## 备份安全和维护

* **存储安全**：考虑将备份文件存储在安全的位置，如外部硬盘或云存储服务。
* **定期检查**：定期检查 `cron` 日志和备份目录，确保备份过程按计划运行。
* **老旧备份清理**：根据存储空间和备份策略，定期清理旧的备份文件。

## 一键恢复

备份文件是有了，那么我们又该如何快速恢复呢？

那自然是再创建一个 Bash 脚本，用于从指定备份目录中恢复 MySQL 数据库。这个脚本假设你的备份是用 `mysqldump` 创建的 SQL 文件，并且你已经知道要恢复的具体文件名（从备份文件中选择具体恢复时间节点的备份文件）。

创建并编辑恢复脚本：

```bash
sudo vi /usr/local/bin/mysql_restore.sh
```

恢复脚本内容如下：

```bash
#!/bin/bash

# 设置数据库凭据
DB_USER="your_username"
DB_PASSWORD="your_password"
DB_NAME="halo"

# 设置备份目录
BACKUP_DIR="/path/to/backup"

# 检查是否提供了文件名
if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <backup-file.sql>"
    exit 1
fi

# 获取备份文件名
BACKUP_FILE=$1

# 完整路径
FULL_PATH="$BACKUP_DIR/$BACKUP_FILE"

# 检查文件是否存在
if [ ! -f "$FULL_PATH" ]; then
    echo "Backup file not found: $FULL_PATH"
    exit 1
fi

# 恢复数据库
mysql -u $DB_USER -p$DB_PASSWORD $DB_NAME < $FULL_PATH

echo "Database restored from $FULL_PATH"
```

赋予脚本执行权限：

```bash
chmod +x /usr/local/bin/mysql_restore.sh
```

在恢复数据时，先查看备份数据：

![img](https://javgo-images.oss-cn-beijing.aliyuncs.com/2024-02-24-74152.png)

然后执行恢复脚本即可：

```bash
/usr/local/bin/mysql_restore.sh halo_backup_20240224_123243.sql
```

注意事项：

1. 确保在运行恢复脚本之前数据库 `halo` 已经存在。如果不存在，需要先创建它。
2. 这个脚本将会替换目标数据库中现有的所有数据。在执行恢复操作之前，请确保没有重要数据会被不可逆地覆盖。
3. 为了安全起见，最好在执行这类操作前备份当前数据库状态。
4. 确保脚本中的用户名和密码与您的 MySQL 安装匹配，并根据需要调整它们。

## 自动清除备份文件

如果服务器资源紧张，我们不能无休止地存储数据库的备份文件，这时就需要在每次定时任务备份前执行一些清理逻辑。（当然，这是可选的操作）

为了自动清除旧的备份文件并仅保留最近七天的备份，可以在现有的 `/usr/local/bin/mysql_backup.sh` 脚本中添加一些代码来实现这一功能。这样，每次脚本运行时，它不仅会创建新的备份，还会清理旧的备份文件。

打开备份脚本进行编辑：

```bash
sudo vi /usr/local/bin/mysql_backup.sh
```

以下是修改后的脚本，包括自动清除旧备份的功能：

```bash
#!/bin/bash

# 设置数据库凭据
DB_USER="your_username"
DB_PASSWORD="your_password"
DB_NAME="halo"

# 设置备份目录
BACKUP_DIR="/path/to/backup"

# 当前日期
DATE=$(date +%Y%m%d_%H%M%S)

# 备份数据库
mysqldump -u $DB_USER -p$DB_PASSWORD $DB_NAME > "$BACKUP_DIR/halo_backup_$DATE.sql"

# 清除超过7天的旧备份
find $BACKUP_DIR -name "halo_backup_*.sql" -type f -mtime +7 -exec rm {} \;

echo "Backup complete, old backups cleared."
```

如果之前没有设置执行权限，运行：

```bash
sudo chmod +x /usr/local/bin/mysql_backup.sh
```

手动运行脚本以确保一切正常:

```bash
/usr/local/bin/mysql_backup.sh
```

检查备份是否创建成功，并且检查是否旧文件被正确删除。

注意事项：

1. 这个脚本中的 `find` 命令会删除超过 7 天的备份文件。`-mtime +7` 表示匹配所有修改时间超过 7 天的文件。
2. 在执行删除操作之前，`find` 命令不会有任何提示。如果你想在删除之前查看哪些文件将被删除，可以先运行 `find $BACKUP_DIR -name "halo_backup_*.sql" -type f -mtime +7` 命令。
3. 确保脚本中的路径和文件名与实际备份文件匹配。
4. 在正式部署此脚本之前，建议在一个安全的环境中进行测试，以避免意外数据丢失。

## 常见问题

如果出现错误提示：

```bash
[root@lavm-zzgegfex4j backup]# /usr/local/bin/mysql_backup.sh
/usr/local/bin/mysql_backup.sh: line 8: mysqldump: command not found
```

是因为脚本在执行时无法找到 `mysqldump` 命令。这通常是由于环境变量未正确设置或 `mysqldump` 未安装在默认的路径下导致的。

确保 `mysqldump` 确实已安装，可以通过运行以下命令来验证：

```bash
which mysqldump
```

或者：

```bash
mysql --version
```

这些命令应该返回 `mysqldump` 的路径或 MySQL 的版本信息。

如果 `mysqldump` 已安装但不在标准路径中，需要找到其完整路径。运行以下命令：

```bash
sudo find / -name mysqldump
```

![](https://javgo-images.oss-cn-beijing.aliyuncs.com/2024-02-24-074152.png)

找到 `mysqldump` 的完整路径后，更新备份脚本，将 `mysqldump` 替换为其完整路径。例如，如果 `mysqldump` 的路径是 `/usr/bin/mysqldump`，则将脚本中的相关行更新为：

```bash
/usr/local/mysql8/bin/mysqldump -u $DB_USER -p$DB_PASSWORD $DB_NAME > "$BACKUP_DIR/halo_backup_$DATE.sql"
```

另一个解决方案是将 `mysqldump` 的路径添加到您的 `PATH` 环境变量中。这可以通过编辑 `/etc/profile` 文件来完成。例如：

```bash
vim /etc/profile

export PATH=$PATH:/usr/local/mysql8/bin
```

之后，运行 `source /etc/profile` 以使更改生效。

## 总结

通过上述步骤，现在已经在你的 CentOS 7 服务器上成功设置了 MySQL 数据库的自动备份。这不仅保护你的数据免受意外丢失的风险，还为可能出现的灾难性事件提供了重要的数据恢复手段。定期备份是任何数据管理策略的关键部分，不容忽视。
