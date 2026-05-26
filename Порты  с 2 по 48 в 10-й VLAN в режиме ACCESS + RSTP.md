Как объединить порты с 2 по 48 в 10-й VLAN, в режиме ACCESS и включить RSTP-протокол?  

en # или enable  
conf t  
spanning-tree mode rapid-pvst  
vlan 10  
name VLAN10  
exit  
int range gi0/2-48  
switch mode access  
switch access vlan 10  
span tree portfast  
end  
copy run start # или можно write memory  
