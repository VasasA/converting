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

in the Drupal 7 .info file:

```
files[] = includes/whatever.inc
```

in the Backdrop .module file:

```
function hook_autoload_info() {
  return array(
    'ClassName' => 'includes/whatever.inc',
  );
}
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

```
<?php
$variable = variable_get('foo', 'bar');
?>
```

Use

```
<?php
$variable = config_get('modulename.settings', 'foo');
?>
```

And instead of

```
<?php
variable_set('foo', $variable);
?>
```

Use

```
<?php
config_set('modulename.settings', 'foo', $variable);
?>
```

The above examples of config will read and write to a JSON file called `modulename.settings.json` in the active config directory.

An alternative method for saving config, usually when getting or  setting more than one value at a time, is to create a config object and  use the object's get and set methods.



```
<?php
// Get the full config object
$config = config('modulename.settings');

// Get individual settings
$value_foo = $config->get('foo');
$value_next = $config->get('next');
?>
```

This is especially useful when creating settings form arrays where you may need a '`#default_value`'  for many form elements. It's also the preferred method for saving  values in a form submit handler, where config is being saved:



```
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
 Modules need to declare config files in `hook_config_info()`.  This is how Backdrop knows that you have a config file that needs to  copied when the module is enabled. It's also how Backdrop compares  changes in config during deployment.



```
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

Backdrop uses the `hook_update_N()` system for upgrades  and updates. For initial porting to Backdrop 1.x N should be in the  1000s.  In this update, retrieve the values of existing variables and  store them as config, then delete the variables. For example:



```
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

Note that when upgrading a module, all previous updates should be  deleted. If a module has updates numbered 6xxx or 7xxx, be sure to  remove all these updates.

When removing these updates, you should provide an implementation of [`hook_update_last_removed()`](https://api.backdropcms.org/api/backdrop/core!modules!system!system.api.php/function/hook_update_last_removed) to indicate which version of the module schema you're expecting when an upgrade is performed:



```
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

Since variables are not used in Backdrop `system_settings_form()`  has been updated to save to a single config file. If your module saves  all the settings in it's form to a single file, you'll need to add a new  `#config` key to the form array so `system_settings_form()` knows where to save the values. 

Old:



```
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



```
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



```
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

```
$node = new stdClass();
$node->type = 'blog';
$node->title = 'New Title';
...
node_save($node);
```

New

```
$node = entity_create('node', array ('type' => 'blog'));
$node->uid = $user->uid;
$node->title = 'New Title';
$node->save();
```

## Core modules removed

There are several modules that were included in Drupal 7 that  have been removed from Backdrop ([see complete list ](https://backdropcms.org/upgrade-from-drupal)). If your module implemented APIs for any of these modules, those hooks can be safely removed.
 A commonly-used Drupal function was `hook_help()`. As the Help module  was removed from Backdrop, this function no longer exists and so you  should instead display any necessary help text on forms/pages directly,  or in your module's GitHub Wiki.

Leaving these hooks in your code will not affect a Backdrop module in  any way, since the code will simply not be called. You may leave them  in place if you would like to be able run the same module on both Drupal  and Backdrop sites.

## Install and uninstall hooks

If the module implements hook_schema(), [hook_install](https://api.backdropcms.org/api/backdrop/core!modules!system!system.api.php/function/hook_install/1)  will create the schema. So, backdrop_install_schema() is not needed to  be called in hook_install(). Similar for backdrop_uninstall_schema().  Check the .install file to see if these hooks can be removed.

If your module provides config files, those files will be deleted automatically if your module has an implementation of `hook_uninstall().`

## Summary

These are a few of the common module changes which will be required  for porting. However, other less common API changes exist. See the [change records](https://api.backdropcms.org/change-records) for a full list. If you find an unlisted API change, please report this on the [Backdrop Issue Queue](https://github.com/backdrop/backdrop-issues/issues). 

## Tagging for release

When tagging your Backdrop module for release, please use the same  version number as the Drupal module being ported. For example, if you  are porting the 7.x-4.15 version of webform module, that should become  the 1.x-4.15.0 version of the Backdrop module. Any bug fixes would  increment the last number to 4.15.1 or 4.15.2.  

When a new version of the Drupal module comes out (7.x-4.16) the  minor-version number of the backdrop version can also be updated to  match (1.x-4.16.0) and that version can contain all/any commits from the  Drupal project you'd like to include.

If the Backdrop and Drupal version numbers do not match for any  reason, the Backdrop release description should contain the Drupal  version that is comparable.

Happy converting!