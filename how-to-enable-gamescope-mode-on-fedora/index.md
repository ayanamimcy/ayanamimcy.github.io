# 如何在fedora系统中开启gamescope


## 安装 gamescope 启动到 Steam Deck 界面

### 前置条件
- Fedora Workstation（Wayland 默认即可）。
- 有 NVIDIA/AMD 独显的机器建议先装好官方驱动。
- 关闭 Secure Boot，避免驱动与 gamescope 依赖冲突。

### 快速安装
1. 安装 gamescope 与 HUD 工具：
   ```bash
   sudo dnf install -y gamescope mangohud
   ```
2. 使用现成脚本一键部署 session：
   ```bash
   git clone https://github.com/shahnawazshahin/steam-using-gamescope-guide.git
   cd steam-using-gamescope-guide
   sudo ./install.sh
   ```
   安装完成后在登录界面选择 `Gamescope+Steam`（名字可能略有差异），即可进入 Steam Deck 风格界面。

### 手动安装（可控性更高）
如果想自己掌握文件位置，可以从 ChimeraOS 的 session 仓库直接复制：
1. 下载仓库：
   ```bash
   git clone https://github.com/ChimeraOS/gamescope-session-steam
   git clone https://github.com/ChimeraOS/gamescope-session
   ```
2. 将仓库中的 `usr/` 内容覆盖到系统 `/usr/`，并刷新 gdm：
   ```bash
   sudo cp -r gamescope-session-steam/usr/* /usr/
   sudo cp -r gamescope-session/usr/* /usr/
   sudo systemctl daemon-reload
   ```
3. 重启后在 gdm 登录界面选择新的 gamescope 会话即可。

### 验证
- 登录 gamescope 会话后，终端运行 `echo $XDG_SESSION_TYPE` 应为 `wayland`，且 `gamescope-session-plus@steam.service` 应在运行。
- `mangohud steam` 可以查看帧率、功耗，确认 HUD 正常。

## Sunshine
目标：在锁屏、GNOME、gamescope 三种场景下都能串流，同时保证有声输出。

思路：
- 系统级 root 服务保证「即使没登录」也能看到画面。
- 登录后切换到用户级服务，拿到正确的 PipeWire/音频权限。

具体步骤：
1. 安装 Sunshine：`sudo dnf install sunshine`（或编译安装）。
2. 创建系统级服务 `sunshine-gdm.service`：

`sunshine-gdm.service`用于系统层级进行启动
```
[Unit]
Description=Self-hosted game stream host for Moonlight
StartLimitIntervalSec=500
StartLimitBurst=5
Requires=gdm.service
After=gdm.service

[Service]
User=root
Group=root
Environment="HOME=/home/fedora"
ExecStartPre=/bin/sleep 2
ExecStart=/usr/bin/sunshine
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
需要注意：`HOME` 要指向日常登录用户的主目录，避免客户端看到多个主机。

3. 配置用户级服务，启动时停掉系统级，退出时再拉起系统级：

基础服务 `sunshine.service`

```
[Unit]
Description=Sunshine is a self-hosted game stream host for Moonlight.
StartLimitIntervalSec=500 
StartLimitBurst=5

[Service]
ExecStartPre=/usr/bin/systemctl stop sunshine-gdm
ExecStartPre=/bin/sleep 2
ExecStopPost=/usr/bin/systemctl start sunshine-gdm
ExecStart=/usr/bin/sunshine
Restart=on-failure
RestartSec=5s

[Install]
```
这个服务只定义启动/停止逻辑，我们通过两个 hook 触发它，利用 `BindsTo` 和 `PartOf` 自动随会话启停。

`sunshine-on-gamescope.service`
```
[Unit]
Description=Start/Stop Sunshine with Gamescope session
BindsTo=gamescope-session-plus@steam.service
After=gamescope-session-plus@steam.service
PartOf=gamescope-session-plus@steam.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/systemctl --user start sunshine.service
ExecStop=/usr/bin/systemctl --user stop sunshine.service

[Install]
WantedBy=gamescope-session-plus@steam.service
```

`sunshine-on-gnome.service`
```
[Unit]
Description=Start/Stop Sunshine with GNOME session
BindsTo=gnome-session.target
After=gnome-session.target
PartOf=gnome-session.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/systemctl --user start sunshine.service
ExecStop=/usr/bin/systemctl --user stop sunshine.service

[Install]
WantedBy=gnome-session.target
```

4. 启用两个 hook（在用户 session 内执行）：
   ```bash
   systemctl --user enable --now sunshine-on-gnome.service
   systemctl --user enable --now sunshine-on-gamescope.service
   ```

5. 免密授权：用户执行 `systemctl` 默认会弹认证，写一条 polkit 规则跳过：

创建文件 `/etc/polkit-1/rules.d/sunshine-sddm.rules`
```
polkit.addRule(function(action, subject) {
  if (action.id == "org.freedesktop.systemd1.manage-units" &&
    action.lookup("unit") == "sunshine-sddm.service")
  {
    return polkit.Result.YES;
  }
})

```

附加：如果在 BazziteOS 下想直接串流 gamescope 会话，可以使用 `sunshine-kms.service` 并将其绑定到 gamescope：
```bash
systemctl --user add-wants gamescope-session-plus@steam.service sunshine-kms.service
```

## Nvidia 显卡
外接显卡偶尔开机未加载驱动，为避免黑屏或没有硬件加速，添加一个自检脚本，缺卡时自动重启一次。

`check_nvidia_reboot.sh`
```
#!/bin/bash

# 标记文件路径（用于防止无限重启）
FLAG_FILE="/var/tmp/nvidia_reboot_attempted"

# 等待几秒确保驱动模块有时间加载（可选，防止启动太快产生误报）
sleep 5

# 检查 nvidia-smi 是否能识别到显卡
# -L 参数列出所有 GPU，如果返回结果包含 UUID 则说明识别正常
if nvidia-smi -L | grep -q "UUID"; then
    echo "[SUCCESS] Nvidia GPU detected."
    
    # 如果之前有过重启尝试的标记，现在检测成功了，就删除标记
    if [ -f "$FLAG_FILE" ]; then
        rm -f "$FLAG_FILE"
        echo "Resetting reboot attempt flag."
    fi
    exit 0
else
    echo "[FAILURE] Nvidia GPU NOT detected!"

    # 检查是否已经尝试过重启
    if [ -f "$FLAG_FILE" ]; then
        echo "System already rebooted once to fix this. Preventing infinite loop."
        # 这里可以选择发送通知或者记录更详细的日志
        exit 1
    else
        echo "First failure detected. Initiating reboot..."
        # 创建标记文件
        touch "$FLAG_FILE"
        # 执行重启
        /usr/bin/systemctl reboot
    fi
fi
```

service
`check-nvidia-boot.service`
```
[Unit]
Description=Check Nvidia GPU presence and reboot once if missing
# 确保在图形界面或多用户环境就绪后运行
After=multi-user.target graphical.target systemd-modules-load.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/check_nvidia_reboot.sh
# 运行完即退出，不驻留后台
RemainAfterExit=no
# 以 root 权限运行以便执行 reboot
User=root

[Install]
WantedBy=multi-user.target
```

启用：
```bash
sudo cp check_nvidia_reboot.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/check_nvidia_reboot.sh
sudo cp check-nvidia-boot.service /etc/systemd/system/
sudo systemctl enable --now check-nvidia-boot.service
```

这样开机会先检查一次，若驱动未加载则自动重启一次，最多尝试一轮避免重启循环。

## 关于 mangohud 无法检测到 CPU 功耗
原因：普通用户没有读取 `energy_uj` 的权限，HUD 无法显示 CPU Package 功耗。

解决：在 `/etc/udev/rules.d/99-intel-rapl.rules` 追加：
```
# 允许所有用户读取 Intel RAPL 功耗数据
ENV{SUBSYSTEM}=="powercap", ACTION=="add|change", OWNER="root", PROGRAM+="/usr/bin/find /sys$env{DEVPATH} -name energy_uj -exec chmod g+r -R {} + -exec chown root:wheel {} +"
```

