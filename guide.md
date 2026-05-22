# Полное решение — Сетевое и системное администрирование

---

## 1. Имена устройств

### ISP
```bash
hostnamectl set-hostname isp; exec bash
vim /etc/sysconfig/network
# Добавить строку:
HOSTNAME=isp
```

### HQ-RTR
```
en
conf t
hostname hq-rtr
ip domain-name au-team.irpo
wr mem
```

### BR-RTR
```
en
conf t
hostname br-rtr
ip domain-name au-team.irpo
wr mem
```

### HQ-SRV
```bash
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
vim /etc/sysconfig/network
# Добавить строку:
HOSTNAME=hq-srv.au-team.irpo
hostname -f
```

### BR-SRV
```bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
vim /etc/sysconfig/network
# Добавить строку:
HOSTNAME=br-srv.au-team.irpo
hostname -f
```

### HQ-CLI
```
ЦУС → задать имя: hq-cli.au-team.irpo
```

---

## 2. IPv4

### HQ-RTR
```
show port brief
conf t
interface vl150
 description "VLAN 150"
 ip address 192.168.100.1/28
exit
interface vl250
 description "VLAN 250"
 ip address 192.168.200.1/26
exit
interface vl888
 description "VLAN 888"
 ip address 192.168.99.1/29
exit
wr mem
show ip interface brief

port te1
service-instance te1/vl150
 encapsulation dot1q 150 exact
 rewrite pop 1
 connect ip interface vl150
exit
service-instance te1/vl250
 encapsulation dot1q 250 exact
 rewrite pop 1
 connect ip interface vl250
exit
service-instance te1/vl888
 encapsulation dot1q 888 exact
 rewrite pop 1
 connect ip interface vl888
exit
exit
wr mem
show ip interface brief
```

> ⚠️ Исправлено: убрана пустая строка между `encapsulation dot1q 888 exact` и `rewrite pop 1`

### BR-RTR
```
conf t
interface int1
 description "BR-Net"
 ip address 192.168.0.1/25
exit
port te1
service-instance te1/int1
 encapsulation untagged
 connect ip interface int1
exit
exit
wr mem
show ip interface brief
```

### HQ-SRV
```bash
echo "192.168.100.2/28" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens18/ipv4route
echo "nameserver 77.88.8.8" > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network
ip -c a
ip -c r
```

### BR-SRV
```bash
echo "192.168.0.2/25" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.0.1" > /etc/net/ifaces/ens18/ipv4route
echo "nameserver 77.88.8.8" > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network
ip -c a
ip -c r
```

---

## 3. Интернет через ISP

### ISP
```bash
mkdir -p /etc/net/ifaces/ens18
mkdir -p /etc/net/ifaces/ens19
mkdir -p /etc/net/ifaces/ens20

echo "TYPE=eth" > /etc/net/ifaces/ens18/options
echo "BOOTPROTO=dhcp" >> /etc/net/ifaces/ens18/options

echo "TYPE=eth" > /etc/net/ifaces/ens19/options
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens19/options
echo "192.168.11.1/28" > /etc/net/ifaces/ens19/ipv4address

echo "TYPE=eth" > /etc/net/ifaces/ens20/options
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens20/options
echo "172.16.63.1/28" > /etc/net/ifaces/ens20/ipv4address

systemctl restart network

vim /etc/net/sysctl.conf
# Добавить строку:
# net.ipv4.ip_forward = 1

systemctl restart network
sysctl -a | grep ip_forward

apt-get update && apt-get install -y iptables
iptables -t nat -A POSTROUTING -s 192.168.11.0/28 -o ens18 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.63.0/28 -o ens18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
iptables -t nat -L -n -v
```

### HQ-RTR
```
en
conf t
interface isp
 description "ISP"
 ip address 192.168.11.2/28
exit
ip route 0.0.0.0/0 192.168.11.1
port te0
service-instance te0/isp
 encapsulation untagged
 connect ip interface isp
exit
exit
wr mem
```
**Проверка:**
```
show ip interface brief
show ip route
ping 192.168.11.1
ping 77.88.8.8
```

### BR-RTR
```
en
conf t
interface isp
 description "ISP"
 ip address 172.16.63.2/28
exit
ip route 0.0.0.0/0 172.16.63.1
port te0
service-instance te0/isp
 encapsulation untagged
 connect ip interface isp
exit
exit
wr mem
```
**Проверка:**
```
show ip interface brief
show ip route
ping 172.16.63.1
ping 77.88.8.8
```

---

## 4. Учётные записи

### HQ-SRV / BR-SRV
```bash
useradd andrew -u 2026
id andrew
passwd andrew
usermod -aG wheel andrew
echo "andrew ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers
```
**Проверка:**
```bash
su - andrew
sudo -i
whoami
```

### HQ-RTR
```
conf t
username net_artem
 password P@ssw0rd
 role admin
exit
wr mem
```

### BR-RTR
```
conf t
username net_artem
 password P@ssw0rd
 role admin
exit
wr mem
```
**Проверка:**
```
show users localdb
```

---

## 5. Коммутация HQ

### Гипервизор
```
HQ-SRV  → VLAN 150
HQ-CLI  → VLAN 250
Управление → VLAN 888
bridge VLAN aware = включить
```

### HQ-SRV
```bash
ip -c r
ping -c3 192.168.100.1
```

---

## 6. SSH

### HQ-SRV / BR-SRV
```bash
vim /etc/ssh/sshd_config
# Изменить/добавить строки:
# Port 2026
# AllowUsers andrew
# MaxAuthTries 6
# Banner /etc/ssh/banner

echo "Andrew access only" > /etc/ssh/banner
systemctl restart sshd
```

> ⚠️ Исправлено: путь `/etc/openssh/sshd_config` → `/etc/ssh/sshd_config`

**Проверка:**
```bash
ssh -p 2026 andrew@localhost
```

---

## 7. GRE

### HQ-RTR
```
en
conf t
interface tunnel.0
 description "GRE"
 ip address 10.10.10.1/30
 ip tunnel 192.168.11.2 172.16.63.2 mode gre
exit
wr mem
```
**Проверка:**
```
show interface tunnel.0
ping 10.10.10.2
```

### BR-RTR
```
en
conf t
interface tunnel.0
 description "GRE"
 ip address 10.10.10.2/30
 ip tunnel 172.16.63.2 192.168.11.2 mode gre
exit
wr mem
```
**Проверка:**
```
show interface tunnel.0
ping 10.10.10.1
```

---

## 8. OSPF

### HQ-RTR
```
en
conf t
router ospf 1
 ospf router-id 10.10.10.1
 passive-interface default
 no passive-interface tunnel.0
 network 10.10.10.0/30 area 0
 network 192.168.100.0/28 area 0
 network 192.168.200.0/26 area 0
 network 192.168.99.0/29 area 0
exit
interface tunnel.0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit
wr mem
```
**Проверка:**
```
show ip ospf interface
show ip ospf neighbor
show ip route
```

### BR-RTR
```
en
conf t
router ospf 1
 ospf router-id 10.10.10.2
 passive-interface default
 no passive-interface tunnel.0
 network 192.168.0.0/25 area 0
 network 10.10.10.0/30 area 0
exit
interface tunnel.0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit
wr mem
```
**Проверка:**
```
show ip ospf interface
show ip ospf neighbor
show ip route
```

### Проверка связности
```bash
# HQ-SRV:
ping -c3 192.168.0.2

# BR-SRV:
ping -c3 192.168.100.2
```

---

## 9. NAT

### HQ-RTR
```
en
conf t
interface isp
 ip nat outside
exit
interface vl150
 ip nat inside
exit
interface vl250
 ip nat inside
exit
interface vl888
 ip nat inside
exit
ip nat pool VLAN150 192.168.100.1-192.168.100.14
ip nat pool VLAN250 192.168.200.1-192.168.200.62
ip nat pool VLAN888 192.168.99.1-192.168.99.6
ip nat source dynamic inside-to-outside pool VLAN150 overload interface isp
ip nat source dynamic inside-to-outside pool VLAN250 overload interface isp
ip nat source dynamic inside-to-outside pool VLAN888 overload interface isp
wr mem
```
**Проверка:**
```
show ip nat translations
```
```bash
# HQ-SRV:
ping -c3 77.88.8.8
```

### BR-RTR
```
en
conf t
interface isp
 ip nat outside
exit
interface int1
 ip nat inside
exit
ip nat pool BR-Net 192.168.0.1-192.168.0.126
ip nat source dynamic inside-to-outside pool BR-Net overload interface isp
wr mem
```
**Проверка:**
```
show ip nat translations
```
```bash
# BR-SRV:
ping -c3 77.88.8.8
```

---

## 10. DNS — пропуск

---

## 11. Часовой пояс

### HQ-RTR / BR-RTR
```
conf t
ntp timezone utc+3
wr mem
```
**Проверка:**
```
show ntp timezone
```

### ISP / HQ-SRV / BR-SRV / HQ-CLI
```bash
timedatectl set-timezone Europe/Moscow
timedatectl
```