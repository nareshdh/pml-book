


## Book 0: "Machine Learning: A Probabilistic Perspective" (2012)

See [this link](https://probml.github.io/pml-book/book0.html)

<!--
See [this link](https://probml.github.io/pml-book/pml0/book0.html)
-->

## Book 1: "Probabilistic Machine Learning: An Introduction" (2022)

See [this link](https://probml.github.io/pml-book/book1.html)


## Book 2: "Probabilistic Machine Learning: Advanced Topics" (2023)

See [this link](https://probml.github.io/pml-book/book2.html)



#!/usr/bin/env bash
set -euo pipefail

TARGET_REPOS=("repoa" "repob" "repoc" "repod")

normalize() {
  echo "$1" | tr '[:upper:]' '[:lower:]'
}

is_target_repo() {
  local repo
  repo="$(normalize "$1")"

  for target in "${TARGET_REPOS[@]}"; do
    if [[ "$repo" == "$target" ]]; then
      return 0
    fi
  done

  return 1
}

TRIGGERED_REPO=""
TRIGGERED_BRANCH=""
TRIGGERED_REVISION=""
TRIGGER_SOURCE=""

echo "Detecting triggering repo and branch..."

# 1. Manual override via Bamboo plan variables
if [[ -n "${bamboo_trigger_branch:-}" ]]; then
  TRIGGERED_BRANCH="${bamboo_trigger_branch}"
  TRIGGERED_REPO="${bamboo_trigger_repo:-repod}"
  TRIGGERED_REVISION=""
  TRIGGER_SOURCE="manual-plan-variable"

  echo "Using manual Bamboo variables:"
  echo "  trigger_repo   = $TRIGGERED_REPO"
  echo "  trigger_branch = $TRIGGERED_BRANCH"
fi

# 2. Repo-triggered build variables
if [[ -z "$TRIGGERED_BRANCH" && -n "${bamboo_repository_branch_name:-}" ]]; then
  TRIGGERED_REPO="${bamboo_repository_name:-unknown}"
  TRIGGERED_BRANCH="${bamboo_repository_branch_name}"
  TRIGGERED_REVISION="${bamboo_repository_revision_number:-}"
  TRIGGER_SOURCE="bamboo-repository-trigger"

  echo "Using Bamboo repository trigger variables."
fi

# 3. Fallback: scan plan repositories
if [[ -z "$TRIGGERED_BRANCH" ]]; then
  echo "Scanning bamboo_planRepository variables..."

  while IFS='=' read -r var repo_name; do
    repo_index="$(echo "$var" | sed -E 's/^bamboo_planRepository_([0-9]+)_name$/\1/')"

    branch_var="bamboo_planRepository_${repo_index}_branchName"
    revision_var="bamboo_planRepository_${repo_index}_revision"
    changed_var="bamboo_planRepository_${repo_index}_changed"

    branch="${!branch_var:-}"
    revision="${!revision_var:-}"
    changed="${!changed_var:-}"

    echo "Repo #$repo_index: $repo_name | branch=${branch:-N/A} | changed=${changed:-N/A}"

    if is_target_repo "$repo_name" && [[ "${changed,,}" == "true" ]]; then
      TRIGGERED_REPO="$repo_name"
      TRIGGERED_BRANCH="$branch"
      TRIGGERED_REVISION="$revision"
      TRIGGER_SOURCE="planRepository-changed"
      break
    fi
  done < <(env | grep -E '^bamboo_planRepository_[0-9]+_name=' | sort -V)
fi

# 4. Final fallback for manual repoD build
if [[ -z "$TRIGGERED_BRANCH" ]]; then
  TRIGGERED_REPO="${bamboo_trigger_repo:-repod}"
  TRIGGERED_BRANCH="${bamboo_planRepository_branchName:-${bamboo_repository_branch_name:-main}}"
  TRIGGERED_REVISION=""
  TRIGGER_SOURCE="default-manual-fallback"

  echo "No trigger repo found. Assuming manual repoD build."
fi

if [[ -z "$TRIGGERED_BRANCH" ]]; then
  echo "ERROR: Could not determine branch."
  env | sort | grep -Ei 'bamboo_(repository|planRepository|branch|trigger|buildReason|revision)' || true
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

cat trigger-info.env
