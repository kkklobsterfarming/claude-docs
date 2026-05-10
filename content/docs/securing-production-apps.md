---
title: Securing AI-Built Apps for Production
weight: 5
---

Agentic coding shipped a lot of code in 2025–2026 that nobody read carefully. The bug isn't the model — the bug is treating "the tests pass and the UI loads" as the bar for production. This page is the security companion to the rest of the workshop: workflows to bolt on at prompt time, workflows to run before merge, the standards your reviewers should reach for, and the news you should read so you understand the threat model.

> 🛑 **Use the smallest workflow that fits the risk.** A throwaway internal CRUD tool does not need an ASVS Level 3 audit. A SaaS handling customer PII does. Calibrate.

---

## Why this page exists

The gap between "agent wrote it" and "safe to expose to the internet" is wider than most people think. A few data points worth keeping in mind:

- **Time-to-exploit is collapsing.** The April 2026 advisory for `lmdeploy` (a popular LLM-inference server) saw working public exploits within roughly 12 hours of the CVE landing — well under the patch window for most teams. Treat any new advisory in your stack as already weaponized.
- **AI-generated code repeats predictable mistakes.** SSRF in fetch helpers, IDOR in REST handlers, missing authorization on internal admin routes, prompt injection sinks in user-facing AI features, and secrets baked into client bundles dominate post-incident reports.
- **The model is also a target.** Prompt injection, indirect prompt injection (via fetched pages or files), tool poisoning, and skill/MCP supply-chain attacks all sit in scope when your app calls an LLM at runtime.
- **"Vibe-coded" SaaS is a new attack surface.** Several public incidents in 2025–2026 traced root cause to authorization rules the agent invented and the developer never read. Reviewers can't catch what they didn't ask the agent to verify.

The takeaway: security can't be an end-of-project audit. It belongs in your CLAUDE.md, in your initial prompts, in your hooks, and in your pre-merge review pipeline.

---

## Threat-model first: security at prompt time

The cheapest place to fix a vulnerability is before it's written. These patterns put guardrails in *before* the agent generates code.

### CLAUDE.md additions for security

Drop something like this into your project's `CLAUDE.md` (or, for cross-project habits, your user-scope `~/.claude/CLAUDE.md`):

```markdown
## Security non-negotiables

- Treat all user input as hostile. Validate at the boundary; never trust shape, length, or charset coming from the network.
- Parameterize every database query. No string concatenation into SQL/NoSQL/ORM raw clauses, no exceptions.
- Output-encode by sink: HTML escape for HTML, JSON escape for JSON, shell-escape for exec, etc. Never the input side.
- Authenticate every endpoint by default. Authorization (who can do what) is a separate check from authentication (who is this).
- Every object lookup must verify the caller's ownership/role — IDOR is the #1 thing you keep missing.
- Never put secrets in client-side code, logs, or error responses. Read from env or a secret manager only.
- All outbound HTTP from server code must use an SSRF-safe client: deny RFC1918, link-local, metadata IPs (169.254.169.254), and resolve-then-connect to prevent DNS rebinding.
- Crypto: use the platform's high-level primitives (libsodium, Web Crypto, AWS KMS). Never roll your own. Never use MD5/SHA1 for anything security-relevant.
- File upload: validate by magic bytes, store outside the web root, serve via signed URLs, never trust the filename.
- Dependencies: prefer the standard library; if you add a package, justify it and pin the version. Flag any package the user has not seen before so they can verify it isn't a typosquat.
- LLM features: treat model output as untrusted input. Never eval/exec/render-as-HTML model output without the same sanitization you'd apply to user input.

## Required reviews before declaring done

- Run the project's static analyzer (semgrep / bandit / eslint security plugin) and fix everything it flags or explicitly justify the suppression.
- Run the project's secret scanner (gitleaks / trufflehog) on the diff.
- Run `/security-review` on the diff before opening a PR.
- For any change touching auth, payments, file I/O, network calls, or LLM features: explicitly enumerate the threat model in the PR description.
```

Tune to your stack — drop the SSRF rule if you have no outbound HTTP, add HIPAA/PCI clauses if you handle that data, etc. Specific beats generic.

### Initial prompt patterns

Add a security framing to feature prompts. Three patterns that work:

**1. Threat model before code.**

```text
Before writing any code, produce a brief threat model for this feature:
- Trust boundaries (what data crosses which boundary)
- STRIDE categories that apply (Spoofing, Tampering, Repudiation, Info Disclosure, DoS, Elevation of Privilege)
- Top 3 most-likely failure modes and their mitigations
Then propose the implementation. Wait for me to approve the threat model before writing code.
```

**2. Secure-by-default scaffolding.**

```text
Build <feature> with these defaults:
- All endpoints require authenticated session by default; mark public ones explicitly with a comment and justification.
- All database access goes through the existing repository layer; no raw queries in handlers.
- Input validated by <zod/pydantic/etc> at the boundary; output serialized through a typed response model.
- Errors return generic messages to the client; full stack traces only in server logs.
- New dependencies require my approval before install.
```

**3. Adversarial self-review during planning.**

```text
Plan this feature, then before executing, role-play as an attacker for one round:
"You are a malicious authenticated user. List 10 things you would try to exploit in this design."
Address every item in the plan or explain why it doesn't apply.
```

These work especially well in **plan mode** (`Shift+Tab`) — the agent commits to the threat model in writing before any file gets touched.

### Hooks that enforce the rules

CLAUDE.md is a suggestion, hooks are a constraint. Add a `PostToolUse` hook that runs your secret scanner on every write:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "gitleaks protect --staged --no-banner --redact -v >/dev/null 2>&1 || printf '{\"hookSpecificOutput\":{\"hookEventName\":\"PostToolUse\",\"additionalContext\":\"BLOCKED: gitleaks detected a likely secret in the latest write. Remove the secret and use env vars or a secret manager.\"}}'",
            "timeout": 15
          }
        ]
      }
    ]
  }
}
```

Other hook ideas:

- `PreToolUse` on `Bash` — block `curl <url> | sh`, `wget ... | bash`, and similar pipe-to-shell patterns.
- `PreToolUse` on `Bash` — block `pip install`, `npm install`, etc., unless the package is on an allowlist (catches slopsquatting).
- `Stop` — refuse to mark the task done if `git diff` adds anything matching `AKIA[0-9A-Z]{16}`, `sk-[a-zA-Z0-9]{32,}`, `ghp_`, `xox[baprs]-`, etc.
- `SessionStart` — print the project's threat model summary so the agent loads it into context every session.

---

## Post-implementation review workflows

The agent thinks it's done. You need to know whether it's safe. This is where you spend review effort.

### Built-in commands

| Command | What it does | When to use |
| --- | --- | --- |
| `/security-review` | Runs Anthropic's built-in security review skill against the current branch diff | Every PR, no exceptions |
| `/review` | General code review covering correctness, style, and security | Day-to-day PRs |
| `/ultrareview` | Multi-agent cloud review across the branch (separate billing) | High-risk changes (auth, payments, data migrations) |
| `/simplify` | Reuse + quality + efficiency pass; catches dead code that often hides bugs | Before any large merge |

`/security-review` reads the diff, looks for OWASP-class issues, and writes a markdown report. It is *not* a SAST tool — it's a model-driven pass. Treat it as one input among several.

### Multi-agent security review pattern

The `/team-code-review` workflow already covered in the main doc (see "Developer self code review before pushing code to GitHub") works well as a security gate when you focus its agents on security. A version specifically tuned for security:

```text
Run a multi-pass security review on the current branch diff. Use these passes in parallel:

Pass 1 — OWASP Top 10:2025 (web): For each category that applies, find concrete instances or
explicitly state "no instances found in this diff." Cite file:line.

Pass 2 — OWASP Top 10 for LLM Applications 2025: Especially LLM01 prompt injection, LLM02
sensitive info disclosure, LLM06 excessive agency, LLM08 vector and embedding weaknesses.
Skip if the diff has no LLM surface area.

Pass 3 — OWASP Agentic AI Threats: tool poisoning, memory poisoning, intent breaking, identity
spoofing, cascading hallucination. Skip if the diff has no agent surface area.

Pass 4 — Codex frontier model adversarial review: invoke /codex:adversarial-review with focus
on auth bypass, IDOR, SSRF, and unsafe deserialization.

Pass 5 — Dependency and secret scan: run semgrep ci, gitleaks detect, and `npm audit` /
`pip-audit` / equivalent. Report any High/Critical.

Then synthesize: severity-ordered findings with file:line, exploit description, and a fix.
End with a go/no-go recommendation.
```

### Tools to wire in

Skills and prompts are not a substitute for proper security tooling. Run these alongside the agent:

| Tool | Type | What it covers |
| --- | --- | --- |
| [Semgrep](https://semgrep.dev/) (+ [MCP](https://github.com/semgrep/mcp)) | SAST | Pattern-based static analysis; the free Community ruleset already catches a huge class of OWASP issues |
| [Snyk Code](https://snyk.io/product/snyk-code/) | SAST | Symbolic-execution-flavored static analysis; strong on data-flow vulns |
| [CodeQL](https://codeql.github.com/) | SAST | GitHub's query language for code; deep but slow; good for hard data-flow problems |
| [Bandit](https://github.com/PyCQA/bandit) | SAST (Python) | Python-specific; catches pickle, exec, weak crypto, hardcoded passwords |
| [ESLint security plugin](https://github.com/eslint-community/eslint-plugin-security) | SAST (JS/TS) | JavaScript regex DoS, eval, child_process patterns |
| [gitleaks](https://github.com/gitleaks/gitleaks) | Secret scan | Pre-commit hook; catches AWS/GCP/Slack/GitHub tokens |
| [TruffleHog](https://github.com/trufflesecurity/trufflehog) | Secret scan | Verifies that found secrets are *live* (reduces false positives) |
| [Trivy](https://aquasecurity.github.io/trivy/) | SCA + container | Dependencies, container images, IaC; one tool covers a lot |
| [Snyk Open Source](https://snyk.io/product/open-source-security-management/) | SCA | Dependency CVEs + license issues |
| [OSV-Scanner](https://github.com/google/osv-scanner) | SCA | Google's dep scanner against the OSV database; fast, free |
| [Dependabot](https://github.com/dependabot) / [Renovate](https://www.mend.io/renovate/) | SCA | Auto-PRs to bump vulnerable deps |
| [ZAP](https://www.zaproxy.org/) | DAST | Dynamic scan of running app; catches things SAST misses (auth, session) |
| [Nuclei](https://github.com/projectdiscovery/nuclei) | DAST | Template-driven scanner; great for known-CVE checks |
| [Promptfoo](https://www.promptfoo.dev/) | LLM eval | Red-team your LLM features for prompt injection, jailbreaks, PII leakage |
| [Garak](https://github.com/NVIDIA/garak) | LLM eval | NVIDIA's LLM vulnerability scanner |

**Wiring them into Claude Code:**

```bash
# Install Semgrep MCP for in-session SAST
claude mcp add semgrep -- npx -y semgrep-mcp@latest

# Then prompt
"Run semgrep on the diff with the p/owasp-top-ten ruleset and fix every finding. Suppressions need a comment justifying them."
```

A reasonable pre-merge ladder:

1. `gitleaks` (1s) — block on any hit
2. `semgrep --config p/ci` (5–30s) — block on High/Critical
3. `osv-scanner` (5s) — block on Critical
4. `/security-review` (1–3 min) — read-only, log findings to PR
5. `/codex:adversarial-review` (1–3 min) — second-model pass
6. For high-risk: `/ultrareview` (cloud, several min) — pre-merge final pass

---

## Standards and frameworks worth knowing

Reviewers ask "compared to what?" These are the answers.

### OWASP Top 10:2025 (Web Application Security Risks)

The default reference for web app security. The 2025 list (a refresh of 2021) covers:

| ID | Category |
| --- | --- |
| A01 | Broken Access Control |
| A02 | Cryptographic Failures |
| A03 | Injection |
| A04 | Insecure Design |
| A05 | Security Misconfiguration |
| A06 | Vulnerable and Outdated Components |
| A07 | Identification and Authentication Failures |
| A08 | Software and Data Integrity Failures |
| A09 | Security Logging and Monitoring Failures |
| A10 | Server-Side Request Forgery (SSRF) |

🔗 [owasp.org/www-project-top-ten](https://owasp.org/www-project-top-ten/)

A01 (Broken Access Control) is consistently the #1 finding in AI-generated apps. If your review only catches one class of bug, make it that one.

### OWASP Top 10 for LLM Applications 2025

The companion list for apps that *embed* an LLM. Different threat model, different mitigations:

| ID | Category | Quick read |
| --- | --- | --- |
| LLM01 | Prompt Injection | Direct (user) or indirect (via fetched/uploaded content) |
| LLM02 | Sensitive Information Disclosure | Model leaks secrets, PII, training data |
| LLM03 | Supply Chain | Compromised models, datasets, plugins, MCPs |
| LLM04 | Data and Model Poisoning | Tainted training/fine-tune data |
| LLM05 | Improper Output Handling | Trusting model output as code/HTML/SQL |
| LLM06 | Excessive Agency | Tools/permissions broader than the use case needs |
| LLM07 | System Prompt Leakage | Prompt extraction reveals secrets/logic |
| LLM08 | Vector and Embedding Weaknesses | RAG poisoning, embedding inversion |
| LLM09 | Misinformation | Confident hallucination harming users |
| LLM10 | Unbounded Consumption | Cost/DoS via runaway generation |

🔗 [genai.owasp.org/llm-top-10](https://genai.owasp.org/llm-top-10/)

If your app exposes a chat box, RAG, or any tool-using agent to end users, this is the list to memorize.

### OWASP Agentic AI: Threats and Mitigations

Newer (2025) and specifically about *agents* — multi-step, tool-using systems. Threat IDs include T1 Memory Poisoning, T2 Tool Misuse, T3 Privilege Compromise, T6 Intent Breaking, T9 Identity Spoofing, T15 Cascading Hallucination, and others. If you ship a Claude-Code-style agent to your users, this is required reading.

🔗 [genai.owasp.org/resource/agentic-ai-threats-and-mitigations](https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/)

### OWASP ASVS (Application Security Verification Standard) 5.0

A line-item checklist of what "secure" actually means, organized into Levels 1–3. Level 1 is "any internet-exposed app should pass this," Level 2 is the realistic bar for SaaS handling user data, Level 3 is for high-assurance (financial, health, government).

🔗 [owasp.org/www-project-application-security-verification-standard](https://owasp.org/www-project-application-security-verification-standard/)

Treat ASVS as the long form of OWASP Top 10. When a reviewer asks "did you check X?", the answer is "ASVS V*N.M*."

### OWASP MASVS (Mobile)

If you ship a mobile app, MASVS is the equivalent of ASVS for iOS/Android — covers storage, crypto, IPC, code quality, and resilience.

🔗 [mas.owasp.org/MASVS/](https://mas.owasp.org/MASVS/)

### CWE Top 25 Most Dangerous Software Weaknesses

MITRE's view of the most common, exploitable weaknesses. Where OWASP Top 10 is categories, CWE is *specific* weaknesses (CWE-79 XSS, CWE-89 SQL injection, CWE-352 CSRF, etc.). When a reviewer says "this is a CWE-918 SSRF," that's where the number comes from.

🔗 [cwe.mitre.org/top25](https://cwe.mitre.org/top25/)

### MITRE ATT&CK and ATLAS

- **ATT&CK** — adversary tactics and techniques for traditional IT. Useful when you're modeling who attacks you and how. 🔗 [attack.mitre.org](https://attack.mitre.org/)
- **ATLAS** — the same idea but for AI/ML systems. Covers tactics like reconnaissance against an LLM, model evasion, model extraction, and so on. 🔗 [atlas.mitre.org](https://atlas.mitre.org/)

### NIST AI RMF and NIST SSDF

- **NIST AI Risk Management Framework** — governance for AI systems; useful for compliance conversations. 🔗 [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
- **NIST Secure Software Development Framework (SSDF, SP 800-218)** — the secure SDLC standard most enterprises map to. 🔗 [csrc.nist.gov/Projects/ssdf](https://csrc.nist.gov/Projects/ssdf)

### EU AI Act and ISO/IEC 42001

If you're shipping into the EU or selling to enterprises:

- **EU AI Act** — risk-based regulation; high-risk systems have compliance obligations. 🔗 [artificialintelligenceact.eu](https://artificialintelligenceact.eu/)
- **ISO/IEC 42001:2023** — AI management system standard; the AI equivalent of ISO 27001. 🔗 [iso.org/standard/81230.html](https://www.iso.org/standard/81230.html)

---

## Vulnerability classes that AI-generated code keeps shipping

These are the specific patterns to grep for in any agent-written diff. The list isn't exhaustive — it's the things that show up over and over in post-incident reports.

### Authorization

- **Missing object-level authorization (IDOR — CWE-639).** Endpoint accepts an `id`, looks it up, returns the row. No check that the caller owns it. Test: as user A, request user B's resource by ID.
- **Function-level authorization gaps.** Admin route gated only by "knowing the URL." Test: hit `/admin/*` as a normal user.
- **Authorization in the frontend only.** UI hides the button; the backend doesn't check. Test: call the API directly.

### Injection (CWE-89, CWE-78, CWE-94)

- String-concatenated SQL / NoSQL queries.
- `eval`, `exec`, `Function()`, `child_process.exec` with user-controlled input.
- Template injection (Jinja2 `{{...}}`, ERB, etc.) where templates are built from user input.
- LDAP, XPath, command, header injection — same root cause, different sink.

### SSRF (A10, CWE-918)

The agent writes a "fetch this URL for the user" feature and skips: blocking `127.0.0.1`, `169.254.169.254` (cloud metadata), private ranges, link-local; resolving DNS *before* connecting; following redirects safely. Cloud metadata SSRF is still routinely how attackers go from "I can make outbound HTTP" to "I have your IAM credentials."

### Secrets

- API keys hardcoded in the repo.
- Secrets in client-side bundles (anything in `NEXT_PUBLIC_*`, `VITE_*`, or `REACT_APP_*` ships to the browser).
- Secrets logged in error messages or APM traces.
- `.env` files committed.
- Tokens in URL query strings (logged everywhere — server logs, CDN logs, browser history, Referer header).

### File handling

- Path traversal (`../../etc/passwd`) — happens whenever the agent uses user input as a filename without normalization.
- Unrestricted upload — no MIME check, no size limit, served from the same origin.
- ZIP bombs / decompression bombs — accepting user archives without limits.

### Crypto

- MD5 / SHA1 for password hashing or anything security-relevant. Use Argon2id, scrypt, or bcrypt for passwords; SHA-256 for hashes.
- Hardcoded IVs / nonces, ECB mode, AES-CBC without HMAC.
- "Custom" token formats instead of JWT/PASETO/signed-cookies from a vetted library.

### LLM-specific

- **Prompt injection sinks:** any feature where user input + retrieved content + system prompt all flow into the model and the output is rendered as HTML, run as code, or used to make a tool call. Treat model output as untrusted.
- **Tool overscope:** the agent has `Bash` access in production. Almost never the right call. Allowlist what it actually needs.
- **Indirect prompt injection:** the agent fetches a webpage / parses a PDF / reads a calendar invite, and the document contains instructions for the agent. Strip or fence untrusted content.
- **Memory poisoning:** long-lived agent memory that user input can write to, then later reads back and obeys.
- **Cost DoS (LLM10):** `/api/chat` with no rate limit, no max token cap, no auth → someone bills your company $50k overnight.

### Dependencies

- **Slopsquatting:** the agent invents a package name that doesn't exist; an attacker registers it shortly after; your next install runs malware. Mitigation: never let the agent install a dep without the human pinning the version and reading the package page.
- **Outdated transitive deps with known CVEs.** Run an SCA on every PR.
- **Lockfile not committed** → reproducibility lost, supply chain unverifiable.

---

## Skills, plugins, and MCPs for security

Beyond the OWASP and SecLists skills already noted in the main doc:

| Name | Type | Purpose | Link |
| --- | --- | --- | --- |
| `/security-review` | Built-in skill | Anthropic's diff-scoped security review | Built into Claude Code |
| OWASP Security skill | Skill | OWASP Top 10:2025 + ASVS 5.0 + Agentic AI rulesets | [agamm/claude-code-owasp](https://github.com/agamm/claude-code-owasp) |
| SecLists & Agents | Skill | Wordlists + injection payloads + pentest helpers | [Eyadkelleh/awesome-claude-skills-security](https://github.com/Eyadkelleh/awesome-claude-skills-security) |
| Semgrep MCP | MCP | Run semgrep rulesets in-session; iterate on findings | [semgrep/mcp](https://github.com/semgrep/mcp) |
| Snyk MCP | MCP | Snyk Code SAST + dependency CVEs in-session | [snyk/snyk-ls](https://github.com/snyk/snyk-ls) |
| Burp Suite MCP | MCP | Drive an authenticated Burp scan from the agent | Various community implementations |
| Trivy MCP / CLI | CLI via Bash | Container + IaC + dependency scanning | [aquasecurity/trivy](https://github.com/aquasecurity/trivy) |
| Garak | CLI | LLM red-teaming (prompt injection, jailbreaks, leakage) | [NVIDIA/garak](https://github.com/NVIDIA/garak) |
| Promptfoo | CLI | Eval + red-team your LLM features | [promptfoo/promptfoo](https://github.com/promptfoo/promptfoo) |
| Devil's Advocate | Skill | Adversarially challenge previous reviews | [Devil's Advocate skill](https://github.com/notmanas/claude-code-skills/tree/main/skills/devils-advocate) |

> ⚠️ **Skills warning, again.** Re-read the *Skill Security* section of the main doc before installing any of these. A skill described as "OWASP Top 10 reviewer" that ships a `scripts/exfil.sh` is a thing that has happened in the wild.

---

## Required reading: news, research, postmortems

These are the receipts. If you only skim one section of this page, make it this one.

### Incidents and write-ups

- **The April 2026 lmdeploy advisory and the 12-hour exploit window.** A reminder that supply-chain CVEs in AI infra are weaponized faster than they can be patched. Search GitHub Security Advisories for `InternLM/lmdeploy` and the corresponding CVE.
- **EchoLeak — Microsoft 365 Copilot zero-click prompt injection (CVE-2025-32711).** Aim Security's disclosure of an indirect-prompt-injection chain that exfiltrated tenant data without a click. The first widely-publicized "AI worm-style" CVE in production enterprise software. 🔗 [aim.security/lp/aim-labs-echoleak-blogpost](https://www.aim.security/lp/aim-labs-echoleak-blogpost)
- **Rules File Backdoor (Pillar Security, 2025).** Hidden Unicode-tag instructions inside `.cursorrules` (and equivalent rules files) coerce the agent into inserting backdoors invisible to human review. Same class of attack lands on `CLAUDE.md`, `AGENTS.md`, custom slash commands, and skill READMEs. 🔗 [pillar.security/blog/new-vulnerability-in-github-copilot-and-cursor-how-hackers-can-weaponize-code-agents](https://www.pillar.security/blog/new-vulnerability-in-github-copilot-and-cursor-how-hackers-can-weaponize-code-agents)
- **ClawHavoc (Snyk, 2025).** Three lines of Markdown in a `SKILL.md` file is enough to escalate to full shell access on the developer's machine. 🔗 [snyk.io/articles/skill-md-shell-access](https://snyk.io/articles/skill-md-shell-access/)
- **The Sondera "invisible sentence" PDF skill hijack.** Hidden instructions inside a PDF bundled with a skill alter the skill's behavior in ways the SKILL.md never describes. 🔗 [blog.sondera.ai/p/claude-skill-hijack-invisible-sentence](https://blog.sondera.ai/p/claude-skill-hijack-invisible-sentence)
- **Slopsquatting research (Lasso Security and others).** Attackers register the package names that LLMs consistently hallucinate; the agent confidently `pip install`s them on your machine. 🔗 [arxiv.org/abs/2406.10279](https://arxiv.org/abs/2406.10279) (the original "package hallucination" paper); search for "slopsquatting" for follow-ups.
- **Anthropic Threat Intelligence Reports.** Quarterly write-ups of how Anthropic sees Claude being misused by attackers (vibe-coded malware, recon assistance, ransomware drafting). Useful for understanding what your defenders are now up against. 🔗 [anthropic.com/news](https://www.anthropic.com/news) (filter for "threat report" / "misuse").
- **HiddenLayer ShadowLogic and adjacent agent attacks.** Research showing how adversaries plant logic in model graphs, RAG corpora, and tool definitions. 🔗 [hiddenlayer.com/research](https://hiddenlayer.com/research/)
- **Snyk's "Top Claude Skills for Cybersecurity" roundup.** Practitioner-level overview of skills worth installing. 🔗 [snyk.io/articles/top-claude-skills-cybersecurity-hacking-vulnerability-scanning](https://snyk.io/articles/top-claude-skills-cybersecurity-hacking-vulnerability-scanning/)

### Research papers worth a skim

- **"Prompt Injection attack against LLM-integrated Applications"** — Liu et al. The canonical taxonomy. 🔗 [arxiv.org/abs/2306.05499](https://arxiv.org/abs/2306.05499)
- **"Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection"** — Greshake et al. The paper that named indirect prompt injection. 🔗 [arxiv.org/abs/2302.12173](https://arxiv.org/abs/2302.12173)
- **"Are Your LLMs Capable of Stable Reasoning?"** and follow-ups on *jailbreak persistence* across model versions — useful when your security argument relies on model alignment.
- **"Universal and Transferable Adversarial Attacks on Aligned Language Models"** — Zou et al. The GCG attack. 🔗 [arxiv.org/abs/2307.15043](https://arxiv.org/abs/2307.15043)

### Postmortems

- **Anthropic's postmortem of three recent quality issues** (already linked in the main doc) — useful because it shows that even the model provider can ship inference-engine bugs that change behavior. Build redundancy accordingly. 🔗 [anthropic.com/engineering/a-postmortem-of-three-recent-issues](https://www.anthropic.com/engineering/a-postmortem-of-three-recent-issues)
- **Public breach reports of "vibe-coded" SaaS.** Search for the 2025–2026 reports on Lovable / v0 / Bolt-built apps shipping with anonymous-readable Postgres tables, missing RLS, or admin endpoints exposed. The pattern repeats: the agent enabled a feature, the developer never verified the access policy.

### Community resources to keep tabs on

- **OWASP GenAI Security Project** — the umbrella for LLM Top 10, Agentic AI threats, and the AI security community. 🔗 [genai.owasp.org](https://genai.owasp.org/)
- **GitHub Security Advisories database** — subscribe to advisories for every dep your stack uses. 🔗 [github.com/advisories](https://github.com/advisories)
- **OSV.dev** — Google's unified vulnerability database. 🔗 [osv.dev](https://osv.dev/)
- **CISA Known Exploited Vulnerabilities catalog** — what's actually being exploited in the wild right now. 🔗 [cisa.gov/known-exploited-vulnerabilities-catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- **HackerOne / Bugcrowd disclosed reports** — read real bugs from real apps; pattern-match into your own code. 🔗 [hackerone.com/hacktivity](https://hackerone.com/hacktivity)

---

## A minimum-viable security workflow

If you take nothing else from this page:

1. **Add the security non-negotiables** to `CLAUDE.md` (above).
2. **Threat-model in plan mode** for any feature touching auth, data, files, network, or LLM input.
3. **Hooks**: gitleaks on `PostToolUse`, allowlist on package installs.
4. **Pre-merge ladder**: gitleaks → semgrep → osv-scanner → `/security-review` → `/codex:adversarial-review`.
5. **For high-risk diffs**: `/ultrareview` on the cloud, plus a human reviewer who reads the diff line-by-line.
6. **Subscribe** to GitHub Security Advisories for your stack and to the OWASP GenAI mailing list.
7. **Quarterly**: re-run a full SAST + SCA + DAST pass, update CLAUDE.md with whatever the agent kept getting wrong.

Security in agentic coding is the same discipline as security anywhere else — just with a faster code-generation loop and a wider blast radius. The defense is the same: assume hostile input, verify every trust boundary, and never let the agent be the only thing that read the diff.

---

[← Back to main page](/docs/agentic-coding-in-terminal/)
