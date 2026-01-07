# Architecture Principles for MCP Security

## Overview

Securing AI agents that interact with production systems requires architectural decisions that differ from traditional application security.

This document outlines design principles derived from production deployments.

---

## Core Principles

### 1. Layered Defense

Combine multiple security controls operating at different abstraction levels.

**Deterministic Layer:**
- Fast pattern matching
- Schema validation
- Rate limiting
- Path sanitization

**Semantic Layer:**
- Intent analysis
- Context evaluation
- Behavioral scoring

**Rationale:** Single-layer approaches miss either speed or accuracy. Layering provides both.

---

### 2. Fail-Closed by Default

When uncertainty exists, default to blocking rather than allowing.

**Examples:**
- Unrecognized tool → Block
- Ambiguous intent → Escalate
- Analysis timeout → Block
- Service unavailable → Block

**Rationale:** In security contexts, false negatives are more costly than false positives.

---

### 3. Explicit Human Approval for Destructive Operations

Certain operations should never be fully automated, regardless of confidence scores.

**Examples:**
- Data deletion
- Bulk modifications
- Privilege changes
- Financial transactions

**Rationale:** AI agents lack understanding of business context and consequences.

---

### 4. Comprehensive Observability

Every security decision must be:
- Logged with full context
- Auditable
- Traceable to specific rules or models
- Reviewable by humans

**Rationale:** Post-incident analysis requires complete decision history.

---

### 5. Principle of Least Privilege

Tools should have minimal necessary permissions.

**Implementation considerations:**
- Tool-level access control
- Per-session credential scoping
- Time-limited permissions
- Read-only defaults

**Rationale:** Limits blast radius of successful attacks.

---

### 6. Defense in Depth

No single control should be relied upon exclusively.

**Example layers:**
- Input validation
- Intent analysis
- Tool access control
- Network egress filtering
- Output sanitization

**Rationale:** Attackers will find ways around individual controls.

---

### 7. Separation of Analysis from Enforcement

Security analysis should be decoupled from enforcement mechanisms.

**Benefits:**
- Independent testing of analysis logic
- Faster iteration on detection
- Clearer audit trail
- Easier compliance demonstration

**Rationale:** Mixing analysis and enforcement creates complexity and reduces reliability.

---

## Reference Architecture

QYRA Labs' commercial implementation uses a dual-layer architecture validated in production environments.

### Layer 1: Deterministic Controls

Fast, rule-based checks that catch known attack patterns:

- **Pattern Matching** - Regex-based detection of injection attempts
- **Schema Validation** - JSON Schema enforcement
- **Rate Limiting** - Sliding window algorithm
- **Path Validation** - Traversal prevention
- **Tool Whitelisting** - Allowed tool enforcement

**Decision:** ALLOW or BLOCK

**Performance:** Sub-millisecond latency typical

**Rationale:** Deterministic checks provide fast, reliable rejection of obvious attacks.

---

### Layer 2: Semantic Analysis

Probabilistic analysis for novel variants and edge cases.

#### Vector-Based Similarity

Attack patterns are embedded in a vector space. Incoming requests are compared against this database to detect semantically similar threats.

- **Pattern database:** Thousands of documented attacks
- **Coverage:** Majority AI-specific (prompt injection, jailbreaks), remainder traditional (SQLi, command injection)
- **Performance:** Low single-digit millisecond latency

**How it works:**
1. Request is embedded into vector space
2. Similarity search against known attack patterns
3. Returns similarity score and matched patterns

#### LLM-Based Intent Classification

When vector similarity is inconclusive, LLM analysis provides additional signal.

- **Use case:** Ambiguous requests, novel attack variants
- **Deployment:** Multiple provider support (local/cloud)
- **Privacy:** Local deployment option for regulated industries
- **Performance:** Tens of milliseconds when triggered

**How it works:**
1. LLM analyzes request intent
2. Classifies as benign or malicious
3. Returns confidence score and reasoning

#### Risk Scoring

Layer 2 outputs a risk level based on combined signals:

- **LOW** - Similarity score below threshold, no concerning patterns
- **MEDIUM** - Moderate similarity or uncertain intent
- **HIGH** - Strong similarity to known attacks or clear malicious intent

---

### Decision Matrix

Combines Layer 1 and Layer 2 results to make final decision:

| Layer 1 | Layer 2 Risk | Destructive | Final Decision |
|---------|--------------|-------------|----------------|
| BLOCK   | any          | any         | **BLOCK** |
| ALLOW   | LOW          | No          | **ALLOW** |
| ALLOW   | MEDIUM       | No          | **MONITOR** |
| ALLOW   | HIGH         | No          | **BLOCK** |
| ALLOW   | any          | Yes         | **HUMAN_APPROVAL** |

**Key principle:** Layer 1 BLOCK is authoritative. Layer 2 provides advisory signal.

---

### Security Modules

The system includes specialized modules for specific threat categories:

1. **Session Management**
   - IP/User-Agent binding
   - Session hijacking detection
   - Timeout enforcement

2. **Network Policy**
   - Egress/ingress filtering
   - Domain whitelisting/blacklisting
   - Data exfiltration prevention

3. **Anomaly Detection**
   - Rate spike detection
   - Behavioral analysis
   - Loop detection

4. **Tool Integrity**
   - Rug pull detection via hashing
   - Tool definition verification
   - Change detection

5. **Typosquatting Detection**
   - Name similarity checking
   - Homoglyph detection
   - Tool impersonation prevention

6. **Destructive Operation Detection**
   - DELETE/DROP/TRUNCATE identification
   - Bulk operation flagging
   - Requires human approval

7. **Supply Chain Verification**
   - Checksum validation
   - Dependency integrity
   - Version tracking

8. **PII Sanitization**
   - Credential detection (API keys, tokens)
   - Personal information redaction
   - Compliance support (GDPR, HIPAA)

---

## Validation Approach

The architecture has been validated against:
- Large-scale attack datasets
- Legitimate production traffic samples
- Edge case scenarios

**Key findings:**
- Detection rates exceed industry targets
- False positive rates remain within acceptable thresholds
- Latency suitable for production deployment

Detailed benchmarks available on request.

---

## Design Considerations

### Performance vs. Security Trade-offs

Security controls add latency. Design must balance:
- Acceptable response time
- Detection accuracy
- False positive rate
- Operational cost

**Approach:** Use fast deterministic checks first, expensive semantic analysis selectively.

---

### Privacy and Data Residency

Analysis of requests may require processing sensitive data.

**Considerations:**
- Local vs. cloud-based analysis
- Data retention policies
- Compliance requirements (GDPR, HIPAA, etc.)
- Audit log encryption

**Approach:** Support both local and cloud deployment models. Local LLM option available for regulated industries.

---

### Evolving Threat Landscape

Attack patterns change rapidly.

**Requirements:**
- Update mechanisms for detection rules
- Versioned pattern databases
- A/B testing for new controls
- Rollback capability

**Approach:** Treat security logic as continuously deployed software.

---

### Multi-Tenancy

Production systems often serve multiple organizations.

**Requirements:**
- Per-tenant policy customization
- Isolated rate limiting
- Separate audit trails
- Policy inheritance

**Approach:** Tenant-aware architecture from foundation.

---

## Anti-Patterns to Avoid

### Over-reliance on Prompt Engineering

**Problem:** "Just tell the AI not to do bad things"

**Reality:** Prompt injection bypasses instructions.

---

### Security as an Afterthought

**Problem:** Adding security controls post-deployment

**Reality:** Architectural decisions constrain security capabilities.

---

### Assuming Intent Equals Action

**Problem:** "The AI meant well"

**Reality:** Intent analysis is probabilistic. Enforce controls regardless.

---

### Treating AI as Deterministic

**Problem:** Expecting consistent behavior

**Reality:** Non-determinism requires statistical validation, not unit tests.

---

### Single-Layer Defense

**Problem:** "Our LLM filter catches everything"

**Reality:** Adversarial examples will bypass single-layer defenses.

---

## Implementation Notes

This document describes principles and architectural patterns, not specific implementations.

Actual systems will vary based on:
- Deployment environment
- Risk tolerance
- Regulatory requirements
- Operational constraints
- Scale requirements

The patterns described here represent production-validated approaches, but are not prescriptive.

---

## References

- Model Context Protocol (MCP) specification
- OWASP Top 10 for LLM Applications
- NIST AI Risk Management Framework
- MITRE ATLAS (Adversarial Threat Landscape for AI Systems)

---

## Contact

Questions or feedback: contact@qyralabs.com
