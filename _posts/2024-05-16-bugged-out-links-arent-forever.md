---
title: "<Bugged Out!/> Links Aren't Forever"
last_modified_at: 2024-05-17T08:09:00+01:00
classes: wide
categories:
  - Blog
tags:
  - Bugged Out
---
This series of blog entries aims to highlight and reflect on some of the more memorable quirks, absurdities and, of course, self-inflicted blunders I encountered during the daily life of engineering digital solutions and how I chose to approach them.
{: .notice--primary}

Connection timeouts and their associated errors are surely nothing out of the ordinary, even in system landscapes that praise themselves for their high availability. In most cases, these intermittent errors are short-lived and can be resolved effectively by a quick restart routine, putting everything back in working order. Sometimes there are issues or runtime failures rooted deeper within the infrastructure that lead to longer outages and require some more work for resolution. But when you are suddenly faced with a database connection that is both alive and well but also consistently timing out at the same time while running on seemingly the exact same infrastructure, you are in for a very interesting debugging session.

Schroedinger's connection occurred to me, as all demoralizing bugs do, on a previously calm Monday morning after returning from a vacation while deploying changes that had been successfully tested during my absence. I went into the deployment quite confidently, looking at the bright green glow of our database connectivity monitor, which indicates that all our database nodes are up and running and responding in a reasonable time. The testing feedback also showed no signs of anything being amiss. With the prospect of being able to check off one of my to-dos within just a few minutes of starting my day and before my morning tea had even cooled down to a drinkable temperature, I fired off the API deployment pipeline. Within about two minutes, the container image had been built, shipped, and deployed. After taking suspiciously long to pass the health checks, I decided to refresh the page of the deployment environment, and the unwelcome sight of an unhealthy container status burned itself into my retina. Still quite optimistic, upon investigation of the log files, I was presented with the error message that would be the bane of my existence for the next few hours.
```
Connection timed out.
```
Something wasn't quite adding up, I thought to myself while observing the still blissfully green database connection monitor, working health check endpoints of both the previously deployed API and locally running containers, along with the now officially failed deployment. Slightly humbled and almost convinced that "Well, then it must be some issue with my code!" I re-run the deployment with the _previously working_ code. Build, ship, deploy again...and now the, let me emphasize again, _previously working_ code also fails due to a connection timeout on not just one but all available database nodes. Even more humbled and mostly just confused, I check around the room for the hidden camera, and someone jumps out at me, revealing that this is all just a sick joke. But, to no one's surprise, it's just me and my tea.

Trying to systematically determine what might be the cause of this annoying but admittedly interesting issue led me to try and connect to the database via all available means. As always, everything worked on my machine, and even locally running containers were showing no signs of connection timeouts. I dive deeper and investigate DNS records, IP mappings for firewall requests, package versions, and dependencies, but to no avail, everything looks just as before. Even StackOverflow didn't really seem to understand my problem since the database listener was listening, all services were up and running, but the deployments were still failing.
My tea had long gone cold.

In a last attempt to somehow Occam's razor my way to the core of the issue, I climb along the connection chain, trying to find the simplest explanation for what could be going wrong. While explaining away a possible driver issue by reminding myself that we had been using the newest and most up-to-date drivers for database connectivity for quite some time now, I decided to check the driver installation in our containers. Once looking at the line in the corresponding Dockerfile, I made a realization that is best described by someone who has a much better way with words than I ever will:

 > [...] the magnitude of his own folly was revealed to him in a blinding flash [...]  
 >~ _J.R.R. Tolkien (The Return of the King, 1955)_

The link that was used to download and install the database drivers in the container was indeed always pointing to the latest version of the available drivers. With all the pent-up tension and self-loathing culminating in a cluster headache of almost biblical extent, I opened the changelog to see what I had been fearing. Support for the specific database version we had in use was indeed dropped in the newest driver release. Everything was working locally due to cached and pre-installed driver versions, the old code was still running the older drivers, and re-deploying always resulted in the new drivers being pulled regardless of which version of the code was being used. When I awoke from this problem-solving binge, I finally realized that my tea was close to freezing at this point, I hadn't eaten anything, and most of my day was spent, but at least I could go on with the certainty that at least it wasn't the code that was the problem. Additionally, I learned a valuable lesson: while diamonds may be forever, links certainly are not.

All humor aside: _Just as you are carefully curating package dependencies with specific versions for your projects, always make sure to also check whether any links that you are accessing within your code or build configuration point to a specific resource in such a versioning-dependent scenario. However, why an unsupported driver attempting connection to the database resulted in a generic timeout instead of returning a more expressive error message is still beyond me._

Thank you for reading!
{: .notice--primary}

---
#### DISCLAIMER
Technical detail is intentionally vague in some areas of these articles as to not expose any security-relevant information about hardware, software or infrastructure utilized by my current or past employers.