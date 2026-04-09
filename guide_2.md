# Полное решение ДЭ — ИСПРАВЛЕННОЕ

> Все исправления учтены: без VLAN, порты ge0/ge1, интерфейсы ens33, одна подсеть HQ, привязка connect port.

---

## ТАБЛИЦА АДРЕСАЦИИ

| Устройство | Интерфейс | IP-адрес | Шлюз |
|---|---|---|---|
| ISP | ens33 (Internet) | DHCP | DHCP |
| ISP | ens34 (к HQ-RTR) | 172.16.1.1/28 | — |
| ISP | ens35 (к BR-RTR) | 172.16.2.1/28 | — |
| HQ-RTR | ge0 → to-isp | 172.16.1.2/28 | 172.16.1.1 |
| HQ-RTR | ge1 → to-lan | 192.168.100.1/24 | — |
| BR-RTR | ge0 → to-isp | 172.16.2.2/28 | 172.16.2.1 |
| BR-RTR | ge1 → to-lan | 192.168.0.1/28 | — |
| HQ-SRV | ens33 | 192.168.100.2/24 | 192.168.100.1 |
| HQ-CLI | ens33 | 192.168.100.3/24 (или DHCP) | 192.168.100.1 |
| BR-SRV | ens33 | 192.168.0.2/28 | 192.168.0.1 |

---

## МОДУЛЬ 1

---

### 1. Имена устройств

**Alt Linux (ISP, HQ-SRV, HQ-CLI, BR-SRV):**
```bash
# ISP:
hostnamectl set-hostname isp.au-team.irpo
# HQ-SRV:
hostnamectl set-hostname hq-srv.au-team.irpo
# HQ-CLI:
hostnamectl set-hostname hq-cli.au-team.irpo
# BR-SRV:
hostnamectl set-hostname br-srv.au-team.irpo
```

**EcoRouter (HQ-RTR, BR-RTR):**
```
en
conf t
hostname HQ-RTR.au-team.irpo
```
```
en
conf t
hostname BR-RTR.au-team.irpo
```

---

### 2. Настройка ISP

**Интерфейсы:**
```bash
# ens33 — интернет (DHCP)
mkdir -p /etc/net/ifaces/ens33
cat > /etc/net/ifaces/ens33/options << EOF
TYPE=eth
BOOTPROTO=dhcp
NM_CONTROLLED=no
EOF

# ens34 — к HQ-RTR
mkdir -p /etc/net/ifaces/ens34
cat > /etc/net/ifaces/ens34/options << EOF
TYPE=eth
BOOTPROTO=static
NM_CONTROLLED=no
EOF
echo "172.16.1.1/28" > /etc/net/ifaces/ens34/ipv4address

# ens35 — к BR-RTR
mkdir -p /etc/net/ifaces/ens35
cat > /etc/net/ifaces/ens35/options << EOF
TYPE=eth
BOOTPROTO=static
NM_CONTROLLED=no
EOF
echo "172.16.2.1/28" > /etc/net/ifaces/ens35/ipv4address

systemctl restart network
```

**IP forwarding и NAT:**
```bash
echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
sysctl -p /etc/net/sysctl.conf

iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

systemctl restart network
```

---

### 3. Настройка HQ-RTR (EcoRouter)

```
en
conf t

port ge0
service-instance to-isp
encapsulation untagged
exit
exit

port ge1
service-instance to-lan
encapsulation untagged
exit
exit

interface to-isp
connect port ge0 service-instance to-isp
ip address 172.16.1.2/28
ip nat outside
no shutdown
exit

interface to-lan
connect port ge1 service-instance to-lan
ip address 192.168.100.1/24
ip nat inside
no shutdown
exit

ip route 0.0.0.0/0 172.16.1.1

ip nat pool NAT 192.168.100.1-192.168.100.254
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

end
wr
```

---

### 4. Настройка BR-RTR (EcoRouter)

```
en
conf t

port ge0
service-instance to-isp
encapsulation untagged
exit
exit

port ge1
service-instance to-lan
encapsulation untagged
exit
exit

interface to-isp
connect port ge0 service-instance to-isp
ip address 172.16.2.2/28
ip nat outside
no shutdown
exit

interface to-lan
connect port ge1 service-instance to-lan
ip address 192.168.0.1/28
ip nat inside
no shutdown
exit

ip route 0.0.0.0/0 172.16.2.1

ip nat pool NAT 192.168.0.1-192.168.0.14
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

end
wr
```

---

### 5. Настройка HQ-SRV

```bash
mkdir -p /etc/net/ifaces/ens33
cat > /etc/net/ifaces/ens33/options << EOF
TYPE=eth
BOOTPROTO=static
NM_CONTROLLED=no
EOF

echo "192.168.100.2/24" > /etc/net/ifaces/ens33/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens33/ipv4route

systemctl restart network
ping 192.168.100.1
```

---

### 6. Настройка HQ-CLI

```bash
systemctl stop NetworkManager
systemctl disable NetworkManager

mkdir -p /etc/net/ifaces/ens33
cat > /etc/net/ifaces/ens33/options << EOF
TYPE=eth
BOOTPROTO=static
NM_CONTROLLED=no
EOF

echo "192.168.100.3/24" > /etc/net/ifaces/ens33/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens33/ipv4route

systemctl restart network
ping 192.168.100.1
```

---

### 7. Настройка BR-SRV

```bash
mkdir -p /etc/net/ifaces/ens33
cat > /etc/net/ifaces/ens33/options << EOF
TYPE=eth
BOOTPROTO=static
NM_CONTROLLED=no
EOF

echo "192.168.0.2/28" > /etc/net/ifaces/ens33/ipv4address
echo "default via 192.168.0.1" > /etc/net/ifaces/ens33/ipv4route

systemctl restart network
ping 192.168.0.1
```

---

### 8. Учётные записи

**На HQ-SRV и BR-SRV:**
```bash
adduser sshuser
echo "sshuser:P@ssw0rd" | chpasswd
usermod -u 2026 sshuser
usermod -aG wheel sshuser

visudo
# Раскомментировать:
# %wheel ALL=(ALL) NOPASSWD: ALL
```

**На HQ-RTR и BR-RTR:**
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

### 9. Настройка SSH

**На HQ-SRV и BR-SRV:**
```bash
cat >> /etc/openssh/sshd_config << EOF
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/banner
EOF

echo "Authorized access only" > /etc/openssh/banner

systemctl enable --now sshd
systemctl restart sshd
```

---

### 10. GRE-туннель

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

**Проверка:**
```
do ping 10.0.0.2   # с HQ-RTR
do ping 10.0.0.1   # с BR-RTR
```

---

### 11. OSPF

**HQ-RTR:**
```
en
conf t
router ospf 1
network 192.168.100.0/24 area 0
network 10.0.0.0/30 area 0
passive-interface to-lan
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
passive-interface to-lan
passive-interface to-isp
exit

interface tunnel.1
ip ospf authentication
ip ospf authentication-key P@ssw0rd
exit
end
wr
```

**Проверка:**
```
do show ip ospf neighbor
```

---

### 12. DHCP (на HQ-RTR для HQ-CLI)

```
en
conf t
ip pool DHCP
range 192.168.100.10-192.168.100.254

dhcp-server 1
pool DHCP 1
mask 24
gateway 192.168.100.1
dns 192.168.100.2
domain-name au-team.irpo
domain-search au-team.irpo
exit

interface to-lan
dhcp-server 1
exit
end
wr
```

> Если HQ-CLI хочешь на DHCP — поменяй BOOTPROTO=dhcp в options и убери ipv4address/ipv4route.

---

### 13. DNS на HQ-SRV

**Установка:**
```bash
apt-get install bind bind-utils -y
```

**Настройка options.conf** — в блок `options { }` добавить:
```
listen-on { any; };
allow-query { any; };
allow-recursion { any; };
forwarders { 77.88.8.8; };
```

**Добавить зоны в /etc/bind/rfc1912.conf:**
```bash
cat >> /etc/bind/rfc1912.conf << 'EOF'

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
EOF
```

**Создать файлы зон:**
```bash
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/au-team.irpo
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/100.168.192.in-addr.arpa
```

**Файл прямой зоны /var/lib/bind/etc/zone/au-team.irpo:**
```bash
cat > /var/lib/bind/etc/zone/au-team.irpo << 'EOF'
$TTL 86400
@       IN  SOA au-team.irpo. root.au-team.irpo. (
            2026040501  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS   au-team.irpo.
@       IN  A    192.168.100.2
hq-rtr  IN  A    192.168.100.1
hq-srv  IN  A    192.168.100.2
hq-cli  IN  A    192.168.100.3
br-rtr  IN  A    192.168.0.1
br-srv  IN  A    192.168.0.2
docker  IN  A    172.16.1.1
web     IN  A    172.16.2.1
EOF
```

> ВАЖНО: строка `@ IN A 192.168.100.2` обязательна — без неё BIND падает с ошибкой "NS has no address record".

**Файл обратной зоны /var/lib/bind/etc/zone/100.168.192.in-addr.arpa:**
```bash
cat > /var/lib/bind/etc/zone/100.168.192.in-addr.arpa << 'EOF'
$TTL 86400
@       IN  SOA au-team.irpo. root.au-team.irpo. (
            2026040501  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS   au-team.irpo.
1       IN  PTR  hq-rtr.au-team.irpo.
2       IN  PTR  hq-srv.au-team.irpo.
3       IN  PTR  hq-cli.au-team.irpo.
EOF
```

**Генерация ключа и запуск:**
```bash
rndc-confgen > /var/lib/bind/etc/rndc.key
sed -i '6,$d' /var/lib/bind/etc/rndc.key

systemctl enable --now bind
systemctl restart bind
systemctl status bind
```

> Если ошибка "'options' redefined" — открой /var/lib/bind/etc/rndc.key и удали всё кроме блока key { }.

**Настройка resolv.conf на HQ-SRV:**
```bash
cat > /etc/resolv.conf << EOF
search au-team.irpo
nameserver 192.168.100.2
EOF
```

**На всех остальных машинах тоже прописать DNS:**
```bash
cat > /etc/resolv.conf << EOF
search au-team.irpo
nameserver 192.168.100.2
EOF
```

---

### 14. Часовой пояс

**Alt Linux (все):**
```bash
apt-get install tzdata -y
timedatectl set-timezone Europe/Moscow
```

**EcoRouter (HQ-RTR, BR-RTR):**
```
en
conf t
ntp timezone utc+3
end
wr
```

---

---

## МОДУЛЬ 2

> Модуль 2 выполняется на отдельном стенде, где уже преднастроены IP, NAT, туннель, OSPF, SSH, DHCP, DNS.

---

### 1. Контроллер домена Samba DC на BR-SRV

**Установка:**
```bash
apt-get install task-samba-dc -y
```

**Подготовка:**
```bash
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba /var/cache/samba
mkdir -p /var/lib/samba/sysvol
```

**Провизионирование:**
```bash
samba-tool domain provision
```
- Realm: **AU-TEAM.IRPO**
- Domain: **AU-TEAM**
- Server Role: **dc**
- DNS backend: **SAMBA_INTERNAL**
- DNS forwarder: **77.88.8.8**

```bash
systemctl enable --now samba
```

**Создание пользователей:**
```bash
for i in 1 2 3 4 5; do
    samba-tool user create hquser${i} P@ssw0rd
    samba-tool user setexpiry hquser${i} --noexpiry
done
```

**Создание группы:**
```bash
samba-tool group add hq
for i in 1 2 3 4 5; do
    samba-tool group addmembers hq hquser${i}
done
```

**Ввод HQ-CLI в домен:**
```bash
# На HQ-CLI:
apt-get install task-auth-ad-sssd -y
system-auth write ad au-team.irpo AU-TEAM BR-SRV.au-team.irpo administrator P@ssw0rd
```

**Права группы hq на HQ-CLI (visudo):**
```
%hq ALL=(ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id
```

---

### 2. RAID 0 на HQ-SRV

### Создание RAID 5 на HQ-SRV

```bash
mdadm --create --level=5 --raid-devices=3 /dev/md/md0 /dev/sdb /dev/sdc /dev/sdd
mkfs -t ext4 /dev/md/md0
```

### Монтирование RAID массива

Файл: `/etc/fstab`
```
/dev/md/md0    /raid5    ext4    defaults    0    0
```

```bash
mkdir -p /raid5
mount -a
systemctl daemon-reload
```

### 3. Настройка NFS сервера

```bash
apt-get update
apt-get install nfs-kernel-server

mkdir -p /raid5/nfs
chown nobody:nogroup /raid5/nfs
chmod 777 /raid5/nfs
```

**Конфигурация экспорта:**

Файл: `/etc/exports`
```
/raid5/nfs 192.168.200.0/28(rw,sync,no_subtree_check,no_root_squash)
```

```bash
exportfs -a
systemctl start nfs-kernel-server
systemctl enable nfs-kernel-server
```

### Автомонтирование на HQ-CLI

```bash
apt-get install nfs-common
mkdir -p /mnt/nfs
mount 192.168.100.10:/raid5/nfs /mnt/nfs
```

Файл: `/etc/fstab`
```
192.168.100.10:/raid5/nfs    /mnt/nfs    nfs    defaults    0    0
```

### 4. Chrony (NTP)

**ISP — сервер:**
```bash
apt-get install chrony -y

cat > /etc/chrony.conf << EOF
local stratum 5
allow all
driftfile /var/lib/chrony/drift
EOF

systemctl enable --now chronyd
systemctl restart chronyd
```

**HQ-RTR:**
```
en
conf t
ntp server 172.16.1.1
end
wr
```

**BR-RTR:**
```
en
conf t
ntp server 172.16.2.1
end
wr
```

**HQ-SRV, HQ-CLI, BR-SRV — клиенты:**
```bash
apt-get install chrony -y

cat > /etc/chrony.conf << EOF
server 172.16.1.1 iburst
driftfile /var/lib/chrony/drift
EOF

systemctl enable --now chronyd
systemctl restart chronyd
```

> Для BR-SRV указать `server 172.16.2.1 iburst`

---

### 5. Ansible на BR-SRV

```bash
apt-get install ansible -y
mkdir -p /etc/ansible
```

**Файл /etc/ansible/hosts:**
```ini
[hq_servers]
hq-srv ansible_host=192.168.100.2 ansible_port=2026

[hq_clients]
hq-cli ansible_host=192.168.100.3 ansible_port=2026

[routers]
hq-rtr ansible_host=192.168.100.1
br-rtr ansible_host=192.168.0.1
```

**Файл /etc/ansible/ansible.cfg:**
```ini
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
remote_user = sshuser

[privilege_escalation]
become = True
become_method = sudo
```

**SSH-ключи:**
```bash
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
ssh-copy-id -p 2026 sshuser@192.168.100.2
ssh-copy-id -p 2026 sshuser@192.168.100.3
```

**Проверка:**
```bash
ansible all -m ping
```

---

### 6. Docker (testapp) на BR-SRV

```bash
apt-get install docker-engine docker-compose -y
systemctl enable --now docker

# Подмонтировать ISO:
mount /dev/cdrom /mnt

# Импорт образов:
docker load -i /mnt/docker/site_latest.tar
docker load -i /mnt/docker/mariadb_latest.tar
```

**Файл /home/user/docker-compose.yml:**
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

```bash
cd /home/user
docker-compose up -d
docker ps
```

---

### 7. Веб-приложение (Apache + MariaDB) на HQ-SRV

```bash
apt-get install apache2 mariadb-server php php-mysql -y
systemctl enable --now mariadb
systemctl enable --now apache2

mount /dev/cdrom /mnt
cp /mnt/web/index.php /var/www/html/
cp -r /mnt/web/images /var/www/html/

mysql -u root << EOF
CREATE DATABASE webdb DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
FLUSH PRIVILEGES;
EOF

mysql -u root webdb < /mnt/web/dump.sql
```

В `/var/www/html/index.php` указать:
```php
$db_host = 'localhost';
$db_name = 'webdb';
$db_user = 'web';
$db_pass = 'P@ssw0rd';
```

```bash
systemctl restart apache2
```

---

### 8. Проброс портов (static NAT)

**На HQ-RTR:**
```
en
conf t
ip nat source static tcp 192.168.100.2 80 interface to-isp 8080
ip nat source static tcp 192.168.100.2 2026 interface to-isp 2026
end
wr
```

**На BR-RTR:**
```
en
conf t
ip nat source static tcp 192.168.0.2 8080 interface to-isp 8080
ip nat source static tcp 192.168.0.2 2026 interface to-isp 2026
end
wr
```

---

### 9. NGINX (обратный прокси) на ISP

```bash
apt-get install nginx -y

cat > /etc/nginx/sites-available/proxy.conf << 'EOF'
server {
    listen 80;
    server_name web.au-team.irpo;

    location / {
        proxy_pass http://172.16.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
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
    }
}
EOF

ln -s /etc/nginx/sites-available/proxy.conf /etc/nginx/sites-enabled/
nginx -t
systemctl enable --now nginx
systemctl reload nginx
```

---

### 10. HTTP Basic Auth на ISP (для web.au-team.irpo)

```bash
apt-get install apache2-utils -y
htpasswd -cb /etc/nginx/.htpasswd WEB P@ssw0rd
```

Добавить в серверный блок web.au-team.irpo (в proxy.conf):
```
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;
```

```bash
systemctl reload nginx
```

---

### 11. Яндекс Браузер на HQ-CLI

```bash
apt-get update
apt-get install yandex-browser-stable -y
```

---

## ПРОВЕРКА

1. `ping 192.168.100.1` с HQ-SRV и HQ-CLI — связь с роутером
2. `ping 8.8.8.8` с HQ-SRV — интернет
3. `ping 10.0.0.2` с HQ-RTR — GRE-туннель
4. `do show ip ospf neighbor` на роутерах — OSPF сосед
5. `ping 192.168.0.2` с HQ-SRV — маршрутизация через туннель
6. `nslookup hq-srv.au-team.irpo` — DNS
7. `ssh -p 2026 sshuser@192.168.100.2` — SSH
8. `chronyc sources` — NTP
9. `curl http://localhost:8080` на BR-SRV — Docker testapp
10. `curl http://localhost` на HQ-SRV — Apache веб-приложение
