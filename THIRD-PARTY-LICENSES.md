# Third-Party Licenses

Third-party assets are vendored into the repository as static files — there is no
runtime dependency on the projects below.

## Icons

Inlined as SVG at build time ([`layouts/partials/icon.html`](layouts/partials/icon.html)),
sourced and named by [`scripts/fetch-icons.sh`](scripts/fetch-icons.sh) under `assets/icons/`.

### Phosphor Icons — MIT

All UI and git/changelog icons (`assets/icons/ui/`, the `*-duotone` style).

- Author: Phosphor Icons — https://github.com/phosphor-icons/core
- License: MIT — https://github.com/phosphor-icons/core/blob/main/LICENSE

### Octicons — MIT

The "closed pull request" git icon (`assets/icons/ui/code-pull-request-closed.svg`),
which Phosphor has no equivalent for.

- Author: GitHub — https://github.com/primer/octicons
- License: MIT — https://github.com/primer/octicons/blob/main/LICENSE

### Simple Icons — CC0 1.0

Brand logos (`assets/icons/brands/`: github, bluesky, linkedin, docker, instagram,
markdown, youtube).

- Author: Simple Icons Collaborators — https://github.com/simple-icons/simple-icons
- License: CC0 1.0 — https://github.com/simple-icons/simple-icons/blob/develop/LICENSE.md

> Brand logos are trademarks of their respective owners; they are used here only to
> link to the corresponding profiles/projects.

## Fonts

Self-hosted from `static/fonts/` and declared in [`assets/css/main.css`](assets/css/main.css).

### Inter — SIL Open Font License 1.1

- Author: Rasmus Andersson — https://rsms.me/inter/
- License: SIL OFL 1.1 — https://github.com/rsms/inter/blob/master/LICENSE.txt

### Rubik — SIL Open Font License 1.1

- Author: Hubert & Fischer, et al. — https://fonts.google.com/specimen/Rubik
- License: SIL OFL 1.1 — https://github.com/googlefonts/rubik/blob/main/OFL.txt

## JavaScript

### Fuse.js — Apache License 2.0

Client-side fuzzy search for the ⌘K palette. Vendored as
`assets/js/fuse.basic.min.mjs` (v7.4.2, the `fuse.basic` ESM build) from the npm
package; loaded lazily by [`layouts/partials/palette.html`](layouts/partials/palette.html).

- Author: Kiro Risk — https://www.fusejs.io/
- License: Apache 2.0 — https://github.com/krisk/Fuse/blob/main/LICENSE
