Модуль 1. 
Задание 1.
На всех устройства пишем команду: hostnamectl set-hostname (HOSTNAME); exec bash 
Пишем без скобочек имя машины.

Задание 2. 
Заполняем таблицу адресации для сети:
|Имя устройства | IpV4       |   IpV6           |
|---------------|------------|------------------|
|CLI            | 3.3.3.2/29 |  fd11::2/125      |      
|ISP            | 3.3.3.1/29 |  fd11::1/125      |
|               | 4.4.4.1/29 |  fd22::1/125     |
|               | 5.5.5.1/29 |  fd33::1/125     |
|HQ-R	        | 4.4.4.2/29 |  fd22::2/125     |
|               | 192.168.1.1/29 | df22::1/125|
|HQ-SRV		| 192.168.1.2/29(DHCP)        | df22::2/125                 | 
|BR-R           | 5.5.5.2/29 | fd33::2/125      |
|               | 172.16.1.1/29 | df33::1/125 | 
|BR-SRV         | 172.16.1.2/29 | df33::2/125 |

Команда для назначения IP: ip addr add 192.168.1.35/24 dev ens0

chmod backup_script.sh

Apt install bind9 (mdam или mdarm вроде так)

Для Branch - 255.255.255.248/29		
Для HQ - 255.255.255.248/29

apt-cdrom add
--------------------------------------------------------------------------------
--------------------------------------------------------------------------------
Туннели:
---------------------------------------------------------------------------------
HQ-R
------
nano /etc/network/interfaces

auto gre1

iface gre1 inet tunnel

address 10.10.10.1

netmask 255.255.255.252

mode gre 

local 4.4.4.2

endpoint 5.5.5.2

ttl 65 

BR-R
---------
nano /etc/network/interfaces

auto gre1

iface gre1 inet tunnel

address 10.10.10.2

netmask 255.255.255.252

mode gre

local 5.5.5.2

endpoint 4.4.4.2

ttl 65 




Задание 2. Настройка маршрутизации.
----------------------------------------------------------------------------------Настройку динамической маршрутизации производим с помощью протокола ospf
Данный протокол позволяет разделить сеть на логическое области, что делает его масштабируемым для больших сетей
Каждая область может иметь свою таблицу маршрутизации, что уменьшает нагрузку маршрутизатора и улучшает производительность.

HQ-R (FRR)
------------
apt install frr -y

nano /etc/frr/daemons

ospfd=no меняем на ospfd=yes

Выходим с nano (ctr+s и ctr+x)
----------------------------------
systemctl enable --now frr

vtysh

conf t

router ospf

passive-interface default

network 192.168.1.0/29 area 0

network 10.10.10.0/29 (30) area 0

exit

int tun1

no ip ospf network broadcast

no ip ospf passive

exit

dp write memory

exit

Временно выключаем сервис служб firewall
--------------------------------------------
systemctl stop firewalld service

systemctl disable —now firewalld.service

systemctl restart frr


BR-R(frr)
-------------
apt install frr -y

nano /etc/frr/daemons

ospfd=no меняем на ospfd=yes

systemctl enable --now frr

vtysh

conf t

router ospf

passive-interface default

network 172.16.1.0/29 area 0

network 10.10.10.0/29 (30) area 0

exit

int tun1

no ip ospf network broadcast

no ip ospf passive

exit

do write memory

exit

systemctl stop firewalld.service

systemctl restart frr

Проверка frr:
----------------
vtysh show ip ospf neighbor

P.s если ospf не заработал можно перезапустить маршрутизаторы


Задание 3. Настройка DHCP
--------------------------------------
--------------------------------------

 DHCP-server:
---------------
apt install isc-dhcp-server -y

nano /etc/default/isc-dhcp-server

В строке с интерфейсами 4-ой версии пишем внутренний порт роутера(что подключается к серверу) (по идее 224, это я от себя добавил)

nano /etc/dhcp/dhcpd.conf

пишем следующую конфигурацию:

option domain-name "HQ-R";

option domain-name-servers 192.168.100.1;

default-lease-time 6000;

max-lease-time 72000;

authoritative;

log-facility local7;

subnet 192.168.1.0 netmask 255.255.255.248 {

range 192.168.1.1 192.168.1.14;

option routers 192.168.1.1;

}

host HQ-SRV {

hardware ethernet <MAC адресс сервера>

fixed-address 192.168.1.2;

}

systemctl enable --now isc-dhcp-server.service

dhcp-lease-list (если не показывается присвоенный адрес, перезапустить сеть на HQ-srv)

systemctl restart isc-dhcp-server.service (Если не работает сервер)

Задание 4. Создание пользователей
-----------------------------------------

useradd <имя>

passwd <имя> и пишем пароль

Задание 5 IPerf
----------------------------------------
Скачать Iperf3 (apt install iperf3) или (apt -y install iperf3)

На ISP (не останавливать запрос)
-------------------------------
iperf3 -s

На HQ-R
----------------
iperf3 -c 4.4.4.1

 При измерении пропускной способности между двумя узлами HQ-R - ISP реальная скорость составит <предпоследний столбец последней строки> Gbits/sec при переданных <transfer> Gb данных.(это пишется в ворде)
——————------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



Задание 6. Создание бэкап файла
-----------------------------------------
На HQ-R 
------------

mkdir backup

nano backup_script.sh

В самом файле пишем:
---------------------------
#!/bin/sh

tar-czf ~ /backup/net_conf.tar.xz /etc/network/interfaces

Выходим с nano (ctr+S и ctr+x)
------------------------------------
./backup_script.sh

ls -l backup

tar -tf ./backup/net_conf.br.xz

/less (опционально)

На BR-R тоже самое
--------------------
		
Задание номер 7. SSH
----------------------------------------------------
На HQ-srv:
-----------------------------------
nano /etc/ssh/sshd.config

меняем порт на 2020

Выходим с nano
--------------------------------
systemctl restart sshd

На HQ-R
------------------
nft add table inet nat

nft add chain inet nat prerouting '{type nat hook prerouting priority 0;}'

nft add rule inet nat prerouting

ip daddr 4.4.4.2 tcp dport 20 dnat to 192.168.1.2:2020

nft list ruleset \tail -n 7 \tee -a /etc/nftables/nftables.nft

systemctl enable --now nftables

nft list ruleset

Проверка подключения: на любой машине пишем:
---------------------------------------------------
ssh admin@192.168.1.2	

![Новый точечный рисунок](https://github.com/Ornihex/demo228/assets/172167032/b85d2e27-c0ea-4632-b4b0-4dc9bba64a96)

![image](https://github.com/Ornihex/demo228/assets/172167032/336e5842-1d0a-4e14-bb54-34717a528a38)

![image](https://github.com/Ornihex/demo228/assets/172167032/39485737-cf4c-4611-b912-e1521b80befa)

