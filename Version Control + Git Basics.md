#W6L2 #W7 #W8L1
# Version Control Philosophy
## Big Picture

There are *Developers* and *Operation Staff* (different).
Alternative school of thought: combine both into *DevOps*.

Developers deal with **version control**, similarly, operations staff have to deal with **backups**.

Backups deal with:
- hardware problems
- software bugs 
- human error 
- cyber attacks

Version Control (Git) deals with similar things, much overlap.

## Backups

Periodically, make a copy of all the data
- data stored on flash or hard drive, and copy it over into a backup device
 - expensive, slow

Suppose the CPU wants to make changes while data is being copied- then the backup data is inconsistent
- need to freeze activity / prevent changes to system while doing backups

<ins>Staging Backups</ins>
First back up to faster secondary storage, then move down the list as you run out of space.

SEASnet does 2 kinds of backups:
1. copy all data periodically + backup (not a snapshot)
2. file system that occasionally takes snapshots
	- method: **copy-on-write (CoW)**

Try: `$ cd .snapshot` (which is hidden even with `ls -al` !!!) and `$ ls -al` to see hourly, nightly, weekly snapshots.

### Copy-On-Write

Copy-on-Write is done at the block level (rather than file level). We split up the file into blocks, and just modify the pointer to the block that is changed.
Example: if you just do `cp f g` then g still points to f (not written to yet). Then when you modify g, the specific block that is changed has its pointer changed.

### What should we back up?

1. Data in files (file contents)
2. Metadata about files (permissions, timestamps, `$ ls -l` output)

### To what extent should we back up?

1. What user sees (data + metadata)
2. Underlying blocks used to implement (bytes)

Note that the metadata is kept separate (don't want users to be modifying it).

### Reducing Backup Costs

1. Backup less frequently
2. Use cheaper (slower) backup devices
3. Back up to local third-party service (ex. AWS)
4. Incremental backups on a schedule (e.g. back up only files that have changed)
	- every so often, do a full backup to avoid worst case
	- Alternatively, backup *only the small changes* (delta)(not the whole file)
5. Discard old backup data on a schedule

### diff and patch

Can use `$ diff -u oldfile newfile` (expanded version) or `$ diff -U0 A B` (concise) output (delta) to get just the changes from old to new file
- Recall: `diff <x> <x+∆x>` shows +∆x
- how is this different from Copy-on-Write? `$ diff` is more powerful, operates on line-by-line basis while CoW operates block-by-block 

If we did `diff A B > diffout`, we can then apply `diffout` file to `A` . So `diffout` is the **patch file**: output of `diff` command that contains series of instructions (additions, deletions, and modifications) needed to transform the original file into the modified version.

Syntax of `$ patch`:
`$ patch target_file patch_file` or `$ patch target_file <patch_file`

**We can also run patch in reverse** (un-patch a file) with `-r`:
- `$ patch -r reject_file target_file patch_file`

### Other Backup Optimizations

- Deduplication
	- many blocks have same content, use a table
	- backup the table too
- Compression
	- ex. `$ gzip` 
	- data that goes to backup device is smaller
- Staging (again)
	- fast device to slower once it fills up
- Encryption
	- if you don't trust backup device as much as original
- Data grooming (clean up original data)

# Version Control System (VCS)

## Basic Needs/ Useful Features

1. Contain history of project (data + metadata), ideally indefinitely
2. Connections (ex: to bug database)
3. Review the code
4. Additionally: Know about plans for *future* changes (not just for the past)

Note: Metadata is for "software archaeology"
1. File metadata
2. Commit metadata: Who made the change, and WHY? (Extra comments "Commit message" separate from code comments)

Note: "future plans" can conflict if there are many different ones

`$ git log` outputs metadata about commit. Shows commit history in reverse chronological order, with most recent commit first.

## Questions & Additional Features

Question: how smart is diff program?
- do we model a file rename as rename or as delete + insert?

<ins>Additional features:</ins>

*Metadata about the history* (this is separate, think like meta-metadata). 
- Example: what was I working on last week?

*Atomic Changes*: change all instances simultaneously in one snapshot or nothing

*Hooks*: increase reliability by issuing check before commit. Goal: want to tailor your VCS to project
- Ex: pre-commit hook that 
- place script into `.git/hooks` directory

*Format Conversion*: convert source code from different machines to standard format
- ex: CR LF <--> LF

*Cryptographically Signed Commits*

# Git Intro

## Intro (from the user's POV)

Distributed VCS.

Git repo has 2 parts:
1. Object database recording history
2. Index/Staging Area (plans for next change)

Outside of Git, have private **working files** on computer (would exist even if git didn't exist at all)

2 Starts:
1. `$ git init` creates new empty git repo
2. `$ git clone <origin> <new_repo_name>` with remote repo URL or local file path clones existing repo + creates empty index 

The `.git` files contain history.

### Cloning example

`git clone diffutils my-diffutils-clone`
- first, it copies the repository
- then it extracts working files from latest commit
`du diffutils my-diffutils-clone`
- for `diffutils`: the `.git` directory is small 2450 KB (compressed)
- for `my-diffutils-clone`: the `.git` is only 180 KB. How??
	- `.git/objects/pack` stores compressed representations of Git objects

### Note about hard links / speed

Important: history is read-only, which means we can just hard link the cloned repo history to the original. This is why you can do a local clone very quickly (using hard links).
Once you modify the clone, Git will do something like CoW.

## Commit IDs

40 hex digits, can just specify the prefix.

## Types of Git Files

There are three possible "values" of "latest version" of any file 
1. Latest commit
2. Index/Staging, plan for next commit. Files staged using `$ git add`. Use `git rm` to remove a file and stage this deletion.
3. Working file (any file you're editing) that can be tracked or untracked

At first, index is empty and contents of 1=2=3 but once you start editing, all 3 can be different (common cause of confusion).

## Using git diff 

### Compare 3 relative to 2
`git diff` shows difference between what's in 2 and 3
-  `+` meaning additions in working that's not in index
`git add` adds a copy of working file into the index.
- `git add -A` adds everything
- `git add -u` adds only files that updated

### Compare 3 relative to 1
`git diff HEAD` compares 1 and 3
- `+` meaning additions in working not in HEAD

### Compare 2 relative to 1
`git diff --cached` compares 1 and 2.
- `+` means additions in the index not in HEAD (essentially shows the differences from `git add`)

### Using two arguments

`git diff file1 file2`
- first  is "old" file
- second is "new" file 
- changes in new not in old will display as +

## Using git add and git commit

`$ git add filename` copies the working file (can be tracked or untracked) into the index.
- if run on tracked file or modified file, stages just the *changes*
- if run on untracked file, *stages entire file and starts tracking it*
- if run on deleted file, stages deletion
- if run on a directory, stages all changes made within directory 

`$ git commit filename` copies whatever's in the index into the object database. **Git creates a new commit object** that contains metadata like author, commit message, timestamp, reference to parent (previous) commit. This commit object is then added to the object database and assigned a hash. Git also updates the branch reference to point to new commit
- Note: `git commit` only adds new history, does NOT change existing history

<ins>Commit Message Conventions:</ins>
- First line: short (!) elevator pitch
- Second line: empty
- Third line: detailed explanation of WHY

## What exactly is HEAD?

A *symbolic reference* that points to **currently checked-out state in repo** ("the newest commit we are currently looking at"), which means acts as a pointer or a placeholder to another reference rather than directly pointing to a specific commit. HEAD can point to a branch, a specific commit, or tag.
- When we say branch, we mean HEAD can point to a branch pointer which in turn points to a commit
- Detached HEAD: it can be detached from any specific branch and point directly to a specific commit

# More Git + Ambiguities

## Manual Page

`man git`
- this is a table of contents, but actual individual separate commands like `man git-add` have their own man pages
- git commands start with `git-`.

## AMBIGUITIES, the -- option

### What is -- ?

The `--` is a common convention used in Unix command-line tools and Git to signify the end of command options. *After this signal, only positional parameters are accepted (like filenames).* 
- `--` works with all git commands to specify filenames

### Examples with Git

Suppose we had a file named HEAD. Then this is ambiguous. 
We can specify the `--` option with a space before any filenames like `git log -- HEAD` to ensure it is interpreted as a filename.
Or we can use the full file path.

Example with `git diff`: 
`git diff abc1345^!` will show just the new changes in the commit `abc1345`.

`git diff HEAD^! -- bar.c` shows only changes in head that affect `bar.c`.

## git status

Tells a short summary of current state of repository.
- current branch and status in relation to master branch
- lists "Changes not staged for commit" (not staged) and "Changes to be committed" (staged)
- if changes already added to commit: "Nothing to commit, working tree clean"

## git show

By default `git show` shows info about latest commit. Can specify a commit ID. Can see the changes.

`git show HEAD~n` means show previous n commit

## git log: Look at Existing History

[Documentation](https://git-scm.com/docs/git-log)

`$ git log`: look at changes made in *reverse time order*

`$ git log <options> filename` (or `$ git log <options> -- filename` ) gives you a subset of git log- lists all commits that affect this file.
- can also put Commit ID

### Useful Options git log

`git log --follow filename` shows changes overtime for file
`git log -S <string>` ("pickaxe" search) searches commit history for *changes* *made* to a particular string.
`git log --grep="some commit message"` search in commit messages
`git log --merges` shows only merge commits
`git log --patch` or `-p` shows the patch/the diff in commit
`git log --author="Author Name"` shows by author
`git log --before="2024-01-01"` ( `--after`  works too)
`git log -n 1` displays 1 commit (limits number), or you could just do `-1`
`git log --reverse` shows oldest commits first.
`git log --oneline` or `git log --pretty=oneline`
 `git log --graph`

 `git log --pretty=format:""`  
- Note: the author and committer do not have to be the same.
- Common Specifiers for pretty format:
	- `%H` commit hash, `%P` parent hash, lowercase is abbreviation
	- `%s` commit message subject (header), `%b` commit message body
	- `%an`, `%ae` , `%aD` author name, email, date
	- `%cn`, `%ce`, `%cD` committer name, email, date
### Operands: Revision Range

Note that these work for `git diff` and `git blame` too.

Git log operands allow us to specify ranges or sets of commits.
- `HEAD`, `^`, `~`, `..`, `...`

#### Double Dots- 2 Argument Revision Range

`git log A..B` shows the commits reachable from `B` but not from `A` 
- like (A, B]
	- like `git log start..end` 
	- but shows **backwards** (starts from B and goes backwards in time to A, but not including A)
	- if you do `git log A..`, will get (A, infinity) so everything up to the current commit excluding A. If you want to include A do `git log A~..`

The following will just give the current commit itself:
- `git log HEAD^..HEAD` 
- `^!` is a shorthand for the above: `<commit>^..<commit>` so equivalently: `git log HEAD^!` 

#### Just one argument (NO DOTS)

This is equivalent to just providing the 2nd/"end" argument.

`git log HEAD` shows all commit history going backwards from current.
`git log HEAD^` shows commit history going backwards from parent of current.
- `HEAD^` refers to the parent commit to HEAD
Each `^` moves one commit back in history.
`HEAD^` = `HEAD~` or `HEAD~1` 
`HEAD^^` = `HEAD~2` 
`HEAD^^^` = `HEAD~3` 
If you go past the initial commit will give error.

#### Triple Dots: Symmetric Difference (Not covered in lec)

Used when you have **branches** in history (more later)
`git log A...B` shows commits either reachable from A or B BUT NOT BOTH (shows commits unique to A or B). Used with
- often used in merge or rebase operations

## git ls-files

Lists all files under version control in current branch.
Can combine w other shell commands:
- `$ ls -ld $(git ls-files) | less`
- `$ grep 'hi' $(git ls-files)`

## git grep: Searching

Searching is so common that Git has its own command `$ git grep <pattern>`. This searches for a pattern in tracked files in current branch (or a specific branch/commit if specified, ex: `git grep "searchString" other_branch`)

To search across all branches:
```sh 
for branch in $(git branch --list --all); do
    git grep "searchString" $branch
done
```

## git blame

[Tutorial](https://www.atlassian.com/git/tutorials/inspecting-a-repository/git-blame)

*Only operates on individual files (not commits)*. Specify file path or name.

`$ git blame example.txt` allows us to see history of file line by line
- can see the commit ID, *last author to modify line*, timestamp, line number, and change

Note that `$ git blame commitID` does not work (this is not a file).
However, we can use commit references to investigate the state of a file at specific points in history.
- Detective work: look at each line in the same file in the parent commit: `git blame abcd123^ -- example.txt`
	- Who was responsible for each line in `example.txt` just before the changes made in `abcd123` were applied?"

If we want to see differences in a file between 2 commits, use git diff:
- `git diff abcd123^..abcd123 -- example.txt

Note that `git blame` only works for lines currently in the file. If you want to see full history, better to do `git log -S` (see Lab 4).

## git clean

Remove untracked files.
`git clean -n` simulates/dry run what would happen if you did the command.
`git clean -f` force removes.


# Git as a DVCS

Git is a Distributed VCS as opposed to Centralized VCS.

## What is a Centralized VCS?

Have 1 repo, sitting in a server. Client sends requests to server to make changes. 
- Huge disadvantage: performance. Super slow, not scalable.
- You can use Git in this way too: put everything on GitHub, don't use local repos.

## Local vs Remote Repos

We have several different repos scattered around network, could be peer-to-peer. Some repos could be more important (could have a "main repo"). *But all have a complete history of the project*. 
- Pros: performance
- Cons: histories start to diverge (hard to resolve)

Simple Git Model:
- have a remote "main repo" (ex: on GitHub) that the local repos talk to
- in the  `.git/config` file, have a `remote` section which names the remote repo (`[remote "origin"]`)
	- within this section, 2 variables `url` (specifies URL to remote repo) and `fetch` (tells Git which branches to fetch and how to map them to your local repository)
	- `url = https://github.com/user/repo.git`
	- default `fetch = +refs/heads/*:refs/remotes/origin/*` which means fetch all branches from remote and store them under `origin/` in local repo

### Remote-Tracking Branch

A **remote-tracking branch** is a branch in local repo with syntax `origin/<branchname>`.
- in other words: `origin/master` is a local representation of what the `master` branch looked like on the remote the last time you updated it,
Note: The remote-tracking branch is **separate** from other local branches like `master` and `feature`.

## Communicating: git fetch, git pull, git push

2 way communication between your repo and remote repo:

`$ git fetch` copies remote to my own remote repo.
- updates the remote-tracking branch `origin/master`
- updates your local repository's opinion of what the remote branches look like, *but it doesn't merge any of those changes into your local branches*
- doesn't cause errors since it doesn't integrate changes into local working tree

`$ git pull = git fetch + git merge`
- `git fetch` updates `origin/master` 
- `git merge` merges local `master` branch with `origin/master
- so essentially `git pull` attempts to integrate these changes/synchronize into working tree (can causes errors)

`$ git push`: 
- WRITE/publish your changes in local working tree to upstream (remote) repository
- potential problems: if your opinion of remote is out of date, push will fail (need to resync your local repo)

# Branching

Generalize the idea of using the index to plan future changes.

Newest commit/HEAD points to parent (going backwards) until reaching the oldest commit (which doesn't have a parent).

A **branch** is a lightweight moveable name for a commit that is intended to follow a line of development.
- branches are essentially pointers to the most recent commit on that branch, often referred to as the "tip" of the branch. When you make a new commit on a branch, the *branch pointer automatically moves forward (like HEAD) to point to this new commit*, which becomes the latest commit on that branch
- always represents the most up-to-date state of the branch 
- unlike commits, branch pointers can change!

Branches represent other possible lines of development. We can look at these by checking out branches.

When you use `git add` and `git commit`, you are only doing so within the scope of the one branch you are currently on. So you are only modifying this branch.

## Why Branches?

1. Multiple views of what the future should be
2. Multiple releases of some software (want updates to apply to old versions too, can cherry-pick the branches to apply to)
	- not very scalable nor maintainable though

## git branch, git checkout 

`$ git branch` lists all branches in repo.
- `$ git branch new_branch` creates without checking out
Useful Options: 
- `-d` or `-D` to (force) delete locally. Note that this doesn't delete any commits, only deletes the branch pointer (but all history is retained). This means that the commit that it points to can now only be accessed through its commit ID (hard)
- `-m` to rename
- `--all` or `-a` lists all local AND remote branches

`$ git checkout branch2` will switch the HEAD to point at the tip of `branch2`. However, note that `main` and `branch2` do not change in the sense that they still point to the same branch tips they did before `checkout`.
Useful options:
- `-b` to create a new branch
- `-` to switch to previous branch
- by default `newbranch` will start from HEAD
To create a branch and change its starting point: `$ git checkout -b newbranch startingbranch`
- `newbranch` will start from the tip of `startingbranch`

# Merging

[Tutorial](https://www.atlassian.com/git/tutorials/using-branches/git-merge), [super comprehensive rundown](https://www.biteinteractive.com/understanding-git-merge/)

So far, we've only seen 2 branches/children diverging from one commit. But what about one commit having 2 (or more) parents? This is called **merging**. 

We now run into collision issues between these different lines of development.

## Collision Resolution

In a normal Git graph where all commits are connected and there is only one commit without a parent (oldest commit), bewteen any 2 commits we can find a common ancestor.

Let's simplify our model and look just at 3 commits: a common ancestor A, and two commits X and Y who share that ancestor (there might be a bunch of commits along each).

To fix collisions for all these methods, typically we have to hand edit files.

### Diff3 method for merging FILES

We can do a diff between the common ancestor and the target files  to see what changes we should worry about.
- `$ diff -u A.txt X.txt`

Now we can use `diff3` to find the merged output. This is a three-way merge that outputs results directly with conflict markers.
- Syntax: `$ diff3 -m B A C >mergedfile` where A is the base file
- Conflict Markers look like `<<<` `===` and `>>>`

So we can do `$ diff3 -m X.txt A.txt Y.txt`
Conflict Markers:
- The `<<<<<<< x.txt` marker indicates the start of the conflicting changes from one version of the file
- The `||||||| common-ancestor.txt` marker separates the original content (from the common ancestor, which is the last agreed-upon version before the branches diverged) from the conflicting changes in `x.txt`.
- The `=======` marker separates the changes from x and y
- The `>>>>>>> y.txt` marker indicates the end of the conflicting changes introduced in the other version of the file (`y.txt` in this case).

### Patch Method (using diff) for merging FILES

`cp A C` creates a copy C of the common ancestor A.
Run the following patches:
1. `patch C <A-X.diff` is guaranteed to work, and now C=X.
2. `patch C <A-Y.diff` might have conflicts.

## git merge 

[Documentation](https://git-scm.com/docs/git-merge)

`$ git merge` uses the same methods as previously discussed (finding deltas, merging multiple ways) *to merge entire branches together*.
- automatically creates a commit!! unless conflict
	- if there is conflict, `git add` the changes then `git commit`

`$ git checkout X`
`$ git merge Y`
This new node will have X and Y as its 2 parents. Note that both X and HEAD follow to point to this new node.

### Octopus Merge

Octopus merge is simultaneously merging multiple branches together.
Suppose we are on main and we do `$ git merge X Y`. This new node will have 3 parents: main, X and Y. Note that main and HEAD follow to point to this new node.

### git merge-base

`git merge-base branch1 branch2` finds the common ancestor between 2+ branches. This will change after merging to the tip of the other branch (see diagram).

### Fast-forward

If you have a situation where the merge-base of 2 branches is one of the branches (ex: `master`), we can slide `master` pointer up to the `updatedbranch` pointer (see diagram). Now both branches point to same place.

##  Non-conflicting merges can also cause problems

For example, if you're allocating two huge arrays and you run out of memory.

# Rebasing

[Tutorial](https://www.atlassian.com/git/tutorials/rewriting-history/git-rebase)

This is a competing branch of thought to merging.
One *straight linear chain* of development.
- Pros: simple history (easier to debug). Cons: you're lying about this history.

How does `$ git rebase` do this?
- it takes the deltas from each commit along one branch and applies them sequentially (adjusting them if needed) to the one you are on (we've effectively copied over the changes to our branch)

Note: although this may seem like a fast-forward, a key difference is that the two branches still point to different commits afterwards.

If I want to rebase main:
`git checkout mybranch`
`git rebase main`
- this takes changes along main and applies them sequentially to mybranch
- if you just look at commits reachable from mybranch, we see a linear chain of development 

## Rewriting History with --interactive

`$ git rebase -i`
- `-i` : interactive 
- drops you into text editor where we can delete, edit, and reorder commits (rewrite history)

Syntax: we specify the start point of what we want to rebase (by default, this goes up to the current HEAD). However, note this syntax:
`git rebase -i <commit-hash>` acts as a marker that says "Start rebasing from the *next commit after* this one." (see diagram)
- To rebase everything up until the last/most recent N commits: `git rebase -i HEAD~N` or until a specified

```bash
Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commi
```

# Comparing Rebase and Merge 

Remember: both are subject to collisions.

## Number of commits created:
- `git rebase` creates `n` new commits, where `n` is the number of commits in the other branch
- `git merge` creates just 1 new commit

## When to do each

When project gets big & messy, merge.
If (sub)project is small, rebase (keep history simple). Also rebase if you want to do `git bisect`.

# Rollback with Reset, Revert

## git revert

This doesn't rewrite history but reverts to a previous commit specified by hash.

## git reset

3 modes: `--soft`, `--mixed`, `--hard`. This has the potential to rewrite history (`git reset --hard` resets the HEAD, index, and working directory to the state of a specified commit.)

# Stashes with git stash

Keep track of changes to working files without actually committing. This is good if we want to switch branches but don't want to lose our work. Only works for local repo.
- Another extension of idea of index. Think: take a snapshot of index so I can come back to it later. 
We have a **stack** of stashes, each with unique index like `stash@{0}, stash@{1}` etc
- `git stash push`, `git stash pop`.
`git stash list` can recover specific stash

# Debugging with git bisect

Let's say our current HEAD is a bad commit, but we know previously there was a good commit. Git bisect runs a *binary search* O(logn) to find the commit that introduced the bug. (Note that this great for rebase!)

`$ git bisect start bad good` where bad and good are the commits.
`$ git bisect run ./test.sh` where this is your testing script. 
- this finds the first bad commit and does a checkout (can then do git log)

## Potential Issues

What if there are MULTIPLE bad commits? Then the result doesn't necessarily find the culprit.
- `git bisect` has many options to help with this

What if we have merges (non-linear chain of development?)
- find some path between good and bad, find bisection on that path




