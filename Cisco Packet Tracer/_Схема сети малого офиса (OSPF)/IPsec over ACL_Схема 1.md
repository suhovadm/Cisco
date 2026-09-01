[ Самый простой вариант настройки ipsec-a поверх OSPF? ]  
[ В данном случае, показан способ, как закрыть локальные сетки внутри OSPF-a, ]  
[ т.е. только ACL-ки. Сам OSPF, будет открыт, но эта схема вполне рабочая. ]  
[ В данной схеме, шифруется только тот трафик, который указан в ACL-ках. ]  
[ Чтобы зашифровать весь OSPF, нужно поднимать GRE over IPSec. ]  

// 5. Настраиваем IPsec между двумя роутерами.  
// Теперь настроим защищённый site-to-site IPsec-туннель между первым и вторым роутером.  
// Через него будут ходить данные между сетями слева и сетями справа.  
// Саму существующую маршрутизацию OSPF не меняем.  

// Сначала настраиваем IPsec на первом роутере слева, 2911.  

en  
conf t  

// Создаём ACL, которая определяет, какой трафик нужно шифровать.  
// Слева у нас две сети: 192.168.10.0/24 и 192.168.20.0/24.  
// Справа две сети: 192.168.30.0/24 и 192.168.40.0/24.  

access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255  
access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.40.0 0.0.0.255  
access-list 110 permit ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255  
access-list 110 permit ip 192.168.20.0 0.0.0.255 192.168.40.0 0.0.0.255  

// Включаем securityk9  
license boot module c2900 technology-package securityk9  
// Здесь печатаем yes, а НЕ ENTER!!!  
end  
copy running-config startup-config  
reload // отправляем Циску в перезагруз  

// После перезагруза - проверяем:  
en  
show license feature  
// Если в большинстве колонок написано "yes" - значит всё ок.  

// Теперь создаём ISAKMP policy.  
// Она определяет параметры первой фазы установления IPsec-соединения.  

en  
conf t  

crypto isakmp policy 10  
encryption aes  
hash sha  
authentication pre-share  
group 5  
lifetime 86400  
exit  

// Указываем общий пароль между двумя роутерами.  
// Пароль должен быть одинаковым на обоих роутерах.  
// 10.10.10.2 — внешний IP второго роутера.  

crypto isakmp key mysuperstrongpassword address 10.10.10.2  

// Создаём transform-set.  
// Он определяет, как именно будет шифроваться наш трафик.  

crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac  
exit  
// Здесь мы выходим в корень, поэтому нужно будет снова набрать:  

en  
conf t  

// Теперь создаём crypto map.  
// В нём связываем ACL, transform-set и адрес второго роутера.  

crypto map VPN-MAP 10 ipsec-isakmp  
set peer 10.10.10.2  
set transform-set VPN-SET  
match address 110  
exit  

// Теперь вешаем crypto map на интерфейс, через который соединены роутеры.  
// Это GigabitEthernet0/1.  

int g0/1  
crypto map VPN-MAP  
// При успешном выполнении, он должен написать:  
*Jan  3 07:16:26.785: %CRYPTO-6-ISAKMP_ON_OFF: ISAKMP is ON  
exit  

end  

// Сохраняем настройки.  
copy running-config startup-config  
enter  
 

// Теперь настраиваем IPsec на втором роутере справа, 2911.  

en  
conf t  

// ACL здесь должна быть зеркальной.  
// Теперь источником являются сети справа,  
// а назначением — сети слева.  

access-list 110 permit ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255  
access-list 110 permit ip 192.168.40.0 0.0.0.255 192.168.10.0 0.0.0.255  
access-list 110 permit ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255  
access-list 110 permit ip 192.168.40.0 0.0.0.255 192.168.20.0 0.0.0.255  

// Включаем securityk9  
license boot module c2900 technology-package securityk9  
// Здесь пишем yes, а не нажимаем ENTER !!!  
end  
copy running-config startup-config  
reload // отправляем Циску в перезагруз  

// После перезагруза - проверяем:  
en  
show license feature  
// Если в большинстве колонок написано "yes" - значит всё ок.  

// Создаём такую же ISAKMP policy,  
// как и на первом роутере.  

en  
conf t  

crypto isakmp policy 10  
encryption aes  
hash sha  
authentication pre-share  
group 5  
lifetime 86400  
exit  

// Здесь указываем тот же самый пароль,  
// но уже адрес первого роутера — 10.10.10.1.  

crypto isakmp key mysuperstrongpassword address 10.10.10.1  

// Создаём такой же transform-set.  

crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac  
exit  

// Здесь снова поднимаемся до преф-режима:  
en  
conf t  

// Создаём crypto map.  
// Здесь peer уже первый роутер — 10.10.10.1.  

crypto map VPN-MAP 10 ipsec-isakmp  
set peer 10.10.10.1  
set transform-set VPN-SET  
match address 110  
exit  

// Вешаем crypto map на интерфейс,  
// который смотрит в сторону первого роутера.  

int g0/1  
crypto map VPN-MAP  
// Если всё ок, должно появиться аналогичное сообщение об успехе: *Jan  3 07:16:26.785: %CRYPTO-6-ISAKMP_ON_OFF: ISAKMP is ON  
exit  

end  

// Сохраняем настройки.  
copy running-config startup-config  
enter  

// Теперь проверяем, поднялся ли IPsec.  
// На первом роутере вводим:  

show crypto isakmp sa  

// Если туннель поднялся и работает, то мы должны увидеть примерно вот такую картину:  

IPv4 Crypto ISAKMP SA  
dst             src             state          conn-id slot status  
10.10.10.2      10.10.10.1      QM_IDLE           1047    0 ACTIVE  

// Этот вывод означает, что IKE-соединение установлено.  
// Но это сработает при условии, что мы пингуем комп <---> комп, или комп <---> сервер.  
// Если мы будем пинговать маршрутизатор <---> маршрутизатор, или маршрутизатор <---> комп,  
// то ничего не заработает, в табличке будет пусто, т.к. данная реализация ipsec-a шифрует только ACL-ки.  

// Затем проверяем вторую фазу:  

show crypto ipsec sa  

// Здесь должны увеличиваться значения:  
// encaps  
// encrypt  
// decaps  
// decrypt  
// Это означает, что пакеты действительно проходят через IPsec.  

// Важно: IPsec-туннель может не появиться сразу после настройки.  
// Чтобы его запустить, с компьютера СПРАВА отправляем ping  
// на компьютер или сервер СЛЕВА.  

// Например, с ПК справа:  

ping 192.168.20.4  

// После первого ping снова проверяем:  

show crypto isakmp sa  
show crypto ipsec sa  

// Если всё настроено правильно, между двумя роутерами  
// будет установлен IPsec-туннель,  
// а трафик между сетями слева и справа будет шифроваться.  
