# Configuración de un servidor DNS Bind9 con vistas en Debian

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## Resumen

El objetivo de esta guía es mostrar cómo configurar un servidor `DNS Bind9` que brinde información distinta tanto a redes privadas como públicas, mediante el uso de la funcionalidad de vistas (`views`). Como tutorial en sí, se le guiará a través de todo el proceso de configuración, pero se requieren conocimientos iniciales de `DNS` y `Bind9`. En las [referencias](#referencias), encontrará enlaces a sitios de `Internet` que le pueden ayudar.

## Escenario

Es común encontrarnos entornos de red, donde se necesite que un mismo servidor de nombres `DNS` devuelva registros tanto de tipo canónico como direcciones `IP`, dependiendo de la red desde donde se originen las consultas. Por ejemplo:

Existencia de un servidor `DNS` dentro del direccionamiento `TCP/IP` de la subred de servicios, o cómo se le conoce comúnmente red `DMZ`, que da servicio a redes públicas (`Internet`, la red externa del provedor `ISP` o la `VPN` externa de una organización) y redes privadas (`Intranet` o red `LAN`, una `DMZ`, una `VPN` interna, o a todas ellas). Si se realiza la consulta desde el exterior, deben devolverse los registros públicos; sin embargo, si se hace la misma consulta desde cualquiera de las subredes internas, la resolución deberá ser a un registro privado. Esto es posible lograrlo, gracias a la funcionalidad de vistas (`views`) en `Bind9`.

## Administración del servidor

El servidor `DNS` de ejemplo utilizá los siguientes parámetros de configuración de red:

* Dirección `IP` del servidor: `192.168.0.1`
* Dominio `DNS`: `example.tld`
* `FQDN` del servidor: `ns.example.tld`
* Subred interna de la zona de servicios: `192.168.0.0/24`
* Subred externa para registros públicos: `172.16.0.0/29`

### Instalación

```bash
apt install bind9 dnsutils
```

Para disponer de la documentación `off-line`, instalar además `bind9-doc`.

### Configuración

En las distribuciones `Debian GNU/Linux`, los ficheros de configuración del paquete `bind9`, se encuentran en `/etc/bind`. Ellos son: `named.conf` (fichero de configuración principal), `named.conf.default-zones` (contiene las zonas predefinidas de reenvío (`forward`), inversa (`reverse`) y difusión (`broadcast`) para el `localhost`), `named.conf.options` (contiene todos los parámetros para la operación del servicio), y `named.conf.local` (contiene las opciones de configuración y las declaraciones de zonas del servidor `DNS` local).

> **NOTA**: Es importante leer el archivo **`/usr/share/doc/bind9/README.Debian.gz`**, para obtener información sobre la estructura de los archivos de configuración del servicio `BIND` en `Debian`. De igual forma es una buena práctica realizar copias de seguridad de los ficheros mencionados en el párrafo anterior en su estado por defecto, **ANTES** de realizar modificaciones.

Editar fichero de configuración principal.

```bash
cp /etc/bind/named.conf{,.org}
nano /etc/bind/named.conf

include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.log";
include "/etc/bind/rndc.key";
```

Editar parámetros para la operación del servicio.

```bash
cp /etc/bind/named.conf.options{,.org}
nano /etc/bind/named.conf.options

options {
	version none;
	directory "/var/cache/bind";
	dnssec-enable yes;
	dnssec-validation yes;
	auth-nxdomain no;
	interface-interval 0;
	listen-on { 127.0.0.1; 192.168.0.1; };
	listen-on-v6 { none; };
	allow-query { any; };
	empty-zones-enable no;
	forwarders { 8.8.8.8; 8.8.4.4; };
	forward first;
	recursion no;
	prefetch 0;
	pid-file "/var/run/named/named.pid";
	session-keyfile "/var/run/named/session.key";
	minimal-responses yes;
	max-cache-size 10m;
	cleaning-interval 15;
	max-cache-ttl 60;
	max-ncache-ttl 60;
	flush-zones-on-shutdown yes;
};

controls {
	inet 127.0.0.1 port 953
	allow { localhost; 192.168.0.1; } keys { rndc-key; };
};
```

Definir opciones de configuración y declaraciones de zonas del servidor `DNS`.

```bash
cp /etc/bind/named.conf.local{,.org}
nano /etc/bind/named.conf.local
```

Crear fichero para almacenamiento de trazas, bitácora de eventos.

```bash
nano /etc/bind/named.conf.log

logging {
	channel "named-query" {
		file "/var/log/named_query.log" versions 3 size 5m;
		severity debug 3;
		print-time yes;
	};
	channel "debug" {
		file "/var/log/named.log" versions 2 size 3m;
		print-time yes;
		print-category yes;
	};
	category "queries" { "named-query"; };
	category "client" { "debug"; };
	category "config" { "debug"; };
	category "database" { "debug"; };
	category "default" { "debug"; };
	category "dispatch" { "debug"; };
	category "dnssec" { "debug"; };
	category "general" { "debug"; };
	category "lame-servers" { "debug"; };
	category "network" { "debug"; };
	category "notify" { "debug"; };
	category "resolver" { "debug"; };
	category "security" { "debug"; };
	category "unmatched" { "debug"; };
	category "update" { "debug"; };
	category "xfer-in" { "debug"; };
	category "xfer-out" { "debug"; };
};
```

## Conclusiones

## Referencias
* [Bind9 - Debian Wiki](https://wiki.debian.org/Bind9)
* [Understanding views in BIND 9, by example](https://kb.isc.org/docs/aa-00851)
* [Internet Systems Consortium](https://www.youtube.com/user/ISCdotorg/videos)
* [Two-in-one DNS server with BIND9](https://www.howtoforge.com/two_in_one_dns_bind9_views)
* [Vistas (views) en el servidor DNS Bind9 ](https://www.josedomingo.org/pledin/2017/12/vistas-views-en-el-servidor-dns-bind9/)
* [Bind9 como DNS con delegación de zona](https://www.sysadminsdecuba.com/2018/04/bind9-como-dns-con-delegacion-de-zona/)
