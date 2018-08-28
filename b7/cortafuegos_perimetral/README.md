# Añadir estados al cortafuegos perimetral

Añadimos a continuación estados permitidos al cortafuegos perimetral,
lo haremos con el módulo state, en lugar de directamente con
conntrack.

## Configuración en un solo paso


```
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
iptables -A OUTPUT -o br0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i br0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A INPUT -i virbr1 -p icmp -s 192.168.100.0/24 -j ACCEPT
iptables -A OUTPUT -o virbr1 -p icmp -d 192.168.100.0/24 -j ACCEPT
iptables -A INPUT -i virbr2 -p icmp -s 192.168.200.0/24 -j ACCEPT
iptables -A OUTPUT -o virbr2 -p icmp -d 192.168.200.0/24 -j ACCEPT
iptables -A FORWARD -i virbr1 -o virbr2 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -o virbr1 -i virbr2 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A FORWARD -i virbr2 -o virbr1 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -o virbr2 -i virbr1 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A FORWARD -i virbr1 -o br0 -s 192.168.100.0/24 \
 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -o virbr1 -i br0 -d 192.168.100.0/24 \
-p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -i virbr2 -o br0 -s 192.168.200.0/24 \
 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -o virbr2 -i br0 -d 192.168.200.0/24 \
 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -i virbr1 -o br0 -s 192.168.100.0/24 \
-p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -o virbr1 -i br0 -d 192.168.100.0/24 \
-p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -i virbr1 -o br0 -s 192.168.100.0/24 \
-p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -o virbr1 -i br0 -d 192.168.100.0/24 \
-p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -i br0 -o virbr2 -p tcp --syn --dport 25 \ 
-m connlimit --connlimit-above 2 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -i br0 -o virbr2 -p tcp --syn --dport 80 \
-m connlimit --connlimit-above 15 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -i br0 -o virbr2 -p tcp --syn --dport 443 \
-m connlimit --connlimit-above 15 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -o virbr2 -d 192.168.200.2/32 -p tcp --dport 80 \
-m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i virbr2 -s 192.168.200.2/32 -p tcp --sport 80 \
-m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -o virbr2 -d 192.168.200.2/32 -p tcp --dport 443 \
-m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i virbr2 -s 192.168.200.2/32 -p tcp --sport 443 \
-m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -o virbr2 -d 192.168.200.2/32 -p tcp --dport 25 \
-m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i virbr2 -s 192.168.200.2/32 -p tcp --sport 25 \
-m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -i virbr2 -o virbr1 -s 192.168.200.2/32 \
-d 192.168.100.2/32 -p tcp --dport 3306 \
-m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -o virbr2 -i virbr1 -d 192.168.200.2/32 \
-s 192.168.100.2/32 -p tcp --sport 3306 \
-m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -i virbr2 -o br0 -p udp --dport 53 -m multiport \
--sports 1024:65535 -s 192.168.200.0/24  -m time \
--timestart 12:00 --timestop 12:30 -j ACCEPT
iptables -A FORWARD -o virbr2 -i br0 -p udp --sport 53 -m multiport \
--dports 1024:65535 -d 192.168.200.0/24  -m time \
--timestart 12:00 --timestop 12:30 -j ACCEPT
iptables -A FORWARD -i virbr2 -o br0 -p tcp --dport 80 -m multiport \
--sports 1024:65535 -s 192.168.200.0/24  -m time \
--timestart 12:00 --timestop 12:30 -j ACCEPT
iptables -A FORWARD -o virbr2 -i br0 -p tcp --sport 80 -m multiport \
--dports 1024:65535 -d 192.168.200.0/24  -m time \
--timestart 12:00 --timestop 12:30 -j ACCEPT
iptables -A FORWARD -i virbr2 -o br0 -p tcp --dport 443 -m multiport \
--sports 1024:65535 -s 192.168.200.0/24  -m time \
--timestart 12:00 --timestop 12:30 -j ACCEPT
iptables -A FORWARD -o virbr2 -i br0 -p tcp --sport 443 -m multiport \
--dports 1024:65535 -d 192.168.200.0/24  -m time \
--timestart 12:00 --timestop 12:30 -j ACCEPT
# Añadimos aquí las reglas de NAT
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o br0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.200.0/24 -o br0 -m time \
--timestart 12:00 --timestop 12:30 -j MASQUERADE
iptables -t nat -A PREROUTING -i br0 -p tcp --dport 80 -j DNAT --to 192.168.200.2
iptables -t nat -A PREROUTING -i br0 -p tcp --dport 443 -j DNAT --to 192.168.200.2
iptables -t nat -A PREROUTING -i br0 -p tcp --dport 25 -j DNAT --to 192.168.200.2
```

