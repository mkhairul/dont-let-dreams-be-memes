# "New Revision" Admin Bar Opens Wrong Post

## Problem

When an editor visits the **Insights landing page** and clicks "New Revision" in the admin bar, it opens an edit window for the **most recently published article** instead of the landing page itself.

## Root Cause

The **Revisionary Pro** plugin's admin bar handler uses `global $post` to determine which post the "New Revision" link should point to. By the time the admin bar renders, the `$post` global has been **corrupted** by blocks that run custom `WP_Query` loops during page rendering.

### How it works step-by-step:

1. WordPress begins rendering the **Insights landing page** → `global $post` = the landing page
2. The page content includes blocks like `latest-posts` and `featured-posts`
3. These blocks run `WP_Query` loops with `$query->the_post()` which **overwrite `global $post`** to each article
4. While `wp_reset_postdata()` is called after the main loop, additional `WP_Query` instances in the `latest-posts` block (for filtering formats/topics) may further contaminate the state
5. When the admin bar renders, `global $post` now points to the **last article** processed, not the landing page

### The specific code path:

**Revisionary's admin bar handler** — [revisionary_main.php:307-330](file:///Users/khairulanuar/Documents/GitHub/rbc-wealth-management/plugins/revisionary-pro/lib/vendor/publishpress/publishpress-revisions/revisionary_main.php#L307-L330):

```php
function adminToolbarItem($admin_bar) {
    global $post;  // ← THIS IS THE PROBLEM
    
    if (!empty($post) && ...) {
        $admin_bar->add_menu([
            'href' => rvy_admin_url("admin.php?page=rvy-revisions&amp;post={$post->ID}&amp;action=revise&amp;front=1"),
            // ↑ Uses $post->ID which is now the WRONG post
        ]);
    }
}
```

**The latest-posts block** — [class-latest-posts.php:484-519](file:///Users/khairulanuar/Documents/GitHub/rbc-wealth-management/plugins/rbc-wealth-management-blocks/blocks/latest-posts/class-latest-posts.php#L484-L519):

```php
while ( $posts_query->have_posts() ) :
    $posts_query->the_post();  // ← Overwrites global $post
    // ... render each article ...
endwhile;
wp_reset_postdata();  // Tries to restore, but additional queries after this may re-corrupt
```

After the main loop, additional `WP_Query` calls at [lines 539](file:///Users/khairulanuar/Documents/GitHub/rbc-wealth-management/plugins/rbc-wealth-management-blocks/blocks/latest-posts/class-latest-posts.php#L539) and [565](file:///Users/khairulanuar/Documents/GitHub/rbc-wealth-management/plugins/rbc-wealth-management-blocks/blocks/latest-posts/class-latest-posts.php#L565) run without calling `wp_reset_postdata()` afterward, which can leave the global state contaminated.

## Fix

The fix should be in the **Revisionary Pro plugin's `adminToolbarItem()` method**. Instead of relying on `global $post` (which is fragile and can be corrupted by any block/widget), use `get_queried_object()` which always returns the actual page/post being viewed.

### Proposed change in [revisionary_main.php](file:///Users/khairulanuar/Documents/GitHub/rbc-wealth-management/plugins/revisionary-pro/lib/vendor/publishpress/publishpress-revisions/revisionary_main.php#L307-L330):

```diff
 function adminToolbarItem($admin_bar) {
-    global $post;
+    $queried = get_queried_object();
+    $post = ($queried instanceof \WP_Post) ? $queried : null;
 
     if (!empty($post) && rvy_get_option('pending_revisions') && ...) {
```

> [!WARNING]
> This file is part of the **Revisionary Pro vendor package** (`plugins/revisionary-pro/lib/vendor/publishpress/publishpress-revisions/`). Editing it directly means the change will be **overwritten on plugin update**. Consider one of these alternatives:
> 
> 1. **Theme-level workaround**: Remove and re-add the admin bar item with the correct post ID via `functions.php`
> 2. **Report upstream**: File a bug with PublishPress — this is a known pattern issue in WordPress plugins
> 3. **MU-plugin fix**: Create a must-use plugin that corrects the admin bar URL

### Alternative: Theme-level workaround (recommended)

Add this to `functions.php` to fix the admin bar link without modifying the vendor plugin:

```php
/**
 * Fix Revisionary Pro "New Revision" admin bar link using the correct post ID.
 * The plugin uses global $post which can be corrupted by block rendering loops.
 */
add_action('admin_bar_menu', function($admin_bar) {
    // Only act on the front end where the bug occurs
    if (is_admin()) {
        return;
    }

    $node = $admin_bar->get_node('rvy-create-revision');
    if (!$node) {
        return;
    }

    // Get the actual page being viewed (not the corrupted global $post)
    $queried = get_queried_object();
    if (!$queried || !($queried instanceof \WP_Post)) {
        return;
    }

    // Rebuild the href with the correct post ID
    $correct_href = rvy_admin_url(
        "admin.php?page=rvy-revisions&amp;post={$queried->ID}&amp;action=revise&amp;front=1"
    );

    $admin_bar->add_menu([
        'id'    => 'rvy-create-revision',
        'href'  => $correct_href,
    ]);
}, 101); // Priority 101 = runs after Revisionary's priority 100
```

> [!IMPORTANT]
> This theme-level fix runs at priority 101 (after Revisionary's priority 100), so it overwrites the incorrect URL with the correct one derived from `get_queried_object()`.
