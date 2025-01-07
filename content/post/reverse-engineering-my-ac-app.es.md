---
title: "Ingeniería inversa de mi aplicación de aire acondicionado"
date: 2025-01-07T00:00:00-06:00
summary: "Analizando el funcionamiento interno de mi aplicación de aire acondicionado"
author: "Sebastian Marines"
categories: ["ingenieria-inversa", "iot", "ac"]
tags: ["ingenieria-inversa", "iot", "ac"]
---

Hace algunos años, mi familia compró algunos aires acondicionados Whirlpool para nuestra casa. Son excelentes e incluso tienen una aplicación que te permite controlarlos desde tu teléfono. Realmente no tengo quejas sobre los aires acondicionados ni la aplicación. Pero tenía curiosidad y quería ver cómo el aire acondicionado se conecta con la aplicación.

No sabía por dónde empezar, y mi idea inicial fue usar Wireshark para analizar el tráfico del aire acondicionado (ya que estaba conectado al Wi-Fi). Lo intenté sin éxito, así que lo siguiente fue probar con la aplicación.

![Aire acondicionado Whirlpool](/reverse_engineering_my_ac_app/whirlpool_ac.png)

## Analizando el tráfico de la aplicación

Comencé instalando la aplicación en mi teléfono e iniciando sesión en mi cuenta. Leí algunos artículos sobre cómo analizar el tráfico de una aplicación, y la forma más común de hacerlo es usando un proxy. Decidí usar `mitmproxy` para esto, una herramienta que te permite interceptar el tráfico entre tu dispositivo e internet, entre otras cosas interesantes.

### Instalando mitmproxy

Como estoy en una Mac, instalé `mitmproxy` usando Homebrew:

```sh
❯ brew install mitmproxy
```

Luego inicié el proxy:

```sh
❯ sudo mitmweb
```

Lo siguiente fue configurar el proxy en mi teléfono e instalar el certificado siguiendo las instrucciones en [https://docs.mitmproxy.org/stable/overview-getting-started/](https://docs.mitmproxy.org/stable/overview-getting-started/). Después de eso, abrí la aplicación y comencé a ver el tráfico en el proxy.

## Fallo al interceptar el tráfico

Después de configurar el proxy en mi teléfono e instalar el certificado, se suponía que debía ver el tráfico en el proxy. Pero no fue así. Intenté abrir la aplicación, iniciar sesión y realizar algunas acciones, pero la aplicación no funcionaba y estaba recibiendo algunos errores en el proxy:

```bash
[23:33:53.365][192.168.0.120:42012] Client TLS handshake failed. The client does not trust the proxy's certificate for api.whrcloud.com (OpenSSL Error([('SSL routines', '', 'ssl/tls alert certificate unknown')]))
```

Busqué el error en Google y encontré [esta](https://github.com/mitmproxy/mitmproxy/discussions/5307) discusión en el repositorio de GitHub de mitmproxy. Parece que la aplicación estaba usando certificate pinning, una técnica que evita que la aplicación acepte cualquier certificado que no sea con el que fue construida.

Pero después de buscar un poco más, encontré una herramienta llamada `apk-mitm` que permite parchear la aplicación y eliminar el certificate pinning.

## Parcheando la aplicación

Primero, instalé `apk-mitm`:

```sh
❯ npm install -g apk-mitm
```

Luego, parcheé el archivo apk que descargué de mi teléfono:

```sh
❯ apk-mitm com-whirlpool-android-wpapp-20560-68462060-32dd70cf658ba3aa2fd541bc7bfa4c08.apk

  ╭ apk-mitm v1.3.0
  ├ apktool v2.9.3
  ╰ uber-apk-signer v1.3.0

  Using temporary directory:
  /private/var/folders/sy/5_lnrl3n153_dsjrhtpf0f3r0000gn/T/apk-mitm-fd1c43f296ab11c72addd0aad3b88dbd

  ✔ Checking prerequisities
  ✔ Decoding APK file
  ✔ Applying patches
  ✔ Encoding patched APK file
  ✔ Signing patched APK file

   WARNING

  This app seems to be using Android App Bundle which means that you
  will likely run into problems installing it. That's because this app
  is made out of multiple APK files and you've only got one of them.

  If you want to patch an app like this with apk-mitm, you'll have to
  supply it with all the APKs. You have two options for doing this:

  – download a *.xapk file (for example from https://apkpure.com​)
  – export a *.apks file (using https://github.com/Aefyr/SAI​)

  You can then run apk-mitm again with that file to patch the bundle.

   Done!  Patched file: ./com-whirlpool-android-wpapp-20560-68462060-32dd70cf658ba3aa2fd541bc7bfa4c08-patched.apk
```

Después de algunos minutos, se generó el archivo apk parcheado. Lo instalé en mi teléfono e intenté iniciar sesión nuevamente. Esta vez, pude ver el tráfico en la interfaz web del proxy MITM.

![alt text](/reverse_engineering_my_ac_app/mitmproxy1.png)

![alt text](/reverse_engineering_my_ac_app/mitmproxy2.png)

Comencé a ver algunos endpoints interesantes como `/oauth/token`, `/api/v3/appliance/all/account/XXXXX`, `/api/v1/appliance/XXXXXXXXX`.

## Probando con un script de Python

Tenía algunas ideas sobre qué hacer después, y encontré una biblioteca de Python llamada `whirlpool-sixth-sense` que te permite interactuar con electrodomésticos de Whirlpool. Así que la instalé:

```sh
❯ pip install whirlpool-sixth-sense
```

Y ejecuté un script básico para ver si podía obtener algunos datos de mis aires acondicionados:

```python
import asyncio

import aiohttp

from whirlpool.aircon import Aircon
from whirlpool.appliancesmanager import AppliancesManager
from whirlpool.auth import Auth
from whirlpool.backendselector import BackendSelector, Brand, Region

USERNAME = "TU_EMAIL"
PASSWORD = "TU_CONTRASEÑA"


async def main():
    backend_selector = BackendSelector(Brand.Whirlpool, Region.US)

    async with aiohttp.ClientSession() as session:
        auth = Auth(
            backend_selector,
            USERNAME,
            PASSWORD,
            session,
        )
        try:
            await auth.do_auth(store=True)
        except Exception as e:
            print(f"Falló la autenticación: {e}")
            return

        appliances_manager = AppliancesManager(backend_selector, auth, session)

        if not await appliances_manager.fetch_appliances():
            print("Falló la obtención de electrodomésticos")
            return

        for ac_data in appliances_manager.aircons:
            said = ac_data["SAID"]

            ac = Aircon(backend_selector, auth, said, session)
            await ac.connect()

            print(f"Conectado al AC {said}")
            print(f"Temperatura actual: {ac.get_temp()}")

        await asyncio.sleep(1)


asyncio.run(main())
```

```sh
❯ python main.py
Conectado al AC WPR4AJV7DN97D
Temperatura actual: 23.0
Conectado al AC WPR4F4KBY7H96
Temperatura actual: 25.0
```

¡Y eso es todo! Pude hacer ingeniería inversa de la aplicación de mi aire acondicionado e interactuar con mis aires acondicionados usando Python. Hice algunas cosas interesantes como crear un script en Go que exporta los datos a Prometheus y Grafana, y una skill de Alexa (ya que la skill oficial de Whirlpool no está disponible en México), pero esa es una historia para otro post.
