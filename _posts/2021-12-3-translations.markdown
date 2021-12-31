---
layout: single
title:  "Localization Automation with Lokalise platform. Part 1"
date:   2021-11-29
categories: Android Mobile
---

This is the first in a series of blogs where we will discuss/describe how we approach the localization of our mobile projects here at Intermedia. We are actively working on a number of improvements right now, so look for more posts on mobile apps localization in the weeks and months to come!

When we started working on localizing our apps for multiple new countries we chose [Lokalise][lokalise] as the localization platform for our mobile projects. It provides a convenient way to store and manage all application translations, greatly simplifies interaction with translators and offers multiple ways to automate localization strings upload and download process.

For the purposes of this post, we use our Unite Android app to demonstrate our localization setup.

We initially chose the most straightforward way to work with Lokalise:
1. The developer added new strings that required translation to `strings_not_translated.xml` file
2. The developer sent new strings to manager
3. The manager reviewed and uploaded new strings manually to Lokalise
4. When the new strings were translated, the manager created a JIRA task for the developer to integrate them back into Unite project
5. The developer manually added translations into the code and removed corresponding strings from `strings_not_translated.xml`

As you might guess, this wasn’t/isn’t a convenient approach as there is zero automation in the above described flow. Stages 2, 3, 4, and 5 should be able to be automated which would in turn greatly increase process efficiency and eliminate the human factor.

The Lokalise platform provides two main tools for translation automation:
- [Lokalise SDK][lokalise-sdk] - allows you to update app localization without redeployment to store
- [Lokalise VCS service integrations][lokalise-integrations] - allows you to integrate Lokalise directly with your version control system

But unfortunately, both options were not suitable for our project as we didn’t want to add new third-party dependencies to our app. So we decided to opt-out from using the client-side SDK. Using Lokalize VCS service integration looked like a suitable option for us but our VCS service is only available under a corporate VPN which makes it impossible to use any webhooks-based integrations.

Fortunately, Lokalise also provides an open API that allows you to upload and download localizable strings to and from their platform. With this API we could automate steps 2 and 3 from the above-described flow thus enabling automatic upload of new strings to the service - and we also could automate steps 4 and 5 for downloading and applying translated strings back to our code base.

The updated flow can now looks like this:
1. Developer adds new strings for translation to `strings_not_translated.xml` file
2. New strings are automatically uploaded to Lokalise using our CI setup
3. New translations are automatically downloaded and applied to the code base via CI

## Translations pull script
Next, we wanted to have a simple and easy to use tool to parse our strings xml files and to communicate with the Lokalise API. To do this, we developed our own solution.

We started with download part of the translation flow. So we've implemented the ability to download new translations from Localise and apply them to our codebase.
This alone made the life of our developers a bit easier as it eliminated the manual work of copying and pasting translated strings into our resource files.
Now the developer could just run a script that performed these routine tasks. However, when required, the developer could still update particular strings manually.

The following diagrams describe how this script works:
State before the `translationspull` script was run:
![How it works. Before pull][how-it-works-before-pull]

State after running the `translationspull` script:
![How it works. After pull][how-it-works-after-pull]

The string `button_save` that was originally placed in `strings_not_translated.xml` was completely
translated so it was removed from `strings_not_translated.xml ` file and
its translations were copied to the corresponding localized strings files. The string `button_close`
had not been fully translated yet so it has been retained in `strings_not_translated.xml `.


## Translations Gradle plugin
For easy integration of the translation script into our Unite project we created [Translations Gradle plugin][translations-gradle-plugin].
Setting it up is quite simple:

```groovy
translations {
  api {
    lokalise {
      apiToken = "<your Lokalise api token>"
      projectId = "<your Lokalise project id>"
    }
  }
  supportedLanguages = new SupportedLanguages(['es'])
  notTranslatedFile = file("./src/main/res/values/strings_not_translated.xml")
  resourcesFolder = file("./src/main/res/")
}
```
where
- `translations.api.lokalise.apiToken` is the [Lokalise API token][lokalise-api-token]
- `translations.api.lokalise.projectId` is the [Lokalise project unique ID][lokalise-projects]
- `supportedLanguages` is the list of supported language codes by your app
- `notTranslatedFile` is the file with new strings to be translated

The plugin supports downloading (using [download files API][lokalise-download-files-api]) 
and applying new translations to the code base using task `translationsPull`:
```shell
./gradlew translationsPull
```

As with all the tools we mention in our blog this one is also available as open-source under MIT license from our [GitHub page][translations-gradle-plugin]. Feel free to play around with it, fork it and apply to your projects. We'd love to hear about your experiences using it!

## Next steps
We continue to automate work with localization of our projects. The next 
step will be creating a `translationsPush` script that will automatically
upload new translations to Lokalise.

That and more are coming soon…so stay tuned!

[lokalise]: https://lokalise.com/
[lokalise-integrations]: https://docs.lokalise.com/en/collections/652240-apps#source-code-and-version-control
[lokalise-sdk]: https://docs.lokalise.com/en/articles/1400668-lokalise-android-sdk
[lokalise-api]: https://app.lokalise.com/api2docs/curl/
[lokalise-api-token]: https://docs.lokalise.com/en/articles/1929556-api-tokens
[lokalise-projects]: https://docs.lokalise.com/en/articles/1400460-projects
[lokalise-download-files-api]: https://app.lokalise.com/api2docs/curl/#transition-download-files-post
[translations-gradle-plugin]: https://github.com/intermedia-net/translations

[how-it-works-before-pull]: {{ site.url }}/assets/translations/how_it_works_before_pull.png
[how-it-works-after-pull]: {{ site.url }}/assets/translations/how_it_works_after_pull.png
