# OG context access callback

> Provide helper function for having OG context part of menu access callbacks.

## Overview 

There are such situations where you would like to have OG context used in your menu item's
access callback.

In different scenarios, such as having [OG Purl](https://drupal.org/project/og_purl) module enabled,
using OG context in the access callback will not work.

## Usage

* Enable the module
* Under `admin/config/group/context` enable the `OG context in access callback` OG context handler and make sure it's 
the first handler in the list. 

## Technical Overview

Let's assume OG purl is indeed enabled. Upon it's `hook_init()` OG context is trying to be determined.
OG context calls `og_context_determine_context()` that itself calls `menu_get_item()`. By trying to get
the current menu item it will trigger our access callback. So if we try to use `og_context` in that place, no context is 
set, so the access will be FALSE.
`menu_get_item()` statically caches the access results, which means we don't have a second chance to wait for the context 
to be properly fetched, and then act upon it. 
 
This is where the module steps in. We take advantage of the fact we can `drupal_static_reset()` our specific paths that 
should get a second chance for determining the access. So, inside our access callback we first use one of OG context 
utility functions to determine if OG context was already retrieved, if not, we register the path and can wait for the 
callback to be called for the second time with the OG context that is already properly set.




```php
/**
 * Implements hook_menu().
 */
function example_menu() {
  $items['node/%node/test-og-context'] = array(
    'title' => 'Test OG context',
    'access callback' => 'example_test_og_context_access',
    'page callback' => 'example_test_og_context',
  );

  return $items;
}

/**
 * Access callback; Determine if user has "administer group" OG permission.
 */
function example_test_og_context_access() {
  if (!og_context_is_init()) {
    // OG context was not determined yet, so register the path and return early.
    // The next time this access callback will be called, it will not enter
    // here.
    og_context_access_callback_register_path($_GET['q']);
    return;
  }

  $og_context = og_context();
  return og_user_access($og_context['group_type'], $og_context['gid'], 'administer group');
}

/**
 * Page callback.
 */
function example_test_og_context() {
  return t('Only members with "administer group" permission can see this');
}
```
