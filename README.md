# Mario Moreno - Personal Website & Blog

This is the repository for my personal website and technical blog, hosted at [mamcer.github.io](https://mamcer.github.io/). Sharing tips, tech stories, and lessons from my personal experience.

## Technology Stack

- **Static Site Generator:** [Hugo](https://gohugo.io/) (Extended version)
- **Theme:** [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
- **CI/CD & Deployment:** GitHub Actions deploying to the `gh-pages` branch

## Directory Structure

- `hugo.yaml` - Main configuration file containing site details, menus, and profile parameters.
- `content/posts/` - Blog posts written in Markdown.
- `content/about.md` - The "About me" page.
- `.github/workflows/hugo.yml` - Deployment workflow that automatically builds and publishes the site on push to the `main` or `master` branches.

## Local Development

### Prerequisites

You need to have [Hugo](https://gohugo.io/installation/) installed locally (the Extended edition is recommended).

### Running the Dev Server

Run the following command in the project root to start the local development server:

```bash
hugo server -D
```

Once started, navigate to `http://localhost:1313/` in your browser.