# Nevma WordPress/WooCommerce Guidelines for Claude

**Production-grade coding standards for Claude Code — AI-assisted WordPress & WooCommerce development.**

Created by [nevma](https://nevma.gr) — a digital agency specializing in WordPress, WooCommerce, and custom web solutions.

## What Is This?

A set of modular guidelines that Claude Code reads automatically when working on WordPress/WooCommerce projects. Instead of explaining your coding standards every time, Claude follows these rules by default.

### Built for Claude Code

These guidelines are designed specifically for [Claude Code](https://claude.ai/claude-code):

- **Modular loading** — Claude loads only what's needed per task
- **Specialized agents** — Security, performance, and testing agents trigger automatically
- **Structured workflows** — 13-step plugin creation, pre-commit checklists

### Modular Loading

Instead of loading 2000+ lines every time, Claude loads only what's needed:

| Task | Lines Loaded |
|------|--------------|
| Writing AJAX handler | ~390 (security only) |
| WooCommerce integration | ~570 (WooCommerce only) |
| Adding caching | ~410 (performance only) |
| New plugin setup | ~320 (workflow only) |

## Installation

### Option 1: Submodule (Recommended)

Keeps guidelines synced across projects.

```bash
# First time: clone locally
git clone git@github.com:mocassinis/nevma-wp-guidelines.git ~/.nevma-wp-guidelines

# Run on any project
~/.nevma-wp-guidelines/setup.sh /path/to/your-project
cd /path/to/your-project
git commit -m "Add nevma-wp-guidelines submodule"
```

### Option 2: Manual Submodule

```bash
cd /path/to/your-project
git submodule add git@github.com:mocassinis/nevma-wp-guidelines.git .claude-guidelines
ln -s .claude-guidelines/.claude .claude
git add .claude
git commit -m "Add nevma-wp-guidelines submodule"
```

### Option 3: Copy

One-time copy, no sync.

```bash
git clone git@github.com:mocassinis/nevma-wp-guidelines.git /tmp/nevma-wp-guidelines
cp -r /tmp/nevma-wp-guidelines/.claude /path/to/your-project/
rm -rf /tmp/nevma-wp-guidelines
```

## Structure

```
.claude/
├── CLAUDE.md                      # Entry point (Claude reads this first)
└── guidelines/
    ├── index.md                   # Task → file mapping
    ├── 00-new-plugin-workflow.md  # Step-by-step new plugin guide
    ├── 01-technical-setup.md      # PHP, naming, plugin registry
    ├── 02-architecture.md         # Directory structure, Plugin class
    ├── 03-modern-php.md           # PHP 8.0/8.1/8.2 features
    ├── 04-security.md             # AJAX, REST, nonces, SQL
    ├── 05-woocommerce.md          # CRUD, HPOS, gateways, shipping, emails
    ├── 06-performance.md          # Caching, Action Scheduler, WC perf
    ├── 07-javascript.md           # Vanilla JS, jQuery admin
    ├── 08-documentation.md        # PHPDoc standards
    ├── 09-testing.md              # PHPUnit, Brain Monkey, TDD
    ├── 10-static-analysis.md      # PHPStan configuration
    ├── 11-checklist.md            # Pre-commit verification
    ├── 12-advanced-patterns.md    # DTOs, RBAC, CLI, middleware
    ├── 13-automation-tooling.md   # PHP-Scoper, auto-fixers
    ├── 14-e2e-testing.md          # Playwright E2E for WooCommerce
    ├── 15-interactivity-api.md    # WordPress Interactivity API
    └── 16-playground.md           # WordPress Playground CLI
```

## Usage

Claude reads `CLAUDE.md` automatically when you open a project with these guidelines.

### Creating a New Plugin

Tell Claude:
> "Create a new WooCommerce plugin for inventory management"

Claude will read `00-new-plugin-workflow.md` and follow the 13-step process.

### Task Reference

| Task | Claude Reads |
|------|--------------|
| New plugin | `00-new-plugin-workflow.md` |
| PHP classes | `03-modern-php.md` |
| AJAX/REST endpoints | `04-security.md` |
| WooCommerce integration | `05-woocommerce.md` |
| Caching/async | `06-performance.md` |
| JavaScript | `07-javascript.md` |
| Writing tests | `09-testing.md` |
| Advanced patterns | `12-advanced-patterns.md` |
| E2E testing | `14-e2e-testing.md` |
| Block interactivity | `15-interactivity-api.md` |
| Local testing | `16-playground.md` |
| Before commit | `11-checklist.md` |

## Mandatory Testing

Every plugin built with these guidelines enforces testing before commit:

| Test Type | Tool | When |
|-----------|------|------|
| Unit Tests | PHPUnit + Brain Monkey | After writing/modifying any business logic |
| E2E Tests | Playwright | After completing user-facing features |
| Static Analysis | PHPStan level 6+ | Before every commit |

```
Write Code → Unit Tests → PHPStan → E2E Tests → Security Audit → Performance Review → Commit
```

## Specialized Agents

Three Claude agents automatically review code:

| Agent | Triggers On | Reviews |
|-------|-------------|---------|
| Security Auditor | AJAX, REST, forms, SQL | Nonces, capabilities, sanitization |
| Performance Optimizer | DB queries, loops, reports | Caching, batching, Action Scheduler |
| Unit Test Writer | Services, handlers | PHPUnit + Brain Monkey patterns |

## Standards Covered

- PHP 8.0+ with `declare(strict_types=1)`
- WordPress Coding Standards (WPCS)
- WooCommerce HPOS compatibility
- Security (sanitization, escaping, nonces, capabilities)
- Performance (caching, Action Scheduler, query limits)
- Testing (PHPUnit + Brain Monkey + TDD)
- Static analysis (PHPStan level 6+)
- Dependency scoping (PHP-Scoper)
- WordPress Interactivity API (data-wp-* directives)
- WordPress Playground for local testing

## Updating

If installed as submodule:

```bash
cd .claude-guidelines
git pull origin main
cd ..
git add .claude-guidelines
git commit -m "Update nevma-wp-guidelines"
```

## About Nevma

[Nevma](https://nevma.gr) is a digital agency based in Athens, Greece, with 15+ years of experience building WordPress and WooCommerce solutions.

**Services:**
- Custom WordPress plugin development
- WooCommerce store optimization
- Enterprise WordPress solutions
- Performance audits

**Contact:** [nevma.gr](https://www.nevma.gr/contact/)

## Contributors

- [Ioannis Kastorinis](https://github.com/mocassinis) — [themoca.eu](https://www.themoca.eu/)
