---
name: sysdesign-scalability-horizontal-vs-vertical
description: Use when asked "como escalo Y" or "vertical ou horizontal?" — decides between scaling up the host vs adding more hosts, and surfaces the stateless/shared-write trap.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando alguém precisa crescer a capacidade de um serviço e formula a
pergunta como "dá pra subir a máquina?" ou "precisamos de mais réplicas?". Também
fires quando um plano propõe "escalar" sem distinguir eixo — o caso mais comum.
A skill separa vertical (upgrade do host) de horizontal (mais hosts) e força o
autor a nomear onde está o estado. Estado compartilhado em write é o teto real
de escalabilidade; quem ignora esse fato dobra infra sem dobrar throughput.

## Preconditions

- Existe um sintoma de capacidade nomeado: latência subindo, CPU saturada, fila
  crescendo, custo por request estável mas tráfego crescendo. Escalar por medo
  é anti-padrão — exija sinal.
- O serviço alvo já foi identificado. "Escalar o sistema" é frase inválida —
  escala-se um componente por vez (API gateway, worker, banco, cache).
- Há medição recente de onde está o gargalo. Sem profiling ou sem métricas dos
  4 golden signals, a decisão vira chute.
- A topologia atual cabe em um diagrama — se ninguém consegue desenhá-la, o
  problema é clareza antes de capacidade.

## Execution Workflow

1. Nomeie o componente que precisa escalar e o golden signal que disparou o
   alarme (Latency, Traffic, Errors, Saturation). Um componente, um signal, uma
   decisão.
2. Classifique o componente em 1 de 3 categorias:
   - **Stateless** (API sem sessão local, worker idempotente) → horizontal é
     quase sempre a resposta.
   - **Stateful com estado local** (cache in-memory, sessão sticky) → vertical
     primeiro, depois refatore para mover estado fora.
   - **Stateful com estado compartilhado em write** (banco de escrita, fila
     sem sharding) → nem vertical nem horizontal resolvem sem redesenho.
3. Para stateless: desenhe horizontal. Coloque LB na frente, garanta que cada
   réplica é fungível, e declare o fator de paralelismo alvo (ex: 10 réplicas
   cobrem 10x o tráfego atual com folga).
4. Para stateful local: escolha vertical como medida imediata (CPU/RAM maior),
   e abra ticket de refactor para externalizar estado (Redis, DB) antes do
   próximo teto — vertical tem ceiling duro (tamanho do maior host disponível
   no provedor).
5. Para shared-write: pare de escalar e comece a particionar. Sharding por
   tenant, por user-id, por geo. Write contention não responde a mais hardware.
6. Declare o custo relativo: vertical geralmente é 1.5-3x mais caro que
   horizontal para capacidade equivalente, mas é mais rápido de aplicar. Marque
   trade explicitamente no `nfrs.md` se afetar NFR-Cost.
7. Mede depois de aplicar. Se o signal não melhorou, a categoria estava errada —
   volte ao passo 2 e reclassifique.

## Rules: Do

- Exija um golden signal antes de aprovar qualquer escala. Escalar porque
  "parece lento" queima dinheiro e não resolve.
- Prefira horizontal sempre que o componente for stateless de fato. Replicação
  horizontal dá resiliência de graça (fault-tolerance via redundância).
- Externalize estado antes de escalar horizontalmente. Estado local em réplica
  cria "sticky sessions" que cancelam metade do ganho de horizontal.
- Use vertical como bandaid conscientemente, com prazo. "Subir a máquina por 30
  dias enquanto refatoro" é válido; "subir a máquina como solução final" é dívida.
- Declare o teto do host vertical antes de escolhê-lo. Se o próximo tamanho de
  instância não existe no provedor, vertical acabou antes de começar.

## Rules: Don't

- Não escale write-heavy shared storage horizontalmente sem sharding. Mais
  réplicas de leitura não ajudam quem está bloqueado em write lock.
- Não adicione réplica horizontal para um serviço com cache local grande. Cada
  réplica paga o custo de warming, e hit rate global cai — pior que vertical.
- Não confie em auto-scaling sem definir limites. Auto-scale sem teto vira
  fatura surpresa quando um bug gera loop de requests.
- Não faça vertical "porque é mais simples" quando horizontal é viável.
  Simplicidade de hoje vira parede de amanhã quando o host maior acabou.
- Não trate "vertical vs horizontal" como a pergunta toda. Sharding, caching,
  async queueing, e read replicas são escalas diferentes — o dualismo é
  didático, não exaustivo.

## Expected Behavior

Depois de aplicar a skill, a decisão de escala vira um documento de uma página:
componente nomeado, categoria (stateless / stateful local / shared write), eixo
escolhido (horizontal / vertical / particionamento), fator alvo, custo estimado,
e prazo para remedição se vertical. Quem herda o sistema entende por que há 12
réplicas de um serviço e 1 réplica (monstruosa) de outro. Escala não fica como
decisão silenciosa de Ops — é decisão arquitetural declarada.

## Quality Gates

- Golden signal nomeado e medido antes da decisão.
- Componente classificado em 1 das 3 categorias com justificativa.
- Se vertical: prazo explícito + ticket de refactor para externalizar estado.
- Se horizontal: estado confirmado externo (DB, Redis) ou sessão fungível.
- Se shared-write: decisão de sharding ou migração declarada, não "escalar
  mais".
- Custo relativo calculado e referenciado no `nfrs.md` se afeta NFR-Cost.

## Companion Integration

Chama `sysdesign-nfr-clarification` antes se não houver NFRs declarados — sem
QPS alvo e orçamento, a decisão não fecha. Pareia com
`sysdesign-fault-tolerance-patterns` quando horizontal é escolhido (réplicas dão
redundância; aproveite). Metodologia matilha: fase 20-30 (spec + plan) para
decisão inicial, fase 60 (den) para captura de aprendizado pós-incidente.

## Output Artifacts

- Seção "Scaling decision" na spec do componente, com categoria, eixo, fator.
- Ticket de refactor (se vertical escolhido como bandaid).
- Atualização em `nfrs.md` se NFR-Cost ou NFR-Scalability mudou.
- Runbook atualizado com novo fator de réplica (se horizontal).

## Example Constraint Language

- Use "must" para: nomear golden signal, classificar componente, declarar teto
  do host (se vertical).
- Use "should" para: preferir horizontal para stateless, externalizar estado
  antes de escalar, declarar prazo de bandaid.
- Use "may" para: combinar horizontal + vertical numa transição, usar
  auto-scaling com limites duros, adotar sharding como terceiro eixo.

## Troubleshooting

- **"Escalamos horizontal mas throughput não dobrou"**: estado compartilhado
  escondido. Procure cache local, session affinity, lock em DB. Sem remover,
  horizontal tem ROI decrescente.
- **"Vertical bateu teto e o refactor não ficou pronto"**: bandaid virou
  estrutura. Aplique sharding de emergência (particionar por user-id) para
  ganhar tempo; refactor completo agora é prioridade 1.
- **"Custo explodiu depois de auto-scale"**: faltou limite. Defina teto de
  réplicas por ambiente + alerta em custo/hora. Auto-scale sem teto é armadilha
  financeira.
- **"Dev fala horizontal, Ops fala vertical"**: a conversa está em abstração
  errada. Volte ao passo 2 e classifique o componente juntos — a categoria
  decide, não preferência.

## CLI shortcut (optional)

> Se o CLI matilha estiver instalado, `matilha sysdesign scale-check
> <componente>` pergunta golden signal, categoria, e eixo, e gera a seção de
> decisão pronta para colar na spec. Preferido para padronizar a saída.

## Sources

- `[[concepts/nfr-system-design]]` — seção Scalability (vertical vs horizontal)
- `[[sources/acing-system-design]]` — Tan, fundamentos de escala
- `[[concepts/scaling-databases]]` — sharding, replicação
