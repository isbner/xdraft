# 在 Ubuntu 上安装与配置 v2rayA 客户端

> 适用环境：Ubuntu 20.04 / 22.04 / 24.04（x86_64 架构）
> v2rayA 是一个开源的 V2Ray/Xray 图形化客户端，通过 Web 界面管理代理节点，支持订阅导入、透明代理、系统代理等功能。

---

## 一、什么是 v2rayA

v2rayA 是一个基于 Web 界面的 V2Ray 客户端，它本身不包含代理内核，而是调用 V2Ray 或 Xray 内核完成代理工作。其架构大致如下：

```
浏览器 (Web UI, 127.0.0.1:2017)
        │
        ▼
   v2rayA 服务 ──调用──▶ V2Ray / Xray 内核
        │                    │
        ▼                    ▼
   系统代理设置           SOCKS5 / HTTP 出站
                           │
                           ▼
                      远端代理服务器
```

**组成部分：**
- **v2rayA 主程序**：负责 Web 管理界面、节点管理和规则路由。
- **V2Ray / Xray 内核**：真正执行代理转发工作的核心程序。

---

## 二、安装前准备

### 1. 更新系统软件源

```bash
sudo apt update
sudo apt upgrade -y
```

### 2. 安装必要工具

```bash
sudo apt install curl wget -y
```

---

## 三、安装步骤

### 第 1 步：安装 V2Ray 内核

使用官方一键脚本安装 V2Ray 内核：

```bash
sudo bash -c "$(curl -L -s https://install.v2raya.org/v2ray-linux-amd64.sh)"
```

> 说明：
> - 脚本会下载并安装 V2Ray 内核到 `/usr/local/bin/v2ray`，配置文件位于 `/usr/local/etc/v2ray/`。
> - 如果该地址无法访问（网络受限），可以改为手动从 GitHub Releases 下载 `v2ray-linux-64.zip` 解压到 `/usr/local/bin/`。
> - 也可以安装 Xray 内核（v2rayA 同样支持），脚本地址为 `https://install.v2raya.org/xray-linux-amd64.sh`。

验证内核是否安装成功：

```bash
v2ray version
```

### 第 2 步：添加 v2rayA 软件源并安装

```bash
# 导入官方 GPG 公钥
wget -qO - https://apt.v2raya.org/key/public.key | sudo tee /etc/apt/trusted.gpg.d/v2raya.asc

# 添加软件源
echo "deb https://apt.v2raya.org v2raya main" | sudo tee /etc/apt/sources.list.d/v2raya.list

# 更新软件源并安装
sudo apt update
sudo apt install v2raya -y
```

### 第 3 步：启动 v2rayA 服务

```bash
# 设置开机自启并立即启动
sudo systemctl enable --now v2raya

# 查看运行状态
sudo systemctl status v2raya
```

看到 `active (running)` 即表示服务启动成功。

---

## 四、Web 界面配置

### 1. 登录管理界面

浏览器访问：`http://127.0.0.1:2017`

首次访问会进入**创建账号**页面，设置用户名和密码（此账号仅用于本机 Web 管理登录），然后登录。

### 2. 添加代理节点

v2rayA 支持两种方式添加节点：

**方式 A：导入订阅链接（推荐）**
1. 点击左侧「订阅」→「新建订阅」。
2. 输入订阅名称，填入你购买服务商提供的订阅 URL。
3. 点击「导入」按钮，节点会自动同步到列表中。

**方式 B：手动添加单个节点**
1. 点击左侧「节点」→「添加节点」。
2. 填入节点地址、端口、用户 ID、传输协议等信息（与服务器商提供的参数对应）。

> 作业演示场景：如果没有真实服务器，可以使用 v2rayA 内置的本地配置，或者创建仅供本机测试的入站出站配置来演示界面功能。

### 3. 连接节点

1. 在「节点」列表中找到目标节点，点击右侧的 **▶ 连接** 按钮。
2. 连接成功后，节点状态变为「已连接」。

### 4. 启用系统代理

1. 点击顶部的「设置」图标（齿轮）。
2. 在「代理」区域点击 **系统代理** 开关，v2rayA 会自动配置系统网络代理。
3. 默认端口说明：
   - `2017`：Web 管理界面端口
   - `20171`：SOCKS5 代理端口（默认 `127.0.0.1:20171`）
   - `20172`：HTTP 代理端口（默认 `127.0.0.1:20172`）
   - `20170`：透明代理端口（TPROXY，透明代理模式下使用）

---

## 五、验证代理是否生效

### 方法 1：命令行测试

```bash
# 通过 v2rayA 的 HTTP 代理访问测试网站
curl -x http://127.0.0.1:20172 https://www.google.com

# 通过 SOCKS5 代理测试
curl -x socks5://127.0.0.1:20171 https://www.google.com
```

返回正常 HTML 内容即表示代理链路通畅。

### 方法 2：浏览器测试

打开浏览器访问 `https://ip.sb` 或 `https://www.ipify.org`，查看显示的出口 IP 是否为代理服务器所在地区。

### 方法 3：查看内核日志

```bash
sudo journalctl -u v2raya -f
```

可实时查看 v2rayA 与内核的运行日志，便于排查问题。

---

## 六、开机自启与常用管理命令

```bash
# 查看状态
sudo systemctl status v2raya

# 停止
sudo systemctl stop v2raya

# 启动
sudo systemctl start v2raya

# 重启
sudo systemctl restart v2raya

# 关闭开机自启
sudo systemctl disable v2raya
```

---

## 七、常见问题排查

| 问题 | 原因 | 解决办法 |
|------|------|---------|
| Web 界面打不开 | v2rayA 服务未启动 | `sudo systemctl start v2raya` 并检查 `2017` 端口是否被占用 |
| 无法连接节点 | 内核未安装 | 确认执行过 `v2ray version` 能输出版本号 |
| 代理不生效 | 系统代理未开启 | 在 Web 界面打开「系统代理」开关 |
| 防火墙拦截 | UFW 防火墙开启 | `sudo ufw allow 2017`（按需开放端口） |
| 订阅导入失败 | 网络无法访问订阅地址 | 先手动添加单个节点测试连通性 |
| 页面提示"未安装内核" | 内核缺失或路径错误 | 重新执行第 1 步内核安装脚本，然后重启 v2rayA |

---

## 八、卸载（可选）

```bash
# 卸载 v2rayA
sudo apt remove v2raya -y

# 删除软件源
sudo rm /etc/apt/sources.list.d/v2raya.list

# 卸载内核
sudo systemctl stop v2ray
sudo systemctl disable v2ray
sudo rm /etc/systemd/system/v2ray.service
sudo rm /usr/local/bin/v2ray
sudo rm -rf /usr/local/etc/v2ray
```

---

## 参考资料

- v2rayA 官方文档：https://v2raya.org/docs/
- v2rayA GitHub：https://github.com/v2rayA/v2rayA
- V2Ray 官方文档：https://www.v2fly.org/
