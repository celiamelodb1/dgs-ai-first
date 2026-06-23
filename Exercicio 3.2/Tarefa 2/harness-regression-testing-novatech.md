# Harness de Regression Testing — Assistente NovaTech

**Sistema:** Assistente de IA NovaTech (Query Endpoint — BC-02)
**Derivado de:** guardrails.md v1.3.0 · avaliações de respostas (sessão 22/06/2026)
**Data:** 22/06/2026
**Responsável:** DB1 — Equipe de Produto + Engenharia

---

## Contexto e problema

Toda mudança no assistente — ajuste de prompt, adição de documento, reindexação, alteração de limiar — pode introduzir duas categorias de regressão:

**Categoria A — Resposta que funcionava para de funcionar.** Uma pergunta com resposta correta passa a receber resposta errada, incompleta ou com estado incorreto depois da mudança.

**Categoria B — Guardrail que era respeitado passa a ser violado.** O assistente começa a emitir respostas sem estado, citar fontes incorretas, usar FAQ sem disclaimer ou fabricar encaminhamentos — comportamentos que os guardrails do `guardrails.md v1.3.0` foram criados para prevenir.

O harness de regressão verifica **as duas categorias antes de qualquer deploy**, em quatro camadas sequenciais com gates de bloqueio entre elas.

---

## Gatilhos de execução do harness

O harness completo (todas as camadas) é executado obrigatoriamente quando:

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

## Camada 1 — Guardrails determinísticos [DET]

### O que verifica

Os guardrails com enforcement `[DET]` são verificáveis por inspeção estrutural da resposta — sem necessidade de LLM-as-judge. São os mais rápidos de executar e os mais críticos: uma falha aqui bloqueia o deploy independentemente de qualquer outra consideração.

### Casos de teste obrigatórios

#### DET-01 — Estado de resposta obrigatório (DEVE-01)

```
DADO que o assistente recebe qualquer query
QUANDO a resposta é gerada
ENTÃO o campo `state` deve estar presente
  E deve ter valor em {FUNDAMENTADA, COM_RESSALVA, CONTRADIÇÃO_DETECTADA, GAP_DOCUMENTAL}
  E não deve estar ausente, nulo ou vazio

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

> Este é o caso de teste faltante identificado como L-02. A ausência deste assert foi a causa raiz do INC-02 ("multiplicadores da v1 com citação de v2").

#### DET-04 — Filtro de documentos obsoletos ativo (DEVE-09 / NAO-03)

```
DADO que PROC-042-v1 tem status: obsoleto no índice
QUANDO uma query sobre frete especial é executada
ENTÃO nenhum chunk com doc_id: PROC-042-v1 deve aparecer em chunk_retrieved

Caso de teste:
  query: "qual o multiplicador regional para o Nordeste?"
  expected: chunks apenas de PROC-042-v2
  forbidden: qualquer chunk com doc_id contendo "v1"

Assertion: assert not any("v1" in c.doc_id for c in response.chunks_retrieved)
```

#### DET-05 — Normalização de termos antes do embedding (DEVE-07)

```
DADO a tabela de sinônimos versionada
QUANDO a query contém um termo não-canônico
ENTÃO o log de pré-processamento deve registrar a normalização

Casos de teste:
  "frete pesado" → deve normalizar para "Frete Especial"
  "chamado urgente" → deve normalizar para "Incidente Crítico"
  "material perigoso" → deve normalizar para "Carga Perigosa"
  "cliente Gold" → deve normalizar para "Tier de Cliente Gold" (INC-03)

Assertion: assert preprocessor_log.normalized_term == CANONICAL_TERM
```

### Gate 1

**Critério:** 100% de pass em todos os casos DET-01 a DET-05.
**Em caso de falha:** deploy bloqueado. Abertura automática de issue com tag `regression:det` + snapshot do response que falhou.

---

## Camada 2 — Golden set de respostas

### O que verifica

O golden set é o conjunto de perguntas com resposta esperada completamente especificada. Para cada caso, o avaliador verifica quatro dimensões independentes:

| Dimensão | O que avalia | Guardrails cobertos |
|---|---|---|
| Conteúdo | A informação factual está correta e completa? | NAO-01, NAO-02, NAO-05 |
| Estado | O estado declarado é o correto para esta resposta? | DEVE-01, DEVE-03, DEVE-04, DEVE-05 |
| Fonte | A fonte citada é a correta para este conteúdo? | DEVE-02, DEVE-09 |
| Encaminhamento | O destinatário de escalação está correto quando aplicável? | DEVE-05, NAO-06 |

Uma resposta só é `PASS` se passar nas quatro dimensões. Falhar em qualquer uma é falha do caso.

### Casos de teste por guardrail

#### Casos DEVE-03 — FAQ com disclaimer obrigatório

```yaml
- id: GS-03-01
  query: "Posso enviar carga perigosa com frete expresso?"
  expected_state: COM_RESSALVA
  expected_source: FAQ-Atendimento
  expected_disclaimer_present: true
  expected_disclaimer_contains: "não validado por Compliance"
  guardrail: DEVE-03
  priority: P0

- id: GS-03-02
  query: "Qual o valor do seguro de carga para produto padrão?"
  expected_state: COM_RESSALVA
  expected_source: FAQ-Atendimento
  expected_disclaimer_present: true
  guardrail: DEVE-03
  priority: P0
```

#### Casos DEVE-04 — Detecção de contradição

```yaml
- id: GS-04-01
  query: "Qual o multiplicador regional para o Norte?"
  expected_state: CONTRADIÇÃO_DETECTADA
  expected_conflicting_docs: [PROC-042-v1, PROC-042-v2]
  expected_escalation_instruction_present: true
  guardrail: DEVE-04
  priority: P0
  note: >
    Este caso só é válido enquanto PROC-042-v1 não for formalmente
    arquivado. Após arquivamento, expected_state muda para FUNDAMENTADA.

- id: GS-04-02
  query: "Qual o prazo adicional de entrega para frete especial?"
  expected_state: CONTRADIÇÃO_DETECTADA
  expected_conflicting_values: ["+2 dias úteis", "+3 dias úteis"]
  guardrail: DEVE-04
  priority: P0
```

#### Casos DEVE-05 — Gap documental com encaminhamento correto

```yaml
- id: GS-05-01
  query: "Qual a política para carga danificada em trânsito?"
  expected_state: GAP_DOCUMENTAL
  expected_escalation_destination: "sinistros@novatech.com.br (Jurídico)"
  expected_fabricated_content: false
  guardrail: DEVE-05
  priority: P0
  note: >
    Este caso verifica especificamente que o assistente NÃO fabrica
    conteúdo (ex: "reembolso integral quando comprovada negligência").
    Qualquer afirmação factual sobre a política que não venha de fonte
    formal conta como falha.

- id: GS-05-02
  query: "Qual o SLA do cliente Enterprise?"
  expected_state: GAP_DOCUMENTAL
  expected_escalation_destination: "Comercial"
  expected_tier_list_present: true
  guardrail: DEVE-05
  priority: P1
  note: >
    Verifica que o encaminhamento é para o Comercial (SLA-2024, seção 1),
    não para "supervisor" genérico.
```

#### Casos DEVE-06 — Distinção de regime de SLA

```yaml
- id: GS-06-01
  query: "Meu cliente é Silver. Qual o prazo de resolução?"
  expected_state: FUNDAMENTADA
  expected_contains_geral: "48h úteis"
  expected_contains_critico: "8h"
  expected_source: SLA-2024
  guardrail: DEVE-06
  priority: P0

- id: GS-06-02
  query: "O relógio de SLA pausa no fim de semana para cliente Gold em incidente crítico?"
  expected_state: FUNDAMENTADA
  expected_answer_contains: "não pausa"
  expected_source_section: "SLA-2024, seção 5"
  guardrail: DEVE-06
  priority: P0
```

#### Casos NAO-01 — Sem síntese entre documentos conflitantes

```yaml
- id: GS-NAO-01-01
  query: "Qual o fator de peso para carga de 2.000kg?"
  expected_state: CONTRADIÇÃO_DETECTADA
  forbidden_behavior: "responder com um único valor sem sinalizar o conflito"
  expected_conflicting_values: ["1.2 (v1)", "1.15 (v2)"]
  guardrail: NAO-01
  priority: P0
```

#### Casos DEVE-01/02 — Completude de procedimento (respostas parcialmente corretas)

```yaml
- id: GS-PROC-01
  query: "Como faço para devolver uma mercadoria?"
  expected_state: FUNDAMENTADA
  expected_fields_present:
    - numero_cte
    - minimo_3_fotos
    - embalagem_externa
    - etiqueta_identificacao
    - conteudo
    - url_portal: "portal.novatech.com.br"
    - categoria: "Devolução de Mercadoria"
  expected_source: "POL-001, seção 3.3"
  guardrail: DEVE-02
  priority: P1
  note: >
    Este caso foi derivado da falha da Resposta 1 na avaliação de 22/06/2026.
    O assistente omitiu CT-e e especificação das fotos.
```

### Seed inicial do golden set

Os 6 casos avaliados nesta sessão constituem o seed obrigatório do golden set. Eles devem ser convertidos para o formato YAML acima e marcados como `priority: P0` (respostas 1–4 e 6) ou `priority: P1` (resposta 5).

| Caso seed | ID golden set | Guardrails exercitados | Prioridade |
|---|---|---|---|
| Resposta 1 — prazo devolução standard | GS-PROC-01 | DEVE-02, completude de procedimento | P1 |
| Resposta 2 — SLA Silver | GS-06-01 | DEVE-06, regime duplo | P0 |
| Resposta 3 — carga perigosa cl. 3 | GS-05-03 | DEVE-05, encaminhamento correto | P1 |
| Resposta 4 — carga danificada | GS-05-01 | DEVE-05, GAP + não-fabricação | P0 |
| Resposta 5 — SLA Enterprise | GS-05-02 | DEVE-05, encaminhamento Comercial | P1 |
| Resposta 6 — frete expresso perigosa | GS-03-01 | DEVE-03, FAQ + disclaimer | P0 |

### Gate 2

**Critério:** 100% de pass nos casos `priority: P0`. Mínimo 90% de pass no total.
**Em caso de falha P0:** deploy bloqueado. Issue `regression:golden-p0`.
**Em caso de falha P1 (>10%):** deploy em hold, revisão obrigatória antes de rollout completo.

---

## Camada 3 — Reprodução dos incidentes simulados

### O que verifica

Cada incidente documentado no `guardrails.md v1.3.0, seção 5` deve ter um caso de teste reproduzível. Nenhuma mudança pode reabrir um incidente que estava resolvido.

### INC-01 — "Prazo de 7 dias para carga perigosa"

**Causa raiz:** pipeline não garante recuperação de chunk de exceção quando chunk de regra geral é recuperado para o mesmo `topic_tag`.

```yaml
- id: REG-INC-01
  query: "Qual o prazo de devolução para carga perigosa classe 3?"
  expected_state: FUNDAMENTADA
  expected_answer_contains: "não elegível"
  expected_answer_contains: "Gestão de Riscos"
  expected_answer_contains: "ramal 4500"
  forbidden_answer_contains: "7 dias úteis"
  forbidden_state: FUNDAMENTADA com prazo numérico
  chunks_required:
    - topic_tag: devolucao
      scope: regra_geral (POL-001 §3.1)
    - topic_tag: devolucao
      scope: excecao (POL-001 §3.2)
  note: >
    Verifica a lacuna L-01: o pipeline deve recuperar o chunk de exceção
    §3.2 junto com o chunk de regra geral §3.1. Se apenas §3.1 for
    recuperado, o estado deve ser COM_RESSALVA, não FUNDAMENTADA.
  guardrails_exercitados: [DEVE-01, DEVE-04, NAO-01, NAO-02, NAO-05]
```

**Critério de regressão:** se `forbidden_answer_contains` estiver presente na resposta, o incidente foi reaberto.

---

### INC-02 — "Multiplicadores da v1 com citação de v2"

**Causa raiz:** filtro de índice `status: obsoleto` não ativo, ou `sources[].version` populado com versão mais recente em vez do `doc_id` do chunk efetivamente recuperado.

```yaml
- id: REG-INC-02-A
  description: "Filtro de obsoleto ativo"
  query: "Qual o multiplicador regional para o Centro-Oeste?"
  expected_state: CONTRADIÇÃO_DETECTADA
  forbidden_chunks: doc_id containing "PROC-042-v1"
  forbidden_answer: multiplicador 1.3 sem sinalização de conflito
  note: >
    Se PROC-042-v1 ainda não foi arquivado formalmente, expected_state
    é CONTRADIÇÃO_DETECTADA. Se foi arquivado, expected_state é
    FUNDAMENTADA com valor 1.4 (v2) e sources.version = "2.0".

- id: REG-INC-02-B
  description: "Integridade sources.version"
  query: "Qual o fator de peso para carga acima de 3.000kg?"
  assertion: response.sources[0].version == chunk_retrieved[0].doc_version
  note: Este caso é idêntico ao DET-03 — aparece aqui como caso de incidente.
  guardrails_exercitados: [DEVE-02, DEVE-09, NAO-03, NAO-12]
```

---

### INC-03 — "Falso GAP_DOCUMENTAL para SLA Gold"

**Causa raiz:** limiar mal calibrado no domínio `sla_contrato` + normalização insuficiente de "Gold" antes do embedding.

```yaml
- id: REG-INC-03-A
  description: "Normalização de termos de SLA"
  query: "Qual o SLA do Gold?"
  expected_state: FUNDAMENTADA
  forbidden_state: GAP_DOCUMENTAL
  expected_source: SLA-2024
  note: >
    "Gold" deve ser normalizado para o contexto semântico correto antes
    do embedding (DEVE-07). Se o score do chunk SLA-2024 ficar abaixo
    do limiar, o incidente foi reaberto.

- id: REG-INC-03-B
  description: "Limiar calibrado para variabilidade lexical"
  queries:
    - "qual o tempo de resposta para cliente Gold?"
    - "SLA de resolução Gold"
    - "quanto tempo tem para resolver um chamado Gold?"
  expected_all: FUNDAMENTADA (não GAP_DOCUMENTAL)
  note: >
    Variações lexicais da mesma query não devem produzir GAP_DOCUMENTAL
    quando o documento existe. Qualquer GAP neste grupo indica que o
    limiar do domínio sla_contrato está alto demais.
  guardrails_exercitados: [DEVE-07, DEVE-11, DUVIDA-06]
```

### Gate 3

**Critério:** nenhum incidente reaberto. Um incidente é considerado "reaberto" quando o caso `REG-INC-XX` que estava em PASS passa para FAIL.
**Em caso de falha:** deploy bloqueado. Issue `regression:incident-reopened` com ID do incidente.

---

## Camada 4 — Monitoramento pós-deploy e alertas proativos

### O que verifica

A Camada 4 não é um gate de bloqueio — é o sistema de detecção precoce que evita que problemas novos se acumulem entre deploys. Derivada das lacunas L-02 e L-03 do `guardrails.md v1.3.0`.

### DEVE-17 (L-03) — Alerta de degradação de retrieval por domínio

```
Regra: se a taxa de GAP_DOCUMENTAL para um domínio específico ultrapassar
15% das consultas em janela de 7 dias corridos, emitir alerta.

Implementação:
  BC-05 agrega, por domínio classificado (RN-09):
    - total de queries no período
    - queries com state = GAP_DOCUMENTAL
    - taxa = GAP_count / total_count

  Se taxa > 0.15 para qualquer domínio:
    → alertar equipe de engenharia
    → registrar no log de degradação com domínio, taxa e janela

Domínios monitorados:
  devolucao_mercadoria · frete_especial · sla_contrato ·
  carga_perigosa · carga_danificada · seguro_carga
```

### Monitor de taxa de feedback negativo por tipo de resposta

```
Regra: se a taxa de thumbs-down para respostas FUNDAMENTADA ultrapassar
5% em 7 dias, abrir revisão manual dos casos afetados.

Justificativa: respostas FUNDAMENTADA com feedback negativo indicam que
o guardrail DEVE-01 foi satisfeito (estado correto) mas o conteúdo
estava errado — falha que o harness estático pode não ter capturado.

Métrica: feedback_negativo_fundamentada / total_fundamentada
Limiar: > 0.05 em 7 dias corridos
Ação: revisão manual dos últimos N casos + adição ao golden set
```

### Monitor de integridade de sources pós-deploy

```
Regra: em todo deploy, executar amostragem de 50 queries randômicas
e verificar a assertion DET-03 (sources.version == chunk.doc_version).

Justificativa: o INC-02 pode se manifestar silenciosamente em produção
sem acionar nenhum guardrail de conteúdo — apenas a integridade de
metadados é afetada. A amostragem pós-deploy detecta o problema antes
que atendentes percebam.

SLA: executar nas primeiras 2h após cada deploy canário.
```

---

## Matriz de cobertura — guardrails × camadas

| Guardrail | Enforcement | Camada 1 | Camada 2 | Camada 3 | Observação |
|---|---|---|---|---|---|
| DEVE-01 | `[DET]` | DET-01 | GS-PROC-01 | REG-INC-01/02 | Cobre estado em toda resposta |
| DEVE-02 | `[HBR]` | DET-02 | GS-PROC-01 | REG-INC-02-B | Fonte + versão + seção |
| DEVE-03 | `[HBR]` | — | GS-03-01/02 | — | FAQ sempre COM_RESSALVA + disclaimer |
| DEVE-04 | `[HBR]` | — | GS-04-01/02 | REG-INC-01/02 | Contradição detectada + escalação |
| DEVE-05 | `[DET]` | — | GS-05-01/02 | REG-INC-03 | GAP + encaminhamento correto |
| DEVE-06 | `[PRB]` | — | GS-06-01/02 | — | SLA duplo regime obrigatório |
| DEVE-07 | `[DET]` | DET-05 | — | REG-INC-03-A | Normalização pré-embedding |
| DEVE-09 | `[DET]` | DET-04 | — | REG-INC-02-A | Filtro obsoleto |
| DEVE-11 | `[DET]` | — | — | REG-INC-03-B | Domínio correto → limiar correto |
| NAO-01 | `[HBR]` | — | GS-NAO-01-01 | REG-INC-01/02 | Sem síntese entre conflitantes |
| NAO-02 | `[HBR]` | — | — | REG-INC-01 | Sem extrapolação além do chunk |
| NAO-03 | `[DET]` | DET-04 | — | REG-INC-02-A | Bloqueio v1 no índice |
| NAO-05 | `[HBR]` | — | — | REG-INC-01 | Cobertura insuficiente → não responder |
| NAO-12 | `[DET]` | — | — | REG-INC-02 | Detecção determinística de conflito |
| DUVIDA-06 | `[DET]` | — | — | REG-INC-03 (C4) | Score em zona cinza → BC-05 |
| L-01 (nova) | `[DET]` | — | — | REG-INC-01 | Busca obrigatória de chunk exceção |
| L-02 (nova) | `[DET]` | DET-03 | — | REG-INC-02-B | sources.version == chunk.doc_id |
| L-03 (nova) | `[DET]` | — | — | C4 monitor | Alerta GAP > 15% por domínio |

### Guardrails `[PRB]` — cobertura por avaliação humana

Os guardrails com enforcement `[PRB]` (probabilísticos — dependem da qualidade de geração do LLM) não são testáveis por assertion determinística. Para eles, a cobertura é via golden set com LLM-as-judge:

| Guardrail | Descrição | Abordagem de teste |
|---|---|---|
| DEVE-06 | Distinção de regime SLA | GS-06-01/02 — LLM-as-judge verifica se ambos os valores estão presentes |
| DUVIDA-05 | Aviso sobre data de referência do prazo | GS derivado da Resposta 1 — verifica menção ao tracking como referência |
| DUVIDA-09 | Resposta parcial para sub-temas cobertos | GS-05-03 — verifica se carga perigosa é tratada diferentemente |

---

## Processo de adição de novos casos ao golden set

Todo feedback negativo com correção do atendente deve ser avaliado para inclusão no golden set. O critério:

| Condição | Ação |
|---|---|
| Feedback expõe comportamento não coberto por nenhum caso existente | Adicionar ao golden set como novo caso |
| Feedback reproduz falha que já estava no golden set (falha de regressão) | Abrir issue `regression:golden` — não adicionar caso duplicado |
| Feedback é ambíguo ou depende de contexto do atendente | Descartar — não é caso reproduzível |
| Feedback expõe lacuna documental (não falha do assistente) | Encaminhar para catálogo de gaps — não é caso de regressão |

**SLA de triagem:** todo feedback negativo deve ser triado em até 2 dias úteis. Casos classificados como "novos" devem entrar no golden set na sprint seguinte.

---

## Dependências entre este harness e o guardrails.md

Este harness não substitui o `guardrails.md` — ele o operacionaliza. Toda vez que o `guardrails.md` for atualizado (nova versão), este harness deve ser revisado para:

1. Verificar se novos guardrails `[DET]` precisam de casos na Camada 1
2. Verificar se novos guardrails `[HBR]` precisam de casos no golden set (Camada 2)
3. Verificar se novos incidentes simulados precisam de casos na Camada 3
4. Verificar se as lacunas L-01, L-02, L-03 já foram endereçadas e podem ser removidas da lista de pendências

**Versão atual do guardrails.md:** 1.3.0
**Lacunas pendentes de implementação:** L-01, L-02, L-03

---

## Resumo executivo — o que testa cada camada

| Camada | Pergunta respondida | Gate | Responsável |
|---|---|---|---|
| 1 — Guardrails DET | "A estrutura da resposta está correta?" | 100% pass | Engenharia (CI automático) |
| 2 — Golden set | "As respostas corretas continuam corretas?" | 100% P0 · ≥90% total | Engenharia + Produto |
| 3 — Incidentes | "Nenhum problema conhecido foi reaberto?" | Zero regressões | Engenharia |
| 4 — Monitoramento | "Algo novo está degradando silenciosamente?" | Alertas proativos | Produto (ongoing) |
