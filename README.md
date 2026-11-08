# SPWS
A simple pi camera web server
# rtsp-web

Relais vidéo pour Debian : reçoit des flux RTSP (caméras IP, enregistreurs…)
et les rend visibles dans un navigateur sur le réseau local.

```
Caméra RTSP ──► FFmpeg (1 processus par flux) ──► segments HLS en RAM ──► serveur web :8080 ──► navigateurs
```

## Installation (Debian 12 ou 13)

```bash
tar xzf rtsp-web.tar.gz && cd rtsp-web
sudo ./install.sh
sudo nano /etc/rtsp-web/config.yaml      # déclarer vos caméras
sudo systemctl restart rtsp-web
```

Ouvrez ensuite `http://<adresse-du-serveur>:8080/` depuis n'importe quel poste du réseau.

Le script installe FFmpeg, Flask et Waitress depuis les dépôts Debian, crée un
utilisateur système `rtsp-web`, installe l'application dans `/opt/rtsp-web` et
un service systemd démarré au boot. Il télécharge une fois le lecteur `hls.js` ;
l'interface fonctionne ensuite sans accès Internet.

## Configuration

Tout se règle dans `/etc/rtsp-web/config.yaml` (fichier commenté). Pour chaque flux :

| Clé | Rôle |
|---|---|
| `id` | identifiant court (minuscules, chiffres, `-`, `_`) |
| `name` | nom affiché |
| `url` | adresse RTSP, identifiants compris si besoin |
| `transcode` | `true` pour réencoder en H.264 |
| `audio` | `aac` (par défaut), `copy` ou `none` |

Avant de déclarer une caméra, vérifiez son flux depuis le serveur :

```bash
ffprobe -rtsp_transport tcp "rtsp://user:pass@192.168.1.50:554/stream1"
```

**Quand activer `transcode` ?** Par défaut la vidéo est recopiée sans
réencodage (quasi aucune charge CPU). Activez-le si la caméra envoie du H.265
(mal lu par Chrome et Firefox) ou si la latence est forte : sans réencodage, la
durée des segments dépend de l'intervalle d'images clés réglé dans la caméra.
Le réencodage coûte du CPU (environ un cœur par flux 1080p).

## Exploitation

```bash
systemctl status rtsp-web
journalctl -u rtsp-web -f        # journal en direct
curl http://localhost:8080/healthz
```

Un flux coupé est relancé automatiquement (délai croissant jusqu'à 30 s). Un
flux qui ne produit plus d'images est détecté et redémarré. Les mots de passe
RTSP sont masqués dans les journaux et ne sont jamais envoyés aux navigateurs.

Si le pare-feu est actif : `sudo ufw allow from 192.168.1.0/24 to any port 8080 proto tcp`.

## Limites

- **Latence** : HLS donne environ 4 à 8 s de décalage. Réduire `segment_seconds`
  à 1 gagne un peu. Pour du temps réel (< 1 s), il faut passer à WebRTC
  (voir MediaMTX, qui le propose clé en main).
- **Accès** : aucune authentification, l'interface est destinée à un réseau
  local de confiance. Ne pas l'exposer sur Internet sans reverse proxy
  (nginx/Caddy) avec authentification et HTTPS.

## Développement

```bash
sudo apt install ffmpeg python3-flask python3-yaml
# dans config.yaml, mettre hls_dir: ./hls
python3 app.py -c config.yaml -v
```
