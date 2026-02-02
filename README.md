# Nextcloud Talk Recording Service

Serviço de gravação do [Nextcloud Talk](https://github.com/nextcloud/nextcloud-talk-recording) v0.2.1.

## Pré-requisitos

- Docker e Docker Compose instalados
- Nextcloud com app Talk instalado e configurado
- Spreed High Performance Backend (HPB) configurado e rodando
- Reverse proxy configurado (nginx-proxy ou similar)
- Certificados SSL configurados (Let's Encrypt ou similar)

## Configuração

### 1. Clone o repositório

```bash
git clone https://github.com/LibreCodeCoop/nextcloud-talk-recording
cd nextcloud-talk-recording
```

### 2. Configure as variáveis de ambiente

Copie o arquivo `.env` de exemplo e edite com suas configurações:

```bash
nano .env
```

#### Variáveis obrigatórias para ajustar:

```bash
# Domínios
NC_DOMAIN=cloud.exemplo.com.br          # Seu domínio do Nextcloud
HPB_DOMAIN=signaling.exemplo.com.br     # Domínio do Spreed HPB
VIRTUAL_HOST=recording.exemplo.com.br   # Domínio para este serviço
LETSENCRYPT_HOST=recording.exemplo.com.br
LETSENCRYPT_EMAIL=admin@exemplo.com.br

# Secrets (IMPORTANTE: gerar novos valores)
RECORDING_SECRET=00000000-0000-0000-0000-000000000000
INTERNAL_SECRET=0000000000000000000000000000000000000000000000000000000000000000
```

#### Gerar secrets seguros:

```bash
# RECORDING_SECRET (novo, único para este serviço)
uuidgen

# INTERNAL_SECRET (usar o MESMO valor do spreedbackend)
# Este secret já existe na configuração do seu HPB
# Localize o valor do "internalsecret" no servidor.conf do spreedbackend
```

**⚠️ IMPORTANTE sobre INTERNAL_SECRET:**

O `INTERNAL_SECRET` **DEVE SER IDÊNTICO** ao `internalsecret` configurado no Spreed High Performance Backend (HPB). Este secret é usado para autenticar a comunicação entre o serviço de gravação e o HPB.

- **Localização no HPB**: Geralmente em `/etc/signaling/server.conf` na seção `[backend]`
- **Não gere um novo valor**: Use o valor existente do HPB
- **Formato**: String hexadecimal de 64 caracteres (32 bytes)

Exemplo de como encontrar o valor no servidor HPB:

```bash
# No servidor do HPB
grep internalsecret /etc/signaling/server.conf
```

### 3. Configuração do HPB Path

**⚠️ Nota importante sobre HPB_PATH:**

Este setup considera que o Spreed High Performance Backend está em um **subdomínio dedicado**, não em um path.

- **Subdomínio** (padrão): `https://signaling.exemplo.com.br/`
  ```bash
  HPB_DOMAIN=signaling.exemplo.com.br
  HPB_PATH=/
  ```

- **Path específico**: `https://cloud.exemplo.com.br/signaling/`
  ```bash
  HPB_DOMAIN=cloud.exemplo.com.br
  HPB_PATH=/signaling/
  ```

Ajuste as variáveis `HPB_DOMAIN` e `HPB_PATH` conforme sua configuração.

### 4. Configurar redes Docker

Certifique-se de que as redes Docker existem:

```bash
# Criar redes se não existirem
docker network create reverse-proxy
docker network create internal
```

Ou ajuste as variáveis `NETWORK_PROXY` e `NETWORK_INTERNAL` no `.env` para usar suas redes existentes.

### 5. Ajustar recursos

Configure limites de CPU e memória conforme seu hardware:

```bash
# Recursos padrão (recomendado mínimo 2GB RAM)
CPUS_LIMIT=4
MEMORY_LIMIT=4048M
CPUS_RESERVATION=2
MEMORY_RESERVATION=2048M
SHM_SIZE=2147483648  # 2GB
```

### 6. Configurar imagem Docker

Por padrão, o `.env` usa a imagem publicada no GitHub Container Registry:

```bash
IMAGE_NAME=ghcr.io/librecodecoop/nextcloud-talk-recording
IMAGE_TAG=latest
```

**Opções de tags disponíveis:**
- `latest` - Última versão da branch main
- `main` - Branch main (mesma coisa que latest)
- `v0.2.1` - Versão específica (quando criar tags)
- `main-<sha>` - Commit específico

**Build local (opcional):**

Se preferir fazer build localmente:

```bash
# Build da imagem
docker build -t nextcloud-talk:0.2.1 .

# Ajuste o .env para usar a imagem local
IMAGE_NAME=nextcloud-talk
IMAGE_TAG=0.2.1
```

## Executar o serviço

### Subir o serviço

```bash
docker compose --env-file .env.local --profile talk-recording up -d
```

### Verificar logs

```bash
docker compose --env-file .env.local logs -f recording
```

### Verificar status

```bash
docker compose --env-file .env.local ps
```

### Healthcheck

O container possui healthcheck automático. Verifique o status:

```bash
docker inspect --format='{{.State.Health.Status}}' recording
```

## Configuração no Nextcloud

### 1. Configurar o serviço de gravação

No Nextcloud, acesse **Configurações > Talk > Gravação**:

1. Habilite o serviço de gravação
2. Configure a URL do servidor: `https://recording.exemplo.com.br`
3. Configure o secret (use o mesmo valor de `RECORDING_SECRET` do `.env`)

### 2. Configurar permissões

Defina quais usuários/grupos podem iniciar gravações nas configurações do Talk.

## Troubleshooting

### Container não inicia

```bash
# Verificar logs
docker compose --env-file .env.local logs recording

# Verificar configuração
docker compose --env-file .env.local config
```

### Erro de conexão com Nextcloud

- Verifique se `NC_DOMAIN` está correto
- Verifique se `NC_PROTOCOL` corresponde ao seu setup (http/https)
- Se usar certificados autoassinados, configure `SKIP_VERIFY=true` (não recomendado para produção)

### Erro de conexão com HPB

- Verifique se `HPB_DOMAIN` e `HPB_PATH` estão corretos
- Teste a conectividade: `docker exec recording nc -zv $HPB_DOMAIN 443`
- Verifique os logs do HPB para erros de autenticação

### Problemas de memória

- Aumente `SHM_SIZE` se houver erros relacionados a memória compartilhada
- Ajuste `MEMORY_LIMIT` conforme necessário
- Monitore uso: `docker stats recording`

### Problemas de certificado SSL

```bash
# Para desenvolvimento/teste apenas (INSEGURO)
SKIP_VERIFY=true

# Para produção, garanta certificados válidos
SKIP_VERIFY=false
```

## Manutenção

### Parar o serviço

```bash
docker compose --env-file .env.local --profile talk-recording down
```

### Atualizar a imagem

```bash
# Pull da última versão
docker compose --env-file .env.local --profile talk-recording pull

# Restart com a nova imagem
docker compose --env-file .env.local --profile talk-recording up -d
```

Se estiver usando build local:

```bash
# Rebuild
docker build -t nextcloud-talk:0.2.1 .

# Restart
docker compose --env-file .env.local --profile talk-recording up -d
```

### Backup da configuração

```bash
# Backup do .env
cp .env.local .env.backup-$(date +%Y%m%d)
```

## Segurança

⚠️ **Importante:**

- Nunca commite o arquivo `.env.local` com secrets reais
- Use secrets fortes e únicos
- Mantenha `ALLOW_ALL=false` em produção
- Use `SKIP_VERIFY=false` em produção
- Configure certificados SSL válidos
- Mantenha o container e dependências atualizados

## Recursos do Container

O container executa com:

- **User**: UID 122 (não-root)
- **Read-only filesystem**: Sim (segurança adicional)
- **Capabilities**: NET_RAW removido
- **Volumes temporários**: `/tmp` e `/conf` em tmpfs
- **Healthcheck**: Automático

## Componentes instalados

- Python 3.14.0
- Firefox + Geckodriver (para gravação via browser)
- FFmpeg (processamento de vídeo)
- Xvfb (X virtual framebuffer)
- PulseAudio (áudio)

## CI/CD

Este repositório possui GitHub Actions configurado para:

- **Build automático**: A cada push na branch `main`
- **Multi-arquitetura**: Builds para `linux/amd64` e `linux/arm64`
- **Publicação no GHCR**: Imagens disponíveis em `ghcr.io/librecodecoop/nextcloud-talk-recording`
- **Tagging automático**: Quando criar tags `v*`, gera versões semânticas
- **Cache otimizado**: Usa GitHub Actions cache para acelerar builds
- **Artifact attestation**: Verificação de proveniência da imagem

### Tags geradas automaticamente:

- `latest` - Última versão da branch main
- `main` - Branch main
- `main-<sha>` - Commit específico (ex: `main-a1b2c3d`)
- `v0.2.1`, `v0.2`, `v0` - Quando criar tag de versão

### Disparar build manual:

1. Vá para **Actions** no GitHub
2. Selecione **Build and Push Docker Image**
3. Clique em **Run workflow**

## Suporte

Para problemas ou dúvidas:

- Logs do container: `docker compose logs recording`
- Documentação oficial: https://github.com/nextcloud/nextcloud-talk-recording
- Issues do projeto upstream: https://github.com/nextcloud/nextcloud-talk-recording/issues
- Implementação no Nextcloud AIO: https://github.com/nextcloud/all-in-one/tree/main/Containers/talk-recording
- Repositório inicial(?) com mais informações: https://github.com/MetaProvide/talked

## Licença

Este serviço é baseado no projeto Nextcloud Talk Recording, mantido pela Nextcloud GmbH.
