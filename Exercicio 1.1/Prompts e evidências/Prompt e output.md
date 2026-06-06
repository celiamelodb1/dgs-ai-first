
### Envio do cenário

Que cenário rico! Vou criar uma proposta técnica visual e completa para esse projeto.Ótimo cenário. Vou criar uma proposta técnica visual e bem estruturada para a DB1 apresentar à NovaTech.

SVG anexado na etapa 1 - RAG arquitetura

Acima está a arquitetura completa — clique em qualquer bloco para aprofundar. Agora a proposta detalhada:

---

## Proposta técnica — Assistente RAG NovaTech

### Diagnóstico do problema

O custo atual é concreto: 320 chamados/dia × 60% com consulta × 12 min = **384 horas/dia gastas em busca manual**. Com 45 atendentes, isso representa cerca de 8,5h por pessoa, ou seja, mais de um turno inteiro de trabalho coletivo perdido em navegação de documentos.

O problema é agravado por três fatores estruturais: ausência de fonte única de verdade, documentos com versões conflitantes sem resolução formal, e conhecimento tácito concentrado em pessoas ("perguntar para quem sabe" não é processo, é risco operacional).

---

### Solução proposta — RAG com stack Microsoft

A abordagem é um sistema de **Retrieval-Augmented Generation (RAG)** totalmente integrado ao ambiente Microsoft já licenciado pela NovaTech.

**Por que RAG e não fine-tuning?** A documentação muda mensalmente. Fine-tuning exigiria re-treino a cada ciclo — caro e lento. RAG indexa novos documentos em minutos e o modelo responde sempre com base nos documentos atuais, com citação rastreável da fonte.

**Stack técnica sugerida:**

| Camada | Serviço | Justificativa |
|---|---|---|
| Ingestão | Azure Data Factory + SharePoint API | Já no tenant M365 E3 |
| Parsing | Azure Document Intelligence | Extrai texto de PDFs complexos com layout preservado |
| Embeddings | Azure OpenAI `text-embedding-3-large` | Alta qualidade semântica para português |
| Índice vetorial | Azure AI Search (com hybrid search) | Combina busca semântica e por palavra-chave |
| Geração | Azure OpenAI GPT-4o | Respostas em linguagem natural com citação obrigatória |
| Interface | Azure Bot Service + Teams | Sem instalação para o usuário final |
| Automação | Azure Logic Apps | Trigger de re-indexação mensal por área |
| Observabilidade | Azure Monitor + Power BI | Painel de adoção e qualidade para a gestão |

---

### Tratamento do problema de conflito de versões

Este é o ponto mais delicado e frequentemente subestimado. A proposta inclui uma camada de **detecção de conflito** no pipeline de indexação:

Cada chunk de documento recebe metadados de versão (data de atualização, área responsável, número de revisão). Quando dois chunks respondem à mesma query com conteúdo contraditório, o assistente **não escolhe silenciosamente** — ele apresenta ambos, identifica a divergência e recomenda consulta ao responsável da área. Isso transforma um problema invisível (respostas inconsistentes hoje) em um problema gerenciável com dados.

Paralelamente, recomendamos que a NovaTech adote um **processo unificado de publicação** — documentos que não passarem por um repositório canônico não entram no índice. O assistente se torna incentivo para a governança que já deveria existir.

---

### Plano de 3 meses

**Mês 1 — Discovery & fundação**
- Mapeamento e auditoria dos 1.200+ documentos
- Definição de taxonomia e metadados obrigatórios
- Provisionamento de ambiente Azure (AI Search, OpenAI, Bot Service)
- Pipeline de ingestão para SharePoint (fonte mais crítica)
- Protótipo funcional com as 3 áreas de maior volume de chamados

**Mês 2 — Desenvolvimento & integração**
- Ingestão completa (Confluence + planilhas)
- Implementação da lógica de detecção de conflito
- Integração com Teams via Azure Bot Service
- Sistema de citação de fonte com link direto ao documento original
- Testes com grupo piloto de 10 atendentes

**Mês 3 — Refinamento & go-live**
- Ajuste de prompts e thresholds de similaridade com base no piloto
- Treinamento de todos os 45 atendentes
- Painel Power BI para gestão (adoção, tempo médio, queries sem resposta)
- Go-live com monitoramento intensivo
- Documentação de processo de re-indexação para as áreas

---

### Meta de negócio: de 12 para menos de 2 minutos

A meta da diretoria é viável — e conservadora. Com um sistema RAG bem calibrado, o tempo de recuperação da informação cai para menos de 30 segundos. O restante do tempo (até 2 min) é para o atendente ler a resposta, validar o contexto e aplicar ao chamado.

O KPI recomendado para acompanhamento não é só o tempo médio, mas também:
- **Taxa de respostas úteis** (baseado no feedback 👍/👎 por sessão)
- **Taxa de escalada por falta de informação** (deve cair)
- **Queries sem resposta relevante** (sinaliza gaps de documentação — ação para as áreas)

---

### Riscos e mitigações

| Risco | Probabilidade | Mitigação |
|---|---|---|
| Documentos com qualidade de scan ruim (PDFs de imagem) | Alta | Azure Document Intelligence com OCR; pipeline de rejeição com lista de documentos para digitalização |
| Resistência dos atendentes ("o bot vai me substituir") | Média | Framing como copiloto, não substituto; envolver líderes de atendimento desde o piloto |
| Documentação conflitante não resolvida pela gestão | Alta | Regra clara: sem versão aprovada, sem indexação; o assistente não resolve conflito de governança |
| Custo Azure acima do esperado | Baixa | Dimensionamento conservador; hybrid search reduz chamadas ao modelo generativo |
| Alucinação em queries sem resposta nos documentos | Média | System prompt com instrução explícita de declinar quando não há base documental; citação obrigatória bloqueia fabricação |

---

### Próximos passos sugeridos para a DB1

1. **Workshop de discovery** com as 3 áreas (Operações, Compliance, Comercial) para mapear os tipos de query mais frequentes — isso guia a priorização da ingestão e os testes do piloto.
2. **Auditoria rápida de 50 documentos** para estimar a proporção de PDFs escaneados vs. digitais nativos — impacta diretamente o custo e prazo do parsing.
3. **Definição do responsável de produto na NovaTech** — sem um PM interno que arbitre conflitos de documentação e valide respostas, o projeto tem risco de travar no mês 2.
4. **Estimativa de custo Azure** com base no volume: ~1.200 docs × tamanho médio estimado → tokens de embedding + custo de busca por query (320 chamados × 60% × custo por busca).

---

### Anexo A fornecido

Arquivo recebido! É o **Anexo A** com a documentação simulada da NovaTech — exatamente o material de base para o projeto do assistente RAG.

Aqui está um resumo do que está dentro:

**5 documentos simulados:**

| Doc | Tipo | Status |
|---|---|---|
| POL-001 | Política de Devolução de Mercadorias | Normativo, v3.1, jan/2024 |
| PROC-042 v1 | Cálculo de Frete Especial | ⚠️ Sem indicação de vigência |
| PROC-042 v2 | Cálculo de Frete Especial (revisado) | ⚠️ Coexiste com v1 sem hierarquia |
| SLA-2024 | Tabela de SLA por Tipo de Cliente | Contratual, jan/2024 |
| FAQ-Atendimento | Perguntas frequentes do time | ⚠️ Informal, não validado |

**Problemas já identificados no próprio arquivo** (ótimos para testar o assistente):

- PROC-042 v1 vs v2: multiplicadores regionais diferentes, fatores de peso diferentes (+1.2/+1.5 vs +1.15/+1.4), prazo adicional diferente (+2 vs +3 dias úteis) — e nenhum está marcado como obsoleto
- FAQ menciona frete expresso para carga perigosa, mas não existe PROC formal para isso
- Política de carga danificada existe só no FAQ informal, sem documento normativo
- Seguro de carga idem — só no FAQ, sem doc formal

O que você quer fazer com esse material? Algumas possibilidades:

- **Simular o assistente** respondendo perguntas com base nesses docs
- **Montar os chunks** para ingestão no pipeline RAG
- **Criar casos de teste** para validar comportamento do assistente em situações de conflito
- **Trabalhar os exercícios** do material de prática (Dev 1.3 e similares mencionados no arquivo)

--- 

### faça um mapa dos temas abordados nos documentos
Vou fazer dois diagramas: o primeiro mostra os 5 documentos e seus temas principais; o segundo destaca os pontos de conflito e lacunas — que são a parte mais crítica para o projeto RAG.

Anexo SVG etapa 1 - Mapa temas

Agora o segundo diagrama, focado exclusivamente nos pontos de atrito — conflitos entre documentos e lacunas sem cobertura formal:

Anexo SVG etapa 1 - conflitos lacunas

Os diagramas são clicáveis — cada bloco dispara uma pergunta relacionada ao tema para continuar a conversa.

Alguns pontos que valem destaque para o projeto:

**Para o pipeline RAG:** os quatro conflitos em vermelho são casos em que o assistente **não pode simplesmente recuperar e apresentar** — ele precisa sinalizar ativamente a divergência. Sem isso, vai ora dar o valor da v1, ora o da v2, dependendo de qual chunk for recuperado, gerando exatamente o tipo de inconsistência que o projeto quer eliminar.

**Para a governança da NovaTech:** as quatro lacunas em coral são oportunidades de formalização. O FAQ do item 38 (carga danificada) e o item 22 (seguro) descrevem processos que claramente existem na prática — só não têm documento normativo. O assistente pode ser o catalisador para a NovaTech perceber isso e endereçar antes do go-live.

**Recomendação prática:** antes de iniciar a ingestão, entregar esse mapa para as áreas de Operações, Compliance e Comercial com uma pergunta simples — "qual é a versão correta de cada item em conflito?" — e registrar as respostas como decisão formal. Isso resolve os conflitos antes que o RAG os herde.

Diagramas anexados na etapa 1


---

### Gere as hipóteses de possíveis problemas na documentação

Vou analisar a documentação sistematicamente e gerar as hipóteses de problemas, organizadas por categoria e grau de risco.

Anexo SVG etapa 1 - hipoteses Parte 1 e 2

No total são **16 hipóteses** distribuídas em quatro categorias. Cada card é clicável para aprofundar o tema.

Alguns padrões que emergem da análise:

**O FAQ é o maior vetor de risco.** Ele aparece em 7 das 16 hipóteses — ora contradizendo documentos formais, ora sendo a única fonte sobre temas críticos, ora referenciando versões antigas de procedimentos. Indexá-lo com o mesmo peso dos documentos normativos é o erro mais provável de acontecer e o de maior impacto.

**A raiz de quase todos os conflitos é a mesma:** ausência de metadado de vigência. Se cada documento tivesse um campo `status: vigente | obsoleto | em revisão` e uma referência à versão que substitui ou é substituída por, o RAG poderia filtrar antes de recuperar. Sem isso, a lógica de resolução de conflito precisa ser construída na camada de aplicação — o que é possível, mas mais caro.

**Três hipóteses têm impacto financeiro direto e mensurável:** os multiplicadores regionais, os fatores de peso e o threshold de desconto por volume. Valem ser priorizadas na validação com o Comercial antes da ingestão.

**A hipótese do limiar de incidente crítico** (R$ 50 mil no FAQ vs. R$ 100 mil no SLA-2024) é particularmente delicada porque envolve compromisso contratual — um cliente Gold poderia argumentar que o atendente prometeu tratamento de incidente crítico com base no FAQ, mesmo que o contrato não respalde.

Quer que eu monte um plano de validação dessas hipóteses — com perguntas específicas para cada área responsável — ou prefere partir para a construção dos casos de teste do assistente?

Arquivos visuais SVG anexados na etapa 1

---

### Fornecido os documentos completos PROC V1 e V2 e solicitado: Faça uma análise de inconsistência dos documentos

Anexo SVG etapa 2 

Use a calculadora para simular qualquer combinação de região e faixa de peso — ela mostra em tempo real quanto os dois documentos divergem para um mesmo frete.


Alguns pontos que merecem atenção especial além do que está no painel:

**O conflito mais perigoso não é o maior numericamente.** Os multiplicadores regionais têm deltas de 7–12%, mas o problema do desconto por volume é estruturalmente diferente: a v1 não define percentual, apenas diz "negociar com o Comercial". Isso significa que clientes antigos podem ter descontos registrados em aditivos contratuais que não correspondem nem à v1 nem à v2 — e o RAG não tem como saber disso. Uma resposta sobre desconto pode estar correta segundo o documento e errada segundo o contrato do cliente específico.

**A cláusula transitória vencida é uma armadilha silenciosa.** A v2 diz que chamados abertos antes de 01/12/2023 usam a v1. Essa data passou há mais de 2 anos, mas o texto ainda está no documento. Se o RAG recuperar esse trecho para uma query sobre "qual versão usar", pode gerar confusão — a resposta correta é "use sempre a v2", mas o documento diz algo mais complexo.

**A nota sobre a PROC-043 "em revisão"** foi escrita em novembro de 2023. Se a revisão já foi concluída, esse aviso virou desinformação. O assistente vai citar a ressalva como se ainda fosse válida.

A recomendação mais importante antes de qualquer ingestão: uma reunião de 30 minutos com o Comercial para obter uma decisão documentada — "a v2 é o documento vigente, a v1 está arquivada". Sem isso, toda a lógica de resolução de conflito no RAG vai trabalhar contra uma ambiguidade que tem solução trivial no mundo real.

rquivo visual SVG anexado na etapa 2

---

### Forneciido o documento completo FAQ e solicitado: Com base na visão geral e análise realizada dos arquivos PROC, cruze as inconsistências encontradas com as práticas informais do FAQ

Anexo SVG etapa 3

O cruzamento revela um padrão claro: o FAQ não é um documento ruim — é um documento que envelheceu de forma irregular. Algumas partes continuam precisas, outras ficaram presas na versão antiga das PROCs, e outras descrevem processos que nunca foram formalizados.

O achado mais crítico é o item 45. É o único caso em que o FAQ está ativamente errado em relação a um documento vigente com impacto financeiro direto e mensurável: qualquer cliente entre 8 e 10 fretes especiais por mês está sendo negado um desconto ao qual tem direito pela PROC-042 v2. Isso provavelmente já está acontecendo hoje, antes mesmo do RAG existir.

O segundo achado mais relevante é estrutural: quatro dos nove itens analisados não têm nenhum documento formal de respaldo. Se o FAQ for indexado sem tratamento, o assistente vai apresentar esses conteúdos com a mesma autoridade de um documento normativo assinado pela Diretoria — sem nenhuma ressalva ao atendente. É precisamente o tipo de problema que o projeto prometeu resolver, reintroduzido pela porta dos fundos.

A recomendação prática é tratar o FAQ como um insumo de discovery, não como fonte de indexação. Ele é valioso para identificar quais temas os atendentes precisam acessar — mas cada item deve ser verificado contra os documentos formais antes de entrar no índice. Onde não existe documento formal, essa é a oportunidade de criar um.

Arquivo visual SVG aneaxdo na etapa 3


---

### Como se baseou para atribuir o nível de risco RAG para os documentos PROC e FAQ?

Boa pergunta — e importante, porque os níveis de risco que atribuí não vieram de uma rubrica formal. Vieram de um raciocínio implícito que vale tornar explícito para você poder questionar e calibrar.

**A lógica subjacente que usei**

Avaliei cada item cruzando três dimensões ao mesmo tempo, sem ter verbalizado isso na resposta:

A primeira foi **o tipo de erro que o RAG cometeria**. Há uma diferença entre o assistente dar uma informação incompleta, dar uma informação desatualizada, e dar uma informação fabricada com aparência de autoridade. Esse último é o mais grave porque o atendente não tem como detectar o problema — a resposta parece legítima. Os itens que classifiquei como Crítico são exatamente os que produziriam esse terceiro tipo de erro.

A segunda foi **quem arca com a consequência**. Informação errada sobre threshold de prioridade (item 27) sobrecarrega o time interno — ruim, mas recuperável. Informação errada sobre desconto por volume (item 45) prejudica o cliente diretamente, sem que ele saiba. Informação errada sobre seguro de carga (item 22) cria uma promessa comercial sem respaldo que pode virar litígio. A severidade aumenta conforme a consequência sai do operacional e vai para o financeiro e jurídico.

A terceira foi **a rastreabilidade da fonte**. Se o RAG citar algo e o atendente ou o cliente questionarem, existe um documento oficial que sustenta a resposta? Itens sem nenhum documento formal de respaldo — como carga danificada (38) e seguro (22) — receberam risco mais alto não porque o conteúdo seja necessariamente errado, mas porque o assistente não teria como sinalizar a limitação da fonte. Ele apresentaria como fato o que é apenas prática informal.

**Os casos onde meu julgamento foi mais discutível**

O item 3 (devolução de carga perigosa) eu classifiquei como Médio e recomendei indexar com flag de informal. Poderia ser Alto, porque envolve carga perigosa e exceções não documentadas. O argumento para Médio foi que o FAQ e a POL-001 convergem na orientação principal — ligar no ramal 4500 — e a nuance sobre exceções é uma ressalva, não uma contradição. Mas se a NovaTech opera com margens de risco mais conservadoras para carga perigosa, esse deveria ser Alto.

O item 41 (SLAs por tier) eu classifiquei como Médio mesmo com a omissão da regra de pausa. Aqui o raciocínio foi que os valores principais estão corretos e a omissão é de uma condição de exceção — não da regra geral. Um atendente informado pelos valores do FAQ estará certo na maioria dos casos. Mas em um incidente crítico Gold às 20h, estará errado de forma consequente. Dependendo do perfil de clientes Gold da NovaTech, isso poderia subir para Alto.

**O que eu não tinha e que afetaria os níveis**

Não tinha o volume de cada tipo de consulta. Se 80% das perguntas sobre frete especial envolvem a região Norte, o impacto dos multiplicadores divergentes é muito maior do que se Norte for 5% do volume. Não tinha o perfil de risco jurídico da empresa — uma NovaTech conservadora em compliance trataria o item 38 (carga danificada) de forma diferente de uma que aceita mais ambiguidade operacional. E não tinha os contratos reais dos clientes, que poderiam revelar que alguns clientes têm cláusulas específicas que tornam certos itens mais ou menos críticos do que parecem.

**O que isso significa na prática**

Os níveis que atribuí são um ponto de partida razoável para uma conversa com a equipe da NovaTech, não uma classificação definitiva. A utilidade real não é o nível em si, mas a lógica por trás dele — que você pode aplicar de forma mais precisa com informações que eu não tinha acesso.

Se quiser, posso montar uma rubrica formal com os três critérios explicitados — tipo de erro, quem arca com a consequência, rastreabilidade da fonte — para que a equipe possa reclassificar cada item com dados reais do contexto da NovaTech.

---

### Identifique ao menos 2 riscos que encontrou e descreva como levaria isso para o discovery humano
Anexo resposta em Riscos - Discovery humano


---

### Gere o mapa de riscos dos documentos

Anexado arquivo SVG em Mapa de riscos