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
* The response includes the ```cache-control: public``` header (avoids accidentally sharing 
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

## Smart API Proxy

The Smart API Proxy runs a smart offline-first proxy that can connect to supported EdTech APIs (e.g. OneRoster, 
Assignment and Gradebook Service, and the Experience API). A simple HTTP cache isn't enough to handle use cases such as:

* __Querying user data subsets__: an app might preload all relevant student results, and then use an API query to search 
those results. This will be a different API URL, so even though all student results for the query may have been loaded, 
a standard HTTP cache would fail because the API URL is different.
* __Saving data when offline and synchronizing__: when a user completes an activity, the data needs to be saved. If offline,
this data needs to be saved and sent to the server as soon as a connection is available (even if the app is closed in 
meantime). If user data is queried in the meantime, this data should be included in the results.

The Smart API Proxy supports:

* Intelligently preloading user data such that it will be available later offline
* Querying user data offline e.g. if all student results are loaded, it can query and return a subset of results offline
as per the API specifications
* Caching data loaded via any query run online for later user offline
* Querying the server for new data automatically when a connection is available
* Periodic polling for new data
* Saving data when offline and submitting to the server as soon as a connection is available

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

