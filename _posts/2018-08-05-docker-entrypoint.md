---
published: true
title: Dangers of ENTRYPOINT's shell form
---
Look at the Dockerfiles for your applications. Do you see `ENTRYPOINT` or `CMD` instructions in the following format: `ENTRYPOINT node app.js` or `ENTRYPOINT ["sh", "-c", "node app.js"]`? If yes, then you might have a problem.


