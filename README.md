# ng-immutable-example

This project demonstrates how to convert to an [Immutable Web App](https://immutablwebapps.org) from an Angular app that was generated with [Angular CLI](https://github.com/angular/angular-cli) version 7.0.3.

## Getting Started

This project was created by running:

```bash
> npx @angular/cli new ng-immutable-example
? Would you like to add Angular routing? (y/N) N
? Which stylesheet format would you like to use? (Use arrow keys)
❯ CSS 
  SCSS   [ http://sass-lang.com   ] 
  SASS   [ http://sass-lang.com   ] 
  LESS   [ http://lesscss.org     ] 
  Stylus [ http://stylus-lang.com ] 

> cd ng-immutable-example
```

## Immutable Conversion

Converting this this new application to build an Immutable Web App requires the following steps:

1. Referencing environment variables defined on `window`
2. Rendering an `index.html` template for deployments
3. Running locally

## Referencing environment variables defined on `window`

Angular has documentation for [using environment-specific variable in your app](https://angular.io/guide/build#using-environment-specific-variables-in-your-app). This pattern compiles environment-specific values into the javascript bundles at build-time. To make the assets immutable, the setting of values must be shifted from build-time to run-time. Fortunately, the Angular environments can still be leveraged by assigning them to an environment object defined on `window` and then the values can be defined in the index.html that is unique to each deploy.

`environment.prod.ts`:
```diff
- export const environment = {
-   production: false
- };
+ export const environment = (window as any).env;
```

A benefit of this approach is that you can continue to use multiple Angular environments with hard-coded values for local development.

## Rendering an `index.html` template for deployments

In theory, the role of `index.html` in an Immutable Web App is to be a deployment manifest that only contains configuration that is unique to the environment where it is deployed.

In practice, there is often a need to include markup or scripts in the `index.html` that are more than just configuration, or that doesn't vary by environment, or that does change between versions of the app. This issue can be resolved by creating an immutable `index.html` template, that is published with the other immutable assets.

### Converting index.html to a template

This example uses [EJS](https://ejs.co/) as the templating language to render `index.html`, but any templating language can be used.

1) Rename `src/index.html` to `src/index.ejs`. 
2) Identify the parts of the `index.html` that vary by environment:

    - [`deployUrl`](https://github.com/angular/angular-cli/blob/v6.0.0-rc.8/packages/%40angular/cli/lib/config/schema.json#L622-L625): URL where the versioned immutable assets will be deployed.
    - [`baseHref`](https://github.com/angular/angular-cli/blob/v6.0.0-rc.8/packages/%40angular/cli/lib/config/schema.json#L618-L621): Base url for the application being built.
    - `window.env`: Environment-specific javascript configuration.

3) Set `deployUrl` and `baseHref` to an EJS template string in `angular.json` and update the name of `index`:

```diff
{
  "projects": {
    "ng-immutable-example": {
      ...
      "architect": {
        "build": {
          ...
          "configurations": {
            "production": {
              ...
+             "deployUrl": "<%=deployUrl%>",
+             "baseHref": "<%=baseHref%>",
-             "index": "src/index.html",
+             "index": "src/index.ejs",
```

4) Update the `href` of `favicon.ico` in `src/index.ejs`:

```diff
- <link rel="icon" type="image/x-icon" href="favicon.ico">
+ <link rel="icon" type="image/x-icon" href="<%=deployUrl%>favicon.ico">
```

5) Add a new script tag to render the javascript environment-specific variables in `src/index.ejs`:

```diff
<head>
    ...
+   <script>
+       env = <%-JSON.stringify(env)%>;
+   </script>
</head>
```

### Example

#### `src/index.ejs`:

__The source template that is converted into an immutable template during the build.__

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>NgImmutableExample</title>
  <base href="/">

  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="<%=deployUrl%>favicon.ico">
  <script>
    env = <%-JSON.stringify(env)%>;
  </script>
</head>
<body>
  <app-root></app-root>
</body>
</html>
```

#### `dist/ng-immutable-example/index.ejs`

__The immutable template that is published along with the other versioned, immutable assets.__ It is combined with `production-config.json` to render the `index.html` that is a deployment manifest.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>NgImmutableExample</title>
  <base href="<%=baseHref%>">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="<%=deployUrl%>favicon.ico">
  <script>
    env = <%-JSON.stringify(env)%>;
  </script>
<link rel="stylesheet" href="<%=deployUrl%>styles.3bb2a9d4949b7dc120a9.css"></head>
<body>
  <app-root></app-root>
<script type="text/javascript" src="<%=deployUrl%>runtime.3afa4a1fd69496214142.js"></script><script type="text/javascript" src="<%=deployUrl%>polyfills.c6871e56cb80756a5498.js"></script><script type="text/javascript" src="<%=deployUrl%>main.61370ef751f4da2a56bc.js"></script></body>
</html>
```

#### `config.json`:

__The environments-specific values that vary.__ This is combined with the published `index.ejs` to be rendered into `index.html`. It is not an immutable asset and should be managed as deployment-specific configuration.

```json
{
    "baseHref": "/",
    "deployUrl": "https://assets.ng-immutable-example.com/1.0.0/",
    "env": {
        "production": true,
        "api": "https://api.ng-immutable-example.com/"
    }
}
```

#### Rendered `index.html`

__The environment-specific deployment manifest.__ It is the product of rendering an immutable template against the `config.json`. Publishing this file to the web application environment is an atomic deployment.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>NgImmutableExample</title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="https://assets.ng-immutable-example.com/1.0.0/favicon.ico">
  <script>
    env = {production:true,api:"https://api.ng-immutable-example.com/"};
  </script>
<link rel="stylesheet" href="https://assets.ng-immutable-example.com/1.0.0/styles.3bb2a9d4949b7dc120a9.css"></head>
<body>
  <app-root></app-root>
<script type="text/javascript" src="https://assets.ng-immutable-example.com/1.0.0/runtime.3afa4a1fd69496214142.js"></script><script type="text/javascript" src="https://assets.ng-immutable-example.com/1.0.0/polyfills.c6871e56cb80756a5498.js"></script><script type="text/javascript" src="https://assets.ng-immutable-example.com/1.0.0/main.61370ef751f4da2a56bc.js"></script></body>
</html>
```

## Running locally

Running `ng serve` with the environment-specific defaults in `src/environments/environments.ts` will continue to run as designed. The only part that is missing, after the immutable conversion, is the `src/index.html` has been replaced with an EJS template. Until there is better support for Immutable Web Apps in Angular CLI, an `index.html` must be rendered from the EJS template prior to the build.

### Rendering `index.html` from `src/index.ejs`

1) __Run `npm i --save-dev @immutablewebapps/ejs-cli`__: This module renders `index.html` provided a template and JSON file. The module [`@immutablewebapps/ejs-cli`](https://www.npmjs.com/package/@immutablewebapps/ejs-cli) is just a lightweight wrapper around `ejs` that only serves this purpose. As mentioned earlier, any templating language may be used.

2) __Create `.immutablewebapps/config.json`__: This file will store the default configuration values for running locally. Often it is simply:

```json
{
    "deployUrl": "",
    "baseHref": "",
    "env": {}
}
```
3) Update the `npm start` script:

```diff
    "scripts": {
-       "start": "ng serve",
+       "start": "cat src/index.ejs | iwa-ejs --d .immutable/config.json > .immutable/index.html && ng serve",
    }
```

4) Update `angular.json` to use `.immutable/index.html`:

```diff
{
  "projects": {
    "ng-immutable-example": {
      "architect": {
        "build": {
          "options": {
-           "index": "src/index.html",
+           "index": ".immutable/index.html",
```

5) Ignore `.immutable/index.html`

`.gitignore`:
```diff
+ # Immutable Web Apps
+ .immutable/index.html
```

6) Run `npm start` and you are running locally!

## Future Work

This project will be updated as better patterns for building Immutable Web Apps are established and as Angular CLI changes.