# GIT Branching Recommendations

The 3 most commonly used branching methodologies include [Git-Flow](https://nvie.com/posts/a-successful-git-branching-model/), [GitHub-Flow](https://githubflow.github.io/), and [Microsoft's Trunk Based Branching](https://docs.microsoft.com/en-us/azure/devops/repos/git/git-branching-guidance?view=azure-devops).  Git-Flow is designed for very large projects with scheduled release cycles but is also the most complex.  GitHub-Flow is the simplest but designed for continuous deployment at the story vs sprint level, with more of a roll-forward than rollback approach.  Microsoft's Trunk Based Branching methodology (also called [Release-Flow](https://docs.microsoft.com/en-us/azure/devops/learn/devops-at-microsoft/release-flow)) is more complex that GitHub-Flow but simpler than Git-Flow, while still supporting enterprise sized applications using sprint based deployment cycles.  None of these methodologies directly address our requirements for 4 or more permanent environments (dev, qa, staging, and production), but Microsoft's trunk-based branching method comes the closest and is the one recommended.

Microsoft has a proven track record, using and recommending GIT for all their applications (since 2012).  Microsoft owns the worlds largest GIT repository (Windows), and also owns GitHub, the world's largest GIT repository.  Their branching methodology is based on a single trunk (named master) which auto-deploys through CI/CD to a dev/test environment.  

The `master` branch represents our development environment and is for code considered complete, unit tested, code reviewed, and ready for QA.  The `master` branch always up to date and buildable.  This allows for fewer merge conflicts, easier code reviews, and simpler and faster deployments.

A `story-<workitem>` branch (sometimes referred to as a feature branch) should be for a small, simple change, merged from `master`.  Once code complete, unit tested, and ready for QA, a `pull request` should be used to merge the story branch back into the `master` branch.  Pull requests include a code review with a recommended 2 minimum reviewers' approval.  An approved pull request should be `squash` merged back into `master` to keep it clean and only containing the final commit of each story.  

A `release-<version-#.#.#>` branch is merged from the `master` branch, but never merged back, as its sole purpose is to deploy code to higher environments (including production).  Release branches are created with limited lifespan and should be named with their version number.  Old release branches should be removed when no longer needed.

Multiple release branches can exist at any one time, but any newly created release branch must contain (at a minimum), all comitts from the previous release branch.  This requirement ensures all comitts in `master` eventually makes it into a release and ultimatly into production or are removed from the `master` branch.

A release branch is attached to the CI/CD build pipeline associated with one or more environments.  The release branch is ususally moved up the heiarchy one environment at a time until ultimatly reaching production.  When a bug is discovered in a release branch (including production), a `hotfix-<workitem>` branch is created from that release branch and when code complete, `squash` merged back into that same release branch, as well as cherry picked into the `master` branch and any newer release branches.  Azure DevOps is optimized for this workflow and guides the developer as follows:

* Create branch from work item
* Create pull request recommended after new commit to origin branch
* Cherry pick button appears after pull request is completed

At any time, new comitts to `master` can be merged or chery picked into a release branch as the sprint progresses, as long as they are also merged into any newer releases.

Review the Microsoft Trunk Based Branching documentation in the Appendix below for more details on this branching strategy.

![](https://github.com/PaulGilchrist/documents/blob/master/articles/git/git-branching-recommendations/git-process-flow.png)

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
* [Resolving a Merge Conflict from the Command Line](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/resolving-a-merge-conflict-using-the-command-line)
* [How to cherry-pick multiple commits](https://stackoverflow.com/questions/1670970/how-to-cherry-pick-multiple-commits)
* [Using Git with Microsoft VS Team Foundation Server or VS Team Server Online](https://channel9.msdn.com/Events/Ignite/2015/BRK3709)
* [Git Branching and Policies in Team Foundation Server 2015](https://channel9.msdn.com/Events/Visual-Studio/Visual-Studio-2015-Final-Release-Event/Git-Branches-and-Policies-in-Team-Foundation-Server-2015)
* [Basic Git Commands](https://confluence.atlassian.com/bitbucketserver/basic-git-commands-776639767.html)


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
