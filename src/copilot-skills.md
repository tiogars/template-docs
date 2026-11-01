# Recommended Copilot Skills

Use these Copilot skills when evolving this monorepo. They help keep work focused, validated, and aligned with the technology in each module.

## Core Project Work

| Skill | Use When |
| --- | --- |
| `project-setup-info-local` | Creating or reshaping the monorepo structure, adding a new top-level module, or bootstrapping a new framework area. |
| `documentation-writer` | Writing or reorganizing MkDocs content, README sections, tutorials, how-to guides, reference pages, or explanations. |
| `refactor` | Improving maintainability without changing behavior, such as extracting methods, simplifying clients, or reducing duplication. |

## Java and Spring Boot

| Skill | Use When |
| --- | --- |
| `java-springboot` | Adding API endpoints, configuration, validation, controllers, services, or Spring Boot production conventions in `api`. |
| `java-junit` | Adding or improving unit and integration-style tests for `java/api` and `java/java-client`. |
| `java-lsp-tools` | Navigating Java symbols precisely before changing Java classes, methods, or package structure. |

## Frontend and TypeScript

| Skill | Use When |
| --- | --- |
| `vite` | Changing Vite configuration, build behavior, dev server settings, or frontend packaging in `backend`. |
| `typescript-dependencies-upgrade` | Updating npm dependencies for `react/backend`, `react/react-client`, or `docs/presentation`, except the TypeScript compiler itself. |
| `typescript-compiler-upgrade` | Upgrading the `typescript` package or testing a major compiler migration. |
| `web-design-guidelines` | Reviewing the management console UI for usability, accessibility, responsive behavior, and visual quality. |

## Dart Client

| Skill | Use When |
| --- | --- |
| `flutter-use-http-package` | Extending the Dart client API methods or HTTP behavior. |
| `flutter-implement-json-serialization` | Adding typed Dart models or changing JSON mapping rules in `flutter/dart-client`. |

## Documentation, Diagrams, and Presentations

| Skill | Use When |
| --- | --- |
| `archify` | Creating architecture, sequence, workflow, data-flow, or lifecycle diagrams from the implementation. |
| `md-to-docx` | Exporting Markdown documentation into a polished Word document. |

## Delivery and Operations

| Skill | Use When |
| --- | --- |
| `cve-remediation` | Scanning and remediating dependency or container vulnerabilities. |
| `runtime-validation` | Designing or executing smoke, integration, startup, or end-to-end validation for the deployable stack. |
| `github-actions-hardening` | Creating or reviewing GitHub Actions workflows for secure CI/CD. |
| `github-actions-efficiency` | Optimizing GitHub Actions runtime, caching, and cost. |

## Suggested Defaults

- For API behavior changes, start with `java-springboot`, then use `java-junit` for tests.
- For frontend build or UI work, start with `vite`; add `web-design-guidelines` for user-facing screens.
- For client model changes, update Java, TypeScript, and Dart clients together, then run all module tests.
- For deployment changes, combine `runtime-validation` with `cve-remediation` before publishing images.