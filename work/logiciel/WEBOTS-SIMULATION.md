# Webots pour Magix2 — périmètre, montage, vetos

Réponse à : « comment utiliser Webots pour ce projet ».

Aligné sur `docs/SPEC-PROTO1.md`, `work/tests/criteres-proto1.md`, `work/architecture/BUDGET-PERF-PROTO1.md`.

---

## 1. Verdict

Webots sert à **une seule chose** ici : un banc d'intégration logiciel **sur le PC de supervision**, pour écrire et faire tourner `magix2_safety`, `slam_toolbox`, Nav2 et le chemin `Intent` → `GOTO` **avant** que la platine Yahboom arrive de Chine.

Le chemin critique est le chassis (CN → FR, DDU). Sans simulateur, le logiciel attend le matériel. Avec Webots, les deux avancent en parallèle et le jour du déballage on débogue du câblage, pas des topics.

Ce que Webots **n'est pas** dans ce projet :

- pas un substitut aux 4 critères bloquants (`T-ESTOP`, `T-HB`, `T-LIDAR-WD`, `T-STOP`) — voir §4 ;
- pas un truc qui tourne sur la Jetson — voir §3 ;
- pas un modèle du Magix II 6 roues. On simule la **platine 2WD**, rien d'autre. Un jumeau numérique du fauteuil est hors proto 1, comme le 6WD physique.

Licence Apache 2.0, coût 0 €, compatible avec la contrainte « open source uniquement ». Le coût réel est du temps d'ingénieur et un PC avec un GPU qui tient OpenGL.

---

## 2. Pourquoi Webots et pas Gazebo

Le pairing officiel de Jazzy est Gazebo Harmonic, donc la question est légitime. Je prends quand même Webots pour ce périmètre précis, pour une raison : `webots_ros2_driver` mappe les capteurs **depuis l'URDF**, via un bloc `<webots>` dans le fichier que l'on écrit de toute façon pour le robot réel. Pas de process de pont séparé, pas de fichier de mapping en double à maintenir. Sur un scope 2WD + un lidar 2D + une UVC, c'est une journée de setup contre plusieurs.

Gazebo redeviendrait le bon choix le jour où on attaque les marches et les trous : physique de contact et depth plus sérieuses. C'est hors proto 1, on tranchera à ce moment-là. Installer Webots aujourd'hui n'engage rien.

Installation : `sudo apt install ros-jazzy-webots-ros2` (paquet publié dans Jazzy depuis le 18/02/2025), qui télécharge Webots au premier lancement s'il n'est pas déjà présent.

---

## 3. Où ça tourne — veto Jetson

**Webots ne tourne jamais sur l'Orin Nano.** Ce n'est pas une préférence, c'est le budget de perf.

Webots passe par OpenGL pour le rendu des capteurs simulés (lidar, caméra, range-finder), même en mode headless : il lui faut un display, réel ou virtuel. Or `BUDGET-PERF-PROTO1.md` §4 fige le GPU à **0** en compute et **0** en display, parce que les 8 Go sont unifiés et que tout ce que le GPU touche sort de la même RAM que ROS. Lancer Webots à côté du graphe, c'est le `std::bad_alloc` de `slam_toolbox` déjà documenté.

Répartition :

| Machine | Rôle en simulation |
|---|---|
| PC de supervision | Webots + `webots_ros2_driver` + `slam_toolbox` + Nav2 + `magix2_safety` + `ui` + heartbeat |
| Jetson | **éteinte**, ou n'importe quoi sauf Webots |

En simulation, tout le graphe robot migre sur le PC. On ne teste donc **pas** le budget RAM/CPU Jetson : ça reste la mesure du premier boot, §10 du budget de perf.

---

## 4. Ce que la simulation couvre, et ce qu'elle ne couvre pas

Mapping sur les IDs de `work/tests/criteres-proto1.md`. Un `S-xx` est un test simulé, il ne remplit **jamais** la case du `T-xx` correspondant dans un rapport de passage.

| ID test banc | Simulable ? | Ce que Webots apporte réellement |
|---|---|---|
| `T-ESTOP` | **Non** | L'e-stop coupe le VIN moteur. C'est de l'électricité. Au mieux on simule le passage en fault du nœud, pas la coupure. |
| `T-HB` | Partiel → `S-HB` | La logique de latch (3 heartbeats d'affilée) se teste très bien. Le délai réel < 500 ms sur WiFi, non. |
| `T-LIDAR-WD` | Partiel → `S-LIDAR-WD` | Couper le lidar simulé valide la règle. Le comportement d'un LD19 qui se déconnecte en USB, non. |
| `T-STOP` | **Oui** → `S-STOP` | Chaîne `Intent.STOP` → `/cmd_vel` zéro, latch jusqu'à `CANCEL`. Pur logiciel, aucun besoin de matériel. |
| `T-UDEV` | Non | Pas de `/dev` en simulation, par construction. |
| `T-SCAN` | Non | Ce test valide le **driver** `ldlidar_stl_ros2` à 230400 bauds. Webots ne le fait pas tourner. |
| `T-SLAM` | **Oui** → `S-SLAM` | Une carte se construit dans un monde indoor. Le meilleur usage de Webots sur ce projet. |
| `T-UVC` | Non | Valide `usb_cam` et le MJPEG natif du C270. Une caméra Webots ne dit rien là-dessus. |
| `T-IMU` | Partiel | L'IMU dans la boucle d'odométrie, oui. `0x4A` sur I2C1 et le clock-stretch SH-2, non. |
| `T-GOTO` | **Oui** → `S-GOTO` | `GOTO_NAMED` / `GOTO_POSE` bout en bout, UI et voix comprises. |
| `T-INTENT` | **Oui** → `S-INTENT` | Un seul publieur sur `/magix2/intent`. Vérifiable en CI. |

Lecture : Webots gagne sur la **navigation et les contrats de topics**, perd sur tout ce qui est **driver, bus et électricité**. C'est exactement le partage attendu — et c'est pour ça que le simulateur ne déplace pas la date de proto 1, il déplace seulement la quantité de code déjà debuggé à cette date.

### Ce qui ne passe pas par Webots du tout

Les règles de `magix2_safety` sont des fonctions pures sur des messages. Les tester mérite des **tests unitaires** avec des publieurs bidons et une horloge injectée, pas un simulateur 3D : c'est plus rapide, déterministe, et ça tourne en CI en quelques secondes. Webots vient **après**, pour vérifier que ces règles se comportent bien une fois branchées à un vrai Nav2 qui pousse du `/cmd_vel_raw` à 20 Hz.

Faire l'inverse — valider la safety uniquement en simulation — est le piège classique.

---

## 5. Le piège `/clock` (à lire avant d'écrire le premier watchdog)

`webots_ros2_driver` publie `/clock`. Tous les nœuds du graphe simulé doivent tourner avec `use_sim_time:=true`, **y compris le nœud heartbeat du PC**. Sinon les timestamps comparés par `magix2_safety` sont dans deux époques différentes et le watchdog part en fault dès la première seconde.

Trois conséquences pour les timeouts de la spec (500 ms heartbeat, 1 s lidar) :

1. **Simulateur en pause = temps simulé arrêté = watchdog qui ne se déclenche jamais.** Un test de watchdog qui passe pendant que la simulation est en pause ne prouve rien. Les tests `S-HB` et `S-LIDAR-WD` tournent en `--mode=realtime`, jamais en `--mode=fast`.
2. Les budgets de latence du §5 du budget de perf (`Intent.STOP` → `/cmd_vel` en < 50 ms) sont des mesures **horloge murale**. Non mesurables en simulation, point.
3. Sur le robot réel il n'y a pas de `/clock`. Il faut donc que `use_sim_time` soit un **paramètre de launch**, jamais une valeur en dur, et que les launch de production le laissent à `false`.

C'est le seul endroit où la simulation peut activement mentir sur la sécurité. D'où la règle : un `S-xx` vert ne coche pas un `T-xx`.

---

## 6. Montage concret

### Monde

Un monde `magix2_indoor.wbt` : un sol, quelques murs formant une pièce de taille réaliste (< 100 m², comme l'hypothèse `slam_toolbox` du budget de perf), la platine. Pas de monde géant : une grande carte fait exploser la RAM de `slam_toolbox`, et on veut reproduire les conditions proto, pas les dépasser.

`basicTimeStep` à 16 ms ou 32 ms. Le lidar tourne à 10 Hz, donc son `refreshRate` doit tomber sur un multiple du pas de temps.

### Lidar — calé sur le LD19 / STL-19P

Le nœud `Lidar` de Webots se paramètre pour coller aux chiffres déjà gelés dans la spec :

| Champ Webots | Valeur | Origine |
|---|---|---|
| `fieldOfView` | 6.28 (360°) | LD19 |
| `numberOfLayers` | 1 | lidar 2D |
| `horizontalResolution` | ~450 | 4500 mesures/s ÷ 10 Hz |
| `minRange` | 0.03 | datasheet |
| `maxRange` | 12 | datasheet |
| nom du device | `laser_link` | frames de la spec §5 |

Deux choses valent vraiment le coup d'être simulées ici, et elles ne coûtent rien :

- **Le masque de FOV.** Le lidar voit à partir de 3 cm alors que la platine fait 25–30 cm de large : une partie du tour tape dans le chassis. Le budget de perf le liste comme risque §7. Placer le lidar dans le monde et regarder le `/scan` montre l'angle mort avant d'avoir la platine, et permet d'écrire le masque URDF à l'avance.
- **Le nombre de points variable.** `slam_toolbox` droppe les scans dont le nombre de points change. Webots en produit un nombre fixe, donc il ne reproduit **pas** le défaut — raison de plus pour que le binning côté driver reste vérifié sur le vrai lidar.

### Base

`diff_drive`, `wheel_radius` 0.0325 m, empattement paramétré, **pas** en dur : il est à mesurer entre 12 et 16 cm à réception. Le mettre en paramètre permet de tester la sensibilité de `S-SLAM` à une erreur d'empattement, ce qui dit d'avance combien la mesure physique doit être précise.

### Graphe

Nav2 et `slam_toolbox` ne voient aucune différence : ils consomment `/scan` et publient `/cmd_vel_raw`. `magix2_safety` reste le seul publieur de `/cmd_vel`, et c'est `/cmd_vel` que le driver Webots consomme. La règle « seul `safety` publie `/cmd_vel` » se teste donc telle quelle en simulation.

Backend `rnet` : toujours un stub. Rien de R-Net ne rentre dans le simulateur.

---

## 7. CI

Webots tourne sans écran via `xvfb-run` :

```
xvfb-run --auto-servernum webots --batch --stdout --stderr --minimize --mode=realtime magix2_indoor.wbt
```

Deux réserves honnêtes :

- Sans GPU, le rendu tombe sur Mesa en logiciel. Pour un lidar 2D et une petite pièce c'est jouable ; ça ne le sera plus le jour où on ajoute des caméras. Garder le monde CI minimal.
- `--no-rendering` allège la vue 3D principale, mais les capteurs à rendu continuent de passer par OpenGL : le display virtuel reste obligatoire. Je n'ai pas vérifié sur banc ce que `--no-rendering` coûte exactement en fidélité lidar — à mesurer avant de l'activer en CI.

Ordre de priorité pour la CI : d'abord les tests unitaires de `magix2_safety` (secondes, déterministes, aucun simulateur), ensuite un seul smoke test Webots qui monte le graphe, vérifie que `/scan` arrive, qu'un `GOTO_POSE` produit du `/cmd_vel`, et qu'un `STOP` le remet à zéro. Un smoke test qui passe vaut mieux que dix scénarios de nav instables.

---

## 8. Vetos

- Webots sur la Jetson : **non**, budget GPU/RAM.
- Un `S-xx` vert dans un `work/tests/rapport-YYYY-MM-DD.md` : **non**. Les rapports de passage sont des mesures de banc.
- Jumeau numérique du Magix II 6 roues : **non**, hors proto 1, même statut que le 6WD physique.
- Écriture R-Net simulée : **non**. Phase 1 = zéro écriture, y compris en simulation, pour qu'aucun code d'écriture n'existe et ne parte par accident sur le vrai bus.
- Toucher `magix2_msgs/Intent` pour les besoins du simulateur : **non**, contrat gelé.

---

## 9. Ce que ça change au planning

Rien sur la date cible du 13 novembre 2026 : elle dépend du chassis, pas du logiciel. Ce que ça change, c'est le **risque** — arriver au banc avec Nav2 et la safety déjà debuggés transforme le déballage en session de câblage au lieu d'une session de débogage ROS sous pression.

Décision à prendre : est-ce qu'on investit les quelques jours de setup. Mon avis : oui, parce que le temps d'attente du chassis est de toute façon du temps mort logiciel.

---

## 10. Sources

- `webots_ros2` publié dans Jazzy, `ros-jazzy-webots-ros2` — [index.ros.org](https://index.ros.org/p/webots_ros2/), [CHANGELOG Jazzy](http://docs.ros.org/en/ros2_packages/jazzy/api/webots_ros2/__CHANGELOG.html)
- Headless / CI, Xvfb obligatoire pour les capteurs à rendu — [cyberbotics/webots #5007](https://github.com/cyberbotics/webots/discussions/5007), [#5655](https://github.com/cyberbotics/webots/discussions/5655)
- Repli Mesa sans GPU en conteneur — [cyberbotics/webots #6921](https://github.com/cyberbotics/webots/discussions/6921)
- `use_sim_time` + `/clock` avec `slam_toolbox` et Nav2 — [Husarion, Webots + SLAM Toolbox](https://husarion.com/tutorials/vulcanexus/webots-rosbot/), [Husarion, Webots + Nav2](https://husarion.com/tutorials/vulcanexus/webots-rosbot-xl/)
- Contraintes internes : `work/architecture/BUDGET-PERF-PROTO1.md` §4, §5, §7 ; `work/tests/criteres-proto1.md` ; `docs/SPEC-PROTO1.md` §3, §4, §5
