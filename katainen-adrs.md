(some AI may have been used in creation of these ARDs)

# Predictive Maintenance ADRs

## ADR001 Scheduled Batch for Feature Engineering

Title: Scheduled Batch for Feature Engineering

Status: proposed

Context: The proposed architecture involves collecting high-volume vehicle IoT data (measurements and events) via Pub/Sub and storing it in BigQuery. To build predictive models (like time-to-failure), we require features that aggregate raw telematics, stress, and component data over fixed time windows (e.g., 1 hour or 1 day).

We must decide between a high-latency, cost-efficient batch processing approach or a low-latency, complex stream processing approach for generating these time-window features.

Since wear-and-tear is a slow process, real-time feature generation is not a strict requirement for training the main predictive models.

Decision: We will use **scheduled batch jobs** (e.g., daily or hourly via a service like Cloud Composer/Vertex AI Pipelines) to do feature engineering and aggregation from the data stored in BigQuery. The resulting aggregated features will be stored back into BigQuery for model training and batch prediction purposes.

Consequences:

* **Positive:**
  + **Simplicity and Cost-Efficiency:** Batch processing is simpler to implement and maintain compared to stateful stream processing, reducing development complexity and operational costs.
  + **Data Consistency:** It ensures features are calculated over complete, fixed, and consistent time windows, simplifying model training and debugging.
  + **Scalability:** BigQuery is highly scalable for handling the massive volume of historical data.
* **Negative:**
  + **Increased Prediction Latency (Training Data):** The features used for model training will always lag the current time by the duration of the batch window (e.g., up to 24 hours). This latency is acceptable as time-to-failure prediction inherently involves slow-moving signals.
  + **Inapplicability for Instantaneous Predictions:** This feature pipeline cannot support real-time, low-latency models (e.g., real-time crash detection). A separate, simpler, streaming pipeline would be required for those in future.

## ADR002 Pre-Calculated Low-Latency Cache for Client API

Title: Pre-Calculated Low-Latency Cache for Client API

Status: proposed

Context: The primary users of the predictive models like Field Force require rapid access to predictions like "Vehicle X has Y days until component Z failure."

While the feature engineering and model scoring processes are inherently slow (due to batch processing), the Client API response time must be low-latency to support interactive field operations. Without caching the entire prediction pipeline would need to be real-time.

Decision: We will implement a **two-step prediction serving mechanism**:

1. A **scheduled batch job** will run the predictive models (trained on BigQuery features) to score the entire fleet (or high-priority vehicles) regularly.
2. The results (e.g., vehicle\_id, component, time\_to\_failure\_days, prediction\_timestamp) will be written to a dedicated **low-latency in-memory cache** (like Redis or Memcached).
3. The **Client API** will serve all prediction requests directly from this low-latency cache.

Consequences:

* **Positive:**
  + **Low Latency API:** Guarantees sub-100ms response times for the Client API, which improves field force tools usability.
  + **Decoupling:** Decouples the low-latency serving requirement from the high-throughput batch scoring process, allowing each to be optimized independently.
  + **Cost Management:** Avoids the high compute costs associated with maintaining an always-on, real-time prediction service.
* **Negative:**
  + **Staleness:** Predictions served will be inherently stale, reflecting the time of the last batch scoring run. This staleness is acceptable given the slow nature of wear-and-tear, but the prediction timestamp must be included in the API response.
  + **Infrastructure Overhead:** Introduces and requires maintenance of a new, highly available in-memory data store (the cache).

### ADR003: Model Performance Monitoring Service

Title: Dedicated Model Performance Monitoring Service

Status: proposed

Context: Predictive models can degrade in performance due to concept drift (changes in vehicle usage, new component types) or data quality issues. We need a system to detect when the model's predictions diverge significantly from reality. Specifically, the architecture must compare "time-to-failure" predictions against actual fault events that occur unexpectedly early.

Decision: We will create a **Comparison Service** that integrates with the live fault event stream (Pub/Sub) and the pre-calculated model predictions (from BigQuery or the cache).

1. **Fault Event Ingestion:** The service consumes fault events from Pub/Sub (real-time).
2. **Comparison Logic:** Upon receiving a fault event for a vehicle, the service retrieves the last available prediction for that vehicle/component. It then calculates the discrepancy: (Actual time-to-failure) - (Model's predicted time-to-failure at T-1).
3. **Aggregation & Reporting:** Discrepancy data is aggregated (e.g., average prediction error, frequency of early failures) and stored for visualization in the Operations Center UI.

Consequences:

* **Positive:**
  + **Early Degradation Detection:** Provides quantitative, real-time feedback on model efficacy, enabling administrators to detect model drift long before business impact becomes severe.
  + **Actionable Insight:** Allows administrators to trigger retraining or investigate specific data sources if model performance starts degrading.
  + **Completeness:** Closes the MLOps loop by ensuring that the model's performance in production is continuously validated against ground truth (actual failures).
* **Negative:**
  + **Requires Ground Truth:** Requires a reliable, low-latency stream of actual failure events (the "ground truth") to function correctly.
  + **Complexity:** The Comparison Service adds a new real-time component to the architecture that requires dedicated logic for time-series comparison and state management.

## ADR004 Select BigQuery as the Unified Data Warehouse

Title: Select BigQuery as the Unified Data Warehouse

Status: proposed

Context: The system requires a highly scalable, centralized data store to manage three critical types of data:

1. **Raw Ingested Data:** High volume, high-velocity time-series data (measurements) from vehicles.
2. **Feature Store Data:** Complex, aggregated features computed over fixed time windows.
3. **Model Training Data:** The finalized, labelled datasets used by Vertex AI for training.

The data store must support massive volumes of historical data and enable complex, analytical SQL queries (for feature engineering) while integrating seamlessly with the chosen machine learning platform (Vertex AI).

Decision: We will use **BigQuery** as the unified, serverless data warehouse for storing all raw vehicle measurements, engineered features, and training datasets.

Consequences:

* **Positive:**
  + **Scalability & Performance:** Provides virtually limitless scalability for storing raw and historical data and excellent performance for the complex, large-scale aggregation queries required for feature engineering.
  + **Ecosystem Integration (GCP):** Offers native, high-performance integration with Vertex AI (for model training) and Pub/Sub (for ingestion), minimizing data movement and ETL complexity.
  + **Cost Model:** Utilizes a pay-per-query model which is efficient for large-scale, scheduled batch jobs.
* **Negative:**
  + **Potential for High Query Cost:** Complex, non-optimized SQL queries run against the massive raw data tables can become expensive. Query optimization and data partitioning are critical. (note: haven’t seen in real scenarios)
  + **No Real-Time Point Reads:** BigQuery is optimized for analytical workloads, not low-latency, single-record reads. This reaffirms the need for the separate low-latency cache (ADR 002) for prediction serving.

# Customer Behavioural Model ADRs

## ADR005 Segment-Based Modeling via Unsupervised Clustering

Title: Adopt Segment-Based Modeling via Unsupervised Clustering

Status: proposed

Context: The customer base is highly heterogeneous, making a single predictive model inefficient for diverse use cases (e.g., "job commuter" vs. "weekend leisure user" vs. “tourist at the city”). The goal is to maximize prediction accuracy for various customer actions (time-to-next-use, promo uptake). The proposal includes an explicit step to segment users first using unsupervised learning (K-Means) to create homogenous groups.

Decision: We will implement a **two-phase modeling pipeline** where customer segmentation is a mandatory prerequisite:

1. **Phase 1 (Segmentation):** Unsupervised clustering (K-Means, using features like recency, frequency, usage options) is executed on the customer base to derive distinct ClusterIds.
2. **Phase 2 (Supervised Training):** The ClusterId is added as a highly informative categorical feature to the training dataset. We will prioritize training a dedicated supervised model for each large or significant segment to achieve localized accuracy gains (e.g., Model A for Cluster 1, Model B for Cluster 2).

Consequences:

* **Positive:**
  + **Increased Accuracy:** Segment-specific models are expected to capture nuanced behavioral patterns better than a single, generalized model.
  + **Targeted Marketing:** The segments themselves are valuable for non-ML business intelligence and personalized promotional targeting.
* **Negative:**
  + **Pipeline Complexity:** Introduces a multi-stage ML pipeline (Clustering -> Feature Enrichment -> Supervised Training), which is more complex to deploy, version, and monitor.
  + **Maintenance Overhead:** The Segmentation model must be re-run and validated periodically (Concept Drift Monitoring) to ensure customer assignments remain relevant. The number of supervised models to manage also increases.

## ADR006: Batch processing for Behavioural Feature Calculation

Title: Centralize Complex Behavioral Feature Calculation in Scheduled Batch Jobs

Status: proposed

Context: The customer behavior models rely heavily on complex, derived features that require aggregation over time windows or across event streams, such as "time-to-next purchase," "recency of last service use," and "cumulative consumption." Calculating these values accurately requires a complete view of historical events. While real-time predictions are needed, the core features themselves are often based on a history that evolves slowly.

Decision: All complex, time-series-dependent behavioral features (metrics needed for clustering and supervised training) will be calculated via **scheduled batch jobs** (e.g., daily) running against the central data store (BigQuery). The feature engineering process will output a consolidated feature table with the calculated and scaled/normalized values (0 to 1) for both phases of the pipeline.

Consequences:

* **Positive:**
  + **Consistency and Reproducibility:** Ensures the features used for training, validation, and prediction are calculated using identical logic and complete historical context, eliminating training-serving skew.
  + **Resource Efficiency:** Batch processing is highly efficient for these large, analytical transformations, minimizing real-time compute load.
  + **Alignment:** Reuses the batch processing pattern established in the Predictive Care architecture (ADR 001).
* **Negative:**
  + **Feature Latency:** The features will lag the current moment by the duration of the batch job schedule (e.g., up to 24 hours). This is acceptable for behavioral traits like "time-to-next-use" but precludes models requiring features based on sub-minute interactions.
  + **Scaling Challenge:** If the volume of users or the complexity of the feature calculations grows too large, the nightly batch window may be exceeded, requiring optimization or a move to higher-cost streaming aggregation in the future.

## ADR007 Split into Training and Validation Data Sets

Title: Enforce Time-Based Data Splitting for Validation

Status: proposed

Context: All predictive models, especially those dealing with future user behavior (like time-to-next-use or churn), must be rigorously validated against data they have not seen. Simply splitting the data randomly can lead to "data leakage" where recent events in the validation set influence a model trained on older events, resulting in an artificially inflated performance metric. The training process explicitly mentions using validation data to evaluate performance before final acceptance.

Decision: The data splitting step for all supervised customer behavior models (Time-to-Next-Use, For-Uptake-Customer-per-Promo, At-Risk-Customer) **must** use a **time-based split (temporal split)**. The training set will consist only of data recorded up to a specific historical cutoff date ($T\_{train}$), and the validation set will consist exclusively of data recorded after this cutoff date ($T\_{validation}$).

Consequences:

* **Positive:**
  + **Accurate Generalization Metric:** This split accurately simulates the model's performance in a true forward-looking production scenario, providing a realistic measure of how well the model predicts future behavior.
  + **Reduced Overfitting Risk:** Prevents data leakage from future events into the training process, leading to more robust models.
* **Negative:**
  + **Fewer Training Examples:** Since the validation set must be large enough to be statistically significant, the amount of data available for the training set is reduced compared to a random split.
  + **Requires Date Management:** The data pipeline must correctly manage and enforce the time boundary during the splitting process, adding a dependency on accurate, consistent timestamps in the source data.

# Conversational AI-chatbot ADRs

## ADR008 Use Guardrails

Title: Implement Multi-Stage Moderation and PII Filtering via Guardrails.ai

Status: proposed

Context: The Conversational AI-bot serves as a feature of the customer app allowing conversations. It can handle both typed and spoken queries. To maintain a safe, professional, and compliant service, we must protect the system from harmful/toxic inputs (safety) and prevent the accidental ingestion or processing of Personally Identifiable Information (PII), which is a key requirement for GDPR and CCPA compliance. The proposal specifically mandates the use of Guardrails.ai for this purpose.

Decision: We will integrate **Guardrails.ai** (or a similar specialized moderation service) as the **first dedicated processing step** after the user's query is finalized (post-transcription correction). This component will perform two critical functions:

1. **Moderation:** Scan the user query for toxic, hateful, or unsafe content and, if detected, trigger a pre-defined rejection response, preventing the query from reaching the core LLM and tools.
2. **PII Removal:** Identify and redact or mask any PII (names, phone numbers, addresses, etc.) within the user query before passing the cleaned text to the LLM for reasoning and tool-use.

Consequences:

* **Positive:**
  + **Safety and Compliance:** Ensures a robust layer of protection against toxic inputs and enforces mandatory PII handling compliance, reducing legal and reputational risk.
  + **LLM Focus:** The core LLM receives a clean, focused query, improving the quality and reliability of its reasoning and tool selection.
* **Negative:**
  + **Increased Latency:** Adds a required step in the processing path, introducing an unavoidable, although likely small, latency overhead to every user request.
  + **False Positives:** PII detection and moderation are not 100% accurate; a risk exists that Guardrails may inadvertently redact non-PII information or reject benign queries, necessitating careful tuning and monitoring.

## ADR009 RAG Architecture

Title: Centralize Knowledge Retrieval via LlamaIndex RAG Architecture

Status: proposed

Context: The core purpose of the chatbot is to answer customer questions about the service, vehicles, and status. This information is stored across various unstructured textual sources (documentation, manuals, policies). The LLM requires a mechanism to access this vast, non-vectorized knowledge base accurately and efficiently to provide grounded answers. The proposal specifies using LlamaIndex for this task.

Decision: We will implement **LlamaIndex (or similar RAG framework)** as the primary knowledge retrieval tool for the LLM.

1. **Indexing:** All unstructured textual material (service guides, vehicle specifications, location rules) will be indexed and stored within a dedicated vector database integrated via the LlamaIndex tool.
2. **LLM Tool Integration:** A tool will be exposed to the core LLM with a clear description (e.g., 'Use this tool for questions regarding service policies, vehicle details, or location information').
3. **Context Building:** When the LLM decides to use this tool, the relevant query will be sent to the index, and the retrieved text snippets (the context) will be passed back to the LLM for final answer formulation.

Consequences:

* **Positive:**
  + **Accuracy and Grounding:** Enables the LLM to generate answers based on verified, up-to-date source material, dramatically reducing the risk of hallucinations.
  + **Scalability:** Allows the knowledge base to scale by simply indexing new documents without requiring retraining of the large, expensive core LLM.
  + **Traceability:** Provides a clear path to audit the source of the LLM's answer, improving debugging and content maintenance.
* **Negative:**
  + **Maintenance:** Requires a robust, periodic process for indexing new and updated documents, ensuring the LlamaIndex source remains current.
  + **Chunking Quality:** The final answer quality is highly dependent on the quality of text chunking and embedding, necessitating careful configuration of the indexing process.

## ADR010 Use Function Calling Architecture

Title: Implement Schema-Vetted Tool-Use (Function Calling) for Backend Data Access

Status: proposed

Context: Answering complex customer queries requires real-time or near-real-time access to operational data, such as pre-calculated demand predictions or the current status of the customer's vehicle. This data is housed in various backend systems, which cannot be accessed directly by the LLM. The proposed architecture uses an LLM's function-calling capability to bridge the gap.

Decision: We will strictly enforce a **Schema-Vetted Tool-Use** pattern for all backend data access:

1. **Tool Definition:** Every operational data source (e.g., predicted demand, vehicle status model) will be exposed to the LLM as a **function definition** with a descriptive name, purpose, and clear parameter schema.
2. **Pydantic Vetting:** Before executing any function call proposed by the LLM, the returned function name and parameters **must be validated** against the expected schema (using Pydantic or similar validation library).
3. **Execution and Context Return:** Only valid function calls are executed. The resulting structured data is then injected into the final LLM prompt as "additional context."

Consequences:

* **Positive:**
  + **Controlled Access:** Provides a secure, auditable, and controlled gateway for the LLM to interact with sensitive or high-cost backend systems.
  + **Resilience:** Vetting parameters using Pydantic prevents the LLM from attempting to call functions with ill-formed or garbage arguments, which would otherwise crash the backend service.
  + **Extensibility:** The architecture is easily extensible by defining new tool schemas for future data sources without modifying the core LLM or conversational logic.
* **Negative:**
  + **Debugging Complexity:** Debugging issues may involve tracing errors through three stages: LLM reasoning (Tool choice) -> Pydantic validation (Schema check) -> Function execution (Backend call).
  + **System Prompt Sensitivity:** The success of the tool-use is heavily dependent on the quality of the System Prompt and the clarity of the tool descriptions.

# Operations center chatbot ADRs

## ADR011 Add Plans to Chatbot

Title: Adopt Structured LLM Output for Problem Solving Plans

Status: proposed

Context: The Operations Center Chatbot requires the capability to receive a problem from staff (e.g., "rebalance vehicles in Sector A") and generate a multi-step solution. Unlike the customer-facing bot which returns only natural language answers, this system must return an executable plan. The architecture proposes using a specialized System Prompt to force the LLM into a PLAN mode, returning the solution in a structured, machine-readable format.

Decision: We will enforce the generation of operational plans using a **structured output format** (e.g., JSON or YAML, defined by a strict Pydantic/JSON schema). This plan will contain a list of sequential actions, parameters, and descriptions.

1. **LLM Generation:** The System Prompt will instruct the LLM to use a PLAN schema when the user requests a solution.
2. **Decoupled Execution:** A non-LLM component, the **Plan Driver**, will be implemented to receive the validated structured plan, iterate over the defined steps, and coordinate the execution of corresponding backend tools. The LLM is responsible only for planning, not for execution control.

Consequences:

* **Positive:**
  + **Interoperability:** The structured output allows the UI to easily parse, display, and enable staff to edit individual plan steps before execution.
  + **Reliability:** Decoupling the LLM (planning) from the Driver (execution) enhances reliability and safety. If the LLM produces a flawed plan, the Driver can still handle execution errors gracefully.
  + **Auditing:** Execution logs can be mapped directly to the structured plan steps, simplifying audit trails.
* **Negative:**
  + **Prompt Engineering Complexity:** Maintaining the System Prompt to reliably coerce the LLM into the correct structured output format for complex plans will be challenging and requires continuous monitoring.
  + **Schema Drift:** Any change to the available action tools requires updating the plan structure/schema, potentially breaking the Plan Driver component.

ADR012 Enforce Permissions for State-Changing Tool Calls

Title: Enforce Strict Write-Tool Permissions and Audit Logging Status: proposed Context: The Operations Center Chatbot introduces a new class of tools that are capable of **affecting the state of the system** (e.g., configuration changes, sending deployment commands). This represents a fundamental security shift compared to the customer-facing bot, which only uses read-only tools. Any failure in LLM reasoning or unauthorized access could lead to operational disruption.

Decision: We will implement an elevated security layer for all operational tools capable of state mutation (write access):

1. **Role-Based Access Control (RBAC):** Access to write-capable tools (e.g., set\_vehicle\_config, rebalance\_fleet) will be strictly gated based on the authenticated operator's user role and permissions.
2. **Pre-Execution Policy Check:** The Plan Driver will perform a final authorization check immediately before invoking any write-tool, ensuring the current operator is authorized for that specific action.
3. **Mandatory Audit Logging:** Every successful and failed execution of a write-capable tool, including the originating LLM plan step, the operator ID, and the full parameters, must be logged to a secure, immutable audit trail.

Consequences:

* **Positive:**
  + **Security and Compliance:** Mitigates the high-risk potential of unauthorized or accidental state changes by the LLM or an unprivileged user.
  + **Traceability:** Provides a clear forensic record of who, why, and when an operational change was executed.
* **Negative:**
  + **Tool Management Overhead:** The definition of each tool must now include metadata related to the required permissions and the logging strategy.
  + **Performance Hit:** The mandatory RBAC and logging checks add a small, non-negotiable overhead to the execution latency of every write operation.

## ADR013 Add Human-in-Da-Loop

Title: Mandate Human-in-the-Loop (HIL) for Plan Execution

Status: proposed

Context: The Operations Center Chatbot's most powerful feature—the ability to generate and execute plans that change system state—is also its greatest risk. An LLM-generated plan, even if well-structured, could be logically flawed, inefficient, or contain commands that an operator did not intend. This necessitates a mandatory human review stage.

Decision: All LLM-generated operational plans that involve execution tools **must** include a **Human-in-the-Loop (HIL)** validation step.

1. **UI Review:** The generated structured plan will be presented to the operational staff in a dedicated UI, allowing them to review, modify, or reject any step before any execution is attempted.
2. **Plan State:** The plan must transition through defined states: DRAFT (LLM output), REVIEW (Operator editing = may save the plan and return to editing later), APPROVED (Ready for execution = pressed the submit button), and EXECUTING.
3. **No Autonomous Execution:** The Plan Driver is explicitly forbidden from initiating execution of any plan that has not been explicitly marked APPROVED by an authenticated human operator.

Consequences:

* **Positive:**
  + **Error Prevention:** Catches and corrects logical flaws, unexpected LLM outputs, or suboptimal plans before they can negatively impact the fleet.
  + **Trust and Adoption:** Ensures operational staff trust the system, knowing they have the final authority and control over high-stakes changes.
  + **Legal/Operational Safety:** Provides a mandatory sign-off process that is critical for regulated environments.
* **Negative:**
  + **Increased Latency:** The HIL step inherently introduces unpredictable latency into the problem-solving workflow, as execution is gated by human availability.
  + **UI/Workflow Complexity:** Requires the development of a sophisticated plan editing interface and a robust plan state management system, significantly increasing UI and backend development complexity.

## ADR014 – Integrate Pre-Execution Simulation (Dry Run)

Title: Integrate Pre-Execution Simulation (Dry Run)

Status: Experiemental Future Direction

Context: ADR 013 established the mandatory Human-in-the-Loop (HIL) process to prevent flawed LLM-generated plans from directly impacting the live system. While HIL prevents accidental execution, operators currently lack quantitative data on the efficiency or predictive outcome of a complex plan (e.g., a multi-step rebalancing strategy). We need a mechanism to validate the anticipated impact of a plan using real-time data before it transitions from REVIEW to APPROVED. This makes sense among others if a tasks is called to rebalance the fleet as this may make fairly big changes that affect business.

Decision: We will implement a **Pre-Execution Simulation Tool** that allows operators to run a "dry-run" of any operational plan against a dedicated, isolated simulation environment or a real-time data mirror.

1. **Simulation Tool Integration:** A new internal tool, simulate\_plan\_impact(plan\_steps), will be created. This tool will run the proposed sequence of actions against a replica of the current fleet state and return the simulated results (e.g., key performance indicators, predicted vehicle distribution, estimated energy cost) without committing any changes to the production system.
2. **Workflow Hook:** This simulation will be integrated into the Plan State workflow defined in ADR 013. The UI will expose a "Simulate" function in the REVIEW state, which calls this tool via the Plan Driver.
3. **Operator Context:** The simulated results (e.g., text, charts, key metrics) will be displayed to the operator as a new piece of **Additional Context** in the UI, enabling them to make an informed APPROVED or EDIT decision.

Consequences:

* **Positive:**
  + **Efficiency Optimization:** Allows operators to test and compare multiple plan modifications, ensuring the final executed plan is highly efficient and achieves the intended operational goals (e.g., predicting the best path to vehicle distribution).
  + **Risk Mitigation:** Catches issues related to resource constraints, timing conflicts, or algorithmic flaws that the LLM may have missed, without risking the live fleet.
  + **Operator Confidence:** Increases operator trust in the planning process by providing data-backed evidence of the plan's likely outcome.
* **Negative:**
  + **Infrastructure Cost:** Requires the establishment and maintenance of a dedicated, high-fidelity simulation or real-time data mirror environment, adding operational complexity and cost.
  + **Increased Latency (Simulation Time):** Complex plans may require several seconds or minutes to simulate, which adds latency to the overall HIL workflow (though this latency is explicitly requested by the operator).
  + **Tool Dependency:** The quality of the HIL process is now dependent on the accuracy and fidelity of the simulation environment. Building and maintainig such simulator is not easy-peasy and may fail to lead to results. (Note: this might be a topic for future university co-operation).

# Anomaly Detection ADRs

## ADR015 Fast and Slow Anomaly Detection Pipelines

Title: Implement Hybrid Anomaly Detection Pipeline

Status: proposed

Context: The anomaly detection system must cover a wide range of fleet and business behaviors (usage, operational, KPIs). The definition of 'normal' behavior is complex and often deviations require rapid response. The proposal suggests two detection mechanisms: simple, fast rule-based filtering for known patterns, and complex, slower unsupervised models for novel deviations.

Decision: We will implement a **Hybrid Anomaly Detection Pipeline** with two distinct detection stages:

1. **Rule-Based Pre-filtering (Stream/Real-Time):** Simple, deterministic rules (derived from post-anomaly analysis) will be executed as part of the standard event processing service. This stage provides immediate, low-latency flagging for previously observed or high-risk, known anomalies (e.g., "incident count > 5 in 1 hour"). Alerts from this stage are sent directly to the dashboard and as notifications.
2. **ML-Based Detection (Batch/Micro-Batch):** Unsupervised ML models will be executed on a scheduled basis (daily or hourly micro-batches) using time-series features and rolling metrics. These models are responsible for identifying novel or subtle deviations, and will include a configurable **Thresholding Service** allowing management to define the severity of the deviation required to flag an 'anomaly'.

Consequences:

* **Positive:**
  + **Speed and Coverage:** Achieves low-latency response for critical, known issues via rules, while maintaining the ability to discover new, complex anomalies via ML.
  + **Resource Efficiency:** Saves compute resources by only running complex models on aggregated data in batch, while reserving real-time processing for simple rules.
* **Negative:**
  + **Maintenance Overhead:** Requires continuous effort to convert newly discovered ML anomalies into robust, maintainable rules for the pre-filtering layer. Often this means a Change Management process where proposals are discussed and decided.
  + **Threshold Management:** The configurable thresholding adds a layer of complexity, requiring a dedicated UI and governance to prevent alert fatigue or missed alerts.

## ADR016 Centralize Anomaly Data for LLM Grounding

Title: Centralize Anomaly Data for LLM Grounding

Status: proposed

Context: Operational staff and management need to query the system about current and historical anomalies using the chatbot ("Are there any usage anomalies in Schwabing?"). For the LLM to answer these questions reliably and with grounding, the anomaly data must be consistently stored and exposed via a dedicated tool.

Decision: We will mandate that all detected anomalies (from both Rule-Based and ML-Based stages) are centralized in a dedicated data warehouse table, which is then exposed via a robust **Anomaly Query API**.

1. **Data Warehouse as Source of Truth:** Every anomaly detection event will be logged in the data warehouse, containing metadata like type (usage, operational, KPI), severity, timestamp, affected vehicle(s)/region, and the underlying model score/rule ID.
2. **LLM Tool Integration:** A new LLM tool, query\_anomalies(type, region, time\_range, severity), will be defined. This tool's description will guide the LLM to call the new Anomaly Query API when a user asks about system health, incidents, or unexpected behavior.
3. **Data Refresh:** The API will expose answers are based on the latest available anomaly detections.

Consequences:

* **Positive:**
  + **LLM Grounding:** Ensures the chatbot's responses about system health are factual, consistent, and traceable to the underlying detection engine.
  + **Unified View:** Provides a single, auditable source of truth for all anomaly-related metrics used by dashboards, notifications, and the LLM.
  + **Flexibility:** Allows management to leverage the same data for external reporting and analysis.
* **Negative:**
  + **API Development:** Requires dedicated development of a performant, indexed API optimized for time-series and filtered queries.
  + **Latency Risk:** If the API or data warehouse is slow, the LLM may experience tool-use delays, impacting the responsiveness of the chatbot.

## ADR017 Multi-Channel Notification

Title: Implement Dedicated Multi-Channel Notification Service

Status: proposed

Context: The goal of anomaly detection is to enable rapid action. Notifications must be immediate and reach the right personnel through their preferred channels (operations center dashboard, Slack, WhatsApp). Relying solely on the operations center dashboard is insufficient for immediate response outside of working hours or for mobile staff.

Decision: We will implement a dedicated, decoupled **Notification Service** responsible for fan-out messaging across multiple channels based on anomaly type and severity.

1. **Decoupled Service:** The Notification Service will subscribe to the anomaly output (e.g., a message queue fed by both the Rule-Based and ML-Based stages) rather than being integrated directly into the detection models.
2. **Configurable Routing:** The service will use a configurable mapping table to route alerts:
   * High Severity/Operational -> Dashboard + Dedicated Slack Channel + WhatsApp Group.
   * Medium Severity/Usage -> Data Warehouse + Marketing Dashboard.
3. **Consolidated Alerting:** The service will handle alert consolidation (preventing spam from repeated triggers) and formatting specific to each channel (e.g., rich embeds for Slack, concise text for WhatsApp).

Consequences:

* **Positive:**
  + **Immediate Action:** Ensures key personnel are notified immediately, regardless of where they are monitoring the system.
  + **Decoupling:** Protects the core detection pipeline from failures in external messaging APIs (Slack, WhatsApp).
  + **Targeted Alerts:** Reduces alert fatigue by allowing precise routing based on the anomaly's relevance to different teams (e.g., marketing only sees KPI anomalies).
* **Negative:**
  + **External Integration Complexity:** Adds complexity in managing API tokens, message formatting, and maintaining external service connections (Slack, WhatsApp).
  + **Configuration Burden:** The routing logic requires careful configuration and testing to ensure alerts reach the correct recipients.

# Edge Execution ADRs

## ADR018 Model Execution @ Edge

Title: Shift Predictive Maintenance Model Execution to Edge Devices

Status: proposed

Context: The current cloud-based predictive maintenance (PM) approach is efficient but relies on constant, high-volume telemetry transmission, leading to high data ingestion and network costs. We are exploring the use of GPU-equipped IoT kits on cars and vans to execute PM models locally.

Decision: We will (experiement with) shift the primary execution of high-frequency, telemetry-heavy Predictive Maintenance (PM) models from the central cloud infrastructure to the **on-vehicle IoT kit (Edge Execution)**.

1. **Hardware Mandate:** The IoT kit for applicable vehicles (cars, vans) must be provisioned with a dedicated GPU or other suitable hardware accelerator to handle inference speed requirements.
2. **Edge Model Deployment:** A new **Model Delivery Service** will be created to securely package, version, and deploy specific PM models (e.g., time-to-failure) to the vehicle fleet over-the-air (OTA).
3. **Local Inference & Filtering:** Models run continuously on the edge, consuming local sensor data. The inference results are **filtered locally** and only reported back to the cloud/data warehouse under specific conditions (see ADR 019).

Consequences:

* **Positive:**
  + **Cost Reduction:** Drastically reduces cellular data transmission costs by sending only filtered, actionable insights instead of raw, constant telemetry.
  + **Latency Reduction:** Failure prediction is instant and local, allowing for near-real-time actions on the vehicle (e.g., adjusting operational parameters, local logging).
  + **Resilience:** Models continue to function even with intermittent or lost cellular connectivity.
* **Negative:**
  + **Increased Hardware Cost:** Significant initial investment and ongoing cost for GPU-enabled IoT kits.
  + **Complex Deployment:** Introducing a secure and reliable OTA model delivery and update pipeline adds complexity to the system architecture.
  + **Debugging Challenges:** Debugging model behavior and data lineage on remote, distributed hardware is inherently more difficult than in a centralized environment.

## ADR019 Edge Reports When it has Stuff to Say

Title: Adopt Event-Driven Reporting for Edge Inference Results

Status: proposed Context: With models executing on the vehicle, we need a new reporting strategy that prevents data flooding while ensuring critical information reaches the cloud quickly. Frequent full status reports are no longer economically prudent (or needed).

Decision: Edge devices will use an **Event-Driven Reporting Protocol** for transmitting Predictive Maintenance results, moving away from constant polling/telemetry upload.

1. **Scheduled Reporting:** A low-frequency, scheduled report (e.g., once daily) will transmit the latest time-to-failure prediction and model health status when the vehicle is **idle or charging** to conserve battery and bandwidth.
2. **Threshold-Based Reporting (Anomaly/Change Events):** The vehicle will immediately emit a high-priority event only when:
   * The predicted time-to-failure **crosses a critical threshold** (e.g., a fault is predicted within the next 30 days).
   * A **significant deviation** is detected (e.g., current battery discharge rate is X% faster than the last reported average). This requires a local comparison engine on the IoT kit.
3. **Cloud Ingestion:** The cloud infrastructure will be modified to ingest these sparse, high-value **PM Status Events** rather than a continuous telemetry stream, triggering immediate action workflows (notifications, scheduling).

Consequences:

* **Positive:**
  + **Data Quality:** Central data stores receive high-signal, actionable information rather than low-signal, voluminous telemetry.
  + **Optimized Bandwidth:** Cellular usage is minimized, directly addressing the cost reduction goal.
  + **Immediate Action:** Critical alerts are transmitted instantly when a threshold is breached, enabling faster scheduling of maintenance.
* **Negative:**
  + **Loss of Granularity:** Troubleshooting historical, non-critical events becomes more difficult, as the raw data leading up to a non-triggered event is lost or only stored locally on the device.
  + **Complexity on Edge:** The IoT kit must now manage complex state, threshold logic, and comparative analysis, increasing firmware complexity.

## ADR020 Containerized Edge AI

Title: Use Containerized Inference Environment for Model Portability

Status: proposed Context: The deployment of complex AI models to varied hardware (even within the "GPU-enabled kit" category, chipsets may differ) necessitates a consistent, portable runtime environment. Packaging models with dependencies and ensuring they execute reliably across the heterogeneous fleet is a significant challenge.

Decision: The on-vehicle inference environment will be implemented using a lightweight **Containerized Runtime** (e.g., a slimmed-down Docker or specialized edge container format).

1. **Model Package:** Each PM model will be packaged alongside its specific dependencies, inference engine, and the model weight files into a single, signed container image.
2. **Standardized API:** The container runtime environment on the IoT kit will expose a standardized local API for the core firmware to interact with (e.g., an HTTP/IPC endpoint to send sensor data and receive predictions).
3. **Version Isolation:** Containers allow multiple model versions (e.g., a legacy version and a new experimental version) to coexist safely on the device during rollouts or A/B testing, minimizing the risk of fleet-wide failures.

Consequences:

* **Positive:**
  + **Deployment Consistency:** Ensures the model's execution environment is identical to the testing environment, dramatically reducing "it worked on my machine" errors during deployment.
  + **Hardware Abstraction:** Makes the models more portable across different generations or types of GPU hardware, simplifying the long-term fleet management.
  + **Security:** Provides separation between the model runtime and the core vehicle operating system/firmware.
* **Negative:**
  + **Resource Overhead:** Containers, even lightweight ones, introduce some overhead in storage and memory usage compared to a monolithic firmware binary.
  + **IoT Kit Requirements:** Mandates a robust operating system on the IoT kit capable of managing and running containerized workloads, potentially increasing the minimum hardware specification.
