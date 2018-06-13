---
title: "git magic"
date: 2018-6-10
excerpt: "Automating repetitive git tasks using custom shell scripts"
---
Tired of running the same 5 git commands over and over again, inevitably forgetting a few, and settling for an impossibly convoluted commit history? Same. Luckily, you can pretty easily transfer your frequently chained git commands into a shell script to greatly simplify your process.  

### Where to Put your Scripts
To start out, you need to locate your `$PATH` for git bash. This is an environment variable referring to the location where all executables/scripts for git live. It can and can be easily found by running `which git` in git bash. The default `$PATH` location on Windows should be `/mingw64/bin/` wherever git bash is installed. Here is where you will save your scripts.

### Quick Aside - Making Git Faster
Next, there are a few global options you can set to make git run faster. The first two are actually set by default in new versions of git. You can find more info on git global options <a href="https://git-scm.com/docs/git-config" target="_blank">here</a>

```bash
git config --global core.preloadindex true # enables parallel index pre-loading and speeds up operations like git status
git config --global core.fscache true # makes it so you don't need to run git as an admin, which causes performance issues
git config --global gc.auto 256 # sets a max limit for .git files
```

### Shell Script Syntax
Now let's dive into the syntax of one of the scripts that I've written to help with creating a new branch and auto-pushing it upstream to `origin`.

```bash
#!/usr/bin/env sh
#
# git-new_branch
#
# Matt Guan
# move all changes to a clean branch, fast forward it to origin/master, and push it to origin
# usage: git new_branch {branch_name}

exists=`git branch --list $1`

if [[ -z $1 ]]; then
	echo "WARNING: Set branch name as argument"
	exit;
elif [[ $exists ]]; then
	echo "WARNING: Branch already exists ; did you mean to git checkout and use git magic?"
	exit;
else
	git stash
	git checkout -b $1
	echo "INFO : branch created"
	git fetch origin
	echo "INFO: fetched origin"
	git rebase origin/master
	git stash apply -q stash@{0}
	git stash drop
	echo "INFO : branch now at origin/master"
	git push -u -q origin $1
	echo "INFO : branch pushed to origin"
fi
```

I start the script out with a standard unix `shebang` that indicates that this script should be run with the first shell found on `$PATH`. `#!/usr/bin/env sh` is the particular argument that has worked for me on Windows. Note that you can substitute the `sh` argument with your scripting language of choice (`ruby`, `bash` etc).  

From there on, you can just list out whatever git commands you want to chain together. In this particular example, I also have some built in checks using an if else statement. 

1. First I define a command `exists` from the command `git branch --list $1`, which checks if `$1` (the first parameter passed into the script) matches the name of an existing branch. If there is a match, `exists` is set to the hashed branch name, else no output is returned.
2. Next, I want to verify that a branch name was even passed in. I do this by using `[[ -z $1 ]]`, a conditional expression checks if the `$1` parameter is Null (think **z**ero length). If no argument is passed, the script stops, and an error message is echoed.
3. Now I want to verify if our previously assigned `exists` command returns any output. If there is an output, it means that `$1` matches an existing branch, so we probably should not be using this command. To perform this check, we simply wrap our command in square brackets, which signals the script to check if an output is returned by `exists`. If the branch already exists, the script stops, and an error message is echoed.
4. Lastly, if the conditions are met, we just run our standard git commands in sequence. In this script I `stash` all un-committed changes, `checkout` a new branch, `fetch` origin, apply origin/master to the branch, re-apply my stash, drop the stash, and `push` the local branch to origin. That's a lot of work that can now be reduced to just one command! 

That's all there is to it! You should now be ready to write shell scripts to simplify your particular git workflow. 

Thanks for reading! If you're interested in taking a look at the rest of my git shell commands, you can find them in this <a href="https://github.com/andmatt/git-shell-scripts" target="_blank">repo</a>