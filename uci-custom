#!/bin/sh

# 修改LAN接口的默认IP地址
uci set network.lan.ipaddr='10.0.0.1'

# 添加eth2和eth3到device配置
uci add_list network.@device[0].ports='eth2'
uci add_list network.@device[0].ports='eth3'

# 设置root密码为password
echo -e "password\npassword" | passwd root

# 应用更改
uci commit network
