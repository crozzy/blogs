Joseph Crosland

## What is Clair?


[Clair](https://github.com/quay/clair) is a static analyzer that is used to index the contents of container images and provide vulnerability matching based on what it found. Clair is not only bundled with Red Hat Quay and used by [Quay.io](https://quay.io) but also used in various other Red Hat products with adoption increasing in an effort to offer customers a more standardized view of their security issues. As well as being used internally at Red Hat, Clair is used by many external teams and we always have a robust open-source community depending on us and working with us.


Our bread and butter is looking at container image layers, searching for clues as to what type of system the layer’s file-system is describing and creating a report that allows users to easily see the installed software the container holds and what vulnerabilities that software might be exposed to.


![Quay.io results page](/images/quay-io-results.jpg)

## The problem


Although we can test how Clair interprets well known images until the cows come home, containers in the wild can be built in many different ways and proliferated extensively as the basis for lots of other containers. This creates a loosely-bounded input problem where the potential inputs we will receive will be orders of magnitude more chaotic than our tests can cover. How can we deal with these problematic layers before Clair gets shipped to customers?


## The Solution


We can’t run Clair against every known container image in our CI pipeline, but something we can do is operate first. At Red Hat this principle means running a high traffic service with the latest code changes to allow you weed out any issues and corner cases. In this way,  you have confidence that when you stamp a release and customers start to upgrade, their instances are fit for purpose and stable. As mentioned earlier, Clair is the engine powering the vulnerability results surfaced by Quay.io. Quay.io holds approximately 10 million container images (the vast majority of which are multi-layered). As Clair scanners evolve or are added, this corpus is periodically re-indexed by Clair as we need to update the bill of materials for each container. By observing error patterns and drilling down into the logs we are able to identify problematic layers. After viewing the error patterns we triage the most frequent errors and either examine the layer directly if the manifest is public or try to build problematic layers from the contextual information in the logs. Sometimes this involves redeploying with additional logging.


Once we have a layer we can add it as test data and are able to derive regression tests from it. From there we can work on fixes to correct the edge-case, deploy a fix and validate. The identification is fast and the feedback after deployment is instantaneous, the amount of time and goodwill this saves us with customers is huge. Once we have confirmed the code is stable in staging and production environments we know we have a battle-tested release fit for other production environments.


Operating first also means that having dealt with running the service at significant enough scale we can start to accurately inform customers of storage/memory/CPU needs as well as high-traffic recommendations for their own environments. Not only that but we can give customers accurate expectations for offline operations like data-migrations and foresee issues that might arise in their environments.


### First Example: Problematic Layers In The Wild


Container images are composed of layers. Layers are just a file system grouped together in a [tar](https://www.gnu.org/software/tar/manual/html_node/Standard.html) file. In order to analyze a container image layer by layer we need to first create a representation of each layer’s file-system to allow the different scanners to look for the pertinent files (a package DB, a jar file etc). As layers are tar files at heart, building an addressable representation of a file-system from files in the archive shouldn’t be so difficult right? Well yes, 95% of the time it works that way. However, 5% of layers in the wild cause 100% of the headaches.


Clair includes a Go package called [tarfs](https://github.com/quay/claircore/tree/main/pkg/tarfs) which implements Go’s [fs.FS](https://pkg.go.dev/io/fs#FS) interface over a tar archive. In our case it is used to interact with the layer file. Most of the time this fs.FS representation is straightforward to build and work with, until it’s not.


Some container image layers are constructed in the deepest bowels of the darkest internet and do things like concatenate two tars together into one layer. Again, that doesn’t seem too crazy, but consider this: 

    ├── tar_1
    │   ├── run
    │   │   └── logs
    │   │       └── log.txt
    │   └── var
    │       └── run -> ../run
    ├── tar_2
    │   └── var
    │       └── run
    │           └── console
    │               └── algo.txt

`var/run` is both a symbolic-link to run (pretty typical) but is also defined as a normal directory in the second tar. Does this mean `algo.txt` should be accessible via `run/console/algo.txt` and `log.txt` should be accessible through `var/run/logs/log.txt`? We concluded yes, but as you might expect, before we found this in the wild (seemingly among CentOS base layers) we had not considered this as a valid use case. The error increased in frequency as Quay repeatedly tried to index the problematic manifests and eventually tripped the 5XX error rate alarm.


This is just one example of a potentially problematic layer which we've seen, we can also detect errors in build systems where documented behavior deviates from reality. This happened recently with the RHEL repository scanner and early RHEL 9 layers. Clair will attempt to parse CPE information from the embedded content-set files located at `root/buildinfo/content_manifests/` however the expected array wasn’t populated, manifesting in an error and our secondary detection method (using the Red Hat Container API) wasn’t triggered.


### Second Example: Operational Concerns


Away from difficult layers, we have recently been able to validate operational changes in our production environment. Because our customer environments can be so variable and they are often operating under restrictive scalability constraints we are continually looking at ways that Clair can play nicely with limited memory, CPU and storage. A part of this is trying to avoid database bloat and by analyzing index usage in production we were able to confirm we had a large unused database index. Once identified we were able to apply the migration to our staging environment and validate the migration was applied as expected and latency was unaffected, we could apply it to production. Apart from saving terabytes of data we are able to ship a highly consequential migration with the knowledge that it is safe for our customers Clair instances.


## Conclusion

This is an ever-moving target, we will always be pushing new code, potentially new bugs, and indexing new bizarre containers, but what is important to us is that our software has already been reviewed, tested and forged in production before we ship it to customers. 


Obviously there are downsides of such an approach that shouldn’t be discounted. First you need meaningful traffic and data and once you have those things then the problems start; on-call schedules, more release machinery, infrastructure costs, SLAs etc. With this in mind it won’t be possible for every team to Operate First with a SaaS deployment but teams could consider how they can foster usage in other ways e.g. internal instances, hosted sandboxes, [Github actions](https://github.com/features/actions) etc.


For us, tackling the unbounded input problem and being able to run consequential migrations using the Operate First mentality has not only opened up new benefits related to how we work but has allowed us to get in front of issues before they become problems.
