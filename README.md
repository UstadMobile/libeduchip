# LibRESPECT

LibRESPECT makes it easy to make easy-to-use, resilient edtech apps that:

* Support single sign-on and rostering
* Use far less data: up to 95% less when a group of users (e.g. students) are accessing
  the same content nearby
* Store/retrieve learner progress data to/from a Learning Management System, online or offline, 
  and syncs when a connection is available.

It has two primary components: a distributed edge HTTP cache and a smart edtech API proxy.

## Scenario

Low and Middle Income Country markets have the fastest growth rates, however there are also some challenges as outlined in the [Build for Billions](https://developer.android.com/docs/quality-guidelines/build-for-billions) guidelines:

- **Connectivity**: most users have limited Internet access via a capped mobile data bundle. Average mobile data usage per subscriber per month is [82% lower in Sub-Saharan Africa (5GB) than in India (29GB)](https://www.ericsson.com/en/reports-and-papers/mobility-report/dataforecasts/mobile-traffic-forecast), Most users are not entirely offline, but also do not have access to always-on high-speed practically unlimited connectivity. The connection may unreliable or slow, especially in rural areas. Using more data often directly increases cost to users.
- **Device capacity**: users are more likely to use a low-cost device which has less RAM, less storage space, and a slower processor than higher income markets. Users in lower income markets are also more likely to have an older version of Android.
- **Fewer opportunities to regularly recharge**: users in areas without reliable grid electricity may have to pay someone with a solar panel/generator to charge their phone

Optimizing performance to provide a better experience in Low and Middle Country markets also benefits users in higher income markets. When was the last time you heard someone complaining that an app started too fast and used too little disk space?

These challenges are unlikely to disappear anytime soon. Mobile data usage will grow, however low and low middle income country markets will continue to use less data per subscriber as [forecasted by Ericsson](https://www.ericsson.com/en/reports-and-papers/mobility-report/dataforecasts/mobile-traffic-forecast). Starlink and other satellite based Internet solutions are still more expensive (per GB) than fiber optic cables.

The education sector has very limited resources. The average low income country education budget is just $53 per child per year (including teacher salaries, books, building, ministry management, etc) as per [an April 2023 WorldBank report - pg5](https://thedocs.worldbank.org/en/doc/9b9ecb979e36e80ed50b1f110565f06b-0200022023/original/Adequacy-Paper-Final.pdf). If a country spent the same percentage (7%) of its education budget on technology as the United States, the edtech budget would be $3.71 per child per year. Reducing costs as far as possible is critical. RESPECT make this easier by:

* Reducing data use: peer-to-peer distributed caching reduces data use around 95% when an activity is being used by a group of 20.
* Making it easier for apps to function when connectivity is interrmittent or unreliable by preloading data via the network or using file storage (eg. SDCard, USB flash drive) and synchronizing progress when a connection is available.
* Single sign-in and interoperability that simplifies the user experience (reducing training and support costs) and enables decision makers to collect data and optimize resource allocation.

## Distributed Edge HTTP Cache

The Distributed Edge HTTP Cache handles all cases a normal HTTP cache handles and adds support for:
* Downloading http requests from nearby devices instead of via the Internet to reduce bandwidth usage
* Downloading and retaining a list of HTTP URLs such that they can be accessed offline later (e.g. when a user selects
a given lesson for offline use)
* Importing/exporting a set of HTTP requests/responses to/from a file so that they can be pre-loaded (e.g. via USB stick)

It can be used via OKHTTP, hence it works out of the box with almost all HTTP related libraries on Android/JVM.

### Get started

Create an instance of DistributedEdgeCache, then add the interceptor to your OKHttp builder:

```
val dCache = DistributedEdgeCacheBuilder(context).build()
val okHttpClient = OKHttpClient.Builder()
    .addInterceptor(DistributedEdgeCacheInterceptor(dCache))
    .build()
```

When a request is made for a URL which is available on a nearby device, it will be automatically
be downloaded from the nearby device instead of from the Internet if it meets the Redistributable 
HTTP criteria below.

Devices discover each other using [DNS-SD](http://www.dns-sd.org/) (the same protocol used to discover
network printers etc) and Google Nearby (where enabled). Devices proactively exchange hashes of all
the URLs they have available, so when a client makes a request there is no additional latency.

### Redistributable HTTP criteria

Distributed Edge Caching enables devices to download files from other nearby devices (peers)
running the same app (as would be common when a class of students use an app) instead of
downloading the files repeatedly the Internet (wasting bandwidth). Network operators (e.g.
mobile network operators, ISPs, etc) may independently operate a proxy cache for their users,
which an app can opt in to using.

An HTTP request may be fulfilled redistributed where:

* The request uses HTTP GET 
* The response includes the ```cache-control: public``` header (to avoid accidentally sharing 
responses that contain personal/sensitive information).
* The response does not vary based on request headers (e.g. it must not vary based on client user-agent etc).
* (Recommended) a checksum is provided to allow clients to verify integrity. This can be done by:
  * Add a request header directly specifying the expected [subresource integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity):
  e.g.  
  ```decache-check-integrity: sha256-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC```
  * Generating the checksum on the command line and then passing a url in the header 
  ```decache-check-integrity: https://example.org/file.zip.sha256sum```
  * The server adding a [Content-Digest](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Digest) header.

Any HTTP request that does not meet the criteria for distributed caching will still be cached 
on disk according to the normal HTTP cache rules.


### Download and retain a set of URLs for use offline

If a user selects a particular piece of content for offline use, then it is likely that you want to retain a given list
of URLs in the cache until the user explicitly unselects the content for offline use. 

DistributedEdgeCache makes this easy. Once the download has been completed, http requests for the given URLs will 
succeed as 'normal', even if the device is offline.

```
val jobId = distributedEdgeCache.downloadAndRetain(
    listOf(
        EntryLockRequest("https://example.org/lesson-1.mp3", remark = "lesson001"),
        EntryLockRequest("https://example.org/lesson-1.html", remark = "lesson001"),
        ...
    ),
)    
```

If/when the user no longer wants to retain the files:
```
distributedEdgeCache.release(jobId)
```

### Import/Export a collection of URLs to/from a file

If an organization (e.g. school) with limited connectivity wants to setup many devices quickly, it may be desirable
to load data to/from a file (e.g. a USB drive, SD-Card, etc).

Export to a [WARC](https://en.wikipedia.org/wiki/WARC_(file_format)) file can be done using external tools (e.g. 
[heritix](https://github.com/internetarchive/heritrix3)) or using DistributedEdgeCache itself:
```
val jobId = distributedEdgeCache.downloadAndExport(
    urls = listOf("https://example.org/lesson-1.mp3", "https://example.org/lesson-1.html"),
    file = "path/to/lesson1.warc",
)
```

Then import:
```
val jobId = distributedEdgeCache.importAndRetain("path/to/lesson1.warc")
```

On Android: you can use a built-in activity that will run the import if desired by adding an intent-filter:

```
<activity android:name="org.respectpolicy.dec.ImportWarc">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="file" />
        <data android:mimeType="*/*" />
        <data android:pathPattern=".*\\.warc.myappname" />
        <data android:host="*" />
    </intent-filter>
</activity>
```

## Smart Edtech API Proxy

The Smart Edtech API Proxy provides an HTTP proxy that can be embedded in edtech apps to make sending/receiving
learner data to/from a Learning Management System or Student Information System easier when connectivity is limited, unreliable, or unavailable. The typical flow is as follows:

* The consumer app (e.g. a math app) gets a token using [OAuth](https://oauth.net/2/) from the provider app using a normal single sign-on flow (e.g. a learning management
  system). If the provider app is installed and linked to the domain this can be done without requiring connectivity.
* The consumer app requests librespect-proxy to start a session using the authentication token. The librespect-proxy provides a localhost http server that
  can receive http requests using common edtech APIs including [OneRoster](https://www.1edtech.org/standards/oneroster), [xAPI](https://xapi.com), etc.
* The consumer app MAY request librespect-proxy to preload relevant data so it's available if the user goes offline later.
* The consumer app can send HTTP API requests to the proxy the same as it would send them to the normal server. The proxy will handle them as follows:
    * __Data retrieval requests__ : When a request is received and connectivity is available, the proxy will connect to the online API server to check for any updates and update its
     local database if required. The proxy will then respond to the request (even if the device is offline) using the local database. The proxy can even return a result when offline when
     that URL was never previously retrieved: because the proxy understands the underlying Edtech APIs, it can still run the query against its local database to provide a result. If the
     consumer app requests to preload a set of records when the user is online, and then requests one of those records individually when the user is offline, it will still work.
    * __Data storage requests__: When a request is received the updated data will be stored in the local database. It will then be sent to the online API server as soon as a connection is
    available. In the meantime the locally stored data will be returned in any relevant queries. If the data is rejected by the online server the local database will be updated accordingly.
    The data will even be sent if the user is connected after the app is closed by using the underlying platform background transfer APIs (e.g. WorkManager on Android).
* If the provider app is installed and supports [HTTP-IPC](https://github.com/UstadMobile/HTTP-IPC-Spec) then data can be sent to the provider app without connectivity.

Edtech apps often communicate with a Learning Management System (LMS) or Student Information System (SIS) to 
retrieve or store learner data to handle use cases such as:

* Retrieve rostering information e.g. class enrolments, role (student, teacher, or parent), grade level, etc.
* Storing learner responses and grades as they progress
* Storing and retrieving state data to enable the user to resume their session from where they left off, even on a 
different device

Edtech apps should not require teachers and students to have a separate account for each app. Users often use several
different apps for different subjects or purposes. Creating new accounts for each app for each member of the class 
creates a poor experience where time is wasted managing multiple accounts and it is far more difficult for teachers to 
track student progress across multiple apps. 

Simple single sign-on (e.g. sign-in with Google, Facebook, etc) is 
insufficient because it does not include the required context (user role e.g. teacher, student, etc) or the ability to 
store/retrieve learner progress and results.

Identity and profile management in an education context is often handled using proprietary services (e.g. [Clever](https://www.clever.com/)) 
or can be done using using open standards over HTTP REST services such as [OneRoster](https://www.1edtech.org/standards/oneroster), 
the [Assignment and Grade Service](https://www.imsglobal.org/spec/lti-ags/v2p0/), and the [Experience API](https://www.xapi.com/). 

Using open standard HTTP REST APIs becomes much more complicated if Internet access is intermittent or unreliable: apps often need 
progress information to determine what a learner should do next. If connectivity is interrupted when a lesson is 
completed then the result needs to be saved locally, shown to the user, and then submitted to the HTTP API as soon as a
connection is available. If the submission is rejected by the server, the local state needs updated accordingly.

### Handling imperfect connectivity

An **offline-first** app is able to perform key functionality without access to the Internet. In the case of an edtech app as above this 
should include:

* Preloading the learner profile information required for a student or teacher session to continue if the user goes 
offline later
* Saving learner progress information when the user is offline and submitting this to the HTTP API when a network 
connection is available.
* Resolving conflicts between the local state and the state on the server e.g. if the server rejects a result that was
recorded offline.

The Smart Edtech API Proxy makes this much easier by supporting:

* **Offline access to data**: Any data retrieved through the proxy when the 
user is online will be stored in a local database. If the user later goes offline, the proxy can serve the request using
the local database. This works even if the query is different (e.g. the app might preload all state data relevant to a 
given user session and later on query a specific subset of the data).
* **Storing/updating data offline** Any data to save or update is saved in the local database and queued for submission 
to the server as soon as a connection is available. The request is retried as required, and the proxy integrates with 
the operating system's background requests to ensure the data is submitted to the server even if the edtech app itself 
was closed in the meantime.
* **Handling conflicts** If an update is rejected by the server when submitted, then the local database must be updated.

The proxy will be used as follows:
```
val apiProxy = ApiProxyBuilder.endpoint("https://example.org/api/oneroster")
    .authorization("token")
    .protocols(ApiProxy.Protocol.ONE_ROSTER)
    .sessionId("session-id")
    .build()

//Get a localhost server that will operate as a proxy 
val proxyUrl = apiProxy.localUrl()
 
//.. Send API requests to/from the proxyUrl    
```

If an app is installed with a verified applink that includes the API endpoint and the app supports 
[HTTP/IPC](https://github.com/UstadMobile/HTTP-IPC-Spec), then data can be sent to the learning 
management system without requiring any connectivity.

