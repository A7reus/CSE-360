# Reading Packet Tracer pkt Files as XML

I wanted to check my Packet Tracer labs without opening Packet Tracer every time, so I looked for a way to convert `.pkt` files into something readable.

## 1. Why a pkt file looks like garbage in a text editor

Current Packet Tracer files are encrypted. If you try to rename one to `.zip` and unzip it, it fails. If you open it in NeoVim, you only see random characters. That is normal. Cisco does not provide an official export, but other students and researchers worked out the format and shared converters.

A quick web search for `packet tracer pkt to xml converter` gives a few options that all do the same job:

- `securedypkt` on PyPI, installed with pip
- `Unpacket` on GitHub, a plain Python script
- `pka-decipher` on GitHub, with a small desktop viewer
- `pt-convert.podik.cz`, an online converter in the browser

I use `securedypkt` because it is a simple pip install, works offline, and needs no extra libraries. The online converter also works, but then you have to upload your lab file, which I prefer to avoid for coursework.

## 2. Converting a file

You need Python 3.8 or newer. I keep this in a virtual environment so it does not mess with system Python.

```bash
python3 -m venv venv
source venv/bin/activate
pip install securedypkt
```

Then convert. Say your lab is in the same folder:

```bash
unpacket lab.pkt -o lab.xml
```

If all goes well you see something like:

```
[*] Reading 'lab.pkt'...
[*] Decrypting...
[*] Writing decrypted data to 'lab.xml'...
[+] Success! Decryption complete.
```

`lab.xml` is now normal XML and starts with:

```xml
<?xml version='1.0' encoding='utf-8'?>
<PACKETTRACER5>
  <VERSION>...</VERSION>
```

No Packet Tracer install is needed just to read it. You still need Packet Tracer if you want to see the drawing. If you edit the XML and want a pkt file again, the same package has the reverse command:

```bash
repacket lab.xml -o lab-new.pkt
```

## 3. What the converter does behind the scenes

From the README files of these tools, a pkt file on disk has four layers:

1. A first unscrambling step that reverses the bytes and applies a simple XOR.
2. Real decryption with the Twofish cipher in EAX mode, using a fixed key that Packet Tracer uses. The last 16 bytes are a check value. If the file is damaged, this check fails.
3. A second unscrambling XOR step.
4. Decompression in Qt format, which is a length number followed by zlib data. After this you get the XML.

You do not need to do any of this by hand. The tool runs it in order for you.

## 4. How to read the XML once you have it

Open `lab.xml` in NeoVim or any text editor. It is large, often close to a megabyte, so use search rather than scrolling.

These are the places I look at:

- Device list: search for `<NAME translate`. That shows each router, switch, hub, PC and server and the label you gave it.
- PC and server addresses: search for `<IP>`, `<SUBNET>`, `<PORT_GATEWAY>`, `<PORT_DNS>`. That block is the IP Configuration window from Packet Tracer. `POWER` tells if the port is on. `true` means Port Status is checked, `false` means the interface is shut down.
- Router commands: search for `<LINE>`. You will find lines like `interface FastEthernet0/0`, `ip address ...`, and `ip route ...`. That is the running config from the CLI.
- DNS records: search for `RESOURCE-RECORD` and `NAMESERVER-DATABASE`. Each record has a `NAME` like `www.example.com` and an `IPADDRESS`. There is also an `ENABLED` flag nearby for the DNS service itself. Both the flag and the records have to be right.
- Web pages: search for the file name, for example `index.html`. The page text is inside `FILE_CONTENT` and `TEXT`, stored with `<` written as `&lt;`. Copy it out and unescape it if you want to read the HTML.
- Cables: search for `<LINK>` and `<CABLE>`. Each entry has a FROM device, a TO device, port names, and `FUNCTIONAL`. That matches the lines you draw on the canvas.

A practical way is to search for one IP or one hostname at a time and check the few lines around it, instead of trying to understand the whole file at once.

## 5. Small notes

- Only convert your own labs or labs your teacher shared for the course.
- These converters are community work for study and interoperability. They are not from Cisco.
- If Packet Tracer updates the format one day and the tool reports a tag error, look for a newer release of the same tool.
