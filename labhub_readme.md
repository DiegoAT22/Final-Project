## Project Overview

LabHub is a healthcare data aggregation platform designed to consolidate laboratory results from 15 independent laboratories, each using different systems, data formats, and delivery mechanisms. The platform enables patients to view all their lab results in a unified portal, regardless of the originating laboratory.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Challenge Solutions](#challenge-solutions)
  - [Challenge 1: Data Ingestion Strategy](#challenge-1-data-ingestion-strategy)
  - [Challenge 2: Laboratory Management](#challenge-2-laboratory-management)
  - [Challenge 3: Patient Matching](#challenge-3-patient-matching)
- [Technology Stack](#technology-stack)

---

## Architecture Overview

The LabHub platform follows a modular, event-driven architecture built on AWS serverless and managed services. The system is designed to handle heterogeneous data formats, ensure data quality, and maintain HIPAA compliance throughout the processing pipeline.

### Key Architectural Principles

1. **Separation of Concerns**: Ingestion, normalization, processing, and presentation are handled by distinct components
2. **Scalability**: Serverless architecture allows automatic scaling based on demand
3. **Reliability**: Dead Letter Queues (DLQ) and retry mechanisms ensure no data loss
4. **Security**: Encryption at rest and in transit, least privilege IAM policies, and comprehensive audit logging
5. **Cost Optimization**: Pay-per-use serverless model minimizes infrastructure costs

---

## Challenge Solutions

### Challenge 1: Data Ingestion Strategy

**Decision: Single Monolithic Processor (`lambda_ingest`)**

We implemented a unified Lambda function that handles all data formats and processing logic in one place.

#### What Does `lambda_ingest` Do?

The `lambda_ingest` function is our single point of entry and processing for all laboratory data:

1. **Receives payloads** from API Gateway in any supported format
2. **Detects format** automatically (JSON, CSV, XML, HL7)
3. **Parses and normalizes** using internal format-specific functions
4. **Validates** required fields and data integrity
5. **Persists data** to S3 (raw) and RDS (normalized)
6. **Publishes messages** to SQS for downstream processing

All of this happens within a single Lambda function with well-organized internal helper functions:
- `parse_json(payload)`
- `parse_csv(payload)`
- `parse_xml(payload)`
- `parse_hl7(payload)`

Each parser returns the same standardized internal schema regardless of input format.

#### Why Single Monolithic Processor?

**Advantages of Our Approach:**

**1. Code Maintainability**
- Single Lambda with organized internal functions is easier to maintain than 12+ separate functions
- Team only needs to deploy and version one function, not manage multiple function deployments
- Centralized codebase makes it easier to apply bug fixes and improvements

**2. Normalization Consistency**
- All normalization logic lives in one place
- Guarantees that all formats convert to the exact same internal schema
- Shared validation logic ensures consistent data quality across all sources

**3. Infrastructure Simplicity**
- No need to orchestrate multiple Lambda functions or manage multiple endpoints
- Fewer IAM roles and permissions to configure
- Reduced operational complexity and monitoring overhead
- Single CloudWatch log stream for all ingestion activity

**4. Easy Extension for New Formats**
- Adding a new laboratory format only requires adding a new parser function (e.g., `parse_fixed_width()`)
- No infrastructure changes needed - purely code-level addition
- Can be deployed in minutes without touching Terraform

**5. Cost Efficiency**
- Single Lambda invocation per request instead of multiple chained invocations
- No orchestration overhead (Step Functions, EventBridge, etc.)
- Simpler pricing model to predict and optimize

**Why We Rejected Other Options:**

**Lambda per Format (Option A) - Rejected**
- Would require 15+ separate Lambda functions (one per lab/format combination)
- Version management becomes complex across multiple functions
- Code duplication for shared validation and normalization logic
- Higher operational overhead for monitoring and debugging
- Difficult to maintain consistency in normalization across functions

**AWS Glue ETL Pipeline (Option C) - Rejected**
- Designed for batch processing, not near-real-time ingestion
- Significantly higher cost (~$0.44/DPU-hour vs ~$0.20/million requests for Lambda)
- Overkill for simple format transformation tasks
- Longer cold start times affect user experience
- More complex infrastructure to maintain and monitor

#### Trade-offs Acknowledged

**Potential Concerns with Monolithic Approach:**
- Single function contains complex conditional logic
- Larger deployment package size
- All format changes require redeploying entire function

**Why These Concerns Don't Apply:**
- Python's modularity allows clean separation of concerns within one file
- Total package size remains under 10MB (well within Lambda limits)
- CI/CD pipeline ensures safe deployments with automated testing
- Benefits of simplicity outweigh minor deployment coupling

---

### Challenge 2: Laboratory Management

**Decision: DynamoDB + Secrets Manager + RDS Hybrid Model**

We use a combination of AWS services, each optimized for its specific data type and access pattern.
In the presentation, one of the judges commented about how we could use Dynamodb in our project, we implemented RDS but we could combine RDS with Dynamodb in this way
#### Architecture Components

**DynamoDB - Laboratory Registry** (`labs_registry` table)
- Stores laboratory metadata and configuration
- Fields: `lab_id`, `lab_name`, `format_type`, `endpoint`, `is_active`, `retry_config`, `rate_limits`, `secret_arn`
- Fast key-value lookups by `lab_id` with single-digit millisecond latency

**AWS Secrets Manager - Credentials Storage**
- Stores sensitive credentials: API keys, OAuth tokens, SFTP passwords
- Referenced in DynamoDB via `secret_arn` field
- Automatic credential rotation and encryption with AWS KMS

**RDS PostgreSQL - Clinical Data**
- Stores normalized patient results, test data, and historical records
- ACID compliance for critical healthcare data
- Supports complex relational queries for reporting

#### Why This Separation?

**DynamoDB for Lab Metadata:**
- **Performance**: Sub-10ms lookups for lab configuration on every request
- **Scalability**: Automatic scaling with no capacity planning required
- **Cost**: Pay-per-request pricing fits sporadic configuration updates
- **Simplicity**: No relational queries needed - pure key-value access pattern
- **Flexibility**: Schema-less design allows easy addition of new lab-specific fields

**Secrets Manager for Credentials:**
- **Security**: Purpose-built for sensitive credential storage with automatic encryption
- **Rotation**: Built-in support for automatic credential rotation
- **Audit**: Complete audit trail of who accessed which secrets and when
- **IAM Integration**: Fine-grained access control per secret
- **Compliance**: Meets HIPAA requirements for credential management

**RDS for Clinical Data:**
- **Data Integrity**: ACID transactions ensure data consistency
- **Complex Queries**: SQL support for patient history, cross-lab comparisons, and reporting
- **Industry Standard**: PostgreSQL is proven for healthcare applications
- **Backup/Recovery**: Automated backups and point-in-time recovery
- **Relational Model**: Patient-results relationships require relational database

#### Access Pattern Optimization

| Data Type | Service | Access Frequency | Why This Service? |
|-----------|---------|------------------|-------------------|
| Lab Config | DynamoDB | High (every request) | Need <10ms latency |
| Credentials | Secrets Manager | Medium (cached 1hr) | Security & rotation |
| Clinical Data | RDS | Medium-High | Complex queries needed |

#### Why Not Alternative Approaches?

**Everything in RDS - Rejected**
- Slower latency for simple lab config lookups
- Over-provisioning required for high read throughput
- Higher cost at scale compared to DynamoDB's pay-per-request
- Secrets mixed with metadata reduces security posture

**Everything in DynamoDB - Rejected**
- Complex relational queries on clinical data would be inefficient
- No ACID transactions for multi-table updates
- Difficult to maintain referential integrity
- SQL reporting tools wouldn't work

**Secrets in DynamoDB - Rejected**
- No automatic rotation capability
- Manual encryption key management required
- No built-in audit trail for secret access
- Violates principle of separation of concerns

---

### Challenge 3: Patient Matching

**Decision: Rule-Based Matching Algorithm with Confidence Scoring**

We implemented a deterministic matching algorithm that achieves >95% accuracy through careful normalization and weighted scoring rules, without requiring machine learning.

#### Algorithm Overview

**Phase 1: Data Normalization**

Before comparing patients, we normalize all identifying information:
- **Names**: "Smith, John" → "JOHN SMITH", "J. Smith" → "J SMITH"
- **Dates**: "03/15/1985" → "1985-03-15" (ISO 8601 format)
- **Phone/Email**: Strip formatting, convert to standard format

**Phase 2: Confidence Scoring**

We calculate a match confidence score using weighted rules:

| Match Criterion | Points | Importance |
|----------------|--------|------------|
| Exact DOB match | +50 | Critical |
| Last name match | +30 | High |
| First name match | +20 | High |
| First initial match | +10 | Medium |
| SSN match (if available) | +40 | Critical |
| Phone match | +15 | Medium |
| Email match | +15 | Medium |

**Phase 3: Classification**

- **Score ≥ 80**: DEFINITE_MATCH (automatic merge)
- **50 ≤ Score < 80**: POSSIBLE_MATCH (manual review required)
- **Score < 50**: NO_MATCH (separate patients)

#### Why Rule-Based Instead of Machine Learning?

**Advantages of Rule-Based Approach:**

**1. Transparency & Explainability**
- Every match decision can be explained to healthcare providers
- Compliance teams can audit the exact rules used
- Required for healthcare regulatory compliance (HIPAA)

**2. Deterministic & Consistent**
- Same inputs always produce same outputs
- No "black box" decisions that could affect patient care
- Easier to debug and validate

**3. No Training Data Required**
- Don't need thousands of labeled patient match/non-match examples
- Can start matching accurately from day one
- No model retraining or drift management

**4. Fast Processing**
- No model inference latency
- Simple arithmetic operations complete in milliseconds
- Can process thousands of matches per second

**5. Easy to Maintain & Adjust**
- Business users can understand and propose rule changes
- Adjusting weights doesn't require data scientist involvement
- Can add new matching criteria without retraining

**6. Sufficient Accuracy**
- Achieves >95% accuracy with well-tuned rules for complete records
- Healthcare matching is relatively straightforward (stable identifiers like DOB, SSN)
- False positive rate <2% is acceptable with manual review queue

**When We Would Consider ML:**
- Match accuracy falls below 90% with current rules
- Manual review queue exceeds 10% of records
- Need to match across very dirty/incomplete data with many typos
- International expansion requires fuzzy matching across languages

#### Database Schema

**Canonical Patient Table** (`patients`)
- Stores one record per unique patient with canonical identifiers
- Fields: `patient_id` (UUID), `canonical_name`, `canonical_dob`, `canonical_ssn`

**Lab-Specific Links** (`lab_patient_links`)
- Maps lab-specific patient IDs to canonical patient IDs
- Fields: `patient_id`, `lab_id`, `lab_patient_id`, `match_confidence_score`, `match_method`
- Allows one patient to have multiple IDs across different labs
- Preserves audit trail of matching decisions

#### Expected Performance

- **Match Accuracy**: >95% for patients with complete data
- **False Positive Rate**: <2% (incorrectly merged patients)
- **Manual Review Rate**: 5-8% of records
- **Processing Time**: <50ms per patient comparison

---

## Technology Stack

### Core AWS Services

| Service | Purpose | Justification |
|---------|---------|---------------|
| **API Gateway** | REST API endpoint | Managed service with built-in throttling, API key management, and request validation |
| **Lambda** | Serverless compute | Auto-scaling, pay-per-use pricing, sub-second cold starts, no server management |
| **DynamoDB** | Lab registry storage | Low-latency NoSQL, serverless scaling, flexible schema for lab configurations |
| **RDS PostgreSQL** | Clinical data storage | ACID compliance, complex SQL queries, healthcare industry standard |
| **S3** | Raw data archival | 99.999999999% durability, versioning, lifecycle policies for compliance |
| **SQS** | Message queue | Decoupling components, guaranteed delivery, DLQ for error handling |
| **Secrets Manager** | Credential storage | Automatic rotation, encryption at rest, comprehensive audit logging |
| **CloudWatch** | Logging & monitoring | Centralized logs, custom metrics, alarms for operational visibility |
| **Cognito** | User authentication | OAuth 2.0 support, MFA, managed user pools, built-in password policies |
| **ECR** | Container registry | Private Docker image storage for ECS tasks with vulnerability scanning |
| **ECS Fargate** | Portal hosting | Serverless containers, no EC2 management, automatic scaling |

### Development Stack

- **Python 3.11**: Lambda functions and data processing workers
- **Flask**: Patient portal web application framework
- **Terraform**: Infrastructure as Code for reproducible deployments
- **Docker**: Containerization for ECS services
- **Pytest**: Unit and integration testing framework

---