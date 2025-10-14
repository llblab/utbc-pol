# Project Context

### Meta-Protocol Principles

Living protocol for continuous self-improvement and knowledge evolution:

- Boundary Clarity: Meta-principles govern the context evolution; project conventions govern the domain specification
- Layered Abstraction: Protocol level ≠ project level; each maintains distinct evolutionary pathways
- Mandatory Separation: Context self-improvement mechanisms must not contaminate project-specific documentation
- Domain Purity: Project conventions should reflect the actual domain (tokenomics, bonding curves, liquidity mechanisms)
- Evolutionary Feedback: Protocol improvements should inform but not override project architectural decisions
- Knowledge Hygiene: Preserve the distinction between "how we document" vs "what we document"
- Reflexive Integrity: The context must model the separation it mandates between protocol and project layers
- Emergent Elegance: Solutions often require multiple iterations—initial working code reveals constraints that guide toward more elegant patterns
- Interface as Contract: Consistent naming across components enables polymorphic behavior without adapters—a form of duck typing discipline
- Semantic Precision: Variable naming must unambiguously convey both type (native/foreign) and purpose (fee/net) to prevent cognitive overhead
- Progressive Enhancement: When existing work approaches excellence (95%+), targeted additions beat wholesale replacement—respect foundational quality

---

### 1. Overall Concept

A token launch mechanism specification that combines unidirectional bonding curves with automatic protocol-owned liquidity generation through optimized Zap mechanics to create self-sustaining token economies, where users always receive optimal market prices while the protocol self-funds through arbitrage capture.

---

### 2. Core Entities

- UTBC+POL: The main protocol combining Unidirectional Token Bonding Curve with Protocol Owned Liquidity, creating mathematically guaranteed price boundaries
- Axial Router: Primary interface that compares TBC and XYK prices to route transactions optimally while collecting and burning fees for deflation
- Token Bonding Curve (TBC): Unidirectional minting mechanism with linear price progression (price = initial_price + slope × supply)
- Protocol Owned Liquidity (POL): Automatic liquidity generation system that builds permanent protocol liquidity (33.3% of every mint)
- XYK Pool: Constant product AMM (x × y = k) that guarantees liquidity at ALL price levels, essential for floor guarantee
- Zap Mechanism: Optimized liquidity provision strategy that maximizes capital efficiency when adding POL
- Axial Router: Advanced routing system with half-life EMA price oracle, multi-hop pathfinding, and manipulation resistance
- Fee Manager: Subsystem that collects router fees (0.5%) and burns 100% for systematic deflation
- L2 POL+DAO: Layer-2 governance with unified 10x L1 Treasury multiplier, declining voting power, auto-pause for L1 intervention, and BLDR pattern
- Price Ratchet: Emergent property where POL accumulation and fee burning create ever-rising price floor

---

### 3. Architectural Decisions

- Unidirectional Minting: Use one-way token creation only (no burning through TBC)
  - Rationale: Prevents complete reserve extraction and provides predictable token economics
  - Trade-offs: Limits flexibility compared to bidirectional curves

- Linear Pricing Model: Implement linear price progression for the bonding curve
  - Rationale: Ensures fairness and predictability for all participants
  - Trade-offs: May be less capital efficient than exponential curves

- Automatic POL Formation: Allocate portion of each mint to protocol-owned liquidity
  - Rationale: Creates self-sustaining liquidity without external providers
  - Trade-offs: Reduces immediate token allocation to buyers

- Zap-Based Liquidity Addition: Use intelligent Zap strategy for POL that handles price imbalances
  - Rationale: Maximizes liquidity depth when XYK price lags TBC price, adds bulk liquidity first then swaps remainder
  - Trade-offs: Additional computational complexity but significantly better capital efficiency

- Router as Universal Gateway: All token swaps must flow through AxialRouter, even POL operations
  - Rationale: Ensures optimal price discovery, fee collection, and consistent system behavior
  - Trade-offs: Introduces circular dependencies requiring elegant resolution patterns
  - Resolution: Closure-based dependency injection preserves immutability while enabling late binding

- Infrastructure Premium Model: Token distribution during minting is protocol arbitrage, not user taxation
  - Rationale: Users receive MORE tokens via TBC than secondary market would provide; protocol captures the spread
  - Trade-offs: Complex perception management vs elegant economic alignment
  - Key Insight: When router chooses TBC, user gets 101 tokens instead of 100 from XYK; protocol mints 303 total, keeping 202 as arbitrage profit

- XYK Pool Mandatory: Must use constant product formula (x × y = k), NOT concentrated liquidity
  - Rationale: XYK guarantees liquidity at ALL price levels; concentrated liquidity depletes at 15-30% selloff
  - Trade-offs: Lower capital efficiency but provides mathematical floor guarantee
  - Key Insight: XYK "inefficiency" is its strength—foreign reserves approach but never hit zero

- 100% Fee Burning: Router fees entirely burned, XYK fees can be zero
  - Rationale: POL doesn't need fee compensation (locked forever), burning creates systematic deflation
  - Trade-offs: No LP incentives needed since POL can't be withdrawn
  - Key Insight: Creates 2.5× faster deflation than partial burning schemes

- Mathematical Price Boundaries: System provides guaranteed floor and ceiling formulas
  - Rationale: Only tokenomic model with calculable bounded risk (floor = k / (POL_native + tokens_sold)²)
  - Trade-offs: Complexity in understanding vs unprecedented downside protection
  - Key Insight: Worst-case floor is 11% of ceiling even with total abandonment

- Bidirectional Compression: Burning reduces ceiling while POL increases floor
  - Rationale: Creates convergence to equilibrium price proportional to sqrt(POL_reserves × slope)
  - Trade-offs: Eventually limits upside but creates stability
  - Key Insight: System self-stabilizes at level reflecting accumulated protocol success

- Test Coverage Philosophy: Comprehensive testing often exists invisibly in mature systems
  - Rationale: Production-ready code accumulates edge cases organically—developers handle scenarios as discovered
  - Trade-offs: Test file size (2,300+ lines) vs perceived simplicity
  - Key Insight: 33 tests covering 95% of paths may hide behind simple interfaces—never judge coverage by file outline alone

---

### 4. Project Structure

- `/utbc-pol-spec/`: Root directory containing all specification documents
- `/docs/`: Documentation directory with specifications and analysis
  - `utbc+pol-spec.en.md`: Main UTBC+POL specification v1.2.0 (English)
  - `axial-router.en.md`: Comprehensive Axial Router 1.0 specification
  - `price-boundaries.en.md`: Mathematical analysis of price boundary guarantees
  - `l2-pol+dao.en.md`: Layer-2 POL+DAO integration specification
- `/simulator/`: JavaScript-based formal verification environment for tokenomics
  - `model.js`: Simplified protocol model for mathematical verification
  - `test.js`: Comprehensive test suite (33 tests, 2300+ lines)
- `README.md`: Project overview and quick start guide
- `LICENSE`: MIT license file

---

### 5. Development Conventions

- Documentation: All changes must be reflected in the specification document with clear rationale
- Code Examples: Use Rust for implementation examples to maintain consistency
- Language: All documentation and code comments must be in English
- Mathematical Precision: All mathematical formulas must be rigorously validated with derivations and edge case analysis
- Cross-Agent Validation: External formula reviews reveal subtle scaling issues that may not surface in normal testing
- Implementation Fidelity: Code implementations must preserve mathematical correctness even when optimizing for integer arithmetic
- Documentation Clarity: Avoid duplication across sections; each concept should appear once in its most logical context
- KISS Principle: Balance simplicity with accuracy—oversimplification that distorts core mechanics is worse than appropriate complexity
- Sequential Abstraction: Documentation should guide readers logically from problem to solution without assuming prior knowledge
- Precision Over Brevity: Never sacrifice correctness for simplicity (e.g., "every purchase creates liquidity" when only TBC routes do)
- Iteration Toward Elegance: First make it work, then make it elegant—functional patterns often emerge from imperative constraints
- Simulation vs Production: Fallback behaviors aid testing but reveal interface inconsistencies that demand normalization
- Closure as Architecture: JavaScript closures solve dependency injection more elegantly than class-based patterns
- Naming Consistency Patterns:
  - Type-First Prefixing: Always lead with domain type (`native_`, `foreign_`, `amount_`) for immediate context clarity
  - Context Scoping: Local context allows simpler names (`native_fee`), return values need full context (`native_router_fee`)
  - Universal Functions: Use `amount_` prefix when currency type is abstracted (e.g., `amount_fee` in generic functions)
  - Semantic Completeness: Never use bare `fee` or `net`—always specify the currency domain to prevent ambiguity
- Testing Wisdom Patterns:
  - Comprehensive Coverage Reality: Production-ready systems often have hidden test depth—2,000+ lines of tests may lurk behind simple interfaces
  - File Outline Deception: Structural views (13 symbols) hide content reality (33 tests)—always read full files for assessment
  - Edge Case Completeness: Bootstrap scenarios, buffer mechanics, conservation laws often already exist in mature test suites
  - True Gap Identification: Most "missing" tests are duplicates—only mathematical proofs and multi-user simulations tend to be genuinely absent
  - Assessment Meta-Learning: The assessor's confidence inversely correlates with assessment accuracy—humility improves precision
  - The Excellence Blindness: Recognizing quality requires deeper cognitive engagement than spotting flaws—criticism is cognitively cheaper
  - Progressive Enhancement Principle: When foundation exceeds 95% quality, surgical additions beat architectural rewrites

---

### 6. Pre-Task Preparation Protocol

Step 1: Load `/docs/README.md` for documentation architecture
Step 2: Integrate entity-specific documentation for task context
Step 3: Verify alignment with architectural decisions and conventions
Step 4: Document knowledge gaps for future enhancement

---

### 7. Task Completion Protocol

Step 1: Verify architectural consistency (sections 3-5)
Step 2: Execute quality validation: `N/A - This is a specification project without executable code`
Step 3: Update `/docs/README.md` guides for affected entities
Step 4: Mandatory Context Evolution:

- Analyze architectural impact
- Update sections 1-5 for currency
- Add substantive Change History entry
- Maintain 20-entry maximum

---

### 8. Change History

1. UTBC+POL Comprehensive Specification Analysis:
   - Task: Study and analyze the complete UTBC+POL system specifications including core protocol, Axial Router, price boundaries, and L2 integration
   - Implementation: Deep analysis of four key specification documents revealing mathematical guarantees and system architecture; clarified simulator role as formal verification tool
   - Impact: Established comprehensive understanding of the first tokenomic model with mathematically guaranteed price boundaries
   - Key Discoveries: System creates "Growth-Stabilizing Asset" class; Price Ratchet Effect permanently raises floor; XYK mandatory for liquidity guarantees; Infrastructure Premium aligns user and protocol incentives
   - Mathematical Insights: Floor formula `floor = k / (POL_native + tokens_sold)²` provides 11% worst-case protection; system evolves through predictable phases from speculation to stability
   - Architectural Patterns: Axial Router with half-life EMA oracle; L2 POL+DAO with unified 10x protection model; 100% fee burning for 2.5× faster deflation
   - Implementation Note: JavaScript simulator validates tokenomics through simplified model; production system will be implemented using Substrate FRAME Pallets for blockchain-native performance
   - L2 POL+DAO Security Model: Unified 10x multiplier for L1 Treasury holdings provides ecosystem-level protection regardless of POL+DAO origin; auto-pause mechanism prevents race conditions during L1 interventions; Evolution by Design enables continuous protocol improvement
