# Prompt — Análise de Domínio NovaTech

## Contexto do Projeto

```markdown
A NovaTech é uma empresa de médio porte do setor de logística com 1.200 funcionários.
Sua operação depende de um conjunto extenso de documentação interna: manuais de
procedimento operacional, políticas de compliance, tabelas de SLA por tipo de cliente,
regras de cálculo de frete, e normas de segurança de carga.

Hoje, essa documentação está espalhada em três fontes:
- SharePoint corporativo com ~800 documentos (PDFs e Word)
- Wiki interna no Confluence com ~400 páginas
- Pasta de rede com planilhas de referência atualizadas mensalmente

O problema: a equipe de atendimento ao cliente (45 pessoas) gasta em média 12 minutos
por chamado buscando informações nessas fontes para responder dúvidas de clientes sobre
prazos, regras de frete, políticas de devolução e procedimentos de reclamação. Isso gera
atrasos, respostas inconsistentes e frustração tanto dos atendentes quanto dos clientes.

A NovaTech contratou a DB1 para construir um assistente de IA que permita aos atendentes
fazer perguntas em linguagem natural e receber respostas fundamentadas na documentação
oficial da empresa, com indicação da fonte. O assistente será integrado ao ambiente
Microsoft da NovaTech (Teams + SharePoint).

Informações adicionais:
- Volume médio: 320 chamados/dia, ~60% envolvem consulta a documentação
- Documentação atualizada mensalmente por 3 áreas (Operações, Compliance, Comercial)
  sem processo unificado de revisão
- Alguns documentos se contradizem entre versões — a equipe de atendimento hoje resolve
  isso "perguntando para quem sabe"
- A NovaTech já tem licenças Microsoft 365 E3 e está disposta a provisionar Azure AI Services
- Projeto com orçamento para 3 meses (discovery + desenvolvimento + go-live)
- Meta da diretoria: reduzir tempo médio de busca de 12 para menos de 2 minutos por chamado
```

---

## Anexo A — Documentação Simulada NovaTech

### Documentos incluídos na base de conhecimento

1. **POL-001 v3.1** — Política de Devolução de Mercadorias (normativo, vigente)
2. **PROC-042 v1** — Procedimento de Cálculo de Frete Especial (⚠️ sem indicação formal de vigência)
3. **PROC-042 v2** — Procedimento de Cálculo de Frete Especial Revisado (⚠️ sem indicação formal de que substitui v1)
4. **SLA-2024** — Tabela de SLA por Tipo de Cliente (contratual, vigente)
5. **FAQ-Atendimento** — Perguntas Frequentes (⚠️ informal, sem responsável, não validado por Compliance)

### POL-001 v3.1 — Política de Devolução de Mercadorias
- Versão: 3.1 | Atualização: 15/01/2024 | Responsável: Diretoria de Operações
- Prazo geral: 7 dias úteis após recebimento confirmado no tracking (exclui sáb, dom e feriados nacionais)
- Não elegíveis ao processo padrão:
  - Cargas perigosas classes 1-6 ANTT (Res. 5.947/2021)
  - Cargas refrigeradas com cadeia de frio rompida (>30min contínuos fora da faixa da NF, via sensor IoT)
  - Cargas com lacre violado (exceto se documentado no ato com assinatura motorista + recebedor)
  - Para essas: contato com Gestão de Riscos (ramal 4500)
- Procedimento: abertura no Portal do Cliente com CT-e + 3 fotos + motivo
  → triagem em 4h úteis → coleta reversa em até 2 dias úteis → reembolso em até 5 dias úteis
- Devoluções parciais: por volume, reembolso proporcional ao CT-e
- Custos: erro NovaTech = sem custo; desistência do cliente = frete reverso pelo cliente;
  prazo expirado = não elegível (encaminhar ao Comercial)

### PROC-042 v1 — Frete Especial (03/03/2023)
- Status: ⚠️ Coexiste com v2 sem hierarquia clara
- Fórmula: Valor base × Multiplicador regional × Fator de peso
- Multiplicadores: Sul 1.2 / Sudeste 1.0 / Centro-Oeste 1.3 / Nordeste 1.4 / Norte 1.6
- Fatores de peso: 500-1000kg: 1.0 / 1001-3000kg: 1.2 / >3000kg: 1.5
- Prazo adicional: +2 dias úteis
- Cargas >5.000kg: aprovação prévia do gerente de operações regional
- Cargas perigosas >500kg: seguir PROC-043
- Desconto de volume: >10 fretes especiais/mês → negociar com Comercial + aditivo contratual

### PROC-042 v2 — Frete Especial Revisado (10/11/2023)
- Status: ⚠️ Não marca v1 como obsoleta formalmente
- Fórmula: igual à v1
- Multiplicadores REVISADOS: Sul 1.2 / Sudeste 1.0 / Centro-Oeste 1.4 / Nordeste 1.5 / Norte 1.8
- Fatores de peso REVISADOS: 500-1000kg: 1.0 / 1001-3000kg: 1.15 / >3000kg: 1.4
- Prazo adicional REVISADO: +3 dias úteis
- Desconto de volume REVISADO: ≥8 fretes/mês → 5% sobre multiplicador regional;
  >15 fretes/mês → 10%; acima disso → aprovação Diretoria Comercial
- Disposição transitória: chamados abertos antes de 01/12/2023 ainda em processamento
  usam multiplicadores da v1; chamados novos a partir de 01/12/2023 usam v2

### SLA-2024 — Tabela de SLA por Tipo de Cliente
- Versão: 2024.1 | Atualização: 02/01/2024 | Responsável: Comercial + Operações
- Classificação de clientes:
  - Gold: contrato anual >R$500.000 OU >200 operações/mês (revisão semestral)
  - Silver: contrato entre R$100K-500K OU 50-200 operações/mês (revisão semestral)
  - Standard: todos os demais (revisão anual)
  - Não existe tier Platinum ou qualquer outro além dos três acima
- SLAs:
  - Primeira resposta (geral): Gold 2h / Silver 4h / Standard 8h
  - Resolução (geral): Gold 24h / Silver 48h / Standard 72h
  - Primeira resposta (crítico): Gold 30min / Silver 1h / Standard 2h
  - Resolução (crítico): Gold 4h / Silver 8h / Standard 24h
  - Disponibilidade tracking: Gold 99,5% / Silver 99,0% / Standard 98,0%
  - Gerente dedicado: Gold sim / Silver e Standard não
- Incidente crítico: ≥1 de 4 critérios:
  1. Carga valor >R$100K com status desconhecido >6h
  2. Carga perigosa com qualquer irregularidade
  3. >5 chamados do mesmo cliente nas últimas 24h sobre o mesmo problema
  4. Risco à segurança de pessoas
- Penalidades: 1ª violação → registro interno; 2ª → crédito 5% do frete do chamado;
  3ª+ → crédito 10% + reunião obrigatória
- Relógio de SLA: pausa fora de 08h-18h úteis para chamados gerais;
  NÃO pausa para incidentes críticos de clientes Gold

### FAQ-Atendimento (documento informal, não validado)
- Item 3: carga perigosa → ramal 4500 (Gestão de Riscos); não dizer "impossível"
- Item 8: frete especial — existem duas versões do PROC-042; na dúvida usar v2
- Item 15: tier Platinum não existe; foi descontinuado em 2022
- Item 22: seguro de carga — 0,3% do valor declarado (padrão) e 0,8% (perigosa),
  para contratos a partir de 2023; contratos antigos podem ter percentuais diferentes
- Item 27: tracking parado >5 dias — Norte pode levar até 10 dias úteis;
  Sul/Sudeste >3 dias é estranho; abrir chamado de rastreamento; prioridade alta se Gold
  ou valor >R$50.000
- Item 32: carga perigosa com frete expresso — sim, com autorização do Compliance
  e documentação ANTT atualizada; na prática demora ~2 dias para aprovação
- Item 38: carga danificada — registrar ocorrência em até 48h com fotos e laudo;
  encaminhar para sinistros@novatech.com.br (Jurídico)
- Item 41: diferença resposta vs resolução — resposta é 1º retorno; resolução é encerramento
- Item 45: atendente não tem autonomia para dar desconto; encaminhar ao Comercial

### Contradições identificadas no Anexo A

| # | Conflito                                          | Documentos         |
|---|---------------------------------------------------|--------------------|
| C1 | Multiplicadores regionais diferentes             | PROC-042 v1 × v2  |
| C2 | Fator de peso >3000kg: 1.5 × 1.4                | PROC-042 v1 × v2  |
| C3 | Prazo adicional: +2 × +3 dias úteis             | PROC-042 v1 × v2  |
| C4 | Frete expresso p/ carga perigosa sem PROC formal | FAQ × —           |
| C5 | Desconto a partir de 10/mês (FAQ) × 8/mês (v2)  | FAQ × PROC-042 v2 |

### Gaps documentais identificados

| Gap                              | Fonte atual      |
|----------------------------------|------------------|
| Carga danificada em trânsito     | Apenas FAQ Item 38 (informal) |
| Seguro de carga                  | Apenas FAQ Item 22 (informal) |
| Frete padrão (abaixo de 500kg)   | Nenhuma fonte    |
| Processo interno Gestão de Riscos| Referenciado na POL-001, sem PROC |

---

## Prompts da Conversa

### Prompt 1 — Contextualização

```
Leia o Anexo A e se contextualize sobre a NovaTech através do cenário acima.
```

### Prompt 2 — Bounded Contexts

```
Identifique os bounded contexts do assistente NovaTech
(ex: "Atendimento ao Cliente", "Gestão Documental", "SLAs e Contratos",
"Logística de Frete"). Para cada contexto, defina: o que está dentro,
o que está fora, e como se relaciona com os outros.
```

### Prompt 3 — Linguagem Ubíqua

```
Extraia a linguagem ubíqua do domínio a partir do Anexo A: termos que precisam
ser usados de forma consistente por humanos e agentes
(ex: "carga perigosa" sempre significa "classes 1-6 da ANTT",
"frete especial" sempre significa "acima de 500kg").
```

### Prompt 4 — Mapa Visual (React)

```
Elabore o mapa de bounded contexts com linguagem ubíqua.
```

### Prompt 5 — Mapa Visual (SVG v1)

```
Pode gerar em SVG?
```

### Prompt 6 — Mapa Visual (SVG v2 completo)

```
Gere novamente o SVG e inclua também o escopo.
```

### Prompt 7 — Export desta conversa

```
Gere para mim o prompt e output desta conversa.
```

### Prompt 8 — Revisão do prompt

```
Gere novamente o prompt completo sem perfil de analista.
```

### Prompt 9 — Requirements.md (SDD)

```
Escreva o requirements.md do query endpoint seguindo a estrutura SDD.
As prior decisions devem referenciar as ADRs da fase anterior (simuladas no contexto).
Os scope boundaries devem derivar dos bounded contexts definidos acima.

Os outcomes no requirements.md são orientados a resultado do usuário,
não a features técnicas.
```

### Prompt 10 — Export final

```
Gere o prompt e output desta conversa.
```
