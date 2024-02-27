## Step 1: Prepare Your AWS Environment
IAM Role for Lambda: Ensure your AWS Lambda function has an IAM role attached with permissions that allow it to create new AWS keys and interact with the necessary services.

HashiCorp Vault Setup: Make sure your HashiCorp Vault is accessible from your AWS environment and properly configured to store AWS keys. This includes setting up the appropriate policies and authentication methods in Vault.

## Step 2: Create the Lambda Function
Lambda Runtime and Permissions: Choose a runtime for your Lambda function (Python, Node.js, etc.). Ensure the Lambda function's execution role has the necessary permissions to create AWS keys and make network requests to external services (like HashiCorp Vault).

Implement Key Creation: Use the AWS SDK within your Lambda function to create a new IAM user or role and generate new AWS access keys.

Implement Vault API Call: After generating the new AWS keys, make an API call to HashiCorp Vault to store these keys. You'll likely use the HTTP API provided by Vault, which means your Lambda function will need to authenticate with Vault and then use the appropriate API endpoint to write the keys into the secret engine you're using.

## Step 3: Triggering the Lambda Function
Decide how you want to trigger this Lambda function. AWS provides various options for triggering Lambda functions, including:

HTTP requests via Amazon API Gateway
Scheduled events using Amazon CloudWatch Events
Direct invocation via AWS SDKs or CLI

## Example Code Snippet
Here's a simplified Python example of how parts of the Lambda function might look, using the boto3 library for AWS interactions and the requests library for the Vault API call. This example assumes you're using the KV Secrets Engine in Vault.

```python
import boto3
import requests

def lambda_handler(event, context):
    # Create new AWS keys
    iam = boto3.client('iam')
    user_name = 'your-iam-user-name'
    response = iam.create_access_key(UserName=user_name)
    access_key = response['AccessKey']['AccessKeyId']
    secret_key = response['AccessKey']['SecretAccessKey']

    # Prepare Vault API call
    vault_addr = 'https://your-vault-address:8200'
    vault_token = 'your-vault-token'
    secret_path = 'aws/creds/my-role'  # Adjust based on your Vault setup
    headers = {'X-Vault-Token': vault_token}
    data = {'access_key': access_key, 'secret_key': secret_key}

    # Store keys in Vault
    vault_response = requests.post(f"{vault_addr}/v1/{secret_path}", headers=headers, json=data)
    if vault_response.ok:
        print("Keys stored successfully in Vault")
    else:
        print("Failed to store keys in Vault")

    return {
        'statusCode': 200,
        'body': 'Lambda execution completed'
    }
```
    
## Security Considerations
Vault Authentication: Ensure your Lambda function uses a secure method to authenticate with Vault. The example uses a static token, which is not recommended for production. Consider using more secure methods like AWS IAM authentication.
Minimal Permissions: Apply the principle of least privilege to both the Lambda execution role and the Vault policies, ensuring they only have permissions necessary for the task.
Secure Storage of Sensitive Information: Make sure any sensitive information (like Vault tokens) is stored securely, using AWS Secrets Manager or similar, instead of hardcoding in your Lambda function.

## Final Steps
Test your Lambda function thoroughly in a development environment before moving to production.
Monitor the function's execution and the AWS keys' usage to ensure everything is working as expected and securely.


## With Rotations 

## Step 1: Implement Credential Rotation Logic
Scheduled Execution: Use AWS CloudWatch Events (or Amazon EventBridge) to trigger your Lambda function on a schedule, such as daily or weekly, depending on your security requirements and usage patterns.

Rotation Logic in Lambda: Modify your Lambda function to include logic for rotating the credentials. This means creating new AWS access keys and deactivating or deleting the old ones.

## Step 2: Update Credentials in Vault
After generating new credentials, the next step is to update the credentials stored in Vault. This can be achieved by making an API call to Vault to overwrite the existing secrets or create new ones at a designated path.

## Step 3: Cleanup Old Credentials
Once the new credentials are stored securely in Vault and verified to be working (if possible, by a test mechanism), proceed to deactivate or delete the old credentials. This step is crucial to ensure that you don't accumulate unused credentials, which could be a security risk.

## Sample Lambda Function Update for Rotation
Below is an updated Python example that includes a basic approach to rotating credentials and updating them in Vault. Note that this example is simplified and needs to be adapted to fit your environment and security practices:

```python
import boto3
import requests

def rotate_credentials(iam, user_name):
    # Create a new access key
    new_key_response = iam.create_access_key(UserName=user_name)
    new_access_key = new_key_response['AccessKey']['AccessKeyId']
    new_secret_key = new_key_response['AccessKey']['SecretAccessKey']

    # Optionally, list and deactivate or delete old access keys
    # This is a simplified approach; consider your policy for handling old keys
    old_keys_response = iam.list_access_keys(UserName=user_name)
    for key in old_keys_response['AccessKeyMetadata']:
        if key['AccessKeyId'] != new_access_key:
            iam.delete_access_key(UserName=user_name, AccessKeyId=key['AccessKeyId'])

    return new_access_key, new_secret_key

def update_vault(vault_addr, vault_token, secret_path, access_key, secret_key):
    headers = {'X-Vault-Token': vault_token}
    data = {'access_key': access_key, 'secret_key': secret_key}
    response = requests.post(f"{vault_addr}/v1/{secret_path}", headers=headers, json=data)
    return response.ok

def lambda_handler(event, context):
    user_name = 'your-iam-user-name'
    vault_addr = 'https://your-vault-address:8200'
    vault_token = 'your-vault-token'
    secret_path = 'aws/creds/my-role'

    iam = boto3.client('iam')

    # Rotate AWS credentials
    new_access_key, new_secret_key = rotate_credentials(iam, user_name)

    # Update credentials in Vault
    if update_vault(vault_addr, vault_token, secret_path, new_access_key, new_secret_key):
        print("Credentials updated successfully in Vault")
    else:
        print("Failed to update credentials in Vault")

    return {
        'statusCode': 200,
        'body': 'Lambda execution and credentials update completed'
    }
```

## Security and Best Practices
Secure Vault Access: Ensure that the method your Lambda uses to authenticate to Vault is secure. AWS IAM roles with Vault's AWS authentication method can provide a secure, automatically rotated method of authentication.
Audit and Monitor: Set up auditing in Vault and CloudTrail in AWS to monitor access and changes to credentials.
Permissions Management: Apply the principle of least privilege both in AWS (for the Lambda function) and in Vault (for the credentials stored).
Test Before Deployment: Thoroughly test your rotation logic in a safe environment to ensure it behaves as expected without interrupting your service.
