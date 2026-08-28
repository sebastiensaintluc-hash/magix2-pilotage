# Go / no-go — compute proto 1 (Jetson Orin Nano Super 8 Go)

**Date :** 2026-08-28
**Auteur :** Architecte (design authority)
**Spec :** [`docs/SPEC-PROTO1.md`](../../docs/SPEC-PROTO1.md)
**Budget détaillé :** [`BUDGET-PERF-PROTO1.md`](BUDGET-PERF-PROTO1.md)

## Verdict

**GO conditionnel.**

Le graphe proto 1 (`slam_toolbox` + Nav2 2D + `magix2_safety` + LD19/STL-19P + UVC C270 + `ros2_control` + BNO085) **tient** sur un Orin Nano Super 8 Go (RAM unifiée, 6× A78AE, pas de NVENC) **si et seulement si** les conditions ci-dessous sont tenues. Hors de ces conditions, le même silicium est un **NO-GO**.

Ce n’est pas un GO Isaac, YOLO, depth, SLAM visuel, R-Net write, ni marches.

## Pourquoi GO

- La spec a déjà coupé les postes qui font sauter 8 Go unifiés : pas d’Isaac ROS, pas de TensorRT, pas de YOLO, pas de depth, GPU compute **off** indoor, Vosk/Piper sur le **PC**, pas d’écran 7" sur le robot.
- `slam_toolbox` 2D async + Nav2 costmaps 2D + lidar UART 10 Hz, c’est un stack CPU, pas GPU. Les 67 TOPS ne servent pas au proto 1, et c’est voulu.
- LD19/STL-19P : ~4500–5000 pts/s, 10 Hz, UART 230400. Charge CPU négligeable. Le risque lidar est le **watchdog 1 s** (spec), pas le compute.
- UVC : Logitech C270 sort du **MJPEG natif**. On **ne ré-encode pas** (Orin Nano n’a **pas** de NVENC ; un x264 1080p30 mange 1–2 cœurs).
- Les critères bloquants (T-ESTOP, T-HB 500 ms, T-LIDAR-WD, T-STOP) sont des chemins courts `safety`. Ils tiennent si `safety` n’est pas starved. C’est une contrainte d’ordonnancement, pas de TOPS.

Chiffres : voir le budget. Ce sont des **enveloppes d’architecture**, pas des RSS mesurés sur le kit (kit pas encore là). Premier boot = mesure, et on gèle la table.

## Conditions de GO (obligatoires)

1. **Headless.** `multi-user.target`, pas de GNOME/gdm, **pas de RViz sur le Jetson**. Visu sur le PC.
2. **Flash Super.** Config `jetson-orin-nano-devkit-super` (sinon seulement 7/15 W). Mode de travail proto 1 : **25 W**. MAXN SUPER seulement si la thermique kit (heatsink + fan, 0–35 °C) tient ; MAXN throttle si TDP dépassé.
3. **Rootfs NVMe**, pas microSD. Maps + logs ROS sur SD = jitter I/O et usure.
4. **Pas de contexte CUDA** au boot proto 1. Aucun nœud Isaac / TensorRT / DeepStream. GPU idle.
5. **UVC : MJPEG caméra, 640×480 @ 15 Hz** (plafond 720p @ 10 Hz). Topic compressé vers le PC. Interdit : raw YUYV sur le DDS, transcode H.264/x264 sur le Jetson.
6. **`slam_toolbox` async**, résolution 5 cm, `map_update_interval` ≥ 2 s, `enable_interactive_mode: false`. Indoor pièce / petit appartement (`< ~80–100 m²`). Pas de warehouse.
7. **Nav2 2D only.** Pas de voxel layer, pas d’AMCL en parallèle du SLAM (un seul estimateur pose : `slam_toolbox`). Costmap locale ~3×3 m @ 5 cm. Controller 20 Hz.
8. **RMW CycloneDDS.** `/scan` **n’est pas** floodé vers le PC (ou throttle). Heartbeat + `/magix2/intent` + image compressée, c’est le contrat WiFi.
9. **Priorité CPU :** `safety` > `base` / `ros2_control` > lidar > Nav2 > slam > UVC. `safety` ne doit jamais attendre le planner.
10. **Filet RAM :** zram 2 Go (pas un working set). Si `MemAvailable` < 800 Mo : nouvel ordre `GOTO_*` refusé, log, **`safety` intact**.
11. **Alim :** brick 19 V kit (ou DC-DC isolé dans 9–20 V). Pas le pack 24 V Magix sur le jack.

## Vetos (ça bascule en NO-GO)

| Si on ajoute / on fait | Pourquoi |
|---|---|
| Isaac ROS, YOLO, Depth-Anything, TensorRT, CUDA compute | RAM unifiée 8 Go + contexte GPU. Spec proto 1 déjà hors scope. |
| Ubuntu Desktop + RViz sur le Jetson | +1–1,5 Go et vol de cœurs. |
| Ré-encode H.264/x264 de l’UVC | Pas de NVENC. 1–2 cœurs A78, famine `safety`/`slam`. |
| AMCL + `slam_toolbox` ensemble | Double carte, double RAM, double CPU. |
| Nav2 voxel / 3D | RAM costmap. |
| `enable_interactive_mode` slam | Fuite RAM linéaire (scans cachés). |
| Carte slam grande (entrepôt) sans mode localize | `bad_alloc` documenté sur Jetson 8 Go. |
| Rootfs SD | I/O et usure. |
| DDS qui republie `/scan` + `image_raw` non compressé | WiFi + RAM DDS. |

## Risques résiduels (GO, à traiter, pas des vetos compute)

- **JetPack 7.2 / Ubuntu 24.04 / Jazzy** est le gel logiciel, pas une preuve que le BSP Orin Nano Super est déjà trivial. C’est un risque d’intégration pour l’ing. logiciel, pas un dépassement RAM. Si JP 7.2 n’est pas flashable Super, on remonte au Chef **avant** d’inventer un autre distro.
- **Cinématique petite platine** (wheel_radius 0,0325 m, empattement 12–16 cm à mesurer) : odom bruyante. Le BNO085 n’est pas un luxe compute, c’est de la qualité slam. Hors budget CPU.
- **Montage lidar :** STL-19P min 0,03 m. Le chassis 25–30 cm ne doit pas manger le FOV. C’est un veto **hardware / URDF**, pas CPU. Ing. hardware.
- **T-HB 500 ms** à 0,3 m/s proto ≈ 15 cm d’errance max après perte WiFi. Acceptable proto miniature. **Pas** transposable tel quel au Magix (masse, vitesse). On ne gèle pas 500 ms pour le fauteuil.
- Chiffres RAM/CPU ci-contre = enveloppes. **Mesure RSS + `top` au premier boot**, table gelée ensuite. Si un nœud sort de son enveloppe, on coupe de la spec (souvent UVC fps ou `map_update`), on n’achète pas 16 Go.

## Conséquences pour les autres rôles

- **Ing. logiciel :** params slam/Nav2/usb_cam/DDS ci-dessus ; nœud RAM watchdog ; `safety` isolé (process dédié, nice). Pas d’Isaac « pour tester ».
- **Ing. hardware :** NVMe, flash Super, FOV lidar, 19 V, thermique kit 25 W.
- **Testeur :** ajouter une mesure de perf au banc (RSS, CPU, `MemAvailable`, latence STOP/heartbeat) à côté des T-*.
- **Acheteur / Chef :** pas de montée Orin NX / 16 Go pour le proto 1. 8 Go Super **suffit** dans cette enveloppe.

## Décision

| Cible | Verdict |
|---|---|
| Proto 1 (spec gelée) | **GO conditionnel** |
| Isaac / depth / YOLO / 360 / R-Net write / marches | **NO-GO** (hors proto 1, et 8 Go ne les porterait pas ensemble de toute façon) |
