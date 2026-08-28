# Magix2 — spec logicielle proto 1 (ROS 2 Jazzy)

Cible : NVIDIA Jetson Orin Nano Super 8 Go, JetPack 7.2 / Ubuntu 24.04, mode **25 W**, rootfs **NVMe** (pas de SD).
Open source uniquement. Headless. RMW : **CycloneDDS**.

GO AD conditionnel proto 1 — contraintes figées :
- GPU / CUDA **off** indoor (pas d’Isaac, pas de YOLO, pas de depth, pas de NVENC)
- UVC **MJPEG 640×480 @ 15 fps**, zéro transcode
- `slam_toolbox` **async**, résolution **5 cm**
- Nav2 **2D**, pas d’AMCL, pas de voxel layer
- Interdit on-device : RViz, x264, rootfs SD
- RAM visée **4,8–5,5 / 8 Go**
- `safety` = seul publisher de `/cmd_vel`

Hors proto 1 : écriture R-Net, détection de marches, SLAM visuel, depth, YOLO, Isaac ROS.

---

## 1. Graphe

**Jetson (robot)** : `map` (slam_toolbox) · `nav` (Nav2) · `safety` · `base` (ros2_control) · lidar · IMU · UVC.

**PC de supervision** : `ui` · Vosk small-fr + Piper · ReSpeaker USB. Publie uniquement `/magix2/intent`.

Même `ROS_DOMAIN_ID`. Heartbeat PC → robot.

---

## 2. `magix2_msgs/Intent`

Fichier `magix2_msgs/msg/Intent.msg` :

```
builtin_interfaces/Time stamp
string command    # GOTO_NAMED | GOTO_POSE | CANCEL | STOP
string name       # pièce / POI si GOTO_NAMED
geometry_msgs/PoseStamped pose   # si GOTO_POSE ; frame_id DOIT être "map"
string source     # voice | screen
```

- Topic : `/magix2/intent` (`magix2_msgs/Intent`)
- QoS : reliable, depth 10, **pas** de latch
- Publieurs : `ui` et `voice` sur le PC, **rien d’autre**
- `nav` traduit `GOTO_*` vers Nav2
- `STOP` est aussi consommé par `safety` (prioritaire)

Contrat figé : on n’y touche plus.

---

## 3. `magix2_safety`

Dernier mot sur la traction.

**Entrées**

| Topic | Type | Rôle |
|---|---|---|
| `/magix2/heartbeat` | `std_msgs/Header` | PC, ≥ 2 Hz |
| `/magix2/intent` | `Intent` | `STOP` / `CANCEL` |
| `/cmd_vel_raw` | `geometry_msgs/Twist` | sortie Nav2 / téléop |
| `/scan` | `sensor_msgs/LaserScan` | watchdog lidar (présence, pas sémantique) |

**Sortie**

| Topic | Type | Rôle |
|---|---|---|
| `/cmd_vel` | `geometry_msgs/Twist` | vers `magix2_base` |

Règles :

1. Timeout heartbeat **> 500 ms** → Twist zéro, latch jusqu’à 3 heartbeats valides d’affilée.
2. `command == STOP` → Twist zéro immédiat, latch jusqu’à `CANCEL` ou nouvel ordre non-STOP.
3. E-stop matériel (coupure VIN moteurs) : `safety` passe en fault, Twist zéro logiciel en plus.
4. Lidar muet **> 1 s** → Twist zéro (pas de nav aveugle).
5. Zéro publication `/cmd_vel` autre que `safety`. Nav2 écrit `/cmd_vel_raw`.

Pas d’écriture CAN / R-Net. Backend `rnet` = stub.

---

## 4. Lidar `/scan` (LD19 / STL-19P)

- Package : `ldlidar_stl_ros2`
- Launch : `ld19.launch.py` (pas A2, pas `rplidar_ros`)
- Port : `/dev/tty-ld19` (udev `by-id`, jamais `ttyUSB0` en dur)
- Baud : 230400
- Topic : `/scan` (`sensor_msgs/LaserScan`), `frame_id`: `laser_link`
- SLAM : `slam_toolbox` async, 5 cm, sur `/scan` uniquement (pas de SLAM visuel)
- Nav2 : 2D, pose via slam_toolbox, **pas d’AMCL**, **pas de voxel**

Moteurs Yahboom : `/dev/tty-motors`, 115200, proto `$mtype:1#` / `$speed:G,0,D,0#`.

---

## 5. Frames et base

`map` → `odom` → `base_link` → `laser_link`

- Cinématique : `diff_drive` (2WD + casters)
- `wheel_radius` : 0,0325 m
- Empattement : à calibrer (12–16 cm) à réception
- IMU BNO085 : I2C1, adresse `0x4A`, 3,3 V ; bus Linux au `i2cdetect` ; fallback UART-RVC

UVC : `usb_cam` MJPEG 640×480@15, topic compressé vers le PC, pas de raw→x264, pas de NVENC. Depth / Depth-Anything : hors indoor, GPU off.

---

## 6. Critères proto 1 (aligné tests)

- `/scan` live + `slam_toolbox` construit une carte
- UVC stream sur le PC
- BNO085 I2C1
- E-stop coupe la traction
- Coupure WiFi / heartbeat → Twist 0 (bloquant)
- Ordre `GOTO_NAMED` / `GOTO_POSE` depuis UI ou voix

Hors passage : marches, vue 360, R-Net.

---

## 7. Décisions ouvertes (tranchables)

Aucune sur `intent`. Reste : empattement URDF à la mesure de la platine, numéro `/dev/i2c-*` au premier boot Jetson.
