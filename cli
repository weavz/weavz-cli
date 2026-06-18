#!/bin/sh
set -eu

PACKAGE_NAME="@weavz-io/cli"
PACKAGE_PATH="@weavz-io%2fcli"
REGISTRY_URL="${WEAVZ_NPM_REGISTRY:-https://registry.npmjs.org}"
VERSION_SELECTOR="${WEAVZ_VERSION:-latest}"
TARBALL_URL="${WEAVZ_CLI_TARBALL_URL:-}"
TARBALL_INTEGRITY=""
TMP_DIR=""

log() {
  printf '%s\n' "$*"
}

fail() {
  printf 'weavz installer: %s\n' "$*" >&2
  exit 1
}

cleanup() {
  if [ -n "$TMP_DIR" ] && [ -d "$TMP_DIR" ]; then
    rm -rf "$TMP_DIR"
  fi
}
trap cleanup EXIT INT TERM

need_command() {
  command -v "$1" >/dev/null 2>&1 || fail "missing required command: $1"
}

need_command curl
need_command tar
need_command node
need_command mktemp
need_command date

NODE_BIN="$(command -v node)"

escape_single_quotes() {
  printf '%s' "$1" | sed "s/'/'\\\\''/g"
}

shell_quote() {
  printf "'%s'" "$(escape_single_quotes "$1")"
}

validate_version() {
  node - "$1" <<'NODE' || fail "invalid Weavz CLI package version: $1"
const version = process.argv[2] || ''
const semver = /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(?:-[0-9A-Za-z-]+(?:\.[0-9A-Za-z-]+)*)?(?:\+[0-9A-Za-z-]+(?:\.[0-9A-Za-z-]+)*)?$/
process.exit(semver.test(version) ? 0 : 1)
NODE
}

node -e '
const [major, minor, patch] = process.versions.node.split(".").map(Number)
const ok = major > 22 || (major === 22 && (minor > 13 || (minor === 13 && patch >= 0)))
process.exit(ok ? 0 : 1)
' || fail "Node.js 22.13.0 or newer is required. Install Node, then rerun this installer."

if [ -z "${WEAVZ_INSTALL_DIR:-}" ] && [ -z "${HOME:-}" ]; then
  fail "HOME is not set. Set WEAVZ_INSTALL_DIR and rerun this installer."
fi

INSTALL_ROOT="${WEAVZ_INSTALL_DIR:-$HOME/.weavz}"
INSTALL_ROOT="$(node -e 'const path = require("node:path"); process.stdout.write(path.resolve(process.argv[1]))' "$INSTALL_ROOT")"
BIN_DIR="$INSTALL_ROOT/bin"

REGISTRY_URL="${REGISTRY_URL%/}"

if [ -z "$TARBALL_URL" ]; then
  METADATA_URL="$REGISTRY_URL/$PACKAGE_PATH/$VERSION_SELECTOR"
  METADATA="$(
    node - "$METADATA_URL" <<'NODE'
const url = process.argv[2]

try {
  const response = await fetch(url, {
    headers: { accept: 'application/json' },
  })
  if (!response.ok) {
    throw new Error(`registry returned ${response.status} for ${url}`)
  }
  const metadata = await response.json()
  if (!metadata.version || !metadata.dist || !metadata.dist.tarball) {
    throw new Error('registry response did not include version and tarball fields')
  }
  process.stdout.write(`${metadata.version}\n${metadata.dist.tarball}\n${metadata.dist.integrity || ''}\n`)
} catch (error) {
  console.error(error && error.message ? error.message : String(error))
  process.exit(1)
}
NODE
  )" || fail "could not resolve $PACKAGE_NAME@$VERSION_SELECTOR from npm"

  RESOLVED_VERSION="$(printf '%s\n' "$METADATA" | sed -n '1p')"
  TARBALL_URL="$(printf '%s\n' "$METADATA" | sed -n '2p')"
  TARBALL_INTEGRITY="$(printf '%s\n' "$METADATA" | sed -n '3p')"
  validate_version "$RESOLVED_VERSION"
else
  RESOLVED_VERSION="${WEAVZ_VERSION:-}"
  if [ -n "$RESOLVED_VERSION" ]; then
    validate_version "$RESOLVED_VERSION"
  fi
fi

TMP_DIR="$(mktemp -d)"
TARBALL_FILE="$TMP_DIR/weavz-cli.tgz"
EXTRACT_DIR="$TMP_DIR/extract"

log "Downloading $PACKAGE_NAME${RESOLVED_VERSION:+@$RESOLVED_VERSION}..."
curl -fsSL --retry 2 --connect-timeout 15 "$TARBALL_URL" -o "$TARBALL_FILE"

if [ -n "$TARBALL_INTEGRITY" ]; then
  node - "$TARBALL_FILE" "$TARBALL_INTEGRITY" <<'NODE' || fail "downloaded tarball failed npm integrity verification"
const fs = await import('node:fs')
const crypto = await import('node:crypto')

const file = process.argv[2]
const integrity = process.argv[3]
const separator = integrity.indexOf('-')
if (separator < 1) {
  throw new Error('invalid npm integrity value')
}

const algorithm = integrity.slice(0, separator)
const expected = integrity.slice(separator + 1)
if (!['sha256', 'sha384', 'sha512'].includes(algorithm)) {
  throw new Error(`unsupported integrity algorithm: ${algorithm}`)
}

const actual = crypto.createHash(algorithm).update(fs.readFileSync(file)).digest('base64')
if (actual !== expected) {
  throw new Error('integrity digest mismatch')
}
NODE
fi

mkdir -p "$EXTRACT_DIR"
tar -xzf "$TARBALL_FILE" -C "$EXTRACT_DIR"

PACKAGE_DIR="$EXTRACT_DIR/package"
if [ ! -f "$PACKAGE_DIR/package.json" ] || [ ! -f "$PACKAGE_DIR/dist/cli.mjs" ]; then
  fail "downloaded package did not contain the Weavz CLI bundle"
fi

PACKAGE_VERSION="$(
  node - "$PACKAGE_DIR/package.json" <<'NODE'
const fs = require('node:fs')
const pkg = JSON.parse(fs.readFileSync(process.argv[2], 'utf8'))
process.stdout.write(pkg.version)
NODE
)"
validate_version "$PACKAGE_VERSION"

if [ -n "${RESOLVED_VERSION:-}" ] && [ "$RESOLVED_VERSION" != "$PACKAGE_VERSION" ]; then
  fail "downloaded package version $PACKAGE_VERSION did not match expected $RESOLVED_VERSION"
fi
RESOLVED_VERSION="$PACKAGE_VERSION"

DEST_PARENT="$INSTALL_ROOT/cli"
DEST="$DEST_PARENT/$RESOLVED_VERSION"
STAGING="$DEST.tmp.$$"

mkdir -p "$DEST_PARENT" "$BIN_DIR"
rm -rf "$STAGING"
mkdir -p "$STAGING"
cp -R "$PACKAGE_DIR/." "$STAGING/"
rm -rf "$DEST"
mv "$STAGING" "$DEST"

CLI_PATH="$(escape_single_quotes "$DEST/dist/cli.mjs")"
NODE_BIN_ESCAPED="$(escape_single_quotes "$NODE_BIN")"
cat > "$BIN_DIR/weavz" <<EOF
#!/bin/sh
if [ -x '$NODE_BIN_ESCAPED' ]; then
  exec '$NODE_BIN_ESCAPED' '$CLI_PATH' "\$@"
fi
exec node '$CLI_PATH' "\$@"
EOF
chmod 0755 "$BIN_DIR/weavz"

managed_begin="# >>> weavz cli >>>"
managed_end="# <<< weavz cli <<<"

configure_path_file() {
  profile_file="$1"
  path_line="$2"

  profile_dir="$(dirname "$profile_file")"
  mkdir -p "$profile_dir"

  if [ -f "$profile_file" ] &&
    grep -Fq "$managed_begin" "$profile_file" &&
    grep -Fq "$BIN_DIR" "$profile_file"; then
    log "PATH already configured in $profile_file"
    return 0
  fi

  if [ -f "$profile_file" ]; then
    backup_file="$profile_file.weavz-backup-$(date +%Y%m%d%H%M%S)"
    cp "$profile_file" "$backup_file"
    log "Backed up $profile_file to $backup_file"

    stripped_file="$profile_file.weavz-tmp.$$"
    awk -v begin="$managed_begin" -v end="$managed_end" '
      $0 == begin { skipping = 1; next }
      $0 == end { skipping = 0; next }
      skipping != 1 { print }
    ' "$profile_file" > "$stripped_file"
    mv "$stripped_file" "$profile_file"
  else
    : > "$profile_file"
  fi

  {
    printf '\n%s\n' "$managed_begin"
    printf '%s\n' "$path_line"
    printf '%s\n' "$managed_end"
  } >> "$profile_file"

  log "Added $BIN_DIR to PATH in $profile_file"
}

if [ "${WEAVZ_NO_PATH_UPDATE:-0}" = "1" ]; then
  CURRENT_SHELL_COMMAND="export PATH=$(shell_quote "$BIN_DIR"):\$PATH"
  log "PATH setup skipped because WEAVZ_NO_PATH_UPDATE=1."
elif [ -n "${HOME:-}" ]; then
  SHELL_NAME="$(basename "${SHELL:-sh}")"
  BIN_DIR_QUOTED="$(shell_quote "$BIN_DIR")"
  case "$SHELL_NAME" in
    fish)
      PROFILE_FILE="$HOME/.config/fish/conf.d/weavz.fish"
      PATH_LINE="fish_add_path -g $BIN_DIR_QUOTED"
      CURRENT_SHELL_COMMAND="set -gx PATH $BIN_DIR_QUOTED \$PATH"
      ;;
    zsh)
      PROFILE_FILE="${ZDOTDIR:-$HOME}/.zshrc"
      PATH_LINE="export PATH=$BIN_DIR_QUOTED:\$PATH"
      CURRENT_SHELL_COMMAND="export PATH=$BIN_DIR_QUOTED:\$PATH"
      ;;
    bash)
      PROFILE_FILE="$HOME/.bashrc"
      PATH_LINE="export PATH=$BIN_DIR_QUOTED:\$PATH"
      CURRENT_SHELL_COMMAND="export PATH=$BIN_DIR_QUOTED:\$PATH"
      ;;
    *)
      PROFILE_FILE="$HOME/.profile"
      PATH_LINE="export PATH=$BIN_DIR_QUOTED:\$PATH"
      CURRENT_SHELL_COMMAND="export PATH=$BIN_DIR_QUOTED:\$PATH"
      ;;
  esac

  configure_path_file "$PROFILE_FILE" "$PATH_LINE"
else
  CURRENT_SHELL_COMMAND="export PATH=$(shell_quote "$BIN_DIR"):\$PATH"
  log "HOME is not set, so PATH setup was skipped."
fi

INSTALLED_VERSION="$("$BIN_DIR/weavz" --version)"
if [ "$INSTALLED_VERSION" != "$RESOLVED_VERSION" ]; then
  fail "installed weavz version $INSTALLED_VERSION did not match expected $RESOLVED_VERSION"
fi

log "Installed Weavz CLI $RESOLVED_VERSION at $BIN_DIR/weavz"
log "For this shell session, run:"
log "  $CURRENT_SHELL_COMMAND"
