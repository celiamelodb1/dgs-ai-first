# Output — Sessão Tech Lead · Query Endpoint
**Projeto:** Assistente de IA NovaTech
**Insumo:** requirements.md v1.0.0
**Entregável:** requirements.md v1.1.0
**Data:** Junho 2026

---

## Prompt 11 — Revisão Técnica como Tech Lead

```
Atue como Tech Lead e aponte ambiguidades.
```

---

## OUTPUT 11 — Ambiguidades Identificadas no requirements.md v1.0.0

**Problema central diagnosticado:** o documento especificava comportamentos que dependem de
capacidades que o RAG não possui nativamente, sem explicitar os componentes adicionais
necessários. Um dev lendo a v1.0.0 assumiria que o sistema consegue fazer coisas que, na
prática, exigem engenharia não trivial.

### Ambiguidades Críticas — Bloqueantes de Implementação

**AMB-01 — Detecção de contradição sem mecanismo definido**
- Onde: ADR-004, Estado 3, AC-03, RN-01, E8
- Problema: o documento dizia "quando recuperar chunks conflitantes acionar
  CONTRADIÇÃO_DETECTADA", mas não definia o que é conflito computacionalmente.
  Por padrão, um pipeline RAG sintetiza — não detecta contradição.
- Perguntas sem resposta: a detecção é no retrieval (metadados) ou no LLM?
  Se LLM, como garantir que ele não "resolva" silenciosamente o conflito?
  Um chunk de Norte e outro de Sul no mesmo documento é contradição?

**AMB-02 — Threshold de GAP_DOCUMENTAL não definido**
- Onde: Estado 4, AC-05, AC-06, RN-04
- Problema: "nenhum chunk acima do limiar mínimo" — limiar não especificado.
  Score alto = falsos gaps; score baixo = falsos positivos. O ajuste é
  empírico e varia por domínio. O documento tratava como parâmetro único e estável.

**AMB-03 — Mapeamento de termos proibidos sem localização no pipeline**
- Onde: Edge Cases E1 e E3, seção Dependencies
- Problema: "frete diferenciado" → "Frete Especial" era mencionado, mas não
  definia onde no pipeline: pré-processamento, embedding, retrieval ou prompt do LLM.
  Cada opção tem implicações técnicas completamente diferentes.

**AMB-04 — Sessão sem definição técnica de storage e isolamento**
- Onde: RN-08, AC-07, UO-05
- Problema: "60 minutos de inatividade" sem definir: o que é inatividade?
  Onde o estado de sessão é armazenado? O que ocorre em reinício do serviço?
  Dois atendentes no mesmo tenant compartilham sessão?

### Ambiguidades Relevantes — Afetam Comportamento em Produção

**AMB-05 — AC-07 não verificável em teste automatizado**
- Onde: AC-07
- Problema: "resposta consistente com o histórico da sessão" não é assertável
  em código. Descreve coerência de linguagem natural — probabilística, não determinística.

**AMB-06 — SLA de latência cobre apenas 1 dos 4 estados, ignora concorrência**
- Onde: ADR-005, AC-01
- Problema: 8s definido só para FUNDAMENTADA. Estados com lógica adicional
  (CONTRADIÇÃO_DETECTADA) têm latência diferente. Sem requisito de concorrência
  para 45 atendentes simultâneos.

**AMB-07 — Exceção da RN-01 não é implementável sem integração fora do escopo**
- Onde: RN-01
- Problema: exceção para "chamados abertos antes de 01/12/2023" exigiria
  identificar a data do chamado — o que depende do Azure DevOps,
  explicitamente fora do escopo na seção 3.2. Via LLM = risco de alucinação.

**AMB-08 — Tabela de encaminhamento do GAP_DOCUMENTAL sem fonte ou owner**
- Onde: Estado 4, AC-05
- Problema: "encaminhamento baseado na categoria da pergunta" — a tabela de
  categoria → encaminhamento não existia. E-mail hardcoded no prompt do LLM.
  E-mails mudam; prompt não tem PR.

**AMB-09 — AC-03 e AC-04 mutuamente exclusivos**
- Onde: AC-03, AC-04
- Problema: AC-04 impede que chunks do PROC-042 v1 sejam recuperados.
  AC-03 usava exatamente o conflito v1×v2 como exemplo de CONTRADIÇÃO_DETECTADA.
  Um dev que implementar AC-04 corretamente torna AC-03 irreproduzível em teste.

**AMB-10 — Schema do request não definido; tier sem campo especificado**
- Onde: RN-06, AC-08, AC-09, OQ-05
- Problema: o contrato da API não estava em lugar nenhum do documento.
  O campo `tier` não tinha nome, tipo, obrigatoriedade nem origem.
  O primeiro dev a tocar o código definiria o schema sozinho.

### Tabela Consolidada de Ambiguidades

| # | Ambiguidade | Severidade | Bloqueia |
|---|---|---|---|
| AMB-01 | Mecanismo de detecção de contradição indefinido | 🔴 Crítica | AC-03, ADR-004 |
| AMB-02 | Threshold de GAP_DOCUMENTAL não definido | 🔴 Crítica | AC-05, AC-06 |
| AMB-03 | Localização do mapeamento de termos proibidos no pipeline | 🔴 Crítica | E1, E3 |
| AMB-04 | Sessão sem storage, escopo de isolamento e comportamento em falha | 🔴 Crítica | AC-07, RN-08 |
| AMB-05 | AC-07 não verificável em teste automatizado | 🟡 Alta | AC-07 |
| AMB-06 | Latência e concorrência subespecificadas | 🟡 Alta | AC-01, NFR |
| AMB-07 | Exceção RN-01 não implementável sem integração fora do escopo | 🟡 Alta | RN-01 |
| AMB-08 | Encaminhamento GAP_DOCUMENTAL sem tabela, fonte ou owner | 🟡 Alta | Estado 4 |
| AMB-09 | AC-03 e AC-04 mutuamente exclusivos no exemplo | 🟡 Alta | AC-03, AC-04 |
| AMB-10 | Schema de request/response e campo tier indefinidos | 🟡 Alta | RN-06, AC-08, AC-09 |

**Recomendação do Tech Lead:** AMB-01 a AMB-04 requerem sessão de design com engenharia
antes do Sprint 1 — são decisões de arquitetura, não detalhes de implementação.
AMB-09 pode ser resolvida no documento sem alinhamento externo.

---

## Prompt 12 — Ajuste das Ambiguidades

```
Faça o ajuste das ambiguidades encontradas.
```

---

## OUTPUT 12 — requirements.md v1.1.0 · Mapa de Mudanças

### Changelog aplicado

| AMB | Seção alterada | O que era (v1.0.0) | O que virou (v1.1.0) |
|---|---|---|---|
| AMB-01 | §2 Prior Decisions | ADR-004 sem mecanismo de detecção | ADR-006 adicionada: detecção determinística por `topic_tag` + `attribute_key` nos metadados do chunk. LLM fora da decisão de detecção. |
| AMB-02 | §5 Estado 4 | "limiar mínimo" sem valor | Tabela de limiares por domínio (0,70–0,78), configuráveis, com responsável e cadência de calibração pós go-live |
| AMB-03 | §6 RN-09 (nova) | "glossário no pré-processamento" como dependência vaga | RN-09: tabela de sinônimos versionada no repositório, responsável DB1, atualização via PR sem redeploy |
| AMB-04 | §6 RN-08 | "60 minutos de inatividade" sem definição técnica | RN-08 revisada: Redis TTL 60min, isolamento por `user_id` AAD, mensagem de erro em reinício explicitada |
| AMB-05 | §8 AC-07 | "retorna resposta consistente com o histórico" (não verificável) | AC-07 reescrito: verifica `session_history` no log, `doc_id` igual nas duas respostas, `session_id` igual — tudo assertável |
| AMB-06 | §7 RNF (nova seção) | Latência de 8s só para FUNDAMENTADA, sem concorrência | RNF-01: tabela de latência p95 por estado + comportamento em timeout. RNF-02: 45 req simultâneas. RNF-03: 99% disponibilidade em horário comercial |
| AMB-07 | §6 RN-01 | Exceção para chamados pré-01/12/2023 via inferência LLM | Exceção removida com justificativa: integração CRM fora do escopo + risco de alucinação > custo de tratamento manual |
| AMB-08 | §6 RN-10 (nova) | Encaminhamento hardcoded no prompt | RN-10: tabela de roteamento como configuração versionada no repositório. E-mail como variável de ambiente, não hardcoded |
| AMB-09 | §8 AC-03 | Exemplo usava conflito v1×v2 — irreproduzível com AC-04 ativo | AC-03 reescrito: exemplo trocado para POL-001 × FAQ (dois docs vigentes). Nota explicativa sobre por que v1×v2 não se aplica |
| AMB-10 | §8 AC-01 + §12 OQ-06 | Schema de request/response indefinido | AC-01 agora contém schemas JSON completos com tipos, obrigatoriedade e campo `tier_hint`. OQ-06 adicionada sobre origem do tier |

### Novas regras de negócio adicionadas

| Regra | Conteúdo |
|---|---|
| RN-09 | Classificador de intenção + mapeamento de termos canônicos no pré-processamento. Tabela de sinônimos versionada. Classificação de domínio por palavras-chave. |
| RN-10 | Tabela de encaminhamento para GAP_DOCUMENTAL por domínio. Configuração externalizada. E-mail como variável de ambiente. |

### Nova seção adicionada

**§7 Requisitos Não-Funcionais**
- RNF-01: latência máxima por estado (p95 e absoluto), comportamento em timeout, medição end-to-end no serviço
- RNF-02: concorrência mínima de 45 requisições simultâneas, baseline e pico estimados, obrigatoriedade de teste de carga pré go-live
- RNF-03: disponibilidade 99,0% em horário comercial para o MVP

### Estrutura de seções final (v1.1.0)

| # | Seção | Status |
|---|---|---|
| 1 | Overview | Sem alteração |
| 2 | Prior Decisions | ADR-006 adicionada (AMB-01) |
| 3 | Scope Boundaries | Sem alteração |
| 4 | User Outcomes | Sem alteração |
| 5 | Estados de Resposta | Estado 4 revisado (AMB-02) |
| 6 | Regras de Negócio | RN-01 revisada (AMB-07) · RN-08 revisada (AMB-04) · RN-09 nova (AMB-03) · RN-10 nova (AMB-08) |
| 7 | **Requisitos Não-Funcionais** | **Seção nova** (AMB-06) |
| 8 | Acceptance Criteria | AC-01 com schema (AMB-10) · AC-03 corrigido (AMB-09) · AC-07 reformulado (AMB-05) |
| 9 | Edge Cases | Sem alteração |
| 10 | Dependencies | Sem alteração |
| 11 | Out of Scope | Sem alteração |
| 12 | Open Questions | OQ-06 adicionada (AMB-10) |

### Schema de request/response definido (v1.1.0)

**Request:**
```json
{
  "query":      "string — obrigatório, max 1000 chars",
  "session_id": "string — UUID; ausente = nova sessão criada",
  "user_id":    "string — AAD object ID do atendente (obrigatório)",
  "tier_hint":  "enum[Gold, Silver, Standard] — opcional (ver OQ-05, OQ-06)"
}
```

**Response:**
```json
{
  "state":              "enum[FUNDAMENTADA, COM_RESSALVA, CONTRADIÇÃO_DETECTADA, GAP_DOCUMENTAL]",
  "answer":             "string",
  "session_id":         "string — UUID da sessão",
  "confidence_level":   "enum[normativo, contratual, informal] — presente em FUNDAMENTADA e COM_RESSALVA",
  "disclaimer":         "string — presente em COM_RESSALVA",
  "sources":            "[{ doc_id, version, section, chunk_id }]",
  "conflicting_docs":   "[string] — presente em CONTRADIÇÃO_DETECTADA",
  "routing_suggestion": "string — presente em GAP_DOCUMENTAL",
  "domain_classified":  "enum[frete_logistica, devolucao_politica, sla_contrato, nao_classificado]"
}
```

### Open Questions em aberto após v1.1.0

| # | Questão | AC afetado | Prazo |
|---|---|---|---|
| OQ-01 | FAQ será ingerido? Com qual peso? | AC-02, E6 | Sprint 1 |
| OQ-02 | Comportamento para seguro de carga (FAQ Item 22)? | AC-05 (escopo) | Sprint 1 |
| OQ-03 | NovaTech formalizará obsolescência do PROC-042 v1 antes do go-live? | AC-03, AC-04 | Sprint 2 |
| OQ-04 | Encaminhamento automático ao supervisor ou apenas orientação textual? | AC-03 | Sprint 2 |
| OQ-05 | Atendente informa o tier ou endpoint infere? | AC-08, AC-09 | Sprint 1 |
| OQ-06 | `tier_hint` preenchido pelo bot (CRM) ou digitado pelo atendente? | AC-08, AC-09, RN-06 | Sprint 1 |
