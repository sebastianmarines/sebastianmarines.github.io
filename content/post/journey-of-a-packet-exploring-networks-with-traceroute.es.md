---
title: "El viaje de un paquete de internet: Explorando redes con traceroute"
date: 2024-08-23T00:00:00-06:00
summary: "Este artículo explica la mecánica detrás de traceroute, desde los encabezados IP y campos TTL hasta los mensajes ICMP. Aprende por qué traceroute es una herramienta crucial para el diagnóstico de redes, cómo difiere de ping y cómo interpretar su salida. Ideal para administradores de redes, profesionales de TI y usuarios curiosos que buscan entender la topología de red y solucionar problemas de conectividad. Obtén conocimientos sobre la infraestructura de internet y mejora tu capacidad para diagnosticar problemas de red de manera eficiente."
author: "Sebastian Marines"
categories: ["redes"]
tags: ["redes"]
---

A menudo asumimos que cuando intentamos conectarnos a un servidor, la conexión va a funcionar. Pero no siempre es el caso. Hay veces hay que algo no funciona en la red que nos impide conectarnos al destino. Pero, ¿cómo podemos saber dónde está el problema?

Algo que hacia muy seguido era usar `ping` para probar si habia conectividad entre dos máquinas, y aunque `ping` es una gran herramienta para probar si hay conexión, no nos da mucha información sobre lo que puede estar mal.

El internet es una red compleja de routers, switches y computadoras, y cuando intentamos conectarnos a un servidor, nuestros paquetes pasan por muchos routers antes de llegar al destino. Si uno de estos routers está mal configurado o no funciona, el paquete de red puede ser descartado y no tendremos conexión con el destino.

En este post, veremos cómo funciona `traceroute` y cómo puede ayudarnos a diagnosticar problemas de red.

![Ping entre dos máquinas](/traceroute/simple-ping.svg)

## La vida de un paquete

Cuando te conectas a un servidor, ya sea un sitio web o cualquier otro servicio, tu computadora verifica si está en la misma red que la dirección IP de destino. Si lo está, envía el paquete directamente al destino. Si no lo está, envía el paquete al gateway predeterminado, que es el router de tu proveedor de internet.

El router luego verifica si la dirección IP de destino está en su tabla de enrutamiento. Si lo está, envía el paquete directamente al destino. Si no lo está, envía el paquete al siguiente router en el camino. Este proceso se repite hasta que el paquete alcanza el destino.

En la siguiente imagen puedes ver este proceso. Primero, la computadora con dirección IP `10.0.0.1` hace ping a la computadora con dirección IP  `10.0.0.2` enviando un paquete ICMP. Como no están en la misma red, el paquete se envía al router con dirección IP `10.1.0.5`. Este router luego reenvía el paquete al router con dirección IP `10.1.0.6`, que reenvía el paquete a la computadora con dirección IP `10.1.0.7`, y el paquete alcanza su destino.

![Ping entre dos máquinas con múltiples routeres](/traceroute/ping-multiple-routers.svg)

Si hacemos ping a la computadora de destino, podemos ver que responde correctamente.

```bash
$ ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: icmp_seq=0 ttl=58 time=5.368 ms
64 bytes from 10.0.0.2: icmp_seq=1 ttl=58 time=4.325 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=58 time=4.225 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=58 time=4.479 ms
```

## ¿Qué sucede cuando hay un problema?

Pero ahora, ¿qué sucede si hay un problema en el camino? Por ejemplo, ¿qué pasa si uno de los routers está mal configurado y descarta el paquete? ¿O qué pasa si la computadora de destino está caída? ¿Cómo podemos saber dónde está el error?

Mira el siguiente ejemplo, donde el router `10.1.0.6` no puede comunicarse con el router `10.1.0.7`.

![Ping entre dos máquinas con un camino roto](/traceroute/ping-broken-path.svg)

Si hacemos ping a la computadora de destino, podemos ver que no tenemos conexión, pero no sabemos dónde está el problema.

```bash
$ ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
^C

--- 10.0.0.2 ping statistics ---
4 packets transmitted, 0 received, +4 errors, 100% packet loss, time 3031ms

```

Aquí es donde `traceroute` resulta útil. `traceroute` es una herramienta de diagnóstico de red que muestra el camino que toma un paquete desde el origen hasta el destino. Nos permite ver cada router en el camino y el tiempo de respuesta de cada uno.

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 30 hops max, 60 byte packets
 1  10.1.0.5  1.123 ms  0.912 ms  1.145 ms
 2  10.1.0.6  2.145 ms  2.023 ms  2.311 ms
 3  * * *
 4  * * *
 5  * * * 
```

En este ejemplo podemos ver que los routers `10.1.0.5` y `10.1.0.6` están funcionando correctamente, pero hay un problema con el siguiente router en el camino. Puedes ver que primero alcanzamos el router `10.1.0.5` y el tiempo de respuesta es de alrededor de 1ms. Luego alcanzamos el router `10.1.0.6`, con un tiempo de respuesta de alrededor de 2ms. Pero luego no tenemos ninguna respuesta de otro router y no podemos alcanzar la computadora de destino.

## ¿Cómo funciona `traceroute`?

Para entender cómo funciona `traceroute`, primero necesitamos entender los campos en un encabezado IP.

![Encabezado IPv4](/traceroute/ipv4-header.svg)

Hay varios campos en el encabezado IP, pero nos centraremos en 3 de ellos:

- **Dirección IP de origen**: La dirección IP de la computadora que envía el paquete.
- **Dirección IP de destino**: La dirección IP de la computadora que recibe el paquete.
- **Tiempo de vida (TTL)**: El número máximo de routers por los que puede pasar el paquete.

Las direcciones IP de origen y destino se explican a si mismas, pero el campo TTL es interesante. Este campo se utiliza para evitar que los paquetes estén recorriendo indefinidamente nodos en una red.

Cuando se envía un paquete, el campo TTL tiene un valor por defecto de 64 (al menos en Linux y macOS). Cuando el paquete llega a un router, el router disminuye el campo TTL en 1. Si el campo TTL llega a 0, el router descarta el paquete y envía un *Mensaje de Tiempo Excedido* de vuelta a la computadora de origen. Pero si el campo TTL es mayor que 0, el router reenvía el paquete al siguiente router en el camino.

Así es como `traceroute` puede mostrar el camino que toma un paquete desde el origen hasta el destino. Envía un paquete ICMP con un TTL de 1, luego un paquete con un TTL de 2, luego un paquete con un TTL de 3, y así sucesivamente. De esta manera, `traceroute` puede ver la respuesta de cada router en el camino.

## ¿Qué es ICMP?

Hasta este punto, hemos mencionado ICMP un par de veces, pero ¿qué es ICMP?

ICMP significa *Protocolo de Mensajes de Control de Internet*. Es un protocolo utilizado por dispositivos de red para diagnosticar problemas de comunicación de red[^1].

[^1]: [¿Qué es el Protocolo de Mensajes de Control de Internet (ICMP)?](https://www.cloudflare.com/learning/ddos/glossary/internet-control-message-protocol-icmp/)

Este protocolo es utilizado por comandos como `ping` y `traceroute` para probar la conectividad de red.

El estándar ICMP[^2] define varios tipos de mensajes. Los más comunes son:

[^2]: [Protocolo de Mensajes de Control de Internet](https://datatracker.ietf.org/doc/html/rfc792)

- **Solicitud de Eco**: Utilizado por el comando `ping` para probar si una computadora es alcanzable.
- **Respuesta de Eco**: Enviado por una computadora en respuesta a una *Solicitud de Eco*.
- **Mensaje de Tiempo Excedido**: Enviado por un router cuando el campo TTL de un paquete llega a 0.
- **Mensaje de Destino Inalcanzable**: Enviado por un router cuando la computadora de destino es inalcanzable.
- **Mensaje de Redirección**: Enviado por un router para informar a una computadora que hay una mejor ruta hacia un destino.
- **Mensaje de Problema de Parámetro**: Enviado por un router cuando un paquete tiene un encabezado incorrecto.
- **Mensaje de Fuente Agotada**: Enviado por un router para informar a una computadora que está enviando demasiados paquetes.

> `traceroute` envia paquetes *Solicitud de Eco* con distintos valores TTL y lee los mensajes ICMP *Mensaje de Tiempo Excedido* que envian los routers en el camino para mostrar el camino que toma un paquete desde el origen hasta el destino.

## Visualizando `traceroute`

Vamos a ejecutar `traceroute` con `10.0.0.2` como destino y veamos qué sucede en cada paso.

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
...
```

Primero, `traceroute` envía un paquete ICMP con un TTL de 1. El paquete llega al primer router en el camino, que disminuye el campo TTL en 1. Como el campo TTL ahora es 0, el router descarta el paquete y envía un *Mensaje de Tiempo Excedido* de vuelta a la computadora de origen.

![Ping con un TTL de 1](/traceroute/ping-ttl-1.svg)

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
 1 10.1.0.5 (10.1.0.5)  1.123 ms  0.912 ms  1.145 ms
...
```

> Puedes ver que se muestran 3 tiempos de respuesta para el primer router. Esto es porque `traceroute` envía 3 paquetes a cada router.

Después, `traceroute` envía un paquete ICMP con un TTL de 2. El paquete llega al primer router en el camino, que disminuye el campo TTL en 1. El campo TTL ahora es 1, por lo que el router reenvía el paquete al siguiente router en el camino.

El paquete llega al segundo router en el camino, que disminuye el campo TTL en 1. Como el campo TTL ahora es 0, el router descarta el paquete y envía un *Mensaje de Tiempo Excedido* de vuelta a la computadora de origen.

![Ping con un TTL de 2](/traceroute/ping-ttl-2.svg)

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
 1 10.1.0.5 (10.1.0.5)  1.123 ms  0.912 ms  1.145 ms
 2 10.1.0.6 (10.1.0.6)  2.145 ms  2.023 ms  2.311 ms
...
```

Ahora, `traceroute` envía un paquete ICMP con un TTL de 3. El paquete llega al primer router en el camino, que disminuye el campo TTL en 1. El campo TTL ahora es 2, por lo que el router reenvía el paquete al siguiente router en el camino.

Cuando el paquete llega al último router en el camino, el router disminuye el campo TTL en 1. Como el campo TTL ahora es 0, el router descarta el paquete y envía un *Mensaje de Tiempo Excedido* de vuelta a la computadora de origen.

![Ping con un TTL de 3](/traceroute/ping-ttl-3.svg)

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
 1 10.1.0.5 (10.1.0.5)  1.123 ms  0.912 ms  1.145 ms
 2 10.1.0.6 (10.1.0.6)  2.145 ms  2.023 ms  2.311 ms
 3 10.1.0.7 (10.1.0.7)  3.145 ms  3.023 ms  3.311 ms
...
```

Finalmente, `traceroute` envía un paquete ICMP con un TTL de 4. El paquete llega al primer router en el camino, que disminuye el campo TTL en 1. Luego cada router en el camino disminuye el campo TTL en 1 hasta que el paquete llega a la computadora de destino.

La computadora de destino recibe el paquete y envía una *Respuesta de Eco ICMP* de vuelta a la computadora de origen.

![Ping con un TTL de 4](/traceroute/ping-ttl-4.svg)

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
 1 10.1.0.5 (10.1.0.5)  1.123 ms  0.912 ms  1.145 ms
 2 10.1.0.6 (10.1.0.6)  2.145 ms  2.023 ms  2.311 ms
 3 10.1.0.7 (10.1.0.7)  3.145 ms  3.023 ms  3.311 ms
 4 10.0.0.2 (10.0.0.2)  4.145 ms  4.023 ms  4.311 ms
```

## Conclusión

Traceroute es una poderosa herramienta de diagnóstico que nos ayuda a entender el camino que toman nuestros datos a través de internet. Nos proporciona una vista sobre la topología de la red y los posibles problemas en el camino, y nos permite:

- Identificar dónde se están descartando o retrasando los paquetes
- Descubrir rutas de enrutamiento inesperadas
- Detectar routers mal configurados o con mal funcionamiento
- Estimar la latencia de la red entre saltos

Mientras que ping puede decirnos si un destino es alcanzable, traceroute nos muestra todo el viaje, convirtiéndolo en una herramienta esencial en el kit de herramientas de cualquier administrador de red o usuario curioso.

Como hemos visto, internet es una compleja red de dispositivos interconectados, y traceroute ayuda a simplificar esta complejidad. La próxima vez que encuentres un problema de red, recuerda usar traceroute, puede ayudarte a encontrar el problema y ahorrar valioso tiempo de resolución de problemas.
