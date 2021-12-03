---
layout: single
title:  "Translations management tool"
date:   2021-11-29
categories: Android Mobile
---

[Lokalise][lokalise] is a great tool for translations management. It allows
you to store all application translations in one place, simplifying interaction with
translators, automate the reception of translations and the transfer of new strings
for translation.

In our project, we adhere to the following flow to implement integration
with [Lokalize][lokalise]:
1. The developer transfers new strings from the design to the code and puts 
them into a file `strings_not_translated.xml`
2. The developer sends new not translated strings to the manager
3. The manager upload new strings manually to [Lokalize][lokalise]
4. When the new strings are fully translated, the manager creates a task for the 
developer to write them into the code
5. The developer manually writes translations into the code and removes the 
corresponding strings from `strings_not_translated.xml`

The bad thing about this flow is that it is not executed automatically, but 
manually. Stages 2, 3, 4, 5 could be automated, completely eliminating the 
human factor.

[Lokalize][lokalise] offers automation of translation implementation using two
tools:
- [Lokalise SDK][lokalise-sdk] - it allows you to receive translations in 
the Runtime of the client application
- [Lokalise VCS service integrations][lokalise-integrations] - it allows to
integrate [Lokalize][lokalise] into the VCS service

Both options are not suitable for our project. [Localise SDK][lokalise-sdk]
complicates the Runtime of the application. It could happen some lines might 
not be translated in production.
Using [Localize VCS service integration][lokalise-integrations] looks
like a reliable and preferred option. However, our VCS service is only available
under a corporate VPN, which makes it impossible to use any integration
based on webhooks.

[Lokalize][lokalise] provides an [open API][lokalise-api] with
which we can download translations and upload new strings for translation. That is, instead
of steps 2 and 3 of our flow, we can automatically upload new lines for translation, and instead
of steps 4 and 5, we can automatically download and apply new translations
to the code. Then the flow would look like this:

1. The developer transfers new strings from the design to the code and puts them into a file
`strings_not_translated.xml`
2. New strings for translation will be automatically uploaded to [Lokalize][lokalise]
in CI
3. New translations will be automatically downloaded and applied to the code base in CI

We didn't find a tool that would work with the lines of the codebase and depend only on
[Localise API][lokalise-api], so we started developing our own.

## Translations pull script
At the first stage, we minimally affected our current flow and implemented
the ability to download new translations and apply them to the codebase. That is, if
earlier the programmer applied new translations with his hands after creating the corresponding
task by the manager, now it will be enough for him to run a script that
will perform this routine work automatically. Also, the developer can ignore
the new script and continue to apply new translations in the old way.

The following diagrams describe how the script works.
State before the `translationspull` script was run:
![How it works. Before pull][how-it-works-before-pull]

State after running the `translationspull` script:
![How it works. After pull][how-it-works-after-pull]

The string `button_save` that was in `strings_not_translated.xml` was completely
translated, so it was removed from `strings_not_translated.xml `, and
its translations were transferred to the corresponding string files. The string `button_close`
has not been fully translated yet, so it has remained untouched.


## Gradle plugin
For easy integration of the translation management tool,
we created the [translations][translations-gradle-plugin] Gradle plugin.
Setting it up is quite simple:

```groovy
translations {
  api {
    lokalise {
      apiToken = "<your lokalise api token>"
      projectId = "<your lokalise project id>"
    }
  }
  supportedLanguages = new SupportedLanguages(['es'])
  notTranslatedFile = file("./src/main/res/values/strings_not_translated.xml")
  resourcesFolder = file("./src/main/res/")
}
```
when 
- `translations.api.lokalise.apiToken` is the [Lokalise API token][lokalise-api-token]
- `translations.api.lokalise.projectId` is the [Lokalise project unique ID][lokalise-projects]
- `supportedLanguages` is the list of supported language codes by your app
- `notTranslatedFile` is the file with new not translated strings

It implements the ability to download (using the [download files API][lokalise-download-files-api]) 
and apply new translations with the task `translationsPull`:
```shell
./gradlew translationsPull
```

Plugin source code is [available on GitHub][translations-gradle-plugin]!

## Next steps
We will continue to automate the work with translations in our project. The next 
step will be creating a `translationsPush` script that will automatically
send new translations to the [Lokalize][lokalise].

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
