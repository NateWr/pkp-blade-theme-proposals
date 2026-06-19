# Theme and Plugin Initialization Order

Themes (and other plugins) often want to want to know whether another plugin is active or configured appropriately before creating a theme option. For example, a theme may wish to:

- add a homepage block when the Partner Logos plugin is active
- add theme options when the Google Fonts plugin has enabled one or more custom fonts

However, the theme's `init()` function is usually called before other plugins have been loaded and registered. As a result, the following code can not be used in the theme's `init()` function:

```php
$plugins = PluginRegistry::getAllPlugins();
foreach ($plugins as $plugin) {
  //
}
```

This effects the `BlocksRegistry` pattern that I am using for [MetadataBlocksRegistry](https://github.com/NateWr/pkp-lib/blob/eidos/classes/view/MetadataBlocksRegistry.php) and [HomepageBlocksRegistry](https://github.com/NateWr/pkp-lib/blob/eidos/classes/view/HomepageBlocksRegistry.php). The registry allows blocks to be created by itself (pkp-lib, ojs), the theme and any parent themes, as well as generic plugins, using the `HasMetadataBlocks` and `HasHomepageBlocks` interfaces.

The theme needs to be able to call `BlocksRegistry::get()` to get a list of all registered blocks early enough to include them as theme options. This means that all enabled plugins need to be loaded and initialized when the theme calls `$this->addOption(...)`.

The theme needs to be able to call `BlocksRegistry::load()` from within a template, so that blocks can be initialized and passed to the template data.

(This problem is primarily about `generic` plugins, although a broader solution to support all plugins would be nice too.)

## Workarounds

There are a couple of current approaches that can be used to address this.

### Database Access

The theme can go directly to the settings in the database:

```php
/** @var PluginSettingsDAO $pluginSettingsDao */
$pluginSettingsDao = DAORegistry::getDAO('PluginSettingsDAO');
$enabledFonts = $pluginSettingsDao->getSetting($contextId, 'googlefontsplugin', 'fonts');
```

This requires an extra call to the database for each plugin (although I think due to the way that `$pluginSettingsDao->getSetting(...)` works, it may only do it once per plugin).

When using this approach, the `enabled` setting needs to be checked or the theme risks loading settings for a disabled plugin. It's easy to forget, and I've just now noticed that this bug exists in my [recommended approach for Google Fonts](https://github.com/NateWr/googleFonts/#custom-theme-integration).

### Use a hook

The theme can use a `Hook` to run all of the code later than the `init()` function.

```php
Hook::add('TemplateManager::display', [$this, 'addTemplateData']);
```

This is more reliable, but not very intuitive. It also means that all code has to run at the _last second_.

I am not sure, for instance, if the form data for theme options has already been generated to pass to the Vue handler. If so, then theme options added at this stage would not get into the form. The theme would have to load and modify the whole settings object passed to the Vue frontend.

In short, some theme options have to be handled differently from others.

## Suggestions

Should we have a simple, consistent, documented approach for handling all theme setup code? Some options could be:

1. Use a defined callback in `ThemePlugin` or `Plugin` to run code after initial "setup" is completed.

```php
class ThemePlugin
{
  public function pluginsLoaded(): void
  {
    //
  }
}
```

2. Does Laravel's Blade/View library have any built-in events similar to the `TemplateManager::display` hook, which we might use here? I am imagining something like this psuedo-code:

```php
view()->setup(function(Request $resquest) {
  // do all my theme stuff
});
```

And this could be built-in to the `ThemePlugin` for consistency, like this:

```php
class ThemePlugin
{
  public function initAfter(): void
  {
    view()->setup([$this, 'setup']);
  }

  public function setup(Request $request): void
  {
    // Override this method in each theme
  }
}
```

This overlaps a bit with the `init()` function, so it would take some thinking about what would go into each function.
