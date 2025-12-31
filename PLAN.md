# ERC-8004 Integration for YudaiV3 Coding Agents

Let me map out the simple integration points between your coding agent workflow and the three ERC-8004 registries.

---

## Agent Lifecycle with ERC-8004

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AGENT LIFECYCLE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │   CREATE     │    │   EXECUTE    │    │    EARN      │              │
│  │   AGENT      │───▶│    TASK      │───▶│   PAYMENT    │              │
│  └──────────────┘    └──────────────┘    └──────────────┘              │
│         │                   │                   │                       │
│         ▼                   ▼                   ▼                       │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │  IDENTITY    │    │  VALIDATION  │    │  REPUTATION  │              │
│  │  REGISTRY    │    │  REGISTRY    │    │  REGISTRY    │              │
│  └──────────────┘    └──────────────┘    └──────────────┘              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. IdentityRegistry — Agent Creation & Discovery

### When to Use

- User connects repos and creates/trains an agent
- Agent NFT minted as portable, verifiable identity

### What to Store

**On-chain metadata (searchable):**
```
agentId: 42
owner: 0xUserWallet
metadata:
  - "languages": "TypeScript,Python,Solidity"
  - "repoCount": "3"
  - "baseModel": "claude-sonnet"
  - "x402Endpoint": "https://api.yudai.app/agent/42/pay"
```

**Token URI (IPFS) - full profile:**
```json
{
  "name": "pranay5255-coding-agent",
  "description": "Multi-repo coding agent",
  "repos": [
    "pranay5255/YudaiV3",
    "pranay5255/erc-8004-contracts"
  ],
  "capabilities": ["issue-solving", "pr-creation", "bug-fixing"],
  "pricing": {
    "issueResolution": "10 USDC",
    "prCreation": "15 USDC"
  },
  "endpoints": {
    "x402": "https://api.yudai.app/agent/42/pay",
    "execute": "https://api.yudai.app/agent/42/task"
  }
}
```

### Integration Code

```typescript
// After user trains agent (30/30 milestone)
async function registerAgent(userId: string, repos: string[]) {
  const profile = buildAgentProfile(userId, repos);
  const tokenURI = await uploadToIPFS(profile);

  const agentId = await identityRegistry.register(
    tokenURI,
    [
      { key: "languages", value: toBytes(detectLanguages(repos)) },
      { key: "repoCount", value: toBytes(repos.length.toString()) },
      { key: "x402Endpoint", value: toBytes(`/agent/${userId}/pay`) }
    ]
  );

  await db.agents.create({ userId, agentId, repos });
  return agentId;
}
```

### Discovery Use Cases

```typescript
// Find agents that work with TypeScript
const agents = await findAgentsByLanguage("TypeScript");

// Get agent's connected repos
const profile = await fetchIPFS(identityRegistry.tokenURI(agentId));
console.log(profile.repos);
```

---

## 2. ValidationRegistry — PR Quality Gates

### When to Use

- Agent creates PR → request validation
- CI completes → submit validation response
- Check validation score before merge/payment

### Validator Types

| Validator        | Address    | What it validates          |
|------------------|------------|----------------------------|
| CI/CD Bot        | 0xCI...    | Tests pass, build succeeds |
| Security Scanner | 0xSEC...   | No vulnerabilities         |
| Human Reviewer   | 0xHUMAN... | Code quality, logic        |

### Integration Flow

```
Agent creates PR
       │
       ▼
┌─────────────────┐
│ validationRequest│ ──────▶ Request sent to validator
└─────────────────┘
       │
       │ (CI runs, security scans, human reviews)
       ▼
┌──────────────────┐
│ validationResponse│ ──────▶ Score: 0-100
└──────────────────┘
       │
       ▼
┌─────────────────┐
│ Check threshold │ ──────▶ Score >= 80? → Auto-merge + pay
└─────────────────┘
```

### Integration Code

```typescript
// 1. When agent creates PR
async function onPRCreated(agentId: number, prUrl: string) {
  const prHash = keccak256(toBytes(prUrl));

  // Request CI validation
  await validationRegistry.validationRequest(
    CI_VALIDATOR,
    agentId,
    prUrl,
    prHash
  );
}

// 2. When CI completes (webhook)
async function onCIComplete(prUrl: string, passed: boolean, coverage: number) {
  const prHash = keccak256(toBytes(prUrl));
  const task = await db.tasks.findByPR(prUrl);

  // Calculate score: tests (80%) + coverage (20%)
  let score = passed ? 80 : 0;
  if (coverage >= 80) score += 20;
  else if (coverage >= 60) score += 10;

  await validationRegistry.validationResponse(
    prHash,
    score,
    `https://github.com/.../actions/runs/123`,
    keccak256(toBytes(JSON.stringify({ passed, coverage }))),
    passed ? "ci-passed" : "ci-failed"
  );

  // Check if ready to merge
  if (score >= 80) {
    await github.mergePR(prUrl);
    await processPayment(task.id);
  }
}

// 3. Query validation status
async function canMergePR(agentId: number, prHash: bytes32) {
  const status = await validationRegistry.getValidationStatus(prHash);
  return status.response >= 80;
}
```

### Multi-Validator Example

```typescript
// For high-value tasks, require multiple validators
async function requestFullValidation(agentId: number, prUrl: string) {
  const prHash = keccak256(toBytes(prUrl));

  // Request from 3 validators in parallel
  await Promise.all([
    validationRegistry.validationRequest(CI_VALIDATOR, agentId, prUrl, prHash),
    validationRegistry.validationRequest(SECURITY_VALIDATOR, agentId, prUrl, prHash),
    validationRegistry.validationRequest(HUMAN_REVIEWER, agentId, prUrl, prHash)
  ]);
}

// Check aggregated score
async function getAggregatedValidation(agentId: number) {
  const { count, avgScore } = await validationRegistry.getSummary(
    agentId,
    [CI_VALIDATOR, SECURITY_VALIDATOR, HUMAN_REVIEWER],
    "" // all tags
  );

  return { count, avgScore };
}
```

---

## 3. ReputationRegistry — Track Agent Performance

### When to Use

- PR merged successfully → positive feedback
- PR rejected/reverted → negative feedback
- Issue closed without resolution → negative feedback

### Scoring Model

| Event                | Base Score | Modifiers                                     |
|----------------------|------------|-----------------------------------------------|
| PR merged            | 85         | +10 if no review comments, -10 if >5 comments |
| PR rejected          | 30         | +10 if quick iteration, -20 if abandoned      |
| Tests failed         | 20         | —                                             |
| Security issue found | 0          | —                                             |

**Tags for filtering:**
- tag1: pr-merged, pr-rejected, issue-closed, reverted
- tag2: bugfix, feature, refactor, security, docs

### Integration Code

```typescript
// When PR is merged
async function onPRMerged(prUrl: string) {
  const task = await db.tasks.findByPR(prUrl);
  const pr = await github.getPR(prUrl);

  // Calculate score
  let score = 85;
  if (pr.reviewComments.length === 0) score += 10;
  if (pr.reviewComments.length > 5) score -= 10;
  if (pr.changesRequested > 0) score -= 15;
  score = Math.max(0, Math.min(100, score));

  // Get pre-authorization from repo owner
  const feedbackAuth = await signFeedbackAuth(
    repoOwner.privateKey,
    task.agentId,
    score
  );

  await reputationRegistry.giveFeedback(
    task.agentId,
    score,
    "pr-merged",
    pr.labels[0] || "general",
    prUrl,
    keccak256(toBytes(prUrl)),
    feedbackAuth
  );
}

// When PR is rejected
async function onPRRejected(prUrl: string, reason: string) {
  const task = await db.tasks.findByPR(prUrl);

  const score = reason === "tests-failed" ? 20 : 30;

  const feedbackAuth = await signFeedbackAuth(...);

  await reputationRegistry.giveFeedback(
    task.agentId,
    score,
    "pr-rejected",
    reason,
    prUrl,
    keccak256(toBytes(prUrl)),
    feedbackAuth
  );
}
```

### Query Reputation

```typescript
// Get agent's overall reputation
async function getAgentReputation(agentId: number) {
  const { count, avgScore } = await reputationRegistry.getSummary(
    agentId,
    [], // all clients
    "pr-merged",
    ""  // all types
  );

  return { totalPRs: count, avgScore };
}

// Get reputation by task type
async function getReputationByType(agentId: number, type: string) {
  return await reputationRegistry.getSummary(
    agentId,
    [],
    "pr-merged",
    type // "bugfix", "feature", etc.
  );
}
```

### Reputation-Based Pricing

```typescript
async function calculateTaskPrice(agentId: number, basePrice: number) {
  const { count, avgScore } = await getAgentReputation(agentId);

  if (count < 10) return basePrice; // Not enough data

  // Premium for high reputation
  if (avgScore >= 90) return basePrice * 1.25;  // +25%
  if (avgScore >= 80) return basePrice * 1.10;  // +10%
  if (avgScore >= 70) return basePrice;         // Standard
  return basePrice * 0.85;                      // -15% discount
}
```

---

## Complete Flow Example

```typescript
// 1. USER CREATES AGENT
const agentId = await registerAgent("pranay5255", [
  "pranay5255/YudaiV3",
  "pranay5255/erc-8004-contracts"
]);
// → Agent NFT minted, metadata on-chain

// 2. USER ASSIGNS ISSUE TO AGENT
const task = await assignIssue(agentId, "https://github.com/repo/issues/42");
// → Agent analyzes issue, writes code

// 3. AGENT CREATES PR
const pr = await agent.createPR(task);
await onPRCreated(agentId, pr.url);
// → Validation requested from CI

// 4. CI RUNS & VALIDATES
await onCIComplete(pr.url, true, 85);
// → Validation response: score 100
// → Score >= 80, auto-merge triggered

// 5. PR MERGED → REPUTATION RECORDED
await onPRMerged(pr.url);
// → Feedback: score 95, tag "pr-merged/bugfix"

// 6. PAYMENT PROCESSED
const price = await calculateTaskPrice(agentId, 15); // $16.50 (10% bonus)
await x402.pay(agentId, price);
// → Agent earns USDC
```

---

## Summary: Integration Points

| Stage           | Registry   | Action                               |
|-----------------|------------|--------------------------------------|
| Agent created   | Identity   | register() with repos, capabilities  |
| Agent updated   | Identity   | setMetadata() for stats updates      |
| PR created      | Validation | validationRequest() to CI            |
| CI completes    | Validation | validationResponse() with score      |
| Check merge     | Validation | getValidationStatus() / getSummary() |
| PR merged       | Reputation | giveFeedback() positive              |
| PR rejected     | Reputation | giveFeedback() negative              |
| Calculate price | Reputation | getSummary() for avg score           |
| Find agents     | Identity   | getMetadata() + tokenURI()           |
