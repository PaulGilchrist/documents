# GIT Tips

## Deleting a story/feature branch after merge

```js
// Delete a remote branch
git push origin --delete <branchName>
```

```js
// Delete a local branch
git branch -d <branchName>
```

```js
// Delete a local remote-tracking branch
git branch -dr <remote>/<branchName>
```

## Moving a GIT repository

The below commands allows for changing a local GIT repository to syncronize with a different remote repository location.  You should both start and end these steps witht he command `git remote -v` to confirm the before and after configuration.

```cmd
git remote -v
git remote rm origin
git remote add origin <urlToNewRepository>
git remote set-url --add --push origin <urlToNewRepository>
git remote -v
```

## Squash Merge

Story/Feature branches usually contain several commits before being ready to merge back into the `master` branch.  To keep `master` clean, it is a good idea to squash merge story/feature branches back into master to reduce them down to a single commit.  This can be done with the following command:

```js
git checkout master
git merge --squash <branchName>
git commit
```

## Syncronizing to multiple GIT repositories

The below steps allows for pushing the local repository to multiple remote repositories while only pulling from the primary remote repository.  You should both start and end these steps witht he command `git remote -v` to confirm the before and after configuration.

```cmd
git remote -v
git remote rm origin
git remote add origin <urlToPrimaryRepository>
git remote set-url --add --push origin <urlToPrimaryRepository>
git remote set-url --add --push origin <urlToSecondaryRepository>
git remote -v
```
