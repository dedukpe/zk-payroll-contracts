# Local Setup and Test Troubleshooting

This guide provides solutions for common issues encountered when setting up the local environment and running tests for ZK Payroll contracts.

> **Environment variables and test expectations:** see [contracts/README.md](../README.md) for the full list of deployment variables (`NETWORK`, `SOURCE`, contract IDs), debugging flags (`RUST_BACKTRACE`), and a manual QA checklist.

## 1. Environment Setup Issues

### Missing `wasm32-unknown-unknown` Target
**Error:** `error[E0463]: can't find crate for std` or similar when running `cargo test` or `stellar contract build`.
**Solution:** Ensure the WASM target is installed for your active Rust toolchain.
```bash
rustup target add wasm32-unknown-unknown
```

### Soroban CLI Version Mismatch
**Error:** Unrecognized commands or unexpected test behavior when invoking contracts.
**Solution:** Verify your Soroban/Stellar CLI version is v21+.
```bash
stellar --version
```
If outdated, upgrade it via cargo:
```bash
cargo install --locked stellar-cli
```

## 2. Common Test Failures

### Missing WASM Fixtures (`no such file`)
**Error:** Integration tests fail because they cannot load a `.wasm` file (e.g., `target/wasm32-unknown-unknown/release/payroll_registry.wasm`).
**Why it happens:** Our integration tests load the compiled WebAssembly binaries of the contracts. If you haven't built them yet, the tests will fail.
**Solution:** Always build the contracts before running integration tests.
```bash
stellar contract build
cargo test
```

### HostError or Contract Panics
**Error:** A test fails with `HostError` or `Status(WasmVm)`.
**Why it happens:** Soroban traps on any Rust `panic!()` or failed `.unwrap()`. The backtrace in WASM is often opaque.
**Solution:**
1. Run tests with the environment variable `RUST_BACKTRACE=1` to get more detailed Rust backtraces.
2. Look for `unwrap()`, `expect()`, or out-of-bounds array access in the contract code. Replace them with proper error handling (`Result<T, Error>`).

### Out of Resources (CPU/Memory/Gas)
**Error:** `CpuLimitExceeded`, `MemLimitExceeded`, or test runner hanging.
**Solution:**
- The default Soroban environment in tests has resource limits mimicking the testnet.
- Check for unbounded loops or overly large data structures (e.g., large `Vec` instead of `Map`).
- If you legitimately need more resources for a test, consider modifying the budget in the test environment (e.g., `env.budget().reset_unlimited()`).

## 3. ZK Proof Setup Failures

### `snarkjs` or Circuit Verification Fails in Tests
**Error:** `Not enough values for input signal` or proof verification fails locally.
**Solution:**
- Ensure you have correctly downloaded the Phase 1 `ptau` file and completed the Phase 2 trusted setup.
- If you modified `payment.circom` or any other circuit, you **must** rerun the Phase 2 setup. The existing `.zkey` and `.wasm` witness generator will be invalidated.
- See the [ZK Trusted Setup](../../CONTRIBUTING.md#zk-trusted-setup-ptau) section in `CONTRIBUTING.md` for the exact regeneration commands.

## Getting More Help
If you encounter an issue not listed here, check our more comprehensive [Soroban Build Troubleshooting](../../docs/troubleshooting-soroban-build.md) guide or ask for help in the `#soroban-dev` channel on the Stellar Discord.

## 4. Contract Test Fixture Builder Pattern & Naming Conventions (#441)

To maintain test readability, reduce duplicate setup logic across PRs, and make contributor contributions easier to review, all contract tests should follow the standardized Fixture Builder Pattern.

### Overview of Fixture Layers

ZK Payroll contracts test infrastructure is partitioned into three distinct fixture tiers:

| Tier | Location | Scope & Purpose | Example |
|---|---|---|---|
| **Unit Fixtures** | `contracts/<crate>/tests/common/` | Local single-contract mocks, basic auth, and standalone method invocation. | `common::setup(&env)` |
| **Integration Fixtures** | `contracts/integration_tests/src/fixtures.rs` | Cross-contract bindings (`Payroll` + `SalaryCommitment` + `ProofVerifier` + `Token`). | `fixtures::ACME_CORP`, `fixtures::ALICE` |
| **Invariant Fixtures** | `contracts/tests/` | Long-running state machines, cross-period liability tracking, and conservation of balance checks. | `CrossPeriodFixture<'a>` |

---

### Expected Pattern: Struct-Based Fixture Builder

When writing integration or multi-contract unit tests, wrap test setup in a struct that owns contract clients, account addresses, and environment references.

#### 1. Struct Definition
Always tie the fixture to the `soroban_sdk::Env` lifetime `'a`:

```rust
pub struct PayrollTestFixture<'a> {
    pub env: &'a Env,
    pub payroll: PayrollClient<'a>,
    pub token: TokenClient<'a>,
    pub verifier: ProofVerifierClient<'a>,
    pub commitment: SalaryCommitmentContractClient<'a>,
    pub admin: Address,
    pub treasury: Address,
    pub employee: Address,
    pub token_address: Address,
}
```

#### 2. Factory / Builder Constructor
Provide a standard `setup(env)` or `default(env)` constructor that registers mock contracts and initializes default roles:

```rust
impl<'a> PayrollTestFixture<'a> {
    pub fn setup(env: &'a Env) -> Self {
        env.mock_all_auths();

        // 1. Deploy & initialize mock verification key
        let verifier_id = env.register_contract(None, ProofVerifier);
        let verifier = ProofVerifierClient::new(env, &verifier_id);
        verifier.init_verifier_admin(&Address::generate(env));
        verifier.initialize_verifier(&mock_vk(env));

        // 2. Deploy salary commitment registry
        let commitment_id = env.register_contract(None, SalaryCommitmentContract);
        let commitment = SalaryCommitmentContractClient::new(env, &commitment_id);
        commitment.init_commitment_admin(&Address::generate(env));

        // 3. Deploy test token and fund treasury
        let token_id = env.register_contract(None, Token);
        let token = TokenClient::new(env, &token_id);
        let treasury = Address::generate(env);
        token.mint(&treasury, &1_000_000_000); // 1,000 stroop equivalent

        // 4. Deploy core payroll contract
        let payroll_id = env.register_contract(None, Payroll);
        let payroll = PayrollClient::new(env, &payroll_id);
        let admin = Address::generate(env);
        let employee = Address::generate(env);

        payroll.initialize(
            &admin,
            &token_id,
            &verifier_id,
            &commitment_id,
            &treasury,
            &Address::generate(env),
        );
        commitment.set_payroll_operator(&payroll_id);

        Self {
            env,
            payroll,
            token,
            verifier,
            commitment,
            admin,
            treasury,
            employee,
            token_address: token_id,
        }
    }

    /// Fluent builder method to customize initial treasury balance.
    pub fn with_treasury_balance(self, amount: i128) -> Self {
        self.token.mint(&self.treasury, &amount);
        self
    }
}
```

---

### Naming Conventions

Maintainers and reviewers expect strict naming consistency across all test files:

1. **Fixture Struct Names**:
   - Must use CamelCase suffixed with `Fixture`:
     - `PayrollTestFixture`
     - `CrossPeriodFixture`
     - `AuditModuleFixture`
     - `PauseManagerFixture`

2. **Actor Constants**:
   - Shared mock company and employee entities must reference canonical fixtures in `contracts/integration_tests/src/fixtures.rs`:
     - Companies: `ACME_CORP` (id: 0), `TECHSTART_INC` (id: 1), `GLOBALPAY_LTD` (id: 2).
     - Employees: `ALICE` (salary: 5,000), `BOB` (salary: 3,500), `CAROL` (salary: 7,200), `DAVE` (salary: 4,200).

3. **Period Identifiers**:
   - Use canonical `"YYYY-MM"` format for test payroll runs (e.g. `"2026-01"`, `"2026-02"`).

4. **Assertion Methods**:
   - Attach invariant and health checks directly to the fixture struct using `assert_` prefixes:
     - `fixture.assert_global_invariant("post-settlement");`
     - `fixture.assert_treasury_balance_conserved();`
     - `fixture.assert_all_reservations_settled();`

---

### Privacy & Confidentiality Rules in Test Fixtures

- **No Plaintext Salaries in Public Events**: Ensure tests assert that emitted events contain cryptographic commitments (`BytesN<32>`) or obfuscated identifiers, never plaintext salary integers.
- **Blinding Factors**: Never hardcode real private keys or production seeds. Use deterministic pseudo-random constants (e.g., `blinding_factor: 123`).
- **Clean Failure Assertions**: When testing failure paths (e.g., `Error::AlreadyInitialized`), match the exact typed enum variant rather than generic panics.
