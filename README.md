# proofbyexample.com

## Setup (Ubuntu 12.04)

* `sudo apt-get install ruby1.9.3`
* `sudo apt-get install libxml2-dev libxslt-dev` (nokogiri, required by jekyll, needs these)
* `sudo gem install jekyll kramdown jekyll-s3`

## Dev

* Run `jekyll serve -w` to run a local server to work against (`-w` watches for changes)

## Deploy

* Configure `_jekyll_s3.yml` based on `_jekyll_s3.yml-template`
* Run `jekyll-s3` to push the blog to s3

## References

*  https://github.com/laurilehmijoki/jekyll-s3
*  http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html

