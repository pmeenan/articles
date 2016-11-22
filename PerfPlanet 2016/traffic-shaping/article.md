# Testing with Realisting Networking Conditions
*by Patrick Meenan*


When testing performance for websites or apps that you are working on it is critical to test them with networking conditions that are representative of your users.  That was one of the main reasons that I originally created [WebPageTest](http://www.webpagetest.org/) so it was easy to test and demonstrate what performance looked like when pages were not being loaded on ultra-fast corporate networks.

There are lots of ways to do this, from testing on real networks to emulating network conditions in a lab to tools provided by the browsers.

Real networks tend to vary performance pretty wildly, particularly mobile networks where signal strength, time of day and other traffic on the network can change performance by orders of magnitude.  What I have seen work best is to do some initial measurements of real networks to get an understanding how they behave and then use packet-level traffic-shaping tools to emulate the last-mile network conditions.  That also solves the problem of not always having access to the various different networks that you might want to optimize for.

![Traffic-shaping the last-mile](http://images.patrickmeenan.com/perf2016/last-mile.png)

It's also important to note that you can only realistically emulate the last-mile conditions and you can't easily emulate the whole network path.  The routes and peering relationships for all of the CDN's and servers involved in any given content means that it usually works best if you test from a location close to the physical location your users will be coming from and then use traffic-shaping to model the link conditions that you want to test.  

# Browser-level traffic-shaping

Chrome includes so support for emulating various different [networking conditions](https://developers.google.com/web/tools/chrome-devtools/network-performance/network-conditions) as part of Dev Tools but I'd only recommend using that for really rough ballpark timings and for seeing how resource sizing impacts the overall times.  Since it is implemented well above the networking stack there are a lot of aspects to the connection that are modeled incorrectly.  Some that come to mind are:
* **[TCP Slow Start](https://en.wikipedia.org/wiki/TCP_congestion_control#Slow_start)** - Generally at a packet-level at the beginning of a connection, the amount of data that can be transmitted in each round trip to the server increases slowly but since the traffic-shaping is done well above the connection layer, the slow start algorithm actually sees the faster real RTT and data ramps up significantly faster than it would otherwise.
* **[Bufferbloat](https://en.wikipedia.org/wiki/Bufferbloat)** - Since the shaping is done above the packet layer there is no underlying buffering of the raw packets.  They are received and Ack'd on the fast network and then slowly delivered inside of the browser.  In the real world if you over-shard your content and deliver over lots of parallel connections with a slow underlying network you can easily get into a situation where the server thinks data queued in buffers had been lost and re-transmits the data causing duplicate data to consume the little available bandwidth ([spurious retransmits](https://insouciant.org/tech/network-congestion-and-web-browsing/)).

There are a lot of server/CDN tuning parameters that are critical to performance ([Initial congestion Window](https://developers.google.com/speed/protocols/#an-argument-for-increasing-tcps-initial-congestion-window), [HTTP/2](https://http2.github.io/), [BBR](http://netdevconf.org/1.2/slides/oct5/04_Making_Linux_TCP_Fast_netdev_1.2_final.pdf), [QUIC](https://en.wikipedia.org/wiki/QUIC)) and the only way to accurately see the impact is by traffic-shaping at the packet level. 

# Recommended Tools

## FreeBSD
[Dummynet](http://info.iet.unipi.it/~luigi/dummynet/) is pretty much the gold-standard when it comes to traffic-shaping and makes it logically easy to reason with the ability to set up "pipes" with various network conditions and use ipfw rules for managing the traffic-flow.  WebPageTest has historically relied very heavily on dymmynet for all of it's traffic-shaping and configures one pipe for inbound traffic and one pipe for outbound traffic.  There is a Windows port which has been part of the WebPageTest desktop agents since the beginning and for mobile devices I usually route their connections through a FreeBSD bridge that does the traffic-shaping.

Unfortunately Windows 10 broke one of the interfaces that the Windows driver used so it only works up through Windows 8.1.  There have also been some incompatibility issues with the accelerated networking drivers in KVM (on Amazon AWS for example) but when it can be used it has worked great. 

## OSX/iOS
As part of the development tools for XCode, Apple makes available a tool called [Network Link Conditioner](http://nshipster.com/network-link-conditioner/) which is basically a GUI on top of dummynet and makes it really easy to get high-quality traffic-shaping in a local dev environment.  If you primarily do your development on a Mac, I HIGHLY recommend using the link conditioner for all of your testing.

![Network Link Conditioner Screenshot](http://images.patrickmeenan.com/perf2016/link-conditioner.png)

## Linux
Traffic control with [NetEm](https://wiki.linuxfoundation.org/networking/netem) on Linux can be used to model networks with extreme flexibility though it can be a bit daunting to come up to speed.  Facebook has released a (front-end)[http://facebook.github.io/augmented-traffic-control/] for it that works particularly well if you are using a Linux device on the network to do the traffic-shaping (providing a web UI to select a profile for the device you are currently using to access the network).

# Introducing winShaper

You might notice the glaring hole for Windows in the recommended tooling list.  Dummynet is excellent but it isn't as user-friendly as Mac's link conditioner and doesn't work on Windows 10.  As part of adding Windows 10 support to WebPageTest I had to create a new packet-level [traffic-shaping driver](https://github.com/WPO-Foundation/win-shaper/tree/master/driver) and once that was working well it was just a few hours of work to create a simple GUI.

The GUI release is available on [GitHub](https://github.com/WPO-Foundation/win-shaper/releases) and is completely stand-alone (no installer).  You need to be an administrator to run it since it needs to run a device driver to do the filtering but otherwise it is just a matter of running winShaper, selecting a connection profile and turning it on.  

![winShaper main dialig](http://images.patrickmeenan.com/perf2016/main.png)

![winShaper connection selection](http://images.patrickmeenan.com/perf2016/select.png)

Once enabled you will see some activity as the "network buffers" fill up and drain based on the traffic-flow and when you want to turn it back off you can either close it or push the disable button.

![winShaper running](http://images.patrickmeenan.com/perf2016/enabled.png)

By default it ships with the same connection profiles as WebPageTest but you can also specify custom profiles including how big the network buffers should be (150KB roughly mirrors the dummynet default of 100 packets though in practice buffers on real networks can be even larger).

![](http://images.patrickmeenan.com/perf2016/profile.png)

The code is all on GitHub and is all under a very liberal Apache license so if you need traffic-shaping capabilities for a project you are more than welcome to use it.  The driver is digitally signed so it doesn't require anything special other than administrator rights for installation.  There is also an stand-alone command-line test app that shows how to use the API to the driver (just a few lines of code to control it) and I expect a console app allowing full control will come along shortly to make it easier to integrate with CI tooling and other projects.  As always, pull requests or suggestions are welcome.