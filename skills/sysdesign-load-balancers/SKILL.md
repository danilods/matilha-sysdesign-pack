---
name: sysdesign-load-balancers
description: Use when asked "qual LB usar?" / "HAProxy vs ALB?" — chooses between hardware, LBaaS, and software (HAProxy/NGINX), picks L4 vs L7, and handles sticky sessions + TLS termination.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando a conversa pede Load Balancer — "preciso distribuir carga",
"onde termino TLS?", "ALB ou NLB?", "HAProxy ou NGINX?". Também fires quando
sticky sessions aparecem como solução para um serviço stateless (pista de que
o estado está no lugar errado) ou quando TLS termina em cada réplica (custo
de CPU evitável). A skill separa três eixos: categoria (hardware vs LBaaS vs
software), camada OSI (L4 vs L7), e tratamento de sessão (none / sticky por
cookie / sticky por IP).

## Preconditions

- Existe um serviço com >1 instância (ou planejado). LB para 1 réplica é
  complexidade sem benefício — use proxy reverso simples.
- Tráfego alvo estimado (RPS, banda). Hardware LB é overkill até ~100k RPS;
  software open-source cobre a maior parte dos casos.
- Decisão sobre TLS: onde termina (LB, réplica, ambos)? Sem decisão, cada
  réplica vai pagar CPU de handshake — custo não-trivial.
- NFR-Availability declarado — LB é SPOF se não for redundante, e redundância
  muda a topologia.

## Execution Workflow

1. Classifique a categoria:
   - **Hardware** (F5, Citrix NetScaler, A10) — $1k-$100k+. Usado em
     on-prem com throughput muito alto ou requisitos regulatórios. Raramente
     justifica em cloud moderna.
   - **LBaaS** (AWS ALB/NLB, GCP Load Balancer, Azure Front Door) —
     pay-per-use, integrado com cloud, manutenção zero. Default em cloud.
   - **Software** (HAProxy, NGINX, Envoy, Traefik) — roda em VM/container
     próprio. Flexível, open-source, exige operar. Preferido quando LBaaS
     não cobre (on-prem, features específicas, multi-cloud).
2. Escolha camada OSI:
   - **L4 (TCP/UDP)** — roteia por IP:porta, sem inspecionar payload.
     Latência menor, throughput maior, sem features HTTP. Usado para TCP
     genérico, gRPC, gaming, baixa latência. Ex: AWS NLB, HAProxy TCP mode.
   - **L7 (HTTP/HTTPS)** — inspeciona cabeçalhos, path, cookies. Enables
     path-based routing, host-based routing, TLS termination, auth, rate
     limiting, WAF. Paga CPU extra. Ex: AWS ALB, NGINX, HAProxy HTTP mode.
3. Decida onde termina TLS:
   - **No LB** (padrão L7) — CPU de handshake concentrada; réplicas falam
     HTTP plano dentro da VPC. Mais barato e simples.
   - **Passthrough para réplica** (L4 ou re-encrypt em L7) — usado quando
     compliance exige TLS end-to-end ou aplicação inspeciona certificado.
     Custa CPU em cada réplica.
4. Defina tratamento de sessão:
   - **Stateless** (sem sticky) — default; réplicas são fungíveis. Exige
     sessão externa (Redis, JWT assinado). Preferido sempre que possível.
   - **Sticky por cookie** (application-controlled ou LB-issued) — LB
     injeta cookie, requests subsequentes do mesmo browser vão para mesma
     réplica. Usado quando sessão local é inevitável.
   - **Sticky por IP** — roteia por IP do cliente. Quebra com proxies/NAT/
     mobile. Use só quando cookie não é opção (TCP não-HTTP).
5. Planeje redundância do próprio LB. LB single-instance é SPOF que derruba
   o SLA inteiro. Em LBaaS, redundância é automática. Em software, use
   Keepalived + VIP ou DNS round-robin com health checks.
6. Configure health check por réplica. HTTP GET em `/health` a cada 5-10s;
   réplica sai do pool após 2-3 falhas consecutivas. Sem health check, LB
   manda tráfego para réplica morta.
7. Rate limiting e WAF opcionais em L7. Use LBaaS ou software (NGINX mod,
   HAProxy stick-tables) para proteção básica contra abuse/DDoS.

## Rules: Do

- Default para LBaaS em cloud. Custo operacional de software LB raramente
  justifica vs 1 click do provedor.
- Termine TLS no LB em L7. Um único certificado, um handshake por conexão,
  réplicas internas plano. Simples, barato.
- Prefira stateless. Se sticky é obrigatório, documente por quê — costuma
  ser sintoma de estado mal posicionado.
- Configure health check com path explícito (`/health`), não raiz (`/`).
  Raiz pode 200 mesmo quando app está degradada.
- Redunde o próprio LB. 99.99 depende de LB que não é SPOF.

## Rules: Don't

- Não use L7 para tráfego que não precisa de inspeção. NLB/HAProxy L4 é mais
  rápido e barato para gRPC/TCP genérico.
- Não termine TLS em cada réplica "por segurança" sem requisito de
  compliance. Duplica CPU de handshake em todas as réplicas.
- Não confie em sticky-by-IP para web pública. Proxy corporativo mascara
  milhares de clientes atrás de 1 IP — load fica desbalanceado.
- Não pule health check. Réplica morta recebendo tráfego = usuário vendo
  5xx que LB deveria ter evitado.
- Não deixe LB como SPOF. É o componente com maior leverage sobre SLA;
  redundância não é opcional em tier 99.99+.

## Expected Behavior

Depois de aplicar a skill, cada serviço com >1 réplica tem LB com categoria,
camada OSI, TLS termination, tratamento de sessão, e redundância declarados
em `lb.md` (ou seção na spec). Decisões ficam auditáveis: "escolhemos ALB
porque L7 com path-based routing + TLS termination em um ponto era exigência
da NFR-Complexity (minimizar plumbing)". O LB deixa de ser artefato silencioso
de Ops e vira componente de arquitetura.

## Quality Gates

- Categoria (hardware / LBaaS / software) justificada por contexto (cloud,
  on-prem, tráfego, budget).
- Camada OSI escolhida por feature necessária, não default.
- Ponto de terminação TLS documentado.
- Tratamento de sessão declarado (stateless / sticky cookie / sticky IP) com
  justificativa se sticky.
- Health check configurado com path explícito.
- LB redundante (LBaaS automático ou software com failover ativo).
- Runbook de failover testado (kill do LB ativo, verificar que tráfego migra).

## Companion Integration

Depende de `sysdesign-nfr-clarification` (Availability, Latency, Cost guiam
escolha). Pareia com `sysdesign-scalability-horizontal-vs-vertical` (LB é
infra que horizontal pressupõe) e `sysdesign-fault-tolerance-patterns`
(health check + failover são patterns aplicados ao próprio LB). Fase 30-40
matilha (skills/agents + hunt) para provisioning, fase 50 (review) para
auditar config de produção.

## Output Artifacts

- `lb.md` ou seção na spec: categoria, L4/L7, TLS termination, sessão,
  redundância.
- Config versionada (HAProxy `haproxy.cfg`, NGINX config, ou Terraform para
  LBaaS).
- Runbook de failover testado.
- Dashboard com health check status + distribuição de carga por réplica.

## Example Constraint Language

- Use "must" para: health check por réplica, redundância de LB em tier
  99.99+, TLS em algum ponto do caminho.
- Use "should" para: LBaaS em cloud como default, TLS termination no LB,
  L7 quando features HTTP são usadas, stateless como default de sessão.
- Use "may" para: hardware LB em on-prem com throughput extremo, L4
  passthrough para compliance TLS end-to-end, sticky cookie quando sessão
  local é inevitável e documentada.

## Troubleshooting

- **"Carga desbalanceada entre réplicas"**: sticky-by-IP com proxies, ou
  health check derrubando réplicas saudáveis (timeout apertado). Migre para
  cookie-based ou afrouxe timeout.
- **"TLS handshake lento"**: certificado em cada réplica ou falta de session
  resumption. Concentre em LB + habilite session tickets.
- **"LB caiu, site fora"**: SPOF. Adicione segundo LB (LBaaS multi-AZ, ou
  Keepalived + VIP em software).
- **"Health check passa, mas app está quebrada"**: `/health` é raso demais
  (200 estático). Enriquecça com check de DB, cache, dependências críticas.
- **"Latency extra do LB estoura budget"**: L7 com inspeção pesada, regex
  caro, ou LB em região distante. Mova para L4 se features não usadas, ou
  aproxime geograficamente.

## CLI shortcut (optional)

> Se o CLI matilha estiver instalado, `matilha sysdesign lb-pick` pergunta
> cloud/on-prem, tráfego, features (path-routing, TLS, auth), sessão, e
> sugere categoria + L4/L7 com template de config.

## Sources

- `[[concepts/nfr-system-design]]` — Load Balancers, L4/L7, sticky sessions
- `[[sources/acing-system-design]]` — Tan, capítulo de common services
