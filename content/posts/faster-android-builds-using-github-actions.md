---
title: "Faster Android builds using Github Actions"
date: 2021-04-22T19:00:00+01:00
draft: false
---

Recently I had to migrate an Android project from TeamCity to Github Actions and I felt like the builds were way slower despite using the same gradle optimizations.

By analyzing the build logs I could see that my Github Actions job was not being able to leverage on most of the gradle's cache, causing it to download gradle on every workflow execution and then build from a clean slate. What a waste! 😱


## The solution 

To speed up your Github Actions job builds by taking advantage of gradle's cache, use the [`actions/cache@v2`](https://github.com/actions/cache) action. Like this:

```
- uses: actions/cache@v2
  with:
	path: |
		~/.gradle/caches
		~/.gradle/wrapper
	key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
	restore-keys: |
		${{ runner.os }}-gradle-
```

Basically it will be able to cache `~/.gradle/caches` and `~/.gradle/wrapper` as long as the `*.gradle` and `gradle-wrapper.properties` files stay untouched.
Use this caching step right before triggering the gradle build command, to make sure that the gradle cache is available right when it's needed 🤗

## Working sample 

Here's how it looks integrated in a workflow that runs the unit tests on every pull request targeting the `master` branch:
```
name: Run tests on pull requests

on:
  pull_request:
    branches: [ master ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Gradle Wrapper Validation
      uses: gradle/wrapper-validation-action@v1
      
    - uses: actions/cache@v2
      with:
        path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
        key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
        restore-keys: |
            ${{ runner.os }}-gradle-
                 
    - name: Test the debug build variant
      run: ./gradlew testDebug
```


The first build after this change will still be slow but the following workflow executions will take advantage of the existing cached resources 🚀