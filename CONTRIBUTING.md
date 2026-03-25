# Contributing

Thanks for your interest in improving the observability stack! This document provides guidelines for contributing.

## Code of Conduct

Be respectful and constructive. Disagreements are welcome; personal attacks are not.

## Reporting Issues

### Bugs
If you find a bug, please create a [Bug Report](https://github.com/your-org/home-server-observability-stack/issues/new?template=bug_report.md) with:
- A clear description of the issue
- Steps to reproduce it
- Expected vs actual behavior
- Relevant logs (with passwords redacted)
- Your environment (OS, Docker version, etc.)

### Configuration Questions
If you have questions about setup or configuration, use the [Configuration Help](https://github.com/your-org/home-server-observability-stack/issues/new?template=config_help.md) template with:
- What you're trying to do
- What you've already tried
- Your setup details
- Relevant config (with secrets redacted)

### Feature Requests
For feature suggestions, create an issue with title `[FEATURE]` and describe:
- What feature you'd like
- Why it would be useful
- How you imagine it working

## Pull Requests

Contributions are welcome! For significant changes, please open an issue first to discuss.

### Guidelines

1. **Fork and branch** — Create a feature branch: `git checkout -b feature/your-feature`
2. **Small, focused PRs** — Each PR should address one thing
3. **Update documentation** — If you add configuration options, update `.env.example` and `README.md`
4. **No secrets** — Never commit `.env`, API keys, passwords, or webhook URLs
5. **Test locally** — Verify the stack starts and functions before submitting

### What We Accept

- Config improvements and clarifications
- Documentation fixes and enhancements
- bug fixes with test cases
- Alert rules improvements
- Grafana dashboard recommendations

### What We Don't Accept

- Significant architectural changes without discussion
- Kubernetes manifests (keep focus on docker-compose)
- Non-observability add-ons (stick to monitoring scope)

## Submission Process

1. Push to your fork
2. Create a Pull Request with a clear title and description
3. Link any related issues
4. Request review from maintainers
5. Address feedback and make requested changes
6. Once approved, your PR will be merged

## Questions?

- Check the [README](README.md) for setup and troubleshooting
- Review [existing issues](https://github.com/your-org/home-server-observability-stack/issues) for similar questions
- Open a [Configuration Help](https://github.com/your-org/home-server-observability-stack/issues/new?template=config_help.md) issue

## Licensing

By contributing, you agree that your contributions will be licensed under the same MIT license as the project.

Thank you for helping make this better! 🎉
