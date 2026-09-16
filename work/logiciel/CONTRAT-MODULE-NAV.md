# Contrat du module de navigation — portable simulation / Yahboom / Magix

**Date :** 2026-09-16
**Demande :** validation de l'architecture « un module qui reçoit lidar, profondeur, odométrie et pose estimée, et produit une consigne (v, ω) ; un pilote par plateforme la traduit ».
**Antérieurs :** [`../../docs/SPEC-PROTO1.md`](../../docs/SPEC-PROTO1.md) §3, [`SIMULATION-PROTO1.md`](SIMULATION-PROTO1.md), [`../architecture/GO-NOGO-RL.md`](../architecture/GO-NOGO-RL.md).

---

## Verdict

**Le découpage est bon, et ce n'est pas une nouveauté : c'est déjà la décision gelée.** `docs/MEMORY.md` dit depuis le 28/08 que les libs R-Net seront un *« futur driver derrière la même interface que le proto »*. La proposition confirme l'architecture, elle ne la change pas.

Trois corrections avant de la figer comme contrat.

---

## Correction 1 — la consigne ne va pas au pilote, elle va à `safety`

La formulation « cette consigne pilote le véhicule virtuel » / « un pilote la traduit en commandes moteurs » saute une couche. La spec §3 règle 5 est explicite :

> Zéro publication `/cmd_vel` autre que `safety`. Nav2 écrit `/cmd_vel_raw`.

Le module de navigation est un producteur au même titre que Nav2 ou la téléop. Il publie donc **`/cmd_vel_raw`**, jamais `/cmd_vel`, jamais le pilote directement.

```
module nav  ──/cmd_vel_raw──>  magix2_safety  ──/cmd_vel──>  pilote plateforme
                                     ^
                          heartbeat, /scan, Intent.STOP
```

Ce n'est pas un détail de plomberie. C'est ce qui fait que les quatre critères bloquants (`T-ESTOP`, `T-HB`, `T-LIDAR-WD`, `T-STOP`) restent vrais quel que soit le module au-dessus — y compris le jour où c'est une politique RL ([`GO-NOGO-RL.md`](../architecture/GO-NOGO-RL.md) §5). Un module qui parle au pilote court-circuite les quatre.

## Correction 2 — sortir « profondeur » du contrat proto 1

La depth est hors spec proto 1, et `GO-NOGO-8GO-SUITE.md` la classe **NO-GO** sur le Jetson à côté de slam + Nav2 (ancrage DAS-S ~626–689 Mo, sans le reste).

L'inscrire dans le contrat d'entrée maintenant a un coût caché : le jour où quelqu'un branche le canal, il tire CUDA/TensorRT sur les 8 Go unifiés sans que ce soit une décision. Le contrat doit donc rendre la depth **explicitement optionnelle et absente en proto 1**, pas « prévue ».

| Entrée | Proto 1 | Type |
|---|---|---|
| Lidar 2D | **requis** | `sensor_msgs/LaserScan` (`/scan`, `laser_link`) |
| Odométrie | **requis** | `nav_msgs/Odometry` |
| Pose estimée | **requis** | TF `map` → `base_link` (`slam_toolbox` seul, cf. GO-NOGO cond. 7 : pas d'AMCL en parallèle) |
| Profondeur | **absent** | canal optionnel, non câblé, à rouvrir avec un critère de test |

## Correction 3 — « ajuster les dimensions et les limites » est vrai jusqu'au Yahboom, faux au-delà

C'est le point sur lequel la proposition est trop optimiste. De la simulation au Yahboom, oui : même cinématique diff-drive, même ordre de grandeur, le pilote est une conversion d'unités vers `$speed:G,0,D,0#`. Du Yahboom au Magix, non, pour trois raisons de nature différente.

**a. Sur le fauteuil, on ne commande pas une vitesse.** Le chemin R-Net ouvert (`can2RNET`, `open-rnet`) est de l'**émulation de joystick** : des trames type `02000000#XxYy` à 100 Hz portant une déflexion X/Y, plus un heartbeat JSM. Le Power Module applique ensuite **son** profil de vitesse et **ses** rampes d'accélération, que l'on ne choisit pas et que l'on n'observe pas directement. Le « pilote » du fauteuil n'est donc pas une conversion d'unités : c'est la calibration d'une boucle fermée non linéaire dont on ne tient pas les gains. Ça se mesure au banc, ça ne se déduit pas.

**b. La cinématique change de modèle, pas d'échelle.** Diff-drive 2 roues + casters contre 6 roues, 2 pneus + 4 omni. Le glissement latéral des omni n'existe pas dans le modèle diff-drive. Ce n'est pas un paramètre à retoucher, c'est un autre modèle.

**c. La dynamique ne se met pas à l'échelle.** ~2–3 kg à 0,3 m/s contre ~132 kg. `GO-NOGO-PROTO1.md` le dit déjà pour le temps de réaction :

> T-HB 500 ms à 0,3 m/s proto ≈ 15 cm d'errance max après perte WiFi. Acceptable proto miniature. **Pas** transposable tel quel au Magix (masse, vitesse). On ne gèle pas 500 ms pour le fauteuil.

La distance d'arrêt n'est pas une limite à régler, c'est une contrainte de sécurité à re-dériver.

**Conséquence sur le vocabulaire :** ce qui se transfère, ce sont les **algorithmes** et le **contrat de topics**. Les limites, la dynamique et la calibration actionneur sont à **re-mesurer** par plateforme, pas à « ajuster ». Un rapport de banc par plateforme, comme `work/tests/`.

---

## Ce qui rend le transfert réellement vrai

Le cœur du module doit être une **fonction pure**, ROS relégué à un adaptateur mince :

```
plan(observation, params) -> (v, omega)
```

Trois bénéfices concrets :

1. **Testable sans ROS et sans simulateur.** La même suite de tests tourne pour les trois plateformes, avec trois jeux de `params`. Les différences de plateforme deviennent une table de paramètres **sous test**, au lieu d'être du code dupliqué.
2. **Substituable.** Une politique RL prend la place de `plan()` sans toucher au reste ([`GO-NOGO-RL.md`](../architecture/GO-NOGO-RL.md)).
3. **Vérifiable en propriété.** Sur des observations aléatoires, la sortie reste dans l'enveloppe des limites — le genre de test qui attrape ce que les cas nominaux ratent.

### Les limites s'appliquent dans le pilote, pas seulement dans le module

Le module borne sa sortie ; le pilote **re-borne** la sienne. C'est de la défense en profondeur, et c'est précisément parce que le module est la pièce qu'on remplacera (par une politique RL, par un autre planner) que le pilote ne doit rien lui faire confiance. Chaque pilote possède : son clamp de limites, sa rampe d'accélération, sa conversion d'unités, et pour le fauteuil sa calibration déflexion → vitesse.

### Comportement en entrée dégradée — à définir, pas à laisser implicite

Que sort le module quand la pose `map` → `base_link` est absente ou périmée (slam non convergé, TF en retard) ? Le laisser implicite, c'est laisser chaque plateforme inventer sa réponse. À figer dans le contrat, au même titre que les timeouts de `safety`.

À noter : `safety` couvre le lidar muet et le heartbeat, **pas** la pose périmée. C'est un trou du contrat actuel, pas un trou du module.

---

## Question ouverte

**D'où vient l'odométrie sur le Magix ?** Sur le Yahboom, des encodeurs moteurs. Sur le fauteuil, la phase 1 est listen-only sur R-Net, et rien ne dit à ce stade que le bus expose une odométrie exploitable. Si elle n'existe pas, le module tourne sur l'IMU et le lidar seuls, ce qui change ses entrées requises — donc son contrat. À trancher avant d'écrire la couche fauteuil, pas après.

---

## Verdict figé

| Point | Statut |
|---|---|
| Un module nav portable, pilotes par plateforme | **OK**, déjà la décision `MEMORY.md` |
| Sortie du module | `/cmd_vel_raw`, **jamais** `/cmd_vel` ni le pilote |
| Entrées proto 1 | lidar + odom + pose. **Depth absente**, canal optionnel |
| « Ajuster limites et dynamique » | OK sim → Yahboom. **Re-mesurer** Yahboom → Magix |
| Cœur du module | fonction pure, ROS en adaptateur, limites re-bornées dans le pilote |
| Pose périmée | comportement **à définir**, trou du contrat actuel |
| Odométrie Magix | **question ouverte** |

## Sources

- Émulation joystick R-Net, trames `02000000#XxYy` à 100 Hz + heartbeat JSM — [redragonx/can2RNET](https://github.com/redragonx/can2RNET), [redragonx/open-rnet](https://github.com/redragonx/open-rnet), [wikilab Can2RNET](https://wikilab.myhumankit.org/index.php?title=Projets:Can2RNET)
- Internes : `docs/SPEC-PROTO1.md` §3 règle 5 et §5 ; `docs/MEMORY.md` ; `work/architecture/GO-NOGO-PROTO1.md` cond. 7 et risques résiduels ; `work/architecture/GO-NOGO-8GO-SUITE.md` ; `work/tests/criteres-proto1.md`
