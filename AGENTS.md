# Citrus Website - AI Agent Guidelines

This file provides guidance to AI agents such as Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the Jekyll-based website for the Citrus integration testing framework, published at [citrusframework.org](https://citrusframework.org). 
It is **not** the main Citrus framework codebase — it is a static site project that uses Maven to orchestrate Jekyll builds via Docker.

## Build & Preview Commands

Requires Docker to be running.

```bash
# Build the site (copies sources with Maven resource filtering, then runs Jekyll via Docker)
mvn clean resources:resources package

# Start a local preview server at http://localhost:4000
mvn docker:start

# Release to GitHub Pages (commits and pushes to citrusframework.github.io)
mvn clean resources:resources install -Prelease-github

# Dry-run release (review changes in target/checkout without pushing)
mvn clean resources:resources package -Prelease-github
```

There are no tests or linters — the build either succeeds or fails.

## How the Build Works

1. **`generate-resources`**: `maven-assembly-plugin` unpacks XSD/JSON schemas from Citrus framework dependency JARs into `target/schemas/`.
2. **`process-resources`**: `maven-resources-plugin` copies `src/main/site/` to `target/site/`, applying Maven property filtering (e.g., `${citrus.version}` is replaced with the current version from `pom.xml`).
3. **`prepare-package`**: `docker-maven-plugin` starts a Jekyll container that reads `target/site/` and generates `target/site/_site/`.
4. **`package`**: Schema files from step 1 are copied into `target/site/_site/schema/` subtrees. The Jekyll container is stopped.

The current Citrus version is set in `pom.xml` as `<citrus.version>`. Files that use `${citrus.version}` (e.g., `latest_version.txt`, posts, docs) get filtered during the resource copy phase. Use `\${...}` to escape and prevent filtering.

## Content Structure

All site content lives under `src/main/site/`:

- **`_posts/`** — Blog posts, samples, and release announcements (Markdown with YAML front matter). Categories: `blog`, `samples`. Posts use layouts `post` or `sample`.
- **`_docs/`** — Documentation pages (getting started, contributing, custom extensions). Navigation order defined in `_data/docs.yml`.
- **`_data/releases.yml`** — Structured release changelog. Each release has `version`, `date`, `tag`, and a list of `changes` with `title`, `type` (fix/update/feature), and `author`.
- **`_data/docs.yml`** — Sidebar navigation structure for docs.
- **`_layouts/`** — Jekyll templates (`default`, `post`, `sample`, `docs`, `news`, `page`, `samples-group`).
- **`_includes/`** — Reusable HTML partials (header, footer, navigation, content sections).
- **`_img/assets`** — Assets such as images and resources that can be referenced in blog posts.

## Adding a New Release

1. Add a new entry at the **top** of `src/main/site/_data/releases.yml` with version, date, tag, and changes.
2. Update `<citrus.version>` in `pom.xml` to the new version.
3. Optionally add a blog post in `_posts/` with category `release` for major releases.

Recent commits follow the pattern: `chore: Add ${citrus.version} release`.

## Adding a New Blog Post

The process to write a new blog post should follow:

1. Create a new Markdown file in `_posts/`, use the file format `YYYY-MM-DD-short-name.md`
2. Choose the right category. Available categories are `blog`, `release`, `samples`, `overview`. Depending on the category consider the following aspects:

- **`_blog/`** — Arbitrary blog post with different topics. Use `post` as a layout.
- **`_release/`** — Release information with focus on what has changed and notable aspects of the release. Use `post` as a layout.
- **`_samples/`** — Represents a quickstart guide that explains a certain Citrus feature with the help of a small sample project and sample code snippets. Use a special layout: `sample`. Sample blog posts should be using a group and a permalink.
- **`_overview/`** — Summarizes a set of samples blogs that relate to each other (e.g. all samples related to the Http transport). These posts use a special layout `samples-group` and can have an icon from `src/main/site/img/icons`. Sample overview blog posts should be using a group and a permalink.

## Archiving Posts

To hide a post from all listing pages without deleting it, add `archived: true` to the post's YAML front matter. The post file stays in the repo but is filtered out of every index, sidebar, and navigation element across the site.

## Key Details

- The `pom.xml` declares dependencies on Citrus framework modules — these are **not runtime deps** but are used solely to extract XSD/JSON schema files for the website's `/schema/` section.
- Jekyll plugins: `jekyll-feed`, `jekyll-redirect-from`, `jemoji`, `jekyll-sitemap`, `jekyll-mentions`.
- The site uses Kramdown for Markdown and Rouge for syntax highlighting.
- GitHub Pages target repo: `citrusframework/citrusframework.github.io` (master branch).
