---
layout: post
title:  "Combining Git Repos"
date:   2017-09-19 14:30:30
categories: blog
tags: git
---
Recently, I came across a git repository that had scala subprojects as submodules within a super repository. Unfortunately this setup was no longer viable as the team size grew, so we needed to combine all the submodule repos into the main repo, all the while preserving history of each repo

Found the guide [here][combine].  

Solution
===
Consider a scenario where you need to move a repository A within a repository B

Create a new branch in repo B
```
cd <repo B folder>
git branch combine
git checkout combine
```

Create a new branch (`combine` in our example) in repo A that you will use to move files within repo B

```
git clone <repo A url>
cd <repo A folder>
git branch combine
git checkout combine
```

Create a directory within repo A which will be moved to repo B (In case there are other directories that need not be moved, consult the link mentioned before to filter history of those commits)

```
mkdir repoAFolderToMove
mv * repoAFolderToMove/
git add .
git commit
git push origin combine
```

Go to repo B folder and pull repo A folder

```
cd <repo B folder>
git remote add <new remote name for repo A> <path to repoAFolderToMove>
git pull <new remote name for repo A> combine --allow-unrelated-histories
```

At this point you might need to resolve any conflicts and then do a commit. That's it, you have successfully combined 2 git repositories along with their histories!



[combine]:                 http://gbayer.com/development/moving-files-from-one-git-repository-to-another-preserving-history/
