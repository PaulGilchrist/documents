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

