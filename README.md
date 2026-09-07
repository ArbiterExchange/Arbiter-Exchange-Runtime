# Arbiter Runtime

**Policy-enforced execution and isolated signing infrastructure for autonomous crypto agents.**

Arbiter Runtime is the machine layer beneath **Arbiter Exchange**. It runs trading agents inside a constrained environment and mediates the path between an agent's decision and an authorized blockchain transaction.

The core idea is deliberately simple:

> **Agents can propose actions. They do not get unrestricted access to the keys that authorize them.**

Arbiter Runtime turns an agent's intent into a typed request, checks it against policy, constructs a constrained transaction, and invokes the Arbiter Secure Signer only when the request is allowed.

## The Arbiter stack

| Component | Purpose |
| --- | --- |
| **Arbiter** | The overall ecosystem and platform. |
| **Arbiter Exchange** | Managed public-facing trading platform. |
| **Arbiter Runtime** | Agent execution environment in this repository. |
| **Arbiter Secure Signer** | Isolated signing component intended to keep private keys outside the agent boundary. |
| **Arbiter Policy Engine** | Authoritative permissions, limits, allowlists, and risk rules. |
| **Arbiter Protocol** | Versioned agent-to-policy-to-signer transaction protocol. |
| **ARBT** | Intended Base ecosystem token. ARBT has not launched. |

## Why Arbiter exists

An autonomous trading agent normally needs enough authority to turn a decision into a transaction. Giving the agent direct access to a hot-wallet key or a generic `sign()` function creates a very large trust boundary: model output, agent code, dependencies, plugins, operating system compromise, prompt injection, and application bugs can all become signing risk.

Arbiter separates those responsibilities.

```text
Market data / model
        |
        v
   Trading agent
  proposes intent
        |
        v
  Arbiter Runtime
 normalizes request
        |
        v
 Arbiter Policy Engine
 validates exact action
        |
   allowed?
   /      \
 no        yes
 |          |
reject      v
      Transaction builder
              |
              v
      Arbiter Secure Signer
      signs only approved form
              |
              v
           Network
```

The agent is therefore allowed to **reason about trading** without automatically inheriting **arbitrary transaction authority**.

## What the Runtime provides

The repository includes:

- isolated/scoped agent execution;
- model-provider profiles;
- authenticated local REST/MCP agent services;
- generated per-agent skill bundles;
- typed policy validation;
- persistent policy and trading-limit state;
- market-data and strategy support;
- reviewed transaction construction paths;
- external signer integration;
- encrypted software-key fallback for development and explicitly accepted risk profiles;
- transaction simulation, relay, and confirmation handling;
- secure-file and credential helpers;
- a versioned platform SDK for Arbiter Exchange;
- firmware and QEMU signing demonstrations;
- security audits, tests, and deployment tooling.

## Security boundary

Arbiter is designed so the proposal system and the signing system are different trust domains.

### Agent domain

The agent may:

- consume permitted market data;
- use the configured model provider;
- select from enabled strategy workflows;
- request an allowed trading action;
- read execution state exposed by its scoped service.

The agent should **not** receive:

- a raw private key;
- a generic arbitrary-message signing method;
- unrestricted transaction construction;
- arbitrary program or contract execution authority;
- the ability to rewrite its own policy.

### Policy domain

Before a transaction reaches the signer, Arbiter can validate controls such as:

- network / chain identity;
- allowed assets or mint pairs;
- allowed venues or programs;
- transaction form;
- maximum input amount;
- daily spend/input budget;
- maximum slippage;
- network and route fees;
- cooldown interval;
- daily execution count;
- destination / receiver rules;
- signer set;
- nonce / replay state;
- manual approval mode where configured.

### Signer domain

The preferred configuration uses an external signer adapter backed by a separately isolated signer, HSM, TPM, enclave, KMS, or other reviewed signing system.

Arbiter passes the signer a constrained authorization request and verifies the returned public-key signature. A production signer should independently validate the policy and transaction semantics it is authorizing.

**Key isolation is not sufficient by itself.** A signer that protects the key but will sign arbitrary bytes on request is not a meaningful policy boundary.

## Agent execution

Arbiter supports scoped autonomous agents that can operate through authenticated local REST/MCP interfaces. Trading capabilities are exposed only when enabled for the profile.

A generated trading skill can describe the exact configuration available to that agent, including supported venues, limits, and workflows such as:

- DCA;
- momentum;
- mean reversion;
- rebalancing;
- venue comparison;
- risk-off behavior.

Provider output is treated as untrusted input. The model does not get to expand the transaction schema or bypass the configured policy.

## Version

Current package version:

```text
arbiter-runtime v0.15.0
```

Node.js requirement:

```text
Node.js >= 20
```

## Quick start

Clone the repository and run the interactive installer:

```bash
./setup.sh install
```

For disposable Devnet development:

```bash
./setup.sh install --devnet
```

The recommended production-oriented path uses a reviewed **external signer**. If no signer adapter is available yet, print the signer setup guide:

```bash
./setup.sh install --signer-guide
```

For disposable development only, Arbiter also supports an explicitly acknowledged software-key path:

```bash
./setup.sh install --insecure
```

Do not treat the software-key path as equivalent to an external custody boundary.

## Runtime shell

The installer exposes the restricted `arbiter` operator command.

Useful starting commands include:

```text
arbiter> capa
arbiter> stat
arbiter> wal status
arbiter> pol show
arbiter> ag on
arbiter> ag st
```

Trading configuration and commands are documented in [`docs/DEX_TRADING.md`](docs/DEX_TRADING.md) and [`docs/QUICKSTART_SHELL.md`](docs/QUICKSTART_SHELL.md).

## Model providers

Arbiter supports pluggable proposal models while keeping the model outside the signer trust domain.

Commercial provider credentials should be stored in owner-only files or provider profiles rather than embedded in commands or general environment files.

## External signer configuration

A preferred deployment points Arbiter at an absolute-path signer adapter:

```bash
ARBITER_SIGNER_COMMAND=/absolute/path/to/reviewed-arbiter-secure-signer-adapter
ARBITER_SIGNER_ARGS_JSON=[]
ARBITER_SIGNER_TIMEOUT_MS=10000
```

The Runtime receives only the configured public key and the bounded signing capability. The private key should remain inside the external signing system.


## Zero-knowledge policy gate

The Runtime also contains an optional external proof-verification boundary for deployments that require a verified proof before authorization.

Example configuration:

```bash
ARBITER_REQUIRE_ZK_PROOF=1
ARBITER_ZK_VERIFIER_COMMAND=/absolute/path/to/reviewed-zk-verifier-adapter
ARBITER_ZK_PROOF_SYSTEM=groth16-bn254
ARBITER_ZK_CIRCUIT_ID=arbiter-private-policy-v1
ARBITER_ZK_VERIFYING_KEY_SHA256=<64-lowercase-hex-characters>
```

## Development

Install Node dependencies:

```bash
npm install
```

Run the complete test suite:

```bash
npm test
```

Run syntax, static, and unit checks:

```bash
npm run check
```

Start the Runtime service:

```bash
npm start
```

Run the included demo:

```bash
npm run demo
```

Build the demo only:

```bash
npm run demo:build
```

Run the agent security audit helper:

```bash
npm run security:agent
```

## Platform SDK

Arbiter Exchange integrates with the Runtime through the versioned export:

```js
import * as arbiter from "arbiter-runtime/platform-sdk";
```

The SDK boundary exists so the managed cloud service does not need direct access to internal Runtime modules.

## Firmware research

The repository includes both host-side Runtime code and lower-level firmware work.

The QEMU/RISC-V demonstration exercises a narrower signing boundary in which firmware receives a typed intent, validates policy, constructs the permitted transaction internally, and returns only the resulting signed transaction/signature material.

The demonstration is useful for testing the architecture, but **QEMU is not a hardware security boundary**. The host controls the emulator and can inspect guest memory. It must not be represented as production hardware custody.


## Who Arbiter is for

Arbiter is intended for systems where automated software needs transaction authority but unrestricted hot-key access is unacceptable, including:

- autonomous trading platforms;
- crypto-native market makers;
- automated treasury and liquidity systems;
- custody and wallet infrastructure;
- policy-enforced signing services;
- infrastructure teams experimenting with hardware-backed execution.

Arbiter Runtime is **not itself a trading strategy** and is not intended to be a general-purpose wallet. It is the execution and authorization layer underneath those applications.

## Product site

**https://arbiter.exchange**

## Disclaimer

Arbiter is active security and infrastructure engineering work. Automated trading and digital-asset signing systems can cause irreversible financial loss if policy, custody, transaction construction, or infrastructure assumptions fail. Treat demos, QEMU environments, and software-key profiles as development tools, and independently review any deployment before using real assets.
