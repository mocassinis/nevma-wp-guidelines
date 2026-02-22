# Gemini CLI Context: WordPress/WooCommerce AI Guidelines

This directory contains modular, high-quality coding standards and workflows for AI assistants to follow when generating code for WordPress and WooCommerce projects.

## Directory Overview

The project is a collection of Markdown-based guidelines designed to be loaded by an AI based on the current task. It ensures consistency, security, and performance across multiple plugins and themes.

### Key Files

*   **`README.md`**: Overview of the modular approach, installation instructions (submodule/copy), and list of standards covered.
*   **`.claude/CLAUDE.md`**: The primary entry point. It contains core rules (PHP 8.0+, strict types, namespacing) and descriptions of specialized audit roles.
*   **`.claude/guidelines/index.md`**: A task-to-file mapping index to help the AI identify which guideline to read for a specific job.
*   **`setup.sh`**: A shell script to automate the process of linking these guidelines into a target WordPress project.

## Guidelines Reference

The guidelines are stored in `.claude/guidelines/` and should be consulted before starting relevant tasks:

| Task / Topic | Guideline File |
| :--- | :--- |
| **New Plugin Walkthrough** | `00-new-plugin-workflow.md` |
| **Project Architecture** | `02-architecture.md` |
| **Modern PHP (8.0+)** | `03-modern-php.md` |
| **Security (AJAX, REST, SQL)** | `04-security.md` |
| **WooCommerce Integration** | `05-woocommerce.md` |
| **Performance & Caching** | `06-performance.md` |
| **JavaScript (Vanilla & jQuery)** | `07-javascript.md` |
| **Testing (PHPUnit, Brain Monkey)** | `09-testing.md` |
| **Automation (Scoping, Fixers)** | `13-automation-tooling.md` |
| **Pre-Commit Checklist** | `11-checklist.md` |
| **Advanced Patterns (DTOs, CLI)** | `12-advanced-patterns.md` |

## Core Development Conventions

When using these guidelines to generate or modify code, always adhere to:

1.  **PHP Standards**: Use PHP 8.0+ syntax with `declare(strict_types=1)` in every file.
2.  **Naming**:
    *   Namespace: `NVM\{Plugin}\`
    *   Hooks: `nvm/{plugin-slug}/`
3.  **Security**:
    *   Never skip nonce verification for `POST`/`AJAX`/`REST` requests.
    *   Always check user capabilities (`current_user_can()`).
    *   Sanitize all inputs and escape all outputs using WordPress-specific functions (e.g., `wp_kses_post`, `esc_html`).
4.  **Frontend**: Prefer Vanilla JS over jQuery for frontend components.
5.  **Author Credits**:
    *   Author: `nevma`
    *   Author URI: `https://nevma.gr`

## Usage for Gemini CLI

When tasked with a WordPress/WooCommerce development job while this context is active:

1.  **Identify the task** (e.g., "Create an AJAX handler for product filtering").
2.  **Locate the relevant guideline** using the table above or `index.md`.
3.  **Read the guideline** to understand the specific patterns and security requirements.
4.  **Implement the code** following the established architecture and naming conventions.
5.  **Self-Audit** the implementation for security and performance before finalizing.
