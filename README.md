# Citrus website

This directory contains the Jekyll sources for the Citrus website, [citrusframework.org](https://citrusframework.org/).

## Contributing

For information about contributing, see the [Contributing page](https://github.com/citrusframework/citrus/blob/master/CONTRIBUTING.md).

## Running locally

You need Docker on your local machine for building this website with Jekyll. The Maven build will automatically load the Jekyll Docker image and perform website
 generation for you. You can preview your contributions before opening a pull request by running from within the directory:

```
mvn clean resources:resources package
```

Now you are able to review the website content in `target/site/_site` folder. You can also start a local Jekyll container hosting the site with:

```
mvn docker:start
```

You can now visit the Citrus site locally by pointing your browser to `http://localhost:4000`.

## Write new posts

The blog posts, release notes, samples and news are located as markdown files in `src/main/site/_posts`. You can add new posts here. You need to chose one of the following categories:

* *blog*
* *samples*
* *release* (@Deprecated not maintained since v2.7.2)

Depending on what category you choose the post is rendered to different sections in the website:

* [/news](https://citrusframework.github.io/news/)
* [/samples](https://citrusframework.github.io/samples/)
* [/releases](https://citrusframework.github.io/news/releases/) (@Deprecated not maintained since v2.7.2)

The [maven-resources-plugin](https://maven.apache.org/plugins/maven-resources-plugin) is used to copy posts when 
releasing the website. When copying it performs filtering so that variables, which can come from the maven project 
properties or from filter resources, can be included in your posts. These variables should be specified using the 
\${...} delimiters. For example to include the current _citrus.version_ you would specify \${citrus.version} in the post.
If you want to escape filtering for a variable you have to use the prefix '\\' prefix on front of the \${..} delimiter - 
for example \\${citrus.version}.  

## Release to github pages

This site is released as github pages using the `citrusframework` organisation [citrusframework.github.io](https://citrusframework.github.io). You can perform the release by calling

```
mvn clean resources:resources install -Prelease-github
```

This will checkout the github pages repository, copy all changes, commit all changes and push the changes to the repository.

In case you want to review the changes made before pushing to the repository use `package` instead of `install`:

```
mvn clean resources:resources package -Prelease-github
```

This will checkout the github pages and perform all changes without committing and pushing the changes. So you can navigate to the checkout folder `target/checkout` and use your favorite
git diff tool in order to see what has been changed.

Have fun! It's just a Jekyll site, after all! :wink:
