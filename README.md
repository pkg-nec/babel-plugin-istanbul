# @pkg-nec/babel-plugin-istanbul

[![npm version](https://img.shields.io/npm/v/@pkg-nec/babel-plugin-istanbul?logo=npm)](https://www.npmjs.com/package/@pkg-nec/babel-plugin-istanbul)
[![CI](https://github.com/pkg-nec/babel-plugin-istanbul/actions/workflows/ci.yml/badge.svg)](https://github.com/pkg-nec/babel-plugin-istanbul/actions/workflows/ci.yml)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/pkg-nec/babel-plugin-istanbul/badge)](https://scorecard.dev/viewer/?uri=github.com/pkg-nec/babel-plugin-istanbul)

@pkg-nec/babel-plugin-istanbul is a security-maintained compatible fork of
[istanbuljs/babel-plugin-istanbul](https://github.com/istanbuljs/babel-plugin-istanbul).
It keeps the upstream plugin API and instrumentation behavior while
independently maintaining its dependency and supply-chain security posture.

For bugs and feature requests, use the
[pkg-nec/babel-plugin-istanbul issue tracker](https://github.com/pkg-nec/babel-plugin-istanbul/issues).
For suspected vulnerabilities, follow the [security policy](SECURITY.md).

A Babel plugin that instruments your code with Istanbul coverage.
It can instantly be used with [karma-coverage](https://github.com/karma-runner/karma-coverage) and mocha on Node.js (through [nyc](https://github.com/bcoe/nyc)).

__Note:__ This plugin does not generate any report or save any data to any file;
it only adds instrumenting code to your JavaScript source code.
To integrate with testing tools, please see the [Integrations](#integrations) section.

## Usage

Install it:

```
npm install --save-dev @pkg-nec/babel-plugin-istanbul
```

Add it to `.babelrc` in test mode:

```js
{
  "env": {
    "test": {
      "plugins": [ "@pkg-nec/babel-plugin-istanbul" ]
    }
  }
}
```

Optionally, use [cross-env](https://www.npmjs.com/package/cross-env) to set
`NODE_ENV=test`:

```json
{
  "scripts": {
    "test": "cross-env NODE_ENV=test nyc --reporter=lcov --reporter=text mocha test/*.js"
  }
}
```

## Integrations

### karma

It _just works_ with Karma. First, make sure that the code is already transpiled by Babel (either using `karma-babel-preprocessor`, `karma-webpack`, or `karma-browserify`). Then, simply set up [karma-coverage](https://github.com/karma-runner/karma-coverage) according to the docs, but __don’t add the `coverage` preprocessor.__ This plugin has already instrumented your code, and Karma should pick it up automatically.

It has been tested with [bemusic/bemuse](https://codecov.io/github/bemusic/bemuse) project, which contains ~2400 statements.

### mocha on node.js (through nyc)

Configure Mocha to transpile JavaScript code using Babel, then you can run your tests with [`nyc`](https://github.com/bcoe/nyc), which will collect all the coverage report.

babel-plugin-istanbul respects the `include`/`exclude` configuration options from nyc,
but you also need to __configure NYC not to instrument your code__ by adding these settings in your `package.json`:

```js
  "nyc": {
    "sourceMap": false,
    "instrument": false
  },
```

## Security

See the [security policy](SECURITY.md) to report suspected vulnerabilities.
GitHub's dependency graph and Dependabot surface newly disclosed dependency
vulnerabilities. The Dependency Review workflow prevents pull requests from
adding high-or-critical vulnerable dependencies. The project treats transitive
and development-tooling dependency risk as a maintenance concern, even when it
does not ship in the npm tarball.

## Funding

Support maintenance at [Buy Me a Coffee](https://buymeacoffee.com/maw629).

## Compatibility

This fork preserves the upstream plugin API and instrumentation behavior. It
will not intentionally diverge unless a future release documents that change.

## Releases

Maintainers: see the [release versioning guide](RELEASING.md). Published npm
releases include npm provenance. Documentation-, CI-, funding-, and
package-discovery-metadata-only changes do not cause an npm release.

## Ignoring files

You don't want to cover your test files as this will skew your coverage results. You can configure this by providing plugin options matching nyc's [`exclude`/`include` rules](https://github.com/bcoe/nyc#excluding-files):

```json
{
  "env": {
    "test": {
      "plugins": [
        ["@pkg-nec/babel-plugin-istanbul", {
          "exclude": [
            "**/*.spec.js"
          ]
        }]
      ]
    }
  }
}
```

If you don't provide options in your Babel config, the plugin will look for `exclude`/`include` config under an `"nyc"` key in `package.json`.

You can also use [istanbul's ignore hints](https://github.com/gotwarlost/istanbul/blob/master/ignoring-code-for-coverage.md#ignoring-code-for-coverage-purposes) to specify specific lines of code to skip instrumenting.

## Source Maps

By default, this plugin will pick up inline source maps and attach them to the instrumented code such that code coverage can be remapped back to the original source, even for multi-step build processes. This can be memory intensive. Set `useInlineSourceMaps` to prevent this behavior.

```json
{
  "env": {
    "test": {
      "plugins": [
        ["@pkg-nec/babel-plugin-istanbul", {
          "useInlineSourceMaps": false
        }]
      ]
    }
  }
}
```

If you're instrumenting code programatically, you can pass a source map explicitly.
```js
import babelPluginIstanbul from '@pkg-nec/babel-plugin-istanbul'

function instrument(sourceCode, sourceMap, filename) {
  return babel.transform(sourceCode, {
    filename,
    plugins: [
      [babelPluginIstanbul, {
        inputSourceMap: sourceMap
      }]
    ]
  })
}
```

## Upstream credit

The approach used in `babel-plugin-istanbul` was inspired by [Thai Pangsakulyanont](https://github.com/dtinth)'s original library [`babel-plugin-__coverage__`](https://github.com/dtinth/babel-plugin-__coverage__).
