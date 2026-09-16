# Simulation pour Magix2 — choix, périmètre, vetos

Répond à : « comment simuler ce projet », et « Webots ou Gazebo ».

Aligné sur `docs/SPEC-PROTO1.md`, `work/tests/criteres-proto1.md`, `work/architecture/BUDGET-PERF-PROTO1.md`.

> **Révision du 16/09/2026.** Une première version de cette note recommandait Webots. Elle avait sous-pondéré un argument qui vient de notre propre spec : `magix2_base` tourne sous `ros2_control`. La recommandation passe à **Gazebo Harmonic**. Motif détaillé en §2.

---

## 1. À quoi sert un simulateur ici

À une seule chose : un **banc d'intégration logiciel sur le PC de supervision**, pour écrire et déboguer `magix2_safety`, `slam_toolbox`, Nav2 et la chaîne `Intent` → `GOTO` **avant** que la platine Yahboom arrive de Chine.

Le chemin critique est le chassis (CN → FR, DDU). Sans simulateur, le logiciel attend le matériel. Avec, les deux avancent en parallèle et le jour du déballage on débogue du câblage, pas des topics.

Ce que la simulation **n'est pas** dans ce projet :

- pas un substitut aux critères bloquants (`T-ESTOP`, `T-HB`, `T-LIDAR-WD`, `T-STOP`) — §4 ;
- pas quelque chose qui tourne sur la Jetson — §3 ;
- pas un modèle du Magix II 6 roues. On simule la **platine 2WD**, rien d'autre. Un jumeau numérique du fauteuil est hors proto 1, comme le 6WD physique.

Coût licence : 0 € dans les deux cas (Gazebo Apache 2.0, Webots Apache 2.0). Le coût réel est du temps d'ingénieur et un PC avec un GPU qui tient OpenGL.

---

## 2. Gazebo Harmonic, pas Webots

**Décision : Gazebo Harmonic + `ros_gz` + `gz_ros2_control`.**

L'argument décisif est la **parité simulation / réel sur le chemin `ros2_control`**. Notre spec (§5) fige `magix2_base` sous `ros2_control` en `diff_drive`. Avec `gz_ros2_control`, le bloc `<ros2_control>` de l'URDF et le YAML de `diff_drive_controller` sont **les mêmes fichiers** en simulation et sur le robot : seul le `<plugin>` du hardware change.

```xml
<!-- simulation -->
<plugin>gz_ros2_control/GazeboSimSystem</plugin>
<!-- robot réel -->
<plugin>magix2_base/Magix2System</plugin>
```

C'est tout le sujet. Le simulateur existe pour que le 13 novembre se passe bien ; sa valeur est proportionnelle à la part de ce qu'on y débogue qui part telle quelle sur la Jetson. Avec Webots, l'URDF reçoit un bloc `<webots>` propre au simulateur et le chemin de commande diffère — une partie de ce qu'on valide n'est pas ce qui tournera.

Deux arguments secondaires, réels mais non décisifs :

- **Nav2 sur Jazzy se réfère à Gazebo.** `nav2_bringup` / `tb3_simulation_launch.py` tournent sur le Gazebo moderne depuis Jazzy. Notre stack est Nav2 + `slam_toolbox` : quand un comportement de costmap ou de BT nous échappe, on veut pouvoir le comparer à l'implémentation de référence sans traduire.
- **La CI headless est plus propre.** `gz sim -s -r --headless-rendering` utilise EGL et n'a **pas besoin de serveur X**. Webots exige un framebuffer virtuel (Xvfb) parce que ses capteurs à rendu passent par OpenGL, et sans GPU il retombe sur Mesa en logiciel.

### Ce que Webots avait pour lui, et pourquoi ça ne suffit pas

Webots mappe les capteurs depuis l'URDF, donc pas de process de pont ni de fichier de mapping séparé. C'est un vrai avantage de simplicité, et `webots_ros2_control` existe et est maintenu par Cyberbotics — l'option n'est pas mauvaise.

Mais l'estimation « une journée de setup contre plusieurs » de la première version de cette note était exagérée. Notre table de pont `ros_gz_bridge` fait environ six entrées (§6), pas un fichier de configuration à rallonge. L'écart de setup se compte en heures, pas en jours, et ne pèse pas contre la parité `ros2_control`.

Webots reste le repli documenté si Gazebo nous coûte plus de deux jours de mise au point.

### Deux pièges Gazebo à connaître d'avance

1. **Gazebo Classic est mort.** `gazebo11` et tout Gazebo Classic sont EOL depuis le 31 janvier 2025, et n'existent pas sur Jazzy. Or la majorité des tutoriels indexés visent encore Classic : `gazebo_ros`, `libgazebo_ros_diff_drive.so`, `roslaunch`. Tout tutoriel qui contient ces chaînes est à jeter. Le moderne, c'est `gz sim`, `ros_gz`, Harmonic — supporté jusqu'en septembre 2028.
2. **`gz_ros2_control` a des accrocs de packaging connus sur Jazzy + Harmonic** : plugin `gz_ros2_control/GazeboSimSystem` introuvable / bibliothèque partagée non résolue (issues `ros-controls/gz_ros2_control` #390 et #720). À vérifier dès le premier `apt install`, avant d'écrire quoi que ce soit d'autre. C'est le genre de chose qui mange une journée si on la découvre tard.

Installation : `sudo apt install ros-jazzy-ros-gz ros-jazzy-gz-ros2-control ros-jazzy-gz-ros2-control-demos`.

### Quand rouvrir la question

Le jour où on attaque les **marches et les trous** : physique de contact et capteurs depth y comptent, et c'est là que Gazebo creuse l'écart. Choisir Gazebo maintenant évite précisément d'avoir à redéménager le monde, l'URDF et les launch à ce moment-là.

---

## 3. Où ça tourne — veto Jetson

**Aucun simulateur ne tourne sur l'Orin Nano.** Ce n'est pas une préférence, c'est le budget de perf, et ça vaut pour Gazebo comme pour Webots.

Les deux ont besoin d'un moteur de rendu pour les capteurs simulés (lidar GPU, caméra), même headless. Or `BUDGET-PERF-PROTO1.md` §4 fige le GPU à **0** en compute et **0** en display, parce que les 8 Go sont unifiés et que tout ce que le GPU touche sort de la même RAM que ROS. Lancer un simulateur à côté du graphe, c'est le `std::bad_alloc` de `slam_toolbox` déjà documenté.

| Machine | Rôle en simulation |
|---|---|
| PC de supervision | `gz sim` + `ros_gz_bridge` + `slam_toolbox` + Nav2 + `magix2_safety` + `ui` + heartbeat |
| Jetson | **éteinte**, ou n'importe quoi sauf un simulateur |

En simulation, tout le graphe robot migre sur le PC. On ne teste donc **pas** le budget RAM/CPU Jetson : ça reste la mesure du premier boot, §10 du budget de perf.

---

## 4. Ce que la simulation couvre, et ce qu'elle ne couvre pas

Mapping sur les IDs de `work/tests/criteres-proto1.md`. Un `S-xx` est un test simulé, il ne remplit **jamais** la case du `T-xx` correspondant dans un rapport de passage.

| ID test banc | Simulable ? | Ce que le simulateur apporte réellement |
|---|---|---|
| `T-ESTOP` | **Non** | L'e-stop coupe le VIN moteur. C'est de l'électricité. Au mieux on simule le passage en fault du nœud, pas la coupure. |
| `T-HB` | Partiel → `S-HB` | La logique de latch (3 heartbeats d'affilée) se teste très bien. Le délai réel < 500 ms sur WiFi, non. |
| `T-LIDAR-WD` | Partiel → `S-LIDAR-WD` | Couper le lidar simulé valide la règle. Le comportement d'un LD19 qui se déconnecte en USB, non. |
| `T-STOP` | **Oui** → `S-STOP` | Chaîne `Intent.STOP` → `/cmd_vel` zéro, latch jusqu'à `CANCEL`. Pur logiciel. |
| `T-UDEV` | Non | Pas de `/dev` en simulation, par construction. |
| `T-SCAN` | Non | Ce test valide le **driver** `ldlidar_stl_ros2` à 230400 bauds. Aucun simulateur ne le fait tourner. |
| `T-SLAM` | **Oui** → `S-SLAM` | Une carte se construit dans un monde indoor. Le meilleur usage du simulateur sur ce projet. |
| `T-UVC` | Non | Valide `usb_cam` et le MJPEG natif du C270. Une caméra simulée ne dit rien là-dessus. |
| `T-IMU` | Partiel | L'IMU dans la boucle d'odométrie, oui. `0x4A` sur I2C1 et le clock-stretch SH-2, non. |
| `T-GOTO` | **Oui** → `S-GOTO` | `GOTO_NAMED` / `GOTO_POSE` bout en bout, UI et voix comprises. |
| `T-INTENT` | **Oui** → `S-INTENT` | Un seul publieur sur `/magix2/intent`. Vérifiable en CI. |

Lecture : la simulation gagne sur la **navigation et les contrats de topics**, perd sur tout ce qui est **driver, bus et électricité**. C'est exactement le partage attendu — et c'est pour ça qu'elle ne déplace pas la date de proto 1, elle déplace seulement la quantité de code déjà debuggé à cette date.

### Ce qui ne passe par aucun simulateur

Les règles de `magix2_safety` sont des fonctions pures sur des messages. Les tester mérite des **tests unitaires** avec des publieurs bidons et une horloge injectée, pas un simulateur 3D : plus rapide, déterministe, quelques secondes en CI. Le simulateur vient **après**, pour vérifier que ces règles se comportent bien une fois branchées à un vrai Nav2 qui pousse du `/cmd_vel_raw` à 20 Hz.

Faire l'inverse — valider la safety uniquement en simulation — est le piège classique.

---

## 5. Le piège `/clock` (à lire avant d'écrire le premier watchdog)

Identique sur Gazebo et sur Webots.

Le simulateur publie `/clock` (sur Gazebo, via une entrée de `ros_gz_bridge`). Tous les nœuds du graphe simulé doivent tourner avec `use_sim_time:=true`, **y compris le nœud heartbeat du PC**. Sinon les timestamps comparés par `magix2_safety` sont dans deux époques différentes et le watchdog part en fault dès la première seconde.

Trois conséquences pour les timeouts de la spec (500 ms heartbeat, 1 s lidar) :

1. **Simulateur en pause = temps simulé arrêté = watchdog qui ne se déclenche jamais.** Un test de watchdog qui passe pendant que la simulation est en pause ne prouve rien. `S-HB` et `S-LIDAR-WD` tournent à vitesse réelle, jamais en accéléré.
2. Les budgets de latence du §5 du budget de perf (`Intent.STOP` → `/cmd_vel` en < 50 ms) sont des mesures **horloge murale**. Non mesurables en simulation, point.
3. Sur le robot réel il n'y a pas de `/clock`. `use_sim_time` est donc un **paramètre de launch**, jamais une valeur en dur, et les launch de production le laissent à `false`.

Spécifique Gazebo : `/clock` transite par le pont. Si `ros_gz_bridge` meurt, le temps simulé gèle côté ROS et **tous** les watchdogs gèlent avec — sans erreur visible. Le smoke test CI doit vérifier que `/clock` avance, pas seulement qu'il existe.

C'est le seul endroit où la simulation peut activement mentir sur la sécurité. D'où la règle : un `S-xx` vert ne coche pas un `T-xx`.

---

## 6. Montage concret

### Monde

Un `magix2_indoor.sdf` : un sol, quelques murs formant une pièce de taille réaliste (< 100 m², comme l'hypothèse `slam_toolbox` du budget de perf), la platine. Pas de monde géant : une grande carte fait exploser la RAM de `slam_toolbox`, et on veut reproduire les conditions proto, pas les dépasser.

Systèmes gz nécessaires : `physics`, `sensors` (moteur `ogre2`, obligatoire pour le lidar GPU), `scene_broadcaster`, `user_commands`.

### Lidar — calé sur le LD19 / STL-19P

`<sensor type="gpu_lidar">`, paramétré sur les chiffres déjà gelés :

| Paramètre SDF | Valeur | Origine |
|---|---|---|
| `horizontal/samples` | ~450 | 4500 mesures/s ÷ 10 Hz |
| `horizontal/min_angle` … `max_angle` | -π … π | 360° |
| `range/min` | 0.03 | datasheet |
| `range/max` | 12 | datasheet |
| `update_rate` | 10 | datasheet |
| `frame_id` | `laser_link` | spec §5 |

Deux choses valent vraiment le coup d'être simulées, et elles ne coûtent rien :

- **Le masque de FOV.** Le lidar voit à partir de 3 cm alors que la platine fait 25–30 cm de large : une partie du tour tape dans le chassis. Le budget de perf le liste comme risque §7. Le placer dans le monde montre l'angle mort avant d'avoir la platine, et permet d'écrire le masque URDF à l'avance.
- **La sensibilité à l'empattement.** `wheel_radius` 0.0325 m est connu, l'empattement est **à mesurer** entre 12 et 16 cm à réception. Le garder en paramètre permet de mesurer combien `S-SLAM` dérive pour 1 cm d'erreur, donc de savoir d'avance quelle précision la mesure physique exige.

Ce que le simulateur ne reproduit **pas** : le nombre de points variable d'un tour à l'autre, qui fait dropper des scans à `slam_toolbox` (risque §7 du budget de perf). Le lidar simulé en sort un nombre fixe. Le binning côté driver reste donc à vérifier sur le vrai LD19.

### Pont

Avec `gz_ros2_control`, `diff_drive_controller` tourne dans le `controller_manager` chargé par le plugin système : **le chemin de commande ne passe pas par le pont**. On ne bride que les capteurs et l'horloge, soit environ six entrées :

| Topic | Sens | Type |
|---|---|---|
| `/clock` | gz → ROS | `rosgraph_msgs/Clock` |
| `/scan` | gz → ROS | `sensor_msgs/LaserScan` |
| `/imu` | gz → ROS | `sensor_msgs/Imu` |
| image caméra | gz → ROS | `sensor_msgs/Image` |
| `camera_info` | gz → ROS | `sensor_msgs/CameraInfo` |
| TF statiques du monde | gz → ROS | `tf2_msgs/TFMessage` |

### Graphe

Nav2 et `slam_toolbox` ne voient aucune différence : ils consomment `/scan` et publient `/cmd_vel_raw`. `magix2_safety` reste le seul publieur de `/cmd_vel`, et c'est `/cmd_vel` que `diff_drive_controller` consomme. La règle « seul `safety` publie `/cmd_vel` » se teste donc telle quelle.

Backend `rnet` : toujours un stub. Rien de R-Net ne rentre dans le simulateur.

---

## 7. CI

```
DISPLAY= gz sim -s -r --headless-rendering magix2_indoor.sdf
```

EGL, pas de serveur X. Sans GPU sur le runner, le rendu reste possible mais lent : garder le monde CI minimal, et ne pas y ajouter de caméra tant que le lidar suffit.

Ordre de priorité : d'abord les **tests unitaires de `magix2_safety`** (secondes, déterministes, aucun simulateur), ensuite **un seul** smoke test Gazebo qui monte le graphe et vérifie que `/clock` avance, que `/scan` arrive, qu'un `GOTO_POSE` produit du `/cmd_vel`, et qu'un `STOP` le remet à zéro. Un smoke test qui passe vaut mieux que dix scénarios de nav instables.

---

## 8. Vetos

- Un simulateur sur la Jetson : **non**, budget GPU/RAM.
- Un `S-xx` vert dans un `work/tests/rapport-YYYY-MM-DD.md` : **non**. Les rapports de passage sont des mesures de banc.
- Gazebo Classic / `gazebo_ros` / `libgazebo_ros_*.so` : **non**, EOL depuis janvier 2025, inexistant sur Jazzy.
- Jumeau numérique du Magix II 6 roues : **non**, hors proto 1, même statut que le 6WD physique.
- Écriture R-Net simulée : **non**. Phase 1 = zéro écriture, y compris en simulation, pour qu'aucun code d'écriture n'existe et ne parte par accident sur le vrai bus.
- Toucher `magix2_msgs/Intent` pour les besoins du simulateur : **non**, contrat gelé.

---

## 9. Ce que ça change au planning

Rien sur la date cible du 13 novembre 2026 : elle dépend du chassis, pas du logiciel. Ce que ça change, c'est le **risque** — arriver au banc avec Nav2 et la safety déjà debuggés transforme le déballage en session de câblage au lieu d'une session de débogage ROS sous pression.

Décision à prendre : est-ce qu'on investit les quelques jours de setup. Mon avis : oui, parce que le temps d'attente du chassis est de toute façon du temps mort logiciel. Garde-fou : si `gz_ros2_control` dépasse deux jours de mise au point (§2, piège 2), on bascule sur Webots plutôt que de s'entêter.

---

## 10. Sources

- Gazebo Harmonic = pairing officiel de Jazzy, `ros_gz` — [gazebosim.org, Installing Gazebo with ROS](https://gazebosim.org/docs/harmonic/ros_installation/), [gazebosim/ros_gz](https://github.com/gazebosim/ros_gz/blob/ros2/README.md)
- `gz_ros2_control`, plugin `gz_ros2_control/GazeboSimSystem` — [control.ros.org Jazzy](https://control.ros.org/jazzy/doc/gz_ros2_control/doc/index.html), [ros-controls/gz_ros2_control](https://github.com/ros-controls/gz_ros2_control)
- Accrocs plugin sur Jazzy + Harmonic — [issue #390](https://github.com/ros-controls/gz_ros2_control/issues/390), [issue #720](https://github.com/ros-controls/gz_ros2_control/issues/720)
- Gazebo Classic EOL 31/01/2025 ; Harmonic supporté jusqu'en 09/2028 — [Open Robotics Discourse](https://discourse.openrobotics.org/t/gazebo-classic-11-has-reached-end-of-life/48458)
- Nav2 Jazzy sur le Gazebo moderne — [Nav2 quickstart Jazzy](https://docs.nav2.org/jazzy/getting_started/quickstart/quickstart/)
- Rendu headless EGL, `--headless-rendering` — [Gazebo Sim: Headless Rendering](https://gazebosim.org/api/sim/9/headless_rendering.html)
- Repli Webots : `webots_ros2` publié dans Jazzy — [index.ros.org](https://index.ros.org/p/webots_ros2/) ; `webots_ros2_control` maintenu — [index.ros.org](https://index.ros.org/p/webots_ros2_control/) ; Xvfb obligatoire — [cyberbotics/webots #5007](https://github.com/cyberbotics/webots/discussions/5007)
- `use_sim_time` + `/clock` avec `slam_toolbox` et Nav2 — [Husarion](https://husarion.com/tutorials/vulcanexus/webots-rosbot-xl/)
- Contraintes internes : `work/architecture/BUDGET-PERF-PROTO1.md` §4, §5, §7 ; `work/tests/criteres-proto1.md` ; `docs/SPEC-PROTO1.md` §3, §4, §5
