# 1.安装必要包
```bash
yum install xorg-x11-xauth xorg-x11-fonts-* xorg-x11-utils xclock -y
```

# 2.确保x11-forwarding打开
编辑`/etc/ssh/sshd_config`确保X11Forwarding yes