# Contributing to AkesoDLP

Start with a focused issue or pull request that explains the observed problem, expected behavior, and a reproducible example. Keep changes small enough to review. Retain upstream attribution and license notices.

## Local checks

Run from the repository root, using a clean dependency installation:

```sh
python -m pip install -r requirements.txt
python -m pytest tests --ignore=tests/integration --ignore=tests/benchmark -q
cd console && npm ci && npm run build
```

See [the CI workflow](.github/workflows/ci.yml) for operating system and runtime versions. Windows projects require Visual Studio 2022 C++ build tools; kernel-driver testing additionally requires a separate authorized Windows lab and the WDK. AkesoNDR requires libpcap development headers. Python checks use Python 3.12; Node checks use Node 24.

## Pull request evidence

Describe what changed, why, and which commands passed. State skipped checks and environment limitations explicitly. A passing unit test or frontend build does not establish production readiness, live connector behavior, or kernel-driver coverage. Include regression coverage for changed behavior when practical.

Use synthetic fixtures and redact logs. For vulnerabilities, follow [SECURITY.md](SECURITY.md) instead of posting exploit details publicly. Attribute AI-assisted work and third-party code accurately; review and validate generated changes before submitting them.
