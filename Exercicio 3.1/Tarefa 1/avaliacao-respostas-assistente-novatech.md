# Avaliação das Respostas do Assistente — NovaTech

**Projeto:** Assistente de IA para Atendimento ao Cliente  
**Responsável pela avaliação:** Analista de Negócios Sênior  
**Data:** 22/06/2026  
**Documento de referência:** Anexo A — Documentação Simulada NovaTech

---

## Resumo consolidado

| # | Pergunta | Veredicto | Severidade | Problema principal |
|---|---|---|---|---|
| 1 | Prazo de devolução para produtos standard | ⚠️ Parcialmente correta | Média | Omissão de campos obrigatórios do chamado (CT-e) e citação de seção incorreta (3.2 em vez de 3.1) |
| 2 | SLA de resolução para cliente Silver | ⚠️ Parcialmente correta | Alta | Ignora o regime de incidentes críticos (8h úteis), válido para o mesmo tier |
| 3 | Devolução de carga perigosa classe 3 | ⚠️ Parcialmente correta | Média | Encaminhamento incorreto ("supervisor") em vez do correto (Gestão de Riscos, ramal 4500) |
| 4 | Política para carga danificada em transporte | ❌ Incorreta | **Crítica** | Gap documental: não existe POL/PROC formal. Resposta fabricada com confiança Alta e sem fonte válida |
| 5 | SLA do cliente Enterprise | ✅ Correta | — | Comportamento exemplar para ausência de dado: recusou fabricar resposta e orientou encaminhamento |
| 6 | Envio de carga perigosa com frete expresso | ❌ Incorreta | **Crítica** | Fonte informal (FAQ não validado) com confiança Alta declarada; risco regulatório ANTT |

---

## Detalhamento por resposta

### Resposta 1 — Prazo de devolução para produtos standard

- **Fonte citada pelo assistente:** POL-001, seção 3.2
- **Confiança declarada:** Alta
- **Veredicto:** ⚠️ Parcialmente correta

**O que estava correto:**
- Prazo de 7 dias úteis (POL-001, seção 3.1 ✓)
- Orientação de abrir chamado no portal (POL-001, seção 3.3, item 1 ✓)

**O que estava incorreto ou ausente:**
- Citação de seção errada: o prazo está na **seção 3.1** (Prazo geral), não na 3.2 (que trata de exceções)
- Omissão do número do CT-e, obrigatório no chamado (POL-001, seção 3.3)
- Omissão da especificação das 3 fotos obrigatórias: embalagem externa, etiqueta de identificação e conteúdo

**Impacto operacional:** Cliente abre chamado sem CT-e → chamado rejeitado na triagem → retrabalho e atraso.

---

### Resposta 2 — SLA de resolução para cliente Silver

- **Fonte citada pelo assistente:** SLA-2024
- **Confiança declarada:** Alta
- **Veredicto:** ⚠️ Parcialmente correta

**O que estava correto:**
- 48h úteis para resolução de chamados gerais (SLA-2024, seção 2 ✓)

**O que estava incorreto ou ausente:**
- Omissão do SLA de **incidentes críticos**: resolução em até **8h** para cliente Silver (SLA-2024, seção 2)
- Omissão da definição de incidente crítico (SLA-2024, seção 3): carga > R$ 100 mil sem status por 6h; carga perigosa com irregularidade; 5+ chamados do mesmo cliente em 24h; risco a pessoas

**Impacto operacional:** Atendente comunica 48h ao cliente Silver em situação crítica, quando o SLA correto é 8h → descumprimento contratual e penalidade (SLA-2024, seção 4).

---

### Resposta 3 — Devolução de carga perigosa classe 3

- **Fonte citada pelo assistente:** POL-001, seção 3.2
- **Confiança declarada:** Alta
- **Veredicto:** ⚠️ Parcialmente correta

**O que estava correto:**
- Carga perigosa classe 3 (líquidos inflamáveis) não é elegível pelo processo padrão (POL-001, seção 3.2 ✓)
- Classes 1 a 6 da ANTT listadas corretamente

**O que estava incorreto ou ausente:**
- Encaminhamento para "supervisor" não consta em nenhum documento
- O correto é: **Gestão de Riscos, ramal 4500**, para tratamento individual (POL-001, seção 3.2)
- Omissão de que exceções são possíveis mediante tratamento individual (FAQ item 3, como referência contextual)

**Impacto operacional:** Cliente encaminhado à pessoa errada → atraso e possível tratamento inadequado de carga regulada pela ANTT.

---

### Resposta 4 — Política para carga danificada em transporte

- **Fonte citada pelo assistente:** Nenhuma
- **Confiança declarada:** Alta
- **Veredicto:** ❌ Incorreta

**Diagnóstico:**
Não existe documento formal (POL ou PROC) sobre carga danificada na base documental da NovaTech. A única referência disponível é o FAQ item 38, classificado explicitamente como informal e não validado por Compliance.

O assistente:
1. Fabricou informação sem fonte rastreável
2. Introduziu o termo "negligência" — ausente em qualquer documento
3. Omitiu o prazo de 48h para registro da ocorrência (FAQ item 38)
4. Omitiu o encaminhamento para sinistros@novatech.com.br via Jurídico (FAQ item 38)
5. Declarou confiança Alta sem nenhuma fonte

**Comportamento esperado:** Informar que não há política formalizada na documentação disponível; apresentar o que consta no FAQ com ressalva explícita sobre sua natureza informal; orientar escalação ao Jurídico.

**Impacto operacional:** Crítico. Informação incorreta sobre reembolso em contexto de transporte pode gerar implicação contratual e jurídica.

---

### Resposta 5 — SLA do cliente Enterprise

- **Fonte citada pelo assistente:** —
- **Confiança declarada:** Baixa
- **Veredicto:** ✅ Correta

**Por que está correta:**
- Tier Enterprise não existe (SLA-2024, seção 1: apenas Gold, Silver e Standard)
- O assistente recusou fabricar resposta
- Informou corretamente os tiers existentes
- Orientou confirmação ou escalação

**Ponto de melhoria (menor):** O encaminhamento ideal seria ao **Comercial** para análise de viabilidade, conforme SLA-2024, seção 1 — não genericamente ao "supervisor".

**Referência:** FAQ item 15 também confirma que não existe tier Platinum/Enterprise e orienta verificar o contrato.

---

### Resposta 6 — Envio de carga perigosa com frete expresso

- **Fonte citada pelo assistente:** FAQ-Atendimento, item 32
- **Confiança declarada:** Alta
- **Veredicto:** ❌ Incorreta

**Diagnóstico:**

**Problema 1 — Fonte inadequada com confiança indevida:**
O FAQ-Atendimento é classificado explicitamente como documento informal, não validado por Compliance ou Operações. Atribuir confiança Alta a informação extraída exclusivamente desse documento contradiz a própria classificação da fonte.

**Problema 2 — Ausência de documento formal:**
Não existe nenhum PROC ou POL na base que autorize frete expresso para carga perigosa. A PROC-042 (seção 4) remete cargas perigosas acima de 500 kg para a **PROC-043** — que está em processo de revisão pelo Compliance (PROC-042 v2, seção 4).

**Comportamento esperado:** Sinalizar que a informação vem de fonte não validada; indicar que não existe documento formal que suporte a prática; orientar confirmação com Compliance antes de qualquer comunicação ao cliente.

**Impacto operacional:** Crítico. Atendente comunica ao cliente que o envio é possível → expectativa criada que a operação pode não cumprir → risco regulatório envolvendo a ANTT.

---

## Padrão de falha identificado

As respostas 4 e 6 revelam o mesmo anti-padrão sistêmico:

> O assistente atribui confiança Alta sempre que encontra conteúdo semanticamente relacionado à pergunta, independente da qualidade ou do status da fonte.

Esse comportamento é mais perigoso do que uma resposta de baixa confiança, pois o atendente não recebe o sinal de que precisa verificar a informação antes de usá-la.

### Regra de negócio ausente no design do assistente

A confiança declarada deve derivar da **qualidade da fonte**, não da existência de um match semântico.

| Tipo de fonte | Confiança máxima permitida |
|---|---|
| POL / PROC / SLA vigente e sem conflito | Alta |
| PROC com versão em conflito (ex: PROC-042) | Média — com sinalização do conflito |
| FAQ informal não validado | Baixa — com ressalva obrigatória |
| Ausência de documento formal (gap) | Nula — com orientação de escalação |

---

## Referências documentais utilizadas

| Documento | Versão | Responsável |
|---|---|---|
| POL-001 — Política de Devolução de Mercadorias | 3.1 | Diretoria de Operações |
| PROC-042 — Cálculo de Frete Especial | 1.0 | Diretoria Comercial |
| PROC-042-v2 — Cálculo de Frete Especial (Revisado) | 2.0 | Diretoria Comercial |
| SLA-2024 — Tabela de SLA por Tipo de Cliente | 2024.1 | Comercial + Operações |
| FAQ-Atendimento — Perguntas Frequentes | Não controlada | Nenhum (informal) |
