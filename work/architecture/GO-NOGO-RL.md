# Go / no-go — RL et plateforme de simulation, cap Jetson Orin 8 Go

**Date :** 2026-09-16
**Demande :** « quelle plateforme de simu pour passer ensuite sur Jetson Orin 8 Go, pour tester le module de nav voire du RL ». **Pas un achat.**
**Antérieurs :** [`GO-NOGO-8GO-SUITE.md`](GO-NOGO-8GO-SUITE.md) (règle des 8 Go), [`../logiciel/SIMULATION-PROTO1.md`](../logiciel/SIMULATION-PROTO1.md) (Gazebo Harmonic pour proto 1).

---

## Verdict

**Il n'y a pas une plateforme, il y en a trois, à trois étages différents.** Chercher celle qui fait à la fois la nav et le RL est ce qui coûte cher.

| Étage | Rôle | Plateforme |
|---|---|---|
| 1 — entraînement RL | des millions de pas, vite, sans ROS | **gym 2D raycasting**, pas Gazebo, pas Isaac |
| 2 — validation | politique dans le vrai graphe ROS, avec `safety` | **Gazebo Harmonic** (déjà décidé) |
| 3 — cible | inférence temps réel | **Jetson Orin Nano 8 Go, CPU, GPU off** |

Et un préalable qui ferme une lecture de la question : **aucun simulateur ne tourne sur un Jetson**. Isaac Sim n'est supporté sur aucun Jetson, Thor compris. Le partage est explicite chez NVIDIA : entraînement sur station RTX, déploiement de la politique sur Jetson. Le veto « pas de simulateur sur l'Orin » de `SIMULATION-PROTO1.md` §3 vaut donc aussi pour Isaac, et pour une raison de plus.

---

## 1. Étage 1 — entraîner

Le RL de navigation consomme des **millions de pas d'environnement**. C'est le seul chiffre qui compte pour choisir.

| Plateforme | Facteur temps réel | Verdict entraînement |
|---|---|---|
| Gazebo Harmonic | ~1–10× | **Non.** Des semaines pour ce qu'un gym fait en heures. |
| Webots | ~1–10× | **Non**, même raison. |
| Isaac Lab / Isaac Sim | milliers d'envs en parallèle sur GPU | Oui, mais voir §4 — c'est un achat de station. |
| **Gym 2D raycasting** | **~800×**, ~140 Mo de RAM | **Oui**, et ça tourne sur n'importe quel portable. |

Pour une politique **lidar 2D → `cmd_vel`** sur une platine diff-drive, un moteur physique 3D ne sert à rien : ce qu'il faut simuler, c'est un raycast sur une grille d'occupation et une cinématique diff-drive. Deux ou trois cents lignes, ou une base existante (`PIC4rl-gym` côté ROS 2, ou une des implémentations légères de la littérature nav-RL).

Ce que ça donne : l'observation d'entraînement est exactement le vecteur de ~450 faisceaux du LD19 (spec §4), la sortie exactement le `Twist` que `safety` filtre. Aucune traduction entre l'entraînement et le robot.

> Note de confiance : le facteur ~800× et les 140 Mo viennent d'une implémentation publiée, pas d'une mesure maison. L'ordre de grandeur est solide (raycast 2D contre moteur physique 3D), le chiffre exact dépendra de notre implémentation.

## 2. Étage 2 — valider

C'est là que Gazebo Harmonic reste le bon choix, et ça ne contredit pas la note précédente : **Gazebo n'est pas là pour entraîner, il est là pour brancher**.

Une politique qui marche dans un gym 2D n'a rien prouvé sur ce qui casse en vrai : la latence DDS, le `use_sim_time`, les costmaps Nav2, l'ordonnancement à 20 Hz, et surtout le comportement de `magix2_safety` quand la politique sort un `Twist` aberrant. Gazebo + `gz_ros2_control` fait tourner la politique dans le **vrai graphe**, avec les vrais fichiers URDF et `diff_drive_controller` qui partiront sur la Jetson.

Ordre : gym pour apprendre, Gazebo pour intégrer, banc pour mesurer. Sauter l'étage 2, c'est découvrir les problèmes de timing sur un robot de 132 kg.

## 3. Étage 3 — le budget Jetson

C'est le point qui décide si le RL est jouable du tout, et notre propre règle y répond déjà.

`GO-NOGO-8GO-SUITE.md` dit : *« No-go quand ? DNN (DAS, YOLO, BEV, VLM) sur le Jetson avec slam+Nav2, ou dans `safety` »*, et *« ce n'est pas une caméra de plus, c'est un DNN (CUDA/TensorRT) sur le robot en parallèle de slam + Nav2 »*.

Appliqué au RL, ça se scinde en deux cas très différents :

| Politique | Ce qu'elle tire | Verdict 8 Go |
|---|---|---|
| **Lidar 2D → `Twist`** : MLP ou petit CNN 1D sur ~450 faisceaux, ~10⁵–10⁶ paramètres | Quelques Mo de poids. **Aucun contexte CUDA** : ONNX Runtime sur les 6 A78 tient 20 Hz sans transpirer. | **GO.** Reste GPU-off, donc reste dans l'enveloppe gelée. |
| **Vision → `Twist`** : politique bout-en-bout sur RGB ou depth | CUDA + TensorRT + backbone. Ancrage du doc 8 Go : DAS-S ~626–689 Mo **sans** le reste. | **NO-GO**, exactement le cas déjà tranché. |

Autrement dit : **le RL de navigation n'a pas besoin du GPU de la Jetson.** C'est contre-intuitif mais c'est le point important — ce qui coûte sur RAM unifiée, ce n'est pas le réseau, c'est le contexte CUDA. Une politique lidar tourne en CPU et ne réveille jamais les 67 TOPS.

Les 67 TOPS restent donc un stock mort après proto 1 aussi, sauf à passer à une politique vision, ce qui rebascule sur le dossier NX 16 Go déjà écrit.

## 4. Quand Isaac Lab devient le bon choix

Pas maintenant, mais la question est légitime pour la suite.

Isaac Lab est le bon outil le jour où l'une de ces deux conditions tombe :

1. **Politique vision / depth** — Isaac Sim rend des images photoréalistes en parallèle, un gym 2D n'a rien à offrir.
2. **Marches, trous, et la cinématique 6 roues du Magix II** (2 pneus + 4 omni) — là il faut de la vraie dynamique de contact, et c'est précisément ce que les planners classiques ne savent pas faire. C'est le scénario où le RL gagne réellement quelque chose sur ce projet.

Le coût n'est pas logiciel, il est matériel : Isaac Sim 5 demande une **RTX 4080 / 16 Go de VRAM minimum**, et Isaac Lab recommande **32 Go de RAM système et 16 Go de VRAM**. Les A100 / H100 ne sont même pas supportées, faute de RT Cores. C'est une station de travail, hors de l'enveloppe proto (1 000 € visés, 1 500 € plafond).

Conséquence pour l'acheteur : **pas de ligne station RTX**, au même titre qu'il n'y a pas de ligne NX 16 Go. On ouvre le dossier quand un critère de test figé l'exige, pas avant. Location de GPU cloud à l'heure = repli à chiffrer le moment venu, bien moins cher qu'une station pour quelques campagnes d'entraînement.

## 5. RL et `safety` — non négociable

Ça finit sur un fauteuil de 132 kg avec une personne dedans. Une politique apprise est exactement le genre de composant qu'on met **derrière** une couche de sécurité déterministe, jamais **dedans**.

Bonne nouvelle : notre architecture le fait déjà correctement, sans rien changer. La politique RL prend la place de Nav2 comme producteur de `/cmd_vel_raw` ; `magix2_safety` reste le seul publieur de `/cmd_vel` et garde ses quatre règles (heartbeat, lidar muet, STOP, e-stop). Une politique qui part en vrille est bornée par la même barrière qu'un Nav2 qui part en vrille.

Deux vetos qui en découlent :

- **Aucun réseau appris dans `safety`.** Les règles de `safety` restent des fonctions pures testables unitairement. C'est déjà la ligne du doc 8 Go (« ou dans `safety` ») ; elle vaut mot pour mot pour le RL.
- **Aucune politique RL sur le bus R-Net**, à plus forte raison en phase 1 où l'écriture est à zéro.

## 6. Ce que je recommande

**Pour proto 1 : pas de RL.** Nav2 classique résout le GOTO indoor sur une pièce cartographiée ; le RL n'y apporte rien qu'on puisse mesurer, et coûte la propriété qu'on veut le plus à ce stade — pouvoir expliquer pourquoi le robot a tourné à gauche. Les critères `T-GOTO` et `T-SLAM` se cochent avec Nav2.

**Ce qui vaut le coup dès maintenant**, parce que ça ne coûte presque rien et que ça prépare tout le reste : monter le gym 2D raycasting pendant l'attente du chassis, avec la même observation (450 faisceaux) et la même sortie (`Twist`) que le robot réel. Même si la politique ne sert jamais, le gym sert de banc de non-régression rapide pour Nav2 et pour `safety`.

**Le RL se justifie à partir des marches / du 6 roues**, et à ce moment-là c'est Isaac Lab + station RTX, avec un critère de test écrit avant l'achat.

## 7. Vetos

- Simulateur sur la Jetson, Isaac compris : **non**. Isaac Sim n'est supporté sur aucun Jetson.
- Entraîner du RL dans Gazebo ou Webots : **non**, mauvais outil, des semaines pour rien.
- Politique vision bout-en-bout sur Orin Nano 8 Go à côté de slam + Nav2 : **non**, cas déjà tranché dans `GO-NOGO-8GO-SUITE.md`.
- Réseau appris dans `magix2_safety` : **non**, jamais.
- Ligne « station RTX » ou « NX 16 Go » au devis : **non**, pas avant un critère figé.
- RL dans les critères de passage proto 1 : **non**, hors scope comme les marches et le 360.

## 8. Sources

- Isaac Sim non supporté sur Jetson (Thor compris), partage entraînement station RTX / inférence Jetson — [DesignSpark, IsaacSim & IsaacLab on Jetson AGX Thor](https://www.rs-online.com/designspark/isaacsim-and-isaaclab-on-nvidia-jetson-agx-thor), [NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/using-jetson-orin-devices-for-isaac-lab-rl-training-in-headless-mode/368193)
- Isaac Sim 5 : RTX 4080 / 16 Go VRAM minimum ; Isaac Lab : 32 Go RAM + 16 Go VRAM recommandés ; A100/H100 non supportées — [Isaac Lab, Local Installation](https://isaac-sim.github.io/IsaacLab/v2.2.0/source/setup/installation/index.html), [NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/is-8gb-vram-enough-for-basic-isaac-sim-5-1-usage/373600)
- Simulateur lidar 2D par raycasting, facteur ~815× temps réel, ~140 Mo — [Train a Real-world Local Path Planner in One Hour](https://arxiv.org/pdf/2305.04180)
- Gym nav-RL sous ROS 2 — [PIC4rl-gym](https://arxiv.org/pdf/2211.10714)
- Contraintes internes : [`GO-NOGO-8GO-SUITE.md`](GO-NOGO-8GO-SUITE.md), [`BUDGET-PERF-PROTO1.md`](BUDGET-PERF-PROTO1.md) §4, [`../../docs/SPEC-PROTO1.md`](../../docs/SPEC-PROTO1.md) §3–§5, [`../tests/criteres-proto1.md`](../tests/criteres-proto1.md)
