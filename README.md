# Smart Vestiny

A token vesting program built on **Solana** with the **Anchor** framework. An employer creates a company vesting account funded from a treasury, registers employees with per-employee vesting schedules, and each employee claims their linearly-vested SPL tokens over time — with a cliff before any tokens unlock.

---

## What it does

- **Create a vesting account** for a company, backed by a program-owned treasury token account for a chosen mint
- **Register an employee** with a vesting schedule: total allocation, start time, end time, and a cliff
- **Claim vested tokens** — an employee withdraws the portion that has vested so far, computed linearly between start and end time, gated by the cliff
- Tracks how much each employee has already withdrawn so repeated claims only release the newly-vested amount

---

## How it works

### Vesting accounts & the treasury
An employer calls `create_vesting_account` with a company name. This initializes a `VestingAccount` PDA and, in the same instruction, a **treasury token account** — an SPL token account that is its own PDA authority. The treasury holds the tokens that will later be distributed to employees. The mint is fixed at creation and stored on the vesting account.

### Employee schedules
For each beneficiary the employer calls `create_employee_account`, which stores an `EmployeeAccount` PDA holding the schedule:

- `total_amount` — the full allocation
- `start_time` / `end_time` — the linear vesting window
- `cliff_time` — the timestamp before which nothing can be claimed
- `total_withdrawal_account` — running total of what has already been claimed

### Linear vesting with a cliff
On `claim_tokens`, the program reads Solana's on-chain `Clock` and computes how much has vested:

```
if now < cliff_time            -> reject (cliff not over)
if now >= end_time             -> vested = total_amount
otherwise                      -> vested = total_amount × (now - start_time) / (end_time - start_time)
```

The claimable amount is the vested amount minus what the employee has already withdrawn. If that is zero, the claim is rejected. The multiplication is done with a checked operation so an overflow surfaces as an error rather than wrapping.

### PDAs & CPI transfer
- The `VestingAccount` is a PDA derived from `[company_name, owner]`.
- The `treasury_token_account` is a PDA derived from `["treasury_token_account", company_name]`, and it is its own token authority.
- Each `EmployeeAccount` is a PDA derived from `["employee_account", beneficiary, vesting_account]`.

To move tokens out of the treasury, the program performs a **CPI to the token program's `transfer_checked`**, signing with the treasury PDA's seeds so only the program can authorize the transfer. The employee's destination is an associated token account, created on-demand via `init_if_needed` if it does not yet exist. After a successful transfer the employee's withdrawn total is updated.

The token accounts use Anchor's **token interface** (`TokenInterface` / `InterfaceAccount`), so the program works with both the SPL Token and Token-2022 programs.

---

## Instructions

| Instruction | Description |
|---|---|
| `create_vesting_account` | Create a company `VestingAccount` and its treasury token account for a given mint |
| `create_employee_account` | Register a beneficiary with a vesting schedule (total amount, start, end, cliff) |
| `claim_tokens` | Claim the tokens vested so far, transferred from the treasury via CPI |

---

## State

**`VestingAccount`** (one per company)
`owner`, `company_name`, `mint`, `treasury_token_account`, `treasury_bump`, `bump`

**`EmployeeAccount`** (one per beneficiary per company)
`beneficiary`, `total_amount`, `total_withdrawal_account`, `start_time`, `end_time`, `cliff_time`, `vesting_account`, `bump`

---

## Tech stack

- **Solana** — on-chain runtime
- **Anchor** — program framework
- **Rust** — program language
- **anchor-spl** — SPL token transfers (`transfer_checked`), associated token accounts, token interface (SPL Token / Token-2022)
- **TypeScript + Jest + solana-bankrun** — test suite (fast in-process Solana runtime, with a mint funded and the clock advanced to exercise vesting)

---

## Project structure

```
programs/smart_vestiny/src/
├── lib.rs                          # program entry — routes the 3 instructions
├── constants.rs                    # seed + Anchor discriminator size
├── error.rs                        # custom error codes
├── state/
│   └── account_states.rs           # VestingAccount + EmployeeAccount
└── instructions/
    ├── create_vesting_account.rs
    ├── create_employee_account.rs
    └── claim_tokens.rs
tests/
└── bankrun.spec.ts                 # integration tests (bankrun)
```

---

## Build & test

```bash
# prerequisites: Rust, Solana CLI, Anchor, Node + Yarn

yarn install          # install JS/TS dependencies
anchor build          # compile the program
anchor test           # run the bankrun test suite
```

**Program ID:** `98vJxJanGrYXsw22T8ERBweXvPWLEQwsxbpXAjiFdeKf`
