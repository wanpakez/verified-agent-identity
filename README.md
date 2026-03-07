name: Verify Human Identity

on:
  workflow_dispatch:
    inputs:
      agent_name:marwan
        description: 'marwan'
        required: true
        default: 'marwan'
      agent_description:
        description: 'marwan'
        required: true
        default: 'marwan AI Agent'

jobs:
  verify:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Step 1 - Install ClawHub skill
        run: npx clawhub@latest install verified-agent-identity

      - name: Install script dependencies
        run: |
          cd scripts
          npm install

      - name: Step 2 - Create new Ethereum Identity
        run: node scripts/createNewEthereumIdentity.js

      - name: Step 3 - Link Human to Agent
        run: |
          node scripts/manualLinkHumanToAgent.js --challenge '{"name": "marwan", "description": "marwan AI Agent"}'3. Generate a verification link to connect your human identity to the agent:

   ```bash
   node scripts/manualLinkHumanToAgent.js --challenge '{"name": "Agent Name", "description": "Short description of the agent"}'
   ```

   This prints the verification URL to the console. Open it in your browser to complete the identity linking process.

## Features

- **Identity Creation**: Generate new DIDs with random or existing Ethereum private keys
- **Identity Management**: List and manage multiple identities with default identity support
- **Human-Agent Linking**: Link a human identity to an agent's DID through signed challenges
- **Proof Generation**: Generate cryptographic proofs to authenticate as a specific identity
- **Proof Verification**: Verify proofs to confirm identity ownership

## Architecture

### Runtime Requirements

- **Node.js `>= v20`** and **npm** are required to run the scripts.
- The **`openclaw` CLI** must be installed and available in `PATH`. It is a hard runtime dependency used exclusively for sending direct messages to other agents or users on the Billions Network.

### Dependency Surface

npm dependencies are intentionally minimal and scoped to well-established, audited packages:

| Package                | Purpose                                                      |
| ---------------------- | ------------------------------------------------------------ |
| `@0xpolygonid/js-sdk`  | iden3/Polygon ID cryptographic primitives and key management |
| `@iden3/js-iden3-core` | DID and identity core types                                  |
| `@iden3/js-iden3-auth` | JWS/JWA authorization response construction and verification |
| `ethers`               | Ethereum key utilities                                       |
| `shell-quote`          | Shell token parsing used **only** for input sanitization     |
| `uuid`                 | UUID generation for protocol message IDs                     |

### Key Storage and Isolation

All cryptographic material is persisted to `$HOME/.openclaw/billions/` — a directory that lives **outside the agent's workspace**:

| File               | Contents                                        |
| ------------------ | ----------------------------------------------- |
| `kms.json`         | Private keys (unencrypted, owner-readable only) |
| `identities.json`  | Identity metadata                               |
| `defaultDid.json`  | Active DID and associated public key            |
| `challenges.json`  | Per-DID challenge history                       |
| `credentials.json` | Verifiable credentials                          |

### Subprocess Execution Safety

**Only one specific command is ever executed: `openclaw message send`**, with a fixed, hardcoded argument structure. The binary name, the subcommand, and all flag names (`--target`, `--message`) are hardcoded. User-supplied values are only ever passed as the **values** of those flags, never as the command name, subcommand, or flag names. Nothing else can be executed. There is no mechanism to change the binary, add flags, or inject subcommands. Additional security properties:

1. **No shell interpolation**: `execFileSync(binary, argsArray)` bypasses the OS shell entirely. The OS `exec*` syscall receives the argument vector directly — shell metacharacters in argument values are treated as literal data.
2. **Argument-level input validation**: Before the call, all user-supplied values pass through two independent validation layers:
   - `validateTarget` — enforces a strict allowlist regex (`/^[A-Za-z0-9:._@\-\/]+$/`) on the `--target` value.
   - `assertNoShellOperators` — uses `shell-quote` to tokenize the input and rejects any token with an `op` property (i.e., `|`, `&`, `;`, `>`, `<`, `$()`, etc.).
3. **No dynamic code execution**: No `eval`, `new Function`, `child_process.exec`, or shell-interpolated `execSync` calls exist anywhere in the codebase.

Prompt injection and arbitrary code execution are structurally impossible: the executed command and its flags are hardcoded constants, and user data can only influence the string values passed to `--target` and `--message` after sanitization.

### Network and External Binary Policy

- All external http calls will be made to trusted resources.
- No external binary other than `openclaw` is invoked.
- Any external URLs or verification links produced by the scripts are delivered to the user as a plain text message via `openclaw message send`. The agent has no ability to follow, fetch, open, or interact with those URLs in any way - it only forwards the string to the user.

## Documentation

See [SKILL.md](SKILL.md) for detailed usage instructions and examples.
