# FireStorm — Writeup complet du challenge

## Introduction

Ce challenge consiste à analyser une application Android nommée **FireStorm** afin de comprendre comment elle génère un mot de passe Firebase, puis à exploiter cette logique pour récupérer le flag.

L’idée centrale du challenge est la suivante : l’application contient bien une fonction capable de construire le mot de passe nécessaire à l’authentification Firebase, mais cette fonction **n’est jamais appelée dans le flux normal de l’application**. Il faut donc :

1. effectuer une **analyse statique** de l’APK avec **JADX** ;
2. identifier la méthode responsable de la génération du mot de passe ;
3. constater que cette méthode est inactive par défaut ;
4. utiliser **Frida** pour forcer son exécution au runtime ;
5. récupérer le mot de passe généré ;
6. s’authentifier sur **Firebase** ;
7. lire la base de données afin d’obtenir le **flag**.

---

## Objectif du challenge

L’objectif est de retrouver le flag stocké dans la base Firebase de l’application en combinant :

- **reverse engineering Android** ;
- **lecture des ressources embarquées** ;
- **hooking Java avec Frida** ;
- **authentification Firebase via Python**.

---

## Informations générales

- **Nom de l’application** : FireStorm
- **Package principal** : `com.pwnsec.firestorm`
- **Classe principale** : `MainActivity`
- **Technologies utilisées** :
  - JADX
  - Frida
  - Firebase Realtime Database
  - Python / Pyrebase

---

# 1. Préparation de l’environnement

Avant toute analyse dynamique, il faut installer l’application sur l’émulateur ou le téléphone de test, puis vérifier que Frida fonctionne correctement.

## Installation de l’APK

```bash
adb install FireStorm.apk
```

## Vérification de Frida

```bash
frida-ps -U
```

Cette commande permet de vérifier que :

- l’appareil est bien détecté ;
- le `frida-server` est actif ;
- l’application cible est visible.

---

# 2. Analyse statique de l’application avec JADX

La première étape utile a été d’ouvrir l’APK dans **JADX-GUI** pour inspecter le code Java décompilé ainsi que les ressources Android.

## Ce qu’il faut rechercher

Dans JADX, on se concentre principalement sur :

- le package `com.pwnsec.firestorm` ;
- la classe `MainActivity` ;
- les méthodes présentes dans cette classe ;
- les ressources `strings.xml` ;
- les éventuels appels à des fonctions natives.

## Observation principale dans `MainActivity`

Dans `MainActivity`, on trouve une méthode très intéressante nommée `Password()`.

![Figure 1 — Méthode `Password()` dans MainActivity](fire/1.png)

## Analyse de la méthode `Password()`

Cette méthode construit un mot de passe à partir de plusieurs morceaux :

- des chaînes statiques récupérées depuis `strings.xml` ;
- des sous-chaînes extraites avec `substring(...)` ;
- une partie générée dynamiquement via une fonction native.

Extrait logique observé :

```java
public String Password() {
    StringBuilder sb = new StringBuilder();
    String string = getString(R.string.Friday_Night);
    String string2 = getString(R.string.Author);
    String string3 = getString(R.string.JustRandomString);
    String string4 = getString(R.string.URL);
    String string5 = getString(R.string.IDKmaybepasswordpassword);
    String string6 = getString(R.string.Token);

    sb.append(string.substring(5, 9));
    sb.append(string4.substring(1, 6));
    sb.append(string2.substring(2, 6));
    sb.append(string5.substring(5, 8));
    sb.append(string3);
    sb.append(string6.substring(18, 26));

    return generateRandomString(String.valueOf(sb));
}
```

## Interprétation

Cette fonction ne contient pas directement le mot de passe final sous forme lisible. Elle :

1. charge plusieurs chaînes depuis les ressources Android ;
2. en extrait certaines portions ;
3. les concatène ;
4. envoie le résultat à une fonction native :
   ```java
   generateRandomString(String str)
   ```
5. retourne finalement le mot de passe généré.

Le point essentiel du challenge est que cette méthode **existe**, mais **n’est jamais appelée normalement** dans l’application.

---

# 3. Rôle des ressources Android dans ce challenge

Les ressources Android jouent un rôle fondamental ici.

Dans une application Android, les chaînes de caractères sont souvent placées dans `res/values/strings.xml`. Cela permet de centraliser les textes utilisés par l’application.

Dans ce challenge, ce fichier ne contient pas seulement des textes d’interface : il contient aussi des **informations sensibles** utiles à l’exploitation.

## Ressources intéressantes trouvées dans `strings.xml`

En explorant `strings.xml`, on retrouve notamment :

- l’email Firebase ;
- l’URL de la base de données ;
- l’API key Firebase ;
- plusieurs chaînes utilisées par la méthode `Password()`.

![Figure 2 — Première partie des chaînes utilisées par la méthode Password()](fire/4.png)

On observe par exemple :

```xml
<string name="Author">TK757567</string>
<string name="Friday_Night">It's Friday, and PwnSec CTF is here!!!!!</string>
<string name="IDKmaybepasswordpassword">v1n4of.5EY?%0z</string>
<string name="JustRandomString">or_is_it_random???</string>
<string name="Token">...</string>
<string name="URL">https://pwnsec.xyz/flag?auth=</string>
```

Plus bas dans le même fichier, on retrouve également la configuration Firebase :

![Figure 3 — Configuration Firebase extraite de `strings.xml`](fire/5.png)

Exemples observés :

```xml
<string name="firebase_database_url">https://firestorm-9d3db-default-rtdb.firebaseio.com</string>
<string name="firebase_email">TK757567@pwnsec.xyz</string>
<string name="google_api_key">AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY</string>
```

## Pourquoi c’est important

Sans `strings.xml`, il aurait été beaucoup plus difficile de :

- comprendre comment la méthode `Password()` construit son entrée ;
- récupérer l’email Firebase ;
- reconstituer la configuration nécessaire pour se connecter à la base.

Les ressources Android servent donc ici à cacher des éléments critiques du challenge de manière discrète, mais accessible à l’analyse statique.

---

# 4. Observation du comportement normal de l’application

Lorsqu’on lance l’application normalement, rien dans l’interface ne permet d’obtenir directement le mot de passe ou le flag.

L’écran affiché est uniquement un fond d’écran avec un meme, sans fonctionnalité exploitable visible pour l’utilisateur.

![Figure 4 — Interface visible au lancement de l’application](fire/2.png)

Cela confirme que la logique intéressante est bien cachée dans le code, et non exposée par l’interface.

---

# 5. Pourquoi l’analyse statique seule ne suffit pas

Même si l’analyse statique permet de comprendre une grande partie de la logique, elle ne donne pas immédiatement le mot de passe final.

En effet :

- la méthode `Password()` n’est jamais invoquée normalement ;
- elle appelle une fonction native :
  ```java
  generateRandomString(String str)
  ```
- cette partie native rend la génération finale plus difficile à reconstituer uniquement à la main.

À ce stade, la meilleure stratégie consiste donc à **forcer l’application à exécuter elle-même la méthode**, plutôt que de tenter de reproduire manuellement toute la logique native.

C’est précisément ce que permet **Frida**.

---

# 6. Exploitation dynamique avec Frida

L’idée est d’injecter un script Frida dans l’application pour :

1. attendre que l’application se lance ;
2. trouver une instance vivante de `MainActivity` ;
3. appeler manuellement la méthode `Password()` ;
4. afficher le mot de passe généré dans la console.

## Script Frida utilisé

Le script suivant a été écrit dans un fichier `frida_firestorm.js` :

![Figure 5 — Script Frida permettant d’appeler `Password()`](fire/6.png)

```javascript
Java.perform(function() {

    function getPassword() {
        console.log("[*] Début de la recherche d'instances de MainActivity...");

        Java.choose('com.pwnsec.firestorm.MainActivity', {

            onMatch: function(instance) {
                console.log("[+] MainActivity instance trouvée : " + instance);

                try {
                    var pass = instance.Password();
                    console.log("[+] Mot de passe Firebase généré : " + pass);
                } catch (e) {
                    console.log("[-] Erreur lors de l'appel de Password() : " + e);
                }
            },

            onComplete: function() {
                console.log("[*] Recherche des instances terminée.");
            }
        });
    }

    console.log("[*] Script chargé. Attente de 3 secondes avant exécution...");
    setTimeout(getPassword, 3000);
});
```

## Explication du script

### `Java.perform(...)`

Cette instruction initialise le contexte Java de Frida. Toute interaction avec les classes Java de l’application doit se faire à l’intérieur.

### `Java.choose('com.pwnsec.firestorm.MainActivity', ...)`

Cette fonction parcourt la mémoire pour trouver les instances actives de `MainActivity`.

### `onMatch: function(instance)`

À chaque instance trouvée, Frida fournit un objet `instance` représentant l’objet Java réel présent en mémoire.

### `instance.Password()`

C’est l’étape clé : on force l’exécution de la méthode `Password()` sur l’instance réelle de l’activité.

Ainsi :

- les chaînes sont récupérées normalement ;
- la fonction native est aussi exécutée dans le bon contexte ;
- le mot de passe final est retourné tel qu’il serait généré par l’application.

### `setTimeout(getPassword, 3000)`

Le délai permet de laisser à l’application le temps de démarrer correctement avant d’essayer de trouver `MainActivity`.

---

# 7. Exécution du script Frida

La commande utilisée pour lancer Frida est :

```bash
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js
```

ou, selon la variante utilisée :

```bash
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js --no-pause
```

## Résultat obtenu

Lors de l’exécution, Frida affiche bien :

- le chargement du script ;
- la détection de l’instance de `MainActivity` ;
- le mot de passe Firebase généré.

![Figure 6 — Sortie console de Frida avec le mot de passe généré](fire/3.png)

Le mot de passe récupéré dans notre cas est :

```text
C7_dotpsC7t7f_-_In_i.IdttpaofoaIIdIdnndIfC
```

Ce mot de passe est l’élément critique qui permet ensuite de s’authentifier auprès de Firebase.

---

# 8. Exploitation de la configuration Firebase

À partir de `strings.xml`, on dispose déjà des paramètres nécessaires :

- **Email Firebase** : `TK757567@pwnsec.xyz`
- **Database URL** : `https://firestorm-9d3db-default-rtdb.firebaseio.com`
- **API Key** : `AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY`

Le challenge consiste alors simplement à utiliser ces informations avec le mot de passe obtenu via Frida.

---

# 9. Script Python pour récupérer le flag

Une fois le mot de passe en main, on peut écrire un script Python pour :

1. se connecter à Firebase ;
2. s’authentifier avec l’email et le mot de passe ;
3. lire la base de données ;
4. afficher le flag.

## Script utilisé

![Figure 7 — Script Python d’authentification Firebase et lecture du flag](fire/7.png)

```python
import pyrebase

config = {
    "apiKey": "AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY",
    "authDomain": "firestorm-9d3db.firebaseapp.com",
    "databaseURL": "https://firestorm-9d3db-default-rtdb.firebaseio.com",
    "storageBucket": "firestorm-9d3db.appspot.com",
    "projectId": "firestorm-9d3db"
}

firebase = pyrebase.initialize_app(config)
auth = firebase.auth()

email = "TK757567@pwnsec.xyz"
password = "C7_dotpsC7t7f_-_In_i.IdttpaofoaIIdIdnndIfC"

user = auth.sign_in_with_email_and_password(email, password)
print("Connexion reussie. Token obtenu.")

db = firebase.database()

flag_data = db.get(user['idToken'])
print("FLAG recupere :")
print(flag_data.val())
```

## Explication

### Initialisation Firebase

Le dictionnaire `config` contient tous les paramètres nécessaires pour joindre le projet Firebase visé.

### Authentification

Le script utilise :

- l’email trouvé dans `strings.xml` ;
- le mot de passe obtenu par exécution forcée de `Password()`.

### Accès à la base

Une fois authentifié, le script appelle :

```python
db.get(user['idToken'])
```

Ce qui permet de lire le contenu de la base avec un token valide.

---

# 10. Résultat final

L’exécution du script Python retourne bien le flag.

![Figure 8 — Exécution finale et récupération du flag](fire/8.png)

## Flag obtenu

```text
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
```

---

# 11. Synthèse méthodologique

## Cheminement de résolution

La résolution du challenge s’est faite en plusieurs étapes logiques :

### 1. Analyse statique de l’APK

L’APK a été ouvert dans JADX afin d’identifier :

- la classe principale ;
- la méthode sensible `Password()` ;
- les ressources embarquées dans `strings.xml`.

### 2. Identification de la logique de validation

La méthode `Password()` génère le secret nécessaire à l’authentification Firebase. Elle combine plusieurs chaînes statiques et une partie calculée par une fonction native.

### 3. Constat important

Cette méthode n’est pas appelée dans le fonctionnement normal de l’application. Il est donc impossible d’obtenir le mot de passe simplement en interagissant avec l’interface.

### 4. Forçage d’exécution avec Frida

Un script Frida a été injecté pour retrouver une instance active de `MainActivity` et appeler manuellement `Password()`.

### 5. Authentification Firebase

Le mot de passe généré a été utilisé avec l’email statique extrait des ressources pour s’authentifier à Firebase.

### 6. Lecture de la base

Après authentification, la lecture de la base Realtime Database a permis d’obtenir directement le flag.

---

# 12. Points importants à retenir

Ce challenge illustre plusieurs notions essentielles en sécurité mobile :

- une logique critique peut être présente dans une application sans être exposée par l’interface ;
- les ressources Android peuvent contenir des secrets ou des éléments de configuration sensibles ;
- l’analyse statique permet de comprendre la structure du code, mais pas toujours de reconstruire facilement une logique native ;
- Frida est particulièrement puissant pour forcer l’exécution de fonctions cachées ;
- une mauvaise protection des secrets Firebase peut conduire à la compromission complète des données.

---

# 13. Conclusion

Le challenge **FireStorm** repose sur une idée simple mais efficace : cacher le vrai secret dans une méthode non utilisée par l’application, tout en stockant dans les ressources suffisamment d’informations pour permettre à un analyste attentif de remonter toute la chaîne.

La résolution combine intelligemment :

- reverse engineering Android ;
- lecture des ressources ;
- instrumentation dynamique ;
- interaction avec Firebase.

La méthode `Password()` constitue le cœur du challenge. Son analyse montre comment le mot de passe est construit, tandis que Frida permet de l’exécuter dans le bon contexte pour éviter de reproduire manuellement la logique native. Enfin, l’accès à Firebase permet de finaliser l’exploitation et de récupérer le flag.

---

# Flag final

```text
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
```

---

# Annexes

## Commandes utilisées

### Installation de l’APK

```bash
adb install FireStorm.apk
```

### Vérification de Frida

```bash
frida-ps -U
```

### Lancement du script Frida

```bash
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js --no-pause
```

### Exécution du script Python

```bash
python get_flag.py
```

---


