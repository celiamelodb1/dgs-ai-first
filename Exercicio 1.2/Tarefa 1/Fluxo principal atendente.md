## Fluxo principal do atendente — NovaTech com Assistente de IA

---

### Pré-condição
Chamado aberto no portal pelo cliente com número de CT-e, descrição do problema e dados de contato. Sistema registra timestamp e inicia contagem de SLA automaticamente conforme tier do cliente (Gold: 2h resposta / Silver: 4h / Standard: 8h).

---

### Passo 1 — Receber e contextualizar o chamado

O atendente acessa o chamado no Azure DevOps integrado ao Teams. Verifica o tier do cliente para saber o SLA vigente e lê a descrição para entender o tipo de dúvida. Nenhuma consulta documental acontece ainda.

**Saída:** atendente sabe quem é o cliente, qual é o SLA e qual é o problema relatado.

---

### Passo 2 — Classificar a dúvida

O atendente identifica a categoria do chamado entre os tipos mais comuns: prazo de entrega, cálculo de frete, devolução de mercadoria, SLA contratual ou procedimento de reclamação. Essa classificação determina qual documentação será relevante na consulta.

**Saída:** categoria definida, atendente sabe que precisa consultar documentação.

---

### Passo 3 — Consultar o assistente de IA

O atendente abre o assistente diretamente no Teams e formula a pergunta em linguagem natural — por exemplo, *"qual o prazo para devolução de carga refrigerada?"* ou *"qual o multiplicador de frete especial para o Nordeste?"*. O assistente realiza busca semântica sobre os documentos indexados (SharePoint, Confluence e planilhas de rede) e retorna uma resposta fundamentada com o trecho relevante, o nome do documento, a versão e a data de atualização. Se necessário, o atendente refina a pergunta para obter mais precisão.

**Saída:** resposta em linguagem natural com fonte identificada, entregue em menos de 30 segundos.

---

### Passo 4 — Avaliar a resposta

O atendente verifica se a resposta cobre o caso específico do cliente. Confere a versão do documento citado e a data de atualização. Se o assistente sinalizar conflito entre versões — como ocorre com PROC-042 v1 e v2 — o atendente observa qual versão está indicada como vigente para o período do chamado e usa essa como referência.

**Saída:** resposta validada e pronta para uso, ou identificação de que o caso exige tratamento de exceção.

---

### Passo 5 — Tratar exceção (quando aplicável)

Acionado quando a resposta da IA não cobre o caso, quando há conflito documental sem resolução clara, ou quando a situação envolve categorias de risco elevado (carga perigosa, incidente crítico, carga com lacre violado). O atendente escala para o supervisor ou contata diretamente a área responsável — Gestão de Riscos (ramal 4500) para cargas perigosas, Comercial para negociações de prazo expirado ou desconto, Jurídico para carga danificada em trânsito. O caso é registrado com flag de exceção no sistema.

**Saída:** decisão tomada com respaldo da área competente e registro formal no chamado.

---

### Passo 6 — Responder o cliente e encerrar o chamado

O atendente redige a resposta ao cliente com base na documentação oficial consultada, sem transcrevê-la literalmente — adaptando a linguagem ao contexto. Registra no chamado qual documento foi usado como fonte. Encerra o chamado ou o encaminha para a fila correta caso dependa de ação de outra área. O sistema recalcula o tempo de resolução e atualiza as métricas de SLA.

**Saída:** cliente respondido com informação precisa e rastreável, chamado encerrado com fonte registrada.

---

### Condições de desvio

| Situação | Desvio |
|---|---|
| Carga ainda em trânsito | Não aplica POL-001 — consultar PROC-088 (Interceptação de Carga) |
| Carga perigosa (classes 1–6 ANTT) | Não usar processo padrão de devolução — acionar Gestão de Riscos (ramal 4500) |
| Solicitação de devolução após 7 dias úteis | Encaminhar ao Comercial para negociação caso a caso |
| Conflito entre PROC-042 v1 e v2 | Usar v2 para chamados a partir de 01/12/2023; v1 para chamados anteriores ainda em processamento |
| Resposta vem apenas do FAQ informal | Não usar como fonte definitiva — confirmar com área responsável antes de responder |
| Incidente crítico de cliente Gold | SLA não pausa fora do horário comercial — tratamento prioritário imediato |
| Cliente menciona tier "Platinum" | Tier inexistente — orientar que os tiers são Gold, Silver e Standard; solicitar número do contrato |

---

### Invariantes do fluxo

O atendente é sempre o responsável pela decisão final — o assistente de IA é uma ferramenta de apoio, não uma autoridade. Toda resposta ao cliente deve ter uma fonte documental rastreável registrada no chamado. Casos sem cobertura documental formal não devem ser respondidos com base no FAQ informal sem validação prévia da área responsável.