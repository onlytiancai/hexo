---
title: 从零搞定 MySQL 备份
date: 2024-11-21 11:55:49
tags:
---
在今天的数字化世界里，数据库崩溃可能带来的损失远远超出我们的想象。所以，备份不再只是一个技术任务，它已经变成了每一个数据管理者和企业领导者必须思考和解决的生死存亡的关键。

今天我们尝试讨论如下问题：
1. 如何快速的备份 MySQL 的所有数据？
2. 如何定时自动化的备份数据？
3. 如何做好远程数据备份？
4. 如何在备份失败时发出告警？
5. 如何建立一套高效可靠的备份策略？
6. 如何做备份可用性检测和恢复演练？


### 1. 如何快速的备份 MySQL 的所有数据？

MySQL 的备份工具有很多，[mysqldump](https://dev.mysql.com/doc/refman/8.4/en/mysqldump.html), [mysqlpump](https://dev.mysql.com/doc/refman/5.7/en/mysqlpump.html), [mysql shell](https://dev.mysql.com/doc/mysql-shell/8.4/en/), [mydumper/myloader](https://github.com/mydumper/mydumper), [xtrabackup](https://www.percona.com/mysql/software/percona-xtrabackup)等，我们从最简单的 mysqldump 开始，它足够简单可靠，而且随 MySQL 一起发行，不需要额外安装。

最普遍的需求是备份整个实例的所有数据库，包含每个数据库的所有表，存储过程，函数，视图等。

```bash
mysqldump -h 127.0.0.1 -u root -ppassword --all-databases --routines --triggers --single-transaction --master-data=2 > ./`date +%Y%m%d%H%M%S`.sql
```

这条命令是使用 `mysqldump` 工具进行 MySQL 数据库备份的命令。让我们逐部分分析它的功能：

1. **`mysqldump`**：
   - `mysqldump` 是 MySQL 提供的一个备份工具，用于导出 MySQL 数据库或数据库中的数据。

2. **`-h 127.0.0.1`**：
   - 这个选项指定 MySQL 服务器的主机地址。`127.0.0.1` 是本地回环地址，表示连接到本地 MySQL 实例。也可以使用 `localhost` 或服务器的 IP 地址。

3. **`-u root`**：
   - 指定登录 MySQL 的用户名为 `root`。`-u` 后面跟的是用户名。

4. **`-ppassword`**：
   - `-p` 后面跟的是密码，表示连接 MySQL 时使用的密码。在这个例子中，密码是 `password`。注意：`-p` 和密码之间不能有空格。

5. **`--all-databases`**：
   - 这个选项指定备份所有的数据库，而不仅仅是某一个数据库。如果省略此选项，则默认只备份指定的数据库。

6. **`--routines`**：
   - 该选项用于备份数据库中的存储过程和存储函数（routines）。如果你有使用存储过程或函数，这个选项很重要，否则它们不会被包含在备份中。

7. **`--triggers`**：
   - 这个选项用于备份触发器（triggers）。触发器是数据库中的一些自动执行的操作，可以在插入、更新或删除时触发。

8. **`--single-transaction`**：
   - 这个选项在备份时确保事务一致性，适用于支持事务的数据库引擎（如 InnoDB）。它确保在备份过程中，数据库中的数据不会被修改，从而保证了备份的一致性。这是通过在开始备份时使用 `START TRANSACTION` 来完成的，适用于使用事务的存储引擎。

9. **`--master-data=2`**：
   - 这个选项用于备份时记录主从复制的相关信息，特别是在设置 MySQL 主从复制时。`--master-data=2` 会在备份文件中加入一个注释，记录主服务器的二进制日志位置及文件名，通常用于恢复时设置复制。
   - `=2` 的含义是将复制信息以注释的形式写入备份文件，而不是执行语句。

10. **`> ./'date +%Y%m%d%H%M%S'.sql`**：
    - `>` 是输出重定向符号，将命令的输出结果保存到一个文件中。
    - `./date +'%Y%m%d%H%M%S'.sql` 是备份文件的名称，使用了 `date` 命令来生成一个基于当前时间的文件名。具体来说，`date +%Y%m%d%H%M%S` 会生成类似 `20241121094500` 格式的日期时间戳。这个时间戳确保每次备份时文件名不同，以避免覆盖现有的备份文件。

我们能想到一些更深入的个性化需求：
1. 如何只备份某个数据库？
2. 如何只备份某个表？
3. 如何只备份数据，不备份表结构？
4. 如何把不同的表备份到各自独立的文件？ 
5. 如何压缩备份文件？
6. 如何加密备份文件？
7. 如何多线程备份数据以提高备份速度？
8. 如何显示备份的进度及预估备份时长？
9. 如何避免在 Shell 命令行中输入密码以提高安全性？

这些高级需求我们以后再讨论，对于新手来说，上面一行代码足以满足 90% 的需求。

### 2. 如何定时自动化的备份数据？

首先想到的是使用 linux 自带的 crontab 机制，这要求我们把备份命令写到一个 shell 文件中，假设我们创建的文件名叫 `mysql_backup.sh`，它看起来像这样：

```shell
#!/bin/bash
set -euo pipefail

FILE=./`date +%Y%m%d%H%M%S`.sql
mysqldump -h 127.0.0.1 -u root -ppassword --all-databases --single-transaction --master-data=2 > \$FILE
```

前两行是 Bash 脚本的惯用写法，用于确保脚本的可靠性和执行的一致性，以下是每一行的解释：

- `#!/bin/bash` 告诉系统用 `/bin/bash` 解释器来执行脚本。
- `set -euo pipefail`
    - `-e`：避免错误被忽略，防止脚本在意外情况下继续运行。
    - `-u`：防止因拼写错误或遗漏变量赋值导致脚本出现难以调试的问题。
    - `-o pipefail`：提高错误检测的可靠性，确保管道中的任意一部分命令失败时，整个管道返回失败状态。

后面两行是我们上一步的备份命令。

输入 `crontab -e`，会用默认编辑器打开 crontab 规则文件，假设上面的 `mysql_backup.sh` 文件在 Home 目录 `~/` 下， 我们在编辑器里输入如下内容：

```
0 0 * * * cd ~/ && ./mysql_backup.sh >>./mysql_backup.log 2>&1 &
```

该命令表示每天的午夜 00:00（凌晨零点）运行一次备份脚本，以下是每个字段解析，可以根据自己的需要进行修改。

| 字段位置 | 值   | 取值范围          | 说明                       |
|----------|-------|-------------------|----------------------------|
| 分钟 (Minute) | `0`   | `0-59`           | 表示任务将在第 0 分钟执行。     |
| 小时 (Hour)   | `0`   | `0-23`           | 表示任务将在第 0 小时执行（午夜）。|
| 日 (Day of Month) | `*`   | `1-31`          | 表示不限制日期（每天都运行）。    |
| 月 (Month)   | `*`   | `1-12` (或 `JAN-DEC`) | 表示不限制月份（每月都运行）。    |
| 星期 (Day of Week) | `*`   | `0-7` (或 `SUN-SAT`) | 表示不限制星期（每天都运行）。    |

编辑完成后要保存退出，如果使用 vi 编辑器，输入 `:wq`，如果使用 nano 编辑器，先按 `ctrl+o` 保存，然后按 `ctrl+x` 退出。

如果你认为上面的操作有些复杂，那么我们可以一步到位，复制如下代码在终端里执行即可，如有特殊需求，在复制前做好修改。

```bash
cat <<EOF > ./mysql_backup.sh
#!/bin/bash
set -euo pipefail

FILE=./`date +%Y%m%d%H%M%S`.sql
mysqldump -h 127.0.0.1 -u root -ppassword --all-databases --single-transaction --master-data=2 > \$FILE
rsync -avzP \$FILE ubuntu@bak-server:/data/backup/
EOF

crontab -l > crontab_backup_$(date +'%Y%m%d_%H%M%S').txt 2>/dev/null
new_rule="0 0 * * * cd $PWD && ./mysql_backup.sh >>./mysql_backup.log 2>&1 &"
(crontab -l 2>/dev/null; echo "$new_rule") | crontab -
```

后面三行的目的是 **备份当前的 crontab 规则** 并 **向 crontab 添加新规则**，以下是逐行的详细解释。

第一行：将当前的 crontab 内容保存到一个带时间戳的文件中，作为备份。
- `crontab -l`: 列出当前用户的所有定时任务。
- `>`: 将输出重定向到文件。
- `crontab_backup_$(date +'%Y%m%d_%H%M%S').txt`:  
  - `$(date +'%Y%m%d_%H%M%S')`: 获取当前时间，格式为 `YYYYMMDD_HHMMSS`（如 `20241121_150300`）。
  - 文件名形如 `crontab_backup_20241121_150300.txt`，用来保存当前 crontab 的内容。
- `2>/dev/null`: 将标准错误（错误信息）重定向到 `/dev/null`，避免在没有 crontab 任务时输出错误信息。

第二行：定义一个任务，每天午夜备份 MySQL 数据，并将日志记录到当前目录下的 mysql_backup.log。
- `new_rule`: 定义了一个变量，内容是新的 crontab 任务。
- `0 0 * * *`: 每天的午夜（00:00）执行该任务。
- `cd $PWD`: 切换到当前工作目录（由环境变量 `$PWD` 确定）。
- `./mysql_backup.sh`: 执行 `mysql_backup.sh` 脚本。
- `>> ./mysql_backup.log`: 将脚本的标准输出 **追加** 到日志文件 `mysql_backup.log`。
- `2>&1`: 将脚本的标准错误 **重定向** 到标准输出（即一并写入 `mysql_backup.log`）。
- `&`: 在后台运行该命令（不阻塞 crontab 进程）。

第三行：将当前 crontab 内容和新规则合并后，重新加载到 crontab 中，最终实现追加新任务的效果。
- `( ... )`: 子进程运行括号内的命令。
- `crontab -l 2>/dev/null`: 获取当前的 crontab 内容。如果没有任务，忽略错误信息（通过 `2>/dev/null`）。
- `echo "$new_rule"`: 将新规则输出到标准输出。
- `;`: 将两个命令（`crontab -l` 和 `echo`）的输出合并。
- `|`: 将合并后的输出通过管道传递给下一个命令。
- `crontab -`: 用新的标准输入内容覆盖当前用户的 crontab。

我们可能会想到一些问题：
1. 如何解决备份过多引起磁盘剩余空间不足的问题？
2. 如何做增量备份和差异备份以减少磁盘占用？

这些问题以后我们会详细讨论。

### 3. 如何做好远程数据备份？

目前为止我们已经可以定期的把数据库备份到本地磁盘了，但如果服务器磁盘损坏或者机房出现火灾时，这些备份也会随之失效，为了提高备份的可用性，我们需要对数据做远程异地备份。

我们可以买一台拥有大容量低性能的磁盘的服务器用来专门保存备份数据，它们通常不会太贵。第一步我们要配置好数据库服务器到备份服务器的 SSH 免密登录，以下是操作步骤。

首先确保 MySQL 服务器上已经生成了密钥对，如果没有，使用以下命令生成 SSH 密钥对：

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

- **`-t rsa`**：指定密钥类型为 RSA。
- **`-b 4096`**：指定密钥长度为 4096 位（更安全）。
- **`-C "your_email@example.com"`**：为密钥添加一个注释，通常是你的邮箱。

过程：
- 系统会提示你输入密钥的存储路径：
  - 如果不需要自定义路径，直接按 **Enter**，密钥将存储在默认位置：`~/.ssh/id_rsa`（私钥）和 `~/.ssh/id_rsa.pub`（公钥）。
- 系统可能询问是否设置密钥密码：
  - 如果需要更高的安全性，可以设置密码；
  - 如果希望完全免密登录，直接按 **Enter** 跳过密码。


然后要将生成的公钥文件 (`~/.ssh/id_rsa.pub`) 复制到远程服务器上。

方法 1：使用 `ssh-copy-id`（推荐），运行以下命令，将公钥自动添加到远程服务器：

```bash
ssh-copy-id user@remote_server
```

- **`user`**：远程服务器的用户名。
- **`remote_server`**：远程服务器的 IP 地址或域名。
- 系统会提示输入远程服务器的密码，验证后自动将公钥添加到远程服务器的 `~/.ssh/authorized_keys` 文件中。

方法 2：如果 `ssh-copy-id` 不可用，可以手动复制公钥到远程服务器：

1. 查看本地公钥内容：
   ```bash
   cat ~/.ssh/id_rsa.pub
   ```
2. 登录到远程服务器：
   ```bash
   ssh user@remote_server
   ```
3. 在远程服务器上，将公钥添加到 `~/.ssh/authorized_keys` 文件中：

   ```bash
   mkdir -p ~/.ssh
   echo "your_public_key_content" >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   chmod 700 ~/.ssh
   ```

---

最后在本地尝试登录远程服务器：
```bash
ssh user@remote_server
```

- 如果配置正确，应该直接登录远程服务器，无需输入密码。
- 如果仍提示输入密码，检查以下几点：
  - 本地私钥是否正确存放在 `~/.ssh/id_rsa`。
  - 远程服务器的 `~/.ssh/authorized_keys` 是否包含公钥，权限是否正确。

免密登录配置好后，我们需要在 `mysql_backup.sh` 中加入 rsync 命令，把本地备份同步到远程服务器，如下：

```bash
cat <<EOF > ./mysql_backup.sh
#!/bin/bash
set -euo pipefail

FILE=./`date +%Y%m%d%H%M%S`.sql
mysqldump -h 127.0.0.1 -u root -ppassword --all-databases --single-transaction --master-data=2 > \$FILE
rsync -avzP \$FILE ubuntu@bak-server:/data/backup/
EOF

crontab -l > crontab_backup_$(date +'%Y%m%d_%H%M%S').txt 2>/dev/null
new_rule="0 0 * * * cd $PWD && ./mysql_backup.sh >>./mysql_backup.log 2>&1 &"
(crontab -l 2>/dev/null; echo "$new_rule") | crontab -
```

新增的命令使用 `rsync` 工具将本地的文件或目录 **\$FILE** 同步到远程服务器 **bak-server** 上的路径 **/data/backup/**。以下是对该命令的详细解释，可根据需要自行修改。

- **`-a` (archive mode)**
 归档模式，表示递归同步目录，同时保留文件的权限、时间戳、符号链接等元数据。
- **`-v` (verbose)**
 显示详细的同步过程。
- **`-z` (compress)**
 在传输时对数据进行压缩，提高传输效率（特别是在网络带宽有限时）。
- **`-P` (progress + partial)**
    - **`--progress`**：显示每个文件的传输进度。
    - **`--partial`**：如果传输中断，可以保留未完成的部分文件，方便下次继续传输。
- **`ubuntu@bak-server:/data/backup/`**
   - **`ubuntu`**：表示远程服务器上的用户名。
   - **`bak-server`**：远程服务器的主机名或 IP 地址。
   - **`/data/backup/`**：远程服务器上的目标目录，文件或目录会被同步到这里。

有时候我们会想把数据库备份在 AWS S3 上，因为它便宜且可靠，我们会在以后讨论这个话题。


### 未完待续...
