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

Lancement :
```
python3 fds_server.py
```

### `license_old.py`
Génère une licence FortiGate VM (`License.lic`) hors ligne, sans dépendre du serveur FDS.

Lancement :
```
python3 license_old.py
```

## Dépendances

- **Python 3** (testé avec 3.13)
- **`fds_server.py`** : uniquement des modules de la bibliothèque standard (`socket`, `ssl`, `threading`, `struct`, `zlib`, `datetime`) — pas de `pip install` nécessaire pour le script lui-même.
  - Nécessite en revanche un certificat TLS et sa clé privée (`cert.cer` / `key.key`). Le script ne les génère pas : `context.load_cert_chain(certfile='cert.cer', keyfile='key.key')` se contente de les charger, il faut donc que ces deux fichiers existent déjà dans le même dossier. Ils sont présents dans le [repo d'origine](https://github.com/rrrrrrri/fgt-gadgets) (commit `4a77e1f`) mais n'ont volontairement pas été repris dans **ce** repo. Pour en générer un toi-même (auto-signé) :
    ```
    openssl req -x509 -newkey rsa:2048 -keyout key.key -out cert.cer -days 3650 -nodes -subj "/CN=fds-server"
    ```
- **`license_old.py`** : nécessite `pycryptodome`
  ```
  pip3 install pycryptodome
  ```

## Comment l'utiliser

Exemple avec un FortiGate VM64 7.4.1 (VMware).

D'abord, déployer la VM et terminer la configuration de l'interface réseau en CLI. Ensuite, démarrer le serveur FDS sur une machine du même réseau que le FortiGate (s'assurer que le FortiGate peut accéder au port 8890 du serveur FDS).

Exécuter les commandes suivantes sur le FortiGate :

```
config system central-management
    set mode normal
    set type fortimanager
    set fmg <adresse IP du serveur FDS>
    config server-list
    edit 1
        set server-type update rating
        set server-address <adresse IP du serveur FDS>
    end

    set fmg-source-ip <adresse IP du FortiGate>
    set include-default-servers disable
    set vdom root
end
```

Exécuter le script `license_old.py` pour générer un fichier de licence, se connecter à l'interface web du FortiGate et importer ce fichier de licence.

Le système redémarre automatiquement. Une fois redémarré, le serveur FDS devrait afficher une sortie de ce type :

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

Se connecter à l'interface web. Si tout s'est bien passé, l'assistant de configuration initial s'affiche. NE PAS s'enregistrer auprès de FortiCare et DÉSACTIVER les mises à jour de correctifs automatiques.

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
