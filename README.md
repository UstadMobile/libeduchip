# LibEduCHIP

LibEduCHIP makes it easy to make easy-to-use, resilient edtech apps that:

* Support single sign-on and rostering
* Use far less data: up to 95% less when a group of users (e.g. students) are accessing
  the same content nearby
* Store/retrieve learner progress data to/from a Learning Management System, online or offline, 
  and syncs when a connection is available.

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
<activity android:name="org.educhip.dec.ImportWarc">
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

An is able to perform key functionality without access to the Internet. In the case of an edtech app as above this 
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

