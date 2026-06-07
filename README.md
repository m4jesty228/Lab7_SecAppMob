# Lab 7 — Analyse Dynamique Android : MobSF + DIVA + Frida

**Auteur :** DOSSAH Yao Landry  
**Filière :** Génie CyberDefense et Systèmes de Télécommunications Embarquées (GCDSTE)  
**Établissement :** ENSA Marrakech

---

## Contexte pédagogique

Ce laboratoire porte sur l'analyse dynamique (runtime) d'une application Android vulnérable à l'aide de **MobSF** (Mobile Security Framework). L'objectif est de détecter des vulnérabilités en temps réel — stockage insecure, secrets hardcodés, intents exposés, trafic réseau non chiffré — en instrumentant l'application **DIVA** (Damn Insecure and Vulnerable Android App) sur un émulateur AVD sans Play Store.

---

## Environnement

| Composant | Détail |
|-----------|--------|
| Émulateur | AVD Android API 29/30, x86_64, sans Play Store |
| MobSF | Docker (`opensecurity/mobile-security-framework-mobsf:latest`) |
| Application cible | DIVA (Damn Insecure and Vulnerable Android App) — 13 challenges |
| Instrumentation | Frida Server (déployé automatiquement par MobSF) |
| Proxy HTTPS | Certificat CA MobSF installé globalement sur l'émulateur |
| Interface | http://127.0.0.1:8000 — login : `mobsf` / `mobsf` |

---

## Pourquoi un AVD sans Play Store ?

Un émulateur sans Google Play Services présente trois avantages directs pour l'analyse de sécurité :

- **Pas de bruit de fond** — aucun processus Google (Play Services, Firebase) ne pollue les logs ni le trafic réseau.
- **Proxy HTTPS global** — MobSF installe son certificat CA sans conflit avec les services système Google.
- **Compatibilité Frida** — l'émulateur est rootable et `/system` est accessible en écriture jusqu'à l'API 30. Les versions supérieures ne sont pas supportées par MobSF pour l'analyse dynamique complète.

---

## Architecture du lab

```
┌─────────────────────────────────────────────────────┐
│                   Machine hôte (PC)                  │
│                                                       │
│  ┌──────────────────┐      ┌────────────────────┐   │
│  │  AVD API 30      │      │  MobSF (Docker)    │   │
│  │  (sans Play)     │      │  :8000             │   │
│  │                  │◄────►│                    │   │
│  │  DIVA installé   │ ADB  │  Frida Server      │   │
│  │  Frida Client    │      │  Proxy HTTPS       │   │
│  │  CA MobSF        │      │  Logcat Stream     │   │
│  └──────────────────┘      │  Network Capture   │   │
│     emulator-5554          │  Report Generator  │   │
│                            └────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## Étape 1 — Création de l'AVD sans Play Store

Dans Android Studio → Tools → AVD Manager → Create Virtual Device :

- Appareil : Pixel 5 ou Pixel 6
- System Image : **Android API 29 ou 30, x86_64, SANS Google Play**
- Nom : `MobSF_DIVA_API_30`

L'absence du Play Store est vérifiable au lancement : aucune icône Play Store dans le launcher, aucun processus `com.google.android.gms` dans les logs.

---

## Étape 2 — Cloner MobSF et lancer l'émulateur

```bash
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
cd Mobile-Security-Framework-MobSF
```

Lancement de l'émulateur via le script MobSF (qui applique les patches root nécessaires) :

```bash
# Linux/Mac
./scripts/start_avd.sh MobSF_DIVA_API_30

# Windows
scripts\start_avd.ps1
```

Vérification ADB :

```bash
adb devices
# Résultat attendu :
# emulator-5554   device
```

L'identifiant `emulator-5554` est noté — il sera injecté dans la variable d'environnement MobSF.

---

## Étape 3 — Lancement de MobSF via Docker

```bash
docker pull opensecurity/mobile-security-framework-mobsf:latest

docker run -it --rm \
  -p 8000:8000 \
  -e MOBSF_ANALYZER_IDENTIFIER=emulator-5554 \
  opensecurity/mobile-security-framework-mobsf:latest
```

**Ordre impératif :** l'émulateur doit être démarré et visible dans `adb devices` avant de lancer le conteneur Docker. MobSF vérifie la connexion ADB au démarrage — si l'émulateur est absent, l'analyse dynamique est désactivée.

Interface accessible sur `http://127.0.0.1:8000` avec les identifiants `mobsf / mobsf`.

---

## Étape 4 — Upload et analyse statique de DIVA

DIVA est uploadé via MobSF → **Upload & Analyze**. L'analyse statique se lance automatiquement et produit un premier rapport couvrant :

- Permissions déclarées dans le Manifest
- Activities, Services, Receivers exportés
- Secrets hardcodés détectés dans le bytecode
- Vulnérabilités OWASP MASVS identifiées statiquement

Ce rapport statique sert de base pour orienter les tests dynamiques sur les zones à risque identifiées.

---

## Étape 5 — Analyse dynamique

Depuis le rapport statique → **Start Dynamic Analyzer**. MobSF exécute automatiquement la séquence suivante :

```
Installation de DIVA sur l'émulateur
         │
         ▼
Déploiement de Frida Server
         │
         ▼
Installation du certificat CA MobSF (interception HTTPS)
         │
         ▼
Configuration du proxy HTTP/HTTPS global
         │
         ▼
Ouverture de l'interface Dynamic Analyzer
```

### Fonctionnalités utilisées

| Outil MobSF | Rôle | Ce qui est observé |
|-------------|------|--------------------|
| **Logcat Stream** | Logs Android en temps réel | Fuites de données sensibles, exceptions, traces de debug |
| **Network Traffic** | Capture HTTP/HTTPS | Requêtes en clair, paramètres POST, tokens exposés |
| **Frida — Spawn & Inject** | Instrumentation runtime | Hook de méthodes Java, bypass de checks |
| **Exported Activity Tester** | Test des activités exposées | Activities accessibles sans permission |
| **TLS/SSL Security Tester** | Vérification de la validation des certificats | Certificate pinning absent, confiance aveugle |
| **File Monitor** | Surveillance du système de fichiers | Fichiers créés en clair dans `/data/data/` |
| **Intent Monitor** | Capture des intents | Intents implicites non sécurisés |
| **Generate Report** | Rapport final consolidé | Export PDF/JSON de tous les éléments collectés |

---

## Étape 6 — Exploration des challenges DIVA

DIVA expose 13 challenges couvrant les principales vulnérabilités mobiles. Les plus significatifs explorés durant le lab :

**Insecure Logging** — des données sensibles (credentials, tokens) sont écrites dans Logcat avec `Log.d()` ou `Log.e()`. Le Logcat Stream MobSF les capture en temps réel sans aucune instrumentation supplémentaire.

**Hardcoded Secrets** — des clés API et mots de passe sont encodés en dur dans le bytecode. Détectés à la fois par l'analyse statique (strings dans le DEX) et confirmés en dynamique via Frida en hookant les méthodes concernées.

**Insecure Data Storage** — les données utilisateur sont écrites en clair dans des fichiers partagés (`SharedPreferences` sans chiffrement, fichiers dans `/sdcard/`). Le File Monitor MobSF trace chaque opération d'écriture avec le chemin complet et le contenu.

**Access Control** — des activités exportées sans vérification de permission sont accessibles depuis n'importe quelle application. L'Exported Activity Tester MobSF les déclenche directement pour confirmer l'exposition.

**Network Security** — des requêtes HTTP en clair et des connexions HTTPS sans validation de certificat sont interceptées par le proxy MobSF. Le trafic est visible en clair dans l'onglet Network Traffic.

---

## Résultats observés

L'analyse dynamique complète de DIVA produit les observations suivantes :

- Logcat expose des credentials en clair lors de la saisie dans les challenges Insecure Logging.
- Le proxy HTTPS intercepte correctement les connexions SSL de DIVA grâce au certificat CA MobSF installé sur l'émulateur — aucun certificate pinning n'est implémenté dans DIVA.
- Frida confirme l'absence de détection de root ou d'anti-tampering dans l'application.
- Le rapport généré par MobSF consolide statique + dynamique en un seul document exportable.

---

## Nettoyage de l'environnement après le lab

| Action | Pourquoi |
|--------|---------|
| **Unset HTTP(S) Proxy** | Désactive la redirection du trafic vers MobSF |
| **Remove Root CA** | Supprime le certificat MobSF de l'émulateur |
| Arrêt du conteneur Docker | `Ctrl+C` dans le terminal Docker |
| Fermeture de l'émulateur | Libère la RAM et les ressources CPU |

---

## Points clés retenus

- L'analyse dynamique révèle des vulnérabilités invisibles à l'analyse statique — notamment les comportements réseau, les écritures fichiers en temps réel et les fuites en mémoire.
- MobSF unifie en un seul outil ce qui nécessiterait normalement Frida, Burp Suite, ADB logcat et un explorateur de fichiers séparés.
- L'émulateur sans Play Store est indispensable : un émulateur Google Play introduit du bruit réseau et des restrictions système qui empêchent MobSF de fonctionner correctement.
- La limite API 30 est une contrainte actuelle de MobSF — au-delà, `/system` n'est plus accessible en écriture et Frida Server ne peut pas être déployé automatiquement.
- DIVA est une cible pédagogique idéale car elle concentre les 13 vulnérabilités les plus fréquentes du classement OWASP Mobile Top 10 dans une seule application.

---

## Références

- MobSF Documentation : https://github.com/MobSF/docs
- DIVA Android : https://github.com/payatu/diva-android
- OWASP MASTG : https://mas.owasp.org/MASTG/

---

*Lab réalisé dans le cadre du cours Sécurité des Applications Mobiles — ENSA Marrakech, Filière GCDSTE*
