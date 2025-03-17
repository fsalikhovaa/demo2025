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
  <img src="">
</p>

Настраиваем GRE через nmtui
### BR-RTR:

<p align="center">
  <img src="">
</p>

### HQ-RTR:
<p align="center">
  <img src="">
</p>

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
