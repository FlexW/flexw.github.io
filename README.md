# Development

Run dev server with `hugo server -D` in the `flexw` folder.

# Build and Deploy

Remove the `flexw/public` and `docs` folder first.
Build with `hugo build` in `flexw`, and then copy `flexw/public` into docs with `cp -r -f flexw/public/* docs`.
