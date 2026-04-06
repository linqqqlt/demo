# Полное решение ДЭ — 09.02.06 Сетевое и системное администрирование

---

## МОДУЛЬ 1. Настройка сетевой инфраструктуры

---

### Таблица адресации (Таблица 2)

| Имя устройства | Интерфейс | IP-адрес | Маска/Префикс | Шлюз по умолчанию |
|---|---|---|---|---|
| ISP | ens18 (Internet) | DHCP | — | DHCP |
| ISP | ens19 (к HQ-RTR) | 172.16.1.1/28 | 255.255.255.240 | — |
| ISP | ens20 (к BR-RTR) | 172.16.2.1/28 | 255.255.255.240 | — |
| HQ-RTR | te0 (к ISP) | 172.16.1.2/28 | 255.255.255.240 | 172.16.1.1 |
| HQ-RTR | te1.100 (к HQ-SRV) | 192.168.100.1/27 | 255.255.255.224 | — |
| HQ-RTR | te1.200 (к HQ-CLI) | 192.168.200.1/24 | 255.255.255.0 | — |
| HQ-RTR | te1.999 (управление) | 192.168.99.1/29 | 255.255.255.248 | — |
| BR-RTR | te0 (к ISP) | 172.16.2.2/28 | 255.255.255.240 | 172.16.2.1 |
| BR-RTR | te1 (к BR-SRV) | 192.168.0.1/28 | 255.255.255.240 | — |
| HQ-SRV | ens18 (VLAN 100) | 192.168.100.2/27 | 255.255.255.224 | 192.168.100.1 |
| HQ-CLI | ens18 (VLAN 200) | DHCP | — | 192.168.200.1 |
| BR-SRV | ens18 | 192.168.0.2/28 | 255.255.255.240 | 192.168.0.1 |

**Расчёт подсетей:**
- VLAN 100 (HQ-SRV): не более 32 адресов → /27 (32 адреса, 30 хостов)
- VLAN 200 (HQ-CLI): не менее 16 адресов → /24 (256 адресов) или /27 (30 хостов)
- VLAN 999 (управление): не более 8 адресов → /29 (8 адресов, 6 хостов)
- BR-SRV: не более 16 адресов → /28 (16 адресов, 14 хостов)

---

### 1. Базовая настройка устройств

#### 1.1 Настройка имён хостов

**ISP, HQ-SRV, HQ-CLI, BR-SRV** (Alt Linux):
```bash
hostnamectl set-hostname <имя_устройства>.au-team.irpo
# Также записать в /etc/hostname:
echo "isp.au-team.irpo" > /etc/hostname   # для ISP
echo "hq-srv.au-team.irpo" > /etc/hostname  # для HQ-SRV
echo "hq-cli.au-team.irpo" > /etc/hostname  # для HQ-CLI
echo "br-srv.au-team.irpo" > /etc/hostname  # для BR-SRV
```

**HQ-RTR и BR-RTR** (EcoRouter):
```
en
conf t
hostname HQ-RTR
# или
hostname BR-RTR
```

#### 1.2 Настройка IPv4 на Alt Linux машинах

**ISP — интерфейс в интернет (ens18, DHCP):**
```bash
mkdir -p /etc/net/ifaces/ens18
cat > /etc/net/ifaces/ens18/options << EOF
TYPE=eth
BOOTPROTO=dhcp
NM_CONTROLLED=no
EOF
```

**ISP — интерфейс к HQ-RTR (ens19):**
```bash
mkdir -p /etc/net/ifaces/ens19
cat > /etc/net/ifaces/ens19/options << EOF
TYPE=eth
BOOTPROTO=static
NM_CONTROLLED=no
EOF

echo "172.16.1.1/28" > /etc/net/ifaces/ens19/ipv4address
```

**ISP — интерфейс к BR-RTR (ens20):**
```bash
mkdir -p /etc/net/ifaces/ens20
cat > /etc/net/ifaces/ens20/options << EOF
TYPE=eth
BOOTPROTO=static
NM_CONTROLLED=no
EOF

echo "172.16.2.1/28" > /etc/net/ifaces/ens20/ipv4address
```

**HQ-SRV (ens18, VLAN 100):**
```bash
mkdir -p /etc/net/ifaces/ens18
cat > /etc/net/ifaces/ens18/options << EOF
TYPE=eth
BOOTPROTO=static
NM_CONTROLLED=no
EOF

echo "192.168.100.2/27" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens18/ipv4route
```

**BR-SRV (ens18):**
```bash
mkdir -p /etc/net/ifaces/ens18
cat > /etc/net/ifaces/ens18/options << EOF
TYPE=eth
BOOTPROTO=static
NM_CONTROLLED=no
EOF

echo "192.168.0.2/28" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.0.1" > /etc/net/ifaces/ens18/ipv4route
```

**HQ-CLI (ens18, DHCP):**
```bash
mkdir -p /etc/net/ifaces/ens18
cat > /etc/net/ifaces/ens18/options << EOF
TYPE=eth
BOOTPROTO=dhcp
NM_CONTROLLED=no
EOF
```

Перезапуск сети на всех Alt Linux:
```bash
systemctl restart network
```

#### 1.3 Настройка HQ-RTR (EcoRouter)

```
en
conf t

port te0
service-instance to-isp
encapsulation untagged
exit
exit

port te1
service-instance to-srv
encapsulation dot1q 100 exact
rewrite pop 1
exit
service-instance to-cli
encapsulation dot1q 200 exact
rewrite pop 1
exit
service-instance vl999
encapsulation dot1q 999 exact
rewrite pop 1
exit
exit

interface to-isp
connect port te0 service-instance to-isp
ip address 172.16.1.2/28
no shutdown
exit

interface to-srv
connect port te1 service-instance to-srv
ip address 192.168.100.1/27
no shutdown
exit

interface to-cli
connect port te1 service-instance to-cli
ip address 192.168.200.1/24
no shutdown
exit

interface vl999
connect port te1 service-instance vl999
ip address 192.168.99.1/29
no shutdown
exit

ip route 0.0.0.0/0 172.16.1.1
end
wr
```

#### 1.4 Настройка BR-RTR (EcoRouter)

```
en
conf t

port te0
service-instance to-isp
encapsulation untagged
exit
exit

port te1
service-instance to-srv
encapsulation untagged
exit
exit

interface to-isp
connect port te0 service-instance to-isp
ip address 172.16.2.2/28
no shutdown
exit

interface to-srv
connect port te1 service-instance to-srv
ip address 192.168.0.1/28
no shutdown
exit

ip route 0.0.0.0/0 172.16.2.1
end
wr
```

---

### 2. Настройка ISP — доступ в интернет

```bash
# Включить IP forwarding
echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
sysctl -p

# NAT (маскарадинг)
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```

---

### 3. Создание учётных записей

**На HQ-SRV и BR-SRV:**
```bash
adduser sshuser
echo "sshuser:P@ssw0rd" | chpasswd
usermod -u 2026 sshuser
usermod -aG wheel sshuser
```

Настройка sudo без пароля:
```bash
visudo
# Раскомментировать/добавить строку:
%wheel ALL=(ALL) NOPASSWD: ALL
```

**На HQ-RTR и BR-RTR (EcoRouter):**
```
en
conf t
username net_admin
password P@ssw0rd
role admin
exit
end
wr
```

---

### 4. Настройка коммутации (VLAN) в сегменте HQ

Коммутация уже реализована в пункте 1.3 через настройку service-instance на HQ-RTR:
- VLAN 100 → HQ-SRV (service-instance to-srv, encapsulation dot1q 100)
- VLAN 200 → HQ-CLI (service-instance to-cli, encapsulation dot1q 200)
- VLAN 999 → управление (service-instance vl999, encapsulation dot1q 999)

Всё реализовано через один физический порт te1 (router-on-a-stick).

Если используется HQ-SW (виртуальный коммутатор или управляемый свитч), настроить транк на порт к HQ-RTR и access-порты для HQ-SRV (VLAN 100) и HQ-CLI (VLAN 200).

---

### 5. Настройка SSH

**На HQ-SRV и BR-SRV** — файл `/etc/openssh/sshd_config`:
```bash
cat >> /etc/openssh/sshd_config << EOF
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/banner
EOF
```

Создание баннера:
```bash
echo "Authorized access only" > /etc/openssh/banner
```

Перезапуск:
```bash
systemctl enable --now sshd
systemctl restart sshd
```

---

### 6. Настройка GRE-туннеля

**HQ-RTR:**
```
en
conf t
interface tunnel.1
ip address 10.0.0.1/30
ip tunnel 172.16.1.2 172.16.2.2 mode gre
no shutdown
exit
end
wr
```

**BR-RTR:**
```
en
conf t
interface tunnel.1
ip address 10.0.0.2/30
ip tunnel 172.16.2.2 172.16.1.2 mode gre
no shutdown
exit
end
wr
```

---

### 7. Настройка OSPF

**HQ-RTR:**
```
en
conf t
router ospf 1
network 192.168.100.0/27 area 0
network 192.168.200.0/24 area 0
network 192.168.99.0/29 area 0
network 10.0.0.0/30 area 0
passive-interface to-srv
passive-interface to-cli
passive-interface vl999
passive-interface to-isp
exit

interface tunnel.1
ip ospf authentication
ip ospf authentication-key P@ssw0rd
exit
end
wr
```

**BR-RTR:**
```
en
conf t
router ospf 1
network 192.168.0.0/28 area 0
network 10.0.0.0/30 area 0
passive-interface to-srv
passive-interface to-isp
exit

interface tunnel.1
ip ospf authentication
ip ospf authentication-key P@ssw0rd
exit
end
wr
```

> **Пояснение:** passive-interface на всех интерфейсах, кроме tunnel.1 — маршрутизаторы обмениваются маршрутами только через туннель, что соответствует требованию задания.

---

### 8. Настройка NAT на маршрутизаторах

**HQ-RTR:**
```
en
conf t
ip nat pool NAT 192.168.100.1-192.168.100.30,192.168.200.1-192.168.200.254
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

interface to-isp
ip nat outside
exit

interface to-cli
ip nat inside
exit

interface to-srv
ip nat inside
exit

interface vl999
ip nat inside
exit
end
wr
```

**BR-RTR:**
```
en
conf t
ip nat pool NAT 192.168.0.1-192.168.0.14
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

interface to-isp
ip nat outside
exit

interface to-srv
ip nat inside
exit
end
wr
```

---

### 9. Настройка DHCP для HQ-CLI

**На HQ-RTR:**
```
en
conf t
ip pool DHCP 192.168.200.2-192.168.200.254

dhcp-server 1
pool DHCP 1
mask 24
gateway 192.168.200.1
dns 192.168.100.2
domain-name au-team.irpo
domain-search au-team.irpo
exit

interface to-cli
dhcp-server 1
exit
end
wr
```

> **Примечание:** Исключён адрес 192.168.200.1 (маршрутизатор) — пул начинается с .2.

---

### 10. Настройка DNS на HQ-SRV

#### Установка BIND:
```bash
apt-get install bind9 bind-utils -y
```

#### Настройка зон — файл `/etc/bind/rfc1912.conf`:
Добавить в конец файла:
```
zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
    allow-update { none; };
};

zone "100.168.192.in-addr.arpa" {
    type master;
    file "100.168.192.in-addr.arpa";
    allow-update { none; };
};

zone "200.168.192.in-addr.arpa" {
    type master;
    file "200.168.192.in-addr.arpa";
    allow-update { none; };
};
```

#### Создание файлов зон:
```bash
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/au-team.irpo
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/100.168.192.in-addr.arpa
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/200.168.192.in-addr.arpa
```

#### Файл прямой зоны `/var/lib/bind/etc/zone/au-team.irpo`:
```
$TTL 86400
@       IN  SOA au-team.irpo. root.au-team.irpo. (
            2023010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS   au-team.irpo.
        IN  A    192.168.100.2
hq-rtr  IN  A    192.168.100.1
hq-rtr  IN  A    192.168.200.1
hq-rtr  IN  A    192.168.99.1
hq-srv  IN  A    192.168.100.2
hq-cli  IN  A    192.168.200.2
br-rtr  IN  A    192.168.0.1
br-srv  IN  A    192.168.0.2
docker  IN  A    172.16.1.1
web     IN  A    172.16.2.1
```

#### Файл обратной зоны `/var/lib/bind/etc/zone/100.168.192.in-addr.arpa`:
```
$TTL 86400
@       IN  SOA au-team.irpo. root.au-team.irpo. (
            2023010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS   au-team.irpo.
1       IN  PTR  hq-rtr.au-team.irpo.
2       IN  PTR  hq-srv.au-team.irpo.
```

#### Файл обратной зоны `/var/lib/bind/etc/zone/200.168.192.in-addr.arpa`:
```
$TTL 86400
@       IN  SOA au-team.irpo. root.au-team.irpo. (
            2023010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS   au-team.irpo.
1       IN  PTR  hq-rtr.au-team.irpo.
2       IN  PTR  hq-cli.au-team.irpo.
```

#### Настройка DNS-пересылки — `/etc/bind/options.conf`:
В блок `options` добавить:
```
forwarders {
    77.88.8.8;
    77.88.8.1;
};
```

Также в `options` убедиться:
```
listen-on { any; };
allow-query { any; };
```

#### Генерация ключа и запуск:
```bash
rndc-confgen > /var/lib/bind/etc/rndc.key
sed -i '6,$d' /var/lib/bind/etc/rndc.key

systemctl enable --now bind
systemctl restart bind
```

#### Настройка resolv.conf на HQ-SRV:
```bash
cat > /etc/resolv.conf << EOF
search au-team.irpo
nameserver 192.168.100.2
EOF
```

> На остальных машинах также указать `nameserver 192.168.100.2` в `/etc/resolv.conf`.

---

### 11. Настройка часового пояса

**На Alt Linux машинах:**
```bash
apt-get install tzdata -y
timedatectl set-timezone <зона>
# Например:
timedatectl set-timezone Europe/Moscow
```

**На EcoRouter (HQ-RTR, BR-RTR):**
```
en
conf t
ntp timezone utc+3
end
wr
```

---

---

## МОДУЛЬ 2. Организация сетевого администрирования

> **Важно:** В модуле 2 используется отдельный стенд, где уже преднастроены IP-адреса, NAT, туннель, OSPF, SSH, DHCP, DNS, пользователи. HQ-SRV имеет 3 дополнительных диска по 1 ГБ (но в задании для RAID используются 2 диска).

---

### 1. Контроллер домена Samba DC на BR-SRV

#### Установка:
```bash
apt-get install task-samba-dc -y
```

#### Подготовка:
```bash
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba /var/cache/samba
mkdir -p /var/lib/samba/sysvol
```

#### Провизионирование домена:
```bash
samba-tool domain provision
```
Параметры:
- **Realm:** AU-TEAM.IRPO
- **Domain:** AU-TEAM
- **Server Role:** dc
- **DNS backend:** SAMBA_INTERNAL
- **DNS forwarder:** 77.88.8.8

```bash
systemctl enable --now samba
```

#### Создание пользователей:
```bash
for i in 1 2 3 4 5; do
    samba-tool user create hquser${i} P@ssw0rd
    samba-tool user setexpiry hquser${i} --noexpiry
done
```

#### Создание группы и добавление пользователей:
```bash
samba-tool group add hq
for i in 1 2 3 4 5; do
    samba-tool group addmembers hq hquser${i}
done
```

#### Ввод HQ-CLI в домен:
На HQ-CLI:
```bash
apt-get install task-auth-ad-sssd -y
# Через интерфейс system-auth или:
system-auth write ad au-team.irpo AU-TEAM BR-SRV.au-team.irpo administrator P@ssw0rd
```
Или ручной ввод:
```bash
realm join au-team.irpo -U Administrator
# Ввести пароль Administrator
```

#### Настройка прав группы hq на HQ-CLI:

Файл `/etc/sudoers` (через `visudo`):
```
%hq ALL=(ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id
```

> Это разрешает только cat, grep, id с sudo. Другие команды с повышенными привилегиями запрещены.

---

### 2. Файловое хранилище — RAID 0 на HQ-SRV

#### Создание RAID 0 из двух дисков:
```bash
mdadm --create /dev/md/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
```

> **Внимание:** В задании модуля 2 указан RAID 0 (уровень 0) из двух дисков, а не RAID 5! Подсказка в документе 1 содержит RAID 5 из трёх дисков — это для другого варианта.

#### Создание файловой системы:
```bash
mkfs.ext4 /dev/md/md0
```

#### Сохранение конфигурации:
```bash
mdadm --detail --scan >> /etc/mdadm.conf
```

#### Автомонтирование:
```bash
mkdir -p /raid
```

Добавить в `/etc/fstab`:
```
/dev/md/md0    /raid    ext4    defaults    0    0
```

```bash
mount -a
systemctl daemon-reload
```

---

### 3. NFS-сервер на HQ-SRV

#### Установка:
```bash
apt-get install nfs-kernel-server -y
```

#### Создание каталога:
```bash
mkdir -p /raid/nfs
chown nobody:nogroup /raid/nfs
chmod 777 /raid/nfs
```

#### Настройка экспорта — файл `/etc/exports`:
```
/raid/nfs 192.168.200.0/24(rw,sync,no_subtree_check,no_root_squash)
```

#### Запуск:
```bash
exportfs -a
systemctl enable --now nfs-kernel-server
```

#### Автомонтирование на HQ-CLI:
```bash
apt-get install nfs-common -y
mkdir -p /mnt/nfs
```

Добавить в `/etc/fstab`:
```
192.168.100.2:/raid/nfs    /mnt/nfs    nfs    defaults    0    0
```

```bash
mount -a
```

---

### 4. Chrony (NTP) на ISP

#### Установка на ISP:
```bash
apt-get install chrony -y
```

#### Настройка сервера — файл `/etc/chrony.conf`:
```
pool ru.pool.ntp.org iburst
allow all
local stratum 5
```

```bash
systemctl enable --now chronyd
```

#### Настройка на HQ-RTR и BR-RTR (EcoRouter):
```
en
conf t
ntp server 172.16.1.1
end
wr
```
(BR-RTR: `ntp server 172.16.2.1`)

#### Настройка клиентов (HQ-SRV, HQ-CLI, BR-SRV):
```bash
apt-get install chrony -y
```

Файл `/etc/chrony.conf`:
```
server 172.16.1.1
# или для BR-SRV:
# server 172.16.2.1
```

```bash
systemctl enable --now chronyd
```

---

### 5. Ansible на BR-SRV

#### Установка:
```bash
apt-get install ansible -y
```

#### Создание каталога и инвентаря:
```bash
mkdir -p /etc/ansible
```

Файл `/etc/ansible/hosts`:
```ini
[hq_servers]
hq-srv ansible_host=192.168.100.2

[hq_clients]
hq-cli ansible_host=192.168.200.2

[routers]
hq-rtr ansible_host=192.168.100.1
br-rtr ansible_host=192.168.0.1
```

> Для EcoRouter может потребоваться отдельный модуль или SSH-подключение напрямую.

Файл `/etc/ansible/ansible.cfg`:
```ini
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
remote_user = sshuser

[privilege_escalation]
become = True
become_method = sudo
```

#### Настройка SSH-ключей для подключения без пароля:
```bash
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
ssh-copy-id -p 2026 sshuser@192.168.100.2
ssh-copy-id -p 2026 sshuser@192.168.200.2
```

Для роутеров (если Alt Linux вместо EcoRouter):
```bash
ssh-copy-id -p 2026 sshuser@192.168.100.1
ssh-copy-id -p 2026 sshuser@192.168.0.1
```

Если порт SSH нестандартный, в инвентаре добавить `ansible_port=2026`:
```ini
[hq_servers]
hq-srv ansible_host=192.168.100.2 ansible_port=2026

[hq_clients]
hq-cli ansible_host=192.168.200.2 ansible_port=2026
```

#### Проверка:
```bash
ansible all -m ping
```
Все машины должны ответить `pong`.

---

### 6. Docker (testapp) на BR-SRV

#### Установка Docker:
```bash
apt-get install docker-engine docker-compose -y
systemctl enable --now docker
```

#### Импорт образов из Additional.iso:
```bash
# Подмонтировать ISO (если не подмонтирован):
mount /dev/cdrom /mnt
# или
mount /path/to/Additional.iso /mnt

# Импорт образов:
docker load -i /mnt/docker/site_latest.tar
docker load -i /mnt/docker/mariadb_latest.tar
```

#### Создание docker-compose файла `/home/user/docker-compose.yml`:
```yaml
version: '3.8'

services:
  testapp:
    image: site_latest
    container_name: testapp
    restart: always
    ports:
      - "8080:80"
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_NAME: testdb
      DB_USER: test
      DB_PASSWORD: P@ssw0rd

  db:
    image: mariadb_latest
    container_name: db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: P@ssw0rd
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: P@ssw0rd
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:
```

#### Запуск:
```bash
cd /home/user
docker-compose up -d
```

#### Проверка:
```bash
docker ps
curl http://localhost:8080
```

---

### 7. Веб-приложение (Apache + MariaDB) на HQ-SRV

#### Установка:
```bash
apt-get install apache2 mariadb-server php php-mysql -y
systemctl enable --now mariadb
systemctl enable --now apache2
# или httpd2 в зависимости от дистрибутива
```

#### Подмонтировать Additional.iso и скопировать файлы:
```bash
mount /dev/cdrom /mnt
cp /mnt/web/index.php /var/www/html/
cp -r /mnt/web/images /var/www/html/
```

#### Настройка БД:
```bash
mysql -u root << EOF
CREATE DATABASE webdb DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
FLUSH PRIVILEGES;
EOF
```

#### Импорт дампа:
```bash
mysql -u root webdb < /mnt/web/dump.sql
```

#### Редактирование index.php:
В файле `/var/www/html/index.php` указать правильные данные подключения:
```php
$db_host = 'localhost';
$db_name = 'webdb';
$db_user = 'web';
$db_pass = 'P@ssw0rd';
```

#### Перезапуск:
```bash
systemctl restart apache2
# или httpd2
```

---

### 8. Статическая трансляция портов (port forwarding)

#### На BR-RTR (EcoRouter) — проброс testapp:
```
en
conf t
ip nat source static tcp 192.168.0.2 8080 interface to-isp 8080
end
wr
```

> Это пробрасывает порт 8080 на внешнем интерфейсе BR-RTR → порт 8080 BR-SRV.

#### На HQ-RTR (EcoRouter) — проброс веб-приложения и SSH:
```
en
conf t
ip nat source static tcp 192.168.100.2 80 interface to-isp 8080
ip nat source static tcp 192.168.100.2 2026 interface to-isp 2026
end
wr
```

#### На BR-RTR — проброс SSH:
```
en
conf t
ip nat source static tcp 192.168.0.2 2026 interface to-isp 2026
end
wr
```

> **Синтаксис EcoRouter static NAT:**
> `ip nat source static tcp <внутренний_IP> <внутренний_порт> interface <внешний_интерфейс> <внешний_порт>`

---

### 9. NGINX как обратный прокси на ISP

#### Установка:
```bash
apt-get install nginx -y
```

#### Конфигурация — файл `/etc/nginx/sites-available/proxy.conf`:
```nginx
server {
    listen 80;
    server_name web.au-team.irpo;

    location / {
        proxy_pass http://172.16.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name docker.au-team.irpo;

    location / {
        proxy_pass http://172.16.2.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Активация:
```bash
ln -s /etc/nginx/sites-available/proxy.conf /etc/nginx/sites-enabled/
nginx -t
systemctl enable --now nginx
systemctl reload nginx
```

---

### 10. HTTP Basic Auth на ISP (для web.au-team.irpo)

#### Установка утилиты:
```bash
apt-get install apache2-utils -y
# или
apt-get install httpd-tools -y
```

#### Создание файла паролей:
```bash
htpasswd -cb /etc/nginx/.htpasswd WEB P@ssw0rd
```

#### Добавление аутентификации в конфигурацию nginx:

Отредактировать серверный блок для `web.au-team.irpo`:
```nginx
server {
    listen 80;
    server_name web.au-team.irpo;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://172.16.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
nginx -t
systemctl reload nginx
```

---

### 11. Установка Яндекс Браузера на HQ-CLI

```bash
# Скачать RPM с официального сайта (если есть интернет):
curl -o yandex-browser.rpm https://repo.yandex.ru/yandex-browser/rpm/stable/x86_64/yandex-browser-stable-latest.x86_64.rpm

# Установить:
rpm -i yandex-browser.rpm
# или
apt-get install /path/to/yandex-browser.rpm

# Альтернативный способ — через репозиторий:
cat > /etc/yum.repos.d/yandex-browser.repo << EOF
[yandex-browser]
name=Yandex Browser
baseurl=https://repo.yandex.ru/yandex-browser/rpm/stable/x86_64/
enabled=1
gpgcheck=0
EOF

apt-get update
apt-get install yandex-browser-stable -y
```

> Если интернета нет, браузер может быть в Additional.iso.

---

## Ключевые моменты для проверки

1. **Связность:** ping между всеми устройствами через GRE-туннель
2. **DNS:** `nslookup hq-srv.au-team.irpo` с любого устройства
3. **DHCP:** HQ-CLI получает IP, шлюз, DNS автоматически
4. **SSH:** `ssh -p 2026 sshuser@<IP>` работает, баннер отображается
5. **NAT:** все устройства имеют доступ в интернет
6. **NFS:** файлы доступны на HQ-CLI в `/mnt/nfs`
7. **Samba DC:** доменные пользователи входят на HQ-CLI
8. **Docker:** `curl http://BR-SRV:8080` отвечает
9. **Web-app:** `curl http://HQ-SRV` отвечает
10. **NGINX proxy:** доступ по доменным именам через ISP
11. **Chrony:** `chronyc sources` показывает синхронизацию
