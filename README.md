# syslog_projet

Réveil de voyage sur carte DE10-Lite (logisim-evolution 3.7.2) – projet Systèmes logiques EPFL BA3.

## Structure du projet

```
MAIN
│
├── Clock_Manager          ✅ validé sur carte
│   ├── Frequency_Divider
│   └── Pulse_Generator
│
├── Input_Manager          ✅ terminé
│   ├── Debouncer          (x2, un par bouton)
│   └── Event_Decoder      (FSM de Mealy : NEXT / ENTER / BACK)
│
├── FSM_Main
│   ├── State_Register
│   ├── Next_State_Logic
│   └── Output_Logic
│
├── Clock
│   ├── Seconds_Counter
│   ├── Minutes_Counter
│   ├── Hours_Counter
│   └── Time_Setting
│
├── Alarm
│   ├── Alarm_Setting
│   ├── Alarm_Comparator
│   └── Alarm_FSM
│
├── Stopwatch
│   ├── Stopwatch_Counter
│   └── Stopwatch_FSM
│
├── Countdown
│   ├── Countdown_Setting
│   ├── Countdown_Counter
│   └── Countdown_FSM
│
├── Display_Manager
│   ├── Display_Multiplexer
│   ├── BCD_to_7Segment
│   ├── LCD_Message_Manager
│   └── LCD_Driver
│
├── Sound_Manager
│   ├── Tone_Generator
│   └── Buzzer_Control
│
└── Music
    ├── Melody_ROM
    ├── Note_Sequencer
    └── Note_Decoder
```

## Conventions générales

- **Une seule horloge globale `clk`** pour tout le circuit. Pas d'horloges dérivées.
- Les fréquences lentes sont des **ticks** (impulsions de 1 cycle de `clk`) branchés sur les **enable** des compteurs et bascules, jamais sur leur entrée d'horloge.
- **Fréquence réelle de `clk` sur la carte : ≈ 1.0417 MHz** (50 MHz / 48), avec la fréquence de téléchargement réglée sur 2.5 MHz dans Logisim.
  - Diviseur pour un tick à f : Maximum Value = N − 1, avec N = 1 041 667 / f.
  - Buzzer : f_son = f_clk / (2N) (compteur mod N + bascule T).
- `rst` est actif haut, branché sur les entrées R des compteurs et bascules. Au niveau de MAIN, il sera relié à un DIP-switch (les deux boutons servent à la navigation).
- **Boutons** : actifs bas sur la carte, mais Logisim inverse automatiquement via la boardmap. Dans les circuits, une pin de bouton vaut **1 quand le bouton est appuyé** (pas de porte NOT).
- **Bascule D avec enable** : la bascule D de logisim-evolution n'a pas d'enable, utiliser un **Register 1 bit** (entrée WE).
- **Mode SIM** : pour simuler, réduire la Maximum Value des gros compteurs (ex. `0xF`), puis **remettre la valeur réelle avant de charger sur la carte**.
- Noms de sous-circuits uniques entre fichiers (pas de `main` dans les modules) pour éviter les conflits lors de la fusion.

## Commandes utilisateur

| Action          | Événement | Rôle                               |
|-----------------|-----------|------------------------------------|
| Bouton 1 seul   | `next`    | Passer à l'état / au mode suivant  |
| Bouton 2 seul   | `enter`   | Valider / entrer dans un mode      |
| Boutons 1 + 2   | `back`    | Retour / annulation                |

Les événements sont émis **au relâchement** des boutons, ce qui permet de détecter l'appui simultané.

Les **DIP-switches ne contrôlent pas la FSM générale** (exigence de la consigne). Ils servent uniquement à la configuration, par exemple reset, 12 h / 24 h, buzzer muet, choix de la mélodie, mode démo.

## Modules

### Clock_Manager

| Port         | Dir. | Description                                   |
|--------------|------|-----------------------------------------------|
| `clk`        | in   | Horloge globale                               |
| `rst`        | in   | Reset                                         |
| `tick_100Hz` | out  | Impulsion 1 cycle, 100 Hz                     |
| `tick_4Hz`   | out  | Impulsion 1 cycle, 4 Hz                       |
| `tick_2Hz`   | out  | Impulsion 1 cycle, 2 Hz                       |
| `tick_1Hz`   | out  | Impulsion 1 cycle, 1 Hz                       |
| `blink`      | out  | Niveau carré 1 Hz (clignotement des réglages) |

- Étage 1 : compteur Maximum Value `0x28B0` (N = 10 417), 14 bits, donne `tick_100Hz`.
- Étages 2 : compteurs avec enable = `tick_100Hz` (max `0x63`, `0x31`, `0x18`), tick = `3CT AND tick_100Hz` (3CT ne dépend pas de l'enable G5).
- `blink` : bascule T, horloge `clk`, T = `tick_2Hz`.

### Input_Manager

| Port         | Dir. | Description                                   |
|--------------|------|-----------------------------------------------|
| `clk`        | in   | Horloge globale                               |
| `rst`        | in   | Reset                                         |
| `key1`       | in   | Bouton 1 (1 = appuyé)                         |
| `key2`       | in   | Bouton 2 (1 = appuyé)                         |
| `next`       | out  | Impulsion 1 cycle : bouton 1 seul             |
| `enter`      | out  | Impulsion 1 cycle : bouton 2 seul             |
| `back`       | out  | Impulsion 1 cycle : boutons 1 + 2             |

Le module est autonome : il n'utilise pas `tick_100Hz`.

#### Debouncer (schéma du cours)

```
key → FF1 → FF2 ───────────────────────────► FF3 (Register, WE = 3CT) → b
              └─ XOR(FF1, FF2) ─(OR rst)─► R du compteur
                                            compteur : G5 = NOT 3CT
```

- FF1 + FF2 : synchronisation (métastabilité).
- XOR = 1 quand le signal change, ce qui remet le compteur à 0.
- Le compteur mesure la durée de stabilité ; arrivé au maximum, `3CT` = 1 active FF3 (recopie de la valeur stable) et bloque le compteur (G5 = NOT 3CT).
- Compteur 15 bits, Maximum Value `0x7FFF`, soit ≈ 31 ms de stabilité exigée (`0x5160` pour exactement 20 ms).

#### Event_Decoder (FSM de Mealy, 4 états)

| État | Code C1 C0 | Signification                      |
|------|------------|------------------------------------|
| IDLE | 00         | Aucun bouton appuyé                |
| P1   | 10         | Seul le bouton 1 a été appuyé      |
| P2   | 01         | Seul le bouton 2 a été appuyé      |
| PB   | 11         | Les deux ont été appuyés ensemble  |

- Transitions : depuis IDLE vers l'état correspondant aux boutons appuyés ; P1 ou P2 passe à PB si l'autre bouton est appuyé ; PB reste PB jusqu'au relâchement complet ; retour à IDLE quand b1 b2 = 00.
- Logique de transition (Karnaugh) :
  - `D1 = b1 + b2·C1`
  - `D0 = b2 + b1·C0`
  - Interprétation : C1 = « b1 a été appuyé », C0 = « b2 a été appuyé ».
- Logique de sortie (Mealy, émise sur la transition vers IDLE) :
  - `next  = C1·¬C0·¬b1·¬b2`
  - `enter = ¬C1·C0·¬b1·¬b2`
  - `back  = C1·C0·¬b1·¬b2`
- b1 et b2 venant des registres du debouncer, les sorties ne changent que juste après un front d'horloge : Mealy par sa structure, mais avec un comportement synchrone.

## Périphériques

- **LCD 16x2 I2C** : driver `lcd1602i2c_driver` (LCD1602I2C_v5.0.circ), SDA = GPIO[8], SCL = GPIO[9]. Le driver écrit les 32 caractères une seule fois après reset : une impulsion sur `rst_n` est nécessaire pour rafraîchir l'affichage (≈130 ms). Prévoir un bloc `LCD_Refresh`.
- **Buzzer** passif : signal carré à la fréquence de la note.

## Rendu (07.12.2026, 08:00)

- Un seul fichier `.circ` contenant tout le projet
- Vidéo de 3 min max (100 MB)
- Rapport de 3 pages A4 (p. 1 : description et mode d'emploi ; p. 2–3 : solutions techniques, FSM, schémas)
