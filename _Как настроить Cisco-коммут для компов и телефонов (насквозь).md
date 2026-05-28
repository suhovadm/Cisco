Как настроить Cisco-коммут для компов и телефонов (насквозь)?  

_____ ЛОГИКА VLAN _____  
VLAN		НАЗНАЧЕНИЕ  
10		DATA (ПК)  
20		VOICE (IP-телефоны)  
999		Native (blackhole, untagged на trunk)  

ПОГНАЛЕ!  

1. РЕЖИМ КОНФИГА.  
en		# enable - вход в привилегированный режим (полный доступ)  
conf t 		# configure terminal - режим настройки устройства  

2. RSTP  
spanning-tree mode rapid-pvst 	# включаем Rapid STP - защита от L2-петель + бьстрое восстановление  

3. QoS (опционально)  
mls qos		# включаем QoS - готовим коммутатор к приоритизации трафика (особенно голос)  

4. LLDP (опционально)  
lldp run 	# включаем LLDP - устройства могут "видеть" друг друга и передавать инфу  

5. VLAN  
vlan 10		# создаём VLAN 10  
name DATA	# имя для VLAN 10  

vlan 20		# создаём VLAN 20  
name VOICE	# имя для VLAN 20  

vlan 999	# native VLAN (неиспользуемый VLAN, для безопасности)  
name NATIVE  

6. ПОРТЫ К ПК + IP-ТЕЛЕФОНАМ  
int range gi0/2-48	# выбираем диапазон пользовательских портов (ПК + телефоны)  
sw mode acc		# switchport mode access - переводим все порты в access, т.е. как конечные устройства  
sw acc vlan 10		# все untagged (ПК) уходят в VLAN 10  
sw voice vlan 20	# весь телефонный трафик получит VLAN 20 через CDP/LLDP (если поддерживается)  
sw nonegotiate		# отключаем DTP - запрет автопереговоров trunk (безопасность)  
spanning-tree portfast	# порт сразу становится активным (без задержек STP)  
spanning-tree bpduguard enable	# если подключат свитч - порт отключится (защита от петель)  
mls qos trust cos 	# доверяем приоритеру (CoS), который выставляет IP-телефон  
no sh			# поднять интерфейс  

7. ТРАНК К РОУТЕРУ  
int gi0/1		# аплинк к маршрутизатору  
desc TO_ROUTER		# описание порта. Можно с пробелами, можно - без  

sw mode trunk		# включаем trunk - передаём несколько VLAN  
sw nonegotiate		# отключаем DTP (фиксированный trunk)  

sw trunk native vlan 999	# native VLAN (untagged трафик)  
sw trunk allowed vlan 10,20,999	# разрешаем только нужные VLAN  

no sh				# включаем порт  
________________________________________________________________________________________________________________  

ТЕПЕРЬ САМ МАРШРУТИЗАТОР.  

1. ВХОД В КОНФИГ.  
en                      # enable — привилегированный режим  
conf t                  # configure terminal — режим настройки  

2. ИСКЛЮЧЕНИЕ IP-АДРЕСОВ.  
ip dhcp excluded-address 192.168.10.1 192.168.10.20 # резервируем диапазон IP в 10-м VLAN-e. Эти адреса НЕ будут выдаваться ПК.  
ip dhcp excluded-address 192.168.20.1 192.168.20.20 # резервируем диапазон IP в 20-м VLAN-e. Эти адреса НЕ будут выдаваться телефонам.  
#Это, как правило, всякое сетевое оборудование, сервера, СХД-шки, шлюз и т.д. и т.п.  

3. ФИЗИЧЕСКИЙ ИНТЕРФЕЙС.  
interface g0/0         # выбираем физический интерфейс к свитчу  
 no shutdown           # включаем интерфейс (обязательно для работы subinterfaces)  
 exit                  # выходим из интерфейса  

4. VLAN 10 (DATA)  
interface g0/0.10              			# создаём логический интерфейс для VLAN 10  
 encapsulation dot1q 10        			# привязываем VLAN 10 (802.1Q тегирование)  
 ip address 192.168.10.1 255.255.255.0   	# шлюз для ПК  
 exit                          			# выход из интерфейса  

5. VLAN 20 (VOICE)  
interface g0/0.20              			# логический интерфейс для VLAN 20  
 encapsulation dot1q 20        			# VLAN 20 через 802.1Q  
 ip address 192.168.20.1 255.255.255.0   	# шлюз для телефонов  
 exit                          			# выход  

6. DHCP POOL  
ip dhcp pool DATA				# Имя пула  
 network 192.168.10.0 255.255.255.0		# Идентификатор сети и маска  
 default-router 192.168.10.1			# Шлюз  
 dns-server 8.8.8.8				# DNS  
exit  

ip dhcp pool VOICE				# Имя пула  
 network 192.168.20.0 255.255.255.0		# Идентификатор сети и маска  
 default-router 192.168.20.1			# Шлюз  
 dns-server 8.8.8.8				# DNS  
 option 150 ip 192.168.20.1			# Говорит телефону, где искать tFTP сервер для прошивок, конфигов, всякого такого (критично для Cisco-телефонии)  
exit  

7. NATIVE VLAN 999  
interface g0/0.999             			# логический интерфейс native VLAN  
 encapsulation dot1q 999 native 		# native VLAN (untagged трафик с switch)  
 exit  

8. ПРОВЕРКА  
show ip interface brief        			# проверяем состояние интерфейсов  
