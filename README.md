


## Book 0: "Machine Learning: A Probabilistic Perspective" (2012)

See [this link](https://probml.github.io/pml-book/book0.html)

<!--
See [this link](https://probml.github.io/pml-book/pml0/book0.html)
-->

## Book 1: "Probabilistic Machine Learning: An Introduction" (2022)

See [this link](https://probml.github.io/pml-book/book1.html)


## Book 2: "Probabilistic Machine Learning: Advanced Topics" (2023)

See [this link](https://probml.github.io/pml-book/book2.html)



#!/bin/bash
set -eu

echo "Detecting triggering repo and branch..."

TRIGGERED_REPO=""
TRIGGERED_BRANCH=""
TRIGGERED_REVISION=""
TRIGGER_SOURCE=""

TARGET_REPOS="repoa repob repoc repod"

lower() {
  echo "$1" | tr '[:upper:]' '[:lower:]'
}

is_target_repo() {
  repo="$(lower "$1")"
  for target in $TARGET_REPOS; do
    [ "$repo" = "$target" ] && return 0
  done
  return 1
}

get_git_branch() {
  git rev-parse --abbrev-ref HEAD 2>/dev/null || true
}

get_git_revision() {
  git rev-parse HEAD 2>/dev/null || true
}

BUILD_REASON="${bamboo_buildReason:-}"
BUILD_REASON_LC="$(lower "$BUILD_REASON")"

echo "Build reason: ${BUILD_REASON:-N/A}"

# ------------------------------------------------------------
# 1. Manual build: assume repoD and read branch from checked-out Git repo
# ------------------------------------------------------------

if echo "$BUILD_REASON_LC" | grep -q "manual"; then
  echo "Manual build detected. Reading repoD branch from Git workspace."

  TRIGGERED_REPO="repod"
  TRIGGERED_BRANCH="$(get_git_branch)"
  TRIGGERED_REVISION="$(get_git_revision)"
  TRIGGER_SOURCE="manual-repod-git-branch"
fi

# ------------------------------------------------------------
# 2. Repository-triggered build variables
# ------------------------------------------------------------

if [ -z "$TRIGGERED_BRANCH" ]; then
  if [ -n "${bamboo_repository_name:-}" ] && [ -n "${bamboo_repository_branch_name:-}" ]; then
    repo_lc="$(lower "$bamboo_repository_name")"

    if is_target_repo "$repo_lc"; then
      TRIGGERED_REPO="$repo_lc"
      TRIGGERED_BRANCH="$bamboo_repository_branch_name"
      TRIGGERED_REVISION="${bamboo_repository_revision_number:-}"
      TRIGGER_SOURCE="bamboo-repository-trigger"
    fi
  fi
fi

# ------------------------------------------------------------
# 3. Automatic build fallback: scan changed plan repositories
# ------------------------------------------------------------

if [ -z "$TRIGGERED_BRANCH" ]; then
  echo "Scanning bamboo_planRepository variables..."

  env | grep -E '^bamboo_planRepository_[0-9]+_name=' | sort -V | while IFS='=' read -r var repo_name
  do
    repo_index="$(echo "$var" | sed -E 's/^bamboo_planRepository_([0-9]+)_name$/\1/')"

    branch_var="bamboo_planRepository_${repo_index}_branchName"
    revision_var="bamboo_planRepository_${repo_index}_revision"
    changed_var="bamboo_planRepository_${repo_index}_changed"

    branch="$(eval echo "\${$branch_var:-}")"
    revision="$(eval echo "\${$revision_var:-}")"
    changed="$(eval echo "\${$changed_var:-}")"
    changed_lc="$(lower "$changed")"
    repo_lc="$(lower "$repo_name")"

    echo "Repo #$repo_index: $repo_name | branch=${branch:-N/A} | changed=${changed:-N/A}"

    if is_target_repo "$repo_lc" && [ "$changed_lc" = "true" ]; then
      {
        echo "TRIGGERED_REPO=$repo_lc"
        echo "TRIGGERED_BRANCH=$branch"
        echo "TRIGGERED_REVISION=$revision"
        echo "TRIGGER_SOURCE=planRepository-changed"
      } > trigger-info.env.tmp
      break
    fi
  done

  if [ -f trigger-info.env.tmp ]; then
    . ./trigger-info.env.tmp
    rm -f trigger-info.env.tmp
  fi
fi

# ------------------------------------------------------------
# 4. Final repoD fallback: read Git branch from repoD workspace
# ------------------------------------------------------------

if [ -z "$TRIGGERED_BRANCH" ]; then
  echo "No triggering repo detected. Assuming repoD/default repository."
  TRIGGERED_REPO="repod"
  TRIGGERED_BRANCH="$(get_git_branch)"
  TRIGGERED_REVISION="$(get_git_revision)"
  TRIGGER_SOURCE="repod-git-fallback"
fi

if [ -z "$TRIGGERED_BRANCH" ] || [ "$TRIGGERED_BRANCH" = "HEAD" ]; then
  echo "ERROR: Could not determine repoD branch from Git."
  echo "Make sure repoD checkout happens before this script task."
  exit 1
fi

echo "=================================="
echo "Trigger Result"
echo "=================================="
echo "Repo    : $TRIGGERED_REPO"
echo "Branch  : $TRIGGERED_BRANCH"
echo "Revision: ${TRIGGERED_REVISION:-N/A}"
echo "Source  : $TRIGGER_SOURCE"
echo "=================================="

cat > trigger-info.env <<EOF
TRIGGERED_REPO=$TRIGGERED_REPO
TRIGGERED_BRANCH=$TRIGGERED_BRANCH
TRIGGERED_REVISION=$TRIGGERED_REVISION
TRIGGER_SOURCE=$TRIGGER_SOURCE
EOF
