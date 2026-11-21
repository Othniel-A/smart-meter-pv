# ⚡ Compteur d'énergie prépayé intelligent (ESP32 + SIM800L)

Compteur électrique **prépayé** basé sur un ESP32. Il mesure la consommation d'une installation monophasée, décompte un crédit en kWh, coupe automatiquement le courant quand le crédit est épuisé, et se pilote **par SMS** (rechargement, alertes). Les mesures sont envoyées toutes les minutes vers **Firebase Realtime Database** via GPRS.

> Projet pensé pour des zones où le Wi-Fi n'est pas fiable : toute la communication passe par le réseau GSM.

---

## ✨ Fonctionnalités

- **Mesure en temps réel** : tension, courant, puissance, énergie cumulée, facteur de puissance, fréquence (PZEM-004T v3).
- **Crédit prépayé** : décompte en kWh restants, sauvegardé en EEPROM (survit aux coupures de courant).
- **Coupure automatique** via relais quand le crédit tombe à 0.
- **Rechargement par SMS** : `montant=<kWh>`.
- **Alertes SMS** : seuil bas (moins de 20 % de la dernière recharge) et crédit épuisé.
- **Signalisation locale** : LED verte (normal), jaune (crédit faible), rouge (épuisé) + afficheur LCD 20×4.
- **Télémétrie cloud** : envoi JSON horodaté vers Firebase (HTTPS via SIM800L).

---

## 🧰 Matériel

| Composant | Rôle |
|---|---|
| ESP32 (DevKit) | Microcontrôleur principal |
| PZEM-004T v3.0 | Mesure V / I / P / E / PF / f (UART/Modbus) |
| SIM800L | SMS + GPRS (HTTP/SSL) |
| LCD 20×4 I2C (adresse `0x27`) | Affichage local |
| Module relais | Coupure de la charge |
| 3 LEDs (verte, jaune, rouge) + résistances | Indicateurs d'état |
| Alimentation 4 V / 2 A pour le SIM800L | Le SIM800L consomme des pics jusqu'à ~2 A |

### Câblage

| Périphérique | Broche ESP32 |
|---|---|
| PZEM-004T (RX / TX) | GPIO 2 / GPIO 4 |
| SIM800L (RX / TX) | GPIO 16 / GPIO 17 (UART2) |
| LCD I2C (SDA / SCL) | GPIO 21 / GPIO 22 (I2C par défaut) |
| Relais | GPIO 27 |
| LED verte / jaune / rouge | GPIO 12 / 13 / 14 |
| Carte SD (optionnelle, non utilisée) | SCK 18 · MISO 19 · MOSI 23 · CS 5 |

> ⚠️ **Sécurité** : le PZEM-004T et le relais sont raccordés au secteur (230 V). Travaillez hors tension, utilisez un boîtier isolé et faites valider le montage par une personne qualifiée.

---

## 🗂️ Structure du code

Le projet est un sketch Arduino multi-fichiers (tous les `.ino` d'un même dossier sont compilés ensemble ; le dossier doit porter le nom du fichier principal).

```
compteur_code/
├── compteur_code.ino                  # Programme principal : setup(), loop(), logique crédit/relais/LED, JSON
├── envoi_donnees.ino                  # GPRS + POST HTTP(S) vers Firebase (commandes AT)
├── envoi_reponse_tel.ino              # Envoi de SMS + utilitaire updateSerial()
├── recuperer_montant.ino              # Lecture/traitement des SMS entrants (rechargement)
└── ecriture_recuperation_memoire.ino  # Sauvegarde/lecture EEPROM (stockeValues / repereValues)
```

### Fonctionnement

```
          ┌────────────┐  UART   ┌───────────┐
Secteur ─►│ PZEM-004T  │────────►│           │──► LCD 20x4
          └────────────┘         │           │──► LEDs / Relais ──► Charge
                                 │   ESP32   │
          ┌────────────┐  UART   │           │──► EEPROM (crédit)
 SMS ◄───►│  SIM800L   │◄───────►│           │
 GPRS ───►└────────────┘         └───────────┘
              │
              └──► Firebase Realtime DB (JSON toutes les 60 s)
```

**Boucle principale :**

1. Toutes les 60 s, si un courant est détecté : sauvegarde en EEPROM + envoi des mesures à Firebase.
2. Lecture du PZEM et affichage sur le LCD.
3. Lecture des SMS entrants (`traiterMessage`).
4. Mise à jour du crédit : `kWh_restant -= (énergie − E)`, puis pilotage du relais et des LEDs selon le niveau.

| État | Condition | Relais | LED | SMS |
|---|---|---|---|---|
| Normal | crédit ≥ 20 % de la dernière recharge | ON | 🟢 | – |
| Crédit faible | crédit < 20 % de la dernière recharge | ON | 🟡 | « 80 % consommés… » (1 fois) |
| Épuisé | crédit ≤ 0 | **OFF** | 🔴 | « Rechargement épuisé » (1 fois) |

---

## 📲 Commandes SMS

| SMS envoyé au compteur | Effet |
|---|---|
| `montant=25` | Ajoute 25 kWh au crédit, sauvegarde en EEPROM, répond par SMS avec le nouveau solde |
| Tout autre texte | Réponse d'erreur indiquant le format attendu |

---

## ☁️ Format des données envoyées (Firebase)

Chemin : `/consoEnTempsReel/compteurID_1`

```json
{
  "date": "25/10/23,14:30",
  "tension": "229.8000",
  "courant": "1.2000",
  "kWh restant": "18.4200",
  "cosphi": "0.9500",
  "puissance": "260.0000",
  "energie": "1.5800"
}
```

---

## 🚀 Installation

### 1. Prérequis

- Arduino IDE ≥ 2.x avec le support **ESP32** (Espressif)
- Bibliothèques :
  - [`PZEM004Tv30`](https://github.com/mandulaj/PZEM-004T-v30)
  - [`LiquidCrystal_I2C`](https://github.com/johnrickman/LiquidCrystal_I2C)
  - `Wire`, `SPI`, `SD`, `EEPROM` (incluses avec le core ESP32)
- Une carte SIM avec **SMS + data** activés, code PIN désactivé

### 2. Configuration

Dans `compteur_code.ino` :

```cpp
String telephone = "+229XXXXXXXXX";   // Numéro qui reçoit les alertes SMS
#define prix_kWh 100.0                // Référence tarifaire
float kWh_restant = 20.0;             // Crédit initial par défaut
```

Dans `envoi_donnees.ino` :

```cpp
const String APN  = "moov";           // APN de votre opérateur
const String USER = "moovweb";
const String PASS = "moov";
const String FIREBASE_HOST   = "https://<votre-projet>-default-rtdb.firebaseio.com/consoEnTempsReel/compteurID_1";
const String FIREBASE_SECRET = "<VOTRE_SECRET>";  // ⚠️ ne jamais commiter en clair
```

### 3. Téléversement

1. Sélectionner la carte **ESP32 Dev Module**.
2. Ouvrir `compteur_code.ino`, compiler et téléverser.
3. Ouvrir le moniteur série à **9600 bauds** pour suivre les échanges AT.

### 4. Premier démarrage

Au tout premier démarrage, l'EEPROM est vierge. Envoyez un SMS `montant=<kWh>` pour initialiser un crédit valide.

---

## 🔐 Sécurité des identifiants

Ne publiez **jamais** le secret Firebase ni votre numéro de téléphone dans un dépôt public. Bonnes pratiques :

- Placer les identifiants dans un fichier `secrets.h` ajouté au `.gitignore`.
- Utiliser des **règles de sécurité Firebase** strictes plutôt que le secret historique de la base (déprécié).
- Filtrer les numéros autorisés à envoyer `montant=` (voir pistes d'amélioration).

---

## ⚠️ Limites connues & pistes d'amélioration

- **Initialisation de `E`** : `E` est initialisé avant la première lecture du PZEM ; au premier passage de `loop()`, le crédit peut être débité de toute l'énergie cumulée du PZEM. Initialiser `E` *après* la lecture initiale (ou remettre le PZEM à zéro avec `pzem.resetEnergy()`).
- **Réception SMS** : `AT+CNMI=2,1,0,0,0` renvoie une notification `+CMTI` ; le code attend `+CMGR:`. Prévoir `AT+CMGR=<index>` après `+CMTI`, ou passer en `AT+CNMI=2,2,0,0,0` pour recevoir le SMS directement.
- **Sauvegarde du crédit** : l'EEPROM n'est écrite que toutes les 60 s ; une coupure juste avant perd jusqu'à 1 min de consommation. À surveiller aussi : l'usure de la flash si l'intervalle diminue.
- **EEPROM vierge** : valeurs `NaN`/aberrantes possibles au premier boot ; ajouter un contrôle de validité (magic number ou `isnan`).
- **Appels bloquants** : `delay()` dans `getDateAndTime()`, `is_gprs_connected()` et l'envoi HTTP bloquent la boucle plusieurs secondes (le relais ne réagit pas pendant ce temps). Migrer vers une architecture **FreeRTOS** (tâches mesure / GSM séparées) ou une machine à états non bloquante.
- **Authentification SMS** : n'importe quel numéro peut recharger le compteur. Ajouter une liste blanche ou un code PIN dans le SMS.
- **Flag `top` unique** : partagé entre l'alerte « crédit faible » et « épuisé » ; utiliser deux flags distincts.
- **Anti-rebond relais** : ajouter une hystérésis pour éviter les commutations répétées autour du seuil.
- **Code mort** : les fonctions carte SD (`STOCKER`, `RECUPERER`) et plusieurs variables (`relaisState`, `efface`, `document`) ne sont pas utilisées.
- **Évolutions possibles** : communication BLE vers une application mobile, tableau de bord web, journalisation locale sur SD, support triphasé.


