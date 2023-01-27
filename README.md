# Spacetime Documentation

This repo contains the web app for Spacetime documentation. This app is built using [Jekyll](https://jekyllrb.com/) and hosted by [Gitlab Pages](https://docs.gitlab.com/ee/user/project/pages/). 

TODO(nihar): Add the URL where the site is hosted.

## Build the app

First, install the [Jekyll dependencies](https://jekyllrb.com/docs/installation/). Then, install Jekyll and use it to [build the app](https://jekyllrb.com/docs/installation/). The app will be running at ```localhost:4000```. 

Note: Pass the ```--livereload``` option to serve to automatically refresh the page with each change you make to the source files: ```bundle exec jekyll serve --livereload```.

## Add documentation

Add a .md file to this repo, and it will automatically be picked up in the Table of Contents and rendered on the site.

TODO(nihar): Add details about page layout and customization options.

## Directory Structure

The ```_config.yml``` file contains the main configuration for the site. Custom CSS rules are in ```_sass/color_schemas/aalyria_styling.css```, assets are in ```assets/images/spacetime-wordmark.png```, and markdown files can be uploaded to the root directory. 

TODO(nihar): Update this when we confirm a directory structure for markdown files.