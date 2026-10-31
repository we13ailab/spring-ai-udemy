# From Java Developer to AI Engineer: Spring AI Fast Track

A practical learning repository for Java and Spring Boot developers exploring LLMs, Spring AI, Retrieval-Augmented Generation (RAG), tool calling, Model Context Protocol (MCP), and AI agents.

The examples progress from calling an LLM from Spring Boot to an email-based customer-support agent that uses MCP tools to retrieve customer/order data, detect duplicate charges, check warranties, issue refunds, and record support tickets.

## Who is this for?

- Java developers familiar with Spring Boot, REST APIs, Maven, and databases.
- Backend engineers learning to integrate LLMs into existing applications.
- Developers who want to expose Java business capabilities as AI tools.
- Engineers learning how MCP clients, MCP servers, and agents work together.

## Learning roadmap

| Directory | Topic | What to learn |
| --- | --- | --- |
| `section01` | LLM providers and first integrations | Connect Spring Boot applications to providers/runtimes such as OpenAI, Ollama, AWS Bedrock, and Docker Model Runner. |
| `section02` | Spring AI fundamentals | ChatClient, prompts, prompt templates, streaming, and structured output. |
| `section04` | Conversation memory | Supplying conversation history across turns. |
| `section05` | RAG foundations | Retrieval-augmented chat, vector stores, and semantic retrieval examples. |
| `section06` | Tool/function calling | Let the model invoke Java application capabilities, such as help-desk operations. |
| `section07` | Model Context Protocol | MCP clients and servers, STDIO and remote connections, filtering, logging, progress, sampling, and elicitation. |
| `section08` | Response evaluation | Self-evaluating AI response examples. |
| `section09` | Advanced examples and observability | Help-desk/RAG examples and Prometheus-oriented monitoring configuration. |
| `section_10` | Multimodal AI | Image and audio examples. |
| `section_11` | End-to-end support agent | A separate MCP server and support-agent application working together. |

The source tree skips section 03 and uses underscores in `section_10` and `section_11`. The table is a guide to the current source layout, not a claim that every optional example is enabled or production-hardened.

## Core concepts

### 1. Calling an LLM from Spring Boot

Spring AI's `ChatClient` gives a Spring application an API for sending prompts and receiving model output.

```text
HTTP request -> Spring MVC controller -> ChatClient
             -> LLM provider -> model response -> HTTP response
```

A minimal illustrative example:

```java
@RestController
@RequestMapping("/api")
public class ChatController {
    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt(message).call().content();
    }
}
```

The model is an external dependency, much like a payment or shipping API. It does not automatically know your application's private data.

### 2. Prompts, streaming, structured output, and memory

- **Prompt templates** separate reusable instructions from request-specific values.
- **Prompt stuffing** adds context to a prompt; it is simple, but not a substitute for a well-designed retrieval system.
- **Streaming** returns generated content incrementally.
- **Structured output** maps model responses to Java types/DTOs. Validate model-generated values before using them.
- **Chat memory** supplies selected conversation history for follow-up questions. It is not the same as a model permanently learning about a user.

### 3. Retrieval-Augmented Generation (RAG)

RAG retrieves relevant external information and supplies it to the model at question time.

```text
Ingestion: documents -> chunking -> embeddings -> vector store

Question -> embedding -> similarity retrieval -> relevant chunks
          -> prompt + context -> LLM -> grounded response
```

A vector database such as Qdrant stores embeddings and metadata. At query time, the application retrieves semantically similar content and adds it to the model's context.

Important topics to investigate:
- Document parsing, chunk size, overlap, and embedding model selection.
- Metadata filters and tenant/user access controls.
- Similarity search, hybrid search, and reranking.
- Retrieval evaluation, citations, freshness, and stale documents.
- Safe behaviour when no useful context is found.

Inspect the enabled source paths before assuming every RAG variation is active.

### 4. Tool calling / function calling

Tool calling lets the model request an operation exposed by the application. The application validates and executes the operation, then returns the result to the model.

```text
User request -> LLM selects a tool -> Spring AI invokes Java method
             -> service applies business rules/persists data
             -> tool result returns to LLM -> user-facing answer
```

For example, a help-desk assistant may create a ticket or look up ticket status. The model may select a tool, but authentication, authorization, validation, and business rules must remain in Java services.

### 5. Model Context Protocol (MCP)

MCP standardises how an AI host/client discovers and invokes capabilities exposed by MCP servers.

```text
User -> Spring Boot AI host -> ChatClient <-> LLM
                            |
                         MCP client
                            |
                         MCP server
                            |
                    Java services / data / APIs
```

The repository demonstrates MCP client/server patterns, STDIO and remote connections, tool discovery/invocation, filtering, logging, progress, sampling, and elicitation.

The MCP client example makes discovered tools available to the ChatClient through a `ToolCallbackProvider`, for example:

```java
this.chatClient = chatClientBuilder
        .defaultTools(toolCallbackProvider)
        .build();
```

MCP is a protocol for tool/context integration; it is not itself a vector database or RAG engine. An MCP server can expose a semantic-search/RAG operation as a tool, while the server-side application implements retrieval.

## End-to-end example: customer support agent

Explore `section_11/support-agent-demo/`. It contains two Spring Boot applications.

### MCP server

Path: `section_11/support-agent-demo/mcp-server/`

The server exposes business capabilities backed by JPA and MySQL. Read/query tools include:

- `lookup_customer_by_email` — find a customer.
- `get_customer_orders` and `get_order_by_number` — retrieve order data.
- `search_products` and `get_product_by_sku` — find catalog details.
- `detect_duplicate_charges` — compare captured payments with the order total.
- `check_warranty` — check product warranty eligibility.
- `get_customer_ticket_history` — retrieve prior support interactions.

Action tools include:
- `issue_refund` — record a refund and optionally associate it with a payment transaction.
- `log_support_ticket` — record the support request and its outcome.

Separating read/query tools from write/action tools is an important safety boundary.

### Support-agent application

Path: `section_11/support-agent-demo/support-agent/`

The application polls a Mailpit inbox, converts incoming messages into an application model, passes the email to a Spring AI ChatClient, and makes MCP tools available through a `ToolCallbackProvider`. The model can gather facts and select tools; the application handles the workflow and result.

The inbox monitor resets a message to unread after a processing failure so it can be retried. Production systems should additionally consider durable processing, deduplication, bounded retries, dead-letter handling, and idempotency.

### Example: duplicate-charge complaint

A customer emails: "My order #4471 was charged twice. Please refund the extra amount."

```text
Customer email -> Mailpit -> scheduled inbox monitor
               -> Support agent / ChatClient
               -> MCP client -> MCP server
               -> customer/order lookup
               -> payment inspection / duplicate-charge detection
               -> refund if permitted
               -> support-ticket logging
               -> tool results -> agent response
```

This is an illustrative sequence; the model selects tools based on the message and its instructions.

**Production safety:** Do not give an agent unrestricted authority to issue refunds. Enforce caller/customer/order authorization, refund limits, approval policies, transaction-level idempotency, audit logging, and a clear boundary between recommendation and execution. Treat email content and retrieved documents as untrusted input to mitigate prompt injection.

## Technologies and tools referenced

| Area | Technologies |
| --- | --- |
| Language/build | Java, Maven |
| Framework | Spring Boot, Spring AI, Spring MVC |
| Models/providers | OpenAI, Ollama, AWS Bedrock, Docker Model Runner |
| Retrieval | RAG concepts, Qdrant |
| Tool integration | Spring AI MCP client/server, STDIO and remote transports |
| Persistence | Spring Data JPA, MySQL |
| Local infrastructure | Docker, Docker Compose, Mailpit |
| Observability | Micrometer, Prometheus, Grafana, OpenTelemetry, Jaeger |

Exact versions and dependencies are defined per module.

## Getting started

1. Clone the repository and open the section you want to study.
2. Inspect that module's `pom.xml` for Java, Spring Boot, and Spring AI versions.
3. Read its `application.properties` and Compose files, if present.
4. Configure required provider credentials and local services.
5. Start infrastructure, then run the Spring Boot module with the Maven wrapper if available or your local Maven installation.
6. Exercise the endpoints and inspect the source code. Use `SpringAI.postman_collection.json` where applicable.

### API keys and configuration

Never commit real API keys, passwords, or production credentials. For example, the support-agent module reads the OpenAI key from an environment variable:

```bash
export OPENAI_API_KEY="your-api-key"
```

Configure database and service settings through environment variables or an appropriate local, untracked configuration file. Review Compose files before starting containers.

### Version compatibility

These are independent course modules and may not share the same Java/Spring versions. Check each module's Maven configuration rather than assuming one JDK runs everything. The support-agent demo currently declares Java 25, Spring Boot 4.1.0, and Spring AI 2.0.0; use compatible tooling for that module.

## Suggested learning order

1. **Section 01:** Provider integration, credentials, and request/response basics.
2. **Section 02:** ChatClient, prompt design, streaming, and structured output.
3. **Section 04:** Conversation history and memory.
4. **Section 05:** RAG, retrieval, and vector stores.
5. **Section 06:** Tool calling and Java service integration.
6. **Section 07:** MCP clients, servers, and reusable capabilities.
7. **Section 08:** Response evaluation and its limitations.
8. **Section 09:** Advanced examples and observability.
9. **Section 10:** Multimodal image/audio interactions.
10. **Section 11:** The complete email-to-support-resolution workflow.

For each module, run the app, send a request, set breakpoints in the controller/service/tool, inspect prompts and tool arguments, trace database access, and test provider/MCP/database failures.

## Beyond the course: production engineering checklist

### RAG quality
- Parsing, chunking, embeddings, metadata, hybrid retrieval, reranking, and evaluation.
- Tenant/user access filtering before retrieved content reaches the model.
- Source attribution, freshness, and safe responses when evidence is missing.

### Agent and tool reliability
- Timeouts, bounded tool-call loops, retries, circuit breakers, and explicit error contracts.
- Idempotency keys for write operations and deduplication for incoming events.
- Durable workflow state for long-running tasks and human approval for high-impact actions.

### Security and privacy
- Authentication and authorization inside tool/business-service layers.
- Secret management, credential rotation, and safe logging.
- Prompt-injection defenses for emails, user messages, and retrieved documents.
- PII minimisation, retention policies, and audit trails.
- Never use prompt instructions as the only authorization boundary.

### Observability and cost
- Track model/provider, latency, token usage, errors, tool calls, and end-to-end workflow time.
- Trace requests across the app, model provider, MCP client/server, and downstream systems.
- Monitor retrieval quality and agent outcomes, not just HTTP success.
- Add budgets, rate limits, and fallback behaviour.

### Testing and deployment
- Unit tests for business rules and tool methods; integration tests for databases and MCP.
- Stub model responses for deterministic tests where practical.
- Evaluation datasets for RAG answers and agent decisions.
- Container security, health checks, deployment configuration, scaling, and rollback plans.

## Official documentation and useful resources

### Spring AI and model providers
- [Spring AI documentation](https://docs.spring.io/spring-ai/reference/index.html) — modules, integrations, and configuration.
- [OpenAI API documentation](https://platform.openai.com/docs/overview) — API concepts and model integration.
- [Ollama](https://ollama.com/) — run supported open models locally.
- [Amazon Bedrock](https://aws.amazon.com/bedrock/) — managed access to foundation models.
- [Docker Model Runner](https://docs.docker.com/ai/model-runner/) — run and manage models with Docker.
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) — local container environment.

### RAG, vectors, and MCP
- [Qdrant](https://qdrant.tech/) — vector database and similarity search.
- [Model Context Protocol](https://modelcontextprotocol.io/) — MCP concepts and specification.

### Observability
- [Micrometer](https://micrometer.io/) — application metrics instrumentation.
- [Prometheus](https://prometheus.io/) — metrics collection and alerting.
- [Grafana](https://grafana.com/) — dashboards and visualisation.
- [OpenTelemetry](https://opentelemetry.io/) — traces, metrics, and logs instrumentation.
- [Jaeger](https://www.jaegertracing.io/) — distributed tracing.

### Foundational reading and utilities
- [Attention Is All You Need](https://arxiv.org/abs/1706.03762) — Transformer architecture research paper.
- [OpenAI Tokenizer](https://platform.openai.com/tokenizer) — inspect tokenisation and estimate token counts.

## Repository contents at a glance

- `README.md` — course overview, learning roadmap, and reference guide.
- `SpringAI.postman_collection.json` — Postman collection for applicable API examples.
- `section01`–`section_11` — independent examples and projects described above.

For questions or issues, open a GitHub issue and include the section, module, Java version, and relevant error details. Never include API keys or other secrets.

Happy Learning! 🚀
