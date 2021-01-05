---
layout: post
title:  "Building an ESB cloud"
date:   2019-06-23 08:00:00
categories: [Kubernetes, Mule, Java, Spring Cloud]
---

You maybe know in a typical manufacturing enterprise, they have many services running under different environments, and there are no connections between them.

The enterprise data maybe be saved or exposed in files, databases,  services' APIs, etc. If we are going to build multiple services above these data, that will be a big challenge, the connections between them will look like a net, and for the developers, they have to write the code in different languages and maintenance the versions separately.

If we are asked to do some changes on these data, that will be hell!
That's why we need to build an ESB to resolve this problem,  for further reading, should navigate to [What is API-led Connectivity?](https://blogs.mulesoft.com/dev/api-dev/what-is-api-led-connectivity/).

In our organization, this will connect to thousands of devices and clients, it requires stable, scalable, and flexible, and the most important thing is the service can't stop if they are online.

We already dockerized our services and managed them in kubernetes, so for the ESB services we can manage them in the same way.  
The following picture shows us the architecture for the ESB cloud.
![ESB Cloud Architecture](/assets/images/2019-06-23-building-an-esb-cloud/esbcloud.png){:width="100%"}


From the ESB router, will generate the unique ID for each request, then put the generated ID into the redirect request's header. In the following steps, the ID will be used to link the request and status object.

For the mule applications, we need to register them before the ESB router running, then we can manage the lifecycle of the mule application through the management service we created.

The mule applications also required to update their process status by publishing the JSON formatted process status object into the MQ.
