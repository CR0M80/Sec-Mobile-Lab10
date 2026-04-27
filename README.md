# Rapport de Lab — Analyse Dynamique Android avec Frida

## A. Informations générales

| Champ | Valeur |
|---|---|
| Titre | Analyse dynamique — Instrumentation avec Frida |
| Date d'analyse | 27 avril 2026 |
| Analyste | AMAR SAAD |
  | Application cible | UncrackableLevel1.apk |
| Outils utilisés | Frida 16.x — frida-tools — ADB — Python 3.11 |
| Plateforme | Android 8+ / Émulateur AVD x86\_64 |
| Version frida-server | frida-server-17.x.x-android-x86\_64 |

---

## B. Résumé exécutif

Ce lab couvre l'installation, le déploiement et la prise en main de **Frida** dans un contexte d'analyse de sécurité mobile Android. Les travaux réalisés incluent : l'installation du client Frida, le déploiement de `frida-server` sur appareil, la validation de la connexion, l'exploration de la console interactive et l'instrumentation de méthodes Java et natives sensibles.

**Niveau de complexité : 🟡 Intermédiaire**

Actions couvertes :
- Installation et vérification de l'environnement Frida.
- Déploiement de `frida-server` et connexion ADB.
- Injection de scripts minimaux (Java + natif).
- Observation des bibliothèques de chiffrement, du réseau et du stockage.
- Hooking de méthodes Java (SharedPreferences, SQLite, Debug, Runtime).

---

## C. Étapes réalisées

### Étape 1 — Installation du client Frida

Installation de `frida` et `frida-tools` via pip. Vérification de la version avec `frida --version` et `python -c "import frida; print(frida.__version__)"`.

> ✅ Version `16.x.y` affichée sans erreur.

---

### Étape 2 — Installation des outils Android (ADB)

Deja fait

> ✅ Appareil reconnu correctement.

<img width="1238" height="359" alt="image" src="https://github.com/user-attachments/assets/27044798-a9b3-4087-ac7d-57c2bd4eabf9" />


---

### Étape 3 — Déploiement de `frida-server`

Architecture détectée : `x86_64` via `adb shell getprop ro.product.cpu.abi`. Téléchargement du binaire correspondant depuis GitHub Releases, transfert vers `/data/local/tmp/`, attribution des droits d'exécution (`chmod 755`), lancement en arrière-plan et redirection des ports `27042`/`27043`.

> ✅ `frida-server` actif sur l'appareil.

---

### Étape 4 — Test de connexion

Validation via `frida-ps -U` et `frida-ps -Uai` : liste des processus et applications de l'appareil visible depuis le PC.

> ✅ Connexion établie.

---

### Étape 5 — Injection minimale

Deux scripts de validation injectés :
- `hello.js` — vérifie `Java.perform()` → sortie : `[+] Frida Java.perform OK`.
- `hello_native.js` — hook sur `recv` → sortie : `[+] recv appelée` lors d'activité réseau.

> ✅ Instrumentation Java et native fonctionnelles.

---

### Étape 6 — Console interactive Frida

Exploration du processus via les commandes de la console :

| Commande | Résultat |
|---|---|
| `Process.arch` | `x64` |
| `Process.platform` | `linux` |
| `Process.id` | PID du processus |
| `Process.mainModule` | Module principal identifié |
| `Process.getModuleByName("libc.so")` | Adresse, chemin et taille de `libc.so` |
| `Process.enumerateModules()` | Liste complète des bibliothèques chargées |
| `Process.enumerateThreads()` | Threads actifs listés |
| `Process.enumerateRanges('r-x')` | Régions mémoire exécutables |
| `Java.available` | `true` |

> ✅ Architecture et organisation mémoire documentées. Runtime Java accessible.

---

### Étape 7 — Bibliothèques de chiffrement, réseau et stockage

Filtrage des modules chargés : bibliothèques `libssl.so` / `libcrypto.so` identifiées. Hooks installés sur `connect`, `send`, `recv` (activité réseau) et `open`, `read` (accès fichiers). Résultat : activité réseau native confirmée, chemins de fichiers internes observés.

> ✅ Composants cryptographiques et réseau localisés.

---

### Étape 8 — Hooking de méthodes Java sensibles

| Script | Cible | Observations |
|---|---|---|
| `hook_prefs.js` | `SharedPreferencesImpl` | Clés et valeurs lues en clair |
| `hook_prefs_write.js` | `SharedPreferencesImpl$EditorImpl` | Écritures interceptées |
| `hook_sqlite.js` | `SQLiteDatabase` | Requêtes SQL visibles |
| `hook_debug.js` | `android.os.Debug` | `isDebuggerConnected()` intercepté |
| `hook_runtime.js` | `java.lang.Runtime` | Commandes `exec()` observées |
| `hook_file_java.js` | `java.io.File` | Chemins de fichiers Java capturés |

> ✅ Méthodes Java sensibles instrumentées avec succès.

---

## D. Constats de sécurité

### Constat #1 — SharedPreferences lisibles à l'exécution

| Champ | Détail |
|---|---|
| Sévérité | 🟡 Moyenne |
| Description | Les clés et valeurs lues/écrites via SharedPreferences sont interceptables en clair par instrumentation Frida. |
| Localisation | `android.app.SharedPreferencesImpl` |
| Impact potentiel | Extraction de tokens, flags de session ou données de configuration sensibles. |
| Remédiation | Utiliser `EncryptedSharedPreferences` ou Android Keystore pour les données sensibles. |

---

### Constat #2 — Requêtes SQLite interceptables

| Champ | Détail |
|---|---|
| Sévérité | 🟡 Moyenne |
| Description | Les requêtes SQL sont visibles en clair à l'exécution via hook sur `SQLiteDatabase`. |
| Localisation | `android.database.sqlite.SQLiteDatabase` |
| Impact potentiel | Cartographie de la structure des données locales. |
| Remédiation | Chiffrer la base locale avec SQLCipher. |

---

### Constat #3 — Détection de débogueur contournable

| Champ | Détail |
|---|---|
| Sévérité | 🟢 Faible |
| Description | `Debug.isDebuggerConnected()` hookable et retour modifiable trivialement via Frida. |
| Localisation | `android.os.Debug` |
| Impact potentiel | Contournement des protections anti-debug Java. |
| Remédiation | Combiner vérifications natives et Play Integrity API. |

---

### Constat #4 — Activité réseau native exposée

| Champ | Détail |
|---|---|
| Sévérité | 🟡 Moyenne |
| Description | Les fonctions `connect`, `send`, `recv` de `libc.so` sont directement interceptables sans protection. |
| Localisation | `libc.so` — fonctions réseau bas niveau |
| Impact potentiel | Observation des métadonnées réseau sans déchiffrement. |
| Remédiation | Implémenter Certificate Pinning et chiffrement applicatif de bout en bout. |

---

## E. Remédiations recommandées

| # | Priorité | Action |
|---|---|---|
| 1 | 🟡 Court terme | Remplacer SharedPreferences par `EncryptedSharedPreferences` |
| 2 | 🟡 Court terme | Chiffrer la base SQLite avec SQLCipher |
| 3 | 🟡 Court terme | Implémenter Certificate Pinning |
| 4 | 🟢 Moyen terme | Renforcer la détection anti-debug via Play Integrity API |

---

## F. Annexes

### Dépannage rencontré

| Problème | Cause | Solution |
|---|---|---|
| `frida: command not found` | PATH non configuré | Ajout du dossier Scripts Python au PATH |
| `unable to connect to remote frida-server` | Serveur arrêté | Relance + vérification `adb shell ps \| grep frida` |
| Mismatch versions client/serveur | Versions différentes | `pip install -U frida frida-tools` + re-téléchargement |
| `Permission denied` | Droits insuffisants | `chmod 755 /data/local/tmp/frida-server` |

---

> ⚠️ **Avertissement :** Ce lab a été réalisé dans un cadre strictement pédagogique sur un environnement autorisé. Aucune application tierce n'a été ciblée. Ce rapport ne doit pas être utilisé à des fins malveillantes.

<img width="1694" height="199" alt="image" src="https://github.com/user-attachments/assets/b263b953-ef79-49f3-a8ed-d597936a3615" />

<img width="1413" height="116" alt="image" src="https://github.com/user-attachments/assets/95659d34-8bf4-4665-828e-1540eb869511" />


