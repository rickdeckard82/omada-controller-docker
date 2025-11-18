# 🧩 GUIA DEFINITIVO — Omada Controller + MongoDB Externo Seguro (Ubuntu 22.04)

---

## 🧰 1. Instalar MongoDB 8.0 (repositório oficial)

```bash
curl -fsSL https://pgp.mongodb.com/server-8.0.asc |   sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt update
sudo apt install -y mongodb-org
```

Habilite e inicie o serviço:
```bash
sudo systemctl enable mongod
sudo systemctl start mongod
```

---

## 🔑 2. Gerar chave de replicação segura (`/etc/mongod.key`)

```bash
openssl rand -base64 756 | sudo tee /etc/mongod.key
sudo chown mongodb:mongodb /etc/mongod.key
sudo chmod 600 /etc/mongod.key
```

---

## ⚙️ 3. Configurar o MongoDB

Arquivo `/etc/mongod.conf`:

```yaml
storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

net:
  bindIp: 0.0.0.0
  port: 27017

replication:
  replSetName: rs0

#security:
#  keyFile: /etc/mongod.key
#  authorization: enabled

setParameter:
  enableLocalhostAuthBypass: false


# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: false
```

Reinicie:
```bash
sudo systemctl restart mongod
```

---

## 🧩 4. Inicializar o ReplicaSet

```bash
mongosh
```

Inicialize com o loopback:

```javascript
rs.initiate({
  _id: "rs0",
  members: [ { _id: 0, host: "127.0.0.1:27017" } ]
})
```

Depois altere para o IP real do servidor:

```javascript
cfg = rs.conf()
cfg.members[0].host = "186.233.16.6:27017"
rs.reconfig(cfg, { force: true })
```

Verifique:
```javascript
rs.status()
```

---

## 👤 5. Criar usuário administrador root

```bash
mongosh
```

```javascript
use admin
db.createUser({
  user: "admin",
  pwd: "Key(@n@)",
  roles: [ { role: "root", db: "admin" } ]
})
```

Reinicie:
```bash
sudo systemctl restart mongod
```

---

## 🔐 6. Testar login seguro

```bash
mongosh -u admin -p 'Key(@n@)' --authenticationDatabase admin
```

---

## 👥 7. Criar usuário do Omada

```javascript
use admin
db.createUser({
  user: "omada",
  pwd: "0m4d4",
  roles: [
    { role: "readWrite", db: "omada" },
    { role: "dbAdmin", db: "omada" },
    { role: "readWrite", db: "omada_data" },
    { role: "dbAdmin", db: "omada_data" },
    { role: "readWrite", db: "omada_stat" },
    { role: "dbAdmin", db: "omada_stat" }
  ]
})
```

Verifique:
```javascript
db.getUser("omada")
```

---

## ⚙️  8. Descomentar segurança

Arquivo `/etc/mongod.conf`:

```yaml

security:
  keyFile: /etc/mongod.key
  authorization: enabled
```

Reinicie:
```bash
sudo systemctl restart mongod
```

---

## 🔐 9. Testar login seguro

```bash
mongosh -u admin -p 'Key(@n@)' --authenticationDatabase admin
```

---

## 🧪 11. Testar a conexão do Omada ao Mongo

```bash
mongosh "mongodb://omada:0m4d4@186.233.16.6:27017/omada?replicaSet=rs0&authSource=admin"
```

---

## 📁 12. Estrutura de diretórios

```bash
mkdir -p /app/omada/{data,logs,autobackup}
chmod -R 777 /app/omada
```

---

## 🐳 13. Criar o container Omada Controller

Arquivo: `/app/omada/docker-compose.yml`

```yaml
services:
  omada-controller:
    image: mbentley/omada-controller:6.0
    container_name: omada-controller
    restart: unless-stopped
    network_mode: host     # <<< ESSENCIAL
    ulimits:
      nofile:
        soft: 4096
        hard: 8192
    environment:
      MONGO_EXTERNAL: "true"
      EAP_MONGOD_URI: "mongodb://omada:0m4d4@186.233.16.6:27017/omada?replicaSet=rs0&authSource=admin"
      AUTOBACKUP_PATH: /opt/tplink/EAPController/data/autobackup
      AUTOBACKUP_CRON: 0 3 * * *
      AUTOBACKUP_RETENTION: 14
      TZ: America/Sao_Paulo
    volumes:
      - /app/omada/data:/opt/tplink/EAPController/data
      - /app/omada/logs:/opt/tplink/EAPController/logs
      - /app/omada/autobackup:/opt/tplink/EAPController/data/autobackup

```

Subir o container:
```bash
cd /app/omada
docker compose up -d
```

Verificar:
```bash
docker logs -f omada-controller
```

Deve exibir:
```
INFO: skipping MongoDB data version check; using external MongoDB
INFO: Omada Controller startup detected
INFO: Autobackup scheduled at 03:00 every day
```

---

## 🌐 14. Acessar o painel Omada

```
https://<IP_DO_SERVIDOR>:8043
```

Finalize a configuração inicial no navegador.

---

## 💾 15. Backups

### Automático
- Caminho: `/app/omada/autobackup`
- Retenção: 14 dias
- Agendado: 03:00

### Manual (MongoDB)
```bash
mongodump --uri "mongodb://admin:Key(@n@)@127.0.0.1:27017/?authSource=admin&replicaSet=rs0"   --out /mnt/md2/backups/mongo-$(date +%Y%m%d)
```

Restaurar:
```bash
mongorestore --uri "mongodb://admin:Key(@n@)@127.0.0.1:27017/?authSource=admin&replicaSet=rs0"   --drop /mnt/md2/backups/mongo-20251031
```

---

## 🔁 16. Reinstalação futura

1. Reinstale o MongoDB (passos 1–6).  
2. Copie `/etc/mongod.key`.  
3. Recrie os usuários `admin` e `omada`.  
4. Suba o Omada (`docker-compose up -d`).  
5. Todos os dados e dispositivos serão restaurados automaticamente.

---

## ✅ Estado final

| Componente | Situação |
|-------------|-----------|
| **MongoDB 8.0.15** | ✅ Replicaset `rs0` configurado |
| **Autenticação** | ✅ Ativada com `keyFile` |
| **Omada Controller 5.15.x** | ✅ Docker com Mongo externo |
| **Acesso HTTPS** | ✅ Porta 8043 |
| **Backups** | ✅ Automático + manual configurado |

---
