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

## Ce qui a été fait dans ce lab

1. VM FortiGate importée sur Proxmox (`FGT_VM64_KVM-v7.4.1`), disque qcow2 attaché sur `local-zfs`.
2. Container LXC `FW-LIC` (Debian, Python3 + git) créé pour héberger les scripts.
3. Configuration `system central-management` appliquée sur le FortiGate pour pointer vers `FW-LIC` comme serveur FortiManager/FDS.
4. `fds_server.py` lancé sur `FW-LIC`, patché pour rester stable face aux handshakes TLS du FortiGate.
5. `license_old.py` exécuté pour générer une licence VM.

## Avertissement

Usage éducatif / lab personnel uniquement. Ne pas utiliser en production ni pour contourner une licence commerciale de manière non autorisée.
