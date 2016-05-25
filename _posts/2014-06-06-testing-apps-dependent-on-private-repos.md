---
layout: post
title: "Testing Apps Dependent On Private Repos"
author: James Barr
categories: [development, tools]
tags: [android, tools, testing]
---

Testing is an important part of our development process. [Travis](https://travis-ci.com/) is an extremely helpful service that integrates with GitHub and runs our unit tests in a continous integration fashion. Often, we are developing apps for clients, and as such, we are working in a private GitHub repo. This is not an issue for us since Travis sets up public & private SSH keys in order to access the app's private repo. The issue we recently ran into, though, was when that app is dependent on another private repo, such as an SDK that we are also developing in private. Here are the steps we took to fix our CI process in a secure manner...<!--more-->

### Prepare Your App's Dependency

Make sure that your app has the private SDK repo added as a submodule that is accessible via SSH. It is important that we use SSH instead of HTTPS when we declare the submodule so that we do not need to enter in any user account or password information. Instead, we will be able to use an SSH key for a user that has access to the repos.

### Create a CI GitHub User

Create a new GitHub user that will be used by the CI machine. We will use the user's repo and collaborator permissions to limit which repos the CI machine has access to. Once the user is created, add that user to your organization with the private repos. After that, create a new GitHub team in your org called Team-CI and add your CI user as the only member of that team. The team should only have read/pull permissions. Finally, add the Team-CI as a collaborator to both the private app repo and the private SDK repo.

### Setup the CI GitHub User's SSH Key

Create an SSH key for the CI GitHub user by running: `ssh-keygen -f id_ci_github` on a Mac (or an otherwise similar command on other environments) where id_ci_github is whatever you would like your key to be named. Since we don't want to require the typing of any credentials from the CI machine in order to use this key, leave the passphrase blank when prompted. Take the public key that was created and add it to your CI GitHub user's SSH key list. On a Mac, it can be easier to copy the key using `pbcopy < ~/.ssh/id_ci_github.pub`.

### Generate a Secret for Key Encryption

We need to give Travis the private key so that it can access GitHub private repos, but in order to do this securely, we first need to encrypt it before we transfer it. We will encrypt the private key with a secret, which will also need to be provided to Travis in order to decrypt the private key. In order to generate the secret, run the command: `cat /dev/urandom | head -c 10000 | openssl sha1`. Take note of this secret value, as it will be needed later. 

### Encrypt and Upload the CI GitHub User's Private Key

In order to encrypt the private key, run: `openssl aes-256-cbc -k "YOUR_SECRET_VALUE_HERE" -in id_ci_github -out id_ci_github.enc -a`. Now that it is safely encrypted, you can upload this file to a Dropbox account to be shared with Travis. When the file is uploaded, generate a share link for the encrypted private key and hold onto this for later.

### Providing the Encrypted Private Key and Secret to Travis

In order to provide Travis with a secure link to the encrypted private key on Dropbox and the secret to be used for decryption, we can add encrypted environment variables to the Travis config file. To start off with, install the command line tool via `gem install travis`. The next commands should be run from the git root of your app. Now we can add a link to the encrypted private key by running `travis encrypt CI_GITHUB_USER_PRIVATE_KEY_URL=https://www.dropbox.com/s/SHARE_LINK_ID_HERE/id_ci_github.enc`. This will generate some secure text in the terminal window which can then be added to your Travis config file. We'll also generate some text to securely add the secret to the Travis config file by running: `travis encrypt CI_GITHUB_USER_PRIVATE_KEY_SECRET=YOUR_SECRET_KEY_HERE`. At this point, your Travis config file should have these two environment variables defined.

```yaml
env:
  global:
    # CI_GITHUB_USER_PRIVATE_KEY_URL
    - secure: "...some secure text here..."

    # CI_GITHUB_USER_PRIVATE_KEY_SECRET
    - secure: "...some more secure text here..."
```

### Decrypt the Encrypted Private Key

In order for Travis to decrypt the encrypted private key with the encrypted passphrase, we will need to run a script during the before_install phase of a Travis build. Add the following section to your Travis config file:

```yaml
before_install: bash -v travis_before_install.sh
```

When this script is run on the Travis build machine, the secure environment variables will have been already decrypted. Now, with all of this info, we can download the key from the Dropbox share URL, decrypt it using the secret, and add the private key to the machine's list of SSH keys. Once the key is added, you can then init your submodules that includes your private SDK repo. Here is how your `travis_before_install.sh` should look:

```bash
#!/bin/bash -v

# Fetch the CI GitHub user's encrypted private key
curl -L -o id_ci_github.enc "$CI_GITHUB_USER_PRIVATE_KEY_URL"

# Decrypt the CI GitHub user's private key and add it to SSH key list
openssl aes-256-cbc -k "$CI_GITHUB_USER_PRIVATE_KEY_SECRET" -in id_ci_github.enc -d -a -out id_ci_github
ssh-add -D
chmod 600 id_ci_github
ssh-add ./id_ci_github

# Init all submodules now that CI GitHub user's private key is added
git submodule update --init --recursive
```

### Wrap Up

Now that the private app repo is cloned and private SDK submodule repo is updated and initialized, your Travis build is free to continue with the rest of its build process.

