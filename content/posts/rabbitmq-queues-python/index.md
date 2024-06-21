---
title: "Efficiently Managing RabbitMQ Queues with Python and REST API"
date: 2024-06-18
draft: false
summary: "Struggling to manage a large number of RabbitMQ queues? In this blog post, we'll show you how to tackle this challenge head-on using Python and regular expressions. Discover how to set up `rabbitmqadmin`, retrieve queues in batches, filter them using regex, and safely delete unnecessary ones. We'll explore the RabbitMQ Management HTTP API and share best practices for exception handling and result storage. By automating queue management, you'll save time and effort in maintaining your messaging infrastructure. Join us as we dive into the world of efficient RabbitMQ queue management and take control of your messaging system!"
tags: ["rabbitmq", "quueues", "requests", "celery", "ignore_result"]
---


## Introduction

[RabbitMQ{{< icon "link" >}}](https://www.rabbitmq.com/) is a powerful message-queueing system that forms the backbone of many distributed applications. However, as the number of queues grows, managing them can become a daunting task. In this blog post, we'll explore how to efficiently manage RabbitMQ queues using the [`rabbitmqadmin`{{< icon "link" >}}](https://www.rabbitmq.com/docs/management-cli) tool and Python scripts.

We will dive into the challenges that arise when dealing with a large number of queues and how to overcome them using Python and the [RabbitMQ Management{{< icon "link" >}}](https://www.rabbitmq.com/docs/management) HTTP API.

You'll learn how to:
- Retrieve queues in batches
- Filter queues based on specific criteria using regular expressions
- Safely delete unnecessary queues

We'll also discuss best practices for:
- Handling exceptions
- Storing the results for further analysis

By the end of this post, you'll have a solid understanding of how to automate the management of RabbitMQ queues, saving you time and effort in maintaining your messaging infrastructure. Let's get started!

## Tools Used

In this blog post, we'll be using the following tools and libraries:

### RabbitMQ
- [**RabbitMQ**{{< icon "link" >}}](https://www.rabbitmq.com/): A powerful message-queueing system that enables reliable communication between distributed application components.
- [**RabbitMQ Management Console**{{< icon "link" >}}](https://www.rabbitmq.com/docs/management): A web-based user interface for managing and monitoring RabbitMQ servers and queues.

### Python
- Python: A versatile programming language used for scripting and automating tasks.
- Python Libraries:
  - `yaml` [{{< icon "link" >}}](https://pypi.org/project/PyYAML/): Used for parsing YAML configuration files.
  - `requests` [{{< icon "link" >}}](https://pypi.org/project/requests/): Used for making HTTP requests to the RabbitMQ Management HTTP API.
  - `re` [{{< icon "link" >}}](https://docs.python.org/3/library/re.html): Used for working with regular expressions to filter queues based on specific criteria.

## Assumptions and Prerequisites

Before diving into the blog post, we assume the following:

1. You have a basic understanding of RabbitMQ and its concepts, such as queues, exchanges, and bindings.
2. You have RabbitMQ installed and running on your system.
3. You have access to the RabbitMQ Management Console and the necessary credentials (username and password) to authenticate.
4. You have Python installed on your system, along with the required libraries (`yaml`, `requests`, and `re`).
5. You are comfortable working with the command line and running Python scripts.

With these tools and assumptions in mind, you'll be well-equipped to follow along with the examples and techniques presented in this blog post to efficiently manage your RabbitMQ queues.

## Genesis of this blog

The rabbitmqadmin tool is a convenient wrapper around RabbitMQ's Management Console HTTP REST APIs, but it has some limitations. My team accidentally pushed around 200,000 (2 hundred thousand or 2 lakhs) undesired queues into the RabbitMQ cluster due to a typo in the Python Celery code.

Let us discuss how we end up here.

### The Dangers of Not Using ignore_result in Celery Tasks

> This is a slight detour
> Skip? [{{< icon "link" >}}]({{< ref "posts/rabbitmq-queues-python#management-console-rest-api" >}})

When working with [Celery](https://docs.celeryq.dev/en/main/index.html), a popular distributed task queue library for Python, it's crucial to be mindful of how you configure your tasks. One often overlooked setting is [`ignore_result`](https://docs.celeryq.dev/en/main/userguide/calling.html#results-options), which can have significant implications for the performance and stability of your RabbitMQ message broker.

By default, when a Celery task is executed, the result of the task is stored in a result backend, such as Redis or RabbitMQ. This allows the worker to retrieve the result later if needed. However, if you don't explicitly set `ignore_result=True` for tasks that don't require the result to be stored, Celery will automatically create a result queue for each task.

Here's where the problem lies. If you have a high volume of tasks being processed and you're not using `ignore_result`, RabbitMQ can quickly become overwhelmed with a large number of result queues. Each task execution will create a new queue, leading to a rapid accumulation of queues in RabbitMQ.

The consequences of this can be severe:

1. **Increased Resource Consumption**: As the number of queues grows, RabbitMQ requires more memory and CPU resources to manage and maintain them. This can put a significant strain on your RabbitMQ server, leading to increased resource utilization and potentially causing performance issues.

2. **Reduced Performance**: When RabbitMQ is burdened with a large number of queues, it can start to slow down. The processing of messages and the overall throughput of the system can be impacted, resulting in longer task execution times and reduced responsiveness.

3. **Instability and Crashes**: In extreme cases, the excessive number of queues can cause RabbitMQ to become unstable or even crash. The server may struggle to handle the sheer volume of queues and the associated overhead, leading to service disruptions and potential data loss.

To mitigate these issues, it's important to use the `ignore_result` parameter judiciously in your Celery tasks. If you don't need the result of a task to be stored, make sure to set `ignore_result=True` when defining the task. This tells Celery not to create a result queue for that task, reducing the burden on RabbitMQ.

Here's an example of how to set `ignore_result` in a Celery task:

```python
@app.task(ignore_result=True)
def my_task():
    ## Task logic goes here
    pass
```

By setting `ignore_result=True`, you ensure that no result queue is created for the task, thereby preventing the accumulation of unnecessary queues in RabbitMQ.

So my team mate mistyped `ignore_result` to `ignore_results`, and as python is python, it gave no warnings at all. This led to a sudden increase in queues and a sudden degradation of rabbitmq cluster performance.

## Management Console REST API


The RabbitMQ management UI and [`rabbitmqadmin`{{< icon "link" >}}]({{< ref "posts/rabbitmqadmin-intro" >}})[{{< icon "link" >}}](https://www.rabbitmq.com/docs/management-cli)  have limited capabilities when it comes to performing bulk operations on queues, exchanges, bindings, etc. For example, if you need to delete hundreds of queues matching a certain naming pattern, it would be quite tedious to do this through the UI or rabbitmqadmin. The REST API allows you to programmatically perform operations on many objects much more efficiently.

### Why learn about it?
> Another slight detour
> Skip? [{{< icon "link" >}}]({{< ref "posts/rabbitmq-queues-python#too-many-queues" >}})

1. The management UI doesn't provide a way to export or save the configuration and state of RabbitMQ objects. Using the REST API, you could write a script to get the definitions of all exchanges, queues, bindings, etc. and save them to a file. This could be useful for backing up the configuration or copying it to another RabbitMQ cluster.

2. Advanced configuration tasks like setting policies and parameters are not possible through the rabbitmqadmin tool, you can only do this through the management UI or REST API. Using the API would allow you to automate these configuration changes.

3. There are some settings and statistics which are only accessible via the REST API and not exposed in the management UI, such as detailed memory usage breakdown. Accessing the REST API programmatically would allow you to retrieve this additional information.

4. The management UI and rabbitmqadmin are great for interactive, ad-hoc administration and monitoring of a RabbitMQ cluster. But for more advanced automation, configuration management, monitoring, and bulk operations, using the REST API directly provides a lot more power and flexibility. Most things you can do in the UI, and more, can be done through the API.

## Too Many Queues

The main issue we encountered was the presence of too many queues, which led to connection timeouts when making HTTP requests through the management console or rabbitmqadmin. To address this problem, we decided to write a custom Python program that interacts directly with the RabbitMQ REST API.

Our Python programs will perform the following steps:
1. Load the RabbitMQ configuration from a YAML file.
2. Retrieve queues in batches using the REST API.
3. Process the retrieved queues and identify the ones to be deleted.
4. Store the queues to be deleted in a separate file.
5. Delete the identified queues using the REST API.

## The programs

```python
# get_queues.py
import yaml
import requests
import re

# Load the configuration from the YAML file
with open('rabbitmq.conf', 'r') as file:
    config = yaml.safe_load(file)

# Extract the configuration values
hostname = config['hostname']
port = config['port']
username = config['username']
password = config['password']
vhost = config['vhost']

# Set the API endpoint URL
api_url = f"http://{hostname}:{port}/api/queues/{vhost}"

# Set the batch size
batch_size = 100

# Set the initial offset
offset = 1

# Open the file to store the queues
with open('queues_to_delete.txt', 'w') as file:
    # Download queues in batches
    while True:
        # Set the pagination parameters
        params = {
            'page': offset,
            'page_size': batch_size,
            'columns': 'name,messages'
        }

        try:
            # Send a GET request to the API endpoint with authentication
            response = requests.get(api_url, auth=(username, password), params=params)

            # Check if the request was successful
            if response.status_code == 200:
                try:
                    # Parse the JSON response
                    queues = response.json()['items']

                    # Process the retrieved queues
                    for queue in queues:
                        queue_name = queue['name']
                        message_count = queue['messages']
                        print(f"Queue: {queue_name}, Messages: {message_count}")

                        # Check if the queue name does not contain "arjun" using regex
                        if not re.search(r'arjun', queue_name):
                            file.write(queue_name + '\n')

                    # Break the loop if no more queues are retrieved
                    if len(queues) < batch_size:
                        break

                    # Increment the offset for the next batch
                    offset += 1

                except (KeyError, TypeError, ValueError) as e:
                    print(f"Error parsing JSON response: {e}")
                    print("Skipping to the next batch...")
                    offset += 1
                    continue

            else:
                print(f"Error: {response.status_code} - {response.text}")
                break

        except requests.exceptions.RequestException as e:
            print(f"Error: {e}")
            break
```

```python
# delete_queues.py
import yaml
import requests

# Load the configuration from the YAML file
with open('rabbitmq.conf', 'r') as file:
    config = yaml.safe_load(file)

# Extract the configuration values
hostname = config['hostname']
port = config['port']
username = config['username']
password = config['password']
vhost = config['vhost']

# Set the API endpoint URL
api_url = f"http://{hostname}:{port}/api/queues/{vhost}"

# Open the file to read the queues
with open('queues_to_delete.txt', 'r') as file:
    queues_to_delete = file.read().splitlines()

# Delete the queues that don't contain "arjun"
for queue_name in queues_to_delete:
    try:
        # Send a DELETE request to the API endpoint with authentication
        delete_url = f"{api_url}/{queue_name}"
        response = requests.delete(delete_url, auth=(username, password))

        # Check if the request was successful
        if response.status_code == 204:
            print(f"Deleted queue: {queue_name}")
        else:
            print(f"Error deleting queue '{queue_name}': {response.status_code} - {response.text}")

    except requests.exceptions.RequestException as e:
        print(f"Error deleting queue '{queue_name}': {e}")
```

Let's dive into the Python code and understand how it works.

### Loading Configuration

We start by loading the RabbitMQ configuration from a YAML file named `rabbitmq.conf`. The configuration file contains the following attributes:

```yaml
hostname: localhost
port: 15672
username: your_username
password: your_password
vhost: your_vhost
```

We use the `yaml` module to parse the configuration file and extract the necessary values.

### Retrieving Queues

To retrieve queues efficiently, we use a pagination approach. We define a batch size and an initial offset. We then make GET requests to the RabbitMQ REST API endpoint `/api/queues/{vhost}` with the appropriate pagination parameters.

```python
params = {
    'page': offset,
    'page_size': batch_size,
    'columns': 'name,messages'
}
```

We retrieve queues in batches until we have processed all the available queues.

### Processing Queues

As we retrieve the queues, we process them to determine which ones need to be deleted. In this example, we use a regular expression to check if the queue name does not contain the string "arjun".

```python
if not re.search(r'arjun', queue_name):
    file.write(queue_name + '\n')
```

We store the names of the queues to be deleted in a separate file named `queues_to_delete.txt`.

### Deleting Queues

After storing the queues to be deleted, we read the `queues_to_delete.txt` file and send DELETE requests to the RabbitMQ REST API endpoint `/api/queues/{vhost}/{queue_name}` to delete each queue.

```python
delete_url = f"{api_url}/{queue_name}"
response = requests.delete(delete_url, auth=(username, password))
```

We handle any errors that may occur during the deletion process and log the appropriate messages.

### Enhancements

To further improve the solution, we made a few enhancements:

1. We added exception handling to gracefully handle any errors that may occur during the JSON parsing or API requests.
2. Instead of using a simple string comparison to identify queues for deletion, we switched to using a regular expression to search for specific queues. This provides a more precise way to match specific queue names.

```bash
grep -E '[0-9]{32}_arjun' queues_to_delete.txt
```

## Conclusion

By leveraging the RabbitMQ REST API and writing a custom Python program, we were able to efficiently manage a large number of queues. The solution allows us to retrieve queues in batches, process them based on specific criteria, store the queues to be deleted, and delete them accordingly.

This approach overcomes the limitations of the management console and `rabbitmqadmin` tool when dealing with a high volume of queues. It provides a scalable and automated way to maintain a clean and organized RabbitMQ setup.

Remember to adapt the code to your specific requirements and test it thoroughly in a non-production environment before running it on your production RabbitMQ instance.

Happy queue management!


