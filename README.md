# Rapport de Lab — Analyse Dynamique Android avec Frida

## A. Informations générales

| Champ | Valeur |
|---|---|
| Titre | Analyse dynamique — Instrumentation avec Frida |
| Date d'analyse | 27 avril 2026 |
| Analyste | AMAR SAAD |
| Application cible | Pizza Recipes — `com.example.pizzarecipes` |
| Provenance | Lab encadré — cadre pédagogique (MLIAEdu) |
| Outils utilisés | Frida 17.9.3 — frida-tools 14.8.1 — ADB 37.0.0 — Python 3.11 |
| Plateforme | Windows 10 — Android Emulator 5556 (x86\_64) |
| Version frida-server | frida-server-17.9.3-android-x86\_64 |

---

## B. Résumé exécutif

Ce lab couvre l'installation, le déploiement et la prise en main de **Frida** dans un contexte d'analyse de sécurité mobile Android. Les travaux réalisés incluent : l'installation du client Frida, le déploiement de `frida-server` sur émulateur, la validation de la connexion, l'exploration de la console interactive et l'instrumentation de méthodes Java et natives sensibles.

Actions couvertes :
- Installation et vérification de l'environnement Frida.
- Déploiement de `frida-server` sur émulateur Android en mode root.
- Injection de scripts minimaux (Java + natif).
- Observation des bibliothèques de chiffrement, du réseau et du stockage.
- Hooking de méthodes Java (SharedPreferences, SQLite, Debug, Runtime).

---

## C. Étapes réalisées

### Étape 1 — Installation du client Frida

Installation de `frida` et `frida-tools` via pip. Vérification :

```
pip install frida-tools
→ frida-tools 14.8.1 déjà satisfait
→ frida 17.9.1 déjà satisfait

python -c "import frida; print(frida.__version__)"
→ 17.9.1
```

> ✅ Client Frida installé et fonctionnel.

---

### Étape 2 — Installation des outils Android (ADB)

```
adb version
→ Android Debug Bridge version 1.0.41 — Version 37.0.0-14910828

adb devices
→ emulator-5554   device
```

> ✅ Émulateur reconnu par ADB.

---

### Étape 3 — Déploiement de `frida-server`

**Architecture détectée :**

```
adb shell getprop ro.product.cpu.abi
→ x86_64
```

Binaire téléchargé : `frida-server-17.9.3-android-x86_64`

**Passage en mode root et désactivation SELinux :**

```
adb root
→ restarting adbd as root

adb shell
# whoami
→ root
# setenforce 0
```

**Transfert et lancement :**

```
adb push frida-server /data/local/tmp
→ 1 file pushed — 110787848 bytes in 0.756s (139.7 MB/s)

adb shell
# cd /data/local/tmp
# chmod +x frida-server
# ./frida-server &
→ [1] 7873
```

> ✅ `frida-server` actif sur l'émulateur en tant que root.

---

### Étape 4 — Test de connexion depuis le PC

```
python -m frida_tools.ps -U
→ Liste complète des processus Android affichée (dont Pizza Recipes PID 5309)

python -m frida_tools.ps -Uai
→  5309  Pizza Recipes  com.example.pizzarecipes
```

> ✅ Connexion établie. Application cible identifiée.

---

### Étape 5 — Injection minimale

**Script `hello_native.js` :**

```javascript
console.log("[+] Script chargé");

Interceptor.attach(Module.getExportByName(null, "recv"), {
  onEnter(args) {
    console.log("[+] recv appelée");
  }
});
```

**Lancement :**

```
frida -U -f com.example.pizzarecipes -l hello_native.js
→ Connected to Android Emulator 5556 (id=emulator-5556)
→ Spawning `com.example.pizzarecipes`...
→ [+] Script chargé
→ Spawned `com.example.pizzarecipes`. Resuming main thread!
```

> ✅ Script injecté avec succès. Hook natif actif sur `recv`.

---

### Étape 6 — Console interactive Frida

Commandes exécutées dans la console après injection sur `com.example.pizzarecipes` :

| Commande | Résultat |
|---|---|
| `Process.arch` | `x64` |
| `Process.platform` | `linux` |
| `Process.id` | PID du processus |
| `Process.mainModule` | Module principal identifié |
| `Process.getModuleByName("libc.so")` | Adresse de base, chemin et taille de `libc.so` |
| `Process.getModuleByName("libc.so").getExportByName("recv")` | Adresse mémoire de `recv` retournée |
| `Process.enumerateModules()` | Liste complète des bibliothèques chargées |
| `Process.enumerateThreads()` | Threads actifs listés |
| `Process.enumerateRanges('r-x')` | Régions mémoire exécutables identifiées |
| `Java.available` | `true` |

> ✅ Architecture et organisation mémoire documentées. Runtime Java accessible.

---

### Étape 7 — Bibliothèques de chiffrement, réseau et stockage

Filtrage des modules chargés :

```javascript
Process.enumerateModules().filter(m =>
  m.name.indexOf("ssl") !== -1 ||
  m.name.indexOf("crypto") !== -1 ||
  m.name.indexOf("boring") !== -1
)
```

Hooks installés et testés :

| Script | Fonction ciblée | Résultat |
|---|---|---|
| `hook_connect.js` | `libc.so::connect` | Connexions réseau observées |
| `hook_network.js` | `libc.so::send` / `recv` | Trafic réseau natif intercepté |
| `hook_file.js` | `libc.so::open` / `read` | Accès fichiers internes observés |

> ✅ Bibliothèques `libssl.so` / `libcrypto.so` présentes. Activité réseau native confirmée.

---

### Étape 8 — Hooking de méthodes Java sensibles

| Script | Cible | Observations |
|---|---|---|
| `hook_prefs.js` | `SharedPreferencesImpl` | Clés et valeurs lues interceptées |
| `hook_prefs_write.js` | `SharedPreferencesImpl$EditorImpl` | Écritures dans les préférences observées |
| `hook_sqlite.js` | `SQLiteDatabase` | Requêtes SQL visibles à l'exécution |
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
| Description | `Debug.isDebuggerConnected()` hookable et son retour modifiable trivialement via Frida. |
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
<img width="822" height="215" alt="Screenshot 2026-05-02 143022" src="https://github.com/user-attachments/assets/238ffecc-a7ba-4cee-81fa-2d269107ffac" />

<img width="752" height="199" alt="Screenshot 2026-05-02 143337" src="https://github.com/user-attachments/assets/a59467e9-c2f2-4675-a7c3-83f376e9244d" />

<img width="764" height="81" alt="Screenshot 2026-05-02 205118" src="https://github.com/user-attachments/assets/0f962642-790f-427e-9f1d-44f4d476c6c4" />

<img width="556" height="167" alt="Screenshot 2026-05-02 133108" src="https://github.com/user-attachments/assets/caa91b6b-01c0-4305-9901-11ed3d112ff9" />

<img width="455" height="127" alt="Screenshot 2026-05-02 133130" src="https://github.com/user-attachments/assets/239d2b76-d3ed-46d1-96a5-c26b8c51cc40" />

<img width="507" height="115" alt="Screenshot 2026-05-02 133154" src="https://github.com/user-attachments/assets/d3a22e37-25b4-41f0-87ff-4c97d90af7b6" />

<img width="821" height="370" alt="Screenshot 2026-05-02 141612" src="https://github.com/user-attachments/assets/e80abd0c-8a6e-445f-8af4-7cdbd1a56fe3" />


---

> ⚠️ **Avertissement :** Ce lab a été réalisé dans un cadre strictement pédagogique sur un environnement autorisé. Aucune application tierce n'a été ciblée. Ce rapport ne doit pas être utilisé à des fins malveillantes.
