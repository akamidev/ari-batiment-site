# PARIS DECOR — Site vitrine BTP

> **Auteur :** AKAMI – Développeur
> **Contact :** [contact@ari-batiment.com](mailto:contact@ari-batiment.com) · 06 99 85 80 05

## Projet

* Site vitrine pour **peinture, vitrerie, carrelage, parquet**.
* Mettre en avant les **réalisations** et capter des **demandes de devis**.
* Optimisé pour le **SEO local** (Saint‑Gratien · Val‑d’Oise · Île‑de‑France).
* Design **responsive**, rapide et propre.

## Stack & structure

* **Front :** HTML5, CSS, JavaScript
* **Librairies :** AOS (animations), Swiper (slider), GLightbox (lightbox), Bootstrap Icons
* **Assets :** images WebP dans `assets/img/`
* **Code :** `assets/css/main.css` (styles) · `assets/js/main.js` (scripts) · `assets/vendor/*` (dépendances)
* **Pages :** `index.html`, `services.html`, `projects.html`, `project-details.html`, `quote.html`, `contact.html`, `team.html`, `terms.html`, `privacy.html`




/**
 * GemConnect – Paywall vendeur via Woo + WCFM (plans HivePress/produits Woo)
 * Coller dans functions.php (thème enfant) ou créer un mu-plugin.
 */

/** 0) CONFIG — IDs produits WooCommerce de tes plans */
function gcp_membership_product_ids() {
    return [
        637, // Diamant – Mensuel
        638, // Diamant – 6 mois
        664, // Saphir – Mensuel
        665, // Saphir – 6 mois
        666, // Tsavorite – Mensuel
        667, // Tsavorite – 6 mois
    ];
}

/** 1) Helper — savoir si l’utilisateur a déjà acheté un plan */
function gcp_user_has_plan( $user_id ) {
    $ids = (array) get_user_meta( $user_id, '_gcp_plan_ids', true );
    return ! empty( $ids );
}

/** 2) À la validation commande → si un plan est acheté => rôle vendeur + trace */
function gcp_grant_vendor_on_order( $order_id ) {
    $order = wc_get_order( $order_id );
    if ( ! $order ) return;

    $user_id = $order->get_user_id();
    if ( ! $user_id ) return;

    $plan_ids = gcp_membership_product_ids();
    $bought_any_plan = false;
    $bought = [];

    foreach ( $order->get_items() as $item ) {
        $pid = (int) $item->get_product_id();
        if ( in_array( $pid, $plan_ids, true ) ) {
            $bought_any_plan = true;
            $bought[] = $pid;
        }
    }

    if ( ! $bought_any_plan ) return;

    // Donne le rôle vendeur WCFM
    $user = new WP_User( $user_id );
    $user->add_role( 'wcfm_vendor' );
    $user->remove_role( 'customer' );

    // Mémorise les plans achetés (simple flag)
    $existing = (array) get_user_meta( $user_id, '_gcp_plan_ids', true );
    $new = array_values( array_unique( array_merge( $existing, $bought ) ) );
    update_user_meta( $user_id, '_gcp_plan_ids', $new );
    update_user_meta( $user_id, '_gcp_became_vendor_at', time() );
}
// On capte "processing" ET "completed" pour couvrir Stripe + autres modes
add_action( 'woocommerce_order_status_processing', 'gcp_grant_vendor_on_order' );
add_action( 'woocommerce_order_status_completed',  'gcp_grant_vendor_on_order' );

/** 3) Sécurité — interdire d’ajouter/publier des produits sans plan */
add_filter( 'wcfm_is_allow_add_products', function( $allow ) {
    $uid = get_current_user_id();
    if ( ! $uid ) return false;

    // Doit être vendeur WCFM
    if ( ! user_can( $uid, 'wcfm_vendor' ) ) return false;

    // Doit avoir acheté au moins un plan (IDs ci-dessus)
    if ( ! gcp_user_has_plan( $uid ) ) return false;

    return true;
}, 999 );

/** 4) Bloquer l’accès dashboard si pas vendeur/plan → redirige vers page Offres */
add_action( 'template_redirect', function () {
    if ( ! is_user_logged_in() ) return;

    $uid = get_current_user_id();
    $uri = $_SERVER['REQUEST_URI'] ?? '';

    $is_store_manager = ( strpos( $uri, '/store-manager/' ) !== false );
    if ( ! $is_store_manager ) return;

    $needs_redirect = ( ! user_can( $uid, 'wcfm_vendor' ) || ! gcp_user_has_plan( $uid ) );
    if ( $needs_redirect ) {
        //  Remplacer /plans/ par l’URL de ta page "Offres / Devenir vendeur"
        wp_safe_redirect( site_url( '/plans/' ) );
        exit;
    }
});

/** 5) Page de remerciement → bouton d’accès au Store Manager */
add_action( 'woocommerce_thankyou', function( $order_id ) {
    if ( ! $order_id ) return;
    $order = wc_get_order( $order_id );
    if ( ! $order ) return;

    $uid = $order->get_user_id();
    if ( ! $uid ) return;

    if ( user_can( $uid, 'wcfm_vendor' ) && gcp_user_has_plan( $uid ) ) {
        echo '<p style="margin-top:16px"><a class="button" href="' . esc_url( site_url('/store-manager/') ) . '">Aller au Store Manager</a></p>';
    }
}, 20 );

/** 6) Après connexion, les vendeurs vont directement au Store Manager (UX) */
add_filter( 'woocommerce_login_redirect', function( $redirect, $user ) {
    if ( $user && user_can( $user, 'wcfm_vendor' ) ) {
        return site_url( '/store-manager/' );
    }
    return $redirect;
}, 10, 2 );

/* =================== Debut interdicition acheter mon propre produit =================== */


// 1) Masquer le bouton "Ajouter au panier" si le vendeur regarde son propre produit
add_filter('woocommerce_is_purchasable', function($purchasable, $product){
    if (!is_user_logged_in()) return $purchasable;
    $owner_id = get_post_field('post_author', $product->get_id());
    return (get_current_user_id() != intval($owner_id));
}, 10, 2);

// 2) Blocage dur côté panier (sécurité)
add_filter('woocommerce_add_to_cart_validation', function($passed, $product_id){
    if (!is_user_logged_in()) return $passed;
    $owner_id = get_post_field('post_author', $product_id);
    if (get_current_user_id() == intval($owner_id)) {
        wc_add_notice(__("Vous ne pouvez pas acheter vos propres produits."), 'error');
        return false;
    }
    return $passed;
}, 10, 2);

// 3) Sécurité checkout (si l’article a été ajouté via un autre flux)
add_action('woocommerce_checkout_process', function(){
    if (!is_user_logged_in()) return;
    $user_id = get_current_user_id();
    foreach (WC()->cart->get_cart() as $item){
        $owner_id = get_post_field('post_author', $item['product_id']);
        if ($user_id == intval($owner_id)) {
            wc_add_notice(__("Votre panier contient vos propres produits, achat interdit."), 'error');
        }
    }
});
/* =================== fin interdicition acheter mon propre produit =================== */


