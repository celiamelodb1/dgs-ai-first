# Human-in-the-Loop — Política de Aprovação de Mudanças
## Assistente NovaTech (Query Endpoint — BC-02)

**Derivado de:** guardrails.md v1.3.0 · harness-regression-testing-novatech.md
**Data:** 22/06/2026
**Responsável:** DB1 — Produto + Engenharia

---

## Premissa de design

A política não classifica mudanças por "tamanho" ou "complexidade técnica" — classifica por **consequência observável para o atendente e para o cliente**.

Três eixos determinam o nível de aprovação exigido:

| Eixo | Pergunta | Impacto no nível |
|---|---|---|
| Comportamento observável | A mudança altera o que o atendente vê na resposta? | Sobe de Nível 0 para Nível 1 |
| Reversibilidade | A mudança pode ser desfeita em menos de 30 min sem impacto residual? | Irreversível sobe de Nível 1 para Nível 2 |
| Origem do conteúdo | A mudança toca informação com implicação jurídica, contratual ou regulatória? | Qualquer ponto sobe para Nível 2 |

---

## Nível 0 — Deploy automático via CI/CD

### Definição

Mudanças que passam no harness completo (Gates 1, 2 e 3) e não alteram o comportamento observável do assistente de forma que um atendente perceba. Nenhum humano precisa ser notificado antes do deploy.

### Mudanças elegíveis

| Mudança | Condição de elegibilidade |
|---|---|
| Ajuste de prompt — regras P1/P2 | Harness completo pass · não toca guardrails P0 |
| Atualização de metadado de confiança de fonte | Mudança de nível (ex: MÉDIA → ALTA) só com documento normativo aprovado como base |
| Reindexação parcial de documento | Documento já aprovado no Nível 2 · apenas rechunking ou re-embedding · conteúdo idêntico |
| Atualização de threshold de logging | Não altera comportamento de resposta · apenas BC-05 |
| Correção de typo em prompt | Não altera semântica de nenhuma regra |

### Salvaguardas automáticas

- Rollback automático se o monitor da Camada 4 detectar degradação em até 2h pós-deploy.
- Log de versão gerado automaticamente com hash do prompt, lista de documentos, versão do golden set executado.
- Nenhum ajuste de regra P0 (gap documental, alucinação, FAQ disclaimer) é elegível para Nível 0, independentemente do resultado do harness.

---

## Nível 1 — Aprovação: Tech Lead + Product Owner

### Definição

Mudanças que alteram comportamento observável do assistente — o atendente pode perceber a diferença — mas cujo conteúdo é de responsabilidade da equipe de produto e engenharia, sem implicação jurídica ou de área de negócio.

### Quem aprova

**Tech Lead da equipe DB1** — valida integridade técnica, harness, cobertura de guardrails.
**Product Owner do projeto** — valida que o comportamento novo está alinhado com o objetivo de produto.

Ambos precisam aprovar. Aprovação via pull request no repositório — nenhum deploy sem PR aprovado.

**SLA de aprovação:** 1 dia útil após submissão. Submissão bloqueada se harness não tiver sido executado com resultado documentado.

### Mudanças elegíveis

| Mudança | Guardrails afetados | Por que exige Nível 1 |
|---|---|---|
| Ajuste de prompt — regras P0 | DEVE-03, DEVE-05, NAO-01 | Altera comportamento em cenários críticos — gap, alucinação, FAQ |
| Alteração de limiar de similaridade por domínio | DEVE-05, DEVE-11 | Pode gerar ou eliminar GAP_DOCUMENTAL para documentos existentes (INC-03) |
| Atualização da tabela de sinônimos (DEVE-07) | DEVE-07 | Altera o que o pré-processador normaliza — pode mudar qual chunk é recuperado |
| Adição de novo guardrail ao sistema | Todos os novos | Guardrail novo pode entrar em conflito com comportamento existente |
| Remoção de guardrail existente | Todos os removidos | Irreversível do ponto de vista de cobertura — exige justificativa documentada |
| Mudança na lógica de detecção de contradição (DEVE-04/NAO-12) | DEVE-04, NAO-12 | Toca o mecanismo de CONTRADIÇÃO_DETECTADA — impacto alto em confiança |
| Alteração do schema do campo `sources` | DEVE-02, L-02 | Pode quebrar a assertion DET-03 (integridade sources × chunk) |

### Checklist de aprovação — Nível 1

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

## Nível 2 — Aprovação: Área Responsável + Compliance

### Definição

Mudanças que tocam o conteúdo que o assistente usa como fonte de verdade — ou que alteram o escopo do que o assistente pode ou não responder. O conteúdo aqui não é de responsabilidade da equipe técnica: é da área de negócio que possui o documento.

### Quem aprova

**Responsável formal da área dona do documento** (Operações, Comercial ou Compliance, conforme a tabela de documentos).
**Compliance da NovaTech** — obrigatório quando o documento toca carga perigosa, termos jurídicos ou penalidades contratuais.

Ambos precisam aprovar **antes** de qualquer execução de harness ou deploy. A aprovação de conteúdo precede a aprovação técnica.

**SLA de aprovação:** 5 dias úteis. Mudanças urgentes (ex: correção de informação com risco regulatório ativo) podem ser escalonadas para 24h mediante justificativa documentada pelo Product Owner.

### Mudanças elegíveis

| Mudança | Responsável da área | Compliance obrigatório |
|---|---|---|
| Adição de novo documento à base (primeiro ingresso) | Dono da área do documento | Sim, sempre |
| Deprecação de documento ativo (ex: arquivar PROC-042-v1) | Diretoria Comercial | Sim — impacto contratual |
| Formalização de gap documental como novo documento | Área responsável pelo tema + Operações | Sim |
| Mudança no catálogo de gaps (novo gap ou gap resolvido) | Área responsável pelo tema | Não — exceto gaps com implicação ANTT ou jurídica |
| Atualização de documento existente (nova versão) | Dono formal do documento | Sim, se tocar carga perigosa, SLA ou penalidades |
| Alteração do mapeamento de domínio para encaminhamento (RN-10) | Operações | Sim — encaminhamento incorreto é risco operacional |

### Tabela de responsáveis por documento

| Documento | Área responsável | Aprovador de Compliance |
|---|---|---|
| POL-001 — Política de Devolução | Diretoria de Operações | Compliance |
| PROC-042 v1/v2 — Frete Especial | Diretoria Comercial | Compliance (toca contratos) |
| PROC-043 — Frete Carga Perigosa (em revisão) | Compliance + Operações | Compliance (ANTT) |
| SLA-2024 — Tabela de SLA | Comercial + Operações | Compliance (doc contratual) |
| FAQ-Atendimento — Informal | Gestor do time de Atendimento | Não — FAQ nunca vira fonte FUNDAMENTADA sem aprovação de Nível 2 |

### Checklist de aprovação — Nível 2

```
[ ] Documento revisado e assinado pelo responsável formal da área
[ ] Compliance validou: sem termos jurídicos novos não revisados
[ ] Mapeamento de conflito com documentos existentes executado
    (o novo documento contradiz algum chunk já indexado?)
[ ] Se sim: decisão sobre deprecação ou coexistência documentada
[ ] Guardrail DEVE-04 precisa ser atualizado?
    (novo atributo de negócio com valor = novo par topic_tag/attribute_key)
[ ] Guardrail DEVE-05 precisa ser atualizado?
    (novo domínio = novo encaminhamento na tabela RN-10)
[ ] Lacuna L-01 coberta? (novo documento tem escopo regra_geral com exceções
    que precisam de chunk separado com scope:excecao)
[ ] Após aprovação de conteúdo: submeter para aprovação técnica Nível 1
```

---

## Casos de borda e regras de desempate

### Caso 1 — Mudança começa no Nível 0 e descobre conflito durante harness

Se durante a execução do harness de uma mudança Nível 0 o Gate 2 detectar que um caso P0 passou a falhar, a mudança é promovida automaticamente para Nível 1. O Tech Lead é notificado imediatamente e decide: corrigir a mudança ou escalar para Nível 2 se o conflito tiver origem em conteúdo de negócio.

### Caso 2 — Área de negócio solicita mudança "urgente" sem documento formal

Frequente quando a equipe de Operações identifica uma informação errada circulando entre os atendentes. O fluxo correto:

```
1. Área identifica o problema
2. Produto abre ticket de urgência com evidência (ex: feedback de atendente)
3. Compliance confirma o risco em até 4h
4. Se confirmado: área produz nota técnica provisória como documento formal
5. Nota técnica passa por Nível 2 comprimido (24h)
6. Apenas após aprovação: engenharia executa harness e faz deploy
```

O assistente não pode ser corrigido diretamente via prompt sem aprovação de área — isso contornaria o Nível 2 e quebraria a rastreabilidade. A pressão de urgência não muda o processo, muda o SLA.

### Caso 3 — Guardrail `[PRB]` falha no golden set mas harness passa nos DET

Guardrails probabilísticos (`[PRB]`) dependem da qualidade de geração do LLM e podem falhar intermitentemente. Se um caso P0 com guardrail `[PRB]` falhar no golden set, o Tech Lead decide:

- Falha pontual (1 run): re-executar. Se pass na segunda execução, prosseguir com Nível 1.
- Falha consistente (2+ runs): tratar como falha de guardrail real. Investigar prompt antes de aprovar.

Guardrail `[PRB]` nunca pode ser aprovado "apesar da falha" — a falha precisa ser endereçada ou o guardrail precisa ser reclassificado com justificativa documentada.

### Caso 4 — FAQ-Atendimento recebe atualização informal da equipe

O FAQ-Atendimento não segue o processo de Nível 2 porque não é um documento normativo. Qualquer informação adicionada ao FAQ continua sendo tratada como `COM_RESSALVA` pelo assistente (DEVE-03) independentemente do conteúdo.

Se a equipe de atendimento quiser que uma informação do FAQ passe a ser tratada como `FUNDAMENTADA`, precisa formalizar o conteúdo como POL ou PROC e passar pelo processo de Nível 2. Não existe atalho.

### Caso 5 — Mudança toca L-01 (busca obrigatória de chunk de exceção)

A lacuna L-01 ainda não tem implementação. Qualquer mudança que envolva documentos com estrutura regra_geral + exceção (ex: POL-001 §3.1 + §3.2) deve incluir no checklist de Nível 1 a verificação explícita:

```
[ ] O documento novo tem seções de exceção?
    Se sim: chunks de exceção têm scope:excecao configurado?
    Se sim: pipeline executa busca secundária para esse topic_tag?
    Se L-01 não estiver implementado: caso REG-INC-01 vai falhar — esperado.
```

Enquanto L-01 não estiver implementado, mudanças que adicionem documentos com estrutura regra+exceção são aprovadas com nota de risco residual documentada no PR.

---

## Matriz completa de mudanças × nível × aprovador × SLA

| Mudança | Nível | Aprovador | SLA |
|---|---|---|---|
| Ajuste de prompt P2 (monitoramento, SLA duplo) | 0 | CI/CD | Imediato |
| Ajuste de prompt P1 (completude, encaminhamento) | 0 | CI/CD | Imediato |
| Atualização de metadado de confiança | 0 | CI/CD | Imediato |
| Reindexação parcial — doc aprovado | 0 | CI/CD | Imediato |
| Ajuste de prompt P0 (gap, alucinação, FAQ) | 1 | Tech Lead + PO | 1 dia útil |
| Alteração de limiar por domínio | 1 | Tech Lead + PO | 1 dia útil |
| Atualização de tabela de sinônimos | 1 | Tech Lead + PO | 1 dia útil |
| Adição ou remoção de guardrail | 1 | Tech Lead + PO | 1 dia útil |
| Mudança no schema do campo `sources` | 1 | Tech Lead + PO | 1 dia útil |
| Adição de novo documento à base | 2 | Área + Compliance | 5 dias úteis |
| Deprecação de documento ativo | 2 | Área + Compliance | 5 dias úteis |
| Formalização de gap como documento | 2 | Área + Compliance | 5 dias úteis |
| Atualização de documento com implicação ANTT/jurídica | 2 | Área + Compliance | 5 dias úteis |
| Mudança no mapeamento de encaminhamento (RN-10) | 2 | Operações + Compliance | 5 dias úteis |
| Mudança urgente com risco regulatório ativo | 2 comprimido | Área + Compliance | 24h |

---

## Rastreabilidade e auditoria

Toda aprovação — de qualquer nível — deve gerar registro auditável com:

- ID da mudança
- Nível de aprovação aplicado
- Nome e cargo do aprovador
- Data e hora da aprovação
- Resultado do harness (link para o run)
- Versão do guardrails.md vigente no momento do deploy
- Hash do prompt implantado
- Lista de documentos ativos no índice após o deploy

O registro de Nível 2 precisa adicionalmente conter o documento de aprovação da área (e-mail ou ticket) e o parecer de Compliance. Esses registros são mantidos por no mínimo 2 anos para fins de auditoria contratual e regulatória.
