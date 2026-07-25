# WooCommerce

> CRUD classes, HPOS + block compatibility, custom order statuses, payment gateways (classic + block checkout), shipping, emails, settings, cart fees, webhooks, and logging.

---

## Use CRUD Classes

Never use direct SQL for products/orders:

```php
$product = wc_get_product( $id );
$product?->set_stock_quantity( 50 );
$product?->save();
```

### Product Types

```php
// Simple product.
$product = new \WC_Product_Simple();
$product->set_name( 'Test Product' );
$product->set_regular_price( '29.99' );
$product->set_stock_quantity( 100 );
$product->set_manage_stock( true );
$product->save();

// Variable product.
$variable = new \WC_Product_Variable();
$variable->set_name( 'T-Shirt' );
$variable->save();

$variation = new \WC_Product_Variation();
$variation->set_parent_id( $variable->get_id() );
$variation->set_attributes( [ 'pa_size' => 'large' ] );
$variation->set_regular_price( '19.99' );
$variation->save();
```

### Querying Products

Use `wc_get_products()`, never `WP_Query`/`get_posts` for products:

```php
$product_ids = wc_get_products( [
    'status' => 'publish',
    'limit'  => 50,       // ALWAYS set a limit. Never -1.
    'return' => 'ids',    // Use 'ids' when you don't need full objects.
] );

// Batched processing over large catalogs.
$page = 1;
do {
    $results = wc_get_products( [
        'limit'    => 100,
        'page'     => $page,
        'paginate' => true,   // Returns object with ->products, ->total, ->max_num_pages.
    ] );

    foreach ( $results->products as $product ) {
        // Process...
    }

    ++$page;
} while ( $page <= $results->max_num_pages );
```

### Order Operations

```php
// Create order programmatically.
$order = wc_create_order( [
    'customer_id' => $user_id,
    'status'      => 'pending',
] );

$order->add_product( wc_get_product( $product_id ), 2 );
$order->set_address( $billing_address, 'billing' );
$order->calculate_totals();
$order->save();

// Update order status with note.
$order->update_status( 'processing', __( 'Payment received via custom gateway.', 'nvm-plugin' ) );

// Add order meta (HPOS-safe).
$order->update_meta_data( '_nvm_custom_field', $value );
$order->save();

// Read order meta.
$value = $order->get_meta( '_nvm_custom_field', true );
```

### Refunds

```php
// Programmatic refund — creates the refund object, adjusts totals, restocks.
$refund = wc_create_refund( [
    'order_id'       => $order->get_id(),
    'amount'         => 10.00,
    'reason'         => __( 'Damaged item', 'nvm-plugin' ),
    'refund_payment' => true,  // Also calls the gateway's process_refund().
    'restock_items'  => true,
] );

if ( is_wp_error( $refund ) ) {
    wc_get_logger()->error( $refund->get_error_message(), [ 'source' => Plugin::SLUG ] );
}
```

Never just add an order note and call it a refund — `wc_create_refund()` is what adjusts totals and stock.

---

## HPOS & Feature Compatibility

Never use `wp_posts`/`wp_postmeta` for orders:

```php
$orders = wc_get_orders( [
    'status' => 'processing',
    'limit'  => 50,       // ALWAYS set a limit.
    'return' => 'ids',    // Use 'ids' when you don't need full objects.
] );
```

### Declare Feature Compatibility

Declare **both** HPOS and block cart/checkout compatibility on `before_woocommerce_init` (see `02-architecture.md` for the Plugin class wiring):

```php
public function declare_feature_compatibility(): void {
    if ( ! class_exists( \Automattic\WooCommerce\Utilities\FeaturesUtil::class ) ) {
        return;
    }

    \Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility(
        'custom_order_tables',
        NVM_PLUGIN_FILE,
        true
    );
    \Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility(
        'cart_checkout_blocks',
        NVM_PLUGIN_FILE,
        true
    );
}
```

Without the `cart_checkout_blocks` declaration, admins see an incompatibility warning when the plugin touches cart/checkout.

### HPOS-Safe Meta Queries

```php
// Correct — uses WooCommerce query args.
$orders = wc_get_orders( [
    'meta_query' => [
        [
            'key'   => '_nvm_sync_status',
            'value' => 'pending',
        ],
    ],
    'limit' => 50,
] );

// WRONG — never do this with HPOS enabled.
// $orders = get_posts( [ 'post_type' => 'shop_order', ... ] );
```

### HPOS Admin Screen Hooks

The HPOS orders screen (`woocommerce_page_wc-orders`) uses **different hooks** than the legacy screen (`edit-shop_order`). Register on both or the feature silently does nothing on one of them:

```php
// Bulk actions.
$add_bulk_action = static function( array $actions ): array {
    $actions['mark_nvm-awaiting'] = __( 'Change status to Awaiting Fulfillment', 'nvm-plugin' );
    return $actions;
};
add_filter( 'bulk_actions-woocommerce_page_wc-orders', $add_bulk_action ); // HPOS screen.
add_filter( 'bulk_actions-edit-shop_order', $add_bulk_action );            // Legacy screen.

// Same pattern applies to list columns:
// 'manage_woocommerce_page_wc-orders_columns' vs 'manage_edit-shop_order_columns'.
```

---

## Custom Order Statuses

With HPOS, orders are not posts — the `wc_order_statuses` filter is what registers the status:

```php
// Add to WooCommerce status list.
add_filter( 'wc_order_statuses', static function( array $statuses ): array {
    $statuses['wc-nvm-awaiting'] = _x( 'Awaiting Fulfillment', 'Order status', 'nvm-plugin' );
    return $statuses;
} );
```

Only add `register_post_status()` if the plugin must also support legacy (non-HPOS) stores — it is a no-op for HPOS orders.

For bulk actions on the order list, see **HPOS Admin Screen Hooks** above — register on both screens.

---

## Custom Product Data Tabs

```php
add_filter( 'woocommerce_product_data_tabs', static function( array $tabs ): array {
    $tabs['nvm_custom'] = [
        'label'    => __( 'Custom Data', 'nvm-plugin' ),
        'target'   => 'nvm_custom_product_data',
        'priority' => 70,
    ];
    return $tabs;
} );

add_action( 'woocommerce_product_data_panels', static function(): void {
    echo '<div id="nvm_custom_product_data" class="panel woocommerce_options_panel">';

    woocommerce_wp_text_input( [
        'id'    => '_nvm_custom_field',
        'label' => __( 'Custom Field', 'nvm-plugin' ),
        'type'  => 'text',
    ] );

    echo '</div>';
} );

// Save via the CRUD object — not update_post_meta().
add_action( 'woocommerce_admin_process_product_object', static function( \WC_Product $product ): void {
    if ( isset( $_POST['_nvm_custom_field'] ) ) {
        $product->update_meta_data(
            '_nvm_custom_field',
            sanitize_text_field( wp_unslash( $_POST['_nvm_custom_field'] ) )
        );
    }
    // WooCommerce calls $product->save() after this hook — do not save here.
} );
```

Use `woocommerce_admin_process_product_object` (receives the `WC_Product`), not the legacy `woocommerce_process_product_meta` + `update_post_meta()` combination.

---

## Settings

Add a section to an existing WooCommerce settings tab — no custom admin page needed:

```php
// Add section under WooCommerce → Settings → Products.
add_filter( 'woocommerce_get_sections_products', static function( array $sections ): array {
    $sections['nvm_plugin'] = __( 'NVM Plugin', 'nvm-plugin' );
    return $sections;
} );

add_filter( 'woocommerce_get_settings_products', static function( array $settings, string $current_section ): array {
    if ( 'nvm_plugin' !== $current_section ) {
        return $settings;
    }

    return [
        [
            'title' => __( 'NVM Plugin Settings', 'nvm-plugin' ),
            'type'  => 'title',
            'id'    => 'nvm_plugin_options',
        ],
        [
            'title'   => __( 'Enable feature', 'nvm-plugin' ),
            'id'      => 'nvm_plugin_enable_feature',
            'type'    => 'checkbox',
            'default' => 'no',
        ],
        [
            'type' => 'sectionend',
            'id'   => 'nvm_plugin_options',
        ],
    ];
}, 10, 2 );

// Read anywhere.
$enabled = 'yes' === get_option( 'nvm_plugin_enable_feature', 'no' );
```

WooCommerce handles rendering, saving, sanitization, and nonces for registered setting types. For a whole custom tab, extend `WC_Settings_Page` and register it via the `woocommerce_get_settings_pages` filter.

---

## Payment Gateways

Gateways need **two** classes: a `WC_Payment_Gateway` for processing (and classic checkout), plus an `AbstractPaymentMethodType` for block checkout. Without the second, the gateway does not appear in block checkout at all (the default since WC 8.3).

### Gateway Class

```php
<?php

declare(strict_types=1);

namespace NVM\Plugin\Gateways;

class Custom_Gateway extends \WC_Payment_Gateway {

    public function __construct() {
        $this->id                 = 'nvm_custom';
        $this->method_title       = __( 'Custom Payment', 'nvm-plugin' );
        $this->method_description = __( 'Accept payments via custom gateway.', 'nvm-plugin' );
        $this->has_fields         = false;
        $this->supports           = [ 'products', 'refunds' ];

        $this->init_form_fields();
        $this->init_settings();

        $this->title   = $this->get_option( 'title' );
        $this->enabled = $this->get_option( 'enabled' );

        add_action(
            'woocommerce_update_options_payment_gateways_' . $this->id,
            [ $this, 'process_admin_options' ]
        );
    }

    public function init_form_fields(): void {
        $this->form_fields = [
            'enabled' => [
                'title'   => __( 'Enable/Disable', 'nvm-plugin' ),
                'type'    => 'checkbox',
                'label'   => __( 'Enable Custom Payment', 'nvm-plugin' ),
                'default' => 'no',
            ],
            'title' => [
                'title'       => __( 'Title', 'nvm-plugin' ),
                'type'        => 'text',
                'description' => __( 'Payment method title shown at checkout.', 'nvm-plugin' ),
                'default'     => __( 'Custom Payment', 'nvm-plugin' ),
            ],
        ];
    }

    public function process_payment( $order_id ): array {
        $order = wc_get_order( $order_id );

        if ( ! $order ) {
            wc_add_notice( __( 'Order not found.', 'nvm-plugin' ), 'error' );
            return [ 'result' => 'failure' ];
        }

        // Process payment logic here. On provider error:
        // wc_add_notice( $message, 'error' ); return [ 'result' => 'failure' ];

        $order->payment_complete();
        WC()->cart->empty_cart();

        return [
            'result'   => 'success',
            'redirect' => $this->get_return_url( $order ),
        ];
    }

    /**
     * Called by wc_create_refund() when 'refund_payment' is true.
     *
     * @return bool|\WP_Error
     */
    public function process_refund( $order_id, $amount = null, $reason = '' ) {
        $order = wc_get_order( $order_id );

        if ( ! $order ) {
            return new \WP_Error( 'nvm_invalid_order', __( 'Order not found.', 'nvm-plugin' ) );
        }

        // Call the payment provider's refund API here.
        // Return a WP_Error on provider failure — WooCommerce shows it to the admin.

        $order->add_order_note(
            /* translators: 1: refund amount 2: refund reason */
            sprintf( __( 'Refunded %1$s. Reason: %2$s', 'nvm-plugin' ), wc_price( (float) $amount ), $reason )
        );

        return true;
    }
}
```

```php
// Register gateway.
add_filter( 'woocommerce_payment_gateways', static function( array $gateways ): array {
    $gateways[] = \NVM\Plugin\Gateways\Custom_Gateway::class;
    return $gateways;
} );
```

### Block Checkout Support

```php
<?php

declare(strict_types=1);

namespace NVM\Plugin\Gateways;

use Automattic\WooCommerce\Blocks\Payments\Integrations\AbstractPaymentMethodType;

class Custom_Gateway_Blocks extends AbstractPaymentMethodType {

    protected $name = 'nvm_custom'; // Must match the gateway's $id.

    public function initialize(): void {
        $this->settings = get_option( 'woocommerce_nvm_custom_settings', [] );
    }

    public function is_active(): bool {
        return ! empty( $this->settings['enabled'] ) && 'yes' === $this->settings['enabled'];
    }

    public function get_payment_method_script_handles(): array {
        wp_register_script(
            'nvm-custom-gateway-blocks',
            plugins_url( 'assets/js/gateway-blocks.js', NVM_PLUGIN_FILE ),
            [ 'wc-blocks-registry', 'wc-settings', 'wp-element', 'wp-i18n' ],
            '1.0.0',
            true
        );
        return [ 'nvm-custom-gateway-blocks' ];
    }

    public function get_payment_method_data(): array {
        return [
            'title'       => $this->get_setting( 'title' ),
            'description' => $this->get_setting( 'description' ),
        ];
    }
}
```

```php
// Register block support.
add_action(
    'woocommerce_blocks_payment_method_type_registration',
    static function( \Automattic\WooCommerce\Blocks\Payments\PaymentMethodRegistry $registry ): void {
        $registry->register( new \NVM\Plugin\Gateways\Custom_Gateway_Blocks() );
    }
);
```

```js
// assets/js/gateway-blocks.js — vanilla, no build step.
const { registerPaymentMethod } = window.wc.wcBlocksRegistry;
const { getSetting } = window.wc.wcSettings;
const { createElement } = window.wp.element;

const settings = getSetting( 'nvm_custom_data', {} );
const label = settings.title || 'Custom Payment';

registerPaymentMethod( {
	name: 'nvm_custom',
	label,
	ariaLabel: label,
	content: createElement( 'div', null, settings.description || '' ),
	edit: createElement( 'div', null, settings.description || '' ),
	canMakePayment: () => true,
} );
```

---

## Shipping Methods

```php
<?php

declare(strict_types=1);

namespace NVM\Plugin\Shipping;

class Custom_Shipping extends \WC_Shipping_Method {

    public function __construct( int $instance_id = 0 ) {
        $this->id                 = 'nvm_custom_shipping';
        $this->instance_id        = absint( $instance_id );
        $this->method_title       = __( 'Custom Shipping', 'nvm-plugin' );
        $this->method_description = __( 'Custom shipping calculation.', 'nvm-plugin' );
        $this->supports           = [ 'shipping-zones', 'instance-settings' ];

        $this->init_form_fields();
        $this->init_settings();

        $this->title = $this->get_option( 'title' );
    }

    public function calculate_shipping( $package = [] ): void {
        $cost = 0;

        foreach ( $package['contents'] as $item ) {
            $product = $item['data'];
            $weight  = (float) $product->get_weight();
            $cost   += $weight * 2.50 * $item['quantity'];
        }

        $this->add_rate( [
            'id'    => $this->get_rate_id(),
            'label' => $this->title,
            'cost'  => $cost,
        ] );
    }
}

add_filter( 'woocommerce_shipping_methods', static function( array $methods ): array {
    $methods['nvm_custom_shipping'] = \NVM\Plugin\Shipping\Custom_Shipping::class;
    return $methods;
} );
```

---

## Custom Emails

```php
<?php

declare(strict_types=1);

namespace NVM\Plugin\Emails;

class Custom_Email extends \WC_Email {

    public function __construct() {
        $this->id             = 'nvm_custom_notification';
        $this->title          = __( 'Custom Notification', 'nvm-plugin' );
        $this->description    = __( 'Sent when a custom event occurs.', 'nvm-plugin' );
        $this->template_html  = 'emails/custom-notification.php';
        $this->template_plain = 'emails/plain/custom-notification.php';
        $this->template_base  = NVM_PLUGIN_PATH . 'templates/';
        $this->placeholders   = [
            '{order_number}' => '',
            '{order_date}'   => '',
        ];

        // Trigger on custom hook.
        add_action( 'nvm/plugin/custom_event', [ $this, 'trigger' ], 10, 2 );

        parent::__construct();
    }

    public function trigger( int $order_id, \WC_Order $order ): void {
        $this->setup_locale();

        if ( $order ) {
            $this->object                          = $order;
            $this->recipient                       = $order->get_billing_email();
            $this->placeholders['{order_number}']  = $order->get_order_number();
            $this->placeholders['{order_date}']    = wc_format_datetime( $order->get_date_created() );
        }

        if ( $this->is_enabled() && $this->get_recipient() ) {
            $this->send(
                $this->get_recipient(),
                $this->get_subject(),
                $this->get_content(),
                $this->get_headers(),
                $this->get_attachments()
            );
        }

        $this->restore_locale();
    }

    public function get_content_html(): string {
        return wc_get_template_html(
            $this->template_html,
            [
                'order'              => $this->object,
                'email_heading'      => $this->get_heading(),
                'additional_content' => $this->get_additional_content(),
                'sent_to_admin'      => false,
                'plain_text'         => false,
                'email'              => $this,
            ],
            '',
            $this->template_base
        );
    }

    public function get_content_plain(): string {
        return wc_get_template_html(
            $this->template_plain,
            [
                'order'              => $this->object,
                'email_heading'      => $this->get_heading(),
                'additional_content' => $this->get_additional_content(),
                'sent_to_admin'      => false,
                'plain_text'         => true,
                'email'              => $this,
            ],
            '',
            $this->template_base
        );
    }
}
```

```php
// Register email class.
add_filter( 'woocommerce_email_classes', static function( array $emails ): array {
    $emails['nvm_custom_notification'] = new \NVM\Plugin\Emails\Custom_Email();
    return $emails;
} );

// REQUIRED for custom trigger hooks: WC_Emails is lazy-loaded, so email classes
// only exist when one of these hooks fires. Without this, the trigger never runs
// in transactional contexts (cron, REST, webhooks).
add_filter( 'woocommerce_email_actions', static function( array $actions ): array {
    $actions[] = 'nvm/plugin/custom_event';
    return $actions;
} );
```

---

## Block Checkout

Classic hooks (`woocommerce_after_checkout_form`, `woocommerce_checkout_process`, etc.) do **not** run with block checkout. See the classic ↔ block hook mapping under **WooCommerce Hooks Reference**.

### Additional Checkout Fields API (WC 8.9+)

The preferred way to add checkout fields — works in **both** block and classic (shortcode) checkout, with validation, persistence, and admin display handled by WooCommerce:

```php
add_action( 'woocommerce_init', static function(): void {
    if ( ! function_exists( 'woocommerce_register_additional_checkout_field' ) ) {
        return;
    }

    woocommerce_register_additional_checkout_field( [
        'id'       => 'nvm-plugin/vat-number',   // namespace/field-name format required.
        'label'    => __( 'VAT Number', 'nvm-plugin' ),
        'location' => 'contact',                  // 'contact', 'address', or 'order'.
        'type'     => 'text',                     // 'text', 'select', or 'checkbox'.
        'required' => false,
    ] );
} );

// Read the value ('contact'/'order' locations store under _wc_other/).
$vat = $order->get_meta( '_wc_other/nvm-plugin/vat-number' );

// 'address' location fields store per address type:
// $order->get_meta( '_wc_billing/address/nvm-plugin/vat-number' );

// Custom validation.
add_action(
    'woocommerce_validate_additional_field',
    static function( \WP_Error $errors, string $field_key, $field_value ): void {
        if ( 'nvm-plugin/vat-number' === $field_key && '' !== $field_value && ! preg_match( '/^[A-Z]{2}\d+$/', (string) $field_value ) ) {
            $errors->add( 'nvm_invalid_vat', __( 'Invalid VAT number format.', 'nvm-plugin' ) );
        }
    },
    10,
    3
);
```

### Store API ExtendSchema (advanced)

For arbitrary data on Store API endpoints (cart/checkout blocks) beyond simple fields:

```php
use Automattic\WooCommerce\StoreApi\StoreApi;
use Automattic\WooCommerce\StoreApi\Schemas\V1\CheckoutSchema;

add_action( 'woocommerce_blocks_loaded', function(): void {
    $extend = StoreApi::container()->get( \Automattic\WooCommerce\StoreApi\Schemas\ExtendSchema::class );

    $extend->register_endpoint_data( [
        'endpoint'        => CheckoutSchema::IDENTIFIER,
        'namespace'       => Plugin::SLUG,
        'data_callback'   => fn() => [ 'custom_field' => '' ],
        'schema_callback' => fn() => [
            'custom_field' => [
                'description' => __( 'Custom checkout field', 'nvm-plugin' ),
                'type'        => 'string',
            ],
        ],
    ] );
} );
```

---

## Cart Fees & Session

```php
// Add a fee (runs on every totals calculation — keep it fast, no queries/HTTP).
add_action( 'woocommerce_cart_calculate_fees', static function( \WC_Cart $cart ): void {
    if ( is_admin() && ! defined( 'DOING_AJAX' ) ) {
        return;
    }

    if ( WC()->session?->get( 'nvm_gift_wrap' ) ) {
        $cart->add_fee( __( 'Gift wrapping', 'nvm-plugin' ), 5.00, true ); // true = taxable.
    }
} );

// Session storage — per-customer, survives page loads, works for guests.
WC()->session->set( 'nvm_gift_wrap', true );
$gift_wrap = WC()->session->get( 'nvm_gift_wrap', false );
WC()->session->__unset( 'nvm_gift_wrap' );
```

Always null-check `WC()->session` outside cart/checkout contexts (it is not initialized in admin, cron, or REST).

---

## WooCommerce REST API Extensions

```php
// Add custom endpoint under WooCommerce namespace.
add_action( 'rest_api_init', static function(): void {
    register_rest_route( 'nvm/plugin/v1', '/stock/(?P<id>\d+)', [
        'methods'             => 'GET',
        'callback'            => static function( \WP_REST_Request $request ): \WP_REST_Response {
            $product = wc_get_product( $request->get_param( 'id' ) );

            if ( ! $product ) {
                return new \WP_REST_Response( [ 'error' => 'Product not found' ], 404 );
            }

            return new \WP_REST_Response( [
                'id'    => $product->get_id(),
                'stock' => $product->get_stock_quantity(),
                'sku'   => $product->get_sku(),
            ] );
        },
        'permission_callback' => static function(): bool {
            return current_user_can( 'manage_woocommerce' );
        },
        'args' => [
            'id' => [
                'required'          => true,
                'validate_callback' => static fn( $v ): bool => is_numeric( $v ) && (int) $v > 0,
                'sanitize_callback' => 'absint',
            ],
        ],
    ] );
} );
```

---

## Webhooks

Register custom webhook topics so merchants can subscribe external systems to plugin events (WooCommerce → Settings → Advanced → Webhooks):

```php
// Map topic to the action hook(s) that fire it.
add_filter( 'woocommerce_webhook_topic_hooks', static function( array $topic_hooks ): array {
    $topic_hooks['order.nvm_synced'] = [ 'nvm/plugin/order_synced' ];
    return $topic_hooks;
} );

// Whitelist the event on the 'order' resource.
add_filter( 'woocommerce_valid_webhook_events', static function( array $events ): array {
    $events[] = 'nvm_synced';
    return $events;
} );

// Human-readable name in the webhook admin dropdown.
add_filter( 'woocommerce_webhook_topics', static function( array $topics ): array {
    $topics['order.nvm_synced'] = __( 'Order synced (NVM)', 'nvm-plugin' );
    return $topics;
} );

// Fire it — the payload is built from the resource ID (here: order ID).
do_action( 'nvm/plugin/order_synced', $order->get_id() );
```

---

## WooCommerce Hooks Reference

### Order Lifecycle (fires for both checkout types)

| Hook | When |
|------|------|
| `woocommerce_new_order` | Order first created |
| `woocommerce_order_status_changed` | Any status transition |
| `woocommerce_order_status_{from}_to_{to}` | Specific transition |
| `woocommerce_payment_complete` | Payment received |
| `woocommerce_order_refunded` | Refund processed |

### Classic ↔ Block Checkout Hook Mapping

Block checkout goes through the Store API — classic checkout hooks never fire. Hook **both** unless the store is confirmed single-checkout:

| Classic (shortcode) checkout | Block checkout (Store API) |
|------|------|
| `woocommerce_checkout_process` (validation) | `woocommerce_store_api_checkout_update_order_from_request` (throw `RouteException` to reject) |
| `woocommerce_checkout_update_order_meta` | `woocommerce_store_api_checkout_update_order_meta` |
| `woocommerce_checkout_order_processed` | `woocommerce_store_api_checkout_order_processed` |
| `woocommerce_after_checkout_form` (UI) | No PHP equivalent — extend via blocks/JS (see Block Checkout section) |

`woocommerce_add_to_cart` and `woocommerce_before_calculate_totals` fire for both, since block carts still use core cart internals.

---

## Version Compatibility

Always check WooCommerce version before using newer APIs:

```php
if ( version_compare( WC_VERSION, '8.0', '>=' ) ) {
    // Use newer API.
} else {
    // Fallback for older versions.
}
```

### Minimum Version Check (Plugin Activation)

```php
public function check_requirements(): bool {
    if ( ! class_exists( 'WooCommerce' ) ) {
        add_action( 'admin_notices', static function(): void {
            echo '<div class="notice notice-error"><p>';
            esc_html_e( 'This plugin requires WooCommerce to be installed and active.', 'nvm-plugin' );
            echo '</p></div>';
        } );
        return false;
    }

    if ( version_compare( WC_VERSION, '8.0', '<' ) ) {
        add_action( 'admin_notices', static function(): void {
            echo '<div class="notice notice-error"><p>';
            esc_html_e( 'This plugin requires WooCommerce 8.0 or higher.', 'nvm-plugin' );
            echo '</p></div>';
        } );
        return false;
    }

    return true;
}
```

---

## Logging

Use WooCommerce's built-in logger:

```php
wc_get_logger()->info(
    'Stock updated for product {product_id}: {old} → {new}',
    [
        'source'     => Plugin::SLUG,
        'product_id' => $product_id,
        'old'        => $old_stock,
        'new'        => $new_stock,
    ]
);
```

Log levels: `debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`.

Logs appear in **WooCommerce → Status → Logs**.

### Structured Logging for Debugging

```php
// Log full context for debugging payment issues.
wc_get_logger()->error(
    'Payment failed for order {order_id}: {error}',
    [
        'source'   => Plugin::SLUG,
        'order_id' => $order->get_id(),
        'error'    => $exception->getMessage(),
        'gateway'  => $order->get_payment_method(),
        'total'    => $order->get_total(),
        'trace'    => $exception->getTraceAsString(),
    ]
);
```
