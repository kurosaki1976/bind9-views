# Configuración de un servidor DNS Bind9 con vistas en Debian

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## Resumen

El objetivo de esta guía es mostrar cómo configurar un servidor `DNS Bind9` que brinde información distinta tanto a redes privadas como públicas, mediante el uso de la funcionalidad de vistas (`views`). Como tutorial en sí, se le guiará a través de todo el proceso de configuración, pero se requieren conocimientos iniciales de `DNS` y `Bind9`. En las [referencias](#referencias), encontrará enlaces a sitios de `Internet` que le pueden ayudar.

## Escenario

Es común encontrarnos entornos de red, donde se necesite que un mismo servidor de nombres `DNS` devuelva registros tanto de tipo canónico como direcciones `IP`, dependiendo de la red desde donde se originen las consultas. Por ejemplo:

Existencia de un servidor `DNS` dentro del direccionamiento `TCP/IP` de la subred de servicios, o cómo se le conoce comúnmente red `DMZ`, que da servicio a redes públicas (`Internet`, la red externa del provedor `ISP` o la `VPN` externa de una organización) y redes privadas (`Intranet` o red `LAN`, una `DMZ`, una `VPN` interna, o a todas ellas). Si se realiza la consulta desde el exterior, deben devolverse los registros públicos; sin embargo, si se hace la misma consulta desde cualquiera de las subredes internas, la resolución deberá ser a un registro privado. Esto es posible lograrlo, gracias a la funcionalidad de vistas (`views`) en `Bind9`.

## Administración del servidor `Bind9 DNS`

El servidor `DNS` de ejemplo utilizá los siguientes parámetros de configuración de red:

* Dirección `IP` del servidor: `192.168.0.1`
* Dominio `DNS`: `example.tld`
* `FQDN` del servidor: `ns.example.tld`
* Subred interna de la zona de servicios: `192.168.0.0/24`
* Subred externa para registros públicos: `172.16.0.0/29`

### Instalación

```
apt install bind9 dnsutils
```

Para disponer de la documentación `off-line`, instalar además `bind9-doc`.

### Configuración

En las distribuciones `Debian GNU/Linux`, los ficheros de configuración del paquete `bind9`, se encuentran en `/etc/bind`. Ellos son: `named.conf` (fichero de configuración principal), `named.conf.default-zones` (contiene las zonas predefinidas de reenvío (`forward`), inversa (`reverse`) y difusión (`broadcast`) para el `localhost`), `named.conf.options` (contiene todos los parámetros para la operación del servicio), y `named.conf.local` (contiene las opciones de configuración y las declaraciones de zonas del servidor `DNS` local).

* Nota: Es importante leer el archivo `/usr/share/doc/bind9/README.Debian.gz` antes de realizar modificaciones en los ficheros mencionados en el párrafo anterior, para obtener información sobre la estructura de los archivos de configuración del servicio `BIND` en `Debian`. De igual forma es una buena práctica realizar copias de seguridad de esos ficheros en su estado por defecto. 

```
cp /etc/bind/named.conf{,.org}
nano /etc/bind/named.conf

include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/rndc.key";
```

## Conclusiones

## Referencias
* [Bind9 - Debian Wiki](https://wiki.debian.org/Bind9)
* [Understanding views in BIND 9, by example](https://kb.isc.org/docs/aa-00851)
* [Internet Systems Consortium](https://www.youtube.com/user/ISCdotorg/videos)
* [Two-in-one DNS server with BIND9](https://www.howtoforge.com/two_in_one_dns_bind9_views)
* [Vistas (views) en el servidor DNS Bind9 ](https://www.josedomingo.org/pledin/2017/12/vistas-views-en-el-servidor-dns-bind9/)
* [Bind9 como DNS con delegación de zona](https://www.sysadminsdecuba.com/2018/04/bind9-como-dns-con-delegacion-de-zona/)
