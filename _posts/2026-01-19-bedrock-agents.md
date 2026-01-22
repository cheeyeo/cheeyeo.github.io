---
layout: post
show_meta: true
title: Introduction to Bedrock Agents
header: Introduction to Bedrock Agents
date: 2026-01-19 00:00:00
summary: Short introduction to Bedrock Agents
categories: aws bedrock agents lambda
author: Chee Yeo
---

[AWS Generative AI and AI Agents with Amazon Bedrock]: https://www.coursera.org/specializations/aws-generative-ai-developers

[Define function details]: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-function.html



In previous posts, I discussed how to build a simple agent using Gemini that runs in the CLI using MCP protocol to provides tools. AWS Bedrock has the same concept of an agent but it differs slightly compared to the previously discussed manual approach.

AWS Bedrock is a model-as-a-service platform whereby you could create generative AI applications by invoking foundational models offered on the platform through API calls. There is no need to host or train models manually as it's a fully managed service. Bedrock Agents allow you to build custom agents that leverages the foundational models offered by the same platform to create agentic workflows that can also integrate with existing AWS services such as Lambda and RDS.

As an example, assuming we are building a HR assistant agent that can query and update an employee's holidays. This example is based on the materials from [AWS Generative AI and AI Agents with Amazon Bedrock].

Firstly, we need to import the dependencies and create the bedrock clients using boto3:

{% highlight python %}
import json
import time
import zipfile
from io import BytesIO
import uuid
import pprint
import boto3

sts_client = boto3.client('sts')
iam_client = boto3.client('iam')
lambda_client = boto3.client('lambda')
bedrock_agent_client = boto3.client('bedrock-agent')
bedrock_agent_runtime_client = boto3.client('bedrock-agent-runtime')

session = boto3.session.Session()
region = session.region_name
account_id = sts_client.get_caller_identity()["Account"]
{% endhighlight %}

Next, we define the foundation models to use for the agent:
{% highlight python %}
  inference_profile = "us.amazon.nova-micro-v1:0"
  foundation_model = inference_profile[3:]
{% endhighlight %}

Bedrock agents don't use MCP protocol to invoke tools. Instead, it uses lambda functions which we will define shortly. We need to create a lambda function that will have the corresponding code to retrieve and update employees' holidays. We also need to store this information. To keep the example simple, we create a sqlite3 database which will be uploaded with the lambda code. The python snippet below generates some random employees and their vacation times:

{% highlight python %}
# creating employee database to be used by lambda function
import sqlite3
import random
from datetime import date, timedelta

# Connect to the SQLite database (creates a new one if it doesn't exist)
conn = sqlite3.connect('employee_database.db')
c = conn.cursor()

# Create the employees table
c.execute('''CREATE TABLE IF NOT EXISTS employees
                (employee_id INTEGER PRIMARY KEY AUTOINCREMENT, employee_name TEXT, employee_job_title TEXT, employee_start_date TEXT, employee_employment_status TEXT)''')

# Create the vacations table
c.execute('''CREATE TABLE IF NOT EXISTS vacations
                (employee_id INTEGER, year INTEGER, employee_total_vacation_days INTEGER, employee_vacation_days_taken INTEGER, employee_vacation_days_available INTEGER, FOREIGN KEY(employee_id) REFERENCES employees(employee_id))''')

# Create the planned_vacations table
c.execute('''CREATE TABLE IF NOT EXISTS planned_vacations
                (employee_id INTEGER, vacation_start_date TEXT, vacation_end_date TEXT, vacation_days_taken INTEGER, FOREIGN KEY(employee_id) REFERENCES employees(employee_id))''')

# Generate some random data for 10 employees
employee_names = ['John Doe', 'Jane Smith', 'Bob Johnson', 'Alice Williams', 'Tom Brown', 'Emily Davis', 'Michael Wilson', 'Sarah Taylor', 'David Anderson', 'Jessica Thompson']
job_titles = ['Manager', 'Developer', 'Designer', 'Analyst', 'Accountant', 'Sales Representative']
employment_statuses = ['Active', 'Inactive']

for i in range(10):
    name = employee_names[i]
    job_title = random.choice(job_titles)
    start_date = date(2015 + random.randint(0, 7), random.randint(1, 12), random.randint(1, 28)).strftime('%Y−%m−%d')
    employment_status = random.choice(employment_statuses)
    c.execute("INSERT INTO employees (employee_name, employee_job_title, employee_start_date, employee_employment_status) VALUES (?, ?, ?, ?)", (name, job_title, start_date, employment_status))
    employee_id = c.lastrowid

    # Generate vacation data for the current employee
    for year in range(date.today().year, date.today().year - 3, -1):
        total_vacation_days = random.randint(10, 30)
        days_taken = random.randint(0, total_vacation_days)
        days_available = total_vacation_days - days_taken
        c.execute("INSERT INTO vacations (employee_id, year, employee_total_vacation_days, employee_vacation_days_taken, employee_vacation_days_available) VALUES (?, ?, ?, ?, ?)", (employee_id, year, total_vacation_days, days_taken, days_available))

        # Generate some planned vacations for the current employee and year
        num_planned_vacations = random.randint(0, 3)
        for _ in range(num_planned_vacations):
            start_date = date(year, random.randint(1, 12), random.randint(1, 28)).strftime('%Y−%m−%d')
            end_date = (date(int(start_date[:4]), int(start_date[5:7]), int(start_date[8:])) + timedelta(days=random.randint(1, 14))).strftime('%Y−%m−%d')
            days_taken = (date(int(end_date[:4]), int(end_date[5:7]), int(end_date[8:])) - date(int(start_date[:4]), int(start_date[5:7]), int(start_date[8:])))
            c.execute("INSERT INTO planned_vacations (employee_id, vacation_start_date, vacation_end_date, vacation_days_taken) VALUES (?, ?, ?, ?)", (employee_id, start_date, end_date, days_taken.days))

# Commit the changes and close the connection
conn.commit()
conn.close()
{% endhighlight %}

The lambda function defines two functions **get_available_vacations_days** and **reserve_vacation_time** which will be added to the tool definition later:

{% highlight python %}
import os
import json
import stat
import shutil
import sqlite3
from datetime import datetime


shutil.copy('employee_database.db', '/tmp/employee_database.db')
os.chmod('/tmp/employee_database.db', stat.S_IRUSR | stat.S_IRGRP | stat.S_IROTH | stat.S_IWUSR | stat.S_IWGRP | stat.S_IWOTH)


def get_available_vacations_days(employee_id):
    conn = sqlite3.connect('/tmp/employee_database.db')
    c = conn.cursor()

    if employee_id:
        c.execute("""
            SELECT employee_vacation_days_available
            FROM vacations
            WHERE employee_id = ?
            ORDER BY year DESC
            LIMIT 1
        """, (employee_id,))

        available_vacation_days = c.fetchone()

        if available_vacation_days:
            available_vacation_days = available_vacation_days[0]
            conn.close()
            return available_vacation_days
        else:
            conn.close()
            return f"No vacation data found for employed_id {employee_id}"
    else:
        conn.close()
        raise Exception("No employee id provided")
    

def reserve_vacation_time(employee_id, start_date, end_date):
    conn = sqlite3.connect("/tmp/employee_database.db")
    c = conn.cursor()

    if not employee_id or not start_date or not end_date:
        conn.close()
        raise Exception("Missing required parameters")
    
    # Calculate vacation days
    start = datetime.strptime(start_date, '%Y-%m-%d').date()
    end = datetime.strptime(end_date, '%Y-%m-%d').date()
    vacation_days = (end - start).days + 1

     # Insert into planned_vacations
    c.execute("""
        INSERT INTO planned_vacations (employee_id, vacation_start_date, vacation_end_date, vacation_days_taken)
        VALUES (?, ?, ?, ?)
    """, (employee_id, start_date, end_date, vacation_days))

    # Update available vacation days
    c.execute("""
        UPDATE vacations 
        SET employee_vacation_days_available = employee_vacation_days_available - ?,
            employee_vacation_days_taken = employee_vacation_days_taken + ?
        WHERE employee_id = ? AND year = (SELECT MAX(year) FROM vacations WHERE employee_id = ?)
    """, (vacation_days, vacation_days, employee_id, employee_id))
    
    conn.commit()
    conn.close()
    
    return f"Reserved {vacation_days} vacation days from {start_date} to {end_date} for employee {employee_id}"


def lambda_handler(event, context):
    print(event)
    print(context)

    try:
        parameters = event.get("parameters", [])
        function_name = event.get("function")

        # Parse parameters
        params_dict = {}
        for param in parameters:
            params_dict[param.get("name")] = param.get("value")

        # Route to appropriate function
        if function_name == "get_available_vacation_days":
            employee_id = int(params_dict.get("employee_id"))
            result = get_available_vacations_days(employee_id)
        elif function_name == "reserve_vacation_time":
            employee_id = int(params_dict.get("employee_id"))
            start_date = params_dict.get("start_date")
            end_date = params_dict.get("end_date")
            result = reserve_vacation_time(employee_id, start_date, end_date)
        else:
            raise Exception(f"Unknown function: {function_name}")
        
        # return Bedrock expected format
        return {
            "response": {
                "actionGroup": event.get("actionGroup"),
                "function": event.get("function"),
                "functionResponse": {
                    "responseBody": {
                        "TEXT": {
                            "body": json.dumps({"result": result})
                        }
                    }
                }
            }
        }
    except Exception as e:
        return {
            "response": {
                "actionGroup": event.get("actionGroup"),
                "function": event.get("function"),
                "functionResponse": {
                    "responseBody": {
                        "TEXT": {
                            "body": json.dumps({"error": str(e)})
                        }
                    }
                }
            }
        }

{% endhighlight %}

After creating the lambda function above, we can start to create the agent. Note that we need to define an IAM role for the agent:

{% highlight python %}
agent_name = "hr-assistant-function-def"
agent_description = "Agent for providing HR assistance to manage vacation time"
agent_instruction = "You are an HR agent, helping employees understand HR policies and manage vacation time"
suffix = f"{region}-{account_id}"
agent_name = "hr-assistant-function-def"
agent_bedrock_allow_policy_name = f"{agent_name}-ba-{suffix}"
agent_role_name = f'AmazonBedrockExecutionRoleForAgents_{agent_name}'


bedrock_agent_bedrock_allow_policy_statement = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AmazonBedrockAgentBedrockFoundationModelPolicy",
            "Effect": "Allow",
            "Action": "bedrock:InvokeModel",
            "Resource": [
                f"arn:aws:bedrock:*::foundation-model/{foundation_model}",
                f"arn:aws:bedrock:*:*:inference-profile/{inference_profile}"
            ]
        },
        {
            "Sid": "AmazonBedrockAgentBedrockGetInferenceProfile",
            "Effect": "Allow",
            "Action":  [
                "bedrock:GetInferenceProfile",
                "bedrock:ListInferenceProfiles",
                "bedrock:UseInferenceProfile"
            ],
            "Resource": [
                f"arn:aws:bedrock:*:*:inference-profile/{inference_profile}"
            ]
        }
    ]
}

bedrock_policy_json = json.dumps(bedrock_agent_bedrock_allow_policy_statement)

agent_bedrock_policy = iam_client.create_policy(
    PolicyName=agent_bedrock_allow_policy_name,
    PolicyDocument=bedrock_policy_json
)

# Create IAM Role for the agent and attach IAM policies
assume_role_policy_document = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Service": "bedrock.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
    }]
}

assume_role_policy_document_json = json.dumps(assume_role_policy_document)
agent_role = iam_client.create_role(
    RoleName=agent_role_name,
    AssumeRolePolicyDocument=assume_role_policy_document_json
)

# Pause to make sure role is created
time.sleep(10)

iam_client.attach_role_policy(
    RoleName=agent_role_name,
    PolicyArn=agent_bedrock_policy['Policy']['Arn']
)

response = bedrock_agent_client.create_agent(
    agentName=agent_name,
    agentResourceRoleArn=agent_role['Role']['Arn'],
    description=agent_description,
    idleSessionTTLInSeconds=1800,
    foundationModel=inference_profile,
    instruction=agent_instruction,
)
agent_id = response['agent']['agentId']
{% endhighlight %}

Next, we create an action group to define a set of actions that the agent can perform on behalf of the user. In this example, we define the action group using function details as specified in [Define function details]:

{% highlight python %}
agent_functions = [
    {
        'name': 'get_available_vacations_days',
        'description': 'get the number of vacations available for a certain employee',
        'parameters': {
            "employee_id": {
                "description": "the id of the employee to get the available vacations",
                "required": True,
                "type": "integer"
            }
        }
    },
    {
        'name': 'reserve_vacation_time',
        'description': 'reserve vacation time for a specific employee - you need all parameters to reserve vacation time',
        'parameters': {
            "employee_id": {
                "description": "the id of the employee for which time off will be reserved",
                "required": True,
                "type": "integer"
            },
            "start_date": {
                "description": "the start date for the vacation time",
                "required": True,
                "type": "string"
            },
            "end_date": {
                "description": "the end date for the vacation time",
                "required": True,
                "type": "string"
            }
        }
    },
]

agent_action_group_response = bedrock_agent_client.create_agent_action_group(
    agentId=agent_id,
    agentVersion='DRAFT',
    actionGroupExecutor={
        'lambda': lambda_function['FunctionArn']
    },
    actionGroupName=agent_action_group_name,
    functionSchema={
        'functions': agent_functions
    },
    description=agent_action_group_description
)
{% endhighlight %}

The agent will determine from the user's conversation which action within an action group it needs to invoke. It will obtain all the required parameters and sends it to a Lambda function or returns control in response to an agent invocation. From the example above, we associate the lambda function created previously with the function schema. 

Next, we need to add a **Resource Policy** to the Lambda function to allow the Bedrock Agent to invoke the lambda directly:

{% highlight python %}
response = lambda_client.add_permission(
    FunctionName=lambda_function_name,
    StatementId='allow_bedrock',
    Action='lambda:InvokeFunction',
    Principal='bedrock.amazonaws.com',
    SourceArn=f"arn:aws:bedrock:{region}:{account_id}:agent/{agent_id}",
)
{% endhighlight %}

Finally, we create a draft version of the agent with an alias for testing:
{% highlight python %}
response = bedrock_agent_client.prepare_agent(
    agentId=agent_id
)

agent_id = response['agentId']

response = bedrock_agent_client.create_agent_alias(
    agentAliasName='test-alias-1',
    agentId=agent_id
)

agent_alias_id = response["agentAlias"]["agentAliasId"]
{% endhighlight %}

To invoke the agent, we call `invoke_agent` on the bedrock agent runtime client:
{% highlight python %}
## create a random id for session initiator id
session_id:str = str(uuid.uuid1())
enable_trace:bool = True
end_session:bool = False

# invoke the agent API
agentResponse = bedrock_agent_runtime_client.invoke_agent(
    inputText="How much vacation does the employee with employee_id set to 2 have available?",
    agentId=agent_id,
    agentAliasId=agent_alias_id, 
    sessionId=session_id,
    enableTrace=enable_trace, 
    endSession= end_session
)


event_stream = agentResponse['completion']
try:
    for event in event_stream:
        print(event)
        if 'chunk' in event:
            data = event['chunk']['bytes']
            print(f"Final answer ->\n{data.decode('utf8')}")
            agent_answer = data.decode('utf8')
            end_event_received = True
        elif 'trace' in event:
            logger.info(json.dumps(event['trace'], indent=2))
        else:
            raise Exception("unexpected event.", event)
except Exception as e:
    raise Exception("unexpected event.", e)
{% endhighlight %}

From above, we send a query to the agent to enquire how much holiday time employee number 2 has. The agent will forward the query to the foundational model which will parse the user query and invoke the matching lambda function in its tools definitions.

The following is an example output of the event object from the lambda function that was invoked from above:

{% highlight python %}
{'messageVersion': '1.0', 'function': 'get_available_vacations_days', 'parameters': [{'name': 'employee_id', 'type': 'integer', 'value': '2'}], 'sessionId': '331e8192-f7c9-11f0-9152-cbe55b511572', 'agent': {'name': 'hr-assistant-function-def', 'version': '1', 'id': 'HOJVVJ4OFP', 'alias': 'SNXM4Q0LRN'}, 'actionGroup': 'VacationsActionGroup', 'sessionAttributes': {}, 'promptSessionAttributes': {}, 'inputText': 'How much vacation does the employee with employee_id set to 2 have available?'}
{% endhighlight %}

![Bedrock Agent Response](/assets/img/aws/bedrock/agents/agent_response.png)


We can make another query to the agent to book 3 days of holidays for a specific employee as a test:

{% highlight python %}
agentResponse = bedrock_agent_runtime_client.invoke_agent(
    inputText="Great. please reserve time off for the employee with employee_id set to 2 from 2026-01-24 to 2026-01-26",
    agentId=agent_id,
    agentAliasId=agent_alias_id, 
    sessionId=session_id,
    enableTrace=enable_trace, 
    endSession= end_session
)

logger.info(pprint.pprint(agentResponse))

event_stream = agentResponse['completion']
try:
    for event in event_stream:        
        if 'chunk' in event:
            data = event['chunk']['bytes']
            print(f"Final answer ->\n{data.decode('utf8')}")
            agent_answer = data.decode('utf8')
            end_event_received = True
            # End event indicates that the request finished successfully
        elif 'trace' in event:
            logger.info(json.dumps(event['trace'], indent=2))
        else:
            raise Exception("unexpected event.", event)
except Exception as e:
    raise Exception("unexpected event.", e)
{% endhighlight %}

![Bedrock Agent Response](/assets/img/aws/bedrock/agents/agent_response_2.png)

The screenshot of the response indicated that the agent has booked the days off.

We can test that the agent has completed the task successfully by querying the remaining holidays for the same employee id again:

{% highlight python %}
agentResponse = bedrock_agent_runtime_client.invoke_agent(
    inputText="How much vacation does the employee with employee_id set to 2 have available?",
    agentId=agent_id,
    agentAliasId=agent_alias_id, 
    sessionId=session_id,
    enableTrace=enable_trace, 
    endSession= end_session
)

logger.info(pprint.pprint(agentResponse))

event_stream = agentResponse['completion']
try:
    for event in event_stream:        
        if 'chunk' in event:
            data = event['chunk']['bytes']
            print(f"Final answer ->\n{data.decode('utf8')}")
            agent_answer = data.decode('utf8')
            end_event_received = True
        elif 'trace' in event:
            logger.info(json.dumps(event['trace'], indent=2))
        else:
            raise Exception("unexpected event.", event)
except Exception as e:
    raise Exception("unexpected event.", e)
{% endhighlight %}

![Bedrock Agent Response](/assets/img/aws/bedrock/agents/agent_response_3.png)

The outputs above should show a reduction in the employee's holidays entitlement from original 22 days to 19 days.

From the perspective of a developer, the following are advantages for building on Bedrock Agents:
* Catalogue of foundational models to use
* Can version and test agents individually
* Use of Lambdas as tools for function calls
* Able to associate IAM roles and policies for both agents and tools
* Able to integrate with other AWS services such as IAM
* Serverless approach means infrastructure is managed for you
* Use of available frameworks such as LangChain and Kiro to speed up development

[github repo]: https://github.com/cheeyeo/AWS_GENERATIVE_AI_FOR_DEVELOPERS/blob/main/genai-exercise4-agents.ipynb

A working example of the agent can be found at this [github repo]
