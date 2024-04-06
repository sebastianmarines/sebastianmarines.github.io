---
title: "Mi intento (fallido) de cancelar la suscripción a los correos electrónicos de UnderArmour"
date: 2022-02-08T11:50:00-06:00
summary: "Comencé a recibir correos electrónicos promocionales de UnderArmour en mi dirección de correo electrónico personal, a pesar de que nunca compré nada de ellos ni me suscribí a ningún boletín promocional. Intenté cancelar la suscripción, pero el enlace no funcionaba. Esta es mi historia"
---

Hace aproximadamente 6 meses, comencé a recibir correos electrónicos promocionales de UnderArmour en mi dirección de correo electrónico personal, a pesar de que nunca compré nada de ellos ni me suscribí a ningún boletín promocional.

Entonces, ¿cuál es el problema? Solo necesito hacer clic en el botón de cancelar suscripción incluido, ¿verdad?

![Página de cancelar suscripción de UnderArmour](/my_failed_attempt_to_unsubscribe_from_underarmour_emails/button.jpeg)

Bueno... no es tan fácil, ya que ese botón te lleva a una página web que no funciona.

## Los filtros de Gmail al rescate

¿Ahora qué? No puedo cancelar la suscripción y estoy recibiendo múltiples correos electrónicos cada semana llenando mi bandeja de entrada. Podría marcar esos correos electrónicos como spam y seguir con mi vida, pero como Gmail ha estado enviando algunos correos electrónicos importantes a la carpeta de spam y la reviso regularmente para asegurarme de no perderme algo importante; no quería seguir viendo los correos electrónicos de UnderArmour.

La solución alternativa que encontré fue usar [filtros](https://support.google.com/mail/answer/6579) que eliminan automáticamente todos los correos electrónicos de _underarmour@e.underarmour.com_ y evitan que lleguen a mi bandeja de entrada nuevamente.

Estaba satisfecho con esta solución ya que ya no veía esos correos electrónicos, pero todo este tiempo me molestaba el hecho de que nunca les di mi correo electrónico y no me permitían cancelar la suscripción.

## Investigando el enlace de cancelar suscripción roto

Siendo un estudiante curioso y molesto por esto, mi única opción era investigar y detener esos correos electrónicos de llegar.

El botón de cancelar suscripción en el correo electrónico te lleva a _trk.e.underarmour.com_ pero responde con un código 302 y redirige a _pages.e.underarmour.com_, así que sigamos.

La siguiente URL para probar es _pages.e.underarmour.com_, y a primera vista, parece que esto está relacionado con DNS.

![](/my_failed_attempt_to_unsubscribe_from_underarmour_emails/cant_connect.jpeg)

Inicialmente, pensé que podría tener algo que ver con mi PiHole ya que a veces rompe algunas páginas, pero incluso con PiHole desactivado no pude acceder a la página que supuestamente me permitiría cancelar la suscripción.

## El registro faltante

Una búsqueda rápida en [securitytrails.com](https://securitytrails.com) muestra que un registro A para _pages.e.underarmour.com_ existía, pero fue eliminado hace 2 años.

![](/my_failed_attempt_to_unsubscribe_from_underarmour_emails/dns.png)

Entonces, ¿ahora qué? Parecía un callejón sin salida, pero había una última cosa que quería intentar.

## Buscando todos los subdominios de underarmour.com

Afortunadamente, hay sitios web que ya realizan un seguimiento de dominios como [este](https://subdomains.whoisxmlapi.com/). El único problema es que hay 344 subdominios para _underarmour.com_.

Intenté con los que incluían la palabra email o alguna variante, y después de probar solo 3 subdominios, me topé con uno que parecía prometedor: _pages.emails.underarmour.com_.

Luego, reemplacé _pages.**e**.underarmour.com_ (el enlace al que el botón de cancelar suscripción envió) con _pages.**email**.underarmour.com_, manteniendo la ruta de la URL.

Et voilà, finalmente pude cancelar la suscripción a los correos electrónicos de UnderArmour. ¿O pude?

![Página de cancelar suscripción de UnderArmour](/my_failed_attempt_to_unsubscribe_from_underarmour_emails/cancel.png)

## Concluyendo

Quiero darle a UnderArmour el beneficio de la duda y pensar que simplemente cometieron un error y fueron redirigidos al subdominio incorrecto, pero parece que hay otras personas con este problema. He seguido recibiendo correos electrónicos promocionales a pesar de haber enviado el formulario para cancelar la suscripción varias veces.

Cada vez más empresas están adoptando este tipo de tácticas para retener usuarios, pero ¿con qué propósito? Después de esta experiencia, nunca querría comprar algo de ellos; de hecho, trataría de disuadir a cualquier persona de hacerlo.

**¿Qué debemos hacer para dejar de recibir sus correos electrónicos, UnderArmor?**