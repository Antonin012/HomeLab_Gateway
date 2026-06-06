# Guide d'Installation et d'Utilisation

Ce document détaille les étapes techniques, les choix de sécurité et les procédures de validation de la **Homelab Gateway**.

## 1. Stratégie de Déploiement

Contrairement à une installation manuelle classique, ce projet utilise **Ansible**. Cela permet de :
- Automatiser la configuration système (Hardening).
- Déployer les stacks Docker de manière cohérente.
- Gérer les secrets (mots de passe, clés) via **Ansible Vault**.

## 2. Configuration du Firewall (UFW)

La sécurité repose sur une politique de **Default Deny**. 
- Seul le port **UDP 51820** (WireGuard) est ouvert sur l'interface publique.
- Les ports **80** (HTTP) et **443** (HTTPS) ne sont accessibles **qu'à travers le VPN** (interface `wg0`). 
- Cela rend les services totalement invisibles pour toute personne scanant votre IP publique.

## 3. Le Reverse Proxy : Caddy

Caddy a été choisi pour sa simplicité. Voici un aperçu de la configuration utilisée pour router les services du Lab :

```caddyfile
# Exemple de routage interne
grafana.vpn {
    tls internal
    reverse_proxy 10.0.0.30:3000
}
```
*Note : Caddy génère ses propres certificats internes pour sécuriser les flux entre le client VPN et la passerelle.*

## 4. Guide de Mise en Œuvre

### 4.1 Préparation du Serveur
Assurez-vous que Debian est à jour :
```bash
sudo apt update && sudo apt upgrade -y
```

### 4.2 Gestion des Secrets avec Ansible Vault
Avant de déployer, vous devez renseigner vos propres valeurs dans `deployments/ansible/group_vars/all/secrets.yml` :
- `vault_pihole_password` : Mot de passe admin Pi-hole.
- `vault_grafana_password` : Mot de passe admin Grafana.
- `vault_wireguard_server_url` : Votre IP publique ou nom de domaine.

Une fois édité, chiffrez le fichier :
```bash
ansible-vault encrypt deployments/ansible/group_vars/all/secrets.yml
```

### 4.3 Exécution du Playbook
Le playbook s'occupe de tout : installation de Docker, création du réseau `vpn_net`, configuration d'UFW et lancement des conteneurs.
```bash
ansible-playbook -i deployments/ansible/inventory.ini deployments/ansible/playbook.yml --ask-vault-pass
```

## 5. Tests et Validation

### 5.1 Audit Externe (Nmap)
Depuis une connexion extérieure (ex: 4G), vérifiez que le serveur est bien protégé :
```bash
nmap -Pn -sS -p 1-1000 [VOTRE_IP_PUBLIQUE]
```
*Résultat attendu : Tous les ports sont "filtered".*

### 5.2 Test de Connectivité VPN
Une fois connecté à WireGuard :
1. Tentez un ping sur l'IP du Pi-hole : `ping 10.0.0.53`.
2. Accédez à l'interface de Caddy ou Pi-hole via votre navigateur.

### 5.3 DNS Leak Test
Utilisez [dnsleaktest.com](https://www.dnsleaktest.com) pour confirmer que vos requêtes DNS passent bien par le Pi-hole et ne fuitent pas vers votre FAI.

## 6. Maintenance Quotidienne

- **Ajouter un client VPN** : `docker exec -it wireguard-gateway /app/show-peer 1`
- **Mettre à jour les services** : Modifiez le playbook ou relancez-le, Ansible gérera la mise à jour des images Docker si nécessaire.
