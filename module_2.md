# Модуль 2: Развёртывание и настройка сервисов

## 1. Установка Yandex Browser (HQ-CLI)
```bash
apt-get update
apt-get install -y yandex-browser-stable
```

## 2. Контроллер домена Samba DC (BR-SRV)

### Установка и подготовка
```bash
apt-get install task-samba-dc -y
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba /var/cache/samba
mkdir -p /var/lib/samba/sysvol
```

### Провизионирование домена
```bash
samba-tool domain provision
```
**Параметры:**
- Realm: `AU-TEAM.IRPO`
- Domain: `AU-TEAM`
- Server Role: `dc`
- DNS backend: `SAMBA_INTERNAL`
- DNS forwarder: `77.88.8.8`

```bash
systemctl enable --now samba
```

### Создание пользователей и групп
```bash
for i in 1 2 3 4 5; do
  samba-tool user create hquser${i} P@ssw0rd
  samba-tool user setexpiry hquser${i} --noexpiry
done

samba-tool group add hq
for i in 1 2 3 4 5; do
  samba-tool group addmembers hq hquser${i}
done
```

### Настройка клиента (HQ-CLI)
1. Временно изменить DNS:
   ```bash
   echo "nameserver 77.88.8.8" > /etc/resolv.conf
   ```
2. Установить пакеты аутентификации:
   ```bash
   apt-get update
   apt-get install task-auth-ad-sssd -y
   ```
3. Вернуть адрес контроллера домена:
   ```bash
   echo "nameserver 192.168.0.2" > /etc/resolv.conf
   ```

### Аутентификация и конфигурация Samba
```bash
kinit administrator@AU-TEAM.IRPO
/usr/sbin/system-auth write ad au-team.irpo AU-TEAM administrator P@ssw0rd
```

**`/etc/samba/smb.conf`**:
```ini
workgroup = AU-TEAM
realm = AU-TEAM.IRPO
security = ads
server role = member server
```

### Ввод в домен
```bash
net ads join -U administrator%P@ssw0rd
net ads testjoin
```

### Настройка SSSD
```bash
systemctl enable --now sssd
getent passwd administrator@au-team.irpo
getent group hq
```

### Настройка sudo для группы `hq`
```bash
echo "%hq ALL=(ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id" > /etc/sudoers.d/hq
chmod 440 /etc/sudoers.d/hq
```

## 3. RAID 0 (HQ-SRV)
```bash
apt-get update && apt-get install -y mdadm
mdadm --zero-superblock --force /dev/sdb /dev/sdc
mdadm --create --verbose /dev/md0 -l 0 -n 2 /dev/sdb /dev/sdc
mdadm --detail --scan --verbose | tee -a /etc/mdadm.conf
mkfs.ext4 /dev/md0
mkdir /raid
echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
mount -a
systemctl daemon-reload
```

## 4. NFS-сервер (HQ-SRV)

### Настройка сервера (192.168.100.2)
```bash
apt-get install -y nfs-server nfs-utils
mkdir -p /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.100.0/24(rw,no_root_squash,sync)" >> /etc/exports
exportfs -arv
systemctl enable --now nfs-server
```

### Настройка клиента (HQ-CLI, 192.168.100.10)
```bash
apt-get install -y nfs-clients nfs-utils
mkdir -p /mnt/nfs
echo "192.168.100.2:/raid/nfs /mnt/nfs nfs defaults 0 0" >> /etc/fstab
mount -av
```

## 5. Chrony (NTP)

### Сервер времени (ISP)
**`/etc/chrony.conf`**:
```conf
local stratum 5
allow all
```
```bash
apt-get install chrony -y
systemctl enable --now chronyd
systemctl restart chronyd
```

### Маршрутизаторы
**HQ-RTR**:
```cisco
en
conf t
ntp server 172.16.1.1
end
wr
```

**BR-RTR**:
```cisco
en
conf t
ntp server 172.16.2.1
end
wr
```

### Клиенты (HQ-SRV, HQ-CLI, BR-SRV)
**`/etc/chrony.conf`**:
```conf
server 172.16.1.1 iburst
```
```bash
apt-get install chrony -y
systemctl enable --now chronyd
systemctl restart chronyd
```

## 6. Ansible (BR-SRV)
```bash
apt-get install ansible -y
mkdir -p /etc/ansible
```

**`/etc/ansible/hosts`**:
```ini
[hq_servers]
hq-srv ansible_host=192.168.100.2 ansible_port=2026

[hq_clients]
hq-cli ansible_host=192.168.100.3 ansible_port=2026

[routers]
hq-rtr ansible_host=192.168.100.1
br-rtr ansible_host=192.168.0.1
```

**`/etc/ansible/ansible.cfg`**:
```ini
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
remote_user = sshuser

[privilege_escalation]
become = True
become_method = sudo
```

**Настройка SSH-ключей**:
```bash
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
ssh-copy-id -p 2026 sshuser@192.168.100.2
ssh-copy-id -p 2026 sshuser@192.168.100.3
```

**Проверка подключения**:
```bash
ansible all -m ping
```

## 7. Docker (testapp) (BR-SRV)
```bash
apt-get install docker-engine docker-compose -y
systemctl enable --now docker
```

**Импорт образов**:
```bash
mount /dev/cdrom /mnt
docker load -i /mnt/docker/site_latest.tar
docker load -i /mnt/docker/mariadb_latest.tar
```

**`/home/user/docker-compose.yml`**:
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
      - db/var/lib/mysql

volumes:
  db
```

**Запуск**:
```bash
cd /home/user
docker-compose up -d
docker ps
```

## 8. Веб-приложение (Apache + MariaDB) (HQ-SRV)
```bash
apt-get install apache2 mariadb-server php8.5 php8.5-myslnd -y
systemctl enable --now mariadb
systemctl enable --now httpd2
```

## 9. Проброс портов (Static NAT)

### HQ-RTR
```cisco
en
conf t
ip nat source static tcp 192.168.100.2 80 interface to-isp 8080
ip nat source static tcp 192.168.100.2 2026 interface to-isp 2026
end
wr
```

### BR-RTR
```cisco
en
conf t
ip nat source static tcp 192.168.0.2 8080 interface to-isp 8080
ip nat source static tcp 192.168.0.2 2026 interface to-isp 2026
end
wr
```

## 10. NGINX (обратный прокси) (ISP)
```bash
apt-get install nginx -y
```

**`/etc/nginx/sites-available/proxy.conf`**:
```nginx
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
```

**Активация и запуск**:
```bash
ln -s /etc/nginx/sites-available/proxy.conf /etc/nginx/sites-enabled/
nginx -t
systemctl enable --now nginx
systemctl restart nginx
```

## 11. HTTP Basic Auth (ISP, для `web.au-team.irpo`)
```bash
apt-get install apache2-utils -y
htpasswd -cb /etc/nginx/.htpasswd WEB P@ssw0rd
```

Добавить в серверный блок `web.au-team.irpo` в файле `proxy.conf`:
```nginx
auth_basic "Restricted Access";
auth_basic_user_file /etc/nginx/.htpasswd;
```

Применить изменения:
```bash
systemctl reload nginx
```
