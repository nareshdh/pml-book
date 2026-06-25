


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

PYTHON_SCRIPT="./extract_jira_info.py"

# Bamboo linked repo position mapping
# Must match Bamboo linked repository order
REPOA_POS=1
REPOB_POS=2
REPOC_POS=3
REPOD_POS=4

TRIGGER_REPO=""
TRIGGER_BRANCH=""

get_var() {
  printenv "$1" 2>/dev/null || true
}

get_branch() {
  local pos="$1"

  local branch
  branch="$(get_var "bamboo_planRepository_${pos}_branchDisplayName")"

  if [[ -z "$branch" ]]; then
    branch="$(get_var "bamboo_planRepository_${pos}_branchName")"
  fi

  echo "$branch"
}

check_repo_changed() {
  local repo_name="$1"
  local pos="$2"

  local revision previous branch

  revision="$(get_var "bamboo_planRepository_${pos}_revision")"
  previous="$(get_var "bamboo_planRepository_${pos}_previousRevision")"
  branch="$(get_branch "$pos")"

  if [[ -n "$revision" && -n "$previous" && "$revision" != "$previous" ]]; then
    TRIGGER_REPO="$repo_name"
    TRIGGER_BRANCH="$branch"
    return 0
  fi

  return 1
}

echo "Detecting triggering repo and branch..."

if [[ "${bamboo_triggerReason_key:-}" == *"ManualBuildTriggerReason"* ]]; then
  echo "Manual build detected."

  TRIGGER_REPO="repod"
  TRIGGER_BRANCH="$(get_branch "$REPOD_POS")"
else
  check_repo_changed "repoa" "$REPOA_POS" || \
  check_repo_changed "repob" "$REPOB_POS" || \
  check_repo_changed "repoc" "$REPOC_POS" || \
  check_repo_changed "repod" "$REPOD_POS" || true
fi

if [[ -z "$TRIGGER_REPO" || -z "$TRIGGER_BRANCH" ]]; then
  echo "ERROR: Could not detect triggering repo or branch."
  echo
  echo "Debug variables:"
  printenv | sort | grep -E '^bamboo_(triggerReason|planRepository)' || true
  exit 1
fi

echo "Trigger repo   : $TRIGGER_REPO"
echo "Trigger branch : $TRIGGER_BRANCH"

python3 "$PYTHON_SCRIPT" \
  --repo "$TRIGGER_REPO" \
  --branch "$TRIGGER_BRANCH"
