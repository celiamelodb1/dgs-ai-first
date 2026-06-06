Anexado como exemplo uma estrutura de PBI utilizado para o Claude absorver
### Vou te passar um exemplo de estrutura de PBI que quero que absorva como modelo para escrita
Absorvi o modelo. Vou usar essa estrutura como referência para os próximos PBIs que você me pedir.

Resumindo o padrão que identifiquei:

**Estrutura do modelo**

1. **Contexto** — explica o problema de negócio, a situação atual e por que a mudança é necessária
2. **Premissas e Riscos** — o que o sistema assume como verdade (2.1) e o que pode dar errado se a regra falhar (2.2)
3. **Diretório** — caminho de navegação no sistema onde a funcionalidade vive
4. **Requisitos** — detalhamento técnico-funcional dividido por subseções numeradas (4.1, 4.2...), cada uma com uma responsabilidade clara
5. **Critérios de Aceite** — no formato Dado/Quando/Então, numerados como CA01, CA02..., cada um cobrindo um comportamento distinto
6. **Cenários** — casos concretos também em Dado/Quando/Então, com situações específicas de uso real

**Características de escrita que vou seguir**

- Linguagem direta, sem ambiguidade
- Requisitos escritos como obrigações do sistema ("o sistema deve...")
- Critérios de aceite curtos e testáveis — um comportamento por CA
- Cenários mais narrativos que os CAs, descrevendo contexto real de uso
- Numeração consistente em todas as seções

Pode mandar o próximo PBI.

### Enviado o texto com explicação simplificada RAG
```markdown
explicação simplificada do pipeline de RAG: *"Documentos são divididos em pedaços (chunks), transformados em representações numéricas (embeddings), armazenados num banco vetorial, e recuperados por similaridade quando o usuário faz uma pergunta. O LLM então gera uma resposta usando os chunks recuperados como contexto."*
```

Absorvi também. Esse é o modelo técnico de referência para o pipeline de RAG do projeto NovaTech — vou usar essa descrição como base conceitual sempre que precisar escrever requisitos, critérios de aceite ou cenários que envolvam o funcionamento do assistente de IA.

O que entendi como estrutura do pipeline:

**Ingestão**
Documentos → divisão em chunks → geração de embeddings → armazenamento no banco vetorial

**Recuperação**
Pergunta do usuário → embedding da pergunta → busca por similaridade no banco vetorial → recuperação dos chunks relevantes

**Geração**
Chunks recuperados + pergunta → contexto para o LLM → resposta em linguagem natural

Quando você me pedir um PBI que envolva o assistente de IA, vou referenciar esse pipeline corretamente — por exemplo, ao escrever requisitos sobre atualização da base documental, vou mencionar reindexação e reprocessamento de chunks, não apenas "atualizar o sistema". E ao escrever sobre confiança da resposta, vou relacionar ao score de similaridade da recuperação, não a uma noção genérica de "a IA sabe ou não sabe".

Pode mandar o próximo PBI.

### Solicitado criação de PBIs para as jornadas do atendente
Crie os PBIS para as jornadas do atendente quando utilizado a assistente para responder os chamados. Considere as jornadas que você já mapeou:
Fluxo principal do atendente — NovaTech com Assistente de IA Jornada do atendente — Tratamento de respostas de baixa confiança ou divergência com a IA Jornada do atendente — Sinalização de resposta errada, desatualizada ou incompleta ao assistente de IA.
Também considere as 3 restrições e regras de comportamento do assistente

### Primeira versão dos PBIs gerados pelo Claude
Vou criar um PBI por jornada mais um PBI dedicado às restrições de comportamento do assistente — 4 PBIs no total, cada um com escopo próprio e critérios de aceite testáveis.

---

## PBI 01 — Consulta ao assistente de IA durante atendimento de chamado

### 1. Contexto

O time de atendimento da NovaTech (45 atendentes) gasta em média 12 minutos por chamado buscando informações em três fontes desconectadas: SharePoint corporativo (~800 documentos), wiki no Confluence (~400 páginas) e pasta de rede com planilhas mensais. Esse processo gera respostas inconsistentes entre atendentes, atrasos no SLA e frustração operacional.

O assistente de IA deve permitir que o atendente faça perguntas em linguagem natural diretamente no Microsoft Teams e receba respostas fundamentadas na documentação oficial da NovaTech, com indicação da fonte, versão e data de atualização — reduzindo o tempo médio de busca de 12 minutos para menos de 2 minutos por chamado.

O assistente opera via pipeline de RAG: os documentos são divididos em chunks, transformados em embeddings e armazenados em um banco vetorial. Quando o atendente faz uma pergunta, o sistema gera um embedding da consulta, recupera os chunks mais similares e envia o contexto ao LLM para geração da resposta.

### 2. Premissas e Riscos

**2.1 Premissas**
- O assistente está integrado ao Microsoft Teams via Azure AI Services.
- A base documental indexada inclui documentos do SharePoint, Confluence e planilhas de rede com status de versão controlado.
- O FAQ informal (não validado pelo Compliance) não compõe a base principal de recuperação.
- O atendente é sempre o responsável pela decisão final — o assistente é ferramenta de apoio.
- O chamado está registrado no Azure DevOps com timestamp de abertura e tier do cliente identificado.

**2.2 Riscos**
- Chunks mal segmentados podem fazer o assistente recuperar trechos fora de contexto, gerando respostas parcialmente corretas.
- Documentos com versões conflitantes coexistindo na base podem resultar em respostas inconsistentes entre atendentes.
- O atendente sob pressão de SLA pode usar a resposta sem validar a fonte citada.

### 3. Diretório

Teams → Assistente NovaTech → Painel de atendimento → Consulta à base documental

### 4. Requisitos

**4.1 Interface de consulta**

O assistente deve estar acessível ao atendente diretamente no Microsoft Teams, sem necessidade de troca de aplicação durante o atendimento.

O atendente deve poder digitar perguntas em linguagem natural referentes a prazos, regras de frete, políticas de devolução, SLA por tier e procedimentos de reclamação.

**4.2 Recuperação e geração da resposta**

O sistema deve realizar busca semântica por similaridade na base vetorial a partir da pergunta do atendente.

A resposta gerada pelo LLM deve ser fundamentada exclusivamente nos chunks recuperados da base documental oficial.

O sistema deve retornar a resposta em até 30 segundos após o envio da pergunta.

**4.3 Exibição de metadados da fonte**

Toda resposta deve exibir obrigatoriamente:
- Nome do documento utilizado como fonte
- Versão do documento
- Data da última atualização
- Seção ou trecho de referência
- Área responsável pelo documento

**4.4 Refinamento da consulta**

O atendente deve poder reformular a pergunta na mesma sessão de consulta para obter maior precisão, sem perda do contexto anterior da interação.

**4.5 Registro da fonte no chamado**

Ao utilizar uma resposta do assistente, o atendente deve registrar no chamado (Azure DevOps) qual documento foi utilizado como fonte da resposta ao cliente.

### 5. Critérios de Aceite

**CA01 – Resposta com fonte identificada**

Dado que o atendente envie uma pergunta ao assistente
Então o sistema deve retornar uma resposta acompanhada do nome do documento, versão, data de atualização e seção de referência

**CA02 – Tempo de resposta**

Dado que o atendente envie uma pergunta ao assistente
Então o sistema deve retornar a resposta em até 30 segundos

**CA03 – Resposta baseada exclusivamente na base oficial**

Dado que o atendente envie uma pergunta sobre um tema coberto pela base documental
Então a resposta deve ser gerada a partir dos chunks recuperados dos documentos oficiais indexados, sem uso do FAQ informal como fonte primária

**CA04 – Refinamento na mesma sessão**

Dado que o atendente reformule a pergunta após receber uma resposta
Então o assistente deve processar a nova consulta mantendo o contexto da sessão de atendimento

**CA05 – Acessibilidade via Teams**

Dado que o atendente esteja logado no Microsoft Teams
Então o assistente deve estar acessível sem necessidade de troca de aplicação

---

## PBI 02 — Tratamento de respostas de baixa confiança ou discordância do atendente

### 1. Contexto

Durante o atendimento, dois cenários exigem um fluxo específico antes que a resposta chegue ao cliente: o assistente pode sinalizar explicitamente que não encontrou cobertura documental suficiente para responder com segurança (baixa confiança), ou o atendente pode discordar da resposta recebida com base em experiência prévia ou percepção de desatualização.

Em ambos os casos, o chamado não pode avançar para a resposta ao cliente sem que a informação seja validada — seja por reformulação da consulta, seja por escalada para a área competente. O sistema deve suportar esse fluxo de forma estruturada, garantindo rastreabilidade e alimentando o ciclo de melhoria da base documental.

### 2. Premissas e Riscos

**2.1 Premissas**
- O assistente emite sinal explícito de baixa confiança quando o score de similaridade da recuperação estiver abaixo do limiar definido ou quando houver conflito entre documentos recuperados.
- O atendente tem autonomia para questionar qualquer resposta do assistente, independentemente do nível de confiança exibido.
- A orientação recebida da área competente deve ser registrada por escrito no chamado antes de ser usada como base da resposta ao cliente.
- O supervisor é responsável pela decisão final quando a fonte humana não responde dentro do SLA do chamado.

**2.2 Riscos**
- O atendente pode escalar desnecessariamente casos que seriam resolvíveis por reformulação da pergunta, sobrecarregando supervisores e áreas.
- A ausência de registro da orientação recebida impede auditoria posterior e não alimenta o ciclo de melhoria documental.
- Em incidentes críticos de clientes Gold, a escalada em sequência (e não em paralelo) à consulta à IA pode causar violação de SLA.

### 3. Diretório

Teams → Assistente NovaTech → Painel de atendimento → Consulta à base documental → Sinalização de baixa confiança

### 4. Requisitos

**4.1 Sinalização de baixa confiança pelo assistente**

O assistente deve sinalizar explicitamente ao atendente quando a resposta gerada não possuir cobertura documental suficiente, indicando o motivo entre as categorias: gap documental, conflito entre versões, cobertura parcial.

A sinalização deve ser visualmente distinta da resposta de nível confirmado, conforme os três níveis definidos no PBI 04.

**4.2 Bloqueio de resposta ao cliente**

Enquanto o chamado estiver em estado de pendência por baixa confiança ou discordância registrada, o sistema deve impedir o avanço para a etapa de resposta ao cliente sem que o atendente registre a fonte de validação utilizada.

**4.3 Reformulação da consulta**

O sistema deve permitir que o atendente reformule a pergunta ao assistente especificando documento, versão ou período, antes de acionar a escalada humana.

Se a reformulação retornar resposta com nível de confiança confirmado, o fluxo deve retornar ao passo de avaliação da resposta do fluxo principal.

**4.4 Escalada para fonte humana**

O sistema deve disponibilizar ao atendente o encaminhamento estruturado para a área competente conforme o tipo de problema identificado:
- Gap documental → área responsável pelo tema
- Conflito entre versões → Diretoria Comercial
- Conhecimento tácito divergente → supervisor imediato

**4.5 Registro obrigatório da orientação recebida**

O atendente deve registrar no chamado a orientação recebida da fonte humana antes de prosseguir com a resposta ao cliente. O sistema deve exigir esse registro como pré-condição para o avanço do chamado.

**4.6 Registro para melhoria da base**

Ao encerrar o chamado que passou pelo fluxo de baixa confiança, o sistema deve solicitar ao atendente o preenchimento do formulário de feedback com: pergunta feita, motivo da incerteza, área acionada, orientação recebida e sugestão de ação documental.

### 5. Critérios de Aceite

**CA01 – Sinalização de baixa confiança**

Dado que o assistente não encontre cobertura documental suficiente para a pergunta
Então deve exibir sinalização explícita com o motivo da incerteza antes de qualquer conteúdo informativo

**CA02 – Impedimento de resposta sem validação**

Dado que o chamado esteja em estado de pendência por baixa confiança
Quando o atendente tentar avançar para a resposta ao cliente
Então o sistema deve bloquear o avanço e solicitar o registro da fonte de validação

**CA03 – Retorno ao fluxo principal após reformulação**

Dado que o atendente reformule a pergunta ao assistente
Quando a nova resposta retornar com nível de confiança confirmado
Então o sistema deve permitir o avanço para a avaliação da resposta sem necessidade de escalada

**CA04 – Encaminhamento estruturado por tipo de problema**

Dado que o motivo da incerteza seja identificado pelo atendente
Então o sistema deve indicar a área competente correspondente ao tipo de problema

**CA05 – Registro obrigatório antes de responder**

Dado que o atendente tenha recebido orientação da fonte humana
Quando tentar registrar a resposta ao cliente
Então o sistema deve exigir que a orientação esteja registrada no chamado como pré-condição

**CA06 – Formulário de feedback ao encerrar**

Dado que o chamado tenha passado pelo fluxo de baixa confiança
Quando o atendente encerrar o chamado
Então o sistema deve solicitar o preenchimento do formulário de feedback documental

### 6. Cenários

**Cenário 1 – Reformulação resolve a incerteza**

Dado que o assistente sinalize baixa confiança por cobertura parcial
Quando o atendente reformule a pergunta especificando a seção da POL-001
Então o sistema deve retornar resposta com nível confirmado e permitir avanço ao fluxo principal

**Cenário 2 – Escalada por conflito de versões**

Dado que o assistente sinalize conflito entre PROC-042 v1 e v2
Quando o atendente acionar a escalada
Então o sistema deve indicar a Diretoria Comercial como área competente e registrar a pendência no chamado

**Cenário 3 – Incidente crítico Gold em paralelo**

Dado que o chamado seja classificado como incidente crítico de cliente Gold
Quando o assistente sinalizar baixa confiança
Então o sistema deve permitir que a escalada humana ocorra em paralelo à tentativa de reformulação, sem pausar a contagem de SLA

---

## PBI 03 — Sinalização de resposta errada, desatualizada ou incompleta ao assistente

### 1. Contexto

O assistente pode retornar respostas com aparente segurança que, ao serem avaliadas pelo atendente, revelam-se factualmente incorretas, baseadas em versões desatualizadas de documentos, ou omissas em relação a regras e exceções relevantes para o caso do cliente.

Diferentemente do fluxo de baixa confiança, neste cenário o problema não é sinalizado pelo sistema — é identificado pelo atendente com base em seu conhecimento da documentação. O sistema deve suportar o registro estruturado desse feedback e impedir que a resposta problemática chegue ao cliente, garantindo ao mesmo tempo que o caso alimente o ciclo de melhoria da base documental.

### 2. Premissas e Riscos

**2.1 Premissas**
- O atendente tem capacidade de identificar divergências entre a resposta do assistente e a documentação oficial que conhece.
- O feedback deve ser baseado em fonte identificável — nome do documento, versão e seção — e não em percepção subjetiva.
- O FAQ informal não é fonte válida para embasar a correção de uma resposta do assistente.
- Todo feedback registrado deve conter a referência documental correta para ter valor de ação pela equipe de projeto.

**2.2 Riscos**
- Feedbacks sem fonte de referência identificada não permitem ação corretiva e sobrecarregam a equipe de projeto sem resultado.
- Caso o erro já tenha sido comunicado ao cliente antes da identificação, o chamado exige tratamento especial de contato corretivo.
- Feedbacks duplicados sobre o mesmo documento elevam a fila desnecessariamente se não houver deduplicação no sistema.

### 3. Diretório

Teams → Assistente NovaTech → Painel de atendimento → Consulta à base documental → Sinalizar problema na resposta

### 4. Requisitos

**4.1 Ação de sinalização na interface**

O assistente deve disponibilizar, em toda resposta retornada, uma ação explícita para o atendente sinalizar problema — acessível sem necessidade de troca de aplicação ou abertura de formulário externo.

**4.2 Formulário estruturado de feedback**

Ao acionar a sinalização, o sistema deve exibir formulário com os seguintes campos obrigatórios:
- Tipo de problema: errada, desatualizada ou incompleta
- Descrição objetiva: o que a IA disse versus o que o documento correto estabelece
- Documento correto: nome, versão e seção
- Documento problemático: nome e versão utilizada pelo assistente
- Impacto no chamado: erro detectado antes ou após comunicação com o cliente

O sistema deve impedir o envio do formulário sem o preenchimento do documento correto com versão e seção identificadas.

**4.3 Bloqueio da resposta problemática**

Após a sinalização, o sistema deve bloquear o uso da resposta original para o cliente e exigir que o atendente localize e registre a fonte correta antes de prosseguir.

**4.4 Localização da fonte correta**

O sistema deve orientar o atendente a buscar a fonte correta diretamente nos documentos normativos do SharePoint, em ordem de prioridade: POL → PROC → SLA → Confluence. O FAQ informal não deve ser listado como opção de fonte para correção.

Caso o atendente não consiga localizar a fonte corretamente, o sistema deve oferecer o encaminhamento para a área competente pelo tema.

**4.5 Vinculação ao chamado**

O feedback registrado deve ser automaticamente vinculado ao ID do chamado em curso para rastreabilidade cruzada.

**4.6 Notificação de resolução ao atendente**

Quando a equipe de projeto resolver o problema reportado na base documental, o sistema deve notificar o atendente que originou o feedback, fechando o ciclo de forma visível.

**4.7 Deduplicação de feedbacks**

O sistema deve verificar se o problema reportado já foi registrado por outro atendente para o mesmo documento e versão. Em caso positivo, deve registrar a nova ocorrência como incidência adicional ao problema existente, sem criar entrada duplicada na fila de revisão.

### 5. Critérios de Aceite

**CA01 – Ação de sinalização disponível em toda resposta**

Dado que o assistente retorne qualquer resposta ao atendente
Então a ação de sinalizar problema deve estar visível na interface sem necessidade de troca de aplicação

**CA02 – Formulário com campos obrigatórios**

Dado que o atendente acione a sinalização de problema
Quando tentar enviar o formulário sem o documento correto com versão e seção
Então o sistema deve bloquear o envio e indicar os campos obrigatórios pendentes

**CA03 – Bloqueio da resposta original**

Dado que o atendente sinalize problema em uma resposta
Então o sistema deve bloquear o uso dessa resposta para o cliente e exigir registro da fonte correta

**CA04 – FAQ excluído como fonte de correção**

Dado que o atendente esteja localizando a fonte correta após sinalizar problema
Então o FAQ informal não deve ser listado pelo sistema como opção válida de fonte corretiva

**CA05 – Vinculação automática ao chamado**

Dado que o atendente envie o formulário de feedback
Então o registro deve ser automaticamente vinculado ao ID do chamado em curso

**CA06 – Deduplicação de ocorrências**

Dado que o problema reportado já exista na fila de revisão para o mesmo documento e versão
Quando outro atendente registrar o mesmo problema
Então o sistema deve incrementar a contagem de ocorrências do problema existente sem criar entrada duplicada

**CA07 – Notificação de resolução**

Dado que a equipe de projeto resolva um problema reportado na base
Então o atendente que originou o feedback deve receber notificação de encerramento do problema

### 6. Cenários

**Cenário 1 – Resposta com multiplicador errado**

Dado que o assistente retorne o multiplicador regional Norte como 1,6 baseado na PROC-042 v1
Quando o atendente identificar que o chamado é posterior a 01/12/2023 e o correto é 1,8 conforme PROC-042 v2 seção 2.1
Então o atendente deve poder sinalizar o problema, registrar a fonte correta e prosseguir com o valor correto sem comunicar o erro ao cliente

**Cenário 2 – Resposta incompleta sobre devolução**

Dado que o assistente retorne o procedimento padrão de devolução da POL-001 sem mencionar a exceção de carga com lacre violado
Quando o atendente identificar a omissão com base na seção 3.2 da POL-001
Então deve poder registrar o feedback como incompleto com referência à seção omitida

**Cenário 3 – Erro detectado após comunicação com cliente**

Dado que o atendente identifique o erro após já ter respondido ao cliente com a informação incorreta
Quando preencher o formulário de feedback
Então o campo de impacto deve registrar "cliente já comunicado" e o sistema deve notificar o supervisor para avaliação de contato corretivo

---

## PBI 04 — Restrições e regras de comportamento do assistente de IA

### 1. Contexto

O assistente de IA opera sobre uma base documental que contém contradições conhecidas entre versões de documentos, gaps de cobertura formal em temas relevantes e documentos informais não validados pelo Compliance. Nesse contexto, o comportamento do assistente quando não possui resposta confiável é tão crítico quanto a qualidade das respostas que fornece.

Três restrições de comportamento devem ser implementadas para garantir que o atendente nunca receba uma resposta de aparente segurança construída sobre fontes inadequadas: (1) proibição de síntese silenciosa a partir de fontes conflitantes ou informais; (2) declaração explícita de gap documental quando o tema não possui cobertura normativa formal; (3) diferenciação visual e verbal obrigatória entre respostas de níveis de confiança distintos.

### 2. Premissas e Riscos

**2.1 Premissas**
- O pipeline de RAG atribui um score de similaridade a cada chunk recuperado — esse score é usado como um dos insumos para determinar o nível de confiança da resposta.
- Os gaps documentais conhecidos da base NovaTech estão catalogados: seguro de carga, política de carga danificada, frete padrão abaixo de 500kg e procedimento formal da Gestão de Riscos.
- O FAQ informal está identificado na base com metadado de status "não validado" e não deve ser tratado como equivalente a documentos normativos.
- As contradições ativas entre PROC-042 v1 e v2 estão documentadas com o critério de aplicação por período (v1 para chamados anteriores a 01/12/2023 ainda em processamento; v2 para chamados novos a partir dessa data).

**2.2 Riscos**
- A sinalização excessiva de baixa confiança em respostas que poderiam ser confirmadas pode gerar desconfiança do atendente na ferramenta e aumento do tempo de atendimento.
- A ausência de atualização do catálogo de gaps conhecidos torna a Restrição 2 progressivamente menos eficaz conforme novos gaps surgem.
- A diferenciação visual entre níveis de confiança pode ser ignorada pelo atendente em situações de alta pressão de SLA se não for suficientemente destacada na interface.

### 3. Diretório

Teams → Assistente NovaTech → Configuração do assistente → Regras de comportamento e confiança

### 4. Requisitos

**4.1 Restrição 1 — Proibição de síntese silenciosa**

O assistente não deve combinar conteúdo de fontes com status de confiabilidade distintos sem declarar explicitamente ao atendente quais fontes foram utilizadas e qual é o status de cada uma.

Quando a resposta exigir uso de fonte informal (FAQ), o assistente deve declarar essa condição antes de qualquer conteúdo informativo, no formato:

*"Atenção: esta resposta combina [documento oficial, versão] com [FAQ informal, sem validação de Compliance]. A parte relativa a [tema] vem exclusivamente do FAQ e não possui cobertura normativa confirmada. Recomendo validar com [área responsável] antes de responder ao cliente."*

Quando houver conflito entre versões, o assistente deve apresentar as duas versões lado a lado com seus metadados e indicar o critério de aplicação existente, sem escolher silenciosamente uma versão.

**4.2 Restrição 2 — Declaração obrigatória de gap documental**

O assistente deve manter catálogo interno dos gaps documentais conhecidos da base NovaTech e reconhecê-los proativamente quando a pergunta do atendente se aproximar desses temas.

Quando a pergunta recair sobre tema sem cobertura documental formal, o assistente deve retornar no formato:

*"Não existe documento normativo formal na base sobre [tema]. A única referência disponível é o FAQ interno [item], que não foi validado pelo Compliance e pode conter informações desatualizadas. Para responder ao cliente com segurança, acione [área responsável / contato]. Registre o gap no formulário de feedback para priorização de criação documental."*

O assistente não deve retornar ausência de resposta sem explicação do motivo — a declaração do gap é informação operacional obrigatória.

**4.3 Restrição 3 — Diferenciação obrigatória de nível de confiança**

O assistente deve classificar e sinalizar cada resposta em um dos três níveis, com diferenciação visual e textual:

**Nível confirmado:** resposta baseada exclusivamente em documento normativo formal com versão controlada, responsável identificado e data de atualização. Formato de exibição: resposta direta seguida da citação da fonte.

**Nível atenção:** resposta baseada em documento formal com conflito ativo entre versões, documento em revisão pela área responsável, ou cobertura parcial do caso. Formato de exibição: marcação visual de atenção seguida da descrição da condição que reduz a confiança.

**Nível sem cobertura segura:** resposta não encontrada em documento formal ou construída a partir de fonte informal. Formato de exibição: diagnóstico do gap, descrição do que existe na base sobre o tema e rota de escalada — sem conteúdo informativo apresentado como resposta.

**4.4 Catálogo de gaps conhecidos**

O sistema deve manter e permitir atualização do catálogo de gaps documentais reconhecidos. Os gaps iniciais a serem catalogados são:
- Política de carga danificada em trânsito (cobertura apenas no FAQ item 38)
- Percentuais de seguro de carga (cobertura apenas no FAQ item 22)
- Tabela de frete padrão abaixo de 500kg (sem cobertura na base)
- Procedimento formal da Gestão de Riscos para cargas perigosas devolvidas (sem cobertura na base)

**4.5 Atualização do catálogo**

Quando um feedback do tipo "gap documental" for registrado por atendente e confirmado pela equipe de projeto, o gap deve ser adicionado ao catálogo automaticamente para que a Restrição 2 passe a reconhecê-lo nas consultas seguintes.

### 5. Critérios de Aceite

**CA01 – Declaração de fonte informal antes do conteúdo**

Dado que a resposta do assistente utilize o FAQ informal como fonte
Então o assistente deve exibir a declaração de status da fonte antes de qualquer conteúdo informativo, sem exceção

**CA02 – Apresentação lado a lado em conflito de versões**

Dado que a pergunta do atendente envolva tema com conflito ativo entre versões de documentos
Então o assistente deve apresentar as duas versões com seus metadados e o critério de aplicação, sem selecionar silenciosamente uma versão

**CA03 – Declaração de gap para temas sem cobertura formal**

Dado que a pergunta do atendente recaia sobre tema catalogado como gap documental
Então o assistente deve retornar a declaração de gap no formato definido, com indicação da área responsável e orientação de registro de feedback

**CA04 – Nenhuma resposta sem nível de confiança atribuído**

Dado que o assistente retorne qualquer resposta ao atendente
Então a resposta deve estar classificada em um dos três níveis de confiança com sinalização visual distinta

**CA05 – Nível sem cobertura segura sem conteúdo informativo**

Dado que a resposta seja classificada como nível sem cobertura segura
Então o assistente não deve exibir conteúdo informativo como se fosse resposta — apenas o diagnóstico do gap e a rota de escalada

**CA06 – Reconhecimento proativo de gaps conhecidos**

Dado que o atendente faça uma pergunta que se aproxime de tema catalogado como gap
Mesmo que a pergunta não seja idêntica às perguntas que revelaram o gap originalmente
Então o assistente deve reconhecer o gap e retornar a declaração correspondente

**CA07 – Atualização do catálogo após feedback confirmado**

Dado que um feedback do tipo gap documental seja confirmado pela equipe de projeto
Então o gap deve ser incorporado ao catálogo automaticamente e passar a ser reconhecido nas consultas seguintes

### 6. Cenários

**Cenário 1 – Pergunta sobre seguro de carga**

Dado que o atendente pergunte sobre percentual de seguro de carga para contrato padrão
Quando o tema for reconhecido como gap catalogado
Então o assistente deve retornar a declaração de gap indicando que a única referência é o FAQ item 22, sem validação de Compliance, e orientar o atendente a acionar o Comercial

**Cenário 2 – Conflito entre PROC-042 v1 e v2**

Dado que o atendente pergunte sobre multiplicador regional para frete especial no Nordeste
Quando o assistente recuperar chunks de ambas as versões
Então deve exibir os dois valores (v1: 1,4 / v2: 1,5) com os metadados de cada versão e o critério de aplicação por período, classificando a resposta como nível atenção

**Cenário 3 – Resposta confirmada sobre prazo de devolução**

Dado que o atendente pergunte sobre prazo de devolução de mercadoria padrão
Quando o assistente recuperar o trecho da POL-001 v3.1 seção 3.1 com score de similaridade suficiente
Então deve retornar a resposta classificada como nível confirmado com citação do documento, versão, seção e data de atualização

**Cenário 4 – Novo gap identificado via feedback**

Dado que três atendentes registrem feedback de gap documental sobre o mesmo tema não catalogado
Quando a equipe de projeto confirmar o gap
Então o catálogo deve ser atualizado automaticamente e as consultas seguintes sobre o tema devem retornar a declaração de gap correspondente

### Enviado Anexo b Chunk referencia rag e solicitado revisão dos PBIs considerando o anexo b e uma escrita não tecnica e precisa
POr favor reveja os PBIs e considere este documento anexo a respeito do chuncks de referencia. O PBI deve ter uma escrita não técnica e mais precisa utilizando a estrutura e documento de PBI que te passei.

### Gerado novamente os 4 PBIs com os ajustes
Vou reler o modelo de PBI original e os chunks antes de reescrever.Entendido. Vou reescrever os 4 PBIs com linguagem não técnica, removendo jargões de engenharia de IA, usando o formato exato do modelo de referência e incorporando os cenários concretos do Anexo B como base para os critérios de aceite e cenários.

---

## PBI 01 — Consulta ao assistente de IA durante atendimento de chamado

### 1. Contexto

O time de atendimento da NovaTech possui 45 atendentes que respondem em média 320 chamados por dia, dos quais 60% exigem consulta à documentação interna para serem respondidos. Hoje, essa consulta é feita manualmente em três fontes separadas — SharePoint, Confluence e pasta de rede — consumindo em média 12 minutos por chamado e gerando respostas inconsistentes entre atendentes.

O assistente de IA deve permitir que o atendente faça perguntas em linguagem natural diretamente no Microsoft Teams e receba respostas fundamentadas na documentação oficial da NovaTech, com indicação do documento de origem, versão e data de atualização. A expectativa é reduzir o tempo de busca de 12 minutos para menos de 2 minutos por chamado.

O assistente responde apenas com base nos trechos de documentos que encontrou como relevantes para a pergunta feita. Se a informação não estiver presente nos documentos indexados, o assistente não deve inventar uma resposta.

### 2. Premissas e Riscos

**2.1 Premissas**
- O assistente está integrado ao Microsoft Teams e acessível durante o atendimento sem troca de aplicação.
- A base de documentos do assistente inclui os documentos oficiais do SharePoint, do Confluence e as planilhas de referência da pasta de rede.
- O FAQ interno do time de atendimento não compõe a base de documentos oficial do assistente por não ter sido validado pelo Compliance.
- O atendente é sempre o responsável pela decisão final — o assistente é uma ferramenta de apoio à consulta, não uma autoridade.
- O chamado está registrado no Azure DevOps com o tier do cliente e o timestamp de abertura identificados.

**2.2 Riscos**
- Documentos com versões conflitantes coexistindo na base — como PROC-042 v1 e v2 — podem gerar respostas com informações divergentes entre atendentes se o assistente não sinalizar o conflito.
- O atendente sob pressão de prazo de atendimento pode usar a resposta sem verificar o documento de origem citado.
- Perguntas sobre temas sem cobertura documental formal — como frete abaixo de 500kg — não possuem trechos na base para embasar resposta, e o assistente deve declarar a ausência em vez de omiti-la silenciosamente.

### 3. Diretório

Benefícios → Assistente NovaTech → Painel de atendimento → Consulta à documentação

*(Ajustar conforme o menu real do sistema)*

### 4. Requisitos

**4.1 Acesso ao assistente durante o atendimento**

O assistente deve estar disponível para o atendente diretamente no Microsoft Teams, sem necessidade de abrir outro sistema durante o atendimento ao cliente.

**4.2 Consulta em linguagem natural**

O atendente deve poder digitar perguntas sobre prazos, regras de frete, políticas de devolução, SLA por tipo de cliente e procedimentos de reclamação usando suas próprias palavras, sem necessidade de usar termos técnicos ou códigos de documento.

**4.3 Resposta fundamentada em documentação oficial**

O assistente deve responder apenas com base nos trechos dos documentos oficiais encontrados como relevantes para a pergunta. O assistente não deve inventar informações que não estejam presentes nos documentos indexados.

**4.4 Exibição obrigatória da fonte**

Toda resposta deve indicar obrigatoriamente:
- Nome do documento utilizado
- Versão do documento
- Data da última atualização
- Seção ou trecho de referência
- Área responsável pelo documento

**4.5 Tempo de resposta**

O assistente deve retornar a resposta ao atendente em até 30 segundos após o envio da pergunta.

**4.6 Refinamento da pergunta**

O atendente deve poder reformular ou complementar a pergunta na mesma sessão para obter maior precisão, sem perder o histórico da conversa.

**4.7 Registro da fonte no chamado**

Ao utilizar uma resposta do assistente para responder ao cliente, o atendente deve registrar no chamado (Azure DevOps) qual documento foi utilizado como fonte.

### 5. Critérios de Aceite

**CA01 – Resposta com identificação da fonte**

Dado que o atendente envie uma pergunta ao assistente
Então o assistente deve retornar a resposta acompanhada do nome do documento, versão, data de atualização e seção de referência

**CA02 – Resposta dentro do prazo**

Dado que o atendente envie uma pergunta ao assistente
Então o assistente deve retornar a resposta em até 30 segundos

**CA03 – Resposta baseada apenas em documentos oficiais**

Dado que o atendente pergunte sobre um tema coberto pela documentação oficial
Então a resposta deve ser gerada exclusivamente a partir dos trechos dos documentos oficiais indexados, sem uso do FAQ informal como fonte principal

**CA04 – Declaração de ausência de informação**

Dado que o atendente pergunte sobre um tema sem cobertura na base de documentos, como frete abaixo de 500kg
Então o assistente deve declarar que não encontrou informação sobre o tema, sem inventar uma resposta

**CA05 – Refinamento na mesma sessão**

Dado que o atendente reformule a pergunta após receber uma resposta
Então o assistente deve processar a nova consulta mantendo o contexto da conversa em andamento

**CA06 – Acesso sem troca de aplicação**

Dado que o atendente esteja logado no Microsoft Teams durante um atendimento
Então o assistente deve estar acessível sem necessidade de abrir outro sistema

### 6. Cenários

**Cenário 1 – Pergunta com resposta em documento oficial**

Dado que o atendente pergunte "qual o prazo para o cliente solicitar devolução?"
Quando o assistente localizar a informação na POL-001 v3.1, seção 3.1
Então deve retornar que o prazo é de 7 dias úteis após o recebimento confirmado, com indicação do documento, versão e seção

**Cenário 2 – Pergunta sobre SLA de cliente Gold**

Dado que o atendente pergunte "qual o SLA de resposta para cliente Gold?"
Quando o assistente localizar a informação na SLA-2024, seção 2
Então deve retornar que o prazo de primeira resposta é de até 2 horas úteis para chamados gerais e 30 minutos para incidentes críticos, com indicação da fonte

**Cenário 3 – Pergunta sem cobertura documental**

Dado que o atendente pergunte "qual o frete para 300kg para Salvador?"
Quando o assistente não encontrar trechos que cubram frete abaixo de 500kg
Então deve declarar que não há informação sobre frete padrão na base de documentos e orientar o atendente a acionar o Comercial

**Cenário 4 – Refinamento após resposta parcial**

Dado que o atendente pergunte "como funciona a devolução?" e receba uma resposta sobre o prazo geral
Quando reformular para "e no caso de carga perigosa?"
Então o assistente deve complementar a resposta com as exceções da seção 3.2 da POL-001, mantendo o contexto da conversa

---

## PBI 02 — Tratamento de respostas de baixa confiança ou discordância do atendente

### 1. Contexto

Durante o atendimento, dois cenários exigem que o chamado não avance para a resposta ao cliente sem validação prévia: o assistente pode indicar que não encontrou documentação suficiente para responder com segurança, ou o atendente pode discordar da resposta recebida com base em sua experiência ou percepção de desatualização.

Em ambos os casos, o sistema deve oferecer ao atendente um caminho estruturado — reformular a pergunta ou acionar a área responsável — e garantir que a orientação recebida seja registrada antes de a resposta chegar ao cliente. Os registros gerados nesses casos alimentam o processo de revisão e atualização da documentação oficial.

### 2. Premissas e Riscos

**2.1 Premissas**
- O assistente indica explicitamente ao atendente quando não encontrou cobertura documental suficiente para a pergunta, informando o motivo.
- O atendente pode questionar qualquer resposta do assistente, independentemente do nível de confiança exibido.
- A orientação recebida da área responsável deve ser registrada por escrito no chamado antes de ser usada como base da resposta ao cliente.
- O supervisor do atendente é responsável pela decisão final quando a área responsável não retornar dentro do prazo do SLA do chamado.

**2.2 Riscos**
- O atendente pode acionar a área responsável em casos que seriam resolvíveis por reformulação da pergunta ao assistente, gerando sobrecarga desnecessária.
- Chamados de incidentes críticos de clientes Gold não pausam o SLA fora do horário comercial — a escalada para validação humana não pode atrasar o atendimento nesses casos.
- Orientações recebidas verbalmente, sem registro escrito no chamado, impedem auditoria posterior e não contribuem para a melhoria da documentação.

### 3. Diretório

Benefícios → Assistente NovaTech → Painel de atendimento → Consulta à documentação → Resposta com sinalização de incerteza

*(Ajustar conforme o menu real do sistema)*

### 4. Requisitos

**4.1 Sinalização de incerteza pelo assistente**

Quando o assistente não encontrar documentação suficiente para responder com segurança, deve indicar ao atendente o motivo da incerteza entre as situações possíveis:
- Tema sem documento formal na base
- Documentos com versões em conflito para o mesmo tema
- Documento existente que cobre o tema apenas parcialmente

**4.2 Bloqueio de avanço sem validação**

Enquanto o chamado estiver com resposta pendente de validação, o sistema deve impedir o registro da resposta ao cliente sem que o atendente informe a fonte utilizada para validar a informação.

**4.3 Reformulação da pergunta antes da escalada**

O sistema deve permitir que o atendente reformule a pergunta ao assistente — especificando o documento, a versão ou o período do chamado — antes de acionar a área responsável.

Se a reformulação retornar uma resposta com cobertura documental confirmada, o chamado deve retornar ao fluxo normal de atendimento sem necessidade de escalada.

**4.4 Encaminhamento para área responsável**

O sistema deve indicar ao atendente a área responsável pelo tema conforme o tipo de incerteza identificado:
- Tema sem cobertura formal: área responsável pelo assunto
- Conflito entre versões de documento: Diretoria Comercial
- Discordância com base em prática conhecida: supervisor imediato

**4.5 Registro obrigatório da orientação recebida**

O sistema deve exigir que o atendente registre no chamado a orientação recebida da área responsável antes de permitir o avanço para a resposta ao cliente.

**4.6 Registro para atualização da documentação**

Ao encerrar um chamado que passou por validação humana, o sistema deve solicitar ao atendente o preenchimento de um formulário de feedback com: a pergunta feita ao assistente, o motivo da incerteza, a área acionada, a orientação recebida e a sugestão de atualização documental.

### 5. Critérios de Aceite

**CA01 – Sinalização de incerteza com motivo identificado**

Dado que o assistente não encontre cobertura documental suficiente para a pergunta do atendente
Então deve exibir a sinalização de incerteza indicando o motivo, antes de qualquer conteúdo informativo

**CA02 – Impedimento de resposta sem validação registrada**

Dado que o chamado esteja com resposta pendente de validação
Quando o atendente tentar registrar a resposta ao cliente
Então o sistema deve bloquear o avanço e solicitar o registro da fonte de validação utilizada

**CA03 – Retorno ao fluxo normal após reformulação bem-sucedida**

Dado que o atendente reformule a pergunta ao assistente com maior especificidade
Quando o assistente retornar resposta com cobertura documental confirmada
Então o sistema deve liberar o chamado para o fluxo normal sem necessidade de acionar área responsável

**CA04 – Indicação da área responsável por tipo de incerteza**

Dado que o motivo da incerteza seja identificado pelo atendente
Então o sistema deve indicar a área responsável correspondente ao tipo de problema identificado

**CA05 – Exigência de registro antes de responder ao cliente**

Dado que o atendente tenha recebido orientação da área responsável
Quando tentar registrar a resposta ao cliente
Então o sistema deve exigir que a orientação recebida esteja registrada no chamado como condição para prosseguir

**CA06 – Solicitação de formulário de feedback ao encerrar**

Dado que o chamado tenha passado por validação humana
Quando o atendente encerrar o chamado
Então o sistema deve solicitar o preenchimento do formulário de feedback para atualização da documentação

### 6. Cenários

**Cenário 1 – Reformulação resolve o conflito entre versões**

Dado que o atendente pergunte "qual o multiplicador de frete para o Nordeste?"
E o assistente sinalize conflito entre PROC-042 v1 (1,4) e PROC-042 v2 (1,5)
Quando o atendente reformule especificando que o chamado foi aberto após 01/12/2023
Então o assistente deve retornar o multiplicador 1,5 da PROC-042 v2 com cobertura confirmada, sem necessidade de escalada

**Cenário 2 – Escalada por tema sem documento formal**

Dado que o atendente pergunte "qual o percentual de seguro de carga para contrato padrão?"
E o assistente identifique que o tema não possui documento formal na base
Quando o assistente sinalizar a ausência de cobertura
Então o sistema deve indicar o Comercial como área responsável e registrar a pendência no chamado

**Cenário 3 – Discordância do atendente com a resposta recebida**

Dado que o atendente receba uma resposta sobre prazo de frete especial indicando +2 dias úteis
E o atendente saiba que a prática atual é de +3 dias úteis conforme versão mais recente
Quando o atendente registrar discordância
Então o sistema deve indicar o supervisor imediato como referência e exigir registro da orientação recebida antes de prosseguir

**Cenário 4 – Incidente crítico Gold com incerteza simultânea**

Dado que o chamado seja classificado como incidente crítico de cliente Gold
E o assistente sinalize incerteza na resposta
Então o sistema deve permitir que o atendente inicie a escalada para validação humana sem pausar a contagem do SLA do chamado

---

## PBI 03 — Sinalização de resposta incorreta, desatualizada ou incompleta ao assistente

### 1. Contexto

O assistente pode retornar respostas com aparente segurança que, ao serem analisadas pelo atendente, revelam-se incorretas, baseadas em versões desatualizadas de documentos, ou omissas em relação a regras e exceções relevantes para o caso do cliente.

Diferentemente do fluxo de incerteza, aqui o problema não é identificado pelo sistema — é o atendente quem percebe a divergência com base em seu conhecimento da documentação oficial. O sistema deve oferecer um caminho estruturado para que o atendente registre o problema com precisão, bloqueie o uso da resposta incorreta e contribua para a correção da base de documentos do assistente.

### 2. Premissas e Riscos

**2.1 Premissas**
- O atendente é capaz de identificar divergências entre a resposta do assistente e a documentação oficial que conhece.
- Todo registro de problema deve identificar o documento correto com nome, versão e seção — registros sem essa referência não permitem ação corretiva pela equipe responsável.
- O FAQ interno do time de atendimento não é fonte válida para embasar a correção de uma resposta do assistente.
- Quando o mesmo problema for reportado por mais de um atendente, os registros devem ser consolidados em uma única ocorrência com contagem de incidências.

**2.2 Riscos**
- Registros de problema sem identificação do documento correto sobrecarregam a equipe responsável sem resultado.
- Se o atendente identificar o erro somente após já ter comunicado a informação incorreta ao cliente, o chamado exige tratamento especial com avaliação do supervisor para contato corretivo.
- A ausência de notificação de resolução ao atendente que reportou o problema desestimula novos registros e enfraquece o ciclo de melhoria.

### 3. Diretório

Benefícios → Assistente NovaTech → Painel de atendimento → Consulta à documentação → Sinalizar problema na resposta

*(Ajustar conforme o menu real do sistema)*

### 4. Requisitos

**4.1 Ação de sinalização disponível em toda resposta**

O assistente deve disponibilizar, em toda resposta retornada ao atendente, uma ação visível para sinalizar problema — acessível diretamente na interface, sem necessidade de abrir outro sistema.

**4.2 Formulário de registro do problema**

Ao acionar a sinalização, o sistema deve exibir formulário com os seguintes campos:
- Tipo de problema: incorreta, desatualizada ou incompleta
- O que o assistente disse versus o que o documento correto estabelece
- Documento correto: nome, versão e seção
- Documento utilizado pelo assistente: nome e versão
- Situação do chamado: problema identificado antes ou depois de comunicar o cliente

O sistema deve impedir o envio do formulário sem a identificação do documento correto com versão e seção.

**4.3 Bloqueio da resposta sinalizada**

Após o registro do problema, o sistema deve bloquear o uso da resposta original para o cliente e exigir que o atendente identifique e registre a fonte correta antes de prosseguir com o atendimento.

**4.4 Orientação para localização da fonte correta**

O sistema deve orientar o atendente a buscar a informação correta diretamente nos documentos normativos, na seguinte ordem de prioridade: políticas (POL) → procedimentos (PROC) → tabelas de SLA → Confluence.

O FAQ interno não deve ser apresentado como opção válida de fonte para correção.

Se o atendente não conseguir localizar a informação, o sistema deve oferecer o encaminhamento para a área responsável pelo tema.

**4.5 Vinculação do registro ao chamado**

O registro do problema deve ser automaticamente vinculado ao número do chamado em curso para rastreabilidade.

**4.6 Consolidação de registros repetidos**

Quando o problema reportado já existir na fila de revisão para o mesmo documento e versão, o sistema deve registrar a nova ocorrência como incidência adicional ao problema existente, sem criar um novo registro separado.

**4.7 Notificação de resolução ao atendente**

Quando a equipe responsável resolver o problema reportado na base de documentos, o atendente que originou o registro deve receber uma notificação de encerramento.

### 5. Critérios de Aceite

**CA01 – Ação de sinalização disponível em toda resposta**

Dado que o assistente retorne qualquer resposta ao atendente
Então a ação de sinalizar problema deve estar visível na tela sem necessidade de abrir outro sistema

**CA02 – Envio bloqueado sem documento correto identificado**

Dado que o atendente acione a sinalização de problema
Quando tentar enviar o formulário sem informar o documento correto com versão e seção
Então o sistema deve bloquear o envio e indicar os campos obrigatórios não preenchidos

**CA03 – Bloqueio do uso da resposta sinalizada**

Dado que o atendente sinalize problema em uma resposta do assistente
Então o sistema deve impedir que essa resposta seja usada para comunicar o cliente, exigindo o registro da fonte correta antes de prosseguir

**CA04 – FAQ excluído como opção de fonte para correção**

Dado que o atendente esteja buscando a fonte correta após sinalizar um problema
Então o FAQ interno não deve ser listado pelo sistema como opção válida de documento para embasar a correção

**CA05 – Registro vinculado automaticamente ao chamado**

Dado que o atendente envie o formulário de registro de problema
Então o registro deve ser automaticamente vinculado ao número do chamado em curso

**CA06 – Consolidação de ocorrências do mesmo problema**

Dado que o problema já exista na fila de revisão para o mesmo documento e versão
Quando outro atendente registrar o mesmo problema
Então o sistema deve incrementar a contagem de ocorrências do registro existente, sem criar entrada duplicada

**CA07 – Notificação ao atendente após resolução**

Dado que a equipe responsável resolva um problema reportado na base de documentos
Então o atendente que originou o registro deve receber uma notificação informando o encerramento

### 6. Cenários

**Cenário 1 – Multiplicador de frete desatualizado**

Dado que o assistente retorne o multiplicador regional Norte como 1,6 com base na PROC-042 v1
E o chamado tenha sido aberto após 01/12/2023, quando o correto é 1,8 conforme PROC-042 v2, seção 2.1
Quando o atendente registrar o problema como desatualizado
Então o sistema deve bloquear o uso do valor 1,6, exigir o registro da PROC-042 v2 seção 2.1 como fonte correta, e prosseguir o atendimento com o valor correto sem comunicar o erro ao cliente

**Cenário 2 – Resposta incompleta sobre exceções de devolução**

Dado que o assistente retorne apenas o prazo de 7 dias úteis da POL-001, seção 3.1, sem mencionar a exceção de cargas perigosas descrita na seção 3.2
Quando o atendente identificar a omissão porque o chamado envolve carga perigosa
Então deve poder registrar o problema como incompleto, indicando a POL-001 v3.1 seção 3.2 como trecho omitido

**Cenário 3 – Inversão de regra sobre carga perigosa**

Dado que o assistente retorne que carga perigosa pode ser devolvida pelo processo padrão
E a POL-001 v3.1, seção 3.2, estabelece que cargas perigosas não são elegíveis para devolução padrão
Quando o atendente registrar o problema como incorreto
Então o sistema deve bloquear a resposta, registrar a POL-001 v3.1 seção 3.2 como fonte correta e notificar a equipe responsável para revisão prioritária da base

**Cenário 4 – Erro identificado após comunicação com o cliente**

Dado que o atendente identifique o erro somente após já ter comunicado a informação incorreta ao cliente
Quando preencher o formulário com a situação "cliente já comunicado"
Então o sistema deve notificar o supervisor para avaliação da necessidade de contato corretivo com o cliente

---

## PBI 04 — Regras de comportamento do assistente quando não há resposta segura

### 1. Contexto

A base de documentos do assistente da NovaTech contém situações que exigem comportamento específico: há documentos em versões conflitantes sem hierarquia formal definida, temas relevantes para o atendimento sem cobertura em documento oficial, e um FAQ interno do time que não foi validado pelo Compliance. Nesse cenário, a forma como o assistente se comporta quando não possui resposta segura é tão importante quanto a qualidade das respostas que fornece.

Três regras de comportamento devem ser implementadas para garantir que o atendente nunca receba uma resposta apresentada com segurança quando a informação de base é insuficiente, conflitante ou não validada: proibição de combinar fontes de qualidade diferente sem declarar isso; declaração obrigatória quando o tema não possui cobertura em documento oficial; e diferenciação clara entre respostas de graus de confiança distintos.

### 2. Premissas e Riscos

**2.1 Premissas**
- O FAQ interno está identificado na base com o status de "não validado pelo Compliance" e não deve ser tratado como equivalente aos documentos oficiais.
- Os conflitos documentais ativos da base NovaTech estão mapeados, em especial a coexistência da PROC-042 v1 e v2 com valores distintos de multiplicadores regionais, fatores de peso e prazo adicional de entrega.
- O critério de aplicação das versões em conflito está definido: PROC-042 v1 para chamados abertos antes de 01/12/2023 ainda em processamento; PROC-042 v2 para chamados novos a partir dessa data.
- Os temas sem cobertura em documento formal são conhecidos: seguro de carga, política de carga danificada em trânsito, frete abaixo de 500kg e procedimento formal da Gestão de Riscos para cargas perigosas.

**2.2 Riscos**
- Sinalizar incerteza com excesso de frequência em respostas que poderiam ser confirmadas pode gerar desconfiança do atendente no assistente e aumentar o tempo de atendimento.
- Se novos gaps documentais surgirem sem serem incorporados ao catálogo de temas sem cobertura, o assistente passará a não reconhecê-los e poderá responder com base no FAQ informal sem sinalização.
- A diferenciação visual entre os graus de confiança pode ser ignorada pelo atendente em situações de pressão de SLA se não estiver suficientemente destacada na interface.

### 3. Diretório

Benefícios → Assistente NovaTech → Configuração → Regras de comportamento da resposta

*(Ajustar conforme o menu real do sistema)*

### 4. Requisitos

**4.1 Regra 1 — Proibição de combinar fontes de qualidade diferente sem declaração**

O assistente não deve combinar trechos de documentos oficiais com conteúdo do FAQ informal em uma mesma resposta sem informar explicitamente ao atendente quais fontes foram utilizadas e qual é o status de cada uma.

Quando a resposta utilizar o FAQ como fonte, o assistente deve declarar essa condição antes de qualquer conteúdo informativo, no seguinte formato:

*"Atenção: esta resposta inclui informação do FAQ interno do time de atendimento, que não foi validado pelo Compliance e pode estar desatualizado. A parte sobre [tema] vem exclusivamente do FAQ. Recomendo confirmar com [área responsável] antes de responder ao cliente."*

Quando houver documentos em versões conflitantes para o mesmo tema, o assistente deve apresentar as informações das duas versões com os respectivos dados de identificação e indicar o critério de uso de cada versão, sem escolher silenciosamente apenas uma delas.

**4.2 Regra 2 — Declaração obrigatória de ausência de cobertura documental**

O assistente deve manter uma lista de temas sem cobertura em documento formal e reconhecer automaticamente quando uma pergunta se enquadra nesses temas, mesmo que a pergunta seja formulada de forma diferente das que originaram o mapeamento.

Quando a pergunta recair sobre tema sem cobertura documental formal, o assistente deve retornar no seguinte formato:

*"Não há documento oficial na base sobre [tema]. A única referência disponível é o FAQ interno [item], que não foi validado pelo Compliance. Para responder ao cliente com segurança, acione [área responsável]. Registre a ausência no formulário de feedback para priorização pela equipe responsável."*

O assistente não deve retornar ausência de resposta sem explicar o motivo — informar ao atendente que o tema não possui cobertura documental é uma resposta operacionalmente útil.

**4.3 Regra 3 — Diferenciação obrigatória entre graus de confiança**

O assistente deve classificar e sinalizar cada resposta em um dos três graus, com diferenciação visual e textual clara:

**Confirmado:** a resposta é baseada exclusivamente em documento oficial com versão controlada, responsável identificado e data de atualização. O assistente apresenta a resposta diretamente, seguida da identificação da fonte.

**Atenção:** a resposta é baseada em documento oficial, mas há pelo menos uma das seguintes condições: conflito com outra versão ativa do mesmo documento, documento em processo de revisão pela área responsável, ou cobertura parcial que não abrange todas as dimensões da pergunta. O assistente exibe sinalização de atenção e descreve a condição que reduz a confiança antes de apresentar o conteúdo.

**Sem cobertura segura:** a resposta não foi encontrada em documento oficial, ou existe apenas no FAQ informal não validado. O assistente apresenta o diagnóstico da ausência, o que existe na base sobre o tema e a indicação da área responsável para o atendente acionar — sem apresentar conteúdo informativo como se fosse uma resposta confirmada.

**4.4 Catálogo de temas sem cobertura documental**

O sistema deve manter e permitir atualização do catálogo de temas reconhecidos como sem cobertura documental formal. Os temas iniciais a serem cadastrados são:
- Seguro de carga: percentuais e condições (cobertura apenas no FAQ, item 22)
- Carga danificada em trânsito: procedimento de registro e reembolso (cobertura apenas no FAQ, item 38)
- Frete padrão para cargas abaixo de 500kg: não há documento na base
- Procedimento formal da Gestão de Riscos para devoluções de carga perigosa (referência apenas ao ramal 4500 na POL-001, sem procedimento detalhado)

**4.5 Atualização automática do catálogo**

Quando um registro de feedback do tipo "tema sem cobertura" for confirmado pela equipe responsável, o tema deve ser incorporado automaticamente ao catálogo para que o assistente passe a reconhecê-lo nas consultas seguintes.

### 5. Critérios de Aceite

**CA01 – Declaração antes do conteúdo quando FAQ for utilizado**

Dado que a resposta do assistente utilize o FAQ interno como fonte
Então o assistente deve exibir a declaração de status da fonte antes de qualquer conteúdo informativo, sem exceção

**CA02 – Apresentação das duas versões em conflito sem escolha silenciosa**

Dado que a pergunta envolva tema com versões conflitantes na base, como multiplicadores regionais de frete
Então o assistente deve apresentar os valores das duas versões com seus dados de identificação e o critério de uso por período, sem selecionar apenas uma versão sem informar o atendente

**CA03 – Declaração de ausência de cobertura para temas sem documento formal**

Dado que o atendente pergunte sobre tema catalogado como sem cobertura documental, como frete abaixo de 500kg ou seguro de carga
Então o assistente deve retornar a declaração de ausência no formato definido, com indicação da área responsável e orientação de registro no formulário de feedback

**CA04 – Toda resposta classificada em um dos três graus**

Dado que o assistente retorne qualquer resposta ao atendente
Então a resposta deve estar classificada em um dos três graus de confiança com sinalização visual distinta

**CA05 – Grau sem cobertura segura sem conteúdo informativo apresentado como resposta**

Dado que a resposta seja classificada como sem cobertura segura
Então o assistente não deve apresentar conteúdo informativo como se fosse uma resposta confirmada — apenas o diagnóstico da ausência, o que existe na base e a área a acionar

**CA06 – Reconhecimento proativo de temas sem cobertura**

Dado que o atendente faça uma pergunta sobre tema catalogado como sem cobertura documental
Mesmo que a pergunta seja formulada de forma diferente da que originou o mapeamento do tema
Então o assistente deve reconhecer o enquadramento e retornar a declaração de ausência correspondente

**CA07 – Atualização do catálogo após confirmação de novo tema sem cobertura**

Dado que um registro de feedback do tipo "tema sem cobertura" seja confirmado pela equipe responsável
Então o tema deve ser incorporado automaticamente ao catálogo e reconhecido nas consultas seguintes

### 6. Cenários

**Cenário 1 – Pergunta sobre seguro de carga**

Dado que o atendente pergunte "qual o percentual de seguro de carga para contrato padrão?"
Quando o assistente identificar que o tema está catalogado como sem cobertura documental formal
Então deve retornar a declaração de ausência indicando que a única referência é o FAQ interno, item 22, sem validação do Compliance, e orientar o atendente a acionar o Comercial antes de responder ao cliente

**Cenário 2 – Pergunta sobre frete que recupera versões conflitantes**

Dado que o atendente pergunte "qual o multiplicador de frete para o Nordeste em um chamado de hoje?"
Quando o assistente encontrar trechos da PROC-042 v1 (multiplicador 1,4) e da PROC-042 v2 (multiplicador 1,5)
Então deve classificar a resposta como grau Atenção, apresentar os dois valores com identificação de cada versão e informar que chamados a partir de 01/12/2023 devem usar a PROC-042 v2

**Cenário 3 – Pergunta com resposta confirmada em documento oficial**

Dado que o atendente pergunte "qual o prazo para o cliente solicitar devolução de mercadoria padrão?"
Quando o assistente encontrar a informação na POL-001 v3.1, seção 3.1, sem conflito com outros trechos
Então deve classificar a resposta como grau Confirmado e retornar que o prazo é de 7 dias úteis, com indicação do documento, versão, seção e data de atualização

**Cenário 4 – Pergunta sobre tier inexistente**

Dado que o atendente pergunte "qual o SLA para cliente Platinum?"
Quando o assistente encontrar na SLA-2024, seção 1, que não existem outros tiers além de Gold, Silver e Standard
Então deve classificar a resposta como grau Confirmado e informar que o tier Platinum não existe na NovaTech, orientando o atendente a verificar o número do contrato do cliente para identificar o tier correto

**Cenário 5 – Novo tema sem cobertura identificado via feedback**

Dado que três atendentes registrem feedback indicando ausência de documento formal sobre o mesmo tema
Quando a equipe responsável confirmar a ausência
Então o catálogo deve ser atualizado automaticamente e as consultas seguintes sobre o tema devem retornar a declaração de ausência correspondente

### Solicitado para alterar os critérios de aceite conforme o modelo encaminhado, no qual usa o termo Garantir
nos critérios de aceite dos PBIs preciso que use o conceito do modelo que anexei aqui, onde é usado o termo Garantir, por exemplo: Garantir que a assistente sinalize nas respostas quando não tem certeza de que a fonte está correta.

### Gerado novos critérios de aceite com base no ajuste solicitado para os 4 PBIs
Entendido. O modelo usa "Garantir que..." como abertura dos critérios de aceite, no lugar do formato Dado/Quando/Então. Vou reescrever apenas as seções de Critérios de Aceite dos 4 PBIs com esse padrão.

---

## PBI 01 — Critérios de Aceite

**CA01 – Resposta com identificação da fonte**
Garantir que toda resposta do assistente exiba o nome do documento utilizado, a versão, a data de atualização e a seção de referência antes de ser apresentada ao atendente.

**CA02 – Resposta dentro do prazo**
Garantir que o assistente retorne a resposta ao atendente em até 30 segundos após o envio da pergunta.

**CA03 – Resposta baseada apenas em documentos oficiais**
Garantir que o assistente utilize exclusivamente os documentos oficiais indexados como base para suas respostas, sem usar o FAQ informal como fonte principal.

**CA04 – Declaração de ausência de informação**
Garantir que o assistente informe explicitamente ao atendente quando não encontrar informação sobre o tema consultado, sem apresentar uma resposta inventada no lugar.

**CA05 – Refinamento na mesma sessão**
Garantir que o atendente possa reformular a pergunta após receber uma resposta e que o assistente processe a nova consulta mantendo o contexto da conversa em andamento.

**CA06 – Acesso sem troca de aplicação**
Garantir que o assistente esteja acessível ao atendente diretamente no Microsoft Teams, sem necessidade de abrir outro sistema durante o atendimento.

---

## PBI 02 — Critérios de Aceite

**CA01 – Sinalização de incerteza com motivo identificado**
Garantir que o assistente informe ao atendente quando não encontrar cobertura documental suficiente para a pergunta, indicando o motivo antes de qualquer conteúdo informativo.

**CA02 – Impedimento de resposta sem validação registrada**
Garantir que o sistema bloqueie o registro da resposta ao cliente enquanto o chamado estiver com validação pendente, liberando o avanço somente após o atendente informar a fonte utilizada para validar a informação.

**CA03 – Retorno ao fluxo normal após reformulação bem-sucedida**
Garantir que o chamado retorne ao fluxo normal de atendimento quando o atendente reformular a pergunta e o assistente retornar resposta com cobertura documental confirmada, sem necessidade de acionar área responsável.

**CA04 – Indicação da área responsável por tipo de incerteza**
Garantir que o sistema indique ao atendente a área responsável correspondente ao tipo de incerteza identificado no chamado.

**CA05 – Exigência de registro antes de responder ao cliente**
Garantir que o sistema exija o registro da orientação recebida pela área responsável como condição obrigatória para o atendente prosseguir com a resposta ao cliente.

**CA06 – Solicitação de formulário de feedback ao encerrar**
Garantir que o sistema solicite o preenchimento do formulário de feedback para atualização da documentação ao encerrar chamados que passaram por validação humana.

---

## PBI 03 — Critérios de Aceite

**CA01 – Ação de sinalização disponível em toda resposta**
Garantir que a ação de sinalizar problema esteja visível em toda resposta retornada pelo assistente, sem necessidade de o atendente abrir outro sistema.

**CA02 – Envio bloqueado sem documento correto identificado**
Garantir que o sistema bloqueie o envio do formulário de registro de problema quando o atendente não informar o documento correto com versão e seção preenchidos.

**CA03 – Bloqueio do uso da resposta sinalizada**
Garantir que o sistema impeça o uso da resposta sinalizada para comunicar o cliente, exigindo o registro da fonte correta como condição para prosseguir com o atendimento.

**CA04 – FAQ excluído como opção de fonte para correção**
Garantir que o FAQ interno não seja apresentado pelo sistema como opção válida de fonte ao atendente que estiver buscando o documento correto após sinalizar um problema.

**CA05 – Registro vinculado automaticamente ao chamado**
Garantir que todo registro de problema seja automaticamente vinculado ao número do chamado em curso no momento do envio do formulário.

**CA06 – Consolidação de ocorrências do mesmo problema**
Garantir que o sistema consolide em um único registro os problemas reportados por atendentes diferentes para o mesmo documento e versão, incrementando a contagem de ocorrências em vez de criar entradas duplicadas.

**CA07 – Notificação ao atendente após resolução**
Garantir que o atendente que originou o registro de problema receba uma notificação quando a equipe responsável concluir a correção na base de documentos.

---

## PBI 04 — Critérios de Aceite

**CA01 – Declaração antes do conteúdo quando FAQ for utilizado**
Garantir que o assistente exiba a declaração de status da fonte antes de qualquer conteúdo informativo sempre que o FAQ interno for utilizado na composição da resposta.

**CA02 – Apresentação das duas versões em conflito sem escolha silenciosa**
Garantir que o assistente apresente os valores das duas versões conflitantes com seus dados de identificação e o critério de uso por período, sem selecionar uma versão sem informar o atendente.

**CA03 – Declaração de ausência de cobertura para temas sem documento formal**
Garantir que o assistente retorne a declaração de ausência no formato definido, com indicação da área responsável e orientação de registro de feedback, sempre que a pergunta envolver tema catalogado como sem cobertura documental.

**CA04 – Toda resposta classificada em um dos três graus de confiança**
Garantir que toda resposta retornada pelo assistente esteja classificada em um dos três graus — Confirmado, Atenção ou Sem cobertura segura — com sinalização visual distinta entre eles.

**CA05 – Grau sem cobertura segura sem conteúdo informativo apresentado como resposta**
Garantir que o assistente não apresente conteúdo informativo como resposta quando a classificação for sem cobertura segura, exibindo apenas o diagnóstico da ausência, o que existe na base sobre o tema e a área a acionar.

**CA06 – Reconhecimento proativo de temas sem cobertura**
Garantir que o assistente reconheça e declare a ausência de cobertura documental quando a pergunta envolver tema catalogado, independentemente da forma como a pergunta for formulada pelo atendente.

**CA07 – Atualização do catálogo após confirmação de novo tema sem cobertura**
Garantir que o catálogo de temas sem cobertura seja atualizado automaticamente após a confirmação pela equipe responsável, fazendo com que o assistente passe a reconhecer o novo tema nas consultas seguintes.

### Solicitado para complementar uma regra e critério de aceite
Preciso que complemente em um dos PBIs quando for publicado novos documentos, em quanto tempo deve estar disponíveis no assistente para consulta. Crie a regra em Requisitos e Critérios de aceite

### Gerado complemento para adicionar ao PBI 1 - Requisitos e critério de aceite
O tema de atualização da base documental se encaixa naturalmente no PBI 01, pois trata da disponibilidade e confiabilidade da base que sustenta as consultas do fluxo principal. Vou adicionar os itens correspondentes nas seções de Requisitos e Critérios de Aceite.

---

## PBI 01 — Adição à seção 4. Requisitos

**4.8 Atualização da base documental com novos documentos**

Quando um novo documento for publicado ou uma nova versão de documento existente for disponibilizada nas fontes oficiais — SharePoint, Confluence ou pasta de rede — o sistema deve processar e disponibilizar o conteúdo atualizado para consulta no assistente em até 24 horas após a publicação.

O sistema deve garantir que, durante o período de processamento, o assistente continue operando normalmente com a versão anterior do documento, sem interrupção do atendimento.

Quando o processamento de um novo documento for concluído, o sistema deve notificar a equipe responsável pela gestão da base documental confirmando a disponibilização do conteúdo para consulta.

Documentos publicados com marcação formal de substituição de versão anterior devem, após o processamento, tornar a versão substituída indisponível para novas consultas, mantendo apenas a versão vigente como fonte de resposta.

---

## PBI 01 — Adição à seção 5. Critérios de Aceite

**CA07 – Disponibilização de novo documento em até 24 horas**
Garantir que todo documento publicado ou atualizado nas fontes oficiais esteja disponível para consulta no assistente em até 24 horas após a publicação, sem necessidade de intervenção manual da equipe de TI.

**CA08 – Continuidade do atendimento durante o processamento**
Garantir que o assistente continue respondendo normalmente com a versão anterior do documento durante o período de processamento de uma nova versão, sem interrupção do serviço de atendimento.

**CA09 – Notificação de conclusão do processamento**
Garantir que a equipe responsável pela gestão da base documental receba uma notificação quando o processamento de um novo documento for concluído e o conteúdo estiver disponível para consulta no assistente.

**CA10 – Substituição da versão anterior após publicação formal**
Garantir que, quando um documento for publicado com marcação formal de substituição de versão anterior, somente a versão vigente fique disponível como fonte de resposta nas consultas seguintes, tornando a versão substituída indisponível para novos atendimentos.


### Solicitado análise dos PBIs e listagem de ambiguidades e inconsistencias
Análise os 4 PBIs gerados e liste possíveis ambiguidades ou inconsistências na funcionalidade

### Gerado a listagem de ambiguidades e inconsistências para os 4 PBIs
## Análise de ambiguidades e inconsistências nos 4 PBIs

---

### PBI 01 — Consulta ao assistente de IA durante atendimento de chamado

**Ambiguidade 1 — Definição de "documentos oficiais indexados"**
O requisito 4.3 determina que o assistente responda apenas com base em documentos oficiais, e o requisito 4.8 trata da atualização da base. Porém nenhum dos dois define quem é responsável por publicar um documento nas fontes oficiais nem qual é o critério formal para um documento ser considerado oficial e elegível para indexação. Sem essa definição, documentos publicados por áreas sem processo de aprovação podem entrar na base sem validação.

**Ambiguidade 2 — Prazo de 24 horas: corrido ou útil**
O CA07 estabelece que novos documentos devem estar disponíveis em até 24 horas após a publicação, mas não define se esse prazo é em horas corridas ou horas úteis. Um documento publicado às 17h de uma sexta-feira tem comportamento diferente dependendo dessa definição — 24 horas corridas significa sábado às 17h; 24 horas úteis significa segunda-feira à tarde.

**Ambiguidade 3 — O que ocorre com chamados abertos durante o processamento de nova versão**
O CA08 garante continuidade com a versão anterior durante o processamento, mas não define o comportamento para chamados que foram iniciados com a versão anterior e são concluídos após a nova versão entrar em produção. O atendente pode receber respostas diferentes para o mesmo chamado dependendo do momento da consulta.

**Ambiguidade 4 — Registro de fonte no chamado: obrigatório ou opcional**
O requisito 4.7 diz que o atendente "deve registrar" a fonte no chamado, mas não define se o sistema impede o encerramento do chamado sem esse registro ou apenas orienta o atendente a fazê-lo. Sem bloqueio, o requisito não é verificável operacionalmente.

**Inconsistência 1 — CA04 conflita com o PBI 04**
O CA04 do PBI 01 determina que o assistente declare ausência de informação quando não encontrar cobertura. O PBI 04, requisito 4.2, define o formato exato dessa declaração. Os dois PBIs tratam do mesmo comportamento sem se referenciar mutuamente, criando risco de implementações divergentes para a mesma regra.

---

### PBI 02 — Tratamento de respostas de baixa confiança ou discordância do atendente

**Ambiguidade 1 — Critério para o assistente identificar "cobertura documental insuficiente"**
O requisito 4.1 determina que o assistente sinalize quando não encontrar cobertura suficiente, mas não define o critério que determina quando a cobertura é suficiente ou insuficiente. Sem um limiar definido — seja por score de similaridade, seja por ausência de documento formal — a sinalização pode ser inconsistente entre perguntas similares.

**Ambiguidade 2 — "Discordância do atendente": como o sistema a reconhece**
O contexto do PBI menciona que o atendente pode discordar da resposta, mas nenhum requisito define como o atendente aciona formalmente essa discordância na interface. Não está claro se é um botão, um campo de texto livre ou o mesmo fluxo de sinalização do PBI 03. Se for o mesmo fluxo, os dois PBIs se sobrepõem sem deixar isso explícito.

**Ambiguidade 3 — Definição de "área responsável pelo assunto" para gaps sem cobertura**
O requisito 4.4 lista três situações de encaminhamento, mas a primeira — "tema sem cobertura formal: área responsável pelo assunto" — não define como o sistema sabe qual área é responsável por cada tema. Sem um mapeamento tema → área, o sistema não consegue fazer essa indicação automaticamente.

**Ambiguidade 4 — Formulário de feedback: momento de exibição**
O requisito 4.6 determina que o sistema solicite o formulário ao encerrar o chamado. Não está definido se a solicitação bloqueia o encerramento até o preenchimento ou se é opcional. Se for opcional, o ciclo de melhoria documental depende exclusivamente da iniciativa do atendente.

**Inconsistência 1 — Cenário 4 sem requisito correspondente**
O cenário 4 descreve o comportamento para incidentes críticos Gold, onde a escalada deve ocorrer sem pausar o SLA. Porém não existe requisito na seção 4 que defina explicitamente esse comportamento. Cenários sem requisito correspondente não são implementáveis com rastreabilidade.

**Inconsistência 2 — CA05 exige registro antes de responder, mas CA03 libera sem registro**
O CA03 libera o chamado para o fluxo normal quando a reformulação retorna resposta confirmada, sem mencionar registro de fonte. O CA05 exige registro da orientação antes de responder ao cliente. Para o caminho de reformulação bem-sucedida, não está claro se o registro de fonte também é exigido ou se o CA05 se aplica apenas ao caminho de escalada humana.

---

### PBI 03 — Sinalização de resposta incorreta, desatualizada ou incompleta

**Ambiguidade 1 — Sobreposição com o fluxo do PBI 02**
O PBI 02 trata de discordância do atendente com a resposta e o PBI 03 trata de sinalização de resposta errada. Na prática, os dois fluxos partem do mesmo gatilho — o atendente percebe um problema na resposta — sem que os PBIs definam onde termina um e começa o outro. Um atendente que discorda de uma resposta não sabe qual dos dois caminhos deve seguir.

**Ambiguidade 2 — "Mesmo documento e versão" como critério de deduplicação**
O requisito 4.6 e o CA06 definem que problemas reportados para o mesmo documento e versão devem ser consolidados. Porém não define o que acontece quando dois atendentes reportam tipos de problema diferentes para o mesmo documento e versão — um registra como desatualizado e outro como incompleto. São ocorrências do mesmo problema ou registros separados?

**Ambiguidade 3 — Prazo para notificação de resolução ao atendente**
O requisito 4.7 e o CA07 determinam que o atendente deve ser notificado quando o problema for resolvido, mas não definem prazo para que a equipe responsável conclua a resolução. Sem SLA de resolução definido, o atendente não tem expectativa de retorno e o ciclo de melhoria não é mensurável.

**Ambiguidade 4 — "Equipe responsável" não está definida**
Os requisitos 4.5, 4.6 e 4.7 mencionam "equipe responsável" como destinatária dos feedbacks e responsável pela resolução, mas não identificam quem é essa equipe — se é a equipe de projeto, uma área de TI, as áreas documentais (Operações, Compliance, Comercial) ou uma combinação delas. Sem essa definição, os feedbacks não têm destinatário claro.

**Inconsistência 1 — Cenário 3 não tem critério de aceite correspondente**
O cenário 3 descreve o comportamento de inversão de regra sobre carga perigosa e menciona "notificar a equipe responsável para revisão prioritária". Não existe critério de aceite nem requisito que defina priorização de resolução com base no tipo de erro — todos os registros parecem ter o mesmo peso na fila de revisão.

---

### PBI 04 — Regras de comportamento do assistente quando não há resposta segura

**Ambiguidade 1 — Critério para classificar uma resposta como "Atenção" versus "Sem cobertura segura"**
O requisito 4.3 define os três graus, mas a fronteira entre Atenção e Sem cobertura segura não está precisa. Um documento com cobertura parcial pode ser classificado como Atenção ou como Sem cobertura segura dependendo de quanto da pergunta ele cobre. Sem critério objetivo — como percentual de cobertura ou número de dimensões não atendidas — a classificação fica sujeita à interpretação.

**Ambiguidade 2 — "Reconhecimento proativo" sem definição de proximidade semântica**
O CA06 determina que o assistente reconheça temas catalogados mesmo quando a pergunta for formulada de forma diferente. Não está definido o critério de proximidade que determina quando uma pergunta "se enquadra" em um tema catalogado. Perguntas muito indiretas sobre seguro de carga, por exemplo, podem não ser reconhecidas sem um critério claro.

**Ambiguidade 3 — Responsável pela manutenção do catálogo de temas sem cobertura**
O requisito 4.4 define os temas iniciais do catálogo e o requisito 4.5 define a atualização automática por feedback. Porém não define quem tem permissão para editar manualmente o catálogo, remover temas que passaram a ter cobertura documental, ou incluir temas sem que exista feedback registrado.

**Ambiguidade 4 — Formato das declarações é prescritivo mas não define variações permitidas**
Os requisitos 4.1 e 4.2 definem formatos exatos de mensagem com texto entre colchetes indicando variáveis. Não está definido quem preenche essas variáveis, como o sistema as obtém automaticamente — nome da área responsável, nome do tema, item do FAQ — nem o que acontece quando o sistema não consegue preencher uma variável por ausência de metadado.

**Inconsistência 1 — PBI 04 redefine comportamento já descrito no PBI 01 e PBI 02 sem referência cruzada**
O CA03 do PBI 04 trata de declaração de ausência de cobertura, comportamento que também aparece no CA04 do PBI 01 e no requisito 4.1 do PBI 02. Os três PBIs descrevem o mesmo comportamento de formas ligeiramente diferentes, sem indicar que são a mesma regra. Isso cria risco de implementações inconsistentes para o mesmo cenário.

**Inconsistência 2 — CA10 do PBI 01 conflita com a premissa 2.1 do PBI 04**
O CA10 do PBI 01 determina que a versão anterior deve ficar indisponível após publicação formal de nova versão. A premissa 2.1 do PBI 04 estabelece que a PROC-042 v1 ainda deve ser usada para chamados anteriores a 01/12/2023 em processamento. Se a v1 for tornada indisponível por uma publicação formal da v2, chamados antigos perderão a fonte de referência correta — contradição direta entre os dois PBIs.

---

### Recomendações gerais

Três pontos atravessam todos os PBIs e merecem definição antes da refinement:

O primeiro é a definição de "equipe responsável" pela base documental — ela aparece em todos os PBIs como destinatária de feedbacks, notificações e atualizações, mas nunca é identificada com nome, área ou papel.

O segundo é a separação clara entre o fluxo do PBI 02 e o fluxo do PBI 03, que partem do mesmo gatilho do ponto de vista do atendente. Uma matriz de decisão simples — quando usar cada fluxo — resolveria a sobreposição.

O terceiro é a definição de SLA de resolução para feedbacks e problemas registrados. Sem prazo definido, os ciclos de melhoria documental descritos nos PBIs 02, 03 e 04 não são mensuráveis nem comprometidos contratualmente.

### Solicitado correção das ambiguidades e inconsistências e gerar nova versão da documentação
Trate as ambiguidades e inconsistências nos PBIs e gere nova versão atualizada

### Gerado a versão final dos PBIs corrigidos
Os PBIs abaixo constam no Exercicio 1.3
PBI 1 - Consulta ao assistente de IA
PBI 2 - Tratamento de respostas de baixa confiança
PBI 3 - Sinalização de resposta incorreta
PBI 4 - Regras de comportamento do assistente