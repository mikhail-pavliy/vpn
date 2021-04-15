# Мосты, туннели и VPN 

1. Между двумя виртуалками поднять vpn в режимах
- tun
- tap
Прочуствовать разницу.

2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку

Выполнение задания

Подготовим серверную и клиентские машины к работе, установив все необходимые пакеты и утилиты для поставленной задачи.
```ruby 
[root@server-ovpn vagrant]# yum install -y epel-release
....

Installed:
  epel-release.noarch 0:7-11

Complete!

[root@client-ovpn vagrant]# yum install -y epel-release

......
Installed:
  epel-release.noarch 0:7-11

Complete!
```
Далее установим ```openvpn```,```easy-rsa```, ```iperf3```:

```ruby
[root@server-ovpn vagrant]# yum install -y openvpn easy-rsa iperf3
[root@client-ovpn vagrant]# yum install -y openvpn iperf3
```
Добавим дополнительные IP-адреса на loopback-интерфейсы:

```ruby
[root@server-ovpn vagrant]# cat > /etc/sysconfig/network-scripts/ifcfg-lo.2 <<EOF
DEVICE=lo:2
IPADDR=10.10.0.1
PREFIX=32
NETWORK=10.10.0.1
ONBOOT=yes
EOF
```
```ruby
[root@client-ovpn vagrant]# cat > /etc/sysconfig/network-scripts/ifcfg-lo.2 <<EOF 
DEVICE=lo:2                                                                       
IPADDR=10.10.0.2                                                                  
PREFIX=32                                                                         
NETWORK=10.10.0.2                                                                 
ONBOOT=yes                                                                        
EOF
```
Так же включим ```forwarding``` пакетов между интерфейсами, после чего перезапустим ```network.service```

```ruby
[root@server-ovpn vagrant]# echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/ip_forwarding.conf
[root@server-ovpn vagrant]# cat /etc/sysctl.d/ip_forwarding.conf
net.ipv4.ip_forward = 1
[root@server-ovpn vagrant]# systemctl restart network
```
```ruby
[root@client-ovpn vagrant]# echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/ip_forwarding.conf
[root@client-ovpn vagrant]# cat /etc/sysctl.d/ip_forwarding.conf
net.ipv4.ip_forward = 1
[root@client-ovpn vagrant]# systemctl restart network
```
Сгенерируем секретный ключ:
```ruby
[root@server-ovpn vagrant]# openvpn --genkey --secret /etc/openvpn/static.key
```
Создадим конфиг файл ```server-ovpn``` для использования ```tap``` режима:

```ruby
[root@server-ovpn vagrant]# cat > /etc/openvpn/server.conf <<EOF
> dev tap
> ifconfig 172.20.0.1 255.255.255.0
> topology subnet
> route 10.10.0.2 255.255.255.255 172.20.0.2
> secret /etc/openvpn/static.key
> compress lzo
> status /var/log/openvpn-status.log
> log /var/log/openvpn.log
> verb 3
> EOF
```
```ruby
[root@server-ovpn vagrant]# systemctl enable --now openvpn@server
[root@server-ovpn vagrant]# systemctl status openvpn@server
● openvpn@server.service - OpenVPN Robust And Highly Flexible Tunneling Application On server
   Loaded: loaded (/usr/lib/systemd/system/openvpn@.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2021-04-15 09:20:01 UTC; 11s ago
 Main PID: 22884 (openvpn)
   Status: "Pre-connection initialization successful"
   CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
           └─22884 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf

Apr 15 09:20:01 server-ovpn systemd[1]: Starting OpenVPN Robust And Highly Flexible Tunneling Application On server...
Apr 15 09:20:01 server-ovpn systemd[1]: Started OpenVPN Robust And Highly Flexible Tunneling Application On server.
```

Настроим клиентскую машину:
В файл ```/etc/openvpn/static.key``` необходимо скопировать секретный ключ с сервера:

```ruby
[root@client-ovpn vagrant]# cat > /etc/openvpn/static.key <<EOF
> #
> # 2048 bit OpenVPN static key
> #
> -----BEGIN OpenVPN Static key V1-----
> 4ed5db425b2e98d0440110774b90a117
> 7361ee3eb3a115d86b01fb85fabe5d55
> c4804e1f918da9e135943886314feaa6
> 1f9d4e80434bf7d9a51eac0f128f5533
> b800eb4ef2ae031bc394c5aa2bc18a44
> 6d748188fe71bd1a17bbc2e1dbc8be8a
> 350b58619427944c062039d547709e83
> 2679abe875093a75ea5acdc777693def
> de8b167c7d01d7c3c4604a35d9aaa135
> 53706452c9fc80c08067706c7dc38429
> 1454ed2c2300d83341aab9642c2af0fc
> 1d7a5454ce3f099db482e6bf7bc7f114
> 822ab131023e61626d736f9a27334094
> 064b2d0c511805287324afd24b27fb28
> 0edb003ef54545c5b74be69a21da5a3b
> 94726b7ea3716ab68d2476a782637021
> -----END OpenVPN Static key V1-----
> EOF
```
Также cоздадим конфиг файл  ```client-ovpn``` для использования ```tap``` режима:

```ruby
[root@client-ovpn vagrant]# cat > /etc/openvpn/server.conf <<EOF
> dev tap
> remote 10.10.10.10
> ifconfig 172.20.0.2 255.255.255.0
> topology subnet
> route 10.10.0.1 255.255.255.255 172.20.0.1
> secret /etc/openvpn/static.key
> compress lzo
> status /var/log/openvpn-status.log
> log /var/log/openvpn.log
> verb 3
> EOF
```
```ruby
[root@client-ovpn vagrant]# systemctl enable --now openvpn@server
Created symlink from /etc/systemd/system/multi-user.target.wants/openvpn@server.service to /usr/lib/systemd/system/openvpn@.service.
[root@client-ovpn vagrant]# systemctl status openvpn@server
● openvpn@server.service - OpenVPN Robust And Highly Flexible Tunneling Application On server
   Loaded: loaded (/usr/lib/systemd/system/openvpn@.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2021-04-15 09:28:32 UTC; 9s ago
 Main PID: 22240 (openvpn)
   Status: "Pre-connection initialization successful"
   CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
           └─22240 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf

Apr 15 09:28:32 client-ovpn systemd[1]: Starting OpenVPN Robust And Highly Flexible Tunneling Application On server...
Apr 15 09:28:32 client-ovpn systemd[1]: Started OpenVPN Robust And Highly Flexible Tunneling Application On server.
```
Проверим наличие маршрутов и доступность
```ruby
[root@server-ovpn vagrant]# ip r
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
10.10.0.2 via 172.20.0.2 dev tap0
10.10.10.0/24 dev eth1 proto kernel scope link src 10.10.10.10 metric 101
172.20.0.0/24 dev tap0 proto kernel scope link src 172.20.0.1

[root@server-ovpn vagrant]# ping -I 10.10.0.1 10.10.0.2
PING 10.10.0.2 (10.10.0.2) from 10.10.0.1 : 56(84) bytes of data.
64 bytes from 10.10.0.2: icmp_seq=1 ttl=64 time=0.643 ms
64 bytes from 10.10.0.2: icmp_seq=2 ttl=64 time=0.556 ms
```

```ruby
[root@client-ovpn vagrant]# ip r
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
10.10.0.1 via 172.20.0.1 dev tap0
10.10.10.0/24 dev eth1 proto kernel scope link src 10.10.10.20 metric 101
172.20.0.0/24 dev tap0 proto kernel scope link src 172.20.0.2

[root@client-ovpn vagrant]# ping -I 10.10.0.2 10.10.0.1
PING 10.10.0.1 (10.10.0.1) from 10.10.0.2 : 56(84) bytes of data.
64 bytes from 10.10.0.1: icmp_seq=1 ttl=64 time=0.647 ms
```
Проверим скорость трафика с помощью ```iperf```

```ruby
[root@server-ovpn vagrant]# iperf3 -s
[root@client-ovpn vagrant]# iperf3 -c 172.20.0.1 -t 40 -i 5
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-40.00  sec   741 MBytes   155 Mbits/sec   33             sender
[  4]   0.00-40.00  sec   739 MBytes   155 Mbits/sec                  receiver
```

Изменим конфиги ```server.conf``` на сервере и клиенте  с ```tap``` на ```tun```:
```ruby
[root@server-ovpn vagrant]# cat /etc/openvpn/server.conf
dev tun
ifconfig 172.20.0.1 255.255.255.0
topology subnet
route 10.10.0.2 255.255.255.255 172.20.0.2
secret /etc/openvpn/static.key
compress lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3

[root@client-ovpn vagrant]# cat /etc/openvpn/server.conf
dev tun
remote 10.10.10.10
ifconfig 172.20.0.2 255.255.255.0
topology subnet
route 10.10.0.1 255.255.255.255 172.20.0.1
secret /etc/openvpn/static.key
compress lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3

```
Перезапустим сервис openvpn и измерим скорость снова.
```ruby
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-40.01  sec   736 MBytes   154 Mbits/sec   38             sender
[  4]   0.00-40.01  sec   735 MBytes   154 Mbits/sec                  receiver
```
# Вывод 
В тестовой среде результаты в режиме tun чуть лучше. В связи с этим, я думаю, что в живой среде тоже будет лучше использовать режим tun. А режим tap лучше использовать в специфических задачах, например, если в архитектуре сети необходимо достичь связности  к примеру по L2.








