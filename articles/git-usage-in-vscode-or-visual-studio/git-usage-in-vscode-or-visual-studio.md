# Git Usage in VS Code or Visual Studio

## Installation

Always make sure you have the latest version of Git installed (`git –version`).  When installed, it should add itself to the path.

   [https://git-scm.com/downloads](https://git-scm.com/downloads)

Optionally, you may also want to install the GitHub desktop at [https://desktop.github.com](https://desktop.github.com), but all needed features can be executed from the command line or from within VS Code of Visual Studio without the desktop installed.

## Branching Recommendations

The below branching recomendation follow the [Git-Flow](http://nvie.com/posts/a-successful-git-branching-model) pattern.  This pattern has long been the defacto standard allowing developers to move between projects or companies being familiar with this standardized workflow.  Regardless of what branch type is being created, they should all be pushed to the server to share with other team members, and to protect from laptop loss/theft, or hard drive failure.  This also allows the server to keep track of who is working on what, and the level of activity each feature has.

### Permanent branches

* `master` and `develop` are the only permanent branches
* `master` always reflects a production ready state
* `develop` always reflect the latest delivered development changes for the next release
   * `develop` comitts usually trigger automatic builds

### Feature Branches

* Used for development of a single new feature(story, topic, etc.), with separate feature branches existing in parallel.
* Named `feature-<name>`
* Must branch from `develop`
* Must merge back into `develop` and then be deleted once feature is complete
   * Some teams may wish to only allow features to be merged back into the `develop` branch using a `pull request`
* If the branch has been open for too long and you feel it’s getting out of sync with the `develop` branch, you can merge `develop` back into your feature branch and keep going.

### Release Branch

* Used for QA, staging, and final release to production for a set of features.  Any bugs found during QA or user acceptance testing will be fixed within this branch.
* Named `release-<version>`.  Version in the format #.#.#
   * The right most number is incremented for a bug fix where no new features or breaking changes are being added
   * The middle number is incremented for a new features where no breaking changes are being added
   * The left most number is only incremented when breaking changes are being added
* Must branch from `develop`
* Must merge back into `master` and `develop`, then be deleted once the release has made it through QA, staging, and finally production.  The master commit should be commented and/or tagged with the version.

### Hotfix Branches

* Used for fixing of bugs found in production.  This branch is treated like a relase branch while it exists, meaning QA or staging environments may be optionally deployed from this branch to ensure the fix is ready for release.
* Named `hotfix-<name>`.  Version in the format #.#.#
   * Only the right most number would be incremented for a hotfix
* Must branch from `master`
* Must merge back into `master` and `develop`, then be deleted once the fix has made it through QA, staging, and finally production.  The master commit should be commented and/or tagged with the new incremented version.
   * When a `release` branch currently exists, the `hotfix` changes need to be merged into that `release` branch, instead of `develop`.  Once the release is deployed, that process will ensure the bug fix makes it back to the `develop` branch.

<small>Git-Flow Model - by Vincent Driessen</small>
![Git-Flow Model](git-model@2x.png "Git-Flow Model - by Vincent Driessen")

Smaller projects, or projects deploying at a much quicker cadence may wish to do a hybrid approach between Git-Flow and [GitHub-Flow](http://scottchacon.com/2011/08/31/github-flow.html).  It is strongly recommended to review and understand both flows.

## Tasks

Many of the below commands can be done from a GUI interface like VS Code, Visual Studio, or GitHub desktop, but will be shown here in their command line form.

### Create new repository for existing project

1. Create new repository in GitHub or VSTS
2. Clone new repository onto PC

```cmd
cd \source
git clone https://<repository-path>/<new-repository-name>
```

3. Create new project or copy existing project into newly cloned folder

### Create a new project from existing template project

```cmd
cd \source
git clone https://<repository-path>/<template-repository-name>.git <new-repository-name>
cd \<new-repository-name>
rd .git /s /q
git init
git add .
git commit –m “Initial commit”
git tag -a 0.1
git remote add origin https://<repository-path>/<new-repository-name>.git
git push -u origin master
```

### Create a new "develop" branch from "master"

```cmd
cd <new-repository-name>
git checkout master
git checkout -b develop
git push –u origin develop
```

### Commit changes from “develop”

```cmd
git add *
git commit –m “commit message”
git pull
git push origin develop
```

### Merge changes from “develop” back into “master”

if merging a feature branch, add param `--no-ff` to the merge command to ensure the historical existence of a feature branch and groups together all commits that together added the feature

```cmd
git checkout master
git pull
git merge develop
git tag -a <version>
git push origin master
```

### View any merge conflicts

```cmd
git diff
git diff --base <filename>
git diff <sourcebranch> <targetbranch>
```

### Delete “develop” branch

```cmd
git branch develop –d
git push origin :develop
```

## References

* [Using Git with Microsoft VS Team Foundation Server or VS Team Server Online](https://channel9.msdn.com/Events/Ignite/2015/BRK3709)
* [Git Branching and Policies in Team Foundation Server 2015](https://channel9.msdn.com/Events/Visual-Studio/Visual-Studio-2015-Final-Release-Event/Git-Branches-and-Policies-in-Team-Foundation-Server-2015)
* [Git-Flow](http://nvie.com/posts/a-successful-git-branching-model/)
   * [Git-Flow - Additional Reference](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)
* [GitHub-Flow](http://scottchacon.com/2011/08/31/github-flow.html)
* [Basic Git Commands](https://confluence.atlassian.com/bitbucketserver/basic-git-commands-776639767.html)