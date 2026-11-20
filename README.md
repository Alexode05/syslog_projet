# syslog_projet

Réveil de voyage sur carte DE10-Lite (logisim-evolution 3.7.2) – projet Systèmes logiques EPFL BA3.

## Structure du projet

```
reveil.circ  (main)                          fichiers
│
├── clock_manager      ✅ ticks 100/4/2/1 Hz + blink          clock_manager.circ
├── input_manager      ✅ Debouncer x2 + Event_Decoder          input_manager.circ
├── FSM_Main           ✅ 12 états Moore + out_mode             FSM_main.circ
├── Horloge            ✅ Time_Setting + 3 compteurs            horloge.circ
├── Alarm              ✅ Alarm_setting, Alram_Comparator, Alarm_FSM   alarm.circ
├── stopwatch          ✅ Stopwatch_Counter + Stopwatch_FSM     stopwatch.circ
├── countdown          ✅ Countdown_Setting/Counter/FSM          countdown.circ
├── display_manager    ✅ MUX 17 bits, Binary_to_BCD, BCD_to_7Segment,
│                         LCD_Message, LCD_Driver               display_manager.circ
├── Field_select       ✅ (dans reveil.circ) choix h/min, Horloge + Alarme
├── Field_select_3     ✅ (dans reveil.circ) choix h/min/s, Minuteur
├── Sound_Manager      ⏳ à faire (buzzer)
└── Jeux 1 / Jeux 2    ⏳ à faire (entrées jeu_1/jeu_2 du display = constante 0)
```

## Conventions générales

- **Une seule horloge globale `clk`**. Les fréquences lentes sont des **ticks** (impulsions de 1 cycle) branchés sur les **enable**, jamais sur une entrée d'horloge.
- **Fréquence de `clk` sur la carte** : fréquence de téléchargement réglée à 2.5 MHz dans Logisim, fréquence réelle supposée ≈ 1.0417 MHz (50 MHz / 48).
  - Diviseur : Maximum Value = N − 1, avec N = f_clk / f_tick.
  - ⚠️ Mesure sur carte : l'horloge prenait ≈ 2 min de retard par heure (≈ 3,3 %), donc la fréquence réelle est plutôt ≈ 1,007 MHz (peut-être 1,000 MHz). Correction : Max du 1er étage de clock_manager = 10 417 × (temps affiché / temps réel) − 1 (ex. `0x270F` si f = 1 MHz).
- `rst` actif haut, sur les R. Au MAIN, il vient de la pin `RESET` (DIP-switch SW9).
- **Boutons** : actifs bas sur la carte, mais Logisim inverse via la boardmap : une pin bouton vaut **1 quand appuyé**.
- **Même mécanisme pour les sorties** : Logisim inverse les HEX (actifs bas) d'après la définition de la carte et **ignore l'attribut « Active On High »** de l'afficheur. Sur la carte, un **1 allume le segment** → la ROM 7 segments doit être **active haute**.
- **Bascule D avec enable** : la bascule D de logisim-evolution n'en a pas → **Register 1 bit** (WE).
- **Mode SIM** : réduire la Maximum Value des gros compteurs pour simuler, puis remettre la vraie valeur.
- Noms de sous-circuits uniques entre fichiers (pas de `main` dans les modules).
- **Ordre des ports d'un sous-circuit = ordre vertical des pins** (entrées à gauche, sorties à droite, triées par y). Déplacer une pin dans un module **décale ses ports** dans les circuits parents : les fils restent en place mais se retrouvent sur d'autres signaux. Ajouter une nouvelle pin **sous** les autres pour ne rien décaler.
- **Tunnels** : reliés uniquement par nom exact (attention aux fautes de frappe et aux largeurs).

### Bus de temps (17 bits)

Chaque mode envoie son temps au display sur un bus de 17 bits construit par un splitter :

| Bits  | Contenu  | Largeur |
|-------|----------|---------|
| 0–5   | secondes | 6       |
| 6–11  | minutes  | 6       |
| 12–16 | heures   | 5       |

- `clk_time` = Horloge, `alm_time` = Alarme (secondes = constante 0), `chr_time` = Chrono, `tmr_time` = Minuteur.
- Splitter : Fan Out 3, Bit Width In 17. Certains splitters du MAIN sont câblés « à l'envers » (heures sur la branche 0) : c'est correct tant que la répartition des bits suit le tableau.

### Brochage du compteur Logisim

| Entrée / sortie       | Rôle                                             |
|-----------------------|--------------------------------------------------|
| R                     | Reset asynchrone                                 |
| M1[load] / M2[count]  | 0 = comptage                                     |
| M3[up] / M4[down]     | 1 = comptage croissant                           |
| G5                    | Enable                                           |
| `2,3,5+/C6`           | Entrée d'horloge                                 |
| `3CT=max`             | 1 quand le compteur est au max (indépendant de G5) |

## Mode d'emploi (pour le rapport, p. 1)

| Action          | Événement |
|-----------------|-----------|
| Bouton 1 seul   | `next`    |
| Bouton 2 seul   | `enter`   |
| Boutons 1 + 2   | `back`    |

Événements émis **au relâchement**.

| Écran (VIEW)   | `next`        | `enter`             | En réglage (EDIT) : `next` | `enter`                   | `back`                         |
|----------------|---------------|---------------------|----------------------------|---------------------------|--------------------------------|
| Horloge        | mode suivant  | régler l'heure      | +1 sur le champ choisi     | champ suivant (h ↔ min)   | quitter (secondes figées pendant le réglage) |
| Réveil         | mode suivant  | régler l'alarme     | +1 sur le champ choisi     | champ suivant (h ↔ min)   | quitter                        |
| Chronomètre    | mode suivant  | entrer              | –                          | start / pause             | **remise à zéro** + quitter    |
| Minuteur       | mode suivant  | régler la durée     | +1 sur le champ choisi     | champ suivant (h → min → s) | **lance le décompte** + quitter |
| Jeux 1 / 2     | mode suivant  | –                   | –                          | –                         | quitter                        |

- Ordre des modes avec `next` : Horloge → Réveil → Chrono → Minuteur → Jeux 1 → Jeux 2 → Horloge.
- `back` en VIEW : sans effet.
- Le LCD affiche le nom du mode sur la ligne 1 et « EDIT » sur la ligne 2 en réglage.
- **Alarme** : DIP-switch `SW_Alarm_ON` (SW0) en haut = alarme armée (LED `LED_Alarm_ON`). Pour arrêter la sonnerie, baisser le switch ; le remonter pour la réarmer.
- **Minuteur** : revenir en réglage annule le décompte. À 00:00:00 il sonne ; n'importe quel bouton coupe la sonnerie.
- **Chrono** : quitter son écran avec `back` le remet à zéro (il ne tourne pas en arrière-plan).
- Les DIP-switches ne pilotent pas la FSM principale (exigence de la consigne) : uniquement `RESET` et `SW_Alarm_ON`.

## Modules

### Clock_Manager

| Port         | Dir. | Description                                   |
|--------------|------|-----------------------------------------------|
| `clk`, `rst` | in   |                                               |
| `tick_100Hz` | out  | Impulsion 1 cycle, 100 Hz                     |
| `tick_4Hz`   | out  | Impulsion 1 cycle, 4 Hz                       |
| `tick_2Hz`   | out  | Impulsion 1 cycle, 2 Hz                       |
| `tick_1Hz`   | out  | Impulsion 1 cycle, 1 Hz                       |
| `blink`      | out  | Niveau carré 1 Hz (clignotement)              |

- Étage 1 : Max `0x28B0` (N = 10 417), 14 bits → `tick_100Hz` (voir la remarque sur la dérive ci-dessus).
- Étages 2 : enable = `tick_100Hz`, Max `0x63`, `0x31`, `0x18` ; tick = `3CT AND tick_100Hz`.
- `blink` : bascule T, T = `tick_2Hz`.

### Input_Manager

| Port              | Dir. | Description                       |
|-------------------|------|-----------------------------------|
| `clk`, `rst`      | in   |                                   |
| `btn_1`, `btn_2`  | in   | Boutons (1 = appuyé)              |
| `next_state`      | out  | Impulsion : bouton 1 seul         |
| `entrer`          | out  | Impulsion : bouton 2 seul         |
| `back`            | out  | Impulsion : boutons 1 + 2         |

#### Debouncer (schéma du cours, x2)

```
key → FF1 → FF2 ───────────────────────────► FF3 (Register, WE = 3CT) → b
              └─ XOR(FF1, FF2) ─(OR rst)─► R du compteur ; G5 = NOT 3CT
```

- Compteur 15 bits, Max `0x7FFF` ≈ 31 ms de stabilité exigée.

#### Event_Decoder (FSM de Mealy, 4 états)

| État | C1 C0 | Signification                  |
|------|-------|--------------------------------|
| IDLE | 00    | Aucun bouton                   |
| P1   | 10    | Bouton 1 a été appuyé          |
| P2   | 01    | Bouton 2 a été appuyé          |
| PB   | 11    | Les deux ont été appuyés       |

- `D1 = b1 + b2·C1`, `D0 = b2 + b1·C0`
- `next = C1·¬C0·¬b1·¬b2`, `enter = ¬C1·C0·¬b1·¬b2`, `back = C1·C0·¬b1·¬b2`
- Mealy par structure, mais comportement synchrone (b1, b2 sortent de registres). Une version Moore demanderait 7 états.

### FSM_Main (FSM principale, Moore, méthode du cours)

- État sur 4 bits `E M2 M1 M0` : `M` = mode, `E` = 1 en réglage (EDIT).
- 12 états utilisés : VIEW_x = `0 M`, EDIT_x = `1 M`.

| Mode        | M2 M1 M0 |
|-------------|----------|
| Horloge     | 000      |
| Réveil      | 001      |
| Chrono      | 010      |
| Minuteur    | 011      |
| Jeux 1      | 100      |
| Jeux 2      | 101      |

- VIEW_x : `next` → mode suivant (mod 6), `enter` → EDIT_x, sinon reste. EDIT_x : `back` → VIEW_x, sinon reste.
- Codes 110 / 111 : retour inconditionnel à 000 (sécurité, jamais atteints en usage normal).
- Entrée de la logique de mode : `A = N·¬E` (en EDIT, `next` appartient au sous-module).
- Équations (Karnaugh 4 variables A, M2, M1, M0) :
  - `D2 = M2·¬M1·¬A + M2·¬M1·¬M0 + ¬M2·M1·M0·A`
  - `D1 = ¬M2·M1·¬A + ¬M2·M1·¬M0 + ¬M2·¬M1·M0·A`
  - `D0 = (A ⊕ M0)·¬(M2·M1)`
  - `DE = V + E·¬B` (V = enter, B = back ; B·V en don't care)
- Logique de sortie `out_mode` : décodeur one-hot H, A, C, M, J1, J2 (sans E) + sorties `M2 M1 M0` (sélection du display) et `E`.
- Au MAIN : `set_x = E · x` (AND) → entrée « mode réglage » de chaque module : `set_H`, `set_A`, `set_C`, `set_M`.

### Horloge

- `Time_Setting` :
  - `Enable_S = Tick_1Hz · ¬Set_Mode`
  - `Enable_M = Enable_S·Carry_S + Set_Mode·Inc_Minute`
  - `Enable_H = Enable_S·Carry_S·Carry_M + Set_Mode·Inc_Hour`
- En réglage : secondes figées, `Inc_Minute` / `Inc_Hour` font +1 sans retenue.

### Alarm

- Entrées : `CLK`, `Reset`, `Hours`, `Minutes`, `Seconds` (heure courante), `Alarm_Set_Mode`, `Inc_Hour`, `Inc_Minute`, `Alarm_ON`. Sorties : `Alarm_Ringing`, `Alarm_Hours`, `Alarm_Minutes`.
- `Match` = heures et minutes égales, secondes = 0 (dure 1 s).
- `Alarm_FSM` (2 bascules) :

| État (Q1 Q0) | Rôle                                         | Ringing |
|--------------|----------------------------------------------|---------|
| 00 IDLE      | attend                                       | 0       |
| 10 RING      | sonne (indéfiniment)                         | 1       |
| 01 STOPPED   | coupée, attend la fin de `Match`             | 0       |

  - IDLE → RING si `Match · Alarm_ON` ; RING → STOPPED si `¬Alarm_ON` ; STOPPED → IDLE quand `Match` = 0.
- L'entrée `Stop_Alarm` a été supprimée : on arrête la sonnerie avec le switch `Alarm_ON`.

### Stopwatch

- Entrées : `CLk`, `Reset`, `Tick_1Hz`, `Start_Stop`, `Stopwatch_Clear`.
- `Stopwatch_FSM` : 1 bascule, `Run⁺ = Run ⊕ Start_Stop` → `Start_Stop` doit être une **impulsion** (un niveau fait basculer `Run` à chaque cycle).
- `Stopwatch_Clear` : OR avec `Reset` sur le R des compteurs et de la bascule (remise à zéro + arrêt).

### Countdown

- `Countdown_Setting` : 3 compteurs (durée réglée), avancent sur `Inc_*` si `Countdown_Set_Mode` = 1.
- `Countdown_Counter` : `Load` copie la durée, décompte sur `Tick_1Hz` si `Run`, `Done` à 00:00:00.
- MUX de sortie : affiche la durée réglée si `Set_Mode` = 1, sinon le décompte.
- `Countdown_FSM` (Q1 Q0) :

| État     | Code | Rôle                           |
|----------|------|--------------------------------|
| IDLE     | 00   | arrêté                         |
| RUN      | 01   | décompte (`Run` = 1)           |
| PAUSE    | 10   | en pause                       |
| RING     | 11   | sonne (`Countdown_Ringing`)    |

  - Tout est piloté par l'impulsion `Start_Stop` : IDLE → RUN (+ `Load`), RUN → PAUSE, PAUSE → RUN, RING → IDLE ; RUN → RING sur `Done`.
  - **`Set_Mode` = 1 force IDLE** → on ne peut pas démarrer en EDIT (d'où le démarrage retardé, voir plus bas).

### Display_Manager

| Entrée                                   | Rôle                                     |
|------------------------------------------|------------------------------------------|
| `clk_time`, `alm_time`, `chr_time`, `tmr_time`, `jeu_1`, `jeu_2` | Bus de temps 17 bits par mode |
| `mode` (3 bits)                          | Sélection de la page                     |
| `Edit`                                   | Affiche « EDIT » sur le LCD              |
| `CLK`, `Reset`                           |                                          |

Sorties : `Diz/Unit_Hours`, `Diz/Unit_Minutes`, `Diz/Unit_Seconds` (7 bits), `SCL`, `SDA`.

- **Afficheurs 7 segments** : MUX 8:1 de 17 bits (sel = `mode`, entrées 6–7 = 0) → splitter → `Binary_to_BCD` x3 (heures complétées à 6 bits par une constante 0) → `BCD_to_7Segment` x6.
- `Binary_to_BCD` (0–59) : comparateurs ≥ 10/20/30/40/50, soustracteurs et MUX en cascade pour les unités ; Priority Encoder pour les dizaines. Sorties dans l'ordre `Tens` puis `Unit` (pin `Tens` au-dessus).
- `BCD_to_7Segment` : ROM 16 x 7, bit 0 = a … bit 6 = g, **active haute** :

| Chiffre | 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10–15 |
|---------|----|----|----|----|----|----|----|----|----|----|-------|
| ROM     | 3F | 06 | 5B | 4F | 66 | 6D | 7D | 07 | 7F | 6F | 00    |

- **LCD** :
  - `LCD_Message` : ligne 1 = 2 MUX 8:1 de 64 bits (sel = `mode`), ligne 2 = 2 MUX 2:1 (sel = `edit`). Constantes 64 bits ASCII, poids forts = caractère de gauche, pas d'accents (police A00).
  - Rafraîchissement : Register 4 bits `{edit, mode}` + Comparator ; `¬(=)` OR `Reset` → Reset du `LCD_Driver` (impulsion à chaque changement de mode / edit).

| Mode         | R1C0                 | R1C8                 |
|--------------|----------------------|----------------------|
| Horloge      | `0x20202020486f726c` | `0x6f67652020202020` |
| Reveil       | `0x2020202020526576` | `0x65696c2020202020` |
| Chronometre  | `0x20204368726f6e6f` | `0x6d65747265202020` |
| Minuteur     | `0x202020204d696e75` | `0x7465757220202020` |
| Jeux 1       | `0x20202020204a6575` | `0x7820312020202020` |
| Jeux 2       | `0x20202020204a6575` | `0x7820322020202020` |
| (ligne 2, EDIT) | `0x2020202020204544` | `0x4954202020202020` |

## Logique de liaison dans le MAIN (reveil.circ)

### set_x

`set_H = E·H`, `set_A = E·A`, `set_C = E·C`, `set_M = E·M` → « mode réglage » du module concerné.

### Field_select (Horloge, Alarme) – FSM de Moore à 2 états

- Entrées : `En` (enter), `Set_x`, `Nex` (next), `clk`, `rst`. Sorties : `inc_h`, `inc_m`.
- `sel` = 0 : heures, 1 : minutes (Register 1 bit, WE = 1).
- `D_sel = Set_x · (sel ⊕ En)` → bascule sur `enter`, retour à 0 hors réglage.
- `inc_h = Nex·¬sel`, `inc_m = Nex·sel` (les modules filtrent déjà avec leur Set_Mode).
- Le `enter` qui fait entrer en EDIT n'est pas vu (E vaut encore 0 à ce cycle) → on commence toujours par les heures.
- Deux instances : `set_H` → Horloge, `set_A` → Alarme.

### Field_select_3 (Minuteur) – compteur mod 3

- Ordre des pins (≠ Field_select !) : `Nex`, `En`, `Set_x`, `clk`, `rst`. Sorties `inc_h`, `inc_m`, `ins_s`.
- S1 S0 : 00 (h) → 01 (min) → 10 (s) → 00 sur `enter`, 00 hors réglage :
  - `D1 = Set·(En·S0 + ¬En·S1)`
  - `D0 = Set·(En·¬S1·¬S0 + ¬En·S0)`
- `inc_h = Nex·¬S1·¬S0`, `inc_m = Nex·¬S1·S0`, `inc_s = Nex·S1·¬S0`.

### Chronomètre

- `Start_Stop = enter · set_C` (impulsion, sinon `Run` bascule à chaque cycle).
- `Stopwatch_Clear = back · set_C`.

### Minuteur

- `Countdown_Set_Mode = set_M`.
- Démarrage : bascule D, `D = back · set_M` → impulsion **retardée d'un cycle** (au cycle du `back`, Set_Mode vaut encore 1 et forcerait IDLE).
- Arrêt de la sonnerie : `Countdown_Ringing · (next + enter + back)`.
- `Start_Stop = Q + Countdown_Ringing · (next + enter + back)`.

### Alarme

- `Alarm_ON` ← pin `SW_Alarm_ON` (aussi sur `LED_Alarm_ON`).

## Brochage carte (FPGA Commander)

| Pin Logisim            | Carte                    |
|------------------------|--------------------------|
| `Btn_1`, `Btn_2`       | les 2 boutons poussoirs  |
| `RESET`                | DIP-switch SW9           |
| `SW_Alarm_ON`          | DIP-switch SW0           |
| `LED_Alarm_ON`         | une LED rouge            |
| `L_7_Segment_Display_1…6` | HEX5 … HEX0 (heures → secondes) |
| `GPIO1` (SDA), `GPIO3` (SCL) | GPIO (LCD I2C)     |

## Pièges rencontrés (utile pour le debug)

- **Chiffres en négatif sur la carte** : ROM 7 segments active basse alors que Logisim inverse déjà vers les HEX → ROM active haute.
- **Dizaines / unités échangées** : pins `Tens`/`Unit` de `Binary_to_BCD` déplacées → ordre des ports changé dans le parent.
- **Minuteur bloqué, « 88 » sur les heures** : `Nex` et `Set_x` de `Field_select_3` inversés au MAIN (ordre des pins différent de `Field_select`) → incrément à chaque cycle.
- **Chrono sans pause** : `Start_Stop` branché sur un niveau (`set_C`) au lieu d'une impulsion.
- **Minuteur qui ne démarre pas** : `Start_Stop = back·set_M` sans retard → Set_Mode force IDLE au même cycle.

## À faire

- Sound_Manager / buzzer (obligatoire pour éviter la pénalité) : utiliser `Alarm_Ringing` et `Countdown_Ringing`.
- Jeux 1 et Jeux 2 (bus 17 bits `jeu_1`, `jeu_2` du display).
- Corriger la dérive de l'horloge (Max du 1er étage de clock_manager).
- Rapport (3 pages) et vidéo.

## Barème (rappel)

- Base 5.0 ; 2 fonctions simples → 5.5 ; 1 complexe → 5.75 ; 2 complexes → 6.0.
- Sans LCD ni buzzer : −1.5 ; buzzer seul : −1.0.

## Rendu (07.12.2026, 08:00, déposé par un seul étudiant)

- Un seul fichier `.circ` contenant tout le projet
- Vidéo de 3 min max (100 MB)
- Rapport de 3 pages A4 : p. 1 description + mode d'emploi ; p. 2–3 solutions techniques, FSM principale, schémas, chronogrammes
- Retour de la carte le même jour, 15:15–17:00, CM-1
