# Citrus website

This directory contains the Jekyll sources for the Citrus website, [citrusframework.org](https://citrusframework.org/).

## Contributing

For information about contributing, see the [Contributing page](https://citrusframework.org/docs/contributing/).

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

It's just a Jekyll site, after all! :wink:
