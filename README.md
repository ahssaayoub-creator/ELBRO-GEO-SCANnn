# ELBRO GEO SCAN

Application Android native (Kotlin + Jetpack Compose) de détection des variations du
champ magnétique de terrain, à l'aide du **magnétomètre réel** et du **GPS réel** du
smartphone.

- **Package** : `com.elbro.geoscan`
- **Version** : `1.0.0`
- **Langue de l'interface** : Français
- **Design** : sombre, technique, professionnel

> ⚠️ **Important sur l'origine de ce projet** : ce dépôt a été généré dans un
> environnement sans SDK Android ni accès réseau, donc **aucun build n'a pu être
> exécuté ici** — le code n'a jamais été compilé par un compilateur Kotlin réel avant
> que vous ne le fassiez. C'est un projet Android complet, prêt à compiler, mais pas
> un binaire déjà vérifié. Suivez la section "Build" ci-dessous : soit Android Studio
> en local, soit GitHub Actions, pour obtenir un premier build réel et corriger toute
> erreur de compilation qui apparaîtrait.

---

## 1. Installation (Android Studio — méthode recommandée)

1. Installez **Android Studio** (Ladybug ou plus récent) : https://developer.android.com/studio
2. Ouvrez Android Studio → **File → Open** → sélectionnez le dossier `ELBRO-GEO-SCAN`.
3. Android Studio va :
   - détecter `gradle/wrapper/gradle-wrapper.properties` et télécharger **Gradle 8.7**,
   - vous proposer d'installer le SDK Android manquant (**API 34**, Build-Tools) si besoin — acceptez,
   - générer automatiquement `local.properties` avec le chemin de votre SDK.
4. Attendez la fin de la synchronisation Gradle ("Gradle sync finished").
5. Branchez un téléphone Android (mode développeur + débogage USB activés) **ou**
   utilisez un émulateur avec capteurs simulés.
6. Cliquez sur **Run ▶** (ou **Build → Build Bundle(s) / APK(s) → Build APK(s)**).
7. L'APK généré se trouve dans :
   ```
   app/build/outputs/apk/debug/app-debug.apk        (build de développement)
   app/build/outputs/apk/release/app-release.apk     (build release, voir §5)
   ```
8. Transférez ce fichier `.apk` sur votre téléphone (câble, e-mail, Drive…) et
   installez-le (autorisez "Installer des applications inconnues" si demandé).

---

## 2. Installation (ligne de commande, sans Android Studio)

Prérequis : JDK 17, Android SDK (API 34) installé, variable `ANDROID_HOME` définie.

```bash
cd ELBRO-GEO-SCAN
cp local.properties.example local.properties
# éditez local.properties : sdk.dir=/chemin/vers/votre/Android/sdk

./gradlew assembleDebug      # build de développement
./gradlew assembleRelease    # build release installable
```

La première exécution de `./gradlew` télécharge automatiquement Gradle 8.7 et toutes
les dépendances (AndroidX, Compose, Room, Play Services) depuis Maven — **connexion
Internet requise** pour ce premier build, ensuite tout est mis en cache localement.

---

## 3. Build automatique avec GitHub Actions (`.github/workflows/build.yml`)

Ce dépôt inclut un workflow complet qui compile réellement l'APK dans le cloud, sans
rien installer sur votre machine :

1. Créez un dépôt GitHub et poussez ce projet :
   ```bash
   cd ELBRO-GEO-SCAN
   git init
   git add .
   git commit -m "ELBRO GEO SCAN v1.0.0"
   git branch -M main
   git remote add origin https://github.com/<votre-compte>/<votre-repo>.git
   git push -u origin main
   ```
2. Allez dans l'onglet **Actions** de votre dépôt GitHub → le workflow
   **"Build ELBRO GEO SCAN APK"** se lance automatiquement à chaque push.
3. Une fois le workflow terminé (icône verte ✅) :
   - Cliquez sur le run → section **Artifacts** en bas de page.
   - Téléchargez **`ELBRO-GEO-SCAN-release-apk`** (ou `-debug-apk`).
   - Le fichier téléchargé est un `.zip` contenant `app-release.apk` — décompressez-le.
4. Transférez cet APK sur votre téléphone Android et installez-le.

Le workflow effectue exactement, dans l'ordre : installation JDK 17 + Android SDK →
génération du wrapper Gradle → génération d'un keystore de signature temporaire →
`./gradlew testDebugUnitTest` → `./gradlew assembleDebug` → `./gradlew assembleRelease`
→ vérification que le fichier APK existe réellement → upload en artifact. Si une étape
échoue, le workflow s'arrête avec le log d'erreur exact — voir §6 "En cas d'erreur de
build".

### Pourquoi le wrapper Gradle est généré par le workflow plutôt que commité

Le fichier binaire `gradle-wrapper.jar` n'a pas pu être créé dans l'environnement où ce
projet a été rédigé (pas d'accès réseau pour le télécharger). Le workflow installe donc
une vraie distribution Gradle 8.7 et exécute `gradle wrapper --gradle-version 8.7`, ce
qui régénère `gradlew`, `gradlew.bat` et `gradle-wrapper.jar` à l'identique de ce
qu'Android Studio aurait produit. Ensuite tout le reste du build utilise `./gradlew`
normalement. **Dans Android Studio, vous n'avez rien à faire** : Android Studio régénère
aussi le wrapper automatiquement à la première synchronisation.

---

## 4. Fonctionnement de l'application

### Écrans (10)
1. **Accueil** — accès rapide à Nouvelle analyse / Historique / Paramètres / À propos.
2. **Nouvelle analyse** — nom de l'analyse.
3. **Calibration** — calibration réelle du magnétomètre (mouvement en « 8 »), lit
   `SensorManager.SENSOR_STATUS_ACCURACY_*` en direct.
4. **Scan terrain** — mesures en temps réel (voir ci-dessous).
5. **Carte** — position GPS, parcours, points de mesure colorés par niveau d'anomalie,
   styles STANDARD / SATELLITE / TERRAIN.
6. **Résultats** — synthèse de l'analyse + export CSV/JSON.
7. **Historique** — liste des analyses précédentes (Room).
8. **Détails d'une analyse** — liste complète des mesures d'une analyse.
9. **Paramètres** — seuils d'anomalie configurables (faible / moyen / élevé).
10. **À propos** — version, package, avertissement scientifique.

### Calibration
Utilise `SensorEventListener.onAccuracyChanged` sur le magnétomètre réel. L'utilisateur
doit atteindre une précision `MEDIUM` ou `HIGH` (ou choisir de continuer quand même) avant
de démarrer un scan.

### Scan terrain
- Lit en continu `Sensor.TYPE_MAGNETIC_FIELD` (Bx, By, Bz réels, fréquence
  `SENSOR_DELAY_UI` pour économiser la batterie) et le GPS réel via
  `FusedLocationProviderClient`.
- Calcule `B = sqrt(Bx² + By² + Bz²)`, une baseline glissante (moyenne des mesures
  précédentes de l'analyse en cours), et `anomalyScore = |magnitude_actuelle - baseline|`.
- Classe le résultat en `NORMAL / LOW ANOMALY / MEDIUM ANOMALY / HIGH ANOMALY` selon des
  seuils configurables dans Paramètres.
- **N'affiche jamais** "Gold detected" / "Treasure detected" / "Metal detected" — toujours
  "Magnetic anomaly detected" + niveau, conformément à la spécification.
- Si l'appareil n'a pas de magnétomètre : affiche
  *"Magnétomètre non disponible sur cet appareil."* et bloque le scan.
- Si la permission de localisation est refusée : affiche un message clair et propose de
  relancer la demande de permission (pas de blocage silencieux).

### Base de données (Room)
Tables `scan` et `measurement` (clé étrangère `measurement.scanId → scan.id`, suppression
en cascade). Chaque `Measurement` stocke : timestamp, latitude, longitude, altitude,
accuracy, bx, by, bz, magnitude, anomalyScore, anomalyLevel.

### Export
CSV et JSON complets, générés dans le cache de l'app puis partagés via
**Android Share Sheet** (`Intent.ACTION_SEND` + `FileProvider`, aucune permission de
stockage externe nécessaire).

### Limite scientifique (affichée dans l'app : Accueil, Résultats, À propos)
> ELBRO GEO SCAN mesure les variations du champ magnétique à l'aide du magnétomètre du
> smartphone. Une anomalie magnétique ne signifie pas nécessairement la présence d'un
> métal. Les résultats peuvent être influencés par le fer, l'acier, les câbles, les
> véhicules, les structures enterrées, certaines roches et l'environnement. Cette
> application ne permet pas d'identifier directement l'or ou un métal spécifique et ne
> remplace pas un détecteur professionnel ou une étude géophysique.

---

## 5. Signature de l'APK release

Par défaut, `assembleRelease` signe l'APK avec le **keystore de debug standard**
(`~/.android/debug.keystore`, mot de passe `android`) — c'est une solution temporaire qui
produit un vrai fichier `.apk` **directement installable sur un téléphone Android** (avec
"Sources inconnues" autorisé), mais **non publiable sur le Google Play Store**.

### Pour une signature release définitive (nécessaire pour le Play Store)

1. Générez votre propre keystore :
   ```bash
   keytool -genkey -v -keystore elbro-release.keystore -alias elbro \
     -keyalg RSA -keysize 2048 -validity 10000
   ```
2. Dans `app/build.gradle.kts`, remplacez le bloc `signingConfigs { create("releaseTemp") ... }`
   par vos propres valeurs (`storeFile`, `storePassword`, `keyAlias`, `keyPassword`), idéalement
   lues depuis des variables d'environnement ou `local.properties` — **ne committez jamais**
   votre keystore ni ses mots de passe dans Git.
3. Relancez `./gradlew assembleRelease`.

---

## 6. En cas d'erreur de build

Le projet n'a pas encore été compilé dans cet environnement (voir avertissement en
haut de ce fichier). Si Android Studio ou GitHub Actions signale une erreur :

1. Lisez le message d'erreur exact (nom de fichier + ligne).
2. La quasi-totalité des erreurs possibles à ce stade seront des versions de
   dépendances : ce projet cible `compileSdk 34`, Kotlin `1.9.24`, AGP `8.5.2`,
   Compose BOM `2024.06.00`. Si Android Studio propose une mise à jour automatique
   ("Update recommended"), vous pouvez l'accepter.
3. Pour toute erreur bloquante, copiez le message complet et demandez de l'aide — la
   structure du projet (packages, noms de classes, routes de navigation) est cohérente
   dans tous les fichiers, donc une erreur restante sera très probablement locale à un
   seul fichier et rapide à corriger.

---

## 7. Configuration Google Maps

L'écran **Carte** utilise Google Maps Compose et nécessite une clé API Google Maps
(gratuite, avec quota) :

1. Créez une clé sur https://console.cloud.google.com/google/maps-apis/credentials
   (activez l'API **"Maps SDK for Android"**).
2. **Ne mettez jamais la clé directement dans le code source ni dans Git.**
3. Ajoutez-la dans `local.properties` (fichier ignoré par `.gitignore`) :
   ```
   MAPS_API_KEY=votre_vraie_cle_ici
   ```
4. `app/build.gradle.kts` lit automatiquement cette valeur et l'injecte dans
   `AndroidManifest.xml` via `manifestPlaceholders["MAPS_API_KEY"]` — aucune autre
   modification n'est nécessaire.
5. Pour GitHub Actions : ajoutez un secret de dépôt nommé `MAPS_API_KEY`
   (**Settings → Secrets and variables → Actions → New repository secret**) ; le
   workflow l'utilise automatiquement s'il est présent, sinon il utilise un placeholder
   et la carte s'affiche avec un filigrane "for development purposes only".

Note : la "heatmap" de l'écran Carte est actuellement représentée par des points colorés
selon le niveau d'anomalie (vert/orange/rouge) plutôt que par une couche `HeatmapTileProvider`
dédiée — c'est une simplification volontaire pour garder le projet compilable sans
dépendance supplémentaire ; elle peut être remplacée par `com.google.maps.android:android-maps-utils`
si une vraie heatmap lissée est souhaitée.

---

## 8. Permissions

| Permission | Usage |
|---|---|
| `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION` | Position GPS réelle pendant le scan |
| `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_LOCATION`, `WAKE_LOCK` | Permettre au scan de continuer si l'app passe brièvement en arrière-plan |
| `INTERNET` | Affichage des tuiles Google Maps |

L'app demande `ACCESS_FINE_LOCATION` au moment de démarrer un scan (pas au lancement de
l'app). En cas de refus, un message clair s'affiche et l'utilisateur peut relancer la
demande.

---

## 9. Limites connues / non couvert dans cette v1.0.0

- Pas de service de scan en avant-plan (`ForegroundService`) implémenté pour l'instant :
  les permissions sont déclarées mais le scan actuel tourne tant que l'écran ScanScreen
  est actif (via `viewModelScope`). Pour un scan de très longue durée écran éteint, ajoutez
  un `Foreground Service` dédié.
  Voir aussi `AnomalyCalculator.gpsDistanceMeters` : déjà prêt pour ça.
- La heatmap est une approximation par points colorés (voir §7).
- Les tests unitaires couvrent la logique pure (`AnomalyCalculator`, `Thresholds`) ; les
  tests Room/instrumentés nécessitent un environnement Android (émulateur ou appareil) et
  ne sont pas inclus dans `app/src/test` (JVM pur) — à ajouter dans `app/src/androidTest`
  si vous voulez les automatiser.
- Écran tourné : `MainActivity` déclare `android:configChanges` pour éviter un redémarrage
  brutal pendant un scan, mais aucune mise en page paysage dédiée n'a été conçue.

---

## Structure du projet

```
ELBRO-GEO-SCAN/
├── .github/workflows/build.yml
├── app/
│   ├── build.gradle.kts
│   ├── proguard-rules.pro
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml
│       │   ├── java/com/elbro/geoscan/
│       │   │   ├── MainActivity.kt
│       │   │   ├── GeoScanApp.kt
│       │   │   ├── Navigation.kt
│       │   │   ├── data/          (Room: Scan, Measurement, DAOs, Database, Repository)
│       │   │   ├── sensors/       (MagnetometerManager, LocationManagerHelper)
│       │   │   ├── logic/         (AnomalyCalculator, Thresholds)
│       │   │   ├── export/        (ExportUtils: CSV/JSON + Share Sheet)
│       │   │   └── ui/
│       │   │       ├── theme/
│       │   │       ├── components/
│       │   │       ├── viewmodel/ (ScanViewModel, HistoryViewModel, SettingsViewModel)
│       │   │       └── screens/   (10 écrans)
│       │   └── res/
│       └── test/java/com/elbro/geoscan/ (unit tests)
├── gradle/wrapper/gradle-wrapper.properties
├── build.gradle.kts
├── settings.gradle.kts
├── gradle.properties
├── local.properties.example
└── README.md
```

## Ce qu'il vous reste à faire pour obtenir l'APK

1. Choisissez **Android Studio** (§1, le plus simple) **ou** **GitHub Actions** (§3).
2. Laissez le premier build s'exécuter — c'est la première fois que ce code est réellement
   compilé, corrigez toute erreur signalée en suivant §6 si nécessaire.
3. Récupérez `app-release.apk` et transférez-le sur votre téléphone Android.
