# Archynt Coverage Upload

Uploads the coverage report your tests already produce to Archynt, which shows the project's line
coverage and the least covered classes next to its architecture diagram.

Archynt never runs your tests: your CI does, and this step sends the resulting report.

## Usage

1. In the Archynt dashboard, open the project → **Line coverage** → **Set up CI upload** →
   **Generate token**, and save it in your repository as the `ARCHYNT_COVERAGE_TOKEN` secret.
2. Add a step after your tests:

```yaml
name: Coverage
on:
  push:
    branches: [main]   # the branch the Archynt project follows; feeds the dashboard
  pull_request:

jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn -B verify
      - uses: archynt/coverage-action@v1
        with:
          token: ${{ secrets.ARCHYNT_COVERAGE_TOKEN }}
          file: target/site/jacoco/jacoco.xml
```

| Language | Generate the report | `file` |
|---|---|---|
| Java | `jacoco-maven-plugin` (`report` goal) or Gradle `jacocoTestReport` with XML enabled | `target/site/jacoco/jacoco.xml` |
| Node/TS | `jest --coverage --coverageReporters=lcov --collectCoverageFrom='src/**/*.{js,ts}'` or `vitest --coverage --coverage.reporter=lcov` | `coverage/lcov.info` |
| Python | `pytest --cov=<package> --cov-report=xml` | `coverage.xml` |
| Go | `go test -coverprofile=coverage.out ./...` | `coverage.out` |

The format is detected from the content. Configure your runner to report **every** source file
(`collectCoverageFrom` in Jest, `--cov=<package>` in pytest); otherwise files no test loads are
missing from the report instead of showing 0%.

## How it works

The step gzips the report and posts it with the branch, the commit and the repository's default
branch. Archynt keeps the latest report per branch: the project's branch is what the dashboard
shows; reports from pull request branches are stored separately and never replace it.

## Other CI

```bash
curl -sS --fail-with-body -X POST \
  "https://api.archynt.com/api/v1/coverage/upload?branch=$BRANCH&commit=$COMMIT_SHA" \
  -H "Authorization: Bearer $ARCHYNT_COVERAGE_TOKEN" \
  --data-binary @coverage.xml
```

Limits: 25 MB per upload (gzip it if larger), 30 uploads per minute per token.
