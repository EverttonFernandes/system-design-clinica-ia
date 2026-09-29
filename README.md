# Clínica IA · System Design (Arquitetura + IA) para estudar, explicar e evoluir

Como projetar um assistente que responde dúvidas sobre uma clínica, encontra profissionais e agenda consultas, mantendo isolamento entre clientes, consistência das reservas e controle sobre a IA?

Este repositório explora essa pergunta por meio de um **SaaS multi-tenant para clínicas**, com atendimento por Web e WhatsApp. O estudo conecta agentes, RAG, tools e backend transacional às decisões de qualidade, custo, latência e segurança que sustentam o sistema — do MVP à escala.

O material nasceu de um desenho de entrevista de System Design e foi desenvolvido em **oito pranchas editáveis no draw.io**, acompanhadas de explicações, exemplos e cenários de falha. O conteúdo publicado é um estudo de arquitetura; os serviços descritos representam uma proposta, sem implementação executável ou resultados de produção neste repositório.

**Comece pelo [fluxo de atendimento](#atendimento), acompanhe a [busca de conhecimento](#rag) e termine na [reserva de uma consulta](#agendamento).** Para editar as oito pranchas, use o [arquivo fonte do draw.io][drawio].

<a id="mapa"></a>

## Mapa de estudo e imagens

Cada linha conecta uma aba do arquivo editável à imagem exportada e à sua explicação neste README. Clique nas imagens para consultar os detalhes em tamanho maior.

| Aba | Imagem | O que estudar | Explicação |
| --- | --- | --- | --- |
| 01 | [System Design — Atendimento][img-01] | Caminho da mensagem entre canais, agente, tools e dados. | [Fluxo principal](#atendimento) |
| 02 | [System Design — RAG e ingestão][img-02] | Separação entre preparar documentos e responder perguntas. | [Dois pipelines](#rag) |
| 03 | [System Design — Agendamento][img-03] | Criação de reservas com idempotência e proteção contra concorrência. | [Fluxo de reserva](#agendamento) |
| 04 | [Guia — Atendimento][img-04] | Responsabilidades R1–R6, memória, cache e fronteiras de confiança. | [Componentes do atendimento](#guia-atendimento) |
| 05 | [Guia — RAG e ingestão][img-05] | Isolamento I1, metadados I2, versões e recuperação I3. | [Detalhamento do RAG](#guia-rag) |
| 06 | [Guia — Tools e agenda][img-06] | Contratos de tools e regras A1–A4 para executar ações. | [Detalhamento da agenda](#guia-agenda) |
| 07 | [Guia — Operação e evals][img-07] | Segurança, traces, qualidade, custos, resiliência e capacidade. | [Operação do sistema](#operacao) |
| 08 | [Guia — Evolução e falhas][img-08] | Escopo do MVP, decisões futuras e comportamento sob falhas. | [Evolução da arquitetura](#evolucao) |

Também neste material: [contexto e requisitos](#contexto) · [exercícios de revisão](#exercicios) · [glossário](#glossario) · [como editar](#editar).

**Leitura visual:** ator representa o paciente; hexágono, gateway ou router; processo, API ou tool; cilindro, armazenamento; nuvem, provider externo. Setas contínuas indicam chamadas síncronas e as tracejadas, comunicação assíncrona. As caixas representam responsabilidades lógicas: a implementação pode reunir várias delas no mesmo processo.

<a id="contexto"></a>

## Contexto, requisitos e limites do estudo

O paciente quer obter informações, conhecer especialidades, encontrar profissionais e reservar um horário. A clínica precisa manter essas informações atualizadas e controlar quem pode acessá-las. A plataforma precisa atender várias clínicas com infraestrutura compartilhada e isolamento verificável.

Neste cenário, cada clínica é um tenant independente. `tenantId` identifica a fronteira de isolamento e `clinicId`, a clínica na qual a operação ocorre. O backend valida a relação entre os dois; escrever esses identificadores no prompt não estabelece autorização.

| Necessidade | Resposta arquitetural | Condição que deve permanecer verdadeira |
| --- | --- | --- |
| Responder dúvidas sobre a clínica | RAG sobre documentos autorizados e ativos. | A resposta deve se apoiar nas fontes recuperadas; ausência de evidência precisa ser reconhecida. |
| Encontrar profissionais | Tool sobre o catálogo do backend. | Especialidade, vínculo e disponibilidade vêm de dados verificáveis. |
| Consultar e reservar horários | Scheduling API e banco transacional. | Uma consulta de disponibilidade não garante uma reserva. |
| Continuar uma conversa | Estado persistido pela aplicação. | Uma sessão não pode acessar o contexto de outro paciente ou tenant. |
| Atender múltiplas clínicas | Contexto confiável propagado entre serviços e dados. | Banco, retriever, cache, fila e state devem respeitar o mesmo isolamento. |
| Evoluir com controle | Observabilidade, evals e limites de execução. | Ganhos de qualidade devem ser avaliados junto de custo, latência e segurança. |

O assistente tem escopo administrativo e informativo. A indicação de profissionais considera catálogo e preferências; diagnóstico e decisões clínicas não fazem parte do fluxo proposto.

**Premissas ainda abertas:** quantidade de clínicas e usuários, pico de mensagens, volume documental, frequência de atualização, orçamento, disponibilidade desejada e tempo aceitável de resposta. O desenho não presume metas numéricas nem uma stack obrigatória. Essas informações orientam o dimensionamento e as escolhas de implementação.

<a id="atendimento"></a>

## 1. Atendimento: da mensagem à resposta

**Imagem 01 · visão do sistema.** Acompanhe a mensagem da esquerda para a direita e use os marcadores R1–R6 para localizar as responsabilidades.

[![Diagrama 01: paciente em Web ou WhatsApp acessa o gateway, o agente, as tools, os bancos e os providers de modelos.][img-01]][img-01]

Considere a mensagem: **“A clínica atende aos sábados? Quero marcar com um dermatologista.”** Ela reúne uma pergunta documental e uma intenção de agir. O sistema pode precisar consultar conhecimento, catálogo e agenda em etapas distintas.

1. **O canal entrega a mensagem.** A Web envia uma requisição; o WhatsApp entrega um evento por webhook validado.
2. **O gateway estabelece o contexto.** Resolve clínica, identidade e permissões aplicáveis, aplica limites e inicia a correlação da requisição.
3. **O Agent Service carrega o estado.** Recupera a conversa autorizada, prepara o contexto e chama o modelo selecionado.
4. **O modelo propõe uma resposta ou uma tool.** A aplicação valida a proposta antes de executar a capacidade solicitada.
5. **As fontes adequadas respondem.** `searchClinicKnowledge` busca informações documentais; `findProfessionals` consulta o catálogo; `getAvailableSlots` consulta a agenda.
6. **A aplicação compõe a resposta e salva o estado.** Uma reserva exige a escolha do paciente e a execução do [fluxo de agendamento](#agendamento).

As chamadas às tools podem exigir novas interações com o modelo. Por isso, o orçamento de tempo e tokens deve abranger toda a tarefa, incluindo essas etapas.

<a id="guia-atendimento"></a>

### Guia 04 · responsabilidades e decisões do atendimento

<details>
<summary>Ver a imagem 04 com as explicações dentro do diagrama</summary>

[![Guia 04: responsabilidades do gateway, orquestrador, estado, tools, router, cache e limites de confiança.][img-04]][img-04]

</details>

| Marcador | Componente | Responsabilidade | Por que ele existe |
| --- | --- | --- | --- |
| R1 | API Gateway / BFF | Validar entrada, autenticar conforme a operação, resolver tenant, aplicar limites e propagar `traceId`. | Estabelecer um contexto confiável antes da execução. |
| R2 | Clinic Assistant / Agent Service | Coordenar modelo, prompt, tools, estado e limites de execução. | Transformar a conversa em um fluxo controlado pela aplicação. |
| R3 | RAG Service | Recuperar conhecimento com filtros obrigatórios e devolver fontes. | Responder com informações da clínica autorizada. |
| R4 | API de domínio | Aplicar regras de catálogo e agendamento, incluindo transações. | Preservar as regras de negócio independentemente da saída do modelo. |
| R5 | Session / State Store | Persistir contexto e progresso do workflow. | Permitir continuidade da conversa e execução em múltiplas instâncias. |
| R6 | Model Router | Selecionar uma configuração de modelo por capacidade e orçamento. | Ajustar a execução aos requisitos da tarefa. |

<a id="entrada-confiavel"></a>

**R1 · Entrada e canais.** O tenant vem de identidade e configuração validadas, como a associação do canal à clínica. Um `tenantId` enviado pelo cliente ou sugerido pela LLM precisa ser confrontado com esse contexto. Validar a origem de um webhook também não dispensa a autorização do paciente para a ação pedida.

Na Web, SSE pode entregar trechos da resposta conforme são produzidos. Isso melhora a percepção de espera; não elimina o tempo total de processamento. No WhatsApp, recebimento do webhook e envio da resposta são etapas separadas. Eventos repetidos precisam ser reconhecidos para evitar a repetição do atendimento. WebSocket fica como opção quando houver necessidade de comunicação bidirecional persistente.

**R2 · Orquestração.** O MVP usa um agente com ferramentas especializadas. A aplicação controla quais tools estão disponíveis, valida argumentos, limita iterações e decide quando encerrar. A LLM pode propor `createAppointment`, mas a criação depende das regras do backend e da intenção confirmada do paciente.

**R3 e R4 · Conhecimento e domínio.** Horários de funcionamento e orientações documentais pertencem ao RAG. Disponibilidade de um profissional e resultado de uma reserva pertencem ao serviço transacional. Um texto recuperado que diga “há vagas” não comprova disponibilidade atual.

<a id="estado-cache"></a>

**R5 · Estado e memória.** Histórico selecionado, preferências informadas, profissional escolhido e etapa atual são dados da aplicação. Uma chave conceitual pode ser `tenantId + clinicId + conversationId`, sempre com verificação do participante autorizado. Conhecer o identificador da conversa não deve conceder acesso a ela.

TTL e política de persistência variam conforme o dado. O contexto temporário pode expirar; o registro de uma reserva permanece no banco de domínio. Se a conversa for resumida, a aplicação deve preservar escolhas e fatos críticos e medir o efeito do resumo na qualidade.

**Cache opcional.** Em cache-aside, a aplicação consulta o cache e, em uma ausência, busca a fonte e armazena o resultado. A chave precisa considerar tenant, clínica e, conforme o conteúdo, permissões, versão documental e identidade do usuário. TTL limita a duração; invalidação trata mudanças que precisam refletir antes do vencimento. Disponibilidade em cache continua sujeita à validação no momento da reserva.

**R6 · Seleção de modelos.** O router pode começar como configuração interna, com um único provider validado. Selecionar outro modelo exige comparar sucesso da tarefa, uso de tools, custo e latência. Extrair um AI Gateway e adicionar providers faz parte da [evolução](#evolucao), quando houver uma necessidade operacional demonstrável.

**Pergunta de revisão:** se duas instâncias do agente receberem mensagens da mesma conversa, como preservar a ordem e evitar atualizações perdidas? Estado externo permite compartilhar dados, mas a implementação ainda precisa de uma estratégia de concorrência por sessão.

<a id="rag"></a>

## 2. RAG e ingestão: preparar conhecimento e consultar evidências

**Imagem 02 · dois pipelines independentes.** A faixa superior responde perguntas; a inferior prepara os documentos consultados.

[![Diagrama 02: consulta com embedding da pergunta e busca filtrada, separada da ingestão assíncrona de documentos com parsing, chunking e indexação.][img-02]][img-02]

RAG significa *Retrieval-Augmented Generation*: a geração recebe contexto obtido por uma busca. Neste caso, esse contexto vem de documentos da clínica. A qualidade depende do conteúdo, da recuperação e da forma como o modelo usa a evidência.

| Aspecto | Ingestão | Consulta / runtime |
| --- | --- | --- |
| Gatilho | Documento novo ou atualizado. | Pergunta que exige conhecimento documental. |
| Entrada | Arquivo original e metadados autorizados. | Pergunta e contexto confiável da requisição. |
| Transformação | Parsing → chunks → embeddings → indexação. | Embedding da pergunta → busca filtrada → contexto. |
| Comunicação | Processamento assíncrono por jobs. | Recuperação dentro do atendimento. |
| Resultado | Versão documental completa disponível para busca. | Trechos relevantes e fontes para a resposta. |

**A pergunta não passa pelo chunking de documentos.** No runtime representado, ela é convertida em um embedding para pesquisar o índice. O chunking ocorre quando documentos são preparados para indexação.

<a id="guia-rag"></a>

### Guia 05 · isolamento, metadados e ciclo de vida

<details>
<summary>Ver a imagem 05 com o detalhamento dos dois pipelines</summary>

[![Guia 05: filtros por tenant, chunks com metadados, estados de ingestão, publicação de versões e recuperação após falhas.][img-05]][img-05]

</details>

<a id="isolamento-rag"></a>

**I1 · Isolamento durante a recuperação.** O retriever impõe `tenantId`, `clinicId`, permissões, status e versão ativa na consulta. Documentos de outro tenant não podem entrar no conjunto devolvido à aplicação, ao cache de resultados ou ao contexto do modelo. Recuperar resultados globais e confiar na LLM para descartá-los quebra essa fronteira.

O contexto recuperado contém dados para análise. Uma frase em um documento pedindo para ignorar regras, acessar outra clínica ou executar uma tool continua sendo conteúdo não confiável. As permissões da execução permanecem sob controle do backend.

**I2 · Fontes e metadados.** O Object Storage mantém o documento original; o índice vetorial guarda representações derivadas. Essa separação permite reindexar sem transformar o banco vetorial na única cópia do conhecimento.

| Campo do chunk | Função no estudo |
| --- | --- |
| `tenantId`, `clinicId` | Delimitar a origem e o escopo autorizado da busca. |
| `documentId`, `chunkId` | Rastrear o trecho e permitir escrita idempotente. |
| `documentType` | Identificar a categoria da informação. |
| `version`, `createdAt` | Identificar a revisão e seu histórico. |
| `permissions`, `status` | Restringir acesso e elegibilidade para consulta, quando aplicável. |
| Texto e referência à origem | Sustentar a resposta e permitir inspeção da fonte. |
| Embedding | Representar o trecho no espaço vetorial usado pela busca. |

O modelo de embedding, sua versão e dimensão também precisam ser rastreados na configuração do índice. Ter a mesma dimensão não basta: documentos e perguntas precisam usar representações compatíveis no mesmo espaço vetorial.

<a id="ingestao-versoes"></a>

**I3 · Ingestão e publicação.** O fluxo proposto é:

1. Validar o upload e salvar o original com sua versão e origem.
2. Registrar o processamento pendente e publicar um job com referência ao objeto.
3. O worker extrai o texto, preserva sua estrutura e cria chunks rastreáveis.
4. Gerar embeddings e gravar os chunks de forma idempotente.
5. Verificar que a nova versão está completa antes de torná-la consultável.
6. Ativar a versão, retirar a anterior das novas consultas e invalidar caches afetados.

```text
PENDING → PROCESSING → INDEXED
                    ↘ FAILED → PENDING, após reprocessamento controlado
```

Uma chave conceitual para upsert é `tenantId + clinicId + documentId + version + chunkId`. Reexecutar o mesmo job não deve criar cópias adicionais dos mesmos chunks. Falhas transitórias recebem tentativas limitadas com espera; falhas persistentes seguem para uma DLQ, onde podem ser inspecionadas e reprocessadas.

Publicar uma versão é uma decisão de consistência. Se 80 de 100 chunks foram gravados, a revisão permanece invisível. O sistema precisa de um mecanismo de ativação que permita consultar somente versões completas. A versão anterior pode continuar ativa enquanto a atualização é preparada, desde que permaneça autorizada e válida.

Salvar o objeto e publicar o job também são operações distintas. Um registro de ingestão e uma rotina de reconciliação devem permitir descobrir documentos salvos cujo evento não foi publicado. A fila, sozinha, não resolve essa lacuna.

**Exemplo:** a clínica altera sua orientação de atendimento de sábado. Até a nova versão ficar pronta, o sistema segue a política de versão ativa. Se a orientação antiga precisar ser revogada imediatamente, sua retirada das consultas tem prioridade sobre a disponibilidade da nova. Retenção, exclusão e invalidação também precisam alcançar índice e caches.

<a id="qualidade-rag"></a>

**Como avaliar o resultado.** Top-K define quantos trechos retornam. Um K maior pode ampliar cobertura e também trazer ruído, consumir contexto e aumentar custo. Similaridade vetorial não representa probabilidade de a informação estar correta. Quando não houver evidência suficiente, o assistente deve explicar o limite ou encaminhar a dúvida.

Na avaliação, separe duas perguntas: **o retriever encontrou a evidência certa?** E **o modelo respondeu de acordo com ela?** Essa separação ajuda a distinguir problemas de indexação, busca e geração.

**Pergunta de revisão:** o que deve acontecer com uma consulta iniciada durante a troca de versão? Defina quando a versão ativa é resolvida e como manter uma visão coerente dos documentos durante aquele atendimento.

<a id="agendamento"></a>

## 3. Tools e agendamento: executar ações com consistência

**Imagem 03 · criação da reserva.** O agente solicita uma capacidade; a Scheduling API valida e grava a operação no banco transacional.

[![Diagrama 03: agente chama a tool de agendamento, API valida regras e reserva atomicamente; aprovação humana aparece como caminho opcional para ações de alto risco.][img-03]][img-03]

O modelo ajuda a interpretar o pedido e explicar o resultado. As regras que impedem acesso indevido, duplicação ou sobreposição de reservas pertencem ao backend e ao banco.

<a id="guia-agenda"></a>

### Guia 06 · contratos de tools e regras A1–A4

<details>
<summary>Ver a imagem 06 com os contratos e as regras da agenda</summary>

[![Guia 06: tools sobre serviços determinísticos, confirmação de intenção, idempotência, concorrência e resultado após commit.][img-06]][img-06]

</details>

| Tool | Fonte consultada ou alterada | Resultado esperado | Validação essencial |
| --- | --- | --- | --- |
| `searchClinicKnowledge` | RAG Service e índice vetorial. | Trechos relevantes e fontes. | Tenant, clínica, permissões e versões ativas. |
| `findProfessionals` | Catálogo da clínica. | Profissionais compatíveis com os filtros. | Vínculo com a clínica e critérios autorizados. |
| `getAvailableSlots` | Scheduling API. | Horários disponíveis no momento da consulta. | Profissional, clínica, período e regras de agenda. |
| `createAppointment` | Scheduling API e banco transacional. | Reserva confirmada ou erro de domínio explícito. | Identidade, autorização, intenção, idempotência e concorrência. |

Uma tool é um adapter com contrato restrito: recebe argumentos definidos, chama uma capacidade e devolve um resultado estruturado. Ela pode invocar um módulo do monólito; uma API remota não é obrigatória para cada tool.

**Exemplo conceitual de argumentos propostos pelo modelo**, após a escolha do paciente:

```json
{
  "professionalId": "prof-123",
  "slotId": "slot-456"
}
```

A aplicação associa identidade autorizada, `tenantId`, `clinicId`, `traceId` e `idempotencyKey` à execução. Esse contexto vem do backend e do workflow, não da confiança em argumentos produzidos pela LLM. Os identificadores do exemplo são fictícios e não definem uma API implementada neste repositório.

<a id="confirmacao"></a>

**A1 · Confirmar a intenção.** O paciente escolhe profissional e horário. Se disser apenas “quais horários estão disponíveis?”, consultar a agenda atende ao pedido; ainda falta uma intenção de reservar. Depois da escolha, a aplicação registra os parâmetros confirmados e inicia a operação.

Há dois momentos distintos: o paciente confirma o que deseja; o sistema confirma a reserva depois do commit. A mensagem de sucesso deve refletir o resultado efetivo do serviço.

<a id="idempotencia"></a>

**A2 · Repetir sem duplicar.** A aplicação cria uma chave por intenção de reserva e a preserva durante retries e retomadas. O escopo inclui tenant, clínica e operação. O serviço associa essa chave ao payload e ao resultado persistido.

| Situação | Comportamento esperado |
| --- | --- |
| Chave nova e payload válido | Tentar criar a reserva e registrar o resultado atomicamente. |
| Mesma chave e mesmo payload, operação concluída | Retornar o resultado já registrado, sem uma segunda reserva. |
| Mesma chave e payload diferente | Rejeitar a reutilização conflitante da chave. |
| Duas chamadas simultâneas com a mesma chave | Coordenar a execução para que somente uma produza o efeito. |
| Nova escolha de horário, confirmada pelo paciente | Registrar uma nova intenção, com sua própria chave. |

O hash do payload ajuda a detectar divergência; ele não substitui a identidade da intenção. Também é necessário definir por quanto tempo a chave permanece válida e o tratamento de requisições atrasadas. Esses problemas são discutidos na [Amazon Builders' Library sobre APIs idempotentes](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/).

<a id="concorrencia"></a>

**A3 · Resolver a disputa pelo horário.** Dois pacientes podem consultar o mesmo horário livre e tentar reservá-lo com chaves diferentes. Deduplicar retries não resolve essa disputa. A persistência precisa garantir que apenas as reservas permitidas pelas regras de capacidade sejam aceitas.

Para uma agenda de slots exclusivos, uma restrição de unicidade pode proteger a combinação clínica, profissional e slot entre reservas ativas. Para consultas com duração variável, horários de início diferentes ainda podem se sobrepor. Nesse caso, é necessário controlar os intervalos. PostgreSQL, por exemplo, oferece constraints de exclusão que permitem expressar restrições desse tipo. Veja a [documentação de constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).

Uma transação, isoladamente, não torna seguro qualquer fluxo “consultar e depois inserir”. A estratégia precisa combinar a regra de integridade com escrita atômica, isolamento ou locks adequados. Cancelamento, capacidade maior que um e fuso horário também precisam estar definidos no modelo de agenda.

**Exemplo de corrida:** Ana e Bruno recebem a opção das 14h. Ana confirma primeiro e a reserva é persistida. A tentativa de Bruno encontra um conflito e recebe alternativas. A decisão vem do serviço transacional, mesmo que o modelo ainda tenha o horário antigo no contexto.

<a id="timeout-agenda"></a>

**A4 · Lidar com timeout e autorização.** Se a API gravou a reserva, mas a resposta se perdeu, um timeout não prova que a operação falhou. A aplicação consulta o resultado da intenção ou repete com a mesma chave. Até reconciliar o estado, informa que a confirmação está pendente.

Cada tentativa revalida autorização e pertencimento de paciente, profissional e horário à clínica. Erros de permissão, payload ou horário ocupado exigem uma resposta de negócio; repetir a mesma operação automaticamente não corrige esses casos.

<a id="aprovacao-humana"></a>

**Aprovação humana proporcional ao risco.** O caminho de Human-in-the-Loop do diagrama serve para ações de alto impacto. A aprovação deve estar vinculada à identidade do aprovador, ao tenant, ao payload e a um prazo. Na execução, o backend revalida as condições. O agendamento comum usa a escolha do paciente e as regras do domínio, sem exigir aprovação interna de um funcionário a cada reserva.

**Pergunta de revisão:** se a confirmação ao paciente falhar depois do commit, é necessário criar uma nova reserva? Explique como separar a persistência da operação da entrega de sua notificação.

<a id="operacao"></a>

## 4. Operação: observar, avaliar e limitar o sistema

**Imagem 07 · responsabilidades transversais.** Segurança, observabilidade, custos e resiliência acompanham os fluxos anteriores; o pipeline de evals avalia mudanças antes da liberação.

[![Guia 07: observabilidade, segurança, controle de custos, resiliência, avaliações offline e dimensionamento do sistema.][img-07]][img-07]

<a id="seguranca"></a>

### Segurança e isolamento de ponta a ponta

O isolamento precisa sobreviver a cada mudança de componente. A verificação começa na entrada e continua nas tools, serviços, consultas e acessos ao estado.

| Fronteira | Controle a projetar | Cenário de verificação |
| --- | --- | --- |
| Canal → entrada | Validar origem, identidade e associação com a clínica. | Evento inválido ou identificador de tenant adulterado. |
| Agente → tool | Restringir capacidades e validar schema, contexto e permissões. | Modelo propõe uma ação fora do escopo permitido. |
| Serviço → banco | Consultar e alterar somente recursos autorizados. | ID de profissional ou agendamento pertencente a outra clínica. |
| RAG → contexto | Recuperar apenas conteúdo elegível no tenant. | Documento de outra clínica é semanticamente muito parecido. |
| Aplicação → state/cache | Isolar chaves e verificar acesso ao conteúdo. | Mesmo texto ou identificador de sessão usado em tenants distintos. |
| Fila → worker | Validar origem e escopo do objeto referenciado. | Job referencia um documento de outro tenant. |
| Aplicação → provider/logs | Minimizar dados e controlar acesso e retenção. | Contexto ou erro contém informações pessoais desnecessárias. |

Prompt injection exige defesa em camadas: separar instruções e conteúdo externo, limitar capacidades e validar a execução. Um guardrail pode auxiliar, mas a autorização continua determinística. O [guia da OWASP sobre prompt injection](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html) detalha validação de tools e aplicação de privilégio mínimo.

O desenho inclui criptografia em trânsito e em repouso, gestão de segredos, minimização, retenção, exclusão e auditoria. Esses pontos orientam a implementação e a análise de privacidade; o diagrama, por si só, não demonstra conformidade com a LGPD. Prompts, documentos e dados pessoais não devem aparecer indiscriminadamente em logs, exemplos públicos ou datasets de avaliação.

<a id="observabilidade"></a>

### Observabilidade: explicar o que aconteceu

Uma resposta errada pode decorrer de uma fonte obsoleta, um filtro incorreto, uma tool inadequada ou uma geração sem evidência. O trace precisa permitir localizar a etapa responsável.

| Sinal | O que ele permite investigar |
| --- | --- |
| `traceId` e spans por etapa | Tempo e falhas entre entrada, agente, retrieval, tools e providers. |
| `conversationId`, tenant e versão do agente | Contexto da execução, com acesso controlado. |
| Modelo, versão de prompt e configuração do retriever | Mudanças de comportamento entre versões. |
| Tool, resultado e número de passos | Ações incorretas, loops e falhas de domínio. |
| Tokens de entrada/saída e custo | Consumo por requisição, tenant e tarefa concluída. |
| Idade da fila e status de ingestão | Atraso na publicação do conhecimento. |
| Sucesso da tarefa e conflitos de reserva | Efeito percebido pelo paciente, além de respostas HTTP bem-sucedidas. |

Traces ajudam a investigar execuções; métricas mostram tendências; audit logs registram ações e resultados relevantes. Identificadores por requisição cabem em traces controlados. Colocá-los indiscriminadamente em labels de métricas pode gerar alta cardinalidade e elevar o custo de observação.

<a id="evals"></a>

### Evals: medir qualidade antes de mudar o comportamento

O pipeline proposto é **dataset → versão do agente → execução → métricas → decisão de liberação**. A versão avaliada inclui prompt, modelo, schemas das tools, configuração de retrieval e revisão do conteúdo ou índice.

| Caso de estudo | Evidência esperada | Tipo de avaliação |
| --- | --- | --- |
| Pergunta cuja resposta existe em um documento | Fonte relevante recuperada e resposta sustentada por ela. | Retrieval + avaliação da resposta. |
| Pergunta sem evidência na base | Reconhecimento do limite, sem inventar política da clínica. | Rubrica e revisão de resposta. |
| Pergunta sobre profissional | Uso do catálogo e explicação coerente com o resultado. | Tool selecionada + argumentos + resultado. |
| Documento com instrução maliciosa | Nenhuma ação não autorizada nem exposição de dados. | Caso adversarial + verificações determinísticas. |
| Acesso cruzado entre tenants | Negativa nas fronteiras de dados e execução. | Testes de autorização e isolamento. |
| Repetição da criação de uma consulta | Um único efeito e resultado recuperável. | Testes de integração e idempotência. |
| Duas reservas para um slot exclusivo | Apenas uma aceita; a outra recebe conflito. | Teste de concorrência. |
| Alteração de modelo ou fallback | Comparação de qualidade, tools, custo e latência. | Regressão sobre o mesmo conjunto de casos. |

LLM-as-a-Judge pode apoiar avaliações de relevância e aderência à evidência, com uma rubrica explícita e calibração humana. Regras como “não houve uma segunda reserva” são verificadas diretamente no estado do sistema. O juiz também pode errar e não substitui essas verificações.

As avaliações começam offline. Um juiz online amostrado pode ser considerado quando trouxer benefício demonstrável frente ao custo e à latência. Critérios de aceitação devem refletir o risco de cada fluxo; uma nota média alta não compensa uma falha de isolamento.

<a id="custos"></a>

### Custos e limites de execução

O custo de um atendimento inclui todas as interações com modelos, retrieval e tools. Medir somente a primeira chamada esconde retries, loops e etapas adicionais.

```text
Custo variável de IA da tarefa ≈
  soma dos custos de entrada e saída de todas as chamadas de modelo
  + embeddings e demais serviços cobrados por uso no atendimento

Custo médio por tarefa bem-sucedida =
  custo total do período, incluindo tentativas malsucedidas
  ÷ número de tarefas concluídas com sucesso no mesmo período
```

Para avaliar o custo completo, acrescente infraestrutura, armazenamento, ingestão e operação com critérios de atribuição definidos. As expressões são um roteiro de medição; o repositório não pressupõe preços de fornecedores.

Limites de passos, tokens, tempo total e consumo por tenant impedem execução sem controle. Reduzir contexto, resumir histórico, ajustar Top-K e usar cache podem economizar recursos; cada mudança deve passar pelas evals para detectar perda de qualidade.

<a id="resiliencia"></a>

### Resiliência: falhas transitórias e efeitos persistidos

| Mecanismo | Papel no fluxo | Cuidado de projeto |
| --- | --- | --- |
| Timeout por etapa | Limitar espera em uma dependência. | Um timeout não desfaz uma operação que já foi gravada. |
| Deadline total | Limitar a duração de toda a tarefa. | Somar chamadas, retries e esperas dentro do mesmo orçamento. |
| Retry com backoff e jitter | Recuperar falhas transitórias e distribuir novas tentativas. | Limitar tentativas e preservar idempotência em operações com efeitos. |
| Circuit breaker | Interromper chamadas repetidas a uma dependência em falha. | Definir recuperação e comportamento alternativo. |
| Fallback | Usar uma alternativa previamente validada. | Outro modelo pode mudar tool calling e qualidade das respostas. |
| Degradação explícita | Informar limites e oferecer funções ainda disponíveis. | A resposta deve refletir o estado real da operação. |

Evite retries independentes em muitas camadas: eles podem multiplicar a carga justamente quando uma dependência está degradada. Encerrar a espera por uma chamada também não garante que o serviço remoto deixou de executá-la.

<a id="capacidade"></a>

### Capacidade e metas de serviço

Antes de escolher quantidade de instâncias, defina o que será medido: latência p95 do atendimento, tempo até o primeiro trecho da resposta, disponibilidade por função, frescor documental e sucesso da tarefa. O p95 é o limite abaixo do qual ficam 95% das observações no período medido.

Para estudo, uma estimativa inicial de concorrência média é **taxa média de chegada × tempo médio no sistema**, usando a mesma janela e assumindo regime estável. Esse cálculo é um ponto de partida; dimensionar picos exige medições de carga e distribuição de latências.

O agente pode escalar horizontalmente com estado externo, mas throughput também depende de quotas do provider, conexões e locks do banco, capacidade do índice e APIs externas. Workers de ingestão podem reagir à profundidade e idade da fila. Limites por tenant e backpressure ajudam a evitar que a carga de uma clínica comprometa as demais.

**Pergunta de revisão:** se o tempo do provider dominar a latência, o que acontece ao dobrar a quantidade de instâncias do agente mantendo a mesma quota externa?

<a id="evolucao"></a>

## 5. Do MVP à escala: evoluir a partir de evidências

**Imagem 08 · decisões, trade-offs e falhas.** Esta prancha conecta os componentes aos problemas que justificam sua adoção.

[![Guia 08: responsabilidades do MVP, evoluções opcionais, trade-offs e cenários de falha para discutir em entrevistas de system design.][img-08]][img-08]

<a id="mvp"></a>

### O que compõe o MVP deste cenário

O recorte inicial atende informações da clínica, busca de profissionais e agendamento. Um monólito modular pode reunir entrada, orquestração, catálogo e agenda; o processamento de documentos ocorre de forma assíncrona.

| Responsabilidade | Recorte inicial | Sinal para considerar evolução |
| --- | --- | --- |
| Canais | Web e/ou WhatsApp conforme o escopo de lançamento. | Necessidade comprovada de novos canais ou comunicação persistente. |
| Orquestração | Um Clinic Assistant, tools restritas e limites de execução. | Evals mostram dificuldades que uma divisão de responsabilidades pode resolver. |
| Modelos | Um provider validado e seleção por configuração. | Requisitos de disponibilidade, capacidade ou custo justificam outras opções. |
| Domínio | Módulos de catálogo e agenda sobre banco transacional. | Carga, isolamento operacional ou autonomia de equipes justificam extração. |
| Conhecimento | Documentos versionados, ingestão assíncrona e retrieval isolado. | Volume, distribuição da carga ou qualidade da busca exigem mudanças. |
| Estado | Storage adequado à continuidade e recuperação da conversa. | Requisitos de throughput, persistência ou coordenação mudam. |
| Cache | Adotar em consultas estáveis quando houver ganho medido. | Repetição e custo justificam ampliar a estratégia. |
| Operação | Traces, evals, quotas, segurança e recuperação básica desde o início. | Metas de serviço exigem maior automação e capacidade. |

O Vector DB representa uma responsabilidade de busca e pode usar uma tecnologia adequada ao contexto, inclusive uma extensão de um banco já adotado. A escolha de produto depende das necessidades de busca, isolamento e operação; o desenho não exige um serviço separado por cilindro.

### O que muda ao adicionar componentes

| Decisão | Benefício possível | Custo ou risco | Evidência a procurar |
| --- | --- | --- | --- |
| Mais agentes | Especialização e contextos menores por tarefa. | Mais chamadas, coordenação, latência e dificuldade de depuração. | Ganho de sucesso que compense o custo total do workflow. |
| Modelo mais capaz | Melhor desempenho em tarefas complexas. | Maior custo ou tempo de resposta, dependendo da opção. | Comparação com o baseline no dataset do domínio. |
| Top-K maior | Mais oportunidades de recuperar evidência. | Ruído, consumo de contexto e custo. | Cobertura da evidência e fidelidade da resposta. |
| Cache | Menos chamadas e menor espera em consultas repetidas. | Conteúdo obsoleto, invalidação e risco de isolamento incorreto. | Taxa de acerto útil, frescor e redução de custo. |
| Ingestão assíncrona | Controle de carga e desacoplamento do upload. | Atraso de publicação e operação de jobs, retries e DLQ. | Tempo entre upload e versão disponível. |
| Serviços separados | Escala e implantação independentes. | Chamadas remotas e consistência distribuída. | Gargalo ou necessidade organizacional identificável. |
| Índices separados por tenant | Maior separação operacional. | Mais índices para criar, atualizar e monitorar. | Requisitos de isolamento, carga e custo de gestão. |
| AI Gateway | Centralização de quotas, roteamento e políticas de providers. | Outra dependência operacional e possível gargalo. | Duplicação real dessas funções entre aplicações. |
| Fallback de provider | Alternativa em falhas ou limitações de capacidade. | Diferenças de comportamento e novas integrações. | Evals e simulações demonstram recuperação aceitável. |

**Frameworks no estudo.** LangChain aparece no desenho como possibilidade de apoio às integrações. LangGraph pode apoiar workflows com estado, persistência e intervenção humana, conforme sua [documentação de arquitetura](https://docs.langchain.com/oss/python/langgraph/overview). A escolha de framework não transfere para ele as regras de tenant, idempotência ou integridade da agenda.

<a id="falhas"></a>

### Cenários de falha para percorrer no diagrama

Use esta tabela como roteiro de análise: encontre a dependência que falhou, descreva o efeito percebido pelo paciente e determine qual evidência demonstraria a recuperação.

| Cenário | Resposta arquitetural esperada | Evidência para estudar ou testar |
| --- | --- | --- |
| Provider de LLM indisponível | Aplicar limites de espera e circuit breaker; usar fallback validado ou informar indisponibilidade. | Encerramento dentro do deadline e comportamento alternativo conhecido. |
| Vector DB indisponível | Informar o limite da busca documental; manter funções de agenda que não dependam dela. | A falha de retrieval não bloqueia uma operação de domínio independente. |
| RAG recupera trechos incorretos | Investigar fontes, filtros e ranking; reconhecer falta de evidência suficiente. | Dataset distingue erro de recuperação de erro de geração. |
| Documento desatualizado | Reindexar, ativar a revisão correta e invalidar caches. | Novas consultas usam a versão elegível esperada. |
| Agente entra em loop | Encerrar por passos, tokens ou tempo, salvando o estado necessário. | Tarefa termina dentro dos limites configurados. |
| Tool de criação chamada duas vezes | Reutilizar a identidade da intenção e recuperar o resultado. | Uma única reserva persistida. |
| Dois pacientes disputam o mesmo horário | Aplicar proteção atômica no domínio/banco e devolver conflito. | Somente a capacidade permitida é ocupada. |
| Tráfego aumenta dez vezes | Medir gargalos, aplicar backpressure e ajustar capacidade de toda a cadeia. | Carga controlada, filas observáveis e efeitos por tenant conhecidos. |
| Uma clínica tenta acessar outra | Negar acesso em serviços, retrieval, state e cache. | Casos cruzados falham mesmo com IDs válidos de outro tenant. |
| Documento contém prompt injection | Tratar a instrução como conteúdo externo e limitar execução por autorização. | Nenhuma ação indevida nem exposição de dados. |
| Consumo de tokens cresce | Alertar, aplicar quotas e investigar contexto, passos e retries. | Custo explicado por tenant, versão e tipo de tarefa. |
| Chamada de LLM ultrapassa o timeout | Encerrar a espera e impedir novas etapas além do deadline; reconciliar ações já iniciadas. | Ausência de execução descontrolada ou duplicação de efeitos. |
| Ingestão falha pela metade | Manter a versão parcial invisível e retomar com escrita idempotente. | Busca retorna apenas versões completas e autorizadas. |

Uma estratégia de recuperação fica mais clara quando especifica **o que continua disponível**, **o que o usuário recebe** e **como verificar o estado final**. “Adicionar retry” não define essas três partes.

<a id="exercicios"></a>

## 6. Exercícios para transformar leitura em prática

As atividades abaixo são propostas de estudo. As verificações descritas ainda precisam ser implementadas em uma aplicação que adote esta arquitetura.

| Exercício | Entrega sugerida | Onde revisar |
| --- | --- | --- |
| Explique um atendimento completo | Trace o pedido “atende sábado e tem dermatologista?” indicando cada fonte consultada. | [Atendimento](#atendimento) e [tools](#guia-agenda). |
| Demonstre isolamento | Use duas clínicas fictícias com documentos parecidos; descreva bloqueios em RAG, agenda, sessão e cache. | [I1](#isolamento-rag) e [segurança](#seguranca). |
| Reproduza um timeout após commit | Descreva primeira execução, resposta perdida e retry da mesma intenção. Conte as reservas finais. | [A2](#idempotencia) e [A4](#timeout-agenda). |
| Simule disputa por horário | Modele duas intenções independentes concorrendo por um slot exclusivo. | [A3](#concorrencia). |
| Interrompa uma ingestão | Escolha uma etapa para falhar e explique como retomar sem expor versão parcial. | [I3](#ingestao-versoes). |
| Compare duas configurações de RAG | Varie chunking ou Top-K e analise recuperação, resposta, custo e latência. | [Qualidade do RAG](#qualidade-rag) e [evals](#evals). |
| Justifique uma evolução | Escolha cache, multiagentes ou AI Gateway e apresente a medição que motivaria a mudança. | [Evolução](#evolucao). |
| Dimensione uma hipótese de carga | Declare premissas, estime concorrência e identifique limites do provider, banco e fila. | [Capacidade](#capacidade). |

**Para apresentar em entrevista:** comece pelo problema e pelas premissas; percorra atendimento, conhecimento e reserva; explique as fronteiras de confiança; escolha um cenário de falha; encerre justificando o MVP e o que faria a arquitetura evoluir. Use cada componente para responder a um problema concreto.

Ao defender uma decisão, registre: **qual problema resolve, quais alternativas existem, qual trade-off aceita e que observação faria você reconsiderar**.

<a id="glossario"></a>

## Glossário de consulta rápida

| Termo | Significado neste cenário |
| --- | --- |
| Tenant | Cliente isolado dentro da plataforma compartilhada; aqui, uma clínica. |
| BFF | Backend orientado às necessidades dos clientes/canais. No desenho, compartilha a camada de entrada com o gateway. |
| AuthN / AuthZ | Autenticação identifica o participante; autorização determina o que ele pode fazer. |
| Agent Service | Aplicação que coordena modelo, estado, tools e limites. |
| Tool calling | Proposta estruturada de chamar uma capacidade, cuja execução é controlada pela aplicação. |
| RAG | Geração apoiada por conteúdo recuperado de uma base de conhecimento. |
| Embedding | Representação numérica usada para comparar conteúdo no espaço vetorial. |
| Chunk / Top-K | Trecho indexado de um documento / quantidade de resultados recuperados. |
| Source of truth | Registro de referência: documentos originais para conteúdo; banco transacional para reservas. |
| Idempotência | Repetir a mesma operação identificada sem produzir efeitos adicionais. |
| Concorrência | Operações independentes disputando o mesmo estado ou recurso. |
| Commit | Confirmação da transação no banco. |
| TTL | Tempo de validade de um dado temporário, como uma entrada de cache. |
| DLQ | Fila para jobs que exigem análise após falhas ou esgotamento de tentativas. |
| Backpressure | Controle da entrada ou do ritmo de trabalho conforme a capacidade de processamento. |
| Evals | Avaliações reproduzíveis do comportamento e da qualidade do sistema. |
| SLO | Objetivo mensurável de nível de serviço. |
| HITL | Participação humana em pontos do fluxo definidos pelo risco e pelas regras da operação. |

<a id="editar"></a>

## Arquivos e edição dos diagramas

```text
.
├── README.md
├── .gitignore
├── draw.io/
│   └── System Design - Clinica IA
└── system-design/
    └── oito imagens JPG, correspondentes às abas 01–08
```

O [arquivo fonte][drawio] contém as oito abas em XML editável do draw.io, embora o nome atual esteja sem extensão. A pasta [system-design](system-design/) contém as exportações usadas neste README.

1. Abra o arquivo fonte no draw.io/diagrams.net pela opção de abrir um arquivo do dispositivo. Se necessário, selecione todos os tipos de arquivo no diálogo.
2. Edite os componentes na aba correspondente, preservando a relação entre a visão do sistema e seu guia: **01 ↔ 04**, **02 ↔ 05**, **03 ↔ 06**.
3. Revise também as abas **07 e 08** quando a mudança afetar operação, segurança, decisões ou cenários de falha.
4. Exporte novamente as abas alteradas para JPG e substitua as imagens correspondentes em `system-design/`.
5. Atualize as explicações do README junto com o desenho. Se mudar o nome de uma imagem, ajuste suas referências no final deste arquivo.

As imagens são estáticas: os botões desenhados nelas não funcionam como navegação do GitHub. Use o [mapa de estudo](#mapa) e os links deste README para transitar entre os assuntos.

As abas 01–03 favorecem a leitura dos fluxos; as abas 04–08 preservam o detalhamento para revisão. O README acompanha as duas camadas com exemplos e critérios para discutir as decisões.

[drawio]: draw.io/System%20Design%20-%20Clinica%20IA
[img-01]: system-design/System%20Design%20-%20Clinica%20IA-01%20%C2%B7%20System%20Design%20%E2%80%94%20Atendimento.jpg
[img-02]: system-design/System%20Design%20-%20Clinica%20IA-02%20%C2%B7%20System%20Design%20%E2%80%94%20RAG%20e%20ingest%C3%A3o.jpg
[img-03]: system-design/System%20Design%20-%20Clinica%20IA-03%20%C2%B7%20System%20Design%20%E2%80%94%20Agendamento.jpg
[img-04]: system-design/System%20Design%20-%20Clinica%20IA-04%20%C2%B7%20Guia%20%E2%80%94%20Atendimento.jpg
[img-05]: system-design/System%20Design%20-%20Clinica%20IA-05%20%C2%B7%20Guia%20%E2%80%94%20RAG%20e%20ingest%C3%A3o.jpg
[img-06]: system-design/System%20Design%20-%20Clinica%20IA-06%20%C2%B7%20Guia%20%E2%80%94%20Tools%20e%20agenda.jpg
[img-07]: system-design/System%20Design%20-%20Clinica%20IA-07%20%C2%B7%20Guia%20%E2%80%94%20Opera%C3%A7%C3%A3o%20e%20evals.jpg
[img-08]: system-design/System%20Design%20-%20Clinica%20IA-08%20%C2%B7%20Guia%20%E2%80%94%20Evolu%C3%A7%C3%A3o%20e%20falhas.jpg
