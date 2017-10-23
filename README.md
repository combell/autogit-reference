# Autogit: Hello World

This repository shows how you can use Combell Autogit to deploy a project.

* [Introduction](#introduction)
* [Combell Git remote](#combell-git-remote)
  * [New repository](#new-repository)
  * [Existing folder](#existing-folder)
  * [Existing Git project](#existing-git-project)
* [Deployment](#deployment)
* [`.autogit.yml`](#autogityml)
* [Branching](#branching)

## Introduction

All software projects benefit from a system like Git. Git is used to
 [track changes of code over time](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control).
You keep track of changes by [performing commits](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository)
 and some commits contain new features or bugfixes.

Since your Git repository now contains new versions of your project after commits, you can use it to
 [create releases](https://git-scm.com/book/en/v2/Git-Basics-Tagging).

With the Autogit feature of Combell, deploying is as simple as [pushing your commits](https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes)
 to a repository hosted on the [Combell servers](https://www.combell.com/en/hosting/web-hosting).

## Combell Git remote

To start using Autogit, we need to register the Combell remote to our local Git repository.

In a normal situation, the directory structure in your SSH home directory looks like this:

    domainbe@ssh:~# ls -l
    total 28
    drwxrwxr-x ... cgi-bin
    drwxrwxr-x ... data
    drwxrwxr-x ... logs
    drwxrwxr-x ... php
    drwxrwxr-x ... subsites
    drwxrwxr-x ... tmp
    drwxrwxr-x ... www

Once you activate Autogit in the control panel, your home directory gets rearranged a bit.

    domainbe@ssh:~# ls -l
    total 40
    drwxr-xr-x ... auto.git
    -rw-r--r-- ... autogit.yml.example
    drwxrwxr-x ... cgi-bin
    drwxr-xr-x ... checkout
    drwxrwxr-x ... data
    drwxrwxr-x ... logs
    drwxrwxr-x ... php
    drwxrwxr-x ... subsites
    drwxrwxr-x ... tmp
    lrwxrwxrwx ... www -> /data/sites/web/domainbe/www_before_autogit
    drwxrwxr-x ... www_before_autogit

In the `auto.git` directory you find a bare git repository you can use as a new remote for your project.

The exact location of the repository is shown in the **control panel**, in this case it is
"domainbe@ssh.domain.be:auto.git".

Now we know the location of the repository, there are a couple of ways to start using it:

 * **Clone the empty repository** and use it as the default remote (origin);
 * **Start a new local repository** and use it as the default remote (origin);
 * **Use an existing repository** and add the combell remote.

### New repository

When there is no repository yet, just clone the combell remote and start working:

    git clone domainbe@ssh.domain.be:auto.git domain.be
    cd domain.be
    touch README.md
    git add README.md
    git commit -m "add README"
    git push -u origin master

### Existing folder

When you have a project folder, but not using git yet:

    cd existing_folder
    git init
    git remote add origin domainbe@ssh.domain.be:auto.git
    git add .
    git commit -m "Initial commit"
    git push -u origin master

### Existing Git project

When you have an existing Git project (with or without existing remotes):

    cd existing_repo
    git remote add combell domainbe@ssh.domain.be:auto.git
    git push -u combell master

## Deployment

Now that we have the remote configured, we are ready to deploy our project.

Autogit uses a [Capistrano-like structure](http://capistranorb.com/documentation/getting-started/structure/)
for deployments. For every branch, the latest commit id is used as a release.

When we push to the Combell remote, a branch folder is created in the `checkout` directory.

    domainbe@ssh:~# ls -l
    total 36
    drwxr-xr-x ... auto.git
    -rw-r--r-- ... autogit.yml.example
               ...
    drwxr-xr-x ... checkout
               ...
    lrwxrwxrwx ... www -> /data/sites/web/domainbe/checkout/master/current/www

Within that branch folder, the release folder is created (using the latest commit id).

    domainbe@ssh:~# ls -l checkout/master/
    total 8
    drwxr-xr-x ... 9af6443805ea8433df64c5bb14fd7e3bc9d01d92
    drwxr-xr-x ... before_autogit
    drwxr-xr-x ... shared
    lrwxrwxrwx ... current -> /data/sites/web/domainbe/checkout/master/9af6443805ea8433df64c5bb14fd7e3bc9d01d92
    
 * **checkout/{branch}** holds all deployments in a commit id folder. These folders are the target of the current symlink.
 * **checkout/{branch}/current** is a symlink pointing to the latest release. This symlink is updated at the end of a successful deployment. If the deployment fails in any step the current symlink still points to the old release.
 * **checkout/{branch}/shared** contains the **shared_files** and **shared_folders** which are symlinked into each release. This data persists across deployments and releases. It should be used for things like database configuration files and static and persistent user storage handed over from one release to the next.

The deployment process runs the following steps:

 1. Receive code (via git hooks)
 1. Create checkout folder (`~/checkout/%branch_name%`)
 1. Export code from commit (`~/checkout/%branch_name%/%commit_id%`)
 1. Symlink shared folders/files into new release folder
 1. Symlink "current" folder to new release folder
 1. Cleanup old release folders (2 directory limit)

It is possible to hook into every step (pre/post) with hooks (using the [.autogit.yml](#autogityml) file).

After the symlink to the "current" folder is replaced, PHP-FPM is reloaded. This way the caches are cleared and the
new code can be served.

## `.autogit.yml`

When you activate Autogit, a sample autogit.yml file is generated for you.

This file must exist in your repository for Autogit to work properly.

    scp domainbe@ssh.domain.be:autogit.yml.example .autogit.yml
    git add .autogit.yml
    git commit -am "Add the default .autogit.yml template"

The template lists all the configurable options.

    #############################################################
    # Autogit - config and deploy hooks                         #
    #############################################################
    # This YAML file, named as ".autogit.yml" should be present #
    # in the root folder of your codebase                       #
    #############################################################
    
    # Example shared files and folders:
    # shared_files: [ etc/config.yml ]
    # shared_folders: [ var/log ]
    
    # Hooks get executed at different stages during deploy
    hooks:
        # SETUP: Create folder structure for newest release
        setup_before: |
            exit 0
        setup_after: |
            exit 0
    
        # INSTALL: Put code in release folder
        install_before: |
            exit 0
        install_after: |
            exit 0
    
        # SHAREDSYMLINK: Create symlink to shared files and folders
        #                present at every release (config, logs, ...)
        sharedsymlink_before: |
            exit 0
        sharedsymlink_after: |
            exit 0
    
        # SYMLINK: Set current symlink to newest release
        symlink_before: |
            exit 0
        symlink_after: |
            exit 0
    
        # CLEANUP: Cleanup old releases, two most recent releases remaining
        cleanup_before: |
            exit 0
        cleanup_after: |
            exit 0

The [autogit-symfony-example](https://github.com/combell/autogit-symfony-example#autogityml) project has some more
specific examples on how to use the .autogit.yml file.

## Branching

It is possible to create subsites based on the branch name in your git repository.

Lets say you want a staging subdomain, you can create a staging branch and do pre-production deployments by pushing that
branch to the Combell remote.

    git checkout -b staging
    git push combell staging

In the output you will see the creation of the subsite and the webserver "reload" request.

Now the new http://staging.domain.be website will be available!
