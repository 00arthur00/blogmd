---
title: Pros/Cons of allowing push in Prometheus
date: 2019-10-30 15:38:34
tags: [监控, prometheus]]
---
Pros/Cons of allowing push in Prometheus
# Pros:
1. Enables IoT use cases with devices at customer / user sites that really can only push

     These currently don’t have any other TSDB that has Prometheus-style data model and querying, together with its other benefits
2. Enables other use cases with difficult firewalling or network segments where the operator is not allowed to deploy a Prometheus server in the same network segment or open up ports
3. Enables use cases where people want to bulk-import data from existing systems? (even if we treat Prometheus’s DB as ephemeral, it could be useful in some scenarios for people who just want to use PromQL to work with a given data set, which doesn’t even need to be system-monitoring-related)

# Cons:
1. People will abuse it:

    Attempt to push events vs. metrics

    Overuse it even if they could use pull, losing pull benefits:

        automatic upness monitoring
        
        Easy HA by running two Prometheus servers
        
        Flexible collection from laptop, etc.

        No need to configure identities into instances
    
    Harder to handle DoS/abuse scenarios
2. People would expect push support in all client libraries and exporters if the server supported push. We don’t want that, but then we’d only support push halfway, which might seem odd.
3. Push-only use cases can arguably be worked around in many cases (e.g. open a tunnel from the pushing device to some proxy, then let Prometheus pull over that; in other cases, it’s possible to run the Prometheus in the same network segment)
4. No "up" metric
    Query execution model is designed around pull
    
    Becomes worse when we fix #398, will make push un-workable in terms of staleness
5. With push comes bottom-up rather than top-down configuration

    This prevents horizontal monitoring

链接：https://docs.google.com/document/d/1H47v7WfyKkSLMrR8_iku6u9VB73WrVzBHb2SB6dL9_g/edit#