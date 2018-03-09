# Autogit: Reference

This repository shows how you can use Combell Autogit to deploy a project.

* [Introduction](#introduction)
* [Combell Git remote](#combell-git-remote)
  * [New repository](#new-repository)
  * [Existing folder](#existing-folder)
  * [Existing Git project](#existing-git-project)
* [Deployment](#deployment)
  * [Directory structure](#directory-structure)
  * [Deployment process](#deployment-process)
* [`.autogit.yml`](#autogityml)
  * [Composer install example](#composer-install-example)
  * [How to manage shared resources](#how-to-manage-shared-resources)
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
 
 The workflow resembles *git pushes* you perform to hosted repositories like *GitHub, GitLab, or Bitbucket*. The main difference is that **the Combell git remote's sole purpose is to perform a deployment of the code that resides within the repository**.
 
 Additional *pre-deployment and post-deployment actions* are defined in the `.autogit.yml` configuration file and can be used to customize the deployment process. 

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

Your original web root is renamed to `www_before_autogit` and symlinked to `www` prior to the first actual Autogit deployment. Once the Autogit deployment process is triggered, the `www` folder will be linked to the folder of the current release.

The exact location of the repository is shown in the **control panel**, in this case it is
`domainbe@ssh.domain.be:auto.git`.

Now we know the location of the repository, there are a couple of ways to start using it:

 * **Clone the empty repository** and use it as the default remote (origin);
 * **Start a new local repository** and use it as the default remote (origin);
 * **Use an existing repository** and add the combell remote.

### New repository

When there is no local git repository yet, just clone the combell remote **to your local computer** and start working:

    git clone domainbe@ssh.domain.be:auto.git domain.be
    cd domain.be
    touch README.md
    git add README.md
    git commit -m "add README"
    git push -u origin master
    
This will result in a fully functional local git repository and an automatic deployment of the files that were added to the repository.

**This is a great approach for brand new projects.**    

### Existing folder

When you have a project folder **on your local computer**, but not using git yet:

    cd existing_folder
    git init
    git remote add origin domainbe@ssh.domain.be:auto.git
    git add .
    git commit -m "Initial commit"
    git push -u origin master
    
These commands above will initialize a git repository in your project folder and will add files and folders that are located within the project folder.    
    
**This approach can be used for exisiting projects that aren't yet using version control.**    

### Existing Git project

When you have an existing Git project **on your local computer** (with or without existing remotes):

    cd existing_repo
    git remote add combell domainbe@ssh.domain.be:auto.git
    git push -u combell master
    
Besides pushing commits to common remotes like *GitHub, GitLab, or Bitbucket*, you'll push them to our remote.    

## Deployment

Now that we have the remote configured, we are ready to deploy our project.

Autogit uses a [Capistrano-like structure](http://capistranorb.com/documentation/getting-started/structure/)
for deployments. For every branch, a folder is created within the `checkout` folder, containing a subfolders for every release. Each release has the corresponding *commit id* as the folder name.

### Directory structure

When we push a branch to the Combell remote, a branch folder is created in the `checkout` directory.

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

### Deployment process

The deployment process runs the following steps:

 1. Receive code (via git hooks) that is triggered via a *git push* to the Combell remote
 1. Create checkout folder (`~/checkout/%branch_name%`)
 1. Export code from commit (`~/checkout/%branch_name%/%commit_id%`)
 1. Symlink shared folders/files into new release folder
 1. Symlink "current" folder to new release folder
 1. Cleanup old release folders (2 directory limit)

It is possible to hook into every step (pre/post) with hooks (using the [.autogit.yml](#autogityml) file).

After the symlink to the "current" folder is replaced, PHP-FPM is reloaded. This way the caches are cleared and the
new code can be served.

## `.autogit.yml`

When you activate Autogit, a sample `.autogit.yml` file is generated for you.

This file must exist in your repository for Autogit to work properly.

The following command will download the `autogit.yml.example` file from your hosting account to your local computer. The file is saved locally as `.autogit.yml`. 

    scp domainbe@ssh.domain.be:autogit.yml.example .autogit.yml
    git add .autogit.yml
    git commit -am "Add the default .autogit.yml template"

The following template lists all the configurable options.

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

The template above defines 10 hooks that can be used to perform custom actions. In the example above, `exit 0` is called for every hook. This does nothing more than a clean exit without executing any actual tasks.

Let's take you through the different hooks:

* `setup_before`: scripts that are executed *before* the folder structure is created
* `setup_after`: scripts that are executed *after* the folder structure is created
* `install_before`: scripts that are executed *before* the code from the latest commit is put into the newly created directory structure
* `install_after`: scripts that are executed *after* the code from the latest commit is put into the newly created directory structure
* `sharedsymlink_before`: scripts that are executed *before* a symlink is created for files that are *shared across deployments* 
* `sharedsymlink_after`: scripts that are executed *after* a symlink is created for files that are *shared across deployments*
* `symlink_before`: scripts that are executed *before* the latest release is symlinked to the *current* folder
* `symlink_after`: scripts that are executed *after* the latest release is symlinked to the *current* folder
* `cleanup_before`: scripts that are executed *before* old releases are cleaned up
* `cleanup_after`: scripts that are executed *after* old releases are cleaned up 

The [autogit-symfony-example](https://github.com/combell/autogit-symfony-example#autogityml) project has some more
specific examples on how to use the .autogit.yml file.

### Composer install example

When your code is written in PHP, chances are that you use Composer to manage your dependencies. 

The snippet below shows to use of the `install_after` hook and how you can run the `composer` binary within that hook. 

```
hooks:
  install_after: |
    composer install --no-dev --classmap-authoritative --prefer-dist
    exit 0
```

### How to manage shared resources

The goal of our Autogit implementation is to provide an easy way to deploy your code. In reality your project contains more than just the code:

* Images
* Javascript files
* CSS files
* Web fonts
* PDFs and other downloadable files

We refer to these files as *static content*, because they don't change. It is common to add template-related static files to your git repository, but user uploads are beyond your control.

In those situations, a set of *shared folders* can be used to store content that will not be overwritten by a new deployment. Instead, these files and folders will be symlinked to their target directory.

Image a situation where you have the following folders in your web root:

* `img`: stores images 
* `css`: stores your template's CSS files
* `js`: stores the Javascript files that are used in your template

If you happen to manage the content of these folders from your CMS, you don't want these folders to be overwritten by the next deployment.

In that case, you'll add the following *shared folder syntax* to your `.autogit.yml`

```
shared_folders:
  - www/img
  - www/js
  - www/css
```

Once the deployment is triggered, your directory structure will slightly change. For your branch in the `checkout` folder, there will be a `shared` folder containing the shared folders you defined.

Please upload the content for these folders to the *shared folder*

    domainbe@ssh:~# ls -l ~/checkout/master/shared/www
    drwxr-xr-x ... css
    drwxr-xr-x ... img
    drwxr-xr-x ... js
    
After the Autogit deployment, these folders will be symlinked to the appropriate target location as illustrated below:

    domainbe@ssh:~# ls -l ~/www
    lrwxrwxrwx ... css -> /data/sites/web/domainbe/checkout/master/shared/www/css
    lrwxrwxrwx ... img -> /data/sites/web/domainbe/checkout/master/shared/www/img
    -rw-r--r-- ... index.php
    lrwxrwxrwx ... js -> /data/sites/web/domainbe/checkout/master/shared/www/js

Per convention, the `www` folder of your hosting account is your webroot and is a symlink to the `www` folder of the current release.

In our case, the `img`, the `js`, and the `css` folders were defined as shared folders and symlinked.     

A more realistic situation would be to provide an `uploads` folder that is shared. This folder is used for *user uploads*, whereas the `css`, `js`, and `img` folders would be part of the deployment. This would result in the following directory layout:


    domainbe@ssh:~# ls -l ~/checkout/master/shared/www
    drwxr-xr-x ... uploads
    
The web root will then contain a symlink to the shared folder:     

    domainbe@ssh:~# ls -l ~/www
    drwxr-xr-x ... css
    drwxr-xr-x ... img
    -rw-r--r-- ... index.php
    drwxr-xr-x ... js
    lrwxrwxrwx ... uploads -> /data/sites/web/domainbe/checkout/master/shared/www/uploads


## Branching

It is possible to create subsites based on the branch name in your git repository.

Lets say you want a staging subdomain, you can create a staging branch and do pre-production deployments by pushing that
branch to the Combell remote.

    git checkout -b staging
    git push combell staging

In the output you will see the creation of the subsite and the webserver "reload" request.

The `checkout` folder will contain a subfolder for your new branch. In our case that is the `staging` branch. Just like the `master` branch, these other branches will also contain releases and shared folders.

    domainbe@ssh:~# ls -l ~/checkout/
    drwxr-xr-x ... master
    drwxr-xr-x ... staging

Now the new *http://staging.domain.be* website will be available!

This is also reflected in the directory structure of your subsites as you can see below:

    domainbe@ssh:~# ls -l ~/subsites/
    lrwxrwxrwx staging.domain.be -> /data/sites/web/domainbe/checkout/staging/current/www
    
The subsite for your `staging` branch is created and symlinked to the `www` folder for the current release of your branch.