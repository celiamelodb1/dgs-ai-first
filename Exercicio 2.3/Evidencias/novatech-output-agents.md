# Output — Sessão AGENTS.md · NovaTech Assistant
**Projeto:** Assistente de IA NovaTech
**Insumos:** guardrails.md v1.3.0 · requirements.md v1.1.0 · Anexo A · Anexo C
**Entregável:** `AGENTS-product-rules.md` (seções do AGENTS.md)
**Data:** Junho 2026

---

## OUTPUT 19 — Seção "Product Rules & Guardrails"

**Prompt:** Com base no Anexo A, Anexo C e guardrails v1.3.0, escreva a seção
"Product Rules & Guardrails" do AGENTS.md com regras de comportamento do assistente.

### Decisões de escrita

**Voz de instrução direta ao agente, não narrativa.** Cada regra começa com o
comportamento esperado. "Nunca", "Sempre", "Ao implementar X, faça Y" — não
"esta regra existe porque".

**Código inline nas regras críticas.** PG-01 (tipo `ResponseState`), PG-02 (filtro
`status ne 'obsoleto'`), PG-03 (assinatura de `detectConflict`), PG-04 (constante
`DISCLAIMER_INFORMAL`), PG-05 (`RETRIEVAL_THRESHOLDS`) e PG-07 (chave de sessão
Redis) têm snippets TypeScript prontos para uso.

**Rastreabilidade dupla: guardrail + incidente.** Cada regra referencia o guardrail
de origem (`DEVE-XX`, `NAO-XX`) e, quando aplicável, o incidente simulado que a
motivou (INC-01, INC-02, INC-03).

**Ponteiros para os arquivos corretos do repositório.** Cada regra aponta para o
arquivo exato em `src/` onde a implementação deve ocorrer.

**PG-17 como meta-regra de governança.** Define o protocolo de versionamento:
AGENTS.md → guardrails.md → system-prompt.md → prompt-changelog.md → eval.

### Conteúdo produzido — 17 regras (PG-01 a PG-17)

| ID | Título | Enforcement | Origem |
|---|---|---|---|
| PG-01 | Todo response tem um estado explícito | `[DET]` | `DEVE-01` · §5 Estados |
| PG-02 | PROC-042 v1 é obsoleto — filtro de índice obrigatório | `[DET]` | `DEVE-09` · `NAO-03` · INC-02 |
| PG-03 | Contradição detectada por metadados, não pelo LLM | `[DET]` | `ADR-006` · `DEVE-04` · `NAO-12` |
| PG-04 | FAQ requer COM_RESSALVA e disclaimer injetado em código | `[HBR]` | `DEVE-03` · `NAO-04` · RN-02 |
| PG-05 | GAP_DOCUMENTAL por limiar configurável por domínio | `[DET]` | `DEVE-05` · RN-09 |
| PG-06 | Termos não-canônicos normalizados antes do embedding | `[DET]` | `DEVE-07` · RN-09 |
| PG-07 | Sessão isolada por user_id AAD no Redis | `[DET]` | `DEVE-08` · RN-08 |
| PG-08 | O assistente não calcula, não mede e não executa | `[DET]`/`[PRB]` | `NAO-06` · `NAO-08` · `NAO-10` |
| PG-09 | SLA exige distinção: chamado geral vs incidente crítico Gold | `[PRB]` | `DEVE-06` · RN-06 |
| PG-10 | Encaminhamentos GAP_DOCUMENTAL em configuração, não no prompt | `[DET]` | `DEVE-15` · RN-10 |
| PG-11 | Carga perigosa: escopo restrito a classes 1-6 ANTT | `[HBR]` | `NAO-05` · INC-01 |
| PG-12 | Platinum → FUNDAMENTADA, não GAP_DOCUMENTAL | `[PRB]` | `DEVE-14` · RN-05 |
| PG-13 | Prazo de devolução: data de referência é o tracking | `[PRB]` | `DUVIDA-05` · POL-001 §3.1 |
| PG-14 | Coleta Reversa ≠ Frete Reverso | `[PRB]` | `DUVIDA-10` · POL-001 §3.3/3.5 |
| PG-15 | Seguro de carga: fonte informal, resposta condicional | `[PRB]` | `NAO-11` · OQ-02 |
| PG-16 | Lacunas documentais conhecidas com comportamento definido | `[DET]`/`[PRB]` | `NAO-02` · RN-04 · INC-01 |
| PG-17 | Mudanças no system prompt seguem protocolo de versionamento | `[DET]` | `DEVE-15` · Anexo C /prompts/ |

---

## OUTPUT 20 — Seção "Glossário de Linguagem Ubíqua do Domínio"

**Prompt:** Adicione ao AGENTS.md o Glossário de linguagem ubíqua do domínio que
os agentes precisam conhecer.

### Decisões de escrita

**Quatro camadas por entrada, todas orientadas ao agente:**
1. Definição canônica — com números exatos, sem ambiguidade
2. Fonte normativa — documento e seção de origem
3. "Nunca usar" — lista explícita de sinônimos proibidos
4. Nota operacional — o que fazer com o termo ao gerar código ou respostas

**Snippets TypeScript nas entradas críticas.** `ClientTier` como type union, nomes
de campo canônicos (`cteNumber`, `clockMode`), `topic_tag` como literal string.

**Tabela comparativa v1 × v2 nos parâmetros de frete.** Multiplicadores e fatores de
peso da versão obsoleta e da vigente lado a lado — para que o agente nunca confunda.

### Conteúdo produzido — 25 termos em 4 grupos + tabela de proibidos

**Grupo 1 — Tipos de Carga e Modalidades de Frete (8 termos):**
Carga Perigosa · Carga Refrigerada · Carga com Lacre Violado · Frete Especial ·
Frete Padrão · Multiplicador Regional · Fator de Peso · Valor Base

**Grupo 2 — Processo de Devolução (6 termos):**
Solicitação de Devolução · Prazo de Devolução · CT-e · Coleta Reversa ·
Frete Reverso · Devolução Parcial

**Grupo 3 — SLA e Classificação de Clientes (6 termos):**
Tier de Cliente · Incidente Crítico · SLA de Primeira Resposta · SLA de Resolução ·
Relógio de SLA · Penalidade de SLA

**Grupo 4 — Governança Documental (5 termos):**
Documento Normativo · Documento Contratual · Documento Informal · Versão Vigente ·
Gap Documental

**Tabela de termos proibidos:** 16 pares canônico → nunca usar.

---

## OUTPUT 21 — Seção "Code Constraints"

**Prompt:** Adicione ao AGENTS.md restrições que impactam geração de código.

### Decisões de escrita

**Nenhuma duplicação com as PGs.** As PGs descrevem comportamento do assistente
em runtime. As CCs descrevem como o código que produz esse comportamento deve ser
escrito. São camadas diferentes.

**Código gerado é código de produção.** Cada snippet é TypeScript válido e compilável
com `strict: true`. Nomes de arquivo coincidem exatamente com a árvore do Anexo C.

**Os incidentes viraram testes de regressão obrigatórios (CC-14).** INC-01, INC-02
e INC-03 têm arquivos de teste nomeados e asserts específicos.

**Tabela de resumo por arquivo** como index de lookup: o agente que está gerando
código para um arquivo sabe quais CCs se aplicam sem ler o documento inteiro.

### Conteúdo produzido — 15 restrições (CC-01 a CC-15) em 8 grupos

| Grupo | CCs | O que cobre |
|---|---|---|
| Contrato de tipos (`src/shared/types.ts`) | CC-01, CC-02, CC-03 | `QueryResponse`, `QueryRequest`, `ChunkMetadata`, `SourceDocument`, tipos canônicos |
| Validação de input com Zod | CC-04 | Schema Zod obrigatório, HTTP 400 para erros de validação |
| Guards de output | CC-05a, CC-05b | Guard de valor monetário, guard de status em tempo real |
| HTTP e comportamento | CC-06, CC-07 | Mapa canônico de status HTTP, `request_id` obrigatório em erros |
| Performance | CC-08, CC-09 | `Promise.race` para timeout 8s, `EXPIRE` Redis após input do atendente |
| Configuração de ambiente | CC-10 | `requireEnv()` na inicialização, variáveis de negócio em env/config |
| Convenções TypeScript | CC-11, CC-12 | `strict: true` implicações, catálogo de erros tipados |
| Testes | CC-13, CC-14, CC-15 | Fixtures canônicos, testes de regressão INC-01/02/03, formato golden queries |

### Tipos TypeScript canônicos definidos

```typescript
type ResponseState = 'FUNDAMENTADA' | 'COM_RESSALVA' | 'CONTRADIÇÃO_DETECTADA' | 'GAP_DOCUMENTAL'
type ConfidenceLevel = 'normativo' | 'contratual' | 'informal'
type Domain = 'frete_logistica' | 'devolucao_politica' | 'sla_contrato' | 'nao_classificado'
type ClientTier = 'Gold' | 'Silver' | 'Standard'  // Platinum não existe

interface QueryResponse { state, answer, session_id, domain_classified, sources,
  confidence_level?, disclaimer?, conflicting_docs?, routing_suggestion?, max_score_obtained? }

interface QueryRequest { query, session_id, user_id, tier_hint? }

interface ChunkMetadata { doc_id, doc_version, doc_type, status, topic_tag,
  attribute_key, section, source_file }

// Erros tipados
class ChunkMetadataError, class OutputGuardError, class TimeoutError,
class SessionNotFoundError, class ConflictDetectedError
```

### Tabela de resumo por arquivo (CC)

| Arquivo | CCs |
|---|---|
| `src/shared/types.ts` | CC-01, CC-03 |
| `src/shared/config.ts` | CC-10 |
| `src/shared/errors.ts` | CC-12 |
| `src/shared/logger.ts` | CC-07 |
| `src/functions/query/handler.ts` | CC-06, CC-07, CC-08, CC-09 |
| `src/functions/query/validator.ts` | CC-04 |
| `src/functions/query/response-builder.ts` | CC-01, CC-02 |
| `src/services/search.ts` | CC-02, CC-03 |
| `src/services/response-validator.ts` | CC-05a, CC-05b |
| `src/pipeline/indexer.ts` | CC-03 |
| `tests/fixtures/chunks.ts` | CC-13 |
| `tests/integration/` | CC-14 |
| `prompts/eval/golden-queries.json` | CC-15 |

---

## OUTPUT 22 — Seção "Spec References"

**Prompt:** Adicione ao AGENTS.md referências a documentos de spec no repositório,
usando o Anexo C como base.

### Conteúdo produzido — 14 subsections (SR-01 a SR-14)

| SR | Título | O que cobre |
|---|---|---|
| SR-01 | Mapa de specs por módulo | Tabela de 5 módulos → pasta → quando ler |
| SR-02 | Três artefatos SDD por módulo | requirements / plan / tasks — ordem e responsáveis |
| SR-03 | Specs do Query Endpoint | `requirements.md v1.1.0` — seções críticas, plan e tasks a criar |
| SR-04 | Specs do Pipeline de Ingestão | Requirements a escrever — campos obrigatórios, lacuna L-01 |
| SR-05 | Specs dos módulos de suporte | feedback-api, teams-bot, painel-web — o que cada requirements deve cobrir |
| SR-06 | ADRs: decisões já tomadas | 5 ADRs com nomes de arquivo sugeridos e status |
| SR-07 | Skills: leia antes de gerar código | Hierarquia foundation/domain/artifact, regra por tarefa |
| SR-08 | Prompts: documentos versionados | Tabela de 6 arquivos em `prompts/`, protocolo de mudança |
| SR-09 | Docs NovaTech em `docs/novatech/` | 5 documentos com tipo, versão e nota sobre gaps |
| SR-10 | Índice de navegação por tarefa | 22 linhas: arquivo → specs + skills + CCs/PGs a ler |
| SR-11 | MCP config (`.mcp/mcp.json`) | 4 servers, escopos, regras de criação |
| SR-12 | Corpus de retrieval (`data/retrieval-corpus/`) | Estrutura de diretórios, formato JSON de chunk, regras críticas |
| SR-13 | Infraestrutura (`infra/`) | Bicep como estado narrativo, mapa módulo → variável de ambiente |
| SR-14 | Runbooks e onboarding | Conteúdo esperado, 4 runbooks prioritários derivados de incidentes |

### SR-10 — Índice de navegação (destaque)

O SR-10 é o principal ponto de entrada operacional: dado qualquer arquivo do
repositório, a tabela indica exatamente o que ler antes de começar a implementar.
Cobre 22 destinos — de `src/functions/query/handler.ts` até `infra/` (estado narrativo).

### Referências consolidadas adicionadas

Tabela de status de todos os 26 artefatos do repositório:

| Status | Artefatos |
|---|---|
| ✅ Vigente | AGENTS.md · guardrails.md v1.3.0 · requirements.md v1.1.0 · docs/novatech/ · docs/adr/template.md |
| ⚠️ Parcial | system-prompt.md (versão básica) · data/retrieval-corpus/ (semeado pelo Anexo B) · infra/ (estado narrativo) |
| 🔲 A criar/escrever | requirements de 4 módulos · plans · tasks · 5 ADRs · onboarding · runbooks · .mcp/mcp.json · prompts/*.json · skills/* |

---

## Estrutura final do AGENTS.md produzido

```
AGENTS.md
│
├── ## Product Rules & Guardrails        PG-01 a PG-17 (17 regras)
│   └── Cada PG: enunciado + snippet TypeScript + origem + violação
│
├── ## Glossário de Linguagem Ubíqua     25 termos em 4 grupos
│   ├── Grupo 1 — Tipos de Carga (8 termos)
│   ├── Grupo 2 — Processo de Devolução (6 termos)
│   ├── Grupo 3 — SLA e Clientes (6 termos)
│   ├── Grupo 4 — Governança Documental (5 termos)
│   └── Tabela de termos proibidos (16 pares)
│
├── ## Code Constraints                  CC-01 a CC-15 (15 restrições)
│   ├── Contrato de tipos (CC-01–03)
│   ├── Validação Zod (CC-04)
│   ├── Guards de output (CC-05)
│   ├── HTTP (CC-06–07)
│   ├── Performance (CC-08–09)
│   ├── Config de ambiente (CC-10)
│   ├── TypeScript conventions (CC-11–12)
│   ├── Testes (CC-13–15)
│   └── Tabela de resumo por arquivo
│
└── ## Spec References                   SR-01 a SR-14 (14 subsections)
    ├── SR-01 Mapa de módulos
    ├── SR-02 Artefatos SDD
    ├── SR-03 Query Endpoint specs
    ├── SR-04 Pipeline specs
    ├── SR-05 Módulos de suporte
    ├── SR-06 ADRs
    ├── SR-07 Skills
    ├── SR-08 Prompts
    ├── SR-09 Docs NovaTech
    ├── SR-10 Índice por tarefa (22 destinos)
    ├── SR-11 MCP config
    ├── SR-12 Corpus de retrieval
    ├── SR-13 Infra/Bicep
    ├── SR-14 Runbooks e onboarding
    └── Referências consolidadas (26 artefatos com status)
```

**Totais:** 17 PGs + 25 termos + 15 CCs + 14 SRs = **71 entradas** em 1.936 linhas.

---

## Rastreabilidade com sessões anteriores

```
Anexo A (documentação NovaTech)
    └── Linguagem ubíqua ────────────────► Glossário (25 termos)
    └── Contradições + Gaps ─────────────► PG-02, PG-11, PG-15, PG-16

Anexo C (estrutura do repositório)
    └── /specs/ ─────────────────────────► SR-01 a SR-05, SR-10
    └── /skills/ ────────────────────────► SR-07
    └── /prompts/ ───────────────────────► SR-08, PG-17
    └── /docs/ ──────────────────────────► SR-06, SR-09, SR-14
    └── /.mcp/ ──────────────────────────► SR-11
    └── /data/ ──────────────────────────► SR-12
    └── /infra/ ─────────────────────────► SR-13
    └── /src/ (nomes de arquivo) ────────► CC-01 a CC-15 (tabela por arquivo)
    └── tsconfig strict: true ───────────► CC-11
    └── Zod em validator.ts ─────────────► CC-04

guardrails.md v1.3.0
    └── 37 guardrails [DET]/[HBR]/[PRB] ► PG-01 a PG-17 (regras de runtime)
    └── 3 incidentes INC-01/02/03 ───────► CC-14 (testes de regressão obrigatórios)
    └── 3 lacunas L-01/02/03 ────────────► SR-04 (requirements pipeline), SR-14 (runbooks)

requirements.md v1.1.0
    └── Schemas request/response ────────► CC-01 (QueryRequest, QueryResponse)
    └── ChunkMetadata (ADR-006) ─────────► CC-03
    └── RNF-01 (latência por estado) ────► CC-06, CC-08
    └── RNF-02 (concorrência 45 req) ────► CC-08
    └── RN-08 (sessão Redis) ────────────► CC-09, PG-07
    └── RN-09 (sinônimos + domínio) ─────► CC-10, PG-06
    └── RN-10 (encaminhamentos) ─────────► CC-10, PG-10
    └── AC-01 a AC-10 ───────────────────► SR-03 (seções críticas do requirements)
    └── OQs pendentes ───────────────────► PG-15 (OQ-02), SR-03 (§12 OQs)
```
