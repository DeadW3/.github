# Contributing to DeadW3

Thank you for your interest in contributing to the DeadW3 Protocol. This document provides guidelines and standards for contributing to any repository within the DeadW3 organization.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Coding Standards](#coding-standards)
- [Pull Request Process](#pull-request-process)
- [Testing Requirements](#testing-requirements)
- [Documentation Standards](#documentation-standards)
- [Security Considerations](#security-considerations)

---

## Code of Conduct

All contributors must adhere to our [Code of Conduct](./CODE_OF_CONDUCT.md). By participating, you agree to uphold these community standards.

---

## Getting Started

### Prerequisites

Before contributing, ensure you have the following installed:

**All Repositories:**
- Git 2.30+
- Docker Desktop 24.0+ and Docker Compose
- Node.js 20 LTS (via nvm recommended)
- pnpm 8.0+

**Contract Development (deadw3-contracts):**
- Foundry (forge, cast, anvil)
- Solidity 0.8.20+

**Backend Development (deadw3-protocol):**
- PostgreSQL 15+ client tools
- Redis 7+

**Verification Service (deadw3-verifier):**
- Python 3.11+
- uv package manager
- ffmpeg and librosa dependencies

**Frontend Development (deadw3-explorer):**
- Vercel CLI (optional, for preview deployments)

### Finding Your First Issue

Look for issues labeled with `good-first-issue` or `help-wanted` across our repositories:
- [deadw3-contracts](https://github.com/DeadW3/deadw3-contracts/issues)
- [deadw3-protocol](https://github.com/DeadW3/deadw3-protocol/issues)
- [deadw3-verifier](https://github.com/DeadW3/deadw3-verifier/issues)
- [deadw3-explorer](https://github.com/DeadW3/deadw3-explorer/issues)
- [deadw3-docs](https://github.com/DeadW3/deadw3-docs/issues)

### Repository Setup

Each repository contains specific setup instructions in its README. General pattern:

```bash
# Clone the repository
git clone https://github.com/DeadW3/<repository-name>.git
cd <repository-name>

# Install dependencies
# For TypeScript/Node projects:
pnpm install

# For Python projects:
uv sync

# For Foundry projects:
forge install

# Copy environment template
cp .env.example .env
# Edit .env with your local configuration

# Start development environment
docker compose up -d
pnpm dev  # or equivalent command
```

---

## Development Workflow

### Branch Strategy

We follow a simplified Git Flow:

- `main` – Production-ready code, protected branch
- `develop` – Integration branch for features, protected branch
- `feature/*` – Feature development branches
- `fix/*` – Bug fix branches
- `docs/*` – Documentation-only changes
- `refactor/*` – Code refactoring without behavior changes

### Creating a Branch

```bash
# Ensure you're up to date
git checkout develop
git pull origin develop

# Create feature branch
git checkout -b feature/short-descriptive-name

# Or for bug fixes
git checkout -b fix/issue-123-brief-description
```

### Commit Message Format

We follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, no logic change)
- `refactor`: Code refactoring
- `perf`: Performance improvements
- `test`: Adding or updating tests
- `chore`: Build process or auxiliary tool changes
- `ci`: CI/CD configuration changes

**Example:**

```
feat(registry): add show deduplication check

Implement BLAKE3 hash comparison before registering new shows
to prevent duplicate entries. Includes database migration for
hash index and integration test coverage.

Closes #142
```

### Commit Best Practices

- Keep commits atomic – one logical change per commit
- Write clear, descriptive commit messages
- Reference issue numbers when applicable
- Squash commits before merging if they represent work-in-progress iterations

---

## Coding Standards

### TypeScript/JavaScript (deadw3-protocol, deadw3-explorer)

**Configuration:**
- TypeScript strict mode enabled
- ESLint with Airbnb TypeScript config
- Prettier for consistent formatting
- Husky pre-commit hooks for linting

**Style Guidelines:**

```typescript
// Use explicit return types for functions
export async function fetchShow(txId: string): Promise<Show | null> {
  // Implementation
}

// Prefer const over let
const showDate = new Date(show.dateYMD);

// Use descriptive variable names
const isVerified = show.status === ShowStatus.Verified;

// Handle errors explicitly
try {
  const result = await verifyShow(txId);
  return result;
} catch (error) {
  logger.error({ error, txId }, 'Show verification failed');
  throw new VerificationError('Failed to verify show', { cause: error });
}

// Use Zod for runtime validation
const ShowSubmissionSchema = z.object({
  arweaveTxId: z.string().regex(/^[a-zA-Z0-9_-]{43}$/),
  dateYMD: z.number().int().min(19650000).max(20501231),
  venue: z.string().min(1).max(200),
});
```

**Import Organization:**

```typescript
// 1. External dependencies
import { z } from 'zod';
import { prisma } from '@deadw3/database';

// 2. Internal packages
import { logger } from '@deadw3/config';
import type { Show, VerificationReport } from '@deadw3/types';

// 3. Relative imports
import { computeHash } from './utils/crypto';
import type { ShowMetadata } from './types';
```

**File Naming:**
- Use kebab-case: `show-registry.ts`
- Test files: `show-registry.test.ts`
- Type definition files: `show-registry.types.ts`

### Solidity (deadw3-contracts)

**Configuration:**
- Solidity 0.8.20 or later
- OpenZeppelin contracts for standard implementations
- Foundry for testing and deployment

**Style Guidelines:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

/// @title ShowRegistry
/// @notice Maintains registry of verified Grateful Dead show uploads
/// @dev Implements staking, verification, and reward distribution
contract ShowRegistry is AccessControl, ReentrancyGuard {
    /// @notice Emitted when a show is successfully registered
    /// @param txId Arweave transaction ID
    /// @param uploader Address that submitted the show
    /// @param dateYMD Show date in YYYYMMDD format
    event ShowRegistered(
        string indexed txId,
        address indexed uploader,
        uint32 dateYMD
    );

    /// @notice Maps Arweave TX IDs to Show structs
    mapping(string => Show) public shows;

    /// @notice Register a new show after AI verification
    /// @param txId Arweave transaction ID containing show data
    /// @param rootHash BLAKE3 root hash of show manifest
    /// @param dateYMD Show date in YYYYMMDD format
    /// @param aiScore Verification confidence score (0-100)
    /// @dev Requires VERIFIER_ROLE, validates uniqueness
    function registerShow(
        string calldata txId,
        bytes32 rootHash,
        uint32 dateYMD,
        uint8 aiScore
    ) external onlyRole(VERIFIER_ROLE) nonReentrant {
        require(shows[txId].uploader == address(0), "Show exists");
        require(aiScore >= MIN_CONFIDENCE, "Score too low");
        require(dateYMD >= 19650000 && dateYMD <= 20501231, "Invalid date");
        
        // Implementation follows checks-effects-interactions pattern
    }
}
```

**Contract Standards:**
- Use explicit visibility for all functions and state variables
- Follow checks-effects-interactions pattern to prevent reentrancy
- Emit events for all state changes
- Use custom errors over require strings for gas efficiency
- Include comprehensive NatSpec documentation
- Implement access control with OpenZeppelin AccessControl
- Use ReentrancyGuard for functions handling value transfers

**Testing Requirements:**
```solidity
// File: test/ShowRegistry.t.sol
contract ShowRegistryTest is Test {
    function testRegisterShow() public {
        // Arrange
        string memory txId = "abc123";
        bytes32 hash = keccak256("test");
        
        // Act
        vm.prank(verifier);
        registry.registerShow(txId, hash, 19770508, 95);
        
        // Assert
        (address uploader, , , uint8 status, ) = registry.shows(txId);
        assertEq(status, uint8(ShowStatus.Verified));
    }
    
    function testFuzz_RegisterShowDateRange(uint32 date) public {
        // Fuzz test date validation
    }
}
```

### Python (deadw3-verifier)

**Configuration:**
- Python 3.11+
- Ruff for linting
- Black for formatting
- mypy for type checking
- Pydantic for data validation

**Style Guidelines:**

```python
"""Show verification service.

This module implements AI-assisted verification of uploaded Grateful Dead
shows, including integrity checks, audio analysis, and policy compliance.
"""

from typing import Optional
from pathlib import Path

from pydantic import BaseModel, Field, validator
from fastapi import FastAPI, HTTPException
import blake3

from .audio import analyze_audio_quality
from .policy import check_compliance

class VerificationRequest(BaseModel):
    """Request to verify a show upload."""
    
    arweave_tx_id: str = Field(..., regex=r"^[a-zA-Z0-9_-]{43}$")
    callback_url: Optional[str] = None
    
    @validator("arweave_tx_id")
    def validate_tx_id(cls, v: str) -> str:
        """Ensure transaction ID is valid Arweave format."""
        if not v or len(v) != 43:
            raise ValueError("Invalid Arweave transaction ID")
        return v

class VerificationReport(BaseModel):
    """Complete verification report for a show."""
    
    version: str = "1.0"
    arweave_tx_id: str
    integrity_score: int = Field(..., ge=0, le=100)
    audio_quality_score: int = Field(..., ge=0, le=100)
    policy_risk: int = Field(..., ge=0, le=100)
    overall_score: int = Field(..., ge=0, le=100)
    verdict: str = Field(..., regex=r"^(AUTO_ACCEPT|MANUAL_REVIEW|REJECT)$")


async def verify_show(tx_id: str) -> VerificationReport:
    """
    Perform comprehensive verification of an uploaded show.
    
    Args:
        tx_id: Arweave transaction ID of the show manifest
        
    Returns:
        Complete verification report with scores and verdict
        
    Raises:
        HTTPException: If verification fails or show cannot be accessed
    """
    try:
        # Fetch manifest from Arweave
        manifest = await fetch_manifest(tx_id)
        
        # Compute integrity hash
        computed_hash = blake3.blake3(manifest.raw_bytes).hexdigest()
        integrity_pass = computed_hash == manifest.claimed_hash
        
        # Analyze audio quality
        audio_score = await analyze_audio_quality(manifest.audio_files)
        
        # Check policy compliance
        policy_risk = await check_compliance(manifest.metadata)
        
        # Generate verdict
        overall = calculate_weighted_score(
            integrity=100 if integrity_pass else 0,
            audio=audio_score,
            policy=100 - policy_risk
        )
        
        return VerificationReport(
            arweave_tx_id=tx_id,
            integrity_score=100 if integrity_pass else 0,
            audio_quality_score=audio_score,
            policy_risk=policy_risk,
            overall_score=overall,
            verdict=determine_verdict(overall, policy_risk)
        )
        
    except Exception as exc:
        logger.exception("Verification failed", extra={"tx_id": tx_id})
        raise HTTPException(
            status_code=500,
            detail=f"Verification failed: {str(exc)}"
        ) from exc
```

**Python Standards:**
- Use type hints for all function parameters and return values
- Implement Pydantic models for data validation
- Use async/await for I/O operations
- Handle exceptions explicitly with appropriate logging
- Write docstrings in Google style
- Use pathlib for file operations
- Prefer f-strings for string formatting
- Keep functions focused and under 50 lines

---

## Pull Request Process

### Before Submitting

**Checklist:**
- [ ] Code follows project style guidelines
- [ ] All tests pass locally (`pnpm test` or equivalent)
- [ ] Linters pass without errors (`pnpm lint`)
- [ ] New code has adequate test coverage (≥85% for contracts, ≥80% for application code)
- [ ] Documentation updated for new features or API changes
- [ ] Commit messages follow Conventional Commits format
- [ ] No merge conflicts with target branch

### Creating the Pull Request

1. **Push your branch:**
   ```bash
   git push origin feature/your-feature-name
   ```

2. **Open PR on GitHub:**
   - Target the `develop` branch (not `main`)
   - Fill out the pull request template completely
   - Link related issues with "Closes #123" or "Relates to #456"
   - Add appropriate labels (bug, enhancement, documentation, etc.)

3. **Request reviews:**
   - Tag relevant maintainers or domain experts
   - Join the #dev channel on Discord for questions

### PR Review Process

**Timeline:**
- Initial review within 48 hours for standard PRs
- Security-sensitive PRs reviewed within 24 hours
- Complex architectural changes may require longer review

**Approval Requirements:**
- All repositories require 1 approving review
- Smart contract changes require 2 approving reviews
- Security-related changes require security team review

**Review Criteria:**
- Code quality and adherence to standards
- Test coverage and quality
- Documentation completeness
- Security considerations addressed
- Performance implications considered
- Breaking changes documented in PR description

### Addressing Review Feedback

```bash
# Make requested changes
git add .
git commit -m "refactor(registry): address review feedback"

# Push updates
git push origin feature/your-feature-name
```

**Best Practices:**
- Respond to all review comments, even if just to acknowledge
- Ask clarifying questions if feedback is unclear
- Mark conversations as resolved after addressing
- Update PR description if scope changes

### Merging

Once approved:
1. Ensure all CI checks pass (tests, lint, build)
2. Rebase on latest `develop` if needed
3. Squash commits if they represent iterative work-in-progress
4. Maintainer will merge using "Squash and merge" or "Rebase and merge"
5. Delete feature branch after merge

---

## Testing Requirements

### Test Coverage Targets

| Component | Minimum Coverage |
|-----------|------------------|
| Smart Contracts | 85% |
| API Endpoints | 80% |
| Verification Logic | 90% |
| Frontend Components | 70% |
| Utility Functions | 85% |

### Test Organization

**Structure:**
```
src/
  feature/
    index.ts
    feature.ts
    feature.test.ts      # Unit tests
    feature.integration.test.ts  # Integration tests
```

### Unit Tests

Test individual functions and components in isolation:

```typescript
// File: src/crypto/hash.test.ts
import { describe, it, expect } from 'vitest';
import { computeBlake3Hash } from './hash';

describe('computeBlake3Hash', () => {
  it('should compute correct hash for known input', () => {
    const input = 'test data';
    const expected = '5d02...abc3';  // Known hash
    
    const result = computeBlake3Hash(input);
    
    expect(result).toBe(expected);
  });
  
  it('should produce different hashes for different inputs', () => {
    const hash1 = computeBlake3Hash('input1');
    const hash2 = computeBlake3Hash('input2');
    
    expect(hash1).not.toBe(hash2);
  });
  
  it('should be deterministic', () => {
    const input = 'test';
    const hash1 = computeBlake3Hash(input);
    const hash2 = computeBlake3Hash(input);
    
    expect(hash1).toBe(hash2);
  });
});
```

### Integration Tests

Test component interactions and external dependencies:

```typescript
// File: src/api/submit.integration.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { createServer } from '../server';
import { testDb } from '../test/db';

describe('POST /api/submit', () => {
  let server: Server;
  
  beforeAll(async () => {
    await testDb.migrate();
    server = await createServer();
  });
  
  afterAll(async () => {
    await testDb.cleanup();
    await server.close();
  });
  
  it('should accept valid show submission', async () => {
    const response = await fetch(`${server.url}/api/submit`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        arweaveTxId: 'abc123...',
        dateYMD: 19770508,
        venue: 'Barton Hall',
      }),
    });
    
    expect(response.status).toBe(201);
    
    const data = await response.json();
    expect(data).toHaveProperty('submissionId');
  });
  
  it('should reject submission with invalid transaction ID', async () => {
    const response = await fetch(`${server.url}/api/submit`, {
      method: 'POST',
      body: JSON.stringify({ arweaveTxId: 'invalid' }),
    });
    
    expect(response.status).toBe(400);
  });
});
```

### End-to-End Tests

Test complete user workflows:

```typescript
// File: tests/e2e/upload-flow.test.ts
import { test, expect } from '@playwright/test';

test('complete show upload workflow', async ({ page }) => {
  // 1. User connects wallet
  await page.goto('/upload');
  await page.click('button:has-text("Connect Wallet")');
  await page.waitForSelector('[data-testid="wallet-connected"]');
  
  // 2. User fills upload form
  await page.fill('[name="arweaveTxId"]', 'test-tx-id-123');
  await page.fill('[name="dateYMD"]', '19770508');
  await page.fill('[name="venue"]', 'Barton Hall');
  
  // 3. User submits
  await page.click('button:has-text("Submit Show")');
  
  // 4. Verify success message
  await expect(page.locator('.success-message')).toContainText(
    'Show submitted for verification'
  );
  
  // 5. Verify show appears in pending list
  await page.goto('/my-submissions');
  await expect(page.locator('.submission-list')).toContainText('Barton Hall');
});
```

### Testing Smart Contracts

```solidity
// File: test/ShowRegistry.t.sol
contract ShowRegistryTest is Test {
    ShowRegistry public registry;
    address public verifier = address(0x1);
    address public uploader = address(0x2);
    
    function setUp() public {
        registry = new ShowRegistry();
        registry.grantRole(registry.VERIFIER_ROLE(), verifier);
        vm.deal(uploader, 1 ether);
    }
    
    function testRegisterShow() public {
        vm.prank(verifier);
        registry.registerShow("abc123", bytes32(0), 19770508, 95);
        
        (address addr, , , uint8 status, ) = registry.shows("abc123");
        assertEq(addr, address(this));
        assertEq(status, uint8(ShowStatus.Verified));
    }
    
    function testCannotRegisterDuplicate() public {
        vm.startPrank(verifier);
        registry.registerShow("abc123", bytes32(0), 19770508, 95);
        
        vm.expectRevert("Show exists");
        registry.registerShow("abc123", bytes32(0), 19770508, 95);
        vm.stopPrank();
    }
    
    function testFuzz_DateValidation(uint32 date) public {
        vm.assume(date < 19650000 || date > 20501231);
        
        vm.prank(verifier);
        vm.expectRevert("Invalid date");
        registry.registerShow("abc123", bytes32(0), date, 95);
    }
}
```

### Running Tests

```bash
# TypeScript/Node projects
pnpm test              # Run all tests
pnpm test:unit         # Unit tests only
pnpm test:integration  # Integration tests
pnpm test:e2e          # End-to-end tests
pnpm test:coverage     # Generate coverage report

# Smart contracts
forge test             # Run all tests
forge test --match-test testRegister  # Specific test
forge test --gas-report  # Include gas usage
forge coverage         # Generate coverage report

# Python projects
pytest                 # Run all tests
pytest -v              # Verbose output
pytest --cov           # With coverage
pytest -k test_verify  # Specific test pattern
```

---

## Documentation Standards

### Code Documentation

**TypeScript/JavaScript:**
```typescript
/**
 * Fetches and validates a show from the registry.
 * 
 * @param txId - Arweave transaction ID of the show
 * @returns Show data if found and valid, null otherwise
 * @throws {ValidationError} If transaction ID format is invalid
 * @throws {NetworkError} If Arweave network is unreachable
 * 
 * @example
 * ```typescript
 * const show = await fetchShow('abc123...');
 * if (show) {
 *   console.log(`Show date: ${show.dateYMD}`);
 * }
 * ```
 */
export async function fetchShow(txId: string): Promise<Show | null> {
  // Implementation
}
```

**Solidity:**
```solidity
/// @notice Register a verified show in the registry
/// @dev Requires VERIFIER_ROLE, emits ShowRegistered event
/// @param txId Arweave transaction ID (43 character base64url)
/// @param rootHash BLAKE3 root hash of the show manifest
/// @param dateYMD Show date in YYYYMMDD format (19650000-20501231)
/// @param aiScore Verification confidence score (0-100)
/// @custom:security Must validate txId uniqueness before calling
function registerShow(
    string calldata txId,
    bytes32 rootHash,
    uint32 dateYMD,
    uint8 aiScore
) external onlyRole(VERIFIER_ROLE);
```

**Python:**
```python
async def verify_show(tx_id: str, options: VerificationOptions | None = None) -> VerificationReport:
    """
    Perform comprehensive verification of an uploaded show.
    
    This function orchestrates multiple verification steps including
    integrity checking, audio analysis, duplicate detection, and
    policy compliance validation.
    
    Args:
        tx_id: Arweave transaction ID of the show manifest (43 chars)
        options: Optional verification configuration overrides
        
    Returns:
        Complete verification report with scores and verdict
        
    Raises:
        ArweaveError: If show manifest cannot be fetched
        ValidationError: If manifest format is invalid
        VerificationError: If verification process fails
        
    Example:
        >>> report = await verify_show("abc123...")
        >>> if report.verdict == "AUTO_ACCEPT":
        ...     await register_show(report)
        
    Note:
        This operation may take 30-120 seconds depending on show size.
        Consider running verification asynchronously for large shows.
    """
```

### README Structure

Each repository should include a comprehensive README:

```markdown
# Repository Name

Brief description (1-2 sentences).

## Features

- Feature 1
- Feature 2
- Feature 3

## Prerequisites

- Tool/version
- Service requirements

## Installation

\`\`\`bash
# Step by step commands
\`\`\`

## Configuration

Environment variables and config files explained.

## Usage

Basic examples and common workflows.

## Development

Local development setup and workflows.

## Testing

How to run tests and interpret results.

## Deployment

Production deployment instructions.

## Contributing

Link to CONTRIBUTING.md.

## License

License information.
```

### API Documentation

Document all API endpoints using OpenAPI 3.0 specification:

```yaml
# File: openapi.yaml
openapi: 3.0.0
info:
  title: DeadW3 Protocol API
  version: 1.0.0
  description: REST API for DeadW3 show submission and retrieval

paths:
  /api/submit:
    post:
      summary: Submit a new show for verification
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ShowSubmission'
      responses:
        '201':
          description: Show submitted successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SubmissionResponse'
        '400':
          description: Invalid submission data
        '429':
          description: Rate limit exceeded

components:
  schemas:
    ShowSubmission:
      type: object
      required:
        - arweaveTxId
        - dateYMD
        - venue
      properties:
        arweaveTxId:
          type: string
          pattern: '^[a-zA-Z0-9_-]{43}$'
          example: 'abc123...'
        dateYMD:
          type: integer
          minimum: 19650000
          maximum: 20501231
          example: 19770508
```

### Architecture Decision Records

Document significant technical decisions in ADR format:

```markdown
# ADR-001: Use Base Network for Smart Contracts

## Status
Accepted

## Context
The protocol requires an Ethereum-compatible network for smart contracts
managing show registry, staking, and rewards distribution. Key requirements:
- Low transaction costs for frequent operations
- Ethereum security guarantees
- Strong developer ecosystem

## Decision
Deploy smart contracts on Base network (Ethereum L2 by Coinbase).

## Rationale
Base provides:
- Low gas costs (~$0.01 per transaction vs $5+ on Ethereum mainnet)
- Ethereum security through optimistic rollups
- Strong ecosystem with RainbowKit, Wagmi support
- Easy onboarding for users via Coinbase integration

Alternatives considered:
- **Optimism:** Similar benefits but less integrated ecosystem
- **Arbitrum:** Higher adoption but more complex security model
- **Polygon:** Lower security guarantees as sidechain

## Consequences
**Positive:**
- Enables sustainable economics for micro-rewards
- Reduces barrier to entry for uploaders
- Simplifies wallet integration

**Negative:**
- 7-day withdrawal period for bridging to Ethereum L1
- Dependent on Base network uptime and governance
- May need multi-chain strategy if user base fragments

## Implementation Notes
- Use Base Sepolia testnet for development
- Maintain contract portability for potential multi-chain deployment
- Monitor Base ecosystem for breaking changes
```

---

## Security Considerations

### Security Review Process

All code undergoes security review before merging:

**Smart Contracts:**
- Manual review by 2+ contract developers
- Automated analysis with Slither
- Test coverage ≥85% including edge cases
- Gas optimization review
- Fuzz testing for numeric operations

**Backend Services:**
- Dependency vulnerability scanning (npm audit, safety)
- SQL injection prevention (parameterized queries only)
- Input validation on all external data
- Rate limiting on all public endpoints
- Authentication and authorization checks

**Frontend:**
- XSS prevention (React auto-escaping, CSP headers)
- Wallet connection security (verify chain ID)
- Sensitive data never in localStorage
- HTTPS required in production

### Reporting Security Issues

**DO NOT** open public GitHub issues for security vulnerabilities.

Instead, follow our [Security Policy](./SECURITY.md):
1. Email security@deadw3.org with details
2. Include steps to reproduce
3. Allow 48 hours for initial response
4. Coordinate disclosure timeline with maintainers

### Security Checklist for PRs

- [ ] No hardcoded secrets or API keys
- [ ] Input validation on all external data
- [ ] SQL queries use parameterized statements
- [ ] Authentication required for sensitive operations
- [ ] Rate limiting implemented where appropriate
- [ ] Error messages don't leak sensitive information
- [ ] Dependencies reviewed for known vulnerabilities
- [ ] Contract functions validate all inputs
- [ ] Gas limits considered for contract operations

---

## Additional Resources

### Community Channels

- **Discord:** [DeadW3 Community Server](https://discord.gg/deadw3) - `#dev` channel
- **GitHub Discussions:** For proposals and design discussions
- **Weekly Dev Sync:** Thursdays 4pm UTC on Discord voice

### Documentation

- [Architecture Overview](https://github.com/DeadW3/deadw3-docs/blob/main/architecture/overview.md)
- [API Reference](https://github.com/DeadW3/deadw3-docs/blob/main/api/reference.md)
- [Smart Contract Documentation](https://github.com/DeadW3/deadw3-contracts/blob/main/docs/contracts.md)
- [Deployment Guides](https://github.com/DeadW3/deadw3-docs/blob/main/deployment/)

### Learning Resources

**Arweave:**
- [Arweave Documentation](https://docs.arweave.org/)
- [Bundlr Network Guide](https://docs.bundlr.network/)

**Base Network:**
- [Base Documentation](https://docs.base.org/)
- [OnchainKit by Coinbase](https://onchainkit.xyz/)

**Smart Contract Development:**
- [Foundry Book](https://book.getfoundry.sh/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)

---

## Questions?

If you have questions not covered in this guide:

1. Check existing documentation in [deadw3-docs](https://github.com/DeadW3/deadw3-docs)
2. Search closed GitHub issues for similar questions
3. Ask in the Discord `#dev` channel
4. Open a GitHub Discussion for design questions

Thank you for contributing to DeadW3! Your efforts help preserve cultural heritage for future generations.
