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

3. QoS  
mls qos		# включаем QoS - готовим коммутатор к приоритизации трфика (особенно голос)  

4. LLDP  
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
sw voice vlan 20	# весь телефонный трафик пойдёт в VLAN 20 (tagged 802.1Q)  
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

ПОДКЛЮЧАЕМСЯ К РОУТЕРУ (МАРШРУТИЗАТОРУ).  

1. ВХОДИМ В КОНФИГУРАЦИЮ  
en                      # enable — переход в привилегированный режим  
conf t                  # configure terminal — вход в режим настройки  

2. STP (ЗАЩИТА ОТ ПЕТЕЛЬ)  
spanning-tree mode rapid-pvst    	# включаем Rapid PVST — быстрое восстановление и защита от L2 петель  
spanning-tree portfast default   	# автоматически включает PortFast на access-портах  
spanning-tree bpduguard default  	# автоматически отключает порт при получении BPDU (защита от подключенных свитчей)  

3. QoS  
mls qos         # включаем систему QoS — подготовка к приоритизации трафика (особенно голос)  

4. LLDP  
lldp run        # включаем LLDP — позволяет устройствам обмениваться информацией о соседях  

5. VLAN  
vlan 10         # создаём VLAN 10 (данные)  
 name DATA      # имя VLAN 10  

vlan 20         # создаём VLAN 20 (голос)  
 name VOICE     # имя VLAN 20  

vlan 999        # создаём VLAN 999 (native / неиспользуемый)  
 name NATIVE    # имя VLAN 999  

6. ACCESS ПОРТЫ (ПК + IP-телефоны)  
interface range gi0/2-48        # выбираем диапазон пользовательских портов  
 switchport mode access         # переводим порт в access режим (только один native VLAN)  
 switchport access vlan 10      # ПК попадает в VLAN 10 (DATA)  
 switchport voice vlan 20       # телефонный трафик уходит в VLAN 20 (VOICE через tagging)  

 switchport nonegotiate         # отключает DTP — защита от автоматического trunk  

 spanning-tree portfast         # порт сразу становится активным (без задержек STP)  
 spanning-tree bpduguard enable # отключает порт при обнаружении другого коммутатора (защита от петель)  
 
 mls qos trust cos              # доверяем CoS меткам IP-телефона (приоритет голосу)  

 no shutdown                    # включаем порт  
 exit                           # выходим из интерфейса  

7. ТРАНК К РОУТЕРУ  
interface gi0/1                 # выбираем порт к маршрутизатору  
 description TO_ROUTER          # описание интерфейса (для удобства)  

 switchport mode trunk          # включаем trunk (несколько VLAN через один порт)  
 switchport nonegotiate         # отключаем DTP (фиксированный trunk без автопереговоров)  

 switchport trunk native vlan 999     # native VLAN (untagged трафик)  
 switchport trunk allowed vlan 10,20,999  # разрешаем только нужные VLAN  

 no shutdown                    # включаем порт  
 exit                           # выходим из интерфейса  

_______________________________________________________________________________________________  

ТЕПЕРЬ САМ МАРШРУТИЗАТОР.  

1. ВХОД В КОНФИГ.  
en                      # enable — привилегированный режим  
conf t                  # configure terminal — режим настройки  

2. ФИЗИЧЕСКИЙ ИНТЕРФЕЙС.  
interface g0/0         # выбираем физический интерфейс к свитчу  
 no shutdown           # включаем интерфейс (обязательно для работы subinterfaces)  
 exit                  # выходим из интерфейса  

3. VLAN 10 (DATA)  
interface g0/0.10              			# создаём логический интерфейс для VLAN 10  
 encapsulation dot1q 10        			# привязываем VLAN 10 (802.1Q тегирование)  
 ip address 192.168.10.1 255.255.255.0   	# шлюз для ПК  
 exit                          			# выход из интерфейса  

4. VLAN 20 (VOICE)  
interface g0/0.20              			# логический интерфейс для VLAN 20  
 encapsulation dot1q 20        			# VLAN 20 через 802.1Q  
 ip address 192.168.20.1 255.255.255.0   	# шлюз для телефонов  
 exit                          			# выход  

5. NATIVE VLAN 999
interface g0/0.999             			# логический интерфейс native VLAN  
 encapsulation dot1q 999 native 		# native VLAN (untagged трафик с switch)  
 exit  
 
6. ПРОВЕРКА  
show ip interface brief        			# проверяем состояние интерфейсов  
