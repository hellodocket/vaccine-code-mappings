# Vaccine Code Mappings

This package contains a Rake task that will scrape the CDC exports of CVX and CPT vaccine codes and generate a JSON file mapping these codes to vaccine information.

## Running the Rake Task

You can run the task to generate the actual mapping with the following

```
bundle install
bundle exec rake load_cdc_mapping
```

## Creating a Release

Releases and updates to the vaccine-code-mapping.json file are performed daily by a Github action.
