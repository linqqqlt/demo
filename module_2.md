11 YANDEX BROWSER (HQ-CLI)
apt-get update
apt-get install -y yandex-browser-stable

1. Контроллер домена Samba DC на BR-SRV
Установка:

apt-get install task-samba-dc -y
Подготовка:

rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba /var/cache/samba
mkdir -p /var/lib/samba/sysvol
Провизионирование:

samba-tool domain provision
Realm: AU-TEAM.IRPO
Domain: AU-TEAM
Server Role: dc
DNS backend: SAMBA_INTERNAL
DNS forwarder: 77.88.8.8
systemctl enable --now samba
Создание пользователей:

for i in 1 2 3 4 5; do
    samba-tool user create hquser${i} P@ssw0rd
    samba-tool user setexpiry hquser${i} --noexpiry
done
Создание группы:

samba-tool group add hq
for i in 1 2 3 4 5; do
    samba-tool group addmembers hq hquser${i}
done

# На HQ-CLI:
поменять /etc/resolv.conf
77.88.8.8
apt-get update
apt-get install task-auth-ad-sssd -y
поменять /etc/resolv.conf
192.168.0.2
kinit administrator@AU-TEAM.IRPO
/usr/sbin/system-auth write ad au-team.irpo AU-TEAM administrator P@ssw0rd
/etc/samba/smb.conf
 workgroup = AU-TEAM
    realm = AU-TEAM.IRPO
    security = ads
    server role = member server
# 5. Ввод в домен
net ads join -U administrator%P@ssw0rd
net ads testjoin

# 6. SSSD
systemctl enable --now sssd
getent passwd administrator@au-team.irpo
getent group hq

# 7. sudo для группы hq
echo "%hq ALL=(ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id" > /etc/sudoers.d/hq
chmod 440 /etc/sudoers.d/hq

2 RAID 0 (HQ-SRV)
apt-get update && apt-get install -y mdadm
mdadm --zero-superblock --force /dev/sdb /dev/sdc
mdadm --create --verbose /dev/md0 -l 0 -n 2 /dev/sdb /dev/sdc
mdadm --detail --scan --verbose | tee -a /etc/mdadm.conf
mkfs.ext4 /dev/md0
mkdir /raid
echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
mount -a
systemctl daemon-reload
3 NFS-сервер (HQ-SRV)
На HQ-SRV (192.168.100.2):
apt-get install -y nfs-server nfs-utils
mkdir -p /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.100.0/24(rw,no_root_squash,sync)" >> /etc/exports
exportfs -arv
systemctl enable --now nfs-server
На HQ-CLI (192.168.100.10):
apt-get install -y nfs-clients nfs-utils
mkdir -p /mnt/nfs
echo "192.168.100.2:/raid/nfs /mnt/nfs nfs defaults 0 0" >> /etc/fstab
mount -av
4. Chrony (NTP)
ISP — сервер:

apt-get install chrony -y

 /etc/chrony.conf  
local stratum 5
allow all


systemctl enable --now chronyd
systemctl restart chronyd
HQ-RTR:

en
conf t
ntp server 172.16.1.1
end
wr
BR-RTR:

en
conf t
ntp server 172.16.2.1
end
wr
HQ-SRV, HQ-CLI, BR-SRV — клиенты:

apt-get install chrony -y

 /etc/chrony.conf 
server 172.16.1.1 iburst



systemctl enable --now chronyd
systemctl restart chronyd
5. Ansible на BR-SRV
apt-get install ansible -y
mkdir -p /etc/ansible
Файл /etc/ansible/hosts:

[hq_servers]
hq-srv ansible_host=192.168.100.2 ansible_port=2026

[hq_clients]
hq-cli ansible_host=192.168.100.3 ansible_port=2026

[routers]
hq-rtr ansible_host=192.168.100.1
br-rtr ansible_host=192.168.0.1
Файл /etc/ansible/ansible.cfg:

[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
remote_user = sshuser

[privilege_escalation]
become = True
become_method = sudo
SSH-ключи:

ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
ssh-copy-id -p 2026 sshuser@192.168.100.2
ssh-copy-id -p 2026 sshuser@192.168.100.3
Проверка:

ansible all -m ping

6. Docker (testapp) на BR-SRV
apt-get install docker-engine docker-compose -y
systemctl enable --now docker

# Подмонтировать ISO:
mount /dev/cdrom /mnt

# Импорт образов:
docker load -i /mnt/docker/site_latest.tar
docker load -i /mnt/docker/mariadb_latest.tar
Файл /home/user/docker-compose.yml:

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
cd /home/user
docker-compose up -d
docker ps

7. Веб-приложение (Apache + MariaDB) на HQ-SRV
apt-get install apache2 mariadb-server php8.5 php8.5-myslnd -y
systemctl enable --now mariadb
systemctl enable --now httpd2

8. Проброс портов (static NAT)
На HQ-RTR:

en
conf t
ip nat source static tcp 192.168.100.2 80 interface to-isp 8080
ip nat source static tcp 192.168.100.2 2026 interface to-isp 2026
end
wr
На BR-RTR:

en
conf t
ip nat source static tcp 192.168.0.2 8080 interface to-isp 8080
ip nat source static tcp 192.168.0.2 2026 interface to-isp 2026
end
wr
9. NGINX (обратный прокси) на ISP
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
systemctl restart nginx

10. HTTP Basic Auth на ISP (для web.au-team.irpo)
apt-get install apache2-utils -y
htpasswd -cb /etc/nginx/.htpasswd WEB P@ssw0rd
Добавить в серверный блок web.au-team.irpo (в proxy.conf):

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;
systemctl reload nginx
