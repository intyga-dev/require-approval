# INTYGA — Require Human Approval (GitHub Action)

A composite Action that makes a production deploy (or any high-stakes step) **impossible without a
cryptographically-signed human approval** — verified offline before the job proceeds. No application code
changes: you wrap the workflow, not the app.

## Usage

```yaml
jobs:
  approve:
    runs-on: ubuntu-latest
    steps:
      - uses: intyga-dev/require-approval@v1
        with:
          gateway-url: ${{ secrets.INTYGA_GATEWAY_URL }}
          web-url: ${{ secrets.INTYGA_WEB_URL }}
          client-id: ${{ secrets.INTYGA_CLIENT_ID }}       # a SERVICE identity key
          client-secret: ${{ secrets.INTYGA_CLIENT_SECRET }}
          target: prod-cluster-01                          # the environment being acted on
          action-type: deploy_production
          action-description: "Deploy ${{ github.repository }}@${{ github.sha }} to production"
          params: '{"repo":"${{ github.repository }}","sha":"${{ github.sha }}"}'
          approvers-json: ${{ vars.INTYGA_APPROVERS_JSON }} # the trust anchor exported from your console (people + keys + origin)
          required-approvals: "1"                           # the minimum YOUR policy demands; a weaker signed requirement is refused
          request-id: "true"                                # this run's approval cannot be reused by another run
          evidence-file: approval-evidence.json             # receipt + verification result, kept as proof
          timeout: "600"
  deploy:
    needs: approve          # cannot start unless approval succeeded
    runs-on: ubuntu-latest
    steps:
      - run: echo "Approved — deploying."
```

Prefer `approvers-json` to `approver-keys`: it names **people**, so a quorum counts people rather than keys,
and it carries the console origin and RP ID, so you do not pass `webauthn-origin` / `webauthn-rp-id` separately.
Export it from your INTYGA console (Approvals → Trust anchor), review it, and keep it as a repository
variable. `approver-keys` (a flat list of base64 public keys, with `webauthn-origin` and `webauthn-rp-id`)
still works.

See [`deploy.yml`](./deploy.yml) for the full workflow.

## How it works
Three steps, so the approver is **notified where they already are** instead of hunting through CI logs:
1. **Request** (`intyga authorize --no-wait`): authenticates as a **SERVICE identity** and creates the
   challenge, returning the approval deep-link *immediately* (no blocking). Separation of duties by
   construction — the pipeline requests, a human approves.
2. **Notify** (`intyga notify`, if a `slack-webhook`/`teams-webhook` is set): posts an interactive message
   to your chat tool with the context and an **Approve** button that deep-links straight to `/approve`.
   Sent from your runner to your webhook — the context stays in your trust boundary.
3. **Await** (`intyga await <nonce> --consume`): blocks until the human signs with a passkey or security
   key, verifies the receipt offline against your trusted approvers (`approvers-json` or `approver-keys`), and single-use-consumes it. Exits non-zero
   on denial/timeout/verification failure, which fails the job — so any `needs:`-dependent deploy cannot run.

The developer gets a phone notification, taps **Approve**, signs with Face ID / a security key, and the
pipeline continues — they never open GitHub. A convenience gate, not a blocking wall.

## Inputs
| Input | Required | Default | Notes |
|---|---|---|---|
| `gateway-url` | ✅ | — | INTYGA gateway base URL |
| `web-url` | ✅ | — | Console base URL hosting `/approve` |
| `client-id` / `client-secret` | ✅ | — | A **SERVICE**-identity API key (passed via env, never on argv) |
| `target` | ✅ | — | The environment/RP being acted on. Bound into the signature (DIV Target Isolation), so an approval minted for staging cannot be replayed against prod |
| `action-type` | ✅ | — | Stable id bound into the signature, e.g. `deploy_production` |
| `action-description` | ✅ | — | Shown to the approver (WYSIWYS) |
| `approvers-json` | one of the two | — | The trusted approvers as an `intyga-trust-anchor` JSON document from your console: people with their keys, plus the console origin and RP ID. Give this **or** `approver-keys`, not both |
| `approver-keys` | one of the two | — | Comma-separated base64 approver public keys **you** trust. Verification must use a key you resolved — a receipt cannot vouch for its own signer, so there is no default |
| `required-approvals` | recommended | — | The minimum number of independent human approvals **your** policy demands. A receipt whose own signed requirement is weaker is refused (DIV §5 step 3d) |
| `request-id` | | `false` | `true` adds a fresh random `approvalRequestId` to the signed params, so one run's approval cannot be reused by another. The Action passes the same params to its await step |
| `evidence-file` | | — | Path to write the evidence JSON (receipt, what was expected, trust anchor, verification result, consumed). Mode `0600` |
| `params` | | `{}` | JSON of the exact parameters that will execute |
| `timeout` | | `600` | Seconds to wait for approval before failing |
| `consume` | | `true` | Single-use: mark the approval consumed once verified |
| `slack-webhook` | | — | Slack Incoming Webhook — posts an interactive Approve message |
| `teams-webhook` | | — | Teams Incoming Webhook — posts an Approve card |
| `sdk-version` | | pinned | `@intyga/sdk` version run via `npx` — defaults to the version this Action release was built against, never `latest`: a gate that resolves `latest` at run time executes whatever was published most recently. Override only to pin backwards |
| `verify-version` | | pinned | `@intyga/verify` version, the offline verifier that checks the receipt. Pinned separately because the SDK declares it as a caret range; both are installed exactly |

## Security notes
- Credentials are passed to the CLI **via the environment**, never as command-line arguments (which are
  visible in process listings).
- Approvals **cannot** be granted inside CI or a chat tool — only via a wallet or passkey signature on the
  authenticated `/approve` page. The Action just gets a human there and verifies the result.
