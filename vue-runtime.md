# Proposal for loading built-in Vue runtime + components

> We can use Blade's PHP component files to load the Vue runtime on-demand.

Theme developers who want to use PKP's built-in Vue runtime or components call `requiresVueRuntime()` in the `init()` method:

```php
class EidosTheme extends ThemePlugin {
    public function init() {
        $this->requiresVueRuntime();
    }
}
```

Currently, OJS calls this automatically on the article page, so that the JS is loaded on every article page.

```php
class ArticleHandler extends Handler {
  public function view(...) {
    $templateMgr->requiresVueRuntime();
  }
}
```

I propose that we use Blade's PHP component files to load the Vue runtime on-demand. From a theme developer's point of view, they would use the built-in components for Open Peer Review and Comments as Blade components.

```html
<!-- article.blade -->

<x-comments />
```

This would be backed by a PHP component that loaded the Vue runtime.

```php
<?php
namespace PKP\view\components;

use APP\core\Application;
use Illuminate\View\Component;

class Comments extends Component
{
    public function __construct() {
        $activeTheme->loadVueCore()
        $activeTheme->loadVueComments();
    }
}
```

This would make the calls to `$templateMgr->loadScript(...)` as necessary. Then the component layout template would call the Vue component.

```html
<!-- view/components/open-peer-review.blade -->

<pkp-comments v-bind="..."></pkp-comments>
```

Benefits:

1. Vue runtime is loaded on-demand.
2. Theme logic is expressed in templates. Theme developers don't need to mess with PHP.

Drawbacks/challenges:

1. We won't be able to load `build_frontend.css` because the `<head>` element will already be rendered. It looks like just some icon styles might need to be handled.
2. The required `localeKeys` are already compiled into the `window.pkp` object, so these need to be moved into the component rendering logic somehow.
