# Critères de passage proto 1 — Magix2

Aligné sur `work/shared/spec-proto1-jazzy.md` (28/08/2026).
Rien ne se commande ici : ce fichier fige **ce qu’on mesure** une fois le kit au banc.

Hors passage proto 1 : marches, vue 360, R-Net, depth / Depth-Anything, YOLO, SLAM visuel.

---

## Bloquants (le proto ne passe pas si un seul échoue)

| ID | Mesure | Pass | Procédure (Sébastien) |
|---|---|---|---|
| T-ESTOP | E-stop sur VIN moteur du driver, pas sur le Jetson | Moteurs stop immédiat. Jetson, `/scan`, IMU restent up. `safety` en fault, `/cmd_vel` = Twist 0. Relâchement e-stop **ne** relance **pas** la traction tout seul. | 1. Robot alimenté, légère consigne avance. 2. Appuyer l’e-stop. 3. Vérifier roues mortes + Jetson toujours pingable. 4. Relâcher : toujours mort tant qu’on n’envoie pas `CANCEL` / nouvel ordre. |
| T-HB | Heartbeat PC → robot | Coupure WiFi ou stop du PC **> 500 ms** → `/cmd_vel` Twist 0. Latch jusqu’à **3** heartbeats valides d’affilée. | 1. Nav ou téléop active. 2. Couper le WiFi du PC (ou `pkill` le nœud heartbeat). 3. Chrono : traction morte avant 0,5 s. 4. Rétablir : 3 beats, puis seulement ça reprend. |
| T-LIDAR-WD | Watchdog `/scan` | Lidar muet **> 1 s** → Twist 0. Pas de nav aveugle. | Débrancher USB lidar en mouvement (banc, roues sur chandelles). Traction morte < 1 s. |
| T-STOP | `Intent.command == STOP` | Twist 0 immédiat, latch jusqu’à `CANCEL` ou `GOTO_*`. | UI ou voix : STOP pendant un GOTO. Vérifier `/cmd_vel` à zéro. |

Seul `safety` publie `/cmd_vel`. Nav2 / téléop écrivent `/cmd_vel_raw`.

---

## Passage (non bloquants entre eux, tous requis pour « proto 1 ok »)

| ID | Mesure | Pass |
|---|---|---|
| T-UDEV | udev `by-id` | `/dev/tty-ld19` et `/dev/tty-motors` existent. Aucun launch ne fige `/dev/ttyUSB0`. |
| T-SCAN | Lidar STL-19P | `ld19.launch.py`, 230400, `/scan` live, `frame_id=laser_link`. |
| T-SLAM | Carte indoor | `slam_toolbox` sur `/scan` uniquement : une carte se construit en poussant le chassis à la main (ou nav lente) dans une pièce. |
| T-UVC | Caméra C270 | `usb_cam`, RGB compressé visible sur le PC. Pas de depth. |
| T-IMU | BNO085 | I2C1, adresse `0x4A`, 3,3 V. `i2cdetect` voit le chip. Topic IMU live. Fallback UART-RVC seulement si clock-stretch SH-2 rame. |
| T-GOTO | Ordres destination | `GOTO_NAMED` et `GOTO_POSE` depuis **écran** et **voix** (`source` = `screen` \| `voice`). `frame_id` pose = `map`. |
| T-INTENT | Contrat | Un seul topic `/magix2/intent`. Publieurs = `ui` et `voice` sur le PC, rien d’autre. |

---

## Hors scope (ne pas tester, ne pas ouvrir de bug)

- Détection de marches / trous (LD19 + UVC ne voient pas un vide).
- Vue 360 / BEV.
- Écriture bus CAN R-Net (`rnet` = stub).
- Depth métrique, Depth-Anything, YOLO.

---

## Prérequis banc (avant de dérouler)

- Chandelles ou roues décollées pour T-ESTOP / T-LIDAR-WD / T-HB (pas de chute).
- Deux rails : brick 19 V Kubii (Jetson) et 3S 12 V (moteurs). E-stop = rail moteur seulement.
- Empattement URDF : **à mesurer** à réception platine (12–16 cm), sinon T-SLAM / T-GOTO dérivent. Ce n’est pas un fail software.
- `/dev/i2c-*` : numéro au premier boot, on fige **I2C1** pas le n° Linux.

---

## Rapport

Un run = une copie de ce tableau, date, opérateur, pass/fail/skip, log ou photo. Fichier : `work/tests/rapport-YYYY-MM-DD.md`.
