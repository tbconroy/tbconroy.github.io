---
layout: post
title:  "Deploying a Jekyll Site on Github Pages with Travis CI"
date:   2017-09-09 17:44:06 -0400
categories:
  - jekyll
  - github pages
  - travis ci
  - ruby
---
I use Github Pages to host this blog because it is free with the basic Developer subscription. Jekyll is a static site generator that is supported by Github Pages. To create a personal site normally all you need to do is create a repo called `<your_github_username>.github.io` and commit your Jekyll project directory to that that repo and all the html and css your site needs will be built and served at your GitHub Pages url. You can even alter the apperance of your site without editing a CSS file using one of a few supported Jekyll themes that Github Pages [supports](https://help.github.com/articles/creating-a-github-pages-site-with-the-jekyll-theme-chooser/).

However, there are plenty more themes out there that are not supported by Github Pages, many of which are packaged in Ruby gems so that they can [easily be added](http://jekyllrb.com/docs/themes/#understanding-gem-based-themes) to your Jekyll generated site. If you use an gem based themes unsupported by Github Pages, as I have, simply committing your Jekyll project directory no longer automatically builds and publishes site. You must either:
1. copy the relevant theme CSS and templates into your project and lose the ability to easily incorporate updates to the theme from it’s maintainer using `bundle update`
2. or manually build the site locally and commit the generated files to the `master` branch of your `<your_github_username>.github.io` repo.

For this blog I use a variation on the second option, but instead of building my site locally and committing it myself, I use Travis CI to automate this whenever I push changes to a `source` branch. Essentially `source` contains the Jekyll project directory, and the `master`contains the Jekyll generated html and css. It is in this `master` branch where GitHub Pages is [configured to look](https://help.github.com/articles/user-organization-and-project-pages/#user--organization-pages) for the resources to publish your site.

Here are the steps to set this up.

## Create source branch
The `source` branch will be the place where your Jekyll project directory is committed. You will add and commit any additional posts or content here. It is from these branch that the site files will be generated. It is a good idea to make this your default branch for the repo.

## Add unsupported gem theme to your Jekyll project
For this site I decided to use the [whiteglass theme](https://github.com/yous/whiteglass), but you can choose whichever theme you like.

Add the theme to your project’s `Gemfile`, in my case:
```ruby
gem "jekyll-whiteglass"
```
Add the theme and its plugins to `_config.yml`:
```yml
theme: jekyll-whiteglass
```
Then install the gem by running the following in the console:
```bash
$ bundle install
```
Commit all changes, including those to the `Gemfile.lock`.

## Install and enable Travis CI
Travis CI is free to use on public repos and has deployment support for GitHub Pages built in! But first you must create a Travis CI account and [enable](https://docs.travis-ci.com/user/getting-started/#To-get-started-with-Travis-CI) it for your `<your_github_username>.github.io` repo, and [generate an GitHub personal access token](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/) so that Travis has permission to automatically commit the built site to master. This token should have the `public_repo` scope enabled. Keep this token safe and secrete, don’t commit it!

For this step you will need install the Travis CLI by running:
```bash
$ gem install travis
```
Login using the personal access token that you generated:
```bash
$ travis login --github-token <your_generated_github_access_token>
```
Then in your project directory type the following in the console:
```bash
$ travis init
```
Follow the prompts and a `.travis.yml` will be added to your project directory. It is in this file that you will configure Travis to build and deploy your site.

Next you will create an encrypted environment variable using the GitHub personal access token and add it to the `.travis.yml`. In the console type:
```bash
$ travis encrypt GITHUB_TOKEN=”<your_generated_github_access_token>” --add
```
Your `.travis.yml` file should look something like this:
```yml
language: ruby
rvm:
  - 2.3.4
env:
  global:
    secure: <encrypted_github_token>
```
Note: make sure you only have one ruby version listed, or Travis will create builds for each version every time you commit!

## Add Jekyll build rake task
When you commit your Jekyll project to the `source` branch, Travis CI will need a way to build the site. For Ruby projects the [default build command](https://docs.travis-ci.com/user/languages/ruby/#Default-Build-Script) is `rake`, so it makes sense to add a default rake task to your project that does just this.

Add a `Rakefile` to the project root containing the following:
```ruby
task :default => [:jekyll_build]

task :jekyll_build do
  `bundle exec jekyll build`
end
```

When travis runs the `rake` command this will run `jekyll build` and generate all the files needed for you site in the `_site` directory in your project directory.

## Configure Travis CI to build and deploy site
Now comes the fun part. As I mentioned before Travis [supports deployments to GitHub Pages](https://docs.travis-ci.com/user/deployment/pages/). To do this add the following to your `.travis.yml`:
```yml
deploy:
  provider: pages
  skip_cleanup: true
  github_token: "$GITHUB_TOKEN"
  local_dir: _site
  target_branch: master
  on:
    branch: source
```

This deploy process runs after Travis builds the `_site` directory. What is does is allows Travis to automatically force commit the generated site files, from the `_site` directory, to `master` from the `source` branch. If you notice the env variable `$GITHUB_TOKEN` references the personal access token that you encrypted and added to the `.travis.yml`.

Now every time commit and push a change to the `source` branch, Travis CI will automatically build and force commit your generated html and css files to `master` which then deploys your updated site to GitHub Pages!

## Going further
Since you are using Travis CI it doesn’t hurt to add tests as part of the build process! These should obviously run and pass before the deploy process. I wouldn’t recommend doing anything too crazy here. The Jekyll maintainers [recommend](http://jekyllrb.com/docs/continuous-integration/travis-ci/) using [html_proofer](https://github.com/gjtorikian/html-proofer) to check your generated html.
