# AI Guidelines

Modular WordPress/WooCommerce coding guidelines for Claude AI.

## Why Modular?

Instead of loading 2000+ lines every time, Claude loads only what's needed:

| Task | Lines Loaded |
|------|--------------|
| Writing AJAX handler | ~390 (security only) |
| Adding caching | ~160 (performance only) |
| New plugin setup | ~320 (workflow only) |

## Installation

### Option 1: Submodule (Recommended)

Keeps guidelines synced across projects.

```bash
curl -fsSL https://raw.githubusercontent.com/mocassinis/ai-guidelines/main/setup.sh | bash
git commit -m "Add AI guidelines submodule"
```

### Option 2: Copy

One-time copy, no sync.

```bash
git clone git@github.com:mocassinis/ai-guidelines.git /tmp/ai-guidelines
cp -r /tmp/ai-guidelines/.claude /path/to/your-project/
rm -rf /tmp/ai-guidelines
```

## Structure

```
.claude/
├── CLAUDE.md                      # Entry point (always loaded)
└── guidelines/
    ├── index.md                   # Task → file mapping
    ├── 00-new-plugin-workflow.md  # Step-by-step new plugin guide
    ├── 01-technical-setup.md      # PHP, naming, plugin registry
    ├── 02-architecture.md         # Directory structure, Plugin class
    ├── 03-modern-php.md           # PHP 8.0/8.1/8.2 features
    ├── 04-security.md             # AJAX, REST, nonces, SQL
    ├── 05-woocommerce.md          # CRUD, HPOS, block checkout
    ├── 06-performance.md          # Caching, Action Scheduler
    ├── 07-javascript.md           # Vanilla JS, jQuery admin
    ├── 08-documentation.md        # PHPDoc standards
    ├── 09-testing.md              # PHPUnit, Brain Monkey
    ├── 10-static-analysis.md      # PHPStan configuration
    └── 11-checklist.md            # Pre-commit verification
```

## Usage

Claude reads `CLAUDE.md` automatically and loads specific guidelines as needed.

### Creating a New Plugin

Tell Claude:
> "Create a new WooCommerce plugin for inventory management"

Claude will read `00-new-plugin-workflow.md` and follow the 13-step process.

### Specific Tasks

| Task | Claude Reads |
|------|--------------|
| New plugin | `00-new-plugin-workflow.md` |
| PHP classes | `03-modern-php.md` |
| AJAX/REST endpoints | `04-security.md` |
| WooCommerce integration | `05-woocommerce.md` |
| Caching/async | `06-performance.md` |
| JavaScript | `07-javascript.md` |
| Writing tests | `09-testing.md` |
| Before commit | `11-checklist.md` |

## Updating

If installed as submodule:

```bash
cd .claude-guidelines
git pull origin main
cd ..
git add .claude-guidelines
git commit -m "Update AI guidelines"
```

## Standards Covered

- PHP 8.0+ with strict typing
- WordPress Coding Standards
- WooCommerce HPOS compatibility
- Security (sanitization, escaping, nonces, capabilities)
- Performance (caching, Action Scheduler, query optimization)
- Testing (PHPUnit + Brain Monkey)
- Static analysis (PHPStan level 6+)

## Author

[nevma](https://nevma.gr)
