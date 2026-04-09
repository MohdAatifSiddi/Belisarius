# Belisarius: Autonomous AI Security for npm Projects

Modern development suffers from tool sprawl and alert fatigue. Out-of-the-box scanners (npm audit, SAST, SCA tools) can list hundreds of issues, but without context, even “Critical” CVEs may be low‐risk. Belisarius aims to replace this with a single, **developer-friendly** system that *continuously* scans, prioritizes by real-world risk, and even auto-remediates. It uses Node.js for the CLI and dependency graph, plus AI for context-aware risk scoring. The core goals are speed, simplicity, and actionability: one command (`npx lice init`) should output a clean, ranked list of issues with fix suggestions. Unlike traditional scanners, Belisarius focuses on **what to do next**, not just what’s wrong.  

Belisarius’s **Dependency Intelligence Engine** will: parse `package.json` (and lockfiles) to build the full npm dependency graph (direct + transitive), then query open databases like OSV.dev or deps.dev for known CVEs/OSVs and metadata【6†L81-L89】【34†L149-L152】. These APIs aggregate GitHub Advisories, NVD, RustSec/PyPA databases, etc., providing comprehensive vulnerability data【6†L81-L89】【34†L149-L152】. In addition to CVEs, Belisarius flags “suspicious” packages – e.g. very-new or rarely-used libraries, or typosquatting look-alikes. (Supply-chain threats are growing: typosquatting and account-takeovers remained dominant in 2025【12†L180-L188】.) The scanner outputs a **prioritized risk list** of packages with issues, tagging each by severity and context. For example, it might note if a package is _older_, _unmaintained_, or likely malicious. 

## CLI Tool (Node.js)

Belisarius’s primary interface is a Node-based CLI. The user runs:  

```
npx lice init
```

This bootstraps a scan without any manual setup. Under the hood, the CLI reads the project’s `package.json` and lockfile (or uses `npm ls --json`) to enumerate all dependencies. It then invokes the scanner engine to analyze them. The output is printed to the console in a concise, human-readable format. By default it shows only **actionable findings**, ordered by risk. For example:

```
[CRITICAL] axios@0.21.0 → SSRF vulnerability  
Fix: upgrade to 0.21.4  

[MEDIUM] lodash@4.17.15 → unused in code (remove or update)  

[LOW] deprecated devDependency: jest@26.x – no CVE, but consider upgrade  
```

Each line starts with a severity tag, the package@version, and a brief description (often an AI-generated summary). A “Fix:” line gives an immediate remediation suggestion or command. This clean, minimal output avoids overwhelming developers. (All output is designed to be machine-readable too – e.g. a `--json` flag can emit structured results for CI). The CLI is fast (<5s) by using concurrent API calls and local caching of vulnerability data.  

## Dependency Scanner Engine

Belisarius’s scanner is responsible for building the **dependency graph** and fetching vulnerability data. It follows a minimal implementation: parse the lockfile (or run `npm ls`) to get all packages and their versions. For each, it queries an API like OSV.dev or deps.dev. For example, deps.dev returns both release dates and advisories for a given package version【6†L116-L124】, while OSV.dev returns all known vulnerabilities for that version【34†L149-L152】. These APIs aggregate multiple sources (GitHub Security Advisories, PyPA/RustSec, etc.【34†L149-L152】) and even non-CVE databases. By comparing the installed version with fixed versions, the scanner identifies which CVEs apply. 

To detect **suspicious packages**, Belisarius also checks metadata: the publisher’s account age, download count, release frequency, etc. (Heuristics similar to Datadog’s *GuardDog* tool can be used – GuardDog flags malicious patterns in npm code using Semgrep rules【18†L317-L325】.) Belisarius might flag a brand-new package with zero stars or one with a look-alike name. For now, any “suspicious” flag is a medium issue; in future versions AI may better classify these. Finally, the scanner outputs a list of findings with metadata (CVE ID, severity, fixed version, package path in the graph, etc.), which feeds into the risk engine.

## AI-Powered Risk Prioritization

Raw vulnerability lists are noisy. Belisarius’s AI layer re-scores each finding based on **real-world risk and context**. It combines static scores (CVSS/OSV severities, EPSS exploit likelihood) with project-specific context. For example, if a vulnerable library is only used in test code or behind strict access, its priority is lower. Conversely, a medium-severity CVE in a core authentication package might be elevated. This follows best practices: *contextual enrichment* drastically improves prioritization over blind CVSS ranking【20†L143-L152】【37†L112-L122】. In practice, Belisarius could embed an LLM or specialized model trained on CVE descriptions and code usage. The model would output a risk tier (Critical/High/Medium/Low) and a one-line summary. For instance, it might learn from code structure that “lodash is used in auth.js”, thus labeling a Lodash RCE CVE as critical “affecting auth flow”. 

Evidence shows this approach is needed: 56% of CVEs are scored “High” by CVSS, yet only ~1.7% are actually exploited in the wild【37†L84-L92】. AI-driven models (like the one described by NopSec) ask “what’s the probability this CVE is weaponized?”【37†L94-L102】. They also incorporate environment factors (asset criticality, network exposure, etc.)【37†L112-L122】. Similarly, Belisarius’s AI will consider signals such as “is this dependency called by production code?” and “does it handle secrets?” alongside known exploit data (e.g. active exploit reports). The result is a prioritized queue of *actionable* issues. Belisarius’s output might look like:

- **[CRITICAL]** lodash@4.17.15 – Prototype Pollution (CVE-2020-8203) *used in auth.js → patch immediately*  
- **[HIGH]** tar@2.2.2 – Arbitrary File Write (CVE-2021-32742) *network-accessible, upgrade to 2.2.3*  
- **[MEDIUM]** left-pad@1.3.0 – Denial of Service (CVE-2015-1596) *non-critical, safe to fix next release*  

The AI ensures findings are context-rich (“affecting X component”) and risk-ranked【20†L143-L152】【37†L84-L92】. The CLI preserves the minimal style: bold tags, short messages, and a **Fix:** hint.

## Auto-Fix Engine

For each finding, Belisarius suggests fixes. The simplest is a version bump: determine the **minimum safe version** by checking the vulnerability database. For example, OSV or deps.dev will list the fixed version (e.g. “CVE-2021-xxxx fixed in 0.21.4”). Belisarius then recommends that upgrade (e.g. `npm install axios@0.21.4`). To automate further, Belisarius can generate a patch: editing `package.json` and optionally running `npm install --package-lock-only`. Google’s OSV-Scanner does this via `osv-scanner fix`, which offers both non-interactive and interactive modes【34†L205-L213】【23†L677-L686】. Belisarius’s MVP can mimic the non-interactive mode: e.g. output a text diff or run the equivalent of `npm install --save axios@0.21.4`. It should also support a “dry-run” or “generate PR” mode (posting a GitHub patch), aligning with step 1 of our roadmap. 

In all cases, Belisarius will ensure safe semver boundaries (no breaking changes) and avoid upgrading if tests fail (it could run `npm test` after applying patches). Developers retain control: fixes are *suggestions*, not automatic commits, to prevent supply-chain backdoors. But as an option, an auto-PR feature (GitHub Action) can be built on top later. The core CLI might include a flag like `--fix` to execute safe upgrades in-place, following strategies akin to `osv-scanner fix --non-interactive`【34†L205-L213】. 

## System Architecture

**Modular layers** keep Belisarius extensible:

- **CLI Layer (Node.js):** Entry point (`lice init`, `lice fix`, etc.). Parses configs, invokes other modules, and renders output. Published via npm so `npx lice` works out-of-the-box. 
- **Scanner Engine (Node.js/TypeScript):** Builds the dependency graph (using `npm ls` or lockfile), queries OSV/dev APIs (with caching), and collects raw findings. This layer could use existing libs (e.g. `@mdx-js/node` for parsing or custom JSON parsing).
- **AI Layer (Python or Node):** A risk-scoring function that ingests scan results and code context. Initially, this might be a simple rule-based classifier (e.g. raise severity if used in `src/api/`), but can be backed by an ML model. We may call a Python microservice or use a JS ML package for LLM calls. All heavy NLP or ML (vector embeddings) can run locally or via an optional cloud API if scale demands.
- **Cache & Data:** Belisarius will locally cache API responses (e.g. in a simple JSON DB or SQLite) keyed by package@version, to achieve fast repeat scans. This cache implements TTL/updates to stay fresh.
- **Optional Cloud API:** For very large apps, a cloud service could do deeper analysis (graph heavy-lifting, model inference). In MVP, we can do everything locally, but the design allows offloading AI scoring if needed.

This architecture allows a 5-second scan: Node.js can fetch dozens of API calls in parallel, and caching means repeated runs are instant. We avoid heavy blockchain or static analysis tools (no SAST/DAST), focusing on dependencies first.

## Repository Structure (MVP)

A recommended folder layout:

```
lice/
├── bin/lice.js           # CLI entrypoint (npm bin)
├── src/
│   ├── cli.js            # command parser (commander.js or yargs)
│   ├── scanner.js        # dependency graph builder & CVE lookup
│   ├── risk.js           # risk scoring and summarization
│   ├── fix.js            # auto-fix logic (version bump)
│   ├── utils.js          # common helpers (HTTP, cache, I/O)
│   └── ai/               # (optional) Python or ML model code
├── package.json
├── README.md
└── .gitignore
```

- `bin/lice.js` is a small Node script that calls the main CLI (in `src/cli.js`). 
- The scanner might use `@npmcli/arborist` or simple file parsing to get the lockfile JSON. 
- For OSV queries, use `node-fetch` to call the REST API. Use `--json` output as needed.
- The risk scorer can be a JS function that labels severities or invokes an LLM (e.g. via OpenAI/GPT or a local ML model). 
- Tests (e.g. using Jest) should verify scanning logic on sample projects.

## Example CLI Output

After implementing the above, an example `npx lice init` run might look like this (with citations as context, not printed):

```
🎯 Belisarius Scan Report (npm project: my-app)

[CRITICAL] debug@2.6.9 → Prototype Pollution (CVE-2020-8116, affects auth module)
Fix: upgrade to debug@2.6.10

[HIGH] lodash@4.17.21 → Lodash RCE (CVE-2023-23358, used in userService)
Fix: upgrade to lodash@4.17.22

[MEDIUM] left-pad@1.3.0 → Denial of Service (CVE-2015-1596, dev dependency, non-critical)
Fix: (none required if not used in production)

[LOW] mocha@8.2.1 → Deprecated (no CVE, test framework)
Note: consider upgrading to mocha@10.x eventually.
```

Each entry is the package and version, the risk level, a short issue summary, and a fix line. This output is **prioritized** by Belisarius’s AI and filtered to minimize noise. For example, a vulnerability in an unused devDependency might be marked Low or hidden entirely. 

## Roadmap for Scaling

Beyond the MVP, Belisarius can grow into a full DevSecOps platform:

1. **GitHub Integration & Auto-PRs:** A GitHub Action or App that runs Belisarius on PRs/commits, automatically opens pull requests with dependency upgrades (like `dependabot` but smarter with AI context).
2. **Runtime Monitoring & Anomaly Detection:** Extend Belisarius to ingest runtime logs or SBOMs. Use AI to detect anomalies (e.g. an unexpected library loaded in production) and alert in real time.
3. **Multi-language Support:** Generalize beyond npm. After mastering JavaScript, add Python (pip/requirements.txt) and Go modules. Since OSV and deps.dev cover many ecosystems, the same model applies.
4. **Team Dashboard & Scorecard:** A web UI/dashboard for teams to view their project health (issues over time, compliance metrics). Possibly integrate with OpenSSF Scorecard.
5. **Continuous Learning:** Improve AI with community feedback. For example, let developers “thumbs up/down” a suggested risk rating to fine-tune the model.

Throughout, Belisarius remains **fast and unobtrusive**. The focus is *real developers fixing real problems*. By unifying scanning, prioritization, and fix advice, Belisarius cuts through the noise of traditional security tools【20†L143-L152】【37†L84-L92】. 

**Sources:** We based Belisarius’s design on state-of-art SCA research and tools. For example, deps.dev/OSV aggregation provides richer data than `npm audit`【6†L81-L89】【34†L149-L152】. Leading security guidance stresses context-aware risk scoring【20†L143-L152】【37†L84-L92】. Open-source tools like Datadog’s GuardDog【18†L317-L325】 and Araptus’s fast CLI scanner【16†L277-L285】 inspired the malicious-package detection and CLI-first UX. Google’s OSV-Scanner demonstrates how to auto-patch vulnerabilities【34†L205-L213】【23†L677-L686】. Belisarius synthesizes these ideas into one modular, AI-driven system optimized for npm developers.

