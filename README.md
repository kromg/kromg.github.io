# Lost in IT | Kromg

This is a Hugo-based static website deployed to GitHub Pages.

## Theme

This site uses the [Ananke theme](https://github.com/theNewDynamic/gohugo-theme-ananke) as a git submodule located in `themes/ananke`.

## Development

### Initial Setup

When cloning this repository, make sure to initialize the submodules:

```bash
git clone --recursive https://github.com/kromg/kromg.github.io.git
# OR if already cloned:
git submodule update --init --recursive
```

### Building the Site

```bash
# Development server with live reload
hugo server -D

# Production build
hugo --minify
```

### Updating the Theme

To update the Ananke theme to the latest version:

```bash
cd themes/ananke
git pull origin main
cd ../..
git add themes/ananke
git commit -m "Update Ananke theme"
```

## Deployment

The site is automatically deployed to GitHub Pages via GitHub Actions whenever changes are pushed to the `main` branch. The workflow:

1. Checks out the code with recursive submodules
2. Updates submodules to ensure they're current
3. Sets up Hugo (extended version)
4. Builds the site with minification
5. Deploys to GitHub Pages

## Content Management

- Add new posts in the `content/posts/` directory
- Use `hugo new posts/my-new-post.md` to create new posts with the correct archetype
- Static files go in the `static/` directory