# 🔐 LAB 14 — Bypass de la Détection Root Android avec Frida

---

## 🧾 Informations générales

| Champ | Détail |
|-------|--------|
| Application cible | DIVA (`jakhar.aseem.diva`) |
| Outil principal | Frida |
| Type d'analyse | Instrumentation dynamique — hooking JNI |
| Environnement | Émulateur Android / Téléphone rooté (USB Debug activé) |

---

## 🎯 Objectif

Comprendre et mettre en œuvre le **contournement de la détection de root** dans une application Android en utilisant **Frida** pour hooker dynamiquement les fonctions sensibles, sans modifier le code source de l'APK.

---

## 🧰 Prérequis

| Outil | Rôle |
|-------|------|
| Python | Runtime pour Frida et frida-tools |
| Frida + frida-tools | Framework d'instrumentation dynamique |
| ADB | Communication avec l'appareil Android |
| frida-server | Daemon Frida côté Android |
| Application DIVA | Cible de l'analyse |

---

## ⚙️ Étape 1 — Vérification de l'environnement

```bash
frida --version
adb devices
```

<img width="1081" height="454" alt="image" src="https://github.com/user-attachments/assets/a7d6b179-d3a6-4ef0-9e2d-c5c40bf966ff" />


---

## ⚙️ Étape 2 — Déploiement de frida-server

Transfert et lancement du daemon Frida sur l'émulateur :

```bash
adb -s emulator-5556 push frida-server /data/local/tmp/
adb -s emulator-5556 shell chmod 755 /data/local/tmp/frida-server
adb -s emulator-5556 shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

<img width="980" height="862" alt="image" src="https://github.com/user-attachments/assets/a1dc2ba5-7e0b-4ef0-b2d1-4e5532a51c01" />


---

## ⚙️ Étape 3 — Vérification de la connexion

```bash
frida-ps -Uai
```



---

## 📝 Étape 4 — Script de bypass root

Création du fichier `bypass.js` avec 3 vecteurs de contournement :

```javascript
Java.perform(function () {

    console.log("[+] Root Bypass Loaded");

    // Vecteur 1 — Neutraliser Build.TAGS
    try {
        var Build = Java.use("android.os.Build");
        Build.TAGS.value = "release-keys";
        console.log("[+] Build.TAGS bypass appliqué");
    } catch (e) {}

    // Vecteur 2 — Intercepter File.exists()
    try {
        var File = Java.use("java.io.File");
        File.exists.implementation = function () {
            var path = this.getAbsolutePath();
            if (path.includes("su") || path.includes("busybox")) {
                console.log("[+] Vérification fichier bloquée : " + path);
                return false;
            }
            return this.exists();
        };
    } catch (e) {}

    // Vecteur 3 — Intercepter Runtime.exec()
    try {
        var Runtime = Java.use("java.lang.Runtime");
        Runtime.exec.overload('java.lang.String').implementation = function(cmd) {
            if (cmd.includes("su") || cmd.includes("busybox")) {
                console.log("[+] Commande bloquée : " + cmd);
                return this.exec("echo");
            }
            return this.exec(cmd);
        };
    } catch (e) {}

});
```

### Détail des vecteurs de bypass

| Vecteur | Cible | Mécanisme |
|---------|-------|-----------|
| `Build.TAGS` | Propriété système | Forçage de la valeur à `"release-keys"` |
| `File.exists()` | Vérification de fichiers root | Retourne `false` si le chemin contient `su` ou `busybox` |
| `Runtime.exec()` | Exécution de commandes shell | Remplace les commandes root par `echo` (no-op) |

---

## 🚀 Étape 5 — Injection du script avec Frida

```bash
frida -U -f jakhar.aseem.diva -l bypass.js
```

<img width="980" height="862" alt="image" src="https://github.com/user-attachments/assets/95b92355-f08f-4d98-bb11-24b5e533e6a5" />


---

## ✅ Résultats obtenus

| Vérification | Résultat |
|-------------|----------|
| Script injecté | ✅ Succès |
| Logs Frida visibles dans le terminal | ✅ Confirmé |
| `Build.TAGS` neutralisé | ✅ Valeur forcée à `release-keys` |
| Vérifications de fichiers (`su`, `busybox`) bloquées | ✅ Retournent `false` |
| Commandes shell root bloquées | ✅ Remplacées par `echo` |
| Détection root contournée dans DIVA | ✅ Application accessible |

---

## 🚨 Analyse des techniques exploitées

### 🔴 Hook de `Build.TAGS`

Certaines apps vérifient `android.os.Build.TAGS` — sur un appareil root, cette valeur vaut `"test-keys"` au lieu de `"release-keys"`. Frida permet de forcer la valeur en mémoire à runtime.

### 🔴 Hook de `File.exists()`

La méthode la plus courante de détection root consiste à vérifier l'existence de fichiers comme `/system/bin/su`. Le hook intercepte chaque appel et retourne `false` pour les chemins suspects.

### 🔴 Hook de `Runtime.exec()`

Certaines apps tentent d'exécuter `su` directement. Le hook intercepte la commande avant son exécution et la remplace par une commande inoffensive.

---

## 🛡️ Recommandations pour les développeurs

**1. Implémenter les vérifications root en code natif (NDK) :**
```c
// Beaucoup plus difficile à hooker via Frida
int checkRootNative() {
    return access("/system/bin/su", F_OK) == 0;
}
```

**2. Utiliser plusieurs vecteurs de détection simultanément :**
- Vérification de fichiers système
- Détection de Magisk / SuperSU
- Vérification des propriétés `ro.debuggable` et `ro.secure`
- Contrôle d'intégrité de la signature APK

**3. Détecter la présence de Frida elle-même :**
```java
// Vérifier les ports Frida ouverts (27042 par défaut)
// Détecter frida-agent dans /proc/maps
```

**4. Intégrer une solution RASP (Runtime Application Self-Protection)**

---

## 🧠 Conclusion

Ce LAB démontre la facilité avec laquelle les protections root **purement Java** peuvent être contournées via Frida en quelques minutes :

| Technique | Difficulté de bypass | Recommandation |
|-----------|---------------------|----------------|
| Vérifications Java (`File`, `Build`, `Runtime`) | 🔴 Très facile | À éviter seul |
| Vérifications natives NDK | 🟠 Modérée | Recommandée |
| Multi-couches + RASP + anti-Frida | 🟢 Difficile | Idéale en production |

> 👉 La sécurité mobile efficace repose sur une **défense en profondeur** — une seule couche de protection Java ne suffit pas face à l'instrumentation dynamique.
