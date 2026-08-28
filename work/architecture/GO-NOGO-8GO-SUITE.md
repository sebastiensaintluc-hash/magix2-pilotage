# Go / no-go — jusqu’où les 8 Go tiennent (après proto 1)

**Date :** 2026-08-28
**Auteur :** Architecte (design authority)
**Demande :** doute Sébastien (ceinture ToF, caméra arrière, 360/BEV, Co-Pilot Magix, GPU). **Pas un achat.**
**Proto 1 gelé :** GO conditionnel, ~4,8–5,5 / 8 Go, GPU off. [`GO-NOGO-PROTO1.md`](GO-NOGO-PROTO1.md)

## Verdict

**Les 8 Go tiennent tant que le graphe de traction reste GPU-off et que BEV / depth / 360 restent de la supervision (PC).**

Le no-go n’est pas « une caméra de plus ». C’est **un DNN (CUDA/TensorRT) sur le robot en parallèle de slam + Nav2**, et surtout **dans `safety`**.

Le swap n’est **pas** « acheter un NX 16 Go par peur ». Par défaut on **déporte** BEV/depth/360 sur le PC (déjà Vosk / Piper / RViz / UI). Un **Orin NX 16 Go** (pas un NX 8 Go) ne se justifie que si un **critère de test figé** exige de la perception GPU **onboard, sans WiFi**, dans le chemin traction (marches, collision 360 qui coupe les moteurs).

## Règle

| Sur le robot (Jetson) | Sur le PC (déjà dans la spec) |
|---|---|
| `safety`, lidar, slam 2D, Nav2, `cmd_vel`, ToF si bumper | UI, voix, RViz, **BEV / 360 / depth d’affichage** |
| Doit marcher **coupure WiFi** (T-HB → Twist 0) | Peut disparaître avec le WiFi |

Si la feature n’a pas à arrêter le fauteuil toute seule → PC. Si elle a à arrêter le fauteuil sans le PC → robot, et là on compte les Go.

## 8 Go : GO (même enveloppe proto 1)

Enveloppes **en plus** du ~4,8–5,5 Go déjà budgétés. GPU **off**.

| Lot | RAM en plus | Verdict |
|---|---|---|
| Caméra arrière UVC MJPEG (même contrat que C270) | +50–80 Mo, +10–25 % d’un cœur | **GO** si compressé, pas de flood DDS |
| Ceinture ToF (VL53L5/L9, ranging, pas une carte 3D) | +20–60 Mo | **GO** ; peut entrer dans `safety` |
| R-Net listen-only puis driver write | +30–80 Mo | **GO** (CAN, pas de vision) |
| Co-Pilot Magix = même `Intent` + nav + safety sur le fauteuil | ~0 | **GO** : c’est proto 1 porté, pas un VLM |
| Alertes extérieur lidar (même LD19, pas de DNN) | ~0 | **GO** |

Ça couvre indoor Magix, une 2ᵉ caméra de supervision, un bumper ToF, le bus. **Pas** la vue 360 savante.

## 8 Go : NO-GO (on-device, avec le graphe proto 1)

RAM unifiée : CUDA + TensorRT + slam + Nav2 mangent le même tas. Marge restante ~2,5–3,5 Go **avant** fragmentation.

| Lot | Pourquoi |
|---|---|
| Depth-Anything Small **sur le Jetson** en même temps que slam+Nav2 | DAS-S TensorRT seul ~0,6–1,2 Go + contexte CUDA. Tient en **labo isolé** (Nano 8 Go, 308², ~1,2 Go). **Pas** à côté de 5 Go de ROS. Build TensorRT sur 8 Go unifiés = OOM documenté. |
| Depth-Anything Base / Large | Explicite : pas sur Orin 8 Go. |
| YOLO / Isaac ROS « pour voir » | Même conflit CUDA vs slam. |
| 360 / BEV onboard (BEVFormer, BEVFusion, multi-cam) | BEVFormer tiny déjà ~2 Go GPU ailleurs ; BEVFormer sur Orin Nano Super = pas un stack prêt. |
| VLM / « Co-Pilot+ » on-device (LLM+image) | Hors 8 Go, hors proto, hors safety. |

Chiffres d’ancrage (pas des `ps` Magix) : DAS-S ~626–689 Mo inférence 308–518² (IRCVLab, Orin 8 Go) ; wrapper ROS DA3-S ~1,2 Go / 308² sur Nano 8 Go ; DA3-S 518² validé **NX 16 Go**. BEVFormer sur Nano Super : échec plugins TRT (forum NVIDIA, 2025).

## Où ça bascule

```
proto 1 GPU off          GO 8 Go          (gelé)
+ UVC arrière / ToF      GO 8 Go
+ R-Net                  GO 8 Go
+ BEV/depth/360 sur PC   GO 8 Go robot    ← défaut
+ DAS/YOLO sur Jetson    NO-GO 8 Go
+ BEV onboard            NO-GO 8 Go
+ depth dans safety      NO-GO 8 Go       ← seul vrai trigger NX 16 Go
                       (ou ToF/stereo sans DNN)
```

**Trigger unique pour un SoM plus gros :** un critère figé du type *« trou / marche → Twist 0 même si le PC est down »* **et** la solution retenue est un **DNN** (Depth-Anything, BEV) plutôt qu’un ToF / stereo classique.

Les marches, la spec proto 1 l’a déjà dit : LD19 + UVC ne voient pas un vide. Ce n’est pas un argument pour 16 Go **maintenant**. C’est un argument pour, plus tard, **ToF dans safety** (reste 8 Go) ou **depth onboard** (là, NX 16 Go).

## NX 16 Go ou déporter ?

| Besoin | Chemin | Compute |
|---|---|---|
| Voir 360 / BEV / depth à l’écran | **Déporter PC** (UVC déjà streamée) | **8 Go** |
| Ceinture, caméra arrière, R-Net, Co-Pilot Magix GPU-off | Rester Nano Super | **8 Go** |
| Depth/BEV **dans** `safety`, latence bornée, sans WiFi | **Orin NX 16 Go** (pas NX 8 Go : même piège unifié) | plus tard, pas A1 |
| Demo DAS isolée sur le kit | Launch à part, GPU on, slam/nav down | 8 Go labo, pas Magix |

NX 8 Go : **non**. Même RAM unifiée trop juste dès qu’on allume le GPU à côté de Nav2. Si on change de silicium, c’est **16 Go** (NX ou AGX), et un carrier qui tient 24 V / CAN — pas le kit 19 V proto.

Le kit Super SODIMM n’est pas un bouton « mettre un NX ». Changer de SoM plus tard = autre carte. D’où : **on n’achète pas le 16 Go par anticipation**. On garde le Super 8 Go pour proto 1 et pour le copilot indoor. On décide le swap **quand** le critère marches/360-safety est écrit, pas avant.

## Co-Pilot Magix

Deux lectures :

1. **Pilotage supervisé** (`/magix2/intent`, voix et écran sur le PC, traction sur le Jetson) → **8 Go**, c’est le produit gelé.
2. **VLM onboard** (« le fauteuil comprend la scène ») → **NO-GO 8 Go**. Soit le modèle est sur le PC (WiFi down = plus de copilot intelligent, `safety` inchangé), soit on change de SoM **après** un critère.

On gèle (1) tant que (2) n’est pas dans les tests.

## Conséquences (toujours pas un achat)

- **Acheteur / Contrôleur :** pas de ligne NX. Super 8 Go + NVMe P300 restent la base.
- **Ing. logiciel :** tout DNN = launch à part, GPU off par défaut. BEV/depth = nœuds PC s’ils existent un jour.
- **Ing. hardware :** ceinture ToF = option 8 Go ; ne pas dimensionner le proto pour un dissipateur GPU 16 Go.
- **Testeur :** aucun T-* proto 1 n’exige de GPU. Un futur T-MARCHE onboard DNN = ouverture dossier NX 16 Go.

## Décision

| Question | Réponse |
|---|---|
| 8 Go jusqu’où ? | Proto 1, Magix indoor GPU-off, 2ᵉ UVC, ToF, R-Net, BEV/depth/360 **sur PC**. |
| No-go quand ? | DNN (DAS, YOLO, BEV, VLM) **sur le Jetson** avec slam+Nav2, ou dans `safety`. |
| Swap ? | **Déporter** par défaut. **NX 16 Go** seulement si un critère figé met le GPU dans la traction sans WiFi. Pas maintenant. |
