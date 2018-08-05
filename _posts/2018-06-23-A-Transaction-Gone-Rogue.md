---
published: true
title: A Transaction Gone Rogue
---
Last Friday I ran into a very peculiar issue with one of our services, an issue that led to a discovery of a bug in one of Spring Boot actuators.

The service in question is a dead simple Spring Boot application that exposes an HTTP endpoint that returns monotonically increasing numbers. Hard to get this wrong.

In the morning we started getting reports of customers getting duplicate numbers from the service. Impossible. The service is super simple, not to mention that it has been tested numerous times by multiple developers and a QA engineer. 

We tried reproducing the issue in a qa environment with no success. It was working perfectly fine, and seemingly there was no way for the old numbers to ever be returned again.

I decided to look at the database stats. What I saw was an open connection with an idling transaction that was used by the application and Spring Boot db health checks. 

The thing is, we didn't use any sort of transaction management. There's only 1 query consisting of 1 statement. No need for transaction management. The statements are atomic in most RDBMS(including ours) and are committed automatically on successful execution.

The query running in a transaction would explain the situation perfectly. If the numbers are incremented in a transaction then if the service is redeployed or the connection to the db is lost, the transaction is rolled back, making us lose all previously served numbers, resetting the last served number in the table.

Where did the transaction come from? It was seemingly random, it would pop up out of nowhere. I looked at the logs at the time of the transaction start and couldn't see anything at all, not a single statement.

Then I turned my watch to the app health and other stats in [Spring Boot Admin](https://github.com/codecentric/spring-boot-admin). It is an extremely useful application that connects to the health and other endpoints or 'actuators' in Spring Boot applications. You can use it to take a heapdump, change logging level, etc.

After going there a couple of times I detected a most bizarre pattern -- every time I open the service details on SBA the transaction begins. How could SBA possibly start a transaction?

Everything became clear when I switched logging level to DEBUG, and went to the SBA while looking at `docker logs --follow`:

`[main] [] [DEBUG] liquibase -- Connected to dev@jdbc:postgresql`

`[main] [] [DEBUG] liquibase -- Setting auto commit to false from true`

`[main] [] [DEBUG] liquibase -- Executing QUERY database command: select count(*) from public.databasechangeloglock`


Looks fine. Liquibase, which is what we use to version our db, was changing the auto commit value. Auto commit is a property of `JDBCConnection` that, if set to false, starts a transaction and keeps it open, making the user of the connection either commit or roll it back.

Ok, I thought. Should be fine as long as Liquibase sets it back. 

I refreshed the SBA page.

`[DEBUG] liquibase -- Not adjusting the auto commit mode; it is already false`

Liquibase was not cleaning it up after itself and was leaving the auto commit set to false for all subsequent consumers of that connection.

Long story short, the problem was a Spring Liquibase actuator bug. It was not properly closing the connection after retrieving the information from Liquibase tables. To fix this I created [a pull request](https://github.com/spring-projects/spring-boot/pull/13559).
