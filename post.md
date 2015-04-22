## Introducing Lambda

At the last Amazon AWS re:Invent the [Lambda](http://aws.amazon.com/lambda/) service was announced. AWS Lambda is essentially a hosting service for software modules. It enables uploading a software module (so far they support only JavaScript) and running it instantly in the cloud – either by calling its functions explicitly or by configuring functions to be run when some specific cloud events are triggered. As an example, it is possible to have Lambda to react on events from AWS Kinesis, S3 and DynamoDB.

## Generating CMS-powered static websites

At [Contentful](https://www.contentful.com) we run internal hack days several times a year, during which everyone works on an experimental project for one working day. AWS Lambda stoked my interest as a possibility to run spiky workloads on demand in an event-driven fashion, without having to worry about any detail of the hosting, and with a highly cost-efficient model.

One possible interesting application that came to mind is static website generation, since AWS S3 is a popular hosting system for static websites. I started wondering how would it be to put together S3 and Lambda to generate a static website. Certainly, I wanted to use the [Contentful Delivery API](https://www.contentful.com/developers/documentation/content-delivery-api/) as the content source – not only because I work at Contentful (which is still a good reason anyway), but also because a JSON-based HTTP content delivery API is exceptionally well suited for use cases as such.

## Half-static, half-dynamic

Here’s a schema of how this proof of concept works.

![Schema](https://images.contentful.com/256tjdsmm689/4pcl1tc1f2OyqwquAwS8Cc/03f42fcf07cc51367ca9130629f8bbac/aws_lambda_blog_post.png)

We manually trigger a Lambda function to call the [Contentful Sync API](https://www.contentful.com/developers/documentation/content-delivery-api/#sync). This function will fetch many content elements and publish each on a [Kinesis stream](http://aws.amazon.com/kinesis/). On the other end a Lambda function is listening to the Kinesis stream and renders a [doT.js](http://olado.github.io/doT/index.html) template, generating one HTML page for every content element and saving it to S3. Saving pages to S3, publishing and listening to the Kinesis stream from the Lambda code is simple, as it’s possible to use the [AWS SDK for Node.js](http://aws.amazon.com/sdk-for-node-js/) from within Lambda.

This solution allows to clearly separate the interaction with the content delivery API from the template rendering, and has the benefit of scaling template rendering independently from any other component. This is especially beneficial when the number of pages to be rendered is high: AWS Lambda will take care of adding computing capacity assigned to the rendering as needed to keep up with the stream.

## Proof of concept

The code of this proof of concept is released as open source [on GitHub](https://github.com/contentful-labs/contentful-aws-lambda-static) for those who are interested in the implementation details and want to engage in technical discussion.

## Discoveries

Here are some observations based on this proof of concept.

* It would be possible to maintain fairly large static websites, as this concept enables generating or regenerating a big collection of static pages in seconds.
* If a dynamic website doesn’t have a great deal of personalization, a solution like this might blur the border between static and dynamic websites, allowing for static hosting of websites which were previously hosted as dynamic.
* This approach is cost-efficient, as the owner would pay only for the freshly generated content.
* While this example renders full HTML, it’s not a problem to use similar pipeline to render only partials, or other intermediate artifacts like JSON representation of query results, or aggregates of queries.
* A similar pipeline could also be used to warm up the caches for high traffic websites by changing the destination of the artifacts from S3 to something like Redis or memcached.
* This setup seems to have robust properties from the functionality perspective, but it’s also highly secure: Lambda modules can be assigned very granular permissions through [IAM roles](http://aws.amazon.com/iam/).
* This setup combines well with a synchronization approach – like the one provided by the Contentful Sync API – since the only pages that are regenerated are the ones which have actually been changed.

Ultimately, this approach also shows how the cloud architecture is offering new solutions for existing problems, and how it changes trade-off balances on choices like static vs. dynamic website hosting. 

## Final notes

There is a number of natural improvements and modifications to this idea. For example, instead of working with barebone rendering systems it would be possible to employ a JavaScript static site generator like [metalsmith](http://www.metalsmith.io/) in combination with the respective [Contentful plugin](https://github.com/contentful-labs/contentful-metalsmith). 

Additionally, tighter integrations with the cloud environment – like the possibility of publishing events not only via webhooks, but, for example, directly on Kinesis streams, replacing the manual trigger – might enable the users to take advantage of approaches like the one described above.


