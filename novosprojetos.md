# Como iniciar um novo projeto com Traefik

## Frase para colar no inicio do planejamento com Claude

Cole o texto abaixo ao iniciar qualquer novo projeto para que o Claude
ja entenda a infraestrutura e estruture o compose corretamente:

```
Este projeto roda na infraestrutura minaspneus.com.br que usa o MP-TRAEFIK
como reverse proxy centralizado (Traefik v3). Nao deve ter proxy proprio
(nginx, caddy, etc). Os containers publicos entram na rede externa
traefik-public e usam labels do Traefik com certresolver=letsencrypt.
O dominio sera XPTO.minaspneus.com.br.
```

Substitua `XPTO` pelo subdominio do novo projeto.

---

## Template de labels para o compose

### Frontend (porta 80, rota /)

```yaml
networks:
  - traefik-public
labels:
  - traefik.enable=true
  - traefik.docker.network=traefik-public
  - traefik.http.routers.PROJETO-frontend.rule=Host(`XPTO.minaspneus.com.br`)
  - traefik.http.routers.PROJETO-frontend.entrypoints=websecure
  - traefik.http.routers.PROJETO-frontend.tls.certresolver=letsencrypt
  - traefik.http.services.PROJETO-frontend.loadbalancer.server.port=80
```

### Backend com rota /api (porta 3000)

```yaml
networks:
  - traefik-public
labels:
  - traefik.enable=true
  - traefik.docker.network=traefik-public
  - traefik.http.routers.PROJETO-backend.rule=Host(`XPTO.minaspneus.com.br`) && PathPrefix(`/api`)
  - traefik.http.routers.PROJETO-backend.entrypoints=websecure
  - traefik.http.routers.PROJETO-backend.tls.certresolver=letsencrypt
  - traefik.http.services.PROJETO-backend.loadbalancer.server.port=3000
```

### Declaracao da rede no final do compose

```yaml
networks:
  internal:
    driver: bridge
  traefik-public:
    external: true
```

---

## Checklist obrigatorio

- [ ] Registro DNS tipo A criado para o subdominio apontando para o IP do servidor
- [ ] Sem servico proxy proprio no compose (remover nginx, caddy, etc.)
- [ ] Containers publicos conectados a rede `traefik-public`
- [ ] Rede `traefik-public` declarada como `external: true` no compose
- [ ] Labels do Traefik adicionadas em cada container exposto
- [ ] `PROJETO` substituido por nome unico (diferente de todos os outros projetos)
- [ ] Porta correta no label `loadbalancer.server.port` (verificar no compose ou Dockerfile)
- [ ] Stack mp-traefik esta rodando no Portainer antes de subir o novo projeto
- [ ] Label `traefik.docker.network=traefik-public` adicionada em cada container exposto (obrigatorio quando o container esta em mais de uma rede)
- [ ] Health checks usando `127.0.0.1` em vez de `localhost` (evita problema IPv6 em containers Alpine)

---

## Regras importantes

| Regra | Detalhe |
|---|---|
| `PROJETO` deve ser unico | Nao pode repetir entre projetos diferentes |
| `PROJETO-frontend` != `PROJETO-backend` | Mesmo projeto, nomes diferentes por container |
| Porta e interna ao container | Varios projetos podem usar a mesma porta interna sem conflito |
| Traefik roteia pelo dominio | Host() na regra define qual projeto recebe a requisicao |
| PathPrefix mais especifico vence | /api vai para o backend; / vai para o frontend |
| Traefik ignora containers unhealthy | Se o router nao aparecer no dashboard, verificar se o container esta healthy |
| Health check: usar 127.0.0.1 | localhost pode resolver para IPv6 em Alpine — usar 127.0.0.1 explicitamente |