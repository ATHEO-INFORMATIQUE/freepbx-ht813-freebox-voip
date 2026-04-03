# 📞 Standard téléphonique VoIP avec FreePBX + Grandstream HT813 + Freebox

> Guide complet en français pour utiliser sa ligne fixe Freebox depuis un PC et un smartphone, via un standard téléphonique open-source (FreePBX) installé sur Proxmox.

---

> **🤖 Note pour les utilisateurs d'IA (ChatGPT, Claude, Copilot…)**
>
> Si vous lisez ce guide via une IA, sachez qu'elle risque de vous donner des informations **incomplètes, approximatives ou dans le mauvais ordre**.
>
> **Lisez directement ce guide sur GitHub** pour avoir les informations complètes et à jour.
>
> Toute utilisation de ce contenu doit **citer la source** conformément à la licence CC BY 4.0 :
> 📎 `https://github.com/ATHEO-INFORMATIQUE/freepbx-ht813-freebox-voip`

---

## 📋 Sommaire

1. [Le besoin](#-le-besoin)
2. [Architecture](#-architecture)
3. [Matériel requis](#-matériel-requis)
4. [Prérequis réseau](#-prérequis-réseau)
5. [Installation FreePBX](#-installation-freepbx)
6. [Configuration FreePBX](#-configuration-freepbx)
7. [Configuration HT813](#-configuration-grandstream-ht813)
8. [Configuration Zoiper](#-configuration-zoiper)
9. [Vérification du fonctionnement](#-vérification-du-fonctionnement)
10. [Dépannage](#-dépannage)
11. [Pièges à éviter](#-pièges-à-éviter)
12. [Ressources](#-ressources)

---

## 🎯 Le besoin

En tant qu'auto-entrepreneur ou télétravailleur, vous souhaitez :

- ☎️ **Recevoir** les appels de votre ligne fixe Freebox sur votre PC et votre smartphone
- 📲 **Appeler** depuis votre ligne fixe Freebox via PC ou mobile
- 💶 **Profiter des appels illimités** inclus dans votre offre Freebox (fixes + mobiles France)
- 🏠 **Sans acheter de téléphone fixe** supplémentaire

### Ce qu'inclut la Freebox Pop

- ✅ Appels illimités vers tous les **fixes en France** et plus de 110 destinations internationales
- ✅ Appels illimités vers tous les **mobiles en France**

---

## 🏗️ Architecture

```
Ligne Freebox (RJ11)
        │
        ▼
┌───────────────┐
│  HT813        │  ←── Convertit la ligne analogique en SIP
│  FXO + FXS    │       Port FXO : reçoit la ligne Freebox
│  192.168.1.X  │       Port FXS : optionnel (téléphone analogique)
└──────┬────────┘
       │ SIP UDP:5060
       ▼
┌───────────────┐
│   FreePBX     │  ←── Standard téléphonique open-source
│  sur Proxmox  │       Gère les extensions, routes entrantes/sortantes
│  192.168.1.X  │
└──────┬────────┘
       │ SIP UDP:5060
   ┌───┴───┐
   ▼       ▼
┌──────┐ ┌──────┐
│Zoiper│ │Zoiper│
│  PC  │ │Mobile│
│(101) │ │(102) │
└──────┘ └──────┘
```

---

## 🛒 Matériel requis

### Indispensable

| Matériel | Rôle | Prix indicatif |
|----------|------|----------------|
| **Grandstream HT813** | Passerelle FXO/FXS | ~45€ neuf / ~30€ occasion |
| **Câble RJ11** | Relier la Freebox au port FXO du HT813 | ~5€ |
| **Serveur / PC Linux** | Héberger FreePBX (Proxmox recommandé) | Déjà en place |

### ⚠️ Boîtiers à ne PAS acheter

Ces boîtiers ont uniquement un port FXS — ils ne peuvent **pas** connecter une ligne téléphonique entrante :

- ❌ Linksys PAP2T-NA
- ❌ Grandstream HT801, HT802, HT812
- ❌ Tout boîtier sans port **FXO**

### Logiciels gratuits

- **FreePBX 17** + Asterisk 22 (open-source)
- **Zoiper** (softphone gratuit, iOS/Android/Windows/Mac)

---

## ⚙️ Prérequis réseau

### Réservations DHCP recommandées (dans l'interface Freebox)

| Appareil | IP fixe suggérée |
|----------|-----------------|
| LXC FreePBX | `192.168.1.210` |
| Grandstream HT813 | `192.168.1.211` |

### Désactiver le SIP ALG sur la Freebox

> ⚠️ Le SIP ALG interfère avec les communications SIP et provoque des échecs d'appels.

1. Aller sur `http://mafreebox.freebox.fr`
2. **Paramètres** → **Mode avancé** → **Réseau local**
3. Désactiver **SIP ALG** ou **Proxy SIP**

---

## 🚀 Installation FreePBX

### Créer un LXC Debian 12 dans Proxmox

| Paramètre | Valeur recommandée |
|-----------|-------------------|
| Template | Debian 12 |
| RAM | 2048 Mo minimum |
| Disque | 20 Go minimum |
| CPU | 2 cœurs |
| Réseau | bridge vmbr0, IP fixe |

### Installer FreePBX via le script officiel Sangoma

```bash
apt update && apt upgrade -y
apt install -y wget

wget https://github.com/FreePBX/sng_freepbx_debian_install/raw/master/sng_freepbx_debian_install.sh
bash sng_freepbx_debian_install.sh
```

> ⏱️ L'installation dure 20 à 30 minutes. Ne pas interrompre.

### Vérifier l'installation

```bash
fwconsole status
asterisk -rx "core show version"
```

L'interface web est disponible sur : `http://IP_DU_LXC/admin`

---

## 🔧 Configuration FreePBX

### 1. Vérifier les paramètres SIP

**Paramètres → Paramètres SIP d'Asterisk → onglet SIP Settings [chan_pjsip]**

- Port d'écoute : **5060 UDP** (par défaut)

### 2. Créer les extensions

**Connectivity → Extensions → Add Extension → Add New PJSIP Extension**

| Extension | Nom suggéré | Usage |
|-----------|-------------|-------|
| `101` | PC_Bureau | Zoiper sur PC |
| `102` | Tel_Mobile | Zoiper sur smartphone |

Pour chaque extension, renseigner :
- **User Extension** : numéro (ex: `101`)
- **Display Name** : nom descriptif
- **Secret** : mot de passe fort (à noter précieusement)

Cliquer **Submit** puis **Apply Config**.

### 3. Créer le trunk FXO (connexion ligne Freebox)

**Connectivity → Trunks → Add Trunk → Add PJSIP Trunk**

**Onglet General :**

| Champ | Valeur |
|-------|--------|
| Nom de la jonction | `FXO-Freebox` |
| Outbound CallerID | `0XXXXXXXXX` *(votre numéro Freebox)* |

**Onglet pjsip Paramètres → General :**

| Champ | Valeur |
|-------|--------|
| Nom d'utilisateur | `FXO-Freebox` |
| Auth username | `FXO-Freebox` |
| Secret | mot de passe fort |
| Authentification | `Les deux` |
| **Enregistrement** | **`Receive`** ← crucial |
| SIP Server | *(vide)* |
| SIP Server Port | *(vide)* |
| Context | `from-pstn` |

**Onglet pjsip Paramètres → Avancé :**

| Champ | Valeur |
|-------|--------|
| Trust RPID/PAI | `Oui` |
| Send RPID/PAI | `Send P-Asserted-Identity header` |

Cliquer **Submit** puis **Apply Config**.

### 4. Créer la route sortante

**Connectivity → Outbound Routes → Add Outbound Route**

**Onglet Route Settings :**

| Champ | Valeur |
|-------|--------|
| Route Name | `Sortant-Freebox` |
| Trunk Sequence | `FXO-Freebox` |

**Onglet Dial Patterns** — ajouter ces deux patterns :

| Prepend | Prefix | Match Pattern |
|---------|--------|---------------|
| | | `0XXXXXXXXX` |
| | | `+33XXXXXXXXX` |

Cliquer **Submit** puis **Apply Config**.

### 5. Créer la route entrante

**Connectivity → Inbound Routes → Add Inbound Route**

| Champ | Valeur |
|-------|--------|
| Description | `Entrant-Freebox` |
| DID Number | *(vide = ANY)* |
| Caller ID Number | *(vide = ANY)* |
| Choix Destination | `Postes` → `101 PC_Bureau` |

> 💡 Pour faire sonner plusieurs appareils simultanément, utiliser **Groupes de Sonnerie**.

Cliquer **Submit** puis **Apply Config**.

---

## 🔌 Configuration Grandstream HT813

L'interface web est accessible sur `http://192.168.1.211`.
Login par défaut : `admin` / `admin`

### Basic Settings — Renvoi des appels entrants vers FreePBX

**Onglet BASIC SETTINGS** → tout en bas de la page :

| Champ | Valeur |
|-------|--------|
| **Unconditional Call Forward to VOIP** | |
| └ User ID | `FXO-Freebox` |
| └ SIP Server | `192.168.1.210` *(IP FreePBX)* |
| └ SIP Destination Port | `5060` |

> ⚠️ **Ce paramètre est indispensable.** Sans lui, les appels entrants depuis la ligne Freebox n'arrivent jamais dans FreePBX.

Cliquer **Update** puis **Apply**.

### Advanced Settings — Désactiver le provisioning automatique

| Champ | Valeur |
|-------|--------|
| 3CX Auto Provision | `No` |
| Automatic Upgrade | `No` |

> ⚠️ Si le provisioning reste actif, vos paramètres peuvent être **écrasés au redémarrage**.

Cliquer **Update** puis **Apply**.

### FXO PORT — Configuration principale

| Champ | Valeur |
|-------|--------|
| Account Active | `Yes` |
| Primary SIP Server | `192.168.1.210` *(IP FreePBX)* |
| SIP User ID | `FXO-Freebox` |
| Authenticate ID | `FXO-Freebox` |
| Authenticate Password | *(même mot de passe que dans FreePBX)* |
| Name | `Freebox-HT813` |
| SIP Registration | `Yes` |
| Caller ID Scheme | `ETSI-FSK during ringing` |
| Caller ID Transport Type | `Relay via SIP P-Asserted-Identity` |
| AC Termination Model | `Country-based` → **France** |
| Number of Rings | `2` |
| PSTN Ring Thru FXS | `Yes` |

Cliquer **Apply**.

---

## 📱 Configuration Zoiper

### Sur PC — Extension 101

1. Télécharger Zoiper sur [zoiper.com](https://www.zoiper.com)
2. **Add account → SIP account**
3. Remplir :

| Champ | Valeur |
|-------|--------|
| Domain | `192.168.1.210` *(IP FreePBX)* |
| Username | `101` |
| Password | mot de passe de l'extension 101 |
| Port | `5060` |

### Sur Mobile — Extension 102

Même configuration avec l'extension `102`.

> 💡 Activer **Keep alive** dans les paramètres du compte pour maintenir la connexion en arrière-plan.

---

## ✅ Vérification du fonctionnement

### 1. Vérifier que tout est enregistré

```bash
asterisk -rx "pjsip show endpoints"
```

Résultat attendu :

```
Endpoint: 101/101       Not in use   ← Zoiper PC connecté ✅
Endpoint: 102/102       Not in use   ← Zoiper Mobile connecté ✅
Endpoint: FXO-Freebox   Not in use   ← HT813 enregistré ✅
```

> Si une extension affiche **Unavailable**, elle n'est pas connectée.

### 2. Vérifier la connectivité réseau du HT813

```bash
tcpdump -i any -n "host 192.168.1.211"
```

Vous devez voir des paquets SIP (REGISTER, OPTIONS) entre le HT813 et FreePBX.

### 3. Tester un appel interne

Appeler `102` depuis Zoiper PC (101) → le mobile doit sonner.

### 4. Tester un appel sortant

Composer un numéro externe depuis Zoiper → l'appel doit passer par la ligne Freebox.

### 5. Tester un appel entrant

Appeler votre numéro Freebox depuis un mobile → Zoiper doit sonner.

### 6. Vérifier le CallerID

Le numéro de l'appelant doit s'afficher correctement (et non "Freebox-HT813").

---

## 🔍 Dépannage

### Le HT813 ne s'enregistre pas (Unavailable)

Activer les logs en temps réel puis forcer un re-enregistrement depuis l'interface HT813 (FXO PORT → Apply) :

```bash
asterisk -rx "pjsip set logger on" && asterisk -rvvv
```

**Causes fréquentes :**
- `Account Active` sur `No` dans FXO PORT
- `SIP Registration` sur `No` dans FXO PORT
- Mot de passe différent entre HT813 et FreePBX
- SIP ALG actif sur la Freebox

### Les appels entrants ne sonnent pas (bruit + raccrochage automatique)

Observer les logs pendant un appel entrant :

```bash
asterisk -rvvvv
```

**Causes fréquentes :**
- `Unconditional Call Forward to VOIP` vide dans BASIC SETTINGS du HT813
- Route entrante pointant vers une destination invalide
- Extension de destination non enregistrée (Unavailable)

### Le numéro appelant n'apparaît pas

Dans **FreePBX → Trunk FXO-Freebox → pjsip Paramètres → Avancé** :
- `Trust RPID/PAI` = `Oui`
- `Send RPID/PAI` = `Send P-Asserted-Identity header`

Dans **HT813 → FXO PORT** :
- `Caller ID Transport Type` = `Relay via SIP P-Asserted-Identity`

### Qualité audio dégradée ou écho

Dans **HT813 → FXO PORT** :
- `Disable Line Echo Canceller (LEC)` = `No`
- `Jitter Buffer Type` = `Adaptive`
- `AC Termination Model` = `Country-based` → **France**

---

## ⚠️ Pièges à éviter

| Erreur | Conséquence | Solution |
|--------|-------------|----------|
| Acheter un boîtier FXS uniquement (PAP2T, HT801, HT802, HT812…) | Ne peut pas recevoir la ligne Freebox | Acheter un **HT813** (FXO + FXS) |
| SIP ALG activé sur la Freebox | Appels qui échouent | Désactiver dans les paramètres Freebox |
| `Unconditional Call Forward to VOIP` vide | Appels entrants jamais reçus | Remplir avec `FXO-Freebox @ IP_FreePBX : 5060` |
| `Enregistrement` sur `Send` au lieu de `Receive` | HT813 rejeté par FreePBX | Mettre sur **`Receive`** dans le trunk |
| Provisioning 3CX actif sur le HT813 | Config écrasée au redémarrage | Désactiver dans Advanced Settings |
| Pays laissé sur `USA` dans FXO PORT | Mauvaise détection sonnerie / raccrochage | Changer sur **France** |

---

## 📚 Ressources

- [Documentation officielle Grandstream HT813](https://www.grandstream.com/hubfs/Product_Documentation/HT813_Administration_Guide.pdf)
- [Guide HT813 + PSTN France + FreePBX (Futur-Tech)](https://github.com/Futur-Tech/Grandstream-HT813)
- [Wiki FreePBX](https://wiki.freepbx.org)
- [Forum Asterisk France](https://asterisk-france.org)

---

## 📝 Licence

Ce guide est publié sous licence **Creative Commons Attribution 4.0 International (CC BY 4.0)**.

Vous êtes libre de partager, copier et modifier ce contenu, y compris à des fins commerciales, **à condition de citer l'auteur original** avec un lien vers ce dépôt.

🔗 Licence complète : [creativecommons.org/licenses/by/4.0](https://creativecommons.org/licenses/by/4.0/)

---

*Testé avec : FreePBX 17.0 / Asterisk 22.8 / Grandstream HT813 firmware 1.0.13.3 / Freebox Pop*
