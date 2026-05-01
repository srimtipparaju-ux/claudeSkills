# claudeSkills

A personal library of Claude Code skills — reusable AI capabilities that extend Claude with domain-specific expertise in performance analysis and diagnostics.

## What are Skills?

Skills are Markdown instruction files that teach Claude how to handle specialized tasks. Claude Code automatically loads them from the `.claude/skills/` directory. When you ask Claude something that matches a skill's description, it reads the skill and follows its step-by-step instructions — like SOPs for Claude.

---

## Skills in this repo

### Oracle Database Performance

| Skill | Input | What it does |
|---|---|---|
| [awr-analysis](./awr-analysis/) | Upload AWR `.html` or `.txt` | Analyzes wait events, top SQL, memory, I/O. Severity-ranked findings + Word report. |
| [sql-monitor-analysis](./sql-monitor-analysis/) | Upload SQL Monitoring `.html` or paste plan | Step-by-step execution plan analysis — E-Rows vs A-Rows, full scans, bad joins, predicate issues. |
| [sql-tuning](./sql-tuning/) | Paste any SQL text (plan optional) | Scans SQL construction for anti-patterns — Cartesian joins, non-SARGable predicates, correlated subqueries, type mismatches. Rewrites bad sections. |

### JVM / Java Performance

| Skill | Input | What it does |
|---|---|---|
| [thread-dump-analysis](./thread-dump-analysis/) | Upload `.txt`/`.log` or paste jstack output | Finds deadlocks, lock contention, thread pool exhaustion, CPU-hot threads. |
| [heap-dump-analysis](./heap-dump-analysis/) | Upload MAT/VisualVM report `.html`/`.txt` | Identifies memory leaks, largest heap consumers, GC root chains, OOM root cause. |
| [jfr-analysis](./jfr-analysis/) | Upload `jfr print` text export or JMC report | CPU hotspots at method level, GC analysis, lock contention, I/O waits, allocation profiling. |

### Application & Frontend

| Skill | Input | What it does |
|---|---|---|
| [ui-console-analysis](./ui-console-analysis/) | Upload `.har`, Lighthouse report, or paste console output | Diagnoses JS errors, slow API calls, LCP/CLS/INP issues, CORS, security headers. |
| [stack-trace-analysis](./stack-trace-analysis/) | Upload log file or paste exception | Root cause analysis for any language — Java, Python, Node.js, .NET. Explains full cause chain. |

---

## Usage Examples

| What you say to Claude | Skill triggered |
|---|---|
| "Analyze this AWR report" + upload file | `awr-analysis` |
| "Why is this SQL slow?" + upload SQL Monitor report | `sql-monitor-analysis` |
| "Tune this query" + paste SQL | `sql-tuning` |
| "Review my SQL, here's the execution plan too" | `sql-tuning` + plan analysis |
| "App is hanging, here's the thread dump" | `thread-dump-analysis` |
| "OutOfMemoryError in prod, here's the MAT report" | `heap-dump-analysis` |
| "CPU spiked, I recorded a JFR" | `jfr-analysis` |
| "Browser console showing errors, page is slow" | `ui-console-analysis` |
| "Getting this exception" + paste stack trace | `stack-trace-analysis` |

---

## Installation

### Prerequisite: Clone this repo

```bash
git clone https://github.com/YOUR_USERNAME/claudeSkills.git ~/claudeSkills
```

### Install globally into Claude Code

```bash
mkdir -p ~/.claude/skills

ln -s ~/claudeSkills/awr-analysis          ~/.claude/skills/awr-analysis
ln -s ~/claudeSkills/sql-monitor-analysis  ~/.claude/skills/sql-monitor-analysis
ln -s ~/claudeSkills/sql-tuning            ~/.claude/skills/sql-tuning
ln -s ~/claudeSkills/thread-dump-analysis  ~/.claude/skills/thread-dump-analysis
ln -s ~/claudeSkills/heap-dump-analysis    ~/.claude/skills/heap-dump-analysis
ln -s ~/claudeSkills/jfr-analysis          ~/.claude/skills/jfr-analysis
ln -s ~/claudeSkills/ui-console-analysis   ~/.claude/skills/ui-console-analysis
ln -s ~/claudeSkills/stack-trace-analysis  ~/.claude/skills/stack-trace-analysis
```

Verify — open Claude Code and run `/skills`. All 8 should be listed.

### Keep skills updated

```bash
cd ~/claudeSkills && git pull
# Symlinks mean this takes effect immediately — no reinstall needed.
```

---

## Repository Structure

```
claudeSkills/
├── README.md
├── SKILL_TEMPLATE.md
├── SETUP_INSTRUCTIONS.txt
│
├── awr-analysis/                        Oracle AWR report analysis
│   ├── SKILL.md
│   └── references/common-problems.md
│
├── sql-monitor-analysis/                Oracle SQL execution plan analysis
│   ├── SKILL.md
│   └── references/
│       ├── plan-operations.md
│       └── index-design.md
│
├── sql-tuning/                          SQL text review & anti-pattern detection
│   ├── SKILL.md
│   └── references/
│       ├── antipatterns.md
│       └── rewrite-patterns.md
│
├── thread-dump-analysis/                Java thread dump analysis
│   ├── SKILL.md
│   └── references/common-patterns.md
│
├── heap-dump-analysis/                  Java heap dump / OOM analysis
│   └── SKILL.md
│
├── jfr-analysis/                        Java Flight Recorder analysis
│   ├── SKILL.md
│   └── references/jvm-flags.md
│
├── ui-console-analysis/                 Browser console / network / HAR analysis
│   └── SKILL.md
│
└── stack-trace-analysis/                Stack trace / exception analysis
    └── SKILL.md
```

---

## Adding New Skills

```bash
mkdir -p ~/claudeSkills/my-new-skill/references
cp ~/claudeSkills/SKILL_TEMPLATE.md ~/claudeSkills/my-new-skill/SKILL.md
# Edit SKILL.md — fill in name, description, and analysis steps
ln -s ~/claudeSkills/my-new-skill ~/.claude/skills/my-new-skill
cd ~/claudeSkills && git add . && git commit -m "Add my-new-skill" && git push
```

See [SKILL_TEMPLATE.md](./SKILL_TEMPLATE.md) for the standard structure.

---

## License

MIT — use and adapt freely.
