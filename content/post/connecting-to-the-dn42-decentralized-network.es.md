---
title: "Conectándome a la red descentralizada DN42"
date: 2024-08-10T11:50:00-06:00
summary: "Conectándose a DN42, una red descentralizada que utiliza VPNs y BGP"
author: "Sebastian Marines"
categories: ["dn42", "networking", "vpn", "bgp"]
tags: ["dn42", "networking", "vpn", "bgp"]
---

Siempre me ha interesado cómo funciona internet, me sorprende cómo podemos comunicarnos con personas de todo el mundo en milisegundos, pero nunca habia entendido cómo funcionaba realmente. Hace unos meses me topé con [este](https://labs.ripe.net/author/samir_jafferali/build-your-own-anycast-network-in-nine-steps/) artículo que explicaba cómo construir tu propia red anycast, y leí sobre algunos conceptos que nunca había escuchado antes, como BGP y ASNs.

La red DN42 es una red descentralizada que utiliza VPNs y BGP para crear un "internet" sobre el internet. Esto significa que cualquiera que le interese aprender a usar las tecnologías que los proveedores de internet utilizan para crear el internet puede unirse y aprender cómo funciona sin necesidad de equipo costoso o interferir con el internet real [^1].

[^1]: [What is BGP, and what role did it play in Facebook’s massive outage](https://www.theverge.com/2021/10/4/22709260/what-is-bgp-border-gateway-protocol-explainer-internet-facebook-outage)

## Preparación

Antes de poder conectarnos a la red DN42, necesitamos registrar un Número de Sistema Autónomo (ASN), prefijos de red y rutas. Para esto, tenemos que crear un pull request al registro de DN42, donde registraremos el ASN que queremos usar, los prefijos de red y las rutas que queremos recibir. La [guía de inicio](https://dn42.dev/howto/Getting-Started) en el sitio web de DN42 explica cómo hacer esto en detalle.

El ASN es un número único que identifica tu red en internet, cada proveedor en el internet real tiene uno o varios ASNs que utilizan para anunciar sus prefijos de red al resto de internet, y lo usaremos para hacer lo mismo en DN42.

Mi ASN es `4242422799` y tengo los siguientes prefijos de red:

- IPv4: `172.20.196.64/27`
- IPv6: `fd44:665c:b6b3::/48`

Usaré estos prefijos para anunciar mi red al resto de la red DN42 y recibir rutas de los otros miembros de la red para poder comunicarme con ellos.

## Requisitos

### Software

Para conectarnos a DN42 necesitamos algunas herramientas:

- Un cliente VPN (Usaré [WireGuard](https://www.wireguard.com/))
- Un demonio BGP (Usaré [BIRD](https://bird.network.cz/))
- Algo para gestionar rutas (Usaré [IPTables](https://en.wikipedia.org/wiki/Iptables))

Hay otras alternativas al software que usaré, como OpenVPN para el cliente VPN o Quagga para el demonio BGP, pero elegí estos porque me parecieron los más fáciles de usar y los que más utiliza la comunidad DN42.

### Hardware

Para conectarnos necesitaremos un router que soporte el software que vamos a usar, o cualquier computadora que pueda actuar como router. Elegí usar un pequeño VPS de IONOS con 1 vCPU, 1GB de RAM y 10GB de almacenamiento, pero puedes usar cualquier computadora que tengas disponible.

## Configuración

### Creación de claves WireGuard

Para conectarnos con otros miembros necesitamos crear un túnel VPN con ellos, y para eso primero necesitamos crear un par de claves públicas y privadas que son necesarias para autenticar la conexión. Podemos hacer esto ejecutando los siguientes comandos:

```bash
# Generar la clave privada
$ wg genkey > privatekey
# Generar la clave pública
$ wg pubkey < privatekey > publickey
```

Necesitaremos la clave pública para los siguientes pasos, así que tenla a mano.

### Eligiendo un vecino

Lo primero que necesitamos hacer para conectarnos a la red DN42 es encontrar un vecino que esté dispuesto a conectarse con nosotros. Podemos usar una herramienta como [PingFinder](https://dn42.us/peers/) para encontrar vecinos que estén geográficamente cerca de nosotros, o podemos preguntar en el canal IRC de DN42 si alguien quiere conectarse con nosotros.

Otras redes como [Kioubit](https://dn42.g-load.eu/) tienen un sistema de conexión automatizado que nos permite conectarnos con otros miembros de la red llenando información sobre tu red en su sitio web. Podemos encontrar una lista de sistemas con conexión automatizada [aquí](https://dn42.dev/services/Automatic-Peering).

Elegí conectarme con Kioubit específicamente por su sistema de conexión automatizado y porque es una de las redes más grandes en la comunidad DN42. Una vez que nos autentiquemos en el sitio web de Kioubit y elijamos un nodo para conectarte, necesitaremos rellenar la información sobre nuestra red en el formulario.

![Kioubit autopeer portal](/connecting-to-the-dn42-decentralized-network/kioubit-peering.png)

Primero necesitamos rellenar nuestra clave pública de WireGuard que generamos previamente. Luego, para la _Dirección IPv4 del túnel_ usaremos la primera dirección IP "no cero" de nuestro prefijo de red, que en mi caso es `172.20.196.65`. Y finalmente, para la **Dirección IPv6 del túnel** generaremos una dirección link-local [^2] usando el prefijo `fe80::/10`, en mi caso `fe80::fe80::abcd`.

[^2]: [What’s the Deal with IPv6 Link-Local Addresses?](https://labs.ripe.net/author/philip_homburg/whats-the-deal-with-ipv6-link-local-addresses/)

Una vez que rellenemos la información y enviemos el formulario, podremos ver la información que necesitamos para conectarnos al nodo de Kioubit.

![Kioubit peering information](/connecting-to-the-dn42-decentralized-network/kioubit-info.png)

### Creando el túnel WireGuard

Ahora que tenemos la información que necesitamos para conectarnos al nodo de Kioubit, podemos crear el túnel WireGuard. Crearemos un archivo de configuración con la información que obtuvimos del sitio web de Kioubit y lo añadiremos a la interfaz de WireGuard.

```ini
# /etc/wireguard/kioubit.conf
[Interface]
PrivateKey = <TU CLAVE PRIVADA>

[Peer]
PublicKey = 6Cylr9h1xFduAO+5nyXhFI1XJ0+Sw9jCpCDvcqErF1s=
Endpoint = us2.g-load.eu:22799
AllowedIPs = 0.0.0.0/0,::/0
```

Con el archivo de configuración creado, podemos añadirlo a la interfaz WireGuard y habilitarla.

```bash
# Crear la interfaz WireGuard
$ ip link add dev "kioubit" type wireguard
# Añadir el archivo de configuración a la interfaz
$ wg setconf "kioubit" /etc/wireguard/kioubit.conf
# Añadir nuestra dirección IPv6 link-local a la interfaz
$ ip addr add fe80::abcd/64 dev "kioubit"
# Conectar con el nodo de Kioubit
# La primera IP es nuestra IP y la segunda IP es la IP del nodo de Kioubit
$ ip addr add 172.20.196.65/32 peer 172.20.53.98/32 dev "kioubit"
# Habilitar la interfaz
$ ip link set "kioubit" up
```

### Configurando BIRD

Ahora que tenemos el túnel WireGuard funcionando, necesitamos configurar BIRD para anunciar nuestros prefijos de red al resto de la red DN42 y recibir las rutas de los otros miembros.

Primero, necesitamos crear el archivo de configuración para bird.

```c
# /etc/bird.conf

################################################
#               Variable header                #
################################################

define OWNAS =  4242422799; # Tu ASN
define OWNIP =  172.20.196.65; # La primera IP de tu prefijo IPv4
define OWNIPv6 = fd44:665c:b6b3::1; # La primera IP de tu prefijo IPv6
define OWNNET = 172.20.196.64/27; # Tu prefijo IPv4
define OWNNETv6 = fd44:665c:b6b3::/48; # Tu prefijo IPv6
define OWNNETSET = [172.20.196.64/27+];
define OWNNETSETv6 = [fd44:665c:b6b3::/48+];

################################################
#                 Header end                   #
################################################

router id OWNIP;

protocol device {
    scan time 10;
}

/*
 *  Utility functions
 */

function is_self_net() {
  return net ~ OWNNETSET;
}

function is_self_net_v6() {
  return net ~ OWNNETSETv6;
}

function is_valid_network() {
  return net ~ [
    172.20.0.0/14{21,29}, # dn42
    172.20.0.0/24{28,32}, # dn42 Anycast
    172.21.0.0/24{28,32}, # dn42 Anycast
    172.22.0.0/24{28,32}, # dn42 Anycast
    172.23.0.0/24{28,32}, # dn42 Anycast
    172.31.0.0/16+,       # ChaosVPN
    10.100.0.0/14+,       # ChaosVPN
    10.127.0.0/16{16,32}, # neonetwork
    10.0.0.0/8{15,24}     # Freifunk.net
  ];
}

roa4 table dn42_roa;
roa6 table dn42_roa_v6;

protocol static {
    roa4 { table dn42_roa; };
    include "/etc/bird/roa_dn42.conf";
};

protocol static {
    roa6 { table dn42_roa_v6; };
    include "/etc/bird/roa_dn42_v6.conf";
};

function is_valid_network_v6() {
  return net ~ [
    fd00::/8{44,64} # ULA address space as per RFC 4193
  ];
}

protocol kernel {
    scan time 20;

    ipv6 {
        import none;
        export filter {
            if source = RTS_STATIC then reject;
            krt_prefsrc = OWNIPv6;
            accept;
        };
    };
};

protocol kernel {
    scan time 20;

    ipv4 {
        import none;
        export filter {
            if source = RTS_STATIC then reject;
            krt_prefsrc = OWNIP;
            accept;
        };
    };
}

protocol static {
    route OWNNET reject;

    ipv4 {
        import all;
        export none;
    };
}

protocol static {
    route OWNNETv6 reject;

    ipv6 {
        import all;
        export none;
    };
}

template bgp dnpeers {
    local as OWNAS;
    path metric 1;

    ipv4 {
        import filter {
          if is_valid_network() && !is_self_net() then {
            if (roa_check(dn42_roa, net, bgp_path.last) != ROA_VALID) then {
              # Reject when unknown or invalid according to ROA
              print "[dn42] ROA check failed for ", net, " ASN ", bgp_path.last;
              reject;
            } else accept;
          } else reject;
        };

        export filter { if is_valid_network() && source ~ [RTS_STATIC, RTS_BGP] then accept; else reject; };
        import limit 9000 action block;
    };

    ipv6 {   
        import filter {
          if is_valid_network_v6() && !is_self_net_v6() then {
            if (roa_check(dn42_roa_v6, net, bgp_path.last) != ROA_VALID) then {
              # Reject when unknown or invalid according to ROA
              print "[dn42] ROA check failed for ", net, " ASN ", bgp_path.last;
              reject;
            } else accept;
          } else reject;
        };
        export filter { if is_valid_network_v6() && source ~ [RTS_STATIC, RTS_BGP] then accept; else reject; };
        import limit 9000 action block; 
    };
}


include "/etc/bird/peers/*";
```

Después, necesitamos descargar los archivos ROA que se utilizarán para validar las rutas que recibimos de los otros miembros de la red DN42.

```bash
$ wget https://dn42.burble.com/roa/dn42_roa_bird2_4.conf -O /etc/bird/roa_dn42.conf
$ wget https://dn42.burble.com/roa/dn42_roa_bird2_6.conf -O /etc/bird/roa_dn42_v6.conf
```

Finalmente, necesitamos crear el archivo de configuración para el peer de Kioubit.

```c
# /etc/bird/peers/kioubit.conf
protocol bgp kioubit from dnpeers {
    # La dirección IPv4 del nodo de Kioubit
    neighbor 172.20.53.98 as 4242423914 ;
}

protocol bgp kioubit_v6 from dnpeers {
    # Dirección link-local del nodo de Kioubit
    neighbor fe80::ade0%kioubit as kioubit;
}
```

Con los archivos de configuración creados, podemos iniciar el demonio BIRD.

```bash
$ systemctl start bird
```


### Configurando el reenvío

Para poder enrutar el tráfico entre la interfaz WireGuard y el resto de la red, necesitamos habilitar el reenvío de IP en el kernel.

```bash
$ sysctl -w net.ipv4.ip_forward=1
```

## Pruebas

Ahora que tenemos todo configurado, podemos probar la conexión haciendo ping al nodo de Kioubit.

```bash
$ ping 172.20.53.98
PING 172.20.53.98 (172.20.53.98) 56(84) bytes of data.
64 bytes from 172.20.53.98: icmp_seq=1 ttl=64 time=3.07 ms
64 bytes from 172.20.53.98: icmp_seq=2 ttl=64 time=3.05 ms
64 bytes from 172.20.53.98: icmp_seq=3 ttl=64 time=3.17 ms
64 bytes from 172.20.53.98: icmp_seq=4 ttl=64 time=3.07 ms
64 bytes from 172.20.53.98: icmp_seq=5 ttl=64 time=3.18 ms
^C
--- 172.20.53.98 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 3.048/3.108/3.182/0.055 ms
```

También podemos intentar resolver un nombre de dominio usando el [servicio DNS anycast](https://dn42.eu/services/DNS)

```bash
$ dig @172.20.0.53 burble.dn42
; <<>> DiG 9.16.23-RH <<>> @172.20.0.53 burble.dn42
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56811
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;burble.dn42.			IN	A

;; ANSWER SECTION:
burble.dn42.		3600	IN	A	172.20.129.3

;; Query time: 27 msec
;; SERVER: 172.20.0.53#53(172.20.0.53)
;; WHEN: Sun Aug 11 11:32:55 UTC 2024
;; MSG SIZE  rcvd: 56
```

## Conclusión

Conectarse a la red DN42 fue una gran experiencia de aprendizaje para mí, aprendí mucho sobre las tecnologías que hacen funcionar internet. Si te interesa aprender sobre redes no dudes en unirte a la red DN42 y aprender cómo funciona internet desde adentro.

Todavía hay mucho por aprender y explorar en la red DN42, como configurar un servidor DNS, un servidor web o conectar otros dispositivos a la red. Exploraré estos temas en futuras publicaciones asi que mantente atento.
