### State management:
- Most bugs ... caused by an application's state becoming corrupted, is a common problem when states manipulated by multiple components unknown to each other.
- Reactive programming is a way of dealing with stateful app, in order to isolate state changes.

Issues of state mgmt get more complex when we start dealing with:

### Deeplinks
- Q: what if app with existing state?
- A: the least amount of non-deterministic behavior => reset the app's state when receiving a deeplink

### Push notifications
further extend problem of deeplink:
- no guarantee that user will opt into them, or device will be online to receive them
- delivery is not guaranteed
- both Apple and Google might throttle push notifications
  - The rules around this throttling is a black box.
- connectivity issues + OS restricting notifications for apps that have not recently been active => people not seeing push notifications you send

### Background notifications 
(probably more OS specific)
handy for real-time updates and user on multiple device scenarios

### App update and compatibility:

**You can not assume that all users will get this updated version, ever.**

Some users might have automated updates disabled. Even when they update, they might skip several versions.
Many who do not update are blocked because of old phones or OSes.
Possible solution - **Force upgrade**

Api backend that return supported app version
- [link](https://medium.com/@yerem.khalatyan/how-to-force-update-a-mobile-app-when-a-new-version-is-available-74c7c9c820d4)
- min api version defined there might lead to customer churn

Need to take into account backward/forward compatibility: usually handled via GraphQL or versioned endpoints

### Offline mode
- The OS could report user online when we connect to WiFi spots that use captive portals, no data might be transmitted.
  - For an edge case like this, the app might need to ping a couple of “always online” domains to determine this.
- Synchronization of device and backend data

multiplied with multiple devices => need for conflict resolution protocol, that works for multiple parallel offline edits and is robust enough to handle connectivity dropping midway

### Retry strategies
- network is not down but just slow
- users frantically retrying and creating multiple parallel requests?
- How app differentiate between the backend not responding or the network being slow?
- 
:warning: Some workaround for the backend: HTTP conditional requests with retries utilizing [ETags](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag) or [if-match headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Match)?

:warning: Client - it is important to be able to handle cancellation of async requests

Many of the above situations can be **solved** relatively simply when **using reactive libraries** to handle network connections [ribs](https://go.mobileatscale.com/ribs)

### On third party libraries

Why version pin can be a good idea:
- in April 2020, the Google Maps SDK team pushed code to their backend that caused the apps embedding it to crash impacted hundreds of apps.
- In May 2020, something similar happened with the Facebook SDK, which had a repeat incident in July 2020.


Good practice to have a **feature flag specifically for your third-party libraries**, so that you can encapsulate the loading of those libraries and all the execution points

**Metrics on choosing those 3rd party libs:**
- App size
- Risk of no maintenance
- Tooling upgrade risks: Could adding this dependency be a blocker when wanting to upgrade to a new XCode or Android Studio version?
- Third-party responsiveness

### Trying new tech:
- shifting to spend time on prototyping can often be more useful than whiteboarding further approaches.
- architecture is a means to control the level of isolation between teams and components


In 2016, Swift was at version 2.2. The engineering team at Uber decided to go 100% with Swift when rewriting Uber’s Rider app.
The decision was almost a disaster, mainly because the team did not expect the binary size for Swift to be multiple times that of Objective C.
gory tech details is [there](https://go.mobileatscale.com/uber-binary-app-size-woes)

Airbnb [deprecate react native](https://medium.com/airbnb-engineering/react-native-at-airbnb-f95aa460be1c) for mobile development after year smth trying it to work

### QA
- Engineers should be confident in merging changes, knowing they did not break other parts of the app.
- German is often a good choice to stress test the app, as the language is more verbose.
- in their terminology - integration tests - multiple components interacting, but not through UI
- Snapshot testing - localisation with UI
- Snapshot tests - our golden tests - Uber end up with spliting code and tracked images to speed up git branch manipulation
- :warning: Airbnb had ~10,000 unit tests and ~30,000 screenshot tests.
- Uber run full set of UI tests only before releases.
- Interesting concept of build train - to track status of what is included into release and what stage of testing it is - they have special UI to track it


### Bug fixes and prioritisation:
- compare the cost of investigation and fixing to the upside of the fix, and the opportunity cost of spending time on something else, like building revenue-generating functionality.
- App-stability score
- ANR - app-non-responsive class of issues and how to monitor them (crashlytics should be enough to tackle it)

### Performance wise:
- instead of multiple network requests - create at backend endpoint that return all necessary data once
- get rid of REST towards QUIC
- the root cause of a page being slow is combination of how the mobile app and the backend work together.
- sampling real-world app performance measurements
- For mimic poor connection - special setup that drop 10% of the network requests on development builds and add latency at the networking layer to foster good “loading” UIs and minimise round trips

### Performance metrics:
- App startup time and tracking how app startup changes, as the application grows.
- Latency of loading screens, identifying screens that are the slowest, and code that contributes most to this delay.
- Networking performance: measured as latency and parallel requests occurring on the client-side, and as error rate by endpoint.
- Memory consumption: measured by the memory used by the app, or simply by the device’s free memory.
- Local store size, if the app uses local storage. For example, when caching locally to ensure cache eviction policy works properly.
- UI performance, for example, by measuring dropped frames, slow frames or frozen frames.

### Dedicated light version of app for specific regions:
in emerging markets, research showed potential customers were not downloading the relatively large app. This led to building and launching Uber Lite: a more limited Uber app, one with a 5MB footprint, optimized for low bandwidth usage

### Security
Logging PII data without end-to-end encryption in place might invite data breaches. - Aim to not log PII data, or instead **anonymize** this information **in the logs, turning it into non-PII data**.

### Functional requirements :sob:
A large part of project delays come from unclear requirements and scope creep, and being unclear on what the business wants to build.
This is usually done by product managers starting up a PRD (Product Requirements Document) process, this document being a formal, “you can now start work” step with engineering.

Various:
Backend-driven mobile apps - when client map UI based on meta [command](https://github.com/MobileNativeFoundation/discussions/discussions/47#discussioncomment-480176) from the server. tldr; not for faint-hearted
