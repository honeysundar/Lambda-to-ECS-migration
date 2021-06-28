# Lambda-to-ECS-migration


Why We Moved From Lambda to ECS


After many months of development, my team just announced the general availability of our platform. That milestone seems like a perfect opportunity to look back and reflect on how the infrastructure that supports Prismatic has evolved over time. (Spoiler: We ended up moving our most important microservice from Lambda to ECS.)

In this post I'll dive into what went well in Lambda, what challenges we faced, and why we eventually made the decision to migrate some services from Lambda to AWS Elastic Container Service (ECS).

What Problem are We Solving?
For some quick context, our product is an integration platform for B2B software companies. That is, we help software companies build integrations and deploy those integrations to their customers. A simple integration might look something like this:

Step 1: Pull down an XML document from Dropbox.
Step 2: Process the XML with some custom JavaScript code.
Step 3: Use some stored credentials to post the processed data to a third-party API.
Our users can configure integrations to run on a schedule, or they can trigger them via a webhook, and our platform takes care of running, logging, and monitoring the integrations (and a whole bunch of other things).

The Early Days
The first incarnation of Prismatic used LocalStack. We knew that we wanted to eventually host Prismatic in AWS (with the possibility of moving to Azure, GCP, etc. as needed), so the ability to spin up our platform locally to simulate AWS was appealing. The LocalStack service that approximates AWS Lambda was easy to iterate on, and ran without any major hiccups. It gave us a great development feedback loop, so we could prototype and test very quickly.

We used Lambda to execute each "step" of an integration, and steps leveraged SQS to pass data and trigger the next step. So, an integration execution would look like this:

Run a Dropbox "fetch a file" action to grab an XML file.
Save the contents of that XML file to SQS, trigger the next step.
Run a customer's custom JavaScript code to process the XML.
Save the resulting transformed data to SQS, trigger the next step.
Run an action to post the processed data to a third-party API.
Save the results of the final step, trigger the end of the integration.
Within LocalStack, this was a very quick process. We could define a 6-step integration, run it, and see results within a couple of seconds.

Our Migration to Real AWS Lambda
Once we had a proof of concept working, we devoted some time to moving Prismatic to an actual production environment, with real Lambdas, queues, databases, etc. We were still a small team, and we didn't want to dedicate a ton of time to DevOps-y, infrastructure problems yet. We wanted to dedicate most of our time to our core product, and Lambda let us do just that.

Lambda was attractive to us for a number of reasons. We didn't need to worry about CPU or memory allocation, server monitoring, or autoscaling; that's all built-in. We were able to throw .zip files full of JavaScript code at Lambda, and AWS took care of the rest. Lambda let us compartmentalize our code into a series of microservices (a service for logging, a service for OAuth key renewals, a service for SMS/email alerting if integrations error out, etc.), so we could keep a good mental map of what code is responsible for doing what task. Costs were pretty reasonable, too - you just pay for compute time, so rather than running servers 24/7, we just paid when our prototypes were executing something.

After a few days monkeying with Terraform, we had our second incarnation of Prismatic in AWS. Our integration runners ran on real Lambda, and were triggered via SQS. This is the point at which we started running into performance issues with our integration runners.

Why Lambda Didn't Work for Us
We had a number of issues, ranging from speed to SQS size limits and lack of process isolation in Lambda, that caused us to reconsider its effectiveness as our integration runner. Let's talk about each of those issues:

Speed. Remember the 6-step integration that I said took a couple of seconds to run within LocalStack? It took a full minute using real Lambda and AWS. The actual Lambda invocations were quick - usually a few milliseconds. The writing of step results to SQS and subsequent execution of the next step, though, ended up taking multiple seconds every step. For more complex integrations, like ones that looped over 500 files, that was a show-stopper - who wants their integrations to take minutes (hours?) to complete?

We tried a number of things to get our Lambda invocations to go faster. We followed guides to keep a number of Lambda instances "warm", and we cranked up the number of vCPUs powering our Lambdas to the highest we could at the time (6 vCPUs / 10GB RAM), but those things only shaved single digit percentages off of our integration run times.

SQS Size Limits. SQS limits message size to 256 kilobytes. The amount of data being passed between steps of an integration often exceeded that size (after all, it's totally reasonable for an integration developer to pull down a multiple megabyte JSON file to process). We were able to work around this size limitation - the recommended solution that we followed was to write out payloads to S3 and pass references to S3 objects via SQS - but this extra API call to S3 only compounded our slowness issues.

Process Isolation. This was the issue that surprised me the most. At first, AWS Lambda seems appealing as a stateless compute engine - run a job, exit, run another job, etc - scale horizontally as needed. We naively assumed that Lambda invocations were isolated from one another, but that turned out to only be half true. Concurrent invocations are isolated (they run in distinct containers within Lambda). However, subsequent invocations reuse previous "warm" environments, so an integration runner might inherit a "dirty" environment from a previous integration run. That's especially a problem if you let customers write their own code, like we do for our customers' integrations.

It turns out that if one customer writes some bad code into their integration - something like this, global.XMLHttpRequest = null;, then subsequent integration runs on that same Lambda that depend on the XMLHttpRequest library error out. This is a big deal, since one customer could break something like axios for another customer. A customer could even be malicious and execute something like global.console.log = (msg) => { nefariousCode(); }, and other integrations that execute on that same Lambda will run nefariousCode() whenever they invoke console.log(). Yikes!

We tried a few things to get around this issue of shared execution space. We toyed with forcing our Lambdas to cold-start every time (which was a terrible idea for obvious reasons), and we tried spinning up distinct Node processes within chroot jails. Neither option panned out - spinning up child Node processes in a Lambda took 3-5 seconds and partially defeated the purpose of being in Lambda in the first place.

Our Move to ECS
Lambda had served us well with development - we were able to iterate quickly and get a prototype out the door, but with the myriad issues we faced in Lambda we decided to bite the bullet and dedicate some dev time to cloud infrastructure.

Our team got to work expanding our existing Terraform scripts, and moved our integration runner to AWS Elastic Container Service (ECS). Within an ECS container we could easily (and quickly!) chroot and isolate Node processes from one another, solving the process isolation issues we were seeing in Lambda. To get around the SQS size limit issues we faced, we swapped in a Redis-backed queuing service. We had to reinvent some wheels that Lambda had given us for free - like logging, autoscaling, and health checks - but in the end we had our 6-step test integration back to running in under 2 seconds.

Now, ECS hasn't been perfect - there are are some trade-offs. For one, ECS doesn't seem to autoscale as quickly as Lambda. A "scale up" seems to take about a minute or so between API call and AWS Fargate pulling down and initializing a container that's ready to accept jobs. We had to pull one of our devs off of product development to work on cloud infrastructure, and there's a ton more to juggle with regards to CPU and memory usage, autoscaling rules, and monitoring, but at this point in product development the pains are worth the gains to give our customers a speedy integration runner.

What Remained in Lambda
We didn't move all of our microservices out of Lambda - plenty still remain in the serverless ecosystem and will for the foreseeable future. Our integration runner didn't fit Lambda well, but there are other tasks for which Lambda seems like the clear choice. We kept all important integration services that aren't critical to the actual execution of the integration in Lambda. Those include:

A logger service that pulls logs from ECS and sends them to DataDog.
A service that writes metadata about integration executions to a PostgreSQL database.
A service that tracks and queues scheduled integrations.
An alerting service that sends SMS or email notifications to users if their integrations error.
An authorization service that renews customers' OAuth 2.0 keys for third party services.
We didn't want any of these services to block execution of an integration, and for all of them it's fine if they take an additional second or two to run, so services like those fit Lambda perfectly.

Conclusion
Our infrastructure definitely changed over time, but I think the decisions we made along the way were the right ones: LocalStack's "Lambda" service let us develop and iterate very quickly, and our first deployment into AWS was simple enough that our small dev team could Terraform our infrastructure without losing a ton of dev hours to it.

Lambda seemed like an attractive solution for hosting and scaling our microservices, and for many of them, especially asynchronous services that might take a second or two to run, it still remains the correct choice. For our integration runner, though, we learned that the size, speed, and process isolation limitations of Lambda made ECS a better option, and it was worth the dev time it took to create an ECS deployment for that particular service.

Lambda let us concentrate on product development early on, and when the time was right the transition to ECS was a fairly smooth one. Even with the issues we faced in Lambda, I'm glad we took the path we did.

