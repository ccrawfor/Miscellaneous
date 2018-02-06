# Exploring AWS Serverless Framework

## Description

A simple example exploring the use of AWS Servless Framework and Lambda based applications.

This example captures temperature data from our device and post the results to a Kinesis Stream where we will have a Lambda function that receives the Kinesis event records and processes them by comparing the measurement values to a specified threshold and trigger an alert via AWS SNS.

We will also leverage AWS's API Gateway to update the devices threshold.

# Producer

Our sensor data is pulled from a messaging queue and batched into sets of 100 records.  

```
 utility = Utility()
 #Record set
 RecordSet = utility.getMeasurement(name='temperature', count=100)
 
 kinesis_client = boto3.client('kinesis', region_name='us-east-2')
        
        put_response = kinesis_client.put_records(
            Records=RecordSet,
            StreamName='telemetry'    
        ) 
 
```


# Consumer


Next is our Lambda function that receives the Kinesis event records and processes them by comparing the measurement values to a specified threshold.  


The function device\_handler is our Lambda Function Handler.  Note our device_handler is invoked asynchronously therefore any return values are discarded.

```

def device_handler(event, context):

    records = event['Records']
    for x in records:
        data=x['kinesis']['data']
        decodedData = json.loads(base64.b64decode(data))
        #perform a dynamodb lookup
        threshold = getThreshold(decodedData)
        if threshold < decodedData['value']:
            sendAlert(decodedData)
            
```

AWS Dynamodb is used to store basic device configuration.  However, with AWS, additional considerations should be given towards the anticipated amount of throughput in the form of writes and reads you expect to incur.   

You could read this from an object stored in S3 and subsequently update that object once an upate event is received from the gateway.


```
def getThreshold(record):

    dynamodb = boto3.resource('dynamodb', region_name='us-east-2')
    table = dynamodb.Table('Sensors')
    deviceId = str(record['id'])
    measurement = str(record['measurement'])

    threshold = table.get_item(
        Key={
            "device-id": deviceId,
            "measurement": measurement
        },
        AttributesToGet=[
            'threshold',
        ],
    ) 

    if not threshold.get("Item", None):    
        return 0
    else:
        return int(threshold.get("Item")["threshold"])

        
```
If the threshold is exceeded we will send an alert using AWS SNS (Simple Notification Service)


```

def sendAlert(data):

	 measuredValue = decodedData['value']

    kinesis_client = boto3.client('sns', region_name='us-east-2')
    response = kinesis_client.publish(TopicArn='Your ARN',
    			Message=json.dumps({'default': 'Default Message',
                                    'email': 'Temperature Alert 								Exceeded:' + str(value)}),
                Subject='Alert',
                MessageStructure='json'
            )    
    
      
```

Next we will setup a simple API Gateway to update the devices threshold.  

AWS Serverless Application Model (AWS SAM) template snippet

```
UpdateThreshold:
          Type: Api
          Properties:
            Path: /devices/{device-id}
            Method: post
            
```

Handler

```
def updateThreshold(event):

    dynamodb = boto3.resource('dynamodb', region_name='us-east-2')

    table = dynamodb.Table('Sensors')
    deviceId = event.get('pathParameters')['device-id']
    threshold = json.loads(event.get('body'))['threshold']
    try:
        item = table.update_item(
            Key={
                    "device-id": deviceId,
                    "measurement": "temperature"
                    },
            UpdateExpression="Set threshold = :n",
            ExpressionAttributeValues={
                    ':n': threshold
                },
                ReturnValues="UPDATED_NEW"
            )
        return item['ResponseMetadata']['HTTPStatusCode']    
    except ClientError as e:
        return 500

def action(method,event):
   return {
        "GET": getCurrentMeasurement,
        "POST": updateThreshold,
    }.get(method, 'GET')

def returnMsg(code):
    return {
        200: "Update Successful",
        500: "Internal Server Error"
    }.get(code, 500)

def device_handler(event, context):
    
    resp = action(event.get('httpMethod'),event)(event)
    message={
        "body": returnMsg(resp)
     }
    return message


```
