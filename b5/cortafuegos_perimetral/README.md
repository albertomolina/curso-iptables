# Añadir reglas de NAT al cortafuegos perimetral

Aunque las reglas de cortafuegos de la tabla filter permitan el acceso
a los servicios alojados en la DMZ, desde el exterior estos servicios
se solicitarán a la máquina que está conectada al exterior, esto es, a
la máquina que implementa el cortafuegos.

Además debemos habilitar SNAT para que los equipos tengan acceso a
servicios del exterior.

## Bit de forward

El primer paso común es habilitar el bit de forward mediante la
instrucción:

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

### Habilitar el forward de forma permanente

Esto último podemos hacerlo de forma permanente (que mantenga el
comportamiento después de reiniciar el equipo) habilitando la línea
"net.ipv4.ip_forward=1" del fichero /etc/sysctl.conf y ejecutando
posteriormente sysctl -p /etc/sysctl.conf

## SNAT

En este caso la regla de SNAT es:

```
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o wlan0 -j MASQUERADE
```

## DNAT

Creamos una regla de DNAT para cada servicio de la DMZ:

```
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 80 -j DNAT --to 192.168.200.2
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 443 -j DNAT --to 192.168.200.2
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 25 -j DNAT --to 192.168.200.2
```


