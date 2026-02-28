# Entos Theme Analysis Plan

Analysis of the "V for Vanilla" (entos) theme to extract patterns for AI guidelines integration.

## Theme Overview

- **Name**: V for Vanilla (vforvanilla)
- **Version**: 2.5
- **Key Feature**: ACF blocks-based content system (replaces Gutenberg default blocks)
- **Includes**: Responsiville (Nevma's responsive CSS/JS framework)

---

## Analysis Phases

### Phase 1: Architecture Patterns
**Files to analyze:**
- [ ] `functions.php` - Entry point, loading pattern
- [ ] `inc/init.php` - Initialization flow
- [ ] `inc/functions/` - Module organization pattern

**Extract:**
- File organization conventions
- Module loading patterns
- Hook naming conventions

---

### Phase 2: ACF Blocks System
**Files to analyze:**
- [ ] `inc/blocks/blocks.php` - Block registration
- [ ] `inc/blocks/common/blocks-common.php` - Utility functions
- [ ] Sample blocks: `section`, `column`, `cards`

**Extract:**
- Block structure pattern (block.json + render.php)
- InnerBlocks usage
- Common utilities (box model, backgrounds, colors, links)
- Block category organization

**Add to guidelines:**
- New section: `13-acf-blocks.md` or integrate into existing

---

### Phase 3: Responsiville Framework
**Files to analyze:**
- [ ] `inc/responsiville/` - CSS and JS files
- [ ] Grid system usage in blocks
- [ ] JavaScript component patterns

**Extract:**
- Grid breakpoint conventions (`small`, `tablet`, `laptop`)
- Column class naming
- JS component initialization patterns

---

### Phase 4: WooCommerce Integration
**Files to analyze:**
- [ ] `woocommerce/` directory templates
- [ ] `single-product.php` customizations
- [ ] `archive-product.php` modifications

**Extract:**
- Template override patterns
- Product display customizations
- Hook usage for WooCommerce

---

### Phase 5: JavaScript Patterns
**Files to analyze:**
- [ ] `js/` directory
- [ ] `theme.*.js` files
- [ ] Admin vs frontend separation

**Extract:**
- Vanilla JS patterns (no jQuery on frontend)
- Module organization
- Event handling patterns

---

### Phase 6: CSS Architecture
**Files to analyze:**
- [ ] `style.css` - Main theme styles
- [ ] `css/` directory structure
- [ ] Block-specific styles

**Extract:**
- CSS naming conventions
- File organization
- Responsive patterns

---

## Integration Points with Guidelines

| Theme Pattern | Guideline to Update |
|--------------|---------------------|
| ACF Blocks | New: `13-acf-blocks.md` |
| Responsiville grid | `07-javascript.md` (add responsive patterns) |
| WooCommerce templates | `05-woocommerce.md` (template overrides) |
| Module loading | `03-modern-php.md` (organization) |
| No-jQuery frontend | Already in guidelines ✓ |

---

## File Count Summary

```
PHP files:     ~50+
JS files:      TBD (in js/ and responsiville/)
CSS files:     TBD (in css/ and responsiville/)
ACF Blocks:    ~20+ blocks
```

---

## Execution Order

1. **Phase 1** - Understand loading/architecture (foundation)
2. **Phase 2** - ACF Blocks (most unique/valuable)
3. **Phase 4** - WooCommerce (client priority)
4. **Phase 3** - Responsiville (framework knowledge)
5. **Phase 5-6** - JS/CSS (refinement)

---

## Notes

- Theme uses Polylang for translations (`pll__()`, `pll_e()`)
- Has WPML config for alternative translation
- Integrates with Gravity Forms
- Has debug flags via ACF options page
