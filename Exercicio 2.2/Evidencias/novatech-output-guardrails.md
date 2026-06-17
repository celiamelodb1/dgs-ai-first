# Output — Sessão Guardrails · Query Endpoint NovaTech
**Projeto:** Assistente de IA NovaTech
**Insumo:** requirements.md v1.1.0
**Entregável:** guardrails.md v1.3.0
**Data:** Junho 2026

---

## OUTPUT 14 — guardrails.md v1.0.0
**Prompt:** Elabore um documento de guardrails organizado em DEVE / NÃO DEVE / QUANDO EM DÚVIDA.

### Estrutura produzida

| Seção | Guardrails | Descrição |
|---|---|---|
| DEVE | 15 (DEVE-01 a DEVE-15) | Comportamentos obrigatórios em toda resposta |
| NÃO DEVE | 12 (NAO-01 a NAO-12) | Comportamentos proibidos em qualquer circunstância |
| QUANDO EM DÚVIDA | 10 (DUVIDA-01 a DUVIDA-10) | Comportamentos de fallback para cenários ambíguos |
| **Total** | **37** | |

### Formato de cada guardrail

```
[ID] Enunciado do comportamento
→ Origem: seção do requirements.md
→ Violação / Risco se violado: consequência
```

### Guardrails por seção (resumo)

**DEVE (obrigatórios):**

| ID | Comportamento | Origem |
|---|---|---|
| DEVE-01 | Toda resposta indica o estado (FUNDAMENTADA / COM_RESSALVA / CONTRADIÇÃO_DETECTADA / GAP_DOCUMENTAL) | §5 Estados |
| DEVE-02 | Resposta cita fonte, versão e seção do documento | ADR-003, AC-01 |
| DEVE-03 | Resposta baseada em FAQ usa COM_RESSALVA + disclaimer obrigatório | §5 Estado 2, AC-02 |
| DEVE-04 | Contradição detectada por metadados → CONTRADIÇÃO_DETECTADA + instrução de escalada | ADR-004, ADR-006, AC-03 |
| DEVE-05 | Score abaixo do limiar → GAP_DOCUMENTAL + encaminhamento da tabela RN-10 | §5 Estado 4, RN-10 |
| DEVE-06 | Resposta sobre SLA distingue chamado geral (pausa) vs incidente crítico Gold (24/7) | RN-06, AC-08, AC-09 |
| DEVE-07 | Normalizar termos não-canônicos antes do embedding (tabela de sinônimos RN-09) | RN-09, E1, E3 |
| DEVE-08 | Sessão isolada por user_id AAD — dois atendentes nunca compartilham sessão | RN-08 |
| DEVE-09 | PROC-042 v2 exclusivo — v1 marcada obsoleta no índice, nunca recuperada | RN-01, AC-04 |
| DEVE-10 | conflicting_docs preenchido com doc_ids envolvidos em CONTRADIÇÃO_DETECTADA | §5 Estado 3, ADR-006 |
| DEVE-11 | domain_classified preenchido em toda resposta pelo classificador RN-09 | RN-09, §5 Estado 4 |
| DEVE-12 | Timeout 8s → HTTP 504 + mensagem fixa + sessão preservada | RNF-01, ADR-005 |
| DEVE-13 | Sessão expirada / reinício → mensagem de encerramento antes de processar | RN-08, E7 |
| DEVE-14 | Perguntas sobre Platinum → FUNDAMENTADA informando descontinuação em 2022 | RN-05, AC-10 |
| DEVE-15 | Tabela de sinônimos e encaminhamentos em config versionada, nunca hardcoded no prompt | RN-09, RN-10 |

**NÃO DEVE (proibidos):**

| ID | Proibição | Risco se violado |
|---|---|---|
| NAO-01 | Nunca escolher entre documentos conflitantes | Valor errado apresentado como correto |
| NAO-02 | Nunca extrapolar sem cobertura documental | Resposta fabricada sem sinal de alerta |
| NAO-03 | Nunca usar PROC-042 v1 | Multiplicadores desatualizados (até +12,5% no frete) |
| NAO-04 | Nunca mesclar normativo + FAQ em uma resposta | Autoridade de fonte misturada sem sinalização |
| NAO-05 | Nunca responder sobre classes 7-9 ANTT | Orientação incorreta sobre carga regulada |
| NAO-06 | Nunca calcular valor final do frete | Valor calculado com base desatualizada |
| NAO-07 | Nunca acessar PII de clientes | Violação de privacidade / LGPD |
| NAO-08 | Nunca medir SLA em tempo real | Dado desatualizado como se fosse atual |
| NAO-09 | Nunca usar limiar R$50K do FAQ — usar R$100K do SLA-2024 | Escaladas desnecessárias |
| NAO-10 | Nunca executar ações em chamados | Ações irreversíveis sem supervisão |
| NAO-11 | Seguro de carga requer ressalva obrigatória (OQ-02 pendente) | Percentual incorreto ao cliente |
| NAO-12 | LLM não decide sobre contradição — decisão é do mecanismo de metadados | Contradição resolvida silenciosamente |

**QUANDO EM DÚVIDA (fallbacks):**

| ID | Cenário | Comportamento de fallback |
|---|---|---|
| DUVIDA-01 | Query com múltiplos domínios | Domínio com maior contagem de palavras-chave; empate → nao_classificado |
| DUVIDA-02 | tier_hint ausente + pergunta sobre SLA | Responder para todos os tiers, orientar a identificar o tier |
| DUVIDA-03 | Domínio não classificado | nao_classificado + limiar 0,70 + encaminhamento padrão |
| DUVIDA-04 | FAQ como única fonte + risco jurídico | COM_RESSALVA + instrução adicional "verifique com Compliance" |
| DUVIDA-05 | Prazo de devolução com data do cliente | Responder + aviso de que prazo conta do tracking, não da data do cliente |
| DUVIDA-06 | Score em zona cinza (0,50–limiar) | GAP_DOCUMENTAL + registrar score máximo em BC-05 |
| DUVIDA-07 | Reinício do serviço mid-session | Detectar null no Redis → mensagem de encerramento antes de processar |
| DUVIDA-08 | Desconto FAQ (10/mês) vs PROC-042 v2 (8/mês) | CONTRADIÇÃO_DETECTADA + escalada |
| DUVIDA-09 | Frete padrão como sub-tema de pergunta maior | GAP_DOCUMENTAL parcial para esse sub-tema; responder o restante |
| DUVIDA-10 | Ambiguidade Coleta Reversa × Frete Reverso | Responder ambos com definição explícita de cada |

---

## OUTPUT 15 — guardrails.md v1.1.0
**Prompt:** Para cada guardrail, classifique como enforcement via prompt (probabilístico) ou enforcement via código (determinístico).

### Critérios de classificação

| Categoria | Símbolo | Critério |
|---|---|---|
| Determinístico | `[DET]` | Garantido por lógica de código — filtro, schema validation, lookup, TTL, timer. LLM não participa da decisão. |
| Probabilístico | `[PRB]` | Depende do LLM seguir instrução no system prompt. Pode falhar silenciosamente. |
| Híbrido | `[HBR]` | Decisão estrutural em código, mas conteúdo gerado pelo LLM ainda carrega risco residual. |

### Distribuição

| Categoria | Quantidade | % |
|---|---|---|
| `[DET]` Determinístico | 18 | 49% |
| `[HBR]` Híbrido | 12 | 32% |
| `[PRB]` Probabilístico | 7 | 19% |

### Guardrails por categoria

**Determinísticos (18):**
DEVE-01, DEVE-05, DEVE-07, DEVE-08, DEVE-09, DEVE-10, DEVE-11, DEVE-12, DEVE-13, DEVE-15,
NAO-03, NAO-07, NAO-08, NAO-10, NAO-12, DUVIDA-03, DUVIDA-06, DUVIDA-07

**Híbridos (12):**
DEVE-02, DEVE-03, DEVE-04, NAO-01, NAO-02, NAO-04, NAO-05, DUVIDA-01, DUVIDA-02, DUVIDA-04, DUVIDA-08

**Probabilísticos (7):**
DEVE-06, DEVE-14, NAO-06, NAO-09, NAO-11, DUVIDA-05, DUVIDA-09, DUVIDA-10

### Risco residual dos guardrails críticos

| ID | Risco residual | Mitigação recomendada |
|---|---|---|
| NAO-01 `[HBR]` | Se BC-01 não atribuir `attribute_key`, detecção falha e LLM recebe chunks conflitantes | Validação obrigatória de metadados na ingestão + alerta em BC-05 para chunks sem `attribute_key` |
| NAO-02 `[HBR]` | LLM pode extrapolar além dos chunks recuperados mesmo com instrução no prompt | Guard de output: validar se `sources` contém apenas chunks efetivamente recuperados |
| NAO-05 `[HBR]` | LLM pode extrapolar de chunks de classes 1-6 para 7-9 | Filtro explícito por classe ANTT nos metadados do chunk |
| NAO-06 `[PRB]` | LLM pode alucinar valor base do frete | Instrução explícita no prompt + monitoramento em BC-05 de padrões de valor monetário |
| NAO-09 `[PRB]` | Chunk do FAQ Item 27 (R$50K) pode ser recuperado | Marcar FAQ Item 27 com metadado `superseded_by: SLA-2024-§3` |
| DUVIDA-08 `[HBR]` | FAQ pode não ter `attribute_key: desconto_volume` | Garantir atribuição de `attribute_key` para itens do FAQ com valores numéricos explícitos |

---

## OUTPUT 16 — guardrails.md v1.2.0
**Prompt:** Justifique a classificação dos guardrails.

### Padrões identificados nas justificativas

**O que torna um guardrail `[DET]`** — em todos os 18 casos, a garantia vem de um de quatro mecanismos:
1. **Filtro de índice** — `status: obsoleto` impede recuperação de chunks (DEVE-09, NAO-03)
2. **Ausência de integração** — o sistema simplesmente não tem acesso aos dados (NAO-07, NAO-08, NAO-10)
3. **Operação de estrutura de dados antes do LLM** — TTL, schema validation, lookup em tabela, substituição de string (DEVE-07, DEVE-08, DEVE-12, DEVE-13, DUVIDA-06, DUVIDA-07)
4. **Pipeline antes da chamada ao LLM** — estado atribuído antes da geração (DEVE-01, DEVE-05, DEVE-10, DEVE-11, NAO-12)

**O que torna um guardrail `[HBR]`** — em todos os 12 casos, a decisão estrutural é feita em código, mas a **pré-condição depende da qualidade dos metadados atribuídos por BC-01 na ingestão**. NAO-01, DEVE-04, NAO-04 e DUVIDA-08 são determinísticos *se* `topic_tag` e `attribute_key` estiverem corretos. Se BC-01 falhar, a garantia cai e o LLM recebe dados que não deveria.

**O que torna um guardrail `[PRB]`** — em todos os 7 casos, o comportamento requerido é **semântico ou composicional**: identificar tipo de chamado na query (DEVE-06), detectar que uma data veio do cliente (DUVIDA-05), decompor query em sub-temas (DUVIDA-09), reconhecer ambiguidade entre termos (DUVIDA-10). São comportamentos que só existem no espaço de linguagem natural — sem equivalente estrutural em código que não seja ele mesmo um modelo de linguagem.

**Conclusão arquitetural:** a fronteira mais vulnerável do sistema não é o prompt — é a qualidade da ingestão em BC-01. Os guardrails `[HBR]` mais críticos (NAO-01, DEVE-04) dependem de `attribute_key` estar correto em cada chunk. Isso deve ser requisito de teste de BC-01, não apenas convenção.

---

## OUTPUT 17 — guardrails.md v1.3.0
**Prompt:** Conecte cada guardrail a ao menos um dos 3 incidentes simulados.

### Incidentes de referência

| ID | Descrição |
|---|---|
| INC-01 | Assistente respondeu "prazo de devolução para carga perigosa é 7 dias" — cargas perigosas não são elegíveis ao processo padrão |
| INC-02 | Assistente citou "PROC-042 seção 2" mas os multiplicadores eram da v1 (desatualizada) |
| INC-03 | Assistente disse "não encontrei informação" para SLA Gold — documento SLA-2024 estava indexado |

### Causa-raiz por incidente

**INC-01 — Falha de cobertura de exceção**
O retrieval recuperou o chunk de regra geral (POL-001 §3.1 — prazo 7 dias) mas não o chunk de exceção (POL-001 §3.2 — cargas perigosas não elegíveis). O LLM respondeu com base no chunk disponível sem sinalizar que havia exceções não verificadas. Causa raiz: ausência de mecanismo que force a busca de chunks de exceção quando o chunk de regra geral é recuperado.

**INC-02 — Falha de filtro de índice + integridade de metadados**
O chunk da PROC-042 v1 foi recuperado (filtro `status: obsoleto` não estava ativo) e o campo `sources[].version` exibiu "v2" em vez de "v1" (integridade entre chunk recuperado e metadado de citação rompida). Pior cenário possível: aparência de correção com conteúdo errado.

**INC-03 — Falso GAP_DOCUMENTAL por limiar mal calibrado**
O score de similaridade entre a query ("SLA Gold") e os chunks do SLA-2024 ficou abaixo do limiar do domínio `sla_contrato` (0,78) — possivelmente por falta de normalização de "Gold" para os termos canônicos antes do embedding. O sistema retornou GAP_DOCUMENTAL corretamente dado o limiar, mas o limiar estava errado.

### Mapeamento guardrail × incidente (resumo)

| Guardrail | INC-01 | INC-02 | INC-03 |
|---|---|---|---|
| DEVE-01 | ✅ Previne | ✅ Previne | — |
| DEVE-02 | — | ✅ Previne | — |
| DEVE-03 | ✅ Mitiga | — | — |
| DEVE-04 | ✅ Previne | ✅ Previne | — |
| DEVE-05 | — | — | ✅ Parcial |
| DEVE-06 | — | — | ✅ Previne |
| DEVE-07 | — | — | ✅ Previne |
| DEVE-09 | — | ✅ Previne | — |
| DEVE-10 | — | ✅ Detecta | — |
| DEVE-11 | — | — | ✅ Previne |
| NAO-01 | ✅ Previne | ✅ Previne | — |
| NAO-02 | ✅ Previne | — | — |
| NAO-03 | — | ✅ Previne | — |
| NAO-05 | ✅ Previne | — | — |
| NAO-12 | — | ✅ Previne | — |
| DUVIDA-05 | ✅ Mitiga | — | — |
| DUVIDA-06 | — | — | ✅ Detecta |
| DUVIDA-09 | ✅ Mitiga | — | — |

### Cobertura por incidente

| Incidente | Previnem | Mitigam/Detectam | Guardrail mais crítico |
|---|---|---|---|
| INC-01 | 5 | 3 | NAO-02 — proibe extrapolação além dos chunks recuperados |
| INC-02 | 4 | 3 | DEVE-09 / NAO-03 — filtro de índice elimina a causa raiz |
| INC-03 | 3 | 1 | DEVE-07 — normalização de termos eleva o score de similaridade |

### Lacunas identificadas pela análise

| # | Lacuna | Incidente que expôs | Ação necessária |
|---|---|---|---|
| L-01 | Ausência de busca obrigatória de chunks de exceção quando chunk de regra geral é recuperado | INC-01 | Novo guardrail `DEVE-16`: pipeline deve buscar `scope:excecao` no mesmo `topic_tag` antes de gerar resposta |
| L-02 | Ausência de teste de integridade entre `doc_id` do chunk recuperado e `version` no campo `sources` | INC-02 | Caso de teste de integração obrigatório: `assert response.sources[i].version == chunk_retrieved[i].doc_version` |
| L-03 | Ausência de alerta proativo de degradação de retrieval por domínio | INC-03 | Novo guardrail `DEVE-17`: BC-05 emite alerta quando taxa de GAP_DOCUMENTAL para um domínio ultrapassar 15% em 7 dias |

### Insight transversal

INC-02 é o mais coberto (7 guardrails) e o que não deveria ter ocorrido — todos os guardrails que o previnem são `[DET]`. Sua ocorrência indica falha de implementação, não lacuna de requisito.

INC-01 revelou a única lacuna arquitetural genuína (L-01) — o pipeline não tem mecanismo para busca de exceções. Não é corrigível por prompt; requer requisito novo em BC-01.

INC-03 é o menos coberto (4 guardrails) e o mais difícil de prevenir estruturalmente — a causa raiz é calibração empírica de threshold, não uma regra de negócio. A solução é observabilidade proativa (L-03), não mais guardrails.

---

## Artefatos produzidos nesta sessão

| Artefato | Versão | Seções |
|---|---|---|
| guardrails.md | v1.0.0 | Seções 1-3: 37 guardrails em DEVE / NÃO DEVE / QUANDO EM DÚVIDA |
| guardrails.md | v1.1.0 | + Seção 4: classificação `[DET]` / `[HBR]` / `[PRB]`, distribuição, risco residual, coluna na tabela de rastreabilidade |
| guardrails.md | v1.2.0 | + Seção 4.4: justificativas individuais para todos os 37 guardrails |
| guardrails.md | v1.3.0 | + Seção 5: causa-raiz dos 3 incidentes, mapeamento guardrail × incidente, cobertura por incidente, 3 lacunas (L-01, L-02, L-03) |

## Rastreabilidade com sessões anteriores

```
requirements.md v1.1.0
    │
    ├── §5 Estados de Resposta ──────────► DEVE-01, DEVE-04, DEVE-05
    ├── §6 Regras de Negócio (RN-01~10) ► DEVE-06~15, NAO-01~12
    ├── §7 RNFs ─────────────────────────► DEVE-12
    ├── §8 ACs ──────────────────────────► DEVE-02, DEVE-03, DEVE-09
    ├── §9 Edge Cases (E1~E8) ───────────► DEVE-07, DUVIDA-01~10
    ├── ADR-004, ADR-006 ────────────────► DEVE-04, NAO-01, NAO-12
    └── §3 Scope Boundaries ─────────────► NAO-06~10
            │
            ▼
    guardrails.md v1.3.0
            │
            ├── 37 guardrails classificados [DET]/[HBR]/[PRB]
            ├── 37 justificativas individuais
            ├── Mapeamento com INC-01, INC-02, INC-03
            └── 3 lacunas → backlog BC-01 (DEVE-16) + testes (L-02) + BC-05 (DEVE-17)
```
