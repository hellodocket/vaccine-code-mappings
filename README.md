# super-duper-giggle

This package contains a Rake task that will scrape the CDC exports of CVX and CPT vaccine codes and generate a JSON file
mapping these codes to vaccine information.

## Running the Rake Task

You can run the task to generate the actual mapping with the following

```
bundle install
bundle exec rake load_mapping
```

## Creating a Release

The repository contains a github actions that creates releases when a new tag is pushed. 

```
git tag v1.0.0
git push origin v1.0.0
```
