# IX入口常见问题

如果遇到断流等现象，以root身份执行下列命令（注意：重启后会失效）

`ip link set mtu 1416 dev eth0`

