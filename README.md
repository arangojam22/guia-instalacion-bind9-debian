<summary><h2><b> 💡BIND9-DEBIAN </b></h2></summary>

# Guía de instalación y configuración de BIND9 en Debian

Esta guía documenta una práctica académica de instalación, configuración, diagnóstico y validación de un servidor DNS BIND9 en Debian 13 dentro de VirtualBox.

> **Entorno de laboratorio:** las direcciones IP privadas, dominios y configuraciones que aparecen en esta documentación pertenecen exclusivamente a una máquina virtual utilizada en un laboratorio académico con VirtualBox. No representan infraestructura pública ni productiva.

## Objetivo

Configurar un servidor DNS autoritativo para la zona `haven.local`, con zona directa e inversa, y comprobar la resolución del nombre `jam.haven.local`.

## Tecnologías y herramientas

- Debian 13
- BIND9
- VirtualBox
- `named-checkzone`
- `nslookup`
- `ss`
- Nano

## Configuración inicial de la zona inversa

La práctica comenzó con una configuración de zona inversa para la red privada `192.168.6.0/24`. El archivo `db.6.168.192` contenía el registro PTR para asociar la dirección del servidor con `ns.haven.local`.

![Configuración inicial de la zona inversa](docs/images/Captura%20de%20pantalla%202026-10-01%20172223.png)

## Incidencia: consultas a la IP inicial

Las primeras consultas se realizaron contra `192.168.6.100`. La respuesta fue un tiempo de espera, lo que indicó que el servicio no estaba accesible en esa dirección configurada inicialmente. Posteriormente se ajustó el laboratorio a la red privada `192.168.1.0/24`.

![Timeout de consultas a la IP inicial](docs/images/Captura%20de%20pantalla%202026-10-01%20172056.png)

## Diagnóstico: BIND9 no escuchaba en el puerto 53

La comprobación con `ss -lntup | grep ':53'` mostró que no había ningún servicio DNS escuchando en el puerto estándar 53. Solo aparecía el puerto 5353, utilizado por mDNS. Por eso las consultas con `nslookup` devolvían `connection refused`.

![Diagnóstico del puerto 53](docs/images/Captura%20de%20pantalla%202026-10-01%20172322.png)

## Corrección de named.conf.options

Se actualizó `/etc/bind/named.conf.options` para que BIND9 escuchara en la IP privada final del laboratorio, `192.168.1.100`. También se configuró una ACL denominada `safeclients`, se limitaron las consultas a los clientes autorizados y se definieron servidores reenviadores.

![Configuración de named.conf.options](docs/images/Captura%20de%20pantalla%202026-10-01%20172416.png)

## Verificación del servicio DNS

Después de aplicar la configuración, BIND9 quedó escuchando en `192.168.1.100:53`. La consulta de `ns.haven.local` respondió correctamente, lo que confirmó que el servidor DNS ya estaba operativo. La consulta de `www.haven.local` devolvió `NXDOMAIN` porque ese registro no se había definido en la zona; esto no indicaba un fallo del servicio.

![BIND9 escuchando en el puerto 53](docs/images/Captura%20de%20pantalla%202026-10-01%20172427.png)

## Validación final

Tras corregir la configuración, se actualizó el serial de la zona de `2` a `3`, se validó la zona con `named-checkzone` y se reinició BIND9. La prueba final confirmó la resolución correcta:

```text
jam.haven.local → 192.168.1.100
```

![Resultado final de la resolución DNS](docs/images/Captura%20de%20pantalla%202026-10-01%20172508.png)

## Comandos de verificación utilizados

```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo systemctl restart bind9
sudo systemctl status bind9 --no-pager
sudo ss -lntup | grep ':53'
nslookup ns.haven.local 192.168.1.100
nslookup jam.haven.local 192.168.1.100
```

## Aprendizajes

- Configuración de zonas DNS directa e inversa con BIND9.
- Uso de registros A, NS, SOA y PTR.
- Importancia de mantener coherentes la red, los archivos de zona y `listen-on`.
- Diagnóstico de servicios de red mediante `ss`, `nslookup`, registros y validación de zonas.
- Incremento del serial de una zona después de realizar cambios.
