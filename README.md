# codeclimate-test-reporter-buildkite-plugin

A BuildKite plugin

https://buildkite.com/docs/agent/plugins

to report coverage with the Code Climate test reporter

https://github.com/codeclimate/test-reporter

Plugin can handle single, [parallel](https://docs.codeclimate.com/docs/configuring-test-coverage#parallel-tests) and [multiple](https://docs.codeclimate.com/docs/configuring-test-coverage#multiple-test-suites) test suites.

Also see: https://docs.codeclimate.com/docs/configuring-test-coverage

## Usage:

This plugin will download build artifact(s) generated by a previous step, compile and report to Code Climate.

```yml
steps:
  - label: ":codeclimate: Report coverage"
    plugins:
      - jobready/codeclimate-test-reporter#v2.2:
          artifact: "coverage/.resultset.json"
          input_type: simplecov
          prefix: /app
    env:
      CC_TEST_REPORTER_ID:
```

### Parallel tests

If you are running parallel builds you'll probably want to create uniquely
named artifacts. That can be done by renaming the `.resultset.json` file
after your tests have run:

```bash
mv -v coverage/.resultset.json coverage/.resultset${BUILDKITE_PARALLEL_JOB}.json
```

### Multiple test suites

If your repository has more than one test suite you'll need to format each artifact (coverage file) before the result can be summed and reported to codeclimate.

To do this you'll need to configure multiple steps persisting the formated codeclimate compatable configuration files as artifacts between steps.

This is best shown via an example;

```
  - label: ":rails: Tests"
    key: "rails-tests"
    command: "rails db:test:prepare && rails test"
    env:
      RUBY_OPT: "-W:deprecated"
    depends_on:
      - "build-base"
    plugins:
      - docker-compose#v3.7.0:
          run: base
          config: .buildkite/docker-compose.yml
          volumes:
            - "./coverage:/app/coverage"
    artifact_paths:
      - "coverage/**/*"

  - label: ":codeclimate: Format Rails Coverage"
    key: "codeclimate-format-rails-report"
    plugins:
      - jobready/codeclimate-test-reporter#v2.2:
          artifact: "coverage/coverage.json"
          input_type: simplecov
          prefix: /app
          report: false
          file_prefix: "codeclimate.rails"
    depends_on:
      - "rails-tests"
    artifact_paths:
      - "coverage/codeclimate.**"

  - label: ":react: Tests"
    key: "react-tests"
    command: "yarn test"
    depends_on:
      - "build-base"
    plugins:
      - docker-compose#v3.7.0:
          run: base
          config: .buildkite/docker-compose.yml
          volumes:
            - "./coverage:/app/coverage"
    artifact_paths:
      - "coverage/**/*"

  - label: ":codeclimate: Format React Coverage"
    key: "codeclimate-format-react-report"
    plugins:
      - jobready/codeclimate-test-reporter#v2.2:
          artifact: "coverage/lcov.info"
          input_type: lcov
          prefix: /app
          report: false
          file_prefix: "codeclimate.react"
    depends_on:
      - "react-tests"
    artifact_paths:
      - "coverage/codeclimate.**"

  - label: ":codeclimate: Report coverage"
    key: "codeclimate-coverage-report"
    plugins:
      - jobready/codeclimate-test-reporter#v2.2:
          artifact: "coverage/codeclimate**"
          format: false
    env:
      CC_TEST_REPORTER_ID:
    depends_on:
      - "codeclimate-format-rails-report"
      - "codeclimate-format-react-report"
```

## Configuration

### `artifact` (required)

Passed through as the `[COVERAGE FILE]` argument to

https://github.com/codeclimate/test-reporter/blob/master/man/cc-test-reporter-format-coverage.1.md

Example: `coverage/.resultset.json`

Would be the artifact path uploaded by a previous step. Use a wildcard for multiple artifacts

Example: `coverage/.resultset*.json`

### `input_type` (required)

Passed through to the --input-type option of

https://github.com/codeclimate/test-reporter/blob/master/man/cc-test-reporter-format-coverage.1.md

Example: `simplecov`

### `prefix` (optional)

Passed through to the --prefix option of

https://github.com/codeclimate/test-reporter/blob/master/man/cc-test-reporter-format-coverage.1.md

Example: `/app`

If the coverage was generated from a Docker container, prefix would be the Dockerfile `WORKDIR`.

### `version` (optional)

The preferred version of the test reporter to download. Defaults to `latest`.

Example: `0.4.3`

### `parts` (optional)

If you expect multiple partial coverage artifacts, set this value to enforce a check. If not set the plugin will proceed with any/all provided parts.

Example: `2`

### `debug` (optional)

Set to true to enable cc-test-reporter --debug flag

Example: `true`

### `format` (optional, default `true`)

Controls if formatting of artifacts should be performed. Set to false when wanting to upload artifacts formatted in a previous step.

Example: `false`

### `report` (optional, default `true`)

Controls if coverage reporting to codeclimate should be performed. Set to false when needing to parse multiple test suites for later upload.

Example: `false`

### `file_prefix` (optional, default `codeclimate`)

Controls the name outputs files formatted by the codeclimate test reporter. Required to stop artifact names clashing when processing muiltple test suites.

### `CC_TEST_REPORTER_ID` (required)

The `CC_TEST_REPORTER_ID` environment variable must be configured.

## Linting

To run the [Buildkite Plugin Linter](https://github.com/buildkite-plugins/buildkite-plugin-linter), run

```sh
docker-compose run --rm lint --name jobready/codeclimate-test-reporter
```

## License

MIT
