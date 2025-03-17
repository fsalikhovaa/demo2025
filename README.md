# Demo2025
<p align="center">
  <img src="https://github.com/fsalikhovaa/demo2025/blob/main/Frame%201%20(1).png"/>
</p>

## МОДУЛЬ 1

### Базовая настройка устройств 
Настройте имена устройств согласно топологии. Используйте полное доменное имя 
```
hostnamectl set-hostname имя_устройства; exec bash
```
На всех устройствах необходимо сконфигурировать IPv4

Для устройств с графическим интерфейсом:
```
Правой клавишей по сети, параметры соединений и НЕ забыть сохранить
```
Для устройств БЕЗ графического интерфейса:
```
nmtui
```
### Адресация
<p align="center">
  <img src="https://github.com/fsalikhovaa/demo2025/blob/main/Снимок%20экрана%202025-03-10%20134140.png"/>
</p>

### Создание локальных учетных записей 
Создайте пользователя sshuser на серверах HQ-SRV и BR-SRV. Пароль: P@ssw0rd, идентификатор 1010, пользователь должен иметь возможность запускать sudo без дополнительной аутентификации.
```
useradd -m -u 1010 sshuser
passwd sshuser
nano /etc/sudoers
sshuser ALL=(ALL:ALL)NOPASSWD:ALL
ctrl+x
y
enter
```
Создайте пользователя net_admin с паролем P@$$word на маршрутизаторах HQ-RTR и BR-RTR. Должен обладать максимальными привилегиями и запускать sudo без дополнительной аутентификации. Создаем так же, как и пользователя выше.

### Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV:
```
nano /etc/mybanner
```
В этом НОВОМ ПУСТОМ файле пишем Authorized access only!

ctrl+x
y
enter
```
nano /etc/openssh/sshd_config
```
Находим строчки:

#port 22, раскоменчиваем и пишем порт 2024

Banner /etc/mybanner

MaxAuthTries 2

ДОБАВЛЯЕМ строчку:

AllowUsers sshuser

ctrl+x
y
enter
```
systemctl restart sshd.service
```

### Между офисами HQ и BR необходимо сконфигурировать ip туннель
Перед настройкой самого тоннеля необходимо убедиться, что на ISP включён forwarding IPv4
### ISP:
```
nano /etc/net/sysctl.conf
```
и меняем строчку net ipv4 forwarding значение на 1

<p align="center">
  <img src="https://github.com/fsalikhovaa/demo2025/blob/main/isp.png">
</p>

Настраиваем GRE через nmtui
### BR-RTR:

<p align="center">
  <img src="https://github.com/fsalikhovaa/demo2025/blob/main/br.png">
</p>

### HQ-RTR:
<p align="center">
  <img src="https://github.com/fsalikhovaa/demo2025/blob/main/hq.png">
</p>

### Обеспечьте динамическую маршрутизацию: ресурсы одного офиса должны быть доступны из другого офиса. Для обеспечения динамической маршрутизации используйте link state протокол на ваше усмотрение.
Настраиваем OSPF
```
nano /etc/frr/daemons
```
меняем строчку
ospfd=no на строчку
ospfd=yes

### HQ-RTR:
```
systemctl enable --now frr
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.0.0/26 area 0
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do write memory
exit
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr
```

### BR-RTR:
Повторяем настройку выше со своими адресами.

### Настройка протокола динамической конфигурации хостов 
Для офиса HQ сервером DHCP выступает HQ-RTR. Клиентом является машина HQ-CLI
```
nano /etc/sysconfig/dhcpd
DHCPARGS=ens35
ctrl+x
y
enter

cp /etc/dhcp/dhcpd.conf{.example,}
nano /etc/dhcp/dhcpd.conf
```
В этом файле должны быить строчки:
```
option domain-name "au-team.irpo";
option domain-name-servers 172.16.0.2;
default-lease-time 6000;
max-lease-time 72000;

authoritative;

subnet 172.16.0.0 netmask 255.255.255.192 {
        range 172.16.0.3 172.16.0.8;
        option routers 172.16.0.;
}
```
ctrl+x
y
enter
```
systemctl enable --now dhcpd
```
### Настройка DNS для офисов HQ и BR
### HQ-SRV:

```
nano /etc/bind/options.conf
```

Меняем выделенные строчки:
<p align="center">
  <img src="https://github.com/fsalikhovaa/demo2025/blob/main/меняем%20строки%20в%20бинде.png"/>
</p>

```
systemctl enable --now bind
nano /etc/bind/local.conf
```

<p align="center">
  <img src="https://github.com/fsalikhovaa/demo2025/blob/main/локалконф%20днс.png"/>
</p>

```
cd /etc/bind/zone
cp localdomain au.db
cp 127.in-addr.arpa 0.db
chown root:named {au,0}.db
nano au.db
```

<p align="center">
  <img src="https://github.com/fsalikhovaa/demo2025/blob/main/audb.png"/>
</p>

```
nano 0.db
```

<p align="center">
  <img src="https://github.com/fsalikhovaa/demo2025/blob/main/0db.png"/>
</p>

```
systemctl restart bind
```

Проверка:

```
host hq-rtr.au-team.irpo
```
Должен выдать IP-адрес

## МОДУЛЬ 2
### Настройте доменный контроллер Samba на машине HQ-SRV
Произведём временное отключение интерфейсов. Обязательно перед началом настройки samba!
nmtui
```
grep -q 'bind-dns' /etc/bind/named.conf || echo 'include "/var/lib/samba/bind-dns/named.conf";' >> /etc/bind/named.conf
nano /etc/bind/options.conf
```

<p align="center">
  <img src=""/>
</p>

```
systemctl stop bind
```

<p align="center">
  <img src=""/>
</p>

```
nano /etc/sysconfig/network
```
Прописываем полное доменное имя хоста 

<p align="center">
  <img src=""/>
</p>

```
hostnamectl set-hostname hq-srv.au-team.irpo;exec bash
domainname au-team.irpo
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
rm -rf /var/cache/samba
mkdir -p /var/lib/samba/sysvol
samba-tool domain provision BIND9_DLZ, P@ssw0rd
```

Проверяем есть ли необходимые записи в файле /etc/resolv.conf и в /etc/krb5.conf 

<p align="center">
  <img src=""/>
</p>

<p align="center">
  <img src=""/>
</p>

Заходим в файл /etc/bind/named.conf и комментим в нем строку как на фото (!)

<p align="center">
  <img src=""/>
</p>

```
systemctl enable --now samba
systemctl enable --now bind
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
samba-tool domain info 127.0.0.1
```
Проверка:
```
host -t SRV _kerberos._udp.au-team.irpo.
host -t SRV _ldap._tcp.au-team.irpo.
host -t A hq-srv.au-team.irpo
```

```
kinit administrator@AU-TEM.IRPO
```
После этой команды выйдет приглашение на ввод пароля, вводим P@ssw0rd, поскольку раннее мы задавали его

### Ввод машины в домен
### HQ-CLI:
В параметрах сети указываем днс 172.16.0.2 и поисковой домен au-team.irpo

Открываем терминал
```
acc
```

<p align="center">
  <img src=""/>
</p>

Вводим пароль администратора P@ssw0rd

<p align="center">
  <img src=""/>
</p>

