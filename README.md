# Guía de instalación y configuración de BIND9 en Debian 13

## Resumen

Práctica realizada en Debian GNU/Linux 13 (Trixie) dentro de Oracle VirtualBox. El objetivo fue instalar y configurar un servidor DNS primario con BIND9, crear una zona directa y una zona inversa, y resolver correctamente el nombre `jam.haven.local`.

Durante la práctica se produjeron varias incidencias relacionadas con la red, los permisos, las zonas inversas, el puerto DNS y las direcciones IP utilizadas en la guía del aula. Todas quedaron identificadas y resueltas.

## Entorno

- Sistema operativo: Debian GNU/Linux 13 (Trixie).
- Virtualización: Oracle VirtualBox.
- RAM: 2 GB.
- Disco virtual: 25 GB.
- Interfaces de red: `enp0s3` y `enp0s8`.
- Dominio interno: `haven.local`.
- Servidor DNS: `ns.haven.local`.
- Host final: `jam.haven.local`.
- IP final del servidor DNS: `192.168.1.100`.
- Puerto DNS: UDP/TCP 53.

## 1. Identificación de interfaces y rutas

Se consultaron las interfaces mediante:

```bash
ip -br addr
ip route
```

Configuración final:

```text
enp0s3 -> DHCP: 10.0.2.15/24
enp0s8 -> IP estática: 192.168.1.100/24
```

La salida a Internet quedó por `enp0s3`:

```text
default via 10.0.2.2 dev enp0s3
```

La red interna para el servidor DNS quedó en:

```text
192.168.1.0/24
```

## 2. Configuración de red estática

Archivo editado:

```bash
sudo nano /etc/network/interfaces
```

Configuración utilizada:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug enp0s3
iface enp0s3 inet dhcp

allow-hotplug enp0s8
iface enp0s8 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    broadcast 192.168.1.255
    dns-nameservers 192.168.1.100 1.1.1.1 9.9.9.9
```

Las interfaces se activaron con:

```bash
sudo ifup enp0s3
sudo ifup enp0s8
```

## 3. Pruebas de conectividad

Se probaron la puerta de enlace, Internet y la resolución DNS externa:

```bash
ping -c 4 10.0.2.2
ping -c 4 1.1.1.1
ping -c 4 google.com
```

Las tres pruebas respondieron correctamente y registraron un 0 % de pérdida de paquetes.

## 4. Instalación de BIND9

```bash
sudo apt update
sudo apt install bind9 bind9-utils dnsutils
sudo mkdir -p /etc/bind/zones
```

BIND9 se utiliza para ofrecer el servicio DNS. Las herramientas `named-checkconf`, `named-checkzone`, `dig` y `nslookup` se utilizaron para validar y probar la configuración.

## 5. Incidencia: se utilizó inicialmente la red 192.168.6.0/24

Al principio se utilizó la red `192.168.6.0/24` porque era la red que aparecía en la práctica del profesor. Por eso se configuraron temporalmente:

```text
IP: 192.168.6.100
Zona inversa: 6.168.192.in-addr.arpa
```

Sin embargo, la red real de la máquina virtual era:

```text
192.168.1.0/24
```

La IP correcta del servidor debía ser:

```text
192.168.1.100
```

Al realizar consultas contra `192.168.6.100`, aparecieron errores como:

```text
communications error: timed out
no servers could be reached
```

### Solución

Se sustituyeron las referencias a la red anterior:

```text
192.168.6.100 -> 192.168.1.100
6.168.192.in-addr.arpa -> 1.168.192.in-addr.arpa
```

También se renombró el archivo de zona:

```bash
sudo mv /etc/bind/zones/db.6.168.192 /etc/bind/zones/db.1.168.192
```

## 6. Incidencia: IP del aula sin sustituir

Después de corregir la red, el dominio todavía no se resolvía porque quedaba una dirección de la configuración del aula, `192.168.111.1`, en lugar de la IP real de nuestro servidor.

La dirección utilizada en la configuración no coincidía con la IP real de la máquina virtual:

```text
IP incorrecta de la configuración del aula: 192.168.111.1
IP correcta del servidor: 192.168.1.100
```

Por este motivo, las consultas no encontraban correctamente el dominio.

### Solución

Se revisaron y actualizaron todas las referencias de IP en:

- `/etc/network/interfaces`.
- `/etc/bind/named.conf.local`.
- `/etc/bind/named.conf.options`.
- `/etc/bind/zones/db.haven.local`.
- `/etc/bind/zones/db.1.168.192`.
- Consultas realizadas con `nslookup`.

Después de reemplazar `192.168.111.1` por `192.168.1.100`, el dominio se resolvió correctamente.

## 7. Configuración de named.conf.local

```bash
sudo nano /etc/bind/named.conf.local
```

Contenido final:

```text
zone "haven.local" {
    type master;
    file "/etc/bind/zones/db.haven.local";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.1.168.192";
};
```

La zona inversa `1.168.192.in-addr.arpa` corresponde a la red `192.168.1.0/24`, escribiendo los octetos de red en orden inverso.

## 8. Configuración de la zona directa

Archivo:

```bash
sudo nano /etc/bind/zones/db.haven.local
```

Contenido final:

```text
$TTL 604800
@       IN  SOA     haven.local. hostmaster.haven.local. (
                3       ; Serial
                12h     ; Refresh
                15m     ; Retry
                3w      ; Expire
                2h      ; Negative Cache TTL
)

@       IN  NS      ns.haven.local.
ns      IN  A       192.168.1.100
jam     IN  A       192.168.1.100
```

El registro:

```text
jam IN A 192.168.1.100
```

crea el nombre completo:

```text
jam.haven.local
```

## 9. Configuración de la zona inversa

Archivo:

```bash
sudo nano /etc/bind/zones/db.1.168.192
```

Contenido:

```text
$TTL 604800
@       IN  SOA     haven.local. hostmaster.haven.local. (
                2       ; Serial
                12h     ; Refresh
                15m     ; Retry
                3w      ; Expire
                2h      ; Negative Cache TTL
)

@       IN  NS      ns.haven.local.
100     IN  PTR     ns.haven.local.
```

El registro `100 IN PTR` representa la dirección `192.168.1.100`.

## 10. Configuración de named.conf.options

Archivo:

```bash
sudo nano /etc/bind/named.conf.options
```

Configuración final:

```text
acl "safeclients" {
    localhost;
    192.168.1.100;
    localnets;
};

options {
    directory "/var/cache/bind";

    listen-on port 53 { 127.0.0.1; 192.168.1.100; };
    listen-on-v6 { none; };

    recursion yes;
    allow-recursion { safeclients; };
    allow-query { safeclients; };
    allow-query-cache { safeclients; };
    allow-transfer { none; };

    forwarders {
        1.1.1.1;
        9.9.9.9;
    };

    dnssec-validation auto;
};
```

## 11. Incidencia: BIND no escuchaba en el puerto 53

En una comprobación inicial se observó actividad en el puerto `5353`, pero las consultas DNS se realizaban contra el puerto `53`. Como consecuencia, `nslookup` devolvía:

```text
connection refused
no servers could be reached
```

### Solución

Se configuró explícitamente el puerto DNS estándar:

```text
listen-on port 53 { 127.0.0.1; 192.168.1.100; };
```

Después se reinició BIND:

```bash
sudo systemctl restart bind9
```

Y se comprobó que escuchaba en el puerto correcto:

```bash
sudo ss -lntup | grep ':53'
```

Resultado relevante:

```text
192.168.1.100:53
```

## 12. Incidencias de comandos y permisos

### Comando `ip` escrito incorrectamente

Se escribió:

```bash
ip /br addr
```

El comando correcto es:

```bash
ip -br addr
```

### Comando `nano` escrito sin espacio

Se escribió:

```bash
nano/etc/network/interfaces
```

El comando correcto es:

```bash
nano /etc/network/interfaces
```

### Falta de permisos sudo

Al principio `vboxuser` no pertenecía al archivo sudoers. Se corrigió añadiendo el usuario al grupo `sudo` desde root:

```bash
usermod -aG sudo vboxuser
```

Después de cerrar sesión y volver a entrar, se verificó:

```bash
sudo whoami
```

Resultado:

```text
root
```

### Permiso denegado al renombrar una zona

Al intentar mover un archivo como usuario normal apareció:

```text
Permission denied
```

La solución fue utilizar `sudo`:

```bash
sudo mv /etc/bind/zones/db.6.168.192 /etc/bind/zones/db.1.168.192
```

### Uso de `su` en PowerShell

El comando `su` pertenece a Linux. Al ejecutarlo en PowerShell de Windows apareció un error porque PowerShell no reconoce ese comando.

La solución fue volver a la terminal de Debian desde VirtualBox. PowerShell se utilizó únicamente para pruebas externas o para una conexión SSH cuando el servicio estuviera configurado.

## 13. Validación final

Se comprobaron los archivos de configuración:

```bash
sudo named-checkconf
```

El comando no mostró errores.

Se validó la zona directa:

```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
```

Resultado:

```text
zone haven.local/IN: loaded serial 3
OK
```

Se validó la zona inversa:

```bash
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.1.168.192
```

Después se reinició el servicio:

```bash
sudo systemctl restart bind9
```

## 14. Pruebas finales de resolución

Servidor DNS:

```bash
nslookup ns.haven.local 192.168.1.100
```

Resultado:

```text
Name:    ns.haven.local
Address: 192.168.1.100
```

Dominio personal:

```bash
nslookup jam.haven.local 192.168.1.100
```

Resultado final:

```text
Name:    jam.haven.local
Address: 192.168.1.100
```

La resolución inversa también se probó con:

```bash
nslookup 192.168.1.100 192.168.1.100
```

## Resultado final

El servidor DNS quedó configurado correctamente con:

```text
Dominio: haven.local
Servidor DNS: ns.haven.local
Nombre configurado: jam.haven.local
IP final: 192.168.1.100
Red interna: 192.168.1.0/24
Zona inversa: 1.168.192.in-addr.arpa
Puerto DNS: 53
Servicio: BIND9 activo
```

La incidencia más importante fue mantener direcciones de la práctica del aula, primero `192.168.6.100` y después `192.168.111.1`, en lugar de utilizar la dirección real de la máquina virtual. Una vez sustituidas todas las referencias por `192.168.1.100`, el dominio `jam.haven.local` se resolvió correctamente.

## Comandos principales

```bash
ip -br addr
ip route
ping -c 4 10.0.2.2
ping -c 4 1.1.1.1
ping -c 4 google.com
sudo apt update
sudo apt install bind9 bind9-utils dnsutils
sudo mkdir -p /etc/bind/zones
sudo named-checkconf
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.1.168.192
sudo systemctl restart bind9
sudo systemctl status bind9 --no-pager
sudo ss -lntup | grep ':53'
nslookup ns.haven.local 192.168.1.100
nslookup jam.haven.local 192.168.1.100
nslookup 192.168.1.100 192.168.1.100
```

## Autor

Jam Luis Arango Jiménez

ASIR — Administración de Sistemas Informáticos en Red y Ciberseguridad
