# Extract Git Version Action

A GitHub composite action for extracting semantic version information from git tags using a gitversion-style approach.

## Features

- Parses version from git tags (e.g., `v1.2.3`)
- Computes dev versions for commits after a tag
- Provides both semver-compliant (`+`) and tag-compatible (`-`) version strings
- Exposes version components, state flags, and build metadata

## Usage

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Required for version detection

- uses: jmylchreest/gha-extract-git-version@v1
  id: version

- run: echo "Version: ${{ steps.version.outputs.version }}"
```

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `prefix` | Tag prefix | `v` |

## Outputs

### Version Strings

| Output | Description | Example |
|--------|-------------|---------|
| `version` | Semver-compliant version (uses `+` for build metadata) | `1.2.3-dev.5+abc1234` |
| `version_tag` | Tag-compatible version (uses `-` instead of `+`) | `v1.2.3-dev.5-abc1234` |
| `version_core` | Core version without prerelease/build metadata | `1.2.3` |

### Version Components

| Output | Description | Example |
|--------|-------------|---------|
| `major` | Major version number | `1` |
| `minor` | Minor version number | `2` |
| `patch` | Patch version number | `3` |
| `commits` | Commits since last tag | `5` |

### Previous/Next Versions

| Output | Description | Example |
|--------|-------------|---------|
| `last_version` | Last tagged version (without prefix) | `1.2.2` |
| `last_tag` | Last tag (with prefix) | `v1.2.2` |
| `next_version` | Next patch version (for dev builds) | `1.2.3` |

### State Flags

| Output | Description | Values |
|--------|-------------|--------|
| `is_tagged` | Current commit has a version tag | `true`/`false` |
| `is_release` | Building from a version tag ref | `true`/`false` |
| `is_prerelease` | Version contains prerelease identifier | `true`/`false` |

### Build Metadata

| Output | Description | Example |
|--------|-------------|---------|
| `sha` | Short commit SHA (7 chars) | `abc1234` |
| `sha_full` | Full commit SHA | `abc1234...` |
| `date` | Build timestamp (ISO 8601 UTC) | `2024-01-15T10:30:00Z` |
| `dirty` | Working tree has uncommitted changes | `true`/`false` |

## Version Calculation Logic

1. **Release build** (triggered by tag push `refs/tags/v*`):
   - Uses the tag version directly
   - `is_release=true`, `is_prerelease=false`

2. **On a tagged commit** (commits=0):
   - Uses the tag version
   - `is_tagged=true`, `is_prerelease=false`

3. **After a tag** (commits > 0):
   - Increments patch, adds prerelease suffix
   - Example: Tag `v1.2.3` + 5 commits = `1.2.4-dev.5+abc1234`
   - `is_prerelease=true`

4. **No tags exist**:
   - Uses `0.0.1-dev.{commit_count}+{sha}`
   - `is_prerelease=true`

## Examples

### Inject Version via ldflags (Go)

```yaml
- uses: jmylchreest/gha-extract-git-version@v1
  id: version

- name: Build
  run: |
    go build -ldflags "-X main.version=${{ steps.version.outputs.version }}" -o app
```

### Conditional Release vs Prerelease

```yaml
- uses: jmylchreest/gha-extract-git-version@v1
  id: version

- name: Create Release
  run: |
    if [[ "${{ steps.version.outputs.is_release }}" == "true" ]]; then
      gh release create "${{ steps.version.outputs.version_tag }}" --title "${{ steps.version.outputs.version }}"
    else
      gh release create "${{ steps.version.outputs.version_tag }}" --prerelease --title "Snapshot ${{ steps.version.outputs.version }}"
    fi
```

### Docker Build Args

```yaml
- uses: jmylchreest/gha-extract-git-version@v1
  id: version

- name: Build Docker Image
  run: |
    docker build \
      --build-arg VERSION=${{ steps.version.outputs.version }} \
      --build-arg COMMIT=${{ steps.version.outputs.sha }} \
      --build-arg DATE=${{ steps.version.outputs.date }} \
      -t myapp:${{ steps.version.outputs.version_core }} .
```

## License

MIT License
