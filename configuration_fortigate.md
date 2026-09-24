# Guide de Configuration FortiGate & VMware

Ce document regroupe les configurations de base, les règles d'accès, la mise en place d'un tunnel VPN IPsec (Site à Siège) et la résolution de problèmes réseau sur VMware.

---

## 1. Configuration des Interfaces FortiGate (CLI)

### 1.1. Configuration de l'interface en DHCP (`port1`)

```haproxy
config system interface
    edit port1
        set mode dhcp
        set defaultgw enable
        set dns-server-override enable
        set allowaccess ping https ssh http
    next
end
```

### 1.2. Configuration de l'interface en IP Statique (`port1`)

```haproxy
config system interface
    edit port1
        set mode static
        set ip 172.16.14.10 255.255.255.0
        set allowaccess ping https ssh http
    next
end
```

### 1.3. Commandes Utiles de Base

* **Afficher l'adresse IP des interfaces physiques :**
  ```bash
  get system interface physical
  ```
* **Redémarrer la machine proprement** *(préférable après chaque modification)* :
  ```bash
  execute reboot
  ```
* **Vérifier la connectivité avec l'hôte :**
  ```bash
  execute ping 192.xx.xx.xx
  ```
* **Forcer la sauvegarde de la configuration sur le Flash :**
  ```bash
  execute backup config flash
  ```

---

## 2. Configuration Hôte VMware (Obtenir les IP DHCP via VMnet1 / VMnet8)

Problème de permissions d'accès aux cartes réseau virtuelles sous Linux.

### Option A : Règles udev (Temporaire)

1. Créez ou modifiez le fichier de règles :
   ```bash
   sudo nano /etc/udev/rules.d/99-vmware-promisc.rules
   ```
2. Ajoutez les lignes suivantes :
   ```text
   KERNEL=="vmnet1", MODE="0666"
   KERNEL=="vmnet8", MODE="0666"
   ```
3. Appliquez immédiatement les règles sans redémarrer :
   ```bash
   sudo udevadm trigger
   ```

### Option B : Modification du script de démarrage VMware (Recommandé / Plus robuste)

1. Ouvrez le script réseau de VMware :
   ```bash
   sudo nano /etc/init.d/vmware
   ```
2. Recherchez la fonction `vmwareStartVmnet` avec `Ctrl + W`.
3. Repérez la section de démarrage :
   ```bash
   vmwareStartVmnet() {
       vmwareLoadModule "$vnet"
       ...
   ```
4. À la fin de la fonction `vmwareStartVmnet()` (juste avant le crochet d'obligation `}`), ajoutez :
   ```bash
   chmod a+rw /dev/vmnet1 /dev/vmnet8
   ```
5. Enregistrez et quittez (`Ctrl + O`, `Entrée`, `Ctrl + X`).

---

## 3. Tunnel VPN IPsec Site à Siège (Phase 1 & Phase 2)

### Justification des paramètres
* **Chiffrement :** `AES-256` (Solide, supporté matériellement par l'accélération ASIC/offloading).
* **Hachage / Authentification :** `SHA-256` (Sécurisé, évite les faiblesses de MD5/SHA-1).
* **Groupe DH (Diffie-Hellman) :** `Groupe 14` (2048-bit) ou `Groupe 19` (ECP256) pour garantir le *Perfect Forward Secrecy* (PFS).
* **Clé Partagée (PSK) :** Chaîne complexe (ex: `YaoKone@2026`).

---

### 3.1. Configuration du FortiGate SIÈGE

#### Phase 1 & Phase 2 VPN
```haproxy
config vpn ipsec phase1-interface
    edit "VPN-AGENCE"
        set interface "port1"              # Interface WAN du Siège
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 192.168.13.137          # IP publique de l'Agence
        set psksecret YaoKone@2026
    next
end

config vpn ipsec phase2-interface
    edit "VPN-AGENCE-P2"
        set phase1name "VPN-AGENCE"
        set proposal des-sha256
        set src-subnet 192.168.10.0/24     # LAN Siège
        set dst-subnet 192.168.20.0/24     # LAN Agence
    next
end
```

#### Route Statique
```haproxy
config router static
    edit 0
        set dst 192.168.20.0/24
        set device "VPN-AGENCE"
    next
end
```

#### Objets d'Adresses & Politiques de Pare-feu (Exemption de NAT)
```haproxy
# 1. Objet LAN Siège
config firewall address
    edit "LAN-SIEGE"
        set subnet 192.168.10.0 255.255.255.0
    next
end

# 2. Objet LAN Agence
config firewall address
    edit "LAN-AGENCE"
        set subnet 192.168.20.0 255.255.255.0
    next
end

# 3. Politiques de Pare-feu (sans NAT)
config firewall policy
    edit 0
        set name "SIEGE-vers-AGENCE"
        set srcintf "port2"                # Interface interne du LAN
        set dstintf "VPN-AGENCE"
        set srcaddr "LAN-SIEGE"
        set dstaddr "LAN-AGENCE"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 0
        set name "AGENCE-vers-SIEGE"
        set srcintf "VPN-AGENCE"
        set dstintf "port2"
        set srcaddr "LAN-AGENCE"
        set dstaddr "LAN-SIEGE"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

---

### 3.2. Configuration du FortiGate AGENCE (Miroir)

```haproxy
config vpn ipsec phase1-interface
    edit "VPN-SIEGE"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 192.168.13.129
        set psksecret YaoKone@2026
    next
end

config vpn ipsec phase2-interface
    edit "VPN-SIEGE-P2"
        set phase1name "VPN-SIEGE"
        set proposal des-sha256
        set src-subnet 192.168.20.0/24
        set dst-subnet 192.168.10.0/24
    next
end

config router static
    edit 0
        set dst 192.168.10.0/24
        set device "VPN-SIEGE"
    next
end
```
### 3.3. Configuration du FortiGate AGENCE (Miroir)
#### *Agence : LAN → VPN*
```haproxy
config firewall address
    edit "LAN-SIEGE"
        set subnet 192.168.10.0 255.255.255.0
    next
    edit "LAN-AGENCE"
        set subnet 192.168.20.0 255.255.255.0
    next
end
```
#### *Puis :*
```haproxy
config firewall policy
    edit 0
        set name "AGENCE-vers-SIEGE"
        set srcintf "port2"
        set dstintf "VPN-SIEGE"
        set srcaddr "LAN-AGENCE"
        set dstaddr "LAN-SIEGE"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```
#### *Et la policy retour :*
```haproxy
config firewall policy
    edit 0
        set name "SIEGE-vers-AGENCE"
        set srcintf "VPN-SIEGE"
        set dstintf "port2"
        set srcaddr "LAN-SIEGE"
        set dstaddr "LAN-AGENCE"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```
#### *Après verifier:*
```haproxy
show firewall policy
```


---

## 4. Vérifications et Diagnostics CLI

### Vérification des configurations :
```bash
show vpn ipsec phase1-interface
show vpn ipsec phase2-interface
show firewall address
show firewall policy
```

### Diagnostics du VPN :
```bash
get vpn ipsec tunnel summary
diagnose vpn ike gateway list
diagnose vpn tunnel list
```

### Diagnostic du Pare-feu (Règles & Traffic) :
* **Afficher l'intégralité des règles et configurations :**
  ```bash
  show firewall policy
  ```
* **Afficher uniquement la liste ordonnée des règles (par ID) :**
  ```bash
  get firewall policy
  ```
* **Voir l'activité et le nombre de correspondances (*Hit Count*) :**
  ```bash
  diagnose firewall iprope policy-list
  ```

---

## 5. Configuration Accès Internet du LAN (Interface Graphique - GUI)

### Étape 1 : Créer la règle de pare-feu (*Firewall Policy*)
1. Allez dans **Policy & Objects > Firewall Policy** (ou *IPv4 Policy*).
2. Cliquez sur **Create New**.
3. Renseignez les paramètres suivants :
   * **Name :** `LAN_to_Internet`
   * **Incoming Interface :** `LAN (port2)`
   * **Outgoing Interface :** `WAN (port1)`
   * **Source :** `all` (ou le sous-réseau spécifique LAN)
   * **Destination :** `all`
   * **Schedule :** `always`
   * **Service :** `ALL` (ou restreindre à HTTP, HTTPS, DNS)
   * **Action :** `ACCEPT`

### Étape 2 : Activer le NAT (Obligatoire)
1. Dans la même page de règle, défilez jusqu'à la section **Firewall / Network Options**.
2. Activez l'option **NAT** (bouton vert).
3. Conservez l'option par défaut : **Use Destination Interface IP**.

### Étape 3 : Vérifier la route statique par défaut (*Static Route*)
1. Allez dans **Network > Static Routes**.
2. Vérifiez la présence de la route par défaut :
   * **Destination :** `0.0.0.0/0`
   * **Gateway IP :** Adresse IP de la passerelle du FAI.
   * **Interface :** `WAN (port1)`