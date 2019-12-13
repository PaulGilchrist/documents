# GIT Branching Recommendations

The 3 most commonly used branching methodologies include [Git-Flow](https://nvie.com/posts/a-successful-git-branching-model/), [GitHub-Flow](https://githubflow.github.io/), and [Microsoft's Trunk Based Branching](https://docs.microsoft.com/en-us/azure/devops/repos/git/git-branching-guidance?view=azure-devops).  Git-Flow is designed for very large projects with scheduled release cycles, but is also the most complex.  GitHub-Flow is the simplest, but designed for continuous deployment at the story vs sprint level with more of a roll-forward than rollback approach.  Microsoft's Trunk Based Branching methodology (also called [Release-Flow](https://docs.microsoft.com/en-us/azure/devops/learn/devops-at-microsoft/release-flow)) is more complex that GitHub-Flow but simpler than Git-Flow, while still supporting enterprise sized applications with sprint based deployment cycles.  None of these methodologies directly addresses our requirements for 4 or more permanent environments (dev, qa, staging, and production), but Microsoft's trunk based branching method comes the closest and is the one recommended.

Microsoft has a proven track record, using and recommending GIT for all their applications (since 2012).  Microsoft owns the worlds largest GIT repository (Windows), and also owns GitHub, the world's largest GIT repository.  This branching methodology is based on a single trunk (named master) which auto-deploys through CI/CD to the development environment.  

The `master` branch represents our development environment, and is for code considered complete or incomplete (with deactivated feature flag) when needed by other features.  Master is kept up to date with all completed code, and always buildable.  This allows for fewer merge conflicts, easy code reviews, and is simpler and faster to ship.  One additional long term branch exists for each environment and goes away if or when environment goes away.  The `qa` branch is a merge from `master` and may be cherry-picked if stories need to be held back.  The `staging` branch is always merged from `qa`, and any release is a merge from `staging`.

Release branches are created with limited lifespan, and are what end up in production (not `master`).  Release branches should be named with version number and later, old versioned releases can be removed when no longer supported.

None of these branches ever merge back into `master`.  All other branches such as stories, bug fixes, etc. are temporary.  Story branches (sometimes referred to as feature branches) should be for small, simple changes, getting them back into the `master` branch as soon as complete and tested.  Story branches are created from `master` and merge back into `master` through pull requests.  A story is not merged back to master until it complete and fully unit tested.  In a scenario where a story is not complete or not ready for a higher environment, but part of it is needed as a dependency for other stories, it can be merged back into master using a Feature Flag (additional reference) set inactive.  In a scenario where a story is complete but not ready for a higher environment, and not needed as a dependency for other stories, it can be merged back into `master` without using a feature flag, and later cherry-picked out when going to the higher environment that is not ready for this story.  This can happen when another team is not ready to integrate with the changes implemented by this story until a later sprint, but the story had already been merged back into `master`.  Version control capabilities can also be used to handle both of these scenarios.

Pull requests include a code review with a recommended 2 minimum reviewers approval.  An approved pull request should be squash merged back into `master` to keep it clean and only containing the final commit of each story.  

When a production bug occurs, a hot-fix branch is created from the current production release branch and when complete, merged back into that same release branch and any newer release branches (if any exist).  A cherry pick button appears after pull requests are completed allowing it to also be merged into the `master` branch.  Bugs found in lower environments should be treated similarly.  Azure DevOps is optimized for this workflow and guides the developer as follows:

* Create branch from work item
* Create pull request recommended after new commit to origin branch
* Cherry pick button appears after pull request is completed

If reviewing Microsoft's branching documentation, not that the 2018 document discusses merging from/to master first then cherry-picking into release, but in the 2019 documentation this was changed to merging from/to release first then cherry-picking into `master`.

## Appendix

### Key Best Practices

* Stay as closely aligned to a industry commonly used branching strategy as possible, only deviating when absolutely necessary.
* Always first push any new branch back into the same branch it was originally created from before merging or cherry-picking into other branches
* Move only up the environment stack.  This even includes hot-fixes that even through they originate from a release branch and merge back into that release branch, they also must merge into the `master` branch to ensure the fix is permanent and part of all future builds
* Keep branches around for the shortest period possible, usually deleting them after they have been squash merged into the `master` branch

### Common/standard GIT branching methodologies

It is valuable to understand how our branching strategy aligns and diverges from the following most commonly used GIT branching strategies to better understand where and why we deviate (if at all ) to best meet our current development and deployment workflows and standards.

* [Git-Flow](https://nvie.com/posts/a-successful-git-branching-model/) - Designed for large projects with scheduled release cycles
* [Microsoft's Trunk Based Branching](https://docs.microsoft.com/en-us/azure/devops/repos/git/git-branching-guidance?view=azure-devops) - Simpler than Git-Flow, while still supporting enterprise sized applications with non-CD sprints (3 weeks for MS)
* [GitHub-Flow](https://githubflow.github.io/) - Simplest design based on support CD with multiple deployments per day
  * Smallest changes with most frequent/continuous production deployments, but also takes greatest risks
  * Less non-production environments, with testing being done in production before accepting code into master

### Addition References

* [How Microsoft uses Git internally](https://docs.microsoft.com/en-us/azure/devops/learn/devops-at-microsoft/use-git-microsoft)
  * Dev environment auto builds from master
    * QA occurs in dev environment and staging (UAT) environment)
  * Staging (UAT) environment (temporary) builds from release branch that is pulled from master when feature complete
    * Bug fixes started after release branch creation go to master first then cherry picked into release branch
* Video - [Microsoft - Git Patterns and Anti-Patterns for Successful Developers](https://www.youtube.com/watch?v=t_4lLR6F_yk)
* [How We Use Git at Microsoft](https://docs.microsoft.com/en-us/azure/devops/learn/devops-at-microsoft/use-git-microsoft)
* [Resolving a Merge Conflict from the Command Line](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/resolving-a-merge-conflict-using-the-command-line)
* [How to cherry-pick multiple commits](https://stackoverflow.com/questions/1670970/how-to-cherry-pick-multiple-commits)
* [Using Git with Microsoft VS Team Foundation Server or VS Team Server Online](https://channel9.msdn.com/Events/Ignite/2015/BRK3709)
* [Git Branching and Policies in Team Foundation Server 2015](https://channel9.msdn.com/Events/Visual-Studio/Visual-Studio-2015-Final-Release-Event/Git-Branches-and-Policies-in-Team-Foundation-Server-2015)
* [Basic Git Commands](https://confluence.atlassian.com/bitbucketserver/basic-git-commands-776639767.html)

### Demo - How to Pull Story Branch at last minute (after commit to development) if preferring the command line

* These steps document and demo moving stories from `development` to `qa`, to `staging`, and finally to `production` with some stories being held at lower branches until the business is ready for them
  * The following branch names are used for this demo
    * (`master,qa,staging,production,storyNotPastDev,storyNotPastQa,storyNotPastStaging,storyThroughProduction`)
* Create story branches from `master` branch (development environment)
  * The demo adds a file to each story with the story name, to make it easier to see what stories make it to each environment
* Pull request each story branch into the dev branch (as usual), using squash merge and name merge using story # (also as usual)
  * Merge branch is not needed
  * Can delete story branch after merge as it is no longer needed
  * Demo code simulating pull requests merging back into `master`

```js
git checkout master
git merge --squash storyNotPastDev
git merge --squash storyNotPastQa
git merge --squash storyNotPastStaging
git merge --squash storyThroughProduction
git push
// Safe to delete all the above story branches after they have merged into master
```

* Cherry pick all stories into `qa` except the one named `storyNotPastDev` simulating a story the business is not yet ready for moving up the environment stack.  It is recommended to cherry-pick in the order from oldest to newest commit to reduce merge conflicts (can also use `gitk --all` or meld for GUI based merge conflicts)

```js
git log --oneline
git checkout qa
git cherry-pick <hash for storyNotPastQa>
git cherry-pick <hash for storyNotPastStaging>
git cherry-pick <hash for storyThroughProduction>
git push
```

* Cherry pick all stories into `staging` except the one named `storyNotPastQa`.  `storyNotPastDev` will not be available because it never made it to `qa`

```js
git log --oneline
git checkout staging
git cherry-pick <hash for storyNotPastStaging>
git cherry-pick <hash for storyThroughProduction>
git push
```

* Cherry pick all stories into `production` except the one named `storyNotPastStaging`. `storyNotPastDev` and `storyNotPastQa` will not be available because they never made it to `staging`

```js
git log --oneline
git checkout master
git cherry-pick <hash for storyThroughProduction>
git push
```

## Tasks - If preferring the command line

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

### Create a new `story` branch from `master`

```cmd
cd <new-repository-name>
git checkout master
git checkout -b story
git push –u origin develop
```

### Commit changes from `story`

```cmd
git add *
git commit –m “commit message”
git pull
git push origin story
```

### Merge changes from `story` back into `master`

if merging a story branch, add param `--no-ff` to the merge command to ensure the historical existence of a story branch and groups together all commits that together added the story

```cmd
git checkout master
git pull
git merge story
git tag -a <version>
git push origin master
```

### View any merge conflicts

```cmd
git diff
git diff --base <filename>
git diff <sourcebranch> <targetbranch>
```

### Delete `story` branch

```cmd
git branch story –d
git push origin story
```
