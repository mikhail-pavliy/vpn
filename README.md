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
