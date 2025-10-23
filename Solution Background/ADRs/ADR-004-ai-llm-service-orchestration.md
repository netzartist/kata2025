# ADR 4: AI/LLM Service with Multi-Model Orchestration

## Status

Decided

## Context

MobilityCorp's competitive advantage relies heavily on AI-powered features serving different use cases with varying requirements:

**Customer-facing AI**:
- Natural language food ordering: "authentic local cuisine, vegetarian, serves 2"
- Welcome package customization: "breakfast items, coffee, snacks for 2 people"
- Conversational support and booking assistance
- Personalized recommendations for vehicles, delivery options, restaurants

**Operational AI**:
- Demand forecasting: Predict vehicle availability needs per location
- Dynamic pricing: Optimize rates based on demand, time, weather, events
- Route optimization: Field staff battery swaps and delivery coordination
- Churn prediction: Identify at-risk customers and intervention opportunities

**Analytical AI**:
- Fleet utilization analysis across cities and vehicle types
- Delivery partner performance and matching
- Customer behavior patterns and lifetime value prediction

**Technical challenges**:
- Different models optimized for different tasks (conversation vs classification vs forecasting)
- Mobile apps need lightweight inference; back office can use complex models
- Field staff app requires offline inference for route optimization
- Real-time requirements vary: chat needs <500ms, forecasting can be batch
- Cost management across different model providers (OpenAI, Anthropic, open-source)
- **API fragmentation**: Many LLM services have different APIs while still having similar capabilities
- Need to switch providers based on cost, availability, and capabilities without code changes

## Decision

Build a centralized **AI/LLM Service** that orchestrates multiple models and provides unified API for all applications.

**Use LiteLLM as gateway** supporting 100s of providers, adapting to an OpenAI format as the de-facto industry standard. This enables:
- Standardized API across all LLM providers (OpenAI, Anthropic, Azure, Cohere, Bedrock, etc.)
- Switch providers without changing application code
- Select least expensive model with required capabilities for cost optimization
- Centralized logging, monitoring, and rate limiting

**Architecture**:
- **LiteLLM Gateway Layer**: Translates OpenAI-compatible requests to provider-specific APIs
- **Model router**: Routes requests to appropriate model based on task type, cost, and requirements
- **Multi-model support**:
  - Conversational AI: Claude/GPT-4/Gemini for natural language understanding
  - Embeddings: Dedicated embedding models for vector store (OpenAI, Cohere, local models)
  - Specialized models: Time-series forecasting, computer vision (vehicle damage), NLP classification
  - Open-source fallbacks: Self-hosted models (Llama, Mistral) for offline and cost-sensitive scenarios
- **Inference modes**:
  - Real-time API: Low-latency for customer chat (<500ms SLA)
  - Batch processing: Overnight jobs for demand forecasting, analytics
  - Edge deployment: Lightweight models on mobile for offline field staff routing
- **RAG integration**: Connect to Vector Store Service (ADR-003) for context-aware responses
- **Prompt management**: Versioned prompts, A/B testing, and observability

**API patterns**:
```
POST /ai/chat - Conversational interactions with context
POST /ai/classify - Intent classification, sentiment analysis
POST /ai/embed - Generate embeddings for vector storage
POST /ai/predict - Forecasting, recommendations, optimization
GET /ai/models - Available models and capabilities
```

## Alternatives Considered

**Direct model provider integration in each app**: Rejected because it creates tight coupling, duplicates prompt engineering, and makes model switching expensive.

**Use native provider APIs without gateway**: Rejected because each provider has different API formats, making switching providers require code changes across all applications. No central point for logging and cost optimization.

**Single model for all use cases**: Rejected because no single model optimizes for conversational AI, time-series forecasting, AND edge deployment constraints.

**Build custom API adapter layer**: Rejected because LiteLLM already provides proven, maintained adapters for 100+ providers. Building custom would duplicate effort and lack community support.

**Build custom models for everything**: Rejected due to data requirements, ML expertise, and time-to-market. Leverage foundation models for general tasks, specialize only where necessary.

## Consequences

**Positive**:
- **Provider flexibility**: Swap models without changing app code (OpenAI → Anthropic, add open-source options)
- **Automatic cost optimization**: Standardizing on one API enables selecting the least expensive model with required capabilities
- **Centralized observability**: Single point for logging, monitoring AI performance, costs, and latency across all providers
- **Consistent experience**: Unified prompt engineering and quality control regardless of underlying provider
- **Vendor negotiation power**: Easy to switch providers based on pricing, putting pressure on vendors
- **Offline capability**: Pre-deploy lightweight models to field staff app
- **Rapid iteration**: A/B test prompts and models without app deployments

**Negative**:
- **Extra hop overhead**: Introduces one additional network hop compared to using native provider services directly
  - *Mitigation*: Deploy LiteLLM gateway close to services, use HTTP/2, cache responses where possible
  - *Acceptable tradeoff*: ~5-10ms added latency for flexibility and observability is worthwhile
- **LiteLLM dependency**: Reliance on open-source project for critical path
  - *Mitigation*: LiteLLM is widely adopted, actively maintained, and can be forked if needed
- **Single point of failure**: AI service outage affects all features
  - *Mitigation*: Graceful degradation, cached responses, fallback to rule-based logic, multi-region deployment
- **Model versioning complexity**: Coordinating model updates across multiple apps
  - *Mitigation*: Semantic versioning, backward compatibility guarantees
- **Cost concentration**: All AI costs flow through one service, need careful budgeting
  - *Mitigation*: Per-app quotas, usage monitoring, automatic cost alerts

## Technical Implications

**Infrastructure**:
- **LiteLLM deployment**: Containerized gateway in Kubernetes with horizontal auto-scaling
- **Provider configuration**: Manage API keys for multiple providers in secrets vault (OpenAI, Anthropic, Azure, etc.)
- **Self-hosted models**: GPU nodes for local models (Llama, Mistral) accessible via LiteLLM
- **Horizontal scaling**: Based on request volume and model type
- **Model registry**: Versioning for specialized models (MLflow or similar)

**Integration**:
- Event-driven: Publish AI insights to event bus for downstream consumption
- Sync API: Real-time chat and customer-facing features
- Batch jobs: Scheduled forecasting and analytics pipelines

**Data flow**:
1. App sends request to AI Service with task type and context (OpenAI-compatible format)
2. AI Service router selects optimal model based on SLA, cost, and availability
3. For RAG tasks: Query Vector Store Service (ADR-003) for relevant context
4. Request forwarded to LiteLLM gateway with selected model identifier
5. LiteLLM translates to provider-specific API format and calls the model
6. Model generates response with confidence scores
7. Response logged for quality monitoring, cost tracking, and fine-tuning
8. Result cached if appropriate for reuse
9. Return to application in standardized format

**Security**:
- API keys and model credentials in secrets management
- Rate limiting per app and per user
- PII scrubbing before logging prompts
- Content filtering for safety

**Observability**:
- **LiteLLM built-in tracking**: Request/response logging, latency, success rates per provider
- **Cost tracking**: Automatic cost calculation by app, model, feature, and provider
- **Provider comparison**: Side-by-side metrics to inform model selection decisions
- **Latency tracking**: P50/P95/P99 by model, task type, and provider
- **Quality metrics**: User feedback, task completion rates, response accuracy
- **Model drift detection**: For specialized models and embedding consistency
- **Alerting**: Anomaly detection for cost spikes, latency degradation, error rates

## Migration Path

**Phase 1**: Deploy LiteLLM gateway with OpenAI and Anthropic providers for natural language food orders
**Phase 2**: Add RAG capabilities connecting to Vector Store Service (ADR-003)
**Phase 3**: Integrate additional providers (Azure OpenAI, Cohere, Gemini) for cost comparison
**Phase 4**: Migrate demand forecasting and dynamic pricing to batch inference
**Phase 5**: Deploy self-hosted models (Llama, Mistral) accessible via LiteLLM for cost-sensitive scenarios
**Phase 6**: Deploy edge models to field staff app for offline route optimization
**Phase 7**: Implement automatic model selection based on cost/performance tradeoffs and A/B testing framework
