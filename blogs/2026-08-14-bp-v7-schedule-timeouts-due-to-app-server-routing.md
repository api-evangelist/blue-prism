---
title: "BP V7 Schedule Timeouts due to App Server routing"
url: "https://community.blueprism.com/t5/Product-Forum/BP-V7-Schedule-Timeouts-due-to-App-Server-routing/m-p/126058#M54709"
date: "2026-08-14"
author: "CJCutting"
feed_url: "https://community.blueprism.com/hfgie69872/rss/Community?interaction.style=forum"
---
As we moved from BP V6 to V7 we noticed that a number of our schedules were failing. The common symptom was an App Server log showing 'no response from resource PC' with a timeout of 10 minutes. We traced this back to the upwards connectivity from the resource VM's back to our App Servers.
