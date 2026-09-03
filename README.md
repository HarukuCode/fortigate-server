# fortigate-server

Outils utilisés dans le cadre d'un lab FortiGate VM (Proxmox) pour faire fonctionner une licence via un serveur FDS (FortiGuard) privé auto-hébergé.

## Contexte

Dans ce lab :
- Une VM FortiGate (`FGT_VM64_KVM`, v7.4.1) tourne sur Proxmox.
- Un container LXC (`FW-LIC`) héberge ces deux scripts et joue le rôle de serveur FDS privé.
- Le FortiGate est configuré en `central-management` pour pointer vers ce serveur (`fmg` / `server-address`), ce qui lui permet de faire son check-in de licence sans dépendre des serveurs FortiGuard officiels.

## Scripts

### `fds_server.py`
Serveur FDS minimal (TLS, port 8890) qui répond aux requêtes de check-in du FortiGate. Implémente le protocole binaire FDS (header/payload avec CRC32) suffisamment pour que le FortiGate considère la licence comme valide.

> Version patchée : le handshake TLS est protégé par un `try/except` et le contexte SSL accepte des versions/ciphers plus anciens (`TLSv1` min, `SECLEVEL=0`), pour rester compatible avec le client TLS de FortiOS et éviter qu'une connexion malformée ne tue tout le serveur.

Nécessite un certificat/clé TLS (`cert.cer` / `key.key`, non inclus ici) à placer dans le même dossier.

Lancement :
```
python3 fds_server.py
```

### `license_old.py`
Génère une licence FortiGate VM (`License.lic`) hors ligne, sans dépendre du serveur FDS. Nécessite `pycryptodome` :
```
pip3 install pycryptodome
python3 license_old.py
```

## How to use

Take FortiGate VM64 7.4.1 (VMware) as an example.

First, you need to deploy the VM and complete the configuration of the network interface in the CLI. Then you need to start the FDS server on a system that is in the same network as the FortiGate (make sure that FortiGate can access port 8890 of the FDS server).

Execute the following commands on FortiGate:

```
config system central-management
    set mode normal
    set type fortimanager
    set fmg <FDS server's ip address>
    config server-list
    edit 1
        set server-type update rating
        set server-address <FDS server's ip address>
    end

    set fmg-source-ip <FortiGate's ip address>
    set include-default-servers disable
    set vdom root
end
```

Run the `license_old.py` script to generate a License file, log in to the web service of FortiGate and import this License file.

The system will restart automatically. After the system starts up, you should be able to see some output on the FDS server, for example:

```
========================
[*] Parsing data
[+] Magic: PUTF
[+] System version: 07004000
[+] Payload length: 363
[+] Header length: 64
[+] Time: 202505301704
[+] Header crc32: 0x946cdd64
[*] Parsing obj
[+] Magic: FCPC
[+] Name: Command Object
[+] Payload length: 235
[+] System version: 07004000
[+] Payload crc32: 0x209e4867
[+] Header crc32: 0xaa2ece4
[+] Payload: b'Protocol=3.0|Command=VMSetup|Firmware=FGVM64-FW-7.04-2463|SerialNumber=FGVM32GVOVCLUK2G|Connection=Internet|Address=192.168.66.150:0|Language=en-US|TimeZone=-7|UpdateMethod=1|Uid=564d678fc9f2506bb8aebfde4052bbbd|VMPlatform=VMWARE\r\n\r\n\r\n'
[*] Packing obj
[*] Packing req
[*] Sending response
```

Log in to the web service. If everything went well, you will enter the configuration wizard. DO NOT register with FortiCare and DISABLE automatic patch upgrades.

## Origine

Ce projet reprend et adapte les scripts du repo original [rrrrrrri/fgt-gadgets](https://github.com/rrrrrrri/fgt-gadgets), avec un patch de robustesse sur `fds_server.py` (voir ci-dessus) pour un lab FortiGate sur Proxmox.

## Ce qui a été fait dans ce lab

1. VM FortiGate importée sur Proxmox (`FGT_VM64_KVM-v7.4.1`), disque qcow2 attaché sur `local-zfs`.
2. Container LXC `FW-LIC` (Debian, Python3 + git) créé pour héberger les scripts.
3. Configuration `system central-management` appliquée sur le FortiGate pour pointer vers `FW-LIC` comme serveur FortiManager/FDS.
4. `fds_server.py` lancé sur `FW-LIC`, patché pour rester stable face aux handshakes TLS du FortiGate.
5. `license_old.py` exécuté pour générer une licence VM.

## Avertissement

Usage éducatif / lab personnel uniquement. Ne pas utiliser en production ni pour contourner une licence commerciale de manière non autorisée.
