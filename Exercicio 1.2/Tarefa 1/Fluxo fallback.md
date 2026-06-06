## Jornada do atendente — Tratamento de respostas de baixa confiança ou divergência com a IA

---

### Contexto de ativação

Este fluxo é acionado em dois cenários distintos, ambos ocorrendo após o Passo 3 do fluxo principal (consulta ao assistente):

**Cenário A — Baixa confiança da IA:** o assistente retorna uma resposta sinalizando explicitamente incerteza. Isso ocorre quando nenhum documento da base cobre o caso com clareza suficiente, quando há conflito entre versões sem resolução definida, ou quando a pergunta envolve tópicos com gap documental conhecido (seguro de carga, carga danificada, frete padrão abaixo de 500kg, procedimento da Gestão de Riscos).

**Cenário B — Discordância do atendente:** o assistente retorna uma resposta com confiança, mas o atendente a questiona com base em experiência prévia, em informação recebida de colega ou supervisor, ou em percepção de que o documento citado está desatualizado.

Ambos os cenários interrompem o fluxo principal antes do Passo 6 (resposta ao cliente) e seguem o tratamento descrito abaixo.

---

### Passo 1 — Identificar o motivo da incerteza

O atendente lê o sinal emitido pelo assistente ou articula sua própria discordância. Os motivos possíveis se enquadram em quatro categorias:

**Gap documental:** a pergunta envolve um assunto sem documento normativo formal na base. Exemplos do contexto NovaTech: política de carga danificada em trânsito, percentuais de seguro de carga, frete abaixo de 500kg. A IA pode ter retornado uma resposta baseada no FAQ informal, que não é validado pelo Compliance.

**Conflito entre versões:** dois documentos ativos respondem à mesma pergunta de forma diferente. Exemplo direto: PROC-042 v1 e v2 coexistem com multiplicadores regionais, fatores de peso e prazo de entrega distintos, sem marcação formal de obsolescência no SharePoint.

**Cobertura parcial:** o documento citado existe e é válido, mas não cobre a especificidade do caso — por exemplo, a POL-001 cobre devolução padrão, mas o caso envolve carga com lacre violado e ausência do motorista no momento da entrega, situação não detalhada na política.

**Conhecimento tácito divergente:** o atendente tem uma prática consolidada que difere do que a IA retornou. Pode indicar que a documentação está desatualizada, que existe uma instrução verbal não documentada, ou que a prática informal está errada e precisa ser corrigida.

**Saída:** motivo da incerteza categorizado antes de qualquer ação.

---

### Passo 2 — Não responder ao cliente ainda

Independentemente do motivo, o atendente não avança para o Passo 6 do fluxo principal. Responder com informação de confiança baixa ou com fonte em conflito gera inconsistência entre atendentes e pode criar compromissos contratuais incorretos com o cliente — especialmente em casos envolvendo SLA, cálculo de frete e devoluções.

O chamado permanece aberto. O atendente registra no campo de notas internas do chamado que a resposta está pendente de validação, com breve descrição do motivo.

**Saída:** cliente não recebeu resposta incorreta; chamado em estado de pendência documentada.

---

### Passo 3 — Reformular a pergunta ao assistente

Antes de escalar, o atendente tenta resolver internamente refinando a consulta. Reformulações eficazes incluem:

- Especificar o documento que deveria cobrir o caso: *"Com base na POL-001 versão 3.1, qual é o procedimento para devolução de carga com lacre violado?"*
- Restringir o escopo temporal: *"Qual o multiplicador regional para o Norte segundo a PROC-042-v2, vigente a partir de dezembro de 2023?"*
- Separar a pergunta em partes menores quando ela combina múltiplas regras.
- Pedir explicitamente que o assistente liste quais documentos cobrem o tema, sem gerar uma resposta sintetizada.

Se a reformulação retornar uma resposta com fonte clara e sem conflito, o fluxo retorna ao Passo 4 do fluxo principal. Se a incerteza persistir, avança para o próximo passo.

**Saída:** incerteza resolvida internamente (retorna ao fluxo principal) ou confirmada como não resolvível pelo assistente (avança).

---

### Passo 4 — Acionar a fonte humana competente

A escalada segue a categoria de motivo identificada no Passo 1:

**Gap documental** — acionar a área responsável pelo tema ausente. Para carga danificada: encaminhar para o e-mail sinistros@novatech.com.br e aguardar orientação antes de responder. Para seguro de carga: contatar o Comercial para confirmar percentual aplicável ao contrato do cliente. Para frete abaixo de 500kg: solicitar ao Comercial a tabela vigente.

**Conflito entre versões** — acionar a área autora do documento mais recente. Para PROC-042: contatar a Diretoria Comercial para confirmar qual versão aplica ao contrato e ao período do chamado. Registrar a resposta recebida no chamado para rastreabilidade.

**Cobertura parcial** — acionar o supervisor de atendimento ou o especialista da área operacional que conhece o caso análogo. Descrever o caso específico e pedir orientação por escrito (e-mail ou mensagem no Teams), não verbal.

**Conhecimento tácito divergente** — acionar o supervisor imediato. Apresentar lado a lado o que a IA retornou (com fonte) e o que o atendente conhece como prática. Solicitar que o supervisor indique qual está correto e, se a prática for a correta, que abra solicitação de atualização documental.

Em todos os casos, a orientação recebida deve ser registrada no chamado antes de responder ao cliente.

**Saída:** orientação validada por fonte humana competente, registrada formalmente no chamado.

---

### Passo 5 — Responder ao cliente com base na orientação validada

Com a orientação registrada, o atendente responde ao cliente. A resposta cita a informação validada sem expor detalhes internos sobre o conflito ou a incerteza. O atendente não menciona ao cliente que houve divergência ou que a IA não soube responder.

Se a resolução exigir ação de outra área — coleta reversa, análise de sinistro, renegociação comercial — o atendente informa ao cliente o próximo passo e o prazo esperado, sem comprometer prazos que não controlam.

**Saída:** cliente respondido com informação validada e rastreável.

---

### Passo 6 — Registrar o caso para melhoria da base documental

Este passo é obrigatório e ocorre após o encerramento do chamado. O atendente registra o caso em um formulário de feedback do assistente — a ser definido pela equipe de projeto — com os seguintes campos:

- Pergunta feita ao assistente
- Motivo da incerteza (gap, conflito, cobertura parcial, tácito divergente)
- Área acionada para resolução
- Orientação recebida
- Sugestão de ação: criar documento, atualizar versão, arquivar versão obsoleta, validar FAQ

Esses registros alimentam o processo mensal de revisão documental das três áreas responsáveis (Operações, Compliance, Comercial), criando um ciclo de melhoria contínua da base de conhecimento do assistente.

**Saída:** caso documentado, base de conhecimento sinalizando gaps ativos para as equipes responsáveis.

---

### Condições de desvio dentro deste fluxo

| Situação | Ação |
|---|---|
| Fonte humana não responde dentro do SLA do chamado | Supervisor do atendente assume responsabilidade pela decisão e registra o risco formalmente |
| Orientação recebida contradiz o documento da IA | Registrar ambas no chamado; acionar o supervisor para definir qual prevalece antes de responder |
| Caso envolve incidente crítico de cliente Gold | SLA não pausa — a escalada humana deve ocorrer em paralelo à consulta à IA, não em sequência |
| Atendente recebe orientação apenas verbal | Solicitar confirmação por escrito antes de responder ao cliente; orientação verbal não é registrável |
| Gap identificado já foi reportado por outro atendente | Verificar se há resolução pendente no sistema; não duplicar escalada, mas registrar ocorrência adicional para priorização |

---

### Princípios que governam este fluxo

A incerteza da IA não é uma falha — é um sinal de qualidade. Um assistente que sinaliza quando não sabe é mais confiável do que um que responde com falsa segurança. O atendente deve tratar esse sinal como informação útil, não como obstáculo.

A discordância do atendente também é um sinal de qualidade. Quando um atendente experiente percebe divergência entre a IA e a prática real, isso quase sempre indica um de dois problemas: a documentação está desatualizada, ou a prática informal está errada. Ambos precisam ser resolvidos — e o registro no Passo 6 é o mecanismo para isso.

Nenhuma resposta ao cliente deve sair sem fonte rastreável registrada no chamado. Isso vale tanto para respostas geradas com confiança alta quanto para as que passaram por este fluxo de exceção.