# AutoStopStart

Ajouter un mécanisme de marche/arrêt à un automate en fonction de la position des visiteurs et de l'action sur un bouton d'arrêt d'urgence. Le mécanisme doit être priorisé pour un réactivité maximale.

## Principes

### États du système

| État            | Description                                                       |
| --------------- | ----------------------------------------------------------------- |
| `En_veille`     | Mécanisme à l’arrêt, en attente de détection de présence à < 1 m. |
| `Actif`         | Mécanisme en fonctionnement pendant la présence.                  |
| `Pause_bouton`  | Pause temporaire de 10 secondes après appui sur le bouton.        |
| `Temporisation` | Délai de 30 secondes sans détection avant arrêt complet.          |

### Règles de transition

| État actuel     | Événement                      | État suivant    | Action                       |
| --------------- | ------------------------------ | --------------- | ---------------------------- |
| `En_veille`     | Détection d’un visiteur < 1 m  | `Actif`         | Démarrer le mécanisme        |
| `Actif`         | Bouton appuyé                  | `Pause_bouton`  | Arrêter le mécanisme 10s     |
| `Actif`         | Plus de détection              | `Temporisation` | Lancer le timer de 30s       |
| `Actif`         | Présence maintenue             | `Actif`         | Maintenir actif, reset timer |
| `Pause_bouton`  | Fin des 10s, présence détectée | `Actif`         | Redémarrer le mécanisme      |
| `Pause_bouton`  | Fin des 10s, aucune détection  | `Temporisation` | Lancer le timer de 30s       |
| `Temporisation` | Détection < 1 m                | `Actif`         | Redémarrer le mécanisme      |
| `Temporisation` | 30s écoulées sans détection    | `En_veille`     | Arrêt complet                |

```mermaid
flowchart TD

A["Boot ESP32] --> B[Initialisation variables & sliders"]
B --> C[LCD allumé à 50%]

%% Boucle principale
C --> D["Lecture LD2410 (distance, présence...)"]
D --> E{"Bouton STOP appuyé ?"}

%% Gestion arrêt d'urgence
E -- Oui --> F["Affiche 'STOP' sur LCD"]
F --> G["Relais OFF"]
G --> H["Countdown = delay_off"]

E -- Non --> I["Affiche distance sur LCD"]

%% Vérification distance
I --> J{"Distance < seuil ?"}
J -- Non --> K["Pas d'action"]
J -- Oui --> L{"Countdown == 0 ?"}

L -- Non --> M["Attendre fin du compte à rebours"]
L -- Oui --> N["Relais ON"]
N --> O["Countdown = delay_on"]

%% Gestion relais actif
O --> P{"Relais ON ?"}
P -- Oui --> Q{"Countdown == 0 ?"}
Q -- Non --> R["Affiche 'ON' et countdown sur LCD"]
Q -- Oui --> S["Relais OFF"]
S --> T["Countdown = delay_off"]

P -- Non --> U["Affiche 'OFF' et countdown sur LCD"]
R --> D
U --> D
T --> D
H --> D
```


### Diagramme de la machine à états

```mermaid
   [En_veille]
       |
   (détection <1m)
       ↓
     [Actif] <-------------------------+
       |       (présence continue)    |
       |                              |
       |                             (détection <1m)
  (plus de détection)                 ↑
       ↓                              |
[Temporisation]                       |
       |      (aucune détection 30s)  |
       ↓                              |
   [En_veille]                        |
       ↑                              |
       |                              |
 (bouton appuyé)                      |
       ↓                              |
 [Pause_bouton] (10s) ----------------+
       ↓
 (fin pause)
       ↓
(présence?) → Actif / Temporisation
```
```mermaid
stateDiagram-v2
    [*] --> En_veille

    En_veille --> Actif : détection < 1m

    Actif --> Pause_bouton : bouton appuyé
    Actif --> Temporisation : plus de détection
    Actif --> Actif : détection maintenue

    Pause_bouton --> Actif : fin des 10s\net détection présente
    Pause_bouton --> Temporisation : fin des 10s\net pas de détection

    Temporisation --> Actif : détection < 1m
    Temporisation --> En_veille : 30s sans détection

    En_veille --> [*]
```

## ESPHome

### Matériel supposé
* LD2410 connecté en UART
* Bouton sur une GPIO avec pull-up
* Mécanisme contrôlé via une GPIO (par exemple un relais)
* ESP32 recommandé (UART + timers + RAM)

### Fonctionnalité
- Actif si présence < 1m
- Bouton = pause 10s
- Timeout d’absence = 30s
- Redémarrage auto si présence persiste après la pause
