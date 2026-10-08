# System Design V2 — Requisitos

Refatore o System Design existente. Não recrie a arquitetura do zero.
Preserve componentes válidos da V1 e evite overengineering.

## Princípios

- Plataforma multi-tenant.
- `tenantId/clinicId` vem de contexto confiável, nunca da LLM.
- LLM interpreta e escolhe capacidades.
- Agent orquestra.
- Tools representam capacidades de negócio.
- Tools não acessam bancos diretamente.
- Fluxo: Agent → Tool → API Gateway → serviço determinístico → datastore.
- AI Gateway governa acesso a LLMs.
- PostgreSQL é source of truth transacional.
- pgvector suporta RAG.
- Redis suporta sessão, cache, rate limit, quota e controle de consumo.

## Entrada

Web Chat e WhatsApp.

WhatsApp:
Provider → Webhook → API Gateway → Conversation Service.

Webhook:
- validar origem/assinatura;
- deduplicar eventos;
- resolver tenant/usuário;
- responder rapidamente;
- delegar processamento pesado para fila quando necessário.

## Redis

Separar responsabilidades e TTLs:

`session:{sessionId}`
- contexto recente;
- últimas mensagens;
- intenção;
- TTL curto.

`answer-cache:{patientId}:{questionHash}`
- respostas reutilizáveis pelo mesmo paciente.

`shared-answer-cache:{clinicId}:{questionHash}`
- apenas conteúdo não pessoal reutilizável.

`usage:{patientId}`
- requests;
- LLM calls;
- RAG calls;
- tool calls;
- input/output tokens;
- custo estimado.

Usar para rate limit, quotas, FinOps e Denial of Wallet.

## Denial of Wallet

Aplicar antes das operações caras:

- rate limit;
- quotas;
- concurrency limit.

Agent Runtime:
- maxSteps;
- maxLLMCalls;
- maxToolCalls;
- maxInputTokens;
- maxOutputTokens;
- maxCost;
- timeout;
- maxRetries.

Limites são determinísticos e não podem ser aumentados pela LLM.

## AI Gateway

Agent → AI Gateway → Model Router → Provider.

Rotas lógicas, por exemplo:
- simple-extraction;
- general-chat;
- summarization;
- complex-reasoning;
- document-analysis.

Responsabilidades:
- model routing;
- fallback;
- timeout;
- retry limitado;
- backoff/jitter;
- circuit breaker;
- token/cost accounting;
- provider credentials;
- observability;
- budget.

Evitar retry amplification.
Fallbacks devem ser cobertos por evals.

## API Gateway

Responsabilidades:
- auth;
- TLS;
- rate limiting;
- quotas;
- routing;
- logs;
- metrics;
- tracing.

Não colocar regra de negócio complexa no Gateway.

## Tools

Exemplos:

- `searchClinicKnowledge`
- `getAvailableSlots`
- `getPatientData`
- `createAppointment`
- `cancelAppointment`
- `updateAppointment`

Evitar:
- `executeSQL`
- `callAnyUrl`
- `executeShell`
- HTTP genérico.

Operações com side effects devem possuir idempotência.

## RAG / pgvector

Ingestão:
document → security validation → chunking → embeddings → pgvector.

Consulta:
question → embedding → authorization/metadata filtering → vector search → Top K → LLM.

Filtros obrigatórios quando aplicável:
- tenantId;
- clinicId;
- visibility;
- documentType;
- patientId.

Top K possui limite determinado pelo backend.

Avaliar HNSW/IVFFlat e índices tradicionais para metadados.

## Prompt Injection

Todo documento é untrusted.

Pipeline deve contemplar:
- formato/tamanho;
- malware scanning;
- extração segura;
- sanitização/classificação.

Conteúdo recuperado é DATA, nunca INSTRUCTION.

Mesmo que a LLM seja manipulada, autorização e tools devem impedir dano.

## PostgreSQL / dados sensíveis

PostgreSQL armazena dados oficiais:
- pacientes;
- consultas;
- agenda;
- prontuário;
- alergias;
- pagamentos.

Hash não é reversível.
Quando o dado precisar ser recuperado, utilizar criptografia.

Aplicar:
- encryption at rest/in transit;
- KMS;
- key rotation;
- least privilege.

## Agendamento e concorrência

Na confirmação de horário, não confiar no cache.

Revalidar no PostgreSQL.

Garantir apenas um agendamento por slot usando estratégia adequada:
- unique constraint;
- operação atômica;
- optimistic/pessimistic locking;
- SELECT FOR UPDATE quando adequado.

A confirmação do slot exige consistência forte.

Consistência eventual é aceitável para cache, analytics, notificações etc.

## Cache de agenda

Pode existir com TTL curto.

Create/cancel/reschedule devem invalidar cache relevante.

PostgreSQL permanece source of truth.

## Escalabilidade

Preparar serviços stateless para escala horizontal:

Load Balancer
→ N instâncias de Webhook/Conversation/Agent/Scheduling/Knowledge Services.

Estado compartilhado em Redis/PostgreSQL/Vector Store/Object Storage.

## Mensageria

Usar processamento assíncrono quando apropriado:
- notificações;
- audit events;
- WhatsApp outbound;
- document ingestion;
- embeddings;
- analytics.

Considerar Kafka/RabbitMQ/SQS conforme necessidade real.

## Resiliência

- timeout;
- retry controlado;
- exponential backoff;
- jitter;
- circuit breaker;
- fallback;
- DLQ;
- bulkhead quando necessário.

Writes só podem sofrer retry quando forem idempotentes.

## Auditoria e compliance

Registrar eventos relevantes, não necessariamente conversas completas:

- actor;
- tenant;
- patient/resource;
- action;
- result;
- timestamp;
- correlationId.

Aplicar:
- LGPD;
- data minimization;
- purpose limitation;
- retention;
- least privilege;
- tenant isolation.

## Observabilidade

Tradicional:
- logs;
- metrics;
- traces;
- latency;
- errors;
- throughput;
- cache hit ratio.

IA:
- provider/model;
- prompt version;
- tokens;
- cost;
- LLM calls;
- tool calls;
- RAG chunks/scores;
- retries/fallbacks;
- agent steps;
- eval results.

## Evals

Cobrir:
- tool selection;
- arguments;
- groundedness;
- hallucination;
- retrieval relevance;
- trajectory;
- security;
- cost;
- latency;
- fallback models.

Combinar deterministic evals, LLM-as-Judge e avaliação humana quando necessário.

## Fluxos obrigatórios no diagrama

1. FAQ: Redis miss → Agent → RAG → LLM → Redis.
2. FAQ repetida: Redis hit, sem RAG/LLM.
3. Consulta de agenda → Scheduling API → PostgreSQL.
4. Agendamento → idempotency → concorrência → transaction → audit.
5. LLM indisponível → AI Gateway → fallback.
6. abuso/script → Gateway → rate limit/quota → block antes da IA.

## Diagrama

Atualizar o System Design existente em Mermaid.

Separar visualmente:
- Channels;
- Edge;
- Application;
- Agent/AI;
- Tools;
- Deterministic Services;
- Data;
- Async;
- Security/Observability.

Não adicionar tecnologia sem necessidade concreta.