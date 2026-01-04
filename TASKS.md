# Backend Architecture Tasks and Integration Tests

This file captures the backend-first architecture plan and a detailed test matrix for integration between on-chain ERC-8004 contracts and off-chain services. The focus is to validate the Identity, Reputation, and Validation registries with orchestration services before building UI.

## Architecture Plan (Backend-First)

### A. Core Services
1) Orchestrator API
- Owns task lifecycle: register agent -> request validation -> collect responses -> submit reputation.
- Emits and consumes on-chain events through the indexer.
- Generates request and response payloads and stores evidence files.

2) Event Indexer
- Subscribes to on-chain events: Registered, MetadataSet, NewFeedback, FeedbackRevoked, ResponseAppended, ValidationRequest, ValidationResponse.
- Mirrors events into a read model (DB) for API access and audit logs.

3) Validation Services
- CI validator: runs tests, lint, build; submits validationResponse.
- Security validator: runs SAST/dep scans; submits validationResponse.
- Human validator (optional): manual review pipeline.

4) Reputation Service
- Applies scoring rubric to task outcomes.
- Creates feedbackAuth signatures (EIP-191 or ERC-1271) on behalf of agent owner/operator.
- Submits giveFeedback, revokeFeedback, and appendResponse.

5) Evidence Store
- Stores request/response files for validation and feedback evidence (IPFS/S3).
- Produces requestHash/responseHash/filehash to commit to data on-chain.

### B. On-Chain Touchpoints
- IdentityRegistry
  - register(tokenURI, metadata)
  - setMetadata(agentId, key, value)
  - tokenURI(agentId), getMetadata(agentId, key)
  - Events: Registered, MetadataSet
- ReputationRegistry
  - giveFeedback(agentId, score, tag1, tag2, fileuri, filehash, feedbackAuth)
  - revokeFeedback(agentId, feedbackIndex)
  - appendResponse(agentId, clientAddress, feedbackIndex, responseUri, responseHash)
  - Read: getSummary, readFeedback, readAllFeedback, getClients, getLastIndex
  - Events: NewFeedback, FeedbackRevoked, ResponseAppended
- ValidationRegistry
  - validationRequest(validator, agentId, requestUri, requestHash)
  - validationResponse(requestHash, score, responseUri, responseHash, tag)
  - Read: getValidationStatus, getSummary, getAgentValidations, getValidatorRequests
  - Events: ValidationRequest, ValidationResponse

## Implementation Plan

### Phase 1: On-Chain Contract Wiring
- Verify contract ABIs and addresses in config.
- Create typed clients (viem/ethers) for each registry.
- Implement signing utilities for feedbackAuth and request/response hashing.

### Phase 2: Event Indexing and Read Models
- Build event indexer with idempotent consumption.
- Store normalized records: agents, metadata, validations, feedback, responses.
- Add reorg handling and block checkpoints.

### Phase 3: Orchestrator API
- Endpoints to create tasks, trigger validations, and finalize reputation.
- Generate request/response payloads and push to evidence store.
- Call validationRequest and giveFeedback on-chain.

### Phase 4: Validator Services
- CI validator: test/lint/build -> responseUri -> validationResponse.
- Security validator: SAST/dep scan -> responseUri -> validationResponse.
- Optional human review queue and response service.

### Phase 5: Reputation Service
- Define scoring rubric for PR/issue outcomes.
- Assemble feedback evidence payloads.
- Sign feedbackAuth, submit giveFeedback, and appendResponse as needed.

### Phase 6: Hardening
- Rate limits, retries, circuit breakers.
- Audit logs for each on-chain submission.
- Key management and signer isolation.

## Integration Test Plan (On-Chain <-> Off-Chain)

### 1) Identity Registry Integration
- 1.1 Registration Round-Trip
  - Create registration JSON in evidence store.
  - register(tokenURI, metadata) and verify Registered event.
  - Indexer ingests Registered + MetadataSet and stores DB records.
  - Fetch tokenURI + getMetadata and compare to stored values.
- 1.2 Metadata Update Flow
  - setMetadata for languages, repoCount, agentWallet.
  - Confirm MetadataSet event and DB update.
  - Ensure idempotent updates across replays.
- 1.3 Ownership / Operator Enforcement
  - Verify only owner/operator can update tokenURI and metadata.
  - Ensure unauthorized updates revert and indexer handles reverts.

### 2) Validation Registry Integration
- 2.1 Validation Request Creation
  - Orchestrator builds requestUri + requestHash and calls validationRequest.
  - Verify ValidationRequest event and DB row creation.
  - Ensure requestHash resolves to correct evidence file.
- 2.2 Validator Response Submission
  - Validator service calls validationResponse with score and responseUri/hash.
  - Verify ValidationResponse event, lastUpdate updates, and DB mirror.
- 2.3 Multi-Validator Aggregation
  - Issue requests to CI + Security validators.
  - Verify getSummary with validator filters and tags.
  - Confirm DB reflects per-validator response status.
- 2.4 Progressive Validation States
  - Submit soft then hard responses with different tags.
  - Confirm latest response reflects most recent tag and score.
- 2.5 Authorization Gate
  - Ensure only validatorAddress from request can respond.
  - Reject unauthorized response and confirm no DB change.

### 3) Reputation Registry Integration
- 3.1 FeedbackAuth Signing
  - Generate feedbackAuth for agentId, client, indexLimit, expiry, chainId.
  - Validate on-chain signature verification passes.
- 3.2 Give Feedback
  - Submit giveFeedback with fileuri/filehash evidence.
  - Verify NewFeedback event and DB record.
  - Read on-chain via readFeedback and compare to DB.
- 3.3 Feedback Indexing and Limits
  - Submit multiple feedbacks per client -> ensure index increments.
  - Validate indexLimit enforcement and expiry handling.
- 3.4 Revoke Feedback
  - Call revokeFeedback and ensure FeedbackRevoked event + DB update.
  - Ensure revoked feedback excluded from readAllFeedback (unless includeRevoked).
- 3.5 Append Response
  - Append dispute or clarification response with responseUri/hash.
  - Verify ResponseAppended event and DB linkage.

### 4) End-to-End Agent Lifecycle
- 4.1 Register -> Validate -> Reputation
  - Register agent with metadata and tokenURI.
  - Trigger validationRequest for CI.
  - Submit validationResponse with passing score.
  - Compute reputation score and submit giveFeedback.
  - Verify chain state and DB state match at each step.
- 4.2 Failure Paths
  - Validation fails -> reputation penalty.
  - Ensure feedbackAuth signature matches correct signer.
  - Ensure retry logic does not duplicate chain submissions.

### 5) Evidence Store and Hash Integrity
- 5.1 Hash Commitments
  - Store evidence file, compute hash, submit on-chain.
  - Re-fetch file and verify hash matches on-chain data.
- 5.2 Content Addressable URIs
  - If IPFS URI used, allow empty filehash (per spec).
  - Validate system handles both hashed and IPFS cases.

### 6) Indexer and Read Model Consistency
- 6.1 Event Replay
  - Replay blocks and confirm idempotent DB state.
- 6.2 Reorg Handling
  - Simulate reorg by rewinding blocks and reprocessing.
  - Confirm DB rolls back and replays correctly.
- 6.3 Partial Failure Recovery
  - Kill indexer mid-stream; restart and catch up from last checkpoint.

### 7) Security and Access Control
- 7.1 Key Isolation
  - Validator signing keys cannot submit feedback; feedback signer cannot validate.
- 7.2 Rate Limits and Abuse
  - Stress test repeated requests and ensure throttling in orchestrator.
- 7.3 Audit Trail
  - Ensure every on-chain write has a corresponding audit log entry.

## Suggested Execution Order
1) Phase 1 + Phase 2 (contract clients + indexer)
2) Identity Registry integration tests (Category 1)
3) Validation Registry integration tests (Category 2)
4) Reputation Registry integration tests (Category 3)
5) End-to-end lifecycle tests (Category 4)
6) Evidence and indexer hardening (Category 5-7)

## Deliverables Checklist
- Contract clients and signing utilities
- Indexer with DB schema
- Orchestrator API and job queue
- Validation services (CI + security)
- Reputation service and scoring rubric
- Integration test suite and logs

