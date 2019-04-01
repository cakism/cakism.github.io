---
layout: post
title:  "Implementing Google Cloud Runtime Config in Kotlin"
image: ''
date:   2019-03-05 11:15
tags:
- Kotlin, GCP
description: ''
categories:
- Kotlin
---

Introduction 
-------------

GCP has as beta service called Cloud Runtime Configuration. It's a simple key-value storage inside the Google Cloud which is accessible by the deployment manager, the gcloud command-line tools and with the supplied API. There are Client API Libraries for Java, Python and Node.js, but you can of course query the API with good ol' HTTP Requests as well. It can be used for a wide variety of things of course, but traditionally you might want to use if for any sort of dynamic configurations
you have in your application, so any type of configuration that you might want to change without having to redeploy your application. 

This post will explain how to create a repository and write a client in Kotlin to perform operations on that repository. 

The prerequisites to follow this guide are:

*   Kotlin
*   Gradle (Maven works too)
*   A Google Cloud account with a project set-up.
*   [gcloud command-line tools](https://cloud.google.com/sdk/docs/quickstarts)

### First things first

First of all, make sure that you have created a project in GCP and have the gcloud command-line tools installed in your terminal of choice. I won't go through how to create a project as that's described in the installation notes for the gcloud tools. 

We're gonna start out by using the gcloud tools to create a new config repository and dip our toes in the service by creating a new variable, getting it and updating it through the CLI before moving on to the actual Kotlin client. 

**If you're just here for the Kotlin-stuff you can Jump straight to the Kotlin implementation!**

While you're still inside the Cloud Developer Console, search for [`Cloud Runtime Configuration API`](https://console.cloud.google.com/apis/api/runtimeconfig.googleapis.com/overview) in the search box and enable the API. That's the first step to being able to actually use it so great job this far!

### Getting started with the `gcloud tools`

Now, let's try to create a runtime configuration store inside your project where we'll keep the key-value objects. 

The base command for everything Cloud Runtime Configuration-related within the gcloud tools is   
`gcloud beta runtime-config` and you can always add `--help` to your command to see available commands, flags and similar.

#### Create a config store

To create a config store within our project we'll run:  
`gcloud beta runtime-config configs create [CONFIG_NAME] --description [DESCRIPTION]`  
so for example:  
`gcloud beta runtime-config configs create myConfigStore --description "This is where I put my cool, unstable variables"`

Now, because this service is still in beta you're gonna be asked to install the gcloud beta commands. Go through the installer and run the command again. You'll get a response with the URL to your freshly minted config store, probably looking something like this:   
`https://runtimeconfig.googleapis.com/v1beta1/projects/[my-project]/configs/myConfigStore`

And that's it! Your store is created and now you're ready to create your first variable.

#### Create your first variable

OK so lets continue with gcloud for now and use it to create a variable. Here's what we'll do:  
`gcloud beta runtime-config configs variables set foo \ "bar" --config-name myConfigStore --is-text`  
If we dissect this command there's a few things to point out. Up until "configs" it's the same as the previous command we ran to create the store, and then we use the "variables" keyword to signify that we'll modify the variables. We'll use `set` to create a value with the key "foo" with the value "bar". The key and value are separated by a backslash. Then we specify which config store we want to store our variable in and use the flag `--is-text` since our value is just a string. Cloud
Runtime Config at the time of writing supports strings or Base64 encoded values. 

Remember, you can always call `gcloud beta runtime-config configs variables --help` to see a list of all available commands for a specific keyword!

Running the above command will generate a response something like this in our terminal:

`Created [https://runtimeconfig.googleapis.com/v1beta1/projects/[my-project]/configs/myConfigStore/variables/foo].`

#### Getting variables

Now, let's see if we can get it out as well. The syntax is very similar.   
`gcloud beta runtime-config configs variables get-value foo --config-name myConfigStore`

This will hopefully print `bar` just below! 

We'll try one more thing before getting started with our Kotlin code, to show a typical use-case.

#### Listing all values

This is handy if you wanna check out what values you have stored so far:  
`gcloud beta runtime-config configs variables list --config-name myConfigStore --values`

This will return a list of all values which you can `get` the value of. There are different permissions though and you can also have `list` permission. To see all values, including those where you only have the permission to `list` the variables, just omit the last `--values` flag!

### Kotlin time!

So basically what we will do is to create a serverside app in Kotlin that will talk to our GCP configstore for some good old fashioned CRUD-functionality on our variables.

To achieve this we will use Google's Java library for Runtime Configurator, authorized with a service account over OAUTH2.  Doesn't sound very complicated does it?

#### Getting pre-authorized

Before we get coding, let's create a service account to be used in our application for authorization. 

1\. Go to [https://console.cloud.google.com/iam-admin/serviceaccounts](https://console.cloud.google.com/iam-admin/serviceaccounts) and choose your project. In the top you can see "Create service account" where you will be asked for a name and ID, which preferably can be the same. If you want you can enter a description as well. 

2\. Press next, where you need to fill in what permissions your service account should hold. Here you'll want to add Cloud Runtime Configuration to make sure that your service account can talk to the config store. 

3\. Then press next, and in the final step, press Create Key to download your newly forged service account credentials as a JSON-file. Keep this file safe as it's all that's needed to authorize anyone to read your variables from the config store! Create a new folder within your `resources` folder and put your credentials file there.

#### Our application

We should only need a single dependency for our project, the Cloud Runtime Configuration Java lib. Add the following line to your build.gradle.kts

`compile("com.google.apis:google-api-services-runtimeconfig:v1beta1-rev458-1.25.0")`

You might wanna check if there's a **later version**, or how to add our dependency with **Maven**,  over on the [Google API Client Lib page](https://developers.google.com/api-client-library/java/apis/runtimeconfig/v1).

Next, let's create a class where we'll have our logic. Let's call it **RuntimeConfigService.kt**

Let's start of our class with some constants for the URL that we'll use for our CRUD-calls, don't forget to input your project name:

`private val PROJECT = "**YOUR-PROJECT-NAME-HERE**"`  
`private val CONFIG = "myConfigStore"`  
`private fun getVariablesPath(key: String)= "projects/$PROJECT/configs/$CONFIG/variables/$key"`

Now we can use getVariablesPath to get a nice URL for our variables!

Next we'll do the authorization function that will use the service account json we put in our resources in the previous step. Again, don't forget to change it to the name of your JSON-file!

`private fun authorize() =  GoogleCredential`  
`       .fromStream(this::class.java.classLoader.getResource("service-accounts/**NAME-OF-YOUR-JSON**.json").openStream())`  
`       .createScoped(Collections.singleton(CloudRuntimeConfigScopes.CLOUDRUNTIMECONFIG))`

This function will return a GoogleCredentials object that contains the keys needed from your service account credentials JSON, and authorize the object to talk to the Cloud Runtime Config API. Let's set that up!

`private val RUNTIME_CONFIG_CLIENT by lazy {`  
`       val httpTransport = GoogleNetHttpTransport.newTrustedTransport()`  
`       val jacksonFactory = JacksonFactory.getDefaultInstance()`  
`       val googleCredential = authorize()`  
`       CloudRuntimeConfig(httpTransport, jacksonFactory, googleCredential)`  
`}`

We'll set it up as a lazy val so that the object is initialized when it's needed, not before. The CloudRuntimeConfig client takes three arguments and for the two first we'll use the standard implementations. The first is a HttpTransporter that the client will use to send http requests to the API and the second is a JSON handler. For that we'll use Jackson, the defacto standard JSON library for Java. If you don't have it in your dependencies already you can add it with the instructions [from
here](https://github.com/FasterXML/jackson). The third argument being, of course, our GoogleCredential object created by our previous function.

Before continuing, we should make sure that we our requests doesn't timeout before having a chance to finish. We'll do this by adding a private function and calling it while we initialize the runtime config client.

`private fun setHttpTimeout(requestInitializer: HttpRequestInitializer): HttpRequestInitializer {`  
`    return _HttpRequestInitializer_ **{**`  
` requestInitializer.initialize(**it**)`  
`        it._connectTimeout_ = 3 * 60000`  
`        it._readTimeout_ = 3 * 60000`  
`    **}**`  
`}`

The HttpRequestInitializer is part of the google api client library, and in fact the GoogleCredential-class is a subclass of it. Because of that, we can easily use the above method while initializing the client, like this:

`CloudRuntimeConfig(httpTransport, jacksonFactory, **setHttpTimeout(**googleCredential**)**)`

Make note of the change, in bold! That will make all our requests have a **connect timeout** of 3 minutes, and a **read timeout** of 3 minutes as well! 

All that's left now is to write the functions that will interact with the variables!

`fun create(key: String, value: String) {`  
`    _require_(key.length < 256)`  
`    val response = RUNTIME_CONFIG_CLIENT.projects().configs().variables()`  
`        .create(getVariablesPath(key), Variable().setText(value)).execute()`  
`    log.info("Created $key with value $value in GCP runtime config. Response: $response")`  
`}`

First we'll make sure that the key we're trying to create is within the limits of how long a key can be, before calling the clients create function. We'll use our helper function to get the correct variable path and then set the value as text. You can also use `Variable().setValue()` for a Base64 encoded value instead of a string. The trailing execute() is what will actually trigger the API call so don't forget it!

The rest of the functions will look very similar:

`fun update(key: String, value: String) {`  
`    _require_(key.length < 256)`  
`    val response = RUNTIME_CONFIG_CLIENT.projects().configs().variables()`  
`        .update(getVariablesPath(key), Variable().setText(value)).execute()`  
`    log.info("Updated $key with value $value in GCP runtime config. Response: $response")`  
`}`  
  
  `fun get(key: String): Variable? {`  
  `    _require_(key.length < 256)`  
  `    val response =`  
  `        RUNTIME_CONFIG_CLIENT.Projects().Configs().variables().get(getVariablesPath(key)).execute()`  
  `    log.info("Fetching $key with value $response from GCP runtime config")`  
  `    return response`  
  `}`  
    
    `fun delete(key: String) {`  
    `    _require_(key.length < 256)`  
    `    RUNTIME_CONFIG_CLIENT.Projects().Configs().variables().delete(getVariablesPath(key)).execute()`  
    `    log.info("Deleted $key from GCP runtime config")`  
    `}`

    Makes sense? 

    There's also the watch function which is a bit special. It will wait for a value change on the specified key for a set amount of time. Default timeout is 60 seconds but that's customizable by sending in different parameters. Here's the most basic example of a watch function:

    `fun watch(key: String): Variable? {`  
    `    _require_(key.length < 256)`  
    `    val response =`  
    `        RUNTIME_CONFIG_CLIENT.Projects().Configs().variables().watch(getVariablesPath(key), WatchVariableRequest()).execute()`  
    `    log.info("Fetching $key with value $response from GCP runtime config")`  
    `    return response`  
    `}`

    This will create a **blocking** call that will wait for a change in the variable for the duration of the timeout you've set, or 60 seconds if you didn't specify any other. If a change comes during that time it will immediately be returned, otherwise it will return the current value when it times out. 

    [**The full code is available as a GitHub Gist here**](https://gist.github.com/cakism/2a8556f31828ab57ce3e40075a6dbec1)**!**

    Hope this was helpful, and if you have any questions you can reach me at hello at cakism.com !

### Resources:

*   [Google Runtime Configuration Fundamentals (Intro)](https://cloud.google.com/deployment-manager/runtime-configurator/)
*   [How to do OAuth2 with service accounts](https://developers.google.com/identity/protocols/OAuth2ServiceAccount)
*   [Cloud Runtime Config Java API Library](https://developers.google.com/resources/api-libraries/documentation/runtimeconfig/v1/java/latest/)
*   [Handling Google Client API timeout errors](https://developers.google.com/api-client-library/java/google-api-java-client/errors)
