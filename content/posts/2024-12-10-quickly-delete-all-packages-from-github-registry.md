+++
author = "hannibal"
categories = ["ghcr", "automation", "script"]
date = "2024-12-10T01:01:00+01:00"
title = "Quickly delete all packages from GitHub registry using a blob"
url = "/2024/12/10/quickly-delete-all-packages-from-github-registry"
comments = true
+++

# Script to remove GitHub registry images

Hey, though I quickly just share a script to get rid of packages you don't want anymore
from your GitHub registry.

```bash
#!/bin/bash

set -e

# Variables
OWNER=$1  # Replace with your GitHub username or organization name
PACKAGE_GLOB=$2  # Glob pattern for package names passed as an argument (e.g., "package*" to match all)

# makes sure that important packages are not removed by accident
contains() {
  found=1
  array=(place-very-important-packages-here-to-make-sure-they-are-not-deleted)
  for v in "${array[@]}"
  do
    if [[ "$1" == "$v" ]]; then
      echo "$v found not deleting"
      found=0
      break
    fi
  done
}

# Function to delete a package version
delete_package_version() {
    contains "$1"

    # echo "found: $found"
    if [[ $found -eq 1 ]]; then
      name=${1//\//%2F}
      echo "deleting package with name: $name"
      gh api -X DELETE "/user/packages/container/$name" --silent
    fi
}

# Fetch the list of all available packages for the user/organization
echo "Fetching packages matching the glob pattern '$PACKAGE_GLOB'..."

# Fetch all package names and filter with globbing
# Change `/users/` to `organization` if you are looking at organization packages.
ALL_PACKAGES=$(gh api "/users/$OWNER/packages?package_type=container" --jq '.[].name' --paginate)
MATCHED_PACKAGES=$(echo "$ALL_PACKAGES" | grep "$PACKAGE_GLOB")
if [[ -z "$MATCHED_PACKAGES" ]]; then
    echo "No packages found matching the pattern '$PACKAGE_GLOB'."
    exit 1
fi

# echo "Deleting the following packages: ${MATCHED_PACKAGES}"

# Loop through matched packages and delete them
SAVEIFS=$IFS   # Save current IFS (Internal Field Separator)
IFS=$'\n'      # Change IFS to newline char
packages=($MATCHED_PACKAGES) # split the `names` string into an array by the same name
IFS=$SAVEIFS   # Restore original IFS

for (( i=0; i<${#packages[@]}; i++ ))
do
  delete_package_version "${packages[$i]}"
done

echo "All matching packages deleted!"

```

That is all.

Thanks!