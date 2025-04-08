---
date: 2024-04-04
date: 2024-04-04
categories:
  - git-tricks
---

# How to remove a file from a commit. 
> _Git Trick n°1_

Let's say you've `git add`ed a file that you did not really want to track. 
It should have been in the `.gitignore`, or was simply not meant to be there in the first place, but you were distracted and now you don't know how to remove it from there. 

There are 3 possibles scenarios : 

1. You added the file, but you've not committed yet.

    ```sh title="Untrack file, but do not delete it"
    git rm --cached path/to/file
    git commit --amend
    ```

2. You just committed locally, but did not push to a remote yet.

    ```sh title="Remove file from the last commit"
    git reset HEAD^
    git restore --staged <file-you-want-to-remove>
    git commit -m "New commit without that file"
    ```

3. You already pushed.

    !!! warning  "This rewrites history"
    
        Be very careful here.  
        Don’t do this on shared branches unless everyone knows. Somebody might already have pulled your commit(s). By changing past commits, you might very well break stuff.

    I have not actually tested this one, but there is [documentation about it](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)