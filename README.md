# proofbyexample.com

## Setup

* `sudo apt-get install libxml2-dev libxslt-dev` (nokogiri, required by jekyll, needs these)
* Install rvm: https://rvm.io/
* `rvm install 1.9.3`
* `rvm use 1.9.3`
* `gem install jekyll kramdown jekyll-s3`

## Dev

* Run `jekyll` (`_config.yml` should have all the right settings) to have a local server to work against

## Deploy

* Configure `_jekyll_s3.yml` based on `_jekyll_s3.yml-template`
* Run `jekyll-s3` to push the blog to s3

## References

*  https://github.com/laurilehmijoki/jekyll-s3
*  http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html

