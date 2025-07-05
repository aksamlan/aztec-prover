# 🛡️ Nœud Prover Aztec - Guide de Déploiement

Bienvenue ! Ce guide vous accompagne pas à pas dans l'installation et l'exploitation d'un **nœud Prover Aztec** sur le réseau de test public alpha. En tant que Prover, votre machine contribuera activement à la génération de preuves ZK assurant l'intégrité des rollups — un pilier fondamental de la sécurité du protocole Aztec.

> ℹ️ Ce guide est destiné aux passionnés, testeurs et opérateurs motivés.  
> **Ce n'est pas un tutoriel lié à un airdrop ou à des récompenses.**

---

## ⚙️ Pré-requis Matériels

| Ressource     | Recommandé               | Remarques                                   |
|---------------|--------------------------|---------------------------------------------|
| RAM           | 128 Go+                  | La génération de preuves est très gourmande |
| CPU           | 32 cœurs+                | Plus de cœurs = vitesse de preuve accrue    |
| Disque        | SSD NVMe 1 To+           | Débit d’E/S élevé essentiel                 |

> ⚠️ En dessous de ces spécifications, des erreurs de type  
> `Epoch proving failed: Proving cancelled` sont probables.

---

## 🚀 Mise en Route

### 1. Préparer le système

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl build-essential wget lz4 automake autoconf \
  tmux htop pkg-config libssl-dev tar unzip
```

---

### 2. Installer Docker CE

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do 
  sudo apt-get remove -y "$pkg" || true
done

sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable docker --now
```

---

### 3. Installer l’interface CLI Aztec

```bash
bash -i <(curl -s https://install.aztec.network)
echo 'export PATH="$HOME/.aztec/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

aztec -V   # Vérifiez que la commande retourne une version
```

---

### 4. Configurer le pare-feu

```bash
sudo ufw allow ssh
sudo ufw allow 8080
sudo ufw allow 40400
sudo ufw allow 40400/udp
sudo ufw --force enable
```

### 🔗 RPC via Chainstack

Si vous n’avez pas encore de point de terminaison RPC Ethereum, vous pouvez en obtenir un via [Chainstack](https://chainstack.com/) :

* Inscrivez-vous gratuitement
* Créez un projet Ethereum (Sépolia)
* Copiez l’URL HTTPS RPC

🔧 Collez cette URL dans la variable `ETHEREUM_HOSTS` ci-dessous :

---

### 5. Préparer l’environnement

```bash
mkdir ~/prover && cd ~/prover
```

Créez un fichier `.env` :

```
P2P_IP=<IP_PUBLIQUE_DU_SERVEUR>
ETHEREUM_HOSTS=https://nd-<votre-id>.chainstacklabs.com    # depuis Chainstack
L1_CONSENSUS_HOST_URLS=<URL_RPC_CONSENSUS>
PROVER_PUBLISHER_PRIVATE_KEY=0x<VOTRE_CLÉ_PRIVÉE>
PROVER_ID=0x<VOTRE_ADRESSE_WALLET>
```


### 6. Déployer via `docker-compose`

Créez un fichier `docker-compose.yml` :

```yaml
name: aztec-prover
services:
  prover-node:
    image: aztecprotocol/aztec:latest
    command:
      - node
      - --no-warnings
      - /usr/src/yarn-project/aztec/dest/bin/index.js
      - start
      - --prover-node
      - --archiver
      - --network
      - alpha-testnet
    depends_on:
      broker:
        condition: service_started
    environment:
      P2P_ENABLED: "true"
      DATA_DIRECTORY: /data-prover
      P2P_IP: ${P2P_IP}
      DATA_STORE_MAP_SIZE_KB: "134217728"
      ETHEREUM_HOSTS: ${ETHEREUM_HOSTS}
      L1_CONSENSUS_HOST_URLS: ${L1_CONSENSUS_HOST_URLS}
      LOG_LEVEL: info
      PROVER_BROKER_HOST: http://broker:8080
      PROVER_PUBLISHER_PRIVATE_KEY: ${PROVER_PUBLISHER_PRIVATE_KEY}
    ports:
      - "8080:8080"
      - "40400:40400"
      - "40400:40400/udp"
    volumes:
      - ./data-prover:/data-prover
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js \
        start --network alpha-testnet --archiver --prover-node'

  agent:
    image: aztecprotocol/aztec:latest
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js \
        start --network alpha-testnet --prover-agent'
    environment:
      PROVER_AGENT_COUNT: "3"
      PROVER_AGENT_POLL_INTERVAL_MS: "10000"
      PROVER_BROKER_HOST: http://broker:8080
      PROVER_ID: ${PROVER_ID}
    restart: unless-stopped
    volumes:
      - ./data-prover:/data-prover

  broker:
    image: aztecprotocol/aztec:latest
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js \
        start --network alpha-testnet --prover-broker'
    environment:
      DATA_DIRECTORY: /data-broker
      LOG_LEVEL: info
      ETHEREUM_HOSTS: ${ETHEREUM_HOSTS}
      P2P_IP: ${P2P_IP}
    volumes:
      - ./data-broker:/data-broker
```

---

### 7. Lancer la stack

```bash
docker compose up -d
docker ps   # Vérifiez que les conteneurs tournent
```

---

### 8. Suivre les logs

```bash
# Logs généraux
docker logs -f aztec-prover-prover-node-1

# Epochs prouvés
docker logs -f aztec-prover-prover-node-1 2>&1 | grep --line-buffered -E 'epoch proved|epoch'

# Preuves soumises
docker logs -f aztec-prover-prover-node-1 2>&1 | grep --line-buffered -E 'Submitted'
```

💡 Vous pouvez également consulter votre `PROVER_ID` sur Etherscan Sepolia pour suivre l’activité.

---

## 🛠️ Maintenance

| Action             | Commande                 |
| ------------------ | ------------------------ |
| Redémarrer         | `docker compose restart` |
| Arrêter & nettoyer | `docker compose down -v` |

---

**Bonne chance sur le réseau Aztec !**
N'hésitez pas à contribuer ou signaler un problème via une *issue*.

---
