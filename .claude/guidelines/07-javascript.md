# JavaScript

> Vanilla JS for frontend, jQuery for admin, and data passing patterns.

---

## Vanilla JS (Frontend — Preferred)

For frontend scripts, use vanilla JS with `fetch()` — no jQuery dependency:

```javascript
( function() {
	'use strict';

	/**
	 * NVM Inventory frontend handler.
	 *
	 * @since 1.0.0
	 */
	const NVM_Inventory = {
		config: null,

		init() {
			this.config = window.nvm_inventory_params ?? null;

			if ( ! this.config ) {
				return;
			}

			this.bindEvents();
		},

		bindEvents() {
			document.querySelectorAll( '.nvm-inv-btn' ).forEach( ( btn ) => {
				btn.addEventListener( 'click', ( e ) => this.handleClick( e ) );
			} );
		},

		async handleClick( e ) {
			e.preventDefault();

			const btn = e.currentTarget;
			btn.disabled = true;

			try {
				const formData = new FormData();
				formData.append( 'action', this.config.action );
				formData.append( 'nonce', this.config.nonce );
				formData.append( 'product_id', btn.dataset.productId );

				const response = await fetch( this.config.ajax_url, {
					method: 'POST',
					credentials: 'same-origin',
					body: formData,
				} );

				const data = await response.json();

				if ( data.success ) {
					this.onSuccess( data.data, btn );
				} else {
					this.onError( data.data?.message ?? 'Unknown error' );
				}
			} catch ( error ) {
				console.error( '[NVM Inventory]', error );
			} finally {
				btn.disabled = false;
			}
		},

		onSuccess( data, btn ) {
			// Handle success.
		},

		onError( message ) {
			// Handle error.
		},
	};

	if ( document.readyState === 'loading' ) {
		document.addEventListener( 'DOMContentLoaded', () => NVM_Inventory.init() );
	} else {
		NVM_Inventory.init();
	}
} )();
```

---

## jQuery (Admin Only)

jQuery is acceptable for admin scripts where WordPress already loads it:

```javascript
( function( $ ) {
	'use strict';

	const NVM_Inventory_Admin = {
		init() {
			this.bindEvents();
		},

		bindEvents() {
			$( document ).on( 'click', '.nvm-inv-admin-btn', ( e ) => this.handleClick( e ) );
		},

		async handleClick( e ) {
			e.preventDefault();

			try {
				const response = await $.post( nvm_inventory_admin_params.ajax_url, {
					action: nvm_inventory_admin_params.action,
					nonce: nvm_inventory_admin_params.nonce,
				} );

				if ( response.success ) {
					// Handle success.
				} else {
					console.error( response.data.message );
				}
			} catch ( error ) {
				console.error( '[NVM Inventory Admin]', error );
			}
		},
	};

	$( () => NVM_Inventory_Admin.init() );
} )( jQuery );
```

---

## Data Passing (PHP Side)

### Use `wp_add_inline_script` for Non-Translatable Data

Preserves types (booleans, integers):

```php
wp_enqueue_script(
	'nvm-inventory-checkout',
	Plugin::url( 'assets/js/checkout.js' ),
	[],
	Plugin::VERSION,
	[ 'strategy' => 'defer' ]
);

wp_add_inline_script(
	'nvm-inventory-checkout',
	'window.nvm_inventory_params = ' . wp_json_encode( [
		'ajax_url'   => admin_url( 'admin-ajax.php' ),
		'nonce'      => wp_create_nonce( Plugin::PREFIX . 'ajax' ),
		'action'     => Plugin::PREFIX . 'update_stock',
		'product_id' => $product_id,
		'is_admin'   => current_user_can( 'manage_woocommerce' ),  // Boolean preserved!
	], JSON_HEX_TAG | JSON_HEX_AMP ) . ';',
	'before'
);
```

### Use `wp_localize_script` Only for Translatable Strings

```php
wp_localize_script( 'nvm-inventory-admin', 'nvm_inventory_i18n', [
	'confirm_delete' => __( 'Are you sure?', 'nvm-inventory' ),
	'saving'         => __( 'Saving…', 'nvm-inventory' ),
	'saved'          => __( 'Saved!', 'nvm-inventory' ),
] );
```

---

## Script Loading Strategy (WP 6.3+)

```php
wp_enqueue_script(
	'nvm-inventory-frontend',
	Plugin::url( 'assets/js/frontend.js' ),
	[],
	Plugin::VERSION,
	[
		'strategy'  => 'defer',    // or 'async'
		'in_footer' => true,
	]
);
```

---

## Summary

| Context | Library | Data Method |
|---------|---------|-------------|
| Frontend | Vanilla JS + `fetch()` | `wp_add_inline_script` |
| Admin | jQuery allowed | `wp_add_inline_script` + `wp_localize_script` for i18n |
