---
title: "Rest API calls made from prod process studio versus control room giving different results"
url: "https://community.blueprism.com/t5/Digital-Exchange/Rest-API-calls-made-from-prod-process-studio-versus-control-room/m-p/126165#M4731"
date: "2026-09-01"
author: "a026833"
feed_url: "https://community.blueprism.com/hfgie69872/rss/Community?interaction.style=forum"
---
We have a process using the Utility HTTP VBO to make HTTPS POST calls to an external vendor using basic authentication. The process succeeds every time when run from the Process Studio on the Runtime Resource but fails (most times) when executed from the Control Room. The error returned on the control failures is a (502) Bad Gateway.
