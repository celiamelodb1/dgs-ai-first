# Prompt — Sessão AGENTS.md · NovaTech Assistant

## Contexto e insumos

Esta sessão parte dos seguintes artefatos já produzidos em sessões anteriores:

- **guardrails.md v1.3.0** — 37 guardrails em DEVE / NÃO DEVE / QUANDO EM DÚVIDA,
  classificados como `[DET]` / `[HBR]` / `[PRB]`, com justificativas individuais e
  conexão com 3 incidentes simulados (INC-01, INC-02, INC-03)
- **requirements.md v1.1.0** — requisitos do Query Endpoint com 6 ADRs, 5 UOs,
  4 estados de resposta, 10 RNs, 7 RNFs, schemas de request/response e 6 OQs
- **Anexo A** — documentação de negócio simulada da NovaTech (POL-001, PROC-042 v1 e v2,
  SLA-2024, FAQ-Atendimento)
- **Anexo C** — estrutura do repositório `novatech-assistant` com árvore de diretórios,
  convenções de organização e exemplo de configuração MCP

---

## Prompt 19 — Seção Product Rules & Guardrails do AGENTS.md

```
Com base no Anexo A, Anexo C e os guardrails v1.3.0 gerado. Escreva a seção
"Product Rules & Guardrails" do AGENTS.md. Ele deve conter regras de comportamento
do assistente (derivadas dos guardrails simulados).
```

---

## Prompt 20 — Glossário de Linguagem Ubíqua no AGENTS.md

```
Adicione ao AGENTS.md o Glossário de linguagem ubíqua do domínio que os agentes
precisam conhecer (ex: "cliente Gold", "carga perigosa", "SLA de resolução",
"multiplicador regional", "frete especial").
```

---

## Prompt 21 — Code Constraints no AGENTS.md

```
Adicione ao AGENTS.md, restrições que impactam geração de código
(ex: "toda resposta DEVE incluir o campo `source_document` no JSON de retorno").
```

---

## Prompt 22 — Spec References no AGENTS.md

```
Adicione ao AGENTS.md, referências a documentos de spec no repositório.
Use o Anexo C como base.
```

---

## Prompt 23 — Export desta sessão

```
Gere novamente o prompt e output da conversa de hoje.
```
