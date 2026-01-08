# Linux 速查表及最佳工程实践指南

## 一、基础命令速查

### 文件与目录操作

```bash
# 导航
pwd                     # 显示当前目录
ls -la                  # 列出文件（含隐藏文件）
cd /path                # 切换目录
cd ~                    # 返回家目录
cd -                    # 返回上一个目录

# 文件操作
cp -r src dest          # 递归复制
mv old new              # 移动/重命名
rm -rf dir              # 递归强制删除（慎用）
mkdir -p a/b/c          # 创建多级目录
touch file              # 创建空文件/更新时间戳

# 查看文件
cat file                # 显示全部内容
less file               # 分页查看
head -n 20 file         # 查看前20行
tail -n 30 file         # 查看后30行
tail -f logfile         # 实时跟踪日志
```



#### mkdir

- **The mkdir(make directory) command allows users to create directories or folders.**

```bash
$ mkdir ubuntu  # 创建一个目录名或者文件夹名为ubuntu
$ ls
ubuntu
```

- The option '-p' is used to create multiple directories or parent directories at once.

```bash
$ mkdir -p dir1/dir2/dir3
$ cd dir1/dir2/dir3
~/Desktop/Linux/dir1/dir2/dir3$
```





### 权限管理

```bash
chmod 755 script.sh     # 设置权限
chmod u+x file          # 给所有者添加执行权限
chown user:group file   # 修改所有者和组
chown -R user:group dir # 递归修改
```

### 搜索与查找
```bash
# 查找文件
find /path -name "*.log" -type f -mtime -7
find . -size +100M      # 查找大于100M的文件
find . -name "*.tmp" -delete  # 查找并删除

# 查找内容
grep -r "pattern" .     # 递归搜索
grep -i "error" log.txt # 忽略大小写
grep -v "debug" file    # 排除匹配行
grep -E "pattern1|pattern2"  # 正则匹配
```



## 二、系统监控与性能

### 资源监控
```bash
# CPU/内存/进程
top                     # 实时监控
htop                    # 增强版top
ps aux | grep process   # 查找进程
pidstat 1 5            # 监控进程资源
uptime                  # 系统运行时间

# 内存
free -h                # 内存使用情况
vmstat 1 10            # 虚拟内存统计

# 磁盘
df -h                  # 磁盘空间
du -sh /path           # 目录大小
du -h --max-depth=1    # 一级目录大小
iostat -x 1            # I/O统计
```



### 网络监控

```bash
# 网络连接
netstat -tulpn         # 监听端口
ss -tulpn              # 更快的netstat
lsof -i :8080          # 查看端口占用
netstat -an | grep ESTABLISHED | wc -l  # 统计连接数

# 带宽监控
iftop                  # 实时流量监控
nethogs                # 按进程监控流量
```

## 三、文本处理三剑客

### grep (文本搜索)
```bash
grep -E "^[A-Z]" file      # 正则匹配
grep -c "pattern" file     # 统计匹配行数
grep -B2 -A2 "error" log   # 显示匹配前后2行
grep -l "pattern" *.txt    # 只显示文件名
```

### sed (流编辑器)
```bash
sed 's/old/new/g' file     # 全局替换
sed -i.bak 's/old/new/' file  # 直接修改并备份
sed '/^#/d' config.conf    # 删除注释行
sed -n '5,10p' file        # 打印5-10行
sed '/pattern/,+3d' file   # 删除匹配行及后3行
```

### awk (数据处理)
```bash
awk '{print $1,$3}' file          # 打印第1和第3列
awk -F',' '{print $2}' file       # 指定分隔符
awk '$3 > 100 {print $0}' file    # 条件过滤
awk '{sum+=$1} END{print sum}'    # 求和
awk 'NR%2==1' file                # 打印奇数行
awk '!a[$0]++' file               # 去重
```

## 四、进程管理与作业控制

### 进程管理
```bash
# 启动与终止
nohup command &         # 后台运行，退出终端不停止
kill -9 PID             # 强制终止
killall process_name    # 终止所有同名进程
pkill -f pattern        # 按模式终止

# 优先级
nice -n 10 command      # 设置优先级
renice 15 -p PID        # 修改运行中进程优先级

# 监控
watch -n 1 'ps aux | grep http'  # 实时监控
```

### 作业控制
```bash
command &               # 后台运行
jobs                    # 查看后台作业
fg %1                   # 调到前台
bg %1                   # 继续后台运行
Ctrl+Z                  # 暂停任务
disown -h %1            # 移除作业从作业表
```

## 五、Shell脚本最佳实践

### 脚本头
```bash
#!/usr/bin/env bash
set -euo pipefail        # 严格模式
# -e: 出错立即退出
# -u: 使用未定义变量报错
# -o pipefail: 管道失败则整个失败
```

### 变量与常量
```bash
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/var/log/$(basename "$0").log"
readonly MAX_RETRY=3

# 变量命名
local_var="value"        # 局部变量
GLOBAL_CONFIG="/etc/app.conf"  # 全局常量用大写
```

### 函数定义
```bash
# 函数注释
# @desc: 下载文件
# @param: $1 URL, $2 保存路径
download_file() {
    local url="$1"
    local dest="$2"
    
    if [[ -z "$url" || -z "$dest" ]]; then
        log_error "参数缺失"
        return 1
    fi
    
    curl -fsSL "$url" -o "$dest" || {
        log_error "下载失败: $url"
        return 1
    }
}
```

### 错误处理
```bash
# 错误处理函数
log_error() {
    echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $*" >&2
}

# 重试机制
retry_with_backoff() {
    local max_attempts=3
    local delay=1
    local attempt=1
    
    until "$@"; do
        if (( attempt == max_attempts )); then
            log_error "重试 $max_attempts 次后失败"
            return 1
        fi
        
        log_error "尝试 $attempt 失败，${delay}秒后重试"
        sleep $delay
        ((attempt++))
        ((delay*=2))
    done
}
```

### 日志管理
```bash
setup_logging() {
    exec 1> >(tee -a "$LOG_FILE")
    exec 2> >(tee -a "$LOG_FILE" >&2)
    exec 19>> "$LOG_FILE"
    BASH_XTRACEFD=19
    set -x  # 开启调试
}
```

## 六、系统管理实践

### 用户与权限
```bash
# 创建受限用户
sudo useradd -r -s /sbin/nologin appuser
sudo usermod -aG docker appuser  # 添加用户组

# sudo配置
echo "appuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart app" >> /etc/sudoers
```

### 服务管理
```bash
# Systemd服务配置最佳实践
# /etc/systemd/system/app.service
[Unit]
Description=My Application
After=network.target
Requires=network.target

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/opt/app
ExecStart=/usr/bin/python3 app.py
Restart=on-failure
RestartSec=5
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"
Environment="APP_ENV=production"

# 限制资源
LimitNOFILE=65535
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```



### 定时任务

```bash
# Crontab配置
# 分钟 小时 日期 月份 星期 命令
0 2 * * * /opt/backup.sh >> /var/log/backup.log 2>&1
# 每5分钟
*/5 * * * * /opt/health_check.sh

# 系统级cron
/etc/cron.daily/    # 每日执行
/etc/cron.hourly/   # 每小时执行
```



## 七、安全最佳实践

### SSH安全
```bash
# /etc/ssh/sshd_config 配置
Port 2222                    # 修改默认端口
PermitRootLogin no           # 禁止root登录
PasswordAuthentication no    # 禁用密码认证
PubkeyAuthentication yes     # 启用密钥认证
AllowUsers deployuser        # 只允许特定用户
MaxAuthTries 3               # 最大尝试次数
ClientAliveInterval 300      # 连接超时设置
ClientAliveCountMax 2

# 生成密钥对
ssh-keygen -t ed25519 -C "user@host"
ssh-copy-id user@host        # 复制公钥
```

### 防火墙配置
```bash
# UFW (Ubuntu)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 80/tcp        # HTTP
sudo ufw allow 443/tcp       # HTTPS
sudo ufw enable

# firewalld (CentOS)
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```



## 八、性能调优

### 系统参数优化
```bash
# /etc/sysctl.conf 优化
# 网络优化
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535

# 内存优化
vm.swappiness = 10
vm.dirty_ratio = 60
vm.dirty_background_ratio = 2

# 文件句柄
fs.file-max = 2097152
fs.nr_open = 2097152

# 生效配置
sysctl -p
```

### 磁盘I/O优化
```bash
# 查看磁盘调度器
cat /sys/block/sda/queue/scheduler

# 使用deadline或noop调度器
echo deadline > /sys/block/sda/queue/scheduler

# 调整虚拟内存参数
echo "vm.dirty_writeback_centisecs = 500" >> /etc/sysctl.conf
```



## 九、故障排查流程

### 系统故障排查
```bash
# 1. 查看系统日志
journalctl -xe              # 查看最近的错误
dmesg -T | tail -100        # 内核消息

# 2. 检查资源使用
top -c                      # 查看CPU使用
free -m                     # 查看内存
df -h                       # 查看磁盘
iostat -dx 1 5              # I/O统计

# 3. 网络检查
ping -c 4 google.com        # 网络连通性
traceroute google.com       # 路由追踪
mtr google.com              # 持续路由追踪

# 4. 进程分析
strace -p PID               # 系统调用跟踪
lsof -p PID                 # 打开文件
/proc/PID/status            # 进程状态
```

### 性能瓶颈分析
```bash
# CPU瓶颈
perf top                    # 实时性能分析
mpstat -P ALL 1             # 多核CPU统计

# 内存瓶颈
cat /proc/meminfo           # 详细内存信息
slabtop                     # 内核缓存使用

# I/O瓶颈
iotop                      # 实时I/O监控
blktrace                   # 块设备跟踪
```



## 十、容器化最佳实践

### Docker使用
```bash
# 最小化镜像
FROM alpine:latest
RUN apk add --no-cache python3

# 非root用户运行
USER appuser

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/health || exit 1

# 资源限制
docker run --memory="512m" --cpus="1.5" app
```

### Docker Compose配置
```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    restart: unless-stopped
    environment:
      - NODE_ENV=production
    volumes:
      - app-data:/data
    networks:
      - app-network
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  app-data:
  
networks:
  app-network:
    driver: bridge
```



## 十一、备份与恢复

### 备份策略
```bash
# 全量备份
tar -czpf backup-$(date +%Y%m%d).tar.gz /data

# 增量备份
rsync -av --link-dest=/backup/yesterday /data /backup/today

# 数据库备份
mysqldump -u root -p dbname | gzip > db-$(date +%F).sql.gz
pg_dump dbname | gzip > db-$(date +%F).sql.gz

# 加密备份
tar -czf - /data | openssl enc -aes-256-cbc -salt -out backup.tar.gz.enc
```



## 十二、实用工具推荐

### 现代化替代工具
```bash
# 文件列表
exa -la --git              # 替代ls

# 文件查看
bat file.txt               # 替代cat，带语法高亮

# 文件查找
fd "pattern"               # 替代find，更快更友好

# 代码搜索
rg "pattern"               # 替代grep，超快

# 磁盘分析
ncdu                       # 交互式磁盘使用分析

# 进程监控
btop                       # 现代化top替代品

# 网络调试
httpie                     # 现代化curl替代
```



## 十三、Shell配置优化

### ~/.bashrc 配置
```bash
# 别名
alias ll='ls -alhF'
alias grep='grep --color=auto'
alias df='df -h'
alias du='du -h'

# 历史记录优化
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTCONTROL=ignoreboth:erasedups
shopt -s histappend

# 提示符
PS1='$$\e[1;32m$$\u@\h$$\e[0m$$:$$\e[1;34m$$\w$$\e[0m$$\$ '

# 安全
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# 函数
extract() {
    if [ -f $1 ]; then
        case $1 in
            *.tar.bz2)   tar xjf $1     ;;
            *.tar.gz)    tar xzf $1     ;;
            *.bz2)       bunzip2 $1     ;;
            *.rar)       unrar x $1     ;;
            *.gz)        gunzip $1      ;;
            *.tar)       tar xf $1      ;;
            *.tbz2)      tar xjf $1     ;;
            *.tgz)       tar xzf $1     ;;
            *.zip)       unzip $1       ;;
            *.Z)         uncompress $1  ;;
            *.7z)        7z x $1        ;;
            *)           echo "'$1' cannot be extracted" ;;
        esac
    else
        echo "'$1' is not a valid file"
    fi
}
```



## 十四、重要目录说明

```bash
/etc/            # 系统配置文件
/var/log/        # 日志文件
/var/spool/      # 队列数据（邮件、打印等）
/var/tmp/        # 临时文件（重启保留）
/tmp/            # 临时文件（重启清空）
/home/           # 用户家目录
/opt/            # 第三方软件
/usr/local/      # 本地安装软件
/proc/           # 进程和内核信息
/sys/            # 系统设备信息
```



## 总结

### 最佳实践要点

1. **脚本安全**：始终使用 `set -euo pipefail`，验证输入
2. **错误处理**：记录日志，合理重试，优雅退出
3. **资源管理**：设置资源限制，监控使用情况
4. **权限最小化**：使用非root用户，严格控制sudo
5. **备份策略**：定期测试恢复流程
6. **文档化**：记录配置变更和故障处理
7. **自动化**：重复任务脚本化，使用配置管理工具
8. **监控告警**：关键指标监控，及时告警

### 紧急情况处理清单

1. **磁盘满**：清理日志，扩展磁盘
2. **内存不足**：检查内存泄漏，优化配置
3. **CPU 100%**：分析进程，优化代码
4. **服务宕机**：检查日志，重启服务
5. **网络异常**：检查配置，重启网络服务

这份指南覆盖了Linux系统管理和工程实践的各个方面，建议结合实际工作场景灵活应用，并持续更新优化。
