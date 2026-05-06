# MP-TRAEFIK

![Traefik](https://img.shields.io/badge/Traefik-v3.0-blue) ![Docker](https://img.shields.io/badge/Docker-Compose-2496ED) ![Let's Encrypt](https://img.shields.io/badge/SSL-Let's%20Encrypt-green)

Stack dedicada ao **Traefik v3** como reverse proxy centralizado para todos os serviços do domínio `minaspneus.com.br`. Único ponto de entrada nas portas 80 e 443. Certificados SSL emitidos e renovados automaticamente via Let's Encrypt.

---

## Como funciona

```
Internet
   │
   ├─ :80  ──────────────────────────────► redirect 301 para HTTPS (automático)
   │
   └─ :443 ──► Traefik (stack isolada)
                   │
                   ├── Host: simulador.minaspneus.com.br  ──► Stack MP-SIMULADORVENDAS
                   ├── Host: vouchers.minaspneus.com.br   ──► Stack MP-BF-VOUCHERS
                   ├── Host: outro.minaspneus.com.br      ──► Stack OUTRO-PROJETO
                   └── Host: traefik.minaspneus.com.br    ──► Dashboard (protegido)
```

**Cada projeto** adiciona labels ao próprio container e entra na rede `traefik-public`. O Traefik detecta automaticamente e roteia o tráfego. Nenhuma configuração é necessária no Traefik em si.

```
Stack: mp-traefik          Stack: mp-simuladorvendas      Stack: mp-vouchers
┌────────────────┐         ┌──────────────────────────┐   ┌─────────────────┐
│    traefik     │         │  frontend  │   backend    │   │    frontend     │
│  (porta 80/443)│         │  (labels)  │   (labels)   │   │    (labels)     │
└───────┬────────┘         └──────┬─────┴──────┬───────┘   └────────┬────────┘
        │                         │            │                     │
        └─────────────────────────┴────────────┴─────────────────────┘
                              rede: traefik-public (compartilhada)
```

---

## Pré-requisitos

- Docker >= 24 e Docker Compose >= 2
- Portainer instalado no servidor
- Acesso SSH ao servidor
- Registro DNS (tipo A) para cada domínio apontando para o IP do servidor
  - Ex: `simulador.minaspneus.com.br` → `200.x.x.x`
  - Incluindo `traefik.minaspneus.com.br` (para o dashboard)
- Portas **80** e **443** abertas no firewall do servidor

---

## Setup inicial

> Execute **uma única vez** no servidor. Não precisa repetir para novos projetos.

### 1. Criar a rede compartilhada

```bash
docker network create traefik-public
```

### 2. Gerar a senha do dashboard

```bash
# Instalar htpasswd se necessário
sudo apt install apache2-utils

# Gerar hash (substitua "suasenha" pela senha desejada)
htpasswd -nb admin suasenha
# Saída: admin:$apr1$abc123...
```

Copie a saída. No `docker-compose.yml`, cole no label `dashboard-auth`, trocando **cada `$` por `$$`**:

```yaml
- "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$abc123..."
```

### 3. Deploy no Portainer

1. Portainer → **Stacks** → **Add Stack**
2. Nome: `mp-traefik`
3. Cole o conteúdo do `docker-compose.yml` (com o hash da senha já substituído)
4. Clique em **Deploy the stack**

### 4. Verificar

Acesse `https://traefik.minaspneus.com.br` — deve pedir usuário/senha e abrir o dashboard do Traefik.

---

## Como adicionar um novo projeto

Copie o template abaixo para o `docker-compose.yml` do seu projeto. Substitua os campos em MAIÚSCULO:

```yaml
services:
  meu-servico:
    image: minha-imagem
    # ⚠️ NÃO defina ports: — o Traefik faz o roteamento
    networks:
      - traefik-public   # conecta ao Traefik
      - internal         # rede interna do projeto (opcional, para comunicação entre containers)
    labels:
      # Habilita este container no Traefik
      - traefik.enable=true

      # Especifica qual rede o Traefik deve usar para alcançar este container
      - traefik.docker.network=traefik-public

      # Nome único do router — use só letras, números e hífens
      - traefik.http.routers.NOME-UNICO.rule=Host(`MEU-DOMINIO.minaspneus.com.br`)
      - traefik.http.routers.NOME-UNICO.entrypoints=websecure
      - traefik.http.routers.NOME-UNICO.tls.certresolver=letsencrypt

      # Porta interna que o container escuta (ex: 80, 3000, 8080)
      - traefik.http.services.NOME-UNICO.loadbalancer.server.port=PORTA

networks:
  traefik-public:
    external: true   # ← sempre assim, para referenciar a rede compartilhada
  internal:
    driver: bridge
```

### Checklist antes do deploy

- [ ] Registro DNS criado e propagado (`dig MEU-DOMINIO.minaspneus.com.br`)
- [ ] Container **não tem** `ports:` definido
- [ ] Container está na rede `traefik-public`
- [ ] `NOME-UNICO` é diferente de todos os outros projetos

---

## Cenários com exemplos completos

### A — Frontend simples (React, Vue, site estático)

```yaml
services:
  frontend:
    image: nginx:alpine
    networks:
      - traefik-public
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.http.routers.meu-site.rule=Host(`meu-site.minaspneus.com.br`)
      - traefik.http.routers.meu-site.entrypoints=websecure
      - traefik.http.routers.meu-site.tls.certresolver=letsencrypt
      - traefik.http.services.meu-site.loadbalancer.server.port=80

networks:
  traefik-public:
    external: true
```

---

### B — Frontend + Backend no mesmo domínio (API em /api)

O Traefik roteia automaticamente por especificidade: `/api` vai para o backend, o restante para o frontend.

```yaml
services:
  frontend:
    image: meu-frontend
    networks:
      - traefik-public
      - internal
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.http.routers.app-front.rule=Host(`app.minaspneus.com.br`)
      - traefik.http.routers.app-front.entrypoints=websecure
      - traefik.http.routers.app-front.tls.certresolver=letsencrypt
      - traefik.http.services.app-front.loadbalancer.server.port=80

  backend:
    image: meu-backend
    networks:
      - traefik-public
      - internal
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      # Regra mais específica — Traefik v3 prioriza automaticamente
      - traefik.http.routers.app-api.rule=Host(`app.minaspneus.com.br`) && PathPrefix(`/api`)
      - traefik.http.routers.app-api.entrypoints=websecure
      - traefik.http.routers.app-api.tls.certresolver=letsencrypt
      - traefik.http.services.app-api.loadbalancer.server.port=3000

networks:
  traefik-public:
    external: true
  internal:
    driver: bridge
```

---

### C — Subdomínio separado para a API

```yaml
services:
  backend:
    image: meu-backend
    networks:
      - traefik-public
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.http.routers.minha-api.rule=Host(`api.minaspneus.com.br`)
      - traefik.http.routers.minha-api.entrypoints=websecure
      - traefik.http.routers.minha-api.tls.certresolver=letsencrypt
      - traefik.http.services.minha-api.loadbalancer.server.port=3000

networks:
  traefik-public:
    external: true
```

---

### D — Serviço interno (sem exposição pública)

Para serviços que só precisam ser acessados internamente (banco de dados, filas, etc.):

```yaml
services:
  banco:
    image: postgres:16
    networks:
      - internal   # só na rede interna, sem traefik-public
    # Sem labels do Traefik, sem ports bind
    # Acesso externo apenas via SSH tunnel:
    # ssh -L 5432:banco:5432 usuario@servidor

networks:
  internal:
    driver: bridge
```

---

## Certificados SSL

### Como funciona

1. Na primeira requisição para um domínio, o Traefik contata o Let's Encrypt
2. O Let's Encrypt verifica o controle do domínio via HTTP: acessa `http://SEU-DOMINIO/.well-known/acme-challenge/TOKEN`
3. O Traefik responde automaticamente ao desafio
4. Certificado emitido e armazenado no volume `letsencrypt`
5. **Renovação automática**: o Traefik verifica diariamente e renova 30 dias antes do vencimento

> Você não precisa fazer nada para renovar certificados. É 100% automático.

### Onde ficam os certificados

Os certificados ficam dentro do volume Docker `mp-traefik_letsencrypt`:

```bash
# Ver informações do volume
docker volume inspect mp-traefik_letsencrypt

# Ver o conteúdo do acme.json (contém todos os certificados — mantenha privado)
docker run --rm -v mp-traefik_letsencrypt:/data alpine cat /data/acme.json | python3 -m json.tool
```

### Verificar validade de um certificado

```bash
# Via openssl
echo | openssl s_client -connect meu-site.minaspneus.com.br:443 2>/dev/null | openssl x509 -noout -dates

# Via curl
curl -vI https://meu-site.minaspneus.com.br 2>&1 | grep -E "expire|issuer"
```

### O que fazer se o certificado não for emitido

1. **Verifique o DNS**: `dig meu-site.minaspneus.com.br` — deve retornar o IP do servidor
2. **Verifique a porta 80**: `curl http://meu-site.minaspneus.com.br` deve ser acessível externamente
3. **Veja os logs do Traefik**: `docker compose logs traefik | grep -i "acme\|cert\|error"`
4. **Limite do Let's Encrypt**: máximo de 5 tentativas falhas por hora por domínio — aguarde se excedeu

---

## Troubleshooting

| Problema | Sintoma | Causa mais comum | Solução |
|---|---|---|---|
| Container não aparece no dashboard | Rota ausente | Sem `traefik.enable=true` ou container não está na rede `traefik-public` | Verificar labels e `networks:` |
| 404 no browser | Página não encontrada | Router com nome errado ou porta interna incorreta | Conferir labels no dashboard do Traefik |
| SSL não emitido | Aviso de certificado inválido no browser | DNS não propagado ou porta 80 bloqueada no firewall | `dig dominio` + testar porta 80 externamente |
| Loop de redirect | ERR_TOO_MANY_REDIRECTS | Projeto ainda tem redirect HTTP→HTTPS próprio | Remover redirect do projeto, deixar o Traefik fazer |
| Dashboard retorna 401 | Usuário/senha recusados | Hash com `$` simples em vez de `$$` | Corrigir label de basicauth e redeployar |
| Dois projetos no mesmo domínio | Um sobrescreve o outro | Nome de router duplicado | Garantir nomes únicos entre todos os projetos |

### Comandos de diagnóstico

```bash
# Logs em tempo real
docker compose logs -f traefik

# Ver todas as rotas ativas (via API do Traefik)
curl -s http://localhost:8080/api/http/routers | python3 -m json.tool

# Verificar quais containers estão na rede traefik-public
docker network inspect traefik-public --format '{{range .Containers}}{{.Name}} {{end}}'

# Ver labels de um container específico
docker inspect NOME-DO-CONTAINER | python3 -c "import sys,json; [print(c['Config']['Labels']) for c in json.load(sys.stdin)]"
```

---

## Migração de projetos existentes

### Exemplo: MP-SIMULADORVENDAS

**Passo 1 — Parar o proxy atual**

```bash
docker compose stop proxy
```

**Passo 2 — Atualizar o docker-compose.yml do projeto**

Remover o serviço `proxy` e todas as suas referências. Nos serviços `frontend` e `backend`, substituir:

```yaml
# ANTES — frontend com proxy próprio
frontend:
  networks:
    - internal

# DEPOIS — frontend roteado pelo Traefik
frontend:
  networks:
    - traefik-public
    - internal
  labels:
    - traefik.enable=true
    - traefik.docker.network=traefik-public
    - traefik.http.routers.simulador-front.rule=Host(`simulador.minaspneus.com.br`)
    - traefik.http.routers.simulador-front.entrypoints=websecure
    - traefik.http.routers.simulador-front.tls.certresolver=letsencrypt
    - traefik.http.services.simulador-front.loadbalancer.server.port=80
```

```yaml
# DEPOIS — backend roteado pelo Traefik
backend:
  networks:
    - traefik-public
    - internal
  labels:
    - traefik.enable=true
    - traefik.docker.network=traefik-public
    - traefik.http.routers.simulador-api.rule=Host(`simulador.minaspneus.com.br`) && PathPrefix(`/api`)
    - traefik.http.routers.simulador-api.entrypoints=websecure
    - traefik.http.routers.simulador-api.tls.certresolver=letsencrypt
    - traefik.http.services.simulador-api.loadbalancer.server.port=3000
```

Ao final do arquivo, adicionar a rede externa:

```yaml
networks:
  traefik-public:
    external: true
  internal:
    driver: bridge
```

**Passo 3 — Redeploy**

```bash
docker compose up -d --force-recreate frontend backend
```

**Passo 4 — Verificar**

Acesse o dashboard do Traefik em `https://traefik.minaspneus.com.br` → HTTP Routers. As rotas `simulador-front` e `simulador-api` devem aparecer com status **Enabled** e TLS **acme**.

O Traefik emite o certificado automaticamente na primeira requisição. Tempo estimado: menos de 1 minuto.

---

## Ordem de deploy

```
1. No servidor (uma única vez):
   docker network create traefik-public

2. Portainer → Stack mp-traefik (este repositório)
   ↓ ocupa as portas 80/443

3. Para cada projeto — adicionar labels + rede traefik-public → redeploy
   ↓ Traefik detecta e emite SSL automaticamente

4. Novos projetos → só usar o template de labels → zero configuração adicional
```
