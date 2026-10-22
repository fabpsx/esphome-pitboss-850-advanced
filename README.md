# Contrôleur DIY Avancé pour BBQ à Pellets (Pit Boss Pro Series 850)

Ce projet propose le remplacement complet de l'électronique d'origine d'un fumoir/barbecue Pit Boss Pro Series 850 par un système intelligent sous **ESPHome** et **Home Assistant**. Il s'appuie sur une interface utilisateur haut de gamme locale gérée par la bibliothèque graphique **LVGL** sur un grand écran tactile.

---

## 🚫 AVIS IMPORTANT — PROJET EN LECTURE SEULE (READ-ONLY)

**Ce dépôt est partagé uniquement à des fins d'information, d'archivage et d'inspiration.** 
* **Aucun support technique ne sera fourni.**
* Les sections *Issues* et *Discussions* sont volontairement désactivées. 
* Merci de ne pas envoyer de messages privés pour demander des adaptations ou de l'aide à la configuration. 
* Vous êtes entièrement responsable des modifications matérielles et logicielles que vous entreprenez sur votre propre équipement.

---

## 🛠️ Architecture Matérielle (Hardware)

Le système est conçu autour de la carte de développement **Waveshare ESP32-S3 Touch LCD 7B** (Écran IPS de 7 pouces, résolution de 1024×600 pixels, dalle tactile GT911). 

L'écran LCD RGB parallèle monopolisant la quasi-totalité des GPIO natifs de l'ESP32-S3, l'interfaçage des capteurs et actionneurs est déporté intelligemment sur le bus I2C (broches SDA: 8, SCL: 9) et le bus UART2 :

* **Extension d'Entrées/Sorties discret (MCP23017, Adresse 0x20)** : Pilote la partie puissance 120 VAC via des relais statiques (SSR25 DA) pour :
  * Le moteur de la vis sans fin (*Auger Motor*) via une broche de modulation PWM lente.
  * L'allumeur à cartouche incandescente (*Igniter Assembly*).
  * Le ventilateur de combustion (*Draft Fan*).
  * Une résistance chauffante PTC 5V 50°C pour éviter la condensation et stabiliser le boîtier électrique.
* **Convertisseur Analogique-Numérique (ADS1115, Adresse 0x48)** : Lecture haute précision (Gain 2.048) des sondes de température d'origine de type RTD PT1000 :
  * 1 sonde de chambre (cuve du BBQ) épaulée par des condensateurs de découplage de 100 nF pour éliminer le bruit électromagnétique.
  * 2 sondes à viande pour le suivi des cuissons.
* **Capteur de Télémétrie de Trémie (DFRobot A02YYUW, bus UART2)** : Capteur ultrasonique étanche connecté sur les broches TX:43 et RX:44 (9600 baud) pour mesurer en temps réel la hauteur de granules restants.
* **Capteur Environnemental (AHT25, Adresse 0x38)** : Fournit la température et l'humidité extérieures, servant de référence de base pour le calcul des offsets de calibration des sondes de cuisson.

---

## 🧠 Algorithme de Régulation et Cheminement des Signaux

La température du barbecue est contrôlée par une logique logicielle de pointe articulée autour de deux modes sélectionnables via l'interface LVGL ou Home Assistant :

1. **Le PID Natif ESPHome** : Pour une régulation classique.
2. **Le Prédicteur de Smith (Smith Predictor) en C++ natif** : Activé par l'interrupteur `smith_predictor_active`. Cet algorithme de classe industrielle neutralise le PID natif (gains forcés à zéro) et compense dynamiquement le fort temps de transport du système de combustion (délai physique entre la rotation de la vis et l'élévation thermique).
   * **Constantes calorifiques calibrées** : Temps de transport (θ) fixé à 140 secondes et constante de temps thermique (τ) fixée à 240 secondes.
   * **Anti-windup & Normalisation** : Calculs normalisés sur la plage utile du procédé (450°F) selon l'architecture PROVOX industrielle.

### Logique d'Aiguillage du Signal (`update_auger_routing`)

La commande finale envoyée à la sortie PWM lente (`auger_slow_pwm`) respecte une cascade stricte de priorités logiques pour garantir la sécurité et la flexibilité du système :

* **Priorités Absolues (Forçage à 100%)** : L'activation de la purge de vis (`purge_active`) ou de l'amorçage initial (`is_priming`) écrase toute consigne et force la vis à 100% de sa capacité.
* **Phase d'Allumage (Forçage à 20%)** : Pendant la séquence de démarrage (`is_igniting`), le système ignore la régulation et distribue un ratio fixe sécuritaire de pellets.
* **Mode Fumée (Smoke Mode)** : Permet de réguler le taux de combustion sur une intensité fixe définie (ex: 11%) pour générer un maximum de fumée aromatique sans intervention du PID.
* **Phase de Pré-refroidissement (`Pre_cooldown`)** : Si l'utilisateur demande l'arrêt du BBQ alors que la cuve dépasse les 300°F, le système bascule dans une séquence de sécurité. La vis est maintenue au ralenti strict (3%) pour abaisser progressivement le foyer sous les 225°F avant coupure totale, évitant ainsi les chocs thermiques et les retours de feu dangereux dans la trémie.

---

## 💻 Optimisations Logicielles et Stabilité de l'Écran

Le microrégulateur exécute le framework `esp-idf` configuré chirurgicalement pour exploiter la carte Waveshare 7B sans sa bête noire d'origine : le scintillement et les gels d'affichage.

* **Stabilisation Immédiate au Boot** : Au démarrage (priorité 800), des commandes bas niveau forcent les registres de la puce intermédiaire CH32V003 pour fixer l'état du PWM et figer le rétroéclairage afin d'éliminer le flash blanc au boot.
* **Temporisation du Tactile** : L'initialisation du tactile GT911 est retardée (`setup_priority: -100`) afin de laisser le concentrateur de la carte Waveshare démarrer complètement, résolvant définitivement le bug de perte de sensibilité tactile lors des coupures de courant.
* **Optimisation de la PSRAM** : Compilation sous `esp-idf` configurée pour forcer l'allocation de la PSRAM en mode **Octal à 80MHz/120MHz**, indispensable pour manipuler les buffers graphiques lourds requis par LVGL sur une dalle 1024x600.
* **Protection Écran (Antiburn)** : Un économiseur d'écran temporisé (5 minutes d'inactivité) bascule la luminosité à 25% et déplace un logo flottant aléatoirement pour prévenir le marquage permanent de la dalle LCD.

---

## 📝 Conseil de Calibration des Sondes

Pour obtenir une précision maximale à la grille de cuisson :
1. Placez toutes vos sondes physiques à l'intérieur du boîtier de contrôle à température ambiante stabilisée.
2. Comparez les valeurs mesurées à celles de la sonde de référence extérieure (AHT25).
3. Ajustez l'élément `offset` sous le bloc `calibrate_linear` de votre fichier de configuration pour corriger d'éventuels écarts fixes.

---

## ⚖️ Licence

Le code et les diagrammes de ce projet sont mis à disposition gratuitement sous licence **Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)**. 
Vous êtes libre de copier et redistribuer ce matériel à des fins personnelles uniquement. Toute utilisation commerciale ou modification du code sans accord préalable est interdite. Aucun support n'est garanti.
