# Part 16: Git Automation

## Module 2: Intermediate Level (ระดับกลาง)

---

## ขั้นตอนที่ 374: Git Basics Review

```bash
#!/usr/bin/env bash
# git_basics.sh - Git พื้นฐาน

echo "=== Git Automation ==="
echo ""
echo "Git workflow:"
echo "  Working Directory → Staging Area → Repository → Remote"
echo ""

echo "Commands ที่ใช้บ่อย:"
cat << 'EOF'
# Setup
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
git config --global core.editor "vim"

# Repository
git init                    # สร้าง repo ใหม่
git clone URL               # clone repo
git clone --depth=1 URL     # shallow clone (เร็วกว่า)

# Status
git status
git status -s               # short format
git diff                    # unstaged changes
git diff --staged           # staged changes
git log --oneline --graph   # visual log

# Staging
git add file.txt            # stage file
git add -p                  # interactive stage
git add .                   # stage all
git reset HEAD file.txt     # unstage

# Committing
git commit -m "message"
git commit --amend          # แก้ commit ล่าสุด
git commit -am "message"    # stage+commit tracked files

# Branches
git branch                  # list branches
git branch feature          # สร้าง branch
git checkout feature        # switch
git checkout -b feature     # สร้าง+switch
git switch -c feature       # modern syntax
git merge feature           # merge
git branch -d feature       # ลบ branch
EOF

echo ""
echo "ดู git config:"
git config --global --list 2>/dev/null | head -10 || echo "(ไม่มี global config)"
```

---

## ขั้นตอนที่ 375: Git Automation Scripts

```bash
#!/usr/bin/env bash
# git_automation.sh - Git Automation

echo "=== Git Automation Scripts ==="

# ==================== Utility Functions ====================

# ตรวจสอบว่าอยู่ใน git repo
is_git_repo() {
    git rev-parse --is-inside-work-tree &>/dev/null
}

# ดู current branch
current_branch() {
    git rev-parse --abbrev-ref HEAD 2>/dev/null
}

# ดู repo root
repo_root() {
    git rev-parse --show-toplevel 2>/dev/null
}

# ตรวจสอบ uncommitted changes
has_changes() {
    ! git diff-index --quiet HEAD -- 2>/dev/null
}

# ตรวจสอบ staged changes
has_staged() {
    ! git diff --cached --quiet 2>/dev/null
}

# ดู remote URL
remote_url() {
    local remote="${1:-origin}"
    git remote get-url "$remote" 2>/dev/null
}

# Demo
if is_git_repo 2>/dev/null; then
    echo "1. Repo info:"
    echo "  Root: $(repo_root)"
    echo "  Branch: $(current_branch)"
    echo "  Remote: $(remote_url)"
    
    echo ""
    echo "2. Status:"
    if has_changes; then
        echo "  มี uncommitted changes"
    else
        echo "  Working tree clean"
    fi
else
    echo "ไม่ได้อยู่ใน git repo"
fi

echo ""
echo "3. Git log formatting:"
cat << 'EOF'
# One-line log
git log --oneline

# With graph
git log --oneline --graph --all

# Custom format
git log --format="%h %an %ar %s" -10

# Files changed in commit
git show --stat HEAD

# Diff of commit
git show HEAD
EOF

echo ""
echo "4. Git search:"
cat << 'EOF'
# ค้นหา commit ที่มี text
git log --grep="keyword"

# ค้นหาใน code (pickaxe)
git log -S "function_name"

# ค้นหา file ใน history
git log --all --full-history -- "*.py"

# ดูว่าไฟล์เปลี่ยนเมื่อไร
git blame file.py
EOF
```

---

## ขั้นตอนที่ 376: Git Hooks

```bash
#!/usr/bin/env bash
# git_hooks.sh - Git Hooks

echo "=== Git Hooks ==="
echo ""
echo "Git Hooks คือ scripts ที่ทำงานโดยอัตโนมัติเมื่อเกิด git events"
echo ""
echo "Hooks location: .git/hooks/"
echo ""
echo "Client-side hooks:"
cat << 'EOF'
  pre-commit          - ก่อน commit (lint, test)
  prepare-commit-msg  - แก้ commit message
  commit-msg          - validate commit message
  post-commit         - หลัง commit
  pre-push            - ก่อน push
  pre-rebase          - ก่อน rebase
  post-checkout       - หลัง checkout
  post-merge          - หลัง merge
EOF

echo ""
echo "Server-side hooks:"
cat << 'EOF'
  pre-receive     - ก่อน push ถึง server
  update          - สำหรับแต่ละ branch ที่ push
  post-receive    - หลัง push ทั้งหมด
EOF

echo ""
echo "1. pre-commit hook:"
cat << 'HOOK'
#!/usr/bin/env bash
# .git/hooks/pre-commit

set -e

echo "Running pre-commit checks..."

# 1. Code formatting check
if command -v black &>/dev/null; then
    echo "Checking Python formatting..."
    black --check . 2>/dev/null || {
        echo "❌ Python code not formatted. Run: black ."
        exit 1
    }
fi

# 2. Lint check
if command -v shellcheck &>/dev/null; then
    echo "Checking shell scripts..."
    while IFS= read -r -d '' file; do
        shellcheck "$file" || {
            echo "❌ ShellCheck failed: $file"
            exit 1
        }
    done < <(git diff --cached --name-only -z | grep -z '\.sh$')
fi

# 3. No debug statements
if git diff --cached --name-only | grep -E '\.(py|js|ts)$' | xargs grep -l 'debugger\|console\.log\|pdb\.set_trace' 2>/dev/null; then
    echo "❌ Debug statements found!"
    exit 1
fi

# 4. No secrets (basic check)
if git diff --cached | grep -E '(password|secret|api_key|token)\s*=\s*["\x27][^"x27]+["\x27]' -i; then
    echo "❌ Possible secrets in commit!"
    exit 1
fi

echo "✓ Pre-commit checks passed"
HOOK

echo ""
echo "2. commit-msg hook:"
cat << 'HOOK'
#!/usr/bin/env bash
# .git/hooks/commit-msg
# Enforce conventional commits: type(scope): description

MSG_FILE="$1"
MSG=$(cat "$MSG_FILE")

# Pattern: type(scope): description
PATTERN='^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\([a-z-]+\))?: .{3,}'

if [[ ! "$MSG" =~ $PATTERN ]]; then
    echo "❌ Commit message ไม่ตรงรูปแบบ!"
    echo ""
    echo "รูปแบบที่ถูกต้อง:"
    echo "  type(scope): description"
    echo ""
    echo "Types: feat, fix, docs, style, refactor, test, chore"
    echo ""
    echo "Examples:"
    echo "  feat(auth): add OAuth login"
    echo "  fix(api): handle null response"
    echo "  docs: update README"
    exit 1
fi
HOOK

echo ""
echo "3. ติดตั้ง hooks:"
cat << 'INSTALL'
#!/usr/bin/env bash
# install_hooks.sh

HOOKS_DIR=".git/hooks"
HOOKS_SOURCE="./scripts/git-hooks"

install_hook() {
    local hook_name="$1"
    local source="$HOOKS_SOURCE/$hook_name"
    local dest="$HOOKS_DIR/$hook_name"
    
    if [[ ! -f "$source" ]]; then
        echo "Warning: $source not found"
        return
    fi
    
    cp "$source" "$dest"
    chmod +x "$dest"
    echo "✓ Installed: $hook_name"
}

mkdir -p "$HOOKS_DIR"
for hook in pre-commit commit-msg pre-push; do
    install_hook "$hook"
done
INSTALL
```

---

## ขั้นตอนที่ 377: Git Flow Automation

```bash
#!/usr/bin/env bash
# git_flow.sh - Git Flow Automation

echo "=== Git Flow Automation ==="

# ==================== Feature Branch Workflow ====================

# สร้าง feature branch
new_feature() {
    local feature_name="$1"
    local base_branch="${2:-main}"
    
    if [[ -z "$feature_name" ]]; then
        echo "Usage: new_feature <name> [base_branch]"
        return 1
    fi
    
    local branch="feature/${feature_name}"
    
    echo "Creating feature branch: $branch"
    git checkout "$base_branch" 2>/dev/null
    git pull origin "$base_branch" 2>/dev/null
    git checkout -b "$branch"
    echo "✓ Branch created: $branch"
}

# สร้าง release branch
new_release() {
    local version="$1"
    
    echo "Creating release: v$version"
    git checkout main 2>/dev/null
    git pull origin main 2>/dev/null
    git checkout -b "release/v$version"
    
    # Update version file
    echo "$version" > VERSION 2>/dev/null
    echo "✓ Release branch: release/v$version"
}

# Finish feature
finish_feature() {
    local branch
    branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
    
    if [[ ! "$branch" =~ ^feature/ ]]; then
        echo "ไม่ได้อยู่ใน feature branch"
        return 1
    fi
    
    local feature_name="${branch#feature/}"
    
    echo "Finishing feature: $feature_name"
    git checkout main 2>/dev/null
    git merge --no-ff "$branch" -m "feat: merge feature/$feature_name"
    git branch -d "$branch"
    echo "✓ Feature merged"
}

echo "Functions available:"
echo "  new_feature <name> [base]  - สร้าง feature branch"
echo "  new_release <version>      - สร้าง release branch"
echo "  finish_feature             - merge feature กลับ main"
echo ""

# ==================== Automated Commit ====================
echo "Git commit conventions:"
cat << 'EOF'
# Conventional Commits
# format: type(scope): description

# Types:
# feat     - new feature
# fix      - bug fix
# docs     - documentation
# style    - formatting
# refactor - code restructure
# test     - add/update tests
# chore    - maintenance
# perf     - performance
# ci       - CI/CD
# build    - build system
# revert   - revert commit

# Examples:
git commit -m "feat(auth): add JWT authentication"
git commit -m "fix(api): handle connection timeout"
git commit -m "docs: update API documentation"
git commit -m "chore(deps): update dependencies"
git commit -m "refactor: extract helper functions"
EOF

echo ""
echo "Auto-generate commit message:"
auto_commit_msg() {
    local type="${1:-chore}"
    local files
    files=$(git diff --staged --name-only 2>/dev/null | tr '\n' ' ')
    
    if [[ -z "$files" ]]; then
        echo "No staged files"
        return 1
    fi
    
    local scope
    scope=$(echo "$files" | awk '{print $1}' | xargs dirname 2>/dev/null | sed 's|./||' | head -1)
    
    echo "$type($scope): update $(echo $files | wc -w | tr -d ' ') file(s)"
}
```

---

## ขั้นตอนที่ 378: Git Statistics และ Reports

```bash
#!/usr/bin/env bash
# git_stats.sh - Git Statistics

echo "=== Git Statistics ==="

# ต้องอยู่ใน git repo
if ! git rev-parse --is-inside-work-tree &>/dev/null; then
    echo "ไม่ได้อยู่ใน git repo"
    exit 0
fi

echo "1. Commit statistics:"
echo "Total commits: $(git rev-list --count HEAD 2>/dev/null || echo 0)"

echo ""
echo "2. Commits per author:"
git log --format="%an" 2>/dev/null | sort | uniq -c | sort -rn | head -10

echo ""
echo "3. Commits per day of week:"
git log --format="%ad" --date=format:"%A" 2>/dev/null | sort | uniq -c | sort -rn

echo ""
echo "4. Files changed:"
echo "Total files: $(git ls-files 2>/dev/null | wc -l)"

echo ""
echo "5. Lines of code:"
git ls-files 2>/dev/null | xargs wc -l 2>/dev/null | tail -1 | awk '{print "Total lines:", $1}'

echo ""
echo "6. Recent activity:"
git log --oneline -10 2>/dev/null

echo ""
echo "7. Branch info:"
echo "Local branches:"
git branch 2>/dev/null | head -10

echo ""
echo "8. Changed files in last commit:"
git diff --stat HEAD~1 HEAD 2>/dev/null | head -10 || echo "(no previous commit)"

echo ""
echo "9. Tag history:"
git tag --sort=-creatordate 2>/dev/null | head -5 || echo "(no tags)"
```

---

## ขั้นตอนที่ 379: Workshop - Git Release Manager

```bash
#!/usr/bin/env bash
# release_manager.sh - Workshop: Git Release Manager

set -euo pipefail

echo "=== Git Release Manager ==="
echo ""

# ==================== Release Manager ====================

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log_info()    { echo -e "${GREEN}[INFO]${NC} $*"; }
log_warn()    { echo -e "${YELLOW}[WARN]${NC} $*"; }
log_error()   { echo -e "${RED}[ERROR]${NC} $*" >&2; }

# ตรวจสอบ git repo
require_git_repo() {
    if ! git rev-parse --is-inside-work-tree &>/dev/null; then
        log_error "ไม่ได้อยู่ใน git repository"
        exit 1
    fi
}

# ดู current version จาก tag
get_current_version() {
    git describe --tags --abbrev=0 2>/dev/null | sed 's/^v//' || echo "0.0.0"
}

# Bump version
bump_version() {
    local version="$1"
    local bump_type="${2:-patch}"  # major, minor, patch
    
    IFS='.' read -ra parts <<< "$version"
    local major="${parts[0]:-0}"
    local minor="${parts[1]:-0}"
    local patch="${parts[2]:-0}"
    
    case "$bump_type" in
        major) echo "$((major+1)).0.0" ;;
        minor) echo "${major}.$((minor+1)).0" ;;
        patch) echo "${major}.${minor}.$((patch+1))" ;;
        *)     echo "$version" ;;
    esac
}

# Generate changelog
generate_changelog() {
    local from_tag="${1:-}"
    local to_ref="${2:-HEAD}"
    
    echo "## Changelog"
    echo ""
    
    local git_log_args=()
    if [[ -n "$from_tag" ]]; then
        git_log_args=("${from_tag}..${to_ref}")
    else
        git_log_args=("-n" "20" "$to_ref")
    fi
    
    # Features
    local features
    features=$(git log "${git_log_args[@]}" --oneline --grep="^feat" 2>/dev/null || true)
    if [[ -n "$features" ]]; then
        echo "### New Features"
        echo "$features" | while read -r line; do
            echo "- $line"
        done
        echo ""
    fi
    
    # Bug fixes
    local fixes
    fixes=$(git log "${git_log_args[@]}" --oneline --grep="^fix" 2>/dev/null || true)
    if [[ -n "$fixes" ]]; then
        echo "### Bug Fixes"
        echo "$fixes" | while read -r line; do
            echo "- $line"
        done
        echo ""
    fi
    
    # Other changes
    local other
    other=$(git log "${git_log_args[@]}" --oneline 2>/dev/null | \
        grep -v "^.*feat\|^.*fix" | head -10 || true)
    if [[ -n "$other" ]]; then
        echo "### Other Changes"
        echo "$other" | while read -r line; do
            echo "- $line"
        done
        echo ""
    fi
}

# Create release
create_release() {
    local version="$1"
    local release_notes="${2:-}"
    
    require_git_repo
    
    local tag="v${version}"
    log_info "Creating release $tag..."
    
    # Check for uncommitted changes
    if ! git diff-index --quiet HEAD -- 2>/dev/null; then
        log_warn "มี uncommitted changes"
    fi
    
    # Generate changelog
    local current_version
    current_version=$(get_current_version)
    
    local changelog
    changelog=$(generate_changelog "v${current_version}" 2>/dev/null || generate_changelog)
    
    # Create annotated tag
    local tag_message="Release $tag\n\n${changelog}"
    
    echo ""
    echo "Release Notes:"
    echo "=============="
    echo -e "$changelog"
    
    log_info "Tag: $tag"
    log_info "Release created (demo - ไม่ได้สร้างจริง)"
    
    # In real usage:
    # git tag -a "$tag" -m "$tag_message"
    # git push origin "$tag"
}

# Main
require_git_repo
current_ver=$(get_current_version)
log_info "Current version: $current_ver"

echo ""
echo "Version bumping examples:"
echo "  patch: $current_ver → $(bump_version "$current_ver" "patch")"
echo "  minor: $current_ver → $(bump_version "$current_ver" "minor")"
echo "  major: $current_ver → $(bump_version "$current_ver" "major")"

echo ""
echo "Changelog preview:"
generate_changelog 2>/dev/null | head -20 || echo "(ไม่มี commits)"

echo ""
log_info "Release manager demo completed"
```

---

## ขั้นตอนที่ 380: สรุป Part 16 - Git Automation

```bash
#!/usr/bin/env bash
# summary_git.sh

echo "=== สรุป Git Automation ==="
echo ""
echo "Git Automation ครอบคลุม:"
echo ""
echo "1. Git utilities:"
echo "   is_git_repo, current_branch, has_changes"
echo ""
echo "2. Git Hooks:"
echo "   pre-commit (lint, test)"
echo "   commit-msg (format validation)"
echo "   pre-push"
echo ""
echo "3. Workflows:"
echo "   Feature branch workflow"
echo "   Git Flow"
echo "   Conventional commits"
echo ""
echo "4. Automation:"
echo "   Auto changelog generation"
echo "   Release management"
echo "   Statistics reports"
echo ""
echo "Next: Part 17 - Docker Integration"
```

---

## สรุป

| หัวข้อ | Steps |
|--------|-------|
| Git basics review | 374 |
| Git automation | 375 |
| Git hooks | 376 |
| Git flow | 377 |
| Git statistics | 378 |
| Workshop | 379 |

**ขั้นตอนต่อไป**: Part 17 - Docker Integration
