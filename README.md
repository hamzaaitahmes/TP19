# TP19
# LAB 19 – Exploitation de SnakeYAML sur Android avec Patch Smali

## Présentation du laboratoire

Ce laboratoire porte sur l'analyse et l'exploitation d'une application Android nommée **Snake.apk**. Bien qu'elle paraisse simple au premier abord, cette application intègre plusieurs mécanismes de protection ainsi qu'une vulnérabilité de désérialisation YAML exploitable.

L'objectif est de contourner les mécanismes de sécurité mis en place, notamment la détection de root, puis d'exploiter la vulnérabilité présente afin d'obtenir le flag caché dans l'application. Pour y parvenir, plusieurs techniques seront utilisées : analyse statique, modification du bytecode Smali, création d'un payload YAML malveillant, interaction avec l'émulateur Android et récupération d'informations via les journaux système.

---

# Environnement et outils utilisés

Les outils suivants ont été utilisés durant ce laboratoire :

* **ADB (Android Debug Bridge)** : communication avec l'émulateur Android ;
* **Jadx-GUI** : décompilation et analyse du code Java ;
* **apktool** : décompilation et recompilation de l'APK ;
* **uber-apk-signer** : signature de l'APK modifié ;
* **Visual Studio Code** : édition des fichiers Smali ;
* **Android Emulator** : environnement d'exécution ;
* **PowerShell et Logcat** : collecte et analyse des journaux Android.

---

## Étape 1 : Analyse de l'application avec Jadx-GUI

La première étape consiste à ouvrir **Snake.apk** dans Jadx-GUI afin d'examiner le code source décompilé.

L'analyse met en évidence le package principal `com.pwnsec.snake` ainsi que deux classes particulièrement importantes :

* `MainActivity`
* `BigBoss`

Dans la classe `MainActivity`, on observe le chargement de la bibliothèque native :

```java
System.loadLibrary("snake");
```

On identifie également plusieurs méthodes destinées à détecter si l'appareil Android est rooté :

```java
checkForDangerousBinaries()
checkForRootManagementApps()
checkForRootShell()
checkForWritableSystem()
isDeviceRooted()
```

<img width="1915" height="1078" alt="Analyse Jadx - MainActivity" src="https://github.com/user-attachments/assets/7524cfe8-58f6-430e-b813-9bedd2a8f761" />

<img width="1919" height="1079" alt="Analyse Jadx - méthodes de détection root" src="https://github.com/user-attachments/assets/f9a5272b-9e0e-4a97-82ac-073d9f9c359f" />

---

## Étape 2 : Identification du mécanisme de détection root

L'application centralise les différentes vérifications dans la méthode `isDeviceRooted()` :

```java
public static boolean isDeviceRooted(Context context) {
    return checkForDangerousBinaries()
        || checkForRootManagementApps(context)
        || checkForWritableSystem()
        || checkForRootShell();
}
```

Lors du démarrage, cette méthode est appelée depuis `onCreate()`. Si une seule vérification retourne `true`, l'application se ferme immédiatement :

```java
if (isDeviceRooted(getApplicationContext())) {
    Log.w("Rooted", "Root detected! Exiting application.");
    finish();
    System.exit(0);
}
```

Comme l'émulateur utilisé est rooté, il est nécessaire de contourner cette protection avant de poursuivre l'analyse.

---

## Étape 3 : Décompilation de l'APK avec apktool

Afin de modifier le comportement de l'application, l'APK est décompilé à l'aide d'**apktool** :

```bash
java -jar apktool.jar d Snake.apk -o snake_smali
```

Cette commande génère le dossier `snake_smali`, contenant l'ensemble du code Smali et des ressources de l'application.

<img width="1462" height="345" alt="Décompilation avec apktool" src="https://github.com/user-attachments/assets/e9e2a7ee-7eef-4010-9120-fda4314ee9b9" />

---

## Étape 4 : Analyse du code Smali

Le dossier généré est ensuite ouvert dans Visual Studio Code afin de localiser la méthode `isDeviceRooted()`.

Le code Smali original contient plusieurs appels successifs aux fonctions de détection root.

<img width="1471" height="197" alt="Code Smali original - isDeviceRooted" src="https://github.com/user-attachments/assets/09f8c917-ea21-4dd8-8950-5c01a1fb9f74" />

---

## Étape 5 : Modification de la détection root

Pour désactiver complètement la protection, la méthode `isDeviceRooted()` est remplacée par une version retournant systématiquement la valeur `false`.

Dans le langage Smali, la valeur booléenne `false` correspond à `0x0`.

La méthode modifiée devient ainsi :

```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .locals 1

.end method
```

Grâce à cette modification, l'application considère désormais que l'appareil n'est jamais rooté.

<img width="862" height="913" alt="Patch Smali - VS Code" src="https://github.com/user-attachments/assets/2a6b5228-8c18-49dc-becb-6170735116df" />

<img width="732" height="184" alt="Code patché final" src="https://github.com/user-attachments/assets/30db5c91-06bd-4a16-801d-0a87bef0d4b2" />

---

## Étape 6 : Recompilation de l'application

Après modification du code Smali, l'application est reconstruite avec apktool :

```bash
java -jar apktool.jar b snake_smali -o snake_patched_unsigned.apk
```

Cette étape génère un nouvel APK contenant les modifications effectuées.

<img width="1461" height="259" alt="Recompilation avec apktool" src="https://github.com/user-attachments/assets/10371d5c-4575-4802-8988-05c357f4ccf8" />

---

## Étape 7 : Signature et installation de l'APK

Android impose qu'un APK soit signé avant son installation.

Après avoir utilisé **uber-apk-signer** pour signer le fichier généré, l'ancienne version de l'application est supprimée :

```bash
adb uninstall com.pwnsec.snake
```

Puis l'APK modifié est installé :

```bash
adb install snake_patched_unsigned-aligned-debugSigned.apk
```

Une installation réussie affiche le message :

```text
Performing Streamed Install
Success
```

<img width="1812" height="210" alt="Installation ADB réussie" src="https://github.com/user-attachments/assets/7b992611-b835-43c3-9b51-e0523b1422a5" />

---

## Étape 8 : Attribution des permissions nécessaires

L'application doit pouvoir lire un fichier YAML stocké sur la mémoire externe.

La permission est accordée à l'aide de la commande suivante :

```bash
adb shell pm grant com.pwnsec.snake android.permission.READ_EXTERNAL_STORAGE
```

---

## Étape 9 : Analyse de la classe BigBoss

L'étude de la classe `BigBoss` révèle qu'elle charge également la bibliothèque native `snake`.

Son constructeur attend une chaîne spécifique :

```java
public BigBoss(String str)
```

Lorsque la valeur fournie est exactement :

```text
Snaaaaaaaaaaaaaake
```

la méthode native `stringFromJNI()` est exécutée. Son résultat est ensuite converti en texte lisible grâce à `hexToAscii()` puis enregistré dans les logs Android.

```java
Log.d("BigBoss: ", hexToAscii(strStringFromJNI));
```

<img width="1681" height="983" alt="Analyse de la classe BigBoss dans Jadx" src="https://github.com/user-attachments/assets/551b697b-44ae-4e0f-b3ab-d0cefc0bce0d" />

---

## Étape 10 : Création du payload YAML

Un fichier nommé `Skull_Face.yml` est créé avec le contenu suivant :

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```

La syntaxe `!!` utilisée par SnakeYAML permet d'instancier directement une classe Java lors de la désérialisation.

Ainsi, lors du chargement du fichier, l'application crée un objet `BigBoss`, déclenche l'appel natif et génère le flag dans les journaux Android.

---

## Étape 11 : Déploiement du payload

Le fichier YAML doit être placé dans le répertoire attendu par l'application :

```bash
adb shell mkdir -p /sdcard/snake
adb push Skull_Face.yml /sdcard/snake/Skull_Face.yml
adb shell ls -l /sdcard/snake/
```

La présence du fichier dans ce dossier confirme que le payload est correctement déployé.

---

## Étape 12 : Déclenchement de l'exploitation

L'application attend également un paramètre Intent nommé `SNAKE` avec la valeur `BigBoss`.

Le lancement s'effectue à l'aide de la commande suivante :

```bash
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
```

Après exécution, l'application démarre normalement et affiche :

```text
Here's to you Boss
1932-1995
```

La protection root ayant été neutralisée, l'exploitation peut désormais s'exécuter correctement.

<img width="395" height="806" alt="Application démarrée avec succès" src="https://github.com/user-attachments/assets/e2d55a1d-10e0-4bc1-8dac-f537a21a716a" />

---

## Étape 13 : Récupération du flag

Le flag n'est pas affiché directement dans l'interface utilisateur. Il est enregistré dans les journaux Android sous le tag `BigBoss`.

Pour le récupérer, il suffit d'utiliser Logcat :

```powershell
adb logcat | Select-String -Pattern "PWNSEC"
```

ou :

```powershell
adb logcat | Select-String -Pattern "BigBoss"
```

Le flag apparaît alors dans les journaux au format :

```text
PWNSEC{...}
```

---

# Conclusion

Ce laboratoire illustre une chaîne complète d'exploitation Android combinant plusieurs techniques complémentaires : analyse statique, reverse engineering, modification du bytecode Smali, exploitation d'une vulnérabilité de désérialisation SnakeYAML et récupération d'informations via Logcat.

L'étude met également en évidence l'importance des mécanismes de validation et les risques liés à la désérialisation non sécurisée d'objets Java dans les applications Android.
