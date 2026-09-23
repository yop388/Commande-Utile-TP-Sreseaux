# Configuration Réseau : VPCS & Routeur Cisco vVIOS

Ce guide détaille les étapes pour configurer une adresse IP fixe, la passerelle, les serveurs DNS et sauvegarder la configuration sur un VPCS (Virtual PC) et un routeur Cisco vVIOS (IOS) dans EVE-NG.

---

## 1. Configuration d'un VPC (Virtual PC / VPCS)

### 1.1. Attribuer l'adresse IP et la passerelle
```bash
ip <adresse_ip>/<masque> <passerelle>
```
*Exemple :*
```bash
ip 192.168.1.10/24 192.168.1.1
```

### 1.2. Configurer le serveur DNS
```bash
ip dns 8.8.8.8
```

### 1.3. Enregistrer la configuration de façon permanente
```bash
save
```

### 1.4. Vérifier la configuration
```bash
show ip
```

---

## 2. Configuration d'un Routeur Cisco vVIOS (IOS)

### 2.1. Attribuer l'adresse IP à une interface

1. Passer en mode d'exécution privilégié et de configuration globale :
   ```text
   enable
   configure terminal
   ```

2. Configurer l'interface réseau et l'activer :
   ```text
   interface GigabitEthernet0/0
    ip address 192.168.1.1 255.255.255.0
    no shutdown
    exit
   ```

### 2.2. Configurer la résolution DNS

1. Activer la recherche DNS et définir le serveur de noms :
   ```text
   ip domain-lookup
   ip name-server 8.8.8.8
   exit
   ```

### 2.3. Sauvegarder la configuration permanente

Pour rendre la configuration persistante au redémarrage :
```text
write memory
```
*(Ou la commande équivalente : `copy running-config startup-config`)*

### 2.4. Commandes de vérification

* **Vérifier la configuration DNS :**
  ```text
  show system dns
  ```

* **Tester la connectivité et la résolution de noms :**
  ```text
  ping google.com
  ```