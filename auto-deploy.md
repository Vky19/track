# Auto-deploying built products to gh-pages with Travis

This is a set up for projects which want to check in only their source files, but have their gh-pages branch automatically updated with some compiled output every time they push.

## Create a compile script

You want a script that does a local compile to e.g. an `out/` directory. Let's call this `compile.sh` for our purposes, but for your project it might be `npm build` or `gulp make-docs` or anything similar.

The `out/` directory should contain everything you want deployed to `gh-pages`. That almost always includes an `index.html`.

Check this script in to your project.

## Create a deploy script

Create a deploy script, call it `deploy.sh`, with the following contents: 

```bash
#!/bin/bash
set -e # Exit with nonzero exit code if anything fails

SOURCE_BRANCH="master"
TARGET_BRANCH="gh-pages"
EMAIL_TO_NOTIFY="you@example.com"

function doCompile {
  ./compile.sh
}

# Pull requests and commits to other branches shouldn't try to deploy, just build to verify
if [ "$TRAVIS_PULL_REQUEST" != "false" -o "$TRAVIS_BRANCH" != "$SOURCE_BRANCH" ]; then
    echo "Skipping deploy; just doing a build."
    doCompile
    exit 0
fi

# Save some useful information
REPO=`git config remote.origin.url`
SSH_REPO=${REPO/https:\/\/github.com\//git@github.com:}
SHA=`git rev-parse --verify HEAD`

# Clone the existing gh-pages for this repo into out/
# Create a new empty branch if gh-pages doesn't exist yet (should only happen on first deply)
git clone $REPO out
cd out
git checkout $TARGET_BRANCH || git checkout --orphan $TARGET_BRANCH
cd ..

# Clean out existing contents
rm -rf out/**/* || exit 0

# Run our compile script
doCompile

# Now let's go have some fun with the cloned repo
cd out
git config user.name "Travis CI"
git config user.email "$EMAIL_TO_NOTIFY"

# If there are no changes to the compiled out (e.g. this is a README update) then just bail.
if [ -z `git diff --exit-code` ]; then
    echo "No changes to the output on this push; exiting."
    exit 0
fi

# Commit the "changes", i.e. the new version.
# The delta will show diffs between new and old versions.
git add .
git commit -m "Deploy to GitHub Pages: ${SHA}"

# Get the deploy key by using Travis's stored variables to decrypt deploy_key.enc
ENCRYPTED_KEY_VAR="encrypted_${ENCRYPTION_LABEL}_key"
ENCRYPTED_IV_VAR="encrypted_${ENCRYPTION_LABEL}_iv"
ENCRYPTED_KEY=${!ENCRYPTED_KEY_VAR}
ENCRYPTED_IV=${!ENCRYPTED_IV_VAR}
openssl aes-256-cbc -K $ENCRYPTED_KEY -iv $ENCRYPTED_IV -in deploy_key.enc -out deploy_key -d
chmod 600 deploy_key
eval `ssh-agent -s`
ssh-add deploy_key

# Now that we're all set up, we can push.
git push $SSH_REPO $TARGET_BRANCH
```

This deploy script depends on an encrypted deploy key file, `deploy_key.enc`, which we will discuss shortly. The basic idea is that Travis stores a couple environment variables for you, not checked in to your repo, which allow the script to decrypt `deploy_key.enc`, which *is* checked in to your repo.

## Sign up for Travis and add your project

Get an account at https://travis-ci.org/. Turn on Travis for your repository in question, using the Travis control panel.

## Get encrypted credentials

The trickiest part of all this is that you want to give Travis the ability to run your deploy script and push changes to gh-pages, without checking in the necessary credentials to your repo. The solution for this is to use Travis's [encrypted file support](https://docs.travis-ci.com/user/encrypting-files/).

_NOTE: an earlier version of this guide recommended generating a GitHub personal access token and encrypting that. Although this is simpler, it is not a good idea in general, since it means any of your repository's collaborators would be able to edit the Travis build script to email them your access token, thus giving them access to all your repositories. The repository-specific deploy key approach is safer._

First, [generate a new SSH key](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/). You should _not_ reuse existing SSH keys, and you should _not_ add the SSH key to your GitHub account.

Next, add that deploy key to your repository at https://github.com/<your name>/<your repo>/settings/keys.

Now use the Travis client to encrypt the generated deploy key. The result should look something like this:

```
$ travis encrypt-file deploy_key
encrypting deploy_key for domenic/travis-encrypt-file-example
storing result as deploy_key.enc
storing secure env variables for decryption

Please add the following to your build script (before_install stage in your .travis.yml, for instance):

    openssl aes-256-cbc -K $encrypted_0a6446eb3ae3_key -iv $encrypted_0a6446eb3ae3_key -in super_secret.txt.enc -out super_secret.txt -d

Pro Tip: You can add it automatically by running with --add.

Make sure to add deploy_key.enc to the git repository.
Make sure not to add deploy_key to the git repository.
Commit all changes to your .travis.yml.
```

Make note of that encryption label, here `"0a6446eb3ae3"`. This can be public information; it just says which environment variables to use on the Travis server when decrypting this file.

You should follow the instructions and commit `deploy_key.enc` to the repository. You should also add `deploy_key` to your `.gitignore`, or delete it.

## Create your `.travis.yml` file

With all this in hand, you can create a `.travis.yml` file. It should look like this:

```yml
language: generic # don't install any environment

script: bash ./deploy.sh
env:
  global:
  - ENCRYPTION_LABEL: "<.... encryption label from previous step ....>"
```

If your compile script depends on certain environment features, you might want to set up the environment using Travis's built-in abilities, e.g. by changing the language lines like so:

```yml
language: node_js
node_js:
- stable
```

(In this case, [by default](http://docs.travis-ci.com/user/languages/javascript-with-nodejs/) Travis will install the latest stable Node.js, then run `npm install`.)

## Finishing up

At this point you should have 3-4 files checked in to your repo: `compile.sh`, `deploy.sh`, `deploy_key.enc`, and `.travis.yml`. If you've also told Travis about your repo, then the first time you push to GitHub with these changes, it will automatically compile and deploy your source!

## See it in action

I use basically this exact setup for https://domenic.github.io/zones/. The relevant files are:

- https://github.com/domenic/zones/blob/master/.travis.yml
- https://github.com/domenic/zones/blob/master/deploy.sh
- https://github.com/domenic/zones/blob/master/deploy_key.enc

(I have inlined the compile script into `deploy.sh`, so there is no separate `compile.sh`.)