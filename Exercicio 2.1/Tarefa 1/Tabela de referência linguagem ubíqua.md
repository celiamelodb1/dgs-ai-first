
---

## Linguagem Ubíqua — Domínio NovaTech

### Como ler este glossário

Cada termo tem:
- **Definição canônica** — o que o termo significa neste domínio, com precisão
- **Fonte** — documento de origem (normativo, contratual ou informal)
- **Termos proibidos** — variações que NÃO devem ser usadas como sinônimos
- **Risco de ambiguidade** — onde o uso inconsistente pode causar problema real

---

## Grupo 1 — Tipos de Carga

---

**Carga Perigosa**
> Mercadoria classificada nas **classes 1 a 6 da ANTT**, conforme Resolução ANTT nº 5.947/2021.

| Classe | Tipo |
|---|---|
| 1 | Explosivos |
| 2 | Gases |
| 3 | Líquidos inflamáveis |
| 4 | Sólidos inflamáveis |
| 5 | Oxidantes e peróxidos |
| 6 | Substâncias tóxicas e infectantes |

- **Fonte:** POL-001 §3.2 (normativo)
- **Termos proibidos:** "carga especial", "carga sensível", "material perigoso" sem referência à classe ANTT
- **Risco:** Classes 7, 8 e 9 da ANTT existem (radioativos, corrosivos, miscelânea) mas **não estão cobertas** pelo POL-001. Se um atendente ou agente usar "carga perigosa" para classe 8, a regra aplicada será incorreta.
- **Impacto no assistente:** toda query sobre "carga perigosa" deve ser respondida com referência explícita às classes 1-6. Se o cliente mencionar outra classe, o assistente deve sinalizar que não há cobertura documental.

---

**Carga Refrigerada**
> Mercadoria que exige controle de temperatura durante o transporte, monitorada por **sensor IoT** com registro contínuo.

- **Condição crítica:** considera-se cadeia de frio rompida quando a temperatura fica fora da faixa especificada na nota fiscal por **mais de 30 minutos contínuos**.
- **Fonte:** POL-001 §3.2 (normativo)
- **Termos proibidos:** "carga fria", "produto refrigerado"
- **Risco:** A definição de ruptura depende de dois parâmetros variáveis — faixa de temperatura (definida na NF, não no documento) e janela de 30 minutos. O assistente não tem acesso à NF nem ao sensor. Deve orientar o atendente a verificar esses dados no sistema de rastreamento.

---

**Carga com Lacre Violado**
> Mercadoria entregue com lacre de segurança rompido.

- **Exceção:** se a violação for **documentada no ato de entrega** com assinatura do motorista e do recebedor, a carga pode seguir o processo padrão de devolução.
- **Fonte:** POL-001 §3.2 (normativo)
- **Risco:** A exceção depende de documentação física gerada no momento da entrega. O assistente não tem acesso a esse registro. Deve sempre perguntar ao atendente se há documentação assinada antes de orientar o fluxo.

---

**Frete Especial**
> Modalidade de frete aplicável a cargas com **peso acima de 500kg**.

- **Fórmula:** `Valor base × Multiplicador regional × Fator de peso`
- **Fonte:** PROC-042 v2 (procedimento — versão vigente pós 01/12/2023)
- **Termos proibidos:** "frete pesado", "frete diferenciado"
- **Risco crítico:** Existe PROC-042 v1 com valores diferentes ainda na base documental. O assistente deve tratar v1 como obsoleta. Qualquer resposta sobre frete especial deve citar explicitamente a versão do documento.
- **Gap:** cargas abaixo de 500kg não têm procedimento documentado. O termo "frete padrão" não possui definição formal na base atual.

---

**Frete Padrão**
> ⚠️ **Termo sem definição formal na base documental.** Inferido como a modalidade aplicável a cargas abaixo de 500kg.

- **Fonte:** ausente — gap documental confirmado
- **Risco:** o assistente não pode responder sobre cálculo de frete padrão com fundamentação. Deve informar o gap e encaminhar ao Comercial.

---

## Grupo 2 — Processo de Devolução

---

**Solicitação de Devolução**
> Pedido formal aberto pelo cliente no **Portal do Cliente** (portal.novatech.com.br) na categoria "Devolução de Mercadoria", contendo: número do CT-e, mínimo de 3 fotos (embalagem externa, etiqueta, conteúdo) e motivo.

- **Fonte:** POL-001 §3.3 (normativo)
- **Termos proibidos:** "pedido de devolução", "solicitação de retorno", "chamado de devolução"
- **Risco:** Abertura de chamado no sistema de chamados (Azure DevOps) é diferente de abertura no Portal do Cliente. São canais distintos para propósitos distintos.

---

**Prazo de Devolução**
> **7 dias úteis** contados a partir da data de recebimento confirmada no sistema de tracking. Exclui sábados, domingos e feriados nacionais.

- **Fonte:** POL-001 §3.1 (normativo)
- **Termos proibidos:** "7 dias corridos", "uma semana"
- **Risco:** A contagem começa na data de recebimento **confirmada no sistema de tracking**, não na data informada pelo cliente. São datas potencialmente diferentes.

---

**Coleta Reversa**
> Serviço de retirada da mercadoria devolvida no endereço do cliente, agendado em até **2 dias úteis** após a aprovação da solicitação de devolução.

- **Fonte:** POL-001 §3.3 (normativo)
- **Termos proibidos:** "coleta de devolução", "frete reverso" (frete reverso é o custo, não o serviço)

---

**Frete Reverso**
> **Custo** do transporte de retorno da mercadoria. Não confundir com coleta reversa (o serviço).

| Motivo da devolução | Responsável pelo custo |
|---|---|
| Defeito ou erro da NovaTech | NovaTech (sem custo ao cliente) |
| Desistência do cliente | Cliente (mesmos multiplicadores do frete original) |
| Prazo expirado | Não elegível ao processo padrão |

- **Fonte:** POL-001 §3.5 (normativo)
- **Risco:** "carga errada" e "avaria em trânsito" estão agrupados como "erro da NovaTech", mas têm processos de investigação distintos. O FAQ Item 38 descreve um fluxo diferente para carga danificada (via Jurídico / sinistros@novatech.com.br) que não está na POL-001.

---

**Devolução Parcial**
> Devolução de **volumes individuais** dentro de uma entrega com múltiplos volumes. O cálculo de reembolso é proporcional ao peso/valor do volume devolvido, conforme o CT-e.

- **Fonte:** POL-001 §3.4 (normativo)
- **Risco:** O assistente precisa saber que cada volume segue o mesmo procedimento da §3.3 individualmente — não há fluxo simplificado para devoluções parciais.

---

**CT-e (Conhecimento de Transporte Eletrônico)**
> Documento fiscal eletrônico que comprova a prestação do serviço de transporte. É o **identificador principal** de uma operação de frete na NovaTech.

- **Fonte:** POL-001 §3.3 (referenciado como dado obrigatório)
- **Termos proibidos:** "nota de transporte", "comprovante de frete"
- **Risco:** O assistente deve sempre solicitar ou referenciar o CT-e ao orientar processos de devolução, rastreamento ou reclamação.

---

## Grupo 3 — SLA e Classificação de Clientes

---

**Tier de Cliente**
> Classificação do cliente em **Gold**, **Silver** ou **Standard**, baseada em volume mensal de operações ou valor do contrato anual.

| Tier | Critério (operador OU) |
|---|---|
| Gold | Contrato anual > R$ 500.000 **OU** > 200 operações/mês |
| Silver | Contrato entre R$ 100K-500K **OU** 50-200 operações/mês |
| Standard | Todos os demais |

- **Fonte:** SLA-2024 §1 (contratual)
- **Termos proibidos:** "categoria de cliente", "nível de cliente", "Platinum" (não existe)
- **Risco de ambiguidade:** o operador **OU** nos critérios cria sobreposição — um cliente com contrato de R$ 600K mas apenas 30 operações/mês é Gold pelo critério de valor, mas Standard pelo critério de volume. O tier mais favorável prevalece, mas isso não está explícito no documento.

---

**Incidente Crítico**
> Chamado classificado como crítico quando atende a **pelo menos um** dos critérios:
> 1. Carga com valor declarado acima de R$ 100.000 com status desconhecido há mais de 6 horas
> 2. Carga perigosa com qualquer irregularidade de documentação ou rastreamento
> 3. Mais de 5 chamados do mesmo cliente nas últimas 24 horas sobre o mesmo problema
> 4. Qualquer situação com risco à segurança de pessoas

- **Fonte:** SLA-2024 §3 (contratual)
- **Termos proibidos:** "chamado urgente", "chamado de alta prioridade", "chamado P1"
- **Risco:** O FAQ Item 27 menciona "prioridade alta se for Gold ou se o valor da carga for acima de R$ 50.000" — limiar diferente do documento formal (R$ 100.000). O assistente deve usar o critério formal do SLA-2024.

---

**SLA de Primeira Resposta**
> Tempo máximo entre a abertura do chamado e o **primeiro retorno ao cliente** (mesmo que seja "estamos verificando").

- **Fonte:** SLA-2024 §2 + FAQ Item 41 (confirma a definição)
- **Distinção crítica:** diferente de SLA de Resolução — não implica que o problema foi resolvido.

---

**SLA de Resolução**
> Tempo máximo entre a abertura do chamado e o **encerramento efetivo do problema**.

- **Fonte:** SLA-2024 §2
- **Risco:** O documento não define o que constitui "resolução" para cada tipo de chamado. Ambiguidade implementável — o assistente deve orientar pelo SLA mas reconhecer que a definição de "resolvido" pode variar por tipo de chamado.

---

**Relógio de SLA**
> Mecanismo de contagem do tempo de SLA, medido a partir do **timestamp de abertura do chamado** no Azure DevOps.

- **Regra de pausa:** pausa fora do horário comercial (08h–18h, dias úteis) para chamados **gerais**
- **Exceção:** para **incidentes críticos de clientes Gold**, o relógio **não pausa** — corre 24/7
- **Fonte:** SLA-2024 §5 (contratual)
- **Risco:** O assistente precisa distinguir entre os dois comportamentos. Uma resposta genérica sobre "o SLA pausa fora do horário" está errada para Gold + crítico.

---

**Penalidade de SLA**
> Crédito concedido ao cliente por violação do SLA, escalonado por recorrência no mês.

| Ocorrência no mês | Consequência |
|---|---|
| 1ª violação | Registro interno, sem impacto contratual |
| 2ª violação | Crédito de 5% sobre o valor do frete do chamado afetado |
| 3ª violação ou mais | Crédito de 10% + reunião obrigatória |

- **Fonte:** SLA-2024 §4 (contratual)
- **Risco:** A penalidade é calculada sobre o frete do **chamado afetado**, não sobre o contrato total. O assistente não tem acesso ao valor do frete do chamado — deve orientar o atendente a verificar no sistema.

---

## Grupo 4 — Documentação e Governança

---

**Documento Normativo**
> Documento que estabelece regras de cumprimento obrigatório. Prefixo: **POL** (política) ou **PROC** (procedimento).

- **Exemplos na base:** POL-001, PROC-042, PROC-043 (referenciado, não disponível)
- **Fonte:** classificação implícita nos documentos

---

**Documento Contratual**
> Documento que estabelece compromissos formais com clientes. Prefixo: **SLA**.

- **Exemplos na base:** SLA-2024
- **Distinção crítica:** violação de documento contratual tem consequência financeira direta. O assistente deve tratar respostas baseadas em SLA com maior nível de precisão e citar a versão explicitamente.

---

**Documento Informal**
> Documento sem responsável formal, sem validação por Compliance ou Operações, mantido colaborativamente pelo time.

- **Exemplo na base:** FAQ-Atendimento
- **Regra de uso pelo assistente:** respostas baseadas exclusivamente em documento informal devem ser sinalizadas com ressalva explícita. Nunca devem ser apresentadas com o mesmo peso de documentos normativos ou contratuais.

---

**Versão Vigente**
> A versão de um documento que deve ser usada como referência atual para respostas e decisões.

- **Risco crítico:** PROC-042 v1 e v2 coexistem na base sem hierarquia formal. A **versão vigente do PROC-042 é a v2** (pós 01/12/2023 pela seção 5 da própria v2), mas isso não foi formalizado pela NovaTech no sistema documental.
- **Impacto no assistente:** o pipeline de ingestão deve marcar v1 como obsoleta e impedir sua recuperação para chamados novos.

---

## Consolidado — Tabela de Referência Rápida

| Termo | Definição canônica resumida | Fonte | Alerta |
|---|---|---|---|
| Carga perigosa | Classes 1-6 ANTT | POL-001 | Classes 7-9 sem cobertura |
| Carga refrigerada | Carga com IoT; ruptura = >30min fora da faixa | POL-001 | Parâmetros na NF, não no doc |
| Frete especial | Acima de 500kg, fórmula × multiplicador × fator | PROC-042 v2 | v1 obsoleta, não arquivada |
| Frete padrão | Abaixo de 500kg | ⚠️ Sem documento | Gap documental |
| Prazo de devolução | 7 dias úteis do recebimento confirmado no tracking | POL-001 | Data do tracking ≠ data do cliente |
| CT-e | Identificador fiscal da operação de transporte | POL-001 | Obrigatório em devoluções |
| Coleta reversa | Serviço de retirada; agendado em 2 dias úteis | POL-001 | Diferente de frete reverso (custo) |
| Tier de cliente | Gold / Silver / Standard — critério por OU | SLA-2024 | Platinum não existe |
| Incidente crítico | 4 critérios; basta 1 | SLA-2024 | FAQ usa limiar diferente (R$50K) |
| Relógio de SLA | Pausa fora do horário, exceto Gold + crítico | SLA-2024 | Dois comportamentos diferentes |
| Documento informal | FAQ — sem validação, uso com ressalva | FAQ | Nunca equiparar a normativo |
| Versão vigente | v2 do PROC-042 para chamados pós 01/12/2023 | PROC-042 v2 §5 | v1 não foi arquivada formalmente |

---

## O que fazer com este glossário no projeto

**Para o time de desenvolvimento:**
incorporar como metadado de chunking — cada chunk deve ser tagueado com os termos ubíquos que contém, para que o retrieval seja preciso mesmo quando o atendente usa sinônimos não-canônicos.

**Para o pipeline RAG:**
o sistema de prompt do assistente deve incluir este glossário como contexto fixo, garantindo que o modelo sempre use a definição canônica ao interpretar a pergunta e ao gerar a resposta.

**Para a NovaTech:**
este glossário é também um entregável de governança — expõe os termos que precisam ser padronizados internamente antes do go-live, especialmente os que têm contradição de fonte.

---

Próximo passo sugerido: modelar o **fluxo de consulta do BC-02** com os estados possíveis de resposta (fundamentada / com ressalva / contradição detectada / gap documental), ou partir para os primeiros PBIs priorizados por risco.