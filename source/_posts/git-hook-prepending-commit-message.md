---
title: Prefix the commit message with the Jira ticket number using a git hook
date: 2018-12-18 12:00:00
featured_image: featured-image.jpg
desc: Set up git hook for prepending the commit message with the Atlassian Jira ticket number
tags:
- Git
- Git hook
- Jira
---
The goal is to provide a way to automate the process of specifying correct ticket numbers in commit messages in order for commits to be well distinguished and linked to the corresponding tickets in the issue tracker.
<!--more-->
Usually, git hooks are stored in .git/hooks directory inside the project repository. These files should be named according to the name of the hook, should not have any extension and should be executable.

These requirements along with the list of available hooks are described in the [git documentation](https://git-scm.com/docs/githooks). However it can be tedious to set up these hooks for each project, so you can specify a global directory for the hooks like this:
```bash
git config --global core.hooksPath '~/.git-templates/hooks'
```
The directory name can be different.

The appropriate hook type for modifying the commit message is prepare-commit-msg as it ensures that the commit message will be edited no matter whether it was passed to the -m console flag or typed inside the text editor.

So we need to create a file `~/.git-templates/hooks/prepare-commit-msg` with the following Bash script:
```bash
#!/bin/bash
# Include any branches for which you wish to disable this script
if [ -z "$BRANCHES_TO_SKIP" ]; then
  BRANCHES_TO_SKIP=(master develop)
fi
# Get the current branch name
BRANCH_NAME=$(git symbolic-ref --short HEAD)
# Ticket name is retrieved as a string between the first slash and the second hyphen
TICKET_NAME=$(echo $BRANCH_NAME | sed -e 's/^[^\/]*\/\([^-]*-[^-]*\)-.*/\1/')

BRANCH_EXCLUDED=$(printf "%s\n" "${BRANCHES_TO_SKIP[@]}" | grep -c "^$BRANCH_NAME$")
ALREADY_IN_MSG=$(grep -c "$TICKET_NAME" $1)
# If it isn't excluded or already in commit message, prepend the ticket name to the given message
if [ -n "$BRANCH_NAME" ] && ! [[ "$BRANCH_EXCLUDED" -eq 1 ]] && ! [[ "$ALREADY_IN_MSG" -eq 1 ]]; then
    sed -i.bak -e "1s/^/$TICKET_NAME /" "$1"
fi
```
In this script, we assume that the ticket number is a project name separated with an auto-incremental number by a hyphen and it exists in the name of the branch. This assumption is valid most of the times as it is a usual naming convention for the Atlassian Stack(Jira, Bitbucket etc) 

During the execution of the script the ticket number will be parsed from the name of the branch upon commit creation and added to the message. This script will be skipped for master and develop branches and it will check if the ticket number is already inside the message to restrict it to be added twice.

If in some case there is no need to use this hook it can be skipped on commit using --no-verify flag.
