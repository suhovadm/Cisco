Как объединить порты с 2 по 48 в 10-й VLAN, в режиме ACCESS и включить RSTP-протокол?  

en # или enable  
conf t  

spanning-tree mode rapid-pvst  

vlan 10  
name VLAN10  
exit  

int range gi0/2-48  
switchport mode access  
switchport access vlan 10  
spanning-tree portfast  
exit  

end  
copy run start # или можно write memory  
