# Configuración de un servidor DNS Bind9 con vistas en Debian 9/10

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

### Ajustes de los parámetros de red

```bash
nano /etc/network/interfaces

auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.0.1/24
    gateway 192.168.0.254
    dns-nameservers 127.0.0.1
```

```bash
nano /etc/resolv.conf

domain example.tld
nameserver 127.0.0.1
```

### Instalación

```bash
apt install bind9 dnsutils
```

Para disponer de la documentación `off-line`, instalar además `bind9-doc`.

### Configuración

En las distribuciones `Debian GNU/Linux`, los ficheros de configuración del paquete `bind9`, se encuentran en `/etc/bind`. Ellos son: `named.conf` (fichero de configuración principal), `named.conf.default-zones` (contiene las zonas predefinidas de reenvío (`forward`), inversa (`reverse`) y difusión (`broadcast`) para el `localhost`), `named.conf.options` (contiene todos los parámetros para la operación del servicio), y `named.conf.local` (contiene las opciones de configuración y las declaraciones de zonas del servidor `DNS` local).

> **NOTA**: Es importante leer el archivo **`/usr/share/doc/bind9/README.Debian.gz`**, para obtener información sobre la estructura de los archivos de configuración del servicio `BIND` en `Debian`. De igual forma es una buena práctica realizar copias de seguridad de los ficheros mencionados en el párrafo anterior en su estado por defecto, **ANTES** de realizar modificaciones.

1. Editar fichero de configuración principal.

```bash
cp /etc/bind/named.conf{,.org}
nano /etc/bind/named.conf

include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.log";
include "/etc/bind/rndc.key";
```

2. Editar parámetros para la operación del servicio.

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

3. Definir opciones de configuración y declaraciones de zonas del servidor `DNS`.

```bash
cp /etc/bind/named.conf.local{,.org}
nano /etc/bind/named.conf.local

acl "INTRANET"  { 192.168.0.0/24; };
view "private" {
	match-clients { localhost; INTRANET; };
	recursion yes;
	allow-recursion { localhost; 192.168.0.0/24; };
	allow-recursion-on { localhost; 192.168.0.1; };
	include "/etc/bind/named.conf.default-zones";
	zone "example.tld" {
		type master;
		file "/etc/bind/db.example.tld_private";
		allow-query { localhost; INTRANET; };
		allow-update { none; };
		notify no;
	};
	zone "0.168.192.IN-ADDR.ARPA" {
		type master;
		file "/etc/bind/db.0.168.192.in-addr.arpa";
		allow-query { localhost; INTRANET; };
		allow-update { none; };
		notify no;
	};
};
view "public" {
	match-clients { !INTRANET; any; };
	recursion no;
	include "/etc/bind/named.conf.default-zones";
	zone "example.tld" {
		type master;
		file "/etc/bind/db.example.tld_public";
		allow-query { any; };
		allow-update { none; };
		notify no;
	};
	zone "0.16.172.IN-ADDR.ARPA" {
		type master;
		file "/etc/bind/db.0.16.172.in-addr.arpa";
		allow-query { any; };
		allow-update { none; };
		notify no;
	};
};
```

> **NOTA**: Cuando se utiliza las funcionalidad de vistas, todas las definiciones de zonas **TIENEN** que estar contenidas dentro de éstas, de lo contrario el servicio no iniciará y generará códigos de error. Es por ello que el fichero `/etc/bind/named.conf.default-zones` fue incluido en cada definición de vista y no en el fichero de configuración principal.

4. Crear ficheros de zonas.

El sistema `DNS` está compuesto por varios registros, conocidos como Registros de Recursos (`Resource Records` o `RR`, en Inglés), que definen la información en el sistema de nombres de dominio, tanto para resolución de nombre -conocida también como directa o canónica (conversión de nombre a dirección `IP`)-, como para resolución de nombre inversa (conversión de dirección `IP` a nombre).

- Zona directa para consultas públicas

```bash
nano /etc/bind/db.example.tld_public

;
; example.tld Public Forward Zone
;
$ORIGIN .
$TTL 604800
example.tld  IN  SOA ns.example.tld.  postmaster.example.tld. (
                2019103101  ; Serial
                600         ; refresh
                180         ; retry
                604800      ; expire
                3600        ; negative cache ttl
                )
;
        NS  ns.example.tld.
        A   172.16.0.2
        MX  10  mail.example.tld.
        TXT "v=spf1 ip4:172.16.0.3 a:mail.example.tld ~all"
;
$ORIGIN example.tld.
ns    IN  A   172.16.0.2
mail  IN  A   172.16.0.3
www   IN  A   172.16.0.4
jb    IN  A   172.16.0.5
;
smtp        IN  CNAME   mail
imap        IN  CNAME   mail
pop3        IN  CNAME   mail
webmail     IN  CNAME   mail
conference  IN  CNAME   jb
;
$ORIGIN _udp.example.tld.
$TTL 900    ; 15 minutes
_domain   IN  SRV    5 0 53 ns.example.tld.
;
$ORIGIN _tcp.example.tld.
$TTL 900    ; 15 minutes
_domain IN    SRV    5 0 53 ns.example.tld.
_http   IN    SRV    5 0 80 www.example.tld.
_https  IN    SRV    5 0 443 www.example.tld.
_smtp   IN    SRV   10 0 25 smtp.example.tld.
_smtps  IN    SRV   10 0 465 smtp.example.tld.
_imap   IN    SRV   10 0 143 imap.example.tld.
_imaps  IN    SRV   10 0 993 imap.example.tld.
_pop3   IN    SRV   10 0 110 pop3.example.tld.
_pop3s  IN    SRV   10 0 995 pop3.example.tld.
_submission  IN    SRV   10 0 587 pop3.example.tld.
$TTL 18000  ; 5 hours
_xmpp-client IN     SRV     5 0 5222 jb.example.tld.
_xmpp-server IN     SRV     5 0 5269 jb.example.tld.
;
$ORIGIN conference.example.tld.
$TTL 18000  ; 5 hours
_xmpp-server._tcp   IN  SRV     5 0 5269 jb.example.tld.
```

- Zona inversa para consultas públicas

```bash
nano /etc/bind/db.0.16.172.in-addr.arpa

;
; 0.16.172.in-addr.arpa Public Reverse Zone
;
$ORIGIN .
$TTL 604800
0.16.172.IN-ADDR.ARPA   IN  SOA ns.example.tld.  postmaster.example.tld. (
                2019103101  ; Serial
                600         ; refresh
                180         ; retry
                604800      ; expire
                3600        ; negative cache ttl
                )
;
        IN  NS  ns.example.tld.
;
$ORIGIN 0.16.172.IN-ADDR.ARPA.
2   IN  PTR ns.example.tld.
3   IN  PTR mail.example.tld.
        PTR smtp.example.tld.
        PTR imap.example.tld.
        PTR pop3.example.tld.
        PTR webmail.example.tld.
4   IN  PTR www.example.tld.
5   IN  PTR jb.example.tld.
```

- Zona directa para consultas privadas

```bash
nano /etc/bind/db.example.tld_private

;
; example.tld Public Forward Zone
;
$ORIGIN .
$TTL 604800
example.tld  IN  SOA ns.example.tld.  postmaster.example.tld. (
                2019103101  ; Serial
                600         ; refresh
                180         ; retry
                604800      ; expire
                3600        ; negative cache ttl
                )
;
        NS  ns.example.tld.
        A   192.168.0.1
        MX  10  mail.example.tld.
        TXT "v=spf1 ip4:192.168.0.2 a:mail.example.tld ~all"
;
$ORIGIN example.tld.
ns    IN  A   192.168.0.1
mail  IN  A   192.168.0.2
www   IN  A   192.168.0.3
jb    IN  A   192.168.0.4
;
smtp        IN  CNAME   mail
imap        IN  CNAME   mail
pop3        IN  CNAME   mail
webmail     IN  CNAME   mail
conference  IN  CNAME   jb
;
$ORIGIN _udp.example.tld.
$TTL 900    ; 15 minutes
_domain   IN  SRV    5 0 53 ns.example.tld.
;
$ORIGIN _tcp.example.tld.
$TTL 900    ; 15 minutes
_domain IN    SRV    5 0 53 ns.example.tld.
_http   IN    SRV    5 0 80 www.example.tld.
_https  IN    SRV    5 0 443 www.example.tld.
_smtp   IN    SRV   10 0 25 smtp.example.tld.
_smtps  IN    SRV   10 0 465 smtp.example.tld.
_imap   IN    SRV   10 0 143 imap.example.tld.
_imaps  IN    SRV   10 0 993 imap.example.tld.
_pop3   IN    SRV   10 0 110 pop3.example.tld.
_pop3s  IN    SRV   10 0 995 pop3.example.tld.
_submission  IN    SRV   10 0 587 pop3.example.tld.
$TTL 18000  ; 5 hours
_xmpp-client IN     SRV     5 0 5222 jb.example.tld.
_xmpp-server IN     SRV     5 0 5269 jb.example.tld.
;
$ORIGIN conference.example.tld.
$TTL 18000  ; 5 hours
_xmpp-server._tcp   IN  SRV     5 0 5269 jb.example.tld.
```

- Zona inversa para consultas privadas

```bash
nano /etc/bind/db.0.168.192.in-addr.arpa

;
; 0.168.192.in-addr.arpa Public Reverse Zone
;
$ORIGIN .
$TTL 604800
0.168.192.IN-ADDR.ARPA   IN  SOA ns.example.tld.  postmaster.example.tld. (
                2019103101  ; Serial
                600         ; refresh
                180         ; retry
                604800      ; expire
                3600        ; negative cache ttl
                )
;
        IN  NS  ns.example.tld.
;
$ORIGIN 0.168.192.IN-ADDR.ARPA.
1   IN  PTR ns.example.tld.
2   IN  PTR mail.example.tld.
        PTR smtp.example.tld.
        PTR imap.example.tld.
        PTR pop3.example.tld.
        PTR webmail.example.tld.
3   IN  PTR www.example.tld.
4   IN  PTR jb.example.tld.
```

5. Crear fichero para almacenamiento de trazas, bitácora de eventos.

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

6. Comprobar la existencia de errores tanto en la configuración como en los ficheros de zonas.

```bash
named-checkconf -z
named-checkzone example.tld /etc/bind/db.example.tld_public
named-checkzone 0.16.172.in-addr.arpa /etc/bind/db.0.16.172.in-addr.arpa
named-checkzone example.tld /etc/bind/db.example.tld_private
named-checkzone 0.168.192.in-addr.arpa /etc/bind/db.0.168.192.in-addr.arpa
```

7. Reiniciar el servicio y realizar comprobaciones.

```bash
systemctl restart bind9.service
tail -fn100 /var/log/syslog
```

- Desde una red pública.

```bash
dig example.tld
host example.tld
host ns.example.tld
host 172.16.0.2
dig -t SRV @example.tld _xmpp-server._tcp.example.tld
host -t SRV _imaps._tcp.example.tld
host -t MX example.tld
tail -fn100 /var/log/named_query.log
tail -fn100 /var/log/named.log
```

- Desde una red privada.

```bash
nslookup example.tld
host ns.example.tld
dig @127.0.0.1 -x 192.168.0.1
dig -t SRV @example.tld _xmpp-client._tcp.example.tld
host -t SRV _domain._udp.example.tld
host -t NS example.tld
tail -fn100 /var/log/named_query.log
tail -fn100 /var/log/named.log
```

## Conclusiones

Con la introducción en `Bind9` de la funcionalidad de vistas, otro mecanismo muy útil en entornos de red que brindan servicios detrás de cortafuegos, es posible presentar una configuración del servidor `DNS` distinta a varios dispositivos. Algo particularmente provechoso si se ejecuta un servidor que recibe consultas desde redes privadas y públicas como es el caso de `Internet`.

## Referencias

* [Bind9 - Debian Wiki](https://wiki.debian.org/Bind9)
* [Understanding views in BIND 9, by example](https://kb.isc.org/docs/aa-00851)
* [Internet Systems Consortium](https://www.youtube.com/user/ISCdotorg/videos)
* [Two-in-one DNS server with BIND9](https://www.howtoforge.com/two_in_one_dns_bind9_views)
* [Vistas (views) en el servidor DNS Bind9 ](https://www.josedomingo.org/pledin/2017/12/vistas-views-en-el-servidor-dns-bind9/)
* [Bind9 como DNS con delegación de zona](https://www.sysadminsdecuba.com/2018/04/bind9-como-dns-con-delegacion-de-zona/)
* [Limiting the Memory a Name Server Uses](https://flylib.com/books/en/2.684.1/limiting_the_memory_a_name_server_uses.html)
