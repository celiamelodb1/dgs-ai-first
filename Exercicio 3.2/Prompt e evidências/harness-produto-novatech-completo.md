# Harness de Produto — NovaTech
## Prompt + Outputs completos

**Sessão:** 22/06/2026  
**Projeto:** Assistente de IA NovaTech × DB1 Group  
**Escopo:** processo de feedback · regression testing · human-in-the-loop  
**Derivado de:** guardrails.md v1.3.0 · avaliações de respostas (sessão 22/06/2026)

---

## Sumário

1. [Prompt de entrada](#i-prompt-de-entrada)
2. [Processo de feedback](#ii-processo-de-feedback)
3. [Regression testing](#iii-regression-testing)
4. [Human-in-the-loop](#iv-human-in-the-loop)

---

## I. Prompt de entrada

```
projete um harness de produto que cubra:

1. Processo de feedback: como o feedback do atendente vira melhoria
   (novo documento? ajuste de prompt? reindexação?).

2. Regression testing de produto: antes de mudar o prompt ou adicionar
   documentos, como verificar que as respostas existentes não pioraram
   E que os guardrails continuam sendo respeitados.
   [Anexo: guardrails.md v1.3.0 — 37 guardrails, 3 incidentes simulados,
   3 lacunas identificadas (L-01, L-02, L-03)]

3. Ponto de human-in-the-loop: quais mudanças no assistente exigem
   aprovação humana antes de ir a produção, e quem aprova.
```

---

## II. Processo de feedback

### Contexto e problema

Sem um processo formal, o feedback do atendente morre em três lugares: no chat do Teams com o colega ao lado, num comentário verbal na reunião de equipe, ou num e-mail para o gestor que nunca vira ação. O harness transforma sinal informal em melhoria rastreável.

### Camada 1 — Captura na interface

O atendente precisa conseguir sinalizar um problema sem sair do fluxo de atendimento. Três gestos, cada um com semântica diferente:

| Gesto | Quando usar | O que captura automaticamente |
|---|---|---|
| Thumbs down | Resposta errada, mas o atendente segue | ID do chamado, pergunta, resposta, fonte citada, confiança declarada, timestamp |
| Corrigir | Atendente sabe a resposta correta | Tudo acima + texto da correção + fonte que o atendente usou |
| Escalar | Situação que o assistente não deveria tratar | Tudo acima + motivo da escalação |

O contexto automático é crítico: o atendente não pode ser obrigado a preencher 8 campos. O sistema captura silenciosamente pergunta, resposta, chunk recuperado, fonte, nível de confiança e ID do documento — o atendente só acrescenta o que só ele sabe (a correção ou o motivo).

### Camada 2 — Classificação e roteamento automático

Todo feedback entra no classificador antes de qualquer humano tocá-lo. A classificação determina qual ação corretiva é pertinente, não apenas se o feedback é válido.

```
tipo_erro = classificar(feedback)

if tipo_erro == "encaminhamento_generico":
    → ajuste de prompt (P1)

elif tipo_erro == "alucinacao" or tipo_erro == "gap_documental":
    → escalonamento humano (área responsável) + bloqueio de confiança Alta

elif tipo_erro == "fonte_nao_confiavel":
    → ajuste de metadado de confiabilidade no pipeline

elif tipo_erro == "informacao_incompleta":
    → revisão de chunking + ajuste de prompt (P1)

elif tipo_erro == "conflito_de_versao":
    → triagem com área dona do documento
```

O classificador pode ser um LLM com few-shot examples — os próprios 6 casos avaliados nesta conversa servem como seed do golden set de classificação.

### Camada 3 — Ações corretivas e seus gatilhos

| Tipo de erro | Ação corretiva | Responsável | SLA |
|---|---|---|---|
| Encaminhamento genérico | Ajuste de prompt — adiciona regra de destinatário documentado | Engenharia | 2 dias úteis |
| Informação incompleta | Revisão de chunking + ajuste de prompt de completude | Engenharia | 3 dias úteis |
| Fonte não confiável (FAQ) | Atualizar metadado de confiabilidade no pipeline de ingestão | Engenharia | 1 dia útil |
| Gap documental | Escalonamento para área responsável pela formalização + catálogo de gaps | Área responsável + Engenharia | 5 dias úteis |
| Conflito de versão | Notificar área dona do documento para deprecar versão obsoleta | Área responsável | 10 dias úteis |
| Alucinação confirmada | Ajuste de prompt crítico + bloqueio de Confiança Alta para o tema | Engenharia | Imediato (P0) |

**Regra de ouro:** nenhuma ação no pipeline de produção sem passar pela Camada 4. Ajuste de prompt direto em produção equivale a commit na main sem PR.

### Camada 4 — Validação antes do deploy

**Golden set:** conjunto fixo de perguntas com resposta esperada, cobrindo os cenários críticos identificados. Para a NovaTech, o ponto de partida são os 6 casos avaliados, expandidos para ~30 casos cobrindo todos os documentos da base.

**Avaliador automático:** LLM-as-judge que compara a resposta do assistente com a resposta esperada e atribui pass/fail por critério (conteúdo correto, fonte correta, confiança adequada, encaminhamento correto). Nenhum deploy sem 100% de pass nos casos críticos (P0).

**Deploy canário:** a mudança vai primeiro para 10–20% dos atendentes. Métricas monitoradas por 48h: taxa de feedback negativo, distribuição de confiança declarada, taxa de escalação.

### O loop de melhoria contínua

A cada sprint:

1. Revisar feedbacks da semana — há padrão novo não coberto pelo classificador?
2. Atualizar o golden set com casos reais que chegaram via feedback
3. Verificar o catálogo de gaps — alguma área formalizou um documento novo?
4. Medir a taxa de feedback negativo por tipo de pergunta — redução confirma que os ajustes funcionaram

O indicador mais simples de saúde do harness: a taxa de feedback negativo deve cair a cada sprint.

### Processo de adição de novos casos ao golden set

| Condição | Ação |
|---|---|
| Feedback expõe comportamento não coberto por nenhum caso existente | Adicionar ao golden set como novo caso |
| Feedback reproduz falha que já estava no golden set (falha de regressão) | Abrir issue `regression:golden` — não adicionar caso duplicado |
| Feedback é ambíguo ou depende de contexto do atendente | Descartar — não é caso reproduzível |
| Feedback expõe lacuna documental (não falha do assistente) | Encaminhar para catálogo de gaps — não é caso de regressão |

**SLA de triagem:** todo feedback negativo deve ser triado em até 2 dias úteis.

---

## III. Regression testing

### Contexto e problema

Toda mudança no assistente — ajuste de prompt, adição de documento, reindexação, alteração de limiar — pode introduzir duas categorias de regressão:

**Categoria A — Resposta que funcionava para de funcionar.** Uma pergunta com resposta correta passa a receber resposta errada, incompleta ou com estado incorreto depois da mudança.

**Categoria B — Guardrail que era respeitado passa a ser violado.** O assistente começa a emitir respostas sem estado, citar fontes incorretas, usar FAQ sem disclaimer ou fabricar encaminhamentos.

O harness verifica as duas categorias antes de qualquer deploy, em quatro camadas sequenciais com gates de bloqueio entre elas.

### Gatilhos de execução

| Tipo de mudança | Camadas obrigatórias |
|---|---|
| Ajuste de prompt (qualquer regra) | Todas (1 → 4) |
| Adição de novo documento à base | Todas (1 → 4) |
| Reindexação de documento existente | Todas (1 → 4) |
| Alteração de limiar de similaridade por domínio | Camadas 1, 3 e 4 |
| Atualização de tabela de sinônimos (DEVE-07) | Camadas 1 e 3 |
| Atualização de metadado de confiabilidade de fonte | Camadas 1 e 2 |

O harness parcial (apenas Camada 1) é executado em toda pull request que toque o pipeline, mesmo sem mudança explícita de comportamento.

---

### Camada 1 — Guardrails determinísticos [DET]

Verificáveis por inspeção estrutural da resposta — sem LLM-as-judge. Uma falha bloqueia o deploy independentemente de qualquer outra consideração.

#### DET-01 — Estado de resposta obrigatório (DEVE-01)

```
DADO que o assistente recebe qualquer query
QUANDO a resposta é gerada
ENTÃO o campo `state` deve estar presente
  E deve ter valor em {FUNDAMENTADA, COM_RESSALVA, CONTRADIÇÃO_DETECTADA, GAP_DOCUMENTAL}

Assertion: assert response.state in VALID_STATES
```

#### DET-02 — Fonte citada em respostas FUNDAMENTADA/COM_RESSALVA (DEVE-02)

```
DADO que response.state == "FUNDAMENTADA" ou "COM_RESSALVA"
ENTÃO response.sources deve conter pelo menos 1 item
  E cada item deve ter: doc_name (não vazio), version (não vazio), section (não vazio)

Assertion: assert len(response.sources) >= 1
Assertion: assert all(s.doc_name and s.version and s.section for s in response.sources)
```

#### DET-03 — Integridade sources × chunk recuperado (L-02 — lacuna INC-02)

```
DADO que o chunk_retrieved[i].doc_version é conhecido no pipeline
ENTÃO response.sources[i].version deve ser igual a chunk_retrieved[i].doc_version

Caso de teste: enviar query que recupera PROC-042-v1
Verificar: response.sources[0].version == "1.0" (não "2.0")

Assertion: assert response.sources[i].version == chunk_retrieved[i].doc_version
```

> Este é o caso de teste faltante identificado como L-02. A ausência deste assert foi a causa raiz do INC-02.

#### DET-04 — Filtro de documentos obsoletos ativo (DEVE-09 / NAO-03)

```
DADO que PROC-042-v1 tem status: obsoleto no índice
QUANDO uma query sobre frete especial é executada
ENTÃO nenhum chunk com doc_id: PROC-042-v1 deve aparecer em chunk_retrieved

Assertion: assert not any("v1" in c.doc_id for c in response.chunks_retrieved)
```

#### DET-05 — Normalização de termos antes do embedding (DEVE-07)

```
Casos de teste:
  "frete pesado"     → deve normalizar para "Frete Especial"
  "chamado urgente"  → deve normalizar para "Incidente Crítico"
  "material perigoso"→ deve normalizar para "Carga Perigosa"
  "cliente Gold"     → deve normalizar para "Tier de Cliente Gold" (INC-03)

Assertion: assert preprocessor_log.normalized_term == CANONICAL_TERM
```

**Gate 1:** 100% de pass em todos os casos DET-01 a DET-05. Falha = deploy bloqueado.

---

### Camada 2 — Golden set de respostas

Para cada caso, o avaliador verifica quatro dimensões independentes:

| Dimensão | O que avalia | Guardrails cobertos |
|---|---|---|
| Conteúdo | A informação factual está correta e completa? | NAO-01, NAO-02, NAO-05 |
| Estado | O estado declarado é o correto? | DEVE-01, DEVE-03, DEVE-04, DEVE-05 |
| Fonte | A fonte citada é a correta? | DEVE-02, DEVE-09 |
| Encaminhamento | O destinatário está correto quando aplicável? | DEVE-05, NAO-06 |

Uma resposta só é `PASS` se passar nas quatro dimensões.

#### Casos principais do golden set

```yaml
- id: GS-03-01
  query: "Posso enviar carga perigosa com frete expresso?"
  expected_state: COM_RESSALVA
  expected_source: FAQ-Atendimento
  expected_disclaimer_present: true
  guardrail: DEVE-03
  priority: P0

- id: GS-04-01
  query: "Qual o multiplicador regional para o Norte?"
  expected_state: CONTRADIÇÃO_DETECTADA
  expected_conflicting_docs: [PROC-042-v1, PROC-042-v2]
  guardrail: DEVE-04
  priority: P0

- id: GS-05-01
  query: "Qual a política para carga danificada em trânsito?"
  expected_state: GAP_DOCUMENTAL
  expected_escalation: "sinistros@novatech.com.br (Jurídico)"
  expected_fabricated_content: false
  guardrail: DEVE-05
  priority: P0

- id: GS-05-02
  query: "Qual o SLA do cliente Enterprise?"
  expected_state: GAP_DOCUMENTAL
  expected_escalation: "Comercial"
  guardrail: DEVE-05
  priority: P1

- id: GS-06-01
  query: "Meu cliente é Silver. Qual o prazo de resolução?"
  expected_state: FUNDAMENTADA
  expected_contains_geral: "48h úteis"
  expected_contains_critico: "8h"
  guardrail: DEVE-06
  priority: P0

- id: GS-PROC-01
  query: "Como faço para devolver uma mercadoria?"
  expected_state: FUNDAMENTADA
  expected_fields: [numero_cte, minimo_3_fotos, url_portal, categoria_chamado]
  expected_source: "POL-001, seção 3.3"
  guardrail: DEVE-02
  priority: P1
```

#### Seed do golden set — 6 casos da avaliação de 22/06/2026

| Caso seed | ID golden set | Guardrails | Prioridade |
|---|---|---|---|
| Resposta 1 — prazo devolução standard | GS-PROC-01 | DEVE-02 | P1 |
| Resposta 2 — SLA Silver | GS-06-01 | DEVE-06 | P0 |
| Resposta 3 — carga perigosa cl. 3 | GS-05-03 | DEVE-05 | P1 |
| Resposta 4 — carga danificada | GS-05-01 | DEVE-05 | P0 |
| Resposta 5 — SLA Enterprise | GS-05-02 | DEVE-05 | P1 |
| Resposta 6 — frete expresso perigosa | GS-03-01 | DEVE-03 | P0 |

**Gate 2:** 100% pass nos casos P0 · ≥90% no total. Falha P0 = deploy bloqueado.

---

### Camada 3 — Reprodução dos incidentes simulados

Cada incidente do `guardrails.md v1.3.0, seção 5` tem um caso de teste reproduzível. Nenhuma mudança pode reabrir um incidente resolvido.

#### REG-INC-01 — "Prazo de 7 dias para carga perigosa"

```yaml
- id: REG-INC-01
  query: "Qual o prazo de devolução para carga perigosa classe 3?"
  expected_contains: ["não elegível", "Gestão de Riscos", "ramal 4500"]
  forbidden_contains: ["7 dias úteis"]
  chunks_required:
    - scope: regra_geral (POL-001 §3.1)
    - scope: excecao (POL-001 §3.2)
  note: Verifica a lacuna L-01 — pipeline deve recuperar chunk de exceção junto com regra geral.
  guardrails: [DEVE-01, DEVE-04, NAO-01, NAO-02, NAO-05]
```

#### REG-INC-02 — "Multiplicadores da v1 com citação de v2"

```yaml
- id: REG-INC-02-A
  query: "Qual o multiplicador regional para o Centro-Oeste?"
  expected_state: CONTRADIÇÃO_DETECTADA
  forbidden_chunks: doc_id containing "PROC-042-v1"
  guardrails: [DEVE-02, DEVE-09, NAO-03, NAO-12]

- id: REG-INC-02-B
  query: "Qual o fator de peso para carga acima de 3.000kg?"
  assertion: response.sources[0].version == chunk_retrieved[0].doc_version
  note: Idêntico ao DET-03 — caso de incidente de integridade.
```

#### REG-INC-03 — "Falso GAP_DOCUMENTAL para SLA Gold"

```yaml
- id: REG-INC-03-A
  query: "Qual o SLA do Gold?"
  expected_state: FUNDAMENTADA
  forbidden_state: GAP_DOCUMENTAL
  guardrails: [DEVE-07, DEVE-11]

- id: REG-INC-03-B
  queries:
    - "qual o tempo de resposta para cliente Gold?"
    - "SLA de resolução Gold"
    - "quanto tempo tem para resolver um chamado Gold?"
  expected_all: FUNDAMENTADA
  note: Variações lexicais não devem gerar GAP — indica limiar alto demais.
```

**Gate 3:** nenhum incidente reaberto. Falha = deploy bloqueado com issue `regression:incident-reopened`.

---

### Camada 4 — Monitoramento pós-deploy

#### DEVE-17 (L-03) — Alerta de degradação de retrieval por domínio

```
Regra: se taxa de GAP_DOCUMENTAL para um domínio ultrapassar 15% das
consultas em janela de 7 dias corridos, emitir alerta.

Domínios monitorados:
  devolucao_mercadoria · frete_especial · sla_contrato ·
  carga_perigosa · carga_danificada · seguro_carga
```

#### Monitor de taxa de feedback negativo

```
Regra: se taxa de thumbs-down em respostas FUNDAMENTADA ultrapassar
5% em 7 dias, abrir revisão manual dos casos afetados.

Justificativa: respostas FUNDAMENTADA com feedback negativo indicam
conteúdo errado com estado correto — falha que o harness estático
pode não ter capturado.
```

#### Monitor de integridade de sources pós-deploy

```
Regra: nos primeiros 2h após cada deploy canário, amostrar 50 queries
randômicas e verificar a assertion DET-03.

Justificativa: INC-02 pode se manifestar silenciosamente sem acionar
nenhum guardrail de conteúdo.
```

### Matriz de cobertura — guardrails × camadas

| Guardrail | Enforcement | C1 | C2 | C3 | Observação |
|---|---|---|---|---|---|
| DEVE-01 | `[DET]` | DET-01 | GS-PROC-01 | REG-INC-01/02 | Estado em toda resposta |
| DEVE-02 | `[HBR]` | DET-02 | GS-PROC-01 | REG-INC-02-B | Fonte + versão + seção |
| DEVE-03 | `[HBR]` | — | GS-03-01/02 | — | FAQ sempre COM_RESSALVA |
| DEVE-04 | `[HBR]` | — | GS-04-01/02 | REG-INC-01/02 | Contradição detectada |
| DEVE-05 | `[DET]` | — | GS-05-01/02 | REG-INC-03 | GAP + encaminhamento |
| DEVE-06 | `[PRB]` | — | GS-06-01/02 | — | SLA duplo regime |
| DEVE-07 | `[DET]` | DET-05 | — | REG-INC-03-A | Normalização pré-embedding |
| DEVE-09 | `[DET]` | DET-04 | — | REG-INC-02-A | Filtro obsoleto |
| NAO-01 | `[HBR]` | — | GS-NAO-01-01 | REG-INC-01/02 | Sem síntese entre conflitantes |
| NAO-03 | `[DET]` | DET-04 | — | REG-INC-02-A | Bloqueio v1 no índice |
| L-01 (nova) | `[DET]` | — | — | REG-INC-01 | Busca obrigatória chunk exceção |
| L-02 (nova) | `[DET]` | DET-03 | — | REG-INC-02-B | sources.version == chunk.doc_id |
| L-03 (nova) | `[DET]` | — | — | C4 monitor | Alerta GAP > 15% por domínio |

### Resumo executivo

| Camada | Pergunta respondida | Gate | Responsável |
|---|---|---|---|
| 1 — Guardrails DET | "A estrutura da resposta está correta?" | 100% pass | Engenharia (CI automático) |
| 2 — Golden set | "As respostas corretas continuam corretas?" | 100% P0 · ≥90% total | Engenharia + Produto |
| 3 — Incidentes | "Nenhum problema conhecido foi reaberto?" | Zero regressões | Engenharia |
| 4 — Monitoramento | "Algo novo está degradando silenciosamente?" | Alertas proativos | Produto (ongoing) |

---

## IV. Human-in-the-loop

### Premissa de design

A política não classifica mudanças por "tamanho" ou "complexidade técnica" — classifica por consequência observável para o atendente e para o cliente.

Três eixos determinam o nível de aprovação:

| Eixo | Pergunta | Impacto no nível |
|---|---|---|
| Comportamento observável | A mudança altera o que o atendente vê na resposta? | Sobe Nível 0 → 1 |
| Reversibilidade | A mudança pode ser desfeita em menos de 30 min sem impacto residual? | Irreversível sobe 1 → 2 |
| Origem do conteúdo | A mudança toca informação com implicação jurídica, contratual ou regulatória? | Sobe diretamente para 2 |

---

### Nível 0 — Deploy automático via CI/CD

Mudanças que passam no harness completo e não alteram o comportamento observável. Nenhum humano precisa ser notificado antes do deploy.

| Mudança | Condição de elegibilidade |
|---|---|
| Ajuste de prompt — regras P1/P2 | Harness completo pass · não toca guardrails P0 |
| Atualização de metadado de confiança | Mudança de nível só com documento normativo aprovado como base |
| Reindexação parcial | Documento já aprovado no Nível 2 · apenas rechunking · conteúdo idêntico |
| Atualização de threshold de logging | Não altera comportamento de resposta |
| Correção de typo em prompt | Não altera semântica de nenhuma regra |

**Salvaguardas automáticas:**
- Rollback automático se monitor da Camada 4 detectar degradação em 2h.
- Log de versão automático com hash do prompt, documentos ativos, versão do golden set.
- Nenhuma regra P0 é elegível para Nível 0, independentemente do harness.

---

### Nível 1 — Aprovação: Tech Lead + Product Owner

Mudanças que alteram comportamento observável, mas cujo conteúdo é de responsabilidade da equipe técnica e de produto.

**Quem aprova:** Tech Lead (integridade técnica) + Product Owner (alinhamento de produto). Ambos via pull request. **SLA: 1 dia útil.**

| Mudança | Guardrails afetados | Por que exige Nível 1 |
|---|---|---|
| Ajuste de prompt — regras P0 | DEVE-03, DEVE-05, NAO-01 | Altera comportamento em cenários críticos |
| Alteração de limiar de similaridade | DEVE-05, DEVE-11 | Pode gerar/eliminar GAP_DOCUMENTAL (INC-03) |
| Atualização da tabela de sinônimos | DEVE-07 | Altera qual chunk é recuperado |
| Adição ou remoção de guardrail | Todos os afetados | Pode conflitar com comportamento existente |
| Remoção de guardrail existente | Todos os removidos | Irreversível — exige justificativa documentada |
| Mudança na lógica de detecção de contradição | DEVE-04, NAO-12 | Impacto alto no estado CONTRADIÇÃO_DETECTADA |
| Alteração do schema do campo `sources` | DEVE-02, L-02 | Pode quebrar assertion DET-03 |

**Checklist de aprovação — Nível 1:**

```
[ ] Harness completo executado (todas as 4 camadas)
[ ] Gate 1: 100% pass nos guardrails DET
[ ] Gate 2: 100% pass nos casos P0 do golden set
[ ] Gate 3: nenhum incidente reaberto (REG-INC-01/02/03)
[ ] Guardrails afetados listados explicitamente no PR
[ ] Comportamento anterior documentado (o que muda e por quê)
[ ] Plano de rollback definido (comando + tempo estimado)
[ ] Responsável pelo monitoramento pós-deploy identificado
```

---

### Nível 2 — Aprovação: Área Responsável + Compliance

Mudanças que tocam o conteúdo que o assistente usa como fonte de verdade. O conteúdo é de responsabilidade da área de negócio — não da equipe técnica.

**Quem aprova:** Responsável formal da área dona do documento + Compliance da NovaTech. Aprovação de conteúdo precede a aprovação técnica. **SLA: 5 dias úteis.** Urgências com risco regulatório ativo: 24h mediante justificativa do PO.

| Mudança | Área responsável | Compliance obrigatório |
|---|---|---|
| Adição de novo documento à base (primeiro ingresso) | Dono da área do documento | Sim, sempre |
| Deprecação de documento ativo (ex: arquivar PROC-042-v1) | Diretoria Comercial | Sim — impacto contratual |
| Formalização de gap documental como novo documento | Área responsável + Operações | Sim |
| Mudança no catálogo de gaps | Área responsável | Só se gap tiver implicação ANTT/jurídica |
| Atualização de documento existente (nova versão) | Dono formal do documento | Sim, se tocar carga perigosa, SLA ou penalidades |
| Alteração do mapeamento de encaminhamento (RN-10) | Operações | Sim — encaminhamento incorreto é risco operacional |

**Tabela de responsáveis por documento:**

| Documento | Área responsável | Compliance |
|---|---|---|
| POL-001 — Política de Devolução | Diretoria de Operações | Compliance |
| PROC-042 v1/v2 — Frete Especial | Diretoria Comercial | Compliance (contratos) |
| PROC-043 — Frete Carga Perigosa | Compliance + Operações | Compliance (ANTT) |
| SLA-2024 — Tabela de SLA | Comercial + Operações | Compliance (doc contratual) |
| FAQ-Atendimento | Gestor de Atendimento | Não — FAQ nunca vira FUNDAMENTADA sem Nível 2 |

**Checklist de aprovação — Nível 2:**

```
[ ] Documento assinado pelo responsável formal da área
[ ] Compliance validou: sem termos jurídicos novos não revisados
[ ] Conflito com documentos existentes mapeado
    (novo documento contradiz algum chunk já indexado?)
[ ] Se sim: decisão sobre deprecação ou coexistência documentada
[ ] Guardrail DEVE-04 precisa ser atualizado?
    (novo atributo de negócio = novo par topic_tag/attribute_key)
[ ] Guardrail DEVE-05 precisa ser atualizado?
    (novo domínio = novo encaminhamento na tabela RN-10)
[ ] Lacuna L-01 coberta?
    (novo documento tem regra_geral com exceções que precisam de
     chunk separado com scope:excecao?)
[ ] Após aprovação de conteúdo: submeter para aprovação técnica Nível 1
```

---

### Casos de borda

#### Caso 1 — Mudança Nível 0 descobre conflito durante harness

Se Gate 2 detectar que um caso P0 passou a falhar, a mudança é promovida automaticamente para Nível 1. Tech Lead decide: corrigir ou escalar para Nível 2 se o conflito tiver origem em conteúdo de negócio.

#### Caso 2 — Área solicita mudança urgente sem documento formal

```
1. Área identifica o problema
2. Produto abre ticket de urgência com evidência
3. Compliance confirma o risco em até 4h
4. Área produz nota técnica provisória como documento formal
5. Nota técnica passa por Nível 2 comprimido (24h)
6. Apenas após aprovação: engenharia executa harness e faz deploy
```

O assistente não pode ser corrigido diretamente via prompt sem aprovação de área — isso contornaria o Nível 2 e quebraria a rastreabilidade.

#### Caso 3 — Guardrail `[PRB]` falha no golden set mas DET passa

- Falha pontual (1 run): re-executar. Pass na segunda → prosseguir com Nível 1.
- Falha consistente (2+ runs): tratar como falha real. Investigar prompt antes de aprovar.

Guardrail `[PRB]` nunca pode ser aprovado "apesar da falha".

#### Caso 4 — FAQ recebe atualização informal da equipe

Qualquer informação adicionada ao FAQ continua sendo `COM_RESSALVA` (DEVE-03) independentemente do conteúdo. Para que vire `FUNDAMENTADA`, precisa ser formalizada como POL ou PROC e passar pelo Nível 2. Não existe atalho.

#### Caso 5 — Mudança toca L-01 (busca obrigatória de chunk de exceção)

L-01 ainda não tem implementação. Checklist obrigatório no Nível 1:

```
[ ] Documento novo tem seções de exceção?
    Se sim: chunks de exceção têm scope:excecao configurado?
    Se sim: pipeline executa busca secundária para esse topic_tag?
    Se L-01 não implementado: REG-INC-01 vai falhar — esperado.
    Aprovar com nota de risco residual documentada no PR.
```

---

### Matriz completa de mudanças × nível × aprovador × SLA

| Mudança | Nível | Aprovador | SLA |
|---|---|---|---|
| Ajuste de prompt P2 | 0 | CI/CD | Imediato |
| Ajuste de prompt P1 | 0 | CI/CD | Imediato |
| Atualização de metadado de confiança | 0 | CI/CD | Imediato |
| Reindexação parcial — doc aprovado | 0 | CI/CD | Imediato |
| Ajuste de prompt P0 | 1 | Tech Lead + PO | 1 dia útil |
| Alteração de limiar por domínio | 1 | Tech Lead + PO | 1 dia útil |
| Atualização de tabela de sinônimos | 1 | Tech Lead + PO | 1 dia útil |
| Adição ou remoção de guardrail | 1 | Tech Lead + PO | 1 dia útil |
| Mudança no schema do campo `sources` | 1 | Tech Lead + PO | 1 dia útil |
| Adição de novo documento à base | 2 | Área + Compliance | 5 dias úteis |
| Deprecação de documento ativo | 2 | Área + Compliance | 5 dias úteis |
| Formalização de gap como documento | 2 | Área + Compliance | 5 dias úteis |
| Atualização de doc com implicação jurídica/ANTT | 2 | Área + Compliance | 5 dias úteis |
| Mudança no mapeamento de encaminhamento (RN-10) | 2 | Operações + Compliance | 5 dias úteis |
| Mudança urgente com risco regulatório ativo | 2 comprimido | Área + Compliance | 24h |

---

### Rastreabilidade e auditoria

Toda aprovação de qualquer nível deve gerar registro com:

- ID da mudança · nível de aprovação · nome e cargo do aprovador · data e hora
- Resultado do harness (link para o run)
- Versão do `guardrails.md` vigente no momento do deploy
- Hash do prompt implantado · lista de documentos ativos no índice

Aprovações de Nível 2 incluem adicionalmente o documento de aprovação da área e o parecer de Compliance. Registros mantidos por no mínimo 2 anos para auditoria contratual e regulatória.

---

## Dependências entre os três componentes do harness

Os três componentes não são independentes — formam um sistema:

```
Processo de feedback
    → alimenta o golden set com novos casos reais
    → aciona ações corretivas que geram mudanças
    → toda mudança passa pelo regression testing
    → regression testing determina o nível de aprovação HITL
    → HITL define quem aprova antes do deploy canário
    → deploy canário alimenta o monitoramento da Camada 4
    → Camada 4 detecta novos problemas → volta ao feedback
```

O indicador de saúde do sistema inteiro é único: **a taxa de feedback negativo por tipo de resposta deve cair a cada sprint**. Se está estagnada, o loop está quebrado em algum ponto — e a matriz de aprovação é o primeiro lugar a verificar.

---

*Documento gerado em 22/06/2026 · NovaTech × DB1 Group · Confidencial*
