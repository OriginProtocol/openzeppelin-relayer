# OpenZeppelin Relayer

This relayer service enables interaction with blockchain networks through transaction submissions. It offers multi-chain support and an extensible architecture for adding new chains.

[User Docs](https://docs.openzeppelin.com/relayer/) | [Quickstart](https://docs.openzeppelin.com/relayer/quickstart)

## Pre-requisites

- Docker installed on your machine
- [Sodium](https://doc.libsodium.org/). See [install sodium section](https://github.com/OpenZeppelin/openzeppelin-relayer?tab=readme-ov-file#install-sodium) for more information.
- `.env` file with the required environment variables & `config/config.json` file with the required configuration. See how to set it up in [config files section](https://github.com/OpenZeppelin/openzeppelin-relayer?tab=readme-ov-file#config-files) for more information.
- Create signers and add them to the `config/config.json` file. See how to set it up in [signers section](https://github.com/OpenZeppelin/openzeppelin-relayer?tab=readme-ov-file#creating-a-signer) for more information.
- Configure webook url in `config/config.json` file. See how to set it up in [webhook section](https://github.com/OpenZeppelin/openzeppelin-relayer?tab=readme-ov-file#configure-webhook-url) for more information.
- Configure webhook signing key in `config/config.json` file. See how to set it up in [webhook section](https://github.com/OpenZeppelin/openzeppelin-relayer?tab=readme-ov-file#configure-webhook-signing-key) for more information.
- Configure Api key in `config/config.json` file. See how to set it up in [api key section](https://github.com/OpenZeppelin/openzeppelin-relayer?tab=readme-ov-file#configure-api-key) for more information.
- Redis server running. See how to set it up in [redis section](https://github.com/OpenZeppelin/openzeppelin-relayer?tab=readme-ov-file#starting-redis-manually-without-docker-compose) for more information.

> ⚠️ Redis is automatically started when using docker compose. If you are not using docker compose, you need to create a dedicated network and start redis manually.

## Running Docker locally

### 1. Setup env vars

```bash
cp .env.example .env
uuidgen -> generates UUID
```

Create a unique uuid for WEBHOOK_SIGNING_KEY & API_KEY and set them in .env

Set the correct AWS_ACCESS_KEY_ID & AWS_SECRET_ACCESS_KEY depending on user / relayer environment requires.
 
AWS user `relayer-kms-signer-dev` 
 - has access to `origin-relayer-development-evm`
 - with a public address: `0xca00ab46d0e009985c84c41e2f712c31102ff967`

AWS user `relayer-kms-signer` 
 - has access to `origin-relayer-production-evm`
 - with a public address: `[todo....]`

### 2. Cofigure the correct config file

```bash
# for development relayer
cp config/config.development.json config/config.json

# for production relayer
cp config/config.production.json config/config.json
```

### 3. Start the service

```bash
docker compose up
```

### 4. Access the service

Once the container is running, you can access the service at `http://localhost:8080`.

```bash
API_KEY=[set the api key] curl -s http://localhost:8080/api/v1/relayers \
  -H "Authorization: Bearer $API_KEY" | jq
```

## Railway production deployment (with Redis)

Use Railway source builds with `Dockerfile.production`. A separate GitHub workflow for image builds is not required.

Reference template: `.env.railway.production.example`.

### 1. Provision Redis in Railway

1. Add a Redis service from Railway templates.
2. Rename it clearly, for example: `redis-prod`.
3. Keep Redis internal/private (disable public TCP proxy unless you explicitly need external access).

### 2. Configure relayer service variables

Set these in the Railway service running this repo:

```env
CONFIG_DIR=./config
CONFIG_FILE_NAME=config.production.json
REPOSITORY_STORAGE_TYPE=redis
REDIS_URL=${{redis-prod.REDIS_URL}}
REDIS_KEY_PREFIX=oz-relayer-prod
REDIS_CONNECTION_TIMEOUT_MS=10000
```

Required application secrets (already needed by the relayer):

```env
API_KEY=<min-32-char-secret>
WEBHOOK_SIGNING_KEY=<secret>
AWS_ACCESS_KEY_ID=<aws-access-key-id>
AWS_SECRET_ACCESS_KEY=<aws-secret-access-key>
```

Recommended for production data protection:

```env
STORAGE_ENCRYPTION_KEY=<base64-encoded-32-byte-key>
DISTRIBUTED_MODE=true
```

### 3. Deploy and validate

1. Trigger a Railway deployment.
2. Confirm logs do not contain `REDIS_URL must be set` or Redis connection errors.
3. Verify health endpoint:

```bash
curl -s https://<your-railway-domain>/api/v1/health
```

4. Verify relayers are loaded from `config.production.json`:

```bash
curl -s https://<your-railway-domain>/api/v1/relayers \
  -H "Authorization: Bearer <API_KEY>" | jq
```

Expected: both `mainnet` and `base` relayers are present.

### 4. Backups and restore runbook

1. Enable Redis backups/snapshots in Railway.
2. During incident recovery, restore the latest healthy snapshot on Redis.
3. Redeploy the relayer service to re-establish clean connections.
4. Re-run the health and relayer validation calls above.
