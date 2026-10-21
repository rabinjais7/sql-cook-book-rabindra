---
title: Where Should Data Quality Checks Happen in Medallion Architecture
date: 2021-03-17
author: Rabindra Jaiswal
categories: [git, utility]
---


# How I Use Git to Manage My Blog, Side Projects, and Almost Everything I Build

I have a habit of keeping important things in Git. Not just production code, but also my personal blog, resume, small Python scripts, side projects, and even experiments that I might want to revisit after a few months.

For me, Git is more than a version control system. It is a place where I keep the things I build, track changes, and make sure I don't lose my work.

I use GitHub to store my projects, and most of my development happens on my Windows machine using VS Code and the terminal. Over time, I have picked up a few small tricks that make working with Git much easier.

Here are some of the configurations and commands I find useful.

## 1. My favourite Git configuration

This is probably one of my favourite Git configurations:

```bash
git config --global push.default current
```

Normally, when pushing a new branch, Git may require you to explicitly specify the remote branch or set an upstream. With this configuration, Git uses the current branch name as the default push target.

For example, I might be working on my resume project:

```powershell
PS C:\Users\rabin\github_projects\rabindra_resume> git checkout -b improve-resume
```

After making my changes, I can simply run:

```bash
git push -u
```

Git pushes the current branch and sets up its upstream tracking branch, assuming the corresponding remote branch exists or can be created by the push.

No need to type the branch name every time.

One important detail: `push.default current` determines which branch to push by default, while `-u` sets the upstream tracking relationship. Both are useful together.

## 2. Set up Git once on a new machine

Whenever I set up Git on a new machine, these are among the first configurations I apply:

```bash
git config --global user.name "Rabindra Jaiswal"
git config --global user.email "your-email@example.com"
git config --global init.defaultBranch main
git config --global push.default current
```

These configure my commit identity, make `main` the default branch for new repositories, and simplify pushing.

I also prefer seeing a little more context when checking Git status:

```bash
git config --global status.short false
```

This keeps the normal, more descriptive status output enabled.

To check what I have configured:

```bash
git config --global --list
```

It's a simple way to troubleshoot Git behaviour when switching between machines or repositories.

## 3. Stop typing long Git commands

I use aliases for commands that I run frequently.

```bash
git config --global alias.st "status"
git config --global alias.co "checkout"
git config --global alias.br "branch"
git config --global alias.cm "commit"
git config --global alias.last "log -1 HEAD"
```

Now I can run:

```bash
git st
git co main
git br
git last
```

These aliases aren't essential, but they reduce repetitive typing.

Another useful one is a compact history view:

```bash
git config --global alias.lg "log --oneline --graph --decorate --all"
```

Then:

```bash
git lg
```

I find this especially useful when I have multiple branches and want to understand how my changes fit together.

## 4. Let Git remember my remote repository

When creating a new branch, I generally use:

```bash
git checkout -b feature-name
```

After making my changes and committing them, I push with:

```bash
git push -u
```

Because I have configured `push.default` as `current`, I don't need to specify the branch name, provided the remote is configured correctly.

After the upstream is set, subsequent pushes are simple:

```bash
git push
```

And to get the latest changes:

```bash
git pull
```

This is a small improvement, but when you work on multiple projects, small improvements add up.

## 5. Keep my blog and side projects safe

I use Git for more than application code.

My personal blog contains technical articles, and I also maintain side projects, Python scripts, and my resume. These are all things I may need to revisit or improve later.

A basic workflow works well for most of these projects:

```bash
git status
git add .
git commit -m "Update blog content"
git push
```

Before committing, `git status` helps me see what has changed.

One small habit worth following: I don't blindly commit everything just because `git add .` is convenient. For projects containing generated files, configuration files, or local environment settings, I check the changes first.

For example:

```bash
git diff
git status
```

And I keep secrets such as API keys, access tokens, and `.env` files out of Git using `.gitignore`.

GitHub is a great place to keep code, but it shouldn't become a place to publish credentials accidentally.

## 6. Use a global .gitignore

Some files are irrelevant to almost every project I work on: Python cache files, editor settings, and operating-system metadata.

Instead of maintaining every common exclusion in every repository, I can create a global ignore file.

For example, on Windows:

```bash
git config --global core.excludesfile "%USERPROFILE%\.gitignore_global"
```

Then I create `C:\Users\rabin\.gitignore_global` with entries such as:

```gitignore
__pycache__/
*.py[cod]
.vscode/
.DS_Store
Thumbs.db
```

This keeps common local files out of Git status across repositories.

I still keep a repository-specific `.gitignore` for files that are unique to a particular project, such as virtual environments, build output, or generated blog files.

## 7. Make Git output easier to read

A few other configurations are worth knowing:

```bash
git config --global color.ui auto
git config --global fetch.prune true
git config --global pull.ff only
```

Here's why I like them:

- `color.ui auto` enables colour in Git output when appropriate.
- `fetch.prune true` removes remote-tracking references for branches that no longer exist on the remote when fetching.
- `pull.ff only` allows a pull only when Git can fast-forward the current branch. It avoids creating an unexpected merge commit, although the pull can fail if the histories have diverged.

That last option is a preference rather than a universal recommendation. If your workflow intentionally uses merge commits, you may want a different pull configuration.

## 8. Undo mistakes without panicking

One reason I like Git is that I can experiment without worrying too much about losing previous work.

A few commands have saved me time:

See the latest commit:

```bash
git log -1 --oneline
```

Discard unstaged changes in a specific file:

```bash
git restore filename.py
```

Unstage a file without discarding its changes:

```bash
git restore --staged filename.py
```

Change the last commit message:

```bash
git commit --amend -m "Correct commit message"
```

The last command rewrites the latest commit. I avoid using it on commits that have already been shared unless I understand the implications.

The most important thing is to know the difference between commands that undo a change and commands that permanently discard work.

## 9. My everyday Git workflow

For most personal projects, I don't need a complicated branching strategy.

My typical workflow looks something like this:

```bash
git status
git checkout -b update-blog-post
# Make changes
git diff
git add .
git commit -m "Update blog post"
git push -u
```

Once the upstream is established, I can continue with the usual `git push` and `git pull` commands.

For a small project, this is usually enough. For a larger project with other contributors, I would use pull requests and a more deliberate branching strategy.

## Final thoughts

I don't think everyone needs to learn every Git command or follow an elaborate Git workflow for every project.

For me, the important thing is to keep the things I build in one place, maintain a useful history, and make it easy to continue working on a project even after leaving it untouched for months.

A few configurations, some useful aliases, and a simple commit-and-push habit have made Git a natural part of my daily development workflow.

If you maintain a personal blog, experiment with Python, build side projects, or keep updating your resume, Git can be useful for all of them—not just your day job.

These are some of the small Git tricks I use. I'm always interested in discovering more shortcuts that make everyday development simpler.