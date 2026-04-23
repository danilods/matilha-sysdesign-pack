---
name: sysdesign-nfr-clarification
description: Use when about to draft system architecture or hear "antes de desenhar X" / "vamos modelar o sistema" — forces clarification of the 10 non-functional requirements before a single box gets drawn.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara sempre que alguém começa a desenhar arquitetura, escolher stack, ou propor uma
topologia distribuída sem antes declarar NFRs. Gatilhos típicos: "vamos desenhar o
sistema de notificações", "como eu modelo este pipeline", "que banco uso aqui",
"antes de começar a arquitetura". A skill pausa a conversa e exige que 10 requisitos
não-funcionais sejam tornados explícitos antes de qualquer caixa no diagrama. O
cliente interno (PM, fundador, outro dev) raramente declara NFRs sozinho — assume que
o sistema "vai funcionar" — e essa lacuna é exatamente o que a skill cobre.

## Preconditions

- Existe pelo menos uma afirmação sobre o que o sistema DEVE fazer (requisito
  funcional mínimo: "receber X, devolver Y"). Sem isso, volte para discovery antes.
- Há um interlocutor acessível (humano ou outro agente) para responder às perguntas
  de clarificação. Se não houver, documente as suposições como defaults explícitos.
- O trabalho é genuinamente arquitetural — não cobre um componente interno de uma
  função já definida. Para refatorações locais, a skill é overkill.
- O escopo do sistema cabe em uma frase. Se precisar de um parágrafo para descrever
  o que ele faz, divida antes.

## Execution Workflow

1. Congele qualquer tentativa de desenhar arquitetura. Se a conversa já pulou para
   "vamos usar Kafka", reverta explicitamente: "antes de escolher tecnologia, preciso
   fechar 10 NFRs".
2. Abra um documento `nfrs.md` no repo (ou seção numa spec existente). NFRs vivem
   como arquivo versionado, não em chat — a decisão vai ser referenciada por meses.
3. Rode as 10 perguntas na ordem abaixo. Para cada uma, registre número concreto ou
   "assumido: <default>" se o interlocutor não tiver resposta. Vago ("rápido",
   "confiável") é banido — vire número ou assume default declarado.
   1. **Scalability** — QPS alvo no pico? Crescimento esperado em 12 meses?
   2. **Availability** — SLA mensurável? 99.9 / 99.99 / 99.999?
   3. **Latency** — P50, P95, P99 aceitáveis? Em que percentil o usuário reclama?
   4. **Consistency** — Strong ou eventual? O que pode divergir entre réplicas?
   5. **Cost** — Orçamento mensal alvo? Qual eixo é barato de gastar?
   6. **Accuracy** — Contagem exata ou aproximada (HLL, count-min) serve?
   7. **Security** — Quem autentica? TLS obrigatório? Encryption at rest?
   8. **Privacy** — PII envolvida? GDPR/LGPD aplicável? Retenção de dados?
   9. **Complexity** — Que serviços comuns (LB, auth, logging) já existem?
   10. **Fault-tolerance** — Que dependências podem cair? MTTR aceitável?
4. Para cada NFR, marque tensão com outros — barato E alta disponibilidade E baixa
   latência é mentira. Escolha no máximo 2 eixos como "duros" e marque os outros
   como "negociáveis". Registre a escolha.
5. Derive API surface e topologia SÓ depois do `nfrs.md` fechado. Se o próximo
   desenho contradiz algum NFR, o desenho perdeu — não o NFR.
6. Publique os NFRs como seção da spec. Todos os diagramas subsequentes devem citar
   qual NFR cada decisão atende.

## Rules: Do

- Substitua cada adjetivo por número. "Rápido" não existe em `nfrs.md` — existe
  "P99 ≤ 200ms sob 10k QPS". A tradução força honestidade.
- Declare defaults quando o interlocutor não souber responder, e marque-os como
  "assumido" para que sejam revisitados. Default silencioso é um bug futuro.
- Ordene os 10 NFRs por impacto no design. Consistency strong e Availability 5-noves
  no mesmo sistema mudam a topologia inteira — decida primeiro essas.
- Versione `nfrs.md` junto com o código. Quando um NFR mudar, o commit aparece no
  histórico e quem herda o sistema vê a intenção.
- Revise NFRs antes de cada wave nova. Escala muda, custo muda, PII que não havia
  passou a existir — a planilha é viva, não artefato de discovery.

## Rules: Don't

- Não desenhe arquitetura antes do `nfrs.md`. Cada caixa desenhada sem NFRs vira
  ancoragem e distorce a escolha de tecnologia nos passos seguintes.
- Não aceite "alta disponibilidade" como resposta. Exija o tier numérico; 99.9 e
  99.999 são dois sistemas diferentes, não variantes.
- Não combine 3+ NFRs como "duros". Sistemas reais negociam — quem exige tudo
  simultaneamente gasta 5x e entrega menos.
- Não misture requisitos funcionais no `nfrs.md`. Função ("enviar email") e
  qualidade ("com P99 ≤ 2s") vivem em listas separadas; misturar esconde decisões.
- Não trate NFRs como entrevista. Este não é prep de interview — o objetivo é
  contratar expectativas para um sistema real que alguém vai manter.

## Expected Behavior

Depois de aplicar a skill, a conversa sai de "vamos desenhar" e chega em um
arquivo versionado com 10 linhas numéricas. A arquitetura que vem depois referencia
NFRs explicitamente: "escolhi Cassandra porque NFR-Consistency aceita eventual e
NFR-Availability exige AP". Decisões técnicas ficam auditáveis. Quando o sistema
falha em produção, o time consulta `nfrs.md` e sabe se violou contrato ou se o
contrato estava otimista.

## Quality Gates

- `nfrs.md` existe no repo, commitado, com os 10 NFRs numéricos ou marcados como
  "assumido: <default>".
- Nenhum NFR tem adjetivo ("alto", "baixo", "bom") — todos têm número ou tier.
- No máximo 2 NFRs marcados como "duros"; os outros são "negociáveis" explícitos.
- Cada decisão arquitetural subsequente no documento cita qual NFR atende.
- O arquivo foi revisado por alguém que vai manter o sistema, não só por quem o
  desenhou.

## Companion Integration

Complementa `matilha-harness-pack:harness-nfrs-as-prompts` — onde aquela skill
codifica NFRs como constraints no system prompt de um agente, esta codifica NFRs
como perguntas numéricas na spec humana. Fase 10-20 da metodologia matilha
(discovery + spec). Pareia com `matilha-scout` (discovery) e `matilha-plan` (quando
o plano de subprojetos precisa referenciar NFRs para decidir ordem de merge).

## Output Artifacts

- `nfrs.md` (ou seção equivalente na spec) com 10 NFRs numéricos + defaults.
- Log de tensões declaradas (quais eixos são duros, quais são negociáveis).
- Commit no repo referenciando os 10 NFRs — histórico preserva intenção.

## Example Constraint Language

- Use "must" para: versionar `nfrs.md` no repo, traduzir cada adjetivo em número,
  decidir NFRs antes de desenhar topologia.
- Use "should" para: revisar NFRs a cada wave, citar NFR em cada decisão técnica,
  marcar defaults explicitamente como "assumido".
- Use "may" para: agrupar NFRs similares (Security + Privacy) quando domínio
  permite, adicionar um 11º NFR customizado (ex: Observability) se o sistema exigir.

## Troubleshooting

- **"O cliente não sabe responder nenhum NFR"**: este é o caso comum. Declare
  defaults do domínio (SaaS B2B típico: 99.9%, P99 500ms, eventual consistency,
  encrypted at rest) e marque como "assumido" — volte em 1 semana para revisar.
- **"O time já começou a codar sem NFRs"**: escreva `nfrs.md` agora com o que já
  existe, marque o que contradiz NFRs realistas, e decida quais ajustar no
  próximo sprint. Evite retrofit arqueológico.
- **"Dois NFRs brigam e ninguém quer escolher"**: a skill falhou em forçar
  tradeoff. Volte ao passo 4, nomeie explicitamente os eixos "duros", e force
  quem manda a decidir — sem decisão, o sistema escolhe sozinho e sempre mal.
- **"NFRs estão declarados mas o desenho ignora"**: quality gate falhou. Reabra
  `nfrs.md` na PR, exija que cada caixa referencie NFR, e bloqueie merge até que
  contradições sejam resolvidas ou NFRs ajustados.

## CLI shortcut (optional)

> Se o CLI matilha estiver instalado (`matilha --version` funciona), você pode
> rodar `matilha sysdesign nfr-init` para gerar o template `nfrs.md` pré-preenchido
> com os 10 NFRs e defaults de domínio. Preferido para CI ou novos projetos. O
> caminho do plugin acima funciona sem CLI.

## Sources

- `[[concepts/nfr-system-design]]` — 10 NFRs, tensões, template de clarificação
- `[[sources/acing-system-design]]` — Zhiyong Tan, Part 1 (Fundamentals)
- `[[entities/zhiyong-tan]]`
