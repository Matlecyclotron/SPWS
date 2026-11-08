# rtsp-web

Interface web de visionnage de caméras pour le réseau local, sous Debian.

```
Raspberry Pi + caméra          Serveur Debian                            Navigateurs
(MediaMTX)          ──RTSP──►  MediaMTX (relais) ──WebRTC / HLS──►       mosaïque, zoom,
                               rtsp-web (interface :8080)                 capture d'image
```

MediaMTX, sur le serveur, va chercher chaque caméra en RTSP et la rediffuse
aux navigateurs en WebRTC (moins d'une seconde de décalage), avec un repli
automatique en HLS si WebRTC est bloqué. Le Pi n'envoie qu'un seul flux, quel
que soit le nombre de personnes qui regardent. L'application `rtsp-web` déclare
les caméras auprès de MediaMTX, suit leur état et sert l'interface.

## Installation (Debian 12 ou 13)

```bash
tar xzf rtsp-web.tar.gz && cd rtsp-web
sudo ./install.sh
sudo nano /etc/rtsp-web/config.yaml      # adresse de la caméra
sudo systemctl restart rtsp-web
```

Ouvrez ensuite `http://<adresse-du-serveur>:8080/`.

Le script installe MediaMTX (version figée, téléchargée depuis GitHub), Flask
et Waitress, crée deux utilisateurs système et deux services démarrés au boot :
`mediamtx` et `rtsp-web`. FFmpeg n'est plus nécessaire.

## Ajouter une caméra

Une entrée dans `/etc/rtsp-web/config.yaml`, puis `sudo systemctl restart rtsp-web` :

```yaml
cameras:
  - id: jardin
    name: "Jardin"
    url: "rtsp://192.168.1.30:8554/cam"
```

Pour un Raspberry Pi sous MediaMTX, l'adresse est `rtsp://<ip-du-pi>:8554/<chemin>`,
où `<chemin>` est le nom donné à la caméra dans le `mediamtx.yml` du Pi
(par exemple `cam` pour une section `paths: cam: source: rpiCamera`).
Pour vérifier l'adresse, ouvrez-la dans VLC (Média › Ouvrir un flux réseau)
depuis n'importe quel poste du réseau.

Inutile de toucher à `/etc/mediamtx/mediamtx.yml` : l'application y déclare
elle-même les caméras par l'API locale de MediaMTX.

## Fonctions de l'interface

- **Zoom numérique** : molette, pincement sur écran tactile, double-clic, ou
  boutons − / + (cliquer sur le pourcentage le réinitialise). On déplace
  l'image zoomée en la faisant glisser.
- **Capture d'image** : enregistre un PNG de ce qui est affiché, zoom compris,
  à la résolution d'origine de la caméra. Le fichier est nommé
  `<caméra>_<date>_<heure>.png`.
- **Son** : le bouton n'apparaît que si la caméra transmet de l'audio.
- **Plein écran**, état de chaque caméra, reconnexion automatique.

## Réseau

Les navigateurs doivent pouvoir joindre le serveur sur ces ports :

| Port | Usage |
|---|---|
| 8080/tcp | interface web |
| 8889/tcp | WebRTC (établissement de la connexion) |
| 8189/udp | WebRTC (vidéo) |
| 8888/tcp | HLS (repli) |
| 8554/tcp | RTSP réémis par le serveur (facultatif : VLC, enregistreur) |

Avec ufw, par exemple pour le réseau 192.168.1.0/24 :

```bash
for p in 8080/tcp 8889/tcp 8189/udp 8888/tcp; do
  sudo ufw allow from 192.168.1.0/24 to any port ${p%/*} proto ${p#*/}
done
```

## Exploitation

```bash
systemctl status mediamtx rtsp-web
journalctl -u rtsp-web -f          # état des caméras
journalctl -u mediamtx -f          # détail des connexions vidéo
curl http://localhost:8080/healthz
```

## Limites et pistes

- **Codec** : WebRTC lit le H.264 partout. Le H.265 n'est lu que par certains
  navigateurs ; si une caméra l'impose, il faudra la réencoder (MediaMTX peut
  lancer FFmpeg pour cela).
- **Accès** : pas d'authentification ; l'interface est destinée à un réseau
  local de confiance. Ne pas l'exposer sur Internet sans reverse proxy avec
  HTTPS et authentification.
- **Évolutions prévues par l'architecture** : enregistrement continu ou sur
  demande (fonction `record` de MediaMTX), capture côté serveur, pilotage PTZ
  des caméras motorisées (ONVIF), comptes utilisateurs.

## Développement

```bash
sudo apt install python3-flask python3-yaml
mediamtx mediamtx.yml &              # MediaMTX local
python3 app.py -c config.yaml -v
```
