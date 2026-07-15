# Update Release Changelog

Update the structured release changelog in `src/main/site/_data/releases.yml` with a new version entry derived from the GitHub compare view between two tags.

## Inputs

- **release-version**: $ARGUMENTS

If no arguments are provided, ask the user for the release version and the previous version before proceeding.
If only one argument is provided, treat it as the release version and ask for the previous version.
If two arguments are provided (space-separated), treat the first as the release version and the second as the previous version.

## Steps

### 1. Resolve inputs

Parse `$ARGUMENTS` to extract `<release-version>` and `<previous-version>`. If either is missing, ask the user.

### 2. Fetch the compare diff

Use the GitHub CLI to list commits between the two version tags:

```
gh api repos/citrusframework/citrus/compare/v<previous-version>...v<release-version> --jq '.commits[] | {sha: .sha, message: .commit.message, author: .author.login}'
```

Also fetch closed issues/PRs referenced in this range to enrich the changelog:

```
gh api repos/citrusframework/citrus/compare/v<previous-version>...v<release-version> --jq '.commits[].commit.message'
```

Extract PR numbers and issue references from commit messages (look for patterns like `#123`, `(#123)`, `fixes #123`, `closes #123`).

For each referenced PR, fetch its details to get the title, labels, and author:

```
gh api repos/citrusframework/citrus/pulls/<PR_NUMBER> --jq '{title: .title, labels: [.labels[].name], author: .user.login, body: .body}'
```

### 3. Classify changes

Categorize each change based on PR labels and commit message prefixes:

- `fix` — PRs/commits with label `bug` or prefix `fix:`, `fix(`, or `bugfix:`
- `add` — PRs/commits with label `enhancement` or `feature`, or prefix `feat:`, `feat(`
- `update` — PRs/commits with label `dependencies`, `maintenance`, or prefix `chore:`, `deps:`, `build:`, `refactor:`
- Default to `update` if no classification can be determined.

Use the **PR title** as the change title (prefer it over commit message since it is typically cleaner). If a commit does not reference a PR, use the first line of the commit message.

Deduplicate: if multiple commits reference the same PR, include only one entry. If a squash-merge commit message matches a PR title, use the PR as the source.

Skip commits that are:
- Merge commits (messages starting with `Merge`)  
- Version bumps with no meaningful change (e.g., `[maven-release-plugin]`)
- CI-only changes (unless they are notable)

### 4. Determine the release date

Use today's date in `YYYY-MM-DD` format.

### 5. Build the YAML entry

Construct a new release entry matching this exact structure (see existing entries in the file for reference):

```yaml
- version: <release-version>
  date: <YYYY-MM-DD>
  tag: v<release-version>
  issues: https://github.com/citrusframework/citrus/issues/
  stories: https://github.com/citrusframework/citrus/issues/
  changes:
    - title: <change title>
      id: <issue or PR number>  # omit if no reference
      type: <add|fix|update>
      author: <github-username>
```

Sort changes by type: `add` first, then `fix`, then `update`.

### 6. Present for review

Show the complete YAML entry to the user and ask for confirmation before writing. Mention:
- How many changes were found in each category (add/fix/update)
- Any commits that were skipped and why
- Any changes where classification was uncertain

### 7. Write the entry

After user confirmation, prepend the new entry at the **top** of `src/main/site/_data/releases.yml` (before the first existing `- version:` line).

### 8. Update pom.xml version

Update `<citrus.version>` in `pom.xml` to `<release-version>`.
