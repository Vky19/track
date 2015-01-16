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
set -e # exit with nonzero exit code if anything fails

# clear and re-create the out directory
rm -rf out || exit 0;
mkdir out;

# run our compile script, discussed above
./compile.sh

# go to the out directory and create a *new* Git repo
cd out
git init

# inside this git repo we'll pretend to be a new user
git config user.name "Travis CI"
git config user.email "<you>@<your-email>"

# the first and only commit to this new Git repo contains all the files present with the commit message "Deploy to GitHub Pages"
git add .
git commit -m "Deploy to GitHub Pages"

# force push from the current repo's master branch to the remote repo's gh-pages branch
# (all previous history on the gh-pages branch will be lost, since we are overwriting it)
# redirect any output to /dev/null to hide any sensitive credential data that might be exposed
git push --force --quiet "https://${GH_TOKEN}@${GH_REF}" master:gh-pages > /dev/null 2>&1
```

You can run this deploy script locally if you like, with

```
GH_TOKEN=<your password> GH_REF=github.com/<your name>/<your repo>.git ./deploy.sh
```

But that's no fun. We want this to happen automatically every time you push. To do that, we'll use Travis CI.

## Sign up for Travis and add your project

Get an account at https://travis-ci.org/. Turn on Travis for your repository in question, using the Travis control panel.

## Get encrypted credentials

The trickiest part of all this is that you want to give Travis the ability to run your deploy script and push changes to gh-pages, without checking in the necessary credentials to your repo. The solution for this is to use Travis's [encryption support](http://docs.travis-ci.com/user/encryption-keys/).

We'll generate a GitHub personal access token (essentially an application-specific password) and encrypt it, then put the encrypted version in our `.travis.yml` file. Then we can check in the `.travis.yml` file with no issues.

First, generate a token at https://github.com/settings/applications

Then, install the Travis client and do

```
travis encrypt GH_TOKEN=<secret token here>
```

This will give you a very long line like

```
secure: "<.... encrypted data ....>"
```

## Create your `.travis.yml` file

With all this in hand, you can create a `.travis.yml` file. It should look like this:

```yml
script: bash ./deploy.sh
env:
  global:
  - GH_REF: github.com/<your name>/<your repo>.git
  - secure: "<.... encrypted data from above ....>"
```

## Finishing up

At this point you should have 2-3 files checked in to your repo: `compile.sh`, `deploy.sh`, and `.travis.yml`. If you've also told Travis about your repo, then the first time you push to GitHub with these changes, it will automatically compile and deploy your source!

## See it in action

I use basically this exact setup for http://www.w3.org/2001/tag/doc/promises-guide (which is hosted on gh-pages; the w3.org URL is a proxy to that). The relevant files are:

- https://github.com/w3ctag/promises-guide/blob/master/.travis.yml
- https://github.com/w3ctag/promises-guide/blob/master/deploy-gh-pages.sh

(I have inlined the compile script into the latter, so there is no separate `compile.sh`.)