# Оптимальный порядок выполнения — ДЭ 09.02.06

---

## 🖥️ УСТРОЙСТВО 1: ISP — всё за один заход
> п.1 (hostname) → п.3 (интернет) → п.11 (timezone)

### Имя устройства
```bash
hostnamectl set-hostname isp; exec bash
vim /etc/sysconfig/network
HOSTNAME=isp
```

### Интернет через ISP
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
net.ipv4.ip_forward = 1
systemctl restart network
sysctl -a | grep ip_forward
apt-get update && apt-get install -y iptables
iptables -t nat -A POSTROUTING -s 192.168.11.0/28 -o ens18 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.63.0/28 -o ens18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
iptables -t nat -L -n -v
```

### Часовой пояс
```bash
timedatectl set-timezone Europe/Moscow
timedatectl
```

---

## 🖥️ УСТРОЙСТВО 2: Гипервизор — коммутация HQ
> п.5 (VLAN)

```
HQ-SRV = VLAN 150
HQ-CLI = VLAN 250
управление = VLAN 888
bridge VLAN aware = включить
```

---

## 🖥️ УСТРОЙСТВО 3: HQ-RTR — всё за один заход
> п.1 → п.2 → п.3 → п.4 → п.7 → п.8 → п.9 → п.11

### Имя устройства
```
en
conf t
hostname hq-rtr
ip domain-name au-team.irpo
wr mem
```

### IPv4 — интерфейсы VLAN
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

### Интернет — интерфейс ISP
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

### Учётная запись
```
conf t
username net_artem
password P@ssw0rd
role admin
exit
wr mem
```

### GRE туннель
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

### OSPF
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

### NAT
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

### Часовой пояс
```
conf t
ntp timezone utc+3
wr mem
```
**Проверка:**
```
show ntp timezone
```

---

## 🖥️ УСТРОЙСТВО 4: BR-RTR — всё за один заход
> п.1 → п.2 → п.3 → п.4 → п.7 → п.8 → п.9 → п.11

### Имя устройства
```
en
conf t
hostname br-rtr
ip domain-name au-team.irpo
wr mem
```

### IPv4
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

### Интернет — интерфейс ISP
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

### Учётная запись
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

### GRE туннель
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

### OSPF
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

### NAT
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

### Часовой пояс
```
conf t
ntp timezone utc+3
wr mem
```
**Проверка:**
```
show ntp timezone
```

---

## 🖥️ УСТРОЙСТВО 5: HQ-SRV — всё за один заход
> п.1 → п.2 → п.4 → п.5 → п.6 → п.8 → п.9 → п.11

### Имя устройства
```bash
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
vim /etc/sysconfig/network
HOSTNAME=hq-srv.au-team.irpo
hostname -f
```

### IPv4
```bash
echo "192.168.100.2/28" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens18/ipv4route
echo "nameserver 77.88.8.8" > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network
ip -c a
ip -c r
```

### Учётная запись
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

### Проверка коммутации
```bash
ip -c r
ping -c3 192.168.100.1
```

### SSH
```bash
vim /etc/openssh/sshd_config
Port 2026
AllowUsers andrew
MaxAuthTries 6
Banner /etc/openssh/banner
echo "Andrew access only" > /etc/openssh/banner
systemctl restart sshd
```
**Проверка:**
```bash
ssh -p 2026 andrew@localhost
ssh andrew@localhost
```

### Проверка OSPF (связность с BR-SRV)
```bash
ping -c3 192.168.0.2
```

### Проверка NAT (интернет)
```bash
ping -c3 77.88.8.8
```

### Часовой пояс
```bash
timedatectl set-timezone Europe/Moscow
timedatectl
```

---

## 🖥️ УСТРОЙСТВО 6: BR-SRV — всё за один заход
> п.1 → п.2 → п.4 → п.6 → п.8 → п.9 → п.11

### Имя устройства
```bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
vim /etc/sysconfig/network
HOSTNAME=br-srv.au-team.irpo
hostname -f
```

### IPv4
```bash
echo "192.168.0.2/25" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.0.1" > /etc/net/ifaces/ens18/ipv4route
echo "nameserver 77.88.8.8" > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network
ip -c a
ip -c r
```

### Учётная запись
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

### SSH
```bash
vim /etc/openssh/sshd_config
Port 2026
AllowUsers andrew
MaxAuthTries 6
Banner /etc/openssh/banner
echo "Andrew access only" > /etc/openssh/banner
systemctl restart sshd
```
**Проверка:**
```bash
ssh -p 2026 andrew@localhost
ssh andrew@localhost
```

### Проверка OSPF (связность с HQ-SRV)
```bash
ping -c3 192