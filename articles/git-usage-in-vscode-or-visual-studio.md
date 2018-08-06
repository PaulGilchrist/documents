# Git Usage in VS Code or Visual Studio

Always make sure you have the latest version of Git installed (`git –version`).  When installed, it should add itself to the path.

   [https://git-scm.com/downloads](https://git-scm.com/downloads)

Optionally, you may also want to install the GitHub desktop at [https://desktop.github.com](https://desktop.github.com), but all needed features can be executed from the command line or from within VS Code of Visual Studio without the desktop installed.

Many of the below commands can be done from a GUI interface like VS Code, Visual Studio, or GitHub desktop, but will be shown here in their command line form.

## Tasks

### Create new repository for existing project

1. Create new repository in GitHub or VSTS
2. Clone new repository onto PC

```cmd
cd \source
git clone https://github.com/PaulGilchrist/<new-repository-name>
```

3. Copy project files into newly cloned folder

### Create a new project from existing template project

```cmd
cd \source
git clone https://github.com/PaulGilchrist/Angular2Template.git <newProjectName>
cd \<newProjectName>
rd .git /s /q
git init
git add .
git commit –m “Initial commit”
git remote add origin <repo-address>
git push -u origin master
npm install
gulp rebuild
npm start
```

### Create a new “dev” branch from “master”

```cmd
cd \newProjectName
git branch dev
git checkout dev
git push –u origin dev
```

### Commit changes from “dev”

```cmd
git add *
git commit –m “commit message”
git pull
git push origin dev
```

### Merge changes from “dev” back into “master”

```cmd
git checkout master
git pull
git merge dev
git push origin master
```

### View any merge conflicts

```cmd
git diff
git diff --base <filename>
git diff <sourcebranch> <targetbranch>
```

### Delete “dev” branch

```cmd
git branch dev –d
git push origin :dev
```

[More Info](https://confluence.atlassian.com/bitbucketserver/basic-git-commands-776639767.html)

## Git Version Control Branching Recomendations

It is recommended to follow a hybrid approach between [git-flow](http://nvie.com/posts/a-successful-git-branching-model) and [github-hub](http://scottchacon.com/2011/08/31/github-flow.html), mainly sticking to the git-flow model only because of an existing strategy around continuous integration with multiple the multiple environments (dev, qa, staing, and production). Like git-flow, there will be a branch maintained for each permanent hosting environment (usually dev, qa. staging, and production). If you have read git-flow, the “qa” and “staging” branches can be thought of as that document’s “release” branches. The production branch will always be called “master” and it should always be in a production ready state. Anytime an environment branch is updated, continuous integration should immediately update its respective hosting environment.

New feature branches should be created from the “dev” branch and merged back into the “dev” branch when ready for integration testing. When QA is ready to test a new release, the “dev” branch can be merged into the “qa” branch. Any bugs found in QA would be fixed in its own branch cloned from “qa”, and when complete pushed back to both the “qa” and “dev” branches. Likewise, any bugs fixed in the “master” branch would have to also be merged into the other environment branches.
