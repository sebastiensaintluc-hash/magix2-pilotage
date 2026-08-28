# Budget de performance — proto 1

**Cible :** NVIDIA Jetson Orin Nano Super 8 Go, kit `945-13766-0005-000`, JetPack 7.2 / Ubuntu 24.04, ROS 2 Jazzy.
**Graphe robot :** `slam_toolbox` + Nav2 + `magix2_safety` + LD19/STL-19P + UVC C270 + `ros2_control` + BNO085.
**Hors budget :** Vosk, Piper, UI, RViz (PC de supervision). Isaac, YOLO, depth, SLAM visuel, R-Net write.

Chiffres **RAM/CPU des nœuds = enveloppes d’architecture** (ordres de grandeur issus de stacks 2D comparables sur Jetson 8 Go, pas un `ps` Magix). À remplacer par RSS réel au premier boot.

## 1. Enveloppe machine (sources NVIDIA)

| Poste | Valeur gelée | Source |
|---|---|---|
| CPU | 6× Cortex-A78AE, 1,7 GHz (Super) | [NVIDIA Super blog](https://developer.nvidia.com/blog/nvidia-jetson-orin-nano-developer-kit-gets-a-super-boost/) |
| GPU | Ampere 1024 CUDA + 32 Tensor, 1020 MHz | idem |
| RAM | **8 Go unifiés** LPDDR5, 102 Go/s (Super) | idem |
| TOPS | 67 INT8 sparse — **non utilisés proto 1** | [page Super](https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-orin/nano-super-developer-kit/) |
| Encode vidéo | **Pas de NVENC.** 1080p30 = 1–2 cœurs CPU | [Jetson Linux — Software Encode in Orin Nano](https://docs.nvidia.com/jetson/archives/r35.4.1/DeveloperGuide/text/SD/Multimedia/SoftwareEncodeInOrinNano.html) |
| Puissance module | 7 / 15 / **25 W** + MAXN SUPER (flash `…-super`) | [JetPack 6.2 Super](https://developer.nvidia.com/blog/nvidia-jetpack-6-2-brings-super-mode-to-nvidia-jetson-orin-nano-and-jetson-orin-nx-modules/) |
| Thermique kit | 0–35 °C, heatsink + fan | carrier spec NVIDIA |
| VIN kit | 9–20 V, PSU 19 V / 45 W | carrier spec |
| Stockage | NVMe M.2 **obligatoire** pour le proto | architecture |

RAM unifiée : tout ce que le GPU touche (même idle CUDA) sort du même 8 Go que ROS. D’où GPU compute **off**.

## 2. Budget RAM (RSS, headless)

Capacité : 8192 Mo. On ne « dispose » pas de 8 Go pour ROS.

| Bloc | RSS (Mo) | Notes |
|---|---|---|
| JetPack 7.2 headless + kernel + iGPU idle | 1800–2300 | Sans gdm. CUDA non chargé. |
| RMW CycloneDDS | 80–150 | Un domain, peu de topics vers le PC. |
| `slam_toolbox` async, 5 cm, pièce < 100 m² | 250–500 | Interactive **off**. Croissance si carte énorme → veto. |
| Nav2 2D (BT, controller, planner, costmaps, smoother, collision_monitor) | 450–800 | Pas de voxel, pas d’AMCL. |
| `usb_cam` MJPEG 640×480@15 | 40–80 | Pas de buffer raw 1080p. |
| `ldlidar_stl_ros2` | 20–40 | UART 230400, ~450 pts/tour @ 10 Hz. |
| `magix2_safety` | 15–30 | Process dédié. |
| `magix2_base` + ros2_control | 30–60 | diff_drive. |
| BNO085 | 15–30 | I2C, pas un gouffre. |
| `robot_state_publisher` + TF | 20–40 | |
| **Marge obligatoire** | **≥ 800** | Sous 800 Mo `MemAvailable` : plus de `GOTO_*`. |
| **Somme haute + marge** | **~4,8–5,5 Go** | Reste ~2,5–3,5 Go. Tient. |

Somme basse (OS bas + nœuds calmes, sans marge) ≈ 2,8–3,5 Go. La marge n’est pas optionnelle : fragmentation, pic planner, sérialisation slam.

**Pic dangereux documenté :** `slam_toolbox` `std::bad_alloc` sur Jetson 8 Go (Xavier NX) avec grande carte / stack trop petit. Proto 1 = indoor petit, `stack_size_to_use` à caler, `ulimit` si sérialisation de carte.

## 3. Budget CPU (6 cœurs = 600 %)

Mode de travail : **25 W** (A78 à 1,7 GHz). Mesure en `%` d’un cœur (100 % = 1 A78 saturé).

| Bloc | CPU typique | Pic | Contrainte |
|---|---|---|---|
| OS + irq USB/UART | 10–20 | 40 | |
| CycloneDDS | 10–20 | 40 | |
| `slam_toolbox` | 50–90 | 120 (loop closure) | `map_update_interval` ≥ 2 s |
| Nav2 2D | 80–150 | 200 (replan) | controller 20 Hz |
| `usb_cam` MJPEG natif | 10–25 | 40 | **0** si on transcode : veto |
| lidar | < 10 | 15 | 10 Hz |
| `safety` + `base` + IMU + TF | 15–30 | 40 | **jamais starved** |
| **Réserve** | **≥ 150** | | 1,5 cœur pour les pics |

Charge moyenne visée : **< 350 / 600 (~58 %)**. Pic planner+slam < 80 % machine. Au-delà on baisse fps UVC ou on ralentit le slam, on ne touche pas à `safety`.

**UVC :** C270 MJPEG hardware **dans la caméra**. Le Jetson démultiplexe, il n’encode pas. x264 sur Orin Nano = famine. Interdit proto 1.

## 4. GPU

| Poste | Budget |
|---|---|
| Compute (CUDA / TensorRT / Isaac) | **0** |
| Encode NVENC | **inexistant** |
| Display | **0** (headless) |

Les 67 TOPS sont un stock mort pour proto 1. Les allumer « pour voir » est un veto RAM.

## 5. Latence (aligné tests)

Vitesses proto miniature (ordre de grandeur 0,2–0,3 m/s). Distances = illustration, pas une spec Magix.

| Chemin | Budget architecture | Spec / test | Commentaire |
|---|---|---|---|
| Heartbeat PC → `safety` → Twist 0 | traitement local < 50 ms, timer 50 ms | T-HB **> 500 ms** | 0,3 m/s × 0,5 s ≈ **15 cm**. OK proto. Pas OK fauteuil. |
| Lidar muet → Twist 0 | détection 1 période scan (100 ms) + marge | T-LIDAR-WD **> 1 s** | 0,3 m/s × 1 s ≈ **30 cm**. OK proto. |
| `Intent.STOP` → `/cmd_vel` | **< 50 ms** (DDS local) | T-STOP immédiat | `safety` subscriber dédié, pas via Nav2. |
| Nav2 `/cmd_vel_raw` | 20 Hz (50 ms) | — | Seul `safety` publie `/cmd_vel`. |
| TF `map`→`odom` | 20–50 Hz | T-SLAM | slam async, pas synchrone. |
| UVC → PC | 100–300 ms | T-UVC (visu) | **Hors** chemin traction. |

`safety` est le dernier mot. Nav2 peut ramer : la traction s’arrête quand même.

## 6. Réseau (même `ROS_DOMAIN_ID`)

| Topic | Direction | Volumétrie visée |
|---|---|---|
| `/magix2/heartbeat` | PC → robot | ≥ 2 Hz, quelques centaines d’octets |
| `/magix2/intent` | PC → robot | rare, reliable, depth 10, **pas de latch** |
| `image_raw/compressed` (MJPEG) | robot → PC | 640×480@15 ≈ 1–2 Mb/s |
| `/scan` | **local robot** | ~450–500 pts × 10 Hz. Pas de flood PC. |
| `/cmd_vel`, `/cmd_vel_raw` | local | Twist 20 Hz |

WiFi kit (M.2) suffit à ça. Ce qui ne suffit pas : raw 640×480 YUYV + LaserScan 10 Hz vers RViz lointain.

## 7. Lidar (charge)

STL-19P / LD19 : 0,03–12 m, 10 Hz typ., 4500 (LD19) / 5000 (STL-19P) mesures/s, UART 230400, 5 V.

Compute : négligeable. Risques hors CPU :

- min range 0,03 m vs platine 25–30 cm → FOV / masque URDF (hardware) ;
- scan à nombre de points **variable** vs `slam_toolbox` qui droppe : driver, binning fixe si besoin (logiciel) ;
- watchdog présence, **pas** sémantique (spec).

## 8. Cinématique vs compute

`diff_drive`, `wheel_radius` 0,0325 m, empattement **à mesurer** 12–16 cm.

Ça ne saute pas le budget CPU. Ça saute le slam si l’URDF est faux (T-SLAM / T-GOTO dérivent = fail mesure, pas software). IMU BNO085 I2C1 `0x4A` dans le graphe pour l’odom, pas pour les TOPS.

## 9. Stockage, thermique, alim

- **NVMe** rootfs + maps. microSD = repli de flash, pas d’exécution.
- zram 2 Go = filet OOM, pas de working set.
- 25 W module + USB lidar/cam/IMU : PSU 19 V / 45 W kit OK. Ne pas tirer le 24 V Magix.
- 25 W indoor kit ventilé : OK. Boîtier fermé sans flux : pas encore dans le proto 1, à revoir avant intégration fauteuil.

## 10. Mesure au premier boot (gèle la table)

À coller dans `work/tests/` au premier allumage, machine **headless 25 W**, graphe complet, pièce unique :

1. `free -h` / `MemAvailable` au repos OS, puis graphe up, puis après 10 min de slam.
2. RSS par nœud (`ps`, `ros2 node list`).
3. CPU `top` : idle ≥ 40 % machine pendant un `GOTO` lent.
4. Latence `Intent.STOP` → `/cmd_vel` zéro (chrono ou stamp).
5. Coupure heartbeat : traction morte **< 500 ms**.
6. Temp SoC sous 25 W, pas de throttle.

Si un nœud sort de son enveloppe : on réduit UVC ou `map_update`, on **ne** monte **pas** de 16 Go pour proto 1.

## 11. Sources

- Spec : `docs/SPEC-PROTO1.md`, critères `work/tests/criteres-proto1.md`
- NVIDIA Super kit / blog clocks RAM CPU GPU puissance
- NVIDIA : Orin Nano **sans NVENC**
- `slam_toolbox` : `enable_interactive_mode` = croissance RAM ; `bad_alloc` Jetson 8 Go, grandes cartes
- Projet comparable Orin Nano + slam_toolbox : `map_update_interval` 2 s, pas de RViz on-device (Project Ogre)
- LDRobot LD19 / STL-19P datasheets (10 Hz, 4500/5000 Hz, UART 230400)
