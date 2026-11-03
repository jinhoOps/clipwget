# Repository Guidelines

## Project Structure & Module Organization

clipwget is a lightweight, clipboard-driven media downloader for Windows, planned in Go and licensed under MIT. The repository currently contains `README.md`, `LICENSE`, the Go-oriented `.gitignore`, architecture and roadmap docs in `docs/` (`docs/software-design.md` and `docs/implementation-plan.md`), and local agent guidance in `CLAUDE.md`. Source packages, tests, and `go.mod` (`github.com/jinhoOps/clipwget`) are scheduled according to `docs/implementation-plan.md`.

When adding implementation, consult `docs/software-design.md` and update milestone checkboxes in `docs/implementation-plan.md`. Organize files by Go package and keep tests beside the code they exercise. Document new directories and entry points in `README.md`; update agent guidance when the architecture changes.

## Build, Test, and Development Commands

No build, run, or test commands work yet because the Go module and application have not been created. After initializing the module, use these standard commands from the repository root:

- `go build ./...` — compile all packages.
- `go test ./...` — run all package tests.
- `go vet ./...` — check for common Go mistakes.
- `gofmt -w <file.go>` — format changed Go files.

Document the actual application launch command once an entry point exists. Validate Windows-specific functionality on Windows using PowerShell or Git Bash.

## Coding Style & Naming Conventions

Use standard Go formatting through `gofmt`, including its tab indentation. Use short, lowercase package names, exported `MixedCaps` identifiers, and unexported `mixedCaps` identifiers. Prefer descriptive filenames such as `clipboard.go` and matching `clipboard_test.go`. No additional formatter or linter is configured.

## Testing Guidelines

Use Go's standard `testing` package for new tests, with filenames ending in `_test.go` and functions named `TestXxx`. No coverage threshold currently exists. Test URL handling and download behavior with deterministic fixtures where practical, and document manual checks for clipboard integration and other Windows-specific behavior.

## Commit & Pull Request Guidelines

Git history currently contains only `Initial commit`, so no established message convention exists. Write concise, imperative commit subjects describing the change, such as `Add clipboard URL detection`.

Pull requests should explain the behavior changed, link relevant issues, and list automated tests and manual Windows checks performed. Include screenshots when a visible interface changes. Update documentation when adding commands, configuration, or dependencies.

## Security & Configuration

Do not commit secrets, `.env` files, generated executables, test binaries, or coverage artifacts. Follow the existing `.gitignore`; document required configuration without including credentials or private clipboard contents.