# Updating to latest Docker Desktop broke Envoy proxy routing in our web application (triggered by a MacOS update that detected docker as malware).

## executive summary
- Our web application uses an envoy proxy for routing gRPC API communication to the backend.
- We deploy envoy proxy as a docker.
- Apple updated MacOS malware detection, which detected that docker was malware, forcing us to update docker.
- Updating to the lateset version of docker desktop broke envoy routing for our web application.
- The solution is to change the envoy config file cluster section to use "dns_lookup_family: V4_ONLY" to force ip address resolution to use IPv4.

## the gory details
UGH

A colleague reported that 1) he started getting a macos dialog saying that docker desktop was determined to be malware 2) so he moved it to the trash and reinstalled docker desktop and 3) the dp web app is now broken, e.g., when you try to do a raw data query you just get a "loading…" message and nothing ever happens and the chrome developer console shows a 503 error trying to call the grpc API.

I did a ton of investigation, most of which was fruitless.  I installed wireshark on the mac and confirmed that port 8080 (the envoy proxy running in docker) received a grpc request from the browser and returned a http 503 error in response.  I also confirmed with wireshark on linux that this worked successfully, returning an http 200 message.

I changed our envoy-docker-create script to use the (apparently) latest image, v1.31-latest (we were on v1.22.0.  This lead to slightly different behavior in wireshark, with an extra message between the browser and the envoy proxy, but still got the 503 error.

I changed the log level of the envoy docker container by adding "-e loglevel=trace" in the docker create command in envoy-docker-create.  This is the general mechanism for setting an environment variable when creating or running a docker container.

With this change, there is now a copious amount of envoy logging.  I tailed the envoy logs (using "docker logs -f envoy") and initiated the web app query from the browser.  I then killed the tail on the envoy logs because there is a lot of output.  I scanned the output and found messages like this:

```
[2025-01-17 20:21:47.173][26][debug][connection] [source/common/network/connection_impl.cc:1017] [Tags: "ConnectionId":"1"] connecting to [fdc4:f303:9324::254]:50052
[2025-01-17 20:21:47.173][26][debug][connection] [source/common/network/connection_impl.cc:1043] [Tags: "ConnectionId":"1"] immediate connect error: Network is unreachable|remote address:[fdc4:f303:9324::254]:50052
```

The ipv6 style IP addresses made me a bit suspicious, so I did some searching about that with respect to macos, docker, and envoy.  Sure enough, there is discussion from last fall that starting with Docker v4.31.0 on the mac, "host.docker.internal" now resolves to an IPv6 address where previously it resolved to an IPv4 address.

This explains why our envoy configuration broke when we updated to the latest version of docker desktop (I'm now on 4.37.2).  I'm not sure why the IPv6 address doesn't work with envoy, but I've seen problems with IPv6 in lots of other contexts so I looked into whether this was a problem with docker or envoy, and found a mechanism in the envoy config to force it to use IPv4 (more documentation here), by adding "dns_lookup_family: V4_ONLY" in the cluster section of the envoy config, e.g., 

```
  clusters:
    - name: query
      connect_timeout: 0.25s
      type: logical_dns
      dns_lookup_family: V4_ONLY

```

Having made this change in envoy.mac.yaml, stopping, removing, and creating the envoy docker container, the web application now works.

I did some investigation into recent MacOS updates (by doing "About This Mac", clicking "System Report", scrolling down to "software", selecting "installations", and sorting by installation date, I can see that something called "XprotectPlistConfigData" version 5285 was installed automatically on 1/14/25.  Turns out that component "prevents known malware from running" and the update presumably started flagging the docker desktop application as malware, forcing us to update to the latest version, which resulted in the new ipv6 problem.  Interestingly, Apple doesn't document what changed in a XProtectPlistConfigData release.
