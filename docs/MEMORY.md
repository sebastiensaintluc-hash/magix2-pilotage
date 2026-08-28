# Mémoire d’équipe — Pilotage Magix2

Faits durables. Mettre à jour ici à chaque décision gelée. Pas de secrets, pas d’identifiants d’agents.

## Produit

- Cible fauteuil : Magix II, New Live (Alsace), 6 roues motrices (2 pneus + 4 omni), ~132 kg, base 58 cm, Li 24 V.
- Commande : R-Net PGDT / Curtiss-Wright (pas Dynamic Controls / LiNX). Joystick CJSM2. Power Module indiqué « R-Net 120A ».
- R-Net : pas d’API conduite publique. Phase 1 = zéro écriture sur le bus fauteuil. Libs open source (open-rnet, can2RNET) = futur driver derrière la même interface que le proto. Magix = listen-only puis write supervisé + e-stop.
- Garantie New Live exclut les modifications produit. Photos plaque / étiquettes : fauteuil au garage, plus tard.

## Proto

- Platine **miniature** ~25–30 cm, **2WD + casters**, pas un jumeau 6WD, pas Scout Mini, pas Create 3, pas Yahboom 4WD.
- Chassis : Yahboom alu 2WD + batterie 12 V (SKU site `6000200712`), alim 24 V Magix isolée.
- Compute : Jetson Orin Nano Super 8 Go (Kubii, `945-13766-0005-000`).
- Lidar : LDRobot STL-19P / LD19 (Gotronic `37942`). A2M12 = repli.
- Caméra proto 1 : UVC cheap (Logitech C270), stream RGB. Pas de D435i/Gemini en base. Depth-Anything Small = option plus tard. VL53L9 = option (stock ~0).
- IMU : BNO085 I2C1 adresse `0x4A`, 3,3 V.
- E-stop matériel + `magix2_safety` logiciel.
- Supervision : PC (UI + Vosk small-fr + Piper), pas d’écran 7" sur le robot.
- Budget proto : viser 1 000 € TTC, plafond 1 500 €. Options chiffrées à part avec plus-value.

## Logiciel (proto 1)

- ROS 2 Jazzy, JetPack 7.2 / Ubuntu 24.04.
- `magix2_msgs/Intent` figé (`/magix2/intent`).
- `magix2_safety` dernier mot sur `/cmd_vel` (heartbeat > 500 ms, lidar muet > 1 s, STOP → Twist 0).
- Lidar : `ldlidar_stl_ros2`, `ld19.launch.py`, `/dev/tty-ld19`.
- Cinématique `diff_drive`. Pas de R-Net write, pas de marches, pas de SLAM visuel, pas d’Isaac ROS.
- Dépôt : ce repo.

## Planning (indicatif)

- Freeze devis : 11 sept 2026. Zéro commande avant validation Sébastien.
- Chemin critique : Yahboom CN → FR (DDU, TVA import à confirmer).
- Proto 1 (scan + stream + IMU + e-stop + GOTO) : cible 13 nov 2026, à recaler sur le délai chassis.
- Marches / 360 / R-Net : hors proto 1.

## Rôles

- Chef de projet : spec, planning, arbitrages, doc, consignes de prod.
- Ingénieur logiciel / hardware / Acheteur / Planificateur / Testeur système.
- Sébastien : production matérielle (montage, câblage, essais), reporte au chef de projet.
