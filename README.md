# This repository

This repository contains the source code of my personal webpage hosted at [vide.bar](
https://vide.bar). It is a static website generated using [Zola](https://www.getzola.org/)
and a theme based on [Terminimal](https://github.com/pawroman/zola-theme-terminimal),
which itself is a forked of the [Hugo](https://gohugo.io/) theme [Terminal](
https://github.com/panr/hugo-theme-terminal).

# Theme modifications

I modified the original theme because I want to have a set of features on my website:

* I want to use the [Nord Theme](https://www.nordtheme.com/) for the colour palette.

* I want to have a landing page with some information about me, which is different
  from the blog page.

* I want to use a different font. I ended up going for [Iosevka](https://typeof.net/Iosevka/).

* Some minor changes like changing the symbol used for lists, adding a link to the
blog archive at the end of blog pagination, and changing the style of code blocks.

# Deployment

The webpage is deployed using Github Pages. To build it I used Github Actions and the
[Zola Deploy Action](https://github.com/shalzz/zola-deploy-action).

# License

The content of this project itself is licensed under the
[Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](
LICENSE-contents), and the underlying source code used
to format and display that content is licensed under the
[GNU General Public License v3.0 license](LICENSE).

The license of the original Terminimal Theme is included in [LICENSE-terminimal](
LICENSE-terminimal).

The license of the Nord Theme is included in [LICENSE-nord](LICENSE-terminimal).

The license of the Iosevka font is included in [LICENSE-iosevka](LICENSE-terminimal).
