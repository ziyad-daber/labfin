# RAPPORT DE CHALLENGE : SNAKE.APK
**Auteur :** Ziyad Daber  
**Date :** 10 Mai 2026  
**Sujet :** Exploitation de désérialisation SnakeYAML et Contournement de Protections Android  
**Niveau :** Hard  

---

## 1. Introduction et Objectifs
L'objectif de ce challenge était d'extraire un flag caché dans une application Android nommée `Snake.apk`. L'application présente un niveau de sécurité élevé avec plusieurs couches de protection :
- **Anti-Reverse Engineering :** Détections natives et Java du root, de l'utilisation d'émulateurs et de l'outil Frida.
- **Logique Cachée :** Le flag est généré via une fonction JNI (native) déclenchée par une classe `BigBoss` qui n'est pas instanciée dans le flux normal de l'application.
- **Vulnérabilité :** Utilisation d'une version vulnérable de la librairie **SnakeYAML (CVE-2022-1471)** permettant une désérialisation non sécurisée.

## 2. Analyse Statique et Investigation
L'analyse a été réalisée principalement avec **Jadx-GUI**. Les découvertes clés sont les suivantes :

### A. Le flux d'exécution (`MainActivity`)
L'application vérifie la présence d'un extra dans l'Intent de démarrage :
- **Clé :** `SNAKE` $\rightarrow$ **Valeur attendue :** `BigBoss`.
- Si cette condition est remplie, l'application tente de lire un fichier spécifique sur le stockage externe : `/sdcard/Snake/Skull_Face.yml`.

### B. La cible : Classe `BigBoss`
La classe `com.pwnsec.snake.BigBoss` est le pivot de l'exploitation. Elle :
1. Charge une librairie native (`System.loadLibrary`).
2. Possède une méthode attendant la chaîne exacte : `"Snaaaaaaaaaaaaaake"`.
3. Appelle une fonction native `stringFromJNI()` qui imprime le flag dans les logs système (`logcat`).

### C. Les protections
L'application implémente des checks rigoureux sur `Build.TAGS`, la présence de binaires `su`, et des signatures propres aux émulateurs et à Frida, rendant l'exécution immédiate impossible sur un environnement de test classique.

## 3. Stratégie d'Exploitation (Walkthrough)

### Étape 1 : Contournement des protections (Bypass)
Frida étant détecté nativement, j'ai opté pour un **patching statique** :
1. **Décompilation :** Utilisation d'`apktool` pour extraire le code Smali.
2. **Modification :** Identification des méthodes de détection de root/émulateur. J'ai modifié les instructions de saut (`if-eqz`, `if-nez`) ou forcé le retour des fonctions de check à `0` (false) pour neutraliser les fermetures brusques.
3. **Recompilation et Signature :** Reconstruction de l'APK et signature via `apksigner` pour permettre l'installation sur l'appareil.

### Étape 2 : Construction du Payload YAML
Pour instancier la classe `BigBoss` et passer le paramètre requis, j'ai exploité la vulnérabilité de désérialisation de SnakeYAML.
**Payload créé :**
```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```
- `!!com.pwnsec.snake.BigBoss` : Force l'instanciation de la classe cible.
- `["Snaaaaaaaaaaaaaake"]` : Injecte l'argument nécessaire pour déclencher la fonction JNI.

### Étape 3 : Déclenchement et Récupération
Le déploiement s'est fait via ADB :
1. **Upload du payload :** `adb push Skull_Face.yml /sdcard/Snake/Skull_Face.yml`
2. **Lancement ciblé :** 
   `adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss`
3. **Capture du flag :** Le flag étant envoyé vers les logs, j'ai utilisé un filtre logcat :
   `adb logcat | grep -i "PWNSEC"`

## 4. Résultat Final
Le flag a été récupéré avec succès dans les logs système :

**Flag :** `PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}`

## 5. Conclusion et Recommandations
Ce challenge illustre l'importance de la sécurisation des librairies de parsing. Pour remédier à cette faille, il est recommandé :
- De mettre à jour **SnakeYAML vers la version 2.0+** (qui désactive par défaut le constructeur non sécurisé).
- D'utiliser un `SafeConstructor` pour limiter les classes pouvant être instanciées lors de la désérialisation.
- De ne pas se fier uniquement aux protections anti-root/anti-debug, car le patching Smali permet de les contourner efficacement.
