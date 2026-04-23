---
name: sysdesign-consistency-cap
description: Use when asked "strong ou eventual consistency?" / "qual banco?" — forces CAP choice (CP vs AP), maps it to HBase/Mongo/Redis vs Cassandra/Dynamo, and selects coordination technique.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando a conversa toca banco de dados distribuído, replicação, cache
multi-region, ou chega na pergunta "posso ter os dois?" (resposta: não sob
partição de rede). Também fires quando alguém propõe Cassandra para dados que
precisam de transação linearizável, ou Postgres single-master para um sistema
com alvo 99.999 de availability — casos em que o CAP está sendo ignorado. A
skill força a escolha CP vs AP, mapeia para tecnologias canônicas, e seleciona
a técnica de coordenação (Full Mesh, Coordination Service, Distributed Cache,
Gossip) proporcional à escala.

## Preconditions

- Existe um dado concreto sob discussão. "Qual banco usar?" sem entidade
  definida é abstrato demais. Dê nome (orders, user_sessions, inventory).
- A natureza do dado está clara: financeiro (exige exatidão), contador
  (tolera eventual), cache (tolera stale), identidade (exige unicidade).
- NFR-Availability e NFR-Consistency já existem em `nfrs.md` — a skill assume
  `sysdesign-nfr-clarification` rodou antes.
- Escala alvo declarada (QPS, regiões, réplicas). Technique que serve 3 nodes
  não serve 300.

## Execution Workflow

1. Nomeie o dado e classifique em 1 de 3:
   - **ACID-strict** — dinheiro, inventário, chaves únicas. Exige transação
     com foreign keys + uniqueness. Postgres/MySQL single-master ou Spanner.
   - **Linearizável mas distribuído** — read-your-write, ordering global.
     CP: HBase, MongoDB (maioria), Redis com Sentinel.
   - **Tolera eventual** — feed, contador, cache, telemetria. AP: Cassandra,
     CouchDB, DynamoDB, S3.
2. Aplique o teorema CAP de forma prática: sob partição de rede, você escolhe
   entre responder com dado possivelmente stale (AP) ou recusar responder
   (CP). "Escolher C e A ao mesmo tempo" só vale sem partição — e partição
   em multi-region é rotina, não exceção.
3. Selecione tecnologia canônica pelo perfil:
   - ACID-strict → Postgres/MySQL (single region, vertical scale + read
     replicas); Spanner/CockroachDB se precisar de ACID global.
   - CP distribuído → MongoDB (write concern majority), HBase, Redis Sentinel.
   - AP distribuído → Cassandra, DynamoDB, Riak.
4. Escolha técnica de coordenação por escala:
   - **Full Mesh** (todos-com-todos) — só funciona até ~10 nodes; N² de
     conexões explode. Use só em clusters pequenos de configuração estática.
   - **Coordination Service** (ZooKeeper, etcd via Raft, Consul) — padrão
     para leader election, service discovery, feature flags até ~10k clients.
   - **Distributed Cache** (Redis Cluster) — consistência "melhor esforço"
     para read-heavy; não use como source of truth.
   - **Gossip Protocol** — Cassandra, Dynamo usam para membership. Escala
     para milhares de nodes; eventually consistent.
5. Declare o que "eventual" significa em janela. "Eventual consistency" sem
   prazo é buraco — escreva "converge em ≤ 5 segundos em 99% dos casos" no
   `nfrs.md`.
6. Desenhe quórum quando aplicável. W + R > N garante strong em Dynamo-like.
   W=3, R=3, N=5 é strong; W=1, R=1, N=3 é eventual barato. Custo vs. garantia
   é escolha explícita.
7. Valide com cenário real: "cliente paga, depois abre página de pedidos".
   Se a leitura é de réplica eventual, pode não ver o pedido. Aí precisa de
   read-your-write (sticky read ao leader, ou invalidação de cache).

## Rules: Do

- Nomeie o dado antes de discutir tecnologia. Banco não é decisão global, é
  por entidade.
- Escolha CP ou AP explicitamente. "Ambos" é idioma de fornecedor, não
  engenharia.
- Declare janela de convergência para dados eventuais. Sem número, "eventual"
  é metáfora.
- Use Coordination Service (ZooKeeper/etcd/Raft) para leader election, não
  heurística caseira.
- Teste split-brain em staging. Derrube rede entre metade do cluster e
  verifique comportamento. Surpresa em prod é cara.

## Rules: Don't

- Não assuma que 1 banco serve todo o sistema. Orders em Postgres, feed em
  Cassandra, cache em Redis — é saudável, não fragmentação.
- Não trate eventual como "quase strong". É diferente qualitativamente —
  read-your-write quebra sem cuidado específico.
- Não monte Full Mesh para 50+ nodes. N² de conexões derruba a rede antes de
  dar throughput.
- Não use Redis Cluster como source of truth. Failover perde writes;
  aceitável para cache, fatal para ordens.
- Não combine "ACID global" com "99.999 availability" sem provisionar
  Spanner-like + pagar o custo. Promessa barata de ambos é fraude técnica.

## Expected Behavior

Depois de aplicar a skill, cada entidade de dado tem entrada em `data-model.md`
com: classe (ACID-strict / CP / AP), tecnologia escolhida, técnica de
coordenação, e janela de convergência (se eventual). Decisões ficam auditáveis.
Quando aparece cenário de split-brain ou dado stale, o time consulta a
classificação e sabe se é comportamento esperado ou bug.

## Quality Gates

- Cada entidade no modelo tem classe (ACID / CP / AP) declarada.
- Tecnologia justifica a classe (não o inverso — escolher DB antes é ancoragem).
- Técnica de coordenação proporcional à escala alvo.
- Janela de convergência numérica para dados eventuais.
- Quórum (W, R, N) explícito quando aplicável.
- Cenário read-your-write testado se o sistema tem leitura pós-escrita
  imediata.
- Split-brain test rodado em staging antes do primeiro release multi-region.

## Companion Integration

Depende de `sysdesign-nfr-clarification` (Consistency e Availability precisam
estar em `nfrs.md`). Pareia com `sysdesign-availability-sla-tiers` (tier
5-noves + strong consistency multi-region é muito caro — force escolha) e
com `sysdesign-fault-tolerance-patterns` (leader election via coordination
service). Fase 20-30 matilha (spec/plan).

## Output Artifacts

- `data-model.md` ou seção na spec, com classe + tecnologia por entidade.
- Janela de convergência declarada para dados eventuais.
- Script de teste split-brain em staging (documentado + versionado).
- Configuração de quórum (W/R/N) no código, não em commit verbal.

## Example Constraint Language

- Use "must" para: classificar cada entidade (ACID / CP / AP), declarar
  janela de convergência para eventuais, usar Coordination Service para
  leader election.
- Use "should" para: múltiplos stores por sistema (1 por classe), testar
  split-brain antes de prod, quórum majority em CP distribuído.
- Use "may" para: Spanner/CockroachDB quando ACID global é obrigatório,
  Gossip para cluster grande, Redis Cluster como cache distribuído (nunca
  como SoT).

## Troubleshooting

- **"Cliente paga mas não vê pedido na página seguinte"**: eventual
  consistency sem read-your-write. Force leitura no leader após write, ou
  use version token no cliente para esperar convergência.
- **"Cluster Cassandra tem dados divergentes entre nodes"**: esperado em
  janelas curtas (gossip converge). Se diverge por minutos, suspeite de
  partição persistente ou clock skew — verifique NTP.
- **"MongoDB escreveu, mas leitura retornou antigo"**: write concern não
  estava em "majority" ou read preference em secondary. Corrija configuração
  ou aceite eventual + documente.
- **"ZooKeeper saturou com 50k clients"**: passou do envelope. Migre para
  etcd (Raft, escala melhor) ou quebre em clusters hierárquicos.
- **"Split-brain corrompeu dados"**: leader election sem quorum de maioria
  ou fencing ausente. Exija quorum N/2+1 antes de aceitar write; adicione
  fencing token.

## CLI shortcut (optional)

> Se o CLI matilha estiver instalado, `matilha sysdesign consistency-classify
> <entity>` pergunta natureza do dado, escala, janela aceitável, e sugere
> tecnologia + coordenação.

## Sources

- `[[concepts/nfr-system-design]]` — Consistency, CAP, técnicas de
  coordenação
- `[[sources/acing-system-design]]` — Tan, capítulo sobre consistency
- `[[concepts/distributed-transactions]]` — 2PC, quórum, leader election
- `[[concepts/scaling-databases]]` — replicação, sharding, read replicas
