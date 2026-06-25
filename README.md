


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

echo "=============================================="
echo "Detect Bamboo Triggering Repository and Branch"
echo "=============================================="

TARGET_REPOS=("repoa" "repob" "repoc")

TRIGGERED_REPO=""
TRIGGERED_BRANCH=""
TRIGGERED_REVISION=""
TRIGGERED_INDEX=""

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

echo
echo "Scanning Bamboo repository variables..."
echo

while IFS='=' read -r var repo_name; do
  repo_index="$(echo "$var" | sed -E 's/^bamboo_planRepository_([0-9]+)_name$/\1/')"

  branch_var="bamboo_planRepository_${repo_index}_branchName"
  revision_var="bamboo_planRepository_${repo_index}_revision"
  changed_var="bamboo_planRepository_${repo_index}_changed"

  branch="${!branch_var:-}"
  revision="${!revision_var:-}"
  changed="${!changed_var:-}"

  echo "Repository #${repo_index}"
  echo "  Name     : ${repo_name}"
  echo "  Branch   : ${branch:-N/A}"
  echo "  Revision : ${revision:-N/A}"
  echo "  Changed  : ${changed:-N/A}"
  echo

  if is_target_repo "$repo_name"; then
    if [[ "$changed" == "true" || "$changed" == "True" || "$changed" == "TRUE" ]]; then
      TRIGGERED_REPO="$repo_name"
      TRIGGERED_BRANCH="$branch"
      TRIGGERED_REVISION="$revision"
      TRIGGERED_INDEX="$repo_index"
      break
    fi
  fi
done < <(env | grep -E '^bamboo_planRepository_[0-9]+_name=' | sort -V)

if [[ -z "$TRIGGERED_REPO" ]]; then
  echo "No repository had bamboo_planRepository_N_changed=true."
  echo "Trying fallback using bamboo_buildReason..."
  echo

  BUILD_REASON="${bamboo_buildReason:-}"

  for target in "${TARGET_REPOS[@]}"; do
    if echo "$BUILD_REASON" | grep -qi "$target"; then
      TRIGGERED_REPO="$target"
      break
    fi
  done
fi

if [[ -z "$TRIGGERED_REPO" ]]; then
  echo "ERROR: Could not determine which repository triggered this build."
  echo
  echo "Debug Bamboo variables:"
  env | grep -E '^bamboo_(planRepository|repository|buildReason)' | sort || true
  exit 1
fi

if [[ -z "$TRIGGERED_BRANCH" ]]; then
  echo "Triggered repository found, but branch was empty."
  echo "Trying fallback branch variables..."

  TRIGGERED_BRANCH="${bamboo_repository_branch_name:-${bamboo_planRepository_branchName:-}}"
fi

if [[ -z "$TRIGGERED_BRANCH" ]]; then
  echo "ERROR: Could not determine triggering branch."
  exit 1
fi

echo "=============================================="
echo "Trigger Detection Result"
echo "=============================================="
echo "Triggered Repo     : $TRIGGERED_REPO"
echo "Triggered Branch   : $TRIGGERED_BRANCH"
echo "Triggered Revision : ${TRIGGERED_REVISION:-N/A}"
echo "Repository Index   : ${TRIGGERED_INDEX:-N/A}"
echo "=============================================="

cat > trigger-info.env <<EOF
TRIGGERED_REPO=$TRIGGERED_REPO
TRIGGERED_BRANCH=$TRIGGERED_BRANCH
TRIGGERED_REVISION=$TRIGGERED_REVISION
TRIGGERED_INDEX=$TRIGGERED_INDEX
EOF

echo
echo "Saved trigger-info.env"
cat trigger-info.env
