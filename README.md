# Serverless Scraping With Scrapy and AWS

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.com/) 

This guide explains how to write a Scrapy Spider, deploy it to AWS Lambda, and store scraped data in an S3 bucket.

- [Prerequisites](#prerequisites)
- [What is Serverless?](#what-is-serverless)
- [Getting Started](#getting-started)
- [Writing the Code](#writing-the-code)
- [Deploying To AWS Lambda](#deploying-to-aws-lambda)
- [Troubleshooting Tips](#troubleshooting-tips)

## Prerequisites

To accomplish this task, you’ll need the following:

- **Basic understanding of Python**: We’ll write the code in Python.
- **AWS (Amazon Web Services) Account**: Since we’re using AWS Lambda, you’ll need an AWS account.
- **A Linux or Windows machine with WSL 2**: Amazon uses _Amazon Linux_ to run code. When we upload our code, it needs to be binary compatible.
- **Basic knowledge of Scrapy**: Basic knowledge of scraping with Scrapy will be helpful.

## What is Serverless?

Serverless architecture eliminates the need to manage servers by charging only for actual usage. For instance, if your scraper runs for one minute a day, services like Lambda can be more cost-effective than a constantly running server.

**Pros**
- **Billing:** Pay only for what you use.
- **Scalability:** Automatic scaling.
- **Server Management:** No maintenance required.

**Cons**
- **Latency:** Cold starts can delay execution.
- **Execution Time:** Limited to 15 minutes maximum.
- **Portability:** Tied to the vendor’s ecosystem.

## Getting Started

### Setting Up Services

Log into your AWS account and create an S3 bucket. To do that, go to the [‘All services’](https://us-east-2.console.aws.amazon.com/console/services) page and scroll down.

![A list of all AWS services](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-108.png)

Click on _S3_—the first option in the _Storage_ section.

![The S3 storage service](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-109.png)

Next, click the _Create bucket_ button.

![Creating a new S3 bucket](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-110.png)

Give the bucket a name and choose your settings. For the purpose of following this guide, you can use default settings.

![Naming the new S3 bucket and choosing your settings](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-113.png)

Click the _Create bucket_ button located in the bottom right corner of the page.

![Creating the new S3 bucket](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-111.png)

The newly created bucket will show up in the _Buckets_ tab under _Amazon S3_.

![Clicking the create bucket button when the configuration is done](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-112.png)

### Setting Up Your Project

Create a new project folder.

```bash
mkdir scrapy_aws
```

Move into the new folder and create a virtual environment.

```bash
cd scrapy_aws
python3 -m venv venv  
```

Activate the environment.

```bash
source venv/bin/activate
```

Install Scrapy.

```bash
pip install scrapy
```

### What To Scrape

Let's use [books.toscrape](https://books.toscrape.com/) as our target site. It’s an educational site devoted entirely to web scraping. Each book is an `article` with the class name, `product_pod`. We want to extract all of these elements from the page.

![Inspecting one of the book in an article tag](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-114.png)

The title of each book is embedded within an `a` element which is nested inside of an `h3` element.

![Inspecting the book title](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-115.png)

Each price is embedded in a `p` which is nested inside of a `div`. It has a class name of `price_color`.

![Inspecting the book price](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-116.png)

## Writing the Code

Open a new Python file, `aws_spider.py`, and paste the following code into it:

```python
import scrapy


class BookSpider(scrapy.Spider):
    name = "books"
    allowed_domains = ["books.toscrape.com"]
    start_urls = ["https://books.toscrape.com"]

    def parse(self, response):
        for card in response.css("article"):
            yield {
                "title": card.css("h3 > a::text").get(),
                "price": card.css("div > p::text").get(),
            }
        next_page = response.css("li.next > a::attr(href)").get()

        if next_page:
            yield scrapy.Request(response.urljoin(next_page))
```

When you test the spider, it should output a JSON file full of books with prices:

```bash
python -m scrapy runspider aws_spider.py -o books.json
```

Let's create two handlers to run the spider: one to do it locally, one to do it on Lambda.

Here is the local handler, `lambda_function_local.py`:

```python
import subprocess

def handler(event, context):
    # Output file path for local testing
    output_file = "books.json"

    # Run the Scrapy spider with the -o flag to save output to books.json
    subprocess.run(["python", "-m", "scrapy", "runspider", "aws_spider.py", "-o", output_file])

    # Return success message
    return {
        'statusCode': '200',
        'body': f"Scraping completed! Output saved to {output_file}",
    }

# Add this block for local testing
if __name__ == "__main__":
    # Simulate an AWS Lambda invocation event and context
    fake_event = {}
    fake_context = {}

    # Call the handler and print the result
    result = handler(fake_event, fake_context)
    print(result)
```

Delete `books.json`. You can test the local handler with the following command:

```bash
python lambda_function_local.py
```

If everything is correct, you’ll see a new `books.json` in your project folder.

Here’s the handler for Lambda. It does basically the same, but stores the data to the S3 bucket:

```python
import subprocess
import boto3

def handler(event, context):
    # Define the local and S3 output file paths
    local_output_file = "/tmp/books.json"  # Must be in /tmp for Lambda
    bucket_name = "aws-scrapy-bucket"
    s3_key = "scrapy-output/books.json"  # Path in S3 bucket

    # Run the Scrapy spider and save the output locally
    subprocess.run(["python3", "-m", "scrapy", "runspider", "aws_spider.py", "-o", local_output_file])

    # Upload the file to S3
    s3 = boto3.client("s3")
    s3.upload_file(local_output_file, bucket_name, s3_key)

    return {
        'statusCode': 200,
        'body': f"Scraping completed! Output uploaded to s3://{bucket_name}/{s3_key}"
    }
```

- We first save the data to a temp file: `local_output_file = "/tmp/books.json"`. This prevents it from being lost.
- We upload it to the bucket with `s3.upload_file(local_output_file, bucket_name, s3_key)`.

## Deploying To AWS Lambda

Let's deploy to AWS Lambda. Make a _package_ folder:

```bash
mkdir package
```

Copy the dependencies over to the `package` folder.

```bash
cp -r venv/lib/python3.*/site-packages/* package/
```

Copy the files. Make sure you copy the handler you made for Lambda rather than the local handler.

```bash
cp lambda_function.py aws_spider.py package/
```

Compress the package folder into a zip file:

```bash
zip -r lambda_function.zip package/
```

Once the ZIP file has been created, head on over to AWS Lambda and select _Create function._ When prompted, enter your basic information such as runtime (Python) and architecture.

Make sure to add give it permission to access your S3 Bucket.

![Creating a new function](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-117.png)

Once you’ve created the function, select the _Upload from_ dropdown at the top right hand corner of the source tab.

![Lambda upload from](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-118.png)

Choose _.zip file_ and upload the ZIP file you created.

![Uploading the created ZIP file](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-119.png)

Click the _test_ button and wait for your function to run. After it runs, check your S3 Bucket and you should have a new file, _books.json_.

![The new books.json file in your S3 bucket](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-120.png)

## Troubleshooting Tips

### Scrapy Cannot Be Found

If you encounter an error stating that Scrapy isn’t found, include the following in your command array for `subprocess.run()`.

![Adding a piece of code in the subprocess.run() function](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-121.png)

### General Dependency Issues

You need to make sure your Python versions are the same. Check your local install of Python.

```bash
python --version
```

If this command outputs a different version than your Lambda function, change your Lambda configuration to match it.

### Handler Issues

![The handler should match the function you wrote](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-122.png)

Ensure your handler matches the function in `lambda_function.py`. For example, `lambda_function.handler` indicates that `lambda_function` is your file name and `handler` is the function name.

### Can’t Write to S3

If you encounter permissions issues when storing output, add the required permissions to your Lambda instance. To do this, go to the [IAM Console](https://console.aws.amazon.com/iam/), locate your Lambda function, and click the _Add permissions_ dropdown.

Click _Attach policies_.

![Clicking on 'attach policies' in the Lambda function](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-123.png)

Select _AmazonS3FullAccess_.

![Selecting AmazonS3FullAccess](https://brightdata.com/wp-content/uploads/2024/12/image-124-1024x214.png)

## Conclusion

If you are not into manual scraping, check out our [Web Scraper APIs](https://brightdata.com/products/web-scraper) and [ready-made datasets](https://brightdata.com/products/datasets). Sign up now to start your free trial!
