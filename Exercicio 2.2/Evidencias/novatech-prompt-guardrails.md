# Prompt — Sessão Guardrails · Query Endpoint NovaTech

## Contexto e insumos

Esta sessão parte dos seguintes artefatos já produzidos:

- **requirements.md v1.1.0** — requisitos funcionais do Query Endpoint (BC-02)
  com 6 ADRs, 5 User Outcomes, 4 estados de resposta, 10 RNs, 7 RNFs, 10 ACs,
  8 edge cases, schemas de request/response e 6 Open Questions
- **guardrails.md v1.0.0** a **v1.3.0** — produzido incrementalmente nesta sessão

---

## Prompt 14 — Documento de Guardrails

```
A partir do requirements realizado, elabore um documento de guardrails
organizado em:

- DEVE (comportamentos obrigatórios)
- NÃO DEVE (comportamentos proibidos)
- QUANDO EM DÚVIDA (comportamentos de fallback)
```

---

## Prompt 15 — Classificação de Enforcement

```
Para cada guardrail, classifique como:
enforcement via prompt (probabilístico) ou enforcement via código (determinístico)
```

---

## Prompt 16 — Justificativa da Classificação

```
Justifique a classificação dos guardrails.
```

---

## Prompt 17 — Conexão com Incidentes Simulados

```
Conecte cada guardrail a ao menos um dos 3 incidentes
(qual incidente esse guardrail previne?).

3 incidentes simulados onde o assistente falhou durante testes internos:

1. "O assistente respondeu que o prazo de devolução para carga perigosa é
   7 dias, quando na verdade cargas perigosas NÃO podem ser devolvidas."

2. "O assistente citou 'PROC-042, seção 2' mas os multiplicadores informados
   eram da versão 1 (desatualizada), não da v2 (vigente)."

3. "O assistente disse 'Não encontrei informação sobre isso' para uma pergunta
   sobre SLA Gold, mas o documento SLA-2024 estava indexado e continha
   a resposta."
```

---

## Prompt 18 — Export desta sessão

```
Gere o prompt e output da conversa de hoje.
```
