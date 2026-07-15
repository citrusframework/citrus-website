# Release to GitHub Pages

Build the Citrus website and publish it to GitHub Pages at [citrusframework.org](https://citrusframework.org).

This command runs the full release pipeline: Maven resource filtering, Jekyll site generation via Docker, and a push to the `citrusframework/citrusframework.github.io` repository.

## Prerequisites

- Docker must be running (Jekyll builds inside a container)
- You must have push access to `citrusframework/citrusframework.github.io`

## Steps

### 1. Pre-flight checks

Verify Docker is running:

```
docker info > /dev/null 2>&1
```

If Docker is not running, inform the user and stop.

### 2. Confirm with user

**This action pushes to a public repository.** Before proceeding, confirm with the user:

- Show the current `citrus.version` from `pom.xml`
- Ask whether they want a preview first or to publish directly

### 3. Preview (optional)

If the user wants to preview first:

1. Build the site without pushing:

```
mvn clean resources:resources package -Prelease-github
```

2. Run the local preview server:

Start this in a new process because the Maven build waits for user termination.

```
mvn docker:run
```

3. Open the browser to http://localhost:4000 for the user to inspect the site.

4. After review, stop the preview server by terminating the `docker:run` process, then remove the leftover Jekyll container to avoid name conflicts with the release build:

```
docker rm -f jekyll
```

5. Ask the user whether to proceed with publishing or abort.

### 4. Build and publish

Run the release command:

```
mvn clean resources:resources install -Prelease-github
```

This will:
1. Copy and filter site sources into `target/site/`
2. Build the Jekyll site via Docker into `target/site/_site/`
3. Copy schema files into the generated site
4. Commit and push the result to `citrusframework/citrusframework.github.io` (master branch)

### 5. Report result

If the build succeeds, confirm that the site was published and will be live at https://citrusframework.org shortly.

If the build fails, show the relevant error output and suggest next steps.
