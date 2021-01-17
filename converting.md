# Converting modules to Backdrop from Drupal 7

## Big picture

Backdrop's codebase is similar to Drupal 7 and generally needs very  few changes to convert. To make things even easier, Backdrop includes a  Drupal compatibility layer which should enable Drupal functions (even  those with the format '`drupal_XXX`', such as `drupal_set_message`),  to just work, for now. However, Backdrop still has significant  advantages over Drupal 7, and it would make sense to fully port your  module rather than depend on the compatibility layer. 

This guide suggests a process for converting Drupal 7 modules to  Backdrop, and the common issues which might arise during the conversion.  (If your module is not Drupal version 7, we recommend you follow  Drupal's module conversion guidelines to upgrade (or downgrade) to  Drupal 7 before continuing.)

[A useful video by @quicksketch demonstrates most of the main issues in this document](https://www.youtube.com/watch?v=by8mMhIpARA)

## Before you begin

Before you start porting a module, please create the two recommended issues stated on the [Communicating](https://api.backdropcms.org/converting-from-drupal#communicate) section of the Converting Drupal code page.

## Drupal Code Base to Backdrop Code Base

This would be for a module that had a 2.x release:



```
# Checkout from drupal.org:
git clone --branch 7.x-2.x http://git.drupal.org/project/[project_name].git

cd [project_name]

# Rename the branch.
git branch -m 7.x-2.x 1.x-2.x

# Remove the old origin and point at Github instead.
git remote remove origin
git remote add origin git@github.com:backdrop-contrib/[project_name].git

# Push to the new repository.
git push origin 1.x-2.x
```

You can see all of releases on the project page: `https://www.drupal.org/project/[project_name]/releases`

(Comment: You cannot push into github.com:backdrop-contrib repo, if you are not a member of the backdrop contrib group yet.)

## Modifying your module's .info file

For reference, also see the [developer documentation for Backdrop modules](https://api.backdropcms.org/modules). Here is a comparison of Drupal and Backdrop module info files:



```
name = Book
description = Allows users to create and organize related content in an outline.
package = Core
version = VERSION
core = 7.x
files[] = book.test
files[] = includes/whatever.inc
stylesheets[all][] = book.css
configure = admin/content/book/settings
dependencies[] = node

; Information added by Drupal.org packaging script on 2014-11-19
version = "7.34"
project = "drupal"
datestamp = "1416429488"
```

and here is Backdrop:



```
type = module
name = Book
description = Allows users to create and organize related content in an outline.
package = Core
version = VERSION
backdrop = 1.x
stylesheets[all][] = book.css
configure = admin/content/book/settings
dependencies[] = node
```

The first and most important change is that a new Backdrop-specific core version needs to be added: '`backdrop = 1.x`'.  For modules that would like to support both Drupal 7 and Backdrop at  the same time, the old Drupal core version can also remain, but for  Backdrop-only modules this line should be removed: `core = 7.x`

Backdrop also distinguishes between different types of projects (modules, themes, layouts) by using the line `type = module` and the packaging script on backdropcms.org will fail if the type line is excluded. 

In Backdrop the class registry has also been removed and replaced with a static class map ([see the related change record](https://api.backdropcms.org/node/26859)).  This means that you'll need to remove any lines in your info file that define files containing PHP classes: `files[] = whatever.inc`.  To have this class loaded you'll need to implement `hook_autoload_info()` in your module file instead.

Except the .test files. For example: `files[] = menu_import.test` Do not need `hook_autoload_info()` for test classes. The .tests.info file will load them. See in next section.

in the Drupal 7 .info file:

```
files[] = includes/whatever.inc
```

in the Backdrop .module file:

```php
function hook_autoload_info() {
  return array(
    'ClassName' => 'includes/whatever.inc',
  );
}
```

In the .info file you should remove the version and datestamp as those will be generated automatically by Drupal.org with a release.

There is a new feature in Backdrop: tags in .info file. See [this.](https://api.backdropcms.org/documentation/module-packages-and-tags)

Example:
```
name = Menu Export/Import
description = Import and export menu hierarchy using indented, JSON-like text files.
package = Development
tags[] = Menus
tags[] = Site Architecture
tags[] = Structure
backdrop = 1.x
dependencies[] = menu
type = module
configure = admin/structure/menu/import
```
## Organization of module files

There are now recommended locations for common module files. Javascript files are always kept in a `js` folder, CSS in a `css` folder, templates in a `templates` folder, and tests in a `tests` folder.

Backdrop recommends the following files be kept in the root of the  module folder. Having these files immediately visible makes it more  obvious that this module provides theme functions (and makes them easy  for front-end developers to find), has a settings page, and provides  user-facing pages. If a module provides too many additional .inc files  it would also be reasonable to place those additional files into an `includes` folder also in the module root, or organize further as necessary.

- `modulename.info`
- `modulename.module`
- `modulename.admin.inc`
- `modulename.pages.inc`
- `modulename.theme.inc`
- `modulename.api.php`
- config
- css
- js
- templates
- tests

If your module contains tests, those tests should live within the `tests` directory. Information about the tests should be moved out of the `getInfo()` method in the test class, and instead moved into a new .info file located at `tests/modulename.tests.info`. Check out the [Book test info file](https://api.backdropcms.org/api/backdrop/core!modules!book!tests!book.tests.info/1) for an example.

Another example:

Old:

```php
class MenuImportTestCase extends DrupalWebTestCase {
  public static function getInfo() {
    return array(
      'name' => 'Menu export/import',
      'description' => 'Perform various tests on menu_import module.',
      'group' => 'Menu',
    );
  }
...
```

In the `tests/menu_import.tests.info`:
```
[MenuImportTestCase]
name = Menu export/import
description = Perform various tests on menu_import module.
group = Menu
file = menu_import.test
```
If your module contains the jQuery core file, you should remove it. Backdrop currently ships with updated jQuery.
If the D7 module uses 3rd party scripts, which are in `DRUPAL_ROOT/sites/all/libraries`, then you should create a new directory `BACKDROP_ROOT/libraries`, and place the script files inside this directory. Or you can integrate these files into the module: place inside the `MODULE_ROOT/js` directory, and register with [hook_library_info()](https://api.backdropcms.org/api/backdrop/core%21modules%21system%21system.api.php/function/hook_library_info/1) in `modulename.module`.
If your module contains .make file, you should consider Drush `make` is [no longer maintained.](https://github.com/drush-ops/drush/issues/3946#issuecomment-467861007) The [Composer](https://getcomposer.org/) is the supported solution.

## Update your module to use Configuration management

Drupal 7 stores most module configuration in the `variables` table in the database. A trio of handy functions were provided to manipulate these variables: `variable_get()`, `variable_set()`, and `variable_del()`. 

In Backdrop, configuration is stored in flat files. These files are  often called configuration (or config) files, and end in a .json  extension. By default, active config files are saved into  `BACKDROP_ROOT/files/config_xxxx/active`  (where xxxx is a long alphanumeric string generated during install and  entered into settings.php). Backdrop provides the corresponding `config_get()` and `config_set()` functions to manage the retrieval and writing to disk.

For the purpose of backwards compatibility with Drupal 7, both the variables database table and the '`variable_`' functions still exist in Backdrop. They have been deprecated and will be removed in Backdrop 2.0, so conversion to '`config_`' is highly recommended.

**Deciding on the config storage file(s)**
 To convert variables to config, first identify all the module's current  variables: use your editor, search, or grep. The aim is to get an idea  of the number or variables and type of configuration your module  requires. 

To store configuration, the module will need its own config file. Config files are commonly named '`modulename.settings.json`'.  Sometimes it is appropriate to have more than one config file. For  example, the settings for node module are saved by node type: `node.type.article.json`, and `node.type.page.json`. For a module with complex configuration `modulename.first_set_of.settings.json`, and `modulename.second_set_of.settings.json`  may be a better fit. Think about how you would like to deploy this  configuration. Would you be moving all module's settings at once, or one  type at a time?

**Changing the code**
 Once you have decided on a single config file or several, you will need to replace every instance of a '`variable_`' function. The standard method is as follows:

Instead of

```php
<?php
$variable = variable_get('foo', 'bar');
?>
```

Use

```php
<?php
$variable = config_get('modulename.settings', 'foo');
?>
```

And instead of

```php
<?php
variable_set('foo', $variable);
?>
```

Use

```php
<?php
config_set('modulename.settings', 'foo', $variable);
?>
```

The above examples of config will read and write to a JSON file called `modulename.settings.json` in the active config directory.

An alternative method for saving config, usually when getting or  setting more than one value at a time, is to create a config object and  use the object's get and set methods.



```php
<?php
// Get the full config object
$config = config('modulename.settings');

// Get individual settings
$value_foo = $config->get('foo');
$value_next = $config->get('next');
?>
```

This is especially useful when creating settings form arrays where you may need a '`#default_value`'  for many form elements. It's also the preferred method for saving  values in a form submit handler, where config is being saved:



```php
<?php
// Get the full config object
$config = config('modulename.settings');

// Set individual config settings
$config->set('setting_one', $form_state['values']['value_one']);
$config->set('setting_two', $form_state['values']['value_two']);

// Save
$config->save();
?>
```

**Create your config defaults**
 The function `variable_get()` has the useful capacity to declare default values; writing `$setting = variable_get('my_setting', 'red')`  allows the developer to initialize the value 'red' if 'my_setting'  variable has not been set. In Backdrop, there's no such thing as a  variable that has not been set, and therefore config defaults need to be  explicitly set at module enabling.

The simplest way to set default values for your config variables is  to provide a default config file. This file should be placed into a `config` directory within your module: `config/modulename.settings.json`.  When the module is enabled for the first time, the config file will be  copied from this location into the active config directory.



```
BACKDROP_ROOT/modules
    modulename
        modulename.info
        modulename.module
        config
            modulename.settings.json
```

Here is a sample config file to get started. Note the lack of a trailing comma after the last value; this is a JSON requirement.



```
{
  "_config_name": "module_name.settings",
  "setting_one": "default value 1",
  "setting_two": 2,
  "array_one": ["value_1", "value_2"],
  "empty_array": []
}
```

**Let Backdrop know about your module's config**
 Modules need to declare config files in `hook_config_info()` in `modulename.module`. This is how Backdrop knows that you have a config file that needs to  copied when the module is enabled. It's also how Backdrop compares changes in config during deployment.



```php
<?php
/**
 * Implements hook_config_info().
 */
function contact_config_info() {
  $prefixes['modulename.first_set_of.settings'] = array(
    'label' => t('First set of settings'),
    'group' => t('My Module'),
  );
  $prefixes['modulename.second_set_of.settings'] = array(
    'label' => t('Second set of settings'),
    'group' => t('My Module'),
  );
  return $prefixes;
}
?>
```

**Create an upgrade path**
 At this stage the module should be nearly completely converted to CMI;  new installs will use CMI exclusively for storing config. However,  existing users with this module installed on a Drupal site and who wish  to convert to Backdrop will still need a process to convert existing  settings stored as variables to the config system.

Backdrop uses the [hook_update_N()](https://api.backdropcms.org/api/backdrop/core%21modules%21system%21system.api.php/function/hook_update_N/1) system for upgrades  and updates in `modulename.install`. For initial porting to Backdrop 1.x N should be in the  1000s.  In this update, retrieve the values of existing variables and  store them as config, then delete the variables. For example:



```php
<?php
/**
 * Move book settings from variables to config.
 */
function book_update_1000() {
  // Migrate variables to config.
  $config = config('book.settings');
  $config->set('book_allowed_types', update_variable_get('book_allowed_types', array('book')));
  $config->set('book_child_type', update_variable_get('book_child_type', 'book'));
  $config->set('book_block_mode', update_variable_get('book_block_mode', 'all pages'));
  $config->save();

  // Delete variables.
  update_variable_del('book_allowed_types');
  update_variable_del('book_child_type');
  update_variable_del('book_block_mode');
}
?>
```

If you have renamed a column in a database table, you have to use [`db_change_field()`](https://api.backdropcms.org/api/backdrop/core%21includes%21database%21database.inc/function/db_change_field/1) function.
For example when 'drupal' string is replaced with 'backdrop' in the name of a column:



```php
<?php
/**
 * Rename a column in the 'book_records' table.
 */
function book_update_1001() {
  db_change_field('book_records', 'drupal_user', 'backdrop_user',
    array('type' => 'varchar', 'length' => 255, 'not null' => FALSE));
}
?>
```

Note that when upgrading a module, all previous updates should be  deleted. If a module has updates numbered 6xxx or 7xxx, be sure to  remove all these updates.

When removing these updates, you should provide an implementation of [`hook_update_last_removed()`](https://api.backdropcms.org/api/backdrop/core!modules!system!system.api.php/function/hook_update_last_removed) in `modulename.install` to indicate which version of the module schema you're expecting when an upgrade is performed:



```php
<?php
/**
 * Implements hook_update_last_removed().
 */
function modulename_update_last_removed() {
  return 7012;
}
?>
```

**Using system_settings_form()**

> "This function adds a submit handler and a submit button  to a form array. The submit function saves all the data in the form,  using variable_set(), to variables named the same as the keys in the  form array."

Since variables are not used in Backdrop `system_settings_form()` has been updated to save to a single config file. If your module saves all the settings in it's form to a single file, you'll need to add a new `#config` key to the form array so `system_settings_form()` knows where to save the values in `modulename.module`.

Old:



```php
<?php
function modulename_settings_form($form, $form_state) {
  $form['modulename_my_setting'] = array(
    '#type' => 'textfield',
    '#title' => t('My setting'),
    '#default_value' => variable_get('modulename_my_setting', ''),
  );

  return system_settings_form($form);
}
?>
```

New:



```php
<?php
function modulename_settings_form($form, $form_state) {
  $form['#config'] = 'modulename.settings';
  $form['my_setting'] = array(
    '#type' => 'textfield',
    '#title' => t('My setting'),
    '#default_value' => config_get('modulename.settings', 'my_setting', ''),
  );

  return system_settings_form($form);
}
?>
```

If you need to save values into multiple config files, we recommend  that you add your own submit button, and submit handler for the form  instead, as follows.



```php
<?php
function modulename_settings_form($form, $form_state) {
  $config = config('modulename.settings');

  $form['my_setting'] = array(
    '#type' => 'textfield',
    '#title' => t('My setting'),
    '#default_value' => $config->get('my_setting'),
  );

  // Add a submit button
  $form['actions']['#type'] = 'actions';
  $form['actions']['submit'] = array(
    '#type' => 'submit',
    '#value' => t('Save configuration'),
  );

  return $form;
}

/**
 * Submit handler for module_settings_form().
 */
function modulename_settings_form_submit($form, &$form_state) {
  $config = config('modulename.settings');
  $config->set('my_setting', $form_state['values']['my_setting']);
  $config->save();

  $config = config('modulename.other-settings');
  $config->set('my_other_setting', $form_state['values']['my_other_setting']);
  $config->save();

  backdrop_set_message(t('The configuration options have been saved.'));
}
?>
```

Notice how with the new approach, you can remove the module name from  the name of your variables (if you like) since the variables are saved  in individual config files that contain the module name.

## Classed entities

Several entities in Backdrop are now classed objects which extend the Entity Class. **Users**, **nodes**, **comments**, **files**, **taxonomy terms** and **vocabularies**  have been converted to extend the new base class. Therefore  programmatic creation of these objects has changed in Backdrop. For  example:

Old

```php
$node = new stdClass();
$node->type = 'blog';
$node->title = 'New Title';
...
node_save($node);
```

New

```php
$node = entity_create('node', array ('type' => 'blog'));
$node->uid = $user->uid;
$node->title = 'New Title';
$node->save();
```

You should search this functions in the code ([further details](https://api.backdropcms.org/node/26881)):

- `node_save()`, `node_load()`, `node_delete()`
- `user_save()`, `user_load()`, `user_delete()`
- `file_save()`, `file_load()`, `file_delete()`
- `comment_save()`, `comment_load()`, `commentr_delete()`
- `taxonomy_vocabulary_save()`, `taxonomy_vocabulary_load()`, `taxonomy_vocabulary_delete()`

Node entity has a `langcode` property instead of `language`property. Replace it:

Old:

```php
$node = new stdClass();
$node->language = LANGUAGE_NONE;
```

New:

```php
$node = entity_create('node', array ('type' => 'blog'));
$node->langcode = LANGUAGE_NONE;
```

## Core modules removed

There are several modules that were included in Drupal 7 that  have been removed from Backdrop ([see complete list ](https://backdropcms.org/node/1682)). If your module implemented APIs for any of these modules, those hooks can be safely removed.
 A commonly-used Drupal function was `hook_help()` in `modulename.module`. As the Help module  was removed from Backdrop, this function no longer exists and so you  should instead display any necessary help text on forms/pages directly,  or in your module's GitHub Wiki.

Leaving these hooks in your code will not affect a Backdrop module in  any way, since the code will simply not be called. You may leave them  in place if you would like to be able run the same module on both Drupal  and Backdrop sites.

## Features added to core
The most commonly used Drupal 7 features are being added into Backdrop core. Check the [list of modules](https://backdropcms.org/node/1683) those are added into Backdrop core, or no longer necessary include.
If any of these is a dependency of your module, and integrated into System module, you should remove the `dependencies[]` from `modulname.info`. You should rewrite the `dependencies[]` if someone is replaced or integrated into an another module.
The integrated modules' functions may have changed. In this case you may need to fix the function call. For example: The functions of Libraries API module are changed: See [this.](https://github.com/backdrop-contrib/libraries/blob/1.x-2.x/README.md)

## Install and uninstall hooks

If the module implements hook_schema(), [hook_install](https://api.backdropcms.org/api/backdrop/core!modules!system!system.api.php/function/hook_install/1)  will create the schema. So, `backdrop_install_schema()` or `drupal_install_schema()` is not needed to  be called in hook_install(). Similar for `backdrop_uninstall_schema()` or `drupal_uninstall_schema()`.  Check the .install file to see if these hooks can be removed.

If your module provides config files, those files will be deleted automatically if your module has an implementation of `hook_uninstall().`

## Without Drupal compatibility

Drupal compatibility is enabled by default in the variable `backdrop_drupal_compatibility` within `BACKDROP_ROOT/settings.php`. The last step to completely port the module to Backdrop CMS, is to check that it works well with the Drupal compatibility disabled. Before it you should replace `drupal` with `backdrop` in all the files of the module, and again `Drupal` with `Backdrop`, and `DRUPAL` with `BACKDROP`. This is case sensitive. With this very easy step we avoid using deprecated functions (could also be the cause of warnings or errors.)

You can also use the [Coder Upgrade modul](https://backdropcms.org/project/coder_upgrade) for replacement and it has other useful features.

## Testing

Some suggestions:

- Set the "All messages" option on the admin/config/development/logging page.
- Log page: admin/reports/dblog
- If your module contains a .test file, then you should enable the Testing module, and run the module's test on the admin/config/development/testing page. If you encounter some error message, you may need [TestCase's classes descriptions.](https://api.backdropcms.org/api/backdrop/core%21modules%21simpletest%21backdrop_web_test_case.php/1)

## Summary

These are a few of the common module changes which will be required  for porting. However, other less common API changes exist. See the [change records](https://api.backdropcms.org/change-records) for a full list. Use the search bar on this page if you encounter an error message. If you find an unlisted API change, please report this on the [Backdrop Issue Queue](https://github.com/backdrop/backdrop-issues/issues).

Some examples:

- Search result of „hook_page_build”: [New Layout module](https://api.backdropcms.org/change-records/new-layout-module-implements-drag-drop-model-building-layouts) -> $page variable has been removed
- If the module uses the language key object within the `$message` array parameter in `hook_mail()`, now has a `langcode` property instead of `language` property. Replace it!

You can learn from the code of previously converted modules. Select a module on the [Backdrop modules' site](https://backdropcms.org/modules), follow the "Project page" link to Github, and study the commits.

If your converted module is a contrib module, and you want to publish on Backdropcms.org, then read [this document.](https://github.com/backdrop-ops/contrib#backdrop-contributed-project-agreement)

## Tagging for release

When tagging your Backdrop module for release, please use the same  version number as the Drupal module being ported. For example, if you  are porting the 7.x-4.15 version of webform module, that should become  the 1.x-4.15.0 version of the Backdrop module. Any bug fixes would  increment the last number to 4.15.1 or 4.15.2.  

When a new version of the Drupal module comes out (7.x-4.16) the  minor-version number of the backdrop version can also be updated to  match (1.x-4.16.0) and that version can contain all/any commits from the  Drupal project you'd like to include.

If the Backdrop and Drupal version numbers do not match for any  reason, the Backdrop release description should contain the Drupal  version that is comparable.

Happy converting!