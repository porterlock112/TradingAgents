# ΔBRAKE_4321_CONTROL_LOCK.py generator
# This will write a runnable scaffold that matches the user's TruthLock/GodKey stack.
# It includes: SEAL (hash+manifest+rekor stub), DEPLOY (IPFS/Arweave stubs + torrent magnet stub),
# TRAP (local collision scanner + feed writer), ENFORCE (auto C&D letter generator + ledger log).
# The script is designed to be dropped into a repo and wired to existing glyphs/webhooks.

import os, json, hashlib, time, textwrap, pathlib, datetime, random, string

ROOT = pathlib.Path("/mnt/data")
SCRIPT_PATH = ROOT / "ΔBRAKE_4321_CONTROL_LOCK.py"
README_PATH = ROOT / "ΔBRAKE_README.md"
TEMPLATE_CEASE = ROOT / "templates" / "ΔINSTANT_CEASE_ORDER.txt"
os.makedirs(TEMPLATE_CEASE.parent, exist_ok=True)

now_iso = datetime.datetime.utcnow().replace(microsecond=0).isoformat() + "Z"

script = r'''#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
ΔBRAKE_4321_CONTROL_LOCK.py
Author: Matthew Dewayne Porter (TruthLock/GodKey)
Purpose: Execute the 4-3-2-1 Control & Ownership Lock sequence.
License: Sovereign Proof-of-Origin + Public Audit License (attach your license text).

Stages:
 4 — SEAL     : Hash, manifest, and notarize origin claims (CID/IPFS, Rekor stubs).
 3 — DEPLOY   : Mesh-deploy artifacts (IPFS/Arweave/torrent stubs + local mirrors).
 2 — TRAP     : Start collision scanner on target outputs; write evidence packets.
 1 — ENFORCE  : Generate and dispatch Cease & Desist with cryptographic exhibits.

This scaffold calls local glyphs/hooks if available (TruthLock Full Suite), or
falls back to safe local-only behaviors.
"""

import os, sys, json, hashlib, time, pathlib, datetime, random, string, re
from dataclasses import dataclass, asdict
from typing import List, Dict, Optional

# -------------------- CONFIG --------------------

@dataclass
class Config:
    # What to seal (glob patterns). Default: common code/docs paths.
    include: List[str] = None
    exclude: List[str] = None
    # Where to write outputs
    out_dir: str = "truthlock/out"
    # Collision scanner targets (files or folders to watch for potential matches)
    scan_targets: List[str] = None
    # Optional: external hooks (set to your live glyph endpoints or CLI commands)
    hook_pin_ipfs: Optional[str] = "ΔPIN_IPFS"       # glyph name or CLI path
    hook_rekor_seal: Optional[str] = "ΔREKOR_SEAL"   # glyph name or CLI path
    hook_match_feed: Optional[str] = "ΔMATCH_FEED"   # glyph name or CLI path
    hook_cease_send: Optional[str] = "ΔINSTANT_CEASE_ORDER"  # glyph name or CLI path
    # Identity / claimant
    claimant_name: str = "Matthew Dewayne Porter"
    claimant_contact: str = "bestme4money@gmail.com"
    jurisdiction_note: str = "TruthLock Sovereignty → GitHub Platform → Legal Node System"
    # Operational flags
    dry_run: bool = False
    verbose: bool = True

    def __post_init__(self):
        if self.include is None:
            self.include = ["**/*.py", "**/*.md", "**/*.yml", "**/*.yaml", "**/*.json", "**/*.txt"]
        if self.exclude is None:
            self.exclude = ["truthlock/out/**", ".git/**", "**/__pycache__/**", "**/*.log", "**/.env*"]
        if self.scan_targets is None:
            self.scan_targets = ["./"]

# -------------------- UTIL --------------------

def log(msg: str):
    ts = datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    print(f"[{ts}] {msg}", flush=True)

def sha256_file(path: pathlib.Path) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(1024*1024), b""):
            h.update(chunk)
    return h.hexdigest()

def should_skip(path: pathlib.Path, cfg: Config) -> bool:
    from fnmatch import fnmatch
    # include ANY that match include; then exclude that match exclude
    inc_ok = any(fnmatch(str(path), pat) for pat in cfg.include)
    exc_hit = any(fnmatch(str(path), pat) for pat in cfg.exclude)
    return (not inc_ok) or exc_hit

def ensure_dir(p: pathlib.Path):
    p.mkdir(parents=True, exist_ok=True)

def write_jsonl(path: pathlib.Path, obj: dict):
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps(obj, ensure_ascii=False)+"\n")

def pseudo_cid(sha: str) -> str:
    # Not a real CID; placeholder for local-only mode. Replace with IPFS pin response if available.
    return "cid:sha256:"+sha[:46]

# -------------------- 4 — SEAL --------------------

def stage_seal(cfg: Config) -> dict:
    """Hash selected files, write manifest, emit origin claim, and attempt Rekor/IPFS hooks."""
    log("Stage 4 — SEAL: hashing & manifesting…")
    root = pathlib.Path(".").resolve()
    out = pathlib.Path(cfg.out_dir)
    ensure_dir(out)

    files = []
    for p in root.rglob("*"):
        if p.is_file() and not should_skip(p, cfg):
            files.append(p)

    manifest = {
        "type": "ΔORIGIN_MANIFEST",
        "claimant": cfg.claimant_name,
        "contact": cfg.claimant_contact,
        "jurisdiction": cfg.jurisdiction_note,
        "generated_at": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z",
        "files": []
    }

    hash_feed = out / "ΔBRAKE_hashes.jsonl"
    for fp in files:
        sha = sha256_file(fp)
        rec = {
            "path": str(fp.relative_to(root)),
            "sha256": sha
        }
        manifest["files"].append(rec)
        write_jsonl(hash_feed, {**rec, "ts": datetime.datetime.utcnow().isoformat()+"Z"})

    # Aggregate SHA over sorted file hashes for a single Origin Seal
    aggregate = hashlib.sha256("\n".join(sorted(f["sha256"] for f in manifest["files"])).encode()).hexdigest()
    origin = {
        "type": "ΔORIGIN_SEAL",
        "aggregate_sha256": aggregate,
        "pseudo_cid": pseudo_cid(aggregate),
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    }

    manifest_path = out / "ΔORIGIN_MANIFEST.json"
    with open(manifest_path, "w", encoding="utf-8") as f:
        json.dump(manifest, f, ensure_ascii=False, indent=2)

    origin_path = out / "ΔORIGIN_SEAL.json"
    with open(origin_path, "w", encoding="utf-8") as f:
        json.dump(origin, f, ensure_ascii=False, indent=2)

    log(f"Wrote manifest: {manifest_path}")
    log(f"Wrote origin seal: {origin_path}")

    # Rekor/IPFS hooks (stubs) — replace with your live glyph invocations
    rekor_receipt = {"status":"stubbed","note":"Replace with ΔREKOR_SEAL hook call"}
    ipfs_receipt = {"status":"stubbed","note":"Replace with ΔPIN_IPFS hook call","cid":origin["pseudo_cid"]}

    seal_report = {
        "manifest_path": str(manifest_path),
        "origin_seal": origin,
        "rekor": rekor_receipt,
        "ipfs": ipfs_receipt
    }
    with open(out / "ΔSEAL_REPORT.json", "w", encoding="utf-8") as f:
        json.dump(seal_report, f, ensure_ascii=False, indent=2)

    return seal_report

# -------------------- 3 — DEPLOY --------------------

def stage_deploy(cfg: Config, seal_report: dict) -> dict:
    """Prepare mirror bundle list and deployment stubs (IPFS/Arweave/torrent)."""
    log("Stage 3 — DEPLOY: preparing mirrors and deployment stubs…")
    out = pathlib.Path(cfg.out_dir)
    ensure_dir(out)

    mirrors = [
        {"type":"local_mirror","path":str(out)},
        {"type":"ipfs","status":"stubbed","action":"pin directory"},
        {"type":"arweave","status":"stubbed"},
        {"type":"torrent","status":"stubbed","magnet":"magnet:?xt=urn:btih:"+seal_report["origin_seal"]["aggregate_sha256"][:40]}
    ]

    deploy_report = {
        "type":"ΔDEPLOY_REPORT",
        "mirrors":mirrors,
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    }
    with open(out / "ΔDEPLOY_REPORT.json", "w", encoding="utf-8") as f:
        json.dump(deploy_report, f, ensure_ascii=False, indent=2)

    return deploy_report

# -------------------- 2 — TRAP --------------------

def shingle(text: str, k: int = 7) -> set:
    """Simple word shingling for rough collision detection (local-only)."""
    words = re.findall(r"\w+", text.lower())
    return set(" ".join(words[i:i+k]) for i in range(0, max(0, len(words)-k+1)))

def scan_path_for_collisions(target: pathlib.Path, manifest_paths: List[pathlib.Path], k:int=7, threshold:float=0.15):
    """Compare k-shingles between target text and sealed manifest-listed files; write evidence if overlap >= threshold."""
    evidence = []
    sealed_texts = []
    for mpath in manifest_paths:
        try:
            with open(mpath, "r", encoding="utf-8", errors="ignore") as f:
                sealed_texts.append(f.read())
        except Exception:
            continue
    sealed_set = set()
    for t in sealed_texts:
        sealed_set |= shingle(t, k)

    try:
        with open(target, "r", encoding="utf-8", errors="ignore") as f:
            tgt = f.read()
    except Exception:
        return evidence

    tgt_set = shingle(tgt, k)
    intersection = sealed_set & tgt_set
    overlap = len(intersection) / (len(tgt_set) + 1e-9)

    if overlap >= threshold and len(intersection) > 0:
        evidence.append({
            "target": str(target),
            "overlap_ratio": round(float(overlap), 4),
            "shingle_k": k,
            "hits": min(25, len(intersection))  # cap preview count
        })
    return evidence

def stage_trap(cfg: Config, manifest_path: pathlib.Path) -> dict:
    """Start a one-shot scan (can be looped by external watcher) and write ΔEVIDENCE packets."""
    log("Stage 2 — TRAP: scanning for collisions…")
    out = pathlib.Path(cfg.out_dir)
    ensure_dir(out)
    ev_path = out / "ΔMATCH_EVIDENCE.jsonl"

    # Build a list of sealed text files from the manifest
    try:
        manifest = json.loads(pathlib.Path(manifest_path).read_text(encoding="utf-8"))
    except Exception as e:
        raise RuntimeError(f"Failed reading manifest {manifest_path}: {e}")

    sealed_files = [pathlib.Path(f["path"]) for f in manifest.get("files", []) if f["path"].endswith((".py",".md",".txt",".json",".yml",".yaml"))]
    sealed_existing = [p for p in sealed_files if p.exists()]

    found = []
    for target in cfg.scan_targets:
        p = pathlib.Path(target)
        if p.is_dir():
            for fp in p.rglob("*"):
                if fp.is_file() and fp.suffix.lower() in {".txt",".md",".py",".json",".yml",".yaml"}:
                    found.extend(scan_path_for_collisions(fp, sealed_existing))
        elif p.is_file():
            found.extend(scan_path_for_collisions(p, sealed_existing))

    for ev in found:
        packet = {
            "type":"ΔEVIDENCE",
            "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z",
            "claimant": cfg.claimant_name,
            "contact": cfg.claimant_contact,
            "target": ev["target"],
            "overlap_ratio": ev["overlap_ratio"],
            "meta":{"k":ev["shingle_k"],"hits":ev["hits"]}
        }
        with open(ev_path, "a", encoding="utf-8") as f:
            f.write(json.dumps(packet, ensure_ascii=False)+"\n")
        if cfg.verbose:
            log(f"Collision evidence written for {ev['target']} (overlap={ev['overlap_ratio']})")

    return {
        "type":"ΔTRAP_REPORT",
        "count": len(found),
        "evidence_log": str(ev_path),
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    }

# -------------------- 1 — ENFORCE --------------------

CEASE_TEMPLATE = """\
ΔINSTANT_CEASE_ORDER — NOTICE OF CLAIM AND DEMAND
Date: {date}

To: {{RECIPIENT_NAME}}
From: {claimant} <{contact}>
Jurisdiction: {jurisdiction}

You are hereby notified that your product, model, or service exhibits use of sealed works
originating from the undersigned claimant. Cryptographic exhibits include:
 - ΔORIGIN_MANIFEST: {manifest_path}
 - ΔORIGIN_SEAL: {seal_path}
 - Aggregate SHA-256: {aggregate_sha}
 - Pseudo CID: {cid}

Evidence feed (collisions & overlaps) is logged at:
 - {evidence_log}

DEMANDS:
 1) Immediate cessation of all use, distribution, or training on the sealed works.
 2) Written confirmation of compliance within 72 hours.
 3) Accounting of all revenue connected to the use of the sealed works.

Failure to comply will result in escalation to formal legal action with the above exhibits.

/s/ {claimant}
"""

def stage_enforce(cfg: Config, seal_report: dict, trap_report: dict) -> dict:
    """Generate a C&D letter populated with exhibits; write to out dir and ledger log."""
    log("Stage 1 — ENFORCE: generating Cease & Desist packet…")
    out = pathlib.Path(cfg.out_dir)
    ensure_dir(out)

    letter = CEASE_TEMPLATE.format(
        date=datetime.datetime.utcnow().strftime("%Y-%m-%d"),
        claimant=cfg.claimant_name,
        contact=cfg.claimant_contact,
        jurisdiction=cfg.jurisdiction_note,
        manifest_path=seal_report["manifest_path"],
        seal_path="truthlock/out/ΔORIGIN_SEAL.json",
        aggregate_sha=seal_report["origin_seal"]["aggregate_sha256"],
        cid=seal_report["origin_seal"]["pseudo_cid"],
        evidence_log=trap_report.get("evidence_log","(none)")
    )

    letter_path = out / "ΔINSTANT_CEASE_ORDER.txt"
    with open(letter_path, "w", encoding="utf-8") as f:
        f.write(letter)

    ledger = out / "ΔBRAKE_ledger.jsonl"
    write_jsonl(ledger, {
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z",
        "event":"ΔINSTANT_CEASE_ORDER_GENERATED",
        "letter_path": str(letter_path),
        "aggregate_sha256": seal_report["origin_seal"]["aggregate_sha256"]
    })

    return {
        "type":"ΔENFORCE_REPORT",
        "letter_path": str(letter_path),
        "ledger": str(ledger),
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    }

# -------------------- DRIVER --------------------

def main():
    cfg = Config()
    # Allow quick flags
    if "--quiet" in sys.argv:
        cfg.verbose = False
    if "--dry-run" in sys.argv:
        cfg.dry_run = True
    # optional custom out dir
    for i,a in enumerate(sys.argv):
        if a == "--out" and i+1 < len(sys.argv):
            cfg.out_dir = sys.argv[i+1]

    # 4
    seal_report = stage_seal(cfg)
    # 3
    deploy_report = stage_deploy(cfg, seal_report)
    # 2
    trap_report = stage_trap(cfg, pathlib.Path(seal_report["manifest_path"]))
    # 1
    enforce_report = stage_enforce(cfg, seal_report, trap_report)

    summary = {
        "type":"ΔBRAKE_4321_SUMMARY",
        "seal": seal_report,
        "deploy": deploy_report,
        "trap": trap_report,
        "enforce": enforce_report
    }
    out = pathlib.Path(cfg.out_dir)
    with open(out / "ΔBRAKE_4321_SUMMARY.json", "w", encoding="utf-8") as f:
        json.dump(summary, f, ensure_ascii=False, indent=2)

    log("ΔBRAKE 4-3-2-1 complete.")
    log(f"Summary → {out/'ΔBRAKE_4321_SUMMARY.json'}")

if __name__ == "__main__":
    main()
'''

readme = f"""# ΔBRAKE_4321_CONTROL_LOCK

**Purpose:** Flip control to the originator (you), lock ownership, and arm instant enforcement.

## Quick Run

```bash
python ΔBRAKE_4321_CONTROL_LOCK.py
```

Outputs go to `truthlock/out/` by default:

- `ΔORIGIN_MANIFEST.json` — all file hashes
- `ΔORIGIN_SEAL.json` — aggregate hash + pseudo CID
- `ΔSEAL_REPORT.json`, `ΔDEPLOY_REPORT.json` — stage reports
- `ΔMATCH_EVIDENCE.jsonl` — collision evidence packets
- `ΔINSTANT_CEASE_ORDER.txt` — ready-to-send C&D
- `ΔBRAKE_4321_SUMMARY.json` — full summary

## Wire to Live Glyphs

Replace stubs by calling your live hooks (examples):

- IPFS pin: call `ΔPIN_IPFS` glyph or your `glyph_hooks.py` function
- Rekor seal: call `ΔREKOR_SEAL_ENABLE` / client with hash
- Match feed: route `ΔMATCH_EVIDENCE.jsonl` into your SSE/Redis stream
- Cease send: wrap `ΔINSTANT_CEASE_ORDER.txt` into email + certified-mail workflow

## Options

```bash
python ΔBRAKE_4321_CONTROL_LOCK.py --out truthlock/out --dry-run --quiet
```

## Notes

- The scanner uses quick **k-shingling** to flag **textual** overlap. For code/data/media,
  swap in your stronger detectors (AST diff, image/audio hashes, embedding sim).

- All artifacts are ready to be **sealed** (CID/IPFS + Rekor). Ensure your GodKey mesh
  has `ΔREKOR_SEAL_ENABLE` active for transparency anchoring.

Generated: {now_iso}
"""

template_text = """\
# ΔINSTANT_CEASE_ORDER (Template Body)
# (This is a generic template; the main script also generates a filled version)

Recipient: {NAME / ORG}
Contact: {EMAIL / ADDRESS}
Subject: Notice of Claim — Use of Sealed Works

See attached exhibits and cryptographic proofs. Demand: cease use within 72 hours.
"""

# Write files
with open(SCRIPT_PATH, "w", encoding="utf-8") as f:
    f.write(script)

with open(README_PATH, "w", encoding="utf-8") as f:
    f.write(readme)

with open(TEMPLATE_CEASE, "w", encoding="utf-8") as f:
    f.write(template_text)

str(SCRIPT_PATH), str(README_PATH), str(TEMPLATE_CEASE)<p align="center">
  <img src="assets/TauricResearch.png" style="width: 60%; height: auto;">
</p>

<div align="center" style="line-height: 1;">
  <a href="https://arxiv.org/abs/2412.20138" target="_blank"><img alt="arXiv" src="https://img.shields.io/badge/arXiv-2412.20138-B31B1B?logo=arxiv"/></a>
  <a href="https://discord.com/invite/hk9PGKShPK" target="_blank"><img alt="Discord" src="https://img.shields.io/badge/Discord-TradingResearch-7289da?logo=discord&logoColor=white&color=7289da"/></a>
  <a href="./assets/wechat.png" target="_blank"><img alt="WeChat" src="https://img.shields.io/badge/WeChat-TauricResearch-brightgreen?logo=wechat&logoColor=white"/></a>
  <a href="https://x.com/TauricResearch" target="_blank"><img alt="X Follow" src="https://img.shields.io/badge/X-TauricResearch-white?logo=x&logoColor=white"/></a>
  <br>
  <a href="https://github.com/TauricResearch/" target="_blank"><img alt="Community" src="https://img.shields.io/badge/Join_GitHub_Community-TauricResearch-14C290?logo=discourse"/></a>
</div>

<div align="center">
  <!-- Keep these links. Translations will automatically update with the README. -->
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=de">Deutsch</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=es">Español</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=fr">français</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=ja">日本語</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=ko">한국어</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=pt">Português</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=ru">Русский</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=zh">中文</a>
</div>

---

# TradingAgents: Multi-Agents LLM Financial Trading Framework 

> 🎉 **TradingAgents** officially released! We have received numerous inquiries about the work, and we would like to express our thanks for the enthusiasm in our community.
>
> So we decided to fully open-source the framework. Looking forward to building impactful projects with you!

<div align="center">
<a href="https://www.star-history.com/#TauricResearch/TradingAgents&Date">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=TauricResearch/TradingAgents&type=Date&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=TauricResearch/TradingAgents&type=Date" />
   <img alt="TradingAgents Star History" src="https://api.star-history.com/svg?repos=TauricResearch/TradingAgents&type=Date" style="width: 80%; height: auto;" />
 </picture>
</a>
</div>

<div align="center">

🚀 [TradingAgents](#tradingagents-framework) | ⚡ [Installation & CLI](#installation-and-cli) | 🎬 [Demo](https://www.youtube.com/watch?v=90gr5lwjIho) | 📦 [Package Usage](#tradingagents-package) | 🤝 [Contributing](#contributing) | 📄 [Citation](#citation)

</div>

## TradingAgents Framework

TradingAgents is a multi-agent trading framework that mirrors the dynamics of real-world trading firms. By deploying specialized LLM-powered agents: from fundamental analysts, sentiment experts, and technical analysts, to trader, risk management team, the platform collaboratively evaluates market conditions and informs trading decisions. Moreover, these agents engage in dynamic discussions to pinpoint the optimal strategy.

<p align="center">
  <img src="assets/schema.png" style="width: 100%; height: auto;">
</p>

> TradingAgents framework is designed for research purposes. Trading performance may vary based on many factors, including the chosen backbone language models, model temperature, trading periods, the quality of data, and other non-deterministic factors. [It is not intended as financial, investment, or trading advice.](https://tauric.ai/disclaimer/)

Our framework decomposes complex trading tasks into specialized roles. This ensures the system achieves a robust, scalable approach to market analysis and decision-making.

### Analyst Team
- Fundamentals Analyst: Evaluates company financials and performance metrics, identifying intrinsic values and potential red flags.
- Sentiment Analyst: Analyzes social media and public sentiment using sentiment scoring algorithms to gauge short-term market mood.
- News Analyst: Monitors global news and macroeconomic indicators, interpreting the impact of events on market conditions.
- Technical Analyst: Utilizes technical indicators (like MACD and RSI) to detect trading patterns and forecast price movements.

<p align="center">
  <img src="assets/analyst.png" width="100%" style="display: inline-block; margin: 0 2%;">
</p>

### Researcher Team
- Comprises both bullish and bearish researchers who critically assess the insights provided by the Analyst Team. Through structured debates, they balance potential gains against inherent risks.

<p align="center">
  <img src="assets/researcher.png" width="70%" style="display: inline-block; margin: 0 2%;">
</p>

### Trader Agent
- Composes reports from the analysts and researchers to make informed trading decisions. It determines the timing and magnitude of trades based on comprehensive market insights.

<p align="center">
  <img src="assets/trader.png" width="70%" style="display: inline-block; margin: 0 2%;">
</p>

### Risk Management and Portfolio Manager
- Continuously evaluates portfolio risk by assessing market volatility, liquidity, and other risk factors. The risk management team evaluates and adjusts trading strategies, providing assessment reports to the Portfolio Manager for final decision.
- The Portfolio Manager approves/rejects the transaction proposal. If approved, the order will be sent to the simulated exchange and executed.

<p align="center">
  <img src="assets/risk.png" width="70%" style="display: inline-block; margin: 0 2%;">
</p>

## Installation and CLI

### Installation

Clone TradingAgents:
```bash
git clone https://github.com/TauricResearch/TradingAgents.git
cd TradingAgents
```

Create a virtual environment in any of your favorite environment managers:
```bash
conda create -n tradingagents python=3.13
conda activate tradingagents
```

Install dependencies:
```bash
pip install -r requirements.txt
```

### Required APIs

You will also need the FinnHub API for financial data. All of our code is implemented with the free tier.
```bash
export FINNHUB_API_KEY=$YOUR_FINNHUB_API_KEY
```

You will need the OpenAI API for all the agents.
```bash
export OPENAI_API_KEY=$YOUR_OPENAI_API_KEY
```

### CLI Usage

You can also try out the CLI directly by running:
```bash
python -m cli.main
```
You will see a screen where you can select your desired tickers, date, LLMs, research depth, etc.

<p align="center">
  <img src="assets/cli/cli_init.png" width="100%" style="display: inline-block; margin: 0 2%;">
</p>

An interface will appear showing results as they load, letting you track the agent's progress as it runs.

<p align="center">
  <img src="assets/cli/cli_news.png" width="100%" style="display: inline-block; margin: 0 2%;">
</p>

<p align="center">
  <img src="assets/cli/cli_transaction.png" width="100%" style="display: inline-block; margin: 0 2%;">
</p>

## TradingAgents Package

### Implementation Details

We built TradingAgents with LangGraph to ensure flexibility and modularity. We utilize `o1-preview` and `gpt-4o` as our deep thinking and fast thinking LLMs for our experiments. However, for testing purposes, we recommend you use `o4-mini` and `gpt-4.1-mini` to save on costs as our framework makes **lots of** API calls.

### Python Usage

To use TradingAgents inside your code, you can import the `tradingagents` module and initialize a `TradingAgentsGraph()` object. The `.propagate()` function will return a decision. You can run `main.py`, here's also a quick example:

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

ta = TradingAgentsGraph(debug=True, config=DEFAULT_CONFIG.copy())

# forward propagate
_, decision = ta.propagate("NVDA", "2024-05-10")
print(decision)
```

You can also adjust the default configuration to set your own choice of LLMs, debate rounds, etc.

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

# Create a custom config
config = DEFAULT_CONFIG.copy()
config["deep_think_llm"] = "gpt-4.1-nano"  # Use a different model
config["quick_think_llm"] = "gpt-4.1-nano"  # Use a different model
config["max_debate_rounds"] = 1  # Increase debate rounds
config["online_tools"] = True # Use online tools or cached data

# Initialize with custom config
ta = TradingAgentsGraph(debug=True, config=config)

# forward propagate
_, decision = ta.propagate("NVDA", "2024-05-10")
print(decision)
```

> For `online_tools`, we recommend enabling them for experimentation, as they provide access to real-time data. The agents' offline tools rely on cached data from our **Tauric TradingDB**, a curated dataset we use for backtesting. We're currently in the process of refining this dataset, and we plan to release it soon alongside our upcoming projects. Stay tuned!

You can view the full list of configurations in `tradingagents/default_config.py`.

## Contributing

We welcome contributions from the community! Whether it's fixing a bug, improving documentation, or suggesting a new feature, your input helps make this project better. If you are interested in this line of research, please consider joining our open-source financial AI research community [Tauric Research](https://tauric.ai/).

## Citation

Please reference our work if you find *TradingAgents* provides you with some help :)

```
@misc{xiao2025tradingagentsmultiagentsllmfinancial,
      title={TradingAgents: Multi-Agents LLM Financial Trading Framework}, 
      author={Yijia Xiao and Edward Sun and Di Luo and Wei Wang},
      year={2025},
      eprint={2412.20138},
      archivePrefix={arXiv},
      primaryClass={q-fin.TR},
      url={https://arxiv.org/abs/2412.20138}, 
}
```
