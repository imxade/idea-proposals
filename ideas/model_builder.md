# Automated Model Builder — Product Requirements

## 1. Objective

Build an automated model-building platform where users define:

* Input schema
* Output schema
* Training examples
* Validation examples
* Feedback on generated outputs
* Quality, speed, memory, and compute constraints

The platform automatically determines the training strategy, trains candidate configurations, evaluates them, optimizes the strongest candidates, and produces a single portable **Model Pack**.

---

## 2. Core Workflow

```text
Define Input & Output
        ↓
Provide Examples
        ↓
Analyze Dataset
        ↓
Determine Training Problem
        ↓
Generate Candidate Configurations
        ↓
Run Initial Training Trials
        ↓
Eliminate Weak Candidates
        ↓
Optimize Promising Candidates
        ↓
Evaluate
        ↓
Select Best Candidate
        ↓
Create Model Pack
        ↓
Collect Feedback
        ↓
Retrain / Improve
```

---

## 3. Input and Output Definition

Users must be able to define typed input and output schemas.

Supported inputs may include:

* Text
* Images
* Documents
* Structured data
* Tabular data
* Audio
* Video
* Time-series
* Multiple combined inputs

Outputs may include:

* Text
* Structured JSON
* Categories
* Numerical values
* Multiple labels
* Embeddings
* Bounding boxes
* Other structured predictions

Schemas must support validation rules, required fields, ranges, enumerations, and structured/nested values.

---

## 4. Dataset Management

The platform must support:

* Manual example creation
* Dataset upload
* API-based example ingestion
* Input/output pair validation
* Duplicate detection
* Invalid-example detection
* Dataset statistics
* Automatic train/validation/test splitting
* Dataset versioning
* Tracking the source of each example
* Adding production feedback to future training datasets

---

## 5. Deterministic Problem Analysis

The platform must deterministically analyze:

* Input types
* Output types
* Dataset size
* Data distribution
* Sequence/image/document dimensions
* Available compute
* Required latency
* Required memory
* Required model size
* Quality target

The same dataset, configuration, constraints, platform version, and seed should produce the same training plan.

---

## 6. Automated Training Planner

The planner must generate a bounded set of viable training configurations.

Each configuration may specify:

* Architecture configuration
* Model size
* Training method
* Loss function
* Optimizer
* Learning rate
* Batch size
* Training duration
* Precision
* Regularization
* Context/input dimensions
* Adapter configuration
* Quantization strategy
* Other task-specific parameters

Planning decisions must be stored as part of the training run for reproducibility.

---

## 7. Progressive Candidate Search

Training should use progressive resource allocation.

### Stage 1 — Probe

Run inexpensive initial training trials to estimate candidate potential.

### Stage 2 — Elimination

Discard configurations that fail predefined quality or efficiency thresholds.

### Stage 3 — Optimization

Allocate additional compute to promising candidates and optimize their parameters.

### Stage 4 — Final Training

Train the strongest configurations using the selected full training procedure.

### Stage 5 — Final Evaluation

Evaluate final candidates using the complete validation and benchmark suite.

---

## 8. Evaluation

The evaluation framework must calculate task-appropriate quality metrics together with operational metrics.

At minimum, selection should be able to consider:

* Validation quality
* Accuracy or task-specific quality score
* Latency
* Throughput
* RAM usage
* VRAM usage
* Model size
* Training compute
* Inference compute

Users must be able to define their priorities and constraints.

Example:

```yaml
objective:
  quality: maximize

constraints:
  latency_ms: 50
  memory_gb: 4
  model_size_gb: 2
```

The final candidate must satisfy required constraints before selection.

---

## 9. Deterministic Selection

The platform must use an explicit scoring and constraint system to select the final candidate.

Selection must be reproducible from:

* Dataset version
* Evaluation dataset
* Candidate results
* User constraints
* Scoring configuration
* Platform version

The complete selection reasoning and measurements must be recorded.

---

## 10. User Modes

### Auto

The platform controls the complete training process.

The user provides examples and optional constraints.

### Guided

The user may specify requirements such as:

* Maximum latency
* Maximum memory
* Maximum model size
* Available hardware
* Training budget
* Minimum quality

The platform determines the training configuration.

### Expert

The user may override planner decisions and directly configure supported training parameters.

---

## 11. Feedback

Predictions must support feedback through the UI and API.

Feedback types should include:

* Correct
* Incorrect
* Corrected output
* Preferred output
* Rating
* Structured correction

Feedback must be stored with:

* Model version
* Original input
* Original prediction
* Corrected/preferred result
* Timestamp
* Feedback source

The platform must convert accepted feedback into appropriate future training and evaluation data.

---

## 12. Continuous Improvement

The platform must support iterative training cycles.

```text
Deployed Model
      ↓
Predictions
      ↓
Feedback
      ↓
Updated Dataset
      ↓
New Training Run
      ↓
Evaluation
      ↓
Version Comparison
      ↓
Promotion
```

The system must support:

* Manual retraining
* Policy-based retraining
* Comparison with the current production version
* Promotion
* Rollback
* Historical evaluation results

---

## 13. Training Reproducibility

Each training run must preserve:

* Dataset version
* Input/output schema
* Planner version
* Generated training plan
* Random seeds
* Training configuration
* Environment information
* Hardware information
* Checkpoints
* Evaluation results
* Final selection results

A completed model should be reproducible from its recorded run wherever deterministic execution is technically available.

---

## 14. Model Pack

The platform will expose **one model export format: Model Pack**.

A Model Pack is the canonical, full-fidelity representation of the trained model and everything necessary to interpret and use it.

Example structure:

```text
model.modelpack
│
├── manifest.json
│
├── schema/
│   ├── input.json
│   └── output.json
│
├── model/
│   ├── configuration
│   └── weights
│
├── preprocessing/
│
├── tokenizer/
│
├── metadata/
│   ├── training.json
│   ├── evaluation.json
│   └── provenance.json
│
└── checksums.json
```

The exact internal files may vary according to the trained model while maintaining a stable Model Pack specification.

---

## 15. Model Pack Manifest

Every Model Pack must contain a versioned manifest describing:

* Model Pack specification version
* Model identifier
* Model version
* Input schema
* Output schema
* Architecture definition
* Weight representation
* Precision
* Required preprocessing
* Required postprocessing
* Runtime requirements
* Training metadata references
* Integrity hashes

The manifest must be machine-readable and independently parseable.

---

## 16. Model Pack Requirements

A Model Pack must:

* Preserve the trained model without unnecessary information loss
* Preserve its architecture definition
* Preserve preprocessing and postprocessing requirements
* Preserve tokenizer/configuration data where required
* Preserve input/output schemas
* Preserve numerical precision of the canonical trained artifact
* Support very large and sharded weight files
* Support deterministic integrity verification
* Be versioned
* Be self-contained
* Be suitable for long-term archival
* Be suitable for future tooling to consume independently of the training platform

---

## 17. Model Versioning

Every completed training result must receive a model version.

Each version must reference:

* Parent version where applicable
* Dataset version
* Feedback dataset version
* Training run
* Evaluation results
* Model Pack
* Deployment status

Versions must be immutable once finalized.

---

## 18. API

All primary platform functionality must be available programmatically.

The API must support:

```text
Create project/model
Define input/output schema
Upload examples
Add individual examples
Start training
Inspect training status
Retrieve candidate results
Retrieve evaluations
Request predictions
Submit feedback
Trigger retraining
Compare versions
Promote a version
Rollback a version
Export Model Pack
```

---

## 19. Automation and Agent Access

External automation and agents must be able to operate the complete model lifecycle through authenticated APIs.

Agents must be able to:

1. Submit data.
2. Request predictions.
3. Verify results externally.
4. Submit corrections.
5. Trigger training according to permissions.
6. Inspect evaluation results.
7. Compare versions.
8. Promote approved versions.

All automated actions must be auditable.

---

## 20. Observability

The platform must expose:

* Training progress
* Candidate status
* Resource utilization
* Training failures
* Evaluation metrics
* Model performance
* Inference latency
* Feedback volume
* Dataset growth
* Model-version history

Training decisions and automatic eliminations must be inspectable.
