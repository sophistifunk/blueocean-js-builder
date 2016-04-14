# Jenkins JS Builder

> __[JIRA](https://issues.jenkins-ci.org/browse/JENKINS/component/21132)__

__Table of Contents__:
<p>
<ul>
    <a href="#overview">Overview</a><br/>
    <a href="#features">Features</a><br/>
    <a href="#install">Install</a><br/>
    <a href="#general-usage">General Usage</a><br/>
    <a href="#predefined-gulp-tasks">Predefined Gulp Tasks</a><br/>
    <a href="#redefining-one-of-the-predefined-gulp-tasks">Redefining one of the predefined Gulp tasks</a><br/>
    <a href="#bundling">Bundling</a><br/>
    <a href="#setting-src-and-test-spec-paths">Setting 'src' and 'test' (spec) paths</a><br/>
    <a href="#command-line-options">Command line options</a><br/>
    <a href="#maven-integration">Maven Integration</a><br/>
    <a href="https://github.com/jenkinsci/js-samples">Sample Plugins (Jenkins - HPI)</a><br/>    
    <a href="https://issues.jenkins-ci.org/browse/JENKINS/component/21132">JIRA</a><br/>    
</ul>    
</p>

<hr/>

# Overview
[NPM] utility for building [CommonJS] module [bundle]s (and optionally making them __[js-modules]__ compatible).

> See __[js-modules]__.

The following diagram illustrates the basic flow (and components used) in the process of building a [CommonJS] module [bundle]. 
It uses a number of popular JavaScript and maven tools ([CommonJS]/[node.js], [Browserify], [Gulp], [frontend-maven-plugin] and more).

<p align="center">
    <a href="https://github.com/jenkinsci/js-modules" target="_blank">
        <img src="res/build_workflow.png" alt="Jenkins Module Bundle Build Workflow">
    </a>
</p>
  
The responsibilities of the components in the above diagram can be summarized as follows:

* __[CommonJS]__: JavaScript module system (i.e. the expected format of JavaScript modules). This module system works with the nice/clean synchronous `require` syntax synonymous with [node.js] (for module loading) e.g. `var mathUtil = require('../util/mathUtil');`. This allows us to tap into the huge [NPM] JavaScript ecosystem.
* __[Browserify]__: A build time utility ([NPM] package - executed as a [Gulp] "task") for "bundling" a graph of [CommonJS] style modules together, producing a single JavaScript file ([bundle]) that can be loaded (from a single request) in a browser. [Browserify] ensures that the `require` calls (see above) resolve properly to the correct module within the [bundle].
* __[Gulp]__: A JavaScript build system ([NPM] package), analogous to what Maven is for Java i.e. executes "tasks" that eventually produce build artifacts. In this case, a JavaScript __[bundle]__ is produced via [Gulp]s execution of a [Browserify] "task".
* __[frontend-maven-plugin]__: A Maven plugin that allows us to hook a [Gulp] "build" into a maven build e.g. for a Jenkins plugin. See <a href="#maven-integration">Maven Integration</a> below.

# Features
`js-builder` does a number of things:

1. Runs [Jasmine] tests/specs and produce a JUnit report that can be picked up by a top level Maven build.
1. Uses [Browserify] to produce a [CommonJS] module __[bundle]__ file from a "main" [CommonJS] module (see the `bundle` task below). The [bundle] file is typically placed somewhere on the filesystem that allows a higher level Maven build to pick it up and include it in e.g. a Jenkins plugin HPI file (so it can be loaded by the browser at runtime). 
1. Pre-process [Handlebars] files (`.hbs`) and include them in the __[bundle]__ file (see 2 above).
1. __Optionally__ pre-process a [LESS] fileset to a `.css` file that can be picked up by the top level Maven build and included in the e.g. a Jenkins plugin HPI file. See the `bundle` task below.
1. __Optionally__ perform module transformations (using a [Browserify Transform](https://github.com/substack/browserify-handbook#transforms)) that "link" in [Framework lib]s (`import` - see [js-modules]), making the [bundle] a lot lighter by allowing it to use a shared instance of the [Framework lib] Vs it being included in the [bundle]. This can easily reduce the size of a [bundle] from e.g. 1Mb to 50Kb or less, as [Framework lib]s are often the most weighty components. See the `bundle` task below.
1. __Optionally__ `export` (see [js-modules]) the [bundle]s "main" [CommonJS] module (see 2 above) so as to allow other [bundle]s `import` it i.e. effectively making the bundle a [Framework lib] (see 5 above). See the `bundle` task below.

# Install

```
npm install --save-dev @jenkins-cd/js-builder
```

> This assumes you have [node.js] (minimum v4.0.0) installed on your local development environment.

> Note this is only required if you intend developing [js-modules] compatible module bundles. Plugins using this should automatically handle all build aspects via maven (see later) i.e. __simple building of a plugin should require no machine level setup__.

# General Usage

Add a `gulpfile.js` (see [Gulp]) in the same folder as the `package.json`. Then use `js-builder` as follows:

```javascript
var builder = require('@jenkins-cd/js-builder');

builder.bundle('./src/main/js/myappbundle.js');

```

After running the the `gulp` command from the command line, you will see an output something like the following.
  
```
[17:16:33] Javascript bundle "myappbundle" will be available in Jenkins as adjunct "org.jenkins.ui.jsmodules.myappbundle".
```

Or if run from a maven project where the `artifactId` is (e.g.) `jenkins-xyz-plugin`.

```
[17:16:33] Javascript bundle "myappbundle" will be available in Jenkins as adjunct "org.jenkins.ui.jsmodules.jenkins_xyz_plugin.myappbundle".
```

From this, you can deduce that the easiest way of using this JavaScript bundle in Jenkins is via the `<st:adjunct>` jelly tag.

```xml
<st:adjunct includes="org.jenkins.ui.jsmodules.jenkins_xyz_plugin.myappbundle"/>
```

The best place to learn how to use this utility as part of building Jenkins plugins is to see the 
[Sample Plugins](https://github.com/jenkinsci/js-samples) repository.

# Predefined Gulp Tasks

The following sections describe the available predefined [Gulp] tasks.

> __Note__: If no task is specified (i.e. you just type `gulp` on its own), then the `bundle` and `test` tasks are auto-installed (i.e. auto-run) as the default tasks.

## 'bundle' Task 
Run the 'bundle' task. See detail on this in the <a href="#bundling">dedicated section titled "Bundling"</a> (below). 

```
gulp bundle
```
 
## 'test' Task

Run tests. The default location for tests is the `spec` folder. The file names need to match the
pattern "*-spec.js". The default location can be overridden by calling `builder.tests(<new-path>)`.

```
gulp test
```

> See [jenkins-js-test] for more on testing.
> See <a href="#command-line-options">command line options</a> for `--skipTest` option.
> See <a href="#command-line-options">command line options</a> for `--test` option (for running a single test spec).

## 'bundle:watch' Task

Watch module source files (`index.js`, `./lib/**/*.js` and `./lib/**/*.hbs`) for change, auto-running the
`bundle` task whenever changes are detected.

Note that this task will not be run by default, so you need to specify it explicitly on the gulp command in
order to run it e.g.

```
gulp bundle:watch
```

## 'test:watch' Task

Watch module source files changes (including test code) and rerun the tests e.g.

```
gulp test:watch
```

## 'lint' Task

Run linting - ESLint or JSHint. ESlint is the default if no `.eslintrc` or `.jshintrc` file is found 
(using [eslint-config-jenkins](https://www.npmjs.com/package/@jenkins-cd/eslint-config-jenkins)) in the working
directory (`.eslintrc` is also searched for in parent directories).

```
gulp lint
```

> See <a href="#command-line-options">command line options</a> for `--skipLint`, `--continueOnLint` and `--fixLint` options.

# Redefining one of the predefined Gulp tasks

There are times when you need to break out and redefine one of the predefined gulp tasks (see previous section).
To redefine a task, you simply call `defineTask` again e.g. to redefine the `test` task to use mocha:

```javascript
builder.defineTask('test', function() {
    var mocha = require('gulp-mocha');
    var babel = require('babel-core/register');

    builder.gulp.src('src/test/js/*-spec.js')
        .pipe(mocha({
            compilers: {js: babel}
        })).on('error', function(e) {
            if (builder.isRetest()) {
                // ignore test failures if we are running test:watch.
                return;
            }
            throw e;
        });
});
```

# Bundling Options

The following sections outline some options that can be specified on a `bundle` instance.

## Generating a bundle to a specific directory

By default, the bundle command will output the bundle to the `target/classes/org/jenkins/ui/jsmodules`, making
the bundle loadable in Jenkins as an adjunct. See the <a href="#general-usage">General Usage</a> section earlier
in this document.

Outputting the generated bundle to somewhere else is just a matter of specifying it on the `bundle` instance
via the `inDir` function e.g.

```javascript
bundleSpec.inDir('<path-to-dir>');
```

## Minify bundle JavaScript

This can be done by calling `minify` on `js-builder`:

```javascript
bundleSpec.minify();
```

Or, by passing `--minify` on the command line. This will result in the minification of all generated bundles.
 
```sh
$ gulp --minify
```

## onPreBundle listeners

There are times when you will need access to the underlying [Browserify] `bundler` just before the
bundling process is executed (e.g. for adding transforms etc).

To do this, you call the `onPreBundle` function. This function takes a `listener` function as an argument.
This `listener` function, when called, receives the `bundle` as `this` and the `bundler` as the only argument to
the supplied `listener`.

```javascript
var builder = require('@jenkins-cd/js-builder');

builder.onPreBundle(function(bundler) {
    var bundle = this;
    
    console.log('Adding the funky transform to bundler for bundle: ' + bundle.as);
    bundler.transform(myFunkyTransform);
});
```

# Setting 'src' and 'test' (spec) paths
The default paths depend on whether or not running in a maven project.

For a maven project, the default source and test/spec paths are:

* __src__: `./src/main/js` and `./src/main/less` (used primarily by the `bundle:watch` task, watching these folders for source changes)
* __test__: `./src/test/js` (used by the `test` task)

Otherwise, they are:

* __src__: `./js` and `./less` (used primarily by the `bundle:watch` task, watching these folders for source changes)
* __test__: `./spec` (used by the `test` task)



Changing these defaults is done through the `builder` instance e.g.:

```javascript
var builder = require('@jenkins-cd/js-builder');

builder.src('src/main/js');
builder.tests('src/test/js');
```

You can also specify an array of `src` folders e.g.

```javascript
builder.src(['src/main/js', 'src/main/less']);
```

# Command line options

A number of `js-builder` options can be specified on the command line. If you are looking for


## `--h` (or `--help`)

Get a link to this documentation.
 
```sh
$ gulp --h
```

## `--minify`

Passing `--minify` on the command line will result in the minification of all generated bundles.
 
```sh
$ gulp --minify
```

## `--test`

Run a single test.
 
```sh
$ gulp --test configeditor
```

The above example would run test specs matching the `**/configeditor*-spec.js` pattern (in the test source directory).

## Skip options: `--skipTest`, `--skipLint`, `--skipBundle`

Skip one or more of the tasks/phases e.g.
 
```sh
$ gulp --skipTest --skipLint
```

## Lint options: `--skipLint`, `--continueOnLint`, `--fixLint`

Many of the more irritating formatting rule errors/warnings can be fixed automatically by running
with the `--fixLint` option, making them a little less irritating e.g.
 
```sh
$ gulp --fixLint
```

Or if you are just running the `lint` task on it's own (explicitly):
 
```sh
$ gulp lint --fixLint
```

Alternatively, if you wish to run `lint` and see all of the lint errors, but not fail the build:
 
```sh
$ gulp --continueOnLint
```

And to skip linting completely:
 
```sh
$ gulp --skipLint
```

# Maven Integration
Hooking a [Gulp] based build into a Maven build involves adding a few Maven `<profile>`s to the
Maven project's `pom.xml`. For Jenkins plugins, the easiest way to get this integration is to simply
have the plugin `pom.xml` depend on the Jenkins [plugin-pom]. For other project types, you'll need
to copy those profiles locally (see [plugin-pom]).

These integrations hook the [Gulp] build into the maven build lifecycles. A few `mvn` build
switches are supported, as described in the following sections.

## `-DcleanNode`

Cleans out the local node and NPM artifacts and resource (including the `node_modules` folder).

```
$ mvn clean -DcleanNode
```

## `-DskipTests`

This switch is a standard `mvn` switch and is honoured by the profiles defined in the [plugin-pom].

```
$ mvn clean -DskipTests
```

`-DskipTests` also skips linting. See `-DskipLint`

## `-DskipLint`

Skip linting.

```
$ mvn clean -DskipLint
```

[bundle]: https://github.com/jenkinsci/js-modules/blob/master/FAQs.md#what-is-the-difference-between-a-module-and-a-bundle
[js-modules]: https://github.com/jenkinsci/js-modules
[js-builder]: https://github.com/jenkinsci/js-builder
[jenkins-js-test]: https://github.com/jenkinsci/js-test
[NPM]: https://www.npmjs.com/
[CommonJS]: http://www.commonjs.org/
[node.js]: https://nodejs.org/en/
[Browserify]: http://browserify.org/
[Gulp]: http://gulpjs.com/
[frontend-maven-plugin]: https://github.com/eirslett/frontend-maven-plugin
[intra-bundle]: https://github.com/jenkinsci/js-modules/blob/master/FAQs.md#what-does-module-loading-mean
[inter-bundle]: https://github.com/jenkinsci/js-modules/blob/master/FAQs.md#what-does-module-loading-mean
[io.js]: https://iojs.org
[Framework lib]: https://github.com/jenkinsci/js-libs
[LESS]: http://lesscss.org/
[Handlebars]: http://handlebarsjs.com/
[Jasmine]: http://jasmine.github.io/
[Moment.js]: http://momentjs.com/
[plugin-pom]: https://github.com/jenkinsci/plugin-pom
