# CLAUDE.md — MP-TRAEFIK

Contexto do projeto para consultas e melhorias futuras.

---

## O que é este projeto

Stack Docker Compose dedicada ao **Traefik v3** como reverse proxy centralizado para todos os serviços do domínio `minaspneus.com.br`. Funciona como único ponto de entrada nas portas 80 e 443, com SSL automático via Let's Encrypt.

Repositório destinado à publicação no GitHub.

---

## Arquitetura

- **Uma stack isolada** (`mp-traefik`) no Portainer, contendo apenas o container Traefik
- **Rede externa compartilhada** `traefik-public` — criada manualmente no servidor, conecta o Traefik a todos os outros projetos
- **Todos os outros projetos** definem labels nos próprios containers e entram na rede `traefik-public`; o Traefik detecta automaticamente via Docker provider
- **SSL**: Let's Encrypt ACME via HTTP challenge — emissão e renovação totalmente automáticas
- **Dashboard**: protegido por BasicAuth em `traefik.minaspneus.com.br`

```
Internet :80/:443
    └── Traefik (stack mp-traefik)
            ├── Host: simulador.minaspneus.com.br  →  Stack mp-simuladorvendas
            ├── Host: vouchers.minaspneus.com.br   →  Stack mp-bf-vouchers
            └── Host: traefik.minaspneus.com.br    →  Dashboard (BasicAuth)

           Todas as stacks conectadas via rede: traefik-public
```

---

## Decisões de design

| Decisão | Motivo |
|---|---|
| Stack isolada para o Traefik | Ciclo de vida independente; reiniciar ou atualizar o proxy não derruba os projetos |
| `exposedbydefault=false` | Segurança: só containers com `traefik.enable=true` ficam expostos |
| Configuração via `command:` (não `traefik.yml`) | Tudo em um único arquivo — mais simples para Portainer |
| Volume nomeado `letsencrypt` | Certificados sobrevivem a `docker compose down` |
| `name: mp-traefik` no compose | Nome do volume previsível independente do diretório no servidor |
| BasicAuth hash como placeholder no compose | Credenciais não expostas no repositório |

---

## Arquivos do projeto

| Arquivo | Finalidade |
|---|---|
| `docker-compose.yml` | Stack Traefik v3 — deployar uma vez no Portainer |
| `README.md` | Documentação completa: setup, templates, cenários, troubleshooting |
| `CLAUDE.md` | Este arquivo — contexto para o assistente em futuras sessões |

---

## Como adicionar novo projeto (resumo)

1. Registrar DNS tipo A para o domínio no servidor
2. Garantir que o container do projeto está na rede `traefik-public`
3. Adicionar labels no container do projeto:

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.NOME_UNICO.rule=Host(`subdominio.minaspneus.com.br`)
  - traefik.http.routers.NOME_UNICO.entrypoints=websecure
  - traefik.http.routers.NOME_UNICO.tls.certresolver=letsencrypt
  - traefik.http.services.NOME_UNICO.loadbalancer.server.port=PORTA_INTERNA
networks:
  - traefik-public

# No nível do compose (fora do service):
networks:
  traefik-public:
    external: true
```

4. Redeploy da stack do projeto no Portainer
5. Confirmar no dashboard (`traefik.minaspneus.com.br`) que o router aparece como ENABLED

---

## Comandos de operação

```bash
# Criar a rede compartilhada (apenas uma vez no servidor)
docker network create traefik-public

# Gerar hash BasicAuth para o dashboard
htpasswd -nb admin suasenha
# Copiar o resultado e substituir cada $ por $$ ao colar no compose

# Verificar certificado SSL de um domínio
echo | openssl s_client -connect subdominio.minaspneus.com.br:443 2>/dev/null | openssl x509 -noout -dates

# Ver logs do Traefik em tempo real
docker logs -f $(docker ps -qf name=traefik)

# Localizar e inspecionar o volume dos certificados
docker volume inspect mp-traefik_letsencrypt
```

---

## Variáveis que precisam ser ajustadas no deploy

| Campo | Localização | Instrução |
|---|---|---|
| Hash BasicAuth | `docker-compose.yml`, linha do middleware `dashboard-auth` | Gerar com `htpasswd -nb admin SENHA`; escapar `$` → `$$` |
| E-mail ACME | Flag `--certificatesresolvers.letsencrypt.acme.email` | Já configurado: `angelo.mcbraga@gmail.com` |
| Domínio do dashboard | Label `traefik.http.routers.dashboard.rule` | Já configurado: `traefik.minaspneus.com.br` |

---

## Contexto histórico (decisões de sessões anteriores)

- Projeto criado do zero a partir de escopo fornecido pelo usuário
- Plataforma de deploy: **Portainer** (stacks)
- Servidor: domínio raiz `minaspneus.com.br`
- A stack Traefik é **completamente isolada** das stacks dos projetos — cada projeto tem sua própria stack no Portainer
- Projeto existente que precisa migrar: **MP-SIMULADORVENDAS** (ver seção de migração no README.md)
- Let's Encrypt: renovação automática pelo Traefik (verifica diariamente, renova 30 dias antes do vencimento)
- Rate limit Let's Encrypt: 5 certificados falhos por hora por domínio — não fazer testes repetidos em produção
- **Bug conhecido**: a ferramenta Write/Read do Claude Code não consegue acessar diretamente este diretório (EACCES); usar Python3 via Bash como alternativa

---

## Melhorias futuras sugeridas

- [ ] Configurar Traefik para enviar alertas por e-mail ou webhook quando certificado falhar
- [ ] Adicionar middleware de rate limiting global para proteção contra abuso
- [ ] Avaliar uso de `traefik.yml` externo (via volume bind-mount) para configurações mais avançadas
- [ ] Considerar adicionar acesso ao dashboard com MFA no futuro (hoje só BasicAuth)
- [ ] Documentar processo de backup do `acme.json` (dentro do volume letsencrypt)
