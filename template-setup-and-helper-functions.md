# Template setup and helper functions

In my original [layout proposal](./old-layout.md), I suggested using a `<x-layout>` component to setup template data and provide common helper functions. That approach has been adopted with a few adjustments.

In this document, I outline some of the challenges that I've encountered while using this approach.

## Goal

My main goal is to isolate theme/frontend code from `PKPTemplateManager`. In an ideal future, `PKPTemplateManager` only houses code necessary for the backend. For the theme/frontend, there would be a smaller class dedicated to supporting themers that would ideally allow a themer to easily use, override, and extend helper functions and data accessors.

## Current Approach

Every page in the Eidos theme uses the layout component:

```php
<x-eidostheme::layout>
  // Page content
<x-eidostheme::layout>
```

This layout is backed by a Blade component class, which extends the built-in `Layout` class.

```php
namespace APP\plugins\themes\eidos\classes\components;

class Layout extends \APP\view\components\Layout
{
    protected function addGlobalData(): void
    {
        parent::addGlobalData();
        view()->share('example', 'Example test data');
    }
}
```

Common global data is provided by the parent class at OJS/pkp-lib level:

```php
namespace APP\view\components;

class Layout extends \PKP\view\components\Layout
{
    protected function addGlobalData(): void
    {
        parent::addGlobalData();
        view()->share('issn', $contrext->getData('printIssn'));
    }
}
```

Helper functions can also be passed as "data".

```php
namespace APP\plugins\themes\eidos\classes\components;

class Layout extends \APP\view\components\Layout
{
    public TemplateManager $templateMgr;

    protected function addGlobalData(): void
    {
        parent::addGlobalData();
        view()->share('homepageBlocks', [$this, 'getHomepageBlocks']);
    }

    public function getHomepageBlocks(?array $blockIds = null): array
    {
      return $this->templateMgr->homepageBlocks->load($blockIds);
    }
}
```

And used in the template to load data on demand.

```php
$enabledBlockIds = $activeTheme->getOption('homepageBlocks');
@foreach($homepageBlocks($enabledBlockIds))
  //
@endforeach
```

On the whole, this is working well.

## Challenges

I have encountered a few problems or unexpected behaviours with this approach.

### Data Scope

In a component, data is scoped to the component itself. That means for the `Layout` component, we can not use the typical Blade component data structure. Blade generally assumes component data would look like this.

```php
class Layout
{
    public string $issn;

    public function __construct(public Context $context)
    {
        $this->issn = $context->getData('printIssn');
    }
}
```

```php
// indexJournal.blade
<x-layout :context="$context" />
```

```php
// layout.blade
<div>
    {{ $context->getLocalizedName() }}:
    {{ $issn }}
</div>
```

In the above example, we can use `$issn` in the layout template, but not in the layout slot. For example, the following, which is the more common use case, will not work.

```php
// indexJournal.blade
<x-layout :context="$context">
    {{ $issn }}
</x-layout>
```

To get around this, all data must be registered globally.

```php
class Layout
{
    public function __construct(public Context $context)
    {
        view()->share('issn', $context->getData('printIssn'));
    }
}
```

This is not a major problem for the `Layout` component, which in almost all cases is meant to be global. However, there are some additional problems that arise from this.

### Can't access layout data if not in a component

Global data is only available to component templates. Let's take the example above.

```php
class Layout
{
    public function __construct(public Context $context)
    {
        view()->share('issn', $context->getData('printIssn'));
    }
}
```

This data should be globally available, but it only works when called in a template in the `/components` directory. Consider the following setup.

```php
// /templates/frontend/indexJournal.blade
<x-layout :context="$context">
    <div>{{ $issn }}</div>
    <x-example-component />
<x-layout>
```

```php
// /templates/components/example-component.blade
<div>{{ $issn }}</div>
```

In this case, the output will be:

```php
<div></div>
<div>2049-3630</div>
```

The `$issn` data is not available to `frontend/indexJournal.blade`. It is only available to `components/example-component.blade`.

I am not sure why this is the case. It may be that Touhidur or Jarda can find a workaround for this. Right now, it requires some counter-intuitive setup if we want the `Layout` component to be our "home" for theme helper functions.

For example, I sometimes need to use a component, even if I'd rather just write the code into the page template. I had to do the following in the index template, which forces me to split the code into a separate file.

```php
// templates/frontend/indexJournal.blade
<x-layout>
    <x-homepage-blocks />
</x-layout>
```

### Plugin Access?

If `Layout` becomes the new `PKPTemplateManager`, how will plugins interact with it? Should they? I haven't explore this at all, but I think that we should adopt a pattern that permits plugin interaction.

Because `Layout` is called late in the request cycle, it may not be easy to do this.

### Class based components for common templates

In general, I have avoided class-based components in favour of keeping all template logic in the `.blade` templates. This means that theme developers only need to interact with one file.

However, I have begun to wonder if it would be a good idea for PKP to provide class components for some data objects.

For example, can an `Article` class provide helper functions to simplify access to galleys, full-text HTML, and other complex metadata?

```php
// This could live in OJS or pkp-lib instead
namespace APP\plugins\themes\eidos\classes\components;

use APP\publication\Publication;
use APP\submission\Submission;
use Closure;
use Illuminate\View\Component;
use Illuminate\Contracts\View\View;
use Illuminate\Support\Facades\View as ViewFacade;
use PKP\context\Context;
use PKP\galley\Galley;
use PKP\template\ViewHelper;

class Article extends Component
{
    public string $coverImageUrl;
    public string $coverImageAltText;
    public string $fullText;

    public function __construct(
        public Submission $submission,
        public Publication $publication,
    ) {
        /**
         * Usage:
         *
         * <img src="{{ $coverImageUrl }}" alt="{{ $coverImageAltText }}">
         */
        $this->coverImageUrl = $publication->getLocalizedCoverImageUrl($submission->getData('contextId'));
        $this->coverImageAltText = $this->publication->getLocalizedData('coverImage')
            ? $this->publication->getLocalizedData('coverImage')['altText']
            : '';
    }

    /**
     * Helper function to get the URL to the article
     *
     * Usage:
     *
     * <a href="{{ $url() }}">
     */
    public function url(): string
    {
        $props = [
            'page' => 'article',
            'op' => 'view',
            'path' => $this->submission->getBestId(),
        ];
        return ViewHelper::url($props);
    }

    /**
     * Helper function to get the full-text of the article
     *
     * Usage:
     *
     * <section>{!! ViewHelper::sanitizeHtml($fullText(true)) !!}</section>
     */
    public function fullText(bool $includeToc = false): string
    {
        return // full text compiled with figures, reference links, etc.
    }
}
```

This would be a great way to provide helper functions on-demand and reduce the amount of globally registered functions and template data. However, I don't think that this will work with slots. For example, I don't think the following will work:

```php
<x-article :submission="$submission" :publication="$publication">
    {{ $fullText() }}
</x-article>
```

Would it be possible to tell a Blade component which template it should render? I have not been able to test this yet. I imagine some code like this:

```php
namespace APP\plugins\themes\eidos\classes\components;

class Article extends Component
{
    public function __construct(
        public Submission $submission,
        public Publication $publication,
        /**
         * Pass the template file
         */
        public string $template = '',
    ) {
        //
    }

    public function render(): View|Closure|string
    {
        return view(
            ViewFacade::resolvePluginComponentViewPath(
                $this,
                $this->template,
            )
        );
    }
}
```

Then a theme could do the following:

```php
<x-article
    :submission="$submission"
    :publication="$publication"
    :template="components.my-example-template"
/>
```

```php
// components/my-example-template.blade
<article>
    {{ $fullText() }}
</article>
```

Whether or not this is worth it is an open question. Personally, I have found the lack of type-hinting in `.blade` files to be difficult to work with. I think the main benefit of a class component is:

- Proper type hinting
- Themers won't have to know OJS internals like primary/supplementary galley types, author ORCID/CRediT logic, etc.

Right now we put these helper functions into the `DataObject`, but I think there is some additional support for templating that doesn't belong there.

## Discussion Points

1. What is the best way to isolate theme-specific code from `PKPTemplateManager`? Is this a good idea or not worth trying? If it is a good idea, is a `Layout` component the best approach for this data? Or should we instead consider doing more with `ViewHelper` or the Blade service provider? Or is there another approach that would solve these problems better?

2. If we use the `Layout` component, is there a solution that would solve the problem of using global data in a template located at `templates/frontend/pages`?

3. Should we use class components to provide helper functions and variables for specific objects, like submissions, issues, galleys, users, etc?
