# WooCommerce

> CRUD classes, HPOS compatibility, block checkout, custom order statuses, payment gateways, shipping, emails, and logging.

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

---

## HPOS Compatibility

Never use `wp_posts`/`wp_postmeta` for orders:

```php
$orders = wc_get_orders( [
    'status' => 'processing',
    'limit'  => 50,       // ALWAYS set a limit.
    'return' => 'ids',    // Use 'ids' when you don't need full objects.
] );
```

### Declare HPOS Compatibility

Already included in Plugin class pattern (see `02-architecture.md`):

```php
public function declare_hpos_compatibility(): void {
    if ( class_exists( \Automattic\WooCommerce\Utilities\FeaturesUtil::class ) ) {
        \Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility(
            'custom_order_tables',
            NVM_INV_FILE,
            true
        );
    }
}
```

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

---

## Custom Order Statuses

```php
// Register custom status.
add_action( 'init', static function(): void {
    register_post_status( 'wc-nvm-awaiting', [
        'label'                     => _x( 'Awaiting Fulfillment', 'Order status', 'nvm-plugin' ),
        'public'                    => true,
        'exclude_from_search'       => false,
        'show_in_admin_all_list'    => true,
        'show_in_admin_status_list' => true,
        /* translators: %s: number of orders */
        'label_count'               => _n_noop(
            'Awaiting Fulfillment <span class="count">(%s)</span>',
            'Awaiting Fulfillment <span class="count">(%s)</span>',
            'nvm-plugin'
        ),
    ] );
} );

// Add to WooCommerce status list.
add_filter( 'wc_order_statuses', static function( array $statuses ): array {
    $statuses['wc-nvm-awaiting'] = _x( 'Awaiting Fulfillment', 'Order status', 'nvm-plugin' );
    return $statuses;
} );

// Add to bulk actions.
add_filter( 'bulk_actions-edit-shop_order', static function( array $actions ): array {
    $actions['mark_nvm-awaiting'] = __( 'Change status to Awaiting Fulfillment', 'nvm-plugin' );
    return $actions;
} );
```

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
    global $post;
    echo '<div id="nvm_custom_product_data" class="panel woocommerce_options_panel">';

    woocommerce_wp_text_input( [
        'id'    => '_nvm_custom_field',
        'label' => __( 'Custom Field', 'nvm-plugin' ),
        'type'  => 'text',
    ] );

    echo '</div>';
} );

add_action( 'woocommerce_process_product_meta', static function( int $post_id ): void {
    $value = isset( $_POST['_nvm_custom_field'] )
        ? sanitize_text_field( wp_unslash( $_POST['_nvm_custom_field'] ) )
        : '';

    update_post_meta( $post_id, '_nvm_custom_field', $value );
} );
```

---

## Payment Gateways

```php
class NVM_Custom_Gateway extends \WC_Payment_Gateway {

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
            throw new \Exception( __( 'Order not found.', 'nvm-plugin' ) );
        }

        // Process payment logic here.
        $order->payment_complete();
        WC()->cart->empty_cart();

        return [
            'result'   => 'success',
            'redirect' => $this->get_return_url( $order ),
        ];
    }

    public function process_refund( $order_id, $amount = null, $reason = '' ): bool {
        $order = wc_get_order( $order_id );

        if ( ! $order ) {
            return false;
        }

        // Process refund logic here.
        $order->add_order_note(
            /* translators: 1: refund amount 2: refund reason */
            sprintf( __( 'Refunded %1$s. Reason: %2$s', 'nvm-plugin' ), $amount, $reason )
        );

        return true;
    }
}

// Register gateway.
add_filter( 'woocommerce_payment_gateways', static function( array $gateways ): array {
    $gateways[] = NVM_Custom_Gateway::class;
    return $gateways;
} );
```

---

## Shipping Methods

```php
class NVM_Custom_Shipping extends \WC_Shipping_Method {

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
    $methods['nvm_custom_shipping'] = NVM_Custom_Shipping::class;
    return $methods;
} );
```

---

## Custom Emails

```php
class NVM_Custom_Email extends \WC_Email {

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

// Register email.
add_filter( 'woocommerce_email_classes', static function( array $emails ): array {
    $emails['NVM_Custom_Email'] = new NVM_Custom_Email();
    return $emails;
} );
```

---

## Block Checkout

Classic hooks (`woocommerce_after_checkout_form`, etc.) may not work with block checkout. Use the Store API `ExtendSchema` for block checkout extensions:

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
                'description' => __( 'Custom checkout field', 'nvm-inventory' ),
                'type'        => 'string',
            ],
        ],
    ] );
} );
```

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

## WooCommerce Hooks Reference

### Order Lifecycle

| Hook | When |
|------|------|
| `woocommerce_new_order` | Order first created |
| `woocommerce_order_status_changed` | Any status transition |
| `woocommerce_order_status_{from}_to_{to}` | Specific transition |
| `woocommerce_payment_complete` | Payment received |
| `woocommerce_order_refunded` | Refund processed |

### Product

| Hook | When |
|------|------|
| `woocommerce_product_set_stock` | Stock quantity changed |
| `woocommerce_low_stock` | Stock reaches low threshold |
| `woocommerce_no_stock` | Stock reaches zero |
| `woocommerce_update_product` | Product saved |

### Cart & Checkout

| Hook | When |
|------|------|
| `woocommerce_add_to_cart` | Item added to cart |
| `woocommerce_before_calculate_totals` | Before cart totals calculated |
| `woocommerce_checkout_process` | Checkout validation |
| `woocommerce_checkout_order_processed` | Order created from checkout |

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
