## Template Directory

I have not properly organized the template directory yet. This is the directory structure that I would like to use.

```php
/templates
  # Re-usable components, like dropdown.blade and
  # article-summary.blade
  /components (things like `dropdown.blade` or `article-summary.blade`)

  # Full-page layout components
  /layouts

  # Page templates
  /pages

  # Small bits of templates
  #
  # These may only be used once, but help split
  # large templates into parts. Examples might
  # a separate fragment for each article tab.
  #
  # If necessary, could probably only be used
  # with @include(), rather than <x-thing />
  #
  # (I'm not set on the name, could be `content`
  # or whatever)
  /template-parts

  # Specifically for svg icons
  #
  # Right now all svg icons are saved as `.blade`
  # and loaded with `includeIf()`.
  #
  # If possible, I would like to find a way to
  # save them as `.svg`. I may also try to create
  # a re-usable component that allows me to
  # manipulate attributes on the outer SVG.
  /icons
```

Right now, only files in the `/components` directory can be loaded using the component syntax:

```php
<x-my-example-component />
```

I can load other templates using the `@include()` syntax.

```php
@include('template-parts/my-example')
```

However, if the [layout](./template-setup-and-helper-functions.md) approach is used, the layout templates can not be in their own directory. The same is true if I want to improve the handling of svg icons.

Basically, everything that can be used as a component has to be in the `components` directory.

Is it possible to make Blade's component system available in other directories?
