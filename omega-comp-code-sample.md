---
layout: default
title: Omega Comp Code Samples
description: 
---

This page includes the code for the spectrum analyzer and the license authentication service I built as a part of this project.

[back to main page](./)

# License authentication service (Python, AWS Lambda, and AWS DynamoDB)

```python
# This is an AWS Lambda function I wrote for Omega Comp in Python to properly verify that a customer has indeed purchased my product.
# The user submits their email and license key, and this function checks an AWS DynamoDB (repeat license authentication requests) and 
# a third-party API connected to my webstore (first time license authentication requests) to verify that they should indeed
# be allowed access. From there, they are sent an encrypted (RSA) version of the product name they are verifying, their email, and most 
# importantly their machine ID, which prevents unauthorized license file sharing between computers. Sharing an email and license key pair 
# with a third party is enough information to allow that third party to initiate a license transfer (typically for profit or other personal 
# gain), which renders the original email and license pair useless. Thus, a situation in which two users share a single licesnse becomes 
# risky for both parties, and not viable for large-scale license sharing. 

import json
from typing import Dict, Any
import boto3
from botocore.exceptions import ClientError
import requests
import os 

def lambda_handler(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
    # Constants
    # The maximum number of machines a user can register a key on
    MACHINE_USAGE_LIMIT = 4 

    # Initialize DynamoDB client
    dynamodb = boto3.resource('dynamodb')
    
    # Specify the table name
    table_name = 'key_table'
    table = dynamodb.Table(table_name)

    # Parse the request body
    body = event.get('body')
    if not body:
        return {
            'statusCode': 400,
            'body': json.dumps({'issues': 'Missing request body'})
        }

    try:
        # Unpack important information from the request
        body_json = json.loads(body)
        request_key_value = body_json.get('primary_key')
        request_email = body_json.get('email')
        request_product = body_json.get('product')
        request_machine_id = body_json.get('machine_id')

        # Bad request
        if not all([request_key_value, request_email, request_product, request_machine_id]):
            return {
                'statusCode': 400,
                'body': json.dumps({'issues': 'Missing required fields'})
            }

    # Bad request
    except json.JSONDecodeError:
        return {
            'statusCode': 400,
            'body': json.dumps({'issues': 'Invalid JSON in request body'})
        }
    
    try:
        # Attempt to get the entry associated with the key from the database
        response = table.get_item(
            Key={
                'key_name': request_key_value
            }
        )
        
        # Check if the item exists in the database
        if 'Item' not in response:
            # Key not in table, check shopify site to see if this is a key for Omega Comp that has been purchased but not yet registered
            edp_url = "https://app-easy-product-downloads.fr/api/get-license-key?license_key=" + request_key_value + "&api_token=" + get_edpapi_token()
            edp_response = requests.post(edp_url)
            edp_json = edp_response.json()

            if edp_json["status"] == "success":
                # Key is valid, but not registered in database

                # Add new key to database
                table.put_item(
                    Item={
                        "key_name": request_key_value,
                        "email": "none",
                        "machine_ids": {"none"},
                        "nfr": False,
                        "product_name": "omega_comp", 
                        "registrations": 0,
                        "shredded": False
                    }
                )

                # Update response 
                response = table.get_item(
                    Key={
                        'key_name': request_key_value
                    }
                )
            else:
                # Key doesn't exist, return that this key is invalid
                return {
                    'statusCode': 404,
                    'body': json.dumps({'issues': 'Invalid license key'})
                }

        # Item exists, unpack important data
        item = response['Item']

        response_key = item['key_name']  
        response_email = item['email']
        response_machine_ids = item['machine_ids']
        response_product_name = item['product_name']
        response_registrations = item['registrations']
        response_shredded = item['shredded']


        # Validity checks, ALL of these must be true in order to verify the key

        # Do the keys actually match? This is mostly for redundancy 
        key_valid = response_key == request_key_value 

        # Does the email address associated with the key match, or is the key unclaimed
        email_valid = (response_email == "none") or (response_email == request_email) 

        # Is this key for the correct product? This line allows this configuration to work for all the products a given company may 
        # have available, not just one product
        product_name_valid = response_product_name == request_product 

        # Check that the key has not been manually deactivated
        not_shredded = not response_shredded

        # Check that the user hasn't reached the limit on the number of machines they're allowed to activate their key on
        machine_limit_not_reached = len(response_machine_ids) <= MACHINE_USAGE_LIMIT

        # Has the user provided a valid license key? 
        valid = key_valid and email_valid and not_shredded and product_name_valid and machine_limit_not_reached
        
        # If key is not valid, return and do not proceed
        if not valid:
            return {
                'statusCode': 400,
                'body': json.dumps({'issues': 'Invalid request'})
            }
        
        # Encrypt response here such that the database updates at the VERY last step
        enc_product_name = str(apply_to_value(string_to_int(response_product_name), get_kp_one(), get_kp_two()))
        enc_machine_id = str(apply_to_value(string_to_int(request_machine_id), get_kp_one(), get_kp_two()))
        enc_email = str(apply_to_value(string_to_int(request_email), get_kp_one(), get_kp_two()))

        # Try to update the item
        try:
            response_machine_ids.add(request_machine_id)
            response = table.update_item(
                Key={
                    'key_name': request_key_value
                },
                UpdateExpression='SET email = :email_val, machine_ids = :machine_id_val, registrations = :registrations_val',
                ExpressionAttributeValues={
                    ':email_val': request_email,
                    ':machine_id_val': response_machine_ids,
                    ':registrations_val': response_registrations + 1
                },
                ReturnValues="UPDATED_NEW"
            )

        # Problem updating the database
        except ClientError as e:
            return {
                'statusCode': 500,
                'body': json.dumps({'issues': 'Error processing data'})
            }

        # Authentication successful, return encrypted user information
        return {
            'statusCode': 200,
            'body': json.dumps({
                'issues': 'none',
                'name': enc_product_name,
                'id': enc_machine_id,
                'user': enc_email
            })
        }
    # Something unexpected went wrong, inform the user 
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'issues': 'Unknown error'})
        }

# Convert string to integer, use big endian
def string_to_int(s: str) -> int:
    return int.from_bytes(s.encode('utf-8'), 'big')

# Encrypt our response with the private key, so the client can verify that this response is legitimate using the public key
def apply_to_value(value: int, key_part1: str, key_part2: str) -> int:
    result = 0
    part1 = int(key_part1, 16)
    part2 = int(key_part2, 16)

    if part1 == 0 or part2 == 0 or value <= 0:
        return result

    while value != 0:
        result = result * part2
        value, remainder = divmod(value, part2)
        result += pow(remainder, part1, part2)

    return result



# Functions for obtaining secrets from environment variables

# Get encryption keys for encrypting user data once a request has been verified
def get_kp_one() -> str:
    return os.environ["kp_one"]

def get_kp_two() -> str:
    return os.environ["kp_two"]

# Get api key for shopify api
def get_edpapi_token() -> str:
    return os.environ["edp_api"]
```

# Spectrum Analyzer Code (C++ and JUCE framework)

Todo: add the code

[back to main page](./)