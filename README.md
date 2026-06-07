# LAB16 – Bypass du SSL Pinning avec Frida et Objection

## Objectif

L'objectif de ce laboratoire est de démontrer comment contourner le mécanisme de SSL Pinning d'une application Android à l'aide de Frida et Objection afin d'intercepter et d'analyser les communications HTTPS via Burp Suite.

---

## Environnement de test

| Composant            | Version                                     |
| -------------------- | ------------------------------------------- |
| Windows              | 10                                          |
| Android Emulator     | Android 11 x86_64                           |
| Burp Suite Community | 2026.4.3                                    |
| Frida                | 17.9.1                                      |
| Objection            | 1.12.4                                      |
| Application cible    | InsecureBankv2 (com.android.insecurebankv2) |

---

## 1. Vérification des prérequis

Validation de la présence des outils nécessaires :

```powershell
python --version
pip --version
adb version
```

Résultats obtenus :

* Python 3.10.11
* pip 25.1.1
* Android Debug Bridge 1.0.41

L'environnement est correctement configuré pour poursuivre les manipulations.

---

## 2. Installation de Frida et Objection

Installation des outils :

```powershell
pip install --upgrade frida frida-tools objection
```

Vérification des versions installées :

```powershell
frida --version
python -c "import frida; print(frida.__version__)"
objection --help
```

Versions validées :

* Frida 17.9.1
* Objection 1.12.4

---

## 3. Préparation de l'appareil Android

### 3.1 Vérification de l'architecture

```powershell
adb shell getprop ro.product.cpu.abi
```

Résultat :

```text
x86_64
```

### 3.2 Déploiement de frida-server

Après téléchargement de la version adaptée depuis le dépôt officiel Frida :

```powershell
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

### 3.3 Vérification de la connexion Frida

```powershell
frida-ps -Uai
```

L'application cible apparaît correctement dans la liste des processus accessibles.

---

## 4. Configuration du proxy Burp Suite

### 4.1 Configuration du listener

Par défaut, Burp Suite écoute uniquement sur l'interface locale (127.0.0.1). Afin de permettre à l'émulateur Android de communiquer avec le proxy, le listener a été configuré sur toutes les interfaces réseau.

Configuration :

```text
Proxy → Proxy Settings → Proxy Listeners → Edit
```

Modification :

```text
Bind to Address : All Interfaces (0.0.0.0)
Port : 8080
```

### 4.2 Identification de l'adresse IP du poste

```powershell
ipconfig
```

Adresse IP retenue :

```text
192.168.1.107
```

### 4.3 Configuration du proxy Android

Paramètres appliqués :

```text
Proxy Host : 192.168.1.107
Proxy Port : 8080
```

### 4.4 Installation du certificat Burp CA

Export du certificat depuis Burp Suite :

```text
Proxy → Proxy Settings → Import / Export CA Certificate
```

Format sélectionné :

```text
DER Format
```

Déploiement sur l'appareil :

```powershell
adb push cacert.der /sdcard/Download/cacert.crt
```

L'extension `.crt` est utilisée afin d'assurer la compatibilité avec le mécanisme d'installation des certificats Android.

Installation :

```text
Paramètres → Sécurité → Chiffrement et identifiants
→ Installer un certificat → Certificat CA
```

---

## 5. Désactivation du SSL Pinning

Le contournement du SSL Pinning est effectué via Objection lors du lancement de l'application.

Commande utilisée :

```powershell
objection -g com.android.insecurebankv2 explore --startup-command "android sslpinning disable"
```

Messages observés :

```text
Custom TrustManager ready
Overriding SSLContext.init()
Overriding TrustManagerImpl.verifyChain()
Overriding TrustManagerImpl.checkTrustedRecursive()
```

Ces messages confirment que les mécanismes de validation des certificats sont interceptés et remplacés par Frida.

L'application accepte désormais le certificat présenté par Burp Suite sans générer d'erreur de sécurité.

---

## 6. Validation de l'interception du trafic

### 6.1 Configuration du backend

Depuis l'application :

```text
Server IP : 192.168.1.107
Server Port : 8888
```

### 6.2 Démarrage du serveur

```powershell
python app.py
```

### 6.3 Génération du trafic

Une tentative d'authentification est réalisée depuis l'application.

Burp Suite intercepte alors la requête suivante :

```http
POST /login HTTP/1.1
Host: 192.168.1.107:8888
Content-Type: application/x-www-form-urlencoded

username=aya&password=aya
```

Les données transitent désormais en clair dans Burp Suite grâce au contournement du SSL Pinning.

---

## Résultats obtenus

| Vérification                        | Statut |
| ----------------------------------- | ------ |
| Python, pip et ADB opérationnels    | ✓      |
| Installation de Frida et Objection  | ✓      |
| frida-server exécuté sur l'appareil | ✓      |
| Configuration du proxy Burp         | ✓      |
| Installation du certificat CA       | ✓      |
| Désactivation du SSL Pinning        | ✓      |
| Interception du trafic HTTPS        | ✓      |

---

## Conclusion

Ce laboratoire a permis de démontrer qu'un mécanisme de SSL Pinning implémenté côté client peut être neutralisé dynamiquement à l'aide de Frida et Objection. Une fois le certificat Burp Suite installé et les contrôles de confiance détournés, l'ensemble des communications HTTPS de l'application devient observable dans Burp Suite, permettant l'analyse détaillée des requêtes et des réponses échangées avec le serveur.
