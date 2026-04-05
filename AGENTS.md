# Repository Guidelines

## Project Structure & Module Organization
`apps/sdkwork-fs` is currently a documentation-first workspace for the SdkworkFS idea and presently contains `README.md` only. It sits inside the larger `spring-ai-plus` monorepo. Backend modules live at the repository root, for example `spring-ai-plus-filesystem`, `spring-ai-plus-files`, and `spring-ai-plus-business-service`. Shared documentation lives in `docs/`, and other app experiments live under `apps/`. If implementation code is added here later, follow the monorepo layout: Java in `src/main/java`, tests in `src/test/java`, and feature-specific docs close to the code they describe.

## Build, Test, and Development Commands
This folder does not have a standalone build yet. Run commands from the monorepo root:

- `mvn clean install` - build all modules and run tests.
- `mvn clean install -pl spring-ai-plus-filesystem -am` - build the filesystem module and required dependencies.
- `mvn test` - run the full test suite.
- `mvn test -pl spring-ai-plus-business-service` - run a focused backend module test set.

Use targeted module commands while iterating, then run a broader root-level build before opening a PR.

## Coding Style & Naming Conventions
Use Java 21, UTF-8, and 4-space indentation. Follow package prefixes such as `com.sdkwork.spring.ai.plus.*`. Use `PascalCase` for classes, `camelCase` for methods and fields, `UPPER_SNAKE_CASE` for constants, and `*Test` for test classes. Keep imports organized and avoid duplicate imports. For Markdown in this folder, prefer short sections, actionable wording, and fenced blocks for commands or config snippets.

## Testing Guidelines
Backend tests use JUnit 5 with Mockito where needed. Cover both happy-path and error-path behavior, and name tests by behavior, for example `testCreateMountConfig`. No app-local automated test harness exists in `apps/sdkwork-fs` yet, so documentation-only changes should still verify commands, paths, and examples against the monorepo.

## Commit & Pull Request Guidelines
Recent history uses short, scoped, imperative commits such as `fix: repair broken sdk generator strings`, `chore: internationalize and translate Java source text`, and `feat: harden bootstrap deployment and sdkwork ui`. Follow the same pattern: `<type>: <brief summary>`.

Pull requests should include the purpose of the change, affected modules or paths, test commands executed with results, linked task or issue IDs, and screenshots only when UI or visual docs change.

## Security & Configuration Tips
Do not commit API keys, tokens, passwords, or machine-local settings. Keep environment-specific values in local config, and do not log sensitive payloads in examples or tests.
