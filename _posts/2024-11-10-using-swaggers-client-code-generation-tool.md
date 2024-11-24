---
layout: post
title: Using Swagger's client code-generation tool
tags: ruby
---

[Swagger Codegen](https://github.com/swagger-api/swagger-codegen) is a neat project than can consume an OpenAPI specification and generate the code required for an API client in a variety of languages. Note that both 2.x and 3.x versions of the tool exist, and both are being maintained. I had more luck with 2.x.

## Installation

On a Mac, this is trivial with HomeBrew:

```bash
brew install swagger-codegen@2
brew link swagger-codegen@2
```

## Example usage

As an example, we could use the tool to generate a Ruby API client for the [Strava API](https://developers.strava.com/playground):

```bash
mkdir strava-client-rb
cd strava-client-rb
```

The generators can take language-specific configuration (see e.g. `swagger-codegen config-help --lang ruby`), so we'll prepare a JSON file with some Ruby overrides:

```javascript
// Contents of codegen-config.json
{"gemName":"strava_client", "moduleName":"StravaClient", "gemVersion":"0.0.1"}
```

Before generating any code, we'll get our manually-written configuration under version control:

```bash
git init
git add codegen-config.json
git commit -m "Add Ruby config for swagger-codegen"
```

Now for the main event! Let's generate some code, using [Strava's Swagger docs](https://developers.strava.com/swagger/swagger.json):

```bash
swagger-codegen generate \
  --input-spec https://developers.strava.com/swagger/swagger.json \
  --lang ruby \
  --config codegen-config.json \
  --output .
```

This generates classes for each endpoint and resource found in the OpenAPI specification, as well as additional wrapping client code and even some skeleton specs!

We can install the dependencies:

```bash
bundle install

# We don't want to version the lockfile:
sed -i '' 's/# Gemfile.lock/Gemfile.lock/' .gitignore

git add .
git commit -am "Output of 'swagger-codegen generate'"
```

...and then take it for a spin (`irb -Ilib -r strava_client`):

```ruby
# Once you're acquired an access token, configure the library:
StravaClient.configure do |config|
  config.base_url #=> => "https://www.strava.com/api/v3"
  config.access_token = "your-access-token-here"
end

# Auto-generated definitions are already present:
StravaClient::ActivityType.constants.grep /ride/i
#=> [:E_BIKE_RIDE, :RIDE, :VIRTUAL_RIDE]

# Reading some data (note that 'read:activities' scope will be required here):
activities = StravaClient::ActivitiesApi.new
activities.get_logged_in_athlete_activities
#=> [StravaClient::SummaryActivity, StravaClient::SummaryActivity, ...]
```
