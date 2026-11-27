# sauravsc.github.io (Hugo source)

This `hugo` branch is the source of https://sauravsc.github.io/. The `master`
branch holds the generated site, which GitHub Pages serves as-is.

The theme is [meh-theme](https://github.com/sauravsc/meh-theme), a Git submodule
in `themes/meh-theme`. Hugo 0.158.0 or later is required.

## Build and publish

```sh
git clone --branch hugo --recurse-submodules https://github.com/sauravsc/sauravsc.github.io.git site-src
cd site-src
git worktree add ../site-master master

hugo server                                       # preview at http://localhost:1313/
hugo --panicOnWarning --destination ../site-master

cd ../site-master
git add -A && git commit -m "Rebuild site" && git push
```

Building into the `master` worktree overwrites the generated pages but does not
delete older files there (for example `css/bootstrap.css` and `img/` from an
earlier theme), so their URLs keep working.

## Content

- `content/_index.md`: the about, experience and projects sections of the home page.
- `content/post/`: posts, listed under "writing" on the home page.
- `content/page/about.md`: https://sauravsc.github.io/page/about/
- `layouts/_markup/render-codeblock.html`: plain (unhighlighted) code blocks.
