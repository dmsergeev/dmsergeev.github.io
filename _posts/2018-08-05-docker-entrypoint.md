---
published: true
title: Dangers of ENTRYPOINT's shell form
---
Look at the Dockerfiles for your applications. If you see `ENTRYPOINT` or `RUN` instructions in the following format, then you might be in trouble:
- `ENTRYPOINT node app.js`
- `ENTRYPOINT ["sh", "-c", "node app.js]`

