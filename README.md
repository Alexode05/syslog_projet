# syslog_projet

Réveil de voyage sur carte DE10-Lite (logisim-evolution 3.7.2) – projet Systèmes logiques EPFL BA3.

## Structure du projet

```
MAIN
│
├── Clock_Manager
│   ├── Frequency_Divider
│   └── Pulse_Generator
│
├── Input_Manager
│   ├── Debouncer        (x2, un par bouton)
│   └── Event_Decoder    (NEXT / ENTER / BACK)
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
- Les fréquences lentes sont des **ticks** (impulsions de 1 cycle de `clk`) branchés sur les **enable** des compteurs et bascules.
- **Fréquence réelle de `clk` sur la carte : ≈ 1.0417 MHz** (50 MHz / 48), avec la fréquence de téléchargement réglée sur 2.5 MHz dans Logisim.
  - Diviseur pour un tick à f : Maximum Value = N − 1, avec N = 1 041 667 / f.
  - Buzzer : f_son = f_clk / (2N) (compteur mod N + bascule T).
- `rst` est actif haut et branché sur les entrées R de tous les compteurs.
- Boutons DE10-Lite actifs bas.
- Noms de sous-circuits uniques entre fichiers (pas de `main` dans les modules) pour éviter les conflits lors de la fusion.

## Commandes utilisateur

| Action          | Événement | Rôle                               |
|-----------------|-----------|------------------------------------|
| Bouton 1 seul   | `next`    | Passer à l'état / au mode suivant  |
| Bouton 2 seul   | `enter`   | Valider / entrer dans un mode      |
| Boutons 1 + 2   | `back`    | Retour / annulation                |

Les événements sont émis **au relâchement** des boutons, ce qui permet de détecter l'appui simultané.

Les **DIP-switches ne contrôlent pas la FSM générale** (exigence de la consigne). Ils servent uniquement à la configuration, par exemple 12 h / 24 h, buzzer muet, choix de la mélodie, mode démo.

## Interfaces des modules

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

### Input_Manager

| Port         | Dir. | Description                                   |
|--------------|------|-----------------------------------------------|
| `clk`        | in   | Horloge globale                               |
| `rst`        | in   | Reset                                         |
| `tick_100Hz` | in   | Cadence d'échantillonnage du debouncer        |
| `key1_n`     | in   | Bouton 1 brut (actif bas)                     |
| `key2_n`     | in   | Bouton 2 brut (actif bas)                     |
| `next`       | out  | Impulsion 1 cycle : bouton 1 seul             |
| `enter`      | out  | Impulsion 1 cycle : bouton 2 seul             |
| `back`       | out  | Impulsion 1 cycle : boutons 1 + 2             |

## Périphériques

- **LCD 16x2 I2C** : driver `lcd1602i2c_driver` (LCD1602I2C_v5.0.circ), SDA = GPIO[8], SCL = GPIO[9]. Le driver écrit les 32 caractères une seule fois après reset : une impulsion sur `rst_n` est nécessaire pour rafraîchir l'affichage (≈130 ms).
- **Buzzer** passif : signal carré à la fréquence de la note.

## Rendu (07.12.2026, 08:00)

- Un seul fichier `.circ` contenant tout le projet
- Vidéo de 3 min max (100 MB)
- Rapport de 3 pages A4
