# tclk on Windows: local deal rehearsal and record audit

**Prepared by:** [ugozfb](https://github.com/ugozfb) · [X](https://x.com/ugozfb_o)
**First rehearsal:** 20 September 2026 · **Tampered-signature test and update:** 21 September 2026
**Status:** Published community guide. Not an official FLOP Labs document.

*Türkçe sürüm: [README.tr.md](README.tr.md)*

I ran tclk, FLOP's agent-to-agent deal protocol, on my Windows machine against a local Technocore server. Then I downloaded the deal's records into two files and fed them to the project's own auditor. Four records were accepted; the final state was `claimed`. I then changed a single character of the lock signature in a copy: the auditor rejected the signature and the deal stayed in `accepted (not terminal)`.

This guide is for Windows users who want to run the same experiment. My contribution is not a new protocol or auditor; it is running the existing official example, documenting the Windows-specific obstacles, and showing how to check the result.

**PAPER carries no real value.** No FLOP was spent, no GPU work was done, and no real payment was made in this experiment. I make no claim about airdrop eligibility or any reward earned. [S1]

## 1. Scope of the experiment

Two temporary test identities played the payer and payee roles in the same example script. The script wrote the offer, accept, lock and reveal steps to the local server. My personal DID and private-key file were not used. These test identities were not sent to the shared Technocore network. [S1; experiment output: section 7]

| Tried | Result |
| --- | --- |
| Starting the local Technocore server | Health check returned `ok`. |
| Building the tclk and MCP components | Two build targets completed without error. |
| Hash-locked example deal on PAPER | The example program reached `claimed`. |
| Downloading two room records as JSONL | Two files downloaded. |
| Auditing the files with the separate official script | Four `ok`, then `fold → claimed`. |
| Changing one character of the lock signature | Signature rejected; result `accepted (not terminal)`. |

What this table states are experiment results observed with the terminal outputs below. It does not mean I ran the whole test suite or performed a security audit of the protocol.

## 2. Environment and version limits

| Component | Value seen in the experiment |
| --- | --- |
| Windows | Windows 10 |
| Windows shell | Windows PowerShell 5.1 |
| Windows Node.js | v24.15.0 |
| Windows Git | 2.54.0.windows.1 |
| pnpm | 11.25.0 |
| Linux environment | Ubuntu under WSL |
| uv | 0.12.17 |
| Technocore venv Python | CPython 3.12.14 |
| Technocore folder | `~/technocore-local` inside Ubuntu |
| tclk folder | `C:\dev\tclk-audit` on Windows |
| Experiment server | `http://127.0.0.1:8080` |

**HEAD records taken from the local repositories on 21 September:**

- tclk: `5cc4ab93efbc8999a3a7e1471b639deca25998ea`
- Technocore: `e4c4f73f3b28612d7161170b11e08e580b02123a`

Source links are pinned to these versions. A HEAD record alone does not prove the working tree was unchanged; `git status` output was not separately recorded during the experiment. The results of the official audit run in the terminal were supplied by the user. The raw records are included in the package below.

## 3. Before you start

This guide starts from the point where **Ubuntu can be opened under WSL** and **Git and Node.js are installed on Windows**. BIOS and WSL setup issues vary by machine, so I give no single universal repair command. If you need setup, see [Microsoft's WSL instructions](https://learn.microsoft.com/en-us/windows/wsl/install).

Two windows are used:

- **Ubuntu:** the Technocore server runs here.
- **Windows PowerShell:** the tclk example and the auditor run here.

Run the code blocks one at a time. Prefixes the terminal already shows, such as `PS C:\...>` or `ugozfb@...$`, are not commands. Do not paste result lines back into the terminal either. If a step errors, do not move to the next one.

If the folders already exist, do not clone again. To use an existing setup, continue from the relevant `cd` step.

## 4. Ubuntu: run Technocore locally

**The commands in this section are typed into the Ubuntu window.**

If uv is missing, the install path used in the experiment:

```bash
cd ~
```

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```bash
source "$HOME/.local/bin/env"
```

```bash
uv --version
```

This downloads and runs uv's official install script. [uv install docs](https://docs.astral.sh/uv/getting-started/installation/)

Get the Technocore source:

```bash
git clone https://github.com/flop-labs/technocore-chat.git ~/technocore-local
```

```bash
cd ~/technocore-local
```

On a fresh, clean clone you can pin the guide's version with the extra step below (this pin command was not separately run during the experiment):

```bash
git checkout --detach e4c4f73f3b28612d7161170b11e08e580b02123a
```

```bash
uv sync --frozen
```

In our setup uv selected Python 3.12.14 for the virtual environment. Ubuntu's `python3 --version` differing from that was not an error by itself. Technocore's development doc uses Python 3.12 and uv. [S4]

Start the server:

```bash
CHAT_ROOT=./data uv run --frozen uvicorn --app-dir src app:app --host 127.0.0.1 --port 8080
```

Success lines seen in the experiment:

```text
Application startup complete.
Uvicorn running on http://127.0.0.1:8080
```

In the Windows browser, open the [local health check](http://127.0.0.1:8080/healthz). Ours returned `ok`. This address goes to the machine of whoever opens it, not to the guide author's machine.

**Leave the Ubuntu window open.** With this startup form, the server's data is kept in the Technocore folder's `data` directory. The startup form is an adaptation of the official `serve` recipe that explicitly binds to the local address. [S5]

## 5. Windows PowerShell: install and build tclk

**Commands from here on are typed into Windows PowerShell.** The working directory is under `C:\dev`.

```powershell
git clone https://github.com/flop-labs/tclk.git C:\dev\tclk-audit
```

```powershell
cd C:\dev\tclk-audit
```

On a fresh, clean clone, pin the guide's version (extra pin step; not separately run during the experiment):

```powershell
git checkout --detach 5cc4ab93efbc8999a3a7e1471b639deca25998ea
```

```powershell
npm.cmd install -g pnpm@11.25.0
```

```powershell
pnpm.cmd install --frozen-lockfile
```

```powershell
pnpm.cmd build:all
```

Two `tsc -p tsconfig.json` builds completed in the experiment. The `package.json` read defines the package manager as `pnpm@11.25.0`. `build:all` builds the root and MCP projects in the workspace. [S3]

Cloning the repo alone is not enough: the example script needs the root `dist` output and the `mcp/dist/signing.js` file. [S1]

## 6. Make sure the rehearsal goes to the local server

**Do not skip this step.** The script's default target is the live `technocore.chat`. Without the environment variable it tries to write to the shared server. [S1]

In the same PowerShell window:

```powershell
$env:TECHNOCORE_URL = "http://127.0.0.1:8080"
```

```powershell
node examples/live-deal.mjs
```

A new PowerShell window requires setting the environment variable again. Do not run the example directly without setting the address.

First line of the run:

```text
venue    http://127.0.0.1:8080
```

The program creates two temporary identities and a new deal. **Your identities, deal id and room name will differ from mine.** You do not need to add your real private-key file to this operation. [S1]

## 7. What happened in our rehearsal?

| Record | Meaning |
| --- | --- |
| `offer` | The payer proposed the terms. |
| `accept` | The other side accepted and stated the value for the hash lock. |
| `lock` | A lock record was created on PAPER and announced to the deal room. |
| `reveal` | The secret opening the lock was revealed. |

The payee also checked the PAPER record itself. At the end of the program, the state was recomputed from the room records without using either party's private key. [S1]

Relevant results from my terminal:

```text
payee checked the rail itself → verifyLock true
replayed 4 frames, ignored 0, final status: claimed
secret in the transcript opens the statement: true
```

The program wrote an example job spec for a TikTok video; it did not produce a video or inspect a video file. The lock opening is not proof that the work was done well. The script's own comment makes this distinction. [S1]

The comment at the top of the source file may state the message count differently. This guide's success criterion is not the number in the comment but the **four protocol records** seen in my run and in the separate audit.

## 8. Download the records

Download the records while the server is running. **The deal room below belongs to my completed experiment. In your own rehearsal, use the room name printed on the program's `deal room` line.** Copying my room name into your setup does not download your deal.

The offer room address is `tclk-offers` in the example. This command can be used directly:

```powershell
curl.exe --fail --output offers-78fb924b.jsonl http://127.0.0.1:8080/r/tclk-offers/export
```

The file name is only a local label. I used my own deal's prefix; if you choose a different file name, use the same name in the audit command.

**The deal-download command run in my experiment:**

```powershell
curl.exe --fail --output deal-78fb924b.jsonl http://127.0.0.1:8080/r/mb-p-tclk-78fb924b2621803a/export
```

In a new experiment, replace the part between `/r/` and `/export` with your own `deal room` name. Take both files from the same experiment's local server.

In Windows PowerShell we used **`curl.exe`** rather than `curl`. In our session bare `curl` was interpreted as `Invoke-WebRequest` and asked for `Uri:`. If you get stuck on such a prompt, exit with Ctrl+C and run the command at a normal prompt.

With `--output` we wrote straight to a file; we did not build the JSONL by copying screen text.

## 9. Find the full contract id and audit

The program shows the start of the id, shortened on screen. The auditor wants the full id: `0x` followed by 64 lowercase hex characters. [S2]

In my experiment, using the known prefix, we pulled the full id from the file:

```powershell
$contract = [regex]::Match((Get-Content -Raw .\deal-78fb924b.jsonl), '0x78fb924b2621803a[0-9a-f]{48}').Value
```

```powershell
Write-Output $contract
```

**In your own experiment** replace the `0x78fb924b2621803a` part with the `0x` and first 16 hex characters the program shows. Do not copy the trailing ellipsis. The form of this command specific to my experiment was run; adapting it to a different id was not separately tested here.

Our full id:

```text
0x78fb924b2621803a933010f1bc69c97acaa9de86356ebd75d23841fe47093c08
```

If the output is empty, do not continue. Check that the file and the prefix belong to the right deal. Do not use the fixed full id above for your own run.

Then, in the same PowerShell window:

```powershell
node examples/audit-export.mjs offers-78fb924b.jsonl deal-78fb924b.jsonl $contract
```

**The real terminal output we got:**

```text
ok  tclk-offers#1 offer
ok  tclk-offers#2 accept
ok  mb-p-tclk-78fb924b2621803a#1 lock
ok  mb-p-tclk-78fb924b2621803a#2 reveal

fold → claimed
```

This section is terminal text recorded during the run. It does not replace the raw signed JSONL files.

## 10. How to read the result

`fold` means applying the records in order to compute the deal's final state. The auditor first finds the verified offer/accept pair, then processes the deal records. It works from files; it makes no network request. [S2]

| Output | How to read it |
| --- | --- |
| Four `ok`, result `claimed` | The result of our successful rehearsal. |
| A `BAD` on a line | That record has an unaccepted condition; check the reason next to it. |
| `no authenticated offer/accept pair` | No verified offer/accept pair found for the given id. |
| `usage: ...` | Arguments are missing or the contract id format is wrong. |
| `(not terminal)` | The records did not reach a final state. |

The `BAD` and `(not terminal)` results were observed in the tampered-signature test below. Other error examples are explained from the source code; they were not separately triggered. [S2]

**Do not look only at the last line or the exit code.** The auditor also treats `refunded` and `cancelled` as terminal. It also prints each step's `ok`/`BAD` separately. In our result there were four `ok` and `claimed` together. [S2]

## 11. Limits of this experiment

- We ran a local, controlled example deal; we did not transact with two independent operators.
- PAPER held no real asset. `claimed` is the protocol state in this rehearsal; it is not a bank or on-chain payment receipt. [S1]
- Work delivery and quality were not audited. [S1]
- One tampered-signature scenario was run. In the Windows experiment the full test suite, the refund path and other attack scenarios were not separately run. Repo tests run afterward in the staging environment are noted separately below.
- Auditing from a file does not mean we provided an independent time witness for the server timestamp. This guide makes no claim of holding an identity at any past date.
- This work's effect on airdrop eligibility or reward was not verified.

**My assessment:** the value of the work is that a Windows user can try the same flow and learn to check a "completed" message with a separate record audit.

## 12. Shutdown and re-auditing

After the two JSONL files are downloaded, you can stop the Ubuntu server with Ctrl+C. As long as the built tclk setup and the files remain, the separate audit script can read these files without a network connection. [S2]

To re-check the same deal, do not start a new rehearsal. A new rehearsal creates new test identities and a new deal. In a new PowerShell window, `$contract` must be reassigned.

## 13. Tampered-signature test: one character, a different result

On 21 September, in Windows PowerShell, I copied the original deal file. In the copy I changed the first character of the `sig` value in the first record from `m` to `A`. The text, nonce, sender, time and the second record's signature were not changed.

The official auditor's terminal output:

```text
ok  tclk-offers#1 offer
ok  tclk-offers#2 accept
BAD mb-p-tclk-78fb924b2621803a#1 record — record signature does not verify
BAD mb-p-tclk-78fb924b2621803a#2 reveal — reveal in status accepted

fold → accepted (not terminal)
```

The first `BAD` is the rejection of the altered lock signature. The second `BAD` is the reveal step not applying because no valid lock was applied. **We did not tamper with the second record's signature.** The result being `accepted (not terminal)` instead of `claimed` shows the expected flow was interrupted. This single scenario does not prove the whole protocol is secure.

### What's in the package

- [Offer and accept](offers-78fb924b.jsonl): the bytes of the uploaded original file are preserved.
- [Lock and reveal](deal-78fb924b.jsonl): the bytes of the uploaded original file are preserved.
- [Tampered test copy](deal-78fb924b-tampered.jsonl): recreated in the staging environment with the same single-character change; it is not a file separately uploaded from the home machine. It differs from the original by exactly one byte.
- [Successful audit output](audit-pass.txt) and [negative test output](audit-tampered.txt): text copies of the Windows terminal outputs the user supplied; not a signed report.
- [Extra signature check](extra-signature-check.txt): in the staging environment, the four original Ed25519 signatures were separately verified and the tampered one rejected. This check does not count as re-running the official state machine.
- [SHA256SUMS.txt](SHA256SUMS.txt): content digests of the files in this package; not an authorship or date attestation.

The signatures belong to two temporary test identities. They do not prove my personal DID or who ran the experiment; run attribution rests on the guide's byline and the reported experiment record. The `secret` field is a revealed rehearsal lock value, not a private signing key. The export of the separate KV job-spec note is not in this package; the offer contains its path.

## 14. Re-audit ready records: no Ubuntu needed

Without creating a new deal, you can check the same records in the package. First install and build the tclk dependencies from section 5. The commands below assume you extracted the package into `C:\dev\tclk-paket`. If you chose a different folder, change the paths. These package-path-adapted commands were not separately run on Windows; they use the same official script and the argument shape tested by the user.

```powershell
node C:\dev\tclk-audit\examples\audit-export.mjs C:\dev\tclk-paket\offers-78fb924b.jsonl C:\dev\tclk-paket\deal-78fb924b.jsonl 0x78fb924b2621803a933010f1bc69c97acaa9de86356ebd75d23841fe47093c08
```

Expected: four `ok`, `fold → claimed`.

```powershell
node C:\dev\tclk-audit\examples\audit-export.mjs C:\dev\tclk-paket\offers-78fb924b.jsonl C:\dev\tclk-paket\deal-78fb924b-tampered.jsonl 0x78fb924b2621803a933010f1bc69c97acaa9de86356ebd75d23841fe47093c08
```

Expected: two `ok`, two `BAD`, `fold → accepted (not terminal)`.

### If you want to repeat the tampering yourself

This step is optional; the tampered example is already in the package. Work on a new copy, not the original. The flow below was run on the home machine and its result recorded above.

```powershell
Copy-Item -LiteralPath C:\dev\tclk-audit\deal-78fb924b.jsonl -Destination C:\dev\tclk-audit\deal-78fb924b-tampered.jsonl -ErrorAction Stop
```

```powershell
$file = 'C:\dev\tclk-audit\deal-78fb924b-tampered.jsonl'
$text = [System.IO.File]::ReadAllText($file)
$rx   = [regex]::new('"sig":"m')
$out  = $rx.Replace($text, '"sig":"A', 1)
if ($out -eq $text) { throw 'Expected signature not found; file unchanged.' }
[System.IO.File]::WriteAllText($file, $out, [System.Text.UTF8Encoding]::new($false))
```

This tampering command is specific to this example file; in a different deal the first signature may not start with `m`.

## 15. Extra checks in the staging environment

These are checks done in a Linux staging environment, separate from Uğur's Windows session. The official export auditor was re-run with both file sets: the original records gave `claimed` (exit 0), the tampered copy `accepted (not terminal)` (exit 1). The real outputs are in [extra-official-audit.txt](extra-official-audit.txt). No message was sent to a live or local server for this rerun.

- In the real deal file, 64-hex values occur five times: `contract` four, `secret` one. `statement` is not in this file; it is in the accept record in the offer room.
- A generic regex finds the correct contract id first in this file. Prefix-anchored matching avoids depending on the field order of the first match; we do not claim the generic search returns the wrong result in this example.
- The SHA-256 of the 32 bytes decoded from the `secret` hex matches `accept.statement`. The `0x...` character string in the text is not hashed directly.
- `accept.ref == offer.id`; accept, lock and reveal are bound to the same `contract`; in the PAPER example `lock.ref == lock.contract`.
- The asset tag in this experiment's offer is `PAPER`, the rail `paper`. The live network's offer distribution or room capacity was not measured by this experiment.

Numeric results are in [extra-record-check.txt](extra-record-check.txt). The separate verification of the four signatures is documented in [extra-signature-check.txt](extra-signature-check.txt).

On a separate tclk working copy with the comment fix applied, on Linux / Node v24.19.0 / pnpm 11.25.0, dependency install, workspace build and repo tests completed successfully: 104 root, 40 MCP, 33 Worker, 177 total. This does not mean the full suite was run on the user's machine; nor is it a security audit on its own. [Extra check summary](extra-test-summary.txt).

## Primary sources

In the 21 September 2026 update the files below were re-read at the reported commits. Links are pinned to file versions.

- **S1:** [tclk / examples/live-deal.mjs](https://github.com/flop-labs/tclk/blob/5cc4ab93efbc8999a3a7e1471b639deca25998ea/examples/live-deal.mjs): local address setting, two test identities, PAPER warning, four steps, the example's own audit and work-quality limit.
- **S2:** [tclk / examples/audit-export.mjs](https://github.com/flop-labs/tclk/blob/5cc4ab93efbc8999a3a7e1471b639deca25998ea/examples/audit-export.mjs): file-based audit, id format, state and error outputs.
- **S3:** [tclk / package.json](https://github.com/flop-labs/tclk/blob/5cc4ab93efbc8999a3a7e1471b639deca25998ea/package.json): pnpm version and build commands.
- **S4:** [technocore-chat / CONTRIBUTING.md](https://github.com/flop-labs/technocore-chat/blob/e4c4f73f3b28612d7161170b11e08e580b02123a/CONTRIBUTING.md): Python/uv setup and health check.
- **S5:** [technocore-chat / justfile](https://github.com/flop-labs/technocore-chat/blob/e4c4f73f3b28612d7161170b11e08e580b02123a/justfile): the local server's `serve` recipe.

The official example and auditor belong to FLOP Labs. This text is a Turkish/English application guide prepared with AI assistance, based on ugozfb's run outputs.
