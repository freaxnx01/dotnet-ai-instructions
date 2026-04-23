# Reference Makefile for .NET projects using the dotnet stack overlay.
# See ../../stacks/dotnet.md → "Essential Make Targets" for the rationale
# behind each target name. Copy this file to your repo root and customize
# the variables below.

# ── Project Configuration ────────────────────────────────────────────────────

# TODO: set to your solution file
SLN             := source/YourProject.sln

# TODO: set to your main API / host project directory
API_DIR         := source/YourProject.Api

# TODO: set to your frontend project directory (delete if no separate frontend)
CLIENT_DIR      := source/YourProject.Client

# Path to the file that holds the single <Version>X.Y.Z</Version> element.
# Convention: Directory.Build.props at the solution root.
PROPS_FILE      := source/Directory.Build.props

VERSION         := $(shell grep -oP '(?<=<Version>)[^<]+' $(PROPS_FILE))

# Compose files
COMPOSE := docker compose -f docker-compose.yml -f docker-compose.override.yml

# Test projects — add/remove as needed
TEST_UNIT_PROJECTS := \
	source/YourProject.UnitTests

# ── Targets ──────────────────────────────────────────────────────────────────

.PHONY: build watch run-edge \
        test test-unit test-coverage \
        docker-run up down logs rebuild \
        lint outdated vuln \
        version version-set bump-major bump-minor bump-patch bump-auto \
        changelog release-notes release release-auto push-release package \
        clean help

# ── Build & run ──────────────────────────────────────────────────────────────

build: ## Build solution in Release mode
	dotnet build $(SLN) -c Release

watch: ## Run API with hot reload
	cd $(API_DIR) && dotnet watch

run-edge: ## Start frontend and open in default/preferred browser
	# TODO: implement per-project. Canonical target name; recipe is host-specific.
	#   Windows/WSL: cd $(CLIENT_DIR) && dotnet run --launch-profile http & \
	#                sleep 3 && powershell.exe -Command "Start-Process 'msedge' 'http://localhost:5173'"
	#   macOS:       cd $(CLIENT_DIR) && dotnet run --launch-profile http & \
	#                sleep 3 && open -a Safari http://localhost:5173
	#   Linux:       cd $(CLIENT_DIR) && dotnet run --launch-profile http & \
	#                sleep 3 && xdg-open http://localhost:5173
	@echo "run-edge: implement in your project Makefile"; exit 1

# ── Testing ──────────────────────────────────────────────────────────────────

test: ## Run all tests
	dotnet test $(SLN)

test-unit: ## Run unit tests only
	@for proj in $(TEST_UNIT_PROJECTS); do \
		dotnet test $$proj || exit 1; \
	done

test-coverage: ## Run all tests with coverage
	dotnet test $(SLN) --collect:"XPlat Code Coverage" --results-directory ./coverage
	@echo "Coverage reports in ./coverage/"

# ── Docker (Compose) ─────────────────────────────────────────────────────────

docker-run: ## Run with docker-compose in foreground (Ctrl+C to stop)
	$(COMPOSE) up --build

up: ## Start in background with docker-compose
	$(COMPOSE) up -d --build

down: ## Stop docker-compose
	$(COMPOSE) down

logs: ## Follow container logs
	$(COMPOSE) logs -f

rebuild: down up ## Rebuild and restart

# ── Quality ──────────────────────────────────────────────────────────────────

lint: ## Check code formatting
	dotnet format --verify-no-changes

outdated: ## Check for outdated packages
	dotnet list $(SLN) package --outdated

vuln: ## Check for vulnerable packages
	dotnet list $(SLN) package --vulnerable --include-transitive

# ── Versioning (single source of truth: Directory.Build.props → <Version>) ───

version: ## Show current version
	@echo $(VERSION)

version-set: ## Set version explicitly (usage: make version-set V=1.2.3)
	@if [ -z "$(V)" ]; then echo "Usage: make version-set V=1.2.3"; exit 1; fi
	@sed -i 's|<Version>$(VERSION)</Version>|<Version>$(V)</Version>|' $(PROPS_FILE)
	@echo "Version: $(VERSION) → $(V)"

bump-major: ## Bump major version (1.2.3 → 2.0.0)
	@MAJOR=$$(echo $(VERSION) | cut -d. -f1); \
	NEW=$$((MAJOR + 1)).0.0; \
	sed -i "s|<Version>$(VERSION)</Version>|<Version>$$NEW</Version>|" $(PROPS_FILE); \
	echo "Version: $(VERSION) → $$NEW"

bump-minor: ## Bump minor version (1.2.3 → 1.3.0)
	@MAJOR=$$(echo $(VERSION) | cut -d. -f1); \
	MINOR=$$(echo $(VERSION) | cut -d. -f2); \
	NEW=$$MAJOR.$$((MINOR + 1)).0; \
	sed -i "s|<Version>$(VERSION)</Version>|<Version>$$NEW</Version>|" $(PROPS_FILE); \
	echo "Version: $(VERSION) → $$NEW"

bump-patch: ## Bump patch version (1.2.3 → 1.2.4)
	@MAJOR=$$(echo $(VERSION) | cut -d. -f1); \
	MINOR=$$(echo $(VERSION) | cut -d. -f2); \
	PATCH=$$(echo $(VERSION) | cut -d. -f3); \
	NEW=$$MAJOR.$$MINOR.$$((PATCH + 1)); \
	sed -i "s|<Version>$(VERSION)</Version>|<Version>$$NEW</Version>|" $(PROPS_FILE); \
	echo "Version: $(VERSION) → $$NEW"

bump-auto: ## Bump version from Conventional Commits via git-cliff (minor/patch only)
	@NEW=$$(git-cliff --bumped-version 2>/dev/null | sed 's/^v//'); \
	if [ -z "$$NEW" ]; then echo "Error: git-cliff did not return a bumped version"; exit 1; fi; \
	CUR_MAJOR=$$(echo $(VERSION) | cut -d. -f1); \
	NEW_MAJOR=$$(echo $$NEW | cut -d. -f1); \
	if [ "$$CUR_MAJOR" != "$$NEW_MAJOR" ]; then \
		echo "Error: auto-bump suggests major version change ($(VERSION) → $$NEW)."; \
		echo "  Major bumps require explicit action. Run: make bump-major"; \
		exit 1; \
	fi; \
	if [ "$$NEW" = "$(VERSION)" ]; then \
		echo "Version unchanged: $(VERSION) (no bump-worthy commits since last tag)"; \
		exit 0; \
	fi; \
	sed -i "s|<Version>$(VERSION)</Version>|<Version>$$NEW</Version>|" $(PROPS_FILE); \
	echo "Version: $(VERSION) → $$NEW (auto)"

# ── Release ──────────────────────────────────────────────────────────────────

changelog: ## Generate CHANGELOG.md from Conventional Commits (git-cliff)
	git-cliff --output CHANGELOG.md
	@echo "CHANGELOG.md updated."

release-notes: ## Generate user-friendly release notes for current version
	# TODO: implement per-project. Canonical target name; recipe is tool-specific.
	#   Claude Code:  claude -p "/release-notes v$(VERSION)"
	#   Copilot CLI:  copilot -p "/release-notes v$(VERSION)"
	#   llm CLI:      llm -m gpt-4o < prompts/release-notes.md > RELEASENOTES.md
	#   Hand-rolled:  your script here
	@echo "release-notes: implement in your project Makefile"; exit 1

release: ## Tag release, regenerate changelog + release-notes, and commit
	@if git tag -l "v$(VERSION)" | grep -q .; then \
		echo "Error: Tag v$(VERSION) already exists."; \
		echo "  Already released? Run: make package && make push-release"; \
		echo "  Need a new version? Run: make bump-patch (or bump-minor / bump-major)"; \
		exit 1; \
	fi
	@echo "── Releasing v$(VERSION)..."
	git-cliff --tag "v$(VERSION)" --output CHANGELOG.md
	$(MAKE) release-notes
	git add $(PROPS_FILE) CHANGELOG.md RELEASENOTES.md
	git commit -m "chore: release v$(VERSION)"
	git tag -a "v$(VERSION)" -m "release: v$(VERSION)"
	@echo "── Released v$(VERSION). Don't forget to push: make push-release"

release-auto: ## Auto-bump version from commits then release (minor/patch only)
	$(MAKE) bump-auto
	$(MAKE) release

push-release: ## Push main branch and current version tag to origin
	git push origin main "v$(VERSION)"

package: ## Build distributable artifact (ZIP / tarball / image) and deliver
	# TODO: implement per-project. Canonical target name; recipe is project-specific.
	#   ZIP into a shared folder:
	#     powershell.exe -ExecutionPolicy Bypass -File build/Build-Package.ps1
	#     cp .build/*.zip /mnt/c/your-deploy-folder/
	#   Tarball + signed checksum:
	#     dotnet publish -c Release -o out && tar czf app-$(VERSION).tar.gz out/
	#     sha256sum app-$(VERSION).tar.gz > app-$(VERSION).sha256
	#   Docker image:
	#     docker build -t registry.example.com/your-image:$(VERSION) .
	#     docker push registry.example.com/your-image:$(VERSION)
	@echo "package: implement in your project Makefile"; exit 1

# ── Cleanup ──────────────────────────────────────────────────────────────────

clean: ## Remove build artifacts
	find . -type d \( -name bin -o -name obj -o -name publish \) -exec rm -rf {} + 2>/dev/null || true
	rm -rf coverage/

# ── Help ─────────────────────────────────────────────────────────────────────

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

.DEFAULT_GOAL := help
