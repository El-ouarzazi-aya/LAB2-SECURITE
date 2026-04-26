# LAB2-SECURITE
# Lab Sécurité Android — Rooting & Intégrité Système

> **Environnement :** AVD uniquement — données fictives — réseau de test isolé  
> **Application testée :** InsecureBankv2 v1.0  
> **Support :** Pixel 4 — Android 11 (API 30) — x86_64  
> **Date :** 26 avril 2026

---

## Objectif

Comprendre l'impact du rooting sur la sécurité Android : mécanismes de protection, vérification d'intégrité au démarrage, et observation des comportements système normalement inaccessibles.

---

## Prérequis

- Android Studio installé
- AVD créé (Pixel 4 / API 30 recommandé)
- ADB accessible dans le PATH

```powershell
# Vérifier que ADB est disponible
adb devices
# Attendu : emulator-5554   device
```

> **PATH manquant ?** Ajoute le dossier platform-tools :
> ```powershell
> $env:Path += ";C:\Users\<USER>\AppData\Local\Android\Sdk\platform-tools"
> ```

---

## Règles du lab

| Règle | Détail |
|---|---|
| Périmètre | App de test + AVD dédié uniquement |
| Données | Fictives — aucune donnée personnelle |
| Réseau | Environnement de test isolé |
| Reset | Wipe AVD obligatoire en fin de séance |

---

## Étapes du lab

### 1. Rooter l'AVD

Ferme l'émulateur depuis Android Studio, puis relance-le avec :

```powershell
& "C:\Users\<USER>\AppData\Local\Android\Sdk\emulator\emulator.exe" -avd Pixel_4 -writable-system
```

Ensuite :

```powershell
adb root      # Relance adbd en mode root
adb remount   # Monte /system en lecture/écriture
```

> Si `adb remount` échoue, désactive d'abord verity :
> ```powershell
> adb disable-verity
> adb reboot
> adb root
> adb remount
> ```

### 2. Vérifications post-root

```powershell
adb shell id                                    # → uid=0(root)
adb shell getprop ro.boot.verifiedbootstate     # → orange
adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.vbmeta.device_state
adb shell "su -c id"
```

Résultats observés dans ce lab :

| Commande | Résultat | Interprétation |
|---|---|---|
| `adb shell id` | `uid=0(root)` | ✅ Privilèges root actifs |
| `verifiedbootstate` | `orange` | ✅ Verity désactivé, système modifié |
| `su -c id` | `invalid uid/gid '-c'` | ⚠️ Normal sur émulateur Google |

### 3. Installer l'application de test

```powershell
# Télécharger InsecureBankv2
Invoke-WebRequest -Uri "https://github.com/dineshshetty/Android-InsecureBankv2/raw/master/InsecureBankv2.apk" -OutFile "InsecureBankv2.apk"

# Installer sur l'AVD
adb install InsecureBankv2.apk
# Attendu : Success

# Vérifier la version
adb shell dumpsys package com.android.insecurebankv2 | findstr versionName
# → versionName=1.0
```

### 4. Scénarios de test

#### Scénario 1 — Ouverture de l'app

```powershell
adb shell am start -n com.android.insecurebankv2/.LoginActivity
```

**Observation :** Écran login avec bouton "Autofill Credentials" — risque de fuite d'identifiants.

#### Scénario 2 — Tentative de connexion

Saisir `username` / `password` et appuyer Login.

**Observation :** Configuration serveur `IP:10.0.2.2 Port:8888` exposée en clair dans l'interface — violation MASVS NETWORK-1.

#### Scénario 3 — Inspection des fichiers (root requis)

```powershell
# Lister les fichiers de l'app
adb shell ls /data/data/com.android.insecurebankv2/

# Inspecter la base de données
adb shell "sqlite3 /data/data/com.android.insecurebankv2/databases/mydb .dump"

# Extraire la DB en local
adb pull /data/data/com.android.insecurebankv2/databases/mydb mydb.sqlite
```

**Observation :** Base SQLite accessible en clair — tables `android_metadata`, `names`, `sqlite_sequence` visibles sans chiffrement — violation MASVS STORAGE-1.

### 5. Journalisation

```powershell
adb logcat -d | Out-File logcat_root_check.txt
adb logcat -d | findstr -i "insecurebank" | Out-File logcat_scenarios.txt
```

---

## Concepts clés

### Verified Boot — états

| Couleur | Signification |
|---|---|
| 🟢 `green` | Système intact, signé par le fabricant |
| 🟠 `orange` | Bootloader déverrouillé, système modifié ← **observé dans ce lab** |
| 🟡 `yellow` | Clé personnalisée utilisée |
| 🔴 `red` | Intégrité compromise |

### Chaine de confiance

```
ROM (immuable) → Bootloader → Kernel → /system (dm-verity) → Applications
```

Chaque maillon vérifie le suivant. Si un maillon est compromis, toute la chaîne l'est.

### AVB (Android Verified Boot 2.0)

- Vérification d'intégrité par hashtrees cryptographiques sur chaque partition
- Protection **anti-rollback** : empêche de revenir à une ancienne version contenant des failles

### Impact du rooting

| Protection | État normal | État rooté |
|---|---|---|
| Sandbox | App isolée, UID dédié | Contournable |
| Permissions | Consentement requis | Ignorable avec `su` |
| `/system` | Lecture seule | Lecture/écriture |
| Verified Boot | Vérifié (green) | Désactivé (orange) |
| `/data/data/` | Inaccessible | Lisible et modifiable |

---

## OWASP MASVS — exigences observées

### STORAGE-1
> Les données sensibles ne doivent jamais être stockées en clair sur l'appareil.

**Violation observée :** Base SQLite `mydb` lisible en clair via `adb shell` avec privilèges root.

### NETWORK-1
> Les communications réseau doivent utiliser TLS avec vérification des certificats.

**Violation observée :** Configuration serveur `IP:Port` exposée en clair dans l'interface cliente.

---

## OWASP MASTG — tests effectués

```powershell
# Test 1 — Inspection du stockage local
adb shell ls /data/data/com.android.insecurebankv2/shared_prefs/
adb shell "sqlite3 /data/data/com.android.insecurebankv2/databases/mydb .dump"

# Test 2 — Analyse des logs
adb logcat -d | findstr -i "insecurebank"
```

---

## Matrice des risques

| Risque | Mesure |
|---|---|
| Intégrité non garantie | AVD dédié, jamais de device personnel |
| Données sensibles exposées | Données fictives uniquement |
| Réseau non isolé | Environnement de test isolé |
| Mauvais nettoyage | Wipe AVD obligatoire en fin de séance |
| Traçabilité insuffisante | Logcat + captures horodatées |

---

## Remise à zéro (obligatoire)

```powershell
# Option 1 — Via Android Studio
# Device Manager → Pixel 4 → Wipe Data

# Option 2 — Via commande
adb emu avd stop

# Option 3 — Via fastboot (device labo uniquement)
# fastboot erase userdata
```

**Preuve requise :** Screenshot de l'assistant de configuration initial d'Android après le wipe.

---

## Checklist finale

### Début de séance
- [ ] Périmètre écrit
- [ ] AVD neuf démarré
- [ ] App InsecureBankv2 installée
- [ ] 3 scénarios documentés
- [ ] Versions Android + app notées

### Fin de séance
- [ ] Données de test supprimées
- [ ] Reset AVD effectué (Wipe Data)
- [ ] Preuve du reset (screenshot)
- [ ] Rapport sauvegardé
- [ ] Aucun compte personnel utilisé
- [ ] Fichiers locaux nettoyés (`mydb.sqlite`, `logcat*.txt`)

---

## Glossaire

| Terme | Définition |
|---|---|
| ADB | Android Debug Bridge — outil CLI pour communiquer avec un appareil Android |
| AVD | Android Virtual Device — émulateur Android dans Android Studio |
| AVB | Android Verified Boot — vérification d'intégrité au démarrage |
| dm-verity | Module kernel vérifiant l'intégrité de la partition système |
| MASVS | Mobile Application Security Verification Standard (OWASP) |
| MASTG | Mobile Application Security Testing Guide (OWASP) |
| Root | Utilisateur avec privilèges administrateur complets (uid=0) |
| Sandbox | Environnement isolé dans lequel une application s'exécute |

---

## Ressources

- [Android Security](https://source.android.com/docs/security)
- [Verified Boot](https://source.android.com/docs/security/features/verifiedboot)
- [AVB](https://source.android.com/docs/security/features/verifiedboot/avb)
- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [OWASP MASTG](https://mas.owasp.org/MASTG/)
- [InsecureBankv2](https://github.com/dineshshetty/Android-InsecureBankv2)

---

*Lab réalisé en environnement isolé — données fictives — aucun appareil personnel utilisé.*
