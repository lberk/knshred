# knshred are couple of helpers for dealing with Knative Eventing

## Background

So every now and then integration tests fail and you want to fetch some logs
like the integration test logs as well as in some cases you might want to fetch
and see what the controllers were up to. So, I hacked up some super (crappy, but
seemingly functional scripts). These scripts allow you to do couple of things:

1. Fetch the logs given a PR number
1. Do some rudimentary shredding of the logs to make it easier to reason about
   failures. 

## Fetching logs

So, you have a b0rk3d integration test, just do this to fetch the logs:

```
./fetchlogs -p <PR>
```

So, for chaos broker testing, the lovely gift that kept on giving was
[3599](https://github.com/knative/eventing/pull/3599) so to fetch the logs to my
local machine, I'd do:

```
./fetchlogs -p 3599
```

That will only fetch the build-log.txt (build / test that will show you what
failed). If you specify the `-k` flag it will fetch also the artifacts, which
contain all the controller logs and some other stuff as well. This is not on by
default because those logs can be huge...

But thanks to good old `3599`, I needed those logs too, so I'd use:

```
./fetchlogs -p 3599 -k
```

It will then download the logs into `/tmp/<PR>` directory, so when I ran the
command I'd have something like this waiting for me, yay!

```
vaikas-a01:eventing vaikas$ ls -altr /tmp/3599
total 1203600
drwxrwxrwt  45 root    wheel       1440 Jul 16 16:53 ..
-rw-r--r--   1 vaikas  wheel   10370807 Jul 16 16:53 build-log.txt
drwxr-xr-x   4 vaikas  wheel        128 Jul 16 16:53 .
-rw-r--r--   1 vaikas  wheel  601411238 Jul 16 16:54 k8s.log.txt
```

## Fetching from different repos

fetchlogs.sh also works for things like eventing-contrib, just use the `-r`
flag. It defaults to eventing, so if you wanted to fetch logs for
`eventing-contrib` you could do it like so:

```
./fetchlogs.sh -p 1378 -r eventing-contrib -k
```

It will then do the same thing and download the logs like so:

```
vaikas-a01:knshred vaikas$ ls -l /tmp/1378/
total 69720
-rw-r--r--  1 vaikas  wheel   1485002 Jul 17 10:17 build-log.txt
-rw-r--r--  1 vaikas  wheel  32788863 Jul 17 10:17 k8s.log.txt
```


## Shredding logs

Another thing is that if there are multiple test failures, we probably want to
slice these a bit. So, for example if you wanted to see the failed tests broken
up and split into their own smaller files that you could then inspect, you could
run the following:

```
./debuglogs.sh -d /tmp/3599/
```

And since there were two lucky winners in that run that failed, after this
you'll see:

```
vaikas-a01:eventing vaikas$ ls -l /tmp/3599/
total 1203624
-rw-r--r--  1 vaikas  wheel   10370807 Jul 16 16:53 build-log.txt
-rw-r--r--  1 vaikas  wheel       3580 Jul 16 16:58 build-log.txt.TestPingSourceV1Alpha2EventTypes
-rw-r--r--  1 vaikas  wheel       4295 Jul 16 16:58 build-log.txt.TestTriggerDependencyAnnotation
-rw-r--r--  1 vaikas  wheel  601411238 Jul 16 16:54 k8s.log.txt
```

That way you can see only the failures for those two tests broken into their own
test files.

Ok, but what can you do for me about them k8s files?? Well, if you specify the
`-k` flag, it will parse the k8s.log.txt and fetch only the relevant bits from
the controller logs that pertain to those failed tests? But, Ville?? How can you
tell that? Well, because each test runs in it's own namespace, we first find the
failed tests from above, figure out which namespace they ran, then and only
after that do we now go out and fetch the relevant entries from the k8s
logs. Since I was working on the `mt-broker-controller` controller, I specified
that I only wanted to look at them with the -m flag.

```
./debuglogs.sh -d /tmp/3599/ -k -m mt-broker-controller
```

And now again we'll see two new files there:

```
vaikas-a01:eventing vaikas$ ls -l /tmp/3599/
total 1203728
-rw-r--r--  1 vaikas  wheel   10370807 Jul 16 16:53 build-log.txt
-rw-r--r--  1 vaikas  wheel       3580 Jul 16 17:05 build-log.txt.TestPingSourceV1Alpha2EventTypes
-rw-r--r--  1 vaikas  wheel       4295 Jul 16 17:05 build-log.txt.TestTriggerDependencyAnnotation
-rw-r--r--  1 vaikas  wheel  601411238 Jul 16 16:54 k8s.log.txt
-rw-r--r--  1 vaikas  wheel      25372 Jul 16 17:05 k8s.log.txt.TestPingSourceV1Alpha2EventTypes
-rw-r--r--  1 vaikas  wheel      24378 Jul 16 17:05 k8s.log.txt.TestTriggerDependencyAnnotation
```

## TODO:

1. Make the grep flags configurable. My bash-fu doesn't really shine there.





