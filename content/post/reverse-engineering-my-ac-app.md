---
title: "Reverse engineering my air conditioner app"
date: 2025-01-07T00:00:00-06:00
summary: "Taking a look at the internals of my AC app"
author: "Sebastian Marines"
categories: ["reverse-engineering", "iot", "ac"]
tags: ["reverse-engineering", "iot", "ac"]
---

A few years ago, my family bought some Whirlpool air conditioners for our house. The are great and they even have an app that allows you to control them from your phone. I really have no complaints about the air conditioners or the app. But I was curious and wanted to see how the AC connects to the app.

I really didn't knew where to start, and my initial idea was to use Wireshark to sniff the actual AC traffic (since it was already connected to the Wi-Fi). I tried that with no luck, so the next thing to test was the actual app.

![Whirlpool air conditioner](/reverse_engineering_my_ac_app/whirlpool_ac.png)

## Sniffing the app traffic

I started by installing the app on my phone and logging in to my account. I read some articles about how to sniff the app traffic, and the most common way to do it is by using a proxy. I decided to use `mitmproxy` for this, a tool that allows you to intercept traffic between your device and the internet, among other cool things.

### Installing mitmproxy

Since I'm on a Mac, I installed `mitmproxy` using Homebrew:

```sh
❯ brew install mitmproxy
```

Then I started the proxy:

```sh
❯ sudo mitmweb
```

The next thing to do was to configure the proxy on my phone and installing the certificate following the instructions on [https://docs.mitmproxy.org/stable/overview-getting-started/](https://docs.mitmproxy.org/stable/overview-getting-started/). After that, I opened the app and started to see the traffic on the proxy.

## Failing to intercept the traffic

After I configured the proxy on my phone and installed the certificate, I was supposed to see the traffic on the proxy. But I didn't. I tried to open the app, log in, and do some actions, but the app was not working and I was getting some errors on the proxy:

```bash
[23:33:53.365][192.168.0.120:42012] Client TLS handshake failed. The client does not trust the proxy's certificate for api.whrcloud.com (OpenSSL Error([('SSL routines', '', 'ssl/tls alert certificate unknown')]))
```

I googled the error and found [this](https://github.com/mitmproxy/mitmproxy/discussions/5307) discussion on the mitmproxy GitHub repo. It seems that the app was using certificate pinning, a technique that prevents the app from accepting any certificate other than the one it was built with.

But after some more googling, I found a tool called `apk-mitm` that allows you to patch the app and remove the certificate pinning. I decided to give it a try.

## Patching the app

First, I installed `apk-mitm`:

```sh
❯ npm install -g apk-mitm
```

Then, I patched the apk file that I downloaded from my phone:

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

After some minutes, the patched apk file was generated. I installed it on my phone and tried to log in again. This time, I was able to see the traffic on the MITM proxy web interface.

![alt text](/reverse_engineering_my_ac_app/mitmproxy1.png)

![alt text](/reverse_engineering_my_ac_app/mitmproxy2.png)

I started to see some interesting endpoints like `/oauth/token`, `/api/v3/appliance/all/account/XXXXX`, `/api/v1/appliance/XXXXXXXXX`.

## Testing with a Python script

I had some ideas about what to do next, and I found a Python library called `whirlpool-sixth-sense` that allows you to interact with Whirlpool appliances. I installed it:

```sh
❯ pip install whirlpool-sixth-sense
```

And ran a basic script to see if I could get some data from my air conditioners:

```python
import asyncio

import aiohttp

from whirlpool.aircon import Aircon
from whirlpool.appliancesmanager import AppliancesManager
from whirlpool.auth import Auth
from whirlpool.backendselector import BackendSelector, Brand, Region

USERNAME = "YOUR_EMAIL"
PASSWORD = "YOUR_PASSWORD"


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
            print(f"Authentication failed: {e}")
            return

        appliances_manager = AppliancesManager(backend_selector, auth, session)

        if not await appliances_manager.fetch_appliances():
            print("Failed to fetch appliances")
            return

        for ac_data in appliances_manager.aircons:
            said = ac_data["SAID"]

            ac = Aircon(backend_selector, auth, said, session)
            await ac.connect()

            print(f"Connected to AC {said}")
            print(f"Current temperature: {ac.get_temp()}")

        await asyncio.sleep(1)


asyncio.run(main())

```

```sh
❯ python main.py
Connected to AC WPR4AJV7DN97D
Current temperature: 23.0
Connected to AC WPR4F4KBY7H96
Current temperature: 25.0
```

And that's it! I was able to reverse engineer my air conditioner app and interact with my air conditioners using Python. I did some cool stuff like creating a Go script that exports the data to Prometheus and Grafana, and an Alexa skill (since the official Whirlpool skill is not available in Mexico), but that's a story for another post.
