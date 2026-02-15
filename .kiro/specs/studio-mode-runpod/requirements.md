# Requirements Document

## Introduction

Studio Mode is a specialized "AI Studio" workflow integrated with RunPod that enables developers to perform heavy AI workloads (fine-tuning, RAG pipeline setup, model deployment) through natural language commands. The agent auto-provisions GPU infrastructure on RunPod, orchestrates multi-step AI engineering pipelines, and delivers deployable endpoints — removing infrastructure barriers and democratizing AI engineering within the OpenHands platform.

## Glossary

- **Studio_Mode**: A specialized workflow mode within OpenHands that orchestrates GPU-accelerated AI workloads on RunPod infrastructure through natural language commands.
- **Studio_Session**: A single end-to-end execution of a Studio Mode workflow, from infrastructure provisioning through job completion and optional deployment.
- **RunPod_Runtime**: The OpenHands runtime implementation that provisions and manages GPU pods on RunPod (as described in the existing RunPod integration design).
- **Pipeline_Orchestrator**: The backend component that decomposes a user's natural language AI task into discrete, ordered pipeline stages and coordinates their execution.
- **Pipeline_Stage**: A discrete step within a Studio Mode workflow (e.g., data preparation, training setup, training execution, deployment).
- **Job_Manifest**: A structured JSON object that describes the full pipeline configuration including GPU requirements, framework, dataset source, hyperparameters, and deployment target.
- **Training_Job**: A GPU-accelerated model training execution running on a provisioned RunPod pod.
- **Endpoint_Deployment**: The process of deploying a fine-tuned model as an inference endpoint accessible via HTTP.
- **Studio_Status_Event**: A structured event emitted by the Pipeline Orchestrator to report progress of each pipeline stage to the frontend.
- **GPU_Spec**: A configuration object specifying the GPU type, count, VRAM, and container image required for a given workload.

## Requirements

### Requirement 1: Studio Mode Activation and Task Parsing

**User Story:** As a developer, I want to describe an AI workload in natural language, so that the system can parse my intent and initiate the appropriate Studio Mode pipeline.

#### Acceptance Criteria

1. WHEN a user submits a message describing an AI workload (e.g., "Fine-tune Llama-3 on our internal documentation"), THE Pipeline_Orchestrator SHALL parse the message and produce a Job_Manifest containing the identified task type, base model, data source, and suggested GPU_Spec.
2. WHEN the Pipeline_Orchestrator cannot determine a required field from the user message, THE Pipeline_Orchestrator SHALL prompt the user for the missing information before proceeding.
3. WHEN a Job_Manifest is produced, THE Pipeline_Orchestrator SHALL present a summary of the planned pipeline stages to the user for confirmation before provisioning infrastructure.
4. IF the user rejects or modifies the proposed plan, THEN THE Pipeline_Orchestrator SHALL update the Job_Manifest accordingly and re-present the summary.

### Requirement 2: GPU Infrastructure Provisioning

**User Story:** As a developer, I want the system to automatically provision the right GPU infrastructure, so that I do not have to manually configure cloud compute.

#### Acceptance Criteria

1. WHEN a Job_Manifest is confirmed by the user, THE RunPod_Runtime SHALL provision a RunPod pod matching the GPU_Spec (GPU type, count, container image, disk size).
2. WHILE a pod is being provisioned, THE Studio_Session SHALL emit Studio_Status_Events reporting provisioning progress to the frontend.
3. IF pod provisioning fails (e.g., GPU type unavailable, API error), THEN THE RunPod_Runtime SHALL retry with an alternative GPU type from a ranked fallback list, and report the fallback to the user.
4. WHEN a pod reaches RUNNING status, THE RunPod_Runtime SHALL verify GPU availability by executing a GPU detection command and report the detected hardware in a Studio_Status_Event.
5. IF GPU verification fails on a running pod, THEN THE RunPod_Runtime SHALL terminate the pod and raise an error with a descriptive message.

### Requirement 3: Data Preparation Pipeline

**User Story:** As a developer, I want the system to automatically gather and prepare my training data, so that I can focus on the model rather than data engineering.

#### Acceptance Criteria

1. WHEN the Job_Manifest specifies a documentation URL or repository as the data source, THE Pipeline_Orchestrator SHALL generate and execute a data scraping script on the provisioned pod.
2. WHEN raw data is collected, THE Pipeline_Orchestrator SHALL generate and execute a data processing script that converts the raw data into the training format required by the specified framework (e.g., Axolotl JSONL, Unsloth format).
3. WHEN data processing completes, THE Pipeline_Orchestrator SHALL validate the output dataset by checking file existence, non-zero size, and correct format structure, and report the dataset statistics (row count, file size) in a Studio_Status_Event.
4. IF data scraping or processing fails, THEN THE Pipeline_Orchestrator SHALL report the error with the failing command output and suggest corrective actions to the user.

### Requirement 4: Training Job Execution

**User Story:** As a developer, I want the system to configure and run model training automatically, so that I get a fine-tuned model without writing training scripts manually.

#### Acceptance Criteria

1. WHEN the data preparation stage completes successfully, THE Pipeline_Orchestrator SHALL generate a training configuration file appropriate for the selected framework (e.g., Axolotl YAML config, Unsloth script) and upload it to the pod.
2. WHEN a training configuration is uploaded, THE Pipeline_Orchestrator SHALL execute the training command on the pod.
3. WHILE a Training_Job is running, THE Studio_Session SHALL poll the pod at a configurable interval and emit Studio_Status_Events containing training metrics (loss, epoch, step) parsed from the training output.
4. WHEN a Training_Job completes successfully, THE Pipeline_Orchestrator SHALL verify the output model artifacts exist on the pod and report completion with artifact paths in a Studio_Status_Event.
5. IF a Training_Job fails, THEN THE Pipeline_Orchestrator SHALL capture the error output, report it to the user, and suggest remediation steps (e.g., reduce batch size, switch GPU type).

### Requirement 5: Model Deployment as Endpoint

**User Story:** As a developer, I want the fine-tuned model deployed as an accessible endpoint, so that I can immediately test and use the model.

#### Acceptance Criteria

1. WHEN a Training_Job completes and the user confirms deployment, THE Pipeline_Orchestrator SHALL deploy the fine-tuned model as a RunPod serverless endpoint or a persistent pod endpoint.
2. WHEN an endpoint is deployed, THE Pipeline_Orchestrator SHALL generate and present a curl command and an API usage example to the user.
3. WHEN an endpoint is deployed, THE Pipeline_Orchestrator SHALL verify endpoint health by sending a test inference request and reporting the result in a Studio_Status_Event.
4. IF endpoint deployment fails, THEN THE Pipeline_Orchestrator SHALL report the error and provide the user with manual deployment instructions as a fallback.

### Requirement 6: Studio Mode Frontend Integration

**User Story:** As a developer, I want to see real-time progress of my AI workload in the OpenHands UI, so that I can monitor each stage of the pipeline.

#### Acceptance Criteria

1. WHEN a Studio_Session is active, THE Frontend SHALL display a pipeline progress panel showing each Pipeline_Stage with its current status (pending, running, completed, failed).
2. WHEN a Studio_Status_Event is received, THE Frontend SHALL update the corresponding Pipeline_Stage status and display the event details (metrics, logs, errors).
3. WHEN a Training_Job is running, THE Frontend SHALL display a live training metrics view showing loss, epoch, and step values as they are reported.
4. WHEN all pipeline stages complete, THE Frontend SHALL display a summary panel with the endpoint URL, curl command, and cost estimate for the session.
5. WHEN a Pipeline_Stage fails, THE Frontend SHALL highlight the failed stage in the progress panel and display the error message with any suggested remediation.

### Requirement 7: Session Cost Tracking and Resource Cleanup

**User Story:** As a developer, I want to track costs and ensure resources are cleaned up, so that I do not incur unexpected charges.

#### Acceptance Criteria

1. WHILE a Studio_Session is active, THE Pipeline_Orchestrator SHALL track elapsed GPU time and compute an estimated cost based on the GPU type hourly rate.
2. WHEN a Studio_Session completes or is cancelled by the user, THE RunPod_Runtime SHALL terminate the provisioned pod within 60 seconds.
3. IF a Studio_Session is idle (no pipeline activity) for longer than a configurable timeout (default 30 minutes), THEN THE RunPod_Runtime SHALL stop the pod and notify the user.
4. WHEN a pod is terminated, THE Pipeline_Orchestrator SHALL emit a final Studio_Status_Event containing the total session duration and estimated cost.

### Requirement 8: Job Manifest Serialization

**User Story:** As a developer, I want my pipeline configuration to be saved and reproducible, so that I can re-run or share my AI workflows.

#### Acceptance Criteria

1. THE Pipeline_Orchestrator SHALL serialize each Job_Manifest to JSON format.
2. THE Pipeline_Orchestrator SHALL deserialize a JSON string back into a valid Job_Manifest object.
3. FOR ALL valid Job_Manifest objects, serializing then deserializing SHALL produce an equivalent Job_Manifest (round-trip property).
4. WHEN a Studio_Session completes, THE Pipeline_Orchestrator SHALL persist the Job_Manifest to the conversation storage.
