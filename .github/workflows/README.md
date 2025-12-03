# GitHub Actions Workflows

Este diretório contém os workflows de CI/CD para o projeto.

## docker-build.yml

Workflow responsável por build e publicação da imagem Docker no GitHub Container Registry (GHCR).

### Triggers

- **Push na branch main**: Build automático da imagem com tag `latest` e `main`
- **Tags v***: Build automático com versionamento semântico (ex: `v0.2.1` gera tags `0.2.1`, `0.2`, `0`)
- **Pull Requests**: Build de teste sem publicação
- **Manual**: Pode ser disparado manualmente via interface do GitHub Actions

### Recursos

- **Multi-arquitetura**: Builds para `linux/amd64` e `linux/arm64`
- **Cache otimizado**: Usa GitHub Actions cache para acelerar builds subsequentes
- **Metadata automático**: Extrai tags e labels baseado em convenções
- **Artifact attestation**: Gera atestados de proveniência para rastreabilidade

### Permissões necessárias

O workflow requer as seguintes permissões:

- `contents: read` - Ler o código do repositório
- `packages: write` - Publicar no GitHub Container Registry
- `id-token: write` - Gerar artifact attestation

Essas permissões são configuradas automaticamente no GitHub Actions.

### Imagens publicadas

As imagens são publicadas em: `ghcr.io/librecodecoop/nextcloud-talk-recording`

Para usar:

```bash
docker pull ghcr.io/librecodecoop/nextcloud-talk-recording:latest
```

### Visibilidade do pacote

Por padrão, pacotes no GHCR são privados. Para tornar público:

1. Vá para https://github.com/orgs/LibreCodeCoop/packages
2. Encontre o pacote `nextcloud-talk-recording`
3. Acesse **Package settings**
4. Em **Danger Zone**, clique em **Change visibility**
5. Selecione **Public**

### Verificar builds

Para verificar o status dos builds:

1. Acesse https://github.com/LibreCodeCoop/nextcloud-talk-recording/actions
2. Clique no workflow **Build and Push Docker Image**
3. Veja o histórico de execuções

### Troubleshooting

**Build falha com erro de permissão:**
- Verifique se as GitHub Actions estão habilitadas no repositório
- Confirme que o repositório tem permissões para publicar pacotes

**Imagem não aparece no GHCR:**
- Verifique se o workflow foi executado com sucesso
- Confirme que não foi um build de PR (PRs não publicam)
- Verifique se o token GITHUB_TOKEN tem permissões adequadas

**Build muito lento:**
- O primeiro build sempre é mais lento (sem cache)
- Builds subsequentes devem ser mais rápidos com cache
- Multi-arch builds levam mais tempo (2 arquiteturas)
