# Output Consolidado — Análise de Domínio NovaTech
**Projeto:** Assistente de IA para Atendimento ao Cliente
**Cliente:** NovaTech Logística
**Fornecedor:** DB1
**Data:** Junho 2026

---

## OUTPUT 1 — Contextualização do Projeto

### Contexto do Negócio
- 45 atendentes · 320 chamados/dia · ~192 com consulta documental
- Tempo médio de busca: 12 min → meta: <2 min
- Documentação em 3 fontes: SharePoint (~800 docs), Confluence (~400 páginas), pasta de rede (planilhas mensais)

### Inventário da Base de Conhecimento

| Documento     | Tipo        | Status                        | Risco    |
|---------------|-------------|-------------------------------|----------|
| POL-001 v3.1  | Normativo   | ✅ Vigente                    | Baixo    |
| PROC-042 v1   | Procedimento| ⚠️ Ambíguo (não arquivado)   | Alto     |
| PROC-042 v2   | Procedimento| ⚠️ Substituto não formal     | Alto     |
| SLA-2024      | Contratual  | ✅ Vigente                    | Médio    |
| FAQ-Atendimento| Informal   | ⚠️ Sem responsável, não validado | Crítico |

### Contradições Mapeadas

| # | Conflito                              | Documentos          | Impacto                   |
|---|---------------------------------------|---------------------|---------------------------|
| C1 | Multiplicadores regionais diferentes | PROC-042 v1 × v2    | Cálculo de frete errado   |
| C2 | Fatores de peso diferentes           | PROC-042 v1 × v2    | Cálculo de frete errado   |
| C3 | Prazo adicional +2 × +3 dias úteis   | PROC-042 v1 × v2    | Prazo comunicado errado   |
| C4 | Frete expresso para carga perigosa   | FAQ × ausência PROC | Risco jurídico/operacional|
| C5 | Desconto 10/mês (FAQ) × 8/mês (v2)  | FAQ × PROC-042 v2   | Expectativa errada        |

### Gaps Críticos

| Gap                          | Fonte atual      | Risco                            |
|------------------------------|------------------|----------------------------------|
| Carga danificada em trânsito | FAQ Item 38      | Sem fundamento normativo         |
| Seguro de carga              | FAQ Item 22      | Informação potencialmente desatualizada |
| Frete padrão (<500kg)        | Nenhuma          | Lacuna total de resposta         |
| Processo Gestão de Riscos    | Referência na POL-001 | Atendente sem saber o que ocorre após encaminhamento |

### Riscos Críticos do Projeto

- **R1** — Assistente pode reproduzir contradições com confiança (PROC-042 v1 vs v2)
- **R2** — FAQ informal pode contaminar respostas normativas
- **R3** — Sem pipeline de re-ingestão, base degrada após primeiro ciclo de revisão
- **R4** — Meta 12→2min irrealista para cenários com contradições (exige fallback humano)
- **R5** — Gaps documentais reais → ~4 categorias sem resposta fundamentada possível

### Premissas a Validar com NovaTech (antes do dev)

1. Qual versão do PROC-042 está vigente formalmente?
2. O FAQ será ingerido? Com qual peso/confiabilidade?
3. Haverá processo de governança documental como entregável?
4. O assistente pode responder sobre seguros e cargas danificadas?
5. Qual o comportamento esperado quando não há resposta confiável?

---

## OUTPUT 2 — Bounded Contexts

### BC-01 — Gestão do Conhecimento
**Missão:** Única fonte de verdade sobre quais documentos existem, quais estão vigentes e com qual grau de confiabilidade.

**Dentro:** Catálogo de documentos · classificação normativo/contratual/informal · controle de versões · pipeline de ingestão/re-ingestão · metadados de fonte · regra de precedência entre conflitos.

**Fora:** Criação/edição de documentos · aprovação de políticas · execução dos processos descritos.

**Risco crítico:** PROC-042 v1 e v2 coexistem sem hierarquia formal — bloqueante de implementação.

---

### BC-02 — Atendimento e Consulta
**Missão:** Receber pergunta, recuperar informação relevante, devolver resposta fundamentada com fonte — ou escalar.

**Dentro:** Interpretação NLP · recuperação semântica (RAG) · geração com citação · classificação de confiabilidade · fluxo de fallback · fluxo de escalada · histórico por sessão.

**Fora:** Abertura de chamados · cálculo de valores de frete · verificação de SLA em tempo real · edição de documentos.

**Comportamentos por tipo de pergunta:**

| Tipo                          | Comportamento                             |
|-------------------------------|-------------------------------------------|
| Coberta por doc normativo     | Resposta com citação                      |
| Coberta só pelo FAQ           | Resposta com ressalva explícita           |
| Documentos conflitantes       | Alerta de contradição + escalada          |
| Sem cobertura documental      | "Não encontrado" + sugestão de encaminhamento |

**Risco crítico:** Comportamento de fallback não especificado formalmente pelo cliente.

---

### BC-03 — SLAs e Contratos
**Missão:** Responder sobre compromissos contratuais de nível de serviço, classificação de clientes e penalidades — sem executar a medição.

**Dentro:** Classificação Gold/Silver/Standard · tabela de SLAs · definição de incidente crítico · regras de penalidade · comportamento do relógio de SLA · resposta a "existe Platinum?".

**Fora:** Medição real de SLA (Azure DevOps) · consulta ao tier real do cliente (CRM/ERP) · negociação de SLA · emissão de créditos.

**Ambiguidade:** Operador OU nos critérios de tier; atendente precisa saber o tier do cliente antes de consultar.

**Risco médio:** FAQ usa limiar R$50K para "prioridade alta"; SLA-2024 usa R$100K para incidente crítico.

---

### BC-04 — Logística e Frete
**Missão:** Responder sobre regras de frete, prazos e condições especiais — orientando o cálculo, não executando.

**Dentro:** Regras de frete especial (>500kg) · fórmula × multiplicador × fator · multiplicadores v2 vigentes · fatores de peso · prazo +3 dias · condições especiais · descontos por volume · encaminhamento PROC-043.

**Fora:** Valor base do frete (tabela mensal externa) · execução do cálculo final · autorização de exceções · frete padrão <500kg (gap documental).

**Delta v1 vs v2 (risco de resposta errada):**

| Parâmetro              | v1  | v2  | Delta     |
|------------------------|-----|-----|-----------|
| Multiplicador Norte    | 1.6 | 1.8 | +12,5%    |
| Multiplicador Nordeste | 1.4 | 1.5 | +7,1%     |
| Multiplicador CO       | 1.3 | 1.4 | +7,7%     |
| Fator >3.000kg         | 1.5 | 1.4 | -6,7%     |
| Prazo adicional        | +2d | +3d | +1 dia    |

**Risco crítico:** Duas versões com valores materialmente diferentes coexistindo na base.

---

### BC-05 — Auditoria e Melhoria Contínua
**Missão:** Registrar uso, identificar padrões de falha e gaps recorrentes — retroalimentar BC-01.

**Dentro:** Log de consultas anonimizadas · registro de fallbacks por categoria · registro de escaladas · métricas de uso · dashboard de gaps recorrentes · feedback do atendente.

**Fora:** PII de clientes · métricas de performance individual de atendentes · integração com RH.

**Risco crítico:** Ausência de requisito formal de observabilidade no escopo do projeto.

---

### Mapa de Relações

```
[Fontes Externas: SharePoint / Confluence / Pasta de rede]
                        │
                        ▼
              ┌─────────────────┐ ◄─── gaps ─── BC-05
              │    BC-01        │
              │  Gestão do      │
              │  Conhecimento   │
              └──────┬──────────┘
                     │ chunks + metadados
                     ▼
          ┌──────────────────┐       ┌──────────────┐
          │     BC-02        │──────►│    BC-03     │
          │  Atendimento     │       │  SLAs e      │
          │  e Consulta      │──────►│  Contratos   │
          │                  │       └──────────────┘
          │                  │       ┌──────────────┐
          │                  │──────►│    BC-04     │
          └────────┬─────────┘       │  Logística   │
                   │                 │  e Frete     │
                   │                 └──────────────┘
                   ▼
         ┌──────────────────┐
         │     BC-05        │──────► retroalimenta BC-01
         │  Auditoria e     │──────► Gestão do Projeto
         │  Melhoria        │
         └──────────────────┘
```

---

## OUTPUT 3 — Linguagem Ubíqua do Domínio

### Grupo 1 — Tipos de Carga

| Termo              | Definição Canônica                                                           | Fonte      | Risco  |
|--------------------|------------------------------------------------------------------------------|------------|--------|
| Carga Perigosa     | Classes 1-6 ANTT (Res. 5.947/2021): explosivos, gases, líq. inflamáveis, sólidos inflamáveis, oxidantes, tóxicos | POL-001 §3.2 | Médio |
| Carga Refrigerada  | Exige controle de temperatura via sensor IoT. Ruptura = >30min contínuos fora da faixa da NF | POL-001 §3.2 | Médio |
| Carga c/ Lacre Violado | Entregue com lacre rompido. Exceção: documentada no ato com assinatura motorista + recebedor | POL-001 §3.2 | Baixo |
| Frete Especial     | Modalidade para cargas >500kg. Fórmula: Valor base × Multiplicador regional × Fator de peso | PROC-042 v2 | Baixo |
| Frete Padrão       | ⚠️ SEM DEFINIÇÃO FORMAL. Inferido: abaixo de 500kg. Gap documental. | — | **Alto** |

### Grupo 2 — Processo de Devolução

| Termo                  | Definição Canônica                                                        | Fonte        | Risco  |
|------------------------|---------------------------------------------------------------------------|--------------|--------|
| Solicitação de Devolução | Pedido formal no Portal do Cliente com CT-e, 3 fotos e motivo           | POL-001 §3.3 | Baixo  |
| Prazo de Devolução     | 7 dias úteis da data de recebimento confirmada no tracking (≠ data do cliente) | POL-001 §3.1 | Médio |
| CT-e                   | Conhecimento de Transporte Eletrônico. Identificador principal da operação | POL-001 §3.3 | Baixo |
| Coleta Reversa         | Serviço de retirada da mercadoria. Agendado em até 2 dias úteis após aprovação | POL-001 §3.3 | Baixo |
| Frete Reverso          | Custo do transporte de retorno. NÃO confundir com Coleta Reversa (o serviço) | POL-001 §3.5 | Médio |
| Devolução Parcial      | Devolução de volumes individuais. Reembolso proporcional ao peso/valor do CT-e | POL-001 §3.4 | Baixo |

**Responsabilidade pelo custo do Frete Reverso:**

| Motivo               | Responsável pelo custo           |
|----------------------|----------------------------------|
| Erro da NovaTech     | NovaTech (sem custo ao cliente)  |
| Desistência do cliente | Cliente (mesmos multiplicadores do frete original) |
| Prazo expirado       | Não elegível ao processo padrão  |

### Grupo 3 — SLA e Classificação de Clientes

| Termo                  | Definição Canônica                                                        | Fonte      | Risco  |
|------------------------|---------------------------------------------------------------------------|------------|--------|
| Tier de Cliente        | Gold / Silver / Standard. Critério com operador OU. Platinum não existe.  | SLA-2024 §1 | Baixo |
| Incidente Crítico      | Atende ≥1 de 4 critérios: valor >R$100K/6h · carga perigosa irregular · >5 chamados/24h · risco a pessoas | SLA-2024 §3 | Médio |
| SLA de Primeira Resposta | Tempo até 1º retorno ao cliente (mesmo "estamos verificando"). ≠ resolução | SLA-2024 §2 | Baixo |
| SLA de Resolução       | Tempo até encerramento efetivo do problema                               | SLA-2024 §2 | Médio |
| Relógio de SLA         | Pausa fora do horário 08h-18h úteis (chamados gerais). NÃO pausa: Gold + incidente crítico | SLA-2024 §5 | **Alto** |
| Penalidade de SLA      | 1ª: registro / 2ª: 5% do frete do chamado / 3ª+: 10% + reunião obrigatória | SLA-2024 §4 | Médio |

**Tabela de Tiers:**

| Tier     | Critério (operador OU)                                        | Revisão   |
|----------|---------------------------------------------------------------|-----------|
| Gold     | Contrato anual >R$500K OU >200 operações/mês                 | Semestral |
| Silver   | Contrato entre R$100K-500K OU 50-200 operações/mês           | Semestral |
| Standard | Todos os demais                                               | Anual     |

### Grupo 4 — Governança Documental

| Termo               | Definição Canônica                                                        | Risco  |
|---------------------|---------------------------------------------------------------------------|--------|
| Documento Normativo | Regra de cumprimento obrigatório. Prefixo POL ou PROC.                   | Baixo  |
| Documento Contratual| Compromisso formal com cliente. Prefixo SLA. Violação tem consequência financeira. | Médio |
| Documento Informal  | Sem responsável, sem validação. Ex: FAQ. Nunca equiparar a normativo.    | **Alto** |
| Versão Vigente      | PROC-042 v2 para chamados pós 01/12/2023. V1 não foi arquivada formalmente. | **Alto** |
| Gap Documental      | Ausência de fonte normativa para tema relevante do atendimento.           | **Alto** |

### Termos Proibidos (não usar como sinônimos)

| Usar                  | Nunca usar                                     |
|-----------------------|------------------------------------------------|
| Carga Perigosa        | "carga especial", "material perigoso"          |
| Frete Especial        | "frete pesado", "frete diferenciado"           |
| Prazo de Devolução    | "7 dias corridos", "uma semana"                |
| CT-e                  | "nota de transporte", "comprovante de frete"   |
| Coleta Reversa        | "coleta de devolução", "frete reverso" (custo) |
| Tier de Cliente       | "categoria de cliente", "Platinum"             |
| Incidente Crítico     | "chamado urgente", "prioridade alta", "P1"     |

---

## OUTPUT 4 — Artefatos Visuais Gerados

| Artefato                          | Formato | Conteúdo                                      |
|-----------------------------------|---------|-----------------------------------------------|
| novatech-bounded-contexts.jsx     | React   | Mapa interativo com 4 abas por BC (termos / escopo / riscos / relações) |
| novatech-bounded-contexts.svg     | SVG     | Mapa estático com BCs, termos ubíquos e relações |
| novatech-bc-completo.svg          | SVG     | Mapa completo com termos + escopo (dentro/fora) + alertas de risco |

---

## Síntese Executiva

### O que foi produzido nesta conversa

1. **Diagnóstico completo da base documental** — contradições, gaps e riscos mapeados antes de qualquer linha de código.
2. **5 Bounded Contexts** com fronteiras claras, responsabilidades e relações — base para o design da arquitetura RAG.
3. **Glossário de linguagem ubíqua** com 25+ termos canônicos — insumo direto para o system prompt do assistente e para o pipeline de chunking.
4. **3 artefatos visuais** exportáveis para Confluence, Miro ou documentação de arquitetura.

### Próximos passos recomendados

| Prioridade | Ação                                                                 | Responsável    |
|------------|----------------------------------------------------------------------|----------------|
| 🔴 Bloqueante | Formalizar versão vigente do PROC-042 (v1 arquivada, v2 vigente)  | NovaTech       |
| 🔴 Bloqueante | Definir comportamento de fallback e escalada do assistente         | DB1 + NovaTech |
| 🟡 Alta     | Definir se FAQ será ingerido e com qual peso de confiabilidade       | NovaTech       |
| 🟡 Alta     | Criar documentos formais para carga danificada e seguro de carga     | NovaTech       |
| 🟡 Alta     | Incluir BC-05 (Auditoria) como requisito formal no backlog           | DB1            |
| 🟢 Média    | Modelar fluxo de consulta do BC-02 (estados de resposta)             | DB1            |
| 🟢 Média    | Escrever primeiros PBIs priorizados por risco                        | DB1            |

---

## OUTPUT 5 — requirements.md · Query Endpoint (SDD)

**Arquivo gerado:** `requirements.md`
**Componente:** Query Endpoint — BC-02 (Atendimento e Consulta)
**Estrutura:** Software Design Document (SDD)

### Seções produzidas

| Seção | Conteúdo |
|---|---|
| 1. Overview | Posicionamento do endpoint no sistema RAG e relação com BC-02 |
| 2. Prior Decisions (ADRs) | 5 ADRs simuladas derivadas das decisões desta conversa |
| 3. Scope Boundaries | Dentro/fora derivados diretamente dos BCs mapeados |
| 4. User Outcomes | 5 outcomes orientados a resultado do usuário |
| 5. Estados de Resposta | Máquina de estados: FUNDAMENTADA / COM_RESSALVA / CONTRADIÇÃO_DETECTADA / GAP_DOCUMENTAL |
| 6. Regras de Negócio | 8 regras com rastreabilidade a documentos e ADRs |
| 7. Acceptance Criteria | 10 ACs verificáveis em teste, rastreados aos UOs |
| 8. Edge Cases | 8 cenários de risco com origem nas contradições C1-C5 e gaps do Anexo A |
| 9. Dependencies | Bloqueantes e não-bloqueantes identificadas |
| 10. Out of Scope | 6 itens explicitamente excluídos com justificativa |
| 11. Open Questions | 5 questões bloqueantes com AC afetado e prazo sugerido |

### ADRs simuladas (derivadas da conversa)

| ADR | Decisão | Consequência principal |
|---|---|---|
| ADR-001 | Arquitetura RAG sobre base estática | Sem dependência de latência com fontes externas em tempo de consulta |
| ADR-002 | Classificação de confiabilidade por tipo de documento | Estado COM_RESSALVA obrigatório para respostas baseadas em FAQ |
| ADR-003 | Controle de versão como metadado obrigatório | PROC-042 v1 obsoleta no índice — chamados novos usam exclusivamente v2 |
| ADR-004 | Comportamento determinístico para contradições | Estado CONTRADIÇÃO_DETECTADA — endpoint nunca escolhe entre versões conflitantes |
| ADR-005 | Integração via Microsoft Teams | Latência máxima 8s (limite de timeout do Teams para bots) |

### User Outcomes (resumo)

| UO | Frase do usuário | Medida de sucesso |
|---|---|---|
| UO-01 | "Preciso responder ao cliente agora, sem abrir três abas." | Tempo de busca cai de 12 min para <2 min |
| UO-02 | "Preciso saber se posso passar essa informação ao cliente." | FAQ sempre marcado como "fonte informal" — atendente não repassa sem verificar |
| UO-03 | "Não quero ser responsabilizado por informação contraditória." | Zero casos de multiplicador v1 usado após go-live |
| UO-04 | "Prefiro saber que não tem resposta do que receber uma inventada." | Nenhuma resposta gerada sobre tema sem cobertura documental |
| UO-05 | "Faço várias perguntas sobre o mesmo chamado." | Follow-up sem necessidade de repetir contexto dentro da sessão |

### Estados de Resposta (máquina de estados)

```
Pergunta do atendente
        │
        ▼
Recuperação semântica (BC-01)
        │
   ┌────┴────────────────┬─────────────────┐
   │                     │                 │
   ▼                     ▼                 ▼
Chunks de          Chunks de docs    Nenhum chunk
1 doc consistente  conflitantes      relevante
   │                     │                 │
   ▼                     ▼                 ▼
Classifica      CONTRADIÇÃO_DETECTADA  GAP_DOCUMENTAL
confiabilidade    Alerta + Escalada    "Não encontrado"
   │                                  + encaminhamento
   ├── Normativo/Contratual → FUNDAMENTADA
   └── Apenas FAQ           → COM_RESSALVA
```

### Acceptance Criteria (resumo)

| AC | Cenário | Estado esperado |
|---|---|---|
| AC-01 | Pergunta coberta por doc normativo | FUNDAMENTADA em ≤8s com citação |
| AC-02 | Única fonte é FAQ | COM_RESSALVA com disclaimer obrigatório |
| AC-03 | Chunks v1 e v2 recuperados para mesma região | CONTRADIÇÃO_DETECTADA — nenhum valor exibido como correto |
| AC-04 | Consulta sobre frete especial sem data | Apenas PROC-042 v2 utilizado |
| AC-05 | Pergunta sobre frete padrão (<500kg) | GAP_DOCUMENTAL + encaminhamento ao Comercial |
| AC-06 | Carga classe 8 ANTT (corrosivos) | GAP_DOCUMENTAL — doc cobre apenas classes 1-6 |
| AC-07 | Follow-up na mesma sessão sem repetir contexto | Resposta consistente com histórico da sessão |
| AC-08 | SLA de chamado geral de cliente Gold | Resposta especifica que relógio PAUSA fora do horário comercial |
| AC-09 | SLA de incidente crítico de cliente Gold | Resposta especifica que relógio NÃO pausa — corre 24/7 |
| AC-10 | Pergunta sobre tier Platinum | FUNDAMENTADA — tier descontinuado em 2022, lista os 3 vigentes |

### Open Questions bloqueantes

| # | Questão | AC afetado |
|---|---|---|
| OQ-01 | FAQ será ingerido? Com qual peso? | AC-02, E6 |
| OQ-02 | Comportamento para seguro de carga (FAQ Item 22)? | AC-05 (escopo) |
| OQ-03 | NovaTech formalizará obsolescência do PROC-042 v1 antes do go-live? | AC-03, AC-04 |
| OQ-04 | Encaminhamento automático ao supervisor ou apenas orientação textual na contradição? | AC-03 |
| OQ-05 | Atendente informa o tier ou o endpoint infere do contexto? | AC-08, AC-09 |

---

## Síntese Executiva Atualizada

### Artefatos produzidos na conversa completa

| # | Artefato | Formato | Descrição |
|---|---|---|---|
| 1 | Contextualização NovaTech | Markdown | Diagnóstico de base documental, contradições, gaps e riscos |
| 2 | Bounded Contexts | Markdown | 5 BCs com missão, escopo e relações |
| 3 | Linguagem Ubíqua | Markdown | 25+ termos canônicos com fonte, risco e termos proibidos |
| 4 | Mapa de BCs | React (.jsx) | Mapa interativo com 4 abas por contexto |
| 5 | Mapa de BCs | SVG v1 | Mapa estático com termos e relações |
| 6 | Mapa de BCs | SVG v2 | Mapa completo com escopo dentro/fora + alertas |
| 7 | requirements.md | Markdown (SDD) | Requisitos do Query Endpoint com 5 ADRs, 5 UOs, 4 estados, 8 RNs, 10 ACs, 8 edge cases |

### Rastreabilidade entre artefatos

```
Anexo A (fonte primária)
    │
    ├── Contradições C1-C5 ──────────► RN-01, RN-02, E1-E8 (requirements.md)
    ├── Gaps documentais ────────────► RN-04, RN-07, GAP_DOCUMENTAL
    ├── Linguagem Ubíqua ────────────► Termos proibidos → E1, E3
    │
    ├── Bounded Contexts (BC-01 a BC-05)
    │       ├── BC-02 ───────────────► Escopo completo do Query Endpoint
    │       ├── BC-01 ───────────────► Scope Boundaries (fora do escopo)
    │       ├── BC-03 ───────────────► Scope Boundaries (fora do escopo)
    │       ├── BC-04 ───────────────► Scope Boundaries + RN-04
    │       └── BC-05 ───────────────► Scope Boundaries + Dependency
    │
    └── Decisões de arquitetura (ADR-001 a ADR-005)
            └── requirements.md § Prior Decisions
```

### Próximos passos recomendados

| Prioridade | Ação | Responsável | Bloqueia |
|---|---|---|---|
| 🔴 Bloqueante | Responder OQ-01 a OQ-05 antes do Sprint 1 | NovaTech | AC-02, AC-03, AC-04, AC-08, AC-09 |
| 🔴 Bloqueante | Formalizar PROC-042 v1 como obsoleto no SharePoint | NovaTech | ADR-003, AC-04 |
| 🟡 Alta | Provisionar Azure AI Services e Teams Bot | NovaTech + DB1 | ADR-001, ADR-005 |
| 🟡 Alta | Implementar BC-01 com metadados de fonte por chunk | DB1 | AC-01, AC-02, AC-03 |
| 🟡 Alta | Mapear glossário de termos proibidos no pré-processamento | DB1 | E1, E3 |
| 🟢 Média | Modelar fluxo de consulta BC-02 (diagrama de estados detalhado) | DB1 | Sprint 2 |
| 🟢 Média | Escrever requirements.md de BC-01 (Gestão do Conhecimento) | DB1 | Sprint 2 |
| 🟢 Média | Escrever requirements.md de BC-05 (Auditoria) | DB1 | Sprint 3 |
