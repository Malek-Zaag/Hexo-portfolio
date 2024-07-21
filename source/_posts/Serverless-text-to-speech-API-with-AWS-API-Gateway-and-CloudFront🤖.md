---
title: Serverless text-to-speech API with AWS API Gateway and CloudFront🤖
date: 2024-04-21 13:18:03
tags:
categories: [Blogs]
# sticky: 999
---

# Serverless text-to-speech API with AWS API Gateway and CloudFront­ƒñû

## Introduction :

In this blog, we will be setting a lambda function that interacts with Amazon Polly which is a text-to-speech managed service to convert a text into a nice hearing and customizable human voice.

![Project workflow](https://cdn-images-1.medium.com/max/2764/1*fjsDhAh_DWGPEWWSgGRpmw.png)_Project workflow_

We start by defining the technologies we used in this mini project :

- **AWS Lambda** is a compute service that lets you run code without provisioning or managing servers. Lambda runs your code on a high-availability compute infrastructure and performs all of the administration of the compute resources, including server and operating system maintenance, capacity provisioning and automatic scaling, and logging.

- **Amazon Simple Storage Service (Amazon S3)** is an object storage service offering industry-leading scalability, data availability, security, and performance. Customers of all sizes and industries can store and protect any amount of data for virtually any use case, such as data lakes, cloud-native applications, and mobile apps.

- **Amazon Polly** is a Text-to-Speech (TTS) cloud service that converts text into lifelike speech. You can use Amazon Polly to develop applications that increase engagement and accessibility.

- **Amazon API Gateway** is an AWS service for creating, publishing, maintaining, monitoring, and securing REST, HTTP, and WebSocket APIs at any scale.

- **Amazon CloudFront** is a web service that speeds up distribution of your static and dynamic web content, such as .html, .css, .js, and image files, to your users. CloudFront delivers your content through a worldwide network of data centers called edge locations.

Setting up our Lambda function :

![Lambda setup](https://cdn-images-1.medium.com/max/2504/1*8uo44pUJLh-YqmXdenH16w.png)_Lambda setup_

Now we have a lambda function with default code, we add a custom event just to try out the function and we test :

![](https://cdn-images-1.medium.com/max/2800/1*BEVA70_i43QdluDfXbiKhg.png)

## IAM Execution Role Setup :

A Lambda functionÔÇÖs execution role is an AWS Identity and Access Management (IAM) role that grants the function permission to access AWS services and resources. For example, you might create an execution role that has permission to send logs to Amazon CloudWatch and upload trace data to AWS X-Ray.

We start by going to the IAM section in the console :

![](https://cdn-images-1.medium.com/max/3024/1*wZoj9nrlCHZEtzo3XW0mMQ.png)

We add the desired permissions :

![Permissions](https://cdn-images-1.medium.com/max/3582/1*lz3Sfr--vB_auWXsu9aEBw.png)_Permissions_

We give a name for our role :

![](https://cdn-images-1.medium.com/max/3482/1*OQAX-LqaQjb5DIDHuW2gVA.png)

Now it is time to add some code to send the request to polly so it can produces the wanted audio :

    ```python
    import json
    import boto3
    import base64

    def generateAudioFromText(text):
        polly= boto3.client('polly')
        response= polly.synthesize_speech(  Text=text,
                                            TextType = "text",
                                            OutputFormat='mp3',
                                            VoiceId='Bianca')
        audio_stream = response['AudioStream'].read()
        audio_base64 = base64.b64encode(audio_stream).decode('utf-8')
        bucket_name = "serverless-api-bucket24"
        filename = "audios/audio.mp3"
        bucket = boto3.client('s3').put_object(Bucket=bucket_name, Key=filename, Body=audio_stream)
        return audio_base64

    def lambda_handler(event, context):
        # TODO implement
        audio_base64 = generateAudioFromText("Hello world!")
        return {
            'statusCode': 200,
            #'body': json.dumps(base64.b64encode(response).decode('utf-8')+)
            'headers': {
                    'Content-Type': 'audio/mpeg',
                },
            'body': audio_base64,
            'isBase64Encoded': True
        }
    ```

since we created a bucket already :

![](https://cdn-images-1.medium.com/max/2920/1*hdACDBqvga1bBSUKV68w5Q.png)

We trigger our function and we see the output in the appropriate folder :

![](https://cdn-images-1.medium.com/max/2000/1*GJ8XhR7ndGImUq67y4XXbg.png)

We also tried to add a longer text to convert :

![](https://cdn-images-1.medium.com/max/2296/1*dId9nUi1bPhstQFLIbRyZA.png)

and we got a satisfying result.

## API Gateway :

Now we proceed by adding an entry for our serverless api through api Gateway :

![](https://cdn-images-1.medium.com/max/2000/1*u_-uHm-lucuUt3ONtaqekQ.png)

since we setup our api gateway, we have a url we can use now to trigger our lambda function :

![](https://cdn-images-1.medium.com/max/2000/1*wmyWe4xIoTigFcvIZLhUuA.png)

we can send a simple GET request to have our output file :

![](https://cdn-images-1.medium.com/max/3324/1*rh3Lg8eWZzlIydn0yYU1oQ.png)

Now we tested that, we can have our own text to send to the converter through adding custom routes in the gateway and adding integrations with the lambda function :

![](https://cdn-images-1.medium.com/max/2566/1*N5uGFb1brSstfi6y-DWXWw.png)

We added integrations :

![](https://cdn-images-1.medium.com/max/2000/1*xGTaDopYKxWg-kMa-4DA9g.png)

Now we made some changes in the lambda code to handle different routes, we can test this routes :

![](https://cdn-images-1.medium.com/max/2000/1*sZ7jPv51wiWEkQRWzS7BpA.png)

We can recover our audio as response in the GET request :

![](https://cdn-images-1.medium.com/max/2620/1*lkg2HcgJWUU6lpx0XQwHEQ.png)

## CloudFront :

Since we want to add a nice looking front page for our api, we proceed to use CloudFront to serve a static HTML file hosted in the same S3 bucket created previously :

![](https://cdn-images-1.medium.com/max/3596/1*cwkNgoRqAOUnnjNEUulOww.png)

Now we configure our CloudFront distribution with the appropriate origin :

![](https://cdn-images-1.medium.com/max/2000/1*iAu5W2pZuyBPEbIUsrevhg.png)

![](https://cdn-images-1.medium.com/max/2000/1*Ew93Xgxm3CGlxLokfCto2g.png)

And we add the resource policy in the S3 bucket to allow CloudFront to access the HTML file :

![](https://cdn-images-1.medium.com/max/2000/1*LU7-swbYxY1LexmXg5FSfA.png)

## Final Result :

Now we can have a nice looking front page which sends a text to a serverless api and gets back the generated audio file, which obviously can be downloaded :

![](https://cdn-images-1.medium.com/max/2000/1*IyhqxcjMVtzyhBKZ4XIvfg.png)
