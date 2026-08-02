# Repository Guidelines

## Project Structure & Module Organization

Keep production source in `src/`; the published CommonJS build is generated in `lib/`. Unit tests, including shared Babel 7 and conditionally supported Babel 8 cases, live in `test/babel-plugin-istanbul.js`; `test/babel-8/` holds the Babel 8 workspace dependency manifest. Add or update representative instrumented input under `fixtures/` when a behavior needs a concrete fixture. Continuous-integration definitions are in `.github/workflows/`; keep local work aligned with the CI matrix and commands.

## Build, Test, and Development Commands

Use the repository's pinned Yarn release and clean-install flow before validating changes:

```bash
corepack enable
yarn install --immutable
yarn test
```

`yarn test` runs StandardJS linting, the Babel build, and NYC/Mocha unit tests. For a focused loop, use `yarn lint`, `yarn build`, or `yarn test:unit` as appropriate, then run the full command before handing off a change.

## Coding Style & Naming Conventions

Follow StandardJS: two-space indentation, no semicolons, and idiomatic modern JavaScript. Preserve the existing module boundaries and naming patterns; use concise camelCase identifiers and descriptive test names. Source files are authored in `src/`; do not hand-edit generated `lib/` output.

## Testing Guidelines

Place test coverage in the matching `test/*.js` area and keep tests deterministic. Changes that affect transformations, configuration loading, or instrumentation should cover the relevant fixture and expected output. Preserve the shared cases' coverage under both Babel 7 and conditionally supported Babel 8; compatibility work must not silently reduce either runtime's protection.

## Change Safety

Keep the committed `yarn.lock` synchronized with intentional dependency changes and retain it in reviews. Avoid incidental reformatting or generated-file churn. Do not change release automation, publishing behavior, or workflow configuration without explicit instruction.

## Commit & Pull Request Guidelines

Use Conventional Commit-style subjects, such as `fix: handle missing input` or `docs: clarify setup`. Keep commits narrowly scoped. Pull requests should explain the behavior change, list validation run, and call out compatibility or lockfile effects; link the motivating issue when available.
