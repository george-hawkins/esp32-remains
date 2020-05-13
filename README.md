ESP32 remains
=============

**Update:** most of the useful content in this repo has been moved to [micropython-wifi-setup](https://github.com/george-hawkins/micropython-wifi-setup).

---

Basic setup:

    $ python3 -m venv env
    $ source env/bin/activate
    $ pip install --upgrade pip
    $ curl -O https://raw.githubusercontent.com/micropython/micropython/master/tools/pyboard.py
    $ chmod a+x pyboard.py
    $ mv pyboard.py env/bin
    $ pip install pyserial

Once set up, the `source` step is the only one you need to repeat - you need to use it whenever you open a new terminal session in order to activate the environment. If virtual environments are new to you, see my notes [here](https://github.com/george-hawkins/snippets/blob/master/python-venv.md).

On Mac:

    $ export PORT=/dev/cu.SLAB_USBtoUART

On Linux:

    $ export PORT=/dev/ttyUSB0

Copying files:

    $ pyboard.py --device $PORT -f cp main.py :

REPL:

    $ screen $PORT 115200

---

To pull in the new minimal `webserver.py`:

    $ pyboard.py --device $PORT -f cp webserver.py :main.py

---

Currently `main.py` pulls in just MicroWebSrv2 and MicroDNSSrv.

slimDNS on its own is pulled in with `main-mdns.py` - to install it instead of `main.py`:

    $ pyboard.py --device $PORT -f cp main-mdns.py :main.py

To query the names advertised by `main-mdns.py`:

    $ dig @224.0.0.251 -p 5353 portal.local
    $ dig @224.0.0.251 -p 5353 dns.local

To see all the queries that slimDNS sees, i.e. not just the ones relating to names that it's advertising, uncomment the `print` in `compare_packed_names`.

My Mac automatically tried to query for all these names:

```
_airplay
_airport
_apple-mobdev
_apple-pairable
_companion-link
_googlecast
_ipp
_ipps
_ippusb
_pdl-datastream
_printer
_ptp
_raop
_rdlink
_scanner
_sleep-proxy
_uscan
_uscans
```

---

Using slimDNS to lookup `dns.local` and then MicroDNSSrv to respond to any arbitrary name:

    $ dig +short @dns.local foobar

If you've overriden your nameserver to something like [8.8.8.8](https://en.wikipedia.org/wiki/Google_Public_DNS) then this is quite slow (I suspect it first tries to resolve `dns.local` via DNS and only then falls back to trying mDNS). In such a situation it's noticeably faster to explicitly resolve `dns.local` via mDNS:

    $ nameserver=$(dig +short @224.0.0.251 -p 5353 dns.local)
    $ dig +short @$nameserver foobar

If you haven't overriden your nameserver, i.e. just accept the one configured when you connect to an AP, then the `@nameserver` can be omitted altogether:

    $ dig +short foobar

---

About 100 lines of `slimDNS.py` involves code to detect name clashes, e.g. if you use the name `alpha` it checks first to see if something else is already advertising this name.

I removed this code and in doing so I also removed `resolve_mdns_address`, i.e. the ability to resolve mDNS addresses - now the code can only advertise addresses.

---

Sources:

* https://github.com/jczic/MicroWebSrv2
* https://github.com/nickovs/slimDNS/blob/638c461/slimDNS.py
* https://github.com/jczic/MicroDNSSrv/blob/ebe69ff/microDNSSrv.py

---

There are various interesting ports of MicroPython, some of which contain, among other things, e.g. more extensive support for ESP32 features.

See Adafruit's page on the "[many forks & ports of MicroPython](https://github.com/adafruit/awesome-micropythons)".

---

In CPython objects have an associated `__dict__`. In MicroPython 1.12 this isn't available, but if you e.g. want to lookup the name of a constant value you can do:

    def name(obj, value):
        for a in dir(obj):
            if getattr(obj, a) == value:
                return a
        return value

    name(network, 201)

Note that this code assumes that all attributes will have unique values, e.g. here that `network` only has one constant with value 201.

It's probably better to filter for the names that you want:

    names = [s for s in dir(network) if s.startswith("STAT_")]
    stats = { getattr(network, k) : k for k in names }

---

MicroPython `sys.print_exception(e)` is equivalent to CPython `traceback.print_exception(e.__class__, e, e.__traceback__)`.

PyCharm MicroPython plugin
--------------------------

Setup:

* Settings / Plugins - added MicroPython plugin.
* Settings / Languages & Frameworks - ticked _Enable MicroPython support_, set device as ESP8266 (ESP32 isn't an option) and manually set _Device path_ (as _Detect_ didn't work) to `/dev/ttyUSB0`.

However this doesn't get you very far - for autocompletion and knowing what methods are available it depends on https://github.com/vlasovskikh/intellij-micropython/tree/master/typehints

The type hints haven't been updated in 2 years and were never very extensive, e.g. the only `stdlib` module it has hints for is `utime`.

So it doesn't know about `sys.print_exception` and other MicroPython specific functions.

Note: `vlasovskikh` is Andrey Vlasovskikh - he is the technical lead on PyCharm.

In the end, I uninstalled the plugin - the other features it offered seemed little more convenient than using `rshell`.
